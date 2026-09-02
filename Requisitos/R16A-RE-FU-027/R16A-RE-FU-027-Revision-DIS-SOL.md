# Revisión — [R16A-RE-FU-027][DIS-SOL] Diseño de la solución

**Documento revisado:** [Diseño de la solución](https://docs.google.com/document/d/1yVyXp5llQniXtgqsBvPzJrsKSbgEtWKY1sYONJ9Pky8/edit?usp=sharing) — v1.0, 02/07/2026, autor Isai Amaury Garcia Flores, revisor Valdemar Farina Sánchez.
**Fecha de revisión:** 2026-08-26
**Revisor:** Juan Garcia (con asistencia de Claude)
**Contexto considerado:** requisito funcional y análisis de impacto de esta misma carpeta (`R16A-RE-FU-027.md`, `-Back.md`, `_BD.md`, `-Tareas.md`) y el requisito del cual depende directamente por Strategy Pattern, `R16A-RE-FU-026` (Paso 2 México) — incluyendo su propio DIS-SOL (Google Doc revisado el 2026-08-26) — y transitivamente `R16A-RE-FU-024` (Paso 1 México, dueño de la fusión `fccCobroCliente`).

Los comentarios están redactados para pegarse directamente como comentarios de Google Docs: cada uno indica el texto exacto a seleccionar en el documento antes de anclar el comentario. El hallazgo #7 no tiene anclaje en el Google Doc (aplica a los `.md` locales de esta carpeta) y se documenta aparte al final.

---

## 🔴 Críticos

### 1. Encabezado con el ID equivocado

📍 **Seleccionar:** `Requisito R16A-RE-FU-002`

💬 **Comentario:**
> Este encabezado dice "R16A-RE-FU-002" pero el contenido del documento es de R16A-RE-FU-027 (Paso 2 Perú). Parece un residuo de copiar la plantilla del DIS-SOL de otro requisito. Corregir para evitar confusión al buscar/archivar el documento.

### 2. El documento está incompleto desde "Criterios de aceptación" hasta el final

📍 **Seleccionar** (texto de ejemplo genérico en la tabla de Criterios de Aceptación): `El usuario debe poder filtrar por notificaciones "Leídas" y "No Leídas"`

💬 **Comentario:**
> Desde aquí hasta el final del documento (Criterios de Aceptación, Reglas técnicas aplicadas, Diagramas, Impacto Técnico, Manejo de Errores y Excepciones, Estrategia de Pruebas) es texto de plantilla sin adaptar — ejemplos genéricos de un módulo Angular/NgRx de notificaciones (OnPush, NgRx Selectors, NotificationTable, app-routing.module.ts), sin relación con Validar Cobro Perú ni con el backend .NET de este proyecto. Falta: mapear los criterios reales del requisito (A1-A2, B1-B3, C1-C5, D1-D2, E1-E3, F1-F3 de `R16A-RE-FU-027.md`), las reglas técnicas reales (análogas a las RT-01..RT-10 del DIS-SOL de RE-026), el impacto real en `Paso2PeruStrategy.cs`, los códigos de error/ProblemDetails, y las pruebas de escenarios de saldo/multi-divisa/rollback. Sin esto, el documento no responde las preguntas de su propia sección "Importante" (¿sabe qué reglas no puede romper?, ¿qué pruebas debe pasar?, ¿dónde impacta?).

### 3. `fccSaldoFavorCliente`: usa un esquema que el propio RE-026 ya descartó

📍 **Seleccionar:** `ALTER TABLE fccSaldoFavorCliente → agregar columna PEN bit`

💬 **Comentario:**
> El DIS-SOL de RE-FU-026 (Google Doc, sección "Específicamente incluye") ya rediseñó esta tabla para no depender de bits por moneda: *"Creación tabla fccSaldoFavorCliente con ClaveMoneda varchar(3) + ClaveRegion varchar(3) (region-ready — evita ALTER para Perú y monedas futuras)"*. Si RE-026 se construye tal como su propio DIS-SOL indica, este ALTER ya no debería hacer falta — el registro de Perú se haría con `ClaveRegion='PER'`/`ClaveMoneda='PEN'`, no con una columna `PEN` nueva. Reconciliar con el dueño de RE-026 antes de ejecutar este ALTER.

---

## 🟠 Importantes

### 4. No incorpora la resolución de DUDA-086 (tolerancia Perú)

📍 **Seleccionar:** `"umbralToleranciaLabel": "PENDIENTE DE DEFINICIÓN (PROQUIFA)"`

💬 **Comentario:**
> Este DIS-SOL es del 02/07/2026; DUDA-086 se resolvió el 21/08/2026 con: *"se aplica la MISMA regla que México (tolerancia equivalente), configurable a nivel BD"*. El umbral ya no está pendiente — hay que actualizar el valor por defecto al de México (`Finanzas:ToleranciaValidarCobroMXN`, mismo valor para Perú vía config) y quitar el estado "pendiente" del DTO y de la lógica (`if (umbral > 0m ...)` actualmente trata umbral=0 como caso válido de "pendiente").

### 5. No incorpora la resolución de DUDA-087 (NC peruana descartada)

📍 **Seleccionar:** `"idReferenciaFiscal": "pendiente-sunat-cat09"`

💬 **Comentario:**
> DUDA-087 (21/08/2026, posterior a este DIS-SOL) dice: *"se cancela la facturación de Perú en R16; no aplica desarrollar la mecánica de NC peruana ni su aplicación al adeudo en este Paso"*. Este bloque `notasCreditoVigentes` en el DTO de respuesta sigue asumiendo que las NCs se ofrecen y aplican en Perú. Debería eliminarse o marcarse explícitamente fuera de alcance para no dar una falsa expectativa de implementación.

### 6. Referencia a `fccPagoCliente.TipoCambio` (esquema pre-fusión)

📍 **Seleccionar:** `el Paso 2 simplemente usa el TC ya almacenado en fccPagoCliente.TipoCambio`

💬 **Comentario:**
> Por la fusión de RE-FU-024 (decisión D1 del DIS-SOL), la tabla ya no es `fccPagoCliente` sino `fccCobroCliente`, y el campo operativo correcto para el Paso 2 es `TipoDeCambioFacturacion` (así lo establece el propio DIS-SOL de RE-026 en "Impacto en modelos": *"se leen TipoDeCambioFacturacion y estatus para validaciones"*). Actualizar la referencia a la tabla/campo correcto.

---

## 🟡 Menores

### 7. `fccNotaCredito`: el ALTER `PEN`/`MontoPEN` (Tarea 1) podría ser innecesario

*(Este hallazgo no tiene texto anclable en el Google Doc — aplica a `R16A-RE-FU-027-Back.md` Parte A/A1, `R16A-RE-FU-027_BD.md` y `R16A-RE-FU-027-Tareas.md` Tarea 1 de esta misma carpeta.)*

> El DIS-SOL de RE-026 describe que R032 remodela `fccNotaCredito` con un campo `Moneda` genérico vía join a `CFDIGenerada`, no con bits `MXN/USD/PEN` por divisa. Confirmar con el dueño de R032 antes de ejecutar el ALTER `PEN`/`MontoPEN` de esta Tarea 1 — podría duplicar o contradecir el esquema real que R032 está definiendo.

### 8. "Empresa emisora... Golocaer S.A.C. (hardcoded)"

📍 **Seleccionar:** `Solo Golocaer S.A.C. (hardcoded)`

💬 **Comentario:**
> Coherente con la regla de negocio (emisor único en Perú), pero vale la pena anotar el riesgo de mantenimiento si PROQUIFA agrega otro emisor en Perú a futuro — un valor hardcodeado no se adaptaría sin un cambio de código.

### 9. Tabla "Control de versiones" sin fila real

📍 **Seleccionar:** `Se crea plantilla genérica para DIS`

💬 **Comentario:**
> Faltaría una fila que registre la versión 1.0 real de este documento (autor Isai Amaury Garcia Flores, 02/07/2026, revisor Valdemar Farina Sánchez) — actualmente solo están las dos filas genéricas de creación de la plantilla, iguales a las del DIS-SOL de RE-024.

---

## ✅ Lo que está bien resuelto

- Los Flujos 1-6 (Carga inicial, Cálculo de saldo, Persistencia transaccional, Modal de inconsistencia, Auto-guardado/Navegación) están bien desarrollados, específicos de Validar Cobro Perú, y correctamente diferenciados de México donde corresponde (ausencia de efecto fiscal, empresa única, `identificadorFiscalLabel: "RUC"`).
- Buena decisión arquitectónica: Strategy Pattern (`Paso2PeruStrategy`) reutilizando `Paso2Service`/`Paso2Controller` en vez de duplicar endpoints — consistente con que la lógica operativa es idéntica a México.
- El Paso A1 de la transacción valida explícitamente que el cliente sea región PER antes de proceder — buena guarda defensiva.
- Correctamente identifica y aísla la diferencia de fondo vs México (sin Complemento de Pago, sin reporte a SUNAT) tanto en el flujo como en el mensaje de respuesta (`"efectoFiscal": false`).
