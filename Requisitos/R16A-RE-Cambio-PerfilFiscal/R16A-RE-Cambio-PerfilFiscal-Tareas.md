# Tareas BackEnd — R16A-RE-Cambio-PerfilFiscal

**Requisito:** Perfil Fiscal — Configuración fiscal por Familia de Producto (MX + PE)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8)
**Alcance de este documento:** Impacto en BD de ProquifaDotNet + Impacto en Back de ProquifaDotNet (Logic.Pqf.Catalogos / Logic.Pqf.Logistica). **No incluye** el impacto en ProquifaDotNet.Finanzas — ese se integra por separado.

---

> **Orden de ejecución obligatorio:** BD Tarea 1 (catálogos + tablas) → BD Tarea 2 (ALTER Familia/vistas) → Back Tarea 3 (Core.Pqf NuGet) → Back Tareas 4–9 (BOs de Logic.Pqf.Logistica).
>
> **Nomenclatura Factura por Adelantada:** el discriminador `ClaveTipoEntidad = 'factura-anticipo'` (con CHECK constraint en `_BD.md`) es el valor canónico y único en todo el cambio — confirmado. `Back.md`, `Back-Finanzas.md` y este documento ya usan `'factura-anticipo'` de forma consistente.

---

## Resumen de tareas

| #         | Clave             | Título simple                                                                                                                                        | Tipo | Aplicativo     | Complejidad | Horas      |
| --------- | ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | -------------- | ----------- | ---------- |
| 1         | CREATE-TABL-M     | T1 - Crear catálogos y tablas de Perfil Fiscal (catImpuesto, catTipoFactorSat, catObjetoImpuestoSat, PerfilFiscal, PerfilFiscalConfiguracionFamilia) | BD   | ProquifaDotNet | Media       | 42.00      |
| 2         | UPDATE-TABL-M     | T2 - ALTER TABLE Familia + ALTER VIEW vMarcaFamilia/vProducto/vFlete — campos fiscales                                                               | BD   | ProquifaDotNet | Media       | 28.00      |
| 3         | IMP-EXIST-SERVICE | T3 - Actualizar consumo de Core.Pqf (NuGet) en Logic.Pqf.Catalogos y Logic.Pqf.Logistica                                                             | Back | ProquifaDotNet | Baja        | 12.00      |
| 4         | IMP-EXIST-SERVICE | T4 - Migrar cálculo de IVA — L01 Cotización (Partidas y Fletes)                                                                                      | Back | ProquifaDotNet | Media       | 36.00      |
| 5         | IMP-EXIST-SERVICE | T5 - Migrar cálculo de IVA — L01 Cotización (Actualización y Cierre de Oferta)                                                                       | Back | ProquifaDotNet | Media       | 24.00      |
| 6         | IMP-EXIST-SERVICE | T6 - Migrar cálculo de IVA — L02 Ajustar Oferta y L03 Promesa de Compra                                                                              | Back | ProquifaDotNet | Baja        | 24.00      |
| 7         | IMP-EXIST-SERVICE | T7 - Migrar cálculo de IVA — L04 Pretramitar Pedido                                                                                                  | Back | ProquifaDotNet | Media       | 36.00      |
| 8         | IMP-EXIST-SERVICE | T8 - Migrar generación de Conceptos CFDI y cálculo de IVA — L05 Tramitar Pedido                                                                      | Back | ProquifaDotNet | Alta        | 36.00      |
| 9         | IMP-EXIST-SERVICE | T9 - Migrar cálculo de IVA en correo interno y PDF de confirmación de pedido                                                                         | Back | ProquifaDotNet | Baja        | 24.00      |
| **Total** |                   |                                                                                                                                                      |      |                |             | **262.00** |

> **Horas** = suma de las estimaciones base del `Catalogo BackEnd.md` por cada objeto/archivo agrupado en la tarea (p. ej. Tarea 1 = 3 × CREATE-TABL-CH de 6h + 2 × CREATE-TABL-M de 12h = 42h). Cada tarea se mantuvo ≤ 45h según lo solicitado.

---

## TAREA 1

**[ R16A-RE-Cambio-PerfilFiscal ] [CREATE-TABL-M] Crear catálogos y tablas de Perfil Fiscal (catImpuesto, catTipoFactorSat, catObjetoImpuestoSat, PerfilFiscal, PerfilFiscalConfiguracionFamilia)**

**Aplicativos:** ProquifaDotNet (BD)

**Módulos:** Base de Datos — Perfil Fiscal

**Consideraciones previas:**
- Las 5 tablas son nuevas — no existen en BD.
- Orden de ejecución obligatorio: `catImpuesto` → `catTipoFactorSat` → `catObjetoImpuestoSat` → `PerfilFiscal` → `PerfilFiscalConfiguracionFamilia` (cada una depende de las FK de la anterior).
- `PerfilFiscal` es tabla única para México (IVA) y Perú (IGV), discriminada por `IdRegion` — sin tablas separadas por país (decisión confirmada, ver memoria `perfil_fiscal_unificado_decision`).
- `PerfilFiscalConfiguracionFamilia` reemplaza el diseño previo `FamiliaRegion`; el discriminador `ClaveTipoEntidad` (`'familia'` | `'flete'` | `'factura-anticipo'`) permite que `IdFamilia` sea `NULL` para configuraciones de flete y factura-anticipo.
- El seed de `PerfilFiscalConfiguracionFamilia` requiere verificar los nombres exactos de familia en BD de producción antes de ejecutar.
- Es **prerrequisito** de la Tarea 2 (ALTER VIEW) y de la Tarea 3 (actualización de Core.Pqf).

**Diccionario de Datos:**

*Tablas incluidas:*

| Nombre de tabla | Descripción |
|---|---|
| `catImpuesto` | Catálogo compartido MX+PE de tipos de impuesto (ISR, IVA, IEPS, IGV) |
| `catTipoFactorSat` | Catálogo maestro SAT de tipos de factor (`Tasa`, `Cuota`, `Exento`), compartido MX+PE |
| `catObjetoImpuestoSat` | Catálogo maestro SAT de objetos de impuesto (`01`–`04`), exclusivo México |
| `PerfilFiscal` | Perfil de impuesto por región (MX/PE): tasa, tipo de factor y objeto de impuesto |
| `PerfilFiscalConfiguracionFamilia` | Configuración fiscal por tipo de entidad (familia / flete / factura-anticipo) + región |

*Columnas:*

| Tabla | Columna | Tipo | Nulo | Descripción |
|---|---|---|---|---|
| catImpuesto | IdCatImpuesto | uniqueidentifier | NO | PK, DEFAULT NEWID() |
| catImpuesto | Clave | varchar(10) | NO | `001` ISR, `002` IVA, `003` IEPS, `IGV` |
| catImpuesto | Descripcion | varchar(100) | NO | Descripción del impuesto |
| catImpuesto | Activo | bit | NO | Borrado lógico, DEFAULT 1 |
| catImpuesto | FechaRegistro | datetime | NO | DEFAULT GETDATE() |
| catImpuesto | FechaUltimaActualizacion | datetime | NO | DEFAULT GETDATE() |
| catTipoFactorSat | IdCatTipoFactorSat | uniqueidentifier | NO | PK, DEFAULT NEWID() |
| catTipoFactorSat | Clave | varchar(10) | NO | `Tasa`, `Cuota`, `Exento` |
| catTipoFactorSat | Descripcion | varchar(100) | NO | Descripción del factor |
| catTipoFactorSat | Activo | bit | NO | Borrado lógico, DEFAULT 1 |
| catTipoFactorSat | FechaRegistro | datetime | NO | DEFAULT GETDATE() |
| catTipoFactorSat | FechaUltimaActualizacion | datetime | NO | DEFAULT GETDATE() |
| catObjetoImpuestoSat | IdCatObjetoImpuestoSat | uniqueidentifier | NO | PK, DEFAULT NEWID() |
| catObjetoImpuestoSat | Clave | varchar(10) | NO | `01`–`04` |
| catObjetoImpuestoSat | Descripcion | varchar(200) | NO | Descripción del objeto de impuesto |
| catObjetoImpuestoSat | Activo | bit | NO | Borrado lógico, DEFAULT 1 |
| catObjetoImpuestoSat | FechaRegistro | datetime | NO | DEFAULT GETDATE() |
| catObjetoImpuestoSat | FechaUltimaActualizacion | datetime | NO | DEFAULT GETDATE() |
| PerfilFiscal | IdPerfilFiscal | uniqueidentifier | NO | PK, DEFAULT NEWID() |
| PerfilFiscal | IdRegion | uniqueidentifier | NO | FK → Region — discrimina MX vs PE |
| PerfilFiscal | IdCatImpuesto | uniqueidentifier | NO | FK → catImpuesto |
| PerfilFiscal | IdCatTipoFactorSat | uniqueidentifier | NO | FK → catTipoFactorSat |
| PerfilFiscal | TasaOCuota | decimal(6,6) | SÍ | NULL solo si TipoFactor = Exento |
| PerfilFiscal | IdCatObjetoImpuestoSat | uniqueidentifier | SÍ | FK → catObjetoImpuestoSat; NULL para PE |
| PerfilFiscal | Fundamento | nvarchar(200) | SÍ | Referencia legal |
| PerfilFiscal | Activo | bit | NO | Borrado lógico, DEFAULT 1 |
| PerfilFiscal | FechaRegistro | datetime | NO | DEFAULT GETDATE() |
| PerfilFiscal | FechaUltimaActualizacion | datetime | NO | DEFAULT GETDATE() |
| PerfilFiscalConfiguracionFamilia | IdPerfilFiscalConfiguracionFamilia | uniqueidentifier | NO | PK, DEFAULT NEWID() |
| PerfilFiscalConfiguracionFamilia | IdFamilia | uniqueidentifier | SÍ | FK → Familia; NULL para flete/factura-anticipo |
| PerfilFiscalConfiguracionFamilia | IdRegion | uniqueidentifier | NO | FK → Region |
| PerfilFiscalConfiguracionFamilia | IdPerfilFiscal | uniqueidentifier | NO | FK → PerfilFiscal |
| PerfilFiscalConfiguracionFamilia | ClaveTipoEntidad | varchar(30) | NO | `familia` \| `flete` \| `factura-anticipo` |
| PerfilFiscalConfiguracionFamilia | ClaveProdServSat | varchar(10) | SÍ | Solo MX |
| PerfilFiscalConfiguracionFamilia | ClaveUnidadSat | varchar(10) | SÍ | Solo MX |
| PerfilFiscalConfiguracionFamilia | Activo | bit | NO | Borrado lógico, DEFAULT 1 |
| PerfilFiscalConfiguracionFamilia | FechaRegistro | datetime | NO | DEFAULT GETDATE() |
| PerfilFiscalConfiguracionFamilia | FechaUltimaActualizacion | datetime | NO | DEFAULT GETDATE() |

*Relaciones:*
- `PerfilFiscal.IdRegion` → `Region.IdRegion`
- `PerfilFiscal.IdCatImpuesto` → `catImpuesto.IdCatImpuesto`
- `PerfilFiscal.IdCatTipoFactorSat` → `catTipoFactorSat.IdCatTipoFactorSat`
- `PerfilFiscal.IdCatObjetoImpuestoSat` → `catObjetoImpuestoSat.IdCatObjetoImpuestoSat` (nullable)
- `PerfilFiscalConfiguracionFamilia.IdFamilia` → `Familia.IdFamilia` (nullable)
- `PerfilFiscalConfiguracionFamilia.IdRegion` → `Region.IdRegion`
- `PerfilFiscalConfiguracionFamilia.IdPerfilFiscal` → `PerfilFiscal.IdPerfilFiscal`

*Índices:*
- `PK_catImpuesto` (Clustered): `IdCatImpuesto` · `UQ_catImpuesto_Clave` (Unique): `Clave`
- `PK_catTipoFactorSat` (Clustered): `IdCatTipoFactorSat` · `UQ_catTipoFactorSat_Clave` (Unique): `Clave`
- `PK_catObjetoImpuestoSat` (Clustered): `IdCatObjetoImpuestoSat` · `UQ_catObjetoImpuestoSat_Clave` (Unique): `Clave`
- `PK_PerfilFiscal` (Clustered): `IdPerfilFiscal` · `IX_PerfilFiscal_IdRegion` (Non-Clustered): `IdRegion`
- `PK_PerfilFiscalConfiguracionFamilia` (Clustered): `IdPerfilFiscalConfiguracionFamilia`
- `IX_PfcFamilia_IdFamilia_IdRegion` (Non-Clustered): `(IdFamilia, IdRegion)`
- `IX_PfcFamilia_ClaveTipoEntidad_IdRegion` (Non-Clustered): `(ClaveTipoEntidad, IdRegion)`

*Consideraciones especiales:*
- `CK_PerfilFiscal_TasaOCuota`: `TasaOCuota` es `NULL` si y solo si `catTipoFactorSat.Clave = 'Exento'`.
- `CK_PfcFamilia_ClaveTipoEntidad`: `ClaveTipoEntidad IN ('familia', 'flete', 'factura-anticipo')`.
- Filas PE en `PerfilFiscal`: `IdCatObjetoImpuestoSat = NULL` (no aplica SAT).
- `PerfilFiscalConfiguracionFamilia.IdFamilia` es `NULL` cuando `ClaveTipoEntidad` = `'flete'` o `'factura-anticipo'`.
- Sin interfaz gráfica — toda la gestión es directamente en BD.

**Objetivo general:**
Crear en ProquifaDotNet las 5 tablas nuevas del dominio de Perfil Fiscal, con su seed de catálogos, perfiles MX/PE y configuración fiscal inicial por familia/flete/factura-anticipo.

**Objetivos específicos:**
- Ejecutar `CREATE TABLE catImpuesto` + `INSERT` (ISR, IVA, IEPS, IGV).
- Ejecutar `CREATE TABLE catTipoFactorSat` + `INSERT` (Tasa, Cuota, Exento).
- Ejecutar `CREATE TABLE catObjetoImpuestoSat` + `INSERT` (01–04).
- Ejecutar `CREATE TABLE PerfilFiscal` con CHECK `CK_PerfilFiscal_TasaOCuota` + seed de 3 filas MX + 3 filas PE.
- Ejecutar `CREATE TABLE PerfilFiscalConfiguracionFamilia` con CHECK `CK_PfcFamilia_ClaveTipoEntidad` + seed por familia (Biológico, Estándares, Reactivos, Publicaciones, Capacitaciones, Labware, Servicios) + flete + factura-anticipo.
- Verificar nombres exactos de familia en BD de producción antes de ejecutar el seed.
- Ejecutar consulta de verificación: familias activas sin configuración fiscal por región (sección "Consultas de referencia" de `_BD.md`).

**Scripts:**

```sql
-- =========================================================
-- 1. catImpuesto
-- =========================================================
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

-- =========================================================
-- 2. catTipoFactorSat
-- =========================================================
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

-- =========================================================
-- 3. catObjetoImpuestoSat
-- =========================================================
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

-- =========================================================
-- 4. PerfilFiscal (requiere catImpuesto, catTipoFactorSat, catObjetoImpuestoSat, Region)
-- =========================================================
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

-- SEED México (3 filas)
INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'mexico'),
       (SELECT IdCatImpuesto FROM [dbo].[catImpuesto] WHERE Clave = '002'),
       (SELECT IdCatTipoFactorSat FROM [dbo].[catTipoFactorSat] WHERE Clave = 'Tasa'),
       0.160000,
       (SELECT IdCatObjetoImpuestoSat FROM [dbo].[catObjetoImpuestoSat] WHERE Clave = '02'),
       'Art. 1 LIVA';

INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'mexico'),
       (SELECT IdCatImpuesto FROM [dbo].[catImpuesto] WHERE Clave = '002'),
       (SELECT IdCatTipoFactorSat FROM [dbo].[catTipoFactorSat] WHERE Clave = 'Tasa'),
       0.000000,
       (SELECT IdCatObjetoImpuestoSat FROM [dbo].[catObjetoImpuestoSat] WHERE Clave = '02'),
       'Art. 2-A LIVA';

INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'mexico'),
       (SELECT IdCatImpuesto FROM [dbo].[catImpuesto] WHERE Clave = '002'),
       (SELECT IdCatTipoFactorSat FROM [dbo].[catTipoFactorSat] WHERE Clave = 'Exento'),
       NULL,
       (SELECT IdCatObjetoImpuestoSat FROM [dbo].[catObjetoImpuestoSat] WHERE Clave = '02'),
       'Art. 9 LIVA';

-- SEED Perú (3 filas — IdCatObjetoImpuestoSat = NULL)
INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'peru'),
       (SELECT IdCatImpuesto FROM [dbo].[catImpuesto] WHERE Clave = 'IGV'),
       (SELECT IdCatTipoFactorSat FROM [dbo].[catTipoFactorSat] WHERE Clave = 'Tasa'),
       0.180000, NULL, 'TUO IGV Art. 1 (D.S. 055-99-EF)';

INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'peru'),
       (SELECT IdCatImpuesto FROM [dbo].[catImpuesto] WHERE Clave = 'IGV'),
       (SELECT IdCatTipoFactorSat FROM [dbo].[catTipoFactorSat] WHERE Clave = 'Tasa'),
       0.000000, NULL, 'TUO IGV Art. 19 — Exonerado/Inafecto';

INSERT INTO [dbo].[PerfilFiscal]
    ([IdRegion], [IdCatImpuesto], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
SELECT (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'peru'),
       (SELECT IdCatImpuesto FROM [dbo].[catImpuesto] WHERE Clave = 'IGV'),
       (SELECT IdCatTipoFactorSat FROM [dbo].[catTipoFactorSat] WHERE Clave = 'Exento'),
       NULL, NULL, 'TUO IGV — Operación no gravada';
GO

-- =========================================================
-- 5. PerfilFiscalConfiguracionFamilia (requiere PerfilFiscal, Region)
-- =========================================================
CREATE TABLE [dbo].[PerfilFiscalConfiguracionFamilia](
    [IdPerfilFiscalConfiguracionFamilia] uniqueidentifier NOT NULL
        CONSTRAINT [DF_PfcFamilia_Id] DEFAULT (NEWID()),
    [IdFamilia]       uniqueidentifier NULL
        CONSTRAINT [FK_PfcFamilia_Familia]
            FOREIGN KEY REFERENCES [dbo].[Familia]([IdFamilia]),
    [IdRegion]        uniqueidentifier NOT NULL
        CONSTRAINT [FK_PfcFamilia_Region]
            FOREIGN KEY REFERENCES [dbo].[Region]([IdRegion]),
    [IdPerfilFiscal]  uniqueidentifier NOT NULL
        CONSTRAINT [FK_PfcFamilia_PerfilFiscal]
            FOREIGN KEY REFERENCES [dbo].[PerfilFiscal]([IdPerfilFiscal]),
    [ClaveTipoEntidad] varchar(30) NOT NULL
        CONSTRAINT [CK_PfcFamilia_ClaveTipoEntidad]
            CHECK ([ClaveTipoEntidad] IN ('familia', 'flete', 'factura-anticipo')),
    [ClaveProdServSat] varchar(10) NULL,
    [ClaveUnidadSat]   varchar(10) NULL,
    [Activo]           bit         NOT NULL CONSTRAINT [DF_PfcFamilia_Activo] DEFAULT (1),
    [FechaRegistro]    datetime    NOT NULL CONSTRAINT [DF_PfcFamilia_FechaRegistro] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion] datetime NOT NULL CONSTRAINT [DF_PfcFamilia_FechaUltimaActualizacion] DEFAULT (GETDATE()),
    CONSTRAINT [PK_PerfilFiscalConfiguracionFamilia]
        PRIMARY KEY CLUSTERED ([IdPerfilFiscalConfiguracionFamilia])
);
GO

CREATE NONCLUSTERED INDEX [IX_PfcFamilia_IdFamilia_IdRegion]
    ON [dbo].[PerfilFiscalConfiguracionFamilia] ([IdFamilia], [IdRegion]);
GO

CREATE NONCLUSTERED INDEX [IX_PfcFamilia_ClaveTipoEntidad_IdRegion]
    ON [dbo].[PerfilFiscalConfiguracionFamilia] ([ClaveTipoEntidad], [IdRegion]);
GO

-- SEED — verificar nombres exactos de familia en BD antes de ejecutar
DECLARE @RegionMX uniqueidentifier = (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'mexico');
DECLARE @RegionPE uniqueidentifier = (SELECT IdRegion FROM [dbo].[Region] WHERE Clave = 'peru');
DECLARE @PfIva16  uniqueidentifier = (SELECT IdPerfilFiscal FROM [dbo].[PerfilFiscal] WHERE IdRegion = @RegionMX AND TasaOCuota = 0.160000);
DECLARE @PfIva0   uniqueidentifier = (SELECT IdPerfilFiscal FROM [dbo].[PerfilFiscal] WHERE IdRegion = @RegionMX AND TasaOCuota = 0.000000);
DECLARE @PfIgv18  uniqueidentifier = (SELECT IdPerfilFiscal FROM [dbo].[PerfilFiscal] WHERE IdRegion = @RegionPE AND TasaOCuota = 0.180000);
DECLARE @PfIgv0   uniqueidentifier = (SELECT IdPerfilFiscal FROM [dbo].[PerfilFiscal] WHERE IdRegion = @RegionPE AND TasaOCuota = 0.000000);

-- Familias (Biológico, Estándares, Reactivos, Publicaciones, Capacitaciones, Labware, Servicios)
-- + Flete (IdFamilia NULL) + Factura Anticipo (IdFamilia NULL, solo MX)
-- Ver script completo por familia en R16A-RE-Cambio-PerfilFiscal_BD.md, sección 5.
GO
```

**Resultado esperado:**
Las 5 tablas del dominio de Perfil Fiscal existen en ProquifaDotNet con sus catálogos, perfiles MX/PE y configuración fiscal inicial por familia, flete y factura-anticipo.

**Entregables:**
- Script DDL + DML: `catImpuesto`, `catTipoFactorSat`, `catObjetoImpuestoSat`
- Script DDL + DML: `PerfilFiscal` (seed 3 MX + 3 PE)
- Script DDL + DML: `PerfilFiscalConfiguracionFamilia` (seed por familia + flete + factura-anticipo)
- Script de validación (familias activas sin configuración fiscal por región)

**Criterios de aceptación:**
- Las 5 tablas existen con la estructura definida en `R16A-RE-Cambio-PerfilFiscal_BD.md`.
- `PerfilFiscal` contiene 3 filas MX + 3 filas PE; filas PE con `IdCatObjetoImpuestoSat = NULL`.
- El CHECK `CK_PerfilFiscal_TasaOCuota` rechaza inserciones inconsistentes (Exento con tasa, o Tasa sin valor).
- `PerfilFiscalConfiguracionFamilia` tiene fila por cada familia facturable (MX + PE) + 1 fila de flete MX + 1 fila PE + 1 fila de factura-anticipo (solo MX).
- La consulta de verificación no retorna familias activas sin configuración fiscal, salvo las intencionalmente no facturables (ej. Dispositivo médico).

**Más información de la tarea:**
Este cambio desbloquea el bloque **GAP-7 / GAP-8** de `R16A-RE-FU-019_BD.md`. Ver diagrama ER en `R16A-RE-Cambio-PerfilFiscal_BD.md` sección "Modelo de Datos".

**Recursos:**
- `R16A-RE-Cambio-PerfilFiscal_BD.md` — secciones 1 a 5 (DDL/DML completo)
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — sección 2 (Prerrequisito Core.Pqf)

---

## TAREA 2

**[ R16A-RE-Cambio-PerfilFiscal ] [UPDATE-TABL-M] ALTER TABLE Familia + ALTER VIEW vMarcaFamilia/vProducto/vFlete — campos fiscales**

**Aplicativos:** ProquifaDotNet (BD)

**Módulos:** Base de Datos — Perfil Fiscal

**Consideraciones previas:**
- Depende de la Tarea 1 (`PerfilFiscal` y `PerfilFiscalConfiguracionFamilia` deben existir antes de estos ALTER).
- Los 3 ALTER VIEW agregan los mismos 4 campos fiscales (`ClaveProdServSat`, `ClaveUnidadSat`, `TasaOCuota`, `TipoFactor`), cada uno con su propio JOIN según el tipo de entidad.
- Todos los JOIN son `LEFT JOIN` — familias/productos/fletes sin configuración fiscal quedan con los 4 campos en `NULL` (no facturables, Regla 7).
- `vProducto` es una vista extensa (100+ columnas); el ALTER solo agrega las líneas de JOIN y columnas indicadas — no se reescribe la vista completa en este documento (ver `_BD.md` sección 8).
- Es **prerrequisito** de la Tarea 3 (actualización de Core.Pqf) — las entidades EDMX no exponen los campos nuevos hasta que las vistas los tengan.

**Diccionario de Datos:**

*Objetos incluidos:*

| Nombre | Tipo | Descripción |
|---|---|---|
| `Familia` | Tabla existente | ALTER — agrega `ClaveProductoServicioCFDI` |
| `vMarcaFamilia` | Vista existente | ALTER — agrega 4 campos fiscales vía JOIN a `PerfilFiscalConfiguracionFamilia` (`ClaveTipoEntidad='familia'`) |
| `vProducto` | Vista existente | ALTER — agrega los mismos 4 campos fiscales, JOIN por `ProductoMarcaFamilia.IdRegion` |
| `vFlete` | Vista existente | ALTER — agrega los mismos 4 campos fiscales, JOIN por `Flete.IdRegion` (`ClaveTipoEntidad='flete'`) |

*Columnas nuevas:*

| Objeto | Columna | Tipo | Descripción |
|---|---|---|---|
| Familia | ClaveProductoServicioCFDI | varchar(10) NULL | Clave SAT c_ClaveProdServ a nivel familia |
| vMarcaFamilia / vProducto / vFlete | ClaveProdServSat | varchar(10) | Clave SAT c_ClaveProdServ resuelta (solo MX) |
| vMarcaFamilia / vProducto / vFlete | ClaveUnidadSat | varchar(10) | Clave SAT c_ClaveUnidad resuelta (solo MX) |
| vMarcaFamilia / vProducto / vFlete | TasaOCuota | decimal(6,6) | Tasa del impuesto resuelta desde `PerfilFiscal` |
| vMarcaFamilia / vProducto / vFlete | TipoFactor | varchar(10) | `catTipoFactorSat.Clave`: Tasa/Cuota/Exento |

*Relaciones (JOINs agregados):*
- `vMarcaFamilia`: `LEFT JOIN PerfilFiscalConfiguracionFamilia` ON `IdFamilia` + `IdRegion` + `ClaveTipoEntidad='familia'`
- `vProducto`: `LEFT JOIN PerfilFiscalConfiguracionFamilia` ON `MarcaFamilia.IdFamilia` + `ProductoMarcaFamilia.IdRegion` + `ClaveTipoEntidad='familia'`
- `vFlete`: `LEFT JOIN PerfilFiscalConfiguracionFamilia` ON `IdFamilia IS NULL` + `Flete.IdRegion` + `ClaveTipoEntidad='flete'`
- Los 3 JOIN encadenan hacia `PerfilFiscal.IdPerfilFiscal` y `catTipoFactorSat.IdCatTipoFactorSat`

*Índices:* sin índices nuevos — las vistas no son indexadas; `Familia` no requiere índice adicional para 1 columna nullable.

*Consideraciones especiales:*
- `Familia.ClaveProductoServicioCFDI` se agrega `NULL` para no afectar filas existentes; poblar vía `UPDATE` después del seed de `PerfilFiscalConfiguracionFamilia` (Tarea 1).
- Verificar que ningún SP, vista o trigger dependiente de `vMarcaFamilia`/`vProducto`/`vFlete` se rompe tras el ALTER.

**Objetivo general:**
Extender `Familia` y las vistas `vMarcaFamilia`, `vProducto`, `vFlete` para exponer los campos fiscales resueltos desde `PerfilFiscalConfiguracionFamilia`.

**Objetivos específicos:**
- `ALTER TABLE Familia ADD ClaveProductoServicioCFDI varchar(10) NULL`.
- `ALTER VIEW vMarcaFamilia` — agregar JOIN a `PerfilFiscalConfiguracionFamilia`/`PerfilFiscal`/`catTipoFactorSat` (`ClaveTipoEntidad='familia'`) + columnas `ClaveProdServSat`, `ClaveUnidadSat`, `TasaOCuota`, `TipoFactor`.
- `ALTER VIEW vProducto` — mismo patrón, usando `pmf.IdRegion` (alias de `ProductoMarcaFamilia`) como discriminador de región.
- `ALTER VIEW vFlete` — mismo patrón, usando `Flete.IdRegion` y `ClaveTipoEntidad='flete'`, `IdFamilia IS NULL`.
- Verificar compilación de SPs/vistas/triggers dependientes tras cada ALTER.

**Scripts:**

```sql
-- 1. Familia — ALTER TABLE
ALTER TABLE [dbo].[Familia]
    ADD [ClaveProductoServicioCFDI] varchar(10) NULL;
GO

-- 2. vMarcaFamilia — ALTER VIEW (agregar al FROM y SELECT existentes)
-- JOIN:
--   LEFT JOIN PerfilFiscalConfiguracionFamilia pfc
--       ON  pfc.IdFamilia        = mf.IdFamilia
--       AND pfc.IdRegion         = mf.IdRegion
--       AND pfc.ClaveTipoEntidad = 'familia'
--       AND pfc.Activo           = 1
--   LEFT JOIN PerfilFiscal      pf2 ON pf2.IdPerfilFiscal     = pfc.IdPerfilFiscal
--   LEFT JOIN catTipoFactorSat  ctf ON ctf.IdCatTipoFactorSat = pf2.IdCatTipoFactorSat
-- SELECT adicional:
--   pfc.ClaveProdServSat, pfc.ClaveUnidadSat, pf2.TasaOCuota, ctf.Clave AS TipoFactor
-- Ver script ALTER VIEW completo en R16A-RE-Cambio-PerfilFiscal_BD.md sección 7.

-- 3. vProducto — ALTER VIEW (agregar al FROM y SELECT existentes, alias pmf = ProductoMarcaFamilia)
-- JOIN:
--   LEFT JOIN PerfilFiscalConfiguracionFamilia pfc
--       ON  pfc.IdFamilia        = dbo.MarcaFamilia.IdFamilia
--       AND pfc.IdRegion         = pmf.IdRegion
--       AND pfc.ClaveTipoEntidad = 'familia'
--       AND pfc.Activo           = 1
--   LEFT JOIN PerfilFiscal     pf2 ON pf2.IdPerfilFiscal     = pfc.IdPerfilFiscal
--   LEFT JOIN catTipoFactorSat ctf ON ctf.IdCatTipoFactorSat = pf2.IdCatTipoFactorSat
-- SELECT adicional: pfc.ClaveProdServSat, pfc.ClaveUnidadSat, pf2.TasaOCuota, ctf.Clave AS TipoFactor
-- Vista completa omitida por extensión (100+ columnas) — ver R16A-RE-Cambio-PerfilFiscal_BD.md sección 8.

-- 4. vFlete — ALTER VIEW completo
ALTER VIEW [dbo].[vFlete]
AS
    SELECT
        F.*,
        r.Nombre,
        r.ClaveISO,
        catUF.UnidadFlete,
        catF.Fletera,
        catF.Clave          AS ClaveFletera,
        catM.Moneda,
        catM.ClaveMoneda,
        -- Campos fiscales (R16A-RE-Cambio-PerfilFiscal)
        pfc.ClaveProdServSat,
        pfc.ClaveUnidadSat,
        pf2.TasaOCuota,
        ctf.Clave           AS TipoFactor
    FROM Flete F
        INNER JOIN catUnidadFlete catUF ON F.IdCatUnidadFlete = catUF.IdCatUnidadFlete
        INNER JOIN catFletera     catF  ON catF.IdCatFletera  = F.IdCatFletera
        INNER JOIN catMoneda      catM  ON catM.IdCatMoneda   = F.IdCatMoneda
        INNER JOIN Region         r     ON r.IdRegion         = F.IdRegion
        LEFT  JOIN PerfilFiscalConfiguracionFamilia pfc
                                         ON  pfc.IdFamilia        IS NULL
                                         AND pfc.IdRegion         = F.IdRegion
                                         AND pfc.ClaveTipoEntidad = 'flete'
                                         AND pfc.Activo           = 1
        LEFT  JOIN PerfilFiscal     pf2  ON pf2.IdPerfilFiscal     = pfc.IdPerfilFiscal
        LEFT  JOIN catTipoFactorSat ctf  ON ctf.IdCatTipoFactorSat = pf2.IdCatTipoFactorSat;
GO
```

**Resultado esperado:**
`Familia`, `vMarcaFamilia`, `vProducto` y `vFlete` exponen los campos fiscales resueltos, listos para ser consumidos por Core.Pqf (Tarea 3) y las entidades derivadas (`cotProductoOferta`, `tpPartidaPedido`, `vFleteObj`, etc.).

**Entregables:**
- Script DDL: `ALTER TABLE Familia ADD ClaveProductoServicioCFDI`
- Script DDL: `ALTER VIEW vMarcaFamilia` (con JOIN fiscal)
- Script DDL: `ALTER VIEW vProducto` (con JOIN fiscal)
- Script DDL: `ALTER VIEW vFlete` (con JOIN fiscal, completo)
- Script de validación (estructura de columnas + smoke test de las 3 vistas)

**Criterios de aceptación:**
- Las 3 vistas retornan los 4 campos fiscales nuevos sin error.
- Familias/productos/fletes sin configuración fiscal devuelven los 4 campos en `NULL` (no rompen la consulta).
- `Familia.ClaveProductoServicioCFDI` existe como columna nullable sin afectar filas existentes.
- Ningún SP, vista o trigger dependiente de los 4 objetos altera su comportamiento previo.

**Más información de la tarea:**
Ver "Entidades consumidoras de campos fiscales" en `R16A-RE-Cambio-PerfilFiscal_BD.md` sección 10 para el mapeo completo vista → entidad.

**Recursos:**
- `R16A-RE-Cambio-PerfilFiscal_BD.md` — secciones 6, 7, 8, 9
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — sección 1 (Arquitectura), sección 4 (Entidades)

---

## TAREA 3

**[ R16A-RE-Cambio-PerfilFiscal ] [IMP-EXIST-SERVICE] Actualizar consumo de Core.Pqf (NuGet) en Logic.Pqf.Catalogos y Logic.Pqf.Logistica**

**Aplicativos:** ProquifaDotNet

**Módulos:** Core.Pqf (dependencia NuGet) — Logic.Pqf.Catalogos, Logic.Pqf.Logistica

**Consideraciones previas:**
- Depende de la Tarea 2 (las vistas deben exponer los 4 campos fiscales antes de regenerar el EDMX).
- `Core.Pqf` es un paquete NuGet externo (repo separado) — no se edita directamente en el repositorio de ProquifaDotNet.
- Las entidades `vProducto`, `vFlete`, `cotProductoOferta`, `tpPartidaPedido`, etc. son clases EDMX-generadas dentro de `Core.Pqf.ProquifaDotNetContext`.
- Los modelos extendidos (`vFleteObj : vFlete`, etc.) usan `AttributeCopycat<TSource, TDest>.CopyAttributes()` (de `Core.CrudTools`) — los campos nuevos de la entidad base se propagan automáticamente sin cambios de código en el modelo extendido.
- **No se crean Controllers ni endpoints nuevos** — este cambio es exclusivamente de actualización de dependencia.
- Es **prerrequisito** de las Tareas 4 a 8 (todas consumen `TipoFactor`/`TasaOCuota`/`ClaveProdServSat`/`ClaveUnidadSat` desde las entidades de Core.Pqf).

**Objetivo general:**
Regenerar el EDMX de Core.Pqf contra la BD actualizada (Tarea 2), publicar la nueva versión del NuGet y actualizarla en ProquifaDotNet para que las entidades expongan los 4 campos fiscales nuevos.

**Objetivos específicos:**
- En el repo de Core.Pqf: regenerar EDMX (Update Model from Database) contra `vProducto` y `vFlete` ya alterados.
- Publicar nueva versión del NuGet `Core.Pqf`.
- En ProquifaDotNet: ejecutar `ActualizarNugget_Core.Pqf.bat <nueva-version>`.
- Verificar que `vProducto`, `vFlete` y las entidades derivadas (`cotProductoOferta`, `tpPartidaPedido`, etc.) exponen `TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat` tras la actualización.
- Verificar que `vFleteObj` y modelos extendidos similares heredan los 4 campos nuevos sin cambio de código (validar `AttributeCopycat`).
- Confirmar que `Logic.Pqf.Catalogos` y `Logic.Pqf.Logistica` compilan sin errores tras la actualización del paquete.

**Resultado esperado:**
Core.Pqf actualizado y consumido por ProquifaDotNet; todas las entidades de partidas y fletes exponen los 4 campos fiscales, listas para ser consumidas por los BOs de Logic.Pqf.Logistica.

**Entregables:**
- Nueva versión publicada del NuGet `Core.Pqf`
- Referencia de paquete actualizada en `Logic.Pqf.Catalogos.csproj` y `Logic.Pqf.Logistica.csproj` (y demás proyectos consumidores)
- Evidencia de compilación exitosa de la solución completa

**Criterios de aceptación:**
- `vProducto` y `vFlete` (Core.Pqf) exponen los 4 campos fiscales nuevos.
- Los modelos extendidos (`vFleteObj`, `cotProductoOfertaObj`, etc.) heredan los campos sin requerir cambios de código.
- La solución ProquifaDotNet compila sin errores tras `ActualizarNugget_Core.Pqf.bat`.
- No se agregó ningún Controller ni endpoint nuevo como parte de esta tarea.

**Más información de la tarea:**
Ver "Arquitectura de ProquifaDotNet" y "Prerrequisito — Actualizar Core.Pqf (NuGet)" en `R16A-RE-Cambio-PerfilFiscal-Back.md` secciones 1 y 2.

**Recursos:**
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — secciones 1, 2, 4
- `R16A-RE-Cambio-PerfilFiscal_BD.md` — secciones 7, 8, 9 (vistas alteradas)
- `ActualizarNugget_Core.Pqf.bat` (raíz del repositorio ProquifaDotNet)

---

## TAREA 4

**[ R16A-RE-Cambio-PerfilFiscal ] [IMP-EXIST-SERVICE] Migrar cálculo de IVA — L01 Cotización (Partidas y Fletes)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Logic.Pqf.Logistica — L01.Cotizacion (Partidas, Fletes)

**Consideraciones previas:**
- Depende de la Tarea 3 (Core.Pqf actualizado).
- Patrón de migración: `region.Impuesto` + `producto.GravaIVA` (bool) → `vProducto.TasaOCuota` + `vProducto.TipoFactor != "Exento"` (mismo patrón para `vFlete`).
- El lookup a la tabla `Region` para obtener `Impuesto` desaparece de estos 3 archivos.
- `GravaIVA` no se elimina de las entidades (permanece por compatibilidad) — solo deja de ser la fuente de verdad para el cálculo del monto de IVA.

**Objetivo general:**
Migrar el cálculo de IVA de partidas y fletes de cotización del patrón `region.Impuesto`/`GravaIVA` al patrón `TasaOCuota`/`TipoFactor`.

**Objetivos específicos:**
- `L01.Cotizacion/Partidas/Adapters/RecalcularProductoOfertaBO.cs`: `gravaIVAPorRegionCliente = region.Impuesto > 0` → `gravaIVA = cotProductoOferta.TipoFactor != "Exento"`; `valorPorcentajeIVA = cotProductoOferta.TasaOCuota ?? 0`; eliminar lookup a `Region`.
- `L01.Cotizacion/Partidas/Desgloses/cotProductoOfertaBO.cs`: `producto.GravaIVA ? ... * region.Impuesto : 0` → `gravaIVA ? ... * (cotProductoOferta.TasaOCuota ?? 0) : 0`.
- `L01.Cotizacion/Fletes/cotCotizacionFleteExpressBO.Extensions.cs`: `impuesto = regionBo.ObtenerRegionPorDefecto(idRegion)?.Impuesto` → `impuesto = cotFlete.TasaOCuota ?? 0`; `GravaIVA` → `cotFlete.TipoFactor != "Exento"`.

**Resultado esperado:**
Las partidas y fletes de cotización calculan el IVA usando `TasaOCuota`/`TipoFactor` de la vista, sin depender de `Region.Impuesto`.

**Entregables:**
- `RecalcularProductoOfertaBO.cs` modificado
- `cotProductoOfertaBO.cs` (Desgloses) modificado
- `cotCotizacionFleteExpressBO.Extensions.cs` modificado

**Criterios de aceptación:**
- El cálculo de IVA de partidas usa `TasaOCuota`/`TipoFactor` en los 3 archivos — sin referencias a `region.Impuesto`.
- Un producto/flete con `TipoFactor = "Exento"` calcula IVA = 0.
- Las pruebas de regresión de cotización (partidas + flete) no muestran diferencias contra los perfiles fiscales vigentes (IVA 16%, IVA 0%, Exento).

**Más información de la tarea:**
Ver tabla "L01 — Cotización" en `R16A-RE-Cambio-PerfilFiscal-Back.md` sección 5.

**Recursos:**
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — secciones 3 (Patrón), 5 (L01)

---

## TAREA 5

**[ R16A-RE-Cambio-PerfilFiscal ] [IMP-EXIST-SERVICE] Migrar cálculo de IVA — L01 Cotización (Actualización y Cierre de Oferta)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Logic.Pqf.Logistica — L01.Cotizacion (Actualizacion, CerrarOferta)

**Consideraciones previas:**
- Depende de la Tarea 3 (Core.Pqf actualizado).
- Continuación de la Tarea 4 sobre el mismo módulo L01 — separada para mantener cada tarea ≤ 45h.
- `SolicitarAjustesCerrarOfertaTransaccionBO.cs` reemplaza tanto la bandera `GravaIVAFleteExpress` como el cálculo de monto de flete.

**Objetivo general:**
Migrar el cálculo de IVA en la actualización de cotización y el cierre de oferta del patrón `region.Impuesto`/`GravaIVA` al patrón `TasaOCuota`/`TipoFactor`.

**Objetivos específicos:**
- `L01.Cotizacion/Actualizacion/ActualizarCotCotizacionTransaccionBO.cs`: `ValorIVA = region.Impuesto` → `ValorIVA = cotProductoOferta.TasaOCuota ?? 0`.
- `L01.Cotizacion/CerrarOferta/SolicitarAjustesCerrarOfertaTransaccionBO.cs`: `GravaIVA = proveedorRegion.GravaIVAFleteExpress` → `GravaIVA = cotFlete.TipoFactor != "Exento"`; `Precio * region.Impuesto` → `Precio * (cotFlete.TasaOCuota ?? 0)`.

**Resultado esperado:**
La actualización de cotización y el cierre de oferta calculan el IVA usando `TasaOCuota`/`TipoFactor`.

**Entregables:**
- `ActualizarCotCotizacionTransaccionBO.cs` modificado
- `SolicitarAjustesCerrarOfertaTransaccionBO.cs` modificado

**Criterios de aceptación:**
- `ActualizarCotCotizacionTransaccionBO` calcula `ValorIVA` desde `TasaOCuota` sin referencia a `Region`.
- `SolicitarAjustesCerrarOfertaTransaccionBO` deja de usar `GravaIVAFleteExpress` para determinar si el flete causa IVA; usa `TipoFactor != "Exento"`.
- Las pruebas de regresión del cierre de oferta no muestran diferencias contra los perfiles fiscales vigentes.

**Más información de la tarea:**
Ver tabla "L01 — Cotización" en `R16A-RE-Cambio-PerfilFiscal-Back.md` sección 5.

**Recursos:**
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — secciones 3, 5 (L01)

---

## TAREA 6

**[ R16A-RE-Cambio-PerfilFiscal ] [IMP-EXIST-SERVICE] Migrar cálculo de IVA — L02 Ajustar Oferta y L03 Promesa de Compra**

**Aplicativos:** ProquifaDotNet

**Módulos:** Logic.Pqf.Logistica — L02.AjustarOferta, L03.PromesaDeCompra

**Consideraciones previas:**
- Depende de la Tarea 3 (Core.Pqf actualizado).
- Se agrupan L02 y L03 en una sola tarea por ser cambios de un único archivo cada uno, de la misma naturaleza (sustitución del cálculo de IVA).
- No existe dependencia funcional entre `AutorizarAjusteOfertaTransaccionBO.cs` (L02) y `PretramitarPromesaDeCompraTransaccionBO.cs` (L03); se agrupan solo por eficiencia de la tarea.

**Objetivo general:**
Migrar el cálculo de IVA en la autorización de ajuste de oferta (L02) y en el pretramitar de promesa de compra (L03) al patrón `TasaOCuota`/`TipoFactor`.

**Objetivos específicos:**
- `L02.AjustarOferta/AjustesCotizacion/AutorizarAjusteOferta/AutorizarAjusteOfertaTransaccionBO.cs`: `porcentajeIVASistema = region.Impuesto` → `porcentajeIVASistema = cotProductoOferta.TasaOCuota ?? 0`; `gravaIVAPorRegionCliente = region.Impuesto > 0` → `gravaIVA = cotProductoOferta.TipoFactor != "Exento"`.
- `L03.PromesaDeCompra/Processors/PretramitarPromesaDeCompraTransaccionBO.cs`: `producto.GravaIVA ? ... * region.Impuesto : 0` → `(pcPartida.TipoFactor != "Exento") ? ... * (pcPartida.TasaOCuota ?? 0) : 0`.

**Resultado esperado:**
La autorización de ajuste de oferta y el pretramitar de promesa de compra calculan el IVA usando `TasaOCuota`/`TipoFactor`.

**Entregables:**
- `AutorizarAjusteOfertaTransaccionBO.cs` modificado
- `PretramitarPromesaDeCompraTransaccionBO.cs` modificado

**Criterios de aceptación:**
- `AutorizarAjusteOfertaTransaccionBO` calcula `porcentajeIVASistema` desde `TasaOCuota` sin referencia a `Region`.
- `PretramitarPromesaDeCompraTransaccionBO` calcula el IVA de la partida usando `pcPartida.TipoFactor`/`TasaOCuota`.
- Las pruebas de regresión de ajuste de oferta y promesa de compra no muestran diferencias contra los perfiles fiscales vigentes.

**Más información de la tarea:**
Ver tablas "L02 — Ajustar Oferta" y "L03 — Promesa de Compra" en `R16A-RE-Cambio-PerfilFiscal-Back.md` sección 5.

**Recursos:**
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — secciones 3, 5 (L02, L03)

---

## TAREA 7

**[ R16A-RE-Cambio-PerfilFiscal ] [IMP-EXIST-SERVICE] Migrar cálculo de IVA — L04 Pretramitar Pedido**

**Aplicativos:** ProquifaDotNet

**Módulos:** Logic.Pqf.Logistica — L04.PretramitarPedido (Partidas, Fabrica/Recalcular)

**Consideraciones previas:**
- Depende de la Tarea 3 (Core.Pqf actualizado).
- `ppPartidaPedidoRecalculoBO.cs` tiene 3 puntos de cálculo distintos (líneas 127, 151, 256 en la versión actual) — todos deben migrarse en la misma tarea para mantener consistencia del archivo.
- `ppPedidoRecalcularBO.cs` tiene puntos de cálculo tanto de partidas (L151) como de flete (L352, L467) — ambos se migran en esta tarea.

**Objetivo general:**
Migrar el cálculo de IVA de partidas y fletes en el flujo de Pretramitar Pedido al patrón `TasaOCuota`/`TipoFactor`.

**Objetivos específicos:**
- `L04.PretramitarPedido/Partidas/ppPartidaPedidoBO.cs`: `producto.GravaIVA ? ... * region.Impuesto : 0` → `(ppPartida.TipoFactor != "Exento") ? ... * (ppPartida.TasaOCuota ?? 0) : 0`.
- `L04.PretramitarPedido/Partidas/ppPartidaPedidoRecalculoBO.cs`: `!gravaIVA ? 0 : ... * region.Impuesto` (3 puntos) → `... * (partida.TasaOCuota ?? 0)`; `gravaIVA` derivado de `TipoFactor != "Exento"`.
- `L04.PretramitarPedido/Fabrica/Recalcular/ppPedidoRecalcularBO.cs`: `region.Impuesto > 0` → `TipoFactor != "Exento"`; `ValorIVA = region.Impuesto` (partidas y flete) → `ValorIVA = vFlete.TasaOCuota ?? 0` / `TasaOCuota ?? 0` según corresponda; eliminar lookup a `Region`.

**Resultado esperado:**
El flujo de Pretramitar Pedido (partidas y fletes) calcula el IVA usando `TasaOCuota`/`TipoFactor` en los 3 archivos, sin lookups a `Region`.

**Entregables:**
- `ppPartidaPedidoBO.cs` modificado
- `ppPartidaPedidoRecalculoBO.cs` modificado (3 puntos de cálculo)
- `ppPedidoRecalcularBO.cs` modificado (partidas + flete)

**Criterios de aceptación:**
- Los 3 archivos calculan IVA desde `TasaOCuota`/`TipoFactor` sin referencias a `region.Impuesto`.
- `ppPartidaPedidoRecalculoBO` migra los 3 puntos de cálculo de forma consistente.
- `ppPedidoRecalcularBO` migra tanto el cálculo de partidas como el de flete.
- Las pruebas de regresión de pretramitar pedido no muestran diferencias contra los perfiles fiscales vigentes.

**Más información de la tarea:**
Ver tabla "L04 — Pretramitar Pedido" en `R16A-RE-Cambio-PerfilFiscal-Back.md` sección 5.

**Recursos:**
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — secciones 3, 5 (L04)

---

## TAREA 8

**[ R16A-RE-Cambio-PerfilFiscal ] [IMP-EXIST-SERVICE] Migrar generación de Conceptos CFDI y cálculo de IVA — L05 Tramitar Pedido**

**Aplicativos:** ProquifaDotNet

**Módulos:** Logic.Pqf.Logistica — L05.TramitarPedido (Facturas/Generadores/Partidas, Utils/CFDIGeneradas)

**Consideraciones previas:**
- Depende de la Tarea 3 (Core.Pqf actualizado).
- Esta tarea elimina los hard-codes de clave SAT y tasa de IVA en la generación de conceptos CFDI — es la de mayor impacto fiscal del cambio (alimenta directamente el timbrado).
- El BO generador de la Factura por Adelantada no existía como cambio explícito antes de este requisito — se agrega la resolución fiscal vía `PerfilFiscalConfiguracionFamilia WHERE ClaveTipoEntidad = 'factura-anticipo' AND IdFamilia IS NULL AND IdRegion = :idRegion`.
- `CFDIGeneradaConceptoToCFDIGeneradaImpuestoIVABO.cs` cambia de firma: la tasa deja de estar hard-codeada y se recibe como parámetro.

**Objetivo general:**
Eliminar los hard-codes de clave de producto/servicio SAT, clave de unidad SAT y tasa de IVA en la generación de conceptos CFDI, sustituyéndolos por los campos fiscales resueltos desde `PerfilFiscalConfiguracionFamilia`.

**Objetivos específicos:**
- `L05.TramitarPedido/Facturas/Generadores/Partidas/tpProformaPartidaPedidoToCFDIGeneradaConceptoBO.cs`: `ClaveProductoServicio = familia.ClaveProductoServicioCFDI` → `ClaveProductoServicio = tpProformaPartidaPedido.ClaveProdServSat`; `ClaveUnidad = "H87"` → `ClaveUnidad = tpProformaPartidaPedido.ClaveUnidadSat`; `if (producto.GravaIVA)` → `if (tpProformaPartidaPedido.TipoFactor != "Exento")`; eliminar lookups a `vProductoBO` y `FamiliaBO` para datos fiscales.
- `Utils/CFDIGeneradas/Conceptos/IVA/CFDIGeneradaConceptoToCFDIGeneradaImpuestoIVABO.cs`: `TasaOCuota = 0.16M` (hard-code) → recibir tasa como parámetro; `Importe = 0.16M * source.Importe` → `Importe = tasa * source.Importe`.
- BO generador de concepto de Factura por Adelantada: resolver datos fiscales del concepto de anticipo consultando `PerfilFiscalConfiguracionFamilia WHERE ClaveTipoEntidad = 'factura-anticipo' AND IdFamilia IS NULL AND IdRegion = :idRegion` → obtener `ClaveProdServSat`, `ClaveUnidadSat`, `TipoFactor`, `TasaOCuota`; eliminar cualquier valor fijo de clave SAT hard-codeado para este concepto.

**Resultado esperado:**
La generación de conceptos CFDI (proforma, factura, factura anticipo) usa exclusivamente los campos fiscales resueltos por `PerfilFiscalConfiguracionFamilia` — sin claves SAT ni tasas de IVA hard-codeadas.

**Entregables:**
- `tpProformaPartidaPedidoToCFDIGeneradaConceptoBO.cs` modificado
- `CFDIGeneradaConceptoToCFDIGeneradaImpuestoIVABO.cs` modificado (nueva firma con tasa como parámetro)
- BO generador de concepto de Factura por Adelantada modificado (resolución `'factura-anticipo'`)

**Criterios de aceptación:**
- Ningún concepto CFDI generado usa `"H87"`, `ClaveProductoServicioCFDI` de familia, ni `0.16M` hard-codeados.
- El nodo de impuesto IVA del CFDI usa la tasa recibida como parámetro, no un literal.
- El concepto de anticipo resuelve su clave SAT y tasa desde `PerfilFiscalConfiguracionFamilia` con `ClaveTipoEntidad = 'factura-anticipo'`.
- Un producto/concepto con `TipoFactor = "Exento"` no genera nodo de traslado de IVA con importe.
- Las pruebas de regresión de timbrado (proforma → factura) no muestran diferencias contra los perfiles fiscales vigentes.

**Más información de la tarea:**
Ver tabla "L05 — Tramitar Pedido" en `R16A-RE-Cambio-PerfilFiscal-Back.md` sección 5.

**Recursos:**
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — secciones 3, 5 (L05)
- `R16A-RE-Cambio-PerfilFiscal_BD.md` — sección 5 (seed `ClaveTipoEntidad = 'factura-anticipo'`)

---

## TAREA 9

**[ R16A-RE-Cambio-PerfilFiscal ] [IMP-EXIST-SERVICE] Migrar cálculo de IVA en correo interno y PDF de confirmación de pedido**

**Aplicativos:** ProquifaDotNet

**Módulos:** Logic.MailXslt (TramitarPedido), Logic.PDF (Pedido)

**Consideraciones previas:**
- Depende de la Tarea 3 (Core.Pqf actualizado).
- Ambos archivos son de presentación (correo interno y PDF de confirmación) — no afectan el cálculo transaccional del pedido, solo su despliegue visual del porcentaje/monto de IVA.

**Objetivo general:**
Migrar el cálculo del monto y porcentaje de IVA mostrado en el correo interno y el PDF de confirmación de pedido al patrón `TasaOCuota`/`TipoFactor`.

**Objetivos específicos:**
- `Logic.MailXslt/TramitarPedido/CorreoPedidoInterno.cs`: `region.Impuesto` en cálculo de flete → `vFlete.TasaOCuota ?? 0`.
- `Logic.PDF/Pedido/ArchivoBOConfirmacionPedidoExtensionsPdf.cs`: `(region.Impuesto * 100M).ToString("0") + "%"` → `((tpPartida.TasaOCuota ?? 0) * 100M).ToString("0") + "%"`.

**Resultado esperado:**
El correo interno y el PDF de confirmación de pedido muestran el porcentaje y monto de IVA calculados desde `TasaOCuota`/`TipoFactor`.

**Entregables:**
- `CorreoPedidoInterno.cs` modificado
- `ArchivoBOConfirmacionPedidoExtensionsPdf.cs` modificado

**Criterios de aceptación:**
- El correo interno calcula el monto de IVA de flete desde `vFlete.TasaOCuota`.
- El PDF de confirmación muestra el porcentaje de IVA formateado desde `tpPartida.TasaOCuota`.
- Ambos archivos no contienen referencias a `region.Impuesto`.

**Más información de la tarea:**
Ver tabla "Otros proyectos" en `R16A-RE-Cambio-PerfilFiscal-Back.md` sección 5.

**Recursos:**
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — secciones 3, 5 (Otros proyectos)

---

## Recursos generales

- `R16A-RE-Cambio-PerfilFiscal_BD.md` — Diccionario de Datos completo
- `R16A-RE-Cambio-PerfilFiscal-Back.md` — Impacto en Back de ProquifaDotNet
- `R16A-RE-Cambio-PerfilFiscal-Back-Finanzas.md` — Impacto en ProquifaDotNet.Finanzas (integración por separado, no cubierta en este documento)
- `Perfil-Fiscal-REQ-00.md` — Requisito funcional
- `Catalogo BackEnd.md` — Catálogo de actividades y estimaciones
- `Estandar Redaccion Tarea.md` — Estándar de redacción de tareas
