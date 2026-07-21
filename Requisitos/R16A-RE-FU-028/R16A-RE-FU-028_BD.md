# Impacto en BD — Validar Cobro: Paso 3 México (Facturación y Envío)
**Requisito:** R16A-RE-FU-028
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (lectura)
**Versión:** 1.1

---

## Resumen

Paso 3 del wizard Validar Cobro México: timbrado y envío individual de documentos fiscales
por cada línea derivada de la asociación cerrada en el Paso 2. El sistema determina por
línea el tipo de CFDI a generar (Factura PUE, Factura PPD + Complemento en cascada,
Factura Anticipo, o solo Complemento de Pago desde FAA existente). Post-envío dispara
automáticamente FEE, transferencia a Legacy y Confirmación de Pedido.

Tablas nuevas: `fccDocumentoFiscalCobro` (control de estado por línea) y `fccConfirmacionPedido`
(documento de confirmación prepago). Tablas alteradas: `CFDIGenerada` (discriminador de
tipo + referencia al CFDI relacionado) y condicionalmente `tpPedido` (FEE).

> **v1.1 — Refinamiento de diseño:** `fccDocumentoFiscalCobro` apunta directamente a los registros
> de asociación del Paso 2 (`fccPagoFacturaPedido` o `fccPagoFacturaAdelanto`) en lugar
> de duplicar las llaves `(IdFCCPagoCliente + IdTPProforma*)`. Los CFDIs generados
> (incluyendo el Complemento en cascada) se rastrean únicamente desde `fccDocumentoFiscalCobro`,
> por lo que no se requieren ALTER adicionales en las tablas del Paso 2.

---

## Impacto en BD

| #   | Cambio                                                                          | Base de Datos             | Tipo      | Prioridad |
| --- | ------------------------------------------------------------------------------- | ------------------------- | --------- | --------- |
| 1   | CREATE TABLE catTipoDocumentoFiscal                                             | ProquifaDotNet            | DDL       | Alta      |
| 2   | CREATE TABLE catDocumentoFiscalCobroEstado                                      | ProquifaDotNet            | DDL       | Alta      |
| 3   | CREATE TABLE catTipoCFDI                                                        | ProquifaDotNet            | DDL       | Alta      |
| 4   | CREATE TABLE fccDocumentoFiscalCobro                                            | ProquifaDotNet            | DDL       | Alta      |
| 5   | CREATE TABLE fccConfirmacionPedido                                              | ProquifaDotNet            | DDL       | Alta      |
| 6   | ALTER TABLE CFDIGenerada ADD IdCatTipoCFDI (FK) + IdCFDIRelacionado             | ProquifaDotNet            | DDL       | Alta      |
| 7   | ALTER TABLE tpPedido ADD FechaEstimadaEntrega (si no existe)                    | ProquifaDotNet            | DDL       | Alta      |
| 8   | CREATE VIEW vfccDocumentoFiscalCobro                                            | ProquifaDotNet            | DDL       | Media     |
| 8   | Reutiliza: CFDIGenerada (timbrado FAA RE-FU-019)                                | ProquifaDotNet            | Existente | —         |
| 9   | Reutiliza: EmpresaFolio (foliador por empresa RE-FU-019)                        | ProquifaDotNet (Finanzas) | Existente | —         |
| 10  | Reutiliza: fccPagoFacturaPedido (cobro ↔ proforma, RE-FU-026)                   | ProquifaDotNet            | Existente | —         |
| 11  | Reutiliza: fccPagoFacturaAdelanto (cobro ↔ FAA, RE-FU-026)                      | ProquifaDotNet            | Existente | —         |
| 12  | Reutiliza: tpProformaPedido.IdCFDIGenerada (ya declarado en RE-FU-026)          | ProquifaDotNet            | Existente | —         |
| 13  | Reutiliza: fccNotaCredito.IdCFDI (UUID NC para nodo CFDIRelacionados RE-FU-026) | ProquifaDotNet            | Existente | —         |

---

## Catálogo Nuevo: catTipoDocumentoFiscal

**Descripción:** Catálogo de tipos de documento fiscal que puede generarse en el Paso 3 del wizard Validar Cobro. Determina el tipo de CFDI a emitir por línea según el origen de la asociación del Paso 2.

```sql
CREATE TABLE [dbo].[catTipoDocumentoFiscal](
    [IdCatTipoDocumentoFiscal]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catTipoDocumentoFiscal_Id]    DEFAULT (NEWID()),
    [Clave]                     varchar(30)      NOT NULL,
        -- 'FACTURA'          -> CFDI Ingreso PUE o PPD (proforma sin controlados)
        -- 'FACTURA_ANTICIPO' -> CFDI Ingreso rel. 07 SAT (proforma con controlados)
        -- 'COMPLEMENTO_PAGO' -> CFDI Pagos 2.0 (FAA existente con cobro asociado)
    [Descripcion]               nvarchar(150)    NOT NULL,
    [Activo]                    bit              NOT NULL
        CONSTRAINT [DF_catTipoDocumentoFiscal_Activo] DEFAULT (1),
    [FechaRegistro]             datetime2(7)     NOT NULL
        CONSTRAINT [DF_catTipoDocumentoFiscal_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catTipoDocumentoFiscal]
        PRIMARY KEY CLUSTERED ([IdCatTipoDocumentoFiscal]),
    CONSTRAINT [UQ_catTipoDocumentoFiscal_Clave]
        UNIQUE ([Clave])
);
GO
-- Datos iniciales
INSERT INTO dbo.catTipoDocumentoFiscal (Clave, Descripcion) VALUES
    ('FACTURA',          'Factura — CFDI Ingreso (proforma sin productos controlados)'),
    ('FACTURA_ANTICIPO', 'Factura Anticipo — CFDI Ingreso rel. 07 SAT (proforma con productos controlados)'),
    ('COMPLEMENTO_PAGO', 'Complemento de Pago — CFDI Pagos 2.0 (Factura por Adelanto existente)');
```

### Diccionario de datos — catTipoDocumentoFiscal

| Nombre de tabla | Descripción |
|-----------------|-------------|
| catTipoDocumentoFiscal | Catálogo de tipos de CFDI generables en el Paso 3 de Validar Cobro. |

**Columnas:**

| Nombre | Tipo | Descripción |
|--------|------|-------------|
| IdCatTipoDocumentoFiscal | uniqueidentifier PK | Identificador único del catálogo |
| Clave | varchar(30) NOT NULL UNIQUE | Clave técnica del tipo: `FACTURA`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO` |
| Descripcion | nvarchar(150) NOT NULL | Descripción legible para el usuario |
| Activo | bit NOT NULL | 1 = vigente |
| FechaRegistro | datetime2(7) NOT NULL | Fecha de inserción |

**Índices:**

| Nombre | Columnas | Tipo |
|--------|----------|------|
| PK_catTipoDocumentoFiscal | IdCatTipoDocumentoFiscal | PRIMARY KEY CLUSTERED |
| UQ_catTipoDocumentoFiscal_Clave | Clave | UNIQUE non-clustered |

---

## Catálogo Nuevo: catDocumentoFiscalCobroEstado

**Descripción:** Catálogo de estados posibles de una línea del Paso 3 del wizard Validar Cobro. Controla el ciclo de vida de cada documento fiscal desde su creación hasta el envío al cliente.

```sql
CREATE TABLE [dbo].[catDocumentoFiscalCobroEstado](
    [IdCatDocumentoFiscalCobroEstado]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catDocumentoFiscalCobroEstado_Id]    DEFAULT (NEWID()),
    [Clave]                            varchar(20)      NOT NULL,
        -- 'PENDIENTE' -> Estado inicial; aún no se ha timbrado ni enviado
        -- 'GENERADO'  -> CFDIs timbrados exitosamente; pendiente de envío al cliente
        -- 'ENVIADO'   -> Enviado al cliente; línea cerrada operativamente
    [Descripcion]                      nvarchar(150)    NOT NULL,
    [Activo]                           bit              NOT NULL
        CONSTRAINT [DF_catDocumentoFiscalCobroEstado_Activo] DEFAULT (1),
    [FechaRegistro]                    datetime2(7)     NOT NULL
        CONSTRAINT [DF_catDocumentoFiscalCobroEstado_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catDocumentoFiscalCobroEstado]
        PRIMARY KEY CLUSTERED ([IdCatDocumentoFiscalCobroEstado]),
    CONSTRAINT [UQ_catDocumentoFiscalCobroEstado_Clave]
        UNIQUE ([Clave])
);
GO
-- Datos iniciales
INSERT INTO dbo.catDocumentoFiscalCobroEstado (Clave, Descripcion) VALUES
    ('PENDIENTE', 'Pendiente — línea creada, aún no timbrada ni enviada'),
    ('GENERADO',  'Generado — CFDIs timbrados exitosamente, pendiente de envío al cliente'),
    ('ENVIADO',   'Enviado — documentos enviados al cliente, línea cerrada operativamente');
```

### Diccionario de datos — catDocumentoFiscalCobroEstado

| Nombre de tabla | Descripción |
|-----------------|-------------|
| catDocumentoFiscalCobroEstado | Catálogo de estados del ciclo de vida de las líneas del Paso 3 de Validar Cobro. |

**Columnas:**

| Nombre | Tipo | Descripción |
|--------|------|-------------|
| IdCatDocumentoFiscalCobroEstado | uniqueidentifier PK | Identificador único del catálogo |
| Clave | varchar(20) NOT NULL UNIQUE | Clave técnica del estado: `PENDIENTE`, `GENERADO`, `ENVIADO` |
| Descripcion | nvarchar(150) NOT NULL | Descripción legible del estado |
| Activo | bit NOT NULL | 1 = vigente |
| FechaRegistro | datetime2(7) NOT NULL | Fecha de inserción |

**Índices:**

| Nombre | Columnas | Tipo |
|--------|----------|------|
| PK_catDocumentoFiscalCobroEstado | IdCatDocumentoFiscalCobroEstado | PRIMARY KEY CLUSTERED |
| UQ_catDocumentoFiscalCobroEstado_Clave | Clave | UNIQUE non-clustered |

---

## Tabla Nueva: fccDocumentoFiscalCobro

**Descripción:** Registra el estado de cada línea del Paso 3 del wizard Validar Cobro para
México. Una línea representa un documento fiscal a generar (Factura, Factura Anticipo o
Complemento de Pago) derivado de la asociación cobro ↔ proforma/FAA cerrada en el Paso 2.
Apunta directamente al registro de asociación del Paso 2 (`fccPagoFacturaPedido` o
`fccPagoFacturaAdelanto`) para evitar duplicar llaves. Persiste el estado entre sesiones y
almacena las selecciones del usuario (UsoCFDI, MetodoPago) hasta el cierre del wizard.

```sql
-- Prerequisito: catTipoDocumentoFiscal y catDocumentoFiscalCobroEstado deben existir
CREATE TABLE [dbo].[fccDocumentoFiscalCobro](
    [IdFCCDocumentoFiscalCobro]              uniqueidentifier NOT NULL
        CONSTRAINT [DF_fccDocumentoFiscalCobro_Id]             DEFAULT (NEWID()),
    -- Origen: exactamente UNO de los dos siguientes debe ser NOT NULL (CK garantiza exclusividad)
    [IdFCCPagoFacturaPedido]                 uniqueidentifier NULL,
        -- NOT NULL cuando el origen es proforma normal o con controlados (RE-FU-026)
    [IdFCCPagoFacturaAdelanto]               uniqueidentifier NULL,
        -- NOT NULL cuando el origen es FAA existente con cobro asociado (RE-FU-026)
    -- Tipo de documento fiscal a generar (determinado al crear la línea)
    [IdCatTipoDocumentoFiscal]               uniqueidentifier NOT NULL,
        -- FK a catTipoDocumentoFiscal: FACTURA | FACTURA_ANTICIPO | COMPLEMENTO_PAGO
    [IdCatDocumentoFiscalCobroEstado]        uniqueidentifier NOT NULL,
        -- FK a catDocumentoFiscalCobroEstado: PENDIENTE | GENERADO | ENVIADO
    [IdCatUsoCFDI]                           uniqueidentifier NULL,   -- FK catUsoCFDI (catálogo SAT c_UsoCFDI)
    [IdCatMetodoDePagoCFDI]                  uniqueidentifier NULL,   -- FK catMetodoDePagoCFDI; NULL para COMPLEMENTO_PAGO
    [IdCFDIGeneradaFactura]                  uniqueidentifier NULL,   -- FK CFDIGenerada: Factura/Anticipo/Complemento FAA timbrado
    [IdCFDIGeneradaComplemento]              uniqueidentifier NULL,   -- FK CFDIGenerada: Complemento cascada PPD (solo MetodoPago=PPD)
    [FechaGeneracion]                        datetime2(7)     NULL,
    [FechaEnvio]                             datetime2(7)     NULL,
    [Activo]                                 bit              NOT NULL
        CONSTRAINT [DF_fccDocumentoFiscalCobro_Activo]         DEFAULT (1),
    [FechaRegistro]                          datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccDocumentoFiscalCobro_FechaReg]       DEFAULT (SYSUTCDATETIME()),
    [FechaUltimaActualizacion]               datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccDocumentoFiscalCobro_FechaUpd]       DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_fccDocumentoFiscalCobro]
        PRIMARY KEY CLUSTERED ([IdFCCDocumentoFiscalCobro]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_PagoFacturaPedido]
        FOREIGN KEY ([IdFCCPagoFacturaPedido])
        REFERENCES dbo.fccPagoFacturaPedido([IdFCCPagoFacturaPedido]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_PagoFacturaAdelanto]
        FOREIGN KEY ([IdFCCPagoFacturaAdelanto])
        REFERENCES dbo.fccPagoFacturaAdelanto([IdFCCPagoFacturaAdelanto]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_TipoDocumentoFiscal]
        FOREIGN KEY ([IdCatTipoDocumentoFiscal])
        REFERENCES dbo.catTipoDocumentoFiscal([IdCatTipoDocumentoFiscal]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_Estado]
        FOREIGN KEY ([IdCatDocumentoFiscalCobroEstado])
        REFERENCES dbo.catDocumentoFiscalCobroEstado([IdCatDocumentoFiscalCobroEstado]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_UsoCFDI]
        FOREIGN KEY ([IdCatUsoCFDI])
        REFERENCES dbo.catUsoCFDI([IdCatUsoCFDI]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_MetodoPago]
        FOREIGN KEY ([IdCatMetodoDePagoCFDI])
        REFERENCES dbo.catMetodoDePagoCFDI([IdCatMetodoDePagoCFDI]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_CFDIFactura]
        FOREIGN KEY ([IdCFDIGeneradaFactura])
        REFERENCES dbo.CFDIGenerada([IdCFDIGenerada]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_CFDIComplemento]
        FOREIGN KEY ([IdCFDIGeneradaComplemento])
        REFERENCES dbo.CFDIGenerada([IdCFDIGenerada]),
    CONSTRAINT [CK_fccDocumentoFiscalCobro_OrigenExclusivo]
        CHECK (
            (IdFCCPagoFacturaPedido IS NOT NULL AND IdFCCPagoFacturaAdelanto IS NULL)
            OR
            (IdFCCPagoFacturaPedido IS NULL  AND IdFCCPagoFacturaAdelanto IS NOT NULL)
        )
);
GO
CREATE INDEX [IX_fccDocumentoFiscalCobro_PagoFacturaPedido]
    ON dbo.fccDocumentoFiscalCobro ([IdFCCPagoFacturaPedido])
    WHERE [IdFCCPagoFacturaPedido] IS NOT NULL;
CREATE INDEX [IX_fccDocumentoFiscalCobro_PagoFacturaAdelanto]
    ON dbo.fccDocumentoFiscalCobro ([IdFCCPagoFacturaAdelanto])
    WHERE [IdFCCPagoFacturaAdelanto] IS NOT NULL;
```

### Diccionario de datos — fccDocumentoFiscalCobro

| Nombre de tabla | Descripción |
|-----------------|-------------|
| fccDocumentoFiscalCobro | Líneas del Paso 3 del wizard Validar Cobro México. Una línea por cada documento fiscal a generar derivado de la asociación del Paso 2. Apunta al registro de asociación del Paso 2 para evitar duplicar llaves. |

**Columnas:**

| Nombre                          | Tipo                         | Descripción                                                                                                                                                                            |
| ------------------------------- | ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IdFCCDocumentoFiscalCobro       | uniqueidentifier PK          | Identificador único de la línea                                                                                                                                                        |
| IdFCCPagoFacturaPedido          | uniqueidentifier FK NULL     | FK a `fccPagoFacturaPedido`; NOT NULL cuando el origen es proforma (normal o con controlados). Provee acceso a `IdFCCPagoCliente` e `IdTPProformaPedido` sin duplicarlos.              |
| IdFCCPagoFacturaAdelanto        | uniqueidentifier FK NULL     | FK a `fccPagoFacturaAdelanto`; NOT NULL cuando el origen es FAA. Provee acceso a `IdFCCPagoCliente`, `IdFccFactura` (RE-FU-015, antes `IdTPProformaAdelanto`) e `IdCFDIGenerada` (UUID de la FAA para CFDIRelacionados). |
| IdCatTipoDocumentoFiscal        | uniqueidentifier FK NOT NULL | FK a `catTipoDocumentoFiscal`. Tipo de CFDI a generar: `FACTURA`, `FACTURA_ANTICIPO` o `COMPLEMENTO_PAGO`. Determinado al crear la línea.                                              |
| IdCatDocumentoFiscalCobroEstado | uniqueidentifier FK NOT NULL | FK a `catDocumentoFiscalCobroEstado`. Estado actual de la línea: `PENDIENTE`, `GENERADO` o `ENVIADO`.                                                                                  |
| IdCatUsoCFDI                    | uniqueidentifier FK NULL     | FK a `catUsoCFDI`. Uso CFDI seleccionado por el usuario antes del timbrado. Default: Uso CFDI configurado en el cliente o en el pedido original.                                       |
| IdCatMetodoDePagoCFDI           | uniqueidentifier FK NULL     | FK a `catMetodoDePagoCFDI`. Método de pago (PPD / PUE). NULL para líneas `COMPLEMENTO_PAGO` (el método PPD es fijo y se infiere del catálogo tipo, no se persiste aquí).               |
| IdCFDIGeneradaFactura           | uniqueidentifier FK NULL     | FK a `CFDIGenerada`: Factura, Factura Anticipo, o Complemento directo desde FAA. Se popula al timbrar exitosamente.                                                                    |
| IdCFDIGeneradaComplemento       | uniqueidentifier FK NULL     | FK a `CFDIGenerada`: Complemento de Pago generado en cascada cuando `MetodoPago = 'PPD'`. NULL para `COMPLEMENTO_PAGO` (ese CFDI va en `IdCFDIGeneradaFactura`) y para PUE.            |
| FechaGeneracion                 | datetime2(7) NULL            | Fecha/hora del timbrado exitoso de los CFDIs de la línea.                                                                                                                              |
| FechaEnvio                      | datetime2(7) NULL            | Fecha/hora del envío exitoso al cliente.                                                                                                                                               |
| Activo                          | bit NOT NULL                 | 1 = activo. Permanece 1 incluso en estados Generado y Enviado para trazabilidad histórica.                                                                                             |
| FechaRegistro                   | datetime2(7) NOT NULL        | Fecha de creación del registro.                                                                                                                                                        |
| FechaUltimaActualizacion        | datetime2(7) NOT NULL        | Fecha de última modificación.                                                                                                                                                          |
|                                 |                              |                                                                                                                                                                                        |

**Relaciones:**

| Tabla relacionada | Tipo | FK |
|-------------------|------|-----|
| fccPagoFacturaPedido | N:1 (nullable) | IdFCCPagoFacturaPedido |
| fccPagoFacturaAdelanto | N:1 (nullable) | IdFCCPagoFacturaAdelanto |
| catTipoDocumentoFiscal | N:1 | IdCatTipoDocumentoFiscal |
| catDocumentoFiscalCobroEstado | N:1 | IdCatDocumentoFiscalCobroEstado |
| catUsoCFDI | N:1 (nullable) | IdCatUsoCFDI |
| catMetodoDePagoCFDI | N:1 (nullable) | IdCatMetodoDePagoCFDI |
| CFDIGenerada | N:1 (nullable, x2) | IdCFDIGeneradaFactura / IdCFDIGeneradaComplemento |

**Índices:**

| Nombre | Columnas | Tipo |
|--------|----------|------|
| PK_fccDocumentoFiscalCobro | IdFCCDocumentoFiscalCobro | PRIMARY KEY CLUSTERED |
| IX_fccDocumentoFiscalCobro_PagoFacturaPedido | IdFCCPagoFacturaPedido (WHERE NOT NULL) | Filtered non-clustered |
| IX_fccDocumentoFiscalCobro_PagoFacturaAdelanto | IdFCCPagoFacturaAdelanto (WHERE NOT NULL) | Filtered non-clustered |

**Consideraciones especiales:**
- `CHECK CONSTRAINT CK_fccDocumentoFiscalCobro_OrigenExclusivo`: garantiza que una línea apunte a exactamente un origen (proforma O FAA, no ambos ni ninguno).
- `IdCFDIGeneradaComplemento` solo se popula en el escenario PPD cascada (proforma con `IdCatMetodoDePagoCFDI` = PPD). Para el Complemento directo de FAA, el CFDI va en `IdCFDIGeneradaFactura`.
- Los datos del cobro (`IdFCCPagoCliente`, `Monto`, `TipoDeCambio`, etc.) se obtienen navegando desde la FK del Paso 2 (`fccPagoFacturaPedido.IdFCCPagoCliente` o `fccPagoFacturaAdelanto.IdFCCPagoCliente`). No se duplican en esta tabla.
- `IdCatTipoDocumentoFiscal` se determina al crear la línea: si el origen es `fccPagoFacturaPedido` → leer `tpProformaPedido.HayControlados` para resolver la clave `FACTURA` vs `FACTURA_ANTICIPO` en `catTipoDocumentoFiscal`; si el origen es `fccPagoFacturaAdelanto` → siempre clave `COMPLEMENTO_PAGO`.
- `IdCatDocumentoFiscalCobroEstado` se inicializa con la clave `PENDIENTE` al crear la línea. Se actualiza a `GENERADO` tras timbrado exitoso y a `ENVIADO` tras envío exitoso.

---

## Tabla Nueva: fccConfirmacionPedido

**Descripción:** Almacena las Confirmaciones de Pedido generadas automáticamente al enviar
cada línea del Paso 3 para pedidos Prepago México. Concepto preexistente en ProquifaNet
para Crédito, extendido a Prepago en R16. El PDF se almacena en Minio; la ruta se guarda
en esta tabla. Se adjunta en el mismo correo de envío de la línea.

```sql
CREATE TABLE [dbo].[fccConfirmacionPedido](
    [IdFCCConfirmacionPedido]   uniqueidentifier NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_Id]      DEFAULT (NEWID()),
    [IdFCCDocumentoFiscalCobro]           uniqueidentifier NOT NULL,
    [IdTPPedido]                uniqueidentifier NOT NULL,
    [FolioConfirmacion]         varchar(80)      NOT NULL,
    [RutaArchivoPDF]            nvarchar(500)    NULL,
    [FechaGeneracion]           datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_FechaGen] DEFAULT (SYSUTCDATETIME()),
    [Activo]                    bit              NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_Activo]  DEFAULT (1),
    [FechaRegistro]             datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_FechaReg] DEFAULT (SYSUTCDATETIME()),
    [FechaUltimaActualizacion]  datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_FechaUpd] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_fccConfirmacionPedido]
        PRIMARY KEY CLUSTERED ([IdFCCConfirmacionPedido]),
    CONSTRAINT [FK_fccConfirmacionPedido_DocumentoFiscalCobro]
        FOREIGN KEY ([IdFCCDocumentoFiscalCobro])
        REFERENCES dbo.fccDocumentoFiscalCobro([IdFCCDocumentoFiscalCobro]),
    CONSTRAINT [FK_fccConfirmacionPedido_Pedido]
        FOREIGN KEY ([IdTPPedido])
        REFERENCES dbo.tpPedido([IdTPPedido])
);
GO
CREATE INDEX [IX_fccConfirmacionPedido_Pedido]
    ON dbo.fccConfirmacionPedido ([IdTPPedido]);
```

### Diccionario de datos — fccConfirmacionPedido

| Nombre de tabla       | Descripción                                                                                                                       |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| fccConfirmacionPedido | Confirmaciones de Pedido generadas en el Paso 3 de Validar Cobro para pedidos Prepago México. Una Confirmación por línea enviada. |

**Columnas:**

| Nombre | Tipo | Descripción |
|--------|------|-------------|
| IdFCCConfirmacionPedido | uniqueidentifier PK | Identificador único |
| IdFCCDocumentoFiscalCobro | uniqueidentifier FK NOT NULL | Línea del Paso 3 que origina la Confirmación |
| IdTPPedido | uniqueidentifier FK NOT NULL | Pedido al que corresponde la Confirmación |
| FolioConfirmacion | varchar(80) NOT NULL | Folio del documento (formato pendiente de definición — Gap G3) |
| RutaArchivoPDF | nvarchar(500) NULL | Ruta del PDF en Minio (bucket `confirmaciones`). NULL mientras no se haya generado. |
| FechaGeneracion | datetime2(7) NOT NULL | Fecha/hora de generación del documento |
| Activo | bit NOT NULL | 1 = activo |
| FechaRegistro | datetime2(7) NOT NULL | Fecha de inserción |
| FechaUltimaActualizacion | datetime2(7) NOT NULL | Fecha de última modificación |

**Relaciones:**

| Tabla relacionada | Tipo | FK |
|-------------------|------|-----|
| fccDocumentoFiscalCobro | N:1 | IdFCCDocumentoFiscalCobro |
| tpPedido | N:1 | IdTPPedido |

**Índices:**

| Nombre | Columnas | Tipo |
|--------|----------|------|
| PK_fccConfirmacionPedido | IdFCCConfirmacionPedido | PRIMARY KEY CLUSTERED |
| IX_fccConfirmacionPedido_Pedido | IdTPPedido | Non-clustered |

**Consideraciones especiales:**
- Se genera exclusivamente en envíos México (Perú no genera Confirmación de Pedido en R16).
- El PDF se genera vía DocumentBuilder y se almacena en Minio antes de armar el correo.
- `RutaArchivoPDF` puede quedar NULL si el proceso falla antes del envío; en ese caso la línea permanece en estado `GENERADO` y se reintenta al volver a enviar.
- El formato de `FolioConfirmacion` es pendiente de confirmar con el equipo de negocio (PMO).

---

## Catálogo Nuevo: catTipoCFDI

**Descripción:** Catálogo de tipos de CFDI timbrados y almacenados en `CFDIGenerada`. Discrimina el tipo de comprobante fiscal generado para facilitar consultas, auditoría y lógica de negocio posterior.

```sql
CREATE TABLE [dbo].[catTipoCFDI](
    [IdCatTipoCFDI]     uniqueidentifier NOT NULL
        CONSTRAINT [DF_catTipoCFDI_Id]       DEFAULT (NEWID()),
    [Clave]             varchar(20)      NOT NULL,
        -- 'FACTURA_PPD'       -> Factura generada con método PPD
        -- 'FACTURA_PUE'       -> Factura generada con método PUE
        -- 'FACTURA_ANTICIPO'  -> Factura Anticipo tipo relación 07 SAT (controlados)
        -- 'COMPLEMENTO_PAGO'  -> CFDI Pagos 2.0
    [Descripcion]       nvarchar(150)    NOT NULL,
    [Activo]            bit              NOT NULL
        CONSTRAINT [DF_catTipoCFDI_Activo]   DEFAULT (1),
    [FechaRegistro]     datetime2(7)     NOT NULL
        CONSTRAINT [DF_catTipoCFDI_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catTipoCFDI]
        PRIMARY KEY CLUSTERED ([IdCatTipoCFDI]),
    CONSTRAINT [UQ_catTipoCFDI_Clave]
        UNIQUE ([Clave])
);
GO
-- Datos iniciales
INSERT INTO dbo.catTipoCFDI (Clave, Descripcion) VALUES
    ('FACTURA_PPD',      'Factura — CFDI Ingreso con método de pago PPD (Pago en parcialidades o diferido)'),
    ('FACTURA_PUE',      'Factura — CFDI Ingreso con método de pago PUE (Pago en una sola exhibición)'),
    ('FACTURA_ANTICIPO', 'Factura Anticipo — CFDI Ingreso con tipo de relación 07 SAT (productos controlados)'),
    ('COMPLEMENTO_PAGO', 'Complemento de Pago — CFDI Pagos 2.0');
```

### Diccionario de datos — catTipoCFDI

| Nombre de tabla | Descripción |
|-----------------|-------------|
| catTipoCFDI | Catálogo de tipos de CFDI. Discrimina el comprobante fiscal almacenado en `CFDIGenerada`. |

**Columnas:**

| Nombre | Tipo | Descripción |
|--------|------|-------------|
| IdCatTipoCFDI | uniqueidentifier PK | Identificador único del catálogo |
| Clave | varchar(20) NOT NULL UNIQUE | Clave técnica: `FACTURA_PPD`, `FACTURA_PUE`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO` |
| Descripcion | nvarchar(150) NOT NULL | Descripción legible del tipo de CFDI |
| Activo | bit NOT NULL | 1 = vigente |
| FechaRegistro | datetime2(7) NOT NULL | Fecha de inserción |

**Índices:**

| Nombre | Columnas | Tipo |
|--------|----------|------|
| PK_catTipoCFDI | IdCatTipoCFDI | PRIMARY KEY CLUSTERED |
| UQ_catTipoCFDI_Clave | Clave | UNIQUE non-clustered |

---

## ALTER TABLE CFDIGenerada

La tabla `CFDIGenerada` (creada en RE-FU-019) es el registro central de todos los CFDIs
timbrados. En RE-FU-028 se requiere diferenciar el tipo de CFDI y relacionar explícitamente
un Complemento de Pago con la Factura PPD a la que complementa.

```sql
-- Prerequisito: catTipoCFDI debe existir
-- Ejecutar en ProquifaDotNet
ALTER TABLE dbo.CFDIGenerada
    ADD IdCatTipoCFDI uniqueidentifier NULL;
        -- FK a catTipoCFDI: FACTURA_PPD | FACTURA_PUE | FACTURA_ANTICIPO | COMPLEMENTO_PAGO
        -- NULL en registros previos (FAA generadas antes de RE-FU-028); normalizar con UPDATE posterior

ALTER TABLE dbo.CFDIGenerada
    ADD CONSTRAINT [FK_CFDIGenerada_TipoCFDI]
        FOREIGN KEY ([IdCatTipoCFDI])
        REFERENCES dbo.catTipoCFDI([IdCatTipoCFDI]);

ALTER TABLE dbo.CFDIGenerada
    ADD IdCFDIRelacionado uniqueidentifier NULL;
        -- Para IdCatTipoCFDI -> 'COMPLEMENTO_PAGO': referencia al IdCFDIGenerada de la Factura PPD relacionada
        -- Para IdCatTipoCFDI -> 'FACTURA_ANTICIPO': NULL (la relación tipo 07 se arma en XML al timbrar)
        -- NULL para FACTURA_PPD y FACTURA_PUE

-- FK blanda (self-referencia); no se declara como FOREIGN KEY de BD para evitar
-- restricciones de inserción cuando el Complemento se inserta en la misma transacción
-- que la Factura PPD. La integridad se garantiza a nivel de servicio (Finanzas/Timbrado).

-- Script de normalización de registros existentes (FAA generadas en RE-FU-019)
-- Ejecutar después del ALTER; ajustar según lógica real de identificación de FAAs previas
UPDATE dbo.CFDIGenerada
    SET IdCatTipoCFDI = (SELECT IdCatTipoCFDI FROM dbo.catTipoCFDI WHERE Clave = 'FACTURA_PPD')
WHERE IdCatTipoCFDI IS NULL;
-- NOTA: validar si las FAA previas son PPD o PUE antes de ejecutar; ajustar el UPDATE según corresponda.
```

| Campo nuevo | Tipo | Default | Descripción |
|-------------|------|---------|-------------|
| IdCatTipoCFDI | uniqueidentifier FK NULL | NULL | FK a `catTipoCFDI`. Discriminador de tipo de CFDI. Se popula al timbrar en Paso 3. Registros existentes (FAA RE-FU-019) quedan NULL hasta ejecutar el script de normalización. |
| IdCFDIRelacionado | uniqueidentifier NULL | NULL | Referencia al `IdCFDIGenerada` de la Factura PPD que este Complemento complementa. NULL para todos los demás tipos. |

---

## ALTER TABLE tpPedido (condicional)

Verificar si el campo `FechaEstimadaEntrega` ya existe en `tpPedido` antes de ejecutar.
La FEE se establece automáticamente al confirmar el envío de cada línea del Paso 3 (solo México).

> **Nota de diseño — FEE en `tpPedido` vs. `tpPartidaPedido`:**
> `tpPartidaPedido` ya contiene `FechaEstimadaEntrega` a nivel de partida/línea de producto,
> calculada al tramitar con base en disponibilidad de stock (RE-FU-010/011/013/014).
> El campo propuesto en `tpPedido` es conceptualmente distinto: es la **FEE oficial confirmada
> post-pago** que se comunica al cliente en la Confirmación de Pedido y se transfiere a Legacy
> como cabecera del pedido. Se propone mantener ambos campos con propósitos diferenciados:
>
> | Campo | Nivel | Momento | Propósito |
> |-------|-------|---------|-----------|
> | `tpPartidaPedido.FechaEstimadaEntrega` | Partida | Al tramitar | FEE inicial por producto/stock |
> | `tpPedido.FechaEstimadaEntrega` | Cabecera | Al enviar Paso 3 | FEE confirmada post-pago → Confirmación de Pedido + Legacy |
>
> ⚠️ **Requiere confirmar con negocio** (parte de Brecha B4): si un pedido tiene partidas con FEEs
> distintas, ¿qué valor se establece en la cabecera? (¿la mayor?, ¿una fecha única por regla de
> negocio?). Esta decisión también determina si `tpPedido.FechaEstimadaEntrega` es necesaria o
> si basta con actualizar `tpPartidaPedido.FechaEstimadaEntrega` en el Paso 3.

```sql
-- CONDICIONAL: ejecutar solo si la columna no existe
IF NOT EXISTS (
    SELECT 1 FROM sys.columns
    WHERE object_id = OBJECT_ID('dbo.tpPedido')
      AND name = 'FechaEstimadaEntrega'
)
BEGIN
    ALTER TABLE dbo.tpPedido
        ADD FechaEstimadaEntrega datetime2(7) NULL;
END
```

| Campo nuevo | Tipo | Default | Descripción |
|-------------|------|---------|-------------|
| FechaEstimadaEntrega | datetime2(7) NULL | NULL | FEE confirmada post-pago. Se establece en el Paso 3 al enviar exitosamente la primera línea del pedido. Cabecera para Confirmación de Pedido y transferencia Legacy. Solo aplica a México. Distinto de `tpPartidaPedido.FechaEstimadaEntrega` (FEE por producto al tramitar). |

---

## CREATE VIEW vfccDocumentoFiscalCobro

Vista operativa que consolida el estado del Paso 3 por cliente. Navega desde `fccDocumentoFiscalCobro`
hacia los registros de asociación del Paso 2 para exponer en una sola consulta los datos
del cobro, documento origen, pedido y CFDIs generados.

```sql
-- Ejecutar DESPUES de CREATE TABLE fccDocumentoFiscalCobro y ALTERs de CFDIGenerada
CREATE VIEW [dbo].[vfccDocumentoFiscalCobro]
AS
SELECT
    p3l.IdFCCDocumentoFiscalCobro,
    p3l.TipoDocumentoFiscal,
    p3l.EstadoLinea,
    p3l.IdCatUsoCFDI,
    ufo.Clave                   AS UsoCFDIClave,
    ufo.Descripcion             AS UsoCFDIDescripcion,
    p3l.IdCatMetodoDePagoCFDI,
    mpc.Clave                   AS MetodoPagoClave,
    mpc.Descripcion             AS MetodoPagoDescripcion,
    -- Origen proforma (cuando IdFCCPagoFacturaPedido IS NOT NULL)
    p3l.IdFCCPagoFacturaPedido,
    pfp.IdFCCPagoCliente        AS IdFCCPagoCliente_PFP,
    pfp.IdTPProformaPedido,
    pp.Folio                    AS FolioProforma,
    pp.MontoTotal               AS MontoProforma,
    pp.MontoPendiente,
    e_pp.Prefijo                AS EmpresaEmisoraProforma,
    -- Origen FAA (cuando IdFCCPagoFacturaAdelanto IS NOT NULL)
    p3l.IdFCCPagoFacturaAdelanto,
    pfa.IdFCCPagoCliente        AS IdFCCPagoCliente_PFA,
    pfa.IdFccFactura,           -- RE-FU-015 (antes: pfa.IdTPProformaAdelanto)
    fc.MontoTotal                AS MontoFAA,
    cg_faa.UUID                 AS UUID_FAA,        -- UUID de la FAA para CFDIRelacionados
    -- Cobro (fuente de verdad: tabla de asociación Paso 2)
    COALESCE(pfp.IdFCCPagoCliente, pfa.IdFCCPagoCliente) AS IdFCCPagoCliente,
    fpc.Folio                   AS FolioCobro,
    fpc.Monto                   AS MontoCobro,
    fpc.MXN                     AS CobroMXN,
    fpc.USD                     AS CobroUSD,
    fpc.TipoDeCambio,
    fpc.IdCliente,
    c.Nombre                    AS ClienteNombre,
    dfc.RazonSocial             AS ClienteRazonSocial,
    dfc.RFC                     AS ClienteRFC,
    -- Pedido (via cadena proforma → pedido)
    tp.IdTPPedido,
    tp.FolioPedidoInterno,
    tp.FechaEstimadaEntrega,
    -- CFDIs generados
    cg_f.IdCFDIGenerada         AS IdCFDIFactura,
    cg_f.UUID                   AS UUID_Factura,
    cg_f.Folio                  AS Folio_Factura,
    cg_f.Serie                  AS Serie_Factura,
    cg_f.TipoCFDI               AS TipoCFDI_Factura,
    cg_f.FechaEmision           AS FechaEmision_Factura,
    cg_c.IdCFDIGenerada         AS IdCFDIComplemento,
    cg_c.UUID                   AS UUID_Complemento,
    cg_c.Folio                  AS Folio_Complemento,
    -- Estado legible
    CASE p3l.EstadoLinea
        WHEN 'ENVIADO'   THEN 'Enviado'
        WHEN 'GENERADO'  THEN 'Generado — pendiente envío'
        ELSE                  'Pendiente'
    END                         AS EstadoDescripcion,
    p3l.FechaGeneracion,
    p3l.FechaEnvio,
    p3l.FechaRegistro,
    p3l.FechaUltimaActualizacion
FROM dbo.fccDocumentoFiscalCobro p3l
-- Asociación Paso 2 — proforma
LEFT JOIN dbo.fccPagoFacturaPedido pfp
    ON p3l.IdFCCPagoFacturaPedido = pfp.IdFCCPagoFacturaPedido
LEFT JOIN dbo.tpProformaPedido pp
    ON pfp.IdTPProformaPedido = pp.IdTPProformaPedido
LEFT JOIN dbo.Empresa e_pp
    ON pp.IdEmpresa = e_pp.IdEmpresa
-- Asociación Paso 2 — FAA
LEFT JOIN dbo.fccPagoFacturaAdelanto pfa
    ON p3l.IdFCCPagoFacturaAdelanto = pfa.IdFCCPagoFacturaAdelanto
LEFT JOIN dbo.fccFactura fc
    ON pfa.IdFccFactura = fc.IdFccFactura   -- RE-FU-015 (antes: LEFT JOIN dbo.tpProformaAdelanto pa ON pfa.IdTPProformaAdelanto = pa.IdTPProformaAdelanto)
LEFT JOIN dbo.CFDIGenerada cg_faa
    ON fc.IdCFDIGenerada = cg_faa.IdCFDIGenerada   -- UUID de la FAA para CFDIRelacionados
-- Cobro (resuelto desde la asociación Paso 2 activa)
LEFT JOIN dbo.fccPagoCliente fpc
    ON fpc.IdFCCPagoCliente = COALESCE(pfp.IdFCCPagoCliente, pfa.IdFCCPagoCliente)
-- Cliente
LEFT JOIN dbo.Cliente c
    ON fpc.IdCliente = c.IdCliente
LEFT JOIN dbo.DatosFacturacionCliente dfc
    ON fpc.IdCliente = dfc.IdCliente AND dfc.Activo = 1
-- Pedido (via proforma cuando aplica)
LEFT JOIN dbo.tpPedidoProformaPedido tpp_pp
    ON pp.IdTPProformaPedido = tpp_pp.IdTPProformaPedido AND tpp_pp.Activo = 1
LEFT JOIN dbo.tpPedido tp
    ON tpp_pp.IdTPPedido = tp.IdTPPedido
-- Catálogos SAT
LEFT JOIN dbo.catUsoCFDI ufo
    ON p3l.IdCatUsoCFDI = ufo.IdCatUsoCFDI
LEFT JOIN dbo.catMetodoDePagoCFDI mpc
    ON p3l.IdCatMetodoDePagoCFDI = mpc.IdCatMetodoDePagoCFDI
-- CFDIs generados en Paso 3
LEFT JOIN dbo.CFDIGenerada cg_f
    ON p3l.IdCFDIGeneradaFactura = cg_f.IdCFDIGenerada
LEFT JOIN dbo.CFDIGenerada cg_c
    ON p3l.IdCFDIGeneradaComplemento = cg_c.IdCFDIGenerada;
```

### Columnas clave de la Vista

| Columna | Origen | Descripción |
|---------|--------|-------------|
| IdFCCDocumentoFiscalCobro | fccDocumentoFiscalCobro | PK de la línea del Paso 3 |
| TipoDocumentoFiscal | fccDocumentoFiscalCobro | `FACTURA`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO` |
| **EstadoLinea** | fccDocumentoFiscalCobro | `PENDIENTE`, `GENERADO`, `ENVIADO` |
| **EstadoDescripcion** | Calculado | Descripción legible del estado para la UI |
| IdCatUsoCFDI / UsoCFDIClave | fccDocumentoFiscalCobro + catUsoCFDI | Uso CFDI seleccionado por el usuario, con clave y descripción del catálogo |
| IdCatMetodoDePagoCFDI / MetodoPagoClave | fccDocumentoFiscalCobro + catMetodoDePagoCFDI | Método de pago (PPD/PUE) con descripción del catálogo |
| UUID_FAA | CFDIGenerada (via fccFactura — RE-FU-015, antes tpProformaAdelanto) | UUID de la FAA existente; se incluye en CFDIRelacionados del Complemento |
| UUID_Factura | CFDIGenerada (alias cg_f) | UUID SAT del CFDI principal timbrado en Paso 3 |
| UUID_Complemento | CFDIGenerada (alias cg_c) | UUID SAT del Complemento (solo cascada PPD) |
| IdTPPedido / FolioPedidoInterno | tpPedido | Pedido asociado al documento |
| FechaEstimadaEntrega | tpPedido | FEE establecida post-envío |

---

## Tablas Leídas (Lectura)

| Tabla | Datos leídos | Uso en Paso 3 |
|-------|-------------|---------------|
| fccDocumentoFiscalCobro | Todas las columnas | Estado del wizard al reingresar a un cliente |
| fccPagoFacturaPedido | IdFCCPagoCliente, IdTPProformaPedido, Monto, NumeroDeParcialidad | Navegar al cobro y proforma desde la línea Paso 3 |
| fccPagoFacturaAdelanto | IdFCCPagoCliente, IdFccFactura (RE-FU-015, antes IdTPProformaAdelanto), IdCFDIGenerada, Monto | Navegar al cobro y FAA; IdCFDIGenerada = UUID FAA para CFDIRelacionados |
| fccPagoCliente | Folio, Monto, MXN, USD, TipoDeCambio, IdCliente | Cobro origen del wizard |
| tpProformaPedido | Folio, MontoTotal, MontoPendiente, IdEmpresa, HayControlados, IdCFDIGenerada | Datos para armar el CFDI de la línea; HayControlados determina Factura vs Anticipo |
| fccFactura (RE-FU-015, reemplaza tpProformaAdelanto) | MontoTotal, IdEmpresa, IdCFDIGenerada (UUID FAA) | Datos para armar el Complemento desde FAA |
| CFDIGenerada | UUID, Folio, Serie, FechaEmision, Total, TipoCFDI | UUID de la FAA (para CFDIRelacionados del Complemento) |
| fccNotaCredito | IdCFDI (UUID NC), Monto, MXN, USD | NCs aplicadas en Paso 2 → nodo CFDIRelacionados al timbrar |
| DatosFacturacionCliente | RazonSocial, RFC, UsoCFDI, RegimenFiscal, DomicilioFiscalReceptor | Datos del receptor CFDI 4.0 |
| catTipoDocumentoFiscal | IdCatTipoDocumentoFiscal, Clave, Descripcion | Resolver FK al crear la línea del Paso 3 |
| catDocumentoFiscalCobroEstado | IdCatDocumentoFiscalCobroEstado, Clave, Descripcion | Resolver FK al crear/actualizar estado de la línea |
| catTipoCFDI | IdCatTipoCFDI, Clave, Descripcion | Resolver FK al insertar en CFDIGenerada al timbrar |
| catUsoCFDI | IdCatUsoCFDI, Clave, Descripcion | Combo Uso CFDI editable por línea; FK desde `fccDocumentoFiscalCobro.IdCatUsoCFDI` |
| catMetodoDePagoCFDI | IdCatMetodoDePagoCFDI, Clave, Descripcion | Método de pago PPD/PUE; FK desde `fccDocumentoFiscalCobro.IdCatMetodoDePagoCFDI` |
| Empresa | Prefijo, Alias, RFC_Emisor, RegimenFiscalEmisor | Datos del emisor CFDI 4.0 por empresa PROQUIFA |
| EmpresaFolio (ProquifaDotNet — Finanzas) | UltimoFolio, Serie, EmpresaClave | Folio a consumir al timbrar exitosamente |
| tpPedido | FolioPedidoInterno, IdContacto | Datos del pedido; contacto para modal Enviar |
| tpPedidoProformaPedido | IdTPPedido, IdTPProformaPedido | Relación pedido ↔ proforma |

---

## Tablas Escritas (runtime)

| Tabla | Momento | Operación |
|-------|---------|-----------|
| fccDocumentoFiscalCobro | Al iniciar Paso 3 | INSERT una línea por documento con `EstadoLinea = 'PENDIENTE'` |
| fccDocumentoFiscalCobro | Auto-guardado Uso CFDI / Método de Pago | UPDATE IdCatUsoCFDI, IdCatMetodoDePagoCFDI |
| fccDocumentoFiscalCobro | Timbrado exitoso | UPDATE EstadoLinea = `'GENERADO'`, IdCFDIGeneradaFactura, [IdCFDIGeneradaComplemento], FechaGeneracion |
| fccDocumentoFiscalCobro | Envío exitoso | UPDATE EstadoLinea = `'ENVIADO'`, FechaEnvio |
| CFDIGenerada | Timbrado exitoso (factura) | INSERT (UUID, Folio, Serie, FechaEmision, Total, IdCatTipoCFDI) vía ProquifaDotNet.Timbrado |
| CFDIGenerada | Timbrado exitoso (complemento cascada) | INSERT segundo CFDI (IdCatTipoCFDI → clave `COMPLEMENTO_PAGO`, IdCFDIRelacionado = IdCFDIGenerada de la Factura PPD) |
| EmpresaFolio | Timbrado exitoso | UPDATE UltimoFolio + 1 (consumo atómico con UPDLOCK) |
| tpProformaPedido | Timbrado exitoso | UPDATE IdCFDIGenerada = @IdCFDIFactura (marca la proforma como facturada) |
| fccConfirmacionPedido | Envío exitoso | INSERT (FolioConfirmacion, RutaArchivoPDF generada en Minio) |
| tpPedido | Envío exitoso | UPDATE FechaEstimadaEntrega (FEE — solo México) |
| CorreoEnviado | Envío exitoso | INSERT (registro de envío) |
| ArchivoCorreoEnviado | Envío exitoso | INSERT x N (PDF + XML por cada CFDI de la línea + PDF Confirmación de Pedido) |

---

## Flujo de Datos

```
1. INICIAR PASO 3 (al avanzar desde Paso 2 con asociación cerrada)
   Lee:  fccPagoFacturaPedido, fccPagoFacturaAdelanto (registros de asociación del Paso 2)
         tpProformaPedido.HayControlados (determina FACTURA vs FACTURA_ANTICIPO)
         DatosFacturacionCliente, Empresa (datos fiscales emisor/receptor)
   Escribe: fccDocumentoFiscalCobro INSERT una línea por documento
            IdFCCPagoFacturaPedido OR IdFCCPagoFacturaAdelanto (FK al Paso 2)
            TipoDocumentoFiscal determinado según origen

2. EDITAR LÍNEA (Uso CFDI / Método de Pago)
   Lee:  catUsoCFDI, catMetodoDePagoCFDI (para poblar los combos de la UI)
   Escribe: fccDocumentoFiscalCobro UPDATE IdCatUsoCFDI, IdCatMetodoDePagoCFDI (auto-guardado)

3. PREVISUALIZAR
   Lee: vfccDocumentoFiscalCobro (todos los datos de la línea)
        fccNotaCredito (NCs a incluir en CFDIRelacionados)
   Sin escrituras en BD. El PDF se genera en memoria para visualización.

4. TIMBRAR
   ProquifaDotNet.Timbrado:
     INSERT CFDIGenerada (Factura/Anticipo/Complemento) — TipoCFDI discriminado
     Llama al PAC TurboPac
     UPDATE CFDIGenerada SET UUID, Folio (número de serie consumido de EmpresaFolio)
     UPDATE EmpresaFolio SET UltimoFolio + 1 (UPDLOCK atómico)
     INSERT StampingLog (trazabilidad)
     [Si PPD cascada]: INSERT segundo CFDIGenerada (TipoCFDI='COMPLEMENTO_PAGO',
                       IdCFDIRelacionado = IdCFDIGenerada de la Factura PPD)
   ProquifaDotNet:
     UPDATE tpProformaPedido SET IdCFDIGenerada = @IdCFDIFactura (cuando origen = proforma)
     UPDATE fccDocumentoFiscalCobro SET EstadoLinea='GENERADO', FechaGeneracion,
                              IdCFDIGeneradaFactura, [IdCFDIGeneradaComplemento]

5. ENVIAR
   ProquifaDotNet:
     INSERT fccConfirmacionPedido (FolioConfirmacion, RutaArchivoPDF en Minio)
     INSERT CorreoEnviado + ArchivoCorreoEnviado (PDFs + XMLs + Confirmación)
     UPDATE fccDocumentoFiscalCobro SET EstadoLinea='ENVIADO', FechaEnvio
     UPDATE tpPedido SET FechaEstimadaEntrega (FEE — solo México)
     Transferencia Legacy (pedido, factura, NCs, cobro) — mecanismo pendiente de definir (B3)
```

---

## Orden de Ejecución de Scripts

| Paso | Script | Base de Datos | Prerequisito |
|------|--------|---------------|--------------|
| 1 | CREATE TABLE catTipoDocumentoFiscal + INSERT datos iniciales | ProquifaDotNet | Ninguno |
| 2 | CREATE TABLE catDocumentoFiscalCobroEstado + INSERT datos iniciales | ProquifaDotNet | Ninguno |
| 3 | CREATE TABLE catTipoCFDI + INSERT datos iniciales | ProquifaDotNet | Ninguno |
| 4 | CREATE TABLE fccDocumentoFiscalCobro | ProquifaDotNet | Pasos 1 y 2; RE-FU-026 (fccPagoFacturaPedido, fccPagoFacturaAdelanto existen) |
| 5 | CREATE TABLE fccConfirmacionPedido | ProquifaDotNet | Paso 4 |
| 6 | ALTER TABLE CFDIGenerada ADD IdCatTipoCFDI (FK a catTipoCFDI) | ProquifaDotNet | Paso 3; RE-FU-019 (CFDIGenerada existe) |
| 7 | ALTER TABLE CFDIGenerada ADD IdCFDIRelacionado | ProquifaDotNet | Paso 6 |
| 8 | UPDATE CFDIGenerada normalización registros previos | ProquifaDotNet | Paso 6; validar tipo correcto de FAAs existentes |
| 9 | ALTER TABLE tpPedido ADD FechaEstimadaEntrega (condicional) | ProquifaDotNet | Verificar existencia previa |
| 10 | CREATE VIEW vfccDocumentoFiscalCobro | ProquifaDotNet | Pasos 1–9 |

---

## Brechas Críticas

| # | Brecha | Bloqueante | Acción |
|---|--------|-----------|--------|
| B1 | Tipo de relación SAT para Factura Anticipo (controlados): ¿07 Aplicación de Anticipo o diferente? | Sí | Confirmar con asesor fiscal PROQUIFA |
| B2 | Plantilla correo Complemento de Pago: asunto y cuerpo pendientes (PMO #31) | Media | Confirmar con PMO/Tesorería |
| B3 | Mecanismo de transferencia a Legacy desde Paso 3: canal (tabla ETL, cola RabbitMQ, API Legacy) no definido | Sí | Definir con arquitectura antes de implementar |
| B4 | FEE: reglas de cálculo y nivel de granularidad. `tpPartidaPedido.FechaEstimadaEntrega` ya existe (FEE por partida al tramitar). Pendiente confirmar: (a) si RE-028 actualiza partidas o solo la cabecera `tpPedido`; (b) si la cabecera requiere valor derivado cuando las partidas tienen FEEs distintas; (c) regla de cálculo (días hábiles, fecha fija, parámetro por empresa) | Sí | Confirmar con operaciones PROQUIFA México antes de implementar el ALTER |
| B5 | Folio de la Confirmación de Pedido: formato y foliador (¿usa EmpresaFolio o foliador propio?) | Media | Confirmar con PMO |
| B6 | Comportamiento si el Complemento en cascada falla tras Factura PPD timbrada exitosamente | Sí | Definir política de reintento (RE-FU-019 patrón de referencia) |
| B7 | Contacto del pedido no disponible al armar modal Enviar: ¿se bloquea el envío? | Media | Confirmar con negocio |

---

## Gaps y Pendientes

| # | Gap | Tipo | Acción |
|---|-----|------|--------|
| G1 | Relación SAT tipo 07 para Factura Anticipo de controlados | Fiscal | Asesor fiscal |
| G2 | Política ante caída del PAC TurboPac en Paso 3 (transversal con RE-FU-019) | Técnico | Definir reintento/encolamiento |
| G3 | Formato y foliador de Confirmación de Pedido para Prepago | Negocio | PMO/Operaciones |
| G4 | UPDATE inicial TipoCFDI en registros CFDIGenerada previos (FAA generadas en RE-FU-019) | Técnico | Script de migración al ejecutar RE-FU-028 |
| G5 | Transferencia Legacy: estructura del payload (qué tablas Legacy recibe) | Técnico | Arquitectura Legacy |
| G6 | `FechaEstimadaEntrega` ya existe en `tpPartidaPedido` (por partida/tramitación) y puede existir en `tpPedido` pre-R16. Verificar ambas tablas en RYNL010 con `sys.columns`. Resolver granularidad (cabecera vs. partida) como parte de Brecha B4 antes del ALTER | Técnico | Revisar `sys.columns` en RYNL010; resolver B4 primero |

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| R16A-RE-FU-019 | CFDIGenerada + EmpresaFolio + patrón timbrado en ProquifaDotNet.Timbrado |
| R16A-RE-FU-024 / 026 | fccPagoCliente, fccPagoFacturaPedido (cobros y asociación Paso 1/Paso 2) |
| R16A-RE-FU-026 | fccPagoFacturaAdelanto, fccNotaCredito (NCs con UUID para CFDIRelacionados) |
| R16A-RE-FU-021 | Diseño PDF Factura México — DocumentBuilder |
| R16A-RE-FU-013/014 | Genera `tpProformaPedido` con flag `HayControlados` (origen de la lógica condicional) |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura`/`vfccFactura` — genera pendiente FAA usada en línea tipo COMPLEMENTO_PAGO (antes referenciaba `tpProformaAdelanto`) |

---

**Bases de Datos:** ProquifaDotNet (incluye EmpresaFolio, propiedad Finanzas)
