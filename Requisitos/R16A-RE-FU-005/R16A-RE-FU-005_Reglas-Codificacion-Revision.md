# Revisión — R16A-RE-FU-005 contra Reglas al Codificar

| Campo | Valor |
|---|---|
| **Requisito** | R16A-RE-FU-005 — Configuración de Cobros y Facturación |
| **Documento base** | `Diseño y Desarrollo/Reglas al codificar.md` |
| **Archivos revisados** | `R16A-RE-FU-005.md`, `R16A-RE-FU-005_BD.md` v1.2, `R16A-RE-FU-005-Back.md` v1.1, `R16A-RE-FU-005-Tareas.md`, `[R16A-RE-FU-005][DIS-SOL] Diseño de la solución.pdf` v1.5 |
| **Revisor** | Juan David García Cruz |
| **Fecha de revisión** | 2026-06-29 |
| **Ámbito** | Cumplimiento del alcance del 005 con las 8 reglas transversales de codificación del proyecto |

---

## 1. Reglas al codificar — Referencia

| # | Regla | Ámbito |
|---|---|---|
| R1 | BD de ProquifaDotNet en español (tablas, vistas, SPs, funciones) | BD existente |
| R2 | BD nuevas: estructura en inglés | BD nueva |
| R3 | Codificación en PQF.Catalogos → español | Catálogos existentes |
| R4 | Codificación en PQF.Logistica → si es actualización de proceso, español | Logística — flujos existentes |
| R5 | Codificación en PQF.Logistica → nuevo endpoint y nuevo proceso: actualizar modelo/controller en inglés | Logística — nuevos endpoints |
| R6 | Nueva solución: todo en inglés (clases, DTOs, modelos, procesos, métodos, funciones, comentarios) | Aplicativos nuevos |
| R7 | Envíos de correo se hacen por Aplicativo Envío de Correo — Aplicativo Nuevo | Notificaciones |
| R8 | Procesos como guardar factura, validar cobro, guardar proforma → llamar a Bitácora General — Aplicativo Nuevo | Auditoría |

---

## 2. Aplicabilidad al 005

| Regla | Aplica al 005 | Justificación |
|---|---|---|
| R1 | ✅ Sí | Los ALTER TABLE y datos iniciales se ejecutan sobre BD ProquifaDotNet existente |
| R2 | ❌ No | 005 no crea BD nueva |
| R3 | ✅ Sí | Los 3 controllers modificados viven en `WebApi.Catalogos` |
| R4 | ❌ No | 005 no toca `PQF.Logistica` (los módulos consumidores están fuera de alcance) |
| R5 | ❌ No | 005 no crea endpoint nuevo en `PQF.Logistica` |
| R6 | ❌ No directamente | 005 no crea solución nueva, pero prepara datos que consumirán aplicativos nuevos (Timbrado Perú) — ver H-03 |
| R7 | ❌ No | 005 no envía correos |
| R8 | ⚠️ **Sí — no documentado en el diseño** | El BO `DatosFacturacionClienteBO._GuardarOActualizar` toca configuración fiscal del cliente que gobierna el timbrado |

---

## 3. Verificación puntual del cumplimiento

### 3.1 R1 — Base de datos en español

Verificación contra el DDL propuesto en `_BD.md` y el PDF de diseño:

| Objeto | Nombre | Estado |
|---|---|---|
| Tablas | `catMetodoDePagoCFDI`, `catUsoCFDI`, `catMedioDePago`, `DatosFacturacionCliente` | ✅ Español |
| Vista | `vDatosFacturacionCliente` | ✅ Español |
| Columna nueva | `IdRegion` | ✅ Español |
| Columnas nuevas PE | `AgenteRetencionIGV`, `SujetoDetraccion`, `TasaDetraccion` | ✅ Español |
| Constraints FK | `FK_catMetodoDePagoCFDI_Region`, `FK_catUsoCFDI_Region`, `FK_catMedioDePago_Region` | ✅ Español |
| Constraints Default | `DF_DFC_AgenteRetencionIGV`, `DF_DFC_SujetoDetraccion` | ✅ Español |
| Datos iniciales PE | `N'Contado'`, `N'Crédito'`, `N'Factura electrónica'`, `N'Boleta de venta electrónica'`, `N'Recibo por Honorarios electrónico'` | ✅ Español (correcto: son descripciones que ve el usuario) |
| Claves | `'CONT'`, `'CRED'`, `'01'`, `'03'`, `'08'` | ✅ Codigos SAT/SUNAT respetados |

**Verdict:** cumple R1. ✅

### 3.2 R3 — Codificación PQF.Catalogos en español

Verificación contra la sección "Cambios en Controllers C#" del diseño y GAP-04 del `-Back.md`:

| Elemento | Estado |
|---|---|
| Ubicación de los 3 controllers: `WebApi.Catalogos\Controllers\Catalogos\` | ✅ Correcto |
| Nombres de controllers: `catMetodoDePagoCFDIController`, `catUsoCFDIController`, `catMedioDePagoController` | ✅ Español |
| Métodos: `Obtener`, `GuardarOActualizar`, `QueryResult`, `Desactivar` | ✅ Español |
| Helpers usados: `BaseApiController`, `ObtenerUsuarioAutenticado()`, `AsegurarFiltroRegion()` | ✅ Español (helpers existentes en `Core.Pqf.WebApi.ControllerOperations`) |
| Cambio de herencia `ApiController` → `BaseApiController` | ✅ Sin introducir nomenclatura en inglés |
| Using nuevo: `Core.Pqf.WebApi.ControllerOperations` | ✅ Español |

**Verdict:** cumple R3. ✅

### 3.3 R8 — Bitácora General (Aplicativo Nuevo)

La regla dice literalmente:
> *"procesos por ejemplo al guardar una factura, al validar un cobro, al guardar una proforma, etc. debe de llamar a Bitácora General - Aplicativo Nuevo"*.

**Aplicabilidad al 005:** el "etc." incluye la configuración fiscal del cliente porque los campos que 005 toca son insumos directos del timbrado:

| Campo | Consumo aguas abajo |
|---|---|
| `IdCatMetodoDePagoCFDI` (MEX: PUE/PPD) | XML del CFDI — nodo `MetodoPago` |
| `IdCatUsoCFDI` (MEX: G01/G03; PE: 01/03/08) | XML del CFDI receptor — nodo `UsoCFDI` / documento fiscal SUNAT |
| `AgenteRetencionIGV` (PE) | Gobierna cálculo de retención IGV en el timbrado peruano |
| `SujetoDetraccion` + `TasaDetraccion` (PE) | Gobierna cálculo de detracción SPOT en emisión |

**Riesgo actual:** Si un usuario modifica estos campos por error o mala intención, y luego se emite un CFDI/CPE mal timbrado, no hay traza en el sistema de:
- Quién hizo el cambio.
- Qué valor tenía antes.
- Qué valor quedó después.
- Cuándo se hizo.

Esto es exactamente lo que Bitácora General resuelve.

**Estado en el diseño 005:** ❌ **No documentado.** El diseño (PDF v1.5) declara explícitamente:
> *"No incluye — Cambios en `DatosFacturacionClienteBO`, `DatosFacturacionClienteFactory` ni `DatosFacturacionClienteVDBO`"*.

Bajo la regla R8, **sí debe haber un cambio en `DatosFacturacionClienteBO`** (o en el nivel donde vive el guardado transaccional) para instrumentar Bitácora General cuando cambien los campos fiscales.

---

## 4. Hallazgos

### H-01 (crítico) — Falta instrumentar Bitácora General en el guardado de datos fiscales

- **Regla origen:** R8
- **Ámbito:** `DatosFacturacionClienteBO._GuardarOActualizar` o `DatosFacturacionClienteVDBO.GuardarOActualizar`
- **Problema:** El diseño excluye explícitamente cambios en esos BOs, dejando la modificación de campos fiscales sin auditoría. Esto viola R8 tal como está redactada.
- **Acción propuesta:**
  1. Reabrir el alcance: quitar `DatosFacturacionClienteBO` de la lista "No incluye" del PDF.
  2. Agregar sub-tarea a Tarea 3 (o crear Tarea 7 nueva) que instrumente la llamada a **Bitácora General — Aplicativo Nuevo** cuando cambien:
     - `IdCatMetodoDePagoCFDI`
     - `IdCatUsoCFDI`
     - `AgenteRetencionIGV`
     - `SujetoDetraccion`
     - `TasaDetraccion`
  3. Payload sugerido para Bitácora:
     - `IdCliente`
     - `IdUsuario` (quien modificó, extraído del token)
     - `Aplicativo`: `"ProquifaDotNet — DatosFacturacionCliente"`
     - `Operacion`: `"ActualizacionConfiguracionFiscal"`
     - `Cambios`: array `[{ Campo, ValorAnterior, ValorNuevo }]`
     - `Timestamp`: `GETDATE()`
     - `IdRegion` del cliente
- **Estimación:** `IMP-EXIST-SERVICE` = 12 h (según `Catalogo BackEnd.md`).
- **Predecesor:** existencia del cliente API/SDK del Aplicativo Nuevo Bitácora General.

### H-02 (moderado) — Ambigüedad semántica del catálogo compartido `catMetodoDePagoCFDI`

- **Regla origen:** R1/R3 (interpretativa)
- **Problema:** El catálogo `catMetodoDePagoCFDI` se reusa con doble semántica:
  - En México representa "Método de Pago CFDI" del SAT (PUE/PPD).
  - En Perú representa "Condición de Pago SUNAT" (Contado/Crédito).
- **Impacto en codificación:** el nombre de la tabla y las claves (`MetodoDePagoCFDI`, `ClaveMetodoDePagoCFDI`) son ambiguas para Perú. No viola R1/R3 estrictamente porque son nombres existentes, pero puede confundir a devs futuros.
- **Acción propuesta:** documentar en el diseño (Regla técnica RT-XX o Notas) el porqué del nombre "CFDI" para el registro peruano — decisión pragmática de reuso, no un error de semántica.
- **Alternativa (opcional, no recomendada por costo):** renombrar la tabla a algo neutro tipo `catCondicionTemporalPago`. Riesgo alto: rompe todos los consumidores existentes.
- **Estimación:** 0 h (solo documentación).

### H-03 (bajo) — Coordinación anticipada con Aplicativo Timbrado Perú (nueva solución en inglés — R6)

- **Regla origen:** R6
- **Problema:** Cuando se habilite el timbrado Perú (fuera del alcance R16), esa será una **nueva solución** codificada en inglés. Esa solución consumirá los campos que 005 guarda (`IdCatMetodoDePagoCFDI` = CONT/CRED, `IdCatUsoCFDI` = 01/03/08, `AgenteRetencionIGV`, `SujetoDetraccion`, `TasaDetraccion`).
- **Riesgo:** si el equipo de Timbrado Perú traduce los nombres en la capa de scaffold EF Core (p. ej. `PaymentMethodId`), aumenta la carga cognitiva al mapear inglés ↔ español a través de la frontera. Peor si se hace mapeo manual con DTOs.
- **Acción propuesta:**
  1. Dejar en Notas de Implementación una advertencia: *"Los nombres de columnas de estos catálogos NO se traducen al scaffold del futuro `PQF.Timbrado.Peru`. Se mantienen en español porque son parte de una BD existente (R1)."*
  2. Este es un punto de coordinación con el equipo Timbrado Perú, no un cambio inmediato en 005.
- **Estimación:** 0 h (solo documentación).

### H-04 (bajo) — Confirmar redacción de R8 con el líder técnico

- **Contexto:** la regla R8 usa "por ejemplo" y "etc.", dejando ambigua la lista exacta de procesos que deben registrar en Bitácora General.
- **Riesgo:** interpretaciones inconsistentes entre requisitos y equipos.
- **Acción propuesta:** solicitar al líder técnico una lista explícita de operaciones que caen bajo R8, o un criterio general (p. ej. *"todo INSERT/UPDATE sobre tablas fiscales, financieras o de facturación"*).
- **Estimación:** 0 h para 005 (solo levantamiento de duda).

---

## 5. Comentarios listos para pegar en el documento de análisis

> Estos son los comentarios sugeridos para agregar directamente al documento de análisis del 005 (Google Doc o el que estés usando):

**Comentario general (arriba del documento):**
> *Revisado contra `Reglas al codificar.md` del proyecto. Cumple R1 (BD español), R3 (Catálogos español). **Hallazgo crítico H-01**: falta instrumentar la llamada a Bitácora General (Aplicativo Nuevo, R8) en `DatosFacturacionClienteBO._GuardarOActualizar` cuando cambien campos fiscales (`IdCatMetodoDePagoCFDI`, `IdCatUsoCFDI`, `AgenteRetencionIGV`, `SujetoDetraccion`, `TasaDetraccion`). Ver documento adjunto `R16A-RE-FU-005_Reglas-Codificacion-Revision.md`.*

**Comentario sobre "No incluye — Cambios en `DatosFacturacionClienteBO`":**
> *⚠️ Este exclusión choca con la regla R8 del proyecto. La modificación de campos fiscales del cliente debe registrarse en Bitácora General. Propongo reabrir el alcance y agregar sub-tarea de instrumentación en `DatosFacturacionClienteBO._GuardarOActualizar`. Estimación: 12h (`IMP-EXIST-SERVICE`).*

**Comentario sobre la tabla de datos iniciales PE (Paso 2 INSERT registros Perú):**
> *ℹ️ Nomenclatura correcta según R1 (español). Nota conceptual: el registro peruano vive en una tabla llamada `catMetodoDePagoCFDI` cuya semántica original es SAT-CFDI. Documentar en RT que el reuso es pragmático y no un error de dominio (H-02).*

**Comentario sobre "Actualización EDMX":**
> *ℹ️ Confirmar que la actualización del EDMX preserva los nombres en español de las propiedades generadas (`IdRegion` tipo `Nullable<Guid>`, `AgenteRetencionIGV` tipo `bool?`, etc.). El EDMX genera propiedades espejo del schema DB — no requiere ajuste manual mientras Update Model from Database respete R1.*

**Comentario sobre "Impacto en modelos por actualización EDMX":**
> *ℹ️ Cuando en el futuro se implemente Timbrado Perú (nueva solución en inglés, R6), esos scaffolds deberán preservar los nombres en español de estas columnas — no traducirlos a inglés. Coordinación anticipada con ese equipo cuando arranquen (H-03).*

---

## 6. Impacto en Plan y Tareas

### 6.1 Tareas afectadas del 005 (según `-Tareas.md`)

| # Tarea | Título | Impacto por Reglas |
|---|---|---|
| Tarea 1 | Agregar `IdRegion` a los 3 catálogos | ✅ Sin cambio (cumple R1) |
| Tarea 2 | INSERT registros PE (CONT/CRED, 01/03/08) | ✅ Sin cambio (cumple R1) |
| Tarea 3 | Agregar campos PE a `DatosFacturacionCliente` | ⚠️ Ampliar alcance para instrumentar Bitácora General (H-01) |
| Tarea 4 | Controllers `BaseApiController` + `AsegurarFiltroRegion` | ✅ Sin cambio (cumple R3) |
| Tarea 5 | Actualizar `vDatosFacturacionCliente` | ✅ Sin cambio (cumple R1) |
| Tarea 6 | `ClaveFormaDePago` en `catMedioDePago` | ✅ Sin cambio |

### 6.2 Nueva tarea propuesta (T7)

**T7 — [ IMP-EXIST-SERVICE ] Instrumentar Bitácora General en `DatosFacturacionClienteBO._GuardarOActualizar`**

- **Aplicativos:** ProquifaDotNet + Aplicativo Nuevo Bitácora General
- **Módulos:** Catálogo de Clientes — Configuración de Facturación
- **Descripción:** cuando se guarde `DatosFacturacionCliente` y cambie alguno de los campos fiscales (`IdCatMetodoDePagoCFDI`, `IdCatUsoCFDI`, `AgenteRetencionIGV`, `SujetoDetraccion`, `TasaDetraccion`), llamar al servicio de Bitácora General enviando `IdCliente`, `IdUsuario`, valor previo, valor nuevo, `IdRegion` y timestamp.
- **Predecesor:** existencia y disponibilidad del cliente/SDK del aplicativo Bitácora General.
- **Estimación:** 12 h (`IMP-EXIST-SERVICE`)
- **Criterios de aceptación:**
  - [ ] `DatosFacturacionClienteBO._GuardarOActualizar` detecta cambios en los 5 campos fiscales antes de persistir.
  - [ ] Se llama al servicio de Bitácora General enviando el payload documentado en H-01.
  - [ ] Si el servicio de Bitácora falla, el UPDATE de `DatosFacturacionCliente` NO se aborta (fallo suave con log local).
  - [ ] Se cubre con prueba unitaria y de integración.

### 6.3 Total horas 005 tras aplicar reglas

| Aspecto | Antes | Después |
|---|---|---|
| Tareas existentes (T1-T6) | según diseño actual | sin cambio |
| Tarea nueva T7 (Bitácora General) | 0 h | **+12 h** |

---

## 7. Referencias

- `Diseño y Desarrollo/Reglas al codificar.md`
- `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005.md`
- `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005_BD.md` v1.2
- `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Back.md` v1.1
- `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Tareas.md`
- `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005_DIS-SOL_Revision.md` (revisión previa del PDF de diseño)
- `Requisitos/Catalogo BackEnd.md`

---

## 8. Veredicto

**Cumplimiento de reglas de codificación: ~95%**

- ✅ R1 (BD español), R3 (Catálogos español): cumplen sin observaciones.
- ⚠️ R8 (Bitácora General): **hallazgo crítico H-01** — falta instrumentación en el BO de guardado. Requiere reabrir alcance del 005 y agregar Tarea T7 (12 h).
- ℹ️ H-02, H-03, H-04: hallazgos menores, solo documentación / coordinación.

**Acción inmediata recomendada:**
1. Pegar los comentarios de la sección 5 en el documento de análisis abierto.
2. Levantar H-01 con el líder técnico y con Sebastián (autor del diseño 005).
3. Ajustar el `-Back.md` para agregar GAP-07 (Bitácora General) y actualizar `-Tareas.md` con T7.
