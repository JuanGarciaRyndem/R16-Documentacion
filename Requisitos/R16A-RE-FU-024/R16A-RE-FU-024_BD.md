# Impacto en BD — R16A-RE-FU-024
**Requisito:** Validar Cobro: Paso 1 México — Captura del Cobro
**Base de Datos:** ProquifaDotNet
**Versión:** 1.2 (actualizado 2026-06-23: editabilidad del cobro hasta el timbrado — se agrega `BloqueadoPorTimbrado` y se redefine la semántica del campo `Confirmado`)

---

## Resumen
Paso 1 del wizard de Validar Cobro: Captura del Cobro para Región México.
El cobro se registra en `fccPagoCliente` (tabla existente).
Se extiende la tabla con 5 campos agregados en RE-FU-023 (captura + notas + moneda como FK) y se suma **1 campo nuevo en RE-FU-024 (`BloqueadoPorTimbrado`) para reflejar el cambio de regla**: el cobro deja de ser inmutable al confirmar la captura y queda inmutable únicamente al timbrar el documento asociado en el Paso 3.
Se crean 2 tablas nuevas (inconsistencias) y 1 SEQUENCE (foliador COB).

> **🔄 Cambio funcional 2026-06-23 — Editabilidad del cobro hasta el timbrado:**
> La regla original "cobro inmutable al confirmar la captura" se actualizó a "cobro inmutable al timbrar el documento asociado en el Paso 3". El campo `Confirmado` mantiene su existencia física pero **cambia su semántica de negocio**: ahora significa "cobro capturado y persistido con folio COB" (estado de lectura con botón Editar), NO "cobro inmutable". La inmutabilidad ahora se determina por el nuevo campo `BloqueadoPorTimbrado` (bit), que el Paso 3 actualiza a `1` cuando se timbra el documento asociado al cobro. Mientras `BloqueadoPorTimbrado=0`, el usuario puede editar el cobro desde el Paso 1 vía botón Editar, aun si el cobro ya está asociado en el Paso 2.

> **✅ Los 5 campos `Confirmado / FechaConfirmacion / IdUsuarioConfirmacion / Notas / IdCatMoneda` ya fueron agregados en RE-FU-023 (ejecutados en BD).** La Tarea 1 de este requisito queda como referencia documental — no requiere ejecución para esos 5 campos.
>
> **❌ El campo `BloqueadoPorTimbrado` es nuevo en RE-FU-024 y SÍ requiere ejecución (DDL pendiente).**
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

| #   | Cambio                                                                                              | Tipo | Estado                   |
| --- | --------------------------------------------------------------------------------------------------- | ---- | ------------------------ |
| 1   | ALTER TABLE fccPagoCliente ADD Confirmado bit NOT NULL DEFAULT(0) *(semántica: cobro capturado)*    | DDL  | ✅ Ejecutado en RE-FU-023 |
| 2   | ALTER TABLE fccPagoCliente ADD FechaConfirmacion datetime NULL                                     | DDL  | ✅ Ejecutado en RE-FU-023 |
| 3   | ALTER TABLE fccPagoCliente ADD IdUsuarioConfirmacion uniqueidentifier NULL                          | DDL  | ✅ Ejecutado en RE-FU-023 |
| 4   | ALTER TABLE fccPagoCliente ADD Notas varchar(500) NULL                                              | DDL  | ✅ Ejecutado en RE-FU-023 |
| 5   | ALTER TABLE fccPagoCliente ADD IdCatMoneda uniqueidentifier NULL FK catMoneda                       | DDL  | ✅ Ejecutado en RE-FU-023 |
| 6   | **ALTER TABLE fccPagoCliente ADD BloqueadoPorTimbrado bit NOT NULL DEFAULT(0)** *(inmutabilidad)*   | DDL  | ❌ Pendiente RE-FU-024    |
| 7   | **ALTER TABLE fccPagoCliente ADD FechaBloqueoTimbrado datetime NULL** *(trazabilidad del bloqueo)* | DDL  | ❌ Pendiente RE-FU-024    |
| 8   | CREATE TABLE catTipoInconsistenciaCobro                                                             | DDL  | ❌ Pendiente              |
| 9   | CREATE TABLE fccInconsistenciaCobro                                                                 | DDL  | ❌ Pendiente              |
| 10  | CREATE SEQUENCE dbo.SeqFolioCobro                                                                   | DDL  | ❌ Pendiente              |

> **Nota #5 — IdCatMoneda:** La pantalla del Paso 1 (Captura del Cobro) muestra un combo
> desplegable de Moneda (ej. "USD"). Esto requiere un FK a `catMoneda` para cargar las opciones.
> Los flags `MXN`/`USD` existentes solo soportan 2 monedas y no sirven para PEN (Perú) ni combos.
> Se agrega `IdCatMoneda` nullable para no romper registros existentes; los flags se mantienen
> por compatibilidad con código legacy hasta que se unifique.

> **Nota #6/#7 — BloqueadoPorTimbrado / FechaBloqueoTimbrado:** Nuevos campos en RE-FU-024
> para implementar la regla actualizada de inmutabilidad. `BloqueadoPorTimbrado=0` indica que
> el cobro está en modo lectura **editable** (botón Editar visible en el Paso 1, incluso si ya está
> asociado en el Paso 2). `BloqueadoPorTimbrado=1` indica inmutabilidad real (sin botón Editar).
> El flip de `0→1` lo dispara el Paso 3 al timbrar el documento asociado al cobro, mediante
> UPDATE sobre todas las `fccPagoCliente` aplicadas a ese documento. `FechaBloqueoTimbrado`
> registra el timestamp del bloqueo para trazabilidad. Para los cobros con etiqueta "Saldo a favor"
> visibles en el Paso 1, `BloqueadoPorTimbrado=1` ya está poblado (corresponden a cobros
> timbrados en sesiones previas).

---

## Tabla: fccPagoCliente — Estructura completa post-R16

| Columna                    | Tipo                    | Nulo | Descripción                                              | Estado                                            |
| -------------------------- | ----------------------- | ---- | -------------------------------------------------------- | ------------------------------------------------- |
| `IdFCCPagoCliente`         | uniqueidentifier        | NO   | PK                                                       | Existente                                         |
| `IdCliente`                | uniqueidentifier        | NO   | FK Cliente                                               | Existente                                         |
| `IdEmpresa`                | uniqueidentifier        | NO   | FK Empresa que recibe el cobro                           | Existente                                         |
| `IdFCCFolioPagoCliente`    | uniqueidentifier        | SÍ   | FK fccFolioPagoCliente — vínculo correo Buzón            | Existente                                         |
| `Folio`                    | varchar(80)             | SÍ   | Formato COB-mmddaa-NNNN al confirmar; NULL en borrador   | Existente                                         |
| `Monto`                    | decimal(18,4)           | NO   | Monto recibido del cliente                               | Existente                                         |
| `FechaPago`                | datetime                | SÍ   | Fecha efectiva del pago                                  | Existente                                         |
| `TipoDeCambio`             | decimal                 | SÍ   | TC del día vs MXN (calculado automático, solo lectura)   | Existente                                         |
| `MXN`                      | bit                     | NO   | Bandera moneda pesos (legacy)                            | Existente                                         |
| `USD`                      | bit                     | NO   | Bandera moneda dólares (legacy)                          | Existente                                         |
| `IdCatMedioDePago`         | uniqueidentifier        | SÍ   | FK catMedioDePago — forma de pago c_FormaPago SAT        | Existente                                         |
| `IdDatosBancarios`         | uniqueidentifier        | SÍ   | FK DatosBancarios — cuenta destino PROQUIFA              | Existente                                         |
| `IdCatBanco`               | uniqueidentifier        | SÍ   | FK catBanco — banco emisor del cliente                   | Existente                                         |
| `CuentaOrdenante`          | varchar(80)             | SÍ   | Cuenta origen del cliente (texto libre)                  | Existente                                         |
| `ReferenciaBancaria`       | varchar(80)             | SÍ   | Referencia bancaria del pago                             | Existente                                         |
| `IdArchivo`                | uniqueidentifier        | SÍ   | FK Archivo — comprobante de pago seleccionado del correo | Existente                                         |
| `Activo`                   | bit                     | NO   | 1=activo; 0=inconsistencia (elimina pendiente del Buzón) | Existente                                         |
| `FechaRegistro`            | datetime            | NO   | Auditoría: cuándo se creó el registro                    | Existente                                         |
| `FechaUltimaActualizacion` | datetime            | NO   | Auditoría: cuándo se modificó por última vez             | Existente                                         |
| `Confirmado`               | bit NOT NULL DEFAULT(0) | NO   | 0=borrador / 1=capturado (editable hasta timbrar)        | **RE-FU-023** (semántica redefinida en RE-FU-024) |
| `FechaConfirmacion`        | datetime               | SÍ   | Timestamp de captura inicial del cobro                   | **RE-FU-023**                                     |
| `IdUsuarioConfirmacion`    | uniqueidentifier        | SÍ   | Quién capturó el cobro inicialmente (trazabilidad)       | **RE-FU-023**                                     |
| `Notas`                    | varchar(500)            | SÍ   | Notas opcionales del formulario del cobro                | **RE-FU-023**                                     |
| `IdCatMoneda`              | uniqueidentifier        | SÍ   | FK catMoneda — moneda del cobro (combo UI Paso 1)        | **RE-FU-023**                                     |
| `BloqueadoPorTimbrado`     | bit NOT NULL DEFAULT(0) | NO   | 0=editable vía botón Editar / 1=inmutable post-timbrado  | **NUEVO RE-FU-024**                               |
| `FechaBloqueoTimbrado`     | datetime               | SÍ   | Timestamp del bloqueo (Paso 3 timbró el doc. asociado)   | **NUEVO RE-FU-024**                               |

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

## DDL Cambios en fccPagoCliente (5 ALTERs ejecutados en RE-FU-023 + 2 ALTERs nuevos en RE-FU-024)

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
-- Verificar objetos dependientes de fccPagoCliente antes de ejecutar

-- ===================================================================
-- Bloque A — ALTERs ejecutados en RE-FU-023 (✅ ya aplicados en BD)
-- ===================================================================

-- 1. Captura del cobro: 0=borrador (auto-guardado) / 1=capturado (editable hasta timbrar)
--    NOTA RE-FU-024: la semántica del campo se actualiza. Ya NO implica inmutabilidad;
--    indica únicamente que la captura fue finalizada y el cobro tiene folio COB.
ALTER TABLE dbo.fccPagoCliente
    ADD Confirmado bit NOT NULL
        CONSTRAINT [DF_fccPagoCliente_Confirmado] DEFAULT (0);

-- 2. Timestamp de captura inicial del cobro (no de inmutabilidad)
ALTER TABLE dbo.fccPagoCliente
    ADD FechaConfirmacion datetime NULL;

-- 3. Trazabilidad: quién capturó el cobro inicialmente
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

-- ===================================================================
-- Bloque B — ALTERs nuevos en RE-FU-024 (❌ pendientes de ejecutar)
-- ===================================================================

-- 6. Inmutabilidad real: 0=editable vía botón Editar / 1=inmutable post-timbrado
--    Flip lo dispara el Paso 3 al timbrar el documento asociado al cobro.
ALTER TABLE dbo.fccPagoCliente
    ADD BloqueadoPorTimbrado bit NOT NULL
        CONSTRAINT [DF_fccPagoCliente_BloqueadoPorTimbrado] DEFAULT (0);

-- 7. Timestamp del bloqueo por timbrado (trazabilidad)
ALTER TABLE dbo.fccPagoCliente
    ADD FechaBloqueoTimbrado datetime NULL;
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
    [FechaRegistro]                datetime NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_FechaReg] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion]     datetime NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_FechaUpd] DEFAULT (GETDATE()),
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
Estado capturado:     fccPagoCliente.Folio = 'COB-mmddaa-NNNNNN'  →  Confirmado = 1, BloqueadoPorTimbrado = 0  (editable via botón Editar)
Estado inmutable:     fccPagoCliente.Folio = 'COB-mmddaa-NNNNNN'  →  Confirmado = 1, BloqueadoPorTimbrado = 1  (post-timbrado Paso 3, sin botón Editar)
```

---

## Ciclo de vida del registro fccPagoCliente

| Estado                         | `Confirmado` | `BloqueadoPorTimbrado` | `Folio`           | `Activo` | `IdCatMoneda`          | Acciones UI Paso 1                 |
| ------------------------------ | ------------ | ---------------------- | ----------------- | -------- | ---------------------- | ---------------------------------- |
| Auto-guardado (borrador)       | 0            | 0                      | NULL              | 1        | Poblado al seleccionar | Edición continua del formulario    |
| Cobro capturado (editable)     | 1            | 0                      | COB-mmddaa-NNNNNN | 1        | Poblado                | Lectura + **botón Editar visible** |
| Cobro inmutable (post-timbrar) | 1            | 1                      | COB-mmddaa-NNNNNN | 1        | Poblado                | Solo lectura (sin botón Editar)    |
| Saldo a favor (post-timbrar)   | 1            | 1                      | COB-mmddaa-NNNNNN | 1        | Poblado                | Solo lectura (sin botón Editar)    |
| Inconsistencia marcada         | 1            | 0 o 1                  | COB-mmddaa-NNNNNN | 0        | Poblado                | Lectura (Editar según bloqueo)     |

> El flip `BloqueadoPorTimbrado = 0 → 1` lo dispara el Paso 3 al timbrar el documento al que se aplicó el cobro (un cobro puede estar asociado a uno o más documentos; en cuanto se timbra **cualquiera**, el cobro queda inmutable).

---

## Tablas Leídas (runtime)

| Tabla | Datos leídos | Uso |
|-------|-------------|-----|
| `fccFolioPagoCliente` | IdFCCFolioPagoCliente, Folio, FechaRecepcion | Listado cobros del Buzón |
| `CorreoRecibidoCliente` | IdCorreoRecibidoCliente | Vínculo cobro-correo |
| `CorreoRecibido` | Asunto, Cuerpo, Fecha, Hora | Detalle del correo |
| `ArchivoCorreoRecibido` | IdArchivo, NombreArchivo | Adjuntos del correo |
| `Archivo` | FileKey | Visualizar adjunto seleccionado |
| `fccPagoCliente` | Folio, Monto, FechaPago, Confirmado, BloqueadoPorTimbrado, IdCatMoneda | Cobros capturados (Confirmado + BloqueadoPorTimbrado determinan si se muestra el botón Editar) |
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
| `fccPagoCliente` | Auto-guardado del borrador | INSERT (nuevo) / UPDATE (existente) con `Confirmado=0`. Guardia: solo si `BloqueadoPorTimbrado=0`. |
| `fccPagoCliente` | Finalización de la captura | UPDATE: `Folio`, `Confirmado=1`, `FechaConfirmacion`, `IdUsuarioConfirmacion`, `IdCatMoneda`. **NO toca `BloqueadoPorTimbrado` (sigue en 0, el cobro queda editable vía botón Editar).** |
| `fccPagoCliente` | Edición del cobro vía botón Editar (Paso 1) | UPDATE de los campos del formulario (Monto, FechaPago, IdCatMedioDePago, CuentaOrdenante, IdDatosBancarios, IdCatMoneda, TipoDeCambio, IdArchivo, Notas). Guardia: solo si `BloqueadoPorTimbrado=0`. **No se regenera el Folio.** |
| `fccPagoCliente` | Timbrado del documento asociado (Paso 3) | UPDATE: `BloqueadoPorTimbrado=1`, `FechaBloqueoTimbrado=GETDATE()`. Disparado desde el flujo del Paso 3 sobre todas las `fccPagoCliente` aplicadas al documento timbrado. |
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
| Paso 3 (Facturación y Envío) | Dispara el UPDATE `BloqueadoPorTimbrado=1` sobre las `fccPagoCliente` aplicadas al documento timbrado (hace efectiva la inmutabilidad del cobro) |

---

**Generado por:** GitHub Copilot
**Base de Datos:** ProquifaDotNet