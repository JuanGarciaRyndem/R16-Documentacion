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
| 3   | CREATE TABLE FiscalDocumentType               | ProquifaDotNetTimbrado     | DDL  | Alta      |
| 4   | CREATE TABLE Cfdi                            | ProquifaDotNetTimbrado     | DDL  | Alta      |
| 5   | CREATE TABLE StampingLog                     | ProquifaDotNetTimbrado     | DDL  | Alta      |
| 6   | INSERT FiscalDocumentType (datos iniciales)  | ProquifaDotNetTimbrado     | DML  | Alta      |
| -   | ProquifaDotNet                               | Solo lectura - sin cambios | -    | -         |

> **Nota (Reglas al diseñar — regla 2):** `ProquifaDotNetTimbrado` es una base de datos **nueva**, por lo que su estructura (tablas, columnas, PK/FK, constraints) se nombra en **inglés**. La BD `ProquifaDotNet` (Parte 1, solo lectura) conserva su nomenclatura en español por ser la base de datos existente (regla 1).
>
> **Nota (nomenclatura Timbrado -> Stamping):** la tabla `StampingLog` (antes `TimbradoLog`) se traduce a inglés para no mezclar idiomas dentro de la estructura de la BD nueva. El nombre de la base de datos (`ProquifaDotNetTimbrado`) y de la solución (`ProquifaDotNet.Timbrado`) se mantienen sin traducir por ser nomenclatura ya establecida en las instrucciones del proyecto — la excepción aplica solo al nombre de la solución/BD, no a las tablas ni al código interno.

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

    FiscalDocumentType (catalogo)
        PK: Id
        UQ: Code

    Cfdi (documento fiscal)
        PK: Id
        FK: FiscalDocumentTypeId -> FiscalDocumentType

    StampingLog (auditoria)
        PK: Id
        FK: CfdiId -> Cfdi

---

### Tabla: AppSetting
**Proposito:** Configuracion del servicio de timbrado (endpoints SAP, timeouts, etc.). No incluye configuracion de reintentos: Timbrado no reintenta, es un servicio de un solo intento por peticion (la politica de reintento la maneja Finanzas, de forma local en cada punto de generacion: Factura, Factura por Adelantado, Nota de Credito, Complemento de Pago).

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| Id | uniqueidentifier | NO | NEWID() | PK |
| Name | varchar(50) | NO | - | Clave de configuracion |
| Value | varchar(max) | NO | - | Valor (puede ser JSON) |
| Description | varchar(100) | NO | - | Descripcion legible |
| CreatedAt | datetime2(7) | NO | SYSUTCDATETIME() | Fecha creacion |
| UpdatedAt | datetime2(7) | NO | SYSUTCDATETIME() | Fecha actualizacion |
| IsActive | bit | NO | 1 | Activo |

---

### Tabla: FiscalDocumentType
**Proposito:** Catalogo de tipos de documentos fiscales.

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| Id | uniqueidentifier | NO | NEWID() | PK |
| Code | varchar(50) | NO | - | Clave unica (UQ) |
| Description | varchar(200) | NO | - | Descripcion del tipo |
| CreatedAt | datetime2(7) | NO | SYSUTCDATETIME() | Fecha creacion |
| UpdatedAt | datetime2(7) | NO | SYSUTCDATETIME() | Fecha actualizacion |
| IsActive | bit | NO | 1 | Activo |

**Datos iniciales:**

| Code                    | Description                                                              |
| ----------------------- | ------------------------------------------------------------------------ |
| AdvanceInvoice          | Factura emitida por adelantado previo a entrega de mercancia (FAA)       |
| RegularInvoice          | Factura estandar de venta                                                |
| AnticipatedInvoice      | Factura de anticipo para pedidos con sustancias controladas              |
| CreditNote              | Nota de credito por devolucion o ajuste                                  |

> Nota: `AdvanceInvoice` (Factura por Adelantado, este requisito) y `AnticipatedInvoice` (Factura Anticipo, sustancias controladas) son instrumentos distintos — ver OBS-037 en R16A-RE-FU-018-Back.md.

---

### Tabla: Cfdi
**Proposito:** Documento fiscal generado y timbrado. Almacena XML y referencia Minio. `Cfdi`, `Uuid` y `Rfc` se mantienen como términos fiscales (no se traducen, son estándar en la industria).

| Columna               | Tipo             | Nulo | Default          | Descripcion                          |
| --------------------- | ---------------- | ---- | ---------------- | ------------------------------------ |
| Id                    | uniqueidentifier | NO   | NEWID()          | PK                                   |
| FiscalDocumentTypeId  | uniqueidentifier | NO   | -                | FK -> FiscalDocumentType             |
| Uuid                  | varchar(36)      | SI   | -                | UUID del timbrado (asignado por PAC) |
| Series                | varchar(25)      | SI   | -                | Serie del CFDI                       |
| Folio                 | varchar(40)      | SI   | -                | Folio del CFDI                       |
| IssueDate             | datetime2(7)     | SI   | -                | Fecha de emision                     |
| IssuerRfc             | varchar(13)      | NO   | -                | RFC de la empresa emisora            |
| ReceiverRfc           | varchar(50)      | NO   | -                | RFC/RUC del cliente receptor         |
| Total                 | decimal(18,2)    | NO   | -                | Monto total del documento            |
| Currency              | varchar(5)       | NO   | -                | Clave moneda (MXN/USD/PEN)           |
| ExchangeRate          | decimal(18,6)    | SI   | -                | Tipo de cambio aplicado              |
| PaymentMethod         | varchar(5)       | SI   | -                | Clave SAT metodo pago (PUE/PPD)      |
| PaymentForm           | varchar(5)       | SI   | -                | Clave SAT forma pago (03/99/etc)     |
| CfdiUse               | varchar(10)      | SI   | -                | Clave SAT uso CFDI (G03/P01/etc)     |
| XmlContent            | varchar(max)     | SI   | -                | XML completo del CFDI timbrado       |
| MinioFileKey          | varchar(600)     | SI   | -                | Path del XML en Minio                |
| MinioBucket           | varchar(100)     | SI   | -                | Bucket en Minio                      |
| Status                | varchar(30)      | NO   | 'Pending'        | Pending/Stamped/Failed (alineado con StampingLog.NewStatus, sin contador de reintentos) |
| ErrorMessage          | varchar(max)     | SI   | -                | Detalle del error si fallo           |
| CreatedAt             | datetime2(7)     | NO   | SYSUTCDATETIME() | Fecha creacion                       |
| UpdatedAt             | datetime2(7)     | NO   | SYSUTCDATETIME() | Fecha actualizacion                  |
| IsActive              | bit              | NO   | 1                | Activo                               |

---

### Tabla: StampingLog
**Proposito:** Auditoria de la peticion de timbrado con el PAC (SAP). Un solo registro por solicitud recibida de Finanzas (sin reintentos internos): Timbrado invoca al PAC una vez y registra el resultado.

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| Id | uniqueidentifier | NO | NEWID() | PK |
| CfdiId | uniqueidentifier | NO | - | FK -> Cfdi |
| Action | varchar(50) | NO | - | Stamp/Cancel |
| PreviousStatus | varchar(30) | SI | - | Estado antes de la accion |
| NewStatus | varchar(30) | NO | - | Estado despues de la accion |
| Request | varchar(max) | SI | - | Payload enviado al PAC |
| Response | varchar(max) | SI | - | Respuesta del PAC |
| ErrorMessage | varchar(max) | SI | - | Error si fallo |
| DurationMs | int | SI | - | Tiempo de respuesta en ms |
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
        [Id] uniqueidentifier NOT NULL CONSTRAINT [DF_AppSetting_Id] DEFAULT (NEWID()),
        [Name] varchar(50) NOT NULL,
        [Value] varchar(max) NOT NULL,
        [Description] varchar(100) NOT NULL,
        [CreatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_AppSetting_CreatedAt] DEFAULT (SYSUTCDATETIME()),
        [UpdatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_AppSetting_UpdatedAt] DEFAULT (SYSUTCDATETIME()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_AppSetting_IsActive] DEFAULT (1),
        CONSTRAINT [PK_AppSetting] PRIMARY KEY CLUSTERED ([Id])
    );
    GO

    CREATE TABLE [dbo].[FiscalDocumentType](
        [Id] uniqueidentifier NOT NULL CONSTRAINT [DF_FiscalDocumentType_Id] DEFAULT (NEWID()),
        [Code] varchar(50) NOT NULL,
        [Description] varchar(200) NOT NULL,
        [CreatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_FiscalDocumentType_CreatedAt] DEFAULT (SYSUTCDATETIME()),
        [UpdatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_FiscalDocumentType_UpdatedAt] DEFAULT (SYSUTCDATETIME()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_FiscalDocumentType_IsActive] DEFAULT (1),
        CONSTRAINT [PK_FiscalDocumentType] PRIMARY KEY CLUSTERED ([Id]),
        CONSTRAINT [UQ_FiscalDocumentType_Code] UNIQUE ([Code])
    );
    GO

    CREATE TABLE [dbo].[Cfdi](
        [Id] uniqueidentifier NOT NULL CONSTRAINT [DF_Cfdi_Id] DEFAULT (NEWID()),
        [FiscalDocumentTypeId] uniqueidentifier NOT NULL,
        [Uuid] varchar(36) NULL,
        [Series] varchar(25) NULL,
        [Folio] varchar(40) NULL,
        [IssueDate] datetime2(7) NULL,
        [IssuerRfc] varchar(13) NOT NULL,
        [ReceiverRfc] varchar(50) NOT NULL,
        [Total] decimal(18,2) NOT NULL,
        [Currency] varchar(5) NOT NULL,
        [ExchangeRate] decimal(18,6) NULL,
        [PaymentMethod] varchar(5) NULL,
        [PaymentForm] varchar(5) NULL,
        [CfdiUse] varchar(10) NULL,
        [XmlContent] varchar(max) NULL,
        [MinioFileKey] varchar(600) NULL,
        [MinioBucket] varchar(100) NULL,
        [Status] varchar(30) NOT NULL CONSTRAINT [DF_Cfdi_Status] DEFAULT ('Pending'),
        [ErrorMessage] varchar(max) NULL,
        [CreatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_Cfdi_CreatedAt] DEFAULT (SYSUTCDATETIME()),
        [UpdatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_Cfdi_UpdatedAt] DEFAULT (SYSUTCDATETIME()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_Cfdi_IsActive] DEFAULT (1),
        CONSTRAINT [PK_Cfdi] PRIMARY KEY CLUSTERED ([Id]),
        CONSTRAINT [FK_Cfdi_FiscalDocumentType] FOREIGN KEY ([FiscalDocumentTypeId])
            REFERENCES [dbo].[FiscalDocumentType]([Id])
    );
    GO

    CREATE TABLE [dbo].[StampingLog](
        [Id] uniqueidentifier NOT NULL CONSTRAINT [DF_StampingLog_Id] DEFAULT (NEWID()),
        [CfdiId] uniqueidentifier NOT NULL,
        [Action] varchar(50) NOT NULL,
        [PreviousStatus] varchar(30) NULL,
        [NewStatus] varchar(30) NOT NULL,
        [Request] varchar(max) NULL,
        [Response] varchar(max) NULL,
        [ErrorMessage] varchar(max) NULL,
        [DurationMs] int NULL,
        [CreatedAt] datetime2(7) NOT NULL CONSTRAINT [DF_StampingLog_CreatedAt] DEFAULT (SYSUTCDATETIME()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_StampingLog_IsActive] DEFAULT (1),
        CONSTRAINT [PK_StampingLog] PRIMARY KEY CLUSTERED ([Id]),
        CONSTRAINT [FK_StampingLog_Cfdi] FOREIGN KEY ([CfdiId])
            REFERENCES [dbo].[Cfdi]([Id])
    );
    GO

    INSERT INTO [dbo].[FiscalDocumentType] ([Code], [Description])
    VALUES
        ('AdvanceInvoice', 'Factura emitida por adelantado previo a entrega de mercancia (FAA)'),
        ('RegularInvoice', 'Factura estandar de venta'),
        ('AnticipatedInvoice', 'Factura de anticipo para pedidos con sustancias controladas'),
        ('CreditNote', 'Nota de credito por devolucion o ajuste');
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
