# Tareas BackEnd — TPSC-NO-FU-001
**Requisito:** Migración de Envío de Correo a ProquifaDotNet.SendInBlue (.NET Core 10)  
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.SendInBlue (.NET Core 10)

---

> **Orden de ejecución sugerido:** BD ProquifaDotNetSendInBlue tablas config (T1) → BD tablas operativas + índices (T2) → Solución Domain (T3) → Application CQRS (T4) → Infrastructure Scaffold + Brevo (T5) → API 3 endpoints (T6) → Worker.SendMail + Sincronización (T7) → Migrar plantillas XSLT (T8) → Refactorizar ProquifaDotNet CorreoEnviadoEnviarController (T9) → Refactorizar ProquifaDotNet CorreoGenericoBO (T10).
>
> **Dependencias externas:** Acceso a servidor de BD para crear `ProquifaDotNetSendInBlue`. Credenciales Brevo (API key + URL por región) para poblar `ConfiguracionSendInBlue`. Configuración RabbitMQ (queue `queueSendInBlue`). Configuración IdentityServer para nueva solución.
>
> **Contexto:** El `_Worker.SendInBlue` actual ya consume RabbitMQ (`ProquifaDotNetRabbitMQWorkerService<SendInBlueMessage>`). La nueva solución reemplaza el Windows Service por un Worker .NET Core 10 con arquitectura CQRS y base de datos propia. ProquifaDotNet se refactoriza para llamar al nuevo API en lugar de encolar directamente o llamar a Brevo directamente. La tabla `CorreoEnviado` y relacionadas en ProquifaDotNet no cambian estructuralmente; la nueva solución las lee vía EF Core Scaffold.

---

## Resumen de tareas

| #  | Clave                 | Título simple                                                                                      | Tipo | Aplicativo                     |
| -- | --------------------- | -------------------------------------------------------------------------------------------------- | ---- | ------------------------------ |
| 1  | CREATE-TABL-M         | Crear ProquifaDotNetSendInBlue: tablas ConfiguracionSendInBlue, AppSettings, PlantillaCorreo       | BD   | ProquifaDotNetSendInBlue        |
| 2  | CREATE-TABL-M         | Crear tablas operativas SolicitudCorreo y BitacoraEnvioCorreo + índices en ProquifaDotNet          | BD   | ProquifaDotNetSendInBlue / ProquifaDotNet |
| 3  | CREATE-SOLUTION       | Crear solución ProquifaDotNet.SendInBlue — estructura de proyectos + capa Domain                  | Back | ProquifaDotNet.SendInBlue       |
| 4  | ALG-COMPLX-LOGIC      | Implementar Application layer (CQRS): Commands, Queries, DTOs, Validators                         | Back | ProquifaDotNet.SendInBlue       |
| 5  | SERV-TRANSACT         | Implementar Infrastructure: Scaffold ProquifaDotNet, repositorios, BrevoMailService, renderers     | Back | ProquifaDotNet.SendInBlue       |
| 6  | CREATE-API-ENDPOINT   | Implementar API — 3 endpoints: enviar, simple, html + autenticación IdentityServer                 | Back | ProquifaDotNet.SendInBlue       |
| 7  | CREATE-WORKER         | Implementar Worker.SendMail: SendMailWorker (RabbitMQ + reintentos) + SincronizacionWorker         | Back | ProquifaDotNet.SendInBlue       |
| 8  | IMP-EXIST-SERVICE     | Migrar plantillas XSLT de Logic.MailXslt a nueva solución + Testing unitario e integración         | Back | ProquifaDotNet.SendInBlue       |
| 9  | IMP-EXIST-SERVICE     | Refactorizar ProquifaDotNet — CorreoEnviadoEnviarController: llamar API SendInBlue                 | Back | ProquifaDotNet                  |
| 10 | IMP-EXIST-SERVICE     | Refactorizar ProquifaDotNet — CorreoGenericoBO y módulos dependientes: llamar API SendInBlue       | Back | ProquifaDotNet                  |

---

## TAREA 1

**[ NO-FU-001 ] [CREATE-TABL-M] Crear ProquifaDotNetSendInBlue — tablas ConfiguracionSendInBlue, AppSettings, PlantillaCorreo**

**Aplicativos:** ProquifaDotNetSendInBlue

**Módulos:** Base de Datos — Infraestructura de Configuración

**Consideraciones previas:**
- Base de datos nueva `ProquifaDotNetSendInBlue` dedicada exclusivamente al servicio de envío de correo.
- No comparte tablas con ProquifaDotNet; la comunicación es via EF Core Scaffold (lectura) y llamadas entre APIs.
- `ConfiguracionSendInBlue` reemplaza a `ConfiguracionSendinBlueBO` (.NET 4.8) que actualmente resuelve credenciales Brevo desde `appSettings`. La clave API se almacena cifrada.
- `PlantillaCorreo` centraliza las plantillas HTML para `CorreoGenericoBO` (actualmente en disco). Clave única por región permite variantes regionales.
- `AppSettings` reemplaza las entradas de `appSettings` en `App.config` del Worker actual (intervalo de polling, reintentos, timeouts, paths de plantillas XSLT).
- Poblar `ConfiguracionSendInBlue` con las credenciales existentes en el `appSettings` del `_Worker.SendInBlue`: una fila por región (México, Perú).

**Objetivo general:**
Crear la base de datos `ProquifaDotNetSendInBlue` con las tres tablas de configuración que sustentan la operación del nuevo servicio: credenciales Brevo por región, plantillas HTML y parámetros operativos.

**Objetivos específicos:**
- Crear base de datos `ProquifaDotNetSendInBlue` en el servidor `RYNL010`.
- Ejecutar DDL para `ConfiguracionSendInBlue`, `AppSettings` y `PlantillaCorreo` con sus constraints e índices.
- Insertar filas iniciales de `ConfiguracionSendInBlue` (México y Perú) con datos de configuración actual.
- Verificar que todos los constraints, defaults y registros iniciales son correctos.

**Resultado esperado:**
Base de datos `ProquifaDotNetSendInBlue` creada con las 3 tablas de configuración, datos iniciales de regiones insertados. La nueva solución puede leer credenciales Brevo por región desde `ConfiguracionSendInBlue` sin depender de `appSettings`.

**Entregables:**
- Script DDL: `CREATE DATABASE` + 3 `CREATE TABLE`
- Script DML: `INSERT` datos iniciales ConfiguracionSendInBlue (México + Perú)
- Script de validación

**Diccionario de datos:**

**Tabla: ConfiguracionSendInBlue**

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdConfiguracionSendInBlue` | `uniqueidentifier` PK, DEFAULT NEWID() | Identificador único |
| `IdRegion` | `uniqueidentifier` NOT NULL | FK lógica a `Region` en ProquifaDotNet |
| `Nombre` | `nvarchar(100)` NOT NULL | Nombre descriptivo (ej. "Configuración México") |
| `UrlEnvioCorreo` | `nvarchar(500)` NOT NULL | URL endpoint Brevo (ej. `https://api.brevo.com/v3/smtp/email`) |
| `ClaveAPI` | `nvarchar(500)` NOT NULL | API key de Brevo (cifrada con DPAPI o AES) |
| `CorreoEmisor` | `nvarchar(200)` NOT NULL | Correo remitente por defecto |
| `NombreEmisor` | `nvarchar(200)` NOT NULL | Nombre del remitente |
| `Activo` | `bit` NOT NULL DEFAULT 1 | Configuración activa |
| `FechaRegistro` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Fecha de creación |
| `FechaUltimaActualizacion` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Última modificación |

**Tabla: AppSettings**

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdAppSettings` | `uniqueidentifier` PK, DEFAULT NEWID() | Identificador único |
| `Clave` | `nvarchar(200)` NOT NULL UNIQUE | Clave (ej. `Worker:IntervalMs`, `Worker:MaxIntentos`) |
| `Valor` | `nvarchar(max)` NOT NULL | Valor de la configuración |
| `Descripcion` | `nvarchar(500)` NULL | Descripción del parámetro |
| `FechaUltimaActualizacion` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Última modificación |

**Tabla: PlantillaCorreo**

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdPlantillaCorreo` | `uniqueidentifier` PK, DEFAULT NEWID() | Identificador único |
| `Clave` | `nvarchar(100)` NOT NULL UNIQUE | Clave única (ej. `COTIZACION_MX`, `AUTORIZACION`) |
| `Nombre` | `nvarchar(200)` NOT NULL | Nombre descriptivo |
| `Asunto` | `nvarchar(500)` NULL | Asunto por defecto |
| `ContenidoHtml` | `nvarchar(max)` NOT NULL | HTML con marcadores `{{variable}}` |
| `IdRegion` | `uniqueidentifier` NULL | NULL = global; con valor = específica de región |
| `Activo` | `bit` NOT NULL DEFAULT 1 | Plantilla activa |
| `FechaRegistro` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Fecha de creación |
| `FechaUltimaActualizacion` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Última modificación |

**Criterios de aceptación:**
- [ ] Base de datos `ProquifaDotNetSendInBlue` creada en RYNL010
- [ ] 3 tablas creadas con PK, constraints y defaults correctos
- [ ] 2 filas en `ConfiguracionSendInBlue` (México y Perú) con datos reales de Brevo
- [ ] `SELECT COUNT(*) FROM ConfiguracionSendInBlue` = 2
- [ ] Acceso verificado desde cadena de conexión de la nueva solución

**Más información de la tarea:**
Ver sección 5.2 de `TPSC-NO-FU-001.md` para el detalle completo de cada tabla y sus relaciones.

**Recursos:**
- `C:\Users\juan.garcia\Documents\R16-Documentacion\Requisitos\TPSC-NO-FU-001\TPSC-NO-FU-001.md` — Sección 5.2

---

## TAREA 2

**[ NO-FU-001 ] [CREATE-TABL-M] Crear tablas operativas SolicitudCorreo y BitacoraEnvioCorreo + índices en ProquifaDotNet**

**Aplicativos:** ProquifaDotNetSendInBlue / ProquifaDotNet

**Módulos:** Base de Datos — Cola de Envío y Auditoría

**Consideraciones previas:**
- `SolicitudCorreo` actúa como cola persistente de trabajo para el Worker. Permite reintentos con backoff exponencial usando `FechaProximoIntento` y `MaxIntentos`.
- `BitacoraEnvioCorreo` registra cada intento individual: HTTP status, messageId, body de respuesta Brevo, duración. Esencial para diagnóstico de fallos de envío.
- Los índices en ProquifaDotNet son opcionales pero recomendados para optimizar las consultas frecuentes del Scaffold (consulta de correos pendientes de sincronización y búsqueda por messageId).
- `Estado` en `SolicitudCorreo` usa valores controlados: `PENDIENTE`, `PROCESANDO`, `ENVIADO`, `ERROR`, `CANCELADO`.
- La FK `IdSolicitudCorreo` en `BitacoraEnvioCorreo` es interna a `ProquifaDotNetSendInBlue`.

**Objetivo general:**
Crear las tablas operativas de la cola de envío y bitácora de auditoría en `ProquifaDotNetSendInBlue`, y los índices de optimización en `ProquifaDotNet`.

**Objetivos específicos:**
- Ejecutar DDL para `SolicitudCorreo` y `BitacoraEnvioCorreo` con FK entre ellas e índices operativos.
- Ejecutar scripts de índices sobre `CorreoEnviado` en `ProquifaDotNet` (sincronización y búsqueda por `IdentificadorCorreo`).
- Verificar que el índice filtrado de `SolicitudCorreo` funciona correctamente con estados `PENDIENTE`/`ERROR`.

**Resultado esperado:**
Tablas `SolicitudCorreo` y `BitacoraEnvioCorreo` creadas. Worker puede encolar solicitudes en `SolicitudCorreo`, registrar cada intento en `BitacoraEnvioCorreo` y filtrar eficientemente por estado/fecha. Consultas de sincronización en `CorreoEnviado` optimizadas.

**Entregables:**
- Script DDL: `SolicitudCorreo` + `BitacoraEnvioCorreo` + FK + índices en `ProquifaDotNetSendInBlue`
- Script DDL: índices en `CorreoEnviado` de `ProquifaDotNet`
- Script de validación

**Diccionario de datos:**

**Tabla: SolicitudCorreo** (ProquifaDotNetSendInBlue)

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdSolicitudCorreo` | `uniqueidentifier` PK, DEFAULT NEWID() | Identificador único |
| `IdCorreoEnviado` | `uniqueidentifier` NOT NULL | FK lógica a `CorreoEnviado` en ProquifaDotNet |
| `TipoEnvio` | `nvarchar(50)` NOT NULL | `TEMPLATE` — vía XSLT; `SIMPLE` — plantilla HTML; `HTML` — contenido explícito |
| `Estado` | `nvarchar(50)` NOT NULL DEFAULT 'PENDIENTE' | Estado del ciclo de vida |
| `Intentos` | `int` NOT NULL DEFAULT 0 | Número de intentos realizados |
| `MaxIntentos` | `int` NOT NULL DEFAULT 3 | Máximo configurable de reintentos |
| `FechaCreacion` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Fecha de encolado |
| `FechaProximoIntento` | `datetime2(7)` NULL | Fecha programada para próximo reintento (backoff exponencial) |
| `FechaProcesado` | `datetime2(7)` NULL | Fecha de procesamiento exitoso |
| `ErrorUltimoIntento` | `nvarchar(max)` NULL | Mensaje de error del último intento fallido |
| `BrevoMessageId` | `nvarchar(200)` NULL | messageId retornado por Brevo |
| `Activo` | `bit` NOT NULL DEFAULT 1 | Registro activo |

**Índices SolicitudCorreo:**
- `IX_SolicitudCorreo_Estado_FechaProximoIntento` — filtrado (`Estado IN ('PENDIENTE','ERROR')`) — usado por Worker en polling

**Tabla: BitacoraEnvioCorreo** (ProquifaDotNetSendInBlue)

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdBitacoraEnvioCorreo` | `uniqueidentifier` PK, DEFAULT NEWID() | Identificador único |
| `IdSolicitudCorreo` | `uniqueidentifier` NOT NULL FK → `SolicitudCorreo` | Solicitud asociada |
| `NumeroIntento` | `int` NOT NULL | Número de intento (1-based) |
| `FechaIntento` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Fecha del intento |
| `Exitoso` | `bit` NOT NULL | Si el envío fue exitoso |
| `HttpStatusCode` | `int` NULL | Código HTTP de respuesta Brevo |
| `BrevoMessageId` | `nvarchar(200)` NULL | messageId de respuesta exitosa |
| `BrevoResponseBody` | `nvarchar(max)` NULL | Body completo de respuesta Brevo |
| `ErrorDetalle` | `nvarchar(max)` NULL | Stack trace / detalle de error |
| `DuracionMs` | `int` NULL | Duración del intento en milisegundos |

**Relaciones:**
- `BitacoraEnvioCorreo.IdSolicitudCorreo` → `SolicitudCorreo.IdSolicitudCorreo` (1:N)

**Índices en ProquifaDotNet (CorreoEnviado):**

```sql
-- Optimiza SincronizacionWorker
CREATE NONCLUSTERED INDEX [IX_CorreoEnviado_Sincronizacion]
ON [ProquifaDotNet].[dbo].[CorreoEnviado] ([Activo], [FechaRegistro], [FechaLectura], [FechaSpam])
INCLUDE ([IdCorreoEnviado], [IdentificadorCorreo], [IdRegion]);

-- Optimiza búsqueda por messageId Brevo
CREATE NONCLUSTERED INDEX [IX_CorreoEnviado_IdentificadorCorreo]
ON [ProquifaDotNet].[dbo].[CorreoEnviado] ([IdentificadorCorreo])
WHERE [IdentificadorCorreo] IS NOT NULL;
```

**Criterios de aceptación:**
- [ ] `SolicitudCorreo` y `BitacoraEnvioCorreo` creadas con FK funcional
- [ ] Índice filtrado en `SolicitudCorreo` creado correctamente
- [ ] 2 índices creados en `CorreoEnviado` de ProquifaDotNet
- [ ] Inserción de prueba en `SolicitudCorreo` + `BitacoraEnvioCorreo` exitosa

**Recursos:**
- `TPSC-NO-FU-001.md` — Sección 5.2

---

## TAREA 3

**[ NO-FU-001 ] [CREATE-SOLUTION] Crear solución ProquifaDotNet.SendInBlue — estructura de proyectos + capa Domain**

**Aplicativos:** ProquifaDotNet.SendInBlue

**Módulos:** Arquitectura — Domain

**Consideraciones previas:**
- Solución nueva en .NET Core 10. Seguir el mismo arquetipo de `ProquifaDotNet.Finanzas` y `ProquifaDotNet.Timbrado`.
- La solución incluye 6 proyectos: `Domain`, `Application`, `Infrastructure`, `API`, `Worker.SendMail`, `Testing`.
- El proyecto `Domain` no tiene dependencias de infraestructura. Solo define entidades, interfaces de repositorios e interfaces de servicios.
- Las entidades `SolicitudCorreo`, `BitacoraEnvioCorreo`, `ConfiguracionSendInBlue`, `PlantillaCorreo` y `AppSettings` corresponden 1:1 a las tablas de `ProquifaDotNetSendInBlue`.
- `IBrevoMailService` define el contrato de envío sin acoplar a la implementación HTTP de Brevo.
- Los enums `TipoEnvioCorreo` y `EstadoSolicitudCorreo` deben estar en Domain para uso transversal.

**Objetivo general:**
Crear la solución `ProquifaDotNet.SendInBlue` en .NET Core 10 con la estructura de proyectos completa y la capa Domain implementada (entidades, interfaces, enums).

**Objetivos específicos:**
- Crear solución `.sln` con los 6 proyectos: `Domain`, `Application`, `Infrastructure`, `API`, `Worker.SendMail`, `Testing`.
- Implementar entidades en Domain: `SolicitudCorreo`, `BitacoraEnvioCorreo`, `ConfiguracionSendInBlue`, `PlantillaCorreo`, `AppSettings`.
- Definir interfaces de repositorios: `ISolicitudCorreoRepository`, `IBitacoraEnvioCorreoRepository`, `IConfiguracionSendInBlueRepository`, `IPlantillaCorreoRepository`.
- Definir `IBrevoMailService` con método `SendMail(BrevoMailRequest) : Task<BrevoMailResponse>`.
- Definir enums `TipoEnvioCorreo` (`TEMPLATE`, `SIMPLE`, `HTML`) y `EstadoSolicitudCorreo`.
- Configurar cadena de conexión y referencias de proyecto entre capas.

**Resultado esperado:**
Solución compilable con Domain completamente definido. Las demás capas pueden referenciar Domain sin dependencias circulares.

**Entregables:**
- Solución `ProquifaDotNet.SendInBlue.sln` con 6 proyectos
- Proyecto `ProquifaDotNet.SendInBlue.Domain` — entidades, interfaces, enums
- `README.md` con instrucciones de configuración local

**Criterios de aceptación:**
- [ ] Solución compila sin errores en .NET Core 10
- [ ] 5 entidades de Domain creadas con propiedades mapeadas a las tablas BD
- [ ] 4 interfaces de repositorio definidas
- [ ] `IBrevoMailService` definida
- [ ] 2 enums completos (`TipoEnvioCorreo`, `EstadoSolicitudCorreo`)
- [ ] Sin referencias circulares entre proyectos

**Más información de la tarea:**
Ver sección 6.1 de `TPSC-NO-FU-001.md` para la estructura completa de la solución.

**Recursos:**
- `TPSC-NO-FU-001.md` — Sección 6.1
- Arquetipo: `ProquifaDotNet.Finanzas` (misma estructura de capas)

---

## TAREA 4

**[ NO-FU-001 ] [ALG-COMPLX-LOGIC] Implementar Application layer (CQRS) — Commands, Queries, DTOs, Validators**

**Aplicativos:** ProquifaDotNet.SendInBlue

**Módulos:** Application — CQRS

**Consideraciones previas:**
- Patrón CQRS con MediatR (igual que Finanzas y Timbrado).
- `EnviarCorreoCommand` es el flujo principal: recibe `IdCorreoEnviado`, crea `SolicitudCorreo` en BD y encola en RabbitMQ. No envía directamente.
- `EnviarCorreoSimpleCommand` y `EnviarCorreoHtmlCommand` son flujos alternativos: envío directo con failover a RabbitMQ si Brevo no responde en timeout configurado.
- `SincronizarEstadoCorreoCommand` consulta Brevo por `IdentificadorCorreo` y actualiza `CorreoEnviado` en ProquifaDotNet via Scaffold.
- Los DTOs de entrada (`EnviarCorreoDto`, `EnviarCorreoSimpleDto`, `EnviarCorreoHtmlDto`) mapean directamente con los request del API.
- Validators con FluentValidation: receptores no vacíos, asunto no vacío, `IdCorreoEnviado` not-empty para el flujo principal.

**Objetivo general:**
Implementar la capa Application con los 4 commands, 2 queries, DTOs de entrada/salida y validators con FluentValidation.

**Objetivos específicos:**
- Implementar `EnviarCorreoCommand` + Handler: crea `SolicitudCorreo` con estado `PENDIENTE`, publica mensaje en RabbitMQ.
- Implementar `EnviarCorreoSimpleCommand` + Handler: envío directo con `IBrevoMailService`; si timeout → crea `SolicitudCorreo` y encola.
- Implementar `EnviarCorreoHtmlCommand` + Handler: igual que Simple pero con `htmlContent` explícito y adjuntos por URL MinIO.
- Implementar `SincronizarEstadoCorreoCommand` + Handler: obtiene correos pendientes via Scaffold, consulta estado Brevo, actualiza fechas.
- Implementar Queries: `ObtenerSolicitudCorreoQuery` y `ObtenerBitacoraCorreoQuery`.
- Implementar DTOs y validators con FluentValidation.

**Resultado esperado:**
Capa Application completamente implementada. Los handlers orquestan la lógica sin depender de infraestructura concreta (inyección de dependencias por interfaces).

**Entregables:**
- 4 Commands + Handlers
- 2 Queries + Handlers
- DTOs de entrada y salida
- Validators FluentValidation

**Criterios de aceptación:**
- [ ] 4 Commands compilados con sus Handlers
- [ ] Handlers usan solo interfaces de Domain (no implementaciones concretas)
- [ ] `EnviarCorreoCommand` crea `SolicitudCorreo` con estado `PENDIENTE` y encola en RabbitMQ
- [ ] `EnviarCorreoSimpleCommand` implementa failover a RabbitMQ si timeout
- [ ] Validators rechazan receptores vacíos y asunto vacío
- [ ] Tests unitarios de validators pasan

**Recursos:**
- `TPSC-NO-FU-001.md` — Sección 6.1 (Application)

---

## TAREA 5

**[ NO-FU-001 ] [SERV-TRANSACT] Implementar Infrastructure — Scaffold ProquifaDotNet, repositorios, BrevoMailService, template renderers**

**Aplicativos:** ProquifaDotNet.SendInBlue

**Módulos:** Infrastructure — Persistencia + Integración Brevo + Templates

**Consideraciones previas:**
- Dos `DbContext`: `SendInBlueDbContext` (acceso a `ProquifaDotNetSendInBlue`) y `ProquifaDotNetScaffoldContext` (acceso de lectura/escritura limitada a tablas de ProquifaDotNet).
- El Scaffold de ProquifaDotNet incluye: `CorreoEnviado`, `ArchivoCorreoEnviado`, `Archivo`, `Region`, y las 10 tablas `*CorreoEnviado` por módulo (cotización, pedido, etc.).
- `BrevoMailService` hace HTTP POST a la URL configurada en `ConfiguracionSendInBlue` con la API key correspondiente. Maneja timeout y deserialización de `BrevoMailResponse`.
- `XsltTemplateRenderer` es la migración de `Logic.MailXslt` de .NET 4.8 a .NET Core 10. Recibe objeto serializado + ruta XSLT y devuelve HTML.
- `HtmlTemplateRenderer` reemplaza la lógica de `CorreoGenericoBO`: lee plantilla, sustituye `{{variable}}` con el diccionario de parámetros.
- `RabbitMQPublisher` publica mensajes a la cola `queueSendInBlue`.
- Configuración IdentityServer para autenticar las llamadas al API.
- Serilog configurado con contexto: usuario, módulo, operación (igual que Finanzas/Timbrado).

**Objetivo general:**
Implementar toda la capa de Infrastructure: acceso a ambas bases de datos, cliente HTTP Brevo, renderers de plantillas, RabbitMQ publisher e IdentityServer.

**Objetivos específicos:**
- Generar Scaffold de ProquifaDotNet con las tablas listadas en Sección 6.4 del análisis.
- Implementar `SendInBlueDbContext` + repositorios (`SolicitudCorreoRepository`, `BitacoraEnvioCorreoRepository`, `ConfiguracionSendInBlueRepository`, `PlantillaCorreoRepository`).
- Implementar `BrevoMailService` con `HttpClient`, timeout configurable y manejo de errores HTTP.
- Implementar `XsltTemplateRenderer` compatible con .NET Core 10 (`System.Xml.Xsl.XslCompiledTransform`).
- Implementar `HtmlTemplateRenderer` con sustitución de marcadores `{{variable}}`.
- Implementar `RabbitMQPublisher` para cola `queueSendInBlue`.
- Configurar IdentityServer en `Program.cs` y Serilog.

**Resultado esperado:**
Infrastructure compilada y funcional. Los repositorios persisten/leen de `ProquifaDotNetSendInBlue`. El Scaffold permite leer `CorreoEnviado` y relacionadas desde ProquifaDotNet. `BrevoMailService` puede enviar correos reales a Brevo.

**Entregables:**
- `SendInBlueDbContext` + 4 repositorios
- `ProquifaDotNetScaffoldContext` con Scaffold de tablas ProquifaDotNet
- `BrevoMailService` implementando `IBrevoMailService`
- `XsltTemplateRenderer` + `HtmlTemplateRenderer`
- `RabbitMQPublisher`
- Configuración IdentityServer + Serilog

**Criterios de aceptación:**
- [ ] Scaffold generado con las 10+ tablas de ProquifaDotNet indicadas
- [ ] `BrevoMailService` envía correo de prueba exitosamente a Brevo staging
- [ ] `XsltTemplateRenderer` renderiza HTML desde plantilla XSLT de cotización
- [ ] Repositorios persisten y leen datos de `ProquifaDotNetSendInBlue`
- [ ] RabbitMQ publisher envía mensaje a la cola configurada

**Recursos:**
- `TPSC-NO-FU-001.md` — Secciones 6.1 y 6.4
- Arquetipo: `ProquifaDotNet.Finanzas` Infrastructure (mismo patrón Scaffold)

---

## TAREA 6

**[ NO-FU-001 ] [CREATE-API-ENDPOINT] Implementar API — 3 endpoints: enviar, simple, html + autenticación IdentityServer**

**Aplicativos:** ProquifaDotNet.SendInBlue

**Módulos:** API — Correo

**Consideraciones previas:**
- 3 endpoints en `CorreoController`, todos protegidos con IdentityServer.
- `POST /api/correo/enviar` es el reemplazo directo de `PATCH /EnviarCorreo` de ProquifaDotNet; recibe `idCorreoEnviado` y delega en `EnviarCorreoCommand`.
- `POST /api/correo/simple` reemplaza `CorreoGenericoBO.GenerarCorreo<T>` para correos con plantilla HTML sin objeto de negocio asociado.
- `POST /api/correo/html` reemplaza los casos de `CorreoGenericoBO` con HTML generado por el caller.
- Respuestas estandarizadas: `{ "idSolicitudCorreo": "guid", "estado": "ENCOLADO" }` para los que usan cola; `{ "brevoMessageId": "string", "enviado": true }` para envío directo.
- Mensajes de error con catálogo de códigos: `{ "errorCode": "SENDINBLUE-001", "message": "...", "details": "..." }`.
- Swagger/OpenAPI documentado para facilitar la integración desde ProquifaDotNet.

**Objetivo general:**
Implementar los 3 endpoints REST del API de ProquifaDotNet.SendInBlue con autenticación IdentityServer, documentación Swagger y respuestas estandarizadas.

**Objetivos específicos:**
- Implementar `POST /api/correo/enviar` → `EnviarCorreoCommand` → respuesta con `IdSolicitudCorreo`.
- Implementar `POST /api/correo/simple` → `EnviarCorreoSimpleCommand` → respuesta directa o encolada.
- Implementar `POST /api/correo/html` → `EnviarCorreoHtmlCommand` → respuesta directa o encolada.
- Configurar middleware de autenticación IdentityServer (Bearer token).
- Definir catálogo de códigos de error (`SENDINBLUE-001` a `SENDINBLUE-010`).
- Documentar con Swagger (XML comments + `[ProducesResponseType]`).

**Resultado esperado:**
API desplegable con los 3 endpoints funcionales. ProquifaDotNet puede llamar `POST /api/correo/enviar` con un `idCorreoEnviado` y el correo queda encolado para procesamiento.

**Entregables:**
- `CorreoController` con 3 endpoints
- Modelos de request/response
- Catálogo de códigos de error
- Documentación Swagger

**Criterios de aceptación:**
- [ ] Los 3 endpoints responden HTTP 200 con estructura correcta
- [ ] Llamada sin token retorna HTTP 401
- [ ] `POST /api/correo/enviar` crea registro en `SolicitudCorreo` con estado `PENDIENTE`
- [ ] Swagger UI accesible y documentado
- [ ] Catálogo de errores definido con al menos 5 códigos

**Recursos:**
- `TPSC-NO-FU-001.md` — Sección 6.2

---

## TAREA 7

**[ NO-FU-001 ] [CREATE-WORKER] Implementar Worker.SendMail — SendMailWorker (RabbitMQ + reintentos) + SincronizacionWorker**

**Aplicativos:** ProquifaDotNet.SendInBlue

**Módulos:** Worker.SendMail

**Consideraciones previas:**
- `SendMailWorker` reemplaza al Windows Service `_Worker.SendInBlue` (.NET 4.8). Consume la misma cola RabbitMQ `queueSendInBlue`.
- Flujo: recibe mensaje → lee `SolicitudCorreo` → lee `CorreoEnviado` + relacionados (Scaffold) → resuelve XSLT → descarga adjuntos MinIO → llama `IBrevoMailService` → actualiza `SolicitudCorreo` + `BitacoraEnvioCorreo` + `CorreoEnviado`.
- **Reintentos con backoff exponencial:** fallo → incrementa `Intentos`, calcula `FechaProximoIntento` = `Now + 2^Intentos` minutos → re-encola (o espera polling). Al alcanzar `MaxIntentos` → estado `CANCELADO` + log crítico.
- El mensaje `IdCFDI` que actualmente maneja `SendInBlueWorker` para facturación debe mantenerse: espera que `cfdi.IdArchivoTimbre != null`, construye `CorreoEnviado` de factura, adjunta XML + PDF, re-encola con `IdCorreoEnviado`.
- `SincronizacionWorker` reemplaza `_Ejecutable.SincronizadorSendInBlue`. Ejecuta según cron configurable (`AppSettings["Worker:SincronizacionCron"]`). Consulta Brevo por `IdentificadorCorreo` y actualiza fechas de estado en `CorreoEnviado`.
- Serilog con contexto en cada paso del procesamiento.

**Objetivo general:**
Implementar `SendMailWorker` que procesa la cola RabbitMQ con reintentos configurables, y `SincronizacionWorker` que sincroniza el estado de entrega de correos desde Brevo hacia ProquifaDotNet.

**Objetivos específicos:**
- Implementar `SendMailWorker : BackgroundService` consumiendo cola `queueSendInBlue`.
- Implementar flujo de procesamiento: Scaffold → XSLT → MinIO → Brevo → actualización BD.
- Implementar lógica de reintentos con backoff exponencial (máximo configurable en `AppSettings`).
- Mantener flujo `IdCFDI` para correos de factura (compatibilidad con flujo actual).
- Implementar `SincronizacionWorker : BackgroundService` con cron configurable.
- Configurar Serilog con contexto por mensaje procesado.

**Resultado esperado:**
Worker funcional que procesa correos desde RabbitMQ con reintentos automáticos. Un correo enviado desde ProquifaDotNet llega a Brevo y actualiza el estado en `SolicitudCorreo`, `BitacoraEnvioCorreo` y `CorreoEnviado`.

**Entregables:**
- `SendMailWorker` + lógica de reintentos
- `SincronizacionWorker` + polling cron
- Configuración en `appsettings.json` (cola, cron, MaxIntentos, timeout)

**Criterios de aceptación:**
- [ ] `SendMailWorker` procesa mensaje de RabbitMQ y envía correo a Brevo exitosamente
- [ ] En caso de fallo, incrementa `Intentos` y programa `FechaProximoIntento`
- [ ] Al alcanzar `MaxIntentos`, estado cambia a `CANCELADO` con log crítico
- [ ] `BitacoraEnvioCorreo` registra cada intento con HTTP status, messageId y duración
- [ ] `SincronizacionWorker` actualiza `FechaLectura`/`FechaSpam` en `CorreoEnviado` de ProquifaDotNet
- [ ] Flujo `IdCFDI` (factura) funciona correctamente

**Recursos:**
- `TPSC-NO-FU-001.md` — Sección 6.3
- Código referencia: `_Worker.SendInBlue\SendInBlueWorker.cs` en ProquifaDotNet-R14

---

## TAREA 8

**[ NO-FU-001 ] [IMP-EXIST-SERVICE] Migrar plantillas XSLT de Logic.MailXslt a ProquifaDotNet.SendInBlue + Testing**

**Aplicativos:** ProquifaDotNet.SendInBlue

**Módulos:** Infrastructure — Templates + Testing

**Consideraciones previas:**
- `Logic.MailXslt` en ProquifaDotNet-R14 contiene plantillas XSLT para 10 tipos de correo (ver Sección 2.5 del análisis).
- Las plantillas usan `System.Xml.Xsl.XslCompiledTransform` que es compatible con .NET Core 10 sin cambios.
- Los archivos XSLT físicos (`CorreoCotizacion.xslt`, `CorreoCotizacionMexico.xslt`, `CorreoCotizacionPeru.xslt`, etc.) deben copiarse a la nueva solución como recursos embebidos o como archivos de contenido configurables.
- `GeneradorHtmlCorreo.ObtenerGeneradoresParaCorreo(objeto)` usa reflección para encontrar la clase de plantilla por tipo de objeto. Este mecanismo debe reimplementarse en `XsltTemplateRenderer`.
- Las pruebas unitarias deben verificar que cada tipo de objeto relacionado (`cotCotizacionCorreoEnviado`, `tpPedidoCorreoEnviado`, etc.) genera HTML válido.
- Las pruebas de integración deben verificar el flujo completo: `SolicitudCorreo` → Scaffold → XSLT → `BrevoMailService` mock → `BitacoraEnvioCorreo`.

**Objetivo general:**
Copiar y adaptar todas las plantillas XSLT de `Logic.MailXslt` a la nueva solución, e implementar el conjunto de tests unitarios e integración de la nueva solución.

**Objetivos específicos:**
- Copiar los 10+ archivos XSLT a `Infrastructure/Templates/Xslt/`.
- Adaptar `XsltTemplateRenderer` para resolver plantilla por tipo de objeto (reemplazando `GeneradorHtmlCorreo` por reflección o registro explícito).
- Implementar soporte de plantillas regionales (México/Perú) para cotización.
- Implementar tests unitarios: validators, commands handlers (con mocks), `XsltTemplateRenderer`.
- Implementar tests de integración: flujo completo `EnviarCorreoCommand` con `BrevoMailService` mockeado.
- Verificar que cada tipo de plantilla genera HTML no vacío.

**Resultado esperado:**
Todas las plantillas XSLT funcionando en la nueva solución. Suite de tests con cobertura de los flujos críticos. `XsltTemplateRenderer` resuelve el tipo de plantilla correcto para cada objeto de negocio.

**Entregables:**
- 10+ archivos XSLT migrados a `Infrastructure/Templates/Xslt/`
- `XsltTemplateRenderer` con resolución de plantilla por tipo
- Tests unitarios: Application (commands, validators)
- Tests de integración: flujo de envío end-to-end con mocks

**Criterios de aceptación:**
- [ ] Los 10 tipos de correo generan HTML válido no vacío
- [ ] Plantillas regionales México/Perú funcionan para cotización
- [ ] Tests unitarios pasan (coverage > 70% en Application layer)
- [ ] Test de integración del flujo principal completo pasa con `BrevoMailService` mockeado

**Recursos:**
- `ProquifaDotNet-R14\Logic.MailXslt\` — fuente de plantillas XSLT
- `ProquifaDotNet-R14\_Data\Mail\Extensions\CorreoEnviadoExtensions.cs` — lógica de resolución de plantilla

---

## TAREA 9

**[ NO-FU-001 ] [IMP-EXIST-SERVICE] Refactorizar ProquifaDotNet — CorreoEnviadoEnviarController: llamar API ProquifaDotNet.SendInBlue**

**Aplicativos:** ProquifaDotNet

**Módulos:** WebApi.Catalogos — Sistema — Correos — Envío

**Consideraciones previas:**
- Cambio puntual en `CorreoEnviadoEnviarController.EnviarCorreo`: reemplazar la llamada directa a `RabbitMQClientFactory.FabricarMailClientSendInBlue()` por `HttpClient.PostAsync` al API de ProquifaDotNet.SendInBlue.
- La URL del nuevo API se configura en `appSettings["SendInBlue:ApiUrl"]` para facilitar la transición (puede apuntar al servicio antiguo o nuevo según ambiente).
- La autenticación entre APIs usa IdentityServer: `WebApi.Catalogos` debe obtener un Bearer token para llamar al nuevo servicio.
- El endpoint `GET /ObtenerHtmlCorreo` no cambia (usa reflección local de ProquifaDotNet para preview de HTML).
- Durante la transición, la configuración puede apuntar al servicio antiguo (RabbitMQ directo) o al nuevo API según variable de entorno.
- No se deben hacer cambios en las tablas de BD en esta tarea.

**Objetivo general:**
Refactorizar `CorreoEnviadoEnviarController.EnviarCorreo` para que delegue el envío al API de ProquifaDotNet.SendInBlue mediante llamada HTTP autenticada con IdentityServer.

**Objetivos específicos:**
- Inyectar `HttpClient` (o `IHttpClientFactory`) configurado para `SendInBlueApi` en `CorreoEnviadoEnviarController`.
- Reemplazar `RabbitMQClientFactory.FabricarMailClientSendInBlue().SendMessage(...)` por `POST /api/correo/enviar` con payload `{ "idCorreoEnviado": guid }`.
- Implementar obtención de Bearer token desde IdentityServer para la llamada.
- Agregar `appSettings["SendInBlue:ApiUrl"]` y `appSettings["SendInBlue:ClientId/Secret"]`.
- Mantener manejo de errores: si el nuevo API no responde, retornar `false` con log de error (no romper el flujo de negocio).

**Resultado esperado:**
`PATCH /EnviarCorreo` de ProquifaDotNet llama al nuevo API de ProquifaDotNet.SendInBlue. El correo queda encolado en `SolicitudCorreo` y procesado por `SendMailWorker`. Funcionalidad idéntica al usuario final.

**Entregables:**
- `CorreoEnviadoEnviarController.cs` refactorizado
- Configuración `appSettings` (`SendInBlue:ApiUrl`, credenciales IdentityServer)

**Criterios de aceptación:**
- [ ] `PATCH /EnviarCorreo?idCorreoEnviado={guid}` llama correctamente al nuevo API
- [ ] Si el nuevo API responde 200, `EnviarCorreo` retorna `true`
- [ ] Si el nuevo API no responde, retorna `false` sin excepción no controlada
- [ ] Registro en `SolicitudCorreo` creado correctamente tras la llamada
- [ ] Prueba manual: enviar correo desde UI → correo recibido exitosamente

**Recursos:**
- `ProquifaDotNet-R14\WebApi.Catalogos\Controllers\Sistema\Correos\Envio\CorreoEnviadoEnviarController.cs`
- `TPSC-NO-FU-001.md` — Sección 4.2

---

## TAREA 10

**[ NO-FU-001 ] [IMP-EXIST-SERVICE] Refactorizar ProquifaDotNet — CorreoGenericoBO y módulos dependientes: llamar API ProquifaDotNet.SendInBlue**

**Aplicativos:** ProquifaDotNet

**Módulos:** Logic.Pqf.Catalogos — CorreosEnviados — Múltiples módulos

**Consideraciones previas:**
- `CorreoGenericoBO.GenerarCorreo<T>` actualmente llama `SendInBlueMailService.SendMail` de forma sincrónica directa. Debe refactorizarse para llamar `POST /api/correo/simple` o `POST /api/correo/html` del nuevo API.
- 9 módulos usan `CorreoGenericoBO` (ver Sección 4.2-B del análisis): `ClienteBO.Extensions`, `GMUsuarioClienteCarteraDetalleBO`, `AutorizacionBO`, `cotCotizacionCorreoEnviadoTransaccionBO`, `CorreoCotPartidaInevtigacionBO`, `cotPartidaInvetigacionCorreoFinalizadoBO`, `PretramitarPromesaDeCompraTransaccionBO`, `ppPedidoIncidenciaCorreoTransaccionBO`, `ppPedidosSolicitarFEATransaccionBO`.
- El cambio en `CorreoGenericoBO` es el único cambio necesario: los 9 módulos no se modifican si se mantiene la misma firma del método.
- La serialización del objeto `Data` (XSLT) y la sustitución de parámetros HTML se delegan al nuevo API. `CorreoGenericoBO` solo construye el request y hace la llamada HTTP.
- Si `Data != null` → usar `POST /api/correo/html` con el HTML pre-generado localmente (XSLT se puede mantener en ProquifaDotNet temporalmente) o migrar completamente al endpoint `simple` con el nombre de plantilla.
- Agregar logs de auditoría en cada llamada al nuevo API (moduleId, operación).

**Objetivo general:**
Refactorizar `CorreoGenericoBO.GenerarCorreo<T>` para delegar el envío al API de ProquifaDotNet.SendInBlue, manteniendo compatibilidad total con los 9 módulos que lo consumen.

**Objetivos específicos:**
- Refactorizar `CorreoGenericoBO.GenerarCorreo<T>` para llamar `POST /api/correo/simple` (con `clavePlantilla` y `parametros`) o `POST /api/correo/html` (con `htmlContent` pre-generado).
- Inyectar `HttpClient` con autenticación IdentityServer (mismo patrón que Tarea 9).
- Mantener la firma pública `bool GenerarCorreo<T>(ParametrosCorreoGenerico<T>)` sin cambios para no afectar los 9 módulos dependientes.
- Verificar que los 9 módulos dependientes compilan y funcionan sin cambios.
- Agregar logs con contexto de módulo/operación vía log4net.

**Resultado esperado:**
`CorreoGenericoBO` delega al nuevo API. Los 9 módulos que lo consumen funcionan sin cambios. Correos de autorización, investigación, incidencias y demás flujos se procesan a través de `ProquifaDotNet.SendInBlue`.

**Entregables:**
- `CorreoGenericoBO.cs` refactorizado
- Verificación de compilación de los 9 módulos dependientes

**Criterios de aceptación:**
- [ ] `CorreoGenericoBO.GenerarCorreo<T>` llama al nuevo API correctamente
- [ ] Los 9 módulos dependientes compilan sin cambios
- [ ] Prueba manual de al menos 2 flujos (ej. correo de cotización + correo de autorización) exitosos
- [ ] Errores de la llamada HTTP manejados: retorna `false` sin excepción no controlada
- [ ] Logs de auditoría registran módulo y operación en cada llamada

**Recursos:**
- `ProquifaDotNet-R14\Logic.Pqf.Catalogos\CorreosEnviados\CorreoGenericoBO.cs`
- `TPSC-NO-FU-001.md` — Sección 4.2-B
