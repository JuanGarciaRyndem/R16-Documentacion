# R16A-RE-FU-008 — Propuesta 2: Impacto de Backend

| Campo                     | Valor                                                 |
| ------------------------- | ----------------------------------------------------- |
| **Requisito**             | R16A-RE-FU-008                                        |
| **Propuesta**             | 2 — Mailbot Worker .NET 10 + Gmail Push Notifications |
| **Repositorio principal** | ProquifaDotNet (.NET Framework 4.8)                   |
| **Repositorio nuevo**     | MailbotWorker (Solución nueva — .NET 10)              |
| **Versión**               | 1.0                                                   |
| **Fecha**                 | 2026-06-02                                            |

---

## Resumen Ejecutivo

La Propuesta 2 genera impacto en **dos repositorios distintos**:

| Repositorio               | Tipo de impacto | Descripción |
|-------------|-----------------|-------------|
| **ProquifaDotNet**        | Cambios en existente | BD, lógica de negocio y endpoints del Buzón de Cobros en PQF2 |
| **MailbotWorker** (nuevo) | Solución nueva .NET 10 | Worker + API que reemplaza el Mailbot actual (Framework 4.8 + Tarea de Windows) |

---

## PARTE 1 — Impacto en ProquifaDotNet

### 1.1 Base de Datos (ProquifaDotNet — SQL Server RYNL010)

#### Cambios en tablas existentes

| # | Objeto | Tipo de cambio | Prioridad |
|:-:|--------|----------------|:---------:|
| 1 | `catProceso` | INSERT — nuevo registro `Cobros` (clave `cobros`) | Alta |
| 2 | `catClasificacionCorreoRecibido` | INSERT nuevo `Cobro` (clave `cobro`) **o** UPDATE renombrar `Pago` a `Cobro` | Alta |
| 3 | `RegionConfiguracionMailBot` | ALTER TABLE — agregar columnas `PubSubTopic`, `WatchHistoryId`, `WatchExpiration` | Alta |

> **Decisión pendiente (clasificación Cobro/Pago):** Ejecutar la siguiente query antes de decidir entre Opción A (renombrar) u Opción B (nuevo registro):
> ```sql
> SELECT COUNT(*) FROM CorreoRecibidoCliente crc
> INNER JOIN catClasificacionCorreoRecibido cat
>     ON crc.IdCatClasificacionCorreoRecibido = cat.IdCatClasificacionCorreoRecibido
> WHERE cat.Clave = 'pago'
> ```

#### Tablas nuevas

| # | Tabla | Propósito | Prioridad |
|:-:|-------|-----------|:---------:|
| 4 | `MailbotEventoCorreo` | Registro de cada notificación Pub/Sub recibida. Trazabilidad e idempotencia por `IdentificadorCorreoGmail` | Alta |
| 5 | `MailbotClasificacionLog` | Auditoría de clasificaciones del Agente IA para reentrenamiento | Alta |

> Las tablas `CorreoRecibido`, `CorreoRecibidoCliente`, `CorreoRecibidoEstatus`, `ArchivoCorreoRecibido`, `fccFolioPagoCliente`, `fccPagoCliente`, `ClienteCartera`, `ClienteCarteraCliente` **no requieren cambios estructurales** — se utilizan tal como están.

---

### 1.2 Lógica de Negocio — ProquifaDotNet

#### Capa `Logic.Pqf.Catalogos` (existente)

|  #  | Archivo sugerido                                       | Tipo  | Descripción                                                                                              |
| :-: | ------------------------------------------------------ | ----- | -------------------------------------------------------------------------------------------------------- |
|  1  | `Buzones\Cobros\BuzonCobrosBO.cs`                      | NUEVO | Lista paginada del Buzón de Cobros filtrada por gestor (`IdUsuarioCobrador`) y región                    |
|  2  | `Buzones\Cobros\BuzonCobrosBO.Reclasificar.cs`         | NUEVO | Reclasificación manual: UPDATE `CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido` al buzón destino |
|  3  | `Buzones\Cobros\Models\BuzonCobrosDetalle.cs`          | NUEVO | DTO de fila del Buzón (correo, cliente, región, folio, estado de lectura)                                |
|  4  | `Buzones\Cobros\Models\GMReclasificarCorreo.cs`        | NUEVO | Modelo de entrada para la acción de reclasificación                                                      |
|  5  | `Cobros\FolioPagoCliente\FolioPagoClienteBO.cs`        | NUEVO | Generación automática de `fccFolioPagoCliente` al clasificar correo como cobro                           |
|  6  | `Cobros\FolioPagoCliente\FolioPagoClienteBO.Cierre.cs` | NUEVO | Cierre automático (`Activo = 0`) al vincular cobro a proforma/factura en Validar Cobro                   |

> Verificar si `FolioPagoClienteBO` ya existe en el proyecto con otra responsabilidad antes de crear los archivos.

#### Capa `_Data\Mail` (existente)

| # | Archivo | Tipo | Descripción |
|:-:|---------|------|-------------|
| 1 | `ProquifaDotNetContext\ProquifaDotNetModel.cs` | Existente — AMPLIAR | Agregar entidades `MailbotEventoCorreo` y `MailbotClasificacionLog` al modelo EF6 |
| 2 | `ProquifaDotNetContext\ProquifaDotNetModel.Context.cs` | Existente — AMPLIAR | Agregar `DbSet<MailbotEventoCorreo>` y `DbSet<MailbotClasificacionLog>` al contexto EF6 |

---

### 1.3 API — ProquifaDotNet

Endpoints nuevos en `WebApi.Catalogos` siguiendo el patrón de Buzones preexistentes (Buzón de Requisiciones, Buzón de Pedidos):

| # | Método | Ruta sugerida | Descripción | Clave Catálogo |
|:-:|--------|--------------|-------------|---------------|
| 1 | GET | `/api/buzon/cobros` | Lista paginada del Buzón filtrada por gestor autenticado y región. Mismos filtros que Buzones existentes | `LIST-PAG-MULT-FILTER` |
| 2 | PUT | `/api/buzon/cobros/{id}/reclasificar` | Mueve el correo a otro buzón (destino seleccionado por el Gestor) | `SERV-SIMPLE-PUT` |
| 3 | PUT | `/api/cobros/folio/{id}/cerrar` | Cierra el pendiente del Buzón al vincular cobro a proforma/factura (invocado desde Validar Cobro) | `SERV-SIMPLE-PUT` |

---

### 1.4 SP Legacy — Evaluación Requerida

| Objeto | Acción requerida |
|--------|-----------------|
| `spActualizarBuzonPagoLegacy` | Evaluar si el SP usa la clave `pago` directamente. Si se renombra a `cobro` (Opción A), el SP puede quedar inconsistente. Confirmar con el equipo antes de ejecutar el script de migración. |

---

## PARTE 2 — Nueva Solución MailbotWorker (.NET 10)

Esta solución es **completamente nueva** y **no forma parte del repositorio ProquifaDotNet**. Reemplaza el Mailbot actual (Framework 4.8 + Tarea de Windows).

### 2.1 Estructura de la Solución

```
MailbotWorker.sln
|
+-- Mailbot.Api              <- Minimal API (.NET 10) — receptor de notificaciones Pub/Sub
+-- Mailbot.Worker           <- Worker Service (.NET 10) — procesamiento de correos
+-- Mailbot.Application      <- Casos de uso y DTOs
+-- Mailbot.Domain           <- Entidades e interfaces
+-- Mailbot.Infrastructure   <- Gmail API, OpenAI, EF Core, Channels, MinIO, Brevo
+-- Mailbot.Tests            <- Pruebas unitarias e integración
```

### 2.2 Componentes y Responsabilidades

| Proyecto | Componente | Descripción |
|----------|-----------|-------------|
| `Mailbot.Api` | `WebhookGmailEndpoints.cs` | `POST /webhook/gmail/{region}` — recibe push de Pub/Sub, valida token, INSERT `MailbotEventoCorreo`, publica en Channel interno |
| `Mailbot.Api` | `PubSubTokenValidationMiddleware.cs` | Valida Bearer token de Google en cada push recibido |
| `Mailbot.Api` | `HealthEndpoints.cs` | `GET /health`, `GET /metrics` |
| `Mailbot.Api` | `ReentrenamientoEndpoints.cs` | `POST /reentrenamiento/trigger` — dispara reentrenamiento del Agente IA |
| `Mailbot.Worker` | `CorreoWorkerMex.cs` | `IHostedService` — consume Channel interno, procesa correos región MEX |
| `Mailbot.Worker` | `CorreoWorkerPer.cs` | `IHostedService` — consume Channel interno, procesa correos región PER |
| `Mailbot.Worker` | `GmailWatchRenovacionWorker.cs` | Renueva el watch de Gmail cada 6 días automáticamente |
| `Mailbot.Application` | `ProcesarCorreoUseCase.cs` | Orquesta: leer correo → clasificar con IA → persistir → generar pendiente si es cobro |
| `Mailbot.Application` | `GenerarPendienteUseCase.cs` | Crea `fccFolioPagoCliente` vía llamada a API de ProquifaDotNet |
| `Mailbot.Application` | `RenovarGmailWatchUseCase.cs` | Llama `Gmail API watch.renew` y actualiza `RegionConfiguracionMailBot` |
| `Mailbot.Domain` | `IClasificadorAgente.cs` | Contrato del Agente IA (clasificación + extracción de datos) |
| `Mailbot.Domain` | `ICorreoRepository.cs` | Contrato de persistencia de correos |
| `Mailbot.Domain` | `IEventoChannel.cs` | Contrato del Channel interno |
| `Mailbot.Infrastructure` | `GmailService.cs` | Lee correo completo vía `Gmail API history.list` |
| `Mailbot.Infrastructure` | `GmailWatchService.cs` | Gestiona `watch.create` y `watch.stop` por región |
| `Mailbot.Infrastructure` | `EventoCorreoChannel.cs` | Implementa `IEventoChannel` con `System.Threading.Channels.Channel<T>` |
| `Mailbot.Infrastructure` | `OpenAIClasificadorAgente.cs` | Implementa `IClasificadorAgente` usando Azure OpenAI / OpenAI |
| `Mailbot.Infrastructure` | `PromptBuilder.cs` | Construye prompts desde archivos `.txt` editables sin recompilar |
| `Mailbot.Infrastructure` | `ProquifaDbContext.cs` | EF Core Scaffold de ProquifaDotNet — tablas requeridas |
| `Mailbot.Infrastructure` | `CorreoRepository.cs` | Implementa `ICorreoRepository` |
| `Mailbot.Infrastructure` | `PendienteRepository.cs` | Persistencia de `fccFolioPagoCliente` |
| `Mailbot.Infrastructure` | `MinioService.cs` | Guarda/lee archivos adjuntos de correos en MinIO |
| `Mailbot.Infrastructure` | `BrevoNotificacionService.cs` | Envía correos de soporte/error al equipo operativo vía Brevo |

### 2.3 Prompts del Agente IA (editables sin recompilar)

```
Mailbot.Infrastructure/Prompts/
  clasificacion_correo.txt    <- Clasificar en: Cotizacion, Pedido, Cobro, Otros
  extraccion_cobro.txt        <- Extraer: monto, moneda, banco, referencia bancaria, fecha de pago
  extraccion_pedido.txt       <- Extraer: productos, cantidades, cliente
  extraccion_cotizacion.txt   <- Extraer: productos solicitados, urgencia
```

> Los prompts son archivos de texto editables. Cualquier ajuste de precision del Agente se realiza modificando estos archivos **sin recompilar** (Regla 2 del requisito: clasificacion basada en entrenamiento, sin criterios configurables en interfaz).

### 2.4 Scaffold EF Core — Tablas ProquifaDotNet requeridas

```bash
dotnet ef dbcontext scaffold "Server=RYNL010;Database=ProquifaDotNet;..." \
  Microsoft.EntityFrameworkCore.SqlServer \
  --output-dir Persistence/Entities \
  --context ProquifaDbContext \
  --tables \
    RegionConfiguracionMailBot \
    CorreoRecibido \
    CorreoRecibidoContenido \
    CorreoRecibidoCliente \
    CorreoRecibidoEstatus \
    ArchivoCorreoRecibido \
    catClasificacionCorreoRecibido \
    catProceso \
    fccFolioPagoCliente \
    Region \
    MailbotEventoCorreo \
    MailbotClasificacionLog \
  --no-onconfiguring --force
```

### 2.5 Integraciones Externas

| Integracion | Uso | Componente |
|-------------|-----|-----------|
| Gmail API | Suscripcion push via `watch`, lectura de correos via `history.list` | `GmailService`, `GmailWatchService` |
| Google Cloud Pub/Sub | Canal de entrega de notificaciones push por region (MEX/PER) + DLQ | `Mailbot.Api` WebhookEndpoints |
| Azure OpenAI / OpenAI | Clasificacion y extraccion de datos del correo con IA | `OpenAIClasificadorAgente` |
| MinIO | Almacenamiento de archivos adjuntos de correos | `MinioService` |
| Brevo | Notificaciones de error y soporte al equipo operativo | `BrevoNotificacionService` |
| SQL Server (ProquifaDotNet) | Persistencia de correos, clasificaciones y pendientes | `ProquifaDbContext`, Repositories |

### 2.6 Flujo de Procesamiento de Cobro (end-to-end)

```
1. Gmail recibe correo en INBOX MEX/PER
2. Gmail push -> Google Cloud Pub/Sub (Topic: mailbot-mex / mailbot-per)
3. Pub/Sub push HTTP -> Mailbot.Api POST /webhook/gmail/{region}
4. Mailbot.Api:
   a. Valida Bearer token de Google (PubSubTokenValidationMiddleware)
   b. Decodifica payload base64 -> { emailAddress, historyId }
   c. INSERT MailbotEventoCorreo (Procesado = 0, Intentos = 0)
   d. Publica EventoCorreoDto en Channel interno (IEventoChannel)
   e. Responde 200 OK -> Pub/Sub hace ACK y elimina el mensaje
5. CorreoWorkerMex/Per consume EventoCorreoDto del Channel interno
6. ProcesarCorreoUseCase:
   a. GmailService.ObtenerCorreo(historyId) -> correo completo (asunto, cuerpo, adjuntos)
   b. OpenAIClasificadorAgente.Clasificar(correo) -> { clasificacion, confianza, datosExtraidos }
   c. INSERT CorreoRecibido + CorreoRecibidoCliente + ArchivoCorreoRecibido
   d. INSERT MailbotClasificacionLog (ClasificacionIA, Confianza, ModeloVersion, tokens)
   e. Si clasificacion == 'cobro':
      -> GenerarPendienteUseCase -> INSERT fccFolioPagoCliente
      -> SubtotalMailBot, IvaMailBot, TotalMailBot, MxnMailBot, UsdMailBot pre-cargados desde extraccion_cobro.txt
   f. Si adjuntos -> MinioService.GuardarArchivo(adjunto)
   g. UPDATE MailbotEventoCorreo SET Procesado = 1, FechaProcesado = GETDATE()
7. Si error -> backoff exponencial (5s, 10s, 20s) -> NACK -> Pub/Sub DLQ (mailbot-dlq)
   -> UPDATE MailbotEventoCorreo SET Error = mensaje_error
   -> BrevoNotificacionService.EnviarAlerta() al equipo de operaciones
```

---

## PARTE 3 — Resumen de Tareas BackEnd

### Tareas en ProquifaDotNet

|  #  | Tarea                                                                                                                                    | Clave Catalogo         | Proyecto              |
| :-: | ---------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- | --------------------- |
| T1  | Script BD: INSERT catProceso Cobros + INSERT/UPDATE catClasificacionCorreoRecibido Cobro + ALTER RegionConfiguracionMailBot (3 columnas) | `UPDATE-TABL-CH`       | BD ProquifaDotNet     |
| T2  | Script BD: CREATE TABLE MailbotEventoCorreo + indices                                                                                    | `CREATE-TABL-CH`       | BD ProquifaDotNet     |
| T3  | Script BD: CREATE TABLE MailbotClasificacionLog                                                                                          | `CREATE-TABL-CH`       | BD ProquifaDotNet     |
| T4  | Ampliar modelo EF6: entidades MailbotEventoCorreo y MailbotClasificacionLog + DbSets en contexto                                         | `UPDATE-TABL-CH`       | `_Data\Mail`          |
| T5  | Crear BuzonCobrosBO.cs — lista paginada del Buzon de Cobros (filtro gestor + region)                                                     | `LIST-PAG-MULT-FILTER` | `Logic.Pqf.Catalogos` |
| T6  | Crear BuzonCobrosBO.Reclasificar.cs — reclasificacion manual de correo                                                                   | `SERV-SIMPLE-PUT`      | `Logic.Pqf.Catalogos` |
| T7  | Crear FolioPagoClienteBO.cs — generacion automatica de pendiente en Validar Cobro                                                        | `ALG-BASIC-LOGIC`      | `Logic.Pqf.Catalogos` |
| T8  | Crear FolioPagoClienteBO.Cierre.cs — cierre/eliminacion automatica del pendiente                                                         | `ALG-BASIC-LOGIC`      | `Logic.Pqf.Catalogos` |
| T9  | Crear endpoint GET /api/buzon/cobros — lista paginada + filtros                                                                          | `LIST-PAG-MULT-FILTER` | `WebApi.Catalogos`    |
| T10 | Crear endpoint PUT /api/buzon/cobros/{id}/reclasificar                                                                                   | `SERV-SIMPLE-PUT`      | `WebApi.Catalogos`    |
| T11 | Crear endpoint PUT /api/cobros/folio/{id}/cerrar                                                                                         | `SERV-SIMPLE-PUT`      | `WebApi.Catalogos`    |
| T12 | Evaluar y actualizar spActualizarBuzonPagoLegacy si aplica                                                                               | `BD-OBJ-CH`            | BD ProquifaDotNet     |
| T25 | Crear BuzonCotizacionesBO — lista paginada del Buzón de Cotizaciones (filtro gestor + región)                                            | `LIST-PAG-MULT-FILTER` | `Logic.Pqf.Catalogos` |
| T26 | Crear BuzonCotizacionesBO.Reclasificar — reclasificación manual de correo de cotización                                                  | `SERV-SIMPLE-PUT`      | `Logic.Pqf.Catalogos` |
| T27 | Crear endpoint GET /api/buzon/cotizaciones — lista paginada con filtros                                                                  | `LIST-PAG-MULT-FILTER` | `WebApi.Catalogos`    |
| T28 | Crear endpoint PUT /api/buzon/cotizaciones/{id}/reclasificar                                                                             | `SERV-SIMPLE-PUT`      | `WebApi.Catalogos`    |
| T29 | Crear BuzonPedidosBO — lista paginada del Buzón de Pedidos (filtro agente + región)                                                      | `LIST-PAG-MULT-FILTER` | `Logic.Pqf.Catalogos` |
| T30 | Crear BuzonPedidosBO.Reclasificar — reclasificación manual de correo de pedido                                                           | `SERV-SIMPLE-PUT`      | `Logic.Pqf.Catalogos` |
| T31 | Crear endpoint GET /api/buzon/pedidos — lista paginada con filtros                                                                       | `LIST-PAG-MULT-FILTER` | `WebApi.Catalogos`    |
| T32 | Crear endpoint PUT /api/buzon/pedidos/{id}/reclasificar                                                                                  | `SERV-SIMPLE-PUT`      | `WebApi.Catalogos`    |

### Tareas en MailbotWorker (Nueva Solucion .NET 10)

| # | Tarea | Clave Catalogo | Proyecto |
|:-:|-------|---------------|---------|
| T13 | Crear solucion MailbotWorker.sln con 6 proyectos: Api, Worker, Application, Domain, Infrastructure, Tests | `ARQ-PROJ-NET` | MailbotWorker |
| T14 | Implementar Mailbot.Domain: entidades + interfaces (IClasificadorAgente, ICorreoRepository, IEventoChannel) | `ARQ-PROJ-NET` | `Mailbot.Domain` |
| T15 | Implementar Mailbot.Infrastructure: GmailService, GmailWatchService, EventoCorreoChannel, OpenAIClasificadorAgente, PromptBuilder, Prompts .txt | `IMPL-THIRD-SERV` | `Mailbot.Infrastructure` |
| T16 | Scaffold EF Core de ProquifaDotNet en Mailbot.Infrastructure\Persistence | `QUERY-CH` | `Mailbot.Infrastructure` |
| T17 | Implementar MinioService — guardar/leer archivos adjuntos de correos | `IMPL-THIRD-SERV` | `Mailbot.Infrastructure` |
| T18 | Implementar BrevoNotificacionService — alertas de error y soporte | `SIMPLE-EMAIL` | `Mailbot.Infrastructure` |
| T19 | Implementar Mailbot.Application: ProcesarCorreoUseCase, GenerarPendienteUseCase, RenovarGmailWatchUseCase, DTOs | `ALG-COMPLX-LOGIC` | `Mailbot.Application` |
| T20 | Implementar Mailbot.Api: Endpoints webhook + middleware validacion token + health + reentrenamiento | `SERV-COMPLEX-TRANSACT` | `Mailbot.Api` |
| T21 | Implementar Mailbot.Worker: CorreoWorkerMex, CorreoWorkerPer, GmailWatchRenovacionWorker, Program.cs con DI | `AUTOMATIC-JOB` | `Mailbot.Worker` |
| T22 | Configurar Google Cloud Pub/Sub: Topics, Subscriptions Push, Dead Letter Topic, permisos IAM | `SERVER-AMB` | Infraestructura GCP |
| T23 | Configurar Gmail API watch por region (MEX/PER) con OAuth2 + gestion de secretos (Key Vault / env vars) | `CONFIG-GMAIL` | `Mailbot.Api` / `Mailbot.Worker` |
| T24 | Escribir pruebas: ProcesarCorreoUseCaseTests, ClasificadorAgenteTests, WebhookGmailEndpointTests, GmailWatchServiceTests, ProquifaDbContextTests | `LIST-NO-FILTER` | `Mailbot.Tests` |
| T33 | Actualizar GenerarPendienteUseCase — implementar cases cotizacion y pedido + actualizar ProcesarCorreoUseCase | `IMP-EXIST-SERVICE` | `Mailbot.Application` |

---

## PARTE 4 — Gaps y Decisiones Pendientes

| # | Gap | Descripcion | Accion requerida | Responsable |
|:-:|-----|-------------|-----------------|-------------|
| 1 | Decision Cobro vs Pago en catClasificacionCorreoRecibido | La clave `pago` ya existe con AnalistaDeCuentasPorCobrar = 1 — equivalente actual de Cobro | Ejecutar query de validacion de registros activos; decidir Opcion A (renombrar) o Opcion B (nuevo registro) | Arquitecto + Cliente |
| 2 | spActualizarBuzonPagoLegacy | Si se renombra la clave `pago` a `cobro`, el SP puede quedar inconsistente | Revisar codigo del SP antes de ejecutar script de migracion | Dev BackEnd |
| 3 | Cuenta GCP y permisos IAM | Definir proyecto GCP, service account y roles pubsub.publisher en los topics MEX y PER | Confirmar con IT / DevOps | IT |
| 4 | Endpoint publico HTTPS para Mailbot.Api | Pub/Sub requiere endpoint HTTPS accesible desde internet para el push | Definir dominio (ej. mailbot.proquifa.mx) + certificado SSL | Infraestructura |
| 5 | Modelo IA: Azure OpenAI vs OpenAI vs modelo propio | Evaluar costo, privacidad de datos y latencia segun volumen estimado de correos | Decision de arquitectura | Arquitecto |
| 6 | Correos de cobro sin cliente identificado | El correo se clasifica como cobro pero el Mailbot no puede identificar al cliente (Riesgo 2 del requisito) | Confirmar con negocio: quien lo visualiza, buzon generico? | Cliente / Negocio |
| 7 | Credenciales Gmail por region (MEX/PER) | ~~Definir cuentas de correo monitoreadas~~ **Cerrado — DUDA-119 (2026-08-21):** el Mailbot debe soportar mas de un correo por region; MEX requiere `ventas@proquifa.com.mx` (existente) + `credito@proquifa.net` (Buzon de Cobros, nuevo) | Confirmar cuentas con IT y watch/topic OAuth2 por cada correo (no solo por region) | IT |
| 8 | Prompt base del Agente IA | Los prompts de clasificacion y extraccion por tipo deben redactarse con el equipo de negocio | Sesion de trabajo negocio + Dev IA | Negocio + Dev IA |
| 9 | Renovacion automatica del watch | GmailWatchRenovacionWorker renueva cada 6 dias; si falla, el watch expira y el Mailbot queda ciego | Definir alerta (Brevo) y monitoreo de WatchExpiration en RegionConfiguracionMailBot | Dev BackEnd |
| 10 | Persistencia de archivos en MinIO | Definir bucket, politica de retencion y naming convention para adjuntos de correos | Confirmar con Arquitecto | Arquitecto |

---

## PARTE 5 — Seguridad

| Elemento | Recomendacion |
|----------|--------------|
| ApiKey (Azure OpenAI) | Azure Key Vault o variable de entorno del host. Nunca en appsettings.json ni en el repositorio |
| PushValidationToken (Google Pub/Sub) | Idem — variable de entorno o secret manager |
| ConnectionString ProquifaDotNet | Idem — nunca en repositorio |
| Credenciales Gmail OAuth2 | Archivos gmail_credentials.json y gmail_token/ fuera del repositorio, montados como secretos en el host |
| Bearer token de Google (push webhook) | Validado en PubSubTokenValidationMiddleware en cada request al webhook |

---

## PARTE 6 — Relacion con el Repositorio ProquifaDotNet Actual

| Elemento actual | Estado | Relacion con Propuesta 2 |
|-----------------|--------|-------------------------|
| `_Data\Mail\SincronizarCorreosBO.cs` | Sin cambios | Sigue sincronizando correos enviados via SendInBlue. El Mailbot nuevo procesa correos recibidos. |
| `_Ejecutables\_Workers\_Worker.SendInBlue` | Sin cambios | Sigue activo para envio de correos. La nueva solucion es independiente. |
| `Logic.Pqf.Catalogos\ProquifaDotNetContext` | Referencia | El modelo EF6 existente es la referencia de estructura. El scaffold EF Core en Mailbot.Infrastructure es independiente. |
| `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraBO.cs` | Sin cambios | Se referencia para el filtro de bandeja por IdUsuarioCobrador. No se modifica. |
| `WebApi.Catalogos` | Se agregan 3 endpoints | Endpoints del Buzon de Cobros siguiendo el patron de Buzones preexistentes. |
