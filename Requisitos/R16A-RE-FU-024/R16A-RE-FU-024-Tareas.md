# Tareas BackEnd — R16A-RE-FU-024
**Requisito:** Validar Cobro: Paso 1 México — Captura del Cobro
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)

---

> **Orden de ejecución sugerido:** BD ProquifaDotNet (ALTERs RE-FU-023 ya ejecutados + **2 ALTERs nuevos RE-FU-024 — BloqueadoPorTimbrado y FechaBloqueoTimbrado** + CREATE tablas + SEQUENCE) → Integración Finanzas con catálogos existentes (moneda, medio de pago, cuenta destino) → Endpoints Finanzas (listado cobros → detalle correo → auto-guardado → finalización captura → **edición del cobro** → TC → modal inconsistencia → estado wizard).
> **Dependencias externas:** `fccFolioPagoCliente` / `CorreoRecibidoCliente` / `ArchivoCorreoRecibido` ya existentes (RE-FU-008). Tareas 1 y 1B (BD) son prerrequisitos de todas las tareas de Finanzas.

> **🔄 Cambio funcional 2026-06-23 — Editabilidad del cobro hasta el timbrado:**
> Se agregan dos tareas nuevas: **Tarea 1B (BD)** para crear los campos `BloqueadoPorTimbrado` y `FechaBloqueoTimbrado` en `fccPagoCliente`, y **Tarea 13 (Back)** para el endpoint de edición del cobro vía botón Editar. Se actualizan las tareas existentes 6 (listado: flag `canEdit`), 8 (auto-guardado: guardia cambia a `BloqueadoPorTimbrado`) y 9 (finalización de la captura: ya no aplica inmutabilidad, sin alerta de confirmación).

---

## Resumen de tareas

| #   | Clave            | Título simple                                                                                       | Tipo | Aplicativo               |
| --- | ---------------- | --------------------------------------------------------------------------------------------------- | ---- | ------------------------ |
| 1   | UPDATE-TABL-CH   | Agregar campos de captura, notas y moneda (IdCatMoneda) a fccPagoCliente *(referencia RE-FU-023)*   | BD   | ProquifaDotNet           |
| 1B  | UPDATE-TABL-CH   | **Agregar campos BloqueadoPorTimbrado y FechaBloqueoTimbrado a fccPagoCliente (RE-FU-024)**         | BD   | ProquifaDotNet           |
| 2   | CREATE-TABL-CH   | Crear tabla catTipoInconsistenciaCobro con datos iniciales                                          | BD   | ProquifaDotNet           |
| 3   | CREATE-TABL-M    | Crear tabla fccInconsistenciaCobro                                                                  | BD   | ProquifaDotNet           |
| 4   | BD-OBJ-CH        | Crear SEQUENCE SeqFolioCobro para foliador COB-mmddaa                                               | BD   | ProquifaDotNet           |
| 5   | LIST-NO-FILTER   | ~~Catálogos del formulario Paso 1~~ — **RETIRADA**, consumo directo de `/Catalogos/*` existentes, sin endpoint propio | Back | — |
| 6   | LIST-NO-FILTER   | Endpoint y servicio — Listado de cobros del cliente (Paso 1) *(incluye flag `canEdit`)*         | Back | ProquifaDotNet.Finanzas  |
| 7   | ALG-BASIC-LOGIC  | ~~Detalle del correo y adjuntos~~ — **RETIRADA**, consumo directo de `/Catalogos/*` existentes, sin endpoint propio | Back | — |
| 8   | SERV-SIMPLE-PUT  | Endpoint y servicio — Auto-guardado del cobro en borrador (Paso 1)                                  | Back | ProquifaDotNet.Finanzas  |
| 9   | SERV-TRANSACT    | Endpoint y servicio — Finalización de la captura con folio COB *(sin inmutabilidad, sin alerta)*    | Back | ProquifaDotNet.Finanzas  |
| 10  | ALG-COMPLX-LOGIC | Servicio — Tipo de Cambio del día automático para el cobro (México)                                 | Back | ProquifaDotNet.Finanzas  |
| 11  | SERV-SIMPLE-PUT  | Endpoint y servicio — Modal de inconsistencia del cobro (Paso 1)                                    | Back | ProquifaDotNet.Finanzas  |
| 12  | ALG-BASIC-LOGIC  | Endpoint y servicio — Estado del wizard para reanudación (OBS-048)                                  | Back | ProquifaDotNet.Finanzas  |
| 13  | SERV-SIMPLE-PUT  | **Endpoint y servicio — Edición del cobro vía botón Editar (mientras no esté timbrado) (RE-FU-024)** | Back | ProquifaDotNet.Finanzas  |

---

## TAREA 1

> **✅ EJECUTADO EN RE-FU-023** — Los 5 campos fueron incorporados al CREATE TABLE de RE-FU-023 y ejecutados en BD (verificado RYNL010). Esta tarea queda como referencia documental. No requiere ejecución.
>
> **Nota de actualización RE-FU-024:** La semántica del campo `Confirmado` cambia (deja de implicar inmutabilidad — ahora significa "cobro capturado y persistido con folio COB"). La inmutabilidad real se gestiona con los nuevos campos `BloqueadoPorTimbrado` y `FechaBloqueoTimbrado`, agregados en la **Tarea 1B** de este requisito.

**[ RE-FU-024 ] [UPDATE-TABL-CH] Agregar campos de captura, notas y moneda (IdCatMoneda) a fccPagoCliente**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 1

**Consideraciones previas:**
- ✅ Los 5 campos ya existen en `fccPagoCliente` (ejecutados en RE-FU-023):
  - `Confirmado bit NOT NULL DEFAULT(0)` *(semántica RE-FU-024: cobro capturado, no inmutable)*
  - `FechaConfirmacion datetime2 NULL` *(timestamp de la captura inicial)*
  - `IdUsuarioConfirmacion uniqueidentifier NULL` *(quién capturó el cobro inicialmente)*
  - `Notas varchar(500) NULL`
  - `IdCatMoneda uniqueidentifier NULL` con FK activa hacia `catMoneda.IdCatMoneda`
- Es **prerrequisito** de todas las tareas de Finanzas del Paso 1 (Tareas 5-13) — prerrequisito ya cumplido.

**Objetivo general:**
~~Extender `fccPagoCliente` con los 5 campos.~~ **Ya ejecutado en RE-FU-023.**

**Objetivos específicos:** *(referencia — ya ejecutado)*
- `ALTER TABLE dbo.fccPagoCliente ADD Confirmado bit NOT NULL CONSTRAINT [DF_fccPagoCliente_Confirmado] DEFAULT (0)`
- `ALTER TABLE dbo.fccPagoCliente ADD FechaConfirmacion datetime2 NULL`
- `ALTER TABLE dbo.fccPagoCliente ADD IdUsuarioConfirmacion uniqueidentifier NULL`
- `ALTER TABLE dbo.fccPagoCliente ADD Notas varchar(500) NULL`
- `ALTER TABLE dbo.fccPagoCliente ADD IdCatMoneda uniqueidentifier NULL CONSTRAINT [FK_fccPagoCliente_CatMoneda] FOREIGN KEY REFERENCES dbo.catMoneda([IdCatMoneda])`

**Resultado esperado:** ✅ Cumplido — campos existen en BD.

**Entregables:** ✅ Ejecutado — ver RE-FU-023 Tarea 2.

**Criterios de aceptación:** ✅ Verificados en BD (RYNL010).

**Más información de la tarea:**
Ver sección *"Parte A / A1 / Bloque A"* en `R16A-RE-FU-024-Back.md` y sección *"DDL Cambios en fccPagoCliente"* en `R16A-RE-FU-024_BD.md`.

**Recursos:**
- `R16A-RE-FU-024_BD.md` — Sección DDL + tabla completa post-R16
- `R16A-RE-FU-024-Back.md` — Parte A, sección A1

---

## TAREA 1B

**[ RE-FU-024 ] [UPDATE-TABL-CH] Agregar campos BloqueadoPorTimbrado y FechaBloqueoTimbrado a fccPagoCliente**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 1

**Consideraciones previas:**
- Tarea NUEVA en RE-FU-024 derivada del cambio funcional del 2026-06-23 (editabilidad del cobro hasta el timbrado).
- Los campos deben existir antes de ejecutar las Tareas 6, 8, 9 y 13 de Finanzas.
- `BloqueadoPorTimbrado` se mantiene en `0` mientras el documento asociado (factura/proforma) no haya sido timbrado en el Paso 3. El flip a `1` lo dispara el flujo del Paso 3.
- Para cobros existentes en BD (si los hubiera), el `DEFAULT(0)` deja inicialmente todos como editables. **Pendiente confirmar con PROQUIFA** si existen cobros ya timbrados en BD que deban inicializarse con `BloqueadoPorTimbrado=1` mediante un UPDATE de migración.
- No requiere índice nuevo (las consultas del Paso 1 filtran por `IdCliente` que ya tiene índice).

**Objetivo general:**
Agregar a `fccPagoCliente` los 2 campos que implementan la inmutabilidad post-timbrado y permiten distinguir cobros editables (botón Editar visible) de cobros inmutables.

**Objetivos específicos:**
- `ALTER TABLE dbo.fccPagoCliente ADD BloqueadoPorTimbrado bit NOT NULL CONSTRAINT [DF_fccPagoCliente_BloqueadoPorTimbrado] DEFAULT (0)`
- `ALTER TABLE dbo.fccPagoCliente ADD FechaBloqueoTimbrado datetime2 NULL`
- Validar que el `DEFAULT (0)` no rompe registros existentes.
- Confirmar con PROQUIFA si se requiere UPDATE de migración para cobros ya timbrados existentes.

**Resultado esperado:**
Tabla `fccPagoCliente` extendida con `BloqueadoPorTimbrado` y `FechaBloqueoTimbrado`, lista para las guardias del auto-guardado, el endpoint de edición y el flag `canEdit` del listado.

**Entregables:**
- Script DDL: 2 ALTER TABLE.
- Script de validación (estructura post-ALTER + verificación de valores DEFAULT en registros existentes).
- (Si aplica) Script DML de migración para cobros ya timbrados existentes.

**Criterios de aceptación:**
- Los 2 campos existen en `fccPagoCliente` con tipos y DEFAULT definidos.
- Registros existentes no se ven afectados (`BloqueadoPorTimbrado=0`, `FechaBloqueoTimbrado=NULL`).
- Ningún objeto dependiente presenta errores tras el ALTER.

**Más información de la tarea:**
Ver sección *"Parte A / A1 / Bloque B"* en `R16A-RE-FU-024-Back.md` y sección *"DDL Cambios en fccPagoCliente"* (Bloque B) en `R16A-RE-FU-024_BD.md`.

**Recursos:**
- `R16A-RE-FU-024_BD.md` — DDL Bloque B + ciclo de vida actualizado
- `R16A-RE-FU-024-Back.md` — Parte A, sección A1 (Bloque B)
- `R16A-RE-FU-024.md` — Regla 12 actualizada (editabilidad hasta el timbrado), Criterios B3, F3, F4

---
## TAREA 2

**[ RE-FU-024 ] [CREATE-TABL-CH] Crear tabla catTipoInconsistenciaCobro con datos iniciales**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 1

**Consideraciones previas:**
- Tabla nueva. No existe en BD.
- `AplicaPaso` varchar(1) discrimina: `'1'` = Paso 1 (cobro), `'2'` = Paso 2 (asociación).
- Datos iniciales son propuesta; catálogo completo pendiente de PROQUIFA Tesorería.
- Es **prerrequisito** de la Tarea 3 (`fccInconsistenciaCobro` tiene FK hacia esta tabla).
- Es **prerrequisito** de la Tarea 11 (modal de inconsistencia en Finanzas).

**Objetivo general:**
Crear el catálogo `catTipoInconsistenciaCobro` con estructura y datos iniciales para soportar el modal de inconsistencias del Paso 1 y del Paso 2 del wizard.

**Objetivos específicos:**
- DDL: PK NEWID, `Clave` varchar(50) UNIQUE, `Descripcion` varchar(200), `AplicaPaso` varchar(1), `Activo` bit DEFAULT(1).
- INSERT datos iniciales: tipos del Paso 1 (DATOS_INCOMPLETOS, COMPROBANTE_INVALIDO, FORMATO_INCORRECTO, MONTO_ILEGIBLE) y Paso 2 (PAGO_INCOMPLETO_VENCIDO, PAGO_INSUFICIENTE).

**Resultado esperado:**
Tabla `catTipoInconsistenciaCobro` con datos iniciales, lista para el modal de inconsistencias.

**Entregables:**
- Script DDL: `CREATE TABLE catTipoInconsistenciaCobro`
- Script DML: INSERT datos iniciales
- Script de validación

**Criterios de aceptación:**
- La tabla existe con la estructura definida en `R16A-RE-FU-024_BD.md`.
- 6 registros iniciales insertados con `Clave` y `AplicaPaso` correctos.
- UNIQUE constraint en `Clave` activo.
- Ningún objeto existente presenta errores.

**Más información de la tarea:**
Ver sección *"Parte A / A2"* en `R16A-RE-FU-024-Back.md` y sección *"Tabla Nueva: catTipoInconsistenciaCobro"* en `R16A-RE-FU-024_BD.md`.

**Recursos:**
- `R16A-RE-FU-024_BD.md` — DDL + INSERT datos iniciales
- `R16A-RE-FU-024-Back.md` — Parte A, sección A2

---

## TAREA 3

**[ RE-FU-024 ] [CREATE-TABL-M] Crear tabla fccInconsistenciaCobro**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 1

**Consideraciones previas:**
- Tabla nueva. No existe en BD.
- Depende de la Tarea 2 (`catTipoInconsistenciaCobro` debe existir primero; FK activo).
- Registra inconsistencias del Paso 1 y del Paso 2 contra `fccPagoCliente`.

**Objetivo general:**
Crear `fccInconsistenciaCobro` para registrar inconsistencias marcadas por el operador sobre un cobro, con trazabilidad de quién y cuándo.

**Objetivos específicos:**
- DDL: PK NEWID, FK `IdFCCPagoCliente`, FK `IdCatTipoInconsistenciaCobro`, `Comentario` varchar(500) NULL, `IdUsuarioRegistro`, `Activo` DEFAULT(1), `FechaRegistro` DEFAULT SYSUTCDATETIME(), `FechaUltimaActualizacion` DEFAULT SYSUTCDATETIME().

**Resultado esperado:**
Tabla `fccInconsistenciaCobro` existente y vacía, lista para recibir inconsistencias del Paso 1 y Paso 2.

**Entregables:**
- Script DDL: `CREATE TABLE fccInconsistenciaCobro`
- Script de validación (estructura + FKs verificadas)

**Criterios de aceptación:**
- La tabla existe con la estructura definida.
- FK activa hacia `fccPagoCliente.IdFCCPagoCliente`.
- FK activa hacia `catTipoInconsistenciaCobro.IdCatTipoInconsistenciaCobro`.
- Tabla vacía al crear.

**Más información de la tarea:**
Ver sección *"Parte A / A3"* en `R16A-RE-FU-024-Back.md` y sección *"Tabla Nueva: fccInconsistenciaCobro"* en `R16A-RE-FU-024_BD.md`.

**Recursos:**
- `R16A-RE-FU-024_BD.md` — DDL fccInconsistenciaCobro
- `R16A-RE-FU-024-Back.md` — Parte A, sección A3

---

## TAREA 4

**[ RE-FU-024 ] [BD-OBJ-CH] Crear SEQUENCE SeqFolioCobro para foliador COB-mmddaa**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 1

**Consideraciones previas:**
- Formato del folio: `COB-mmddaa-NNNNNN` (mmddaa = fecha efectiva del cobro, NNNNNN = consecutivo).
- `START WITH` debe ajustarse al MAX consecutivo existente + 1 si ya hay folios en `fccPagoCliente.Folio`.
- ⚠️ Pendiente confirmar: ¿consecutivo global (MEX+PER) o independiente por región?
- Prerrequisito de la Tarea 9 (confirmación del cobro en Finanzas).

**Objetivo general:**
Crear la SEQUENCE `SeqFolioCobro` para generar el consecutivo del folio del cobro.

**Objetivos específicos:**
- `CREATE SEQUENCE dbo.SeqFolioCobro AS INT START WITH 1 INCREMENT BY 1 NO CYCLE`
- Verificar `START WITH` correcto revisando MAX en `fccPagoCliente.Folio`.
- Documentar el patrón de uso de la SEQUENCE.

**Resultado esperado:**
SEQUENCE `SeqFolioCobro` existente, lista para el endpoint de confirmación de Finanzas.

**Entregables:**
- Script DDL: `CREATE SEQUENCE dbo.SeqFolioCobro`
- Script de validación

**Criterios de aceptación:**
- La SEQUENCE existe en ProquifaDotNet.
- `START WITH` configurado sin colisionar con folios existentes.

**Más información de la tarea:**
Ver sección *"Parte A / A4"* en `R16A-RE-FU-024-Back.md` y sección *"SEQUENCE SeqFolioCobro"* en `R16A-RE-FU-024_BD.md`.

**Recursos:**
- `R16A-RE-FU-024_BD.md` — SEQUENCE + lógica del folio COB
- `R16A-RE-FU-024-Back.md` — Parte A, sección A4

---

## TAREA 5

**[ RE-FU-024 ] [LIST-NO-FILTER] Catálogos del formulario Paso 1 (monedas, medios de pago, cuentas destino) — RETIRADA**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

> ⚠️ **Tarea retirada — no aplica.** Los 3 combos del formulario del Paso 1 (Moneda, Medio de pago, Cuenta destino) consumen **directamente** los endpoints existentes de ProquifaDotNet: `POST /Catalogos/catMoneda`, `POST /Catalogos/catMedioDePago`, `POST /Catalogos/vEmpresaDatosBancarios` — todos activos, no deprecados, ya usados por Venta Interna y ya con filtro de región implementado (`catMedioDePago` en RE-FU-005, `vEmpresaDatosBancarios` en RE-FU-001). Finanzas **no crea endpoints propios** para estos catálogos (no es una nueva app ni un wrapper) — no hay Query/Handler/DTO que implementar en esta tarea. Ver `R16A-RE-FU-024-Back.md` Parte C, sección C1, y `Endpoints-ProquifaDotNet.md` (RE-024 — Catálogo de Monedas / Buzón de Correo).
>
> Pendiente real (ver Brecha en `R16A-RE-FU-024-Back.md`): confirmar si `catMoneda` ya filtra por región como `catMedioDePago`/`vEmpresaDatosBancarios`; si no, requiere el mismo patrón de RE-FU-005 (GAP-04) — trabajo de BD/ProquifaDotNet, no de Finanzas.

---

## TAREA 6

**[ RE-FU-024 ] [LIST-NO-FILTER] Endpoint y servicio — Listado de cobros del cliente (Paso 1)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Tareas 1, **1B**, 2, 3, 4 deben estar ejecutadas.
- Dos bloques: capturados (`Confirmado=1`) arriba y sin capturar abajo.
- Si un cobro tiene saldo a favor (del Paso 2), etiqueta `isCreditBalance=true` en el DTO. Por construcción, todo cobro con saldo a favor visible en el Paso 1 está timbrado (`BloqueadoPorTimbrado=1`).
- **Actualización RE-FU-024:** el DTO debe exponer el flag `canEdit` calculado como `Confirmado && !BloqueadoPorTimbrado`, para que el Front condicione la visibilidad del botón Editar del Paso 1.

**Objetivo general:**
Implementar en Finanzas el endpoint que retorna el listado de cobros del cliente para el panel izquierdo del Paso 1, ordenado en dos bloques, incluyendo el flag de editabilidad.

**Objetivos específicos:**
- `GET /api/v1/validate-collection/payment?idCliente={idCliente}` en Finanzas.
- Crear `GetPaymentValidationStep1PaymentListQuery` + Handler.
- Bloque 1 (capturados): `fccPagoCliente.Confirmado=1`, ordenados por `FechaPago ASC`, con folio, fecha, monto, `catMoneda.ClaveMoneda` y flag `canEdit`.
- Bloque 2 (sin capturar): correos del Buzón sin cobro confirmado, ordenados por `FechaRecepcion ASC`.
- DTO: `PaymentValidationStep1ItemDto` (estado, folio, fecha, monto, currencyCode, isCreditBalance, **canEdit**, **blockedByStamping**, **stampingBlockDate**).

**Resultado esperado:**
Endpoint `GET /api/v1/validate-collection/payment?idCliente={idCliente}` en Finanzas que retorna el listado de dos bloques para el panel izquierdo del Paso 1 y permite al Front decidir cuándo mostrar el botón Editar.

**Entregables:**
- Endpoint + Query + Handler: `GetPaymentValidationStep1PaymentListQuery`
- DTO: `PaymentValidationStep1ItemDto` (con flag `canEdit`)
- Pruebas unitarias (incluyendo casos: capturado editable, capturado bloqueado por timbrado, saldo a favor)

**Criterios de aceptación:**
- Bloque 1: cobros capturados ordenados por `FechaPago ASC` con moneda desde `IdCatMoneda → catMoneda.ClaveMoneda`.
- Bloque 2: correos sin cobro capturado ordenados por `FechaRecepcion ASC`.
- Items en modo lectura. `canEdit=true` solo si `Confirmado=1 AND BloqueadoPorTimbrado=0`.
- Items con `isCreditBalance=true` retornan siempre `canEdit=false`.

**Más información de la tarea:**
Ver sección *"Parte B / B1"* en `R16A-RE-FU-024-Back.md`. Ver criterios B1, B2, B3 y regla 12 actualizada en `R16A-RE-FU-024.md`.

**Recursos:**
- `R16A-RE-FU-024-Back.md` — Parte B, sección B1
- `R16A-RE-FU-024_BD.md` — Tablas leídas + ciclo de vida

---

## TAREA 7

**[ RE-FU-024 ] [ALG-BASIC-LOGIC] Detalle del correo y adjuntos para selección de comprobante — RETIRADA**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

> ⚠️ **Tarea retirada — no aplica.** Confirmado por captura de tráfico HTTP real (07/07/2026): el frontend de Validar Cobro consume **directamente** `GET /Catalogos/CorreoRecibido`, `GET /Catalogos/CorreoRecibidoContenido`, `POST /Catalogos/ArchivoCorreoRecibido` y `GET /Catalogos/Archivo` — todos endpoints existentes de ProquifaDotNet, activos, no deprecados. Finanzas **no crea un endpoint propio** de detalle de correo/adjuntos (no hay Query/Handler/DTO que implementar aquí). El frontend envía el `IdArchivo` seleccionado directamente al auto-guardado de Finanzas (Tarea 8). Ver `R16A-RE-FU-024-Back.md` Parte B, sección B2, y `Endpoints-ProquifaDotNet.md` (RE-024 — Buzón de Correo).

---

## TAREA 8

**[ RE-FU-024 ] [SERV-SIMPLE-PUT] Endpoint y servicio — Auto-guardado del cobro en borrador (Paso 1)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Auto-guardado transparente: sin botón "Guardar" manual.
- Si no existe `fccPagoCliente`: INSERT con `Confirmado=0`, `BloqueadoPorTimbrado=0`, `Folio=NULL`, `IdCatMoneda=@Id`.
- Si existe con `Confirmado=0`: UPDATE con datos actuales incluido `IdCatMoneda`.
- **Guardia actualizada RE-FU-024:** si `BloqueadoPorTimbrado=1`, no sobreescribir (cobro inmutable post-timbrado). Si `Confirmado=1 AND BloqueadoPorTimbrado=0`, la edición debe canalizarse por el endpoint de edición (Tarea 13), no por este auto-guardado.
- Tareas 1 y 1B son prerrequisitos.
- **OBS-048:** El estado del cobro en borrador (`Confirmado=0`) es la señal que usa el endpoint de estado del wizard (Tarea 12) para determinar que el cliente tiene el Paso 1 en progreso y reanudarlo ahí. El auto-guardado es el mecanismo que persiste ese estado.

**Objetivo general:**
Implementar en Finanzas el endpoint de auto-guardado que persiste el formulario del cobro como borrador de forma transparente, incluyendo la moneda seleccionada (`IdCatMoneda`), respetando la guardia de inmutabilidad por timbrado.

**Objetivos específicos:**
- `PUT /api/v1/validate-collection/payment` (INSERT borrador / UPDATE borrador, Body incluye `Id` si ya existe)
- Crear `AutoSavePaymentDraftCommand` + Handler.
- DTO de request: `AutoSavePaymentDraftRequestDto` (monto, fechaPago, idCatMedioDePago, cuentaOrdenante, idDatosBancarios, **idCatMoneda**, tipoCambio, notas, idArchivo).
- **Guardia por timbrado:** no actualizar si `BloqueadoPorTimbrado=1` (retorna 409 Conflict).
- Guardia por estado capturado: no actualizar si `Confirmado=1` (la modificación va por Tarea 13 / endpoint Editar).

**Resultado esperado:**
Endpoint `PUT .../payment` que auto-guarda el cobro sin requerir acción del usuario, preservando todos los campos del formulario incluyendo la moneda seleccionada, sin tocar cobros bloqueados por timbrado ni cobros ya capturados.

**Entregables:**
- Endpoint + Command + Handler: `AutoSavePaymentDraftCommand`
- DTO: `AutoSavePaymentDraftRequestDto`
- Pruebas unitarias (incluyendo guardia por timbrado y guardia por captura finalizada)

**Criterios de aceptación:**
- INSERT correcto si no existe cobro, con `IdCatMoneda` poblado y `BloqueadoPorTimbrado=0`.
- UPDATE correcto si existe con `Confirmado=0`, actualizando `IdCatMoneda`.
- Si `BloqueadoPorTimbrado=1`, el endpoint NO modifica el registro y retorna 409 Conflict.
- Si `Confirmado=1`, el endpoint NO modifica el registro (la edición debe canalizarse por Tarea 13).

**Más información de la tarea:**
Ver sección *"Parte B / B4"* en `R16A-RE-FU-024-Back.md`. Ver criterios F1, F2 en `R16A-RE-FU-024.md`.

**Recursos:**
- `R16A-RE-FU-024-Back.md` — Parte B, sección B4
- `R16A-RE-FU-024_BD.md` — Tablas escritas runtime

---

## TAREA 9

**[ RE-FU-024 ] [SERV-TRANSACT] Endpoint y servicio — Finalización de la captura del cobro con folio COB (sin inmutabilidad, sin alerta)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Tarea 4 (SEQUENCE) y Tarea 1B (`BloqueadoPorTimbrado`) deben existir antes.
- Flujo en Finanzas: (1) validar comprobante seleccionado, (2) validar campos obligatorios (incluido `IdCatMoneda`), (3) calcular folio COB. **No se muestra alerta de confirmación al Front** (la captura no es inmutable hasta el timbrado del documento asociado).
- UPDATE en ProquifaDotNet incluye `IdCatMoneda` en el SET del cobro capturado. **NO toca `BloqueadoPorTimbrado` (permanece en 0)**: el cobro queda editable vía botón Editar (Tarea 13) hasta el timbrado.
- Guardia `Confirmado=0` en el WHERE del UPDATE (este endpoint solo finaliza borradores, no recaptura).

**Objetivo general:**
Implementar en Finanzas el endpoint transaccional que finaliza la captura del cobro: genera folio COB-mmddaa-NNNNNN, persiste `Confirmado=1` con trazabilidad e `IdCatMoneda`. **El cobro NO queda inmutable** — la inmutabilidad se aplica en el Paso 3 al timbrar el documento asociado (ver Tarea 1B y endpoint B10).

**Objetivos específicos:**
- `POST /api/v1/validate-collection/payment/{idFCCFolioPagoCliente}/finalize`
- Crear `FinalizePaymentCaptureCommand` + Handler.
- Validar en Finanzas: comprobante seleccionado + `IdCatMoneda` obligatorio + demás campos requeridos.
- Calcular folio con `NEXT VALUE FOR SeqFolioCobro`.
- UPDATE en ProquifaDotNet: `Folio=@Folio, Confirmado=1, FechaConfirmacion=..., IdUsuarioConfirmacion=..., IdCatMoneda=@IdCatMoneda WHERE Confirmado=0`. **No tocar `BloqueadoPorTimbrado`.**
- Sin alerta de confirmación al Front (la captura no es definitiva hasta el timbrado).

**Resultado esperado:**
Endpoint `POST .../payment/{id}/finalize` que finaliza la captura con folio definitivo y moneda registrada, dejando el cobro en modo lectura **editable** vía botón Editar (`Confirmado=1, BloqueadoPorTimbrado=0`).

**Entregables:**
- Endpoint + Command + Handler: `FinalizePaymentCaptureCommand`
- Pruebas unitarias (fallo si comprobante no seleccionado, fallo si `IdCatMoneda` nulo, fallo si ya `Confirmado=1`, verificación de `BloqueadoPorTimbrado=0` post-finalización)

**Criterios de aceptación:**
- `Confirmado=1`, `FechaConfirmacion`, `IdUsuarioConfirmacion`, `Folio` y `IdCatMoneda` poblados.
- `BloqueadoPorTimbrado` permanece en `0` tras la finalización.
- Folio en formato `COB-mmddaa-NNNNNN`.
- Error si `IdCatMoneda` es nulo al finalizar.
- Error si el cobro ya tiene `Confirmado=1` (no recapturar).
- Tras la finalización, el cobro puede editarse vía Tarea 13 mientras `BloqueadoPorTimbrado=0`.

**Más información de la tarea:**
Ver sección *"Parte B / B5"* en `R16A-RE-FU-024-Back.md`. Ver criterios F3, F4 y regla 12 actualizada en `R16A-RE-FU-024.md`.

**Recursos:**
- `R16A-RE-FU-024-Back.md` — Parte B, sección B5
- `R16A-RE-FU-024_BD.md` — Lógica del Folio COB + ciclo de vida actualizado

---

## TAREA 10

**[ RE-FU-024 ] [ALG-COMPLX-LOGIC] Servicio — Tipo de Cambio del día automático para el cobro (México)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- El TC se calcula automáticamente al seleccionar la moneda del cobro (`IdCatMoneda`).
- La lógica usa `catMoneda.ClaveMoneda` del `IdCatMoneda` seleccionado para determinar qué TC calcular.
- Fuente: TC FIX Banxico/DOF — ⚠️ pendiente confirmar con PROQUIFA.
- El TC se persiste en `fccPagoCliente.TipoDeCambio` al confirmar el cobro.

**Objetivo general:**
Implementar en Finanzas el servicio `ExchangeRateService` que calcula el TC del día para la moneda del cobro seleccionada en el combo (`IdCatMoneda`), según la regla de la moneda vs MXN.

**Objetivos específicos:**
- Crear `IExchangeRateService.GetDailyExchangeRateAsync(idCatMonedaCobro, idCatMonedaFacturacion)`.
- Lógica: si ambas son MXN → N/A; si cobro=MXN y facturación≠MXN → TC de la moneda de facturación; si cobro≠MXN → TC de la moneda del cobro.
- `GET /api/v1/validate-collection/exchangeRate?sourceCurrencyId=...&targetCurrencyId=...`

**Resultado esperado:**
Servicio en Finanzas que retorna el TC del día según la moneda seleccionada en el combo, listo para el formulario del Paso 1 y la confirmación del cobro.

**Entregables:**
- `IExchangeRateService` + `ExchangeRateService`
- Endpoint `GET /api/v1/validate-collection/exchangeRate`
- Pruebas unitarias (3 escenarios: N/A, TC moneda facturación, TC moneda cobro)

**Criterios de aceptación:**
- Si ambas monedas son MXN: retorna `{ "exchangeRate": null, "isNotApplicable": true }`.
- Si cobro=MXN y facturación≠MXN: retorna TC de la moneda de facturación vs MXN.
- Si cobro≠MXN: retorna TC de la moneda del cobro vs MXN.
- Valor solo lectura en el formulario.

**Más información de la tarea:**
Ver sección *"Parte B / B6"* en `R16A-RE-FU-024-Back.md`. Ver regla 7 y criterio D8 en `R16A-RE-FU-024.md`.

**Recursos:**
- `R16A-RE-FU-024-Back.md` — Parte B, sección B6
- `R16A-RE-FU-024.md` — Regla 7, Criterio D8

---

## TAREA 11

**[ RE-FU-024 ] [SERV-SIMPLE-PUT] Endpoint y servicio — Modal de inconsistencia del cobro (Paso 1)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Tareas 2 y 3 deben existir (`catTipoInconsistenciaCobro` y `fccInconsistenciaCobro`).
- Solo tipos con `AplicaPaso='1'` para el combo del modal del Paso 1.
- Tipo `PAGO_INCOMPLETO_VENCIDO` (`AplicaPaso='2'`) NO debe aparecer en el Paso 1.

**Objetivo general:**
Implementar en Finanzas el endpoint de catálogo filtrado y el endpoint de registro de inconsistencia del Paso 1.

**Objetivos específicos:**
- `GET /api/v1/validate-collection/inconsistencyType?step=1` → catálogo filtrado `AplicaPaso='1'`.
- `POST /api/v1/validate-collection/client/{idCliente}/payment/{idFCCPagoCliente}/inconsistency` → INSERT en `fccInconsistenciaCobro`.
- Crear `RegisterPaymentInconsistencyCommand` + Handler.
- Validar que el tipo seleccionado tiene `AplicaPaso='1'`.
- DTO: `RegisterInconsistencyRequestDto` (idCatTipoInconsistenciaCobro, comentario).

**Resultado esperado:**
Modal de inconsistencias del Paso 1 con combo filtrado correctamente y registro persistido en `fccInconsistenciaCobro`.

**Entregables:**
- Endpoint `GET /api/v1/validate-collection/inconsistencyType?step=1`
- Endpoint `POST .../inconsistency`
- Command + Handler: `RegisterPaymentInconsistencyCommand`
- DTOs + pruebas unitarias

**Criterios de aceptación:**
- Combo del modal solo muestra tipos `AplicaPaso='1'`.
- `PAGO_INCOMPLETO_VENCIDO` NO aparece en el Paso 1.
- Inconsistencia insertada correctamente en `fccInconsistenciaCobro`.
- Comentario es opcional (puede ser null).

**Más información de la tarea:**
Ver sección *"Parte B / B7"* en `R16A-RE-FU-024-Back.md`. Ver criterios G1-G4 y regla 14 en `R16A-RE-FU-024.md`.

**Recursos:**
- `R16A-RE-FU-024-Back.md` — Parte B, sección B7
- `R16A-RE-FU-024_BD.md` — catTipoInconsistenciaCobro + fccInconsistenciaCobro

---

## TAREA 12

**[ RE-FU-024 ] [ALG-BASIC-LOGIC] Endpoint y servicio — Estado del wizard para reanudación (OBS-048)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Wizard (estado general)

**Consideraciones previas:**
- **OBS-048:** El wizard no siempre debe abrir en el Paso 1. Al entrar al wizard para un cliente, la UI consulta este endpoint para saber en qué paso reanudar.
- La lógica evalúa el estado de cada paso: si el Paso 1 tiene borrador activo (`Confirmado=0`) o cobros sin capturar, se reanuda en Paso 1. Si todos los cobros están confirmados y el Paso 2 tiene trabajo pendiente, se reanuda en Paso 2. Y así sucesivamente.
- Esta tarea cubre la base del Paso 1. Los requisitos RE-FU-025 (Paso 2) y Paso 3 ampliarán la lógica de evaluación.

**Objetivo general:**
Implementar en Finanzas el endpoint `GET /api/v1/validate-collection/client/{idCliente}/wizardStatus` que retorna el paso activo donde el usuario debe continuar el wizard de Validar Cobro.

**Objetivos específicos:**
- Crear `GetWizardStatusQuery` + Handler.
- Evaluar: si hay `fccPagoCliente.Confirmado=0` para el cliente → Paso 1 activo. Si no, determinar si hay trabajo pendiente en Paso 2 (lógica extendida en RE-FU-025).
- DTO de respuesta: `WizardStatusDto` con campos `ActiveStep` (int), `Description` (string), `HasDraft` (bool), `DraftPaymentId` (guid?, solo si aplica).

**Resultado esperado:**
Endpoint `GET .../wizardStatus` que permite a la UI redirigir al usuario directamente al paso activo del wizard, sin forzarlo a comenzar siempre desde el Paso 1.

**Entregables:**
- Endpoint + Query + Handler: `GetWizardStatusQuery`
- DTO: `WizardStatusDto`
- Pruebas unitarias (escenarios: sin cobros → Paso 1, borrador activo → Paso 1, todos confirmados → Paso 2)

**Criterios de aceptación:**
- Si no hay `fccPagoCliente` para el cliente, retorna `ActiveStep=1`.
- Si existe `fccPagoCliente.Confirmado=0`, retorna `ActiveStep=1` con `HasDraft=true` e `DraftPaymentId` poblado.
- Si todos los cobros están `Confirmado=1` y hay trabajo pendiente en Paso 2, retorna `ActiveStep=2` (lógica base; se detalla en RE-FU-025).
- La UI no muestra el Paso 1 si el usuario ya completó todos los cobros del Buzón (OBS-048).

**Más información de la tarea:**
Ver sección *"Parte B / B8"* en `R16A-RE-FU-024-Back.md`. Ver Regla 11 y Criterio F1 en `R16A-RE-FU-024.md`.

**Recursos:**
- `R16A-RE-FU-024-Back.md` — Parte B, sección B8
- `R16A-RE-FU-024.md` — Regla 11, Criterio F1

---

## TAREA 13

**[ RE-FU-024 ] [SERV-SIMPLE-PUT] Endpoint y servicio — Edición del cobro vía botón Editar (mientras no esté timbrado)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Tarea NUEVA en RE-FU-024 derivada del cambio funcional del 2026-06-23.
- Prerrequisito: Tarea 1B (campos `BloqueadoPorTimbrado` y `FechaBloqueoTimbrado`) y Tarea 9 (un cobro debe estar capturado para poder editarse).
- Se ejecuta al presionar el botón **Editar** del item del listado del Paso 1, disponible solo cuando `canEdit=true` (flag retornado por Tarea 6).
- Aplica aun si el cobro ya está asociado en el Paso 2 (no requiere desasociar para editar).
- Si el usuario cambia `IdCatMoneda` o `FechaPago`, el handler debe **recalcular TC** invocando al servicio de la Tarea 10 (`ExchangeRateService`).
- NO regenerar `Folio` (el folio capturado se mantiene).
- La validación de comprobante seleccionado sigue siendo obligatoria.
- **Impacto en Paso 2 (RE-FU-026):** si el cobro editado ya estaba aplicado a un documento, las conversiones operativas pueden quedar desactualizadas; el detalle de la re-evaluación se documenta en RE-FU-026.

**Objetivo general:**
Implementar en Finanzas el endpoint que permite editar todos los campos de un cobro capturado mientras el documento asociado no haya sido timbrado, respetando la inmutabilidad por timbrado y recalculando TC si la moneda o fecha cambian.

**Objetivos específicos:**
- `PUT /api/v1/validate-collection/payment/{idFCCPagoCliente}/edit`
- Crear `EditPaymentCommand` + Handler.
- Validar `Confirmado=1` (solo cobros capturados son editables aquí) y `BloqueadoPorTimbrado=0` (sino retornar `409 Conflict`).
- DTO de request: `EditPaymentRequestDto` (monto, fechaPago, idCatMedioDePago, cuentaOrdenante, idDatosBancarios, idCatMoneda, idArchivo, notas).
- Recalcular TC si cambió `IdCatMoneda` o `FechaPago`.
- UPDATE en ProquifaDotNet sobre todos los campos editables + `FechaUltimaActualizacion=SYSUTCDATETIME()`. Guardia en WHERE: `Confirmado=1 AND BloqueadoPorTimbrado=0`.

**Resultado esperado:**
Endpoint `PUT .../payment/{id}/edit` que permite corregir errores de captura sobre un cobro capturado y editable, manteniendo su folio y dejando el cobro listo para ser re-aplicado/validado en Paso 2 si ya estaba asociado.

**Entregables:**
- Endpoint + Command + Handler: `EditPaymentCommand`
- DTO: `EditPaymentRequestDto`
- Pruebas unitarias: éxito en edición de cobro editable, 409 si `BloqueadoPorTimbrado=1`, recálculo de TC al cambiar moneda/fecha, error si comprobante no seleccionado, error si campos obligatorios vacíos.

**Criterios de aceptación:**
- El endpoint actualiza todos los campos editables del cobro cuando `Confirmado=1 AND BloqueadoPorTimbrado=0`.
- Si `BloqueadoPorTimbrado=1`, retorna `409 Conflict` con código de error estandarizado y no modifica el registro.
- El folio (`Folio`) no se regenera.
- Si el usuario cambió la moneda o la fecha, el `TipoDeCambio` queda recalculado vía servicio de Tarea 10.
- La validación de comprobante sigue siendo obligatoria (error 400 si no hay `IdArchivo`).
- `FechaUltimaActualizacion` queda actualizada.

**Más información de la tarea:**
Ver sección *"Parte B / B9"* en `R16A-RE-FU-024-Back.md`. Ver Regla 12 actualizada y Criterios B3, F3, F4 en `R16A-RE-FU-024.md`.

**Recursos:**
- `R16A-RE-FU-024-Back.md` — Parte B, sección B9 (Editar) y B10 (bloqueo por timbrado disparado desde Paso 3)
- `R16A-RE-FU-024_BD.md` — Ciclo de vida del registro + tablas escritas runtime
- `R16A-RE-FU-024.md` — Regla 12, Criterios B3, F3, F4
