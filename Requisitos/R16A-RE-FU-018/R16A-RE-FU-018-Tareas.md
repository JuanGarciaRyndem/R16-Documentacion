# Tareas BackEnd - R16A-RE-FU-018
**Requisito:** Factura por Adelantado (Pantalla Inicial)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10, NUEVA)

---

## Parte A - Creacion de Solucion ProquifaDotNet.Timbrado

---

### Tarea 1

**Titulo:** [ R16A-RE-FU-018 ] [CONFIG-INIT] Crear solucion ProquifaDotNet.Timbrado con estructura Clean Architecture

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Solucion completa

**Consideraciones previas:**
- Tomar como referencia la arquitectura del repositorio proquifa-punchout-backend (.NET 10, Clean Architecture)
- La solucion gestiona timbrado fiscal: recepcion solicitudes, integracion SAP, generacion CFDI, almacenamiento XML
- Requiere .NET 10 como target framework
- BD independiente: ProquifaDotNetTimbrado

**Objetivo general:**
Crear la solucion ProquifaDotNet.Timbrado con la estructura base de proyectos siguiendo Clean Architecture.

**Objetivos especificos:**
- Crear archivo Proquifa.Timbrado.sln
- Crear proyecto Proquifa.Timbrado.Domain.csproj (net10.0)
- Crear proyecto Proquifa.Timbrado.Application.csproj (net10.0)
- Crear proyecto Proquifa.Timbrado.Infrastructure.csproj (net10.0)
- Crear proyecto Proquifa.Timbrado.API.csproj (net10.0)
- Crear proyecto Proquifa.Timbrado.Testing.csproj (net10.0)
- Configurar referencias entre proyectos (API -> Application -> Domain, Infrastructure -> Domain)

> Nota: Timbrado no incluye proyecto Worker. Es un servicio sincrono de un solo intento por peticion; no hay cola de reintentos propia (esa responsabilidad es de Finanzas, implementada localmente en cada punto de generacion: Factura, Factura por Adelantado, Nota de Credito, Complemento de Pago).

**Resultado esperado:**
Solucion compilable con 5 proyectos vacios conectados correctamente segun Clean Architecture.

**Entregables:**
- Archivo .sln con 5 proyectos referenciados
- Cada .csproj con target net10.0
- Estructura de carpetas base creada

**Criterios de aceptacion:**
- La solucion compila sin errores
- Las referencias entre capas respetan la arquitectura (Domain sin dependencias, Application depende de Domain, Infrastructure depende de Domain y Application, API depende de todos)

**Mas informacion de la tarea:**
Referencia: repositorio proquifa-punchout-backend. Corresponde a GAP-01.

**Recursos:**
- R16A-RE-FU-018-Back.md (Parte A - Estructura de la Solucion)
- Soluciones Nuevas.md

---

### Tarea 2

**Titulo:** [ R16A-RE-FU-018 ] [ARQ-PROJ-NET] Crear capa Domain - Entities, Interfaces y Models

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Domain

**Consideraciones previas:**
- Basado en el patron de proquifa-punchout-backend (QueryInfo, FilterItem, SortDirection)
- Entidades corresponden a tablas de BD ProquifaDotNetTimbrado: Cfdi, StampingLog, FiscalDocumentType, AppSetting
- Interfaces definen contratos para: repositorios, SAP client, Minio, UnitOfWork
- **Nomenclatura (Reglas al diseñar — regla 6):** ProquifaDotNet.Timbrado es solucion nueva, por lo que entidades, propiedades, DTOs, interfaces y metodos se codifican en ingles, sin mezclar palabras en español. `Cfdi`, `Rfc` y `Uuid` se mantienen como terminos fiscales estandar (no se traducen). La palabra "Timbrado" se traduce a "Stamping" dentro del codigo (StampingLog, StampingService, SapStampingClient, StampingRequest/Response, StampingContext); solo el nombre de la solucion/proyecto (ProquifaDotNet.Timbrado, Proquifa.Timbrado.*.csproj/.sln) y de la BD (ProquifaDotNetTimbrado) se mantienen sin traducir por ser nomenclatura ya establecida

**Objetivo general:**
Implementar la capa Domain con entidades, interfaces de servicio y modelos del dominio de timbrado.

**Objetivos especificos:**
- Crear Common/QueryInfo.cs, Common/FilterItem.cs, Common/SortDirection.cs
- Crear Entities/Cfdi.cs con campos: Id, FiscalDocumentTypeId, Uuid, Series, Folio, IssueDate, IssuerRfc, ReceiverRfc, Total, Currency, ExchangeRate, PaymentMethod, PaymentForm, CfdiUse, XmlContent, MinioFileKey, MinioBucket, Status, ErrorMessage, etc.
- Crear Entities/StampingLog.cs con campos: Id, CfdiId, Action, PreviousStatus, NewStatus, Request, Response, ErrorMessage, DurationMs, etc.
- Crear Entities/FiscalDocumentType.cs con campos: Id, Code, Description
- Crear Entities/AppSetting.cs con campos: Id, Name, Value, Description
- Crear Interfaces/IGenericRepository.cs, ICfdiRepository.cs, IStampingLogRepository.cs
- Crear Interfaces/ISapStampingClient.cs, IMinioStorageService.cs, IUnitOfWork.cs
- Crear Models/StampingRequest.cs, StampingResponse.cs

**Resultado esperado:**
Capa Domain completa con todas las abstracciones necesarias para el servicio de timbrado.

**Entregables:**
- Archivos .cs en carpetas Common/, Entities/, Interfaces/, Models/
- Proyecto compilable sin dependencias externas

**Criterios de aceptacion:**
- El proyecto Domain no tiene referencias a otros proyectos de la solucion
- Las entidades reflejan exactamente la estructura de BD (R16A-RE-FU-018_BD.md)
- ISapStampingClient define metodos StampAsync y CancelAsync

**Mas informacion de la tarea:**
Corresponde a GAP-02.

**Recursos:**
- R16A-RE-FU-018-Back.md
- R16A-RE-FU-018_BD.md

---

### Tarea 3

**Titulo:** [ R16A-RE-FU-018 ] [ARQ-PROJ-NET] Crear capa Application - DTOs, Services y Validators

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Application

**Consideraciones previas:**
- Depende de Tarea 2 (Domain)
- StampingService orquesta: validar request, crear CFDI, llamar SAP, almacenar XML, registrar log
- CfdiService: CRUD + listado paginado con QueryInfo
- FluentValidation para validaciones de request

**Objetivo general:**
Implementar la capa Application con DTOs, servicios de orquestacion y validadores.

**Objetivos especificos:**
- Crear DTOs/QueryResultDto.cs (wrapper generico paginado)
- Crear DTOs/CfdiDto.cs, StampingRequestDto.cs, StampingResponseDto.cs, StampingLogDto.cs
- Crear Interfaces/IStampingService.cs (StampAsync, CancelAsync, GetStatusAsync)
- Crear Interfaces/ICfdiService.cs (CRUD + SearchAsync)
- Crear Services/StampingService.cs (orquestacion completa del flujo de timbrado)
- Crear Services/CfdiService.cs (operaciones CRUD sobre Cfdi)
- Crear Mappers/ApplicationMappingProfile.cs (AutoMapper)
- Crear Validators/StampingRequestDtoValidator.cs (FluentValidation)

**Resultado esperado:**
Capa Application completa con logica de orquestacion de timbrado.

**Entregables:**
- StampingService con flujo: validar -> crear Cfdi -> llamar SAP -> almacenar -> log -> retornar
- CfdiService con CRUD + listado paginado
- Validador de request de timbrado

**Criterios de aceptacion:**
- Application solo depende de Domain
- StampingService implementa flujo completo de timbrado (sincrono)
- El validador rechaza requests sin campos obligatorios (IssuerRfc, ReceiverRfc, Total, Currency, FiscalDocumentTypeId)

**Mas informacion de la tarea:**
Corresponde a GAP-03.

**Recursos:**
- R16A-RE-FU-018-Back.md

---

### Tarea 4

**Titulo:** [ R16A-RE-FU-018 ] [SIMPLE-CRUD] Crear StampingContext con DbSet y mapping de entidades

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Infrastructure/Persistence/Context

**Consideraciones previas:**
- Depende de Tarea 2 (Domain - entidades definidas)
- BD: ProquifaDotNetTimbrado (servidor WIN-R14-DEV\DEV_R17_APPS)
- 4 tablas: AppSetting, FiscalDocumentType, Cfdi, StampingLog
- Cfdi tiene FK a FiscalDocumentType; StampingLog tiene FK a Cfdi

**Objetivo general:**
Crear el StampingContext de EF Core con los DbSets y configuracion de mapping para las entidades de timbrado.

**Objetivos especificos:**
- Crear Persistence/Context/StampingContext.cs
- Configurar DbSet para Cfdi, StampingLog, FiscalDocumentType, AppSetting
- Configurar relaciones FK: Cfdi -> FiscalDocumentType, StampingLog -> Cfdi
- Configurar propiedades con tipos SQL correctos (uniqueidentifier, varchar(max), decimal(18,2), datetime2)
- Configurar connection string desde appsettings.json

**Resultado esperado:**
StampingContext funcional conectado a BD ProquifaDotNetTimbrado.

**Entregables:**
- StampingContext.cs con OnModelCreating completo
- Configuracion de entidades con Fluent API
- Registro del contexto en DI

**Criterios de aceptacion:**
- El contexto mapea correctamente las 4 tablas
- Las FKs se configuran correctamente
- Cfdi.Status tiene default 'Pending'
- El contexto se inyecta correctamente via DI

**Mas informacion de la tarea:**
Corresponde a GAP-04.

**Recursos:**
- R16A-RE-FU-018-Back.md
- R16A-RE-FU-018_BD.md (DDL completo)

---

### Tarea 5

**Titulo:** [ R16A-RE-FU-018 ] [IMPL-THIRD-SERV] Implementar SapStampingClient para integracion con PAC SAP

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Infrastructure/Services

**Consideraciones previas:**
- Depende de Tarea 2 (ISapStampingClient definido en Domain)
- SAP es el proveedor de timbrado fiscal (PAC)
- Operaciones: Stamp (enviar documento, recibir Cfdi con Uuid/XML) y Cancel
- Un solo intento por peticion: Timbrado no reintenta ante fallo del PAC (esa politica es responsabilidad de Finanzas, implementada localmente en cada punto de generacion del documento). Polly se usa unicamente para timeout, no para retry
- Configuracion via SapSettings (endpoint, credenciales, timeout)

**Objetivo general:**
Implementar el cliente HTTP para invocar al proveedor SAP y generar/cancelar documentos fiscales CFDI.

**Objetivos especificos:**
- Crear Services/SapStampingClient.cs implementando ISapStampingClient
- Implementar metodo StampAsync(StampingRequest) -> StampingResponse (Uuid, XML, Series, Folio)
- Implementar metodo CancelAsync(string uuid) -> bool
- Configurar HttpClient con Polly solo para timeout (sin politica de retry: 1 intento por peticion)
- Crear Configuration/SapSettings.cs (Endpoint, ApiKey, Timeout)
- Manejar errores de SAP: timeout, respuesta invalida, rechazo fiscal
- Registrar en DI con HttpClientFactory

**Resultado esperado:**
Cliente HTTP funcional que invoca SAP una sola vez por peticion para timbrar y cancelar CFDI.

**Entregables:**
- SapStampingClient.cs con metodos StampAsync y CancelAsync
- SapSettings.cs con configuracion tipada
- Manejo de errores con Polly (timeout unicamente, sin retry)
- Registro en DI

**Criterios de aceptacion:**
- StampAsync envia request a SAP y retorna StampingResponse con Uuid y XML
- CancelAsync envia Uuid a SAP y retorna confirmacion
- Si SAP falla o no responde, se propaga el error de inmediato sin reintentar internamente (1 intento = 1 peticion HTTP)
- Timeout configurable via SapSettings

**Mas informacion de la tarea:**
Corresponde a GAP-05.

**Recursos:**
- R16A-RE-FU-018-Back.md
- Soluciones Nuevas.md (Flujo funcional Timbrado)

---

### Tarea 6

**Titulo:** [ R16A-RE-FU-018 ] [IMP-EXIST-SERVICE] Implementar MinioStorageService

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Infrastructure/Services

**Consideraciones previas:**
- Depende de Tarea 4 (Infrastructure base)
- Minio: almacenamiento de XML timbrados (bucket 'timbrado')
- Patron basado en proquifa-punchout-backend
- No aplica RabbitMQ ni Brevo en Timbrado: no hay cola de reintentos ni notificaciones de error criticas propias del servicio (responsabilidad de Finanzas, implementada localmente en cada punto de generacion: Factura, Factura por Adelantado, Nota de Credito, Complemento de Pago)

**Objetivo general:**
Implementar el servicio de infraestructura para Minio del servicio de timbrado.

**Objetivos especificos:**
- Crear Services/MinioStorageService.cs (UploadAsync XML, DownloadAsync XML)
- Crear Configuration/MinioSettings.cs

**Resultado esperado:**
Servicio de infraestructura funcional para el almacenamiento de XML del servicio de timbrado.

**Entregables:**
- MinioStorageService con Upload/Download de XML (bucket 'timbrado')
- Settings tipados para la integracion

**Criterios de aceptacion:**
- MinioStorageService sube y descarga XML correctamente
- El servicio se registra en DI via InfrastructureServiceExtensions

**Mas informacion de la tarea:**
Corresponde a GAP-06.

**Recursos:**
- R16A-RE-FU-018-Back.md

---

### Tarea 7

**Titulo:** [ R16A-RE-FU-018 ] [CONFIG-INIT] Crear capa API - Program.cs, Controllers y configuracion

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** API

**Consideraciones previas:**
- Depende de Tareas 3, 4, 5 y 6
- Integracion con IdentityServer para autenticacion (llamadas desde Finanzas)
- Serilog para logging con contexto
- Swagger para documentacion de endpoints
- **Rutas (Reglas al diseñar — regla 9):** un unico recurso `cfdi` en ingles bajo `api/v1/`, CRUD por metodo HTTP, acciones especiales (`cancel`, `xml`, `search`) como subrecurso — un solo CfdiController, sin controller separado "Timbrado" para las acciones de escritura

**Objetivo general:**
Configurar la capa API con Program.cs, el controlador de CFDI y toda la configuracion necesaria.

**Objetivos especificos:**
- Crear Program.cs con configuracion (DI, EF Core, Swagger, Serilog, IdentityServer)
- Crear Controllers/CfdiController.cs con endpoints: POST /api/v1/cfdi (timbrar), POST /api/v1/cfdi/{id}/cancel, GET /api/v1/cfdi/{id}, GET /api/v1/cfdi/{id}/xml, POST /api/v1/cfdi/search
- Crear appsettings.json con connection strings (ProquifaDotNetTimbrado, SAP, Minio, IdentityServer)
- Configurar CORS para comunicacion con ProquifaDotNet.Finanzas

**Resultado esperado:**
API ejecutable con Swagger funcional y endpoints de timbrado accesibles.

**Entregables:**
- Program.cs con toda la configuracion de servicios
- CfdiController con 5 endpoints (stamp, cancel, getById, getXml, search)
- appsettings.json con placeholders
- API ejecutable via Swagger

**Criterios de aceptacion:**
- La API inicia sin errores
- Swagger muestra todos los endpoints documentados
- IdentityServer valida tokens de autenticacion desde Finanzas
- POST /api/v1/cfdi recibe StampingRequestDto y retorna StampingResponseDto
- GET /api/v1/cfdi/{id}/xml retorna XML desde Minio

**Mas informacion de la tarea:**
Corresponde a GAP-07, GAP-08 y GAP-09.

**Recursos:**
- R16A-RE-FU-018-Back.md

---

> **Tarea eliminada (ex Tarea 8 — Worker.Timbrado para reintentos via RabbitMQ):** Se retira del alcance. Timbrado es un servicio sincrono de un solo intento por peticion; no tiene Worker ni cola de reintentos propia. La politica de reintento ante fallo es responsabilidad de Finanzas, implementada de forma local en cada punto de generacion del documento (Factura: R16A-RE-FU-028/029; Factura por Adelantado: R16A-RE-FU-019/020; Nota de Credito: R16A-RE-FU-032/033/034/035; Complemento de Pago: R16A-RE-FU-030). Ver `Diagramas/Diagrama Secuencia Finanzas y Timbrado Factura.md` y `Diagramas/Diagrama Secuencia Encolamiento Finanzas y Timbrado Factura.md`.

---

### Tarea 8

**Titulo:** [ R16A-RE-FU-018 ] [CREATE-SCRIPT-CONTROL] Ejecutar scripts DDL para BD ProquifaDotNetTimbrado

**Aplicativos:** ProquifaDotNet.Timbrado (Base de Datos)

**Modulos:** Base de Datos - Scripts DDL

**Consideraciones previas:**
- Servidor: Mismo que PQF en el que se Ambiente
- Compatibility: 160 (SQL Server 2022)
- 4 tablas: AppSetting, FiscalDocumentType, Cfdi, StampingLog
- DML inicial: 4 registros en FiscalDocumentType
- **Nomenclatura (Reglas al diseñar — regla 2):** ProquifaDotNetTimbrado es BD nueva, por lo que tablas y columnas se nombran en ingles

**Objetivo general:**
Ejecutar los scripts DDL para crear la base de datos ProquifaDotNetTimbrado con sus 4 tablas y datos iniciales.

**Objetivos especificos:**
- Ejecutar CREATE DATABASE ProquifaDotNetTimbrado con rutas de archivos
- Ejecutar CREATE TABLE AppSetting
- Ejecutar CREATE TABLE FiscalDocumentType
- Ejecutar CREATE TABLE Cfdi (con FK a FiscalDocumentType)
- Ejecutar CREATE TABLE StampingLog (con FK a Cfdi)
- Ejecutar INSERT datos iniciales FiscalDocumentType (4 registros: AdvanceInvoice, RegularInvoice, AnticipatedInvoice, CreditNote)

**Resultado esperado:**
BD ProquifaDotNetTimbrado creada y operativa con tablas y datos iniciales.

**Entregables:**
- Script DDL ejecutado exitosamente
- BD accesible desde EF Core (StampingContext)
- 4 registros en FiscalDocumentType

**Criterios de aceptacion:**
- La BD existe en el servidor indicado
- Las 4 tablas se crean correctamente con PKs, FKs y defaults
- FiscalDocumentType tiene 4 registros activos
- EF Core (StampingContext) puede conectar y consultar

**Mas informacion de la tarea:**
Corresponde a GAP-10.

**Recursos:**
- R16A-RE-FU-018_BD.md (Script completo DDL)

---

## Parte B - Modulo Factura por Adelantado en ProquifaDotNet.Finanzas

---

### Tarea 9

**Titulo:** [ R16A-RE-FU-018 ] [ALG-COMPLX-LOGIC] Crear AdvanceInvoiceRepository con query agrupada por cliente

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Infrastructure/Persistence/Repository

**Consideraciones previas:**
- Depende de RE-FU-016 (Finanzas ya creada)
- Query compleja con JOINs a 12+ tablas de BD ProquifaDotNet (lectura)
- Agrupacion GROUP BY Cliente con: COUNT, SUM (conversion USD), MIN (antiguedad)
- Filtro por cartera: ClienteCarteraCliente -> ClienteCartera.IdUsuarioCobrador
- Buscador por: RazonSocial LIKE, RFC LIKE, FolioPedidoInterno LIKE
- **OBS-041:** El sistema aplica trim automático al texto del buscador antes de ejecutar el filtrado (ignorar espacios al inicio y al final)
- **OBS-032/033:** El listado incluye únicamente pedidos de región México (`tpPedido.IdRegion = RegionMex`). Los clientes Perú se excluyen del query para evitar huérfanos/ruido en el módulo FAA
- **OBS-037:** Si excepcionalmente existiera un pedido con FAA=1 que incluya productos controlados (no debería ocurrir por validación en RE-FU-012/011), el repositorio lo excluye del listado
- **Nomenclatura (Reglas al diseñar — regla 6):** clase de repositorio en ingles (ProquifaDotNet.Finanzas es solucion nueva); las tablas/columnas de BD ProquifaDotNet consultadas en modo lectura conservan su nomenclatura en español (regla 1)

**Objetivo general:**
Implementar el repositorio que consulta los pendientes de Factura por Adelantado agrupados por cliente con paginacion y filtros.

**Objetivos especificos:**
- Crear AdvanceInvoiceRepository.cs con metodo SearchPendingByClientAsync(QueryInfo)
- Implementar JOINs: tpPedido -> tpPedidoProformaPedido -> tpProformaPedido -> tpProformaAdelantoProformaPedido -> fccPagoFacturaAdelanto -> tpProformaAdelanto
- Filtrar: FacturaPorAdelantado=1, Tramitado=1, Activo=1, IdCFDIGenerada IS NULL o no enviada
- Filtrar por cartera: ClienteCarteraCliente.IdClienteCartera -> ClienteCartera.IdUsuarioCobrador
- Agrupar por Cliente: COUNT(pendientes), SUM(monto convertido USD), MIN(FechaRegistro)
- Implementar buscador por RazonSocial, RFC/RUC, FolioPedidoInterno
- Soportar paginacion con QueryInfo

**Resultado esperado:**
Repositorio funcional que retorna listado agrupado por cliente con pendientes FAA.

**Entregables:**
- AdvanceInvoiceRepository.cs con query compleja
- Soporte QueryInfo (filtros, ordenamiento, paginacion)
- Conversion de montos a USD

**Criterios de aceptacion:**
- Retorna solo clientes con pendientes FAA en la cartera del usuario
- El conteo incluye pedidos sin factura y con factura no enviada
- El monto se convierte a USD correctamente
- Se ordena por antiguedad del pendiente mas antiguo (ASC)
- El buscador filtra por RazonSocial, RFC/RUC o FolioPedidoInterno
- La paginacion funciona correctamente
- El query filtra exclusivamente pedidos de region Mexico (OBS-032/033): clientes Peru no aparecen en el listado
- El buscador aplica trim automatico al texto ingresado antes de ejecutar el filtrado (OBS-041)
- Pedidos con productos controlados y FAA=1 (caso excepcional) son excluidos del listado (OBS-037)

**Mas informacion de la tarea:**
Corresponde a GAP-12.

**Recursos:**
- R16A-RE-FU-018-Back.md (Cadena de Datos)
- R16A-RE-FU-018_BD.md (Parte 1 - Lectura)

---

### Tarea 10

**Titulo:** [ R16A-RE-FU-018 ] [LIST-PAG-MULT-FILTER] Crear AdvanceInvoiceService y Controller con endpoint listado

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Application/Services, API/Controllers

**Consideraciones previas:**
- Depende de Tarea 9 (AdvanceInvoiceRepository)
- El servicio delega al repositorio y aplica logica de negocio
- El controller expone POST /api/v1/advanceInvoice/search (Reglas al diseñar — regla 9)
- Response: QueryResultDto con AdvanceInvoiceClientDto
- **Nomenclatura (regla 6):** DTO, servicio y controller en ingles (solucion nueva)

**Objetivo general:**
Implementar el servicio y controller para el listado de pendientes FAA agrupados por cliente y por usuario utilizando la Cartera del Cliente, incluyendo la region.

**Objetivos especificos:**
- Crear DTOs/AdvanceInvoiceClientDto.cs (CustomerId, CompanyName, TaxId, RegionCode, PendingInvoices, TotalAmountUsd, OldestPendingDate)
- Crear Services/AdvanceInvoiceService.cs con metodo SearchPendingAsync(QueryInfo)
- Crear Controllers/AdvanceInvoiceController.cs con endpoint POST /api/v1/advanceInvoice/search
- Inyectar el filtro collectorUserId (QueryInfo) desde token IdentityServer (claims del usuario logueado) — mapeado internamente a ClienteCartera.IdUsuarioCobrador al consultar la BD ProquifaDotNet
- Retornar QueryResultDto con resultados paginados

**Resultado esperado:**
Endpoint funcional de listado FAA paginado con filtros.

**Entregables:**
- AdvanceInvoiceClientDto.cs
- AdvanceInvoiceService.cs
- AdvanceInvoiceController.cs con endpoint POST /search
- Registro en DI

**Criterios de aceptacion:**
- POST /api/v1/advanceInvoice/search acepta QueryInfo
- Retorna QueryResultDto con TotalResults y Results paginados
- El filtro de cartera se aplica automaticamente desde el token del usuario
- El buscador funciona con coincidencia parcial
- El endpoint aparece documentado en Swagger
- Al acceder al modulo FAA en Venta Interna, se muestra el listado agrupado por cliente con pendientes.
- **OBS-034:** El listado incluye pendientes donde el CFDI ya fue timbrado pero el usuario aún no ha ejecutado "Enviar Factura". El envío es una acción explícita posterior al timbrado, realizado a traves del **Aplicativo para Envio de Correo (Aplicativo Nuevo)** (regla 7) — **no se envía automáticamente**. El estado "timbrada — pendiente de envío" debe ser visible en el listado.

**Mas informacion de la tarea:**
Corresponde a GAP-11, GAP-13 y GAP-14.

**Recursos:**
- R16A-RE-FU-018-Back.md (Parte B)

---

### Tarea 11

**Titulo:** [ R16A-RE-FU-018 ] [SIMPLE-CRUD] Ampliar FinanzasContext con DbSets para tablas FAA

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Infrastructure/Persistence/Context

**Consideraciones previas:**
- Depende de RE-FU-016 Tarea 7 (FinanzasContext ya creado)
- Se deben agregar DbSets para tablas del flujo FAA: tpProformaAdelanto, tpProformaAdelantoProformaPedido, fccPagoFacturaAdelanto, ClienteCartera, ClienteCarteraCliente
- Lectura solamente (no CRUD sobre estas tablas)

**Objetivo general:**
Ampliar el FinanzasContext con los DbSets necesarios para las consultas del modulo Factura por Adelantado.

**Objetivos especificos:**
- Agregar DbSet para tpProformaAdelanto (con mapping HasKey, propiedades)
- Agregar DbSet para tpProformaAdelantoProformaPedido
- Agregar DbSet para fccPagoFacturaAdelanto
- Agregar DbSet para ClienteCartera
- Agregar DbSet para ClienteCarteraCliente
- Configurar mapeos con Fluent API (nombres de tabla, tipos)

**Resultado esperado:**
FinanzasContext ampliado con soporte para consultas del modulo FAA.

**Entregables:**
- Nuevos DbSets en FinanzasContext
- Entidades nuevas en Domain/Entities/ (si no existen)
- Configuracion Fluent API para las tablas FAA

**Criterios de aceptacion:**
- Las nuevas entidades se consultan correctamente via EF Core
- No se afectan los DbSets existentes (Proforma, etc.)
- Las relaciones FK se configuran correctamente

**Mas informacion de la tarea:**
Corresponde a GAP-15.

**Recursos:**
- R16A-RE-FU-018-Back.md
- R16A-RE-FU-018_BD.md (Parte 1 - tablas consultadas)

---

## Parte C - ProquifaDotNet (Venta Interna)

---

### Tarea 12

**Titulo:** [ R16A-RE-FU-018 ] [IMP-EXIST-SERVICE] Crear Servicio para consultar FAA en ProquifaDotNet con llamada a API Finanzas

**Aplicativos:** ProquifaDotNet (.NET Framework 4.8)

**Modulos:** Logic.Pqf.Catalogos/ApiCaller

**Consideraciones previas:**
- Depende de Tarea 10 (endpoint Finanzas disponible)
- Modulo FAA es nuevo en Venta Interna (no existe controlador)
- Usa ApiCallerFinanzas existente (creado en RE-FU-016) para llamar a Finanzas

**Objetivo general:**
Crear el servicio para consulta de Factura por Adelantado en ProquifaDotNet para su uso con Pedido

**Objetivos específicos:**
- Agregar metodo ListarPendientesFAA en ApiCallerFinanzas (invoca POST /api/v1/advanceInvoice/search)
- Pasar QueryInfo (buscador, paginacion) desde el Front hacia Finanzas

**Resultado esperado:**
Acceder a la API de Finanzas para obtener los FAA y los pueda procesar/leer Venta Interna.

**Entregables:**
- Método en ApiCallerFinanzas para acceder a FAA
- Manejo de errores y logs

**Criterios de aceptación:**
- El endpoint en Venta Interna retorna correctamente el listado FAA desde Finanzas
- El buscador y la paginacion funcionan end-to-end
- Si Finanzas no responde, se muestra error controlado
- El filtro de cartera se aplica correctamente (usuario logueado)

**Mas informacion de la tarea:**
Corresponde a GAP-16 y GAP-17.

**Recursos:**
- R16A-RE-FU-018-Back.md (Parte C)
- Logic.Pqf.Catalogos/ApiCaller/ApiCallerFinanzas.cs (referencia)

---

## Resumen de Tareas

| # | Clave | Titulo | Aplicativo | Predecesora |
|---|-------|--------|-----------|-------------|
| 1 | CONFIG-INIT | Crear solucion ProquifaDotNet.Timbrado | Timbrado | - |
| 2 | ARQ-PROJ-NET | Crear capa Domain (Entities, Interfaces, Models) | Timbrado | 1 |
| 3 | ARQ-PROJ-NET | Crear capa Application (DTOs, Services, Validators) | Timbrado | 2 |
| 4 | SIMPLE-CRUD | StampingContext con DbSet y mapping | Timbrado | 2 |
| 5 | IMPL-THIRD-SERV | SapStampingClient (integracion PAC SAP) | Timbrado | 2 |
| 6 | IMP-EXIST-SERVICE | MinioStorageService | Timbrado | 4 |
| 7 | CONFIG-INIT | Crear capa API (Program.cs, Controllers) | Timbrado | 3, 4, 5, 6 |
| 8 | CREATE-SCRIPT-CONTROL | Scripts DDL BD ProquifaDotNetTimbrado | BD | - |
| 9 | ALG-COMPLX-LOGIC | AdvanceInvoiceRepository (query agrupada 12+ tablas) | Finanzas | RE-FU-016 |
| 10 | LIST-PAG-MULT-FILTER | AdvanceInvoiceService + Controller listado | Finanzas | 9, 11 |
| 11 | SIMPLE-CRUD | Ampliar FinanzasContext con DbSets FAA | Finanzas | RE-FU-016 T7 |
| 12 | IMP-EXIST-SERVICE | Controlador FAA en ProquifaDotNet + ApiCallerFinanzas | ProquifaDotNet | 10 |

> Se elimino la tarea Worker.Timbrado/RabbitMQ (ex Tarea 8): sin reintentos en Timbrado. RabbitMQ y Brevo salen de esta solucion.

---

## Dependencias con otros requisitos

| Componente requerido | Origen | Usado por |
|---------------------|--------|-----------|
| ProquifaDotNet.Finanzas (solucion completa) | RE-FU-016 | Tareas 9, 10, 11 |
| FinanzasContext base | RE-FU-016 Tarea 7 | Tarea 11 |
| ApiCallerFinanzas | RE-FU-016 Tarea 15 | Tarea 12 |
| BD ProquifaDotNet (tablas FAA) | Existente | Tarea 9 |
| Politica de reintento ante fallo del PAC | Finanzas, local por punto de generacion: RE-FU-019/020 (FAA), RE-FU-028/029 (Factura), RE-FU-030 (CDP), RE-FU-032..035 (NC) | Todo el flujo de timbrado (Parte A) |
