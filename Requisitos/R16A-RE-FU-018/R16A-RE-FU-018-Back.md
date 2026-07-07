# Impacto en Back - R16A-RE-FU-018
**Requisito:** Factura por Adelantado (Pantalla Inicial)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10, NUEVA)
**Modulo:** Factura por Adelantado (nuevo) + Timbrado (nuevo)
**Impacto:** Creacion de solucion ProquifaDotNet.Timbrado (servicio tecnico) + CfdiController/CFDIGenerada en Finanzas + modulo FAA (listado agrupado por cliente)

---

## Resumen

Este requisito tiene **triple impacto**:

1. **Funcionalidad:** Pantalla inicial del modulo Factura por Adelantado (listado agrupado por cliente con pendientes FAA, filtrado por cartera, buscador, paginacion)
2. **Infraestructura tecnica:** Creacion de la solucion ProquifaDotNet.Timbrado (.NET Core 10) — servicio tecnico de timbrado, **sin tabla de negocio propia**
3. **Infraestructura de negocio:** Creacion de `CfdiController` y persistencia del CFDI (`CFDIGenerada`) en **ProquifaDotNet.Finanzas**

> **Nota de arquitectura (correccion — el CFDI no va en Timbrado, va en Finanzas):** el registro de negocio del CFDI (`CfdiController`, tabla `CFDIGenerada`) es propiedad de **ProquifaDotNet.Finanzas**, no de Timbrado. Timbrado se reduce a un servicio tecnico que invoca al PAC (SAP) y regresa UUID/XML/estatus — no persiste el CFDI como entidad de negocio. Ver `R16A-RE-FU-018_BD.md` (Parte 2 y 3) para el detalle de BD.

La funcionalidad del listado se implementa en **ProquifaDotNet.Finanzas** (modulo FAA). La solucion **ProquifaDotNet.Timbrado** se crea como prerequisito arquitectonico tecnico para el flujo completo de facturacion.

### Soluciones involucradas

| Solucion | Rol en RE-FU-018 | Estado |
|----------|-----------------|--------|
| ProquifaDotNet (.NET Framework 4.8) | Consumidor: llama API Finanzas para obtener listado FAA | Existente |
| ProquifaDotNet.Finanzas (.NET Core 10) | Modulo FAA (listado) + CfdiController + persistencia CFDIGenerada | Creada en RE-FU-016 |
| ProquifaDotNet.Timbrado (.NET Core 10) | Servicio tecnico de timbrado: integracion SAP, sin tabla de negocio propia | **NUEVA - crear en este requisito** |

---

## Parte A - Creacion de Solucion ProquifaDotNet.Timbrado (servicio tecnico)

### Descripcion

Solucion independiente en .NET Core 10 que gestiona **unicamente el aspecto tecnico** del timbrado fiscal: recibe de Finanzas los datos fiscales ya armados, invoca al proveedor SAP para generar el CFDI, y **regresa el resultado a Finanzas** (UUID, XML, Series, Folio, estatus) sin persistir el documento como entidad de negocio propia. Finanzas es quien decide que hacer con ese resultado (guardarlo en `CFDIGenerada`, subir el XML a Minio via `Archivo`, etc.).

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
|   |   +-- StampingLog.cs
|   |   +-- AppSetting.cs
|   +-- Interfaces/
|   |   +-- IStampingLogRepository.cs
|   |   +-- ISapStampingClient.cs
|   |   +-- IUnitOfWork.cs
|   +-- Models/
|       +-- StampingRequest.cs
|       +-- StampingResponse.cs
+-- Application/
|   +-- Proquifa.Timbrado.Application.csproj (net10.0)
|   +-- DTOs/
|   |   +-- StampingRequestDto.cs
|   |   +-- StampingResponseDto.cs
|   |   +-- StampingLogDto.cs
|   +-- Interfaces/
|   |   +-- IStampingService.cs
|   +-- Services/
|   |   +-- StampingService.cs
|   +-- Mappers/
|   |   +-- ApplicationMappingProfile.cs
|   +-- Validators/
|       +-- StampingRequestDtoValidator.cs
+-- Infrastructure/
|   +-- Proquifa.Timbrado.Infrastructure.csproj (net10.0)
|   +-- Persistence/
|   |   +-- Context/
|   |   |   +-- StampingContext.cs (BD ProquifaDotNetTimbrado: AppSetting + StampingLog)
|   |   +-- Repository/
|   |       +-- StampingLogRepository.cs
|   +-- Services/
|   |   +-- SapStampingClient.cs
|   +-- Configuration/
|   |   +-- SapSettings.cs
|   +-- Extensions/
|       +-- InfrastructureServiceExtensions.cs
+-- API/
|   +-- Proquifa.Timbrado.API.csproj (net10.0)
|   +-- Program.cs
|   +-- Controllers/
|   |   +-- StampingController.cs
|   +-- appsettings.json
+-- Testing/
    +-- Proquifa.Timbrado.Testing.csproj (net10.0)
```

> **Nota de diseño (reintentos):** Timbrado **no tiene Worker ni cola de reintentos propia**. Es un servicio sincrono de un solo intento por peticion: recibe la solicitud, invoca una vez al PAC, registra el resultado en `StampingLog` (`NewStatus` = Pending -> Stamped o Failed) y responde de inmediato a Finanzas, exito o error. La politica de reintento ante fallo (cuantos intentos, cuando notificar a soporte) es responsabilidad exclusiva de **Finanzas**, implementada de forma local en cada punto de generacion del documento: Factura (R16A-RE-FU-028/029), Factura por Adelantado (R16A-RE-FU-019/020), Nota de Credito (R16A-RE-FU-032/033/034/035) y Complemento de Pago (R16A-RE-FU-030) — no de forma centralizada en un unico componente. Ver diagramas `Diagramas/Diagrama Secuencia Finanzas y Timbrado Factura.md` y `Diagramas/Diagrama Secuencia Encolamiento Finanzas y Timbrado Factura.md` (el patron de referencia: queda pendiente, incrementa contador de reintentos, notifica a soporte si supera el limite).

---

### Flujo Funcional de Timbrado (sincrono, un solo intento, sin persistencia de negocio)

```
1. Finanzas arma los datos fiscales del CFDI y llama -> POST /api/v1/stamp (servicio tecnico, uso interno)
2. StampingService valida request y registra StampingLog (NewStatus=Pending, CfdiGeneradaId=null aun)
3. StampingService invoca SapStampingClient -> PAC SAP genera CFDI (una sola llamada, sin retry automatico)
4a. Si SAP responde exito: Uuid, XML firmado, Series, Folio
    - Actualiza StampingLog (NewStatus=Stamped)
    - Registra en Bitacora General el timbrado exitoso (regla 8)
    - Retorna StampingResponse a Finanzas (Uuid, Series, Folio, XML, FechaEmision) — SIN persistir CFDI en Timbrado
4b. Si SAP responde error o no responde: actualiza StampingLog (NewStatus=Failed, ErrorMessage con el error) y registra en Bitacora General el timbrado fallido (regla 8)
    - Retorna el error a Finanzas de inmediato (sin reintentar internamente)
    - Finanzas decide si reintenta mas tarde, en su propio flujo de generacion del documento (Factura, Factura por Adelantado, Nota de Credito o Complemento de Pago)
5. Finanzas, al recibir un StampingResponse exitoso, es quien INSERT/UPDATE CFDIGenerada (ProquifaDotNet), sube el XML a Minio y crea el registro Archivo (ver Parte B) — Timbrado ya no participa en este paso.
```

> Timbrado no publica colas ni reintenta: el llamado es 1 peticion HTTP = 1 intento al PAC. Los reintentos ante fallo son responsabilidad de Finanzas. Timbrado tampoco persiste el CFDI como entidad de negocio — solo audita la llamada tecnica en `StampingLog`.

---

### Capas y Componentes Principales

#### Domain - Entidades

| Entidad | Tabla BD | Descripcion |
|---------|---------|-------------|
| StampingLog | StampingLog | Auditoria tecnica de la peticion (un registro por solicitud de timbrado): Request, Response, DurationMs, NewStatus (Pending/Stamped/Failed), ErrorMessage, CfdiGeneradaId (referencia informativa, no FK) |
| AppSetting | AppSetting | Configuracion del servicio: endpoints SAP, timeouts |

> **Nomenclatura (Reglas al diseñar — regla 6):** al ser una solucion nueva, las entidades, propiedades, DTOs, metodos y comentarios de codigo de ProquifaDotNet.Timbrado se codifican en ingles, sin mezclar palabras en español. `Cfdi`, `Rfc` y `Uuid` se mantienen como terminos fiscales estandar (no se traducen) cuando aparecen dentro de DTOs/modelos de intercambio (`StampingRequest`, `StampingResponse`). La palabra "Timbrado" (timbrar = stampar/sellar fiscalmente) se traduce a **"Stamping"** dentro del codigo (`StampingLog`, `StampingService`, `SapStampingClient`, `StampingRequest/Response`, `StampingContext`) para no mezclar idiomas. Unica excepcion: el nombre de la solucion/proyecto (`ProquifaDotNet.Timbrado`, `Proquifa.Timbrado.*.csproj/.sln`) y el nombre de la base de datos (`ProquifaDotNetTimbrado`) se mantienen sin traducir por ser nomenclatura ya establecida en las instrucciones del proyecto.

#### Application - Servicios

| Servicio        | Responsabilidad                                                                 |
| --------------- | ------------------------------------------------------------------------------- |
| StampingService | Orquesta: validar request, llamar SAP, registrar StampingLog, regresar el resultado a Finanzas (sin persistir CFDI) |

#### Infrastructure - Integraciones

| Integracion    | Componente              | Descripcion                                             |
| -------------- | ----------------------- | ------------------------------------------------------- |
| SAP (PAC)      | SapStampingClient       | Llamada HTTP unica (sin retry) al proveedor de timbrado para generar CFDI |
| IdentityServer | Autenticacion           | Validacion de tokens desde Finanzas                     |
| Serilog        | Logs                    | Contexto: usuario, modulo, operacion, CfdiGeneradaId (si Finanzas lo envia en el request) |
| ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo) | ApiCallerBitacoraCambios | Registro de auditoria de negocio de cada timbrado (exitoso o fallido) — Reglas al diseñar, regla 8 |

> **Minio se retira de Timbrado.** La subida del XML a Minio y su registro en `Archivo` (patron ya usado por RE-FU-016 y por `fccNotaCredito.IdArchivoXml`/`IdArchivoPdf`) ahora es responsabilidad de **Finanzas**, no de Timbrado — Timbrado solo regresa el contenido del XML en la respuesta tecnica (`StampingResponse.Xml`).
>
> RabbitMQ, Worker.Timbrado y Brevo se retiran del alcance de esta solucion: no hay reintentos ni notificaciones de error dentro de Timbrado. El reintento y la notificacion a soporte se implementan del lado de Finanzas, en cada punto de generacion del documento (Factura, Factura por Adelantado, Nota de Credito, Complemento de Pago — ver requisitos respectivos), y cualquier envio de correo (a soporte o al cliente) se realiza a traves de **ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo)**, no con un cliente SMTP/Brevo propio de Finanzas ni de Timbrado (Reglas al diseñar, regla 7). ProquifaDotNet.EnvioCorreo es una solucion independiente de ProquifaDotNet.SendInBlue (R16A-NO-FU-001), que solo migra el envio de correo del sistema legacy ProquifaDotNet/Venta Interna.

> **ProquifaDotNet.BitacoraCambios (regla 8):** Timbrado invoca a este Aplicativo Nuevo al registrar cada resultado de timbrado (exitoso o fallido) en `StampingLog`, igual que Finanzas debe hacerlo al guardar el CFDI, validar un cobro o guardar una proforma. El detalle de la integracion (contrato, endpoint) aun no esta documentado en un requisito propio; aqui solo se referencia el punto de integracion, no su detalle tecnico.

#### API - Endpoints (uso interno, consumidos solo por Finanzas)

> Servicio tecnico, no expone un recurso de negocio: las rutas usan una accion (`stamp`) en vez de un sustantivo CRUD, ya que no hay una entidad persistida detras. Alineado en lo demas a `api/v1/{resource}/{id}/{subresource}` (Reglas al diseñar — regla 9).

| Metodo | Endpoint                | Descripcion                                                |
| ------ | ------------------------ | ----------------------------------------------------------- |
| POST   | /api/v1/stamp             | Recibe datos fiscales armados por Finanzas, timbra ante SAP y regresa Uuid/XML/Series/Folio/estatus |
| POST   | /api/v1/stamp/cancel      | Solicita cancelacion de un CFDI ante SAP (recibe Uuid + datos minimos, sin leer tabla propia) |

---

### Base de Datos: ProquifaDotNetTimbrado (Nueva)

| Tabla | Proposito |
|-------|-----------|
| AppSetting | Configuracion del servicio (endpoints SAP, timeouts) |
| StampingLog | Auditoria tecnica de la peticion de timbrado (un registro por solicitud, con NewStatus Pending/Stamped/Failed) |

> Servidor: WIN-R14-DEV\DEV_R17_APPS
> BD creada con script DDL documentado en R16A-RE-FU-018_BD.md (Parte 2)

---

### Paquetes NuGet Requeridos (Timbrado)

| Proyecto | Paquete | Uso |
|----------|---------|-----|
| Infrastructure | Microsoft.EntityFrameworkCore.SqlServer | EF Core + SQL Server |
| Infrastructure | Serilog 4.x | Logs |
| Infrastructure | Polly 8.x | Timeout/circuit-breaker en la llamada HTTP a SAP (sin politica de retry) |
| Application | FluentValidation 11.x | Validaciones |
| API | Serilog.AspNetCore 9.x | Logs en API |
| API | Swashbuckle.AspNetCore 6.x | Swagger |

---

## Parte B - CfdiController y persistencia de CFDIGenerada en ProquifaDotNet.Finanzas

### Descripcion

El recurso de negocio **CFDI** (crear, consultar, cancelar, listar) vive en **ProquifaDotNet.Finanzas**, no en Timbrado. `CfdiController` orquesta: arma los datos fiscales, llama al servicio tecnico de Timbrado (`POST /api/v1/stamp`), y persiste el resultado en `CFDIGenerada` (ProquifaDotNet) + `Archivo` (XML en Minio).

### Componentes (Finanzas)

| Capa | Componente | Descripcion |
|------|-----------|-------------|
| Domain | Entidad `CfdiGenerada` (EF Core Scaffold de `CFDIGenerada`) | Registro central de negocio del CFDI |
| Application | `CfdiService` | Orquesta: arma request tecnico, llama `IApiCallerStamping`, persiste `CFDIGenerada` + `Archivo`, dispara Bitacora |
| Application | `IApiCallerStamping` | Cliente HTTP hacia ProquifaDotNet.Timbrado (`POST /api/v1/stamp`, `POST /api/v1/stamp/cancel`) |
| Infrastructure | `ApiCallerStamping` | Implementacion del cliente HTTP anterior |
| API | `CfdiController` | Expone el recurso de negocio `cfdi` al resto de Finanzas y a Venta Interna |

### API - Endpoints (recurso de negocio, expuesto)

> Ruta alineada a `api/v1/{resource}/{id}/{subresource}` (Reglas al diseñar — regla 9): recurso singular en ingles (`cfdi`), CRUD por metodo HTTP, acciones especiales (`cancel`, `xml`, `search`) como subrecurso.

| Metodo | Endpoint                      | Descripcion                                          |
| ------ | ----------------------------- | ----------------------------------------------------- |
| POST   | /api/v1/cfdi                  | Arma datos fiscales, llama a Timbrado, persiste CFDIGenerada + Archivo (XML) |
| POST   | /api/v1/cfdi/{id}/cancel      | Llama a Timbrado para cancelar ante SAP y actualiza Estado en CFDIGenerada |
| GET    | /api/v1/cfdi/{id}             | Consulta CFDIGenerada por Id |
| GET    | /api/v1/cfdi/{id}/xml         | Descarga XML desde Minio via Archivo.FileKey/FileBucket |
| POST   | /api/v1/cfdi/search           | Listado paginado con QueryInfo |

### Flujo de persistencia al timbrar exitosamente

```
1. CfdiController recibe la solicitud de negocio (ej. desde AdvanceInvoiceGenerateService, R16A-RE-FU-019)
2. CfdiService arma StampingRequestDto y llama IApiCallerStamping.StampAsync (-> Timbrado POST /api/v1/stamp)
3. Si Timbrado responde exito (Uuid, XML, Series, Folio, FechaEmision):
   - INSERT/UPDATE CFDIGenerada (IdCatTipoCFDI, RFCEmisor, RFCReceptor, Serie, Folio, FechaEmision, UUID,
     Total, IdCatUsoCFDI, IdCatMetodoDePagoCFDI, IdCatMoneda, TipoCambio, Estado='Timbrado')
   - INSERT Archivo (FileKey/FileBucket con el XML recibido) + UPDATE CFDIGenerada SET IdArchivoXml
   - Retorna CfdiResponseDto al llamador (ej. AdvanceInvoiceGenerateService) con el Id real de CFDIGenerada
4. Si Timbrado responde error:
   - UPDATE CFDIGenerada SET Estado='Fallido', MensajeError (si ya existia el registro Pendiente)
   - Retorna el error al llamador; el reintento es responsabilidad del flujo que invoco (ej. RE-FU-019/020)
```

> **Nomenclatura (regla 6):** clases/DTOs/metodos en ingles (`CfdiService`, `CfdiController`, `IApiCallerStamping`, `ApiCallerStamping`, `CfdiResponseDto`). `Cfdi`, `Rfc`, `Uuid` se mantienen como terminos fiscales estandar. Las columnas de `CFDIGenerada` (tabla existente/extendida en ProquifaDotNet) conservan su nomenclatura en español (regla 1) al ser mapeadas por el Scaffold — solo las clases/propiedades de codigo C# nuevas se traducen donde no reflejen 1:1 una columna de BD.

---

## Parte C - Modulo Factura por Adelantado en ProquifaDotNet.Finanzas (listado)

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

### Cadena de Datos (Consulta BD ProquifaDotNet) — migrada a `vfccFactura` (06/07/2026)

> **Migración:** la cadena de 5 saltos (`tpPedido → tpPedidoProformaPedido → tpProformaPedido → tpProformaAdelantoProformaPedido → fccPagoFacturaAdelanto → tpProformaAdelanto`) dejaba fuera del listado a los pedidos Prepago con FAA (RE-FU-015 no genera `tpProformaPedido`/`tpProformaAdelanto`) — hallazgo H-01 de `R16A-RE-FU-018_DIS-SOL_Revision.md`. Se reemplaza por lectura directa de `vfccFactura` (vista sobre `fccFactura`, definida en `R16A-RE-FU-015_BD.md`), que cubre ambos orígenes (Prepago y Crédito) con FK directa a `tpPedido`.

```
vfccFactura (FacturaPorAdelantado=1, Activo=1, RegionClave='MEX')

Filtro: vfccFactura.Activo=1 AND vfccFactura.EstadoFAA IN ('PendienteGenerar','PendienteEnviar')
        AND vfccFactura.RegionClave = 'MEX'  -- OBS-032/033: FAA solo aplica para Mexico; clientes Peru se excluyen del listado
        -- OBS-034: 'PendienteEnviar' incluye los casos donde el CFDI ya fue timbrado pero el usuario AÚN NO ha ejecutado "Enviar Factura"
        --          El envío de la factura es una acción EXPLÍCITA del usuario, NO automática post-timbrado.

Agrupacion: GROUP BY IdCliente
  COUNT(vfccFactura.IdFccFactura) AS FacturasPendientes
  SUM(MontoConvertidoUSD) AS MontoTotalUsd
  MIN(vfccFactura.FechaRegistro) AS FechaPendienteMasAntiguo

Filtro cartera: ClienteCarteraCliente -> ClienteCartera.IdUsuarioCobrador = @IdUsuario
Buscador: TRIM(vfccFactura.ClienteRazonSocial) LIKE / TRIM(vfccFactura.ClienteRFC) LIKE / TRIM(vfccFactura.FolioPedidoInterno) LIKE
          -- OBS-041: trim automatico aplicado al texto ingresado antes de ejecutar el filtrado
```

### Tablas consultadas (ProquifaDotNet - Lectura)

| Tabla | Rol |
|-------|-----|
| vfccFactura | Vista consolidada (reemplaza la cadena `tpProformaAdelanto`/`vtpProformaAdelanto`): MontoTotal, IdCFDIGenerada, Enviada, EstadoFAA, FechaRegistro, ClienteRazonSocial, ClienteRFC, FolioPedidoInterno, RegionClave |
| ClienteCarteraCliente | Vinculo cliente-cartera |
| ClienteCartera | IdUsuarioCobrador |
| catMoneda | Para conversion a USD |

> `tpPedido`, `tpProformaAdelanto`, `tpProformaPedido`, `tpPedidoProformaPedido`, `tpProformaAdelantoProformaPedido`, `fccPagoFacturaAdelanto`, `Cliente` y `DatosFacturacionCliente` **ya no se consultan directamente** en este listado — sus datos ya vienen resueltos en `vfccFactura`.

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

## Parte D - ProquifaDotNet (Venta Interna)

### Impacto

| #   | Componente        | Accion                                                         |
| --- | ----------------- | -------------------------------------------------------------- |
| 1   | ApiCallerFinanzas | Agregar metodo para llamar POST /api/v1/advanceInvoice/search |
| 2   | Controlador FAA   | Nuevo controlador en WebAPI que expone el listado al Front     |

> El modulo FAA es nuevo en Venta Interna. Se crea un controlador que delega a Finanzas.

---

## Gaps de Desarrollo

### En ProquifaDotNet.Timbrado (Solucion NUEVA, servicio tecnico)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Crear solucion y proyectos | sln + 5 csproj (Domain, Application, Infrastructure, API, Testing) | Medio |
| GAP-02 | Domain: Entities + Interfaces | StampingLog, AppSetting + interfaces | Bajo |
| GAP-03 | Application: DTOs + StampingService | StampingRequestDto, StampingResponseDto, StampingLogDto, Validators | Medio |
| GAP-04 | Infrastructure: StampingContext | EF Core con scaffold BD ProquifaDotNetTimbrado (2 tablas: AppSetting, StampingLog) | Bajo |
| GAP-05 | Infrastructure: SapStampingClient | Cliente HTTP para invocar PAC SAP (timbrar/cancelar CFDI), un solo intento por peticion, sin retry ni cola | Alto |
| GAP-06 | API: Program.cs + configuracion | DI, EF Core, Swagger, Serilog, IdentityServer, CORS | Medio |
| GAP-07 | API: StampingController | Endpoints POST /api/v1/stamp, POST /api/v1/stamp/cancel (uso interno, consumido solo por Finanzas) | Medio |
| GAP-08 | BD: Ejecutar scripts DDL | CREATE DATABASE + 2 tablas (AppSetting, StampingLog) | Bajo |

> Se retiraron los gaps de Worker.Timbrado/RabbitMQ (reintentos), de notificacion directa via Brevo, y de persistencia/Minio del CFDI (`Cfdi`, `FiscalDocumentType`, `CfdiRepository`, `MinioStorageService`): esa responsabilidad pasa a **ProquifaDotNet.Finanzas** (ver Gaps de Finanzas — CfdiController). El envio de correo se canaliza a traves de ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo, regla 7).

### En ProquifaDotNet.Finanzas (CfdiController + CFDIGenerada)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-09 | Infrastructure: Scaffold CFDIGenerada extendida | Agregar/actualizar entidad `CfdiGenerada` en el Scaffold de Finanzas tras el ALTER TABLE (R16A-RE-FU-018_BD.md Parte 3) | Bajo |
| GAP-10 | Application: CfdiService | Arma request tecnico, llama IApiCallerStamping, persiste CFDIGenerada + Archivo, dispara Bitacora | Alto |
| GAP-11 | Application/Infrastructure: IApiCallerStamping / ApiCallerStamping | Cliente HTTP hacia ProquifaDotNet.Timbrado (POST /api/v1/stamp, /cancel) | Medio |
| GAP-12 | Infrastructure: MinioStorageService (Finanzas) | Upload XML a Minio + INSERT Archivo (reutiliza patron de RE-FU-016 si ya existe) | Bajo |
| GAP-13 | API: CfdiController | Endpoints POST /api/v1/cfdi, POST /api/v1/cfdi/{id}/cancel, GET /api/v1/cfdi/{id}, GET /api/v1/cfdi/{id}/xml, POST /api/v1/cfdi/search | Medio |

### En ProquifaDotNet.Finanzas (Modulo FAA — listado)

> **Nomenclatura (regla 6):** nombres de clase/DTO en ingles (solucion nueva). **Ruta (regla 9):** recurso `advanceInvoice`.

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-14 | Domain: DTO AdvanceInvoiceClientDto | Modelo del listado agrupado | Bajo |
| GAP-15 | Infrastructure: AdvanceInvoiceRepository | Query compleja agrupada por cliente con JOINs a 12+ tablas | Alto |
| GAP-16 | Application: AdvanceInvoiceService | Listado con QueryInfo, filtro cartera, buscador, conversion USD | Alto |
| GAP-17 | API: AdvanceInvoiceController | Endpoint POST /api/v1/advanceInvoice/search | Bajo |
| GAP-18 | Infrastructure: FinanzasContext ampliacion | Agregar DbSet de solo lectura para `vfccFactura` (antes: `tpProformaAdelanto`, etc.) | Medio |

### En ProquifaDotNet (Venta Interna)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-19 | ApiCallerFinanzas: metodo SearchAdvanceInvoices | Llamada POST /api/v1/advanceInvoice/search | Bajo |
| GAP-20 | Controlador FAA nuevo | Expone listado al Front, delega a Finanzas | Bajo |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-016 | Prerequisito: ProquifaDotNet.Finanzas ya creada (estructura, DI, API base, patron persistencia Minio/Archivo) |
| R16A-RE-FU-019 | CREATE TABLE CFDIGenerada (ER-Finanzas.md) — prerequisito para el ALTER TABLE de este requisito |
| R16A-RE-FU-028 | catTipoCFDI (catalogo de tipo de CFDI usado por CfdiController) |
| R16A-RE-FU-005 | Brecha timbrado SUNAT Peru (OSE) - bloquea flujo FAA para Peru |

---

## Resumen de Gaps

| Repositorio | Cantidad | Detalle |
|-------------|----------|---------|
| ProquifaDotNet.Timbrado (NUEVA, servicio tecnico) | 8 (GAP-01 a GAP-08) | Creacion de solucion + BD (solo AppSetting/StampingLog) |
| ProquifaDotNet.Finanzas — CfdiController | 5 (GAP-09 a GAP-13) | CFDIGenerada extendida + CfdiController + Minio/Archivo |
| ProquifaDotNet.Finanzas — Modulo FAA (listado) | 5 (GAP-14 a GAP-18) | Modulo FAA listado agrupado |
| ProquifaDotNet | 2 (GAP-19 a GAP-20) | Consumidor del listado |
| **Total** | **20 gaps** | |
