# Tareas BackEnd - R16A-RE-FU-018
**Requisito:** Factura por Adelantado (Pantalla Inicial)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10, NUEVA)

> **Nota de arquitectura (correccion — el CFDI no va en Timbrado, va en Finanzas):** el registro de negocio del CFDI (`CfdiController`, tabla `CFDIGenerada`) es propiedad de **ProquifaDotNet.Finanzas**. Timbrado (Parte A) se reduce a un servicio tecnico sin tabla de negocio propia — solo `AppSetting` y `StampingLog`. Ver `R16A-RE-FU-018-Back.md` y `R16A-RE-FU-018_BD.md` para el detalle completo.

---

## Parte A - Creacion de Solucion ProquifaDotNet.Timbrado (servicio tecnico)

---

### Tarea 1

**Titulo:** [ R16A-RE-FU-018 ] [CONFIG-INIT] Crear solucion ProquifaDotNet.Timbrado con estructura Clean Architecture

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Solucion completa

**Consideraciones previas:**
- Tomar como referencia la arquitectura del repositorio proquifa-punchout-backend (.NET 10, Clean Architecture)
- La solucion gestiona **unicamente el aspecto tecnico** del timbrado fiscal: recepcion de datos ya armados por Finanzas, integracion SAP, retorno de UUID/XML/estatus — **sin tabla de negocio propia**
- Requiere .NET 10 como target framework
- BD independiente: ProquifaDotNetTimbrado (solo AppSetting + StampingLog)

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

**Titulo:** [ R16A-RE-FU-018 ] [ARQ-PROJ-NET] Crear capa Domain - Entities, Interfaces y Models (solo tecnico)

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Domain

**Consideraciones previas:**
- Basado en el patron de proquifa-punchout-backend (QueryInfo, FilterItem, SortDirection)
- Entidades corresponden a las 2 tablas de BD ProquifaDotNetTimbrado: StampingLog, AppSetting — **no existe entidad `Cfdi` ni `FiscalDocumentType` en Timbrado**, esa entidad vive en ProquifaDotNet.Finanzas (`CFDIGenerada`)
- Interfaces definen contratos para: repositorio de StampingLog, SAP client, UnitOfWork (sin Minio: la subida de XML es responsabilidad de Finanzas)
- **Nomenclatura (Reglas al diseñar — regla 6):** ProquifaDotNet.Timbrado es solucion nueva, por lo que entidades, propiedades, DTOs, interfaces y metodos se codifican en ingles, sin mezclar palabras en español. `Cfdi`, `Rfc` y `Uuid` se mantienen como terminos fiscales estandar (no se traducen) cuando aparecen en los modelos de intercambio. La palabra "Timbrado" se traduce a "Stamping" dentro del codigo (StampingLog, StampingService, SapStampingClient, StampingRequest/Response, StampingContext); solo el nombre de la solucion/proyecto (ProquifaDotNet.Timbrado, Proquifa.Timbrado.*.csproj/.sln) y de la BD (ProquifaDotNetTimbrado) se mantienen sin traducir por ser nomenclatura ya establecida

**Objetivo general:**
Implementar la capa Domain con entidades, interfaces de servicio y modelos del dominio de timbrado (sin persistencia de negocio del CFDI).

**Objetivos especificos:**
- Crear Common/QueryInfo.cs, Common/FilterItem.cs, Common/SortDirection.cs
- Crear Entities/StampingLog.cs con campos: Id, CfdiGeneradaId (referencia informativa, no FK), Action, PreviousStatus, NewStatus, Request, Response, ErrorMessage, DurationMs, etc.
- Crear Entities/AppSetting.cs con campos: Id, Name, Value, Description
- Crear Interfaces/IStampingLogRepository.cs, ISapStampingClient.cs, IUnitOfWork.cs
- Crear Models/StampingRequest.cs (datos fiscales armados por Finanzas: IssuerRfc, ReceiverRfc, Items, CfdiUse, PaymentMethod, PaymentForm, Currency, ExchangeRate, Total), StampingResponse.cs (Uuid, Series, Folio, IssueDate, Xml, Status, ErrorMessage)

**Resultado esperado:**
Capa Domain completa con todas las abstracciones necesarias para el servicio tecnico de timbrado.

**Entregables:**
- Archivos .cs en carpetas Common/, Entities/, Interfaces/, Models/
- Proyecto compilable sin dependencias externas

**Criterios de aceptacion:**
- El proyecto Domain no tiene referencias a otros proyectos de la solucion
- Las entidades reflejan exactamente la estructura de BD (R16A-RE-FU-018_BD.md, Parte 2: solo AppSetting y StampingLog)
- ISapStampingClient define metodos StampAsync y CancelAsync

**Mas informacion de la tarea:**
Corresponde a GAP-02.

**Recursos:**
- R16A-RE-FU-018-Back.md
- R16A-RE-FU-018_BD.md

---

### Tarea 3

**Titulo:** [ R16A-RE-FU-018 ] [ARQ-PROJ-NET] Crear capa Application - DTOs, StampingService y Validators

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Application

**Consideraciones previas:**
- Depende de Tarea 2 (Domain)
- StampingService orquesta: validar request, llamar SAP, registrar StampingLog, regresar el resultado a Finanzas — **no crea/actualiza ninguna entidad de negocio propia**
- FluentValidation para validaciones de request

**Objetivo general:**
Implementar la capa Application con DTOs y el servicio de orquestacion tecnica del timbrado.

**Objetivos especificos:**
- Crear DTOs/StampingRequestDto.cs, StampingResponseDto.cs, StampingLogDto.cs
- Crear Interfaces/IStampingService.cs (StampAsync, CancelAsync)
- Crear Services/StampingService.cs (validar -> llamar SAP -> registrar StampingLog -> retornar resultado, sin persistir CFDI)
- Crear Mappers/ApplicationMappingProfile.cs (AutoMapper)
- Crear Validators/StampingRequestDtoValidator.cs (FluentValidation)

**Resultado esperado:**
Capa Application completa con logica de orquestacion tecnica de timbrado.

**Entregables:**
- StampingService con flujo: validar -> llamar SAP -> registrar log -> retornar resultado
- Validador de request de timbrado

**Criterios de aceptacion:**
- Application solo depende de Domain
- StampingService implementa flujo completo de timbrado (sincrono) sin persistir ninguna entidad de negocio del CFDI
- El validador rechaza requests sin campos obligatorios (IssuerRfc, ReceiverRfc, Total, Currency)

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
- **2 tablas:** AppSetting, StampingLog — sin Cfdi ni FiscalDocumentType
- StampingLog no tiene FK real (CfdiGeneradaId es solo informativo, apunta a otra base de datos)

**Objetivo general:**
Crear el StampingContext de EF Core con los DbSets y configuracion de mapping para las entidades tecnicas de timbrado.

**Objetivos especificos:**
- Crear Persistence/Context/StampingContext.cs
- Configurar DbSet para StampingLog, AppSetting
- Configurar propiedades con tipos SQL correctos (uniqueidentifier, varchar(max), int, datetime2)
- Configurar connection string desde appsettings.json

**Resultado esperado:**
StampingContext funcional conectado a BD ProquifaDotNetTimbrado.

**Entregables:**
- StampingContext.cs con OnModelCreating completo
- Configuracion de entidades con Fluent API
- Registro del contexto en DI

**Criterios de aceptacion:**
- El contexto mapea correctamente las 2 tablas
- El contexto se inyecta correctamente via DI

**Mas informacion de la tarea:**
Corresponde a GAP-04.

**Recursos:**
- R16A-RE-FU-018-Back.md
- R16A-RE-FU-018_BD.md (Script completo DDL, Parte 2)

---

### Tarea 5

**Titulo:** [ R16A-RE-FU-018 ] [IMPL-THIRD-SERV] Implementar SapStampingClient para integracion con PAC SAP

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** Infrastructure/Services

**Consideraciones previas:**
- Depende de Tarea 2 (ISapStampingClient definido en Domain)
- SAP es el proveedor de timbrado fiscal (PAC)
- Operaciones: Stamp (enviar documento, recibir Uuid/XML) y Cancel
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

> **Tarea eliminada (ex Tarea 6 — MinioStorageService en Timbrado):** Se retira de Timbrado. La subida del XML a Minio y su registro en `Archivo` ahora es responsabilidad de **ProquifaDotNet.Finanzas** (ver Parte B, tarea de MinioStorageService en Finanzas). Timbrado solo regresa el contenido del XML en `StampingResponse.Xml`.

---

### Tarea 6

**Titulo:** [ R16A-RE-FU-018 ] [CONFIG-INIT] Crear capa API - Program.cs, StampingController y configuracion

**Aplicativos:** ProquifaDotNet.Timbrado

**Modulos:** API

**Consideraciones previas:**
- Depende de Tareas 3, 4 y 5
- Integracion con IdentityServer para autenticacion (llamadas desde Finanzas)
- Serilog para logging con contexto
- Swagger para documentacion de endpoints
- **Rutas (Reglas al diseñar — regla 9):** servicio tecnico de uso interno, sin recurso de negocio propio; se usa una accion (`stamp`) en vez de un sustantivo CRUD, ya que no hay entidad persistida detras

**Objetivo general:**
Configurar la capa API con Program.cs, el controlador tecnico de timbrado y toda la configuracion necesaria.

**Objetivos especificos:**
- Crear Program.cs con configuracion (DI, EF Core, Swagger, Serilog, IdentityServer)
- Crear Controllers/StampingController.cs con endpoints: POST /api/v1/stamp (timbrar), POST /api/v1/stamp/cancel
- Crear appsettings.json con connection strings (ProquifaDotNetTimbrado, SAP, IdentityServer)
- Configurar CORS para comunicacion exclusiva con ProquifaDotNet.Finanzas

**Resultado esperado:**
API ejecutable con Swagger funcional y endpoints tecnicos de timbrado accesibles solo desde Finanzas.

**Entregables:**
- Program.cs con toda la configuracion de servicios
- StampingController con 2 endpoints (stamp, cancel)
- appsettings.json con placeholders
- API ejecutable via Swagger

**Criterios de aceptacion:**
- La API inicia sin errores
- Swagger muestra todos los endpoints documentados
- IdentityServer valida tokens de autenticacion desde Finanzas
- POST /api/v1/stamp recibe StampingRequestDto y retorna StampingResponseDto (Uuid, XML, Series, Folio, estatus)

**Mas informacion de la tarea:**
Corresponde a GAP-06 y GAP-07.

**Recursos:**
- R16A-RE-FU-018-Back.md

---

### Tarea 7

**Titulo:** [ R16A-RE-FU-018 ] [CREATE-SCRIPT-CONTROL] Ejecutar scripts DDL para BD ProquifaDotNetTimbrado

**Aplicativos:** ProquifaDotNet.Timbrado (Base de Datos)

**Modulos:** Base de Datos - Scripts DDL

**Consideraciones previas:**
- Servidor: Mismo que PQF en el que se Ambiente
- Compatibility: 160 (SQL Server 2022)
- **2 tablas:** AppSetting, StampingLog — sin Cfdi ni FiscalDocumentType
- **Nomenclatura (Reglas al diseñar — regla 2):** ProquifaDotNetTimbrado es BD nueva, por lo que tablas y columnas se nombran en ingles

**Objetivo general:**
Ejecutar los scripts DDL para crear la base de datos ProquifaDotNetTimbrado con sus 2 tablas.

**Objetivos especificos:**
- Ejecutar CREATE DATABASE ProquifaDotNetTimbrado con rutas de archivos
- Ejecutar CREATE TABLE AppSetting
- Ejecutar CREATE TABLE StampingLog (sin FK, CfdiGeneradaId es solo informativo)

**Resultado esperado:**
BD ProquifaDotNetTimbrado creada y operativa con sus 2 tablas tecnicas.

**Entregables:**
- Script DDL ejecutado exitosamente
- BD accesible desde EF Core (StampingContext)

**Criterios de aceptacion:**
- La BD existe en el servidor indicado
- Las 2 tablas se crean correctamente con PKs y defaults
- EF Core (StampingContext) puede conectar y consultar

**Mas informacion de la tarea:**
Corresponde a GAP-08.

**Recursos:**
- R16A-RE-FU-018_BD.md (Script completo DDL, Parte 2)

---

## Parte B - CfdiController y persistencia de CFDIGenerada en ProquifaDotNet.Finanzas

---

### Tarea 8

**Titulo:** [ R16A-RE-FU-018 ] [SIMPLE-CRUD] Actualizar Scaffold de Finanzas con la extension de CFDIGenerada

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Infrastructure/Persistence

**Consideraciones previas:**
- Depende de RE-FU-019 (CREATE TABLE CFDIGenerada) y del ALTER TABLE de este requisito (R16A-RE-FU-018_BD.md, Parte 3)
- Las tablas de ProquifaDotNet que Finanzas necesita se agregan al EF Core Scaffold en Finanzas.Infrastructure (patron ya establecido)
- Columnas nuevas: IdCatUsoCFDI, IdCatMetodoDePagoCFDI, IdCatMoneda, TipoCambio, Total, IdArchivoXml, Estado, MensajeError, FechaUltimaActualizacion

**Objetivo general:**
Actualizar la entidad `CfdiGenerada` en el Scaffold de Finanzas para reflejar las columnas tecnicas agregadas a `CFDIGenerada`.

**Objetivos especificos:**
- Re-scaffold o actualizar manualmente la entidad `CfdiGenerada` con las columnas nuevas
- Configurar FKs hacia catUsoCFDI, catMetodoDePagoCFDI, catMoneda, Archivo
- Verificar que las columnas existentes (IdCFDIGenerada, RFCEmisor, RFCReceptor, Serie, Folio, FechaEmision, IdCatTipoCFDI, IdCFDIRelacionado, UUID, Activo, FechaRegistro) no se dupliquen

**Resultado esperado:**
Entidad `CfdiGenerada` completa en el Scaffold de Finanzas, lista para ser usada por CfdiService.

**Entregables:**
- Entidad CfdiGenerada actualizada
- Configuracion Fluent API de las nuevas FKs

**Criterios de aceptacion:**
- EF Core mapea correctamente todas las columnas de CFDIGenerada (originales + extension)
- Las FKs se resuelven contra los catalogos existentes (catUsoCFDI, catMetodoDePagoCFDI, catMoneda) y contra Archivo

**Mas informacion de la tarea:**
Corresponde a GAP-09.

**Recursos:**
- R16A-RE-FU-018_BD.md (Parte 3 - Extension de CFDIGenerada)

---

### Tarea 9

**Titulo:** [ R16A-RE-FU-018 ] [ALG-COMPLX-LOGIC] Crear CfdiService (orquestacion de timbrado y persistencia de CFDIGenerada)

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Application/Services

**Consideraciones previas:**
- Depende de Tarea 8 (Scaffold) y Tarea 10 (IApiCallerStamping)
- CfdiService arma los datos fiscales, llama a Timbrado (servicio tecnico) y persiste el resultado en CFDIGenerada + Archivo
- **Nomenclatura (regla 6):** CfdiService en ingles; `Cfdi`, `Rfc`, `Uuid` se mantienen como terminos fiscales estandar

**Objetivo general:**
Implementar el servicio de aplicacion que orquesta el timbrado (via Timbrado) y persiste el CFDI como entidad de negocio en Finanzas.

**Objetivos especificos:**
- Crear Services/CfdiService.cs con metodo GenerateAsync (arma StampingRequestDto, llama IApiCallerStamping, persiste CFDIGenerada + Archivo)
- Crear metodo CancelAsync (llama a Timbrado /cancel, actualiza Estado en CFDIGenerada)
- Crear metodo GetByIdAsync, SearchAsync (QueryInfo)
- Registrar en Bitacora General cada resultado (regla 8) — ver R16A-RE-FU-018-Back.md Parte B
- Manejar el caso de error de Timbrado: UPDATE CFDIGenerada SET Estado='Fallido', MensajeError

**Resultado esperado:**
CfdiService funcional que centraliza la logica de negocio del CFDI en Finanzas.

**Entregables:**
- CfdiService.cs con GenerateAsync, CancelAsync, GetByIdAsync, SearchAsync
- Registro en DI

**Criterios de aceptacion:**
- Al timbrar exitosamente, se crea/actualiza CFDIGenerada con Estado='Timbrado' y se sube el XML via Archivo
- Al fallar el timbrado, CFDIGenerada queda en Estado='Fallido' con MensajeError
- El Id retornado corresponde siempre al Id real de CFDIGenerada (no a un Id de Timbrado)

**Mas informacion de la tarea:**
Corresponde a GAP-10.

**Recursos:**
- R16A-RE-FU-018-Back.md (Parte B - Flujo de persistencia)

---

### Tarea 10

**Titulo:** [ R16A-RE-FU-018 ] [IMP-EXIST-SERVICE] Crear IApiCallerStamping / ApiCallerStamping (cliente hacia Timbrado)

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Application/Interfaces, Infrastructure/Services

**Consideraciones previas:**
- Cliente HTTP desde Finanzas hacia ProquifaDotNet.Timbrado (POST /api/v1/stamp, POST /api/v1/stamp/cancel)
- **Nomenclatura (regla 6):** ApiCallerStamping (no ApiCallerTimbrado), consistente con el precedente ya aplicado a ApiCallerMail

**Objetivo general:**
Implementar el cliente HTTP que Finanzas usa para invocar al servicio tecnico de Timbrado.

**Objetivos especificos:**
- Crear Application/Interfaces/IApiCallerStamping.cs (StampAsync, CancelAsync)
- Crear Infrastructure/Services/ApiCallerStamping.cs (implementacion HTTP)
- Configurar base URL y autenticacion (IdentityServer) via appsettings

**Resultado esperado:**
Cliente HTTP funcional que Finanzas usa para timbrar/cancelar via Timbrado.

**Entregables:**
- IApiCallerStamping.cs, ApiCallerStamping.cs
- Registro en DI

**Criterios de aceptacion:**
- StampAsync envia StampingRequestDto y recibe StampingResponseDto
- CancelAsync envia Uuid y recibe confirmacion
- Maneja errores/timeouts de Timbrado propagandolos a CfdiService

**Mas informacion de la tarea:**
Corresponde a GAP-11.

**Recursos:**
- R16A-RE-FU-018-Back.md (Parte B)

---

### Tarea 11

**Titulo:** [ R16A-RE-FU-018 ] [IMP-EXIST-SERVICE] Implementar MinioStorageService en Finanzas (XML del CFDI)

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Infrastructure/Services

**Consideraciones previas:**
- Reutiliza el patron de persistencia Minio ya establecido en RE-FU-016 (si el servicio ya existe ahi, extenderlo en vez de duplicarlo)
- El XML llega desde Timbrado en StampingResponse.Xml; Finanzas lo sube a Minio y crea el registro Archivo

**Objetivo general:**
Implementar (o extender) el servicio de Minio en Finanzas para almacenar el XML del CFDI y registrar el Archivo asociado.

**Objetivos especificos:**
- Crear/extender Services/MinioStorageService.cs (UploadAsync del XML del CFDI)
- INSERT Archivo (FileKey, FileBucket, IdRegion) tras la subida exitosa
- Vincular CFDIGenerada.IdArchivoXml con el Archivo recien creado

**Resultado esperado:**
XML del CFDI almacenado en Minio y referenciado correctamente desde CFDIGenerada via Archivo.

**Entregables:**
- MinioStorageService (nuevo o extendido) con soporte para XML de CFDI
- Logica de creacion de Archivo + vinculo a CFDIGenerada

**Criterios de aceptacion:**
- El XML se sube correctamente al bucket correspondiente
- Archivo.FileKey/FileBucket permiten descargar el XML posteriormente (GET /api/v1/cfdi/{id}/xml)
- CFDIGenerada.IdArchivoXml queda correctamente vinculado

**Mas informacion de la tarea:**
Corresponde a GAP-12.

**Recursos:**
- R16A-RE-FU-016-Back-Finanzas.md (patron de persistencia Minio, si ya definido)
- R16A-RE-FU-018_BD.md (Parte 3)

---

### Tarea 12

**Titulo:** [ R16A-RE-FU-018 ] [CONFIG-INIT] Crear CfdiController en ProquifaDotNet.Finanzas

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** API/Controllers

**Consideraciones previas:**
- Depende de Tareas 9, 10 y 11
- **Rutas (Reglas al diseñar — regla 9):** recurso `cfdi` en ingles bajo `api/v1/`, CRUD por metodo HTTP, acciones especiales (`cancel`, `xml`, `search`) como subrecurso

**Objetivo general:**
Exponer el recurso de negocio `cfdi` en Finanzas mediante CfdiController.

**Objetivos especificos:**
- Crear Controllers/CfdiController.cs con endpoints: POST /api/v1/cfdi, POST /api/v1/cfdi/{id}/cancel, GET /api/v1/cfdi/{id}, GET /api/v1/cfdi/{id}/xml, POST /api/v1/cfdi/search
- Delegar toda la logica a CfdiService
- Documentar en Swagger

**Resultado esperado:**
API de Finanzas con el recurso `cfdi` completamente funcional.

**Entregables:**
- CfdiController con 5 endpoints (create/stamp, cancel, getById, getXml, search)

**Criterios de aceptacion:**
- POST /api/v1/cfdi arma el timbrado (via Timbrado) y persiste CFDIGenerada + Archivo
- GET /api/v1/cfdi/{id}/xml retorna el XML descargandolo desde Minio via Archivo
- Los endpoints aparecen documentados en Swagger

**Mas informacion de la tarea:**
Corresponde a GAP-13.

**Recursos:**
- R16A-RE-FU-018-Back.md (Parte B)

---

## Parte C - Modulo Factura por Adelantado en ProquifaDotNet.Finanzas (listado)

---

### Tarea 13

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
- Consultar directamente la vista `vfccFactura` (RE-FU-015) — **ya no** se implementan los JOINs `tpPedido -> tpPedidoProformaPedido -> tpProformaPedido -> tpProformaAdelantoProformaPedido -> fccPagoFacturaAdelanto -> tpProformaAdelanto` (migración 06/07/2026, ver `R16A-RE-FU-015_BD.md`)
- Filtrar: `vfccFactura.FacturaPorAdelantado=1, Activo=1, EstadoFAA IN ('PendienteGenerar','PendienteEnviar')`
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
Corresponde a GAP-15.

**Recursos:**
- R16A-RE-FU-018-Back.md (Cadena de Datos)
- R16A-RE-FU-018_BD.md (Parte 1 - Lectura)

---

### Tarea 14

**Titulo:** [ R16A-RE-FU-018 ] [LIST-PAG-MULT-FILTER] Crear AdvanceInvoiceService y Controller con endpoint listado

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Application/Services, API/Controllers

**Consideraciones previas:**
- Depende de Tarea 13 (AdvanceInvoiceRepository)
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
Corresponde a GAP-14, GAP-16 y GAP-17.

**Recursos:**
- R16A-RE-FU-018-Back.md (Parte C)

---

### Tarea 15

**Titulo:** [ R16A-RE-FU-018 ] [SIMPLE-CRUD] Ampliar FinanzasContext con DbSets para tablas FAA

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Infrastructure/Persistence/Context

**Consideraciones previas:**
- Depende de RE-FU-016 Tarea 7 (FinanzasContext ya creado)
- **Actualización (06/07/2026):** ya no se agregan DbSets para `tpProformaAdelanto`/`tpProformaAdelantoProformaPedido` — se reemplazan por un único DbSet de solo lectura sobre la vista `vfccFactura` (RE-FU-015). Se mantienen `ClienteCartera`, `ClienteCarteraCliente`
- Lectura solamente (no CRUD sobre estas tablas/vista)

**Objetivo general:**
Ampliar el FinanzasContext con los DbSets necesarios para las consultas del modulo Factura por Adelantado.

**Objetivos especificos:**
- Agregar `DbSet<VfccFactura>` mapeado con `.ToView("vfccFactura").HasNoKey()` (solo lectura, sin tracking) — reemplaza los DbSets previos de `tpProformaAdelanto`/`tpProformaAdelantoProformaPedido`
- Agregar DbSet para `fccPagoFacturaAdelanto` (FK migrada a `IdFccFactura`, ver RE-FU-026/027)
- Agregar DbSet para ClienteCartera
- Agregar DbSet para ClienteCarteraCliente
- Configurar mapeos con Fluent API (nombres de tabla/vista, tipos)

**Resultado esperado:**
FinanzasContext ampliado con soporte para consultas del modulo FAA.

**Entregables:**
- Nuevos DbSets en FinanzasContext
- Entidades nuevas en Domain/Entities/ (si no existen)
- Configuracion Fluent API para las tablas FAA

**Criterios de aceptacion:**
- Las nuevas entidades se consultan correctamente via EF Core
- No se afectan los DbSets existentes (Proforma, CfdiGenerada, etc.)
- Las relaciones FK se configuran correctamente

**Mas informacion de la tarea:**
Corresponde a GAP-18.

**Recursos:**
- R16A-RE-FU-018-Back.md
- R16A-RE-FU-018_BD.md (Parte 1 - tablas consultadas)

---

## Parte D - ProquifaDotNet (Venta Interna)

---

### Tarea 16

**Titulo:** [ R16A-RE-FU-018 ] [IMP-EXIST-SERVICE] Crear Servicio para consultar FAA en ProquifaDotNet con llamada a API Finanzas

**Aplicativos:** ProquifaDotNet (.NET Framework 4.8)

**Modulos:** Logic.Pqf.Catalogos/ApiCaller

**Consideraciones previas:**
- Depende de Tarea 14 (endpoint Finanzas disponible)
- Modulo FAA es nuevo en Venta Interna (no existe controlador)
- Usa ApiCallerFinanzas existente (creado en RE-FU-016) para llamar a Finanzas

**Objetivo general:**
Crear el servicio para consulta de Factura por Adelantado en ProquifaDotNet para su uso con Pedido

**Objetivos específicos:**
- Agregar metodo ListPendingAdvanceInvoices en ApiCallerFinanzas (invoca POST /api/v1/advanceInvoice/search)
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
Corresponde a GAP-19 y GAP-20.

**Recursos:**
- R16A-RE-FU-018-Back.md (Parte D)
- Logic.Pqf.Catalogos/ApiCaller/ApiCallerFinanzas.cs (referencia)

---

## Resumen de Tareas

| # | Clave | Titulo | Aplicativo | Predecesora |
|---|-------|--------|-----------|-------------|
| 1 | CONFIG-INIT | Crear solucion ProquifaDotNet.Timbrado | Timbrado | - |
| 2 | ARQ-PROJ-NET | Crear capa Domain (solo tecnico: StampingLog, AppSetting) | Timbrado | 1 |
| 3 | ARQ-PROJ-NET | Crear capa Application (StampingService) | Timbrado | 2 |
| 4 | SIMPLE-CRUD | StampingContext con DbSet y mapping (2 tablas) | Timbrado | 2 |
| 5 | IMPL-THIRD-SERV | SapStampingClient (integracion PAC SAP) | Timbrado | 2 |
| 6 | CONFIG-INIT | Crear capa API (Program.cs, StampingController) | Timbrado | 3, 4, 5 |
| 7 | CREATE-SCRIPT-CONTROL | Scripts DDL BD ProquifaDotNetTimbrado (2 tablas) | BD | - |
| 8 | SIMPLE-CRUD | Actualizar Scaffold CfdiGenerada (extension) | Finanzas | RE-FU-019, ALTER TABLE Parte 3 |
| 9 | ALG-COMPLX-LOGIC | CfdiService (orquestacion + persistencia CFDIGenerada) | Finanzas | 8, 10 |
| 10 | IMP-EXIST-SERVICE | IApiCallerStamping / ApiCallerStamping | Finanzas | 6 |
| 11 | IMP-EXIST-SERVICE | MinioStorageService en Finanzas (XML CFDI) | Finanzas | RE-FU-016 |
| 12 | CONFIG-INIT | CfdiController en Finanzas | Finanzas | 9, 10, 11 |
| 13 | ALG-COMPLX-LOGIC | AdvanceInvoiceRepository (query agrupada 12+ tablas) | Finanzas | RE-FU-016 |
| 14 | LIST-PAG-MULT-FILTER | AdvanceInvoiceService + Controller listado | Finanzas | 13, 15 |
| 15 | SIMPLE-CRUD | Ampliar FinanzasContext con DbSets FAA | Finanzas | RE-FU-016 T7 |
| 16 | IMP-EXIST-SERVICE | Controlador FAA en ProquifaDotNet + ApiCallerFinanzas | ProquifaDotNet | 14 |

> Se elimino la tarea Worker.Timbrado/RabbitMQ: sin reintentos en Timbrado. RabbitMQ y Brevo salen de esta solucion. Se elimino/movio la tarea de MinioStorageService de Timbrado a Finanzas (Tarea 11), ya que el registro de negocio del CFDI (y su XML) es propiedad de Finanzas.

---

## Dependencias con otros requisitos

| Componente requerido | Origen | Usado por |
|---------------------|--------|-----------|
| ProquifaDotNet.Finanzas (solucion completa) | RE-FU-016 | Tareas 8-15 |
| FinanzasContext base | RE-FU-016 Tarea 7 | Tarea 15 |
| ApiCallerFinanzas | RE-FU-016 Tarea 15 | Tarea 16 |
| CFDIGenerada (CREATE TABLE) | RE-FU-019 | Tarea 8 |
| catTipoCFDI | RE-FU-028 | Tarea 8, 9 |
| BD ProquifaDotNet (tablas FAA) | Existente | Tarea 13 |
| Politica de reintento ante fallo del PAC | Finanzas, local por punto de generacion: RE-FU-019/020 (FAA), RE-FU-028/029 (Factura), RE-FU-030 (CDP), RE-FU-032..035 (NC) | Todo el flujo de timbrado (Partes A y B) |
