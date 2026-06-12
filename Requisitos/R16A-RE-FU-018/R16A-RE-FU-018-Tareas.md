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
- Crear proyecto Proquifa.Timbrado.Worker.csproj (net10.0)
- Crear proyecto Proquifa.Timbrado.Testing.csproj (net10.0)
- Configurar referencias entre proyectos (API -> Application -> Domain, Infrastructure -> Domain)

**Resultado esperado:**
Solucion compilable con 6 proyectos vacios conectados correctamente segun Clean Architecture.

**Entregables:**
- Archivo .sln con 6 proyectos referenciados
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
- Entidades corresponden a tablas de BD ProquifaDotNetTimbrado: CFDI, TimbradoLog, TipoDocumentoFiscal, AppSetting
- Interfaces definen contratos para: repositorios, SAP client, Minio, Email, UnitOfWork

**Objetivo general:**
Implementar la capa Domain con entidades, interfaces de servicio y modelos del dominio de timbrado.

**Objetivos especificos:**
- Crear Common/QueryInfo.cs, Common/FilterItem.cs, Common/SortDirection.cs
- Crear Entities/Cfdi.cs con campos: IdCFDI, UUID, Serie, Folio, RFCEmisor, RFCReceptor, Total, Moneda, XmlContent, EstatusTimbrado, etc.
- Crear Entities/TimbradoLog.cs con campos: IdTimbradoLog, IdCFDI, Accion, Request, Response, DuracionMs, etc.
- Crear Entities/TipoDocumentoFiscal.cs con campos: IdTipoDocumentoFiscal, Clave, Descripcion
- Crear Entities/AppSetting.cs con campos: IdAppSetting, Name, Value, Description
- Crear Interfaces/IGenericRepository.cs, ICfdiRepository.cs, ITimbradoLogRepository.cs
- Crear Interfaces/ISapTimbradoClient.cs, IMinioStorageService.cs, IEmailService.cs, IUnitOfWork.cs
- Crear Models/TimbradoRequest.cs, TimbradoResponse.cs, EmailModel.cs

**Resultado esperado:**
Capa Domain completa con todas las abstracciones necesarias para el servicio de timbrado.

**Entregables:**
- Archivos .cs en carpetas Common/, Entities/, Interfaces/, Models/
- Proyecto compilable sin dependencias externas

**Criterios de aceptacion:**
- El proyecto Domain no tiene referencias a otros proyectos de la solucion
- Las entidades reflejan exactamente la estructura de BD (R16A-RE-FU-018_BD.md)
- ISapTimbradoClient define metodos Timbrar y Cancelar

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
- TimbradoService orquesta: validar request, crear CFDI, llamar SAP, almacenar XML, registrar log
- CfdiService: CRUD + listado paginado con QueryInfo
- FluentValidation para validaciones de request

**Objetivo general:**
Implementar la capa Application con DTOs, servicios de orquestacion y validadores.

**Objetivos especificos:**
- Crear DTOs/QueryResultDto.cs (wrapper generico paginado)
- Crear DTOs/CfdiDto.cs, TimbradoRequestDto.cs, TimbradoResponseDto.cs, TimbradoLogDto.cs
- Crear Interfaces/ITimbradoService.cs (Timbrar, Cancelar, ConsultarEstatus)
- Crear Interfaces/ICfdiService.cs (CRUD + Listar)
- Crear Services/TimbradoService.cs (orquestacion completa del flujo de timbrado)
- Crear Services/CfdiService.cs (operaciones CRUD sobre CFDI)
- Crear Mappers/ApplicationMappingProfile.cs (AutoMapper)
- Crear Validators/TimbradoRequestDtoValidator.cs (FluentValidation)

**Resultado esperado:**
Capa Application completa con logica de orquestacion de timbrado.

**Entregables:**
- TimbradoService con flujo: validar -> crear CFDI -> llamar SAP -> almacenar -> log -> retornar
- CfdiService con CRUD + listado paginado
- Validador de request de timbrado

**Criterios de aceptacion:**
- Application solo depende de Domain
- TimbradoService implementa flujo completo de timbrado (sincrono)
- El validador rechaza requests sin campos obligatorios (RFCEmisor, RFCReceptor, Total, Moneda, TipoDocumento)

**Mas informacion de la tarea:**
Corresponde a GAP-03.

**Recursos:**
- R16A-RE-FU-018-Back.md

---

### Tarea 4

**Titulo:** [ R16A-RE-FU-018 ] [SIMPLE-CRUD] Crear TimbradoContext con DbSet y mapping de entidades

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Infrastructure/Persistence/Context

**Consideraciones previas:**
- Depende de Tarea 2 (Domain - entidades definidas)
- BD: ProquifaDotNetTimbrado (servidor WIN-R14-DEV\DEV_R17_APPS)
- 4 tablas: AppSetting, TipoDocumentoFiscal, CFDI, TimbradoLog
- CFDI tiene FK a TipoDocumentoFiscal; TimbradoLog tiene FK a CFDI

**Objetivo general:**
Crear el TimbradoContext de EF Core con los DbSets y configuracion de mapping para las entidades de timbrado.

**Objetivos especificos:**
- Crear Persistence/Context/TimbradoContext.cs
- Configurar DbSet para Cfdi, TimbradoLog, TipoDocumentoFiscal, AppSetting
- Configurar relaciones FK: CFDI -> TipoDocumentoFiscal, TimbradoLog -> CFDI
- Configurar propiedades con tipos SQL correctos (uniqueidentifier, varchar(max), decimal(18,2), datetime2)
- Configurar connection string desde appsettings.json

**Resultado esperado:**
TimbradoContext funcional conectado a BD ProquifaDotNetTimbrado.

**Entregables:**
- TimbradoContext.cs con OnModelCreating completo
- Configuracion de entidades con Fluent API
- Registro del contexto en DI

**Criterios de aceptacion:**
- El contexto mapea correctamente las 4 tablas
- Las FKs se configuran correctamente
- EstatusTimbrado tiene default 'Pendiente'
- El contexto se inyecta correctamente via DI

**Mas informacion de la tarea:**
Corresponde a GAP-04.

**Recursos:**
- R16A-RE-FU-018-Back.md
- R16A-RE-FU-018_BD.md (DDL completo)

---

### Tarea 5

**Titulo:** [ R16A-RE-FU-018 ] [IMPL-THIRD-SERV] Implementar SapTimbradoClient para integracion con PAC SAP

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Infrastructure/Services

**Consideraciones previas:**
- Depende de Tarea 2 (ISapTimbradoClient definido en Domain)
- SAP es el proveedor de timbrado fiscal (PAC)
- Operaciones: Timbrar (enviar documento, recibir CFDI con UUID/XML) y Cancelar
- Requiere Polly para reintentos y manejo de errores de red
- Configuracion via SapSettings (endpoint, credenciales, timeout)

**Objetivo general:**
Implementar el cliente HTTP para invocar al proveedor SAP y generar/cancelar documentos fiscales CFDI.

**Objetivos especificos:**
- Crear Services/SapTimbradoClient.cs implementando ISapTimbradoClient
- Implementar metodo TimbrarAsync(TimbradoRequest) -> TimbradoResponse (UUID, XML, Serie, Folio)
- Implementar metodo CancelarAsync(string uuid) -> bool
- Configurar HttpClient con Polly para reintentos (3 intentos, backoff exponencial)
- Crear Configuration/SapSettings.cs (Endpoint, ApiKey, Timeout)
- Manejar errores de SAP: timeout, respuesta invalida, rechazo fiscal
- Registrar en DI con HttpClientFactory

**Resultado esperado:**
Cliente HTTP funcional que invoca SAP para timbrar y cancelar CFDI.

**Entregables:**
- SapTimbradoClient.cs con metodos Timbrar y Cancelar
- SapSettings.cs con configuracion tipada
- Manejo de errores y reintentos con Polly
- Registro en DI

**Criterios de aceptacion:**
- Timbrar envia request a SAP y retorna TimbradoResponse con UUID y XML
- Cancelar envia UUID a SAP y retorna confirmacion
- Si SAP falla, Polly reintenta hasta 3 veces con backoff exponencial
- Si falla definitivamente, lanza excepcion controlada con detalle del error SAP
- Timeout configurable via SapSettings

**Mas informacion de la tarea:**
Corresponde a GAP-05.

**Recursos:**
- R16A-RE-FU-018-Back.md
- Soluciones Nuevas.md (Flujo funcional Timbrado)

---

### Tarea 6

**Titulo:** [ R16A-RE-FU-018 ] [IMP-EXIST-SERVICE] Implementar servicios de infraestructura - Minio, Brevo, RabbitMQ

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Infrastructure/Services, Infrastructure/RabbitMQ

**Consideraciones previas:**
- Depende de Tarea 4 (Infrastructure base)
- Minio: almacenamiento de XML timbrados (bucket 'timbrado')
- Brevo: notificaciones de errores criticos de timbrado
- RabbitMQ: cola para reintentos asincronos de timbrado fallido
- Patron basado en proquifa-punchout-backend

**Objetivo general:**
Implementar los servicios de infraestructura para Minio, Brevo y RabbitMQ del servicio de timbrado.

**Objetivos especificos:**
- Crear Services/MinioStorageService.cs (UploadAsync XML, DownloadAsync XML)
- Crear Services/BrevoEmailService.cs (notificacion de errores criticos)
- Crear RabbitMQ/RabbitMQClient.cs + IRabbitMQClient.cs + RabbitMQSettings.cs
- Configurar cola 'timbrado-pendiente' para reintentos
- Crear Configuration/MinioSettings.cs, BrevoSettings.cs

**Resultado esperado:**
Servicios de infraestructura funcionales para el servicio de timbrado.

**Entregables:**
- MinioStorageService con Upload/Download de XML (bucket 'timbrado')
- BrevoEmailService con envio de alertas de error
- RabbitMQClient con Publish/Subscribe para cola de reintentos
- Settings tipados para cada integracion

**Criterios de aceptacion:**
- MinioStorageService sube y descarga XML correctamente
- RabbitMQClient publica mensajes en cola 'timbrado-pendiente'
- BrevoEmailService envia notificacion cuando timbrado falla despues de N reintentos
- Todos los servicios se registran en DI via InfrastructureServiceExtensions

**Mas informacion de la tarea:**
Corresponde a GAP-06, GAP-07 (parcial) y GAP-08.

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

**Objetivo general:**
Configurar la capa API con Program.cs, los controladores de timbrado y toda la configuracion necesaria.

**Objetivos especificos:**
- Crear Program.cs con configuracion (DI, EF Core, Swagger, Serilog, IdentityServer)
- Crear Controllers/TimbradoController.cs con endpoints: POST /timbrar, POST /cancelar
- Crear Controllers/CfdiController.cs con endpoints: GET /{id}, GET /{id}/xml, POST /listar
- Crear appsettings.json con connection strings (ProquifaDotNetTimbrado, SAP, Minio, RabbitMQ, Brevo, IdentityServer)
- Configurar CORS para comunicacion con ProquifaDotNet.Finanzas

**Resultado esperado:**
API ejecutable con Swagger funcional y endpoints de timbrado accesibles.

**Entregables:**
- Program.cs con toda la configuracion de servicios
- TimbradoController con 2 endpoints (timbrar, cancelar)
- CfdiController con 3 endpoints (getById, getXml, listar)
- appsettings.json con placeholders
- API ejecutable via Swagger

**Criterios de aceptacion:**
- La API inicia sin errores
- Swagger muestra todos los endpoints documentados
- IdentityServer valida tokens de autenticacion desde Finanzas
- POST /timbrar recibe TimbradoRequestDto y retorna TimbradoResponseDto
- GET /cfdi/{id}/xml retorna XML desde Minio

**Mas informacion de la tarea:**
Corresponde a GAP-09, GAP-10 y GAP-11.

**Recursos:**
- R16A-RE-FU-018-Back.md

---

### Tarea 8

**Titulo:** [ R16A-RE-FU-018 ] [AUTOMATIC-JOB] Crear Worker.Timbrado para reintentos asincronos via RabbitMQ

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Worker

**Consideraciones previas:**
- Depende de Tareas 5 y 6 (SapTimbradoClient y RabbitMQ)
- Cuando el timbrado sincrono falla, se publica mensaje en cola RabbitMQ
- Worker consume la cola y reintenta timbrado con SAP
- Reintentos configurables (max N intentos)
- Si falla despues de N reintentos: marca CFDI como Error + notifica via Brevo

**Objetivo general:**
Crear el Worker que procesa la cola de reintentos de timbrado fiscal de forma asincrona.

**Objetivos especificos:**
- Crear Worker/Program.cs con configuracion de hosted service
- Crear Worker/TimbradoWorker.cs (BackgroundService que consume cola RabbitMQ)
- Implementar logica de reintento: consumir mensaje -> invocar SAP -> actualizar CFDI
- Implementar DLQ (Dead Letter Queue) para mensajes irrecuperables
- Notificar via Brevo cuando se agotan reintentos
- Configurar reintentos maximos desde AppSetting

**Resultado esperado:**
Worker funcional que procesa reintentos de timbrado de forma asincrona.

**Entregables:**
- TimbradoWorker.cs como BackgroundService
- Logica de consumo de cola y reintento con SAP
- DLQ para mensajes irrecuperables
- Notificacion Brevo en caso de fallo definitivo
- Actualizacion de CFDI.EstatusTimbrado y CFDI.Intentos

**Criterios de aceptacion:**
- El Worker arranca como servicio en segundo plano
- Consume mensajes de cola 'timbrado-pendiente'
- Reintenta timbrado con SAP (hasta N intentos configurables)
- Si exito: actualiza CFDI a EstatusTimbrado=Timbrado
- Si falla definitivamente: actualiza a EstatusTimbrado=Error + envio Brevo
- Registra TimbradoLog por cada intento

**Mas informacion de la tarea:**
Corresponde a GAP-07 (completo).

**Recursos:**
- R16A-RE-FU-018-Back.md (Flujo Asincrono Worker)

---

### Tarea 9

**Titulo:** [ R16A-RE-FU-018 ] [CREATE-SCRIPT-CONTROL] Ejecutar scripts DDL para BD ProquifaDotNetTimbrado

**Aplicativos:** ProquifaDotNet.Timbrado (Base de Datos)

**Modulos:** Base de Datos - Scripts DDL

**Consideraciones previas:**
- Servidor: Mismo que PQF en el que se Ambiente
- Compatibility: 160 (SQL Server 2022)
- 4 tablas: AppSetting, TipoDocumentoFiscal, CFDI, TimbradoLog
- DML inicial: 4 registros en TipoDocumentoFiscal

**Objetivo general:**
Ejecutar los scripts DDL para crear la base de datos ProquifaDotNetTimbrado con sus 4 tablas y datos iniciales.

**Objetivos especificos:**
- Ejecutar CREATE DATABASE ProquifaDotNetTimbrado con rutas de archivos
- Ejecutar CREATE TABLE AppSetting
- Ejecutar CREATE TABLE TipoDocumentoFiscal
- Ejecutar CREATE TABLE CFDI (con FK a TipoDocumentoFiscal)
- Ejecutar CREATE TABLE TimbradoLog (con FK a CFDI)
- Ejecutar INSERT datos iniciales TipoDocumentoFiscal (4 registros: FacturaPorAdelantado, FacturaNormal, FacturaAnticipo, NotaCredito)

**Resultado esperado:**
BD ProquifaDotNetTimbrado creada y operativa con tablas y datos iniciales.

**Entregables:**
- Script DDL ejecutado exitosamente
- BD accesible desde EF Core (TimbradoContext)
- 4 registros en TipoDocumentoFiscal

**Criterios de aceptacion:**
- La BD existe en el servidor indicado
- Las 4 tablas se crean correctamente con PKs, FKs y defaults
- TipoDocumentoFiscal tiene 4 registros activos
- EF Core (TimbradoContext) puede conectar y consultar

**Mas informacion de la tarea:**
Corresponde a GAP-12.

**Recursos:**
- R16A-RE-FU-018_BD.md (Script completo DDL)

---

## Parte B - Modulo Factura por Adelantado en ProquifaDotNet.Finanzas

---

### Tarea 10

**Titulo:** [ R16A-RE-FU-018 ] [ALG-COMPLX-LOGIC] Crear FacturaAdelantadoRepository con query agrupada por cliente

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

**Objetivo general:**
Implementar el repositorio que consulta los pendientes de Factura por Adelantado agrupados por cliente con paginacion y filtros.

**Objetivos especificos:**
- Crear FacturaAdelantadoRepository.cs con metodo ListarPendientesPorCliente(QueryInfo)
- Implementar JOINs: tpPedido -> tpPedidoProformaPedido -> tpProformaPedido -> tpProformaAdelantoProformaPedido -> fccPagoFacturaAdelanto -> tpProformaAdelanto
- Filtrar: FacturaPorAdelantado=1, Tramitado=1, Activo=1, IdCFDIGenerada IS NULL o no enviada
- Filtrar por cartera: ClienteCarteraCliente.IdClienteCartera -> ClienteCartera.IdUsuarioCobrador
- Agrupar por Cliente: COUNT(pendientes), SUM(monto convertido USD), MIN(FechaRegistro)
- Implementar buscador por RazonSocial, RFC/RUC, FolioPedidoInterno
- Soportar paginacion con QueryInfo

**Resultado esperado:**
Repositorio funcional que retorna listado agrupado por cliente con pendientes FAA.

**Entregables:**
- FacturaAdelantadoRepository.cs con query compleja
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
Corresponde a GAP-14.

**Recursos:**
- R16A-RE-FU-018-Back.md (Cadena de Datos)
- R16A-RE-FU-018_BD.md (Parte 1 - Lectura)

---

### Tarea 11

**Titulo:** [ R16A-RE-FU-018 ] [LIST-PAG-MULT-FILTER] Crear FacturaAdelantadoService y Controller con endpoint listado

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Application/Services, API/Controllers

**Consideraciones previas:**
- Depende de Tarea 10 (FacturaAdelantadoRepository)
- El servicio delega al repositorio y aplica logica de negocio
- El controller expone POST /api/factura-adelantado/listar
- Response: QueryResultDto con FacturaAdelantadoClienteDto

**Objetivo general:**
Implementar el servicio y controller para el listado de pendientes FAA agrupados por cliente y por usuario utilizando la Cartera del Cliente, incluyendo la region.

**Objetivos especificos:**
- Crear DTOs/FacturaAdelantadoClienteDto.cs (IdCliente, RazonSocial, RfcRuc, RegionClave, FacturasPendientes, MontoTotalUsd, FechaPendienteMasAntiguo)
- Crear Services/FacturaAdelantadoService.cs con metodo ListarPendientes(QueryInfo)
- Crear Controllers/FacturaAdelantadoController.cs con endpoint POST /api/factura-adelantado/listar
- Inyectar IdUsuarioCobrador desde token IdentityServer (claims del usuario logueado)
- Retornar QueryResultDto con resultados paginados

**Resultado esperado:**
Endpoint funcional de listado FAA paginado con filtros.

**Entregables:**
- FacturaAdelantadoClienteDto.cs
- FacturaAdelantadoService.cs
- FacturaAdelantadoController.cs con endpoint POST /listar
- Registro en DI

**Criterios de aceptacion:**
- POST /api/factura-adelantado/listar acepta QueryInfo
- Retorna QueryResultDto con TotalResults y Results paginados
- El filtro de cartera se aplica automaticamente desde el token del usuario
- El buscador funciona con coincidencia parcial
- El endpoint aparece documentado en Swagger
- Al acceder al modulo FAA en Venta Interna, se muestra el listado agrupado por cliente con pendientes.
- **OBS-034:** El listado incluye pendientes donde el CFDI ya fue timbrado pero el usuario aún no ha ejecutado "Enviar Factura". El envío es una acción explícita posterior al timbrado — **no se envía automáticamente**. El estado "timbrada — pendiente de envío" debe ser visible en el listado.

**Mas informacion de la tarea:**
Corresponde a GAP-13, GAP-15 y GAP-16.

**Recursos:**
- R16A-RE-FU-018-Back.md (Parte B)

---

### Tarea 12

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
Corresponde a GAP-17.

**Recursos:**
- R16A-RE-FU-018-Back.md
- R16A-RE-FU-018_BD.md (Parte 1 - tablas consultadas)

---

## Parte C - ProquifaDotNet (Venta Interna)

---

### Tarea 13

**Titulo:** [ R16A-RE-FU-018 ] [IMP-EXIST-SERVICE] Crear Servicio para consultar FAA en ProquifaDotNet con llamada a API Finanzas

**Aplicativos:** ProquifaDotNet (.NET Framework 4.8)

**Modulos:** Logic.Pqf.Catalogos/ApiCaller

**Consideraciones previas:**
- Depende de Tarea 11 (endpoint Finanzas disponible)
- Modulo FAA es nuevo en Venta Interna (no existe controlador)
- Usa ApiCallerFinanzas existente (creado en RE-FU-016) para llamar a Finanzas

**Objetivo general:**
Crear el servicio para consulta de Factura por Adelantado en ProquifaDotNet para su uso con Pedido

**Objetivos específicos:**
- Agregar metodo ListarPendientesFAA en ApiCallerFinanzas 
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
Corresponde a GAP-18 y GAP-19.

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
| 4 | SIMPLE-CRUD | TimbradoContext con DbSet y mapping | Timbrado | 2 |
| 5 | IMPL-THIRD-SERV | SapTimbradoClient (integracion PAC SAP) | Timbrado | 2 |
| 6 | IMP-EXIST-SERVICE | Servicios infraestructura (Minio, Brevo, RabbitMQ) | Timbrado | 4 |
| 7 | CONFIG-INIT | Crear capa API (Program.cs, Controllers) | Timbrado | 3, 4, 5, 6 |
| 8 | AUTOMATIC-JOB | Worker.Timbrado (reintentos asincronos RabbitMQ) | Timbrado | 5, 6 |
| 9 | CREATE-SCRIPT-CONTROL | Scripts DDL BD ProquifaDotNetTimbrado | BD | - |
| 10 | ALG-COMPLX-LOGIC | FacturaAdelantadoRepository (query agrupada 12+ tablas) | Finanzas | RE-FU-016 |
| 11 | LIST-PAG-MULT-FILTER | FacturaAdelantadoService + Controller listado | Finanzas | 10, 12 |
| 12 | SIMPLE-CRUD | Ampliar FinanzasContext con DbSets FAA | Finanzas | RE-FU-016 T7 |
| 13 | IMP-EXIST-SERVICE | Controlador FAA en ProquifaDotNet + ApiCallerFinanzas | ProquifaDotNet | 11 |

---

## Dependencias con otros requisitos

| Componente requerido | Origen | Usado por |
|---------------------|--------|-----------|
| ProquifaDotNet.Finanzas (solucion completa) | RE-FU-016 | Tareas 10, 11, 12 |
| FinanzasContext base | RE-FU-016 Tarea 7 | Tarea 12 |
| ApiCallerFinanzas | RE-FU-016 Tarea 15 | Tarea 13 |
| BD ProquifaDotNet (tablas FAA) | Existente | Tarea 10 |
