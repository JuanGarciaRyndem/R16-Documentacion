# Impacto en BD - Factura por Adelantado: Detalle Mexico
**Requisito:** R16A-RE-FU-019
**Bases de Datos:** ProquifaDotNet (lectura/escritura)
**Version:** 3.1 - EmpresaFolio corregida: IdEmpresa FK (no EmpresaClave), columnas de auditoría estándar, UPDLOCK atómico como mecanismo único de folio (sin integración Legacy consecutivo)

---

## Resumen
Pantalla de Detalle por cliente: generar, timbrar y enviar la factura PPD por cada pedido
pendiente. Post-envio: Credito transfiere a Legacy / Prepago genera pendiente Validar Cobro.
Solo clientes Mexico. Sin sustancias controladas.

> **Nota de arquitectura (correccion — el CFDI no va en Timbrado, va en Finanzas):** este requisito
> es el que **crea `CFDIGenerada`** (ER-Finanzas.md, propiedad de ProquifaDotNet.Finanzas) sobre la
> base de datos `ProquifaDotNet`. El paso "TIMBRAR" del flujo de datos se corrigio: Timbrado
> (`ProquifaDotNet.Timbrado`) es un servicio tecnico que consume el folio y llama al PAC, pero
> **no persiste el CFDI como entidad de negocio** — es Finanzas quien hace `INSERT CFDIGenerada`
> y quien pobla `fccFactura.IdCFDIGenerada` con el Id real de ese registro. Ver
> `R16A-RE-FU-018_BD.md` (Parte 2 y 3) para el detalle completo de esta correccion.

> **Migración (06/07/2026) — ya NO se crea `ALTER TABLE tpProformaAdelanto`/`CREATE VIEW vtpProformaAdelanto` aquí:**
> Este requisito originalmente creaba el `ALTER TABLE tpProformaAdelanto ADD Enviada` y la vista
> `vtpProformaAdelanto`. Ambos objetos se retiran: `fccFactura` (con las columnas `Enviada` e
> `IdCFDIGenerada`) y la vista `vfccFactura` se crean y son propiedad de **R16A-RE-FU-015**, que
> unifica el pendiente FAA de Crédito (RE-FU-012) y Prepago (RE-FU-015) en una sola tabla. Este
> requisito **consume** `fccFactura`/`vfccFactura`, no las crea. Ver `R16A-RE-FU-015_BD.md`,
> sección "Migración de tpProformaAdelanto".

---

## Impacto en BD

| #   | Cambio                                                                               | Base de Datos             | Tipo                                                          | Prioridad |
| --- | ------------------------------------------------------------------------------------ | ------------------------- | ------------------------------------------------------------- | --------- |
| 1   | CREATE TABLE dbo.CFDIGenerada (incl. IdCatMetodoDePagoCFDI FK, IdCatFormaPagoSAT col, Exportacion) | ProquifaDotNet            | DDL                                                           | Alta      |
| 2   | CREATE TABLE dbo.EmpresaFolio                                                        | ProquifaDotNet (Finanzas) | DDL                                                           | Alta      |
| 3   | INSERT EmpresaFolio (4 empresas MEX)                                                 | ProquifaDotNet (Finanzas) | DML                                                           | Alta      |
| 4   | CREATE TABLE dbo.catFormaPagoSAT + INSERT seed (22 claves c_FormaPago SAT)           | ProquifaDotNet (Finanzas) | DDL + DML                                                     | Alta      |
| 5   | CREATE TABLE dbo.catImpuestoSat + INSERT seed (IVA, ISR, IEPS)                       | ProquifaDotNet (Finanzas) | DDL + DML                                                     | Alta      |
| 6   | CREATE TABLE dbo.catTipoFactorSat + INSERT seed (Tasa, Cuota, Exento)                | ProquifaDotNet (Finanzas) | DDL + DML                                                     | Alta      |
| 7   | CREATE TABLE dbo.catObjetoImpuestoSat + INSERT seed (01-04)                          | ProquifaDotNet (Finanzas) | DDL + DML                                                     | Alta      |
| 8   | CREATE TABLE dbo.PerfilFiscal + INSERT seed (IVA 16% Tasa / Exento)                  | ProquifaDotNet (Finanzas) | DDL + DML                                                     | Alta      |
| -   | ~~ALTER TABLE tpProformaAdelanto ADD Enviada~~ / ~~CREATE VIEW vtpProformaAdelanto~~ | ProquifaDotNet            | **Movido a RE-FU-015** (`fccFactura.Enviada` + `vfccFactura`) | -         |

> **Nota (origen de CFDIGenerada):** este requisito ejecuta el `CREATE TABLE CFDIGenerada` con todas sus columnas iniciales (incluye `IdCatMetodoDePagoCFDI` FK, `IdCatFormaPagoSAT` col y `Exportacion`). Los catalogos `IdCatTipoCFDI`/`IdCFDIRelacionado` se agregan despues via `ALTER TABLE` en R16A-RE-FU-028, y las columnas tecnicas (Estado, MensajeError, IdArchivoXml, IdCatUsoCFDI, IdCatMoneda, TipoCambio) se agregan via `ALTER TABLE` en R16A-RE-FU-018 (Parte 3) — ambos posteriores a la creacion aqui.
>
> **Orden de dependencia:** `fccFactura.IdCFDIGenerada` (RE-FU-015) requiere que `CFDIGenerada` exista — este requisito (paso 1 de esta tabla) debe ejecutarse antes de que RE-FU-015 pueda agregar esa FK.

---

## CREATE TABLE CFDIGenerada (ProquifaDotNet — propiedad de Finanzas)

**Proposito:** Registro central de negocio de todo CFDI emitido (diseñado en `ER-Finanzas.md`,
gestionado por `ProquifaDotNet.Finanzas` via EF Core Scaffold). Incluye las columnas base más
los campos CFDI 4.0 requeridos por la Guía Técnica SAT (Guía Técnica Facturas Ingreso MX).
`IdCatTipoCFDI`/`IdCFDIRelacionado` se agregan en R16A-RE-FU-028 y las columnas técnicas de
timbrado en R16A-RE-FU-018 (Parte 3) via `ALTER TABLE` posteriores.

```sql
    -- Ejecutar sobre ProquifaDotNet
    -- Requiere: catMetodoDePagoCFDI (tabla existente), catFormaPagoSAT (creada en este mismo requisito, paso anterior)
    CREATE TABLE [dbo].[CFDIGenerada](
        [IdCFDIGenerada]        uniqueidentifier NOT NULL
            CONSTRAINT [DF_CFDIGenerada_Id] DEFAULT (NEWID()),
        [RFCEmisor]             varchar(13)   NOT NULL,
        [RFCReceptor]           varchar(50)   NOT NULL,
        [Serie]                 varchar(25)   NULL,
        [Folio]                 varchar(40)   NULL,
        [FechaEmision]          datetime2(7)  NULL,
        [UUID]                  varchar(36)   NULL,
        [Total]                 decimal(18,2) NULL,
        [IdCatMetodoDePagoCFDI] uniqueidentifier NULL
            CONSTRAINT [FK_CFDIGenerada_catMetodoDePagoCFDI]
            FOREIGN KEY REFERENCES [dbo].[catMetodoDePagoCFDI]([IdCatMetodoDePagoCFDI]),
        [IdCatFormaPagoSAT]     uniqueidentifier NULL
            CONSTRAINT [FK_CFDIGenerada_catFormaPagoSAT]
            FOREIGN KEY REFERENCES [dbo].[catFormaPagoSAT]([IdCatFormaPagoSAT]),
        [Exportacion]           varchar(2)    NULL
            CONSTRAINT [DF_CFDIGenerada_Exportacion] DEFAULT ('01'),
        [Activo]                bit NOT NULL
            CONSTRAINT [DF_CFDIGenerada_Activo] DEFAULT (1),
        [FechaRegistro]         datetime2(7)  NOT NULL
            CONSTRAINT [DF_CFDIGenerada_FechaRegistro] DEFAULT (SYSUTCDATETIME()),
        CONSTRAINT [PK_CFDIGenerada] PRIMARY KEY CLUSTERED ([IdCFDIGenerada])
    );
    GO
```

| Columna               | Tipo             | Nulo | Default          | Descripcion                                                  |
| --------------------- | ---------------- | ---- | ---------------- | ------------------------------------------------------------ |
| IdCFDIGenerada        | uniqueidentifier | NO   | NEWID()          | PK                                                           |
| RFCEmisor             | varchar(13)      | NO   | -                | RFC de la empresa emisora                                    |
| RFCReceptor           | varchar(50)      | NO   | -                | RFC/RUC del cliente receptor                                 |
| Serie                 | varchar(25)      | SI   | -                | Serie del CFDI                                               |
| Folio                 | varchar(40)      | SI   | -                | Folio del CFDI (consecutivo formateado, consumido de `EmpresaFolio` con UPDLOCK atómico) |
| FechaEmision          | datetime2(7)     | SI   | -                | Fecha de emisión (timbrado)                                  |
| UUID                  | varchar(36)      | SI   | -                | UUID asignado por el PAC al timbrar                          |
| Total                 | decimal(18,2)    | SI   | -                | Monto total del CFDI (usado por vfccFactura)                 |
| IdCatMetodoDePagoCFDI | uniqueidentifier | SI   | -                | FK → `catMetodoDePagoCFDI` — `PPD` (FAA/crédito) o `PUE`     |
| IdCatFormaPagoSAT     | uniqueidentifier | SI   | -                | FK → `catFormaPagoSAT`; `99` (Por definir) en FAA            |
| Exportacion           | varchar(2)       | SI   | `01`             | c_Exportacion SAT, obligatorio CFDI 4.0; `01` = No aplica    |
| Activo                | bit              | NO   | 1                | Activo                                                       |
| FechaRegistro         | datetime2(7)     | NO   | SYSUTCDATETIME() | Fecha de creación del registro                               |

> Ver R16A-RE-FU-028_BD.md (ALTER: IdCatTipoCFDI, IdCFDIRelacionado) y R16A-RE-FU-018_BD.md Parte 3 (ALTER: Estado, MensajeError, IdArchivoXml, IdCatUsoCFDI, IdCatMoneda, TipoCambio, FechaUltimaActualizacion) para las extensiones posteriores de esta tabla.

---

## CREATE TABLE catFormaPagoSAT (ProquifaDotNet — Finanzas)

> **⏸ Pendiente** — Esta tabla queda en espera junto con `PerfilFiscal`, `catImpuestoSat`, `catTipoFactorSat` y `catObjetoImpuestoSat` (ver sección PerfilFiscal). No ejecutar hasta resolver ese bloque.
>
> **⚠️ Pendiente revisar adicionalmente — ¿es la misma que `catMedioDePago`?**
> `catMedioDePago` es una tabla existente en ProquifaDotNet que almacena las formas de pago del sistema (con campo `ClaveFormaDePago` que debería mapear al catálogo c_FormaPago SAT). `catFormaPagoSAT` sería un catálogo nuevo con las mismas 22 claves SAT. Antes de crear esta tabla confirmar si `catMedioDePago` ya cubre este rol o si ambas deben coexistir (una para la operación comercial, otra como catálogo SAT de referencia para el CFDI). Si son equivalentes, la FK de `CFDIGenerada.IdCatFormaPagoSAT` podría apuntar directamente a `catMedioDePago` y esta tabla no sería necesaria.

Catálogo c_FormaPago del SAT. Seed con las 22 claves vigentes. Requerido como prerequisito de `CFDIGenerada` (FK `IdCatFormaPagoSAT`). La clave `99` (Por definir) se usa en FAA; las demás claves son utilizadas en facturas PUE y Complementos de Pago (RE-FU-030).

```sql
-- Ejecutar sobre ProquifaDotNet — ANTES de CREATE TABLE CFDIGenerada
CREATE TABLE [dbo].[catFormaPagoSAT](
    [IdCatFormaPagoSAT]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catFormaPagoSAT_Id]      DEFAULT (NEWID()),
    [Clave]              varchar(10)      NOT NULL,
    [Descripcion]        nvarchar(150)    NOT NULL,
    [Activo]             bit              NOT NULL
        CONSTRAINT [DF_catFormaPagoSAT_Activo]  DEFAULT (1),
    [FechaRegistro]      datetime2(7)     NOT NULL
        CONSTRAINT [DF_catFormaPagoSAT_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catFormaPagoSAT]
        PRIMARY KEY CLUSTERED ([IdCatFormaPagoSAT]),
    CONSTRAINT [UQ_catFormaPagoSAT_Clave]
        UNIQUE ([Clave])
);
GO

INSERT INTO [dbo].[catFormaPagoSAT] ([Clave], [Descripcion]) VALUES
    ('01', 'Efectivo'),
    ('02', 'Cheque nominativo'),
    ('03', 'Transferencia electrónica de fondos'),
    ('04', 'Tarjeta de crédito'),
    ('05', 'Monedero electrónico'),
    ('06', 'Dinero electrónico'),
    ('08', 'Vales de despensa'),
    ('12', 'Dación en pago'),
    ('13', 'Pago por subrogación'),
    ('14', 'Pago por consignación'),
    ('15', 'Condonación'),
    ('17', 'Compensación'),
    ('23', 'Novación'),
    ('24', 'Confusión'),
    ('25', 'Remisión de deuda'),
    ('26', 'Prescripción o caducidad'),
    ('27', 'A satisfacción del acreedor'),
    ('28', 'Tarjeta de débito'),
    ('29', 'Tarjeta de servicios'),
    ('30', 'Aplicación de anticipos'),
    ('31', 'Intermediario pagos'),
    ('99', 'Por definir');
GO
```

| Columna           | Tipo             | Nulo | Descripcion                                                                  |
| ----------------- | ---------------- | ---- | ---------------------------------------------------------------------------- |
| IdCatFormaPagoSAT | uniqueidentifier | NO   | PK                                                                           |
| Clave             | varchar(10)      | NO   | Código c_FormaPago SAT. UNIQUE. `99` = Por definir (usado en FAA)            |
| Descripcion       | nvarchar(150)    | NO   | Descripción legible de la forma de pago                                      |
| Activo            | bit              | NO   | 1 = vigente                                                                  |
| FechaRegistro     | datetime2(7)     | NO   | Fecha de inserción                                                           |

---

## CREATE TABLE catImpuestoSat (ProquifaDotNet — Finanzas)

> **⏸ Pendiente** — En espera junto con `PerfilFiscal`, `catTipoFactorSat`, `catObjetoImpuestoSat` y `catFormaPagoSAT`. No ejecutar hasta resolver el bloque de PerfilFiscal (GAP-7 / GAP-8).

Catálogo c_Impuesto del SAT. Seed inicial con los 3 valores vigentes.
```sql
    -- Ejecutar sobre ProquifaDotNet
    CREATE TABLE [dbo].[catImpuestoSat](
        [IdCatImpuestoSat] uniqueidentifier NOT NULL
            CONSTRAINT [DF_catImpuestoSat_Id] DEFAULT (NEWID()),
        [Clave]       varchar(10)  NOT NULL,
        [Descripcion] varchar(100) NOT NULL,
        [Activo]      bit NOT NULL
            CONSTRAINT [DF_catImpuestoSat_Activo] DEFAULT (1),
        [FechaRegistro] datetime2(7) NOT NULL
            CONSTRAINT [DF_catImpuestoSat_FechaRegistro] DEFAULT (SYSUTCDATETIME()),
        CONSTRAINT [PK_catImpuestoSat] PRIMARY KEY CLUSTERED ([IdCatImpuestoSat]),
        CONSTRAINT [UQ_catImpuestoSat_Clave] UNIQUE ([Clave])
    );
    GO

    INSERT INTO [dbo].[catImpuestoSat] ([Clave], [Descripcion])
    VALUES
        ('001', 'ISR'),
        ('002', 'IVA'),
        ('003', 'IEPS');
```
---

## CREATE TABLE catTipoFactorSat (ProquifaDotNet — Finanzas)

> **⏸ Pendiente** — En espera junto con `PerfilFiscal`, `catImpuestoSat`, `catObjetoImpuestoSat` y `catFormaPagoSAT`. No ejecutar hasta resolver el bloque de PerfilFiscal (GAP-7 / GAP-8).

Catálogo c_TipoFactor del SAT. Seed inicial con los 3 valores vigentes.
```sql
    -- Ejecutar sobre ProquifaDotNet
    CREATE TABLE [dbo].[catTipoFactorSat](
        [IdCatTipoFactorSat] uniqueidentifier NOT NULL
            CONSTRAINT [DF_catTipoFactorSat_Id] DEFAULT (NEWID()),
        [Clave]       varchar(20)  NOT NULL,
        [Descripcion] varchar(100) NOT NULL,
        [Activo]      bit NOT NULL
            CONSTRAINT [DF_catTipoFactorSat_Activo] DEFAULT (1),
        [FechaRegistro] datetime2(7) NOT NULL
            CONSTRAINT [DF_catTipoFactorSat_FechaRegistro] DEFAULT (SYSUTCDATETIME()),
        CONSTRAINT [PK_catTipoFactorSat] PRIMARY KEY CLUSTERED ([IdCatTipoFactorSat]),
        CONSTRAINT [UQ_catTipoFactorSat_Clave] UNIQUE ([Clave])
    );
    GO

    INSERT INTO [dbo].[catTipoFactorSat] ([Clave], [Descripcion])
    VALUES
        ('Tasa',   'Tasa'),
        ('Cuota',  'Cuota'),
        ('Exento', 'Exento');
```
---

## CREATE TABLE catObjetoImpuestoSat (ProquifaDotNet — Finanzas)

> **⏸ Pendiente** — En espera junto con `PerfilFiscal`, `catImpuestoSat`, `catTipoFactorSat` y `catFormaPagoSAT`. No ejecutar hasta resolver el bloque de PerfilFiscal (GAP-7 / GAP-8).

Catálogo c_ObjetoImp del SAT. Seed inicial con los 4 valores vigentes.
```sql
    -- Ejecutar sobre ProquifaDotNet
    CREATE TABLE [dbo].[catObjetoImpuestoSat](
        [IdCatObjetoImpuestoSat] uniqueidentifier NOT NULL
            CONSTRAINT [DF_catObjetoImpuestoSat_Id] DEFAULT (NEWID()),
        [Clave]       varchar(5)   NOT NULL,
        [Descripcion] varchar(200) NOT NULL,
        [Activo]      bit NOT NULL
            CONSTRAINT [DF_catObjetoImpuestoSat_Activo] DEFAULT (1),
        [FechaRegistro] datetime2(7) NOT NULL
            CONSTRAINT [DF_catObjetoImpuestoSat_FechaRegistro] DEFAULT (SYSUTCDATETIME()),
        CONSTRAINT [PK_catObjetoImpuestoSat] PRIMARY KEY CLUSTERED ([IdCatObjetoImpuestoSat]),
        CONSTRAINT [UQ_catObjetoImpuestoSat_Clave] UNIQUE ([Clave])
    );
    GO

    INSERT INTO [dbo].[catObjetoImpuestoSat] ([Clave], [Descripcion])
    VALUES
        ('01', 'No objeto de impuesto'),
        ('02', 'Sí objeto de impuesto'),
        ('03', 'Sí objeto del impuesto y no obligado al desglose'),
        ('04', 'Sí objeto del impuesto y no causa impuesto');
```
---

## CREATE TABLE PerfilFiscal (ProquifaDotNet — Finanzas)

> **⏸ Pendiente** — La creación de `PerfilFiscal` y sus catálogos dependientes (`catImpuestoSat`, `catTipoFactorSat`, `catObjetoImpuestoSat`) queda en espera hasta confirmar el nivel de configuración (Producto / Familia / Producto→Familia con precedencia) para `ClaveProdServ`, `ClaveUnidad` e `IdPerfilFiscal`. Ver GAP-7 y GAP-8. No ejecutar scripts hasta resolver ese punto.

Catálogo de negocio de 3-4 filas que traduce la tasa de IVA de un producto a las claves técnicas exigidas por el XML del CFDI (nodo `Impuestos/Traslados`). Lo administra PROQUIFA. Ver sección "Datos del producto — ClaveProdServ, ClaveUnidad y PerfilFiscal" para el modelo conceptual completo.
```sql
    -- Ejecutar sobre ProquifaDotNet (requiere catImpuestoSat, catTipoFactorSat, catObjetoImpuestoSat)
    CREATE TABLE [dbo].[PerfilFiscal](
        [IdPerfilFiscal] uniqueidentifier NOT NULL
            CONSTRAINT [DF_PerfilFiscal_Id] DEFAULT (NEWID()),
        [Nombre]                  nvarchar(100)   NOT NULL,   -- 'IVA General 16%', 'IVA Tasa 0%', 'Exento'
        [IdCatImpuestoSat]        uniqueidentifier NOT NULL
            CONSTRAINT [FK_PerfilFiscal_catImpuestoSat]
            FOREIGN KEY REFERENCES [dbo].[catImpuestoSat]([IdCatImpuestoSat]),
        [IdCatTipoFactorSat]      uniqueidentifier NOT NULL
            CONSTRAINT [FK_PerfilFiscal_catTipoFactorSat]
            FOREIGN KEY REFERENCES [dbo].[catTipoFactorSat]([IdCatTipoFactorSat]),
        [TasaOCuota]              decimal(6,6)    NULL,       -- NULL cuando IdCatTipoFactorSat = 'Exento'
        [IdCatObjetoImpuestoSat]  uniqueidentifier NOT NULL
            CONSTRAINT [FK_PerfilFiscal_catObjetoImpuestoSat]
            FOREIGN KEY REFERENCES [dbo].[catObjetoImpuestoSat]([IdCatObjetoImpuestoSat]),
        [Fundamento]              nvarchar(200)   NULL,       -- referencia legal, ej. 'Art. 2-A LIVA'
        [Activo]        bit NOT NULL
            CONSTRAINT [DF_PerfilFiscal_Activo] DEFAULT (1),
        [FechaRegistro] datetime2(7) NOT NULL
            CONSTRAINT [DF_PerfilFiscal_FechaRegistro] DEFAULT (SYSUTCDATETIME()),
        CONSTRAINT [PK_PerfilFiscal] PRIMARY KEY CLUSTERED ([IdPerfilFiscal]),
        CONSTRAINT [CK_PerfilFiscal_TasaOCuota] CHECK (
            ([TasaOCuota] IS NULL     AND EXISTS (SELECT 1 FROM [dbo].[catTipoFactorSat] t WHERE t.[IdCatTipoFactorSat] = [IdCatTipoFactorSat] AND t.[Clave] = 'Exento'))
            OR
            ([TasaOCuota] IS NOT NULL AND NOT EXISTS (SELECT 1 FROM [dbo].[catTipoFactorSat] t WHERE t.[IdCatTipoFactorSat] = [IdCatTipoFactorSat] AND t.[Clave] = 'Exento'))
        )
    );
    GO

    -- Seed inicial — 3 filas conocidas para la operación de PROQUIFA
    -- IVA General 16%
    INSERT INTO [dbo].[PerfilFiscal] ([Nombre], [IdCatImpuestoSat], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
    SELECT 'IVA General 16%',
           (SELECT IdCatImpuestoSat  FROM catImpuestoSat       WHERE Clave = '002'),
           (SELECT IdCatTipoFactorSat FROM catTipoFactorSat     WHERE Clave = 'Tasa'),
           0.160000,
           (SELECT IdCatObjetoImpuestoSat FROM catObjetoImpuestoSat WHERE Clave = '02'),
           'Art. 1 LIVA';

    -- IVA Tasa 0%
    INSERT INTO [dbo].[PerfilFiscal] ([Nombre], [IdCatImpuestoSat], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
    SELECT 'IVA Tasa 0%',
           (SELECT IdCatImpuestoSat  FROM catImpuestoSat       WHERE Clave = '002'),
           (SELECT IdCatTipoFactorSat FROM catTipoFactorSat     WHERE Clave = 'Tasa'),
           0.000000,
           (SELECT IdCatObjetoImpuestoSat FROM catObjetoImpuestoSat WHERE Clave = '02'),
           'Art. 2-A LIVA';

    -- Exento de IVA (TasaOCuota = NULL)
    INSERT INTO [dbo].[PerfilFiscal] ([Nombre], [IdCatImpuestoSat], [IdCatTipoFactorSat], [TasaOCuota], [IdCatObjetoImpuestoSat], [Fundamento])
    SELECT 'Exento',
           (SELECT IdCatImpuestoSat  FROM catImpuestoSat       WHERE Clave = '002'),
           (SELECT IdCatTipoFactorSat FROM catTipoFactorSat     WHERE Clave = 'Exento'),
           NULL,
           (SELECT IdCatObjetoImpuestoSat FROM catObjetoImpuestoSat WHERE Clave = '02'),
           'Art. 9 LIVA';
    -- IEPS: agregar SOLO si PROQUIFA confirma que algún producto lo requiere (ver GAP-8)
```

> **Por qué 3 filas son suficientes:** el valor timbrado es idéntico para todos los productos que caen en la misma categoría (ej. todos los "IVA Tasa 0%" usan el mismo número independientemente de si la razón legal es Art. 2-A LIVA, medicinas, publicaciones, o exportación). La columna `Fundamento` recoge la referencia legal como dato informativo, no como dato que cambia el cálculo.

---

## Datos del producto — ClaveProdServ, ClaveUnidad y PerfilFiscal

> **⏸ Pendiente** — Toda esta sección (Modelo conceptual, Datos del emisor y Jerarquía de resolución) queda en espera hasta confirmar con PROQUIFA el nivel de configuración de cada campo (GAP-7). No crear FKs ni lógica de resolución hasta tener esa definición.

### Modelo conceptual

Al construir cada concepto del CFDI se necesitan 3 valores que no los captura el usuario al facturar — se resuelven de la configuración del producto:

| Campo            | Catálogo                                                | Origen                            |
| ---------------- | ------------------------------------------------------- | --------------------------------- |
| `ClaveProdServ`  | `catClaveProdServSAT` (c_ClaveProdServ, ~55,000 claves) | Asignado al producto (RE-021)     |
| `ClaveUnidad`    | `catClaveUnidadSAT` (c_ClaveUnidad, estándar UN/ECE)    | Asignado al producto (RE-021)     |
| `IdPerfilFiscal` | `PerfilFiscal` (3-4 filas)                              | Asignado al producto o su Familia |

> `catClaveProdServSAT` y `catClaveUnidadSAT` son catálogos de referencia completos (import del catálogo oficial SAT) — no se administran fila por fila. Su DDL y la columna `Producto.ClaveProdServSAT` son propiedad de **RE-021**. Este requisito solo los consume.

### Datos del emisor — `EmpresaEmisora` = tabla `Empresa` existente (sin tabla nueva)

Los datos del emisor requeridos para el CFDI ya existen en la tabla `Empresa` (BD `ProquifaDotNet`). **No se crea tabla `EmpresaEmisora`.**

| Campo CFDI | Fuente en BD | Notas |
|------------|-------------|-------|
| `RazonSocial` | `Empresa.RazonSocial` | |
| `RFC` | `Empresa.RFC` | |
| `RegimenFiscal` (código SAT) | `catRegimenFiscal.RegimenFiscal` vía `Empresa.IdCatRegimenFiscal` | ej. `601` |
| `LugarExpedicion` | `Direccion.CodigoPostal` vía `Empresa.IdDireccion` | Código postal del domicilio fiscal |

> **Eje independiente de `PerfilFiscal`:** que un producto tribute al 0% es una propiedad del **producto** (`PerfilFiscal`), no de la empresa emisora. No acoplar tasa de IVA a empresa emisora — siempre se resuelve por producto/familia.

### Jerarquía de resolución — Producto → Familia (con precedencia)

Los 3 campos se pueden configurar a 2 niveles, con precedencia:

```
SI el Producto tiene su propio valor capturado (override específico) → se usa ese
SI NO → se hereda el valor configurado en la Familia/Categoría del producto
```

Esto permite definir "toda la Familia X tributa al 16%" una sola vez y solo capturar excepciones puntuales por producto (ej. una publicación dentro de una familia que normalmente no las tiene).

**Ninguno de los 3 campos es editable por el usuario al momento de facturar** — siempre se resuelven por la jerarquía Producto→Familia antes de llegar a la pantalla de generación.

> **GAP-8 (pendiente bloqueante):** Confirmar con PROQUIFA el nivel de configuración para cada campo: ¿`ClaveProdServ` siempre a nivel Producto? ¿`IdPerfilFiscal` a nivel Familia como default con override en Producto? ¿`ClaveUnidad` igual para toda la Familia? Cada campo podría tener un nivel distinto — no asumir que los 3 comparten el mismo nivel. Esto determina si se agrega FK directa en `Producto`, en `FamiliaProducto`, o en ambas.

---

## `fccFactura.Enviada` y vista `vfccFactura` — movidos a RE-FU-015 (06/07/2026)

> Este requisito creaba originalmente `ALTER TABLE tpProformaAdelanto ADD Enviada` y
> `CREATE VIEW dbo.vtpProformaAdelanto`. **Ambos objetos se retiraron de aquí**: la columna
> `Enviada` es ahora parte de `fccFactura` (creada en RE-FU-015) y la vista equivalente se
> llama `vfccFactura` (también en RE-FU-015). No se repite el DDL — ver el diccionario de
> datos y el `CREATE VIEW vfccFactura` completo en `R16A-RE-FU-015_BD.md`.
>
> **Diferencias relevantes para este requisito:**
> - `vfccFactura` ya NO necesita la cadena `fccPagoFacturaAdelanto → tpProformaPedido → tpPedidoProformaPedido → tpPedido` para llegar al pedido: `fccFactura.IdTPPedido` es FK directa.
> - Los datos del receptor (`ClienteRazonSocial`/`ClienteRFC`) ya no requieren JOIN a `DatosFacturacionCliente`: están fijados como snapshot en `fccFactura` (`RazonSocialReceptor`/`RfcReceptor`) desde que se generó el pendiente.
> - `EstadoFAA` se calcula con la misma fórmula (`IdCFDIGenerada IS NULL` → `PendienteGenerar`; `IdCFDIGenerada IS NOT NULL AND Enviada=0` → `PendienteEnviar`; si no, `Completada`).
> - `vfccFactura` cubre **ambos orígenes** (Prepago RE-FU-015 y Crédito RE-FU-012) en una sola vista — `vtpProformaAdelanto` solo cubría lo que existiera en `tpProformaAdelanto`.

---

## CREATE TABLE EmpresaFolio (ProquifaDotNet — propiedad Finanzas, movida de ProquifaDotNetTimbrado el 07/07/2026)

Foliador por empresa/serie. `UltimoFolio` es el **consecutivo** — el entero que se incrementa al timbrar. `CFDIGenerada.Folio` almacena el **folio formateado** resultante (varchar, ej. `A002374`). La fuente de verdad es PQF2/Finanzas — sin integración con tabla legacy.

```sql
    -- Ejecutar en ProquifaDotNet
    CREATE TABLE [dbo].[EmpresaFolio](
        [IdEmpresaFolio] uniqueidentifier NOT NULL
            CONSTRAINT [DF_EmpresaFolio_Id] DEFAULT (NEWID()),
        [IdEmpresa] uniqueidentifier NOT NULL
            CONSTRAINT [FK_EmpresaFolio_Empresa]
                FOREIGN KEY REFERENCES [dbo].[Empresa]([IdEmpresa]),
        [Serie] varchar(25) NULL,
        [UltimoFolio] int NOT NULL
            CONSTRAINT [DF_EmpresaFolio_UltimoFolio] DEFAULT (0),
        [FormatoFolio] varchar(50) NOT NULL
            CONSTRAINT [DF_EmpresaFolio_Formato] DEFAULT ('{folio}'),
        [LongitudMaxima] int NOT NULL
            CONSTRAINT [DF_EmpresaFolio_Longitud] DEFAULT (6),
        [Activo] bit NOT NULL
            CONSTRAINT [DF_EmpresaFolio_Activo] DEFAULT (1),
        [FechaRegistro] datetime2(7) NOT NULL
            CONSTRAINT [DF_EmpresaFolio_FechaRegistro] DEFAULT (SYSUTCDATETIME()),
        [FechaUltimaActualizacion] datetime2(7) NOT NULL
            CONSTRAINT [DF_EmpresaFolio_FechaActualizacion] DEFAULT (SYSUTCDATETIME()),
        CONSTRAINT [PK_EmpresaFolio] PRIMARY KEY CLUSTERED ([IdEmpresaFolio]),
        CONSTRAINT [UQ_EmpresaFolio_EmpresaSerie] UNIQUE ([IdEmpresa], [Serie])
    );
    GO

    -- Serie NULL = factura (serie por empresa); 'P' = CDP; 'P2' = NC
    INSERT INTO [dbo].[EmpresaFolio] ([IdEmpresa], [Serie], [UltimoFolio])
    SELECT e.[IdEmpresa], NULL, 0
    FROM   [dbo].[Empresa] e
    WHERE  e.[Prefijo] IN ('GOL', 'MUN', 'PRO', 'PQF');
    -- Ajustar UltimoFolio al valor actual en producción antes del go-live (ver Gap-1)
```

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| IdEmpresaFolio | uniqueidentifier | NO | NEWID() | PK |
| IdEmpresa | uniqueidentifier | NO | — | FK → `Empresa` — empresa emisora |
| Serie | varchar(25) | SÍ | — | Serie del foliador: NULL = factura, `'P'` = CDP, `'P2'` = NC, `'F001'` = GOLPERU |
| UltimoFolio | int | NO | 0 | **Consecutivo** — último entero asignado; se incrementa con UPDLOCK atómico |
| FormatoFolio | varchar(50) | NO | `'{folio}'` | Patrón de formato del folio presentado (ej. `'A{folio:D6}'`) |
| LongitudMaxima | int | NO | 6 | Longitud máxima del campo folio en caracteres |
| Activo | bit | NO | 1 | Borrado lógico |
| FechaRegistro | datetime2(7) | NO | SYSUTCDATETIME() | Fecha de alta del registro |
| FechaUltimaActualizacion | datetime2(7) | NO | SYSUTCDATETIME() | Última actualización del consecutivo |

### Consumo del folio — UPDLOCK atómico (EmpresaFolioRepository)

`EmpresaFolio` **es** el foliador de PQF2/Finanzas. El consecutivo se incrementa y lee en una sola instrucción atómica, sin dependencia de Legacy:

```sql
    -- EmpresaFolioRepository.ConsumeNextFolioAsync — ejecutar en ProquifaDotNet
    UPDATE [dbo].[EmpresaFolio]
    SET    [UltimoFolio]              = [UltimoFolio] + 1,
           [FechaUltimaActualizacion] = SYSUTCDATETIME()
    OUTPUT inserted.[UltimoFolio]        -- nuevo consecutivo asignado
    WHERE  [IdEmpresa] = @IdEmpresa
      AND  ([Serie] = @Serie OR ([Serie] IS NULL AND @Serie IS NULL));
```

> El `OUTPUT inserted.UltimoFolio` devuelve el consecutivo ya incrementado — es el valor que se formatea y se persiste en `CFDIGenerada.Folio`.
>
> El folio se consume **solo al timbrar exitosamente** — sin huecos por errores de PAC.
>
> `UltimoFolio` debe inicializarse al valor de producción vigente antes del go-live (ver Gap-1).

---

## Flujo de Datos (corregido — CFDI se persiste en Finanzas, Timbrado solo timbra)

    1. GENERAR (modal revision + previsualizacion)
       Lee: vfccFactura (RE-FU-015 — antes: tpPedido, tpProformaAdelanto, DatosFacturacionCliente, Empresa), catUsoCFDI

    2. TIMBRAR (al confirmar previsualizacion) -- CfdiController (ProquifaDotNet.Finanzas)
       ProquifaDotNet.Timbrado (servicio tecnico, POST /api/v1/stamp/invoice):
         UPDATE EmpresaFolio SET UltimoFolio = UltimoFolio + 1 OUTPUT inserted.UltimoFolio (UPDLOCK atómico, EmpresaFolioRepository, ProquifaDotNet)
         Arma el CFDI con ese consecutivo formateado como folio -> llama PAC -> recibe UUID
         INSERT StampingLog (ProquifaDotNetTimbrado, auditoria tecnica de la llamada)
         Regresa a Finanzas: UUID, Serie, Folio, XML, FechaEmision (sin persistir el CFDI como negocio)
       ProquifaDotNet.Finanzas (CfdiService, tras respuesta exitosa de Timbrado):
         INSERT CFDIGenerada (ProquifaDotNet): UUID, Serie, Folio, FechaEmision, IdCatTipoCFDI, Total,
           IdCatUsoCFDI, IdCatMetodoDePagoCFDI, IdCatMoneda, TipoCambio, Estado='Timbrado'
         INSERT Archivo x2 (PDF+XML, FileBucket='facturas') + UPDATE CFDIGenerada SET IdArchivoXml
         UPDATE fccFactura SET IdCFDIGenerada = @IdCFDIGenerada, EsFacturaPorAdelantado = 0,
           IdCatFacturaEstado = GENERADA (catFacturaEstado, RE-FU-015 v2.1)
           (Id real de CFDIGenerada, no un Id de Timbrado; antes: UPDATE tpProformaAdelanto SET IdCFDIGenerada)

    3. ENVIAR (modal envio)
       ProquifaDotNet:
         INSERT CorreoEnviado + ArchivoCorreoEnviado
         UPDATE fccFactura SET Enviada = 1, FechaEnvio = SYSUTCDATETIME(), IdCatFacturaEstado = ENVIADA (antes: UPDATE tpProformaAdelanto SET Enviada = 1)
         Segun tipo (fccFactura.IdTPProformaPedido NOT NULL = origen Credito):
           Credito -> transferencia Legacy
           Prepago -> pendiente Validar Cobro

---

## Estados del Pedido (via vfccFactura)

| EstadoFAA | Condicion | Accion UI |
|-----------|-----------|-----------|
| PendienteGenerar | IdCFDIGenerada IS NULL | 'Generar Factura' (azul) |
| PendienteEnviar | IdCFDIGenerada IS NOT NULL AND Enviada=0 | 'Enviar Factura' (verde) |
| Completada | Enviada=1 | Desaparece del listado |

---

## Orden de Ejecucion de Scripts

| Paso | Script | BD |
|------|--------|-----|
| 1 | CREATE TABLE catFormaPagoSAT + INSERT seed (22 claves) | ProquifaDotNet (Finanzas) |
| 2 | CREATE TABLE CFDIGenerada (requiere Paso 1) | ProquifaDotNet |
| 3 | CREATE TABLE fccFactura + fccFacturaPartida + fccFacturaReferenciaBancaria + CREATE VIEW vfccFactura (RE-FU-015, requiere Paso 2) | ProquifaDotNet |
| 4 | CREATE TABLE EmpresaFolio | ProquifaDotNet (Finanzas) |
| 5 | INSERT EmpresaFolio (datos iniciales) | ProquifaDotNet (Finanzas) |
| 6 | CREATE TABLE catImpuestoSat + INSERT seed | ProquifaDotNet (Finanzas) |
| 7 | CREATE TABLE catTipoFactorSat + INSERT seed | ProquifaDotNet (Finanzas) |
| 8 | CREATE TABLE catObjetoImpuestoSat + INSERT seed | ProquifaDotNet (Finanzas) |
| 9 | CREATE TABLE PerfilFiscal + INSERT seed (requiere Pasos 6-8) | ProquifaDotNet (Finanzas) |

> Los pasos 2 (`fccFactura`/`vfccFactura`) se ejecutan como parte de RE-FU-015, no de este requisito — se listan aquí solo para dejar clara la dependencia de orden.

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | UltimoFolio inicial por empresa | Técnico | Inicializar `EmpresaFolio.UltimoFolio` al valor de producción vigente (Mungen 2374, Golocaer 7156, Proquifa 20913, Proveedora QF 143103) antes del go-live |
| 2 | Lote del producto al timbrar FAA | Negocio | No disponible - confirmar |
| 3 | Politica ante caida del PAC | Tecnico | Definir reintento/encolamiento |
| ~~4~~ | ~~Rol operativo~~ | Negocio | **[Resuelto — Duda 047]** Rol: **Gestor de Cobranza**. Puesto de trabajo: **Analista de Cuentas por Pagar**. |
| 5 | Alias vs RazonSocial | Negocio | Confirmar dato fuente |
| ~~6~~ | ~~Estructura tabla legacy `consecutivo`~~ | — | **[Resuelto]** El folio de Factura México proviene de `EmpresaFolio` (PQF2/Finanzas) con UPDLOCK atómico — sin dependencia de Legacy. Ver `Analisis/Foliados-Documentos.md`, sección Factura. |
| 7 | Nivel de configuración ClaveProdServ/ClaveUnidad/PerfilFiscal | Negocio/Técnico | Confirmar si cada campo se configura a nivel Producto, Familia, o con precedencia Producto→Familia — y si los 3 campos comparten el mismo nivel o cada uno puede tener el suyo (ver sección "Datos del producto") |
| 8 | IEPS — confirmar si algún producto lo requiere | Negocio | Solo agregar la 4ª fila de PerfilFiscal (IEPS) si PROQUIFA confirma que algún producto lo requiere; no crear sin esa confirmación |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-018 | Pantalla inicial + creacion ProquifaDotNet.Timbrado (servicio tecnico) + CfdiController/extension CFDIGenerada en Finanzas |
| R16A-RE-FU-012 | Genera pendiente FAA (`fccFactura`, origen Credito) |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria`/`vfccFactura` (incluye columnas `Enviada`/`IdCFDIGenerada` migradas desde este requisito) — genera pendiente FAA de Prepago |
| R16A-RE-FU-016 | Patron persistencia Minio |

> Este requisito (RE-FU-019) es el que ejecuta el `CREATE TABLE CFDIGenerada` (ER-Finanzas.md) referenciado como prerequisito por R16A-RE-FU-018_BD.md (Parte 3, extension de columnas) y R16A-RE-FU-028 (catTipoCFDI).

---

**Generado por:** GitHub Copilot in SSMS
**Bases de Datos:** ProquifaDotNet + ProquifaDotNetTimbrado
