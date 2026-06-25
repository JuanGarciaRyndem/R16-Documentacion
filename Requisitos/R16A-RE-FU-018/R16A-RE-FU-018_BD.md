# Impacto en BD - Factura por Adelantado (Pantalla Inicial)
**Requisito:** R16A-RE-FU-018
**Bases de Datos:** ProquifaDotNet (lectura) + ProquifaDotNetTimbrado (nueva)
**Version:** 2.0

---

## Resumen
Pantalla inicial del modulo NUEVO Factura por Adelantado. Listado agrupado por cliente
con pedidos pendientes de facturar por adelantado. Filtrado por cartera del usuario (Cobrador).
Monto total dolarizado. Buscador por RazonSocial/RFC-RUC/FolioPedido.

Adicionalmente se crea la base de datos **ProquifaDotNetTimbrado** como soporte al flujo
completo de timbrado fiscal que se dispara desde este modulo.

---

## Impacto en BD

| #   | Cambio                                       | Base de Datos              | Tipo | Prioridad |
| --- | -------------------------------------------- | -------------------------- | ---- | --------- |
| 1   | CREATE DATABASE ProquifaDotNetTimbrado       | Nueva                      | DDL  | Alta      |
| 2   | CREATE TABLE AppSetting                      | ProquifaDotNetTimbrado     | DDL  | Alta      |
| 3   | CREATE TABLE TipoDocumentoFiscal             | ProquifaDotNetTimbrado     | DDL  | Alta      |
| 4   | CREATE TABLE CFDI                            | ProquifaDotNetTimbrado     | DDL  | Alta      |
| 5   | CREATE TABLE TimbradoLog                     | ProquifaDotNetTimbrado     | DDL  | Alta      |
| 6   | INSERT TipoDocumentoFiscal (datos iniciales) | ProquifaDotNetTimbrado     | DML  | Alta      |
| -   | ProquifaDotNet                               | Solo lectura - sin cambios | -    | -         |

---

## Parte 1: Lectura de ProquifaDotNet (sin cambios)

### Cadena de Datos - Pendientes FAA

    tpPedido (FacturaPorAdelantado=1, Tramitado=1)
        -> tpPedidoProformaPedido -> tpProformaPedido (IdCliente, MontoTotal)

    tpProformaAdelanto (pendiente FAA)
        IdCFDIGenerada IS NULL = factura NO generada
        IdCFDI IS NULL = factura NO timbrada

    tpProformaAdelantoProformaPedido
        IdTPProformaPedido -> tpProformaPedido
        IdFCCPagoFacturaAdelanto -> fccPagoFacturaAdelanto -> tpProformaAdelanto

### Identificacion de Pendientes

    tpPedido.FacturaPorAdelantado = 1
    AND tpPedido.Tramitado = 1
    AND tpPedido.Activo = 1
    AND tpProformaAdelanto.Activo = 1
    AND tpProformaAdelanto.IdCFDIGenerada IS NULL (o generada pero no enviada)

### Filtro por Cartera del Usuario

    Cliente -> ClienteCarteraCliente (IdCliente, IdClienteCartera)
        -> ClienteCartera (IdUsuarioCobrador = @IdUsuarioLogueado)

| Tabla | Columna clave | Uso |
|-------|--------------|-----|
| ClienteCartera | IdUsuarioCobrador | Filtro por usuario logueado |
| ClienteCartera | IdRegion | Segregacion MEX/PER |
| ClienteCarteraCliente | IdCliente + IdClienteCartera | Vinculo N:N |

### Datos del Listado

| Columna UI | Tabla Fuente | Campo |
|------------|-------------|-------|
| Razon Social | DatosFacturacionCliente | RazonSocial |
| RFC/RUC | DatosFacturacionCliente | RFC |
| Facturas Pendientes | COUNT(tpProformaAdelanto) | WHERE Activo=1 AND no enviada |
| Monto Total (USD) | SUM(tpProformaAdelanto.Monto) | Convertido a USD |
| Antiguedad | MIN(tpProformaAdelanto.FechaRegistro) | Pendiente mas antiguo |

### Buscador

| Campo | Tabla | Busqueda |
|-------|-------|----------|
| Razon Social | DatosFacturacionCliente.RazonSocial | LIKE '%texto%' |
| RFC/RUC | DatosFacturacionCliente.RFC | LIKE '%texto%' |
| Folio Pedido | tpPedido.FolioPedidoInterno | LIKE '%texto%' |

### Tablas Consultadas (ProquifaDotNet - Lectura)

| Tabla | Rol |
|-------|-----|
| tpPedido | FacturaPorAdelantado=1, Tramitado=1, FolioPedidoInterno |
| tpProformaAdelanto | Pendiente FAA: Monto, IdCFDIGenerada, FechaRegistro |
| tpProformaPedido | Vinculo proforma-pedido |
| tpPedidoProformaPedido | Relacion pedido-proforma |
| tpProformaAdelantoProformaPedido | Relacion adelanto-proforma |
| fccPagoFacturaAdelanto | Relacion pago-adelanto |
| Cliente | IdCliente |
| DatosFacturacionCliente | RazonSocial, RFC (RFC/RUC) |
| ClienteCarteraCliente | Vinculo cliente-cartera |
| ClienteCartera | IdUsuarioCobrador |
| catMoneda | Para conversion a USD |
| Region | MEX/PER |

---

## Parte 2: Base de Datos Nueva - ProquifaDotNetTimbrado

### Proposito
BD independiente para el servicio ProquifaDotNet.Timbrado (.NET 10).
Almacena peticiones, respuestas del PAC (SAP), CFDI generados y configuracion.
Se comunica con ProquifaDotNet.Finanzas via API.

### Servidor y rutas

| Aspecto | Valor |
|---------|-------|
| Servidor | WIN-R14-DEV\\DEV_R17_APPS |
| Data file | R:\\SSQL DATA R17 APP\\ProquifaDotNetTimbrado.mdf |
| Log file | Y:\\SSQL LOG R17 PQF\\ProquifaDotNetTimbrado_log.ldf |
| Compatibility | 160 (SQL Server 2022) |

### Diagrama de tablas

    AppSetting (configuracion del servicio)

    TipoDocumentoFiscal (catalogo)
        PK: IdTipoDocumentoFiscal
        UQ: Clave

    CFDI (documento fiscal)
        PK: IdCFDI
        FK: IdTipoDocumentoFiscal -> TipoDocumentoFiscal

    TimbradoLog (auditoria)
        PK: IdTimbradoLog
        FK: IdCFDI -> CFDI

---

### Tabla: AppSetting
**Proposito:** Configuracion del servicio de timbrado (endpoints SAP, reintentos, etc.)

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| IdAppSetting | uniqueidentifier | NO | NEWID() | PK |
| Name | varchar(50) | NO | - | Clave de configuracion |
| Value | varchar(max) | NO | - | Valor (puede ser JSON) |
| Description | varchar(100) | NO | - | Descripcion legible |
| CreatedAt | datetime2(7) | NO | SYSUTCDATETIME() | Fecha creacion |
| UpdatedAt | datetime2(7) | NO | SYSUTCDATETIME() | Fecha actualizacion |
| IsActive | bit | NO | 1 | Activo |

---

### Tabla: TipoDocumentoFiscal
**Proposito:** Catalogo de tipos de documentos fiscales.

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| IdTipoDocumentoFiscal | uniqueidentifier | NO | NEWID() | PK |
| Clave | varchar(50) | NO | - | Clave unica (UQ) |
| Descripcion | varchar(200) | NO | - | Descripcion del tipo |
| CreatedAt | datetime2(7) | NO | SYSUTCDATETIME() | Fecha creacion |
| UpdatedAt | datetime2(7) | NO | SYSUTCDATETIME() | Fecha actualizacion |
| IsActive | bit | NO | 1 | Activo |

**Datos iniciales:**

| Clave                | Descripcion                                                  |
| -------------------- | ------------------------------------------------------------ |
| FacturaPorAdelantado | Factura emitida por adelantado previo a entrega de mercancia |
| FacturaNormal        | Factura estandar de venta                                    |
| FacturaAnticipo      | Factura de anticipo para pedidos con sustancias controladas  |
| NotaCredito          | Nota de credito por devolucion o ajuste                      |

---

### Tabla: CFDI
**Proposito:** Documento fiscal generado y timbrado. Almacena XML y referencia Minio.

| Columna               | Tipo             | Nulo | Default          | Descripcion                          |
| --------------------- | ---------------- | ---- | ---------------- | ------------------------------------ |
| IdCFDI                | uniqueidentifier | NO   | NEWID()          | PK                                   |
| IdTipoDocumentoFiscal | uniqueidentifier | NO   | -                | FK -> TipoDocumentoFiscal            |
| UUID                  | varchar(36)      | SI   | -                | UUID del timbrado (asignado por PAC) |
| Serie                 | varchar(25)      | SI   | -                | Serie del CFDI                       |
| Folio                 | varchar(40)      | SI   | -                | Folio del CFDI                       |
| FechaEmision          | datetime2(7)     | SI   | -                | Fecha de emision                     |
| RFCEmisor             | varchar(13)      | NO   | -                | RFC de la empresa emisora            |
| RFCReceptor           | varchar(50)      | NO   | -                | RFC/RUC del cliente receptor         |
| Total                 | decimal(18,2)    | NO   | -                | Monto total del documento            |
| Moneda                | varchar(5)       | NO   | -                | Clave moneda (MXN/USD/PEN)           |
| TipoCambio            | decimal(18,6)    | SI   | -                | Tipo de cambio aplicado              |
| MetodoPago            | varchar(5)       | SI   | -                | Clave SAT metodo pago (PUE/PPD)      |
| FormaPago             | varchar(5)       | SI   | -                | Clave SAT forma pago (03/99/etc)     |
| UsoCFDI               | varchar(10)      | SI   | -                | Clave SAT uso CFDI (G03/P01/etc)     |
| XmlContent            | varchar(max)     | SI   | -                | XML completo del CFDI timbrado       |
| MinioFileKey          | varchar(600)     | SI   | -                | Path del XML en Minio                |
| MinioBucket           | varchar(100)     | SI   | -                | Bucket en Minio                      |
| EstatusTimbrado       | varchar(30)      | NO   | 'Pendiente'      | Pendiente/Enviado/Timbrado/Error     |
| MensajeError          | varchar(max)     | SI   | -                | Detalle del error si fallo           |
| Intentos              | int              | NO   | 0                | Contador de reintentos               |
| CreatedAt             | datetime2(7)     | NO   | SYSUTCDATETIME() | Fecha creacion                       |
| UpdatedAt             | datetime2(7)     | NO   | SYSUTCDATETIME() | Fecha actualizacion                  |
| IsActive              | bit              | NO   | 1                | Activo                               |

---

### Tabla: TimbradoLog
**Proposito:** Auditoria de cada intento de timbrado con el PAC (SAP).

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| IdTimbradoLog | uniqueidentifier | NO | NEWID() | PK |
| IdCFDI | uniqueidentifier | NO | - | FK -> CFDI |
| Accion | varchar(50) | NO | - | Timbrar/Cancelar/Reintento |
| EstatusAnterior | varchar(30) | SI | - | Estado antes de la accion |
| EstatusNuevo | varchar(30) | NO | - | Estado despues de la accion |
| Request | varchar(max) | SI | - | Payload enviado al PAC |
| Response | varchar(max) | SI | - | Respuesta del PAC |
| MensajeError | varchar(max) | SI | - | Error si fallo |
| DuracionMs | int | SI | - | Tiempo de respuesta en ms |
| CreatedAt | datetime2(7) | NO | SYSUTCDATETIME() | Fecha del intento |
| IsActive | bit | NO | 1 | Activo |

---

### Script completo

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Crear base de datos ProquifaDotNetTimbrado y tablas iniciales

    CREATE DATABASE [ProquifaDotNetTimbrado]
    ON PRIMARY (
        NAME = N'ProquifaDotNetTimbrado',
        FILENAME = N'R:\SSQL DATA R17 APP\ProquifaDotNetTimbrado.mdf',
        SIZE = 64MB,
        FILEGROWTH = 64MB
    )
    LOG ON (
        NAME = N'ProquifaDotNetTimbrado_log',
        FILENAME = N'Y:\SSQL LOG R17 PQF\ProquifaDotNetTimbrado_log.ldf',
        SIZE = 32MB,
        FILEGROWTH = 32MB
    );
    GO

    ALTER DATABASE [ProquifaDotNetTimbrado] SET COMPATIBILITY_LEVEL = 160;
    GO

    USE [ProquifaDotNetTimbrado];
    GO

    CREATE TABLE [dbo].[AppSetting](
        [IdAppSetting] uniqueidentifier NOT NULL CONSTRAINT [DF_AppSetting_Id] DEFAULT (NEWID()),
        [Name] varchar(50) NOT NULL,
        [Value] varchar(max) NOT NULL,
        [Description] varchar(100) NOT NULL,
        [CreatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_AppSetting_CreatedAt] DEFAULT (SYSUTCDATETIME()),
        [UpdatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_AppSetting_UpdatedAt] DEFAULT (SYSUTCDATETIME()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_AppSetting_IsActive] DEFAULT (1),
        CONSTRAINT [PK_AppSetting] PRIMARY KEY CLUSTERED ([IdAppSetting])
    );
    GO

    CREATE TABLE [dbo].[TipoDocumentoFiscal](
        [IdTipoDocumentoFiscal] uniqueidentifier NOT NULL CONSTRAINT [DF_TipoDocumentoFiscal_Id] DEFAULT (NEWID()),
        [Clave] varchar(50) NOT NULL,
        [Descripcion] varchar(200) NOT NULL,
        [CreatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_TipoDocumentoFiscal_CreatedAt] DEFAULT (SYSUTCDATETIME()),
        [UpdatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_TipoDocumentoFiscal_UpdatedAt] DEFAULT (SYSUTCDATETIME()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_TipoDocumentoFiscal_IsActive] DEFAULT (1),
        CONSTRAINT [PK_TipoDocumentoFiscal] PRIMARY KEY CLUSTERED ([IdTipoDocumentoFiscal]),
        CONSTRAINT [UQ_TipoDocumentoFiscal_Clave] UNIQUE ([Clave])
    );
    GO

    CREATE TABLE [dbo].[CFDI](
        [IdCFDI] uniqueidentifier NOT NULL CONSTRAINT [DF_CFDI_Id] DEFAULT (NEWID()),
        [IdTipoDocumentoFiscal] uniqueidentifier NOT NULL,
        [UUID] varchar(36) NULL,
        [Serie] varchar(25) NULL,
        [Folio] varchar(40) NULL,
        [FechaEmision] datetime2(7) NULL,
        [RFCEmisor] varchar(13) NOT NULL,
        [RFCReceptor] varchar(50) NOT NULL,
        [Total] decimal(18,2) NOT NULL,
        [Moneda] varchar(5) NOT NULL,
        [TipoCambio] decimal(18,6) NULL,
        [MetodoPago] varchar(5) NULL,
        [FormaPago] varchar(5) NULL,
        [UsoCFDI] varchar(10) NULL,
        [XmlContent] varchar(max) NULL,
        [MinioFileKey] varchar(600) NULL,
        [MinioBucket] varchar(100) NULL,
        [EstatusTimbrado] varchar(30) NOT NULL CONSTRAINT [DF_CFDI_Estatus] DEFAULT ('Pendiente'),
        [MensajeError] varchar(max) NULL,
        [Intentos] int NOT NULL CONSTRAINT [DF_CFDI_Intentos] DEFAULT (0),
        [CreatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_CFDI_CreatedAt] DEFAULT (SYSUTCDATETIME()),
        [UpdatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_CFDI_UpdatedAt] DEFAULT (SYSUTCDATETIME()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_CFDI_IsActive] DEFAULT (1),
        CONSTRAINT [PK_CFDI] PRIMARY KEY CLUSTERED ([IdCFDI]),
        CONSTRAINT [FK_CFDI_TipoDocumentoFiscal] FOREIGN KEY ([IdTipoDocumentoFiscal])
            REFERENCES [dbo].[TipoDocumentoFiscal]([IdTipoDocumentoFiscal])
    );
    GO

    CREATE TABLE [dbo].[TimbradoLog](
        [IdTimbradoLog] uniqueidentifier NOT NULL CONSTRAINT [DF_TimbradoLog_Id] DEFAULT (NEWID()),
        [IdCFDI] uniqueidentifier NOT NULL,
        [Accion] varchar(50) NOT NULL,
        [EstatusAnterior] varchar(30) NULL,
        [EstatusNuevo] varchar(30) NOT NULL,
        [Request] varchar(max) NULL,
        [Response] varchar(max) NULL,
        [MensajeError] varchar(max) NULL,
        [DuracionMs] int NULL,
        [CreatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_TimbradoLog_CreatedAt] DEFAULT (SYSUTCDATETIME()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_TimbradoLog_IsActive] DEFAULT (1),
        CONSTRAINT [PK_TimbradoLog] PRIMARY KEY CLUSTERED ([IdTimbradoLog]),
        CONSTRAINT [FK_TimbradoLog_CFDI] FOREIGN KEY ([IdCFDI])
            REFERENCES [dbo].[CFDI]([IdCFDI])
    );
    GO

    INSERT INTO [dbo].[TipoDocumentoFiscal] ([Clave], [Descripcion])
    VALUES
        ('FacturaPorAdelantado', 'Factura emitida por adelantado previo a entrega de mercancia'),
        ('FacturaNormal', 'Factura estandar de venta'),
        ('FacturaAnticipo', 'Factura de anticipo para pedidos con sustancias controladas'),
        ('NotaCredito', 'Nota de credito por devolucion o ajuste');
    GO

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | Campo/logica 'factura generada pero no enviada' | Tecnico | Verificar flag de envio en tpProformaAdelanto |
| 2 | Tipo de cambio para dolarizacion | Negocio | Confirmar TC historico vs dia actual |
| 3 | Rol operativo: Gestor Cobranza vs Analista CxC | Negocio | Confirmar denominacion |
| 4 | Timbrado Peru (OSE/SUNAT) | Tecnico | Brecha mayor - RE-FU-005 B5 |
| 5 | Cliente sin Cobrador asignado | Operativo | Pendiente invisible |

---

## Dependencias

| Requisito      | Relacion                                     |
| -------------- | -------------------------------------------- |
| R16A-RE-FU-012 | Genera pendiente FAA para Credito con FAA    |
| R16A-RE-FU-015 | Genera pendiente FAA para Prepago con FAA    |
| R16A-RE-FU-005 | Timbrado Peru (brecha)                       |
| R16A-RE-FU-016 | Proforma MEX (flujo previo a FAA en Prepago) |

---

**Generado por:** GitHub Copilot in SSMS
**Bases de Datos:** ProquifaDotNet (lectura) + ProquifaDotNetTimbrado (nueva)
