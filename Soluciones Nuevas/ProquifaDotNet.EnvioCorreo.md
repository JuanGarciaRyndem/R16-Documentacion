### Creación de solución **ProquifaDotNet.EnvioCorreo** (SendInBlue)

**Descripción:** Se generará una solución denominada **ProquifaDotNet.EnvioCorreo** (internamente `ProquifaDotNet.SendInBlue`), desarrollada en **.NET Core 10**, cuyo objetivo es centralizar y modernizar el envío de correo electrónico del ecosistema ProquifaDotNet, reemplazando el Windows Service `_Worker.SendInBlue` (.NET 4.8) y el ejecutable `_Ejecutable.SincronizadorSendInBlue`, con soporte multi-región, reintentos automáticos, plantillas XSLT/HTML, plantillas nativas de Brevo y bitácora completa de envíos.

**Alcance funcional:** La solución incluirá las siguientes funcionalidades:
- **Envío por CorreoEnviado** (`POST /api/correo/enviar`): flujo principal vía XSLT — recibe `IdCorreoEnviado` existente en ProquifaDotNet, encola en RabbitMQ y Worker procesa: resolución XSLT → descarga adjuntos MinIO → Brevo.
- **Envío simple** (`POST /api/correo/simple`): notificaciones ad-hoc con plantilla HTML local (`PlantillaCorreo`); envío directo con failover a RabbitMQ si timeout.
- **Envío HTML** (`POST /api/correo/html`): contenido HTML explícito generado por el caller con adjuntos desde MinIO.
- **Envío por plantilla Brevo** (`POST /api/correo/plantilla-brevo`): plantillas nativas de Brevo referenciadas por clave lógica (`CatalogoPlantillaBrevo`) o por `IdTemplateBrevo` directo; HTML renderizado en servidores Brevo.
- **Cola RabbitMQ** con reintentos automáticos y backoff exponencial configurable desde `AppSettings` en BD.
- **SincronizacionWorker**: sincronización periódica de estado de entrega (FechaLectura, FechaSpam, FechaRecepcion) desde Brevo hacia `CorreoEnviado` en ProquifaDotNet.
- **Bitácora por intento**: `BitacoraEnvioCorreo` registra HTTP status, messageId, body de respuesta y duración de cada intento de envío.
- **Soporte multi-región**: credenciales Brevo configuradas por región (México / Perú) en `ConfiguracionSendInBlue`.

**Relación con otros sistemas:**
La integración de ProquifaDotNet.EnvioCorreo con el ecosistema se realiza de la siguiente manera:
- **ProquifaDotNet** (caller principal): refactoriza `CorreoEnviadoEnviarController` (`PATCH /EnviarCorreo`) y `CorreoGenericoBO` para delegar en el nuevo API mediante llamadas HTTP con token IdentityServer, en lugar de encolar directamente a RabbitMQ o llamar a Brevo de forma sincrónica.
- **ProquifaDotNet** (datos): la nueva solución lee `CorreoEnviado`, `ArchivoCorreoEnviado`, `Archivo`, `Region` y las 10 tablas `*CorreoEnviado` (cotización, pedido, etc.) vía EF Core Scaffold — sin modificar las tablas ni los endpoints existentes.
- **ProquifaDotNet.Finanzas**: consumirá `POST /api/correo/enviar` y `POST /api/correo/html` para notificaciones de Factura, Complemento de Pago y Nota de Crédito.
- ** ProquifaDotNetEnvioCorreo**: base de datos propia de la solución — cola de solicitudes, bitácora, configuración y catálogo de plantillas.
- **Brevo (ex-SendInBlue)**: proveedor externo de envío; integración vía HTTP REST (`POST /v3/smtp/email`).
- **MinIO**: origen de archivos adjuntos en los flujos XSLT y HTML.
- **RabbitMQ**: cola `queueSendInBlue` para desacoplamiento del envío asincrónico.
- **IdentityServer**: autenticación de llamadas inter-API (ProquifaDotNet → EnvioCorreo).

**Objetivo principal:**
- Proveer un **API centralizado de envío de correo**, modular y escalable, que reemplace los componentes .NET 4.8 dispersos en múltiples proyectos.
- Garantizar la **trazabilidad completa de cada envío** con bitácora por intento, reintentos automáticos y notificación de fallos definitivos.
- Facilitar la **incorporación de nuevos tipos de correo** sin modificar ProquifaDotNet — cada módulo solo llama al API con los parámetros correspondientes.

## 📂 Estructura de la solución **ProquifaDotNet.EnvioCorreo** (SendInBlue)

### 1. Capas principales

- **Domain**
    - Entidades: `SolicitudCorreo`, `BitacoraEnvioCorreo`, `ConfiguracionSendInBlue`, `PlantillaCorreo`, `CatalogoPlantillaBrevo`, `AppSettings`.
    - Interfaces: `ISolicitudCorreoRepository`, `IBitacoraEnvioCorreoRepository`, `IConfiguracionSendInBlueRepository`, `IPlantillaCorreoRepository`, `ICatalogoPlantillaBrevoRepository`, `IBrevoMailService`.
    - Enums: `TipoEnvioCorreo` (TEMPLATE, SIMPLE, HTML, BREVO_TEMPLATE), `EstadoSolicitudCorreo` (PENDIENTE, PROCESANDO, ENVIADO, ERROR, CANCELADO).
- **Application**
    - Implementación de **CQRS** con MediatR.
    - Commands: `EnviarCorreoCommand`, `EnviarCorreoSimpleCommand`, `EnviarCorreoHtmlCommand`, `EnviarCorreoPlantillaBrevoCommand`, `SincronizarEstadoCorreoCommand`.
    - Queries: `ObtenerSolicitudCorreoQuery`, `ObtenerBitacoraCorreoQuery`.
    - DTOs: `EnviarCorreoDto`, `EnviarCorreoSimpleDto`, `EnviarCorreoHtmlDto`, `EnviarCorreoPlantillaBrevoDto`, `SolicitudCorreoDto`.
    - Validators FluentValidation por cada command de envío.
- **Infrastructure**
    - Persistencia con EF Core — 2 contextos:
        - ` ProquifaDotNetEnvioCorreoDbContext` — acceso a ` ProquifaDotNetEnvioCorreo` (cola, bitácora, configuración, plantillas).
        - `ProquifaDotNetScaffoldContext` — Scaffold de tablas leídas/escritas en ProquifaDotNet (CorreoEnviado y relacionadas).
    - `BrevoMailService` — cliente HTTP hacia Brevo; soporta envío directo y envío por `templateId` (plantillas nativas Brevo).
    - `XsltTemplateRenderer` — migración de `Logic.MailXslt`; resuelve plantilla XSLT por tipo de objeto de negocio.
    - `HtmlTemplateRenderer` — sustitución de marcadores `{{variable}}` en plantillas HTML locales.
    - **RabbitMQ** Publisher/Consumer — cola `queueSendInBlue`.
    - **IdentityServer** — autenticación de llamadas entrantes al API.
    - **Logs** con Serilog enriquecido con contexto (usuario, módulo, operación, intento).
    - Base de datos ` ProquifaDotNetEnvioCorreo` (SolicitudCorreo, BitacoraEnvioCorreo, ConfiguracionSendInBlue, PlantillaCorreo, CatalogoPlantillaBrevo, AppSettings).
- **API**
    - `CorreoController` — 4 endpoints RESTful protegidos con IdentityServer.
    - Respuestas estandarizadas: directo (`brevoMessageId, enviado`) o encolado (`idSolicitudCorreo, estado`).
    - Catálogo de errores: `SENDINBLUE-001` a `SENDINBLUE-010`.
    - Swagger/OpenAPI documentado.
- **Worker.SendMail**
    - `SendMailWorker` — escucha cola RabbitMQ `queueSendInBlue`; procesa los 4 tipos de envío (TEMPLATE/SIMPLE/HTML/BREVO_TEMPLATE) con backoff exponencial.
    - `SincronizacionWorker` — cron configurable; consulta estado de entrega en Brevo y actualiza `CorreoEnviado` en ProquifaDotNet.
    - Reintentos hasta `MaxIntentos` (desde `AppSettings` en BD); al agotar → estado `CANCELADO` + log crítico.
- **Testing**
    - Pruebas unitarias: validators, command handlers (con mocks de repositorios y `IBrevoMailService`), `XsltTemplateRenderer`.
    - Pruebas de integración: flujo completo `EnviarCorreoCommand` con `BrevoMailService` mockeado; verificación de `SolicitudCorreo` y `BitacoraEnvioCorreo`.

### 2. Bases de datos

| Base de datos                | Contexto EF Core                | Propósito                                                                                                                                     |
| ---------------------------- | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| ` ProquifaDotNetEnvioCorreo` | ` ProquifaDotNetEnvioCorreoDbContext`           | Cola de envío, bitácora, configuración Brevo por región, plantillas HTML y catálogo de plantillas nativas Brevo                               |
| `ProquifaDotNet`             | `ProquifaDotNetScaffoldContext` | Lectura de `CorreoEnviado`, adjuntos y tablas `*CorreoEnviado`; escritura de `IdentificadorCorreo`, `FechaEnvio`, `FechaLectura`, `FechaSpam` |

### 3. Flujo funcional

```
1. Caller (ProquifaDotNet / Finanzas)
       │
       └─ POST /api/correo/enviar | /simple | /html | /plantilla-brevo
               │
               ├─ Directo (simple/html/plantilla-brevo)
               │    → BrevoMailService.SendMail / EnviarConPlantillaBrevo
               │    → Éxito: { brevoMessageId, enviado: true }
               │    → Timeout: encola en RabbitMQ
               │
               └─ Encolado (enviar / failover)
                    → INSERT SolicitudCorreo (Estado='PENDIENTE')
                    → Publica en RabbitMQ
                    → { idSolicitudCorreo, estado: 'PENDIENTE' }

2. Worker.SendMail (RabbitMQ)
       │
       └─ Lee SolicitudCorreo + TipoEnvio
            ├─ TEMPLATE  → Scaffold CorreoEnviado → XsltTemplateRenderer → MinIO adjuntos → BrevoMailService
            ├─ SIMPLE    → PlantillaCorreo → HtmlTemplateRenderer → BrevoMailService
            ├─ HTML      → htmlContent almacenado → BrevoMailService
            └─ BREVO_TEMPLATE → CatalogoPlantillaBrevo → BrevoMailService.EnviarConPlantillaBrevo(templateId, params)
                 │
                 ├─ Éxito → UPDATE SolicitudCorreo (ENVIADO) + INSERT BitacoraEnvioCorreo
                 │          UPDATE CorreoEnviado (IdentificadorCorreo, FechaEnvio) en ProquifaDotNet
                 │
                 └─ Error → Backoff exponencial (FechaProximoIntento = Now + 2^Intentos min)
                            INSERT BitacoraEnvioCorreo (Exitoso=0, HttpStatusCode, ErrorDetalle)
                            Al agotar MaxIntentos → UPDATE SolicitudCorreo (CANCELADO) + log crítico

3. SincronizacionWorker (cron configurable)
       └─ Consulta CorreoEnviado enviados sin FechaLectura/FechaSpam
            → Consulta estado en Brevo por IdentificadorCorreo
            → UPDATE CorreoEnviado: FechaLectura / FechaSpam / FechaRecepcion
```

### 4. Tipos de correo soportados (plantillas XSLT)

| Plantilla XSLT                     | Tabla relacionada                                | Módulo                   |
| ---------------------------------- | ------------------------------------------------ | ------------------------ |
| `CorreoCotizacion.xslt` (MX/PE)    | `cotCotizacionCorreoEnviado`                     | Cotización               |
| `CorreoCotPartidaInvetigacion`     | `cotInvestigacionCorreoEnviado`                  | Investigación            |
| `CorreoCotInvestigacionFinalizada` | `cotPartidaInvestigacionFinalizadaCorreoEnviado` | Investigación finalizada |
| `CorreoPretramitacionFEA`          | `ppPedidoCorreoEnviado`                          | Pretramitar Pedido       |
| `CorreoPretramitarPedidoVD`        | `ppPedidoVDCorreoEnviado`                        | Pretramitar Pedido VD    |
| `CorreoPPPedidoIncidencia`         | `ppPedidoIncidenciaCorreoEnviado`                | Incidencias              |
| `CorreoPPPedidoOcNoAmparada`       | `ppPedidoOcNoAmparadaCorreoEnviado`              | OC no amparada           |
| `CorreoPedidoInterno`              | `tpPedidoCorreoEnviado`                          | Tramitar Pedido          |
| `CorreoModPedidoOCTemporal`        | `tpOCTemporalCorreoEnviado`                      | Modificación de Pedido   |
| `CorreoFCCPagoCliente`             | `fccPagoClienteCorreoEnviado`                    | Cobranza — Pago cliente  |

### 5. Estándares transversales

- **Mensajes de error**: catálogo con códigos únicos y respuestas JSON (`errorCode`, `message`, `details`).
- **Logs**: Serilog enriquecido con contexto `{IdSolicitudCorreo, TipoEnvio, NumeroIntento, IdRegion}`.
- **Seguridad**: autenticación IdentityServer — token JWT en header `Authorization: Bearer {token}`.
- **Multi-región**: credenciales Brevo (URL, API key, remitente) configuradas por región en `ConfiguracionSendInBlue`; resolución automática por `IdRegion` en cada envío.
- **Configuración en runtime**: `MaxIntentos`, `BackoffMinutos`, cron de sincronización y flags de activación modificables desde ` ProquifaDotNetEnvioCorreo.AppSettings` sin redeployar.
