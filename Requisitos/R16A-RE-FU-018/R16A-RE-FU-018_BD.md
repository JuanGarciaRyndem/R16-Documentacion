# Impacto en BD - Factura por Adelantado (Pantalla Inicial)
**Requisito:** R16A-RE-FU-018
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (nueva)
**Version:** 3.0 - CFDI se persiste en Finanzas (CFDIGenerada), Timbrado sin tabla de negocio propia

---

## Resumen
Pantalla inicial del modulo NUEVO Factura por Adelantado. Listado agrupado por cliente
con pedidos pendientes de facturar por adelantado. Filtrado por cartera del usuario (Cobrador).
Monto total dolarizado. Buscador por RazonSocial/RFC-RUC/FolioPedido.

Adicionalmente se crea la base de datos **ProquifaDotNetTimbrado** como soporte tecnico
del servicio de timbrado (config + auditoria de llamadas al PAC). El **registro de negocio
del CFDI vive en `CFDIGenerada` (ProquifaDotNet), propiedad de ProquifaDotNet.Finanzas** —
ver nota de arquitectura abajo.

---

> **Nota de arquitectura (correccion v3.0 — el CFDI no va en Timbrado, va en Finanzas):**
> Las versiones previas de este documento creaban una tabla `Cfdi` + catalogo `FiscalDocumentType`
> dentro de `ProquifaDotNetTimbrado`, duplicando lo que `ER-Finanzas.md` ya diseño para
> `ProquifaDotNet.Finanzas`: la tabla **`CFDIGenerada`** (mas `catTipoCFDI`, `CFDIGeneradaConcepto`,
> `CFDIGeneradaRelacionado`, `CFDICancelacion`) fisicamente en la base de datos **ProquifaDotNet**,
> gestionada via EF Core Scaffold por Finanzas. Se corrige de la siguiente forma:
> - Se **elimina** la tabla `Cfdi` y el catalogo `FiscalDocumentType` de `ProquifaDotNetTimbrado`.
>   El tipo de documento fiscal se resuelve con el catalogo **`catTipoCFDI`** (ya existente en el
>   diseño de Finanzas / `ER-Finanzas.md`), no con un catalogo propio de Timbrado.
> - **`ProquifaDotNet.Finanzas`** es quien crea/actualiza el registro de negocio del CFDI —
>   `INSERT`/`UPDATE` sobre `CFDIGenerada` (ProquifaDotNet) — despues de invocar a Timbrado.
> - **`ProquifaDotNet.Timbrado`** deja de tener tabla de negocio propia (`Cfdi`). Se reduce a un
>   **servicio tecnico** interno: recibe los datos fiscales ya armados por Finanzas, invoca al PAC
>   (SAP), y regresa UUID + XML + estatus del timbrado a Finanzas. Su unica persistencia propia es
>   `AppSetting` (configuracion) y `StampingLog` (auditoria tecnica de la llamada al PAC).
> - `CFDIGenerada` se **extiende** (ALTER TABLE, Parte 3 de este documento) con las columnas
>   tecnicas que antes vivian en `Cfdi` (estatus, error, moneda, tipo de cambio, uso CFDI,
>   metodo de pago, referencia al XML) para que Finanzas tenga todo lo necesario en un solo lugar.
> - Ver tambien `R16A-RE-FU-018-Back.md` y `R16A-RE-FU-018-Tareas.md` para el cambio de
>   `CfdiController` (pasa de Timbrado a Finanzas).

---

## Impacto en BD

| #   | Cambio                                                          | Base de Datos              | Tipo | Prioridad |
| --- | ---------------------------------------------------------------- | -------------------------- | ---- | --------- |
| 1   | CREATE DATABASE ProquifaDotNetTimbrado                          | Nueva                      | DDL  | Alta      |
| 2   | CREATE TABLE AppSetting                                         | ProquifaDotNetTimbrado     | DDL  | Alta      |
| 3   | CREATE TABLE StampingLog                                        | ProquifaDotNetTimbrado     | DDL  | Alta      |
| 4   | ALTER TABLE CFDIGenerada (columnas tecnicas de timbrado)        | ProquifaDotNet              | DDL  | Alta      |
| -   | ProquifaDotNet (Parte 1 — pendientes FAA)                        | Solo lectura - sin cambios | -    | -         |

> **Nota (Reglas al diseñar — regla 2):** `ProquifaDotNetTimbrado` es una base de datos **nueva**, por lo que su estructura (tablas, columnas, PK/FK, constraints) se nombra en **inglés**. La BD `ProquifaDotNet` (Parte 1 y Parte 3, incluyendo `CFDIGenerada`) conserva su nomenclatura en español por ser la base de datos existente (regla 1) — `CFDIGenerada` es una tabla nueva, pero vive dentro de `ProquifaDotNet`, por lo que sigue la convención española ya establecida en `ER-Finanzas.md`.
>
> **Nota (nomenclatura Timbrado -> Stamping):** la tabla `StampingLog` (antes `TimbradoLog`) se traduce a inglés para no mezclar idiomas dentro de la estructura de la BD nueva. El nombre de la base de datos (`ProquifaDotNetTimbrado`) y de la solución (`ProquifaDotNet.Timbrado`) se mantienen sin traducir por ser nomenclatura ya establecida en las instrucciones del proyecto — la excepción aplica solo al nombre de la solución/BD, no a las tablas ni al código interno.

---

## Parte 1: Lectura de ProquifaDotNet (migrada a `fccFactura`/`vfccFactura` — 06/07/2026)

> **Migración:** este requisito consultaba la cadena `tpPedido → tpPedidoProformaPedido → tpProformaPedido → tpProformaAdelantoProformaPedido → fccPagoFacturaAdelanto → tpProformaAdelanto`. Esa cadena de 5 saltos **ya no existe** para pedidos Prepago (RE-FU-015 no genera `tpProformaPedido` ni `tpProformaAdelanto`), lo cual dejaba fuera del listado a los pedidos Prepago con FAA — hallazgo H-01 de `R16A-RE-FU-018_DIS-SOL_Revision.md`. Se resuelve unificando el origen de lectura en `fccFactura` (RE-FU-015), que tiene FK directa a `tpPedido` y cubre ambos orígenes (Prepago y Crédito, este último vía `IdTPProformaPedido`). Ver `R16A-RE-FU-015_BD.md`, sección "Migración de tpProformaAdelanto".

### Cadena de Datos - Pendientes FAA (nueva)

    fccFactura (pendiente FAA, EsFacturaPorAdelantado=1)
        IdTPPedido -> tpPedido (FK directa, cubre Prepago y Crédito)
        IdTPProformaPedido -> tpProformaPedido (poblado solo en origen Crédito, RE-FU-012)
        IdCFDIGenerada IS NULL = factura NO generada
        Enviada = 0 = factura generada pero NO enviada

    vfccFactura (vista, ver R16A-RE-FU-015_BD.md) resuelve EstadoFAA:
        'PendienteGenerar' | 'PendienteEnviar' | 'Completada'

### Identificacion de Pendientes

    vfccFactura.FacturaPorAdelantado = 1
    AND vfccFactura.Activo = 1
    AND vfccFactura.EstadoFAA IN ('PendienteGenerar', ' ')

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
|------------|-------------|-----|
| Razon Social | fccFactura (snapshot) / vfccFactura | RazonSocialReceptor / ClienteRazonSocial |
| RFC/RUC | fccFactura (snapshot) / vfccFactura | RfcReceptor / ClienteRFC |
| Facturas Pendientes | COUNT(vfccFactura) | WHERE Activo=1 AND EstadoFAA IN ('PendienteGenerar','PendienteEnviar') |
| Monto Total (USD) | SUM(vfccFactura.MontoTotal) | Convertido a USD |
| Antiguedad | MIN(vfccFactura.FechaRegistro) | Pendiente mas antiguo |

### Buscador

| Campo | Tabla | Busqueda |
|-------|-------|----------|
| Razon Social | DatosFacturacionCliente.RazonSocial | LIKE '%texto%' |
| RFC/RUC | DatosFacturacionCliente.RFC | LIKE '%texto%' |
| Folio Pedido | tpPedido.FolioPedidoInterno | LIKE '%texto%' |

### Tablas Consultadas (ProquifaDotNet - Lectura)

| Tabla | Rol |
|-------|-----|
| vfccFactura | Vista consolidada del pendiente FAA — MontoTotal, IdCFDIGenerada, Enviada, EstadoFAA, FechaRegistro, FolioPedidoInterno, Region (reemplaza la cadena `tpProformaAdelanto`/`vtpProformaAdelanto`) |
| fccFactura | Tabla base de la vista — snapshot RazonSocialReceptor/RfcReceptor (ya no requiere JOIN a `DatosFacturacionCliente` para estos campos) |
| tpPedido | FacturaPorAdelantado=1, Tramitado=1, FolioPedidoInterno (vía FK directa `fccFactura.IdTPPedido`) |
| Cliente | IdCliente |
| ClienteCarteraCliente | Vinculo cliente-cartera |
| ClienteCartera | IdUsuarioCobrador |
| catMoneda | Para conversion a USD |
| Region | MEX/PER |

> `tpProformaAdelanto`, `tpProformaPedido`, `tpPedidoProformaPedido`, `tpProformaAdelantoProformaPedido` **ya no se consultan directamente** desde este requisito — sustituidos por `fccFactura`/`vfccFactura`. `fccPagoFacturaAdelanto` deja de ser necesaria en la cadena de lectura de este listado (aunque sigue vigente para RE-FU-026/027/028/029/030, con su FK migrada a `IdFccFactura`).

---

## Parte 2: Base de Datos Nueva - ProquifaDotNetTimbrado

### Proposito
BD independiente y **puramente tecnica** para el servicio ProquifaDotNet.Timbrado (.NET 10).
Almacena configuracion y auditoria de la llamada al PAC (SAP). **No almacena el registro de
negocio del CFDI** — ese vive en `CFDIGenerada` (ProquifaDotNet), propiedad de Finanzas.
Se comunica con ProquifaDotNet.Finanzas via API: Finanzas envia los datos fiscales ya armados,
Timbrado invoca al PAC y regresa UUID + XML + estatus (sin persistirlos como entidad de negocio propia).

### Servidor y rutas

| Aspecto | Valor |
|---------|-------|
| Servidor | WIN-R14-DEV\\DEV_R17_APPS |
| Data file | R:\\SSQL DATA R17 APP\\ProquifaDotNetTimbrado.mdf |
| Log file | Y:\\SSQL LOG R17 PQF\\ProquifaDotNetTimbrado_log.ldf |
| Compatibility | 160 (SQL Server 2022) |

### Diagrama de tablas

    AppSetting (configuracion del servicio)

    StampingLog (auditoria tecnica de la llamada al PAC)
        PK: Id
        CfdiGeneradaId: uniqueidentifier (referencia informativa a CFDIGenerada.IdCFDIGenerada
                        en ProquifaDotNet; NO es FK real por tratarse de bases de datos distintas)

---

### Tabla: AppSetting
**Proposito:** Configuracion del servicio de timbrado (endpoints SAP, timeouts, etc.). No incluye configuracion de reintentos: Timbrado no reintenta, es un servicio de un solo intento por peticion (la politica de reintento la maneja Finanzas, de forma local en cada punto de generacion: Factura, Factura por Adelantado, Nota de Credito, Complemento de Pago).

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| Id | uniqueidentifier | NO | NEWID() | PK |
| Name | varchar(50) | NO | - | Clave de configuracion |
| Value | varchar(max) | NO | - | Valor (puede ser JSON) |
| Description | varchar(100) | NO | - | Descripcion legible |
| CreatedAt | datetime | NO | GETDATE() | Fecha creacion |
| UpdatedAt | datetime | NO | GETDATE() | Fecha actualizacion |
| IsActive | bit | NO | 1 | Activo |

---

### Tabla: StampingLog
**Proposito:** Auditoria tecnica de la peticion de timbrado con el PAC (SAP). Un solo registro por solicitud recibida de Finanzas (sin reintentos internos): Timbrado invoca al PAC una vez y registra el resultado. No tiene FK real a `CFDIGenerada` (esta en otra base de datos); `CfdiGeneradaId` se guarda solo como referencia informativa para trazabilidad/soporte.

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| Id | uniqueidentifier | NO | NEWID() | PK |
| CfdiGeneradaId | uniqueidentifier | SI | - | Referencia informativa a `CFDIGenerada.IdCFDIGenerada` (ProquifaDotNet) — no es FK real, es cross-database |
| Action | varchar(50) | NO | - | Stamp/Cancel |
| PreviousStatus | varchar(30) | SI | - | Estado antes de la accion |
| NewStatus | varchar(30) | NO | - | Estado despues de la accion |
| Request | varchar(max) | SI | - | Payload enviado al PAC |
| Response | varchar(max) | SI | - | Respuesta del PAC |
| ErrorMessage | varchar(max) | SI | - | Error si fallo |
| DurationMs | int | SI | - | Tiempo de respuesta en ms |
| CreatedAt | datetime | NO | GETDATE() | Fecha del intento |
| IsActive | bit | NO | 1 | Activo |

---

### Script completo (ProquifaDotNetTimbrado)

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Crear base de datos ProquifaDotNetTimbrado y tablas iniciales (solo tecnicas)

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
        [CreatedAt] datetime NOT NULL CONSTRAINT [DF_AppSetting_CreatedAt] DEFAULT (GETDATE()),
        [UpdatedAt] datetime NOT NULL CONSTRAINT [DF_AppSetting_UpdatedAt] DEFAULT (GETDATE()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_AppSetting_IsActive] DEFAULT (1),
        CONSTRAINT [PK_AppSetting] PRIMARY KEY CLUSTERED ([Id])
    );
    GO

    CREATE TABLE [dbo].[StampingLog](
        [Id] uniqueidentifier NOT NULL CONSTRAINT [DF_StampingLog_Id] DEFAULT (NEWID()),
        [CfdiGeneradaId] uniqueidentifier NULL,
        [Action] varchar(50) NOT NULL,
        [PreviousStatus] varchar(30) NULL,
        [NewStatus] varchar(30) NOT NULL,
        [Request] varchar(max) NULL,
        [Response] varchar(max) NULL,
        [ErrorMessage] varchar(max) NULL,
        [DurationMs] int NULL,
        [CreatedAt] datetime NOT NULL CONSTRAINT [DF_StampingLog_CreatedAt] DEFAULT (GETDATE()),
        [IsActive] bit NOT NULL CONSTRAINT [DF_StampingLog_IsActive] DEFAULT (1),
        CONSTRAINT [PK_StampingLog] PRIMARY KEY CLUSTERED ([Id])
        -- Sin FK: CfdiGeneradaId referencia CFDIGenerada en la base de datos ProquifaDotNet (otra BD)
    );
    GO

---

## Parte 3: Extension de CFDIGenerada (ProquifaDotNet — propiedad de Finanzas)

### Proposito
`CFDIGenerada` (diseñada en `ER-Finanzas.md`, propiedad de `ProquifaDotNet.Finanzas`) es el
registro central de negocio de todo CFDI emitido. Se extiende aqui con las columnas tecnicas
que el flujo de timbrado de este requisito necesita (estatus, error, moneda, tipo de cambio,
uso CFDI, metodo de pago, referencia al archivo XML). El catalogo de tipo de documento sigue
siendo **`catTipoCFDI`** (ya diseñado en Finanzas) — no se crea un catalogo adicional.

> **Catalogos reutilizados (ya existen en ProquifaDotNet, sin cambios):** `catUsoCFDI`, `catMetodoDePagoCFDI`, `catMoneda`. `catTipoCFDI` se crea/altera en `R16A-RE-FU-028` (no se duplica aqui, solo se referencia como dependencia).

### ALTER TABLE CFDIGenerada

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Ejecutar sobre ProquifaDotNet. Prerequisito: CFDIGenerada debe existir (ER-Finanzas.md / RE-FU-019)
    ALTER TABLE dbo.CFDIGenerada
        ADD [IdCatUsoCFDI]         uniqueidentifier NULL,
            [IdCatMetodoDePagoCFDI] uniqueidentifier NULL,
            [IdCatMoneda]           uniqueidentifier NULL,
            [TipoCambio]            decimal(18,6) NULL,
            [Total]                 decimal(18,2) NULL,
            [IdArchivoXml]          uniqueidentifier NULL,
            [Estado]                varchar(30) NOT NULL
                CONSTRAINT [DF_CFDIGenerada_Estado] DEFAULT ('Pendiente'),
            [MensajeError]          varchar(max) NULL,
            [FechaUltimaActualizacion] datetime NOT NULL
                CONSTRAINT [DF_CFDIGenerada_FechaUltimaActualizacion] DEFAULT (GETDATE());
    GO

    ALTER TABLE dbo.CFDIGenerada
        ADD CONSTRAINT [FK_CFDIGenerada_CatUsoCFDI]
            FOREIGN KEY ([IdCatUsoCFDI]) REFERENCES dbo.catUsoCFDI([IdCatUsoCFDI]),
        CONSTRAINT [FK_CFDIGenerada_CatMetodoDePagoCFDI]
            FOREIGN KEY ([IdCatMetodoDePagoCFDI]) REFERENCES dbo.catMetodoDePagoCFDI([IdCatMetodoDePagoCFDI]),
        CONSTRAINT [FK_CFDIGenerada_CatMoneda]
            FOREIGN KEY ([IdCatMoneda]) REFERENCES dbo.catMoneda([IdCatMoneda]),
        CONSTRAINT [FK_CFDIGenerada_Archivo]
            FOREIGN KEY ([IdArchivoXml]) REFERENCES dbo.Archivo([IdArchivo]);
    GO

### Columnas agregadas a CFDIGenerada

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| IdCatUsoCFDI | uniqueidentifier | SI | - | FK -> catUsoCFDI (clave SAT uso CFDI, ej. G03/P01) |
| IdCatMetodoDePagoCFDI | uniqueidentifier | SI | - | FK -> catMetodoDePagoCFDI (PUE/PPD) |
| IdCatMoneda | uniqueidentifier | SI | - | FK -> catMoneda (MXN/USD/PEN) |
| TipoCambio | decimal(18,6) | SI | - | Tipo de cambio aplicado al emitir |
| Total | decimal(18,2) | SI | - | Monto total del CFDI (usado ya por `vtpProformaAdelanto`, formalizado aqui) |
| IdArchivoXml | uniqueidentifier | SI | - | FK -> Archivo (XML del CFDI timbrado, mismo patron que `fccNotaCredito.IdArchivoXml`) |
| Estado | varchar(30) | NO | 'Pendiente' | Pendiente/Timbrado/Fallido — mapea al enum `StampStatus` (Pending/Stamped/Failed) del lado de Finanzas |
| MensajeError | varchar(max) | SI | - | Detalle del error si el timbrado fallo |
| FechaUltimaActualizacion | datetime | NO | GETDATE() | Fecha de la ultima actualizacion de estatus |

> **Nota:** el XML no se guarda como blob en `CFDIGenerada` — se sube a Minio y se registra en `Archivo` (patron ya usado por `RE-FU-016` y por `fccNotaCredito.IdArchivoXml`/`IdArchivoPdf`), y `CFDIGenerada.IdArchivoXml` solo referencia ese registro.

---

## Flujo de Datos (timbrado, corregido)

    1. Finanzas arma los datos fiscales (RFC emisor/receptor, conceptos, uso CFDI, metodo de pago)
    2. Finanzas llama a ProquifaDotNet.Timbrado (servicio tecnico) con esos datos
    3. Timbrado invoca al PAC (SAP), registra el intento en StampingLog (ProquifaDotNetTimbrado)
       y regresa a Finanzas: UUID, Serie, Folio, XML, estatus
    4. Finanzas:
       - INSERT/UPDATE CFDIGenerada (ProquifaDotNet) con UUID, Folio, Serie, FechaEmision,
         IdCatTipoCFDI, Total, IdCatUsoCFDI, IdCatMetodoDePagoCFDI, IdCatMoneda, TipoCambio, Estado
       - INSERT Archivo (XML en Minio) + UPDATE CFDIGenerada SET IdArchivoXml
       - UPDATE fccFactura SET IdCFDIGenerada = @IdCFDIGenerada (el Id real de CFDIGenerada;
         antes: UPDATE tpProformaAdelanto SET IdCFDIGenerada — ver migración RE-FU-015)

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | ~~Campo/logica 'factura generada pero no enviada'~~ | Tecnico | **Resuelto**: `fccFactura.Enviada` (migrado de `tpProformaAdelanto.Enviada`, ver RE-FU-015/019) |
| 2 | Tipo de cambio para dolarizacion | Negocio | Confirmar TC historico vs dia actual |
| 3 | Rol operativo: Gestor Cobranza vs Analista CxC | Negocio | Confirmar denominacion |
| 4 | Timbrado Peru (OSE/SUNAT) | Tecnico | Brecha mayor - RE-FU-005 B5 |
| 5 | Cliente sin Cobrador asignado | Operativo | Pendiente invisible |

---

## Dependencias

| Requisito      | Relacion                                     |
| -------------- | --------------------------------------------- |
| R16A-RE-FU-012 | Genera pendiente FAA (`fccFactura`, `IdTPProformaPedido` poblado) para Credito con FAA |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria`/`vfccFactura` — genera el pendiente FAA para Prepago (`IdTPProformaPedido` NULL) |
| R16A-RE-FU-005 | Timbrado Peru (brecha)                       |
| R16A-RE-FU-016 | Proforma MEX (flujo previo a FAA en Prepago) |
| R16A-RE-FU-019 | CREATE TABLE CFDIGenerada (ER-Finanzas.md)    |
| R16A-RE-FU-028 | catTipoCFDI (catalogo de tipo de CFDI)        |

---

**Generado por:** GitHub Copilot in SSMS
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (nueva)
