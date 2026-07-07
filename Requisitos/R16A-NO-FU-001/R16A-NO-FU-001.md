# R16A-NO-FU-001 — Migración de Envío de Correo a ProquifaDotNet.SendInBlue

**Tipo:** No Funcional  
**Estado:** Análisis  
**Fecha:** 2026-06-09  

---

## 1. Objetivo

Migrar la infraestructura de envío de correo electrónico desde el Windows Service `_Worker.SendInBlue` (.NET 4.8) hacia una nueva solución independiente denominada **ProquifaDotNet.SendInBlue** (.NET Core 10), modernizando la arquitectura con CQRS, RabbitMQ, IdentityServer y soporte multi-región, sin afectar la operación actual de ProquifaDotNet.

---

## 2. Estado Actual — Análisis del Sistema Existente

### 2.1 Componentes involucrados en el envío de correo

El sistema actual distribuye la responsabilidad de envío de correo en tres ejecutables y múltiples librerías de lógica:

| Componente                            | Tipo            | Tecnología | Responsabilidad                                  |
| ------------------------------------- | --------------- | ---------- | ------------------------------------------------ |
| `_Worker.SendInBlue`                  | Windows Service | .NET 4.8   | Worker RabbitMQ — procesa mensajes de envío      |
| `_Ejecutable.SincronizadorSendInBlue` | Console App     | .NET 4.8   | Sincroniza estado de entrega desde Brevo         |
| `WebApi.Catalogos`                    | Web API         | .NET 4.8   | Expone endpoint `PATCH /EnviarCorreo`            |
| `Logic.Pqf.Mail`                      | Librería        | .NET 4.8   | `EnviarCorreoSendInBlueBO`, extensiones, proxies |
| `Logic.Pqf.Catalogos`                 | Librería        | .NET 4.8   | `CorreoEnviadoBO`, `CorreoGenericoBO`            |
| `Logic.MailXslt`                      | Librería        | .NET 4.8   | Plantillas XSLT por tipo de correo               |
| `Core.Pqf.SendInblue`                 | Librería        | .NET 4.8   | Modelos y servicio HTTP hacia Brevo              |

### 2.2 Flujo de envío de correo (flujo principal)

```
[Business Logic]
    │
    ├─ 1. Crea registro CorreoEnviado en BD (con ReceptoresCSV, Asunto, IdRegion)
    ├─ 2. Adjunta archivos en ArchivoCorreoEnviado (FK → Archivo en MinIO)
    └─ 3. Llama PATCH /EnviarCorreo?idCorreoEnviado={guid}
              │
              ▼
    [WebApi.Catalogos — CorreoEnviadoEnviarController]
         Encola SendInBlueMessage { IdCorreoEnviado } en RabbitMQ
              │
              ▼
    [_Worker.SendInBlue — SendInBlueWorker]
    Extiende ProquifaDotNetRabbitMQWorkerService<SendInBlueMessage>
              │
    ┌─────────┴──────────────────────────────────────────────┐
    │ Mensaje con IdCFDI                                     │
    │  → Espera que cfdi.IdArchivoTimbre != null (hasta 20h) │
    │  → Crea CorreoEnviado de Factura                       │
    │  → Adjunta XML timbre + PDF factura desde MinIO        │
    │  → Re-encola mensaje con IdCorreoEnviado               │
    └──────────────────────┬─────────────────────────────────┘
                           │ Mensaje con IdCorreoEnviado
                           ▼
    [EnviarCorreoSendInBlueBO.Procesar]
         │
         ├─ Obtiene ConfiguracionSendinBlueBO(IdRegion)
         │    → URL API, ClaveAPI, CorreoEmisor (por región)
         │
         ├─ Reflection: Relaciones.ObtenerTiposConRelacion(typeof(CorreoEnviado))
         │    → Encuentra todas las tablas relacionadas (cotización, pedido, etc.)
         │
         ├─ Parallel.ForEach sobre tipos relacionados:
         │    → Query: WHERE IdCorreoEnviado = {guid} AND Activo = true
         │    → correoEnviado.AsSendinBlueMails(objeto)
         │         → GeneradorHtmlCorreo.ObtenerGeneradoresParaCorreo(objeto)
         │              → Selecciona plantilla XSLT según tipo de objeto
         │         → Renderiza HTML (con soporte regional MX/PE para cotización)
         │         → Descarga adjuntos de MinIO a ruta temp local
         │
         ├─ SendInBlueProxy(IdRegion).Process(mail)
         │    → SendInBlueMailService.SendMail(MailRequest)
         │    → HTTP POST a Brevo API
         │
         └─ Actualiza CorreoEnviado:
              FechaEnvio = Now
              IdentificadorCorreo = response.messageId
              GuardarOActualizar(correoEnviado)
```

### 2.3 Flujo de correo genérico (CorreoGenericoBO)

Utilizado para notificaciones internas (clientes, usuarios, autorizaciones). **Envío directo, sin RabbitMQ:**

```
[Business Logic]
    └─ CorreoGenericoBO.GenerarCorreo<T>(ParametrosCorreoGenerico<T>)
         ├─ Valida configuración Brevo (URL, API key, sender)
         ├─ Valida receptores, CC, asunto, nombre de plantilla
         ├─ Lee plantilla HTML desde disco (Plantillas:HTML config)
         │    → Sustituye {{parametro}} con valores dinámicos
         │    → O: serializa Data a XML + transforma con XSLT
         ├─ SendInBlueMailService.SendMail(MailRequest)
         └─ Retorna bool (Enviado)
```

### 2.4 Flujo de sincronización de estado

El ejecutable `_Ejecutable.SincronizadorSendInBlue` corre periódicamente (tarea programada):

```
SincronizarCorreosBO.ConsumirServicioExterno()
    └─ CorreoEnviadoBO.ObtenerCorreosPendientesSincronizar()
         → Activo = true
         → IdentificadorCorreo NOT NULL (ya enviados)
         → FechaLectura = null AND FechaSpam = null
         → FechaRegistro = hoy
    └─ SincronizarCorreoEnviadoBO.Procesar(correoEnviado) — por cada uno
         → Consulta estado en Brevo API por IdentificadorCorreo
         → Actualiza FechaLectura / FechaSpam / FechaRecepcion
```

### 2.5 Tipos de correo soportados (plantillas XSLT)

| Tipo objeto relacionado | Plantilla | Módulo |
|---|---|---|
| `cotCotizacionCorreoEnviado` | `CorreoCotizacion.xslt` / regional | L01 — Cotización |
| `cotInvestigacionCorreoEnviado` | `CorreoCotPartidaInvetigacion` | L01 — Investigación |
| `cotPartidaInvestigacionFinalizadaCorreoEnviado` | `CorreoCotInvestigacionFinalizada` | L01 — Investigación |
| `ppPedidoCorreoEnviado` | `CorreoPretramitacionFEA` | L04 — Pretramitar Pedido |
| `ppPedidoVDCorreoEnviado` | `CorreoPretramitarPedidoVD` | L04 — Pretramitar Pedido |
| `ppPedidoIncidenciaCorreoEnviado` | `CorreoPPPedidoIncidencia` | L04 — Incidencias |
| `ppPedidoOcNoAmparadaCorreoEnviado` | `CorreoPPPedidoOcNoAmparada` | L04 — OC No Amparada |
| `tpPedidoCorreoEnviado` | `CorreoPedidoInterno` | L05 — Tramitar Pedido |
| `tpOCTemporalCorreoEnviado` | `CorreoModPedidoOCTemporal` | L05 — Modificación Pedido |
| `fccPagoClienteCorreoEnviado` | `CorreoFCCPagoCliente` | Cobranza |

### 2.6 Configuración por región

`ConfiguracionSendinBlueBO(Guid? IdRegion)` resuelve por región:

- `UrlEnvioCorreo` — URL endpoint Brevo
- `ClaveAPI` — API key de Brevo
- `CorreoEmisor` — correo remitente (`sender`)

Las regiones soportadas actualmente son **México** y **Perú** (con plantillas XSLT específicas para cotización regional).

---

## 3. Diccionario de Datos — Tablas Relevantes en ProquifaDotNet

### 3.1 Tabla: `CorreoEnviado`

**Descripción:** Registro central de todos los correos electrónicos salientes del sistema. Actúa como cola de trabajo y bitácora de estado de entrega.

| Columna | Tipo | Descripción |
|---|---|---|
| `IdCorreoEnviado` | `uniqueidentifier` PK | Identificador único del correo |
| `IdRegion` | `uniqueidentifier` NULL FK → `Region` | Región desde la que se envía (determina remitente y API key) |
| `Emisor` | `nvarchar` NULL | Correo emisor explícito (override de la configuración) |
| `ReceptoresCSV` | `nvarchar` NOT NULL | Lista de destinatarios separados por coma |
| `ConCopiaCSV` | `nvarchar` NOT NULL | Lista CC separados por coma |
| `Asunto` | `nvarchar` NOT NULL | Asunto del correo |
| `IdentificadorCorreo` | `nvarchar` NULL | `messageId` retornado por Brevo tras envío exitoso |
| `FechaRegistro` | `datetime` NOT NULL | Fecha de creación del registro |
| `FechaUltimaActualizacion` | `datetime` NOT NULL | Última modificación |
| `FechaEnvio` | `datetime` NULL | Fecha en que se procesó el envío |
| `FechaRecepcion` | `datetime` NULL | Fecha de recepción confirmada por Brevo |
| `FechaAceptacion` | `datetime` NULL | Fecha de aceptación por servidor destino |
| `FechaLectura` | `datetime` NULL | Fecha de lectura por el destinatario |
| `FechaSpam` | `datetime` NULL | Fecha en que fue marcado como spam |
| `Activo` | `bit` NOT NULL | Registro activo |

**Relaciones:** `CorreoEnviado` → (1:N) → `ArchivoCorreoEnviado`  
**Relaciones implícitas via reflección:** Todas las tablas `*CorreoEnviado` relacionan con esta tabla por `IdCorreoEnviado`.

### 3.2 Tabla: `ArchivoCorreoEnviado`

**Descripción:** Archivos adjuntos asociados a un correo saliente. Los archivos residen en MinIO y se descargan temporalmente durante el envío.

| Columna | Tipo | Descripción |
|---|---|---|
| `IdArchivoCorreoEnviado` | `uniqueidentifier` PK | Identificador del adjunto |
| `IdCorreoEnviado` | `uniqueidentifier` NOT NULL FK → `CorreoEnviado` | Correo al que pertenece |
| `IdArchivo` | `uniqueidentifier` NOT NULL FK → `Archivo` | Archivo en MinIO |
| `Activo` | `bit` NOT NULL | Registro activo |
| `FechaRegistro` | `datetime` NOT NULL | Fecha de creación |
| `FechaUltimaActualizacion` | `datetime` NOT NULL | Última modificación |

### 3.3 Tablas relacionadas (por módulo)

Cada tabla a continuación tiene FK `IdCorreoEnviado → CorreoEnviado` y proporciona el contexto de negocio para generar el HTML del correo vía XSLT:

| Tabla | Módulo |
|---|---|
| `cotCotizacionCorreoEnviado` | Cotización |
| `cotInvestigacionCorreoEnviado` | Investigación cotización |
| `cotPartidaInvestigacionFinalizadaCorreoEnviado` | Investigación finalizada |
| `ppPedidoCorreoEnviado` | Pretramitar Pedido (FEA) |
| `ppPedidoVDCorreoEnviado` | Pretramitar Pedido VD |
| `ppPedidoIncidenciaCorreoEnviado` | Incidencias pedido |
| `ppPedidoOcNoAmparadaCorreoEnviado` | OC no amparada |
| `tpPedidoCorreoEnviado` | Tramitar Pedido |
| `tpOCTemporalCorreoEnviado` | Modificación de Pedido |
| `fccPagoClienteCorreoEnviado` | Cobranza — Pago cliente |

---

## 4. Impacto en Backend — ProquifaDotNet (.NET 4.8)

### 4.1 Contexto

ProquifaDotNet continuará siendo el sistema de Venta Interna. La nueva solución **ProquifaDotNet.SendInBlue** se integrará como servicio externo mediante **llamadas entre APIs**. La comunicación es: ProquifaDotNet llama al API de ProquifaDotNet.SendInBlue para solicitar el envío.

### 4.2 Cambios requeridos en ProquifaDotNet

#### A. Deprecar `CorreoEnviadoEnviarController` (o migrar)

**Situación actual:** `PATCH /EnviarCorreo` encola directamente en RabbitMQ desde WebApi.Catalogos.  
**Cambio:** Refactorizar para que llame al API de ProquifaDotNet.SendInBlue en lugar de encolar directamente.

```
PATCH /EnviarCorreo?idCorreoEnviado={guid}
    Antes → RabbitMQClientFactory.FabricarMailClientSendInBlue().SendMessage(...)
    Después → HttpClient.PostAsync("https://sendinblue-api/api/v1/mail/send", ...)
```

#### B. Deprecar `CorreoGenericoBO` (o adaptar)

**Situación actual:** Envío sincrónico directo a Brevo dentro de ProquifaDotNet.  
**Cambio:** `CorreoGenericoBO.GenerarCorreo<T>` deberá llamar al nuevo API de ProquifaDotNet.SendInBlue en lugar de invocar `SendInBlueMailService` directamente.

Existen llamadas a `CorreoGenericoBO` en los siguientes módulos que deben migrar:
- `ClienteBO.Extensions` — correos de cliente
- `GMUsuarioClienteCarteraDetalleBO` — notificaciones de cartera
- `AutorizacionBO` — correos de autorización
- `cotCotizacionCorreoEnviadoTransaccionBO` — cotización (flujo alternativo)
- `CorreoCotPartidaInevtigacionBO` — investigación
- `cotPartidaInvetigacionCorreoFinalizadoBO` — investigación finalizada
- `PretramitarPromesaDeCompraTransaccionBO` — promesa de compra
- `ppPedidoIncidenciaCorreoTransaccionBO` — incidencias
- `ppPedidosSolicitarFEATransaccionBO` — FEA

#### C. Deprecar `_Worker.SendInBlue`

El Windows Service completo será reemplazado por **Worker.SendMail** en la nueva solución. Una vez migrada y validada la nueva solución, el servicio antiguo puede desactivarse.

#### D. Deprecar `_Ejecutable.SincronizadorSendInBlue`

La sincronización de estado de entrega pasará a ser responsabilidad del **Worker.SendMail** de la nueva solución.

#### E. Mantener tabla `CorreoEnviado` y relacionadas

Las tablas `CorreoEnviado` y `ArchivoCorreoEnviado` en ProquifaDotNet **no cambian**. La nueva solución las leerá vía Scaffold (EF Core) para obtener datos al procesar el envío.

### 4.3 Resumen de impacto

| Componente | Acción |
|---|---|
| `_Worker.SendInBlue` (Windows Service) | **Deprecar** — reemplazado por Worker.SendMail |
| `_Ejecutable.SincronizadorSendInBlue` | **Deprecar** — migrado a Worker.SendMail |
| `CorreoEnviadoEnviarController` | **Refactorizar** — llamar API SendInBlue en lugar de RabbitMQ directo |
| `CorreoGenericoBO` | **Refactorizar** — llamar API SendInBlue en lugar de Brevo directo |
| `EnviarCorreoSendInBlueBO` | **Deprecar** — lógica migrada a Application layer nueva solución |
| `SendInBlueProxy` / `ConfiguracionSendinBlueBO` | **Deprecar** — migrado a Infrastructure nueva solución |
| `Logic.MailXslt` (plantillas XSLT) | **Migrar** — copiar a nueva solución o acceder via path compartido |
| Tablas `CorreoEnviado`, `ArchivoCorreoEnviado` | **Sin cambios estructurales** — Scaffold en nueva solución |
| Tablas `*CorreoEnviado` (por módulo) | **Sin cambios estructurales** — Scaffold en nueva solución |

---

## 5. Impacto en Base de Datos

### 5.1 ProquifaDotNet (base existente)

**Sin cambios estructurales en tablas existentes.**

Se recomienda agregar los siguientes índices para optimizar las consultas frecuentes de la nueva solución:

```sql
-- Índice para consulta de correos pendientes de sincronización
CREATE NONCLUSTERED INDEX [IX_CorreoEnviado_Sincronizacion]
ON [dbo].[CorreoEnviado] ([Activo], [FechaRegistro], [FechaLectura], [FechaSpam])
INCLUDE ([IdCorreoEnviado], [IdentificadorCorreo], [IdRegion]);

-- Índice para búsqueda por IdentificadorCorreo (messageId de Brevo)
CREATE NONCLUSTERED INDEX [IX_CorreoEnviado_IdentificadorCorreo]
ON [dbo].[CorreoEnviado] ([IdentificadorCorreo])
WHERE [IdentificadorCorreo] IS NOT NULL;
```

### 5.2 ProquifaDotNetSendInBlue (nueva base de datos)

Base de datos nueva dedicada exclusivamente a la gestión operativa del servicio de envío. **No comparte tablas con ProquifaDotNet.**

#### Tabla: `ConfiguracionSendInBlue`

**Descripción:** Configuración de credenciales Brevo por región. Reemplaza a `ConfiguracionSendinBlueBO` que actualmente resuelve desde otra fuente.

| Columna | Tipo | Descripción |
|---|---|---|
| `IdConfiguracionSendInBlue` | `uniqueidentifier` PK | Identificador |
| `IdRegion` | `uniqueidentifier` NOT NULL | Región de ProquifaDotNet (FK lógica) |
| `Nombre` | `nvarchar(100)` NOT NULL | Nombre descriptivo de la configuración |
| `UrlEnvioCorreo` | `nvarchar(500)` NOT NULL | URL endpoint Brevo |
| `ClaveAPI` | `nvarchar(500)` NOT NULL | API key de Brevo (cifrada) |
| `CorreoEmisor` | `nvarchar(200)` NOT NULL | Correo remitente por defecto |
| `NombreEmisor` | `nvarchar(200)` NOT NULL | Nombre del remitente |
| `Activo` | `bit` NOT NULL DEFAULT 1 | Configuración activa |
| `FechaRegistro` | `datetime2` NOT NULL DEFAULT GETDATE() | Fecha de creación |
| `FechaUltimaActualizacion` | `datetime2` NOT NULL DEFAULT GETDATE() | Última modificación |

#### Tabla: `SolicitudCorreo`

**Descripción:** Cola de solicitudes de envío recibidas. Desacopla la recepción del procesamiento y permite reintentos.

| Columna | Tipo | Descripción |
|---|---|---|
| `IdSolicitudCorreo` | `uniqueidentifier` PK | Identificador |
| `IdCorreoEnviado` | `uniqueidentifier` NOT NULL | FK lógica a `CorreoEnviado` en ProquifaDotNet |
| `TipoEnvio` | `nvarchar(50)` NOT NULL | `'TEMPLATE'`, `'SIMPLE'`, `'HTML'` |
| `Estado` | `nvarchar(50)` NOT NULL | `'PENDIENTE'`, `'PROCESANDO'`, `'ENVIADO'`, `'ERROR'`, `'CANCELADO'` |
| `Intentos` | `int` NOT NULL DEFAULT 0 | Número de intentos de envío |
| `MaxIntentos` | `int` NOT NULL DEFAULT 3 | Máximo de reintentos configurado |
| `FechaCreacion` | `datetime2` NOT NULL DEFAULT GETDATE() | Fecha de encolado |
| `FechaProximoIntento` | `datetime2` NULL | Fecha programada para próximo reintento |
| `FechaProcesado` | `datetime2` NULL | Fecha de procesamiento exitoso |
| `ErrorUltimoIntento` | `nvarchar(max)` NULL | Mensaje de error del último intento fallido |
| `BrevoMessageId` | `nvarchar(200)` NULL | `messageId` retornado por Brevo |
| `Activo` | `bit` NOT NULL DEFAULT 1 | Registro activo |

**Índices:**

```sql
CREATE NONCLUSTERED INDEX [IX_SolicitudCorreo_Estado_FechaProximoIntento]
ON [dbo].[SolicitudCorreo] ([Estado], [FechaProximoIntento])
WHERE [Estado] IN ('PENDIENTE', 'ERROR');
```

#### Tabla: `BitacoraEnvioCorreo`

**Descripción:** Bitácora detallada de cada intento de envío. Permite auditoría y diagnóstico.

| Columna | Tipo | Descripción |
|---|---|---|
| `IdBitacoraEnvioCorreo` | `uniqueidentifier` PK | Identificador |
| `IdSolicitudCorreo` | `uniqueidentifier` NOT NULL FK → `SolicitudCorreo` | Solicitud asociada |
| `NumeroIntento` | `int` NOT NULL | Número de intento (1-based) |
| `FechaIntento` | `datetime2` NOT NULL DEFAULT GETDATE() | Fecha del intento |
| `Exitoso` | `bit` NOT NULL | Si el envío fue exitoso |
| `HttpStatusCode` | `int` NULL | Código de respuesta HTTP de Brevo |
| `BrevoMessageId` | `nvarchar(200)` NULL | messageId de respuesta |
| `BrevoResponseBody` | `nvarchar(max)` NULL | Body completo de respuesta |
| `ErrorDetalle` | `nvarchar(max)` NULL | Stack trace / detalle de error |
| `DuracionMs` | `int` NULL | Duración del envío en milisegundos |

#### Tabla: `PlantillaCorreo`

**Descripción:** Registro de plantillas HTML disponibles para envío de correos simples y genéricos. Permite administración sin despliegue.

| Columna                    | Tipo                                   | Descripción                                       |
| -------------------------- | -------------------------------------- | ------------------------------------------------- |
| `IdPlantillaCorreo`        | `uniqueidentifier` PK                  | Identificador                                     |
| `Clave`                    | `nvarchar(100)` NOT NULL UNIQUE        | Clave única de la plantilla (ej. `COTIZACION_MX`) |
| `Nombre`                   | `nvarchar(200)` NOT NULL               | Nombre descriptivo                                |
| `Asunto`                   | `nvarchar(500)` NULL                   | Asunto por defecto                                |
| `ContenidoHtml`            | `nvarchar(max)` NOT NULL               | Contenido HTML con marcadores `{{variable}}`      |
| `IdRegion`                 | `uniqueidentifier` NULL                | NULL = global, con valor = específica de región   |
| `Activo`                   | `bit` NOT NULL DEFAULT 1               | Plantilla activa                                  |
| `FechaRegistro`            | `datetime2` NOT NULL DEFAULT GETDATE() | Fecha de creación                                 |
| `FechaUltimaActualizacion` | `datetime2` NOT NULL DEFAULT GETDATE() | Última modificación                               |

#### Tabla: `CatalogoPlantillaBrevo`

**Descripción:** Catálogo de plantillas nativas de Brevo disponibles para envío. Mapea una clave lógica al `IdTemplateBrevo` (entero) de la plataforma Brevo. A diferencia de `PlantillaCorreo`, el HTML es renderizado por Brevo en sus servidores; el servidor solo pasa el ID y los parámetros dinámicos.

| Columna | Tipo | Descripción |
|---|---|---|
| `IdCatalogoPlantillaBrevo` | `uniqueidentifier` PK | Identificador interno |
| `Clave` | `nvarchar(100)` NOT NULL UNIQUE | Clave lógica (e.g. `BIENVENIDA_CLIENTE`) |
| `IdTemplateBrevo` | `int` NOT NULL | ID de plantilla en la plataforma Brevo |
| `Nombre` | `nvarchar(200)` NOT NULL | Nombre descriptivo |
| `Descripcion` | `nvarchar(500)` NOT NULL DEFAULT '' | Descripción del uso de la plantilla |
| `EsquemaParametros` | `nvarchar(MAX)` NOT NULL DEFAULT '{}'  | JSON con los parámetros esperados por la plantilla |
| `IdRegion` | `uniqueidentifier` NULL | Región de la cuenta Brevo (NULL = aplica a todas) |
| `FechaRegistro` | `datetime2` NOT NULL DEFAULT GETDATE() | Fecha de alta |
| `FechaUltimaActualizacion` | `datetime2` NOT NULL DEFAULT GETDATE() | Último timestamp |
| `Activo` | `bit` NOT NULL DEFAULT 1 | Borrado lógico |

**Índices:**
- `UQ_CatalogoPlantillaBrevo_Clave` — único sobre `Clave`
- `IX_CatalogoPlantillaBrevo_IdTemplateBrevo` — non-clustered sobre `IdTemplateBrevo`

---

#### Tabla: `AppSettings`

**Descripción:** Configuración operativa del servicio (equivalente a `appSettings` de .config).

| Columna | Tipo | Descripción |
|---|---|---|
| `IdAppSettings` | `uniqueidentifier` PK | Identificador |
| `Clave` | `nvarchar(200)` NOT NULL UNIQUE | Clave de configuración |
| `Valor` | `nvarchar(max)` NOT NULL | Valor |
| `Descripcion` | `nvarchar(500)` NULL | Descripción del parámetro |
| `FechaUltimaActualizacion` | `datetime2` NOT NULL DEFAULT GETDATE() | Última modificación |

---

## 6. Arquitectura — Nueva Solución ProquifaDotNet.SendInBlue (.NET Core 10)

### 6.1 Estructura de capas

```
ProquifaDotNet.SendInBlue/
├── Domain/
│   ├── Entities/
│   │   ├── SolicitudCorreo.cs
│   │   ├── BitacoraEnvioCorreo.cs
│   │   ├── ConfiguracionSendInBlue.cs
│   │   ├── PlantillaCorreo.cs
│   │   ├── CatalogoPlantillaBrevo.cs
│   │   └── AppSettings.cs
│   ├── Interfaces/
│   │   ├── ISolicitudCorreoRepository.cs
│   │   ├── IBitacoraEnvioCorreoRepository.cs
│   │   ├── IConfiguracionSendInBlueRepository.cs
│   │   ├── IPlantillaCorreoRepository.cs
│   │   ├── ICatalogoPlantillaBrevoRepository.cs
│   │   └── IBrevoMailService.cs
│   └── Enums/
│       ├── TipoEnvioCorreo.cs         (TEMPLATE, SIMPLE, HTML, BREVO_TEMPLATE)
│       └── EstadoSolicitudCorreo.cs   (PENDIENTE, PROCESANDO, ENVIADO, ERROR, CANCELADO)
│
├── Application/
│   ├── Commands/
│   │   ├── EnviarCorreoCommand.cs                 (envío template vía XSLT — flujo principal)
│   │   ├── EnviarCorreoSimpleCommand.cs           (envío simple sin plantilla XSLT)
│   │   ├── EnviarCorreoHtmlCommand.cs             (envío con HTML explícito)
│   │   ├── EnviarCorreoPlantillaBrevoCommand.cs   (envío por plantilla nativa de Brevo)
│   │   └── SincronizarEstadoCorreoCommand.cs
│   ├── Queries/
│   │   ├── ObtenerSolicitudCorreoQuery.cs
│   │   └── ObtenerBitacoraCorreoQuery.cs
│   ├── DTOs/
│   │   ├── SendEmailDto.cs
│   │   ├── EnviarCorreoSimpleDto.cs
│   │   ├── EnviarCorreoHtmlDto.cs
│   │   ├── EnviarCorreoPlantillaBrevoDto.cs
│   │   └── SolicitudCorreoDto.cs
│   └── Validators/
│       ├── EnviarCorreoCommandValidator.cs
│       ├── EnviarCorreoHtmlCommandValidator.cs
│       └── EnviarCorreoPlantillaBrevoCommandValidator.cs
│
├── Infrastructure/
│   ├── Persistence/
│   │   ├── SendInBlueDbContext.cs              (EF Core — ProquifaDotNetSendInBlue)
│   │   ├── ProquifaDotNetScaffoldContext.cs    (EF Core Scaffold — tablas leídas de ProquifaDotNet)
│   │   └── Repositories/
│   │       ├── SolicitudCorreoRepository.cs
│   │       ├── BitacoraEnvioCorreoRepository.cs
│   │       └── ConfiguracionSendInBlueRepository.cs
│   ├── Brevo/
│   │   ├── BrevoMailService.cs              (implementa IBrevoMailService — HTTP a Brevo)
│   │   ├── BrevoMailRequest.cs              (request para envío con HTML/texto)
│   │   ├── BrevoTemplateRequest.cs          (request para envío por templateId Brevo)
│   │   └── BrevoMailResponse.cs
│   ├── Templates/
│   │   ├── XsltTemplateRenderer.cs    (migrado de Logic.MailXslt)
│   │   └── HtmlTemplateRenderer.cs    (marcadores {{variable}})
│   ├── RabbitMQ/
│   │   ├── RabbitMQPublisher.cs
│   │   └── RabbitMQConsumer.cs
│   ├── Identity/
│   │   └── IdentityServerExtensions.cs
│   └── Logging/
│       └── SerilogExtensions.cs
│
├── API/
│   ├── Controllers/
│   │   └── CorreoController.cs        (4 endpoints)
│   └── Program.cs
│
├── Worker.SendMail/
│   ├── Workers/
│   │   ├── SendMailWorker.cs          (procesa cola RabbitMQ)
│   │   └── SincronizacionWorker.cs    (sincroniza estado de entrega)
│   └── Program.cs
│
└── Testing/
    ├── Application.Tests/
    │   ├── Commands/
    │   └── Queries/
    └── Infrastructure.Tests/
        └── Brevo/
```

### 6.2 API — Endpoints

#### `POST /api/v1/mail/send`

Solicita envío de correo basado en `CorreoEnviado` existente en ProquifaDotNet (flujo principal con XSLT).

**Request:**
```json
{
  "idCorreoEnviado": "guid",
  "idProcesoSistema": "guid"
}
```

**Comportamiento:** Encola en RabbitMQ → Worker procesa → resuelve plantilla XSLT → envía a Brevo.

---

#### `POST /api/v1/mail/simple`

Envío simple sin asociación a `CorreoEnviado`. Para notificaciones ad-hoc internas.

**Request:**
```json
{
  "idRegion": "guid",
  "receptores": [{ "nombre": "string", "correo": "string" }],
  "conCopia": [{ "nombre": "string", "correo": "string" }],
  "asunto": "string",
  "clavePlantilla": "string",
  "parametros": [{ "nombre": "string", "valor": "string" }]
}
```

**Comportamiento:** Envío directo (sin cola) o con fallover a RabbitMQ si Brevo no responde.

---

#### `POST /api/v1/mail/html`

Envío con contenido HTML explícito. Para correos donde el caller ya generó el HTML.

**Request:**
```json
{
  "idRegion": "guid",
  "receptores": [{ "nombre": "string", "correo": "string" }],
  "conCopia": [{ "nombre": "string", "correo": "string" }],
  "asunto": "string",
  "htmlContent": "string",
  "adjuntos": [{ "nombre": "string", "urlMinIO": "string" }]
}
```

**Comportamiento:** Envío directo (sin cola) o con failover a RabbitMQ.

---

#### `POST /api/v1/mail/template`

Envío usando una plantilla nativa de Brevo. El HTML es renderizado por Brevo en sus servidores; el servidor solo envía el `IdTemplateBrevo` y los parámetros dinámicos. La plantilla se puede referenciar por `clave` lógica (lookup en `CatalogoPlantillaBrevo`) o por `idTemplateBrevo` directo.

**Request:**
```json
{
  "idRegion": "guid",
  "receptores": [{ "nombre": "string", "correo": "string" }],
  "conCopia": [{ "nombre": "string", "correo": "string" }],
  "clave": "BIENVENIDA_CLIENTE",
  "idTemplateBrevo": null,
  "params": {
    "nombre": "Juan García",
    "empresa": "Proquifa",
    "urlAccion": "https://app.proquifa.mx/login"
  }
}
```

> `clave` o `idTemplateBrevo` — debe estar presente exactamente uno de los dos.

**Comportamiento:** Resuelve `IdTemplateBrevo` si se pasó `clave` (consulta `CatalogoPlantillaBrevo`). Intenta envío directo; si hay timeout → encola en RabbitMQ con `TipoEnvioCorreo.BREVO_TEMPLATE`.

**Response (envío directo):**
```json
{ "brevoMessageId": "string", "enviado": true }
```

**Response (encolado):**
```json
{ "idSolicitudCorreo": "guid", "estado": "PENDIENTE" }
```

---

### 6.3 Worker.SendMail

Procesa dos tipos de trabajo asíncrono:

**SendMailWorker** — escucha cola RabbitMQ `queueSendInBlue`:
1. Recibe `SolicitudCorreoMessage { IdSolicitudCorreo }`
2. Lee `SolicitudCorreo` + `TipoEnvioCorreo`
3. Según tipo de envío:
   - **TEMPLATE**: Lee `CorreoEnviado` (via Scaffold) → resuelve plantilla XSLT → descarga adjuntos MinIO → llama `IBrevoMailService.SendMail`
   - **SIMPLE**: Lee `PlantillaCorreo` por clave → sustituye `{{params}}` → llama `IBrevoMailService.SendMail`
   - **HTML**: Usa `HtmlContent` almacenado en `SolicitudCorreo` → llama `IBrevoMailService.SendMail`
   - **BREVO_TEMPLATE**: Lee `IdTemplateBrevo` + `ParamsJson` de `SolicitudCorreo` → llama `IBrevoMailService.EnviarConPlantillaBrevo(templateId, receptores, params)`
4. Actualiza `SolicitudCorreo.Estado` y `BitacoraEnvioCorreo`
5. Para tipo TEMPLATE: actualiza `CorreoEnviado.IdentificadorCorreo` / `FechaEnvio` en ProquifaDotNet
6. **Reintento:** en caso de fallo, programa `FechaProximoIntento` con backoff exponencial (hasta `MaxIntentos`)
7. **Notificación:** al agotar intentos, notifica vía Brevo o log crítico

**SincronizacionWorker** — ejecuta según cron configurado:
1. Consulta `CorreoEnviado` enviados hoy sin `FechaLectura` (via Scaffold)
2. Consulta estado en Brevo por `IdentificadorCorreo`
3. Actualiza `FechaLectura`, `FechaSpam`, `FechaRecepcion` en ProquifaDotNet

### 6.4 Integración con ProquifaDotNet (Scaffold)

Las tablas de ProquifaDotNet que la nueva solución lee/escribe se agregan al Scaffold en `Infrastructure`:

**Tablas para leer (solo lectura):**
- `CorreoEnviado` — datos del correo a enviar
- `ArchivoCorreoEnviado` — adjuntos
- `Archivo` — metadatos de archivo en MinIO
- `Region` — para resolución de configuración por región
- Todas las tablas `*CorreoEnviado` — contexto para plantillas XSLT

**Tablas para escribir:**
- `CorreoEnviado` — actualizar `IdentificadorCorreo`, `FechaEnvio`, `FechaLectura`, `FechaSpam`, `FechaRecepcion`

### 6.5 Integraciones externas

| Integración | Uso |
|---|---|
| **Brevo (SendInBlue)** | Envío de correos — HTTP REST API |
| **MinIO** | Descarga de archivos adjuntos (`ArchivoDetalle.Url`) |
| **RabbitMQ** | Cola `queueSendInBlue` — desacoplamiento envío async |
| **IdentityServer** | Autenticación de llamadas entre APIs |
| **Serilog** | Logs estructurados (usuario, módulo, operación) |

---

## 7. Consideraciones de Migración

- La migración es **incremental**: la nueva solución puede correr en paralelo con `_Worker.SendInBlue` durante la transición.
- ProquifaDotNet se reconfigura para apuntar al nuevo API (variable de entorno / appSettings) sin cambios de datos.
- Las plantillas XSLT de `Logic.MailXslt` se copian a la nueva solución y se adaptan para funcionar con .NET Core 10.
- `ConfiguracionSendinBlueBO` se migra como datos a la tabla `ConfiguracionSendInBlue` de `ProquifaDotNetSendInBlue`.
- El endpoint `PATCH /EnviarCorreo` de ProquifaDotNet se refactoriza en una sola iteración para todos los módulos.
- El executible `_Ejecutable.SincronizadorSendInBlue` puede desactivarse una vez que `SincronizacionWorker` esté en producción.

---

## 8. Dependencias y Orden de Implementación

1. **Base de datos** — Crear `ProquifaDotNetSendInBlue` con todas sus tablas, incluyendo `CatalogoPlantillaBrevo`
2. **Domain + Application** — Entidades, interfaces, commands/queries, DTOs (incluye `EnviarCorreoPlantillaBrevoCommand`)
3. **Infrastructure** — Scaffold, repositorios, `BrevoMailService` (con `EnviarConPlantillaBrevo`), `XsltTemplateRenderer`
4. **API** — 4 endpoints con autenticación IdentityServer
5. **Worker.SendMail** — `SendMailWorker` (4 tipos de envío) + `SincronizacionWorker`
6. **Refactorizar ProquifaDotNet** — `CorreoEnviadoEnviarController` + `CorreoGenericoBO`
7. **Testing** — Pruebas unitarias e integración
8. **Validación en paralelo** — Correr ambos sistemas y comparar resultados
9. **Deprecación** — Desactivar `_Worker.SendInBlue` y `_Ejecutable.SincronizadorSendInBlue`
