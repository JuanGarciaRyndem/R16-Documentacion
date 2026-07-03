# Revisión — [R16A-RE-FU-008][DIS-SOL] Diseño de la solución

| Campo | Valor |
|---|---|
| **Documento revisado** | `[R16A-RE-FU-008][DIS-SOL] Diseño de la solución.pdf` |
| **Versión revisada** | 1.0 |
| **Fecha de emisión del PDF** | 26 jun 2026 |
| **Autor del diseño** | Carlos Iván Morales Carreón |
| **Revisor** | Juan David García Cruz |
| **Fecha de la revisión** | 2026-06-29 |
| **Criterios usados** | 7 requisitos planteados verbalmente sobre rediseño Mailbot + clasificación múltiple + bajada a modelos por proceso + lectura de PDF/imágenes |
| **Documentos cruzados** | `R16A-RE-FU-008_BD.md` v1.0, `R16A-RE-FU-008-P2-Back.md` v1.0, `R16A-RE-FU-008-v2_Propuesta2.md`, `R16A-RE-FU-008-Faltante-Bitacora-Reclasificacion.md` |

---

## Resumen ejecutivo

El diseño es **muy sólido en la arquitectura** (Gmail Push + Pub/Sub + Worker .NET 10, idempotencia, DLQ, bitácora `MailbotEventoCorreo`/`MailbotClasificacionLog`, ETL a Legacy vía `ProquifaDotNet.LegacyBridge`) y resuelve correctamente puntos como segregación regional, retroactividad de Bandeja del Coordinador, decisiones D1–D5 y P5–P20. **Cobertura ~75% de los 7 puntos del requisito**, con **3 brechas funcionales críticas** que no resuelve y que deben atenderse antes de pasar a construcción.

| # | Punto del requisito | Estado en el diseño |
|---|---|---|
| 1 | Rediseño del procesamiento de Mailbot | ✅ Cubierto (arquitectura Gmail Push + Pub/Sub + Worker .NET 10) |
| 2 | Clasificar 4 buzones (Requisición/Cotización, Pedido, Cobros, Otros) | ✅ Cubierto en clasificación; ⚠️ buzón Pedidos limitado a `pcPromesaDeCompra` |
| 3 | Selección y entrenamiento del Agente IA | ✅ Proveedor (Gemini Flash-Lite via Vertex AI) + ⚠️ "entrenamiento" reducido a prompts `.txt` — divergencia conceptual |
| 4 | Procesar título + cuerpo + adjuntos | ✅ DTO contempla los 3 campos |
| 5 | **Clasificación MÚLTIPLE (un correo puede ser de 2 o 3 categorías)** | ❌ **NO CUBIERTO** — el modelo asume una sola clasificación por correo |
| 6 | Bajar a modelos por proceso (cotCotizacion + partidas; pcPromesaDeCompra + ppPedido + partidas; pendiente cobro) | ⚠️ Parcial — falta `cotPartidaCotizacion`, `ppPedido` y partidas |
| 7 | Lectura de PDF e imágenes en adjuntos | ⚠️ No explícito — Gemini lo soporta nativamente pero el diseño no lo declara |

---

## Hallazgos críticos (bloqueantes)

### H-01 — Clasificación múltiple NO contemplada

**Punto del requisito:** *"Terminando de procesar o entender el contexto de cada correo debe de ser capaz de categorizar el correo, incluso se puede poner en más de una categoría. Un correo puede ser de cotización y pedido, o cotización y cobro, combinaciones entre los 3 o los 3 buzones."*

**Estado en el diseño:**
- Sección 3 (Interfaces del dominio) — `ResultadoClasificacionDto` tiene **una sola propiedad** `string Clasificacion` con valores `"cotizacion" | "pedido" | "cobro" | "otros"`.
- Sección "0. Vista de alto nivel" — el flowchart CLASIF tiene salidas únicas mutuamente excluyentes: COBRO ó PEDIDO ó REQUISICION ó OTROS.
- Sección 13 (Detalle entidades insertadas) — *"El esquema permite 1:N pero **en R16 cada correo tiene exactamente una clasificación**"* (comentario sobre `CorreoRecibidoCliente`).
- Sección 14 (paso 14): *"Si `clasificacion == 'cobro'` → INSERT `fccFolioPagoCliente`"*. Es un `if` excluyente, no un fan-out.

**Problema:** el diseño rechaza explícitamente la clasificación múltiple. Si un correo entra y dice "te adjunto la cotización aprobada (Pedido) y el comprobante de pago (Cobro)", el sistema lo clasifica una sola vez, genera un solo pendiente, y queda invisible para los otros 2 buzones que también deberían procesarlo.

**Acción recomendada:**
1. **Modelo de datos:** aprovechar que `CorreoRecibidoCliente` ya es 1:N (el esquema lo permite — confirmado en el propio diseño) e insertar N filas (una por clasificación detectada), no 1. Reflejar la decisión en el documento (eliminar la nota *"en R16 cada correo tiene exactamente una clasificación"*).
2. **Agente IA:** cambiar `ResultadoClasificacionDto.Clasificacion: string` a `IReadOnlyList<ClasificacionConDatos>`, donde cada elemento tiene `{ clasificacion, confianza, datosExtraidos }`. El prompt `clasificacion_correo.txt` y el `ResponseSchema` de Vertex AI deben permitir array de 1..4 elementos.
3. **Flujo principal:** convertir el switch del paso 14 en un loop:
   ```
   foreach (var c in resultado.Clasificaciones) {
     if (c.clasificacion == "cobro")      → fccFolioPagoCliente
     if (c.clasificacion == "pedido")     → pcPromesaDeCompra + ppPedido + partidas
     if (c.clasificacion == "cotizacion") → cotCotizacion + cotPartidaCotizacion
   }
   ```
4. **Bitácora:** `MailbotClasificacionLog` debe registrar 1..N filas por correo (una por clasificación), o agregar columna `ClasificacionesJSON` para guardar el array completo.
5. **Buzones / UI:** confirmar que el mismo correo aparezca simultáneamente en los 3 buzones si fue clasificado en los 3; el filtro por `IdCatClasificacionCorreoRecibido` ya soporta esto sin cambios si el modelo es 1:N.
6. **Reclasificación:** el endpoint actual `PUT /api/buzon/cobros/{id}/reclasificar` opera sobre `IdCorreoRecibidoCliente` (la fila específica del buzón), no sobre el correo entero — esto funciona bien con el modelo 1:N, solo hay que documentar que reclasificar la fila de Cobros NO afecta las filas de Cotización ni Pedido del mismo correo.

**Impacto en tareas:** afecta T15 (Agente IA), T19 (GenerarPendienteUseCase), T9 (Buzón query) y la sección "Detalle — Entidades insertadas". Estimo +24-32 horas adicionales sobre el plan actual.

### H-02 — Buzón de Pedidos no contempla la cadena completa (`pcPromesaDeCompra` → `ppPedido` → partidas)

**Punto del requisito:** *"Buzón de Orden de Compra (Pedido), bajar a `pcPromesaDeCompra`, `ppPedido` y sus partidas, así como lo relacionado para continuar el flujo de venta interna."*

**Estado en el diseño:**
- Sección "Diseño — Buzones de Cotizaciones y Pedidos (T25-T32)" — tabla "Diferencias frente al Buzón de Cobros": *"Pendiente generado al clasificar — Buzón de Pedidos: `pcPromesaDeCompra`"*. **Solo `pcPromesaDeCompra`**, no la cadena.
- Sección H-08 hace referencia explícita a que `ppPedido` y `tpPedido` *"son más adelante en el flujo de venta interna"* — el diseño los excluye deliberadamente.
- No hay mención de `pcPromesaDeCompraPartida` ni de cómo se procesan las partidas extraídas por el Agente IA.

**Problema:** el requisito del usuario pide que el Mailbot baje **toda la cadena** del flujo de pedido (`pcPromesaDeCompra` + `ppPedido` + partidas) y "lo relacionado para continuar el flujo de venta interna". El diseño solo crea el nodo inicial (`pcPromesaDeCompra`) y deja todo lo demás al flujo de venta interna manual.

**Acción recomendada:**
- **Aclarar el alcance con el cliente:** ¿el Mailbot debe crear la cadena completa automáticamente, o solo el primer paso (`pcPromesaDeCompra`) y el usuario continúa el flujo manualmente en venta interna? Esta es la lectura literal del requisito vs la lectura del diseño.
- Si la respuesta es "cadena completa": agregar al diseño la creación de `ppPedido` + partidas a partir de `DatosExtraidosPedidoDto`. Esto requiere ampliar el DTO con campos por partida (cantidad, código, descripción, precio si viene en el correo).
- Si la respuesta es "solo `pcPromesaDeCompra`": dejar la decisión explícita en el documento, no inferida.

### H-03 — Partidas de cotización (`cotPartidaCotizacion`) no contempladas

**Punto del requisito:** *"Buzón de Cotización a `cotCotizacion` y `cotPartidaCotizacion` y sus relacionados."*

**Estado en el diseño:**
- Sección "Diferencias frente al Buzón de Cobros" — *"Buzón de Cotizaciones: `cotCotizacion`"*. Solo la cabecera, sin `cotPartidaCotizacion`.
- `DatosExtraidosCotizacionDto(IReadOnlyList<string> ProductosSolicitados, string? Urgencia)` — el DTO **sí extrae** `ProductosSolicitados`, pero el diseño NO los persiste como `cotPartidaCotizacion`.
- T33 (`GenerarPendienteUseCase`) menciona usar `cotCotizacionFactoryTransaction.Fabricar()` (reutilizando legacy), pero no especifica si esa factory crea las partidas.

**Acción recomendada:**
- Agregar explícitamente al diseño: tras crear `cotCotizacion`, recorrer `DatosExtraidosCotizacionDto.ProductosSolicitados` e insertar 1 fila en `cotPartidaCotizacion` por cada producto extraído. Si la `cotCotizacionFactoryTransaction.Fabricar()` legacy ya hace esto, documentarlo; si no, ampliarla.
- Aclarar el comportamiento cuando `ProductosSolicitados` venga vacío (caso documentado en las muestras: los productos suelen venir en el adjunto). ¿Se crea `cotCotizacion` sin partidas? ¿Se marca como pendiente de captura manual?

---

## Hallazgos moderados

### H-04 — "Entrenamiento" del Agente IA: divergencia conceptual con el requisito

**Punto del requisito:** *"Seleccionar el Agente de IA y de entrenarla para que se pueda procesar..."*

**Estado en el diseño:**
- Sección "GET /metrics" / "Fuera de alcance — POST /reentrenamiento/trigger" — *"se confirmó contra el código legacy real que nunca existió un proceso de reentrenamiento operativo... el Agente IA (LLM) tampoco reentrena pesos."*
- RT-06: *"Los prompts del Agente IA son archivos `.txt`. Son el único mecanismo de ajuste de precisión del clasificador."*

**Comentario:** técnicamente la decisión es correcta — un LLM no se "entrena" como ML.NET, la mejora vive en el **prompt engineering** y few-shot examples. PERO el requisito habla literalmente de "entrenarla", lo cual genera expectativa equivocada en el cliente.

**Acción recomendada:**
- Agregar una sección "Sobre el 'entrenamiento' del Agente IA" al diseño que explique: (a) por qué no hay reentrenamiento operativo de pesos; (b) qué sí se hace para mejorar precisión (ajuste de prompts en `Mailbot.Infrastructure/Prompts/`, validación con `MailbotClasificacionLog`, ciclo de revisión); (c) cómo el cliente puede pedir mejoras (cambio de prompt vs cambio de modelo).
- Dejar nota al cliente confirmando que la elección de Gemini Flash-Lite es por costo/latencia y se puede escalar a modelos más grandes si la precisión no alcanza.

### H-05 — Lectura de PDFs e imágenes en adjuntos no explícita

**Punto del requisito:** *"Para leer los archivos adjuntos debe de leer imágenes o PDF."*

**Estado en el diseño:**
- `CorreoCompletoDto(string Asunto, string CorreoEmisor, string Cuerpo, IReadOnlyList<AdjuntoDto> Adjuntos)` — el DTO sí incluye adjuntos.
- Sección "MinioService" — *"Guarda adjuntos del correo en bucket mailbot/"*. **Solo guardado, no lectura.**
- Sección "GeminiClasificadorAgente" — no especifica que envíe los adjuntos al modelo ni cómo los procesa.
- `extraccion_cobro.txt` y `extraccion_pedido.txt` son prompts mencionados — no se aclara si reciben los binarios de los adjuntos como input multimodal.

**Realidad técnica:** Gemini Flash-Lite vía Vertex AI **sí soporta multimodal** (PDFs, imágenes JPEG/PNG, audio). El SDK `Google.Cloud.AIPlatform.V1` permite enviar `Part` con `InlineData` o `FileData` para archivos. PERO el diseño NO declara esto explícitamente.

**Acción recomendada:**
- Agregar al diseño una sección **"Procesamiento multimodal de adjuntos"** que documente:
  - Tipos MIME aceptados (`application/pdf`, `image/jpeg`, `image/png`, `image/webp` — los que Gemini soporta).
  - Tamaño máximo por adjunto (Gemini Flash-Lite acepta hasta 20MB inline; arriba de eso se usa Vertex AI File API).
  - Comportamiento ante adjuntos no soportados (DOC, XLS, ZIP): se guardan en MinIO pero el modelo solo recibe texto del cuerpo + metadatos del adjunto.
  - Estrategia de **OCR fallback** si el PDF es escaneado (imagen embebida): Gemini puede leer texto en imágenes, pero declarar la calidad esperada y si se requiere pre-procesamiento.
- Reflejar en `GeminiClasificadorAgente.ClasificarAsync()` la inclusión de los `byte[]` de los adjuntos como `Part`s del request a Vertex AI.
- Casos críticos de pruebas: cobro con comprobante PDF, pedido con cotización adjunta en PDF, cotización con imagen JPG de captura de WhatsApp.

### H-06 — `Otros` como destino vs `Otros` como bandeja compartida (RT-18) — efecto colateral aceptado

**Punto del diseño:** RT-18 fuerza `catClasificacionCorreoRecibido.AnalistaDeCuentasPorCobrar = 1` en la clave `'otros'` para que Tesorería vea los correos no enrutables.

**Efecto colateral documentado:** *"correos `otros` de Cotizaciones y Pedidos también aparecerán en la bandeja de Tesoreria"* — decisión de Carlos 2026-06-25.

**Comentario:** la decisión está documentada y aceptada. Solo conviene confirmar que el cliente (Tesorería) está informado: van a ver correos que no son cobros en su bandeja. Si en algún momento eso causa fricción operativa, la solución sería separar `otros-cobro` de `otros-general` con dos claves distintas.

**Acción recomendada:**
- Levantar OBS al cliente: documentar el efecto colateral en RT-18 y obtener confirmación expresa de Tesorería antes del go-live.

### H-07 — `Buzón de Pedidos` filtra por `IdUsuarioESAC` pero no documenta `Buzón de Cotizaciones` igual

**Estado:** la tabla de diferencias dice *"Buzón de Pedidos — Igual que Cotizaciones"* sobre el filtro de visibilidad. Cotizaciones está bien definido (`IdUsuarioESAC`). Esto está bien.

**Comentario:** revisar que el método `ObtenerIDUsuarioESACPorCliente()` ya existente realmente devuelve un único ESAC y no una lista (un cliente podría tener varios ESACs en distintas carteras). Si devuelve múltiples, definir cuál se usa para filtrar el buzón.

---

## Hallazgos menores

### H-08 — Plan de implementación y dependencias entre tareas no aparece en el PDF principal

El PDF referencia `TPSC-RE-FU-008-Plan-v1.md` pero el plan no está incluido. Sugerencia: agregar una tabla resumen de tareas (T1–T41+) con clave de catálogo, horas estimadas y predecesoras dentro del propio diseño, o adjuntar el plan como anexo.

### H-09 — DAR de proveedor IA y muestras anonimizadas son referencias externas

Sugerencia: incluir un anexo en el diseño con el resumen del DAR (Gemini Flash-Lite vs alternativas evaluadas, criterios de decisión) y al menos una muestra anonimizada por tipo. Hoy son archivos sueltos referenciados; quien lea solo el PDF no tiene contexto.

### H-10 — Control de versiones — fila única, sin versiones intermedias

El PDF dice v1.0 / 26-jun-2026 pero el contenido referencia múltiples decisiones cerradas en fechas 2026-06-19, 2026-06-22, 2026-06-24, 2026-06-25, 2026-06-26 — sugiere iteración interna. Agregar al control de versiones esas iteraciones como v0.x para que quede el histórico.

### H-11 — Validar cobro: relación con sub-requisitos RE-023 a RE-030

El diseño aclara correctamente que `PUT /api/cobros/folio/{id}/cerrar` se invoca desde el frontend del Paso 2 de Validar Cobro (Finanzas), y que los sub-requisitos RE-023 a RE-030 están fuera del alcance de FU-008. Bien.

Conviene agregar: ¿qué pasa si el endpoint `/cerrar` se llama dos veces? El diseño no lo dice (idempotencia del cierre). Sugerencia: si `Activo` ya es 0, responder 200 sin reescribir.

---

## Lo que está muy bien y se debe conservar

1. **Idempotencia explícita** (RT-02, sección "Detalle — Idempotencia ante push duplicado") — explica por qué se responde 200 y no 409, índice único filtrado y captura de excepción específica. Modelo a seguir.
2. **Bitácora dual técnica + IA** (`MailbotEventoCorreo` + `MailbotClasificacionLog` con RT-21 que registra TODOS los intentos incluso fallidos) — permite auditar "por qué este correo no apareció" sin revisar logs de aplicación.
3. **Endpoints admin** (`/admin/mailbot/eventos`, retry, descartar) — operación productiva sin manipulación directa de BD.
4. **Estrategia por ambiente Pull (dev) vs Push (prod)** — soluciona el problema real del endpoint HTTPS público en dev sin comprometer la arquitectura.
5. **Mecanismo de detección y disparo del ETL (P20)** — capa push inmediato + capa barrido de respaldo es robusto.
6. **Diseño de LegacyBridge como infraestructura base reutilizable** — bien separado de Canal ETLCobros específico de FU-008.
7. **Bandeja del Coordinador de Tesorería con retroactividad automática** (OBS-021, Casos 1 y 2) — diseño elegante: la query del Gestor filtra por `IdUsuarioCobrador`, no se requiere lógica de migración.
8. **Decisiones D1–D5 cerradas y trazadas** con su impacto en RT-XX, CA-XX y tareas — modelo de trazabilidad a seguir.
9. **Manejo de `CorreoRecibidoEstatus` como tabla insert-only** documentado con conteo real contra BD (8325 filas para 3705 correos).
10. **Reglas técnicas RT-01 a RT-21** numeradas, accionables y con justificación. Muy bueno.

---

## Acciones para Carlos (resumen accionable)

| # | Acción | Hallazgo | Prioridad |
|---|---|---|---|
| 1 | **Implementar clasificación múltiple**: pasar de `Clasificacion: string` a `Clasificaciones: List<>` en DTO, prompt y schema Vertex AI; convertir el switch del paso 14 en loop; insertar N filas en `CorreoRecibidoCliente` | H-01 | **Crítica** |
| 2 | Confirmar con cliente: ¿Buzón de Pedidos debe bajar cadena completa (`pcPromesaDeCompra` + `ppPedido` + partidas) o solo `pcPromesaDeCompra`? Documentar decisión expresa | H-02 | **Crítica** |
| 3 | Agregar al diseño la creación de `cotPartidaCotizacion` tras `cotCotizacion` desde `DatosExtraidosCotizacionDto.ProductosSolicitados` | H-03 | **Crítica** |
| 4 | Agregar sección "Sobre el entrenamiento del Agente IA" que aclare prompts vs reentrenamiento de pesos | H-04 | Alta |
| 5 | Agregar sección "Procesamiento multimodal de adjuntos" con tipos MIME, tamaño máximo, comportamiento ante no soportados y OCR fallback | H-05 | Alta |
| 6 | Levantar OBS al cliente sobre el efecto colateral de RT-18 (Tesorería verá `otros` de Cotizaciones/Pedidos) | H-06 | Media |
| 7 | Confirmar `ObtenerIDUsuarioESACPorCliente()` no devuelve lista | H-07 | Media |
| 8 | Incluir resumen de plan de tareas + DAR proveedor IA + muestras como anexos del PDF | H-08, H-09 | Baja |
| 9 | Sincronizar carátula con historial de iteraciones internas | H-10 | Baja |
| 10 | Documentar idempotencia del endpoint `/cerrar` | H-11 | Baja |

---

## Veredicto

**Estructura del documento:** excelente. Sigue IEEE 1016 e incluye los 5 puntos del checklist (flujo, errores, reglas, pruebas, impacto).

**Cumplimiento del requisito del usuario:** ~75%. Los 3 hallazgos críticos (H-01 clasificación múltiple, H-02 cadena de pedidos, H-03 partidas de cotización) tocan capacidades funcionales centrales que el usuario describió explícitamente. **Sin atender estos, el sistema construido no cumple la intención original.**

**Listo para desarrollo:** parcialmente.
- Las tareas de infraestructura (T1–T3 BD, T13 solución, T20–T24 GCP/Worker, T36–T39 Coordinador y plan, T22–T23 Pub/Sub y watch, T38 corte legacy) pueden arrancar tal cual están.
- Las tareas de clasificación y persistencia (T15, T19, T25–T33) requieren actualizar primero el modelo para clasificación múltiple y completar la persistencia de partidas/cadena de pedidos antes de codificarse.

---

## Referencias

- Requisito funcional: planteamiento verbal del 2026-06-29 (7 puntos sobre rediseño Mailbot, clasificación múltiple, persistencia por proceso, lectura de PDF/imágenes).
- BD vigente: `Requisitos/R16A-RE-FU-008/R16A-RE-FU-008_BD.md` v1.0
- Back (Propuesta 2): `Requisitos/R16A-RE-FU-008/R16A-RE-FU-008-P2-Back.md` v1.0
- Solución .NET 10: `Requisitos/R16A-RE-FU-008/R16A-RE-FU-008-v2_Propuesta2.md` v1.0
- Faltante con bitácora y reclasificación (revisión previa): `Requisitos/R16A-RE-FU-008/R16A-RE-FU-008-Faltante-Bitacora-Reclasificacion.md`
- Canvas operativo: `Diagramas/Canvas/10 - Flujo Mailbot Buzones y Modulos Consumidores.canvas`
