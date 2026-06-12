# Tareas BackEnd — TPSC-RE-FU-024
**Requisito:** Validar Cobro: Paso 1 México — Captura del Cobro
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)

---

> **Orden de ejecución sugerido:** BD ProquifaDotNet (5 ALTERs + CREATE tablas + SEQUENCE) → Endpoints Finanzas (catálogos → listado cobros → detalle correo → auto-guardado → confirmación → TC → modal inconsistencia)
> **Dependencias externas:** `fccFolioPagoCliente` / `CorreoRecibidoCliente` / `ArchivoCorreoRecibido` ya existentes (RE-FU-008). Tarea 1 (BD) es prerrequisito de todas las tareas de Finanzas.

---

## Resumen de tareas

| # | Clave | Título simple | Tipo | Aplicativo |
|---|-------|--------------|------|-----------|
| 1 | UPDATE-TABL-CH | Agregar campos de inmutabilidad, notas y moneda (IdCatMoneda) a fccPagoCliente | BD | ProquifaDotNet |
| 2 | CREATE-TABL-CH | Crear tabla catTipoInconsistenciaCobro con datos iniciales | BD | ProquifaDotNet |
| 3 | CREATE-TABL-M | Crear tabla fccInconsistenciaCobro | BD | ProquifaDotNet |
| 4 | BD-OBJ-CH | Crear SEQUENCE SeqFolioCobro para foliador COB-mmddaa | BD | ProquifaDotNet |
| 5 | LIST-NO-FILTER | Endpoint y servicio — Catálogos del formulario Paso 1 (monedas, medios de pago, cuentas destino) | Back | ProquifaDotNet.Finanzas |
| 6 | LIST-NO-FILTER | Endpoint y servicio — Listado de cobros del cliente (Paso 1) | Back | ProquifaDotNet.Finanzas |
| 7 | ALG-BASIC-LOGIC | Endpoint y servicio — Detalle del correo y adjuntos para selección de comprobante | Back | ProquifaDotNet.Finanzas |
| 8 | SERV-SIMPLE-PUT | Endpoint y servicio — Auto-guardado del cobro en borrador (Paso 1) | Back | ProquifaDotNet.Finanzas |
| 9 | SERV-TRANSACT | Endpoint y servicio — Confirmación del cobro con folio COB e inmutabilidad | Back | ProquifaDotNet.Finanzas |
| 10 | ALG-COMPLX-LOGIC | Servicio — Tipo de Cambio del día automático para el cobro (México) | Back | ProquifaDotNet.Finanzas |
| 11 | SERV-SIMPLE-PUT | Endpoint y servicio — Modal de inconsistencia del cobro (Paso 1) | Back | ProquifaDotNet.Finanzas |
| 12 | ALG-BASIC-LOGIC | Endpoint y servicio — Estado del wizard para reanudación (OBS-048) | Back | ProquifaDotNet.Finanzas |

---

## TAREA 1

> **✅ EJECUTADO EN RE-FU-023** — Los 5 campos fueron incorporados al CREATE TABLE de RE-FU-023 y ejecutados en BD (verificado RYNL010). Esta tarea queda como referencia documental. No requiere ejecución.

**[ RE-FU-024 ] [UPDATE-TABL-CH] Agregar campos de inmutabilidad, notas y moneda (IdCatMoneda) a fccPagoCliente**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 1

**Consideraciones previas:**
- ✅ Los 5 campos ya existen en `fccPagoCliente` (ejecutados en RE-FU-023):
  - `Confirmado bit NOT NULL DEFAULT(0)`
  - `FechaConfirmacion datetime2 NULL`
  - `IdUsuarioConfirmacion uniqueidentifier NULL`
  - `Notas varchar(500) NULL`
  - `IdCatMoneda uniqueidentifier NULL` con FK activa hacia `catMoneda.IdCatMoneda`
- Es **prerrequisito** de todas las tareas de Finanzas del Paso 1 (Tareas 5-11) — prerrequisito ya cumplido.

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
Ver sección *"Parte A / A1"* en `TPSC-RE-FU-024-Back.md` y sección *"DDL Cambios en fccPagoCliente"* en `TPSC-RE-FU-024_BD.md`.

**Recursos:**
- `TPSC-RE-FU-024_BD.md` — Sección DDL + tabla completa post-R16
- `TPSC-RE-FU-024-Back.md` — Parte A, sección A1

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
- La tabla existe con la estructura definida en `TPSC-RE-FU-024_BD.md`.
- 6 registros iniciales insertados con `Clave` y `AplicaPaso` correctos.
- UNIQUE constraint en `Clave` activo.
- Ningún objeto existente presenta errores.

**Más información de la tarea:**
Ver sección *"Parte A / A2"* en `TPSC-RE-FU-024-Back.md` y sección *"Tabla Nueva: catTipoInconsistenciaCobro"* en `TPSC-RE-FU-024_BD.md`.

**Recursos:**
- `TPSC-RE-FU-024_BD.md` — DDL + INSERT datos iniciales
- `TPSC-RE-FU-024-Back.md` — Parte A, sección A2

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
Ver sección *"Parte A / A3"* en `TPSC-RE-FU-024-Back.md` y sección *"Tabla Nueva: fccInconsistenciaCobro"* en `TPSC-RE-FU-024_BD.md`.

**Recursos:**
- `TPSC-RE-FU-024_BD.md` — DDL fccInconsistenciaCobro
- `TPSC-RE-FU-024-Back.md` — Parte A, sección A3

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
Ver sección *"Parte A / A4"* en `TPSC-RE-FU-024-Back.md` y sección *"SEQUENCE SeqFolioCobro"* en `TPSC-RE-FU-024_BD.md`.

**Recursos:**
- `TPSC-RE-FU-024_BD.md` — SEQUENCE + lógica del folio COB
- `TPSC-RE-FU-024-Back.md` — Parte A, sección A4

---

## TAREA 5

**[ RE-FU-024 ] [LIST-NO-FILTER] Endpoint y servicio — Catálogos del formulario Paso 1 (monedas, medios de pago, cuentas destino)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Tarea 1 debe estar ejecutada (`IdCatMoneda` disponible en `fccPagoCliente`).
- El formulario del Paso 1 tiene 3 combos que necesitan catálogos desde ProquifaDotNet:
  - **Moneda** → `catMoneda WHERE Activo=1` (persiste como `IdCatMoneda` en `fccPagoCliente`)
  - **Medio de pago** → `catMedioDePago WHERE ClaveFormaDePago IS NOT NULL` (SAT, México)
  - **Cuenta destino** → `EmpresaDatosBancarios + DatosBancarios` (empresas GOL/MUN/PRO/PQF, región MEX)
- Estos endpoints se adaptan para Perú en RE-FU-025 (con distintos filtros por región).

**Objetivo general:**
Implementar en Finanzas los 3 endpoints de catálogos que poblan los combos del formulario del Paso 1.

**Objetivos específicos:**
- `GET /api/validar-cobro/catalogos/monedas` → retorna `catMoneda` activas.
- `GET /api/validar-cobro/catalogos/medios-pago?region=MEX` → retorna `catMedioDePago` con `ClaveFormaDePago IS NOT NULL`.
- `GET /api/validar-cobro/catalogos/cuentas-destino?region=MEX` → retorna cuentas bancarias de PROQUIFA México.
- Llamar a ProquifaDotNet vía API para obtener los datos.
- Crear DTOs: `MonedaDto`, `MedioPagoDto`, `CuentaDestinoDto`.

**Resultado esperado:**
3 endpoints de catálogos en Finanzas que retornan los datos correctos para poblar los combos del formulario del Paso 1, con soporte de filtro por región para reutilización en Perú (RE-FU-025).

**Entregables:**
- 3 endpoints: `GET /catalogos/monedas`, `GET /catalogos/medios-pago`, `GET /catalogos/cuentas-destino`
- Queries + Handlers: `GetMonedasQuery`, `GetMediosPagoQuery`, `GetCuentasDestinoQuery`
- DTOs: `MonedaDto`, `MedioPagoDto`, `CuentaDestinoDto`
- Pruebas unitarias de los Handlers

**Criterios de aceptación:**
- El combo "Moneda" retorna todas las monedas activas de `catMoneda` (MXN, USD, PEN, etc.).
- El combo "Medio de pago" para MEX retorna solo registros con `ClaveFormaDePago IS NOT NULL`.
- El combo "Cuenta destino" retorna solo cuentas de empresas GOL/MUN/PRO/PQF para región MEX.
- Los endpoints aceptan parámetro `region` para reutilización en Perú.

**Más información de la tarea:**
Ver sección *"Parte B / B3"* en `TPSC-RE-FU-024-Back.md`.

**Recursos:**
- `TPSC-RE-FU-024-Back.md` — Parte B, sección B3
- `TPSC-RE-FU-024_BD.md` — catMoneda, catMedioDePago, EmpresaDatosBancarios

---

## TAREA 6

**[ RE-FU-024 ] [LIST-NO-FILTER] Endpoint y servicio — Listado de cobros del cliente (Paso 1)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Tareas 1-4 deben estar ejecutadas.
- Dos bloques: capturados (`Confirmado=1`) arriba y sin capturar abajo.
- Si un cobro tiene saldo a favor (del Paso 2), etiqueta `esSaldoAFavor=true` en el DTO.

**Objetivo general:**
Implementar en Finanzas el endpoint que retorna el listado de cobros del cliente para el panel izquierdo del Paso 1, ordenado en dos bloques.

**Objetivos específicos:**
- `GET /api/validar-cobro/clientes/{idCliente}/cobros` en Finanzas.
- Crear `GetValidarCobroPaso1CobroListQuery` + Handler.
- Bloque 1 (capturados): `fccPagoCliente.Confirmado=1`, ordenados por `FechaPago ASC`, con folio, fecha, monto y `catMoneda.ClaveMoneda`.
- Bloque 2 (sin capturar): correos del Buzón sin cobro confirmado, ordenados por `FechaRecepcion ASC`.
- DTO: `ValidarCobroPaso1ItemDto` (estado, folio, fecha, monto, claveMoneda, esSaldoAFavor).

**Resultado esperado:**
Endpoint `GET /api/validar-cobro/clientes/{idCliente}/cobros` en Finanzas que retorna el listado de dos bloques para el panel izquierdo del Paso 1.

**Entregables:**
- Endpoint + Query + Handler: `GetValidarCobroPaso1CobroListQuery`
- DTO: `ValidarCobroPaso1ItemDto`
- Pruebas unitarias

**Criterios de aceptación:**
- Bloque 1: cobros confirmados ordenados por `FechaPago ASC` con moneda desde `IdCatMoneda → catMoneda.ClaveMoneda`.
- Bloque 2: correos sin cobro confirmado ordenados por `FechaRecepcion ASC`.
- Items capturados en modo solo lectura (sin edición).

**Más información de la tarea:**
Ver sección *"Parte B / B1"* en `TPSC-RE-FU-024-Back.md`. Ver criterios B1, B2, B3 en `TPSC-RE-FU-024.md`.

**Recursos:**
- `TPSC-RE-FU-024-Back.md` — Parte B, sección B1
- `TPSC-RE-FU-024_BD.md` — Tablas leídas

---

## TAREA 7

**[ RE-FU-024 ] [ALG-BASIC-LOGIC] Endpoint y servicio — Detalle del correo y adjuntos para selección de comprobante**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Retorna detalle del correo seleccionado (asunto, cuerpo, fecha, hora, contacto del cliente) y lista de adjuntos como radio buttons.
- La selección del adjunto como comprobante se guarda en `fccPagoCliente.IdArchivo` al auto-guardar (Tarea 8).
- Es prerrequisito de la validación de comprobante en la confirmación (Tarea 9).

**Objetivo general:**
Implementar en Finanzas el endpoint que retorna el detalle del correo y la lista de adjuntos para la selección del comprobante de pago oficial.

**Objetivos específicos:**
- `GET /api/validar-cobro/clientes/{idCliente}/correos/{idFCCFolioPagoCliente}/detalle`
- Crear `GetValidarCobroDetalleCorreoQuery` + Handler.
- Retornar: asunto, cuerpo, fecha, hora, contacto del cliente y adjuntos (`List<AdjuntoDto>`).
- Si el cobro ya tiene `IdArchivo` previo (borrador), marcarlo como seleccionado en la respuesta.

**Resultado esperado:**
Endpoint `GET .../correos/{id}/detalle` que retorna detalle del correo con lista de adjuntos para selección de comprobante.

**Entregables:**
- Endpoint + Query + Handler: `GetValidarCobroDetalleCorreoQuery`
- DTOs: `ValidarCobroDetalleCorreoDto`, `AdjuntoDto`
- Pruebas unitarias

**Criterios de aceptación:**
- Retorna asunto, cuerpo, fecha, hora del correo y datos de contacto del cliente.
- Adjuntos retornados con nombre de archivo e `IdArchivo` para radio button.
- Si hay adjunto previamente seleccionado en borrador, se indica en el DTO.

**Más información de la tarea:**
Ver sección *"Parte B / B2"* en `TPSC-RE-FU-024-Back.md`. Ver criterios C1, C2, C3 en `TPSC-RE-FU-024.md`.

**Recursos:**
- `TPSC-RE-FU-024-Back.md` — Parte B, sección B2
- `TPSC-RE-FU-024_BD.md` — Tablas leídas (CorreoRecibido, ArchivoCorreoRecibido)

---

## TAREA 8

**[ RE-FU-024 ] [SERV-SIMPLE-PUT] Endpoint y servicio — Auto-guardado del cobro en borrador (Paso 1)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Auto-guardado transparente: sin botón "Guardar" manual.
- Si no existe `fccPagoCliente`: INSERT con `Confirmado=0`, `Folio=NULL`, `IdCatMoneda=@Id`.
- Si existe con `Confirmado=0`: UPDATE con datos actuales incluido `IdCatMoneda`.
- Guardia: si `Confirmado=1`, no sobreescribir.
- Tarea 1 es prerrequisito.
- **OBS-048:** El estado del cobro en borrador (`Confirmado=0`) es la señal que usa el endpoint de estado del wizard (Tarea 12) para determinar que el cliente tiene el Paso 1 en progreso y reanudarlo ahí. El auto-guardado es el mecanismo que persiste ese estado.

**Objetivo general:**
Implementar en Finanzas el endpoint de auto-guardado que persiste el formulario del cobro como borrador de forma transparente, incluyendo la moneda seleccionada (`IdCatMoneda`).

**Objetivos específicos:**
- `PUT /api/validar-cobro/clientes/{idCliente}/cobros/{idFCCFolioPagoCliente}/borrador`
- Crear `AutoGuardarCobroBorradorCommand` + Handler.
- DTO de request: `AutoGuardarCobroBorradorRequestDto` (monto, fechaPago, idCatMedioDePago, cuentaOrdenante, idDatosBancarios, **idCatMoneda**, tipoCambio, notas, idArchivo).
- Guardia de inmutabilidad: no actualizar si `Confirmado=1`.

**Resultado esperado:**
Endpoint `PUT .../borrador` que auto-guarda el cobro sin requerir acción del usuario, preservando todos los campos del formulario incluyendo la moneda seleccionada.

**Entregables:**
- Endpoint + Command + Handler: `AutoGuardarCobroBorradorCommand`
- DTO: `AutoGuardarCobroBorradorRequestDto`
- Pruebas unitarias (incluyendo guardia de inmutabilidad)

**Criterios de aceptación:**
- INSERT correcto si no existe cobro, con `IdCatMoneda` poblado.
- UPDATE correcto si existe con `Confirmado=0`, actualizando `IdCatMoneda`.
- Si `Confirmado=1`, el endpoint retorna sin modificar el registro.

**Más información de la tarea:**
Ver sección *"Parte B / B4"* en `TPSC-RE-FU-024-Back.md`. Ver criterios F1, F2 en `TPSC-RE-FU-024.md`.

**Recursos:**
- `TPSC-RE-FU-024-Back.md` — Parte B, sección B4
- `TPSC-RE-FU-024_BD.md` — Tablas escritas runtime

---

## TAREA 9

**[ RE-FU-024 ] [SERV-TRANSACT] Endpoint y servicio — Confirmación del cobro con folio COB e inmutabilidad**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 México

**Consideraciones previas:**
- Tarea 4 (SEQUENCE) debe existir antes.
- Flujo en Finanzas: (1) validar comprobante seleccionado, (2) validar campos obligatorios (incluido `IdCatMoneda`), (3) calcular folio COB, (4) alerta de confirmación al Front.
- UPDATE en ProquifaDotNet incluye `IdCatMoneda` en el SET del cobro confirmado.
- Guardia `Confirmado=0` en el WHERE del UPDATE.

**Objetivo general:**
Implementar en Finanzas el endpoint transaccional que confirma el cobro: genera folio COB-mmddaa-NNNNNN, persiste `Confirmado=1` con trazabilidad e `IdCatMoneda`, haciendo el cobro inmutable.

**Objetivos específicos:**
- `POST /api/validar-cobro/clientes/{idCliente}/cobros/{idFCCFolioPagoCliente}/confirmar`
- Crear `ConfirmarCobroCommand` + Handler.
- Validar en Finanzas: comprobante seleccionado + `IdCatMoneda` obligatorio + demás campos requeridos.
- Calcular folio con `NEXT VALUE FOR SeqFolioCobro`.
- UPDATE en ProquifaDotNet: `Folio=@Folio, Confirmado=1, FechaConfirmacion=..., IdUsuarioConfirmacion=..., IdCatMoneda=@IdCatMoneda WHERE Confirmado=0`.

**Resultado esperado:**
Endpoint `POST .../confirmar` que confirma el cobro con folio definitivo, moneda registrada e inmutabilidad garantizada.

**Entregables:**
- Endpoint + Command + Handler: `ConfirmarCobroCommand`
- Pruebas unitarias (fallo si comprobante no seleccionado, fallo si `IdCatMoneda` nulo, fallo si ya `Confirmado=1`)

**Criterios de aceptación:**
- `Confirmado=1`, `FechaConfirmacion`, `IdUsuarioConfirmacion`, `Folio` y `IdCatMoneda` poblados.
- Folio en formato `COB-mmddaa-NNNNNN`.
- Error si `IdCatMoneda` es nulo al confirmar.
- Error si el cobro ya tiene `Confirmado=1`.
- El cobro no puede editarse tras la confirmación.

**Más información de la tarea:**
Ver sección *"Parte B / B5"* en `TPSC-RE-FU-024-Back.md`. Ver criterios F3, F4 y regla 12 en `TPSC-RE-FU-024.md`.

**Recursos:**
- `TPSC-RE-FU-024-Back.md` — Parte B, sección B5
- `TPSC-RE-FU-024_BD.md` — Lógica del Folio COB + ciclo de vida

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
Implementar en Finanzas el servicio `TipoCambioMexicoService` que calcula el TC del día para la moneda del cobro seleccionada en el combo (`IdCatMoneda`), según la regla de la moneda vs MXN.

**Objetivos específicos:**
- Crear `ITipoCambioMexicoService.ObtenerTCDelDiaAsync(idCatMonedaCobro, idCatMonedaFacturacion)`.
- Lógica: si ambas son MXN → N/A; si cobro=MXN y facturación≠MXN → TC de la moneda de facturación; si cobro≠MXN → TC de la moneda del cobro.
- `GET /api/validar-cobro/tipo-cambio?idCatMonedaCobro=...&idCatMonedaFacturacion=...`

**Resultado esperado:**
Servicio en Finanzas que retorna el TC del día según la moneda seleccionada en el combo, listo para el formulario del Paso 1 y la confirmación del cobro.

**Entregables:**
- `ITipoCambioMexicoService` + `TipoCambioMexicoService`
- Endpoint `GET /api/validar-cobro/tipo-cambio`
- Pruebas unitarias (3 escenarios: N/A, TC moneda facturación, TC moneda cobro)

**Criterios de aceptación:**
- Si ambas monedas son MXN: retorna `{ "tipoCambio": null, "esNA": true }`.
- Si cobro=MXN y facturación≠MXN: retorna TC de la moneda de facturación vs MXN.
- Si cobro≠MXN: retorna TC de la moneda del cobro vs MXN.
- Valor solo lectura en el formulario.

**Más información de la tarea:**
Ver sección *"Parte B / B6"* en `TPSC-RE-FU-024-Back.md`. Ver regla 7 y criterio D8 en `TPSC-RE-FU-024.md`.

**Recursos:**
- `TPSC-RE-FU-024-Back.md` — Parte B, sección B6
- `TPSC-RE-FU-024.md` — Regla 7, Criterio D8

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
- `GET /api/validar-cobro/inconsistencias/tipos?paso=1` → catálogo filtrado `AplicaPaso='1'`.
- `POST /api/validar-cobro/clientes/{idCliente}/cobros/{idFCCPagoCliente}/inconsistencias` → INSERT en `fccInconsistenciaCobro`.
- Crear `RegistrarInconsistenciaCobroCommand` + Handler.
- Validar que el tipo seleccionado tiene `AplicaPaso='1'`.
- DTO: `RegistrarInconsistenciaRequestDto` (idCatTipoInconsistenciaCobro, comentario).

**Resultado esperado:**
Modal de inconsistencias del Paso 1 con combo filtrado correctamente y registro persistido en `fccInconsistenciaCobro`.

**Entregables:**
- Endpoint `GET /api/validar-cobro/inconsistencias/tipos?paso=1`
- Endpoint `POST .../inconsistencias`
- Command + Handler: `RegistrarInconsistenciaCobroCommand`
- DTOs + pruebas unitarias

**Criterios de aceptación:**
- Combo del modal solo muestra tipos `AplicaPaso='1'`.
- `PAGO_INCOMPLETO_VENCIDO` NO aparece en el Paso 1.
- Inconsistencia insertada correctamente en `fccInconsistenciaCobro`.
- Comentario es opcional (puede ser null).

**Más información de la tarea:**
Ver sección *"Parte B / B7"* en `TPSC-RE-FU-024-Back.md`. Ver criterios G1-G4 y regla 14 en `TPSC-RE-FU-024.md`.

**Recursos:**
- `TPSC-RE-FU-024-Back.md` — Parte B, sección B7
- `TPSC-RE-FU-024_BD.md` — catTipoInconsistenciaCobro + fccInconsistenciaCobro

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
Implementar en Finanzas el endpoint `GET /api/validar-cobro/clientes/{idCliente}/estado-wizard` que retorna el paso activo donde el usuario debe continuar el wizard de Validar Cobro.

**Objetivos específicos:**
- Crear `GetEstadoWizardQuery` + Handler.
- Evaluar: si hay `fccPagoCliente.Confirmado=0` para el cliente → Paso 1 activo. Si no, determinar si hay trabajo pendiente en Paso 2 (lógica extendida en RE-FU-025).
- DTO de respuesta: `EstadoWizardDto` con campos `PasoActivo` (int), `Descripcion` (string), `TieneBorrador` (bool), `IdFCCPagoClienteBorrador` (guid?, solo si aplica).

**Resultado esperado:**
Endpoint `GET .../estado-wizard` que permite a la UI redirigir al usuario directamente al paso activo del wizard, sin forzarlo a comenzar siempre desde el Paso 1.

**Entregables:**
- Endpoint + Query + Handler: `GetEstadoWizardQuery`
- DTO: `EstadoWizardDto`
- Pruebas unitarias (escenarios: sin cobros → Paso 1, borrador activo → Paso 1, todos confirmados → Paso 2)

**Criterios de aceptación:**
- Si no hay `fccPagoCliente` para el cliente, retorna `PasoActivo=1`.
- Si existe `fccPagoCliente.Confirmado=0`, retorna `PasoActivo=1` con `TieneBorrador=true` e `IdFCCPagoClienteBorrador` poblado.
- Si todos los cobros están `Confirmado=1` y hay trabajo pendiente en Paso 2, retorna `PasoActivo=2` (lógica base; se detalla en RE-FU-025).
- La UI no muestra el Paso 1 si el usuario ya completó todos los cobros del Buzón (OBS-048).

**Más información de la tarea:**
Ver sección *"Parte B / B8"* en `TPSC-RE-FU-024-Back.md`. Ver Regla 11 y Criterio F1 en `TPSC-RE-FU-024.md`.

**Recursos:**
- `TPSC-RE-FU-024-Back.md` — Parte B, sección B8
- `TPSC-RE-FU-024.md` — Regla 11, Criterio F1