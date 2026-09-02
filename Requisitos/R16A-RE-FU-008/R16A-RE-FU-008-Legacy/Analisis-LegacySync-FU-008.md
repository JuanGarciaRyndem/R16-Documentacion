# Análisis LegacySync — R16A-RE-FU-008 (Buzón de Cobros)

**Fuentes revisadas:**
- Google Docs: `[R16A-RE-FU-008][DIS-SOL] Diseño de la solución` — versión 3.0, 2 jul 2026, autor Carlos Iván Morales Carreón, revisor Juan David García Cruz (documento vigente).
- Local: `Requisitos/R16A-RE-FU-008/R16A-RE-FU-008-Legacy/R16A-RE-FU-008-Legacy.md` y `...-Tareas.md` — internamente identificados como **R16A-NO-FU-003**, fecha 2026-06-11.
- Local: `TPSC-RE-FU-008-DAR-Mecanismo-Orquestacion-LegacySync.md` — versión 1.3, 22/06/2026, autor Carlos Iván Morales Carreón (DAR de la elección Hangfire vs. RabbitMQ y otras alternativas, decisión confirmada).
- Estándares aplicados a los ejemplos de código y BD: `ryndem-standards:ryndem-dotnet` (capítulos 01, 02, 05) y `ryndem-standards:ryndem-sqlserver` (capítulos 01, 04).

---

## 1. Hallazgo principal

Existen **dos diseños de LegacySync para el mismo requisito, y no coinciden**. La carpeta local `R16A-RE-FU-008-Legacy` contiene una versión de análisis fechada 2026-06-11, identificada internamente como **R16A-NO-FU-003** (no como RE-FU-008), que describe una plataforma genérica multi-entidad. El Google Doc es la versión 3.0 del diseño de FU-008, fechada 2 jul 2026, con decisiones ya confirmadas por el equipo entre el 19 y el 24 de junio — **posteriores** a la fecha del documento local — que cambian puntos estructurales del diseño, no solo detalles menores.

Si el equipo de desarrollo construye contra `R16A-RE-FU-008-Legacy-Tareas.md` tal como está, va a implementar un mecanismo de disparo, un almacenamiento de Hangfire y una integración de notificaciones/archivos que el equipo **descartó explícitamente**. Antes de iniciar codificación de LegacySync recomiendo actualizar o reemplazar ambos documentos locales para que reflejen la v3.0, o al menos anexarles una nota de remisión al Google Doc como fuente de verdad.

---

## 2. Comparativo — carpeta local vs. Google Doc v3.0

| Aspecto                            | Carpeta local (`R16A-NO-FU-003`, 2026-06-11)                                                                                  | Google Doc FU-008 v3.0 (vigente, 2 jul 2026)                                                                                                                                                                                                                                                             |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Nombre de la solución              | `ProquifaDotNet.LegacySync`                                                                                                   | `Proquifa.Pqf2LegacySync`                                                                                                                                                                                                                                                                                |
| Alcance funcional                  | Multi-entidad: Clientes (RE-002/006), Pedidos Crédito (RE-010/011/012), Buzón Cobros (RE-008), Factura/CP/NC (RE-028/030/032) | **Solo Canal ETLCobros** (Buzón de Cobros) por decisión confirmada. Cotizaciones y Pedidos permanecen en su ETL SSIS actual, **sin migración planeada**. LB-T1–T10 quedan como base reutilizable para requisitos *futuros*, no como compromiso de absorber lo existente                                  |
| `PConnectProquifaDotNet`           | "Base de datos **nueva**" creada en `RYNL010` (sección 8.1)                                                                   | BD **existente desde 2023-10-02** — ya es la intermedia real del ETL SSIS de Cotizaciones/Pedidos (`Transferencia`/`EjecucionInterfaz`/`BitacoraInterface`/`catEstadoTransferencia`, con miles de filas reales). Solo 4 objetos son nuevos; LegacySync no toca ni reutiliza las tablas legacy de control |
| Disparo de sincronización          | `INSERT` directo en `SyncControl` "desde Finanzas" (cross-database)                                                           | El `INSERT` directo cross-base y el trigger de BD cross-database fueron **descartados explícitamente** (P20, 2026-06-24). Diseño de dos capas, ambas vía API: push inmediato (`POST /sync/v1/register` desde MailBot) + barrido de respaldo (`PendingSweepJob`, cada 30 min)                             |
| Storage de Hangfire                | "Puede usar `PConnectProquifaDotNet`... confirmar con DBA en T6" (pendiente)                                                  | **Confirmado**: `TaskSchedulerPqf` — BD dedicada ya existente, esquema `HangFire.*` reutilizado. `PConnectProquifaDotNet` queda solo como BD de control de negocio                                                                                                                                       |
| Notificación de fallos             | `BrevoNotificationService` — integración directa con Brevo                                                                    | `NotificacionesClient` — proxy HTTP hacia el microservicio `Proquifa.Pqf2Notificaciones` (.NET 10), no integración directa                                                                                                                                                                               |
| Sincronización de archivos         | `FileSyncService` descarga directo desde MinIO                                                                                | Obtiene archivos vía **endpoint de descarga de ProquifaDotNet** (patrón proxy análogo a `ProquifaDotNetFileClient`) — endpoint exacto pendiente de confirmar con Juan David antes de implementar                                                                                                         |
| Vista de pendientes                | `vw_SyncPendientes`                                                                                                           | `vSyncPending` — distinto nombre; ver nota de nomenclatura en §5.3                                                                                                                                                                                                                                       |
| Arquitectura de despacho           | `SyncJobBase` genérico por entidad, sin mecanismo de registro                                                                 | Se agrega `SyncChannelRegistry`/`ISyncChannelRegistry` + `SyncJobDispatcher`: el despacho se resuelve por `EntityName` en runtime, no atado al nombre de un job — un canal nuevo no requiere tocar el contrato                                                                                           |
| Idempotencia                       | No se especifica constraint de BD                                                                                             | Constraint **`UNIQUE`** explícito sobre la clave natural de origen (ej. `IdCorreoRecibidoCliente`) en cada tabla Legacy nueva, además del chequeo en código — defensa en profundidad                                                                                                                     |
| Evaluación del mecanismo           | No se menciona                                                                                                                | Hangfire evaluado formalmente contra RabbitMQ, Quartz.NET, Azure Service Bus/SQS, SQL Server Service Broker, canal en memoria y Kafka en un DAR dedicado (**DAR-Mecanismo de Orquestación LegacySync, v1.3, 22/06/2026** — sí está en la carpeta local, ver §2.3), decisión confirmada 2026-06-22        |
| Endpoints de API                   | 5 endpoints: `/sync/status/{entidad}`, `/sync/pendientes/{entidad}`, `/sync/log/{id}`, `/sync/reintentar/{id}`, `/sync/jobs`  | 3 endpoints versionados: `POST /sync/v1/register`, `GET /sync/v1/status`, `POST /sync/v1/retry`. No hay endpoint documentado de historial de log ni de dashboard de jobs                                                                                                                                 |
| Despliegue                         | No detallado (T6 solo dice "Windows Service")                                                                                 | `LegacySync.Worker` → Windows Service (`sc.exe create`); `LegacySync.API` → sitio IIS, **mismo IIS donde ya corre ProquifaDotNet** (.NET 4.8, requiere ANCM) — no es infraestructura nueva                                                                                                               |
| Bases de datos que toca LegacySync | 3 contextos: `ProquifaDotNet`, `PConnect`, `PConnectProquifaDotNet`                                                           | **4 bases de datos reales**: las 3 anteriores + `TaskSchedulerPqf` (storage de Hangfire, no mencionada en el documento local)                                                                                                                                                                            |

### 2.1 Por qué importa la discrepancia del disparador

Es el cambio más riesgoso de toda la lista. El documento local instruye un `INSERT` cross-database directo desde Finanzas hacia `SyncControl` — una integración por base de datos compartida. El equipo lo evaluó y lo **rechazó explícitamente** como restricción de arquitectura ("toda comunicación entre servicios se hace vía llamadas API"), a favor de un contrato HTTP (`POST /sync/v1/register`) con un mecanismo de respaldo por si el push falla. Construir el `INSERT` directo no solo sería trabajo desechable: rompería el desacoplamiento que el resto del diseño de las tres soluciones (MailBot / Finanzas / LegacySync) da por sentado.

### 2.2 Nota sobre el nombre del documento local

El archivo vive en `Requisitos/R16A-RE-FU-008/R16A-RE-FU-008-Legacy/` pero su encabezado interno dice `R16A-NO-FU-003 — Creación de Solución Base ProquifaDotNet.LegacySync`. Todo indica que **NO-FU-003 fue la iniciativa original** (infraestructura genérica de sincronización, sin requisito funcional específico que la disparara) y que **FU-008 v3.0 la absorbió y acotó** a un solo canal (ETLCobros) como parte del rediseño del Mailbot. Vale la pena decidir explícitamente si:
- se retira/archiva `R16A-NO-FU-003` y su carpeta pasa a documentar solo lo que FU-008 v3.0 define, o
- se conservan ambos como historia de decisión, dejando claro en el propio archivo cuál es la fuente vigente.

**Precisión importante:** la carpeta local no es uniformemente antigua. El DAR de mecanismo de orquestación (`TPSC-RE-FU-008-DAR-Mecanismo-Orquestacion-LegacySync.md`, v1.3, 22/06/2026 — ver §2.3) sí está correctamente encuadrado como TPSC-RE-FU-008 / Canal ETLCobros, y es consistente con el Google Doc v3.0. El problema de vigencia es específico de `R16A-RE-FU-008-Legacy.md` y su Tareas (los que arrastran el diseño de NO-FU-003), no de todo lo que hay en la carpeta.

### 2.3 DAR — Mecanismo de orquestación (Hangfire vs. RabbitMQ), decisión confirmada

El Google Doc v3.0 menciona que Hangfire fue "evaluado formalmente contra RabbitMQ, Kafka y otras alternativas en el DAR de Mecanismo de Orquestación LegacySync (v1.2)". El DAR real en la carpeta local está en **v1.3** (22/06/2026) — la v1.2 solo confirmaba la decisión; la v1.3 es un cambio menor de nomenclatura ("Canal E1" → "Canal ETLCobros"). Puntos que vale la pena que Juan David tenga presentes:

- **Descartadas sin comparativa:** Quartz.NET, Azure Service Bus/AWS SQS, SQL Server Service Broker (existe en `ProquifaDotNet` vía `SERVICE_QUEUE` pero es T-SQL de bajo nivel), canal en memoria (`System.Threading.Channels` — no sobrevive un reinicio, inaceptable para sincronización financiera) y Kafka (cero precedente en el ecosistema, sobredimensionado para el volumen real).
- **Comparativa real:** Hangfire vs. RabbitMQ. RabbitMQ **sí está desplegado y activo** en el ecosistema (`Core.Pqf`/`ProquifaDotNet`: colas `generadorDocumento`, `envioCorreo`, `PqfTimbrado`, `embalar`, `transaccionesSTP`; y `proquifa-punchout-backend`, una implementación más moderna con Outbox pattern, Dead Letter Exchange con TTL de 72 h, reintentos configurables vía `AppSetting` y alerta Brevo al agotar intentos) — no se descarta por falta de opción real, sino porque el patrón de Canal ETLCobros es polling/batch sobre `SyncControl`, no reacción a un evento publicado por otro proceso, y forzar RabbitMQ exigiría que `MailbotWorker` publicara explícitamente hacia `LegacySync`, rompiendo el desacoplamiento entre las tres soluciones que el propio DIS-SOL diseñó.
- **Precedente más fuerte de Hangfire:** `SincronizadorPqfLegacy` ya resuelve, con Hangfire, el mismo tipo de problema (ETL hacia Legacy/PConnect, ~95-98% de éxito) que ataca LegacySync, solo que para Cotizaciones en vez de Cobros — es el precedente más cercano al dominio real, más que cualquier precedente de RabbitMQ (que resuelve documentos/correo/órdenes externas, no sincronización a Legacy).
- **Decisión confirmada:** Hangfire, sin cambios al DIS-SOL — RabbitMQ no se retira de donde ya se usa, solo no se fuerza dentro de `LegacySync.Worker`. Si en el futuro `MailbotWorker` necesitara notificar a `LegacySync` en tiempo real (en vez de que `LegacySync` haga polling), el propio DAR ya deja anotado que RabbitMQ sería la primera opción a evaluar, con `proquifa-punchout-backend` como plantilla concreta.
- **Corrección de paso a otro documento:** el DAR corrige una imprecisión en `TPSC-RE-FU-008-Contexto.md` (línea 113) — la Propuesta 1 del Mailbot no se descartó por falta de RabbitMQ (sí existe), sino porque Gmail Push + Pub/Sub sigue siendo superior por latencia. Queda anotada como pendiente fuera de alcance del propio DAR; no vi ese archivo en la carpeta local para confirmar si ya se corrigió.
- **Hallazgo de seguridad, fuera de alcance del DAR pero vale la pena escalarlo:** `proquifa-punchout-backend/Infrastructure/RabbitMQ/RabbitMQSettings.cs` tiene una contraseña real hardcodeada como valor por defecto en el código fuente (no en configuración separada). El propio DAR lo señala como pendiente de reportar "a quien corresponda" — no encontré evidencia de que ya se haya hecho. Recomiendo confirmarlo con el equipo dueño de esa solución.

---

## 3. Impacto en Backend

Estructura de proyectos según el Google Doc (LB-T1 a LB-T10, reutilizable por futuros canales):

| Proyecto | Responsabilidad |
|---|---|
| `LegacySync.Domain` | Entidades `SyncControl`, `SyncJobLog`, `AppSettings`. Interfaces `IRepository<T>`, `ISyncJobLog`, `IExceptionClassifier`, `IFileSync`, `ISyncChannelHandler` |
| `LegacySync.Application` | CQRS con MediatR, `SyncControlService`, `SyncChannelRegistry`/`ISyncChannelRegistry`, `SyncJobDispatcher`, DTOs |
| `LegacySync.Infrastructure` | 3 `DbContext` (origen, Legacy, control), `ExceptionClassifier`, `FileSyncService` (proxy a ProquifaDotNet), `NotificacionesClient` |
| `LegacySync.Worker` | Hangfire + `SyncJobBase`, `PendingSweepJob` (cada 30 min), extensión de `AutomaticRetryFailureLogFilter` |
| `LegacySync.API` | `POST /sync/v1/register`, `GET /sync/v1/status`, `POST /sync/v1/retry` — IdentityServer + Swagger |

Los siguientes fragmentos son **código ilustrativo** que respeta el contrato documentado (no vienen del diseño — el Google Doc no incluye código, solo tablas de responsabilidad) y siguen el arquetipo Ryndem: capas Domain→Application→Infrastructure/API, `namespace` file-scoped, `nullable enable`, catálogo de errores con `Result`/`SemanticException` en vez de excepciones para flujos esperados.

### 3.1 Domain — contrato del canal ETL

```csharp
namespace LegacySync.Domain.Interfaces;

/// <summary>
/// Contrato que implementa cada canal ETL (p. ej. CobrosInboxEtlJob) para sincronizar
/// una entidad de origen hacia el sistema Legacy (PConnect). SyncChannelRegistry
/// indexa los handlers registrados en DI por EntityName.
/// </summary>
public interface ISyncChannelHandler
{
    /// <summary>Nombre de entidad que resuelve este handler (p. ej. "CobrosInbox").</summary>
    string EntityName { get; }

    Task<Result> ExecuteSyncAsync(SyncPendienteDto pendiente, CancellationToken cancellationToken);
}
```

### 3.2 Domain — catálogo de errores del dominio Sync

Todo error nace del catálogo (`05-errors-and-resilience`); el prefijo `SYN` es nuevo y debe revisarse en el mismo PR que lo introduce.

```csharp
namespace LegacySync.Domain.Errors;

public static partial class Errors
{
    public static class Sync
    {
        public static readonly ErrorDefinition EntityNotRegistered = new(
            Code: "SYN-001",
            Title: "Entity not registered for synchronization",
            Description: "El EntityName recibido no tiene un handler registrado en SyncChannelRegistry.",
            Category: ErrorCategory.Validation,
            Severity: ErrorSeverity.Low,
            HttpStatusCode: 400,
            IsTransient: false);

        public static readonly ErrorDefinition RetryNotAllowed = new(
            Code: "SYN-002",
            Title: "Retry not allowed for permanent error",
            Description: "El registro tiene TipoError = Permanent; no es candidato a reintento manual ni automático.",
            Category: ErrorCategory.Business,
            Severity: ErrorSeverity.Medium,
            HttpStatusCode: 400,
            IsTransient: false);
    }
}
```

### 3.3 Application — registro idempotente y despacho por canal

```csharp
namespace LegacySync.Application.Services;

public sealed class SyncControlService : ISyncControlService
{
    private readonly ISyncControlRepository _repository;

    public SyncControlService(ISyncControlRepository repository) => _repository = repository;

    // POST /sync/v1/register puede recibir el mismo (EntityName, RecordId) más de una vez
    // (push inmediato + barrido de respaldo) — 201 la primera vez, 200 idempotente después.
    public async Task<Result<SyncRegisterResultDto>> RegisterPendingAsync(
        string entityName, Guid recordId, CancellationToken cancellationToken)
    {
        var existente = await _repository.FindAsync(entityName, recordId, cancellationToken);
        if (existente is not null)
            return Result.Ok(new SyncRegisterResultDto(existente.IdSyncControl, IsNew: false));

        var registro = SyncControl.CrearPendiente(entityName, recordId);
        await _repository.AddAsync(registro, cancellationToken);

        return Result.Ok(new SyncRegisterResultDto(registro.IdSyncControl, IsNew: true));
    }
}

public sealed class SyncChannelRegistry : ISyncChannelRegistry
{
    private readonly IReadOnlyDictionary<string, ISyncChannelHandler> _handlers;

    // Los handlers se inyectan por DI (uno por canal ETL) y se indexan por EntityName —
    // agregar un canal nuevo no requiere tocar el dispatcher.
    public SyncChannelRegistry(IEnumerable<ISyncChannelHandler> handlers) =>
        _handlers = handlers.ToDictionary(h => h.EntityName, StringComparer.OrdinalIgnoreCase);

    public bool TryResolve(string entityName, out ISyncChannelHandler handler) =>
        _handlers.TryGetValue(entityName, out handler!);
}
```

### 3.4 Worker — canal ETLCobros y barrido de respaldo

```csharp
namespace LegacySync.Worker.Jobs;

/// <summary>
/// Canal ETLCobros: sincroniza fccFolioPagoCliente (Buzón de Cobros) hacia BuzonCobros en PConnect.
/// Hereda el flujo estándar de SyncJobBase (pendientes → EnProceso → ejecutar → log → Completado/Error)
/// y solo implementa la lógica propia del canal.
/// </summary>
public sealed class CobrosInboxEtlJob : SyncJobBase
{
    public override string EntityName => "CobrosInbox";

    private readonly IEtlCollectionsInboxLegacyService _etlService;

    public CobrosInboxEtlJob(
        IEtlCollectionsInboxLegacyService etlService,
        ISyncControlRepository syncControlRepository,
        ISyncJobLog syncJobLog,
        IExceptionClassifier exceptionClassifier,
        INotificationService notificationService)
        : base(syncControlRepository, syncJobLog, exceptionClassifier, notificationService) =>
        _etlService = etlService;

    protected override async Task<Result> ExecuteSyncAsync(
        SyncPendienteDto pendiente, CancellationToken cancellationToken)
    {
        // Idempotencia de negocio: si ya existe en Legacy por la clave natural
        // (IdCorreoRecibidoCliente), se marca Completado sin reinsertar.
        var yaExiste = await _etlService.ExisteEnLegacyAsync(pendiente.IdRegistroOrigen, cancellationToken);
        return yaExiste
            ? Result.Ok()
            : await _etlService.SincronizarAsync(pendiente.IdRegistroOrigen, cancellationToken);
    }
}

/// <summary>
/// Respaldo del push inmediato (POST /sync/v1/register): cada 30 minutos cruza
/// GET /api/v1/collections/inbox/recent (nuevo endpoint de solo lectura en ProquifaDotNet)
/// contra SyncControl, para el caso donde la llamada de MailBot nunca llegó.
/// </summary>
public sealed class PendingSweepJob
{
    private readonly ICollectionsInboxClient _collectionsInboxClient;
    private readonly ISyncControlService _syncControlService;

    public PendingSweepJob(ICollectionsInboxClient collectionsInboxClient, ISyncControlService syncControlService)
    {
        _collectionsInboxClient = collectionsInboxClient;
        _syncControlService = syncControlService;
    }

    public async Task EjecutarAsync(CancellationToken cancellationToken)
    {
        var desde = DateTime.UtcNow.AddHours(-1); // ventana con margen sobre el intervalo de 30 min
        var recientes = await _collectionsInboxClient.ObtenerRecientesAsync(desde, cancellationToken);

        foreach (var folio in recientes)
            await _syncControlService.RegisterPendingAsync("CobrosInbox", folio.Id, cancellationToken);
    }
}
```

### 3.5 API — endpoint de registro

```csharp
namespace LegacySync.API.Controllers;

[ApiController]
[Route("sync/v1")]
public sealed class SyncController : ControllerBase
{
    private readonly ISyncControlService _syncControlService;
    private readonly ISyncChannelRegistry _channelRegistry;

    public SyncController(ISyncControlService syncControlService, ISyncChannelRegistry channelRegistry)
    {
        _syncControlService = syncControlService;
        _channelRegistry = channelRegistry;
    }

    /// <summary>Registers a new pending synchronization record for a given entity and record id.</summary>
    /// <param name="request">Entity name and source record identifier.</param>
    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterSyncRequest request, CancellationToken cancellationToken)
    {
        if (!_channelRegistry.TryResolve(request.EntityName, out _))
            return Problem(Errors.Sync.EntityNotRegistered); // 400 problem+json vía handler global

        var resultado = await _syncControlService.RegisterPendingAsync(
            request.EntityName, request.RecordId, cancellationToken);

        return resultado.Value!.IsNew
            ? StatusCode(StatusCodes.Status201Created, resultado.Value)
            : Ok(resultado.Value);
    }
}

public sealed record RegisterSyncRequest(string EntityName, Guid RecordId);
```

### 3.6 Impacto adicional fuera de `LegacySync`

- **`Proquifa.Pqf2MailBot`** — `GeneratePendingUseCase` debe llamar a `POST /sync/v1/register` inmediatamente después de insertar `fccFolioPagoCliente` (Flujo 1, paso 14). Si la llamada falla, **no se reintenta en caliente**: queda a cargo del `PendingSweepJob`.
- **`ProquifaDotNet` (Finanzas)** — nuevo endpoint de solo lectura `GET /api/v1/collections/inbox/recent` en `CollectionsInboxController`, consumido exclusivamente por el barrido (no expuesto a UI).
- **`Proquifa.Pqf2Notificaciones`** — LegacySync deja de integrar Brevo directo; pasa a ser cliente de este microservicio vía `NotificacionesClient`.

---

## 4. Impacto en Base de Datos

LegacySync toca **cuatro** bases de datos, no tres — el documento local omite `TaskSchedulerPqf`:

| Base de datos | Rol | Estado |
|---|---|---|
| `ProquifaDotNet` | Origen — lectura vía `ProquifaDotNetDbContext` | Existente |
| `PConnect` (Legacy) | Destino — escritura vía `PConnectDbContext`; aquí vive la tabla nueva `BuzonCobros` | Existente |
| `PConnectProquifaDotNet` | Control de negocio de LegacySync (`SyncControl`/`SyncJobLog`/`AppSettings`/`vSyncPending`) | **Existente desde 2023-10-02** — corrige al documento local, que la describe como nueva |
| `TaskSchedulerPqf` | Storage de Hangfire (esquema `HangFire.*` reutilizado) | Existente — **no mencionada en el documento local** |

### 4.1 Objetos nuevos en `PConnectProquifaDotNet`

Solo 4 objetos son nuevos dentro de esta base ya existente. **LegacySync no toca ni reutiliza** las tablas legacy de control que ya viven ahí (`Transferencia`, `EjecucionInterfaz`, `BitacoraInterface`, `catEstadoTransferencia`) — usa su propio esquema por decisión explícita.

```sql
-- SyncControl: estado de sincronización por registro de origen
CREATE TABLE SyncControl
(
    IdSyncControl              UNIQUEIDENTIFIER NOT NULL CONSTRAINT DF_SyncControl_Id DEFAULT NEWID(),
    Entidad                    NVARCHAR(100)    NOT NULL,              -- EntityName del canal (ej. 'CobrosInbox')
    IdRegistroOrigen           UNIQUEIDENTIFIER NOT NULL,               -- PK del registro en ProquifaDotNet
    Estado                     NVARCHAR(20)     NOT NULL CONSTRAINT DF_SyncControl_Estado DEFAULT 'Pendiente',
    NumeroIntentos             INT              NOT NULL CONSTRAINT DF_SyncControl_Intentos DEFAULT 0,
    MaxReintentos              INT              NOT NULL CONSTRAINT DF_SyncControl_MaxReintentos DEFAULT 3,
    FechaUltimoIntento         DATETIME2(7)     NULL,
    FechaCompletado            DATETIME2(7)     NULL,
    MotivoError                NVARCHAR(MAX)    NULL,
    TipoError                  NVARCHAR(20)     NULL,                  -- Transient / Permanent
    FechaRegistro              DATETIME2(7)     NOT NULL CONSTRAINT DF_SyncControl_FechaRegistro DEFAULT SYSUTCDATETIME(),
    FechaUltimaActualizacion   DATETIME2(7)     NOT NULL CONSTRAINT DF_SyncControl_FechaActualizacion DEFAULT SYSUTCDATETIME(),

    CONSTRAINT PK_SyncControl PRIMARY KEY (IdSyncControl),
    -- Idempotencia de POST /sync/v1/register: mismo (Entidad, IdRegistroOrigen) no se duplica.
    CONSTRAINT UQ_SyncControl_Entidad_IdRegistroOrigen UNIQUE (Entidad, IdRegistroOrigen)
)
GO

CREATE NONCLUSTERED INDEX IX_SyncControl_Entidad_Estado
    ON SyncControl (Entidad, Estado, NumeroIntentos)
GO

-- SyncJobLog: log estructurado por ejecución, con snapshot del payload
CREATE TABLE SyncJobLog
(
    IdSyncJobLog        UNIQUEIDENTIFIER NOT NULL CONSTRAINT DF_SyncJobLog_Id DEFAULT NEWID(),
    IdSyncControl       UNIQUEIDENTIFIER NOT NULL,
    Entidad             NVARCHAR(100)    NOT NULL,
    IdRegistroOrigen    UNIQUEIDENTIFIER NOT NULL,
    NombreJob           NVARCHAR(200)    NOT NULL,
    Estado              NVARCHAR(20)     NOT NULL,                    -- Exito / Error
    PayloadJson         NVARCHAR(MAX)    NULL,
    RespuestaJson       NVARCHAR(MAX)    NULL,
    MensajeError        NVARCHAR(MAX)    NULL,
    TipoError           NVARCHAR(20)     NULL,
    DuracionMs          INT              NULL,
    NumeroIntento       INT              NOT NULL,
    FechaRegistro       DATETIME2(7)     NOT NULL CONSTRAINT DF_SyncJobLog_FechaRegistro DEFAULT SYSUTCDATETIME(),

    CONSTRAINT PK_SyncJobLog PRIMARY KEY (IdSyncJobLog),
    CONSTRAINT FK_SyncJobLog_SyncControl_IdSyncControl
        FOREIGN KEY (IdSyncControl) REFERENCES SyncControl (IdSyncControl)
)
GO

CREATE NONCLUSTERED INDEX IX_SyncJobLog_Entidad_FechaRegistro
    ON SyncJobLog (Entidad, FechaRegistro DESC)
GO

-- AppSettings: configuración en caliente (reintentos, destinatarios, flags por entidad)
CREATE TABLE AppSettings
(
    IdAppSettings               UNIQUEIDENTIFIER NOT NULL CONSTRAINT DF_AppSettings_Id DEFAULT NEWID(),
    Clave                       NVARCHAR(200)    NOT NULL,
    Valor                       NVARCHAR(MAX)    NOT NULL,
    Descripcion                 NVARCHAR(500)    NULL,
    FechaUltimaActualizacion    DATETIME2(7)     NOT NULL CONSTRAINT DF_AppSettings_FechaActualizacion DEFAULT SYSUTCDATETIME(),

    CONSTRAINT PK_AppSettings PRIMARY KEY (IdAppSettings),
    CONSTRAINT UQ_AppSettings_Clave UNIQUE (Clave)
)
GO

-- vSyncPending: origen de todos los recurring jobs de Hangfire
CREATE VIEW vSyncPending AS
SELECT sc.*
  FROM SyncControl sc
 WHERE sc.Estado IN ('Pendiente', 'Error')
   AND sc.NumeroIntentos < sc.MaxReintentos
GO
```

### 4.2 Tabla nueva en `PConnect` (Legacy) — `BuzonCobros`

Confirmada contra el esquema real de `PConnectProquifaDotNet` que **no existe** `BuzonCobros` hoy — solo `BuzonPagos` (vacía) y `BuzonPedido` (292 filas). El molde a seguir es el mismo patrón de esas tablas legacy, con el agregado del índice único que ellas nunca tuvieron:

```sql
CREATE TABLE BuzonCobros
(
    IdBuzonCobros                       UNIQUEIDENTIFIER NOT NULL CONSTRAINT DF_BuzonCobros_Id DEFAULT NEWID(),
    IdCorreoRecibidoClienteReferencia   UNIQUEIDENTIFIER NOT NULL,   -- FK al correo en ProquifaDotNet
    IdCorreoRecibidoCliente             UNIQUEIDENTIFIER NOT NULL,   -- clave natural de origen
    CorreoEmisor                        VARCHAR(500)     NOT NULL,
    DoctoR                              INT              NULL,       -- folio que asigna Legacy al sincronizar; sin FK (ETLCobros-T10 pendiente)
    FechaRegistro                       DATETIME2(7)     NOT NULL CONSTRAINT DF_BuzonCobros_FechaRegistro DEFAULT SYSUTCDATETIME(),
    FechaTramitacion                    DATETIME2(7)     NULL,
    Bitacora                            VARCHAR(500)     NULL,
    ArchivoSincronizado                 BIT              NOT NULL CONSTRAINT DF_BuzonCobros_ArchivoSincronizado DEFAULT 0,
    Actualizado                         BIT              NOT NULL CONSTRAINT DF_BuzonCobros_Actualizado DEFAULT 0,
    RegistroCompleto                    BIT              NOT NULL CONSTRAINT DF_BuzonCobros_RegistroCompleto DEFAULT 0,
    Insertado                           BIT              NOT NULL CONSTRAINT DF_BuzonCobros_Insertado DEFAULT 0,

    CONSTRAINT PK_BuzonCobros PRIMARY KEY (IdBuzonCobros),
    -- Defensa en profundidad de idempotencia: BuzonPedido/BuzonPagos nunca tuvieron esto.
    CONSTRAINT UQ_BuzonCobros_IdCorreoRecibidoCliente UNIQUE (IdCorreoRecibidoCliente)
)
GO
```

> **Pendiente abierto (ETLCobros-T10):** el valor exacto que debe llevar `DoctoR` se resuelve contra `PConnect.dbo.DoctosR` en la instancia legacy `RYNL015` (linked server `LegacyAux`) — no fue posible confirmarlo por incompatibilidad TLS con las herramientas actuales. El esquema de arriba ya alcanza para arrancar LB-T1/T2 con el stub (`EtlCollectionsInboxLegacyServiceStub`).

### 4.3 Notas de nomenclatura frente al estándar `data/sqlserver`

- `PConnectProquifaDotNet` es una base **anclada en español** (aloja tablas legacy como `Transferencia`/`BitacoraInterface`). Los 4 objetos nuevos de LegacySync (`SyncControl`, `SyncJobLog`, `AppSettings`, `vSyncPending`) están en inglés — es una decisión ya tomada en ambos documentos de diseño, pero conviene dejarla explícita como excepción, porque el estándar solo prevé inglés para **bases de datos nuevas**, no para objetos nuevos agregados a una base anclada en español.
- El nombre exacto de la vista difiere entre documentos: local usa `vw_SyncPendientes` (prefijo `vw_`, plural), el Google Doc usa `vSyncPending`. El estándar pide prefijo `v` + nombre de la instancia principal, singular (`vPartidaCotizacion`) — `vSyncPending` es la forma más cercana a la convención; recomiendo fijar ese nombre como el vigente y corregir el documento local.
- Ninguna de las tres tablas nuevas de `PConnectProquifaDotNet` lleva el campo de control `Activo` que el estándar exige en toda tabla. Es defendible como excepción (son tablas de auditoría/estado, no de negocio con baja lógica), pero vale la pena documentarla como tal en vez de dejarla como omisión silenciosa.

---

## 5. Recomendaciones

1. Actualizar `R16A-RE-FU-008-Legacy.md` y `R16A-RE-FU-008-Legacy-Tareas.md` (o reemplazarlos) para reflejar: alcance acotado a Canal ETLCobros, disparo vía API (no INSERT cross-base), `TaskSchedulerPqf` como storage de Hangfire, `NotificacionesClient`/`Proquifa.Pqf2Notificaciones` en vez de Brevo directo, y el proxy de archivos vía ProquifaDotNet en vez de MinIO directo.
2. Aclarar en el propio documento local la relación entre `R16A-NO-FU-003` y `R16A-RE-FU-008` (ver §2.2), para que quien abra la carpeta no asuma que es la fuente vigente. El DAR de orquestación (§2.3) no tiene este problema — ya está alineado con FU-008/Canal ETLCobros.
3. Confirmar con Juan David el endpoint de descarga de archivos en ProquifaDotNet antes de LB-T13 (`FileSyncService`), y resolver ETLCobros-T10 (`DoctoR`) contra `RYNL015` antes de cerrar el diseño de `BuzonCobros`.
4. Dar seguimiento a los dos pendientes que el propio DAR deja fuera de su alcance: la imprecisión sobre RabbitMQ en `TPSC-RE-FU-008-Contexto.md` (línea 113) y, sobre todo, la **contraseña hardcodeada** en `proquifa-punchout-backend/Infrastructure/RabbitMQ/RabbitMQSettings.cs` — es un hallazgo de seguridad real, no de arquitectura, y no encontré evidencia de que ya se haya reportado.
