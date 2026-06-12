# Tareas BackEnd - R16A-RE-FU-016
**Requisito:** Diseno y generacion de Documentos: Proforma Mexico
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder

---

## Parte A - Creacion de Solucion ProquifaDotNet.Finanzas

---

### Tarea 1

**Titulo:** [ R16A-RE-FU-016 ] [CONFIG-INIT] Crear solucion ProquifaDotNet.Finanzas con estructura Clean Architecture
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Solucion completa

**Consideraciones previas:**
- Tomar como referencia la arquitectura del repositorio proquifa-punchout-backend (.NET 10, Clean Architecture)
- La solucion centraliza: Cobros, Facturacion, Proforma, Notas de Credito, CDP
- Requiere .NET 10 como target framework

**Objetivo general:**
Crear la solucion ProquifaDotNet.Finanzas con la estructura base de proyectos siguiendo Clean Architecture.

**Objetivos especificos:**
- Crear archivo Proquifa.Finanzas.sln
- Crear proyecto Proquifa.Finanzas.Domain.csproj (net10.0)
- Crear proyecto Proquifa.Finanzas.Application.csproj (net10.0)
- Crear proyecto Proquifa.Finanzas.Infrastructure.csproj (net10.0)
- Crear proyecto Proquifa.Finanzas.API.csproj (net10.0)
- Crear proyecto Proquifa.Finanzas.Worker.csproj (net10.0)
- Crear proyecto Proquifa.Finanzas.Testing.csproj (net10.0)
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
Referencia: repositorio proquifa-punchout-backend

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md

---

### Tarea 2

**Titulo:** [ R16A-RE-FU-016 ] [ARQ-PROJ-NET] Crear capa Domain - Common, Entities, Views e Interfaces
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Domain

**Consideraciones previas:**
- Basado en el patron de proquifa-punchout-backend (QueryInfo, FilterItem, SortDirection)
- Las entidades corresponden a tabla tpProformaPedido y vista vtpProformaPedido de BD ProquifaDotNet
- Campos nuevos: FolioProforma (varchar 80) y ConsecutivoProforma (int)

**Objetivo general:**
Implementar la capa Domain con clases comunes, entidades de dominio, vistas e interfaces de repositorio.

**Objetivos especificos:**
- Crear Common/QueryInfo.cs, Common/FilterItem.cs, Common/SortDirection.cs
- Crear Entities/TpProformaPedido.cs con todos los campos incluyendo FolioProforma y ConsecutivoProforma
- Crear Views/VtpProformaPedido.cs con campos de la vista vtpProformaPedido
- Crear Interfaces/IGenericRepository.cs (CRUD generico)
- Crear Interfaces/IProformaRepository.cs (consulta con QueryInfo)
- Crear Interfaces/IEmailService.cs, IMinioStorageService.cs, IUnitOfWork.cs
- Crear Models/EmailModel.cs, EmailTemplateModel.cs

**Resultado esperado:**
Capa Domain completa con todas las abstracciones necesarias para el modulo Proforma.

**Entregables:**
- Archivos .cs en carpetas Common/, Entities/, Views/, Interfaces/, Models/
- Proyecto compilable sin dependencias externas

**Criterios de aceptacion:**
- El proyecto Domain no tiene referencias a otros proyectos de la solucion
- Las entidades reflejan exactamente la estructura de BD (R16A-RE-FU-016_BD.md v2.5)
- QueryInfo soporta filtros dinamicos, ordenamiento y paginacion

**Mas informacion de la tarea:**
Ver codigo de referencia en Back-Finanzas.md secciones 1, 3, 4.

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md
- R16A-RE-FU-016_BD.md

---

### Tarea 3

**Titulo:** [ R16A-RE-FU-016 ] [ARQ-PROJ-NET] Crear capa Application - DTOs, Interfaces, Mappers y Validators
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Application

**Consideraciones previas:**
- Depende de Tarea 2 (Domain)
- Patron CQRS para Commands/Queries
- FluentValidation para validaciones
- AutoMapper para mapeo entidad-DTO
- El ProformaService se crea en Tarea 8 (separada)

**Objetivo general:**
Implementar la capa Application con DTOs, interfaces de servicio, mappers y validadores.

**Objetivos especificos:**
- Crear DTOs/QueryResultDto.cs (wrapper generico paginado)
- Crear DTOs/ProformaDto.cs (DTO para API)
- Crear DTOs/ProformaPdfModel.cs (modelo para DocumentBuilder)
- Crear Interfaces/IProformaService.cs (CRUD + Listar + GenerarPdf)
- Crear Interfaces/IDocumentBuilderClient.cs
- Crear Mappers/ApplicationMappingProfile.cs (AutoMapper)
- Crear Validators/ProformaDtoFluentValidator.cs

**Resultado esperado:**
Capa Application completa con DTOs, interfaces y validadores para el modulo Proforma.

**Entregables:**
- Archivos .cs en carpetas DTOs/, Interfaces/, Mappers/, Validators/
- Interfaces de servicio definidas para implementacion en Infrastructure y en tareas posteriores

**Criterios de aceptacion:**
- Application solo depende de Domain
- QueryResultDto es generico y reutilizable
- Validador verifica campos requeridos de ProformaDto
- Las interfaces IProformaService e IDocumentBuilderClient definen contratos claros

**Mas informacion de la tarea:**
Ver codigo de referencia en Back-Finanzas.md secciones 2, 7.

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md

---

### Tarea 4

**Titulo:** [ R16A-RE-FU-016 ] [ARQ-PROJ-NET] Crear capa Infrastructure - Repositories, Settings y Extensions
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Infrastructure

**Consideraciones previas:**
- Depende de Tarea 2 y 3 (Domain y Application)
- GenericRepository implementa CRUD generico con EF Core
- ProformaRepository implementa consulta con filtros dinamicos y paginacion
- El FinanzasContext se crea en Tarea 7 (separada)

**Objetivo general:**
Implementar la capa Infrastructure con repositorios, configuracion de settings y extensiones de DI.

**Objetivos especificos:**
- Crear Persistence/Repository/GenericRepository.cs (CRUD generico con EF Core)
- Crear Persistence/Repository/ProformaRepository.cs (consulta vtpProformaPedido con QueryInfo, filtros dinamicos y paginacion)
- Crear Configuration/BrevoSettings.cs, MinioSettings.cs, DocumentBuilderSettings.cs
- Crear Extensions/InfrastructureServiceExtensions.cs (DI de todos los servicios)

**Resultado esperado:**
Capa Infrastructure con repositorios funcionales y configuracion de DI.

**Entregables:**
- GenericRepository generico reutilizable
- ProformaRepository con filtros por: clientenombre, folioproforma, foliopedidointerno, empresaprefijo, regionclave, controlados, cancelada, tramitado
- Settings tipados para cada integracion
- InfrastructureServiceExtensions para registro centralizado

**Criterios de aceptacion:**
- ProformaRepository soporta filtros dinamicos, ordenamiento y paginacion
- Los settings se cargan desde appsettings.json via IOptions pattern
- InfrastructureServiceExtensions registra todos los servicios correctamente

**Mas informacion de la tarea:**
Ver codigo de referencia en Back-Finanzas.md seccion 6.

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md

---

### Tarea 5

**Titulo:** [ R16A-RE-FU-016 ] [IMP-EXIST-SERVICE] Implementar servicios de infraestructura - Minio, Brevo, RabbitMQ, DocumentBuilder
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Infrastructure/Services, Infrastructure/RabbitMQ

**Consideraciones previas:**
- Depende de Tarea 4 (Infrastructure base)
- Minio: SDK Minio 6.x para almacenamiento de PDFs
- Brevo: sib_api_v3_sdk para envio de correos transaccionales
- RabbitMQ: RabbitMQ.Client 7.x con ACK/NACK y DLQ
- DocumentBuilder: HttpClient con Polly para reintentos
- Patron basado en proquifa-punchout-backend

**Objetivo general:**
Implementar los servicios de infraestructura necesarios para las integraciones externas del modulo Proforma.

**Objetivos especificos:**
- Crear Services/MinioStorageService.cs (UploadAsync y DownloadAsync con bucket/fileName)
- Crear Services/BrevoEmailService.cs (envio transaccional via REST API v3)
- Crear RabbitMQ/RabbitMQClient.cs + IRabbitMQClient.cs + RabbitMQSettings.cs (Publish/Subscribe con ACK/NACK, DLQ)
- Crear Services/DocumentBuilderHttpClient.cs (PostAsJsonAsync a api/Report/proforma, retorna byte[])
- Configurar Polly para reintentos en DocumentBuilderHttpClient

**Resultado esperado:**
Servicios de infraestructura funcionales listos para integracion con modulo Proforma.

**Entregables:**
- MinioStorageService con Upload/Download de PDFs
- BrevoEmailService con envio via Brevo API
- RabbitMQClient con Publish/Subscribe
- DocumentBuilderHttpClient con llamada a endpoint proforma

**Criterios de aceptacion:**
- MinioStorageService sube y descarga archivos correctamente del bucket configurado
- DocumentBuilderHttpClient maneja errores HTTP y reintentos con Polly
- RabbitMQClient soporta DLQ para mensajes fallidos
- Todos los servicios se registran en DI via InfrastructureServiceExtensions

**Mas informacion de la tarea:**
Ver codigo de referencia en Back-Finanzas.md seccion 9.

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md

---

### Tarea 6

**Titulo:** [ R16A-RE-FU-016 ] [CONFIG-INIT] Crear capa API - Program.cs y configuracion general
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** API

**Consideraciones previas:**
- Depende de Tareas 3, 4 y 5
- Integracion con IdentityServer para autenticacion
- Serilog para logging con contexto
- Swagger para documentacion de endpoints
- Los controllers se crean en Tareas 8 y 9 (separados)

**Objetivo general:**
Configurar la capa API con Program.cs y toda la configuracion necesaria para ejecutar la solucion.

**Objetivos especificos:**
- Crear Program.cs con configuracion de servicios (DI, EF Core, Swagger, Serilog, IdentityServer)
- Crear appsettings.json con connection strings (BD ProquifaDotNet, Minio, RabbitMQ, DocumentBuilder, Brevo, IdentityServer)
- Configurar CORS para comunicacion con ProquifaDotNet (Venta Interna)
- Configurar paquetes NuGet requeridos (ver tabla en Back-Finanzas.md seccion 10)

**Resultado esperado:**
API ejecutable con Swagger funcional y configuracion de infraestructura lista.

**Entregables:**
- Program.cs con toda la configuracion de servicios
- appsettings.json con placeholders de configuracion
- API ejecutable y accesible via Swagger

**Criterios de aceptacion:**
- La API inicia sin errores
- Swagger muestra documentacion (controllers se agregan en tareas posteriores)
- IdentityServer valida tokens de autenticacion
- Serilog registra logs con contexto (usuario, modulo, operacion)

**Mas informacion de la tarea:**
Ver codigo de referencia en Back-Finanzas.md seccion 8.

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md

---

### Tarea 7

**Titulo:** [ R16A-RE-FU-016 ] [SIMPLE-CRUD] Crear FinanzasContext con DbSet y mapping de entidades
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Infrastructure/Persistence/Context

**Consideraciones previas:**
- Depende de Tarea 2 (Domain - entidades y vistas definidas)
- EF Core con scaffold parcial de BD ProquifaDotNet (solo tablas necesarias para Proforma)
- La vista vtpProformaPedido se mapea con HasNoKey()
- La tabla tpProformaPedido se mapea con HasKey(IdTPProformaPedido)

**Objetivo general:**
Crear el FinanzasContext de EF Core con los DbSets y configuracion de mapping para las entidades del modulo Proforma.

**Objetivos especificos:**
- Crear Persistence/Context/FinanzasContext.cs
  - Configurar DbSet TpProformaPedido con mapping a tabla tpProformaPedido y HasKey
- Configurar DbSet VtpProformaPedido con mapping a vista vtpProformaPedido y HasNoKey
- Configurar propiedades con tipos SQL correctos (varchar, decimal precision, datetime2)
- Configurar connection string desde appsettings.json
- Agregar DbSets adicionales necesarios para consultas del modulo (Archivo, Empresa, Cliente, etc.)

**Resultado esperado:**
FinanzasContext funcional con mapping correcto de entidades y vistas.

**Entregables:**
- FinanzasContext.cs con OnModelCreating completo
- Configuracion de entidades con Fluent API
- Registro del contexto en DI (InfrastructureServiceExtensions)

**Criterios de aceptacion:**
- El contexto mapea correctamente la tabla tpProformaPedido con todos sus campos
- La vista vtpProformaPedido se configura con HasNoKey() y ToView()
- Las migraciones se generan sin errores (o se usa scaffold sin migraciones)
- El contexto se inyecta correctamente via DI

**Mas informacion de la tarea:**
Ver codigo de referencia en Back-Finanzas.md seccion 5.

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md
- R16A-RE-FU-016_BD.md

---

### Tarea 8

**Titulo:** [ R16A-RE-FU-016 ] [SIMPLE-CRUD] Crear ProformaService y ProformaController con endpoints CRUD
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Application/Services, API/Controllers

**Consideraciones previas:**
- Depende de Tareas 4, 6 y 7 (Repositories, API configurada, FinanzasContext)
- ProformaService implementa IProformaService definido en Tarea 3
- Endpoints REST estandar para entidad TpProformaPedido
- Usa GenericRepository para operaciones CRUD

**Objetivo general:**
Implementar el servicio ProformaService con operaciones CRUD y el ProformaController con los endpoints REST correspondientes.

**Objetivos especificos:**
- Crear Services/ProformaService.cs con implementacion de: GetById, Create, Update, Delete
- Crear Controllers/ProformaController.cs con endpoints:
  - GET /api/proforma/{id} - Obtener proforma por Id
  - POST /api/proforma - Crear nueva proforma
  - PUT /api/proforma/{id} - Actualizar proforma existente
  - DELETE /api/proforma/{id} - Eliminar proforma (soft delete)
- Registrar ProformaService en DI
- Aplicar validacion con FluentValidator en endpoints de escritura

**Resultado esperado:**
Servicio y controller CRUD funcionales para la entidad TpProformaPedido.

**Entregables:**
- ProformaService.cs en Application/Services/
- ProformaController.cs en API/Controllers/
- Registro en DI

**Criterios de aceptacion:**
- GET /{id} retorna 200 con la proforma o 404 si no existe
- POST retorna 201 con la proforma creada
- PUT retorna 200 con el Id actualizado
- DELETE retorna 204 al eliminar correctamente
- FluentValidator rechaza requests invalidos con 400
- Los endpoints aparecen documentados en Swagger

**Mas informacion de la tarea:**
Ver codigo de referencia en Back-Finanzas.md secciones 7 y 8.

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md

---

### Tarea 9

**Titulo:** [ R16A-RE-FU-016 ] [LIST-PAG-MULT-FILTER] Crear servicio y controller para VtpProformaPedido con listado paginado
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Application/Services, API/Controllers

**Consideraciones previas:**
- Depende de Tareas 4, 6 y 7 (ProformaRepository, API configurada, FinanzasContext)
- VtpProformaPedido es una vista de solo lectura (no CRUD)
- Soporta filtros multiples, ordenamiento dinamico y paginacion
- Usa ProformaRepository para consultas especializadas

**Objetivo general:**
Implementar el servicio de consulta y el controller para la vista VtpProformaPedido con listado paginado y filtros multiples.

**Objetivos especificos:**
- Crear Services/VtpProformaService.cs con consulta paginada usando QueryInfo
- Crear endpoint POST /api/proforma/listar en ProformaController (o controller dedicado)
- Soportar filtros dinamicos por: clientenombre, folioproforma, foliopedidointerno, empresaprefijo, regionclave, controlados, cancelada, tramitado
- Soportar ordenamiento ascendente/descendente por cualquier campo
- Retornar QueryResultDto VtpProformaPedido con TotalResults y Results paginados

**Resultado esperado:**
Endpoint funcional de listado paginado con filtros multiples sobre la vista VtpProformaPedido.

**Entregables:**
- Servicio de consulta con QueryInfo implementado
- Endpoint POST /api/proforma/listar
- QueryResultDto con resultados paginados

**Criterios de aceptacion:**
- El endpoint acepta QueryInfo con filtros, ordenamiento y paginacion
- Retorna QueryResultDto con TotalResults (total sin paginar) y Results (pagina solicitada)
- Los filtros se aplican correctamente (combinables entre si)
- El ordenamiento funciona en ambas direcciones para cualquier campo
- Sin filtros ni paginacion retorna todos los registros
- El endpoint aparece documentado en Swagger

**Mas informacion de la tarea:**
Ver codigo de referencia en Back-Finanzas.md seccion 6 (ProformaRepository con QueryInfo).

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md

---

## Parte B - Tareas de Funcionalidad (Gaps de Desarrollo)

---

### Tarea 10

**Titulo:** [ R16A-RE-FU-016 ] [ALG-COMPLX-LOGIC] Crear modulo Proforma en Finanzas - Command/Query y endpoints de generacion PDF
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Application, API (Modulo Proforma)

**Consideraciones previas:**
- Depende de Tareas 1-9 (solucion Finanzas creada con CRUD y listado funcional)
- El modulo Proforma es el componente central que orquesta la generacion del PDF
- Consulta datos de 15+ tablas de BD ProquifaDotNet via EF Core

**Objetivo general:**
Implementar la logica de Command para generar PDF y Query para consultar historicos del modulo Proforma.

**Objetivos especificos:**
- Implementar Command GenerarProformaPdf que orqueste: consulta datos, armado DTO, llamada DocumentBuilder, retorno byte[]
- Implementar Query ConsultarProformaPdf que descargue PDF de Minio sin regenerar
- Crear endpoint POST /api/proforma/generar-pdf (previsualizacion, sin persistir)
- Crear endpoint GET /api/proforma/{id}/pdf (consulta historica desde Minio)
- Orquestar flujo completo: consulta BD -> armado modelo -> DocumentBuilder -> respuesta

**Resultado esperado:**
Endpoints funcionales para generacion y consulta de PDFs de Proforma.

**Entregables:**
- Command GenerarProformaPdf implementado
- Query ConsultarProformaPdf implementado
- Endpoints en ProformaController para generacion y consulta

**Criterios de aceptacion:**
- POST /api/proforma/generar-pdf retorna byte[] del PDF sin persistir en Minio
- GET /api/proforma/{id}/pdf retorna PDF almacenado en Minio
- El comando consulta correctamente las 15+ tablas necesarias
- Manejo de errores cuando el pedido no existe o DocumentBuilder falla

**Mas informacion de la tarea:**
Corresponde a GAP-01, GAP-05 y GAP-06 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-016-Back.md

---

### Tarea 11

**Titulo:** [ R16A-RE-FU-016 ] [ALG-COMPLX-LOGIC] Crear DTO ProformaModel y BO de armado con datos de 15+ tablas
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Application (DTOs, Services)

**Consideraciones previas:**
- Depende de Tarea 10 (modulo Proforma)
- El DTO ProformaModel es el objeto completo que se envia a DocumentBuilder
- Requiere consultar: tpPedido, tpProformaPedido, tpProformaPartidaPedido, Producto, MarcaFamilia, Cliente, DatosFacturacionCliente, Empresa, EmpresaDatosBancarios, DatosBancarios, catBanco, catMoneda, catCondicionesDePago, DireccionCliente, ContactoCliente

**Objetivo general:**
Implementar el DTO ProformaModel con toda la estructura de datos y el Business Object que lo arma consultando 15+ tablas.

**Objetivos especificos:**
- Crear DTO ProformaModel con secciones: header, cliente, partidas[], pago, datosBancarios, facturacion, entrega
- Implementar BO de armado que consulte todas las tablas relacionadas via EF Core
- Determinar TemplateKey dinamicamente: Empresa.Prefijo + '_MEX_PRO'
- Armar datos bancarios segun empresa (cuenta MN y DLS)
- Armar partidas con: numero, cantidad, descripcion (producto + marca), precioUnitario, importe
- Calcular subtotal, IVA, granTotal con formato moneda

**Resultado esperado:**
DTO ProformaModel completamente armado listo para enviar a DocumentBuilder.

**Entregables:**
- Clase ProformaModel con estructura JSON documentada
- BO ProformaModelBuilder con metodo ArmarProforma(Guid idPedido)
- Queries EF Core para las 15+ tablas

**Criterios de aceptacion:**
- El modelo generado coincide con la estructura JSON definida en el documento Back
- TemplateKey se determina correctamente segun la empresa del pedido
- Los montos se formatean con formato moneda
- Los datos bancarios corresponden a la empresa correcta

**Mas informacion de la tarea:**
Corresponde a GAP-02 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-016-Back.md

---

### Tarea 12

**Titulo:** [ R16A-RE-FU-016 ] [ALG-BASIC-LOGIC] Implementar foliador global con SEQUENCE y logica de Codigo Validador
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Application, Infrastructure

**Consideraciones previas:**
- Depende de Tarea 10 (modulo Proforma)
- El foliador consume NEXT VALUE FOR dbo.SeqFolioProforma
- Formato BD: MMDDAA-Consecutivo (ej: 031826-691)
- Formato visual PDF: PRF-MMDDAA-Consecutivo (prefijo PRF solo en render, no se persiste)
- Codigo Validador (REF. CLIENTE): Banamex = 7 segmentos numericos; no-Banamex = nombre cliente directo
- Momento de consumo: al confirmar envio exitoso (sin huecos)

**Objetivo general:**
Implementar el foliador global de proformas usando SQL SEQUENCE y la logica del Codigo Validador para referencia de pago.

**Objetivos especificos:**
- Implementar consumo de SEQUENCE: NEXT VALUE FOR dbo.SeqFolioProforma
- Generar FolioProforma con formato MMDDAA-Consecutivo
- Persistir FolioProforma y ConsecutivoProforma en tpProformaPedido al confirmar envio
- Implementar logica Codigo Validador: Si banco es Banamex generar 7 segmentos numericos; si no, usar nombre del cliente
- Agregar prefijo PRF- solo en el DTO para DocumentBuilder (visual)

**Resultado esperado:**
Foliador funcional que asigna consecutivos unicos y logica de Codigo Validador correcta segun banco.

**Entregables:**
- Servicio FoliadorProforma con metodo GenerarFolio()
- Logica de Codigo Validador con metodo ObtenerReferenciaCliente(banco, cliente)
- Integracion con flujo de confirmacion de envio

**Criterios de aceptacion:**
- Cada proforma confirmada obtiene un consecutivo unico sin huecos
- El formato es MMDDAA-Consecutivo en BD y PRF-MMDDAA-Consecutivo en PDF
- El Codigo Validador genera 7 segmentos para Banamex y nombre directo para otros bancos
- No se consume SEQUENCE en previsualizacion (solo al confirmar)

**Mas informacion de la tarea:**
Corresponde a GAP-07, GAP-08 y GAP-10 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-016-Back.md
- R16A-RE-FU-016_BD.md

---

### Tarea 13

**Titulo:** [ R16A-RE-FU-016 ] [ALG-BASIC-LOGIC] Implementar conversion de monto a letras (MXN y USD)
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Application

**Consideraciones previas:**
- Depende de Tarea 11 (ProformaModel)
- Se usa en el campo granTotalEnLetra del DTO ProformaModel
- Formato MXN: (MONTO EN LETRAS PESOS XX/100 M.N.)
- Formato USD: (MONTO EN LETRAS DOLARES XX/100)

**Objetivo general:**
Implementar utilidad de conversion de montos numericos a su representacion en letras segun la moneda.

**Objetivos especificos:**
- Crear clase MontoALetrasConverter con metodo Convertir(decimal monto, string moneda)
- Soportar montos en MXN con sufijo PESOS XX/100 M.N.
- Soportar montos en USD con sufijo DOLARES XX/100
- Manejar centavos como fraccion XX/100
- Generar texto en mayusculas
- Soportar montos hasta millones

**Resultado esperado:**
Utilidad funcional que convierte cualquier monto a su representacion textual correcta.

**Entregables:**
- Clase MontoALetrasConverter en Application/Utilities/
- Tests unitarios para conversion MXN y USD

**Criterios de aceptacion:**
- Monto 2972.00 MXN genera: DOS MIL NOVECIENTOS SETENTA Y DOS PESOS 00/100 M.N.
- Monto 1500.50 USD genera: MIL QUINIENTOS DOLARES 50/100
- Todos los montos se generan en mayusculas
- Centavos se representan como fraccion XX/100

**Mas informacion de la tarea:**
Corresponde a GAP-09 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-016-Back.md

---

### Tarea 14

**Titulo:** [ R16A-RE-FU-016 ] [IMP-EXIST-SERVICE] Integracion Finanzas con DocumentBuilder y persistencia Minio
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Infrastructure (Services)

**Consideraciones previas:**
- Depende de Tareas 5 y 11 (servicios infraestructura y ProformaModel)
- DocumentBuilder endpoint: POST api/Report/proforma
- Minio bucket: pedidos, key: proformas/PRF-{FolioProforma}.pdf
- Tabla Archivo: FileKey + FileBucket para registro de PDFs
- Flujo: generar PDF (DocumentBuilder) -> confirmar envio -> persistir (Minio + BD)

**Objetivo general:**
Implementar la integracion completa entre Finanzas, DocumentBuilder y Minio para el ciclo de vida del PDF de Proforma.

**Objetivos especificos:**
- Implementar llamada a DocumentBuilder: POST api/Report/proforma con ProformaModel, retorna byte[]
- Implementar persistencia en Minio al confirmar envio: bucket pedidos, key proformas/PRF-{Folio}.pdf
- Implementar registro en tabla Archivo (INSERT con FileKey, FileBucket, IdRegion=MEX)
- Implementar UPDATE tpPedido.IdArchivo con el Id del archivo creado
- Implementar descarga de PDF desde Minio para consulta historica

**Resultado esperado:**
Flujo completo de generacion, persistencia y consulta de PDFs funcional.

**Entregables:**
- Integracion DocumentBuilderHttpClient con endpoint proforma
- Logica de persistencia Minio con naming convention
- Registro en tabla Archivo + actualizacion tpPedido
- Descarga desde Minio funcional

**Criterios de aceptacion:**
- DocumentBuilder retorna byte[] valido del PDF
- El PDF se almacena en Minio con la key correcta (proformas/PRF-MMDDAA-XXX.pdf)
- La tabla Archivo tiene el registro correcto
- tpPedido.IdArchivo se actualiza con el Id del archivo
- La descarga historica retorna el PDF almacenado

**Mas informacion de la tarea:**
Corresponde a GAP-03 y GAP-04 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-016-Back.md

---

### Tarea 15

**Titulo:** [ R16A-RE-FU-016 ] [IMP-EXIST-SERVICE] Llamada a API Finanzas desde tramitacion en ProquifaDotNet
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8)
**Modulos:** Logic.Pqf.Logistica/L05.TramitarPedido

**Consideraciones previas:**
- Depende de Tarea 10 (endpoints Finanzas disponibles)
- Al tramitar Prepago sin FAA Mexico, se debe llamar a API Finanzas para generar PDF
- Referencia existente: ApiCallerDocumentBuilder.cs (patron EnvioPost)
- El PDF se recibe como byte[] y se retorna al Front para previsualizacion
- Si el ESAC cancela, el PDF se descarta (sin persistencia)

**Objetivo general:**
Implementar la llamada desde ProquifaDotNet (Venta Interna) hacia la API de Finanzas para solicitar generacion de PDF de Proforma al tramitar un pedido.

**Objetivos especificos:**
- Crear ApiCallerFinanzas con metodo para llamar POST /api/proforma/generar-pdf
- Integrar la llamada en el flujo de tramitacion de pedido Prepago sin FAA Mexico
- Recibir byte[] del PDF y retornarlo al controlador para el Front
- Manejar errores de comunicacion con Finanzas (timeout, 500, etc.)

**Resultado esperado:**
Al tramitar un pedido Prepago sin FAA Mexico, se genera el PDF de Proforma y se retorna para previsualizacion.

**Entregables:**
- ApiCallerFinanzas.cs en Logic.Pqf.Catalogos/ApiCaller/
- Integracion en flujo de tramitacion L05
- Manejo de errores y logs

**Criterios de aceptacion:**
- La llamada se ejecuta solo para pedidos Prepago sin FAA de region Mexico
- El byte[] del PDF se retorna correctamente al Front
- Si Finanzas falla, se muestra error controlado al usuario
- No se persiste nada en esta etapa (solo previsualizacion)

**Mas informacion de la tarea:**
Corresponde a GAP-11 y GAP-12 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-016-Back.md
- Logic.Pqf.Catalogos/ApiCaller/ApiCallerDocumentBuilder.cs (referencia de patron)

---

### Tarea 16

**Titulo:** [ R16A-RE-FU-016 ] [IMPL-THIRD-SERV] Crear servicio GenerateProformaTemplate en DocumentBuilder
**Aplicativos:** DocumentBuilder
**Modulos:** Application/Services

**Consideraciones previas:**
- Patron existente: GenerateDocumentService.QuotationExtension.cs
- El servicio recibe un DTO tipado y renderiza HTML con datos JSON para generar PDF
- Mas simple que Quotation (sin merge de documentos)
- Requiere crear DTO tipado DocumentGenerateProformaDto + ProformaModel

**Objetivo general:**
Crear el servicio de generacion de documentos de Proforma en DocumentBuilder siguiendo el patron existente de Quotation.

**Objetivos especificos:**
- Crear GenerateDocumentService.ProformaExtension.cs con metodo de generacion
- Crear DTO DocumentGenerateProformaDto con: FileName, TemplateKey, ClientName, Base64Images, ProformaModel
- Crear endpoint POST api/Report/proforma en ReportController.cs
- Crear DocumentGenerateProformaDtoFluentValidator con validaciones de campos requeridos
- Registrar servicio en DI

**Resultado esperado:**
Endpoint funcional que recibe DTO de Proforma y retorna byte[] del PDF generado.

**Entregables:**
- GenerateDocumentService.ProformaExtension.cs
- DocumentGenerateProformaDto.cs + ProformaModel.cs (tipado)
- Endpoint en ReportController
- FluentValidator
- Registro en DI

**Criterios de aceptacion:**
- El endpoint POST api/Report/proforma recibe DTO y retorna byte[] (PDF)
- El servicio busca template por TemplateKey en BD
- Los datos del ProformaModel se inyectan correctamente en el HTML
- El validator rechaza requests sin campos requeridos (FileName, TemplateKey, ProformaModel)
- El patron es consistente con QuotationExtension existente

**Mas informacion de la tarea:**
Corresponde a GAP-13, GAP-14, GAP-15 y GAP-16 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-016-Back.md
- Application/Services/GenerateDocumentService.QuotationExtension.cs (referencia)

---

### Tarea 17

**Titulo:** [ R16A-RE-FU-016 ] [CREATE-PDF] Crear 4 templates HTML de Proforma Mexico (12 archivos)
**Aplicativos:** DocumentBuilder
**Modulos:** Templates

**Consideraciones previas:**
- 4 empresas: Golocaer (GOL), Mungen (MUN), Proquifa (PQF), Proveedora (PRO)
- Cada template tiene 3 archivos: _H.html (Header), _B.html (Body), _F.html (Footer)
- Total: 12 archivos HTML
- 3 variantes visuales: Naranja/Banorte (GOL), Verde/Banamex (MUN), Teal/Banamex (PQF+PRO)
- TemplateKeys: GOL_MEX_PRO, MUN_MEX_PRO, PQF_MEX_PRO, PRO_MEX_PRO

**Objetivo general:**
Crear los 12 archivos HTML de templates de Proforma Mexico para las 4 empresas con sus variantes visuales.

**Objetivos especificos:**
- Crear carpeta GOL_MEX_PRO/ con GOL_MEX_PRO_H.html, GOL_MEX_PRO_B.html, GOL_MEX_PRO_F.html
- Crear carpeta MUN_MEX_PRO/ con MUN_MEX_PRO_H.html, MUN_MEX_PRO_B.html, MUN_MEX_PRO_F.html
- Crear carpeta PQF_MEX_PRO/ con PQF_MEX_PRO_H.html, PQF_MEX_PRO_B.html, PQF_MEX_PRO_F.html
- Crear carpeta PRO_MEX_PRO/ con PRO_MEX_PRO_H.html, PRO_MEX_PRO_B.html, PRO_MEX_PRO_F.html
- Maquetacion HTML/CSS: cabecera con logo, tabla de partidas, panel inferior 4 columnas (pago, bancarios, facturacion, entrega), pie con certificaciones
- Aplicar colores por variante visual (Naranja, Verde, Teal)

**Resultado esperado:**
12 archivos HTML funcionales que renderizan correctamente la Proforma de cada empresa.

**Entregables:**
- 12 archivos HTML en carpetas por empresa
- CSS embebido con variantes de color
- Placeholders Handlebars/Mustache para datos dinamicos del ProformaModel

**Criterios de aceptacion:**
- Cada template renderiza correctamente con datos de prueba
- Los logos corresponden a cada empresa
- Los colores de marca son correctos (Naranja=GOL, Verde=MUN, Teal=PQF/PRO)
- La estructura del documento coincide con las imagenes de referencia del requisito
- El panel inferior muestra 4 columnas: datos de pago, bancarios, facturacion y entrega

**Mas informacion de la tarea:**
Corresponde a GAP-17 y GAP-19 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-016-Back.md
- Imagenes de referencia de proformas (4 empresas)

---

### Tarea 18

**Titulo:** [ R16A-RE-FU-016 ] [BD-OBJ-CH] Registrar templates de Proforma en BD DocumentBuilder y preparar logos
**Aplicativos:** DocumentBuilder
**Modulos:** Base de Datos, Assets

**Consideraciones previas:**
- Depende de Tarea 17 (templates HTML creados)
- La tabla DocumentTemplate almacena los registros de templates disponibles
- Patron existente: registros de COT y PED ya existen
- Los logos deben estar en base64 o en repositorio de assets accesible

**Objetivo general:**
Registrar los 4 templates de Proforma en la base de datos de DocumentBuilder y preparar los assets graficos necesarios.

**Objetivos especificos:**
- INSERT en tabla DocumentTemplate para GOL_MEX_PRO, MUN_MEX_PRO, PQF_MEX_PRO, PRO_MEX_PRO
- Preparar logos por empresa en formato adecuado (base64 o path)
- Preparar sellos y certificaciones en formato adecuado
- Verificar que los templates se resuelven correctamente con TemplateKey

**Resultado esperado:**
Templates registrados en BD y assets graficos disponibles para renderizacion.

**Entregables:**
- Script SQL con 4 INSERTs en DocumentTemplate
- Logos de 4 empresas preparados
- Sellos de certificaciones preparados

**Criterios de aceptacion:**
- Los 4 templates se encuentran en BD con TemplateKey correcto
- DocumentBuilder resuelve correctamente cada template por su key
- Los logos se renderizan correctamente en el PDF generado
- Los sellos de certificaciones aparecen en el pie de pagina

**Mas informacion de la tarea:**
Corresponde a GAP-18 y GAP-20 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-016-Back.md

---

### Tarea 19

**Titulo:** [ R16A-RE-FU-016 ] [SERV-TRANSACT] Implementar flujo transaccional de confirmacion de envio de Proforma
**Aplicativos:** ProquifaDotNet.Finanzas
**Modulos:** Application, Infrastructure

**Consideraciones previas:**
- Depende de Tareas 12, 13 y 14 (foliador, monto a letras, integracion Minio)
- El flujo transaccional se ejecuta al confirmar el envio (despues de previsualizacion)
- Debe ser atomico: si falla Minio o BD, se hace rollback completo
- Orden: SEQUENCE -> subir PDF Minio -> INSERT Archivo -> UPDATE tpPedido -> UPDATE tpProformaPedido

**Objetivo general:**
Implementar el flujo transaccional completo que se ejecuta al confirmar el envio de la Proforma (post-previsualizacion).

**Objetivos especificos:**
- Consumir NEXT VALUE FOR dbo.SeqFolioProforma para obtener consecutivo
- Generar FolioProforma con formato MMDDAA-Consecutivo
- Subir PDF a Minio: bucket pedidos, key proformas/PRF-{FolioProforma}.pdf
- INSERT en tabla Archivo (FileKey, FileBucket, IdRegion=MEX)
- UPDATE tpPedido SET IdArchivo = @IdArchivo
- UPDATE tpProformaPedido SET FolioProforma = @FolioProforma, ConsecutivoProforma = @Consecutivo
- Garantizar atomicidad con IUnitOfWork (rollback si falla cualquier paso)

**Resultado esperado:**
Flujo transaccional que confirma el envio, asigna folio, persiste PDF y actualiza registros de forma atomica.

**Entregables:**
- Command ConfirmarEnvioProforma implementado
- Transaccion con IUnitOfWork
- Manejo de errores y rollback

**Criterios de aceptacion:**
- El folio se asigna solo al confirmar (no en previsualizacion)
- Si falla la subida a Minio, no se actualiza BD (rollback)
- Si falla el UPDATE de BD, se elimina el archivo de Minio (compensacion)
- El consecutivo nunca se repite (garantizado por SEQUENCE)
- Los registros de Archivo y tpPedido se actualizan correctamente

**Mas informacion de la tarea:**
Corresponde al flujo de persistencia documentado en Back.md. Integra GAP-07 con GAP-04.

**Recursos:**
- R16A-RE-FU-016-Back.md
- R16A-RE-FU-016_BD.md

---

## Resumen de Tareas

| # | Clave | Titulo | Aplicativo | Predecesora |
|---|-------|--------|-----------|-------------|
| 1 | CONFIG-INIT | Crear solucion ProquifaDotNet.Finanzas | Finanzas | - |
| 2 | ARQ-PROJ-NET | Crear capa Domain | Finanzas | 1 |
| 3 | ARQ-PROJ-NET | Crear capa Application (DTOs, Interfaces, Validators) | Finanzas | 2 |
| 4 | ARQ-PROJ-NET | Crear capa Infrastructure (Repositories, Settings, Extensions) | Finanzas | 2, 3 |
| 5 | IMP-EXIST-SERVICE | Servicios infraestructura (Minio, Brevo, RabbitMQ, DocumentBuilder) | Finanzas | 4 |
| 6 | CONFIG-INIT | Crear capa API (Program.cs, configuracion) | Finanzas | 3, 4, 5 |
| 7 | SIMPLE-CRUD | FinanzasContext con DbSet y mapping | Finanzas | 2 |
| 8 | SIMPLE-CRUD | ProformaService + ProformaController CRUD | Finanzas | 4, 6, 7 |
| 9 | LIST-PAG-MULT-FILTER | Servicio y Controller VtpProformaPedido (listado paginado) | Finanzas | 4, 6, 7 |
| 10 | ALG-COMPLX-LOGIC | Modulo Proforma (Command/Query/Endpoints PDF) | Finanzas | 8, 9 |
| 11 | ALG-COMPLX-LOGIC | DTO ProformaModel y BO armado (15+ tablas) | Finanzas | 10 |
| 12 | ALG-BASIC-LOGIC | Foliador SEQUENCE + Codigo Validador | Finanzas | 10 |
| 13 | ALG-BASIC-LOGIC | Conversion monto a letras (MXN/USD) | Finanzas | 11 |
| 14 | IMP-EXIST-SERVICE | Integracion DocumentBuilder + Minio | Finanzas | 5, 11 |
| 15 | IMP-EXIST-SERVICE | Llamada a API Finanzas desde ProquifaDotNet | ProquifaDotNet | 10 |
| 16 | IMPL-THIRD-SERV | Servicio ProformaTemplate en DocumentBuilder | DocumentBuilder | - |
| 17 | CREATE-PDF | 4 templates HTML Proforma (12 archivos) | DocumentBuilder | 16 |
| 18 | BD-OBJ-CH | Registrar templates en BD + logos | DocumentBuilder | 17 |
| 19 | SERV-TRANSACT | Flujo transaccional confirmacion envio | Finanzas | 12, 13, 14 |
