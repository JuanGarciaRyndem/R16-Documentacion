# Propuesta 1 — Mailbot Worker .NET 10 con n8n + RabbitMQ

| Campo | Valor |
|-------|-------|
| **Requisito** | TPSC-RE-FU-008-v2 |
| **Base de Datos** | ProquifaDotNet |
| **Versión** | 1.0 |
| **Fecha** | 2026-06-02 |

---

## Resumen

Rediseño del Mailbot actual (Framework 4.8 + Tarea Windows) usando:

- **n8n** como orquestador que detecta correos nuevos en Gmail y publica eventos
- **RabbitMQ** como cola de mensajes que desacopla la recepción y el procesamiento
- **Worker .NET 10** que consume la cola, clasifica con IA y persiste en BD

---

## Flujo General

```
Gmail
  │
  │ (correo nuevo)
  ▼
n8n  (trigger Gmail → Workflow)
  │
  │ PUBLISH evento { IdGmail, Region, FechaRecepcion }
  ▼
RabbitMQ  Exchange: mailbot.correos
  │  Queue MEX: mailbot.correos.mex
  │  Queue PER: mailbot.correos.per
  ▼
Mailbot.Worker (.NET 10)  ← consume la cola por región
  │
  │ Lee correo completo vía Gmail API
  │ Llama Agente IA (clasificar + extraer datos)
  ▼
ProquifaDotNet (SQL Server)
  CorreoRecibido / CorreoRecibidoContenido / ArchivoCorreoRecibido
  CorreoRecibidoCliente / CorreoRecibidoEstatus
  fccFolioPagoCliente (si clasificación = cobro)
  MailbotEventoCorreo / MailbotClasificacionLog
```

---

## Componentes de la Solución

| Componente | Tecnología | Responsabilidad |
|------------|-----------|-----------------|
| Trigger de correo | n8n + Gmail trigger | Detecta correos nuevos y publica evento en RabbitMQ |
| Cola de mensajes | RabbitMQ | Desacopla recepción y procesamiento; garantiza entrega |
| Worker | .NET 10 Worker Service | Consume cola, lee Gmail API, clasifica con IA, persiste en BD |
| Clasificador IA | OpenAI / Azure OpenAI | Clasifica correo y extrae datos según tipo |
| Persistencia | SQL Server — ProquifaDotNet | Almacena correos, clasificaciones y pendientes |
| API de salud | .NET 10 Minimal API | `/health` y `/metrics` para monitoreo del Worker |

---

## Arquitectura de la Solución .NET 10

```
MailbotWorker.sln
│
├── Mailbot.Worker
│     Program.cs                        ← Host builder, DI, configuración
│     Workers/
│       CorreoWorker.cs                 ← IHostedService: consume RabbitMQ por región
│       CorreoWorkerMex.cs              ← Worker especializado Región MEX
│       CorreoWorkerPer.cs              ← Worker especializado Región PER
│
├── Mailbot.Api
│     Program.cs                        ← Minimal API
│     Endpoints/
│       HealthEndpoints.cs              ← GET /health, GET /metrics
│       ReentrenamientoEndpoints.cs     ← POST /reentrenamiento/trigger
│
├── Mailbot.Application
│     UseCases/
│       ProcesarCorreoUseCase.cs        ← orquesta clasificación y persistencia
│       GenerarPendienteUseCase.cs      ← crea fccFolioPagoCliente para cobros
│     DTOs/
│       EventoCorreoDto.cs              ← payload del mensaje RabbitMQ
│       ResultadoClasificacionDto.cs    ← respuesta del Agente IA
│
├── Mailbot.Domain
│     Entities/
│       CorreoRecibido.cs
│       ClasificacionCorreo.cs
│       EventoCorreo.cs
│     Interfaces/
│       IClasificadorAgente.cs          ← contrato del Agente IA
│       ICorreoRepository.cs
│       IEventoQueue.cs                 ← contrato de la cola
│
├── Mailbot.Infrastructure
│     Gmail/
│       GmailService.cs                 ← lee correo completo vía Gmail API
│     RabbitMQ/
│       RabbitMQConsumer.cs             ← consume cola por región
│       RabbitMQConfig.cs               ← exchange, queues, bindings
│     IA/
│       OpenAIClasificadorAgente.cs     ← implementa IClasificadorAgente
│       PromptBuilder.cs                ← construye prompts desde archivos .txt
│     Prompts/
│       clasificacion_correo.txt        ← prompt editable sin recompilar
│       extraccion_cobro.txt
│       extraccion_pedido.txt
│       extraccion_cotizacion.txt
│     Persistence/
│       ProquifaDbContext.cs            ← EF Core Scaffold ProquifaDotNet
│       Repositories/
│         CorreoRepository.cs
│         PendienteRepository.cs
│
└── Mailbot.Tests
      Unit/
        ProcesarCorreoUseCaseTests.cs
        ClasificadorAgenteTests.cs
      Integration/
        RabbitMQConsumerTests.cs
        ProquifaDbContextTests.cs
```

---

## Configuración n8n — Workflow de Trigger Gmail

```json
{
  "nodes": [
    {
      "name": "Gmail Trigger",
      "type": "n8n-nodes-base.gmailTrigger",
      "parameters": {
        "pollTimes": { "item": [{ "mode": "everyMinute" }] },
        "filters": { "labelIds": ["INBOX"] }
      }
    },
    {
      "name": "Publicar en RabbitMQ",
      "type": "n8n-nodes-base.rabbitmq",
      "parameters": {
        "exchange": "mailbot.correos",
        "routingKey": "={{ $json.region == 'MEX' ? 'correos.mex' : 'correos.per' }}",
        "content": "={{ JSON.stringify({ idGmail: $json.id, region: $json.region, fechaRecepcion: $now }) }}"
      }
    }
  ]
}
```

### Configuración RabbitMQ

```
Exchange:  mailbot.correos         (type: direct, durable: true)
Queues:
  mailbot.correos.mex              (durable: true, routing key: correos.mex)
  mailbot.correos.per              (durable: true, routing key: correos.per)
Dead Letter Exchange:
  mailbot.correos.dlx              (para mensajes con errores repetidos)
  mailbot.correos.dlq              (cola de mensajes fallidos para revisión)
```

---

## Configuración `appsettings.json` del Worker

```json
{
  "RabbitMQ": {
    "Host": "rabbitmq-server",
    "Port": 5672,
    "VirtualHost": "/",
    "Username": "mailbot_user",
    "Password": "*** reemplazar con secret ***",
    "Queues": {
      "MEX": "mailbot.correos.mex",
      "PER": "mailbot.correos.per"
    }
  },
  "Gmail": {
    "CredentialsPath": "/secrets/gmail_credentials.json",
    "TokenPath": "/secrets/gmail_token/"
  },
  "IA": {
    "Provider": "AzureOpenAI",
    "Endpoint": "https://tu-recurso.openai.azure.com/",
    "ApiKey": "*** reemplazar con secret ***",
    "DeploymentName": "gpt-4o",
    "MaxTokens": 2000,
    "Temperature": 0.1
  },
  "ConnectionStrings": {
    "ProquifaDotNet": "Server=RYNL010;Database=ProquifaDotNet;..."
  },
  "Worker": {
    "MaxReintentos": 3,
    "BackoffSegundosBase": 5
  }
}
```

> ⚠️ **Seguridad:** `ApiKey`, `Password` y `ConnectionString` deben gestionarse vía Azure Key Vault, AWS Secrets Manager o variables de entorno del host. **Nunca** almacenar credenciales en `appsettings.json` en el repositorio.

---

## Scaffold EF Core — Tablas ProquifaDotNet

```bash
# Comando scaffold desde Mailbot.Infrastructure
dotnet ef dbcontext scaffold \
  "Server=RYNL010;Database=ProquifaDotNet;..." \
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
    catDominioCorreoRecibidoPendiente \
    fccFolioPagoCliente \
    Region \
    MailbotEventoCorreo \
    MailbotClasificacionLog \
  --no-onconfiguring \
  --force
```

---

## Tablas BD Nuevas Requeridas

### `MailbotEventoCorreo`

**Propósito:** Registro de cada evento recibido desde RabbitMQ. Permite trazabilidad, reintentos y detección de duplicados.

```sql
CREATE TABLE dbo.MailbotEventoCorreo (
    IdMailbotEventoCorreo    uniqueidentifier NOT NULL
        CONSTRAINT PK_MailbotEventoCorreo PRIMARY KEY CLUSTERED
        CONSTRAINT DF_MailbotEventoCorreo_Id DEFAULT (NEWID()),
    IdRegion                 uniqueidentifier NOT NULL
        CONSTRAINT FK_MailbotEventoCorreo_Region
            FOREIGN KEY REFERENCES dbo.Region(IdRegion),
    IdentificadorCorreoGmail varchar(200)     NOT NULL,
    CorreoEmisor             varchar(180)     NULL,
    Asunto                   varchar(350)     NULL,
    FechaEvento              datetime         NOT NULL,
    Procesado                bit              NOT NULL
        CONSTRAINT DF_MailbotEventoCorreo_Procesado DEFAULT (0),
    FechaProcesado           datetime         NULL,
    Intentos                 int              NOT NULL
        CONSTRAINT DF_MailbotEventoCorreo_Intentos DEFAULT (0),
    Error                    varchar(MAX)     NULL,
    FechaRegistro            datetime         NOT NULL
        CONSTRAINT DF_MailbotEventoCorreo_FechaRegistro DEFAULT (GETDATE()),
    FechaUltimaActualizacion datetime         NOT NULL
        CONSTRAINT DF_MailbotEventoCorreo_FechaActualizacion DEFAULT (GETDATE()),
    Activo                   bit              NOT NULL
        CONSTRAINT DF_MailbotEventoCorreo_Activo DEFAULT (1)
);

CREATE NONCLUSTERED INDEX IX_MailbotEventoCorreo_Gmail
    ON dbo.MailbotEventoCorreo (IdentificadorCorreoGmail, IdRegion);

CREATE NONCLUSTERED INDEX IX_MailbotEventoCorreo_Pendientes
    ON dbo.MailbotEventoCorreo (Procesado, Intentos) WHERE Procesado = 0;
```

### `MailbotClasificacionLog`

**Propósito:** Registro de cada clasificación realizada por el Agente IA. Permite auditoría del modelo y reentrenamiento futuro.

```sql
CREATE TABLE dbo.MailbotClasificacionLog (
    IdMailbotClasificacionLog uniqueidentifier NOT NULL
        CONSTRAINT PK_MailbotClasificacionLog PRIMARY KEY CLUSTERED
        CONSTRAINT DF_MailbotClasificacionLog_Id DEFAULT (NEWID()),
    IdMailbotEventoCorreo     uniqueidentifier NOT NULL
        CONSTRAINT FK_MailbotClasificacionLog_Evento
            FOREIGN KEY REFERENCES dbo.MailbotEventoCorreo(IdMailbotEventoCorreo),
    IdCorreoRecibido          uniqueidentifier NULL
        CONSTRAINT FK_MailbotClasificacionLog_Correo
            FOREIGN KEY REFERENCES dbo.CorreoRecibido(IdCorreoRecibido),
    ClasificacionIA           varchar(150)     NOT NULL,
    Confianza                 decimal(5,2)     NULL,
    ClasificacionFinal        varchar(150)     NULL,
    EsCorrecta                bit              NULL,
    ModeloVersion             varchar(50)      NULL,
    PromptTokens              int              NULL,
    CompletionTokens          int              NULL,
    FechaRegistro             datetime         NOT NULL
        CONSTRAINT DF_MailbotClasificacionLog_FechaRegistro DEFAULT (GETDATE()),
    Activo                    bit              NOT NULL
        CONSTRAINT DF_MailbotClasificacionLog_Activo DEFAULT (1)
);
```

---

## Lógica de Reintentos del Worker

```
SI mensaje falla al procesarse:
    Intentos++
    SI Intentos < MaxReintentos (3):
        ESPERAR BackoffSegundosBase × 2^(Intentos-1)   ← backoff exponencial
        RE-ENCOLAR el mensaje
    SI Intentos >= MaxReintentos:
        MOVER a Dead Letter Queue (mailbot.correos.dlq)
        UPDATE MailbotEventoCorreo SET Error = mensaje_error
        ALERTA al equipo de operaciones
```

| Intento | Espera |
|:-------:|--------|
| 1 | 5 segundos |
| 2 | 10 segundos |
| 3 | 20 segundos |
| 4+ | Dead Letter Queue |

---

## Cambios BD Requeridos

| # | Paso | Script | Prioridad |
|:-:|------|--------|:---------:|
| 1 | `CREATE TABLE MailbotEventoCorreo` | Script sección Tablas Nuevas | 🔴 Alta |
| 2 | `CREATE TABLE MailbotClasificacionLog` | Script sección Tablas Nuevas | 🔴 Alta |
| 3 | Renombrar `'Pago'` → `'Cobro'` en `catClasificacionCorreoRecibido` | `UPDATE Clave='pago'` | 🔴 Alta |
| 4 | `INSERT` proceso `'Cobros'` en `catProceso` | `INSERT INTO catProceso` | 🔴 Alta |

---

## Ventajas y Desventajas de esta Propuesta

| Aspecto | Ventaja | Desventaja |
|---------|---------|------------|
| Desacoplamiento | Worker y Gmail completamente desacoplados vía cola | Infraestructura adicional: n8n + RabbitMQ |
| Escalabilidad | Múltiples Workers consumen la misma cola en paralelo | Mayor complejidad operativa |
| Reintentos | RabbitMQ + DLQ garantizan que ningún correo se pierde | Requiere monitoreo de la DLQ |
| Flexibilidad | n8n permite agregar filtros y transformaciones sin recompilar | Dependencia de disponibilidad de n8n |
| Latencia | Baja (~segundos según intervalo de polling de n8n) | No es tiempo real puro (depende del trigger de n8n) |
| Visibilidad | n8n ofrece dashboard de ejecuciones y errores | Costo de licencia n8n si se usa versión cloud |

---

## Diagrama de Infraestructura

```
+------------------+     OAuth2      +------------------+
|   Gmail API      | <-------------- |       n8n        |
|  (INBOX MEX/PER) |                 | (Workflow Trigger)|
+------------------+                 +--------+---------+
                                              |
                                       PUBLISH evento
                                              |
                                     +--------v---------+
                                     |    RabbitMQ      |
                                     | Exchange:        |
                                     | mailbot.correos  |
                                     |  Queue MEX       |
                                     |  Queue PER       |
                                     |  DLQ (errores)   |
                                     +--------+---------+
                                              |
                                       CONSUME mensaje
                                              |
                                +-------------v-----------+
                                |   Mailbot.Worker        |
                                |   (.NET 10)             |
                                |   CorreoWorkerMex       |
                                |   CorreoWorkerPer       |
                                +------+----------+-------+
                                       |          |
                                Gmail API      Agente IA
                                (leer correo) (clasificar)
                                       |          |
                                +------v----------v-------+
                                |   ProquifaDotNet        |
                                |   (SQL Server)          |
                                +-------------------------+
```

---

## Gaps y Pendientes

| # | Gap | Descripción | Acción |
|:-:|-----|-------------|--------|
| 1 | Modelo IA | OpenAI vs Azure OpenAI vs modelo propio | Definir según costo y privacidad |
| 2 | Licencia n8n | Versión self-hosted (gratis) vs cloud (costo) | Confirmar con infraestructura |
| 3 | Cliente no identificado | Correos sin cliente asignado — ¿quién los atiende? | Confirmar con cliente |
| 4 | Clasificación `'Pago'` en uso | Verificar registros activos antes de renombrar | `SELECT COUNT(*) WHERE Clave='pago'` |
| 5 | Prompt base del Agente | Texto de clasificación y extracción por tipo | Redactar con equipo de negocio |
| 6 | Credenciales Gmail multi-cuenta | Una cuenta por región (MEX/PER) | Confirmar cuentas con IT |
| 7 | Monitoreo DLQ | Alertas cuando hay mensajes en Dead Letter Queue | Configurar alerta en RabbitMQ Management |

---

**Versión:** 1.0 — Propuesta 1: n8n + RabbitMQ + Worker .NET 10  
**Base de Datos:** ProquifaDotNet  
**Referencia:** `TPSC-RE-FU-008-v2_Diccionario_Datos.md`
