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
|   |   +-- StampingLog.cs
|   |   +-- FiscalDocumentType.cs
|   |   +-- AppSetting.cs
|   +-- Interfaces/
|   |   +-- IGenericRepository.cs
|   |   +-- ICfdiRepository.cs
|   |   +-- IStampingLogRepository.cs
|   |   +-- ISapStampingClient.cs
|   |   +-- IMinioStorageService.cs
|   |   +-- IUnitOfWork.cs
|   +-- Models/
|       +-- StampingRequest.cs
|       +-- StampingResponse.cs
+-- Application/
|   +-- Proquifa.Timbrado.Application.csproj (net10.0)
|   +-- DTOs/
|   |   +-- QueryResultDto.cs
|   |   +-- CfdiDto.cs
|   |   +-- StampingRequestDto.cs
|   |   +-- StampingResponseDto.cs
|   |   +-- StampingLogDto.cs
|   +-- Interfaces/
|   |   +-- IStampingService.cs
|   |   +-- ICfdiService.cs
|   +-- Services/
|   |   +-- StampingService.cs
|   |   +-- CfdiService.cs
|   +-- Mappers/
|   |   +-- ApplicationMappingProfile.cs
|   +-- Validators/
|       +-- StampingRequestDtoValidator.cs
+-- Infrastructure/
|   +-- Proquifa.Timbrado.Infrastructure.csproj (net10.0)
|   +-- Persistence/
|   |   +-- Context/
|   |   |   +-- StampingContext.cs (BD ProquifaDotNetTimbrado)
|   |   +-- Repository/
|   |       +-- GenericRepository.cs
|   |       +-- CfdiRepository.cs
|   |       +-- StampingLogRepository.cs
|   +-- Services/
|   |   +-- SapStampingClient.cs
|   |   +-- MinioStorageService.cs
|   +-- Configuration/
|   |   +-- SapSettings.cs
|   |   +-- MinioSettings.cs
|   +-- Extensions/
|       +-- InfrastructureServiceExtensions.cs
+-- API/
|   +-- Proquifa.Timbrado.API.csproj (net10.0)
|   +-- Program.cs
|   +-- Controllers/
|   |   +-- CfdiController.cs
|   +-- appsettings.json
+-- Testing/
    +-- Proquifa.Timbrado.Testing.csproj (net10.0)
```

> **Nota de diseño (reintentos):** Timbrado **no tiene Worker ni cola de reintentos propia**. Es un servicio sincrono de un solo intento por peticion: recibe la solicitud, invoca una vez al PAC, registra el resultado en `StampingLog` (`NewStatus` = Pending -> Stamped o Failed) y responde de inmediato a Finanzas, exito o error. La politica de reintento ante fallo (cuantos intentos, cuando notificar a soporte) es responsabilidad exclusiva de **Finanzas**, implementada de forma local en cada punto de generacion del documento: Factura (R16A-RE-FU-028/029), Factura por Adelantado (R16A-RE-FU-019/020), Nota de Credito (R16A-RE-FU-032/033/034/035) y Complemento de Pago (R16A-RE-FU-030) — no de forma centralizada en un unico componente. Ver diagramas `Diagramas/Diagrama Secuencia Finanzas y Timbrado Factura.md` y `Diagramas/Diagrama Secuencia Encolamiento Finanzas y Timbrado Factura.md` (el patron de referencia: queda pendiente, incrementa contador de reintentos, notifica a soporte si supera el limite).

---

### Flujo Funcional de Timbrado (sincrono, un solo intento)

```
1. Finanzas solicita timbrado -> POST /api/v1/cfdi (con datos del documento)
2. StampingService valida request y registra StampingLog (NewStatus=Pending)
3. StampingService invoca SapStampingClient -> PAC SAP genera CFDI (una sola llamada, sin retry automatico)
4a. Si SAP responde exito: Uuid, XML firmado, Series, Folio
    - Crea/actualiza Cfdi en BD (XML, Uuid)
    - Sube XML a Minio (bucket timbrado)
    - Actualiza StampingLog (NewStatus=Stamped)
    - Registra en Bitacora General el timbrado exitoso (regla 8)
    - Retorna StampingResponse a Finanzas (Uuid, Series, Folio, XML)
4b. Si SAP responde error o no responde: actualiza StampingLog (NewStatus=Failed, ErrorMessage con el error) y registra en Bitacora General el timbrado fallido (regla 8)
    - Retorna el error a Finanzas de inmediato (sin reintentar internamente)
    - Finanzas decide si reintenta mas tarde, en su propio flujo de generacion del documento (Factura, Factura por Adelantado, Nota de Credito o Complemento de Pago)
```

> Timbrado no publica colas ni reintenta: el llamado es 1 peticion HTTP = 1 intento al PAC. Los reintentos ante fallo son responsabilidad de Finanzas.

---

### Capas y Componentes Principales

#### Domain - Entidades

| Entidad | Tabla BD | Descripcion |
|---------|---------|-------------|
| Cfdi | Cfdi | Documento fiscal: Uuid, Series, Folio, XML, Status, IssuerRfc, ReceiverRfc (nombres en ingles, ver R16A-RE-FU-018_BD.md) |
| StampingLog | StampingLog | Auditoria de la peticion (un registro por solicitud de timbrado): Request, Response, DurationMs, NewStatus (Pending/Stamped/Failed), ErrorMessage |
| FiscalDocumentType | FiscalDocumentType | Catalogo: AdvanceInvoice, RegularInvoice, AnticipatedInvoice, CreditNote |
| AppSetting | AppSetting | Configuracion del servicio: endpoints SAP, timeouts |

> **Nomenclatura (Reglas al diseñar — regla 6):** al ser una solucion nueva, las entidades, propiedades, DTOs, metodos y comentarios de codigo de ProquifaDotNet.Timbrado se codifican en ingles, sin mezclar palabras en español. `Cfdi`, `Rfc` y `Uuid` se mantienen como terminos fiscales estandar (no se traducen). La palabra "Timbrado" (timbrar = stampar/sellar fiscalmente) se traduce a **"Stamping"** dentro del codigo (`StampingLog`, `StampingService`, `SapStampingClient`, `StampingRequest/Response`, `StampingContext`) para no mezclar idiomas. Unica excepcion: el nombre de la solucion/proyecto (`ProquifaDotNet.Timbrado`, `Proquifa.Timbrado.*.csproj/.sln`) y el nombre de la base de datos (`ProquifaDotNetTimbrado`) se mantienen sin traducir por ser nomenclatura ya establecida en las instrucciones del proyecto.

#### Application - Servicios

| Servicio        | Responsabilidad                                                                 |
| --------------- | ------------------------------------------------------------------------------- |
| StampingService | Orquesta: validar request, crear CFDI, llamar SAP, almacenar XML, registrar log |
| CfdiService     | CRUD de CFDIs + consultas con QueryInfo + listado paginado                      |

#### Infrastructure - Integraciones

| Integracion    | Componente              | Descripcion                                             |
| -------------- | ----------------------- | ------------------------------------------------------- |
| SAP (PAC)      | SapStampingClient       | Llamada HTTP unica (sin retry) al proveedor de timbrado para generar CFDI |
| Minio          | MinioStorageService     | Almacenamiento de XML timbrados (bucket 'timbrado')     |
| IdentityServer | Autenticacion           | Validacion de tokens desde Finanzas                     |
| Serilog        | Logs                    | Contexto: usuario, modulo, operacion, IdCFDI            |
| Bitacora General (Aplicativo Nuevo) | BitacoraGeneralClient | Registro de auditoria de negocio de cada timbrado (exitoso o fallido) — Reglas al diseñar, regla 8 |

> RabbitMQ, Worker.Timbrado y Brevo se retiran del alcance de esta solucion: no hay reintentos ni notificaciones de error dentro de Timbrado. El reintento y la notificacion a soporte se implementan del lado de Finanzas, en cada punto de generacion del documento (Factura, Factura por Adelantado, Nota de Credito, Complemento de Pago — ver requisitos respectivos), y cualquier envio de correo (a soporte o al cliente) se realiza a traves del **Aplicativo para Envio de Correo (Aplicativo Nuevo)**, no con un cliente SMTP/Brevo propio de Finanzas ni de Timbrado (Reglas al diseñar, regla 7).

> **Bitacora General (regla 8):** Timbrado invoca al Aplicativo Bitacora General al registrar cada resultado de timbrado (exitoso o fallido) en `StampingLog`, igual que Finanzas debe hacerlo al guardar la factura, validar un cobro o guardar una proforma. El detalle de la integracion (contrato, endpoint) se define de forma transversal y se referencia aqui, no se documenta completo en este requisito.

#### API - Endpoints

> Rutas alineadas a `api/v1/{resource}/{id}/{subresource}` (Reglas al diseñar — regla 9): recurso singular en ingles (`cfdi`), CRUD por metodo HTTP, acciones especiales (`cancel`, `xml`, `search`) como subrecurso.

| Metodo | Endpoint                      | Descripcion                                          |
| ------ | ----------------------------- | ----------------------------------------------------- |
| POST   | /api/v1/cfdi                  | Recibe solicitud de timbrado (crea y timbra el CFDI) |
| POST   | /api/v1/cfdi/{id}/cancel      | Solicita cancelacion de CFDI ante SAP                |
| GET    | /api/v1/cfdi/{id}             | Consulta CFDI por Id                                 |
| GET    | /api/v1/cfdi/{id}/xml         | Descarga XML desde Minio                             |
| POST   | /api/v1/cfdi/search           | Listado paginado con QueryInfo                       |

---

### Base de Datos: ProquifaDotNetTimbrado (Nueva)

| Tabla | Proposito |
|-------|-----------|
| AppSetting | Configuracion del servicio (endpoints SAP, timeouts) |
| FiscalDocumentType | Catalogo de tipos de documentos fiscales |
| Cfdi | Documento fiscal generado: Uuid, XML, Status |
| StampingLog | Auditoria de la peticion de timbrado (un registro por solicitud, con NewStatus Pending/Stamped/Failed) |

> Servidor: WIN-R14-DEV\DEV_R17_APPS
> BD creada con script DDL documentado en R16A-RE-FU-018_BD.md

---

### Paquetes NuGet Requeridos (Timbrado)

| Proyecto | Paquete | Uso |
|----------|---------|-----|
| Infrastructure | Microsoft.EntityFrameworkCore.SqlServer | EF Core + SQL Server |
| Infrastructure | Minio 6.x | Almacenamiento XML |
| Infrastructure | Serilog 4.x | Logs |
| Infrastructure | Polly 8.x | Timeout/circuit-breaker en la llamada HTTP a SAP (sin politica de retry) |
| Application | FluentValidation 11.x | Validaciones |
| API | Serilog.AspNetCore 9.x | Logs en API |
| API | Swashbuckle.AspNetCore 6.x | Swagger |

---

## Parte B - Modulo Factura por Adelantado en ProquifaDotNet.Finanzas

### Descripcion

Nuevo modulo en la solucion Finanzas que expone el endpoint de listado agrupado por cliente con pedidos pendientes de Factura por Adelantado. Consulta datos de BD ProquifaDotNet (lectura).

### Endpoint Listado FAA

> Ruta alineada a `api/v1/{resource}/{id}/{subresource}` (Reglas al diseñar — regla 9): recurso `advanceInvoice` en ingles, accion de busqueda como subrecurso `search`.

| Metodo | Endpoint                       | Descripcion                                                  |
| ------ | ------------------------------ | ------------------------------------------------------------ |
| POST   | /api/v1/advanceInvoice/search | Listado agrupado por cliente, paginado, filtrado por cartera |

### Request

```
QueryInfo {
  SortField: "oldestPendingDate" (default),
  SortDirection: Asc,
  Filters: [
    { FieldName: "collectorUserId", Value: "{IdUsuarioLogueado}" },
    { FieldName: "search", Value: "texto libre (companyName/RFC/folio)" }
  ],
  PageSize: 20,
  DesiredPage: 1
}
```

### Response

```
QueryResultDto<AdvanceInvoiceClientDto> {
  TotalResults: int,
  Results: [
    {
      CustomerId: Guid,
      CompanyName: string,
      TaxId: string,
      RegionCode: string (MEX/PER),
      PendingInvoices: int,
      TotalAmountUsd: decimal,
      OldestPendingDate: DateTime
    }
  ]
}
```

> **Nomenclatura (Reglas al diseñar — regla 6):** DTO, campos y nombres de metodo/clase de este modulo se codifican en ingles por tratarse de ProquifaDotNet.Finanzas (solucion nueva). Las tablas y campos de la BD ProquifaDotNet consultados en modo lectura (seccion "Cadena de Datos" y "Tablas consultadas") conservan su nomenclatura en español (regla 1), pues pertenecen a la base de datos existente.

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
> 4. Solo entonces se envía la factura al cliente por correo, usando el **Aplicativo para Envio de Correo (Aplicativo Nuevo)** (Reglas al diseñar — regla 7); Finanzas no implementa un cliente SMTP propio para este envio.
>
> Las tareas de los requisitos RE-FU-019/020 deben respetar esta separación: timbrado y envío son dos pasos distintos con acción de usuario entre ellos.

### Nota: FAA vs Factura Anticipo — OBS-037

> **Decisión OBS-037:** La Factura por Adelantado (FAA) y la Factura Anticipo son instrumentos distintos e incompatibles:
>
> - **FAA (este requisito):** Pedidos de crédito con flag `FacturaPorAdelantado=1`. Exclusiva para región México. **NUNCA aplica para productos Sustancias Controladas.**
> - **Factura Anticipo:** Generada desde el flujo de Validar Cobro para clientes prepago + productos controlados. No corresponde al módulo FAA.
>
> El back debe validar que no se procese un pendiente FAA si el pedido incluye productos controlados. Esta validación ya está cubierta en RE-FU-012 (depende de RE-FU-011 `fnEsProductoControlado`). El AdvanceInvoiceRepository nunca debería encontrar pedidos controlados con FAA=1, pero si llegara a existir, se excluye del listado.

---

## Parte C - ProquifaDotNet (Venta Interna)

### Impacto

| #   | Componente        | Accion                                                         |
| --- | ----------------- | -------------------------------------------------------------- |
| 1   | ApiCallerFinanzas | Agregar metodo para llamar POST /api/v1/advanceInvoice/search |
| 2   | Controlador FAA   | Nuevo controlador en WebAPI que expone el listado al Front     |

> El modulo FAA es nuevo en Venta Interna. Se crea un controlador que delega a Finanzas.

---

## Gaps de Desarrollo

### En ProquifaDotNet.Timbrado (Solucion NUEVA)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Crear solucion y proyectos | sln + 5 csproj (Domain, Application, Infrastructure, API, Testing) | Medio |
| GAP-02 | Domain: Entities + Interfaces | Cfdi, StampingLog, FiscalDocumentType, AppSetting + interfaces | Medio |
| GAP-03 | Application: DTOs + Services | StampingService, CfdiService, DTOs, Validators | Alto |
| GAP-04 | Infrastructure: StampingContext | EF Core con scaffold BD ProquifaDotNetTimbrado (4 tablas) | Medio |
| GAP-05 | Infrastructure: SapStampingClient | Cliente HTTP para invocar PAC SAP (timbrar/cancelar CFDI), un solo intento por peticion, sin retry ni cola | Alto |
| GAP-06 | Infrastructure: MinioStorageService | Upload/Download XML al bucket 'timbrado' | Bajo |
| GAP-07 | API: Program.cs + configuracion | DI, EF Core, Swagger, Serilog, IdentityServer, CORS | Medio |
| GAP-08 | API: CfdiController | Endpoints POST /api/v1/cfdi, POST /api/v1/cfdi/{id}/cancel | Medio |
| GAP-09 | API: CfdiController (consulta) | Endpoints GET /api/v1/cfdi/{id}, GET /api/v1/cfdi/{id}/xml, POST /api/v1/cfdi/search | Medio |
| GAP-10 | BD: Ejecutar scripts DDL | CREATE DATABASE + 4 tablas + DML catalogo | Bajo |

> Se retiraron los gaps de Worker.Timbrado/RabbitMQ (reintentos) y BrevoEmailService (notificacion de error critico): esa responsabilidad es de Finanzas, en cada punto de generacion del documento (Factura, Factura por Adelantado, Nota de Credito, Complemento de Pago). GAP-08/09 se consolidan en un unico CfdiController (Reglas al diseñar — regla 9: un solo recurso `cfdi`, sin controller separado "Timbrado" para las acciones de escritura).

### En ProquifaDotNet.Finanzas (Modulo FAA)

> **Nomenclatura (regla 6):** nombres de clase/DTO en ingles (solucion nueva). **Ruta (regla 9):** recurso `advanceInvoice`.

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-11 | Domain: DTO AdvanceInvoiceClientDto | Modelo del listado agrupado | Bajo |
| GAP-12 | Infrastructure: AdvanceInvoiceRepository | Query compleja agrupada por cliente con JOINs a 12+ tablas | Alto |
| GAP-13 | Application: AdvanceInvoiceService | Listado con QueryInfo, filtro cartera, buscador, conversion USD | Alto |
| GAP-14 | API: AdvanceInvoiceController | Endpoint POST /api/v1/advanceInvoice/search | Bajo |
| GAP-15 | Infrastructure: FinanzasContext ampliacion | Agregar DbSets para tablas FAA (tpProformaAdelanto, etc.) | Medio |

### En ProquifaDotNet (Venta Interna)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-16 | ApiCallerFinanzas: metodo SearchAdvanceInvoices | Llamada POST /api/v1/advanceInvoice/search | Bajo |
| GAP-17 | Controlador FAA nuevo | Expone listado al Front, delega a Finanzas | Bajo |

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
| ProquifaDotNet.Timbrado (NUEVA) | 10 (GAP-01 a GAP-10) | Creacion completa de solucion + BD |
| ProquifaDotNet.Finanzas | 5 (GAP-11 a GAP-15) | Modulo FAA listado agrupado |
| ProquifaDotNet | 2 (GAP-16 a GAP-17) | Consumidor del listado |
| **Total** | **17 gaps** | |
