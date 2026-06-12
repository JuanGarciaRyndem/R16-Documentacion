# Impacto en BD — R16A-RE-FU-024
**Requisito:** Validar Cobro: Paso 1 México — Captura del Cobro
**Base de Datos:** ProquifaDotNet
**Versión:** 1.1 (actualizado: se agrega IdCatMoneda detectado en pantalla)

---

## Resumen
Paso 1 del wizard de Validar Cobro: Captura del Cobro para Región México.
El cobro se registra en `fccPagoCliente` (tabla existente).
Se extiende la tabla con 5 campos nuevos (inmutabilidad + notas + moneda como FK).
Se crean 2 tablas nuevas (inconsistencias) y 1 SEQUENCE (foliador COB).

> **✅ Los 5 campos de inmutabilidad+moneda ya fueron agregados en RE-FU-023 (ejecutados en BD).** La Tarea 1 de este requisito queda como referencia documental — no requiere ejecución.
>
> `fccPagoCliente` ya existe con: `Folio`, `Monto`, `FechaPago`, `TipoDeCambio`,
> `IdCatMedioDePago` (c_FormaPago SAT via ClaveFormaDePago), `CuentaOrdenante`,
> `IdDatosBancarios` (cuenta destino), `IdFCCFolioPagoCliente` (vínculo con Buzón),
> `IdCatBanco`, `ReferenciaBancaria`, `IdArchivo` (comprobante del correo),
> `MXN`/`USD` (flags bit — se complementa con `IdCatMoneda FK catMoneda` para el combo de la pantalla),
> `FechaRegistro`, `FechaUltimaActualizacion` (auditoría).
> Tabla creada en RE-FU-023 — los ALTERs de este requisito la extienden.
> `catMedioDePago` ya mapea a c_FormaPago SAT via campo `ClaveFormaDePago`.

---

## Impacto en BD

| #   | Cambio                                                                        | Tipo | Estado                   |
| --- | ----------------------------------------------------------------------------- | ---- | ------------------------ |
| 1   | ALTER TABLE fccPagoCliente ADD Confirmado bit NOT NULL DEFAULT(0)             | DDL  | ✅ Ejecutado en RE-FU-023 |
| 2   | ALTER TABLE fccPagoCliente ADD FechaConfirmacion datetime2 NULL               | DDL  | ✅ Ejecutado en RE-FU-023 |
| 3   | ALTER TABLE fccPagoCliente ADD IdUsuarioConfirmacion uniqueidentifier NULL    | DDL  | ✅ Ejecutado en RE-FU-023 |
| 4   | ALTER TABLE fccPagoCliente ADD Notas varchar(500) NULL                        | DDL  | ✅ Ejecutado en RE-FU-023 |
| 5   | ALTER TABLE fccPagoCliente ADD IdCatMoneda uniqueidentifier NULL FK catMoneda | DDL  | ✅ Ejecutado en RE-FU-023 |
| 6   | CREATE TABLE catTipoInconsistenciaCobro                                       | DDL  | ❌ Pendiente              |
| 7   | CREATE TABLE fccInconsistenciaCobro                                           | DDL  | ❌ Pendiente              |
| 8   | CREATE SEQUENCE dbo.SeqFolioCobro                                             | DDL  | ❌ Pendiente              |

> **Nota #5 — IdCatMoneda:** La pantalla del Paso 1 (Captura del Cobro) muestra un combo
> desplegable de Moneda (ej. "USD"). Esto requiere un FK a `catMoneda` para cargar las opciones.
> Los flags `MXN`/`USD` existentes solo soportan 2 monedas y no sirven para PEN (Perú) ni combos.
> Se agrega `IdCatMoneda` nullable para no romper registros existentes; los flags se mantienen
> por compatibilidad con código legacy hasta que se unifique.

---

## Tabla: fccPagoCliente — Estructura completa post-R16

| Columna                 | Tipo                    | Nulo | Descripción                                              | Estado              |
| ----------------------- | ----------------------- | ---- | -------------------------------------------------------- | ------------------- |
| `IdFCCPagoCliente`      | uniqueidentifier        | NO   | PK                                                       | Existente           |
| `IdCliente`             | uniqueidentifier        | NO   | FK Cliente                                               | Existente           |
| `IdEmpresa`             | uniqueidentifier        | NO   | FK Empresa que recibe el cobro                           | Existente           |
| `IdFCCFolioPagoCliente` | uniqueidentifier        | SÍ   | FK fccFolioPagoCliente — vínculo correo Buzón            | Existente           |
| `Folio`                 | varchar(80)             | SÍ   | Formato COB-mmddaa-NNNN al confirmar; NULL en borrador   | Existente           |
| `Monto`                  | decimal(18,4)           | NO   | Monto recibido del cliente                               | Existente           |
| `FechaPago`             | datetime                | SÍ   | Fecha efectiva del pago                                  | Existente           |
| `TipoDeCambio`          | decimal                 | SÍ   | TC del día vs MXN (calculado automático, solo lectura)   | Existente           |
| `MXN`                   | bit                     | NO   | Bandera moneda pesos (legacy)                            | Existente           |
| `USD`                   | bit                     | NO   | Bandera moneda dólares (legacy)                          | Existente           |
| `IdCatMedioDePago`      | uniqueidentifier        | SÍ   | FK catMedioDePago — forma de pago c_FormaPago SAT        | Existente           |
| `IdDatosBancarios`      | uniqueidentifier        | SÍ   | FK DatosBancarios — cuenta destino PROQUIFA              | Existente           |
| `IdCatBanco`            | uniqueidentifier        | SÍ   | FK catBanco — banco emisor del cliente                   | Existente           |
| `CuentaOrdenante`       | varchar(80)             | SÍ   | Cuenta origen del cliente (texto libre)                  | Existente           |
| `ReferenciaBancaria`    | varchar(80)             | SÍ   | Referencia bancaria del pago                             | Existente           |
| `IdArchivo`             | uniqueidentifier        | SÍ   | FK Archivo — comprobante de pago seleccionado del correo | Existente           |
| `Activo`                | bit                     | NO   | 1=activo; 0=inconsistencia (elimina pendiente del Buzón) | Existente           |
| `FechaRegistro`         | datetime2(7)            | NO   | Auditoría: cuándo se creó el registro                    | Existente           |
| `FechaUltimaActualizacion` | datetime2(7)         | NO   | Auditoría: cuándo se modificó por última vez             | Existente           |
| `Confirmado`            | bit NOT NULL DEFAULT(0) | NO   | 0=borrador / 1=confirmado e inmutable                    | **NUEVO RE-FU-024** |
| `FechaConfirmacion`     | datetime2               | SÍ   | Timestamp de confirmación del cobro                      | **NUEVO RE-FU-024** |
| `IdUsuarioConfirmacion` | uniqueidentifier        | SÍ   | Quién confirmó el cobro (trazabilidad)                   | **NUEVO RE-FU-024** |
| `Notas`                 | varchar(500)            | SÍ   | Notas opcionales del formulario del cobro                | **NUEVO RE-FU-024** |
| `IdCatMoneda`           | uniqueidentifier        | SÍ   | FK catMoneda — moneda del cobro (combo UI Paso 1)        | **NUEVO RE-FU-024** |

---

## Catálogo: catMoneda (existente — referencia)

| Columna | Descripción |
|---------|-------------|
| `IdCatMoneda` | PK |
| `ClaveMoneda` | Clave ISO (MXN, USD, PEN, ...) |
| `Moneda` | Nombre completo |
| `Activo` | bit |

> `catMoneda` ya existe en ProquifaDotNet. No requiere cambios.
> El combo "Moneda" del formulario Paso 1 se carga desde `catMoneda WHERE Activo=1`.
> `IdCatMoneda` en `fccPagoCliente` es NULL en registros existentes; se popula al crear/confirmar el cobro.

---

## Catálogo: catMedioDePago (existente — referencia)

| Columna | Descripción |
|---------|-------------|
| `ClaveFormaDePago` | varchar(2) — clave c_FormaPago SAT (01, 02, 03...) |
| `MedioDePago` | nvarchar(200) — descripción |

> No requiere cambios para México. Para Perú se agregan registros con `ClaveFormaDePago = NULL` (RE-FU-025).

---

## DDL Cambios en fccPagoCliente (5 ALTERs)

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
-- Verificar objetos dependientes de fccPagoCliente antes de ejecutar

-- 1. Inmutabilidad: 0=borrador / 1=confirmado (cobro inmutable tras confirmación)
ALTER TABLE dbo.fccPagoCliente
    ADD Confirmado bit NOT NULL
        CONSTRAINT [DF_fccPagoCliente_Confirmado] DEFAULT (0);

-- 2. Timestamp de confirmación
ALTER TABLE dbo.fccPagoCliente
    ADD FechaConfirmacion datetime2 NULL;

-- 3. Trazabilidad: quién confirmó el cobro
ALTER TABLE dbo.fccPagoCliente
    ADD IdUsuarioConfirmacion uniqueidentifier NULL;

-- 4. Notas del cobro (campo opcional del formulario)
ALTER TABLE dbo.fccPagoCliente
    ADD Notas varchar(500) NULL;

-- 5. Moneda del cobro como FK (para combo en UI — detectado en pantalla Paso 1)
ALTER TABLE dbo.fccPagoCliente
    ADD IdCatMoneda uniqueidentifier NULL
        CONSTRAINT [FK_fccPagoCliente_CatMoneda]
            FOREIGN KEY REFERENCES dbo.catMoneda([IdCatMoneda]);
```

---

## Tabla Nueva: catTipoInconsistenciaCobro

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
CREATE TABLE [dbo].[catTipoInconsistenciaCobro](
    [IdCatTipoInconsistenciaCobro] uniqueidentifier NOT NULL
        CONSTRAINT [DF_catTipoInconsistenciaCobro_Id] DEFAULT (NEWID()),
    [Clave]       varchar(50)  NOT NULL,
    [Descripcion] varchar(200) NOT NULL,
    [AplicaPaso]  varchar(1)   NOT NULL,  -- '1'=Paso 1 (cobro), '2'=Paso 2 (asociación)
    [Activo]      bit          NOT NULL CONSTRAINT [DF_catTipoInconsistenciaCobro_Activo] DEFAULT (1),
    CONSTRAINT [PK_catTipoInconsistenciaCobro] PRIMARY KEY CLUSTERED ([IdCatTipoInconsistenciaCobro]),
    CONSTRAINT [UQ_catTipoInconsistenciaCobro_Clave] UNIQUE ([Clave])
);

-- Datos iniciales (catálogo completo pendiente de PROQUIFA Tesorería)
INSERT INTO [dbo].[catTipoInconsistenciaCobro] ([Clave], [Descripcion], [AplicaPaso])
VALUES
    ('DATOS_INCOMPLETOS',      'Datos del cobro incompletos o ilegibles',   '1'),
    ('COMPROBANTE_INVALIDO',   'Comprobante de pago inválido o ilegible',   '1'),
    ('FORMATO_INCORRECTO',     'Formato del comprobante incorrecto',         '1'),
    ('MONTO_ILEGIBLE',         'Monto del comprobante ilegible',             '1'),
    ('PAGO_INCOMPLETO_VENCIDO','Pago incompleto con vigencia vencida',       '2'),
    ('PAGO_INSUFICIENTE',      'Cobro insuficiente respecto al documento',   '2');
```

---

## Tabla Nueva: fccInconsistenciaCobro

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
CREATE TABLE [dbo].[fccInconsistenciaCobro](
    [IdFCCInconsistenciaCobro]     uniqueidentifier NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_Id] DEFAULT (NEWID()),
    [IdFCCPagoCliente]             uniqueidentifier NOT NULL,
    [IdCatTipoInconsistenciaCobro] uniqueidentifier NOT NULL,
    [Comentario]                   varchar(500) NULL,
    [IdUsuarioRegistro]            uniqueidentifier NOT NULL,
    [Activo]                       bit NOT NULL CONSTRAINT [DF_fccInconsistenciaCobro_Activo] DEFAULT (1),
    [FechaRegistro]                datetime2(7) NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_FechaReg] DEFAULT (SYSUTCDATETIME()),
    [FechaUltimaActualizacion]     datetime2(7) NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_FechaUpd] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_fccInconsistenciaCobro]
        PRIMARY KEY CLUSTERED ([IdFCCInconsistenciaCobro]),
    CONSTRAINT [FK_fccInconsistenciaCobro_PagoCliente]
        FOREIGN KEY ([IdFCCPagoCliente]) REFERENCES dbo.fccPagoCliente([IdFCCPagoCliente]),
    CONSTRAINT [FK_fccInconsistenciaCobro_TipoInconsistencia]
        FOREIGN KEY ([IdCatTipoInconsistenciaCobro]) REFERENCES dbo.catTipoInconsistenciaCobro([IdCatTipoInconsistenciaCobro])
);
```

---

## SEQUENCE: SeqFolioCobro (Foliador COB)

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
-- Ajustar START WITH al MAX consecutivo existente + 1 si ya hay folios en fccPagoCliente
CREATE SEQUENCE dbo.SeqFolioCobro
    AS INT
    START WITH 1
    INCREMENT BY 1
    NO CYCLE;
-- Uso: 'COB-' + FORMAT(FechaPago,'MMddyy') + '-' + RIGHT('000000' + CAST(NEXT VALUE FOR dbo.SeqFolioCobro AS VARCHAR), 6)
-- Pendiente confirmar: global (MEX+PER compartido) o por región (SeqFolioCobroMEX / SeqFolioCobroPER)
```

| Aspecto | Valor |
|---------|-------|
| Campo BD | `fccPagoCliente.Folio` varchar(80) — ya existe |
| Formato | `COB-mmddaa-NNNNNN` |
| `mmddaa` | Fecha efectiva del cobro (`FechaPago`) |
| Consecutivo | Global o por región — **pendiente confirmar** |
| Momento generación | Al confirmar el cobro (post-alerta de confirmación) |

---

## Lógica del Folio COB

```
Estado pre-captura:   fccPagoCliente.Folio = NULL  →  UI muestra 'COB-N' (consecutivo sesión)
Estado borrador:      fccPagoCliente.Folio = NULL  →  Confirmado = 0
Estado confirmado:    fccPagoCliente.Folio = 'COB-mmddaa-NNNNNN'  →  Confirmado = 1
```

---

## Ciclo de vida del registro fccPagoCliente

| Estado | `Confirmado` | `Folio` | `Activo` | `IdCatMoneda` |
|--------|-------------|---------|----------|--------------|
| Auto-guardado (borrador) | 0 | NULL | 1 | Poblado al seleccionar moneda |
| Cobro confirmado | 1 | COB-mmddaa-NNNNNN | 1 | Poblado |
| Inconsistencia marcada | 1 | COB-mmddaa-NNNNNN | 0 | Poblado |

---

## Tablas Leídas (runtime)

| Tabla | Datos leídos | Uso |
|-------|-------------|-----|
| `fccFolioPagoCliente` | IdFCCFolioPagoCliente, Folio, FechaRecepcion | Listado cobros del Buzón |
| `CorreoRecibidoCliente` | IdCorreoRecibidoCliente | Vínculo cobro-correo |
| `CorreoRecibido` | Asunto, Cuerpo, Fecha, Hora | Detalle del correo |
| `ArchivoCorreoRecibido` | IdArchivo, NombreArchivo | Adjuntos del correo |
| `Archivo` | FileKey | Visualizar adjunto seleccionado |
| `fccPagoCliente` | Folio, Monto, FechaPago, Confirmado, IdCatMoneda | Cobros capturados |
| `Cliente` | Nombre, Alias | Cabecera del wizard |
| `DatosFacturacionCliente` | RFC, RazonSocial, IdCatMoneda | Cabecera + moneda facturación |
| `catMedioDePago` | MedioDePago, ClaveFormaDePago | Combo forma de pago SAT |
| `catMoneda` | IdCatMoneda, ClaveMoneda, Moneda | Combo moneda del cobro |
| `EmpresaDatosBancarios` + `DatosBancarios` | Cuentas PROQUIFA MEX | Combo cuenta destino |
| `catBanco` | Nombre banco | Cuenta destino |
| `catTipoInconsistenciaCobro` | Clave, Descripcion, AplicaPaso | Modal inconsistencia |

---

## Tablas Escritas (runtime)

| Tabla | Momento | Operación |
|-------|---------|-----------|
| `fccPagoCliente` | Auto-guardado | INSERT (nuevo) / UPDATE (existente) con `Confirmado=0` |
| `fccPagoCliente` | Confirmar cobro | UPDATE: `Folio`, `Confirmado=1`, `FechaConfirmacion`, `IdUsuarioConfirmacion`, `IdCatMoneda` |
| `fccInconsistenciaCobro` | Confirmar inconsistencia modal | INSERT |

---

## Gaps y Pendientes

| # | Gap | Tipo | Acción |
|---|-----|------|--------|
| 1 | Catálogo completo catTipoInconsistenciaCobro | Negocio | Solicitar a PROQUIFA Tesorería |
| 2 | Foliador COB global vs por región | Negocio | Confirmar con cliente |
| 3 | Fuente oficial TC del día para México (TC FIX Banxico/DOF) | Técnico | Confirmar con PROQUIFA |
| 4 | Flags MXN/USD vs IdCatMoneda — coexistencia o deprecación | Técnico | Confirmar si se unifican o conviven |
| 5 | Alcance asistencia automatizada IA | Técnico | No comprometido en R16 |
| 6 | Moneda base del TC: MXN vs moneda facturación | Fiscal | Confirmar con asesor fiscal |

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| R16A-RE-FU-008 | Buzón Cobros — `fccFolioPagoCliente` / `CorreoRecibidoCliente` |
| R16A-RE-FU-023 | Pantalla principal VC — lista cobros pendientes |
| R16A-RE-FU-025 | Paso 1 Perú — reutiliza toda esta estructura sin DDL nuevo |
| R16A-RE-FU-026 | Paso 2 México — extiende `catTipoInconsistenciaCobro` y lee `fccPagoCliente` |

---

**Generado por:** GitHub Copilot
**Base de Datos:** ProquifaDotNet