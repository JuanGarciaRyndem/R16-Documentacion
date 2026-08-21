# Impacto en BD — R16A-RE-FU-023
**Requisito:** Validar Cobro — Pantalla Principal
**Base de Datos:** ProquifaDotNet
**Versión:** 3.0 (tablas ya existentes — CREATE TABLE removidos; todos los ALTER pendientes)

---

## Resumen

Pantalla principal del módulo Validar Cobro. Listado de clientes con pendientes,
filtrado por cartera y región del usuario. Acciones: Realizar Cobros (wizard) o
Gestionar Cobranza (modal con fechas estimadas de pago y cancelación de pedido).
MEX y PER con estructura idéntica.

> ✅ `fccFolioPagoCliente` — ya existe en BD  
> ✅ `fccPagoCliente` — ya existe en BD.
> ❌ ALTER TABLE `fccPagoCliente` — pendiente de ejecutar (`Confirmado`, `FechaConfirmacion`, `IdUsuarioConfirmacion`, `Notas`, `IdCatMoneda`)
> ❌ ALTER TABLE `tpPedido` — pendiente de ejecutar

Ambas tablas son referenciadas por todos los requisitos del módulo (023–029).

---

## Impacto en BD

| #   | Cambio                                                                        | Tipo      | Estado      |
| --- | ----------------------------------------------------------------------------- | --------- | ----------- |
| 1   | ALTER TABLE tpPedido ADD FechaCancelacionPorFaltaPago datetime NULL          | DDL       | ❌ Pendiente |
| 2   | ALTER TABLE tpPedido ADD IdUsuarioCancelacion uniqueidentifier NULL           | DDL       | ❌ Pendiente |
| 3   | ALTER TABLE fccPagoCliente ADD Confirmado bit NOT NULL DEFAULT(0)             | DDL       | ❌ Pendiente |
| 4   | ALTER TABLE fccPagoCliente ADD FechaConfirmacion datetime NULL               | DDL       | ❌ Pendiente |
| 5   | ALTER TABLE fccPagoCliente ADD IdUsuarioConfirmacion uniqueidentifier NULL    | DDL       | ❌ Pendiente |
| 6   | ALTER TABLE fccPagoCliente ADD Notas varchar(500) NULL                        | DDL       | ❌ Pendiente |
| 7   | ALTER TABLE fccPagoCliente ADD IdCatMoneda uniqueidentifier NULL FK catMoneda | DDL       | ❌ Pendiente |
| 8   | FechaEstimadaPago (tpProformaPedido.FechaPromesaPagoMonitoreoCobros)          | Existente | —           |
| 9   | Filtro cartera (ClienteCartera.IdUsuarioCobrador)                             | Existente | —           |
| 10  | ALTER TABLE tpPedido ADD FechaSolicitudCancelacion datetime NULL (OBS-042)   | DDL       | ❌ Pendiente |
| 11  | ALTER TABLE tpPedido ADD EstadoCancelacionCFDI varchar(50) NULL (OBS-042)     | DDL       | ❌ Pendiente |
| 12  | ~~CREATE TABLE fccFechaEstimadaPagoHistorial (OBS-044) — condicional Opción A~~ — Opción A DESCARTADA (DUDA-066, 21/08) | DDL       | ❌ Descartada |
| 13  | ALTER tpProformaPedido ADD FechaEstimadaPagoAnterior + IdUsuarioCambio/FechaCambio (OBS-044) — Opción B CONFIRMADA (DUDA-066, 21/08) | DDL       | ❌ Pendiente |

---

## 1. Diccionario — fccFolioPagoCliente

> ✅ Tabla ya existe en BD. Estructura verificada contra ProquifaDotNet.

**Descripción:** Pendiente en Validar Cobro generado automáticamente al clasificar un correo
como Cobro en el Buzón (Mailbot o manual). Cada fila representa un cobro recibido pendiente
de ser procesado en el wizard. El registro se cierra (`Activo = 0`) al vincular el cobro
confirmado a una proforma/factura o al marcar inconsistencia.

| Columna                    | Tipo             | Nulo | Descripción                                                          |
| -------------------------- | ---------------- | ---- | -------------------------------------------------------------------- |
| `IdFCCFolioPagoCliente`    | uniqueidentifier | NO   | PK — DEFAULT NEWID()                                                 |
| `IdArchivo`                | uniqueidentifier | SÍ   | FK Archivo — archivo adjunto del comprobante                         |
| `IdCorreoRecibidoCliente`  | uniqueidentifier | NO   | FK CorreoRecibidoCliente — correo del Buzón que originó el pendiente |
| `Folio`                    | varchar(80)      | SÍ   | Folio de referencia interna                                          |
| `Consecutivo`              | int              | SÍ   | Consecutivo del folio                                                |
| `FechaRecepcion`           | datetime         | NO   | Fecha de recepción del correo origen                                 |
| `Stp`                      | bit              | NO   | Indica si es cobro vía STP — DEFAULT (0)                             |
| `FechaRegistro`            | datetime         | NO   | Auditoría: cuándo se creó el pendiente — DEFAULT GETDATE()           |
| `FechaUltimaActualizacion` | datetime         | NO   | Auditoría: cuándo se modificó por última vez — DEFAULT GETDATE()     |
| `Activo`                   | bit              | NO   | 1 = abierto / **0 = cerrado** — DEFAULT (1)                          |
| `SubtotalMailBot`          | decimal(18,6)    | SÍ   | Subtotal pre-extraído por Mailbot                                    |
| `IvaMailBot`               | decimal(18,6)    | SÍ   | IVA pre-extraído por Mailbot                                         |
| `TotalMailBot`             | decimal(18,6)    | SÍ   | Total pre-extraído por Mailbot                                       |
| `MxnMailBot`               | bit              | SÍ   | Moneda MXN detectada por Mailbot                                     |
| `UsdMailBot`               | bit              | SÍ   | Moneda USD detectada por Mailbot                                     |

**Relaciones**

| Tabla relacionada          | Columna FK                  | Tipo de relación |
|----------------------------|-----------------------------|------------------|
| `CorreoRecibidoCliente`    | `IdCorreoRecibidoCliente`   | N:1              |
| `Archivo`                  | `IdArchivo`                 | N:1 (nullable)   |

**Índices**

| Nombre                  | Columnas               | Tipo          |
|-------------------------|------------------------|---------------|
| `PK_fccFolioPagoCliente`| `IdFCCFolioPagoCliente`| PRIMARY KEY   |

**Consideraciones especiales**

- `Activo = 0` cierra el pendiente: al confirmar cobro (vinculado a proforma/factura) o al registrar inconsistencia
- El INSERT lo realiza `CorreoRecibidoClienteToPagoBO.cs` (Mailbot, RE-FU-008) — no duplicar en Finanzas
- `SubtotalMailBot`, `IvaMailBot`, `TotalMailBot`, `MxnMailBot`, `UsdMailBot` pre-cargan datos en el wizard desde extracción automática del Mailbot

---

## 2. Diccionario — fccPagoCliente

> ✅ Tabla ya existe en BD. Estructura completa verificada contra ProquifaDotNet.
> Campos adicionales presentes en BD: `IdContactoCliente`, `InformacionComplementoPago`, `Broker`, `IdCatBrokerCliente`, `IdCFDI`, `Serie`.
> Los campos `Confirmado`, `FechaConfirmacion`, `IdUsuarioConfirmacion`, `Notas` e `IdCatMoneda` se documentan y agregan vía ALTER en este requisito — consumidos funcionalmente en RE-FU-024.

**Descripción:** Almacena los datos completos del cobro capturados en el wizard de Validar Cobro
(Paso 1). Cada fila representa un cobro que un cliente realizó. Tabla central del módulo
Validar Cobro — leída y extendida por RE-FU-024 a RE-FU-029.

| Columna                      | Tipo                    | Nulo | Descripción                                                                      |
| ---------------------------- | ----------------------- | ---- | -------------------------------------------------------------------------------- |
| `IdFCCPagoCliente`           | uniqueidentifier        | NO   | PK — DEFAULT NEWID()                                                             |
| `IdCliente`                  | uniqueidentifier        | NO   | FK Cliente                                                                       |
| `IdEmpresa`                  | uniqueidentifier        | NO   | FK Empresa PROQUIFA que recibe el cobro                                          |
| `IdContactoCliente`          | uniqueidentifier        | SÍ   | FK ContactoCliente — contacto del cliente asociado al cobro                      |
| `MXN`                        | bit                     | NO   | Flag legacy: cobro en pesos — DEFAULT (0)                                        |
| `USD`                        | bit                     | NO   | Flag legacy: cobro en dólares — DEFAULT (1)                                      |
| `TipoDeCambio`               | decimal(18,6)           | SÍ   | TC del día vs MXN — DEFAULT (0); calculado automáticamente en Paso 1             |
| `Monto`                      | decimal(18,6)           | NO   | Monto recibido del cliente                                                       |
| `FechaPago`                  | datetime                | SÍ   | Fecha efectiva del pago                                                          |
| `IdCatMedioDePago`           | uniqueidentifier        | SÍ   | FK catMedioDePago — forma de pago (c_FormaPago SAT vía ClaveFormaDePago)         |
| `IdDatosBancarios`           | uniqueidentifier        | SÍ   | FK DatosBancarios — cuenta bancaria PROQUIFA destino                             |
| `InformacionComplementoPago` | bit                     | NO   | Indica si requiere complemento de pago — DEFAULT (0)                             |
| `IdCatBanco`                 | uniqueidentifier        | SÍ   | FK catBanco — banco emisor del cliente                                           |
| `CuentaOrdenante`            | varchar(80)             | SÍ   | Cuenta del cliente emisor                                                        |
| `ReferenciaBancaria`         | varchar(80)             | SÍ   | Referencia bancaria del cobro                                                    |
| `Broker`                     | bit                     | NO   | Indica si el cobro es vía Broker — DEFAULT (0)                                   |
| `IdCatBrokerCliente`         | uniqueidentifier        | SÍ   | FK catBroker — broker del cliente (cuando aplica)                                |
| `FechaRegistro`              | datetime                | NO   | Auditoría: cuándo se creó el registro — DEFAULT GETDATE()                        |
| `FechaUltimaActualizacion`   | datetime                | NO   | Auditoría: cuándo se modificó por última vez — DEFAULT GETDATE()                 |
| `Activo`                     | bit                     | NO   | 1 = activo / **0 = inconsistencia**: cierra el pendiente del Buzón — DEFAULT (1) |
| `IdCFDI`                     | uniqueidentifier        | SÍ   | FK CFDI — documento fiscal asociado al cobro                                     |
| `Folio`                      | varchar(80)             | SÍ   | Folio COB-mmddaa-NNNNNN — NULL en borrador; se llena al confirmar (RE-FU-024)    |
| `Serie`                      | varchar(80)             | SÍ   | Serie del comprobante fiscal                                                     |
| `IdArchivo`                  | uniqueidentifier        | SÍ   | FK Archivo — comprobante de pago seleccionado del correo                         |
| **`IdFCCFolioPagoCliente`**      | **uniqueidentifier**        | **SÍ**   | **FK fccFolioPagoCliente — pendiente del Buzón que originó el cobro**                |
| `Confirmado`                | bit NOT NULL DEFAULT(0) | NO   | 0 = borrador / 1 = confirmado e inmutable — ❌ Pendiente ALTER          |
| `FechaConfirmacion`         | datetime               | SÍ   | Timestamp de confirmación del cobro — ❌ Pendiente ALTER                |
| `IdUsuarioConfirmacion`     | uniqueidentifier        | SÍ   | Quién confirmó el cobro (trazabilidad) — ❌ Pendiente ALTER             |
| `Notas`                     | varchar(500)            | SÍ   | Notas opcionales del formulario del cobro — ❌ Pendiente ALTER          |
| `IdCatMoneda`               | uniqueidentifier        | SÍ   | FK catMoneda — moneda del cobro (combo UI Paso 1) — ❌ Pendiente ALTER  |

**Relaciones**

| Tabla relacionada          | Columna FK                  | Tipo de relación |
|----------------------------|-----------------------------|------------------|
| `Cliente`                  | `IdCliente`                 | N:1              |
| `Empresa`                  | `IdEmpresa`                 | N:1              |
| `ContactoCliente`          | `IdContactoCliente`         | N:1 (nullable)   |
| `catMedioDePago`           | `IdCatMedioDePago`          | N:1 (nullable)   |
| `DatosBancarios`           | `IdDatosBancarios`          | N:1 (nullable)   |
| `catBanco`                 | `IdCatBanco`                | N:1 (nullable)   |
| `catBroker`                | `IdCatBrokerCliente`        | N:1 (nullable)   |
| `CFDI`                     | `IdCFDI`                    | N:1 (nullable)   |
| `Archivo`                  | `IdArchivo`                 | N:1 (nullable)   |
| `fccFolioPagoCliente`      | `IdFCCFolioPagoCliente`     | N:1 (nullable)   |
| `catMoneda`                 | `IdCatMoneda`               | N:1 (nullable)   |


**Índices**

| Nombre               | Columnas            | Tipo        |
|----------------------|---------------------|-------------|
| `PK_fccPagoCliente`  | `IdFCCPagoCliente`  | PRIMARY KEY |

**Consideraciones especiales**

- `Confirmado = 1` hace el registro inmutable — guardia en el endpoint de auto-guardado (RE-FU-024)
- Los flags `MXN`/`USD` se mantienen por compatibilidad legacy; `IdCatMoneda` es el campo oficial para el combo UI y soporte de PEN (Perú)
- `Folio` = NULL en borrador; se genera formato `COB-mmddaa-NNNNNN` al confirmar mediante `SeqFolioCobro` (RE-FU-024)
- `Activo = 0` + INSERT en `fccInconsistenciaCobro` al registrar inconsistencia (RE-FU-024)

---



## 3. ALTER TABLE fccPagoCliente — Campos de Confirmación y Moneda

> ❌ PENDIENTE de ejecutar. Ninguno de los 5 campos existe aún en `fccPagoCliente` (verificado).
> Documentados en RE-FU-023 para que estén listos antes de RE-FU-024.

**Propósito:** Extender `fccPagoCliente` con los campos necesarios para soportar:
inmutabilidad del cobro confirmado, trazabilidad de confirmación, notas del formulario
y moneda del cobro como FK a `catMoneda` para el combo desplegable del Paso 1 (RE-FU-024).

```sql
-- ❌ PENDIENTE — revisar objetos dependientes antes de ejecutar en ProquifaDotNet

-- 1. Inmutabilidad: 0 = borrador / 1 = confirmado (cobro inmutable tras confirmación)
ALTER TABLE dbo.fccPagoCliente
    ADD Confirmado bit NOT NULL
        CONSTRAINT [DF_fccPagoCliente_Confirmado] DEFAULT (0);

-- 2. Timestamp de confirmación
ALTER TABLE dbo.fccPagoCliente
    ADD FechaConfirmacion datetime NULL;

-- 3. Trazabilidad: quién confirmó el cobro
ALTER TABLE dbo.fccPagoCliente
    ADD IdUsuarioConfirmacion uniqueidentifier NULL;

-- 4. Notas opcionales del formulario
ALTER TABLE dbo.fccPagoCliente
    ADD Notas varchar(500) NULL;

-- 5. Moneda del cobro como FK a catMoneda (combo UI Paso 1)
ALTER TABLE dbo.fccPagoCliente
    ADD IdCatMoneda uniqueidentifier NULL
        CONSTRAINT [FK_fccPagoCliente_CatMoneda]
            FOREIGN KEY REFERENCES dbo.catMoneda([IdCatMoneda]);
```

**Diccionario de datos — campos nuevos en fccPagoCliente**

| Nombre de tabla  | Descripción                                                |
|------------------|------------------------------------------------------------|
| `fccPagoCliente` | Registro del cobro capturado en el wizard de Validar Cobro |

| Columna                 | Tipo                    | Nulo | Descripción                                                      |
|-------------------------|-------------------------|------|------------------------------------------------------------------|
| `Confirmado`            | bit NOT NULL DEFAULT(0) | NO   | 0 = borrador / 1 = confirmado e inmutable                        |
| `FechaConfirmacion`     | datetime               | SÍ   | Timestamp de confirmación del cobro                              |
| `IdUsuarioConfirmacion` | uniqueidentifier        | SÍ   | ID del usuario que confirmó el cobro (trazabilidad de auditoría) |
| `Notas`                 | varchar(500)            | SÍ   | Notas opcionales del formulario del cobro                        |
| `IdCatMoneda`           | uniqueidentifier        | SÍ   | FK catMoneda — moneda del cobro (combo UI Paso 1)                |

**Relaciones**

| Tabla relacionada | Columna FK    | Tipo de relación |
|-------------------|---------------|------------------|
| `catMoneda`       | `IdCatMoneda` | N:1 (nullable)   |

**Índices**

| Observación                                                                         |
|-------------------------------------------------------------------------------------|
| Sin índice adicional requerido — campos de auditoría y moneda consultados por cobro |

**Consideraciones especiales**

- `Confirmado DEFAULT(0)` — no rompe registros existentes
- `Confirmado = 1` hace el registro inmutable: el endpoint de auto-guardado (RE-FU-024) valida esta guardia antes de actualizar
- Los flags `MXN`/`USD` existentes se mantienen por compatibilidad legacy; `IdCatMoneda` es el campo oficial para el combo UI y soporte de PEN (Perú)
- `FechaConfirmacion` e `IdUsuarioConfirmacion` se populan al confirmar el cobro en RE-FU-024
- Verificar SPs, vistas y triggers dependientes de `fccPagoCliente` antes de ejecutar

**Script de validación** *(ejecutar después del ALTER)*

```sql
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'fccPagoCliente'
  AND COLUMN_NAME IN ('Confirmado','FechaConfirmacion','IdUsuarioConfirmacion','Notas','IdCatMoneda');
-- Resultado esperado: 5 filas
```

---
## 4. ALTER TABLE tpPedido — Trazabilidad de Cancelación

> ❌ PENDIENTE de ejecutar. Campos `FechaCancelacionPorFaltaPago` e `IdUsuarioCancelacion` verificados: NO existen aún en `tpPedido`.

**Propósito:** Registrar quién y cuándo ejecutó la cancelación de un pedido por falta de
pago desde el modal Gestionar Cobranza de la pantalla principal.

```sql
-- ❌ PENDIENTE — revisar objetos dependientes antes de ejecutar en ProquifaDotNet
ALTER TABLE dbo.tpPedido
    ADD FechaCancelacionPorFaltaPago datetime NULL;

ALTER TABLE dbo.tpPedido
    ADD IdUsuarioCancelacion uniqueidentifier NULL;
```

**Diccionario de datos — campos nuevos en tpPedido**

| Nombre de tabla | Descripción |
|-----------------|-------------|
| `tpPedido`      | Pedido generado en el flujo de venta interna de ProquifaDotNet |

| Columna                         | Tipo             | Nulo | Descripción                                                                              |
|---------------------------------|------------------|------|------------------------------------------------------------------------------------------|
| `FechaCancelacionPorFaltaPago`  | datetime        | SÍ   | Fecha y hora en que se ejecutó la cancelación del pedido por falta de pago               |
| `IdUsuarioCancelacion`          | uniqueidentifier | SÍ   | ID del usuario autenticado en Finanzas que ejecutó la cancelación (trazabilidad auditoría)|

**Relaciones**

| Columna                        | Relación                                                                         |
|--------------------------------|----------------------------------------------------------------------------------|
| `IdUsuarioCancelacion`         | Sin FK definida en BD — referencia lógica al usuario autenticado vía IdentityServer |

**Índices**

| Observación                                                              |
|--------------------------------------------------------------------------|
| Sin índice adicional requerido — campos de auditoría consultados por pedido |

**Consideraciones especiales**

- Ambos campos son `NULL` para no romper registros existentes ni objetos dependientes actuales
- Solo se populan al ejecutar "Cancelar Pedido" desde el modal Gestionar Cobranza (RE-FU-023 Tarea 10)
- `tpProformaPedido.Cancelada = 1` al cancelar (ya existe — no requiere cambios)
- Verificar SPs, vistas y triggers dependientes de `tpPedido` antes de ejecutar

**Script de validación** *(ejecutar después del ALTER)*

```sql
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'tpPedido'
  AND COLUMN_NAME IN ('FechaCancelacionPorFaltaPago', 'IdUsuarioCancelacion');
-- Resultado esperado: 2 filas
```

---

## 5. ALTER TABLE tpPedido — Trazabilidad Cancelación CFDI (OBS-042)

> ❌ PENDIENTE de ejecutar. Campos `FechaSolicitudCancelacion` y `EstadoCancelacionCFDI` NO existen aún en `tpPedido`.

**Propósito:** Registrar la fecha en que se solicitó la cancelación del CFDI ante el SAT y el estado actual de esa cancelación, para permitir trazabilidad completa del proceso de cancelación fiscal del pedido.

```sql
-- ❌ PENDIENTE — revisar objetos dependientes antes de ejecutar en ProquifaDotNet
-- OBS-042: trazabilidad de cancelación CFDI

ALTER TABLE dbo.tpPedido
    ADD FechaSolicitudCancelacion datetime NULL;

ALTER TABLE dbo.tpPedido
    ADD EstadoCancelacionCFDI varchar(50) NULL;
```

**Diccionario de datos — campos nuevos en tpPedido (OBS-042)**

| Columna                     | Tipo         | Nulo | Descripción                                                                               |
|-----------------------------|--------------|------|-------------------------------------------------------------------------------------------|
| `FechaSolicitudCancelacion` | datetime    | SÍ   | Fecha y hora en que se envió la solicitud de cancelación del CFDI al SAT                  |
| `EstadoCancelacionCFDI`     | varchar(50)  | SÍ   | Estado de la cancelación devuelto por el SAT / PAC (ej. "Pendiente", "Cancelado", "Rechazado") |

**Consideraciones especiales**

- Ambos campos NULL — no rompen registros existentes.
- Se populan al ejecutar la cancelación del CFDI desde el módulo de Gestionar Cobranza (Tarea 9 de RE-FU-023).
- `EstadoCancelacionCFDI` se actualiza si el SAT devuelve un estado asíncrono (cancelación en dos pasos CFDI 4.0).
- Complementa `FechaCancelacionPorFaltaPago` + `IdUsuarioCancelacion` — son conceptos distintos: cancelación del pedido vs. cancelación del CFDI ante el SAT.
- **Trazabilidad (DUDA-121, 21/08):** los reintentos de cancelación fiscal ante rechazo del receptor quedan FUERA del sistema (gestión manual, aceptado) — no se implementa una pantalla de reintentos para este flujo; `EstadoCancelacionCFDI` solo refleja el último estado recibido del SAT.

---

## 6. ~~CREATE TABLE fccFechaEstimadaPagoHistorial (OBS-044) — CONDICIONAL~~ — Opción A DESCARTADA (DUDA-066)

> ❌ **DESCARTADA — Resuelto por DUDA-066 (21/08/2026).** Ya NO aplica el CREATE TABLE de esta sección (Opción A / bitácora completa). Se conserva el contenido a continuación únicamente como referencia histórica de la alternativa evaluada — NO ejecutar.
>
> **Resolución final (DUDA-066):** se conservan en BD DOS valores — el actual (vigente) y el inmediatamente anterior, con autor y fecha (Opción B, ver abajo). Ante una nueva modificación, el "actual" pasa a "anterior" y el anterior previo se sobrescribe (NO es bitácora completa). Solo registra cambios hechos desde el sistema; NO requiere vista en pantalla dedicada (se considera parte de la bitácora de movimientos general del sistema).
>
> ~~**Contexto:** El cliente confirmó (10/07) que el histórico de FechaEstimadaPago está dentro del alcance R16 (Riesgo 1 eliminado). Sin embargo queda **sub-duda viva** de presentación/desempeño:~~
> ~~- **Opción A — Bitácora completa (append-only):** esta tabla `fccFechaEstimadaPagoHistorial`. Cada cambio genera un INSERT. Permite reporte histórico completo y visualización en pantalla.~~
> - **Opción B — Solo el último cambio — CONFIRMADA (DUDA-066, 21/08):** NO se crea esta tabla. En su lugar, se agregan campos a `tpProformaPedido` (`FechaEstimadaPagoAnterior datetime NULL`, `IdUsuarioCambioFechaEstimada uniqueidentifier NULL`, `FechaCambioFechaEstimada datetime NULL`) — al modificar, el valor "actual" pasa a "anterior" y el anterior previo se sobrescribe. Misma mecánica que OBS-014 (FU-006 Código Validador). No requiere vista en pantalla; solo bitácora general del sistema.
>
> ~~**Recomendación técnica:** Opción B (menor costo de mantenimiento y desempeño; alineada a patrón OBS-014 ya aceptado).~~ — Confirmada como decisión final por DUDA-066.
>
> El script CREATE TABLE de abajo **NO se ejecuta** (Opción A descartada). Se deja documentado solo como referencia.

**Propósito (Opción A):** Guardar el historial completo de cambios de `FechaPromesaPagoMonitoreoCobros` (FechaEstimadaPago) por proforma/pedido. Cada cambio genera una fila nueva — no se sobrescribe el valor anterior.

```sql
-- ❌ PENDIENTE — crear en ProquifaDotNet
-- OBS-044: historial de FechaEstimadaPago

CREATE TABLE dbo.fccFechaEstimadaPagoHistorial (
    IdFccFechaEstimadaPagoHistorial uniqueidentifier NOT NULL
        CONSTRAINT PK_fccFechaEstimadaPagoHistorial PRIMARY KEY
        CONSTRAINT DF_fccFechaEstimadaPagoHistorial_Id DEFAULT (NEWID()),
    IdTpProformaPedido             uniqueidentifier NOT NULL
        CONSTRAINT FK_fccFechaEstimadaPagoHistorial_ProformaPedido
            FOREIGN KEY REFERENCES dbo.tpProformaPedido(IdTpProformaPedido),
    FechaEstimadaPagoAnterior      datetime        NULL,
    FechaEstimadaPagaNueva         datetime        NULL,
    FechaCambio                    datetime        NOT NULL
        CONSTRAINT DF_fccFechaEstimadaPagoHistorial_FechaCambio DEFAULT (GETDATE()),
    IdUsuarioCambio                uniqueidentifier NOT NULL,
    Motivo                         varchar(300)     NULL
);
```

**Diccionario de datos**

| Nombre de tabla                    | Descripción                                                              |
|------------------------------------|--------------------------------------------------------------------------|
| `fccFechaEstimadaPagoHistorial`    | Historial completo de cambios de la fecha estimada de pago por proforma  |

| Columna                               | Tipo             | Nulo | Descripción                                                             |
|---------------------------------------|------------------|------|-------------------------------------------------------------------------|
| `IdFccFechaEstimadaPagoHistorial`     | uniqueidentifier | NO   | PK — DEFAULT NEWID()                                                    |
| `IdTpProformaPedido`                  | uniqueidentifier | NO   | FK tpProformaPedido — proforma cuya fecha cambió                        |
| `FechaEstimadaPagoAnterior`           | datetime        | SÍ   | Valor previo de FechaPromesaPagoMonitoreoCobros (NULL si era primer valor) |
| `FechaEstimadaPagaNueva`              | datetime        | SÍ   | Nuevo valor asignado                                                    |
| `FechaCambio`                         | datetime        | NO   | Timestamp UTC del cambio — DEFAULT GETDATE()                     |
| `IdUsuarioCambio`                     | uniqueidentifier | NO   | ID del usuario que realizó el cambio (trazabilidad)                     |
| `Motivo`                              | varchar(300)     | SÍ   | Justificación opcional del cambio                                       |

**Relaciones**

| Tabla relacionada  | Columna FK              | Tipo de relación |
|--------------------|-------------------------|------------------|
| `tpProformaPedido` | `IdTpProformaPedido`    | N:1              |

**Índices**

| Nombre                                            | Columnas                            | Tipo       |
|---------------------------------------------------|-------------------------------------|------------|
| `PK_fccFechaEstimadaPagoHistorial`                | `IdFccFechaEstimadaPagoHistorial`   | PRIMARY KEY|
| `IX_fccFechaEstimadaPagoHistorial_ProformaPedido` | `IdTpProformaPedido, FechaCambio`   | NONCLUSTERED (DESC) |

**Consideraciones especiales**

- Cada UPDATE a `tpProformaPedido.FechaPromesaPagoMonitoreoCobros` genera un INSERT en esta tabla (no se actualiza la fila existente).
- `FechaEstimadaPagoAnterior = NULL` es válido para el primer registro (cuando no había fecha previa).
- La tabla es append-only (solo INSERT, nunca UPDATE ni DELETE).

---

## Ciclo de vida del flujo Validar Cobro

```
CorreoRecibidoCliente (clasificado como 'cobro' por Mailbot)
    → INSERT fccFolioPagoCliente (pendiente abierto, Activo = 1)
        → Pantalla principal RE-FU-023: COUNT pendientes → AccionContextual = REALIZAR_COBROS
            → INSERT fccPagoCliente (borrador Paso 1, Confirmado = 0, Activo = 1)
                → RE-FU-024: confirmación cobro (Folio generado, Confirmado = 1)
                    → RE-FU-026: asociación cobro↔documento (Paso 2)
                        → RE-FU-028: emisión documento fiscal (Paso 3)
                            → fccFolioPagoCliente.Activo = 0 (pendiente cerrado)
```

---

## Cadena de Datos — Conteo de Cobros Recibidos

```
fccFolioPagoCliente (cobros recibidos del Buzón)
    → IdCorreoRecibidoCliente → CorreoRecibidoCliente
        → IdCliente (vínculo con el cliente)
    Activo = 1 → pendiente de aplicar

COUNT(fccFolioPagoCliente.Activo = 1) por cliente = CobrosRecibidosPendientes
```

---

## Cadena de Datos — Saldo Pendiente

```
tpProformaPedido.MontoPendiente > 0 AND Cancelada = 0 AND Activo = 1
SUM por cliente = SaldoPendienteTotal

Vinculación al cliente:
tpProformaPedido.IdCliente → Cliente
```

---

## Tablas Consultadas (Lectura)

| Tabla                     | Rol en la pantalla                                            |
|---------------------------|---------------------------------------------------------------|
| `fccFolioPagoCliente`     | Conteo de cobros recibidos pendientes por cliente             |
| `fccPagoCliente`          | Verificar si el cliente ya tiene cobros en proceso (evitar duplicados al iniciar wizard) |
| `tpPedido`                | FolioPedidoInterno, NumeroOrdenDeCompra                       |
| `tpProformaPedido`        | MontoPendiente, FechaPromesaPagoMonitoreoCobros, Cancelada    |
| `tpPedidoProformaPedido`  | Vínculo pedido–proforma                                       |
| `Cliente`                 | Nombre del cliente                                            |
| `DatosFacturacionCliente` | RFC (MEX) / RUC (PER)                                         |
| `ContactoCliente`         | Nombre, Correo, Teléfono para modal Gestionar Cobranza        |
| `CorreoRecibidoCliente`   | Vínculo cobro → cliente                                       |
| `ClienteCarteraCliente`   | Vínculo cliente–cartera                                       |
| `ClienteCartera`          | IdUsuarioCobrador (filtro usuario) + IdRegion (filtro región) |
| `Region`                  | MEX / PER                                                     |
| `catMoneda`               | Moneda del saldo pendiente y del cobro (IdCatMoneda)          |

---

## Tablas Escritas (runtime)

| Tabla              | Momento                     | Operación                                                  |
|--------------------|-----------------------------|------------------------------------------------------------|
| `tpProformaPedido` | Al confirmar fecha en modal | UPDATE FechaPromesaPagoMonitoreoCobros (OBS-044: + INSERT historial) |
| `tpProformaPedido` | Al cancelar pedido          | UPDATE Cancelada = 1                                       |
| `tpPedido`         | Al cancelar pedido          | UPDATE FechaCancelacionPorFaltaPago, IdUsuarioCancelacion  |
| `tpPedido`         | Al cancelar CFDI            | UPDATE FechaSolicitudCancelacion, EstadoCancelacionCFDI (OBS-042) |
| ~~`fccFechaEstimadaPagoHistorial`~~ | ~~Al cambiar fecha estimada pago~~ | ~~INSERT nueva fila (OBS-044)~~ — descartado (DUDA-066): se sustituye por UPDATE de `tpProformaPedido.FechaEstimadaPagoAnterior`/`IdUsuarioCambioFechaEstimada`/`FechaCambioFechaEstimada` (Opción B, sección 6) |

---

## Lógica de Acción Contextual

| Condición | Acción |
|-----------|--------|
| COUNT(fccFolioPagoCliente pendientes) > 0 | `REALIZAR_COBROS` → abre wizard Validar Cobro |
| COUNT = 0 AND SUM(MontoPendiente) > 0 | `GESTIONAR_COBRANZA` → abre modal |

---

## Modal Gestionar Cobranza — Datos por Pedido

| Campo                 | Tabla fuente       | Campo BD                        |
|-----------------------|--------------------|----------------------------------|
| Pedido Interno        | `tpPedido`         | FolioPedidoInterno              |
| Ref. del Cliente      | `tpProformaPedido` | NumeroOrdenDeCompra             |
| Contacto              | `ContactoCliente`  | Nombre, Correo, Teléfono        |
| Fecha Estimada Pago   | `tpProformaPedido` | FechaPromesaPagoMonitoreoCobros |
| Monto Total Pendiente | `tpProformaPedido` | MontoPendiente                  |

---

## Gaps y Pendientes

| # | Gap | Tipo | Acción |
|---|-----|------|--------|
| 1 | Saldo Pendiente: dolarizado vs moneda cliente | Negocio | ✅ Resuelta OBS-046 (10/07) — trazabilidad DUDA-067 (21/08): siempre en USD. **TC por documento origen** (proforma/factura), NO TC del día ni unificado. Sumar montos ya dolarizados. Usar ConversorDivisas existente. **Pendiente operativo no bloqueante:** documentar TC del proceso (proforma, cobro, factura) y su trazabilidad por documento. |
| 2 | Orden del listado por defecto | Negocio | ✅ Resuelta OBS-047: ordenar por antigüedad del cobro recibido más antiguo (MIN fccFolioPagoCliente.FechaRecepcion, ASC). Clientes sin cobros al final. SLA 72h como indicador visual. |
| 3 | Historial de cambios FechaPromesaPago | Negocio | ✅ Resuelta OBS-044 (10/07), sub-duda CERRADA por DUDA-066 (21/08): **Opción B confirmada** — solo se conservan 2 valores en BD (actual + anterior, con usuario y fecha), NO bitácora completa; sin vista en pantalla dedicada. Misma naturaleza que OBS-014 (FU-006 Código Validador). Diseño de sección 6 (Opción A, append-only) queda descartado. |
| 4 | Cancelación propaga a proforma/factura | Negocio | ✅ Resuelta OBS-042: se agregan FechaSolicitudCancelacion + EstadoCancelacionCFDI en tpPedido para trazabilidad. Ver sección 5. |
| 5 | ~~Buzón de Cobros Perú sin datos~~ | Operativo | ✅ Resuelta DUDA-069 (21/08): las brechas de referencias de pago y modelo de cuentas Perú ya están resueltas (ver DUDA-001, DUDA-018) — ya no bloquean que lleguen cobros de Perú al listado de Validar Cobro. |
| 6 | Rol: Gestor Cobranza vs Analista CxC | Negocio | Confirmar denominación |

---

## Dependencias

| Requisito      | Relación |
|----------------|----------|
| R16A-RE-FU-008 | Buzón de Cobros — `CorreoRecibidoCliente`, clasificación 'cobro' Mailbot |
| R16A-RE-FU-016 | Proforma MEX (`tpProformaPedido.MontoPendiente`) |
| R16A-RE-FU-019 | FAA MEX genera pendientes en Validar Cobro (Prepago) |
| R16A-RE-FU-020 | FAA PER genera pendientes en Validar Cobro |
| R16A-RE-FU-024 | Confirmación cobro, folio COB, inconsistencias — reutiliza `fccPagoCliente` con campos ya incorporados en RE-FU-023 |
| R16A-RE-FU-026 | Lee `fccPagoCliente` + `fccFolioPagoCliente` para Paso 2 Asociación |

---

**Generado por:** GitHub Copilot
**Base de Datos:** ProquifaDotNet