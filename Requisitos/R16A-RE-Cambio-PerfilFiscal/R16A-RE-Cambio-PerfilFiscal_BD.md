# Diccionario de Datos — R16A-RE-Cambio-PerfilFiscal

| Campo | Valor |
|---|---|
| **Cambio** | R16A-RE-Cambio-PerfilFiscal |
| **Nombre** | Perfil Fiscal — Configuración fiscal por Familia de Producto |
| **Base de datos** | ProquifaDotNet |
| **Servidor** | RYNL010 |
| **Versión** | 1.0 |

---

## Resumen Ejecutivo

Este cambio desbloquea el bloque **GAP-7 / GAP-8** de `R16A-RE-FU-019_BD.md`. Confirma que la configuración fiscal se define **a nivel Familia + Región** (sin override por Producto individual). Se crean 5 tablas nuevas en ProquifaDotNet. La tabla `Familia` **no se modifica**.

**Decisión confirmada:**
- `FamiliaRegion` → nueva tabla de configuración fiscal por Familia+Región; una fila por combinación; contiene `ClaveProdServSat`, `ClaveUnidadSat` (SAT — solo MX) y `IdPerfilFiscal` (único FK, discriminado por región)
- `PerfilFiscal` → tabla única con `IdRegion`; filas MX (IVA) + filas PE (IGV); **sin override por Producto**; sin columnas `Nombre` ni `CodigoImpuesto` — el código del impuesto se deriva de `catImpuesto.Clave`
- `Familia` → **sin cambios** — la configuración fiscal vive en `FamiliaRegion`
- `catImpuesto` → catálogo compartido MX+PE: IVA (México) + IGV (Perú)
- `catTipoFactorSat`, `catObjetoImpuestoSat` → catálogos SAT; `catObjetoImpuestoSat` exclusivo MX (FK nullable en PE)

---

## Modelo de Datos

```
FamiliaRegion  (nueva — configuración fiscal por Familia + Región)
    ├── IdFamilia          FK → Familia (NOT NULL)
    ├── IdRegion           FK → Region  (NOT NULL)
    ├── IdPerfilFiscal     FK → PerfilFiscal (NOT NULL)
    ├── ClaveProdServSat   varchar(10) nullable  [solo MX — SAT; NULL para PE]
    └── ClaveUnidadSat     varchar(10) nullable  [solo MX — SAT; NULL para PE]
    * UQ (IdFamilia, IdRegion)

PerfilFiscal  (nueva — MX + PE separados por IdRegion)
    ├── IdRegion               FK → Region (NOT NULL)
    ├── IdCatImpuesto          FK → catImpuesto (NOT NULL — IVA para MX, IGV para PE)
    ├── IdCatTipoFactorSat     FK → catTipoFactorSat   [Tasa / Exento — compartido]
    ├── TasaOCuota             decimal(6,6)
    └── IdCatObjetoImpuestoSat FK nullable → catObjetoImpuestoSat  [NULL en filas PE]

catImpuesto           (nueva — catálogo compartido: IVA México + IGV Perú)
catTipoFactorSat      (nueva — catálogo maestro SAT, compartido MX+PE)
catObjetoImpuestoSat  (nueva — catálogo maestro SAT, exclusivo México)

Familia  (sin cambios)
```

---

## Entidades Afectadas

| Objeto                 | Tipo            | BD             | Impacto                | Descripción                                                                           |
| ---------------------- | --------------- | -------------- | ---------------------- | ------------------------------------------------------------------------------------- |
| `catImpuesto`          | Tabla nueva     | ProquifaDotNet | CREATE + INSERT seed   | Catálogo compartido MX+PE: IVA (México), IGV (Perú)                                   |
| `catTipoFactorSat`     | Tabla nueva     | ProquifaDotNet | CREATE + INSERT seed   | Catálogo SAT: Tasa, Cuota, Exento (compartido MX+PE)                                  |
| `catObjetoImpuestoSat` | Tabla nueva     | ProquifaDotNet | CREATE + INSERT seed   | Catálogo SAT: objetos de impuesto 01–04 (exclusivo México)                             |
| `PerfilFiscal`         | Tabla nueva     | ProquifaDotNet | CREATE + INSERT seed   | Perfiles MX (IVA 16%, IVA 0%, Exento) + PE (IGV 18%, IGV 0%, Exento), con `IdRegion` |
| `FamiliaRegion`        | Tabla nueva     | ProquifaDotNet | CREATE + INSERT seed   | Configuración fiscal por Familia+Región: `IdPerfilFiscal`, `ClaveProdServSat`, `ClaveUnidadSat` |
| `Familia`              | Tabla existente | ProquifaDotNet | Sin cambios            | No recibe columnas nuevas — configuración fiscal delegada a `FamiliaRegion`            |

**Orden de ejecución obligatorio:**
`catImpuesto` → `catTipoFactorSat` → `catObjetoImpuestoSat` → `PerfilFiscal` → `FamiliaRegion`

---

## 1. catImpuesto

**Propósito:** Catálogo compartido MX+PE de tipos de impuesto. Incluye los impuestos aplicables en México (SAT) y Perú (SUNAT). No se modifica en operación; se precargan los datos.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `IdCatImpuesto` | `uniqueidentifier` | NO | `NEWID()` | PK |
| `Clave` | `varchar(10)` | NO | — | Clave del impuesto: `001` ISR, `002` IVA, `003` IEPS (SAT MX) · `IGV` (Perú) |
| `Descripcion` | `varchar(100)` | NO | — | Descripción del impuesto |
| `Activo` | `bit` | NO | `1` | Borrado lógico |
| `FechaRegistro` | `datetime` | NO | `GETDATE()` | Fecha de alta |
| `FechaUltimaActualizacion` | `datetime` | NO | `GETDATE()` | Fecha de última modificación |

**Índices:**
- `PK_catImpuesto` (Clustered): `IdCatImpuesto`
- `UQ_catImpuesto_Clave` (Unique): `Clave`

**Relaciones:** ninguna FK hacia otras tablas (catálogo raíz).

```sql
-- Ejecutar en ProquifaDotNet
CREATE TABLE [dbo].[catImpuesto](
    [IdCatImpuesto] uniqueidentifier NOT NULL
        CONSTRAINT [DF_catImpuesto_Id] DEFAULT (NEWID()),
    [Clave]         varchar(10)  NOT NULL,
    [Descripcion]   varchar(100) NOT NULL,
    [Activo]        bit          NOT NULL CONSTRAINT [DF_catImpuesto_Activo] DEFAULT (1),
    [FechaRegistro] datetime NOT NULL CONSTRAINT [DF_catImpuesto_FechaRegistro] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion] datetime NOT NULL CONSTRAINT [DF_DF_catImpuesto_FechaUltimaActualizacion] DEFAULT (GETDATE()),
    CONSTRAINT [PK_catImpuesto] PRIMARY KEY CLUSTERED ([IdCatImpuesto]),
    CONSTRAINT [UQ_catImpuesto_Clave] UNIQUE ([Clave])
);
GO

INSERT INTO [dbo].[catImpuesto] ([Clave], [Descripcion]) VALUES
    ('001', 'ISR'),
    ('002', 'IVA'),
    ('003', 'IEPS'),
    ('IGV', 'Impuesto General a las Ventas');
GO
```

---

## 2. catTipoFactorSat

**Propósito:** Catálogo maestro SAT de tipos de factor de impuesto.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `IdCatTipoFactorSat` | `uniqueidentifier` | NO | `NEWID()` | PK |
| `Clave` | `varchar(10)` | NO | — | Clave SAT: `Tasa`, `Cuota`, `Exento` |
| `Descripcion` | `varchar(100)` | NO | — | Descripción del tipo de factor |
| `Activo` | `bit` | NO | `1` | Borrado lógico |
| `FechaRegistro` | `datetime` | NO | `GETDATE()` | Fecha de alta |
| `FechaUltimaActualizacion` | `datetime` | NO | `GETDATE()` | Fecha de última modificación |

**Índices:**
- `PK_catTipoFactorSat` (Clustered): `IdCatTipoFactorSat`
- `UQ_catTipoFactorSat_Clave` (Unique): `Clave`

```sql
CREATE TABLE [dbo].[catTipoFactorSat](
    [IdCatTipoFactorSat] uniqueidentifier NOT NULL
        CONSTRAINT [DF_catTipoFactorSat_Id] DEFAULT (NEWID()),
    [Clave]       varchar(10)  NOT NULL,
    [Descripcion] varchar(100) NOT NULL,
    [Activo]      bit          NOT NULL CONSTRAINT [DF_catTipoFactorSat_Activo] DEFAULT (1),
    [FechaRegistro] datetime NOT NULL CONSTRAINT [DF_catTipoFactorSat_FechaRegistro] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion] datetime NOT NULL CONSTRAINT [DF_DF_catTipoFactorSat_FechaUltimaActualizacion] DEFAULT (GETDATE()),
    CONSTRAINT [PK_catTipoFactorSat] PRIMARY KEY CLUSTERED ([IdCatTipoFactorSat]),
    CONSTRAINT [UQ_catTipoFactorSat_Clave] UNIQUE ([Clave])
);
GO

INSERT INTO [dbo].[catTipoFactorSat] ([Clave], [Descripcion]) VALUES
    ('Tasa',   'Tasa'),
    ('Cuota',  'Cuota'),
    ('Exento', 'Exento');
GO
```

---

## 3. catObjetoImpuestoSat

**Propósito:** Catálogo maestro SAT de objetos de impuesto (campo `ObjetoImp` del CFDI 4.0).

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `IdCatObjetoImpuestoSat` | `uniqueidentifier` | NO | `NEWID()` | PK |
| `Clave` | `varchar(10)` | NO | — | Clave SAT: `01`–`04` |
| `Descripcion` | `varchar(200)` | NO | — | Descripción del objeto de impuesto |
| `Activo` | `bit` | NO | `1` | Borrado lógico |
| `FechaRegistro` | `datetime` | NO | `GETDATE()` | Fecha de alta |
| `FechaUltimaActualizacion` | `datetime` | NO | `GETDATE()` | Fecha de última modificación |

**Índices:**
- `PK_catObjetoImpuestoSat` (Clustered): `IdCatObjetoImpuestoSat`
- `UQ_catObjetoImpuestoSat_Clave` (Unique): `Clave`

```sql
CREATE TABLE [dbo].[catObjetoImpuestoSat](
    [IdCatObjetoImpuestoSat] uniqueidentifier NOT NULL
        CONSTRAINT [DF_catObjetoImpuestoSat_Id] DEFAULT (NEWID()),
    [Clave]       varchar(10)  NOT NULL,
    [Descripcion] varchar(200) NOT NULL,
    [Activo]      bit          NOT NULL CONSTRAINT [DF_catObjetoImpuestoSat_Activo] DEFAULT (1),
    [FechaRegistro] datetime NOT NULL CONSTRAINT [DF_catObjetoImpuestoSat_FechaRegistro] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion] datetime NOT NULL CONSTRAINT [DF_DF_catObjetoImpuestoSat_FechaUltimaActualizacion] DEFAULT (GETDATE()),
    CONSTRAINT [PK_catObjetoImpuestoSat] PRIMARY KEY CLUSTERED ([IdCatObjetoImpuestoSat]),
    CONSTRAINT [UQ_catObjetoImpuestoSat_Clave] UNIQUE ([Clave])
);
GO

INSERT INTO [dbo].[catObjetoImpuestoSat] ([Clave], [Descripcion]) VALUES
    ('01', 'No objeto de impuesto'),
    ('02', 'Sí objeto de impuesto'),
    ('03', 'Sí objeto del impuesto y no obligado al desglose'),
    ('04', 'Sí objeto del impuesto y no causa impuesto');
GO
```

---

## 4. PerfilFiscal

**Propósito:** Catálogo de negocio unificado MX + PE que asocia la tasa de impuesto de una Familia con los datos necesarios para calcular impuestos y construir documentos fiscales. Discriminado por `IdRegion`. Lo administra PROQUIFA directamente en BD.

- **México (MX):** 3 filas. `IdCatImpuesto` apunta a IVA (`002`). Incluye `IdCatObjetoImpuestoSat`.
- **Perú (PE):** 3 filas. `IdCatImpuesto` apunta a IGV. `IdCatObjetoImpuestoSat` = `NULL` (no aplica SAT).

| Columna                  | Tipo               | Nulo | Default            | Descripción                                                                |
| ------------------------ | ------------------ | ---- | ------------------ | -------------------------------------------------------------------------- |
| `IdPerfilFiscal`         | `uniqueidentifier` | NO   | `NEWID()`          | PK                                                                         |
| `IdRegion`               | `uniqueidentifier` | NO   | —                  | FK → `Region` — discrimina MX vs PE                                        |
| `IdCatImpuesto`          | `uniqueidentifier` | NO   | —                  | FK → `catImpuesto` — IVA para MX, IGV para PE                              |
| `IdCatTipoFactorSat`     | `uniqueidentifier` | NO   | —                  | FK → `catTipoFactorSat` — `Tasa` o `Exento` (compartido MX+PE)             |
| `TasaOCuota`             | `decimal(6,6)`     | SÍ   | —                  | Valor de la tasa: `0.160000`, `0.180000`, `0.000000`, o `NULL` para Exento |
| `IdCatObjetoImpuestoSat` | `uniqueidentifier` | SÍ   | `NULL`             | FK → `catObjetoImpuestoSat`. Solo filas MX (`02`). `NULL` para PE          |
| `Fundamento`             | `nvarchar(200)`    | SÍ   | —                  | Referencia legal: `Art. 1 LIVA`, `TUO IGV Art. 1`, etc.                    |
| `Activo`                 | `bit`              | NO   | `1`                | Borrado lógico                                                             |
| `FechaRegistro`          | `datetime`     | NO   | `GETDATE()` | Fecha de alta                                                              |
| `FechaUltimaActualizacion`          | `datetime`     | NO   | `GETDATE()` | Fecha de última modificación |

**Índices:**
- `PK_PerfilFiscal` (Clustered): `IdPerfilFiscal`
- `IX_PerfilFiscal_IdRegion` (Non-Clustered): `IdRegion` (para filtrar por región)

**Relaciones:**
- `FK_PerfilFiscal_Region` → `Region.IdRegion`
- `FK_PerfilFiscal_catImpuesto` → `catImpuesto.IdCatImpuesto`
- `FK_PerfilFiscal_catTipoFactorSat` → `catTipoFactorSat.IdCatTipoFactorSat`
- `FK_PerfilFiscal_catObjetoImpuestoSat` → `catObjetoImpuestoSat.IdCatObjetoImpuestoSat` (nullable)

**Consideraciones especiales:**
- `CHECK CK_PerfilFiscal_TasaOCuota`: `TasaOCuota` es `NULL` si y solo si `catTipoFactorSat.Clave = 'Exento'`.
- Filas PE: `IdCatObjetoImpuestoSat` = `NULL` — el campo `ObjetoImp` del CFDI SAT no aplica para documentos de Perú.

```sql
-- Ejecutar en ProquifaDotNet (requiere catImpuesto, catTipoFactorSat, catObjetoImpuestoSat y Region)
CREATE TABLE [dbo].[PerfilFiscal](
    [IdPerfilFiscal] uniqueidentifier NOT NULL
        CONSTRAINT [DF_PerfilFiscal_Id] DEFAULT (NEWID()),
    [IdRegion]                uniqueidentifier NOT NULL
        CONSTRAINT [FK_PerfilFiscal_Region]
            FOREIGN KEY REFERENCES [dbo].[Region]([IdRegion]),
    [IdCatImpuesto]           uniqueidentifier NOT NULL
        CONSTRAINT [FK_PerfilFiscal_catImpuesto]
            FOREIGN KEY REFERENCES [dbo].[catImpuesto]([IdCatImpuesto]),
    [IdCatTipoFactorSat]      uniqueidentifier NOT NULL
        CONSTRAINT [FK_PerfilFiscal_catTipoFactorSat]
            FOREIGN KEY REFERENCES [dbo].[catTipoFactorSat]([IdCatTipoFactorSat]),
    [TasaOCuota]              decimal(6,6)     NULL,
    [IdCatObjetoImpuestoSat]  uniqueidentifier NULL
        CONSTRAINT [FK_PerfilFiscal_catObjetoImpuestoSat]
            FOREIGN KEY REFERENCES [dbo].[catObjetoImpuestoSat]([IdCatObjetoImpuestoSat]),
    [Fundamento]              nvarchar(200)    NULL,
    [Activo]      bit         NOT NULL CONSTRAINT [DF_PerfilFiscal_Activo] DEFAULT (1),
    [FechaRegistro] datetime NOT NULL CONSTRAINT [DF_PerfilFiscal_FechaRegistro] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion] datetime NOT NULL CONSTRAINT [DF_DF_PerfilFiscal_FechaUltimaActualizacion] DEFAULT (GETDATE()),
    CONSTRAINT [PK_PerfilFiscal] PRIMARY KEY CLUSTERED ([IdPerfilFiscal]),
    CONSTRAINT [CK_PerfilFiscal_TasaOCuota] CHECK (
        ([TasaOCuota] IS NULL     AND EXISTS (SELECT 1 FROM [dbo].[catTipoFactorSat] t WHERE t.[IdCatTipoFactorSat] = [IdCatTipoFactorSat] AND t.[Clave] = 'Exento'))
        OR
        ([TasaOCuota] IS NOT NULL AND NOT EXISTS (SELECT 1 FROM [dbo].[catTipoFactorSat] t WHERE t.[IdCatTipoFactorSat] = [IdCatTipoFactorSat] AND t.[Clave] = 'Exento'))
    )
);
GO

CREATE NONCLUSTERED INDEX [IX_PerfilFiscal_IdRegion]
    ON [dbo].[PerfilFiscal] ([IdRegion]);
GO

-- =====================================================================
-- SEED México (3 filas MX)
-- =====================================================================

-- IVA General 16%
INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT
    (SELECT IdRegion        FROM [dbo].[Region]              WHERE Clave = 'mexico'),
    (SELECT IdCatImpuesto   FROM [dbo].[catImpuesto]         WHERE Clave = '002'),
    (SELECT IdCatTipoFactorSat     FROM [dbo].[catTipoFactorSat]     WHERE Clave = 'Tasa'),
    0.160000,
    (SELECT IdCatObjetoImpuestoSat FROM [dbo].[catObjetoImpuestoSat] WHERE Clave = '02'),
    'Art. 1 LIVA';

-- IVA Tasa 0% (publicaciones MX)
INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT
    (SELECT IdRegion        FROM [dbo].[Region]              WHERE Clave = 'mexico'),
    (SELECT IdCatImpuesto   FROM [dbo].[catImpuesto]         WHERE Clave = '002'),
    (SELECT IdCatTipoFactorSat     FROM [dbo].[catTipoFactorSat]     WHERE Clave = 'Tasa'),
    0.000000,
    (SELECT IdCatObjetoImpuestoSat FROM [dbo].[catObjetoImpuestoSat] WHERE Clave = '02'),
    'Art. 2-A LIVA';

-- Exento MX
INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT
    (SELECT IdRegion        FROM [dbo].[Region]              WHERE Clave = 'mexico'),
    (SELECT IdCatImpuesto   FROM [dbo].[catImpuesto]         WHERE Clave = '002'),
    (SELECT IdCatTipoFactorSat     FROM [dbo].[catTipoFactorSat]     WHERE Clave = 'Exento'),
    NULL,
    (SELECT IdCatObjetoImpuestoSat FROM [dbo].[catObjetoImpuestoSat] WHERE Clave = '02'),
    'Art. 9 LIVA';

-- =====================================================================
-- SEED Perú (3 filas PE — IdCatObjetoImpuestoSat = NULL)
-- =====================================================================

-- IGV 18% (tasa general Perú)
INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT
    (SELECT IdRegion        FROM [dbo].[Region]          WHERE Clave = 'peru'),
    (SELECT IdCatImpuesto   FROM [dbo].[catImpuesto]     WHERE Clave = 'IGV'),
    (SELECT IdCatTipoFactorSat FROM [dbo].[catTipoFactorSat] WHERE Clave = 'Tasa'),
    0.180000,
    NULL,
    'TUO IGV Art. 1 (D.S. 055-99-EF)';

-- IGV 0% (publicaciones / exportaciones PE)
INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT
    (SELECT IdRegion        FROM [dbo].[Region]          WHERE Clave = 'peru'),
    (SELECT IdCatImpuesto   FROM [dbo].[catImpuesto]     WHERE Clave = 'IGV'),
    (SELECT IdCatTipoFactorSat FROM [dbo].[catTipoFactorSat] WHERE Clave = 'Tasa'),
    0.000000,
    NULL,
    'TUO IGV Art. 19 — Exonerado/Inafecto';

-- Exento PE
INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT
    (SELECT IdRegion        FROM [dbo].[Region]          WHERE Clave = 'peru'),
    (SELECT IdCatImpuesto   FROM [dbo].[catImpuesto]     WHERE Clave = 'IGV'),
    (SELECT IdCatTipoFactorSat FROM [dbo].[catTipoFactorSat] WHERE Clave = 'Exento'),
    NULL,
    NULL,
    'TUO IGV — Operación no gravada';
GO
```

---

## 5. FamiliaRegion

**Propósito:** Tabla de configuración fiscal por Familia y Región. Una fila por combinación `(IdFamilia, IdRegion)`. Vincula cada familia con el perfil fiscal correcto para cada región y almacena las claves SAT que aplican para México. La tabla `Familia` no se modifica.

| Columna                    | Tipo               | Nulo | Default     | Descripción                                                             |
| -------------------------- | ------------------ | ---- | ----------- | ----------------------------------------------------------------------- |
| `IdFamiliaRegion`          | `uniqueidentifier` | NO   | `NEWID()`   | PK                                                                      |
| `IdFamilia`                | `uniqueidentifier` | NO   | —           | FK → `Familia`                                                          |
| `IdRegion`                 | `uniqueidentifier` | NO   | —           | FK → `Region`                                                           |
| `IdPerfilFiscal`           | `uniqueidentifier` | NO   | —           | FK → `PerfilFiscal` — debe corresponder a la misma `IdRegion`           |
| `ClaveProdServSat`         | `varchar(10)`      | SÍ   | `NULL`      | Clave SAT c_ClaveProdServ. Solo MX. `NULL` = no facturable MX o fila PE |
| `ClaveUnidadSat`           | `varchar(10)`      | SÍ   | `NULL`      | Clave SAT c_ClaveUnidad: `E48`, `H87`, `ACT`. Solo MX                   |
| `Activo`                   | `bit`              | NO   | `1`         | Borrado lógico                                                          |
| `FechaRegistro`            | `datetime`         | NO   | `GETDATE()` | Fecha de alta                                                           |
| `FechaUltimaActualizacion` | `datetime`         | NO   | `GETDATE()` | Fecha de última modificación                                            |

**Índices:**
- `PK_FamiliaRegion` (Clustered): `IdFamiliaRegion`
- `UQ_FamiliaRegion_Familia_Region` (Unique): `(IdFamilia, IdRegion)` — una configuración por familia/región
- `IX_FamiliaRegion_IdFamilia` (Non-Clustered): `IdFamilia`
- `IX_FamiliaRegion_IdPerfilFiscal` (Non-Clustered): `IdPerfilFiscal`

**Relaciones:**
- `FK_FamiliaRegion_Familia` → `Familia.IdFamilia`
- `FK_FamiliaRegion_Region` → `Region.IdRegion`
- `FK_FamiliaRegion_PerfilFiscal` → `PerfilFiscal.IdPerfilFiscal`

**Consideraciones especiales:**
- La integridad de `IdPerfilFiscal.IdRegion = FamiliaRegion.IdRegion` se garantiza en la capa de aplicación (no hay FK compuesta en SQL Server sin tabla auxiliar).
- `ClaveProdServSat IS NULL` en fila MX → familia no facturable en México (Regla 7).
- Ausencia de fila para una región → familia sin configuración fiscal en esa región.

```sql
-- Ejecutar en ProquifaDotNet (requiere PerfilFiscal y Region ya creadas)
CREATE TABLE [dbo].[FamiliaRegion](
    [IdFamiliaRegion]   uniqueidentifier NOT NULL
        CONSTRAINT [DF_FamiliaRegion_Id] DEFAULT (NEWID()),
    [IdFamilia]         uniqueidentifier NOT NULL
        CONSTRAINT [FK_FamiliaRegion_Familia]
            FOREIGN KEY REFERENCES [dbo].[Familia]([IdFamilia]),
    [IdRegion]          uniqueidentifier NOT NULL
        CONSTRAINT [FK_FamiliaRegion_Region]
            FOREIGN KEY REFERENCES [dbo].[Region]([IdRegion]),
    [IdPerfilFiscal]    uniqueidentifier NOT NULL
        CONSTRAINT [FK_FamiliaRegion_PerfilFiscal]
            FOREIGN KEY REFERENCES [dbo].[PerfilFiscal]([IdPerfilFiscal]),
    [ClaveProdServSat]  varchar(10)  NULL,
    [ClaveUnidadSat]    varchar(10)  NULL,
    [Activo]            bit          NOT NULL CONSTRAINT [DF_FamiliaRegion_Activo] DEFAULT (1),
    [FechaRegistro]     datetime NOT NULL CONSTRAINT [DF_FamiliaRegion_FechaRegistro] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion] datetime NOT NULL CONSTRAINT [DF_DF_FamiliaRegion_FechaUltimaActualizacion] DEFAULT (GETDATE()),
    CONSTRAINT [PK_FamiliaRegion] PRIMARY KEY CLUSTERED ([IdFamiliaRegion]),
    CONSTRAINT [UQ_FamiliaRegion_Familia_Region] UNIQUE ([IdFamilia], [IdRegion])
);
GO

CREATE NONCLUSTERED INDEX [IX_FamiliaRegion_IdFamilia]
    ON [dbo].[FamiliaRegion] ([IdFamilia]);
GO

CREATE NONCLUSTERED INDEX [IX_FamiliaRegion_IdPerfilFiscal]
    ON [dbo].[FamiliaRegion] ([IdPerfilFiscal]);
GO

-- ==========================================================================
-- SEED — mapeo de familias según definición PROQUIFA
-- Verificar los nombres exactos de familia en BD antes de ejecutar.
-- ==========================================================================

-- Helper: GUIDs de región y perfiles
DECLARE @RegionMX uniqueidentifier = (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'mexico');
DECLARE @RegionPE uniqueidentifier = (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'peru');
DECLARE @PfIva16  uniqueidentifier = (SELECT IdPerfilFiscal FROM [dbo].[PerfilFiscal] WHERE IdRegion = @RegionMX AND IdCatImpuesto = (SELECT IdCatImpuesto FROM dbo.catImpuesto WHERE Clave = '002') AND TasaOCuota = 0.160000);
DECLARE @PfIva0   uniqueidentifier = (SELECT IdPerfilFiscal FROM [dbo].[PerfilFiscal] WHERE IdRegion = @RegionMX AND IdCatImpuesto = (SELECT IdCatImpuesto FROM dbo.catImpuesto WHERE Clave = '002') AND TasaOCuota = 0.000000);
DECLARE @PfIgv18  uniqueidentifier = (SELECT IdPerfilFiscal FROM [dbo].[PerfilFiscal] WHERE IdRegion = @RegionPE AND TasaOCuota = 0.180000);
DECLARE @PfIgv0   uniqueidentifier = (SELECT IdPerfilFiscal FROM [dbo].[PerfilFiscal] WHERE IdRegion = @RegionPE AND TasaOCuota = 0.000000);

-- Biológico
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal], [ClaveProdServSat], [ClaveUnidadSat])
SELECT IdFamilia, @RegionMX, @PfIva16, '41116132', 'H87' FROM [dbo].[Familia] WHERE Nombre = 'Biológico';
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal])
SELECT IdFamilia, @RegionPE, @PfIgv18 FROM [dbo].[Familia] WHERE Nombre = 'Biológico';

-- Estándares
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal], [ClaveProdServSat], [ClaveUnidadSat])
SELECT IdFamilia, @RegionMX, @PfIva16, '41116107', 'H87' FROM [dbo].[Familia] WHERE Nombre = 'Estándares';
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal])
SELECT IdFamilia, @RegionPE, @PfIgv18 FROM [dbo].[Familia] WHERE Nombre = 'Estándares';

-- Reactivos
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal], [ClaveProdServSat], [ClaveUnidadSat])
SELECT IdFamilia, @RegionMX, @PfIva16, '41116105', 'H87' FROM [dbo].[Familia] WHERE Nombre = 'Reactivos';
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal])
SELECT IdFamilia, @RegionPE, @PfIgv18 FROM [dbo].[Familia] WHERE Nombre = 'Reactivos';

-- Publicaciones: IVA 0% MX / IGV 0% PE (confirmar con negocio)
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal], [ClaveProdServSat], [ClaveUnidadSat])
SELECT IdFamilia, @RegionMX, @PfIva0, '55101500', 'H87' FROM [dbo].[Familia] WHERE Nombre = 'Publicaciones';
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal])
SELECT IdFamilia, @RegionPE, @PfIgv0 FROM [dbo].[Familia] WHERE Nombre = 'Publicaciones';

-- Capacitaciones
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal], [ClaveProdServSat], [ClaveUnidadSat])
SELECT IdFamilia, @RegionMX, @PfIva16, '86101600', 'E48' FROM [dbo].[Familia] WHERE Nombre = 'Capacitaciones';
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal])
SELECT IdFamilia, @RegionPE, @PfIgv18 FROM [dbo].[Familia] WHERE Nombre = 'Capacitaciones';

-- Labware
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal], [ClaveProdServSat], [ClaveUnidadSat])
SELECT IdFamilia, @RegionMX, @PfIva16, '41116100', 'H87' FROM [dbo].[Familia] WHERE Nombre = 'Labware';
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal])
SELECT IdFamilia, @RegionPE, @PfIgv18 FROM [dbo].[Familia] WHERE Nombre = 'Labware';

-- Fletes
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal], [ClaveProdServSat], [ClaveUnidadSat])
SELECT IdFamilia, @RegionMX, @PfIva16, '78102205', 'E48' FROM [dbo].[Familia] WHERE Nombre = 'Fletes';
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal])
SELECT IdFamilia, @RegionPE, @PfIgv18 FROM [dbo].[Familia] WHERE Nombre = 'Fletes';

-- Servicios
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal], [ClaveProdServSat], [ClaveUnidadSat])
SELECT IdFamilia, @RegionMX, @PfIva16, '85131701', 'H87' FROM [dbo].[Familia] WHERE Nombre = 'Servicios';
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal])
SELECT IdFamilia, @RegionPE, @PfIgv18 FROM [dbo].[Familia] WHERE Nombre = 'Servicios';

-- Factura Anticipo — solo MX (anticipo no aplica PE)
INSERT INTO [dbo].[FamiliaRegion] ([IdFamilia], [IdRegion], [IdPerfilFiscal], [ClaveProdServSat], [ClaveUnidadSat])
SELECT IdFamilia, @RegionMX, @PfIva16, '84111506', 'ACT'
FROM   [dbo].[Familia] WHERE Nombre IN ('Factura Anticipo', 'Partidas de factura anticipo');

-- Dispositivo médico: no se inserta ninguna fila (no facturable — Regla 7)
GO
```

> ⚠️ **Verificar los nombres exactos de familia en BD antes de ejecutar el seed.** El perfil PE de Publicaciones (`IGV 0%`) debe confirmarse con el área de negocio.

---

## Consultas de referencia

### Resolución fiscal de una partida por producto (con región)

```sql
-- Al armar una partida de documento fiscal, dado @IdProducto y @ClaveRegion ('mexico' | 'peru'):
DECLARE @IdRegion uniqueidentifier = (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = @ClaveRegion);

SELECT
    fr.ClaveProdServSat,
    fr.ClaveUnidadSat,
    imp.Clave                  AS CodigoImpuesto,   -- 'IVA' (MX) | 'IGV' (PE)
    pf.TasaOCuota,
    tf.Clave                   AS TipoFactor,
    oi.Clave                   AS ObjetoImpuestoSat -- NULL para PE
FROM       dbo.MarcaFamilia          mf
INNER JOIN dbo.FamiliaRegion         fr  ON fr.IdFamilia          = mf.IdFamilia
                                        AND fr.IdRegion            = @IdRegion
                                        AND fr.Activo              = 1
INNER JOIN dbo.PerfilFiscal          pf  ON pf.IdPerfilFiscal      = fr.IdPerfilFiscal
INNER JOIN dbo.catImpuesto           imp ON imp.IdCatImpuesto      = pf.IdCatImpuesto
INNER JOIN dbo.catTipoFactorSat      tf  ON tf.IdCatTipoFactorSat  = pf.IdCatTipoFactorSat
LEFT  JOIN dbo.catObjetoImpuestoSat  oi  ON oi.IdCatObjetoImpuestoSat = pf.IdCatObjetoImpuestoSat
WHERE mf.IdProducto = @IdProducto
  AND mf.Activo     = 1;
```

### Verificar familias sin configuración fiscal por región

```sql
-- Familias activas que no tienen fila en FamiliaRegion para cada región
SELECT f.IdFamilia, f.Nombre, r.Clave AS Region
FROM       dbo.Familia f
CROSS JOIN dbo.Region  r
WHERE f.Activo = 1
  AND r.Clave IN ('mexico', 'peru')
  AND f.Nombre <> 'Dispositivo médico'   -- no facturable intencionalmente
  AND NOT EXISTS (
        SELECT 1 FROM dbo.FamiliaRegion fr
        WHERE fr.IdFamilia = f.IdFamilia
          AND fr.IdRegion  = r.IdRegion
          AND fr.Activo    = 1
      )
ORDER BY r.Clave, f.Nombre;
```

---

## Impacto en requisitos dependientes

| Requisito | Impacto | Descripción |
|---|---|---|
| RE-FU-019 | **Desbloquea GAP-7 / GAP-8** | `catImpuesto`, `catTipoFactorSat`, `catObjetoImpuestoSat`, `PerfilFiscal`, `FamiliaRegion` ya no están en espera |
| RE-FU-018 | Depende de FamiliaRegion | Al armar cada partida del CFDI, resuelve `ClaveProdServSat`, `ClaveUnidadSat` e `IdPerfilFiscal` de `FamiliaRegion` (región MX) |
| RE-FU-025 | Depende de FamiliaRegion | Al calcular impuestos PE (proforma/pedido), resuelve `IdPerfilFiscal` de `FamiliaRegion` (región PE) → `TasaOCuota` IGV |
| RE-FU-021 | Parcialmente sustituido | `ClaveProdServSat` se define a nivel `FamiliaRegion` (este cambio), no a nivel `Producto`. `RE-021` queda ⏸ en espera respecto a catálogos a nivel producto |
| RE-FU-015 | `fccFacturaPartida` | `ClaveProductoServicioSAT` y `ClaveUnidadSAT` se resuelven de `FamiliaRegion` al crear la partida |

---

## Consideraciones especiales

| # | Consideración | Detalle |
|---|---|---|
| 1 | Sin fila FamiliaRegion → no facturable en esa región | Ausencia de fila para `(IdFamilia, IdRegion)` equivale a familia no configurable — Regla 7 |
| 2 | `ClaveProdServSat IS NULL` en fila MX → no facturable MX | Debe validarse en servicio antes de timbrar |
| 3 | Seed requiere verificación de nombres | Confirmar nombres exactos de familias en BD de producción antes del script |
| 4 | `catImpuesto`, `catTipoFactorSat`, `catObjetoImpuestoSat` no se modifican | Son catálogos maestros de solo lectura |
| 5 | Sin interfaz gráfica | Toda la gestión de `FamiliaRegion` es directamente en BD |
| 6 | IGV Publicaciones PE pendiente confirmar | Mapeo provisional `IGV 0%`; confirmar con negocio si aplica exonerado o inafecto |
