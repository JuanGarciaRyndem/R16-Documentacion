# Revisión — [R16A-RE-FU-005][DIS-SOL] Diseño de la solución

| Campo | Valor |
|---|---|
| **Documento revisado** | `[R16A-RE-FU-005][DIS-SOL] Diseño de la solución.pdf` |
| **Versión revisada** | 1.5 (carátula del PDF) |
| **Fecha de emisión del PDF** | 18-jun-2026 (carátula) |
| **Autor del diseño** | Cristóbal Sebastián García Coss |
| **Revisor** | Juan David García Cruz |
| **Fecha de la revisión** | 2026-06-29 |
| **Versión del requisito base** | R16A-RE-FU-005 (post OBS-009) |
| **Archivos consultados** | `R16A-RE-FU-005.md`, `R16A-RE-FU-005_BD.md` (v1.2), `R16A-RE-FU-005-Back.md` (v1.1), `R16A-RE-FU-005-Tareas.md` (6 tareas), `R16A-RE-FU-005 Revision.md` |

---

## Resumen ejecutivo

El diseño cubre el ~92% de la matriz funcional y técnica de RE-FU-005. La estrategia técnica (agregar `IdRegion` a catálogos existentes en vez de crear tablas paralelas), el patrón `BaseApiController` + `AsegurarFiltroRegion` + `ObtenerUsuarioAutenticado`, los cambios de código C# completos en los tres controllers y la documentación de pruebas técnicas y casos críticos son sólidos y listos para construcción.

Hay **3 hallazgos críticos** que deben atenderse antes de pasar a desarrollo, **4 hallazgos moderados** que conviene cerrar y **4 hallazgos menores** que mejoran la calidad del documento.

---

## Cobertura de la matriz

### Reglas R1–R9 del requisito

| Regla | Descripción | Estado en el diseño |
|---|---|---|
| R1 | Configuración default | ✅ Cubierta (CA E4) |
| R2 | Catálogos diferenciados por Región | ✅ Cubierta (CA A1 + Paso 1 BD + RT-02) |
| R3 | Condición de Pago como dimensión temporal Perú | ✅ Cubierta (CA C1 + INSERT Paso 2) |
| R4 | Método de Pago solo México | ✅ Cubierta (CA B3) |
| R5 | Forma de Pago solo México | ✅ Cubierta (CA B1 + C3) |
| R6 | Uso CFDI y Tipo Comprobante independientes | ✅ Cubierta (CA B2 + C2) |
| R7 | Agente Retención IGV | ⚠️ Condicional a P5 (CA D1) |
| R8 | Sujeto a Detracción | ⚠️ Condicional a P6 (CA D2) |
| R9 | Edición sin restricción de rol | ✅ Cubierta (CA E3) |

### Criterios de aceptación (12)

A1, B1–B3, C1–C3, D1–D2, E1–E4 — todos presentes y mapeados 1:1 con el requisito.

### Tareas existentes (6)

| Tarea | GAP | Estado en el diseño |
|---|---|---|
| T1 | GAP-01 ALTER `IdRegion` en 3 catálogos | ✅ Paso 1 |
| T2 | GAP-02 INSERT registros PE (CONT/CRED + 01/03/08) | ✅ Paso 2 |
| T3 | GAP-03 ALTER banderas tributarias PE | ✅ Paso 3 (condicional) |
| T4 | GAP-04 Controllers `BaseApiController` + filtro región | ✅ Sección "Cambios en Controllers C#" con código completo |
| T5 | GAP-05 `vDatosFacturacionCliente` | ✅ Paso 4 (condicional) |
| **T6** | **GAP-06 `ClaveFormaDePago` SAT en `catMedioDePago`** | ❌ **No documentado como cambio** |

---

## Hallazgos críticos (bloqueantes)

### H-01 — Tarea 6 / GAP-06 ausente en la sección "Cambios en Base de Datos"

- **Sección del PDF:** "Cambios en Base de Datos" (Pasos 1–4).
- **Referencia matriz:**
  - `R16A-RE-FU-005-Back.md` GAP-06 (líneas 260–272).
  - `R16A-RE-FU-005-Tareas.md` Tarea 6 (líneas 376–440).
  - `R16A-RE-FU-005.md` Notas de Implementación (línea 210).
- **Problema:** El diseño solo lista P4 ("Clave SAT para Aba, NA, —NINGUNO— y Swift en `catMedioDePago`") como pendiente, pero **no incluye un Paso 5 con el UPDATE** que aplica las claves SAT cuando P4 se resuelva. Si el desarrollador toma el PDF como spec única, la Tarea 6 no se ejecuta.
- **Acción recomendada:** Agregar `Paso 5 — UPDATE ClaveFormaDePago en catMedioDePago (CONDICIONAL — solo tras confirmar P4)` en la sección "Cambios en Base de Datos", con el snippet SQL del `-Back.md` GAP-06 o del `-Tareas.md` Tarea 6. Alternativamente, declarar explícitamente que GAP-06 queda fuera del alcance del diseño actual y se gestiona por separado.

### H-02 — Las 5 Brechas Reconocidas SUNAT no están enumeradas

- **Sección del PDF:** "Alcance — No incluye" (línea: *"Habilitación de facturación electrónica SUNAT (brechas 1–5 fuera de alcance)"*) y "Acciones requeridas antes de implementar" #4 (*"Presentar brechas consolidadas al cliente (OBS-009) — Listar las 5 brechas"*).
- **Referencia matriz:** `R16A-RE-FU-005.md` sección "Brechas Reconocidas — Facturación Electrónica Perú" (líneas 220–241), con 5 brechas tituladas:
  1. Datos fiscales SUNAT en catálogo de productos.
  2. Guía de Remisión Electrónica (GRE).
  3. Tipo de Operación SUNAT (Catálogo 51) por factura.
  4. Régimen de Percepción del IGV de PROQUIFA como Agente de Percepción.
  5. Configuración del emisor PROQUIFA Perú para facturación electrónica SUNAT (certificado, OSE/SEE-SOL, CDR, etc.).
- **Problema:** El diseño dice "brechas 1–5" pero no las enumera con título ni descripción. El desarrollador y el cliente no tienen la lista a la vista en este documento. La acción #4 referencia OBS-009 sin desarrollarla.
- **Acción recomendada:** Agregar una sub-sección "Brechas Reconocidas — Facturación Electrónica Perú (fuera de alcance)" con tabla de 5 filas (título + descripción corta), copiando del requisito. Vincular cada brecha con su Pendiente correspondiente (P7 → Brecha 4, etc.).

### H-03 — Archivo de Equivalencias MX↔PE no referenciado

- **Sección del PDF:** No aparece.
- **Referencia matriz:** `R16A-RE-FU-005.md` Notas de Implementación (línea 204): *"Para el detalle de los campos de la sección Cobros por Región, los catálogos y el análisis de los mecanismos tributarios peruanos, ver archivo adjunto `R16A-RE-FU-005_Equivalencias_Cobros_MX_PE.xlsx`."*
- **Problema:** El Excel es fuente única para el detalle de claves SAT, equivalencias y análisis tributario. Si cambia (p. ej. cliente confirma claves SAT para Aba/Swift), el diseño debería redirigir al lector a ese archivo.
- **Acción recomendada:** Referenciar el Excel en la sección "Componentes involucrados" o en "Acciones requeridas antes de implementar".

---

## Hallazgos moderados

### H-04 — Tabla de equivalencia conceptual PUE↔Contado / PPD↔Crédito no incluida

- **Sección del PDF:** No aparece.
- **Referencia matriz:** `R16A-RE-FU-005_BD.md` sección 2 (líneas 130–135), tabla "Equivalencia conceptual MX↔PE".
- **Problema:** Es información clave para entender por qué se reutiliza `catMetodoDePagoCFDI` para ambos países. Su ausencia obliga al desarrollador a deducirlo del contexto.
- **Acción recomendada:** Copiar la tabla de equivalencia a la sección "Componentes involucrados" o "Reglas técnicas" del diseño.

### H-05 — Pendiente P3 (nivel de captura de TasaDetraccion) tratado como no-bloqueante

- **Sección del PDF:** Tabla de Pendientes (P3): *"Tasa de detracción: ¿se captura a nivel cliente, producto o catálogo SUNAT?"* — Estado: "Pendiente".
- **Problema:** El diseño asume que `TasaDetraccion` vive en `DatosFacturacionCliente` (nivel cliente — Paso 3 del PDF). Si el cliente responde "nivel producto", el modelo cambia y el Paso 3 queda obsoleto. P3 es **bloqueante del Paso 3**, no solo "Pendiente".
- **Acción recomendada:** Elevar P3 a "Bloqueante" junto con P5/P6. Documentar explícitamente que el Paso 3 solo se ejecuta cuando P3, P5 y P6 estén confirmados.

### H-06 — Discrepancia de versión/fecha entre carátula y control de versiones

- **Sección del PDF:** Carátula: **v1.5 — 18/06/2026**. Tabla "Control de versiones" al final del documento: solo aparece **v1.0 — 22/06/2026**.
- **Problemas:**
  1. La carátula registra v1.5; el historial solo llega a v1.0.
  2. La fecha de la carátula (18-jun) es anterior a la del v1.0 del historial (22-jun) — invierte la línea de tiempo.
- **Acción recomendada:** Sincronizar carátula y changelog. Agregar entradas v1.1 a v1.5 (o bajar la carátula a la versión real) y corregir fechas para que sean consistentes.

### H-07 — "Acciones requeridas antes de implementar" no incluye P4

- **Sección del PDF:** "Acciones requeridas antes de implementar" lista P5, P6, P1, P2 y OBS-009 — pero omite P4.
- **Problema:** Si se atiende H-01 (Paso 5 con UPDATE de ClaveFormaDePago), P4 también debe ser una precondición de la implementación.
- **Acción recomendada:** Agregar P4 a la lista de acciones, condicionado a la decisión sobre H-01.

---

## Hallazgos menores

### H-08 — `ConfiguracionPagos` ausente de "Componentes involucrados"

- **Referencia matriz:** `R16A-RE-FU-005_BD.md` sección 4 (líneas 173–191) — tabla con su estructura y relación con `Cliente.IdConfiguracionPagos`.
- **Problema:** El diseño menciona `ConfiguracionPagos` solo de paso en Flujo 3. Por completitud y trazabilidad, debería aparecer en la tabla de "Componentes involucrados" como "existente, sin cambios".

### H-09 — Consultas SQL de diagnóstico/QA no referenciadas

- **Referencia matriz:** `R16A-RE-FU-005_BD.md` sección "Consultas SQL Principales" (líneas 305–397) — Configuración cliente MX, Configuración cliente PE, Selector por Región, Clientes con cobros incompletos.
- **Acción recomendada:** Referenciar estas consultas en "Pruebas técnicas / Integración" o como anexo de QA para validación post-deployment.

### H-10 — P8 introducido en el diseño no está en `-Back.md`

- **Sección del PDF:** Pendiente P8 ("Tipo de Revisión" Digital/Física/Híbrida).
- **Referencia matriz:** El requisito (`R16A-RE-FU-005.md` línea 206) lo menciona como duda en Notas de Implementación, pero el `-Back.md` solo llega hasta P7.
- **Acción recomendada:** Es una buena adición del diseño. Propagar P8 al `-Back.md` y al `-Tareas.md` para mantener consistencia entre los 4 archivos del requisito.

### H-11 — Criterios E1/E2 marcados como "Frontend" — aclaración sobre nullabilidad

- **Sección del PDF:** Tabla de criterios — E1/E2 estado "Frontend", justificación: "Backend: FKs son NULLABLE en BD. Obligatoriedad se valida en frontend antes de enviar."
- **Comentario:** correcto. Solo conviene confirmar que el documento de diseño Frontend (referenciado al inicio: *"El diseño Frontend (Angular, NgRx) se documenta por separado"*) recoja explícitamente E1 y E2 para que no quede el criterio sin owner.

---

## Lo que el diseño hace bien y debe conservarse

1. **Estrategia técnica `IdRegion` en catálogos existentes** — alineada con `_BD.md` v1.2; evita crear tablas paralelas y mantiene compatibilidad con el modelo legacy.
2. **Patrón `BaseApiController` + `AsegurarFiltroRegion` + `ObtenerUsuarioAutenticado`** — consistente con `catRutaEntregaController.cs` ya implementado y vigente.
3. **Código C# completo de los 3 controllers en estado final** — el desarrollador puede copiar y aplicar sin ambigüedad.
4. **Reglas técnicas RT-01 a RT-07** numeradas y accionables, especialmente RT-04 (manejo de `using`) y RT-05 (orden estricto de scripts).
5. **Diagrama de secuencia** clara para el flujo de consulta con filtro de región.
6. **Casos críticos anticipados** — usuario sin región, concurrencia de scripts, rollback de EDMX.
7. **Manejo de errores anticipado** — HTTP 500 para usuario sin región, HTTP 401 para token inválido, idempotencia con `NEWID()`.
8. **Pendientes etiquetados con "Impacto en este diseño" y estado "Bloqueante / Pendiente"** — claro para priorización.
9. **"Archivos NO modificados (confirmado)"** — evita que el desarrollador toque BO/Factory/VDBO innecesariamente.
10. **Mapeo 1:1 entre criterios del requisito y la tabla de criterios del diseño** (12 criterios A1–E4) — sin gaps de numeración.

---

## Acciones para Sebastián (resumen accionable)

| # | Acción | Hallazgo | Prioridad |
|---|---|---|---|
| 1 | Agregar Paso 5 con UPDATE de `ClaveFormaDePago` o declarar GAP-06 fuera del alcance del diseño | H-01 | Alta |
| 2 | Enumerar las 5 Brechas SUNAT con título y descripción copiando del requisito | H-02 | Alta |
| 3 | Referenciar el Excel `R16A-RE-FU-005_Equivalencias_Cobros_MX_PE.xlsx` | H-03 | Media |
| 4 | Copiar tabla de equivalencia PUE↔Contado / PPD↔Crédito | H-04 | Media |
| 5 | Elevar P3 a Bloqueante del Paso 3 | H-05 | Media |
| 6 | Sincronizar versión/fecha entre carátula y control de versiones | H-06 | Media |
| 7 | Agregar P4 a "Acciones requeridas antes de implementar" si se atiende H-01 | H-07 | Media |
| 8 | Agregar `ConfiguracionPagos` a "Componentes involucrados" | H-08 | Baja |
| 9 | Referenciar consultas SQL de diagnóstico para QA | H-09 | Baja |
| 10 | Propagar P8 ("Tipo de Revisión") al `-Back.md` y `-Tareas.md` | H-10 | Baja |
| 11 | Confirmar que el diseño Frontend recoge E1 y E2 | H-11 | Baja |

---

## Veredicto

**Estructura del documento:** muy buena. Sigue formato AUI-FOR-01 y abarca todas las secciones esperadas (propósito, alcance, objetivo, componentes, flujos, criterios, reglas técnicas, BD, controllers, EDMX, pruebas, casos críticos, pendientes).

**Cumplimiento del requisito:** ~92%. Los 12 criterios y 9 reglas están cubiertos, pero faltan **3 elementos críticos**: Tarea 6 / GAP-06 en BD (H-01), enumeración de las 5 brechas SUNAT (H-02) y referencia al Excel de Equivalencias (H-03). Una vez atendidos, el diseño queda completo para construcción.

**Listo para desarrollo:** parcial. Si se atienden H-01, H-02 y H-05 (P3 como bloqueante), el desarrollador puede arrancar las Tareas 1, 2 y 4 sin riesgo. Las Tareas 3 y 5 dependen de P5/P6 (y P3 si H-05 se aplica), correctamente marcadas como condicionales. La Tarea 6 sigue bloqueada por P4 hasta su confirmación con el cliente.

---

## Referencias

- Requisito funcional: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005_BD.md` (v1.2)
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Back.md` (v1.1)
- Tareas de desarrollo: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Tareas.md` (6 tareas)
- Revisión funcional aplicada: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005 Revision.md`
- Excel de equivalencias (referenciado en el requisito): `R16A-RE-FU-005_Equivalencias_Cobros_MX_PE.xlsx`
- Observación aplicada: OBS-009 (Brechas Facturación Electrónica Perú).
