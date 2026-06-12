# Impacto en Back - R16A-RE-FU-018
**Requisito:** Factura por Adelantado (Pantalla Inicial)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10, NUEVA)
**Modulo:** Factura por Adelantado (nuevo) + Timbrado (nuevo)
**Impacto:** Creacion de solucion ProquifaDotNet.Timbrado + modulo FAA en Finanzas + listado agrupado por cliente

---

## Resumen

Este requisito tiene **doble impacto**:

1. **Funcionalidad:** Pantalla inicial del modulo Factura por Adelantado (listado agrupado por cliente con pendientes FAA, filtrado por cartera, buscador, paginacion)
2. **Infraestructura:** Creacion de la solucion ProquifaDotNet.Timbrado (.NET Core 10) que gestiona el timbrado fiscal (CFDI) necesario para completar el flujo FAA

La funcionalidad del listado se implementa en **ProquifaDotNet.Finanzas** (modulo FAA). La solucion **ProquifaDotNet.Timbrado** se crea como prerequisito arquitectonico para el flujo completo de facturacion.

### Soluciones involucradas

| Solucion | Rol en RE-FU-018 | Estado |
|----------|-----------------|--------|
| ProquifaDotNet (.NET Framework 4.8) | Consumidor: llama API Finanzas para obtener listado FAA | Existente |
| ProquifaDotNet.Finanzas (.NET Core 10) | Modulo FAA: endpoint listado agrupado por cliente, consulta BD | Creada en RE-FU-016 |
| ProquifaDotNet.Timbrado (.NET Core 10) | Servicio de timbrado fiscal: recepcion solicitudes, integracion SAP, CFDI | **NUEVA - crear en este requisito** |

---

## Parte A - Creacion de Solucion ProquifaDotNet.Timbrado

### Descripcion

Solucion independiente en .NET Core 10 que gestiona el timbrado fiscal de documentos. Recibe solicitudes de timbrado desde Finanzas, invoca al proveedor SAP para generar CFDI, almacena XML en BD y Minio, y retorna el resultado a Finanzas.

### Referencia arquitectonica

Basada en el repositorio proquifa-punchout-backend (.NET 10, Clean Architecture).

### Estructura de la Solucion

```
ProquifaDotNet.Timbrado/
+-- Proquifa.Timbrado.sln
+-- Domain/
|   +-- Proquifa.Timbrado.Domain.csproj (net10.0)
|   +-- Common/
|   |   +-- QueryInfo.cs
|   |   +-- FilterItem.cs
|   |   +-- SortDirection.cs
|   +-- Entities/
|   |   +-- Cfdi.cs
|   |   +-- TimbradoLog.cs
|   |   +-- TipoDocumentoFiscal.cs
|   |   +-- AppSetting.cs
|   +-- Interfaces/
|   |   +-- IGenericRepository.cs
|   |   +-- ICfdiRepository.cs
|   |   +-- ITimbradoLogRepository.cs
|   |   +-- ISapTimbradoClient.cs
|   |   +-- IMinioStorageService.cs
|   |   +-- IEmailService.cs
|   |   +-- IUnitOfWork.cs
|   +-- Models/
|       +-- TimbradoRequest.cs
|       +-- TimbradoResponse.cs
|       +-- EmailModel.cs
+-- Application/
|   +-- Proquifa.Timbrado.Application.csproj (net10.0)
|   +-- DTOs/
|   |   +-- QueryResultDto.cs
|   |   +-- CfdiDto.cs
|   |   +-- TimbradoRequestDto.cs
|   |   +-- TimbradoResponseDto.cs
|   |   +-- TimbradoLogDto.cs
|   +-- Interfaces/
|   |   +-- ITimbradoService.cs
|   |   +-- ICfdiService.cs
|   +-- Services/
|   |   +-- TimbradoService.cs
|   |   +-- CfdiService.cs
|   +-- Mappers/
|   |   +-- ApplicationMappingProfile.cs
|   +-- Validators/
|       +-- TimbradoRequestDtoValidator.cs
+-- Infrastructure/
|   +-- Proquifa.Timbrado.Infrastructure.csproj (net10.0)
|   +-- Persistence/
|   |   +-- Context/
|   |   |   +-- TimbradoContext.cs (BD ProquifaDotNetTimbrado)
|   |   +-- Repository/
|   |       +-- GenericRepository.cs
|   |       +-- CfdiRepository.cs
|   |       +-- TimbradoLogRepository.cs
|   +-- Services/
|   |   +-- SapTimbradoClient.cs
|   |   +-- MinioStorageService.cs
|   |   +-- BrevoEmailService.cs
|   +-- RabbitMQ/
|   |   +-- IRabbitMQClient.cs
|   |   +-- RabbitMQClient.cs
|   |   +-- RabbitMQSettings.cs
|   +-- Configuration/
|   |   +-- SapSettings.cs
|   |   +-- MinioSettings.cs
|   |   +-- BrevoSettings.cs
|   |   +-- RabbitMQSettings.cs
|   +-- Extensions/
|       +-- InfrastructureServiceExtensions.cs
+-- API/
|   +-- Proquifa.Timbrado.API.csproj (net10.0)
|   +-- Program.cs
|   +-- Controllers/
|   |   +-- TimbradoController.cs
|   |   +-- CfdiController.cs
|   +-- appsettings.json
+-- Worker/
|   +-- Proquifa.Timbrado.Worker.csproj (net10.0)
|   +-- Program.cs
|   +-- TimbradoWorker.cs
+-- Testing/
    +-- Proquifa.Timbrado.Testing.csproj (net10.0)
```

---

### Flujo Funcional de Timbrado

```
1. Finanzas solicita timbrado -> POST /api/timbrado/timbrar (con datos del documento)
2. TimbradoService valida request y crea registro CFDI en BD (EstatusTimbrado=Pendiente)
3. TimbradoService invoca SapTimbradoClient -> PAC SAP genera CFDI
4. SAP retorna: UUID, XML firmado, Serie, Folio
5. TimbradoService actualiza CFDI en BD (EstatusTimbrado=Timbrado, XML, UUID)
6. Sube XML a Minio (bucket timbrado)
7. Registra TimbradoLog con Request/Response/Duracion
8. Retorna TimbradoResponse a Finanzas (UUID, Serie, Folio, XML)
```

### Flujo Asincrono (Worker)

```
1. Si timbrado sincrono falla -> publica mensaje en cola RabbitMQ
2. Worker.Timbrado consume cola -> reintenta timbrado con SAP
3. Si exito: actualiza CFDI + notifica a Finanzas via API
4. Si falla despues de N reintentos: marca como Error + notifica via Brevo
```

---

### Capas y Componentes Principales

#### Domain - Entidades

| Entidad | Tabla BD | Descripcion |
|---------|---------|-------------|
| Cfdi | CFDI | Documento fiscal: UUID, Serie, Folio, XML, estatus, empresa, receptor |
| TimbradoLog | TimbradoLog | Auditoria de cada intento: request, response, duracion, error |
| TipoDocumentoFiscal | TipoDocumentoFiscal | Catalogo: FacturaPorAdelantado, FacturaNormal, NotaCredito, etc. |
| AppSetting | AppSetting | Configuracion del servicio: endpoints SAP, reintentos, timeouts |

#### Application - Servicios

| Servicio | Responsabilidad |
|----------|----------------|
| TimbradoService | Orquesta: validar request, crear CFDI, llamar SAP, almacenar XML, registrar log |
| CfdiService | CRUD de CFDIs + consultas con QueryInfo + listado paginado |

#### Infrastructure - Integraciones

| Integracion | Componente | Descripcion |
|-------------|-----------|-------------|
| SAP (PAC) | SapTimbradoClient | Llamada HTTP al proveedor de timbrado para generar CFDI |
| Minio | MinioStorageService | Almacenamiento de XML timbrados (bucket 'timbrado') |
| RabbitMQ | RabbitMQClient + Worker | Cola para reintentos asincronos de timbrado fallido |
| Brevo | BrevoEmailService | Notificaciones de errores criticos de timbrado |
| IdentityServer | Autenticacion | Validacion de tokens desde Finanzas |
| Serilog | Logs | Contexto: usuario, modulo, operacion, IdCFDI |

#### API - Endpoints

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | /api/timbrado/timbrar | Recibe solicitud de timbrado, retorna CFDI generado |
| POST | /api/timbrado/cancelar | Solicita cancelacion de CFDI ante SAP |
| GET | /api/cfdi/{id} | Consulta CFDI por Id |
| GET | /api/cfdi/{id}/xml | Descarga XML desde Minio |
| POST | /api/cfdi/listar | Listado paginado con QueryInfo |

---

### Base de Datos: ProquifaDotNetTimbrado (Nueva)

| Tabla | Proposito |
|-------|-----------|
| AppSetting | Configuracion del servicio (endpoints SAP, reintentos, etc.) |
| TipoDocumentoFiscal | Catalogo de tipos de documentos fiscales |
| CFDI | Documento fiscal generado: UUID, XML, estatus, intentos |
| TimbradoLog | Auditoria de cada intento con el PAC |

> Servidor: WIN-R14-DEV\DEV_R17_APPS
> BD creada con script DDL documentado en R16A-RE-FU-018_BD.md

---

### Paquetes NuGet Requeridos (Timbrado)

| Proyecto | Paquete | Uso |
|----------|---------|-----|
| Infrastructure | Microsoft.EntityFrameworkCore.SqlServer | EF Core + SQL Server |
| Infrastructure | RabbitMQ.Client 7.x | Colas de timbrado |
| Infrastructure | Minio 6.x | Almacenamiento XML |
| Infrastructure | Serilog 4.x | Logs |
| Infrastructure | Polly 8.x | Reintentos HTTP a SAP |
| Infrastructure | sib_api_v3_sdk 4.x | Brevo (notificaciones) |
| Application | FluentValidation 11.x | Validaciones |
| API | Serilog.AspNetCore 9.x | Logs en API |
| API | Swashbuckle.AspNetCore 6.x | Swagger |

---

## Parte B - Modulo Factura por Adelantado en ProquifaDotNet.Finanzas

### Descripcion

Nuevo modulo en la solucion Finanzas que expone el endpoint de listado agrupado por cliente con pedidos pendientes de Factura por Adelantado. Consulta datos de BD ProquifaDotNet (lectura).

### Endpoint Listado FAA

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | /api/factura-adelantado/listar | Listado agrupado por cliente, paginado, filtrado por cartera |

### Request

```
QueryInfo {
  SortField: "antiguedad" (default),
  SortDirection: Asc,
  Filters: [
    { FieldName: "idUsuarioCobrador", Value: "{IdUsuarioLogueado}" },
    { FieldName: "busqueda", Value: "texto libre (razonSocial/RFC/folio)" }
  ],
  PageSize: 20,
  DesiredPage: 1
}
```

### Response

```
QueryResultDto<FacturaAdelantadoClienteDto> {
  TotalResults: int,
  Results: [
    {
      IdCliente: Guid,
      RazonSocial: string,
      RfcRuc: string,
      RegionClave: string (MEX/PER),
      FacturasPendientes: int,
      MontoTotalUsd: decimal,
      FechaPendienteMasAntiguo: DateTime
    }
  ]
}
```

### Cadena de Datos (Consulta BD ProquifaDotNet)

```
tpPedido (FacturaPorAdelantado=1, Tramitado=1, Activo=1, Region=MEX)
  -> tpPedidoProformaPedido -> tpProformaPedido (IdCliente)
  -> tpProformaAdelantoProformaPedido -> fccPagoFacturaAdelanto -> tpProformaAdelanto

Filtro: tpProformaAdelanto.Activo=1 AND (IdCFDIGenerada IS NULL OR no enviada)
        AND tpPedido.IdRegion = @IdRegionMexico  -- OBS-032/033: FAA solo aplica para Mexico; clientes Peru se excluyen del listado
        -- OBS-034: "no enviada" incluye los casos donde el CFDI ya fue timbrado pero el usuario AÚN NO ha ejecutado "Enviar Factura"
        --          El envío de la factura es una acción EXPLÍCITA del usuario, NO automática post-timbrado.

Agrupacion: GROUP BY Cliente
  COUNT(tpProformaAdelanto) AS FacturasPendientes
  SUM(MontoConvertidoUSD) AS MontoTotalUsd
  MIN(tpProformaAdelanto.FechaRegistro) AS FechaPendienteMasAntiguo

Filtro cartera: ClienteCarteraCliente -> ClienteCartera.IdUsuarioCobrador = @IdUsuario
Buscador: TRIM(DatosFacturacionCliente.RazonSocial) LIKE / TRIM(RFC) LIKE / TRIM(tpPedido.FolioPedidoInterno) LIKE
          -- OBS-041: trim automatico aplicado al texto ingresado antes de ejecutar el filtrado
```

### Tablas consultadas (ProquifaDotNet - Lectura)

| Tabla | Rol |
|-------|-----|
| tpPedido | FacturaPorAdelantado=1, Tramitado=1, FolioPedidoInterno |
| tpProformaAdelanto | Monto, IdCFDIGenerada, FechaRegistro, Activo |
| tpProformaPedido | Vinculo proforma-pedido, IdCliente |
| tpPedidoProformaPedido | Relacion pedido-proforma |
| tpProformaAdelantoProformaPedido | Relacion adelanto-proforma |
| fccPagoFacturaAdelanto | Relacion pago-adelanto |
| Cliente | IdCliente |
| DatosFacturacionCliente | RazonSocial, RFC (RFC/RUC) |
| ClienteCarteraCliente | Vinculo cliente-cartera |
| ClienteCartera | IdUsuarioCobrador |
| catMoneda | Para conversion a USD |
| Region | MEX/PER |

### Nota: Enviar Factura — acción explícita del usuario (OBS-034)

> **OBS-034:** El envío de la factura al cliente es una **acción explícita del usuario**, NO ocurre de forma automática después del timbrado.
>
> Flujo esperado:
> 1. ESAC/Cobrador abre el pendiente FAA y ejecuta el proceso de timbrado (genera CFDI).
> 2. El sistema queda en estado "Timbrada — Pendiente de envío".
> 3. El usuario revisa y hace clic en **"Enviar Factura"** explícitamente.
> 4. Solo entonces se envía la factura al cliente (por correo, etc.).
>
> Las tareas de los requisitos RE-FU-019/020 deben respetar esta separación: timbrado y envío son dos pasos distintos con acción de usuario entre ellos.

### Nota: FAA vs Factura Anticipo — OBS-037

> **Decisión OBS-037:** La Factura por Adelantado (FAA) y la Factura Anticipo son instrumentos distintos e incompatibles:
>
> - **FAA (este requisito):** Pedidos de crédito con flag `FacturaPorAdelantado=1`. Exclusiva para región México. **NUNCA aplica para productos Sustancias Controladas.**
> - **Factura Anticipo:** Generada desde el flujo de Validar Cobro para clientes prepago + productos controlados. No corresponde al módulo FAA.
>
> El back debe validar que no se procese un pendiente FAA si el pedido incluye productos controlados. Esta validación ya está cubierta en RE-FU-012 (depende de RE-FU-011 `fnEsProductoControlado`). El FacturaAdelantadoRepository nunca debería encontrar pedidos controlados con FAA=1, pero si llegara a existir, se excluye del listado.

---

## Parte C - ProquifaDotNet (Venta Interna)

### Impacto

| #   | Componente        | Accion                                                         |
| --- | ----------------- | -------------------------------------------------------------- |
| 1   | ApiCallerFinanzas | Agregar metodo para llamar POST /api/factura-adelantado/listar |
| 2   | Controlador FAA   | Nuevo controlador en WebAPI que expone el listado al Front     |

> El modulo FAA es nuevo en Venta Interna. Se crea un controlador que delega a Finanzas.

---

## Gaps de Desarrollo

### En ProquifaDotNet.Timbrado (Solucion NUEVA)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Crear solucion y proyectos | sln + 6 csproj (Domain, Application, Infrastructure, API, Worker, Testing) | Medio |
| GAP-02 | Domain: Entities + Interfaces | Cfdi, TimbradoLog, TipoDocumentoFiscal, AppSetting + interfaces | Medio |
| GAP-03 | Application: DTOs + Services | TimbradoService, CfdiService, DTOs, Validators | Alto |
| GAP-04 | Infrastructure: TimbradoContext | EF Core con scaffold BD ProquifaDotNetTimbrado (4 tablas) | Medio |
| GAP-05 | Infrastructure: SapTimbradoClient | Cliente HTTP para invocar PAC SAP (timbrar/cancelar CFDI) | Alto |
| GAP-06 | Infrastructure: MinioStorageService | Upload/Download XML al bucket 'timbrado' | Bajo |
| GAP-07 | Infrastructure: RabbitMQ + Worker | Cola de reintentos asincronos + Worker consumer | Alto |
| GAP-08 | Infrastructure: BrevoEmailService | Notificaciones de errores criticos | Bajo |
| GAP-09 | API: Program.cs + configuracion | DI, EF Core, Swagger, Serilog, IdentityServer, CORS | Medio |
| GAP-10 | API: TimbradoController | Endpoints POST /timbrar, POST /cancelar | Medio |
| GAP-11 | API: CfdiController | Endpoints GET /{id}, GET /{id}/xml, POST /listar | Medio |
| GAP-12 | BD: Ejecutar scripts DDL | CREATE DATABASE + 4 tablas + DML catalogo | Bajo |

### En ProquifaDotNet.Finanzas (Modulo FAA)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-13 | Domain: DTO FacturaAdelantadoClienteDto | Modelo del listado agrupado | Bajo |
| GAP-14 | Infrastructure: FacturaAdelantadoRepository | Query compleja agrupada por cliente con JOINs a 12+ tablas | Alto |
| GAP-15 | Application: FacturaAdelantadoService | Listado con QueryInfo, filtro cartera, buscador, conversion USD | Alto |
| GAP-16 | API: FacturaAdelantadoController | Endpoint POST /api/factura-adelantado/listar | Bajo |
| GAP-17 | Infrastructure: FinanzasContext ampliacion | Agregar DbSets para tablas FAA (tpProformaAdelanto, etc.) | Medio |

### En ProquifaDotNet (Venta Interna)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-18 | ApiCallerFinanzas: metodo ListarFAA | Llamada POST /api/factura-adelantado/listar | Bajo |
| GAP-19 | Controlador FAA nuevo | Expone listado al Front, delega a Finanzas | Bajo |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-016 | Prerequisito: ProquifaDotNet.Finanzas ya creada (estructura, DI, API base) |
| R16A-RE-FU-005 | Brecha timbrado SUNAT Peru (OSE) - bloquea flujo FAA para Peru |

---

## Resumen de Gaps

| Repositorio | Cantidad | Detalle |
|-------------|----------|---------|
| ProquifaDotNet.Timbrado (NUEVA) | 12 (GAP-01 a GAP-12) | Creacion completa de solucion + BD |
| ProquifaDotNet.Finanzas | 5 (GAP-13 a GAP-17) | Modulo FAA listado agrupado |
| ProquifaDotNet | 2 (GAP-18 a GAP-19) | Consumidor del listado |
| **Total** | **19 gaps** | |
