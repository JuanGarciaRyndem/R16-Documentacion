# TPSC-NO-FU-003 — Creación de Solución Base ProquifaDotNet.LegacyBridge

**Tipo:** No Funcional
**Estado:** Análisis
**Fecha:** 2026-06-11

---

## 1. Objetivo

Crear la solución base **ProquifaDotNet.LegacyBridge** (.NET Core 10), cuyo propósito es centralizar y gestionar la transferencia de datos desde ProquifaDotNet hacia el sistema Legacy (PCconnect), reemplazando los paquetes SSIS existentes y proveyendo una plataforma extensible, auditada y con reintentos automáticos para la sincronización de entidades entre ambos sistemas.

La solución actúa como puente de integración entre ProquifaDotNet y Legacy, con soporte para reintentos configurables, logs estructurados por ejecución (con snapshot JSON del payload), notificaciones de fallos vía Brevo y API de monitoreo operativo.

---

## 2. Contexto y Problema

### 2.1 Situación actual

La transferencia de datos de ProquifaDotNet hacia el sistema Legacy se realiza actualmente mediante paquetes SSIS en PCconnect. Este mecanismo presenta las siguientes limitaciones:

| Limitación | Descripción |
|---|---|
| Sin visibilidad de errores | Los fallos en SSIS no generan notificaciones ni logs estructurados accesibles desde el aplicativo |
| Sin reintentos automáticos | Un fallo en la transferencia requiere intervención manual para reejecutar |
| Difícil de versionar | Los paquetes `.dtsx` no se integran fácilmente con el flujo de desarrollo Git del aplicativo |
| Sin extensibilidad | Agregar nuevas entidades o variantes requiere modificar paquetes externos a ProquifaDotNet |
| Unidireccional por diseño | SSIS no facilita la comunicación inversa (Legacy → ProquifaDotNet) en caso de requerirse |

### 2.2 Alcance inicial

En esta primera versión, la comunicación es **ProquifaDotNet → Legacy** únicamente. La arquitectura está diseñada para soportar comunicación bidireccional en futuras iteraciones sin cambios estructurales a la solución.

### 2.3 Entidades que migran de SSIS a LegacyBridge

Las siguientes transferencias, actualmente en SSIS, serán migradas progresivamente a LegacyBridge conforme avancen los requisitos funcionales:

| Entidad | Requisito origen | Observaciones |
|---|---|---|
| Clientes (Datos Generales, Direcciones, Contactos, Datos Legales, Fiscal, Cobros, Referencia Bancaria) | RE-002 al RE-006 | Solo México |
| Buzón de Cobros (datos + archivos adjuntos) | RE-008 | Solo México |
| Pedidos Crédito (base, pago contra entrega, controlados, FAA) | RE-010, RE-011, RE-012 | Solo México |
| Buzón Cobros ETL — E1 | RE-028 | Solo México |
| Proforma — E2 | RE-028 | Solo México |
| Factura + PDF — E3 + E6 | RE-028 | Solo México |
| Complemento de Pago + PDF — E4 + E7 | RE-030 | Solo México |
| Nota de Crédito + PDF — E5 + E8 | RE-032 | Solo México |

---

## 3. Relación con Otros Sistemas

| Sistema | Rol | Canal de integración |
|---|---|---|
| **ProquifaDotNet** | Origen de datos — entidades a sincronizar | EF Core Scaffold (`ProquifaDotNetDbContext`) — solo lectura |
| **ProquifaDotNet.Finanzas** | Disparador de eventos E1–E6 al completar wizard Validar Cobro | INSERT en `SyncControl` desde Finanzas |
| **PCconnect (Legacy)** | Destino de sincronización | EF Core Scaffold (`PConnectDbContext`) — escritura |
| **PConnectProquifaDotNet** | BD de control operativo propia de LegacyBridge | EF Core (`LegacyBridgeDbContext`) — lectura/escritura |
| **MinIO** | Almacenamiento de archivos origen (PDFs, adjuntos) | HTTP download vía `FileSyncService` |
| **Brevo** | Envío de notificaciones de fallos de integración | API Brevo (mismo proveedor que el ecosistema ProquifaDotNet) |
| **IdentityServer** | Autenticación/autorización para la API de monitoreo | JWT Bearer — misma infraestructura que Finanzas y Timbrado |
| **Hangfire** | Motor de jobs asíncronos y reintentos | SQL Server storage en `PConnectProquifaDotNet` |

> **Regla de negocio crítica:** Solo México transfiere datos a Legacy. Perú nunca ejecuta transferencias de LegacyBridge. La evaluación de región se realiza en el servicio de sincronización, no en la configuración del job.

---

## 4. Solución Propuesta — ProquifaDotNet.LegacyBridge

### 4.1 Stack tecnológico

| Componente | Tecnología |
|---|---|
| Framework | .NET Core 10 (alineado con ProquifaDotNet.Finanzas y ProquifaDotNet.Timbrado) |
| API | RESTful con ASP.NET Core |
| ORM | Entity Framework Core (3 contextos independientes) |
| Jobs asíncronos | Hangfire |
| Notificaciones | Brevo (mismo proveedor que el resto del ecosistema) |
| Autenticación | IdentityServer |
| Logs | Serilog |
| Almacenamiento de archivos | MinIO (origen) → directorio de red Legacy (destino) |

### 4.2 Contextos de base de datos

| Contexto EF Core | Base de datos | Propósito |
|---|---|---|
| `ProquifaDotNetDbContext` | ProquifaDotNet | Lectura de entidades origen (Clientes, Pedidos, Facturas, etc.) — Scaffold incremental por requisito |
| `PConnectDbContext` | PConnect | Escritura en tablas Legacy destino — Scaffold de tablas receptoras |
| `LegacyBridgeDbContext` | PConnectProquifaDotNet | SyncControl, SyncJobLog, AppSettings, vw_SyncPendientes y tablas internas de Hangfire |

---

## 5. Estructura de la Solución

```
ProquifaDotNet.LegacyBridge/
├── LegacyBridge.Domain/
│   ├── Entities/               # SyncControl, SyncJobLog, AppSettings
│   ├── Enums/                  # SyncEstado, ExceptionType, SyncEntidad
│   ├── Exceptions/             # SyncPermanentException, SyncTransientException
│   └── Interfaces/             # IRepository<T>, ISyncJobLog, IExceptionClassifier,
│                               # IFileSyncService, INotificationService, ISyncJobBase
│
├── LegacyBridge.Application/
│   ├── CQRS/
│   │   ├── Commands/           # MarcarEnProcesoCommand, MarcarCompletadoCommand,
│   │   │                       # MarcarErrorCommand, ForzarReintentoCommand
│   │   └── Queries/            # ObtenerPendientesQuery, ObtenerEstadoSyncQuery,
│   │                           # ObtenerLogsJobQuery
│   ├── DTOs/                   # SyncControlDto, SyncJobLogDto, SyncPendienteDto,
│   │                           # SyncResultadoDto, ArchivoSyncDto
│   └── Services/               # SyncControlService (ciclo de vida + reintentos)
│
├── LegacyBridge.Infrastructure/
│   ├── Persistence/
│   │   ├── ProquifaDotNetContext/    # Scaffold tablas origen (incremental)
│   │   ├── PConnectContext/          # Scaffold tablas Legacy destino
│   │   └── LegacyBridgeContext/      # SyncControl, SyncJobLog, AppSettings, vistas
│   ├── Repositories/           # Repository<T> genérico + AppSettingsRepository
│   ├── Jobs/
│   │   ├── SyncJobBase.cs      # Flujo estándar abstracto reutilizable
│   │   └── HealthCheckJob.cs   # Validación de conectividad a 3 BDs (cada 5 min)
│   ├── Hangfire/               # Configuración Hangfire: storage, dashboard, políticas
│   ├── FileSync/               # FileSyncService + MinIOFileResolver
│   ├── Notifications/          # BrevoNotificationService
│   ├── Logging/                # SyncJobLogService — log estructurado por ejecución
│   └── ExceptionClassification/ # ExceptionClassifier (Transient/Permanent)
│
├── LegacyBridge.API/
│   ├── Controllers/
│   │   └── MonitoreoController # /sync/status, /sync/pendientes, /sync/log,
│   │                           # /sync/reintentar, /sync/jobs
│   └── Program.cs / appsettings
│
├── LegacyBridge.Worker/
│   └── Workers/                # Host Hangfire: HealthCheckJob + jobs por entidad
│
└── LegacyBridge.Testing/
    ├── Unit/                   # ExceptionClassifierTests, SyncControlServiceTests,
    │                           # SyncJobBaseTests
    └── Integration/            # Ciclo E2E, conectividad 3 BDs, FileSync, Brevo
```

---

## 6. Flujo Funcional

```
Requisito funcional (RE-028 / RE-030 / RE-032 / etc.)
    │
    └─ Acción completada → INSERT SyncControl (Estado='Pendiente')
                                │
                   [Hangfire - próximo ciclo del recurring job]
                                │
                   SyncJobBase.ExecuteAsync
                      ├─ Leer vw_SyncPendientes
                      ├─ Marcar EnProceso
                      ├─ EjecutarSyncAsync (job concreto por entidad)
                      │     └─ Escribe en PConnectDbContext (Legacy)
                      │
                      ├─ Éxito → Marcar Completado
                      │          INSERT SyncJobLog (Estado='Exito', PayloadJson)
                      │
                      └─ Error → ExceptionClassifier.Clasificar(ex)
                                    ├─ Transient → Hangfire reintenta con backoff
                                    │              INSERT SyncJobLog (Estado='Error', NumeroIntento++)
                                    │
                                    └─ Permanent → Marcar Error definitivo
                                                   INSERT SyncJobLog (Estado='Error', TipoError='Permanent')
                                                   BrevoNotificationService.NotificarFalloAsync

Si hay archivos asociados (PDF, adjuntos):
    FileSyncService.SincronizarArchivosAsync
        ├─ Descarga desde MinIO
        ├─ Copia a directorio Legacy configurado por entidad
        └─ Log individual por archivo (no cancela el registro padre si falla)

API Monitoreo (operaciones manuales):
    GET  /sync/status/{entidad}         → Resumen conteo por estado
    GET  /sync/pendientes/{entidad}     → Lista paginada elegibles
    GET  /sync/log/{idSyncControl}      → Historial de intentos
    POST /sync/reintentar/{idSyncControl} → ForzarReintentoCommand
    GET  /sync/jobs                     → Estado recurring jobs Hangfire
```

---

## 7. Componentes Clave

### 7.1 Repositorio Genérico

Provee operaciones CRUD base sobre cualquier entidad, queries SQL parametrizados y modelos genéricos de respuesta. Evita duplicar acceso a datos en cada job o servicio de sincronización.

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IEnumerable<T>> QueryAsync(string sql, object? parameters = null);
    Task<Guid> SaveOrUpdateAsync(T entity);
    Task<bool> DeactivateAsync(Guid id);
}
```

### 7.2 Sistema de Control de Sincronización

Tablas en `PConnectProquifaDotNet` que registran el estado de sincronización por entidad. Incluye vistas de pendientes para que Hangfire identifique qué registros están sin sincronizar o con error reintentable.

| Tabla / Vista | Propósito |
|---|---|
| `SyncControl` | Estado por registro: Pendiente / EnProceso / Completado / Error |
| `SyncJobLog` | Log estructurado por ejecución: snapshot JSON del payload, resultado, duración |
| `AppSettings` | Configuración en runtime (MaxReintentos, BackoffSegundos, destinatarios notificación) |
| `vw_SyncPendientes` | Expone registros en estado Pendiente o Error con `NumeroIntentos < MaxReintentos` |

### 7.3 SyncJobLog

Log estructurado por ejecución de job. Cada ejecución registra:
- Entidad sincronizada, cantidad de registros procesados, cantidad con error
- Snapshot JSON del payload enviado a Legacy
- Resultado por registro (éxito / error + detalle)
- Duración de la ejecución en milisegundos
- Número de intento en la secuencia de reintentos

### 7.4 ExceptionClassifier

Clasifica las excepciones ocurridas durante la transferencia para determinar la estrategia de reintento:

| Tipo | Descripción | Acción |
|---|---|---|
| `Transient` | Error temporal (timeout, conexión, deadlock SQL) | Reintento automático con backoff configurable |
| `Permanent` | Error de datos o negocio (violación FK, valor inválido, error lógico) | Sin reintento — notificación inmediata vía Brevo |

La lista de tipos de excepción Transient es configurable desde `AppSettings` sin redeployar.

### 7.5 SyncJobBase — Job Abstracto Genérico

Implementa el flujo estándar de sincronización. Cada entidad hereda y solo implementa `EjecutarSyncAsync`:

```csharp
public abstract class SyncJobBase : ISyncJobBase
{
    // Flujo estándar implementado aquí:
    // ObtenerPendientes → MarcarEnProceso → EjecutarSync → MarcarCompletado/Error
    public async Task ExecuteAsync(string entidad) { ... }

    // Solo este método implementa cada job concreto:
    protected abstract Task EjecutarSyncAsync(SyncPendienteDto pendiente);
}
```

### 7.6 Servicio de Notificación de Fallos (Brevo)

Cuando un job agota sus reintentos o encuentra un error Permanent, envía notificación vía Brevo al equipo de operaciones con:
- Entidad afectada e identificador del registro fallido
- Tipo de error y mensaje de excepción
- Número de intentos realizados
- Enlace al endpoint de monitoreo para reintento manual

Si el envío del correo falla, se loguea en Serilog pero **no se lanza excepción** — la notificación es best-effort para no bloquear el flujo del job.

### 7.7 FileSyncService — Sincronización de Archivos

Mecanismo genérico para transferir archivos relacionados con las entidades (PDFs de factura, adjuntos de Buzón de Cobros) desde MinIO al directorio Legacy configurado:

- Directorio destino configurable por entidad desde `AppSettings`
- Log independiente por archivo dentro del `SyncJobLog` del registro padre
- Un archivo fallido **no cancela** los demás archivos ni el registro de datos
- Política de reintento por archivo independiente de la política del registro padre

### 7.8 API de Monitoreo

| Endpoint | Método | Descripción |
|---|---|---|
| `/sync/status/{entidad}` | GET | Resumen de estado por entidad (conteo Pendiente/EnProceso/Completado/Error) |
| `/sync/pendientes/{entidad}` | GET | Lista paginada de registros elegibles para reintento |
| `/sync/log/{idSyncControl}` | GET | Historial completo de intentos de un registro |
| `/sync/reintentar/{idSyncControl}` | POST | Fuerza reintento (solo si `TipoError != Permanent`) |
| `/sync/jobs` | GET | Estado de todos los recurring jobs de Hangfire |

---

## 8. Impacto en Base de Datos

### 8.1 Base de datos nueva: PConnectProquifaDotNet

Base de datos propia de LegacyBridge creada en el servidor `RYNL010`. No almacena datos de negocio — solo control operativo, logs y configuración.

**Tabla: SyncControl**

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdSyncControl` | `uniqueidentifier` PK DEFAULT NEWSEQUENTIALID() | Identificador único |
| `Entidad` | `nvarchar(100)` NOT NULL | Nombre de la entidad (Clientes, PedidosCredito, BuzonCobros, etc.) |
| `IdRegistroOrigen` | `uniqueidentifier` NOT NULL | PK del registro en ProquifaDotNet |
| `Estado` | `nvarchar(20)` NOT NULL DEFAULT `'Pendiente'` | Pendiente / EnProceso / Completado / Error |
| `NumeroIntentos` | `int` NOT NULL DEFAULT 0 | Cantidad de intentos realizados |
| `MaxReintentos` | `int` NOT NULL DEFAULT 3 | Máximo de reintentos configurado al crear el registro |
| `FechaUltimoIntento` | `datetime2(7)` NULL | Fecha del último intento de sincronización |
| `FechaCompletado` | `datetime2(7)` NULL | Fecha en que el estado llegó a Completado |
| `MotivoError` | `nvarchar(max)` NULL | Detalle del último error (mensaje de excepción) |
| `TipoError` | `nvarchar(20)` NULL | Transient / Permanent |
| `FechaRegistro` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Fecha de creación |
| `FechaUltimaActualizacion` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Última modificación |

Índices: `IX_SyncControl_Entidad_Estado` (Entidad, Estado, NumeroIntentos), `IX_SyncControl_IdRegistroOrigen` (IdRegistroOrigen).

**Tabla: SyncJobLog**

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdSyncJobLog` | `uniqueidentifier` PK DEFAULT NEWSEQUENTIALID() | Identificador único |
| `IdSyncControl` | `uniqueidentifier` NOT NULL FK → SyncControl | Registro de control asociado |
| `Entidad` | `nvarchar(100)` NOT NULL | Nombre de la entidad sincronizada |
| `IdRegistroOrigen` | `uniqueidentifier` NOT NULL | PK del registro en ProquifaDotNet |
| `NombreJob` | `nvarchar(200)` NOT NULL | Nombre del job Hangfire que ejecutó |
| `Estado` | `nvarchar(20)` NOT NULL | Exito / Error |
| `PayloadJson` | `nvarchar(max)` NULL | Snapshot JSON del payload enviado a Legacy |
| `RespuestaJson` | `nvarchar(max)` NULL | Respuesta de Legacy (si aplica) |
| `MensajeError` | `nvarchar(max)` NULL | Detalle del error |
| `TipoError` | `nvarchar(20)` NULL | Transient / Permanent |
| `DuracionMs` | `int` NULL | Duración de la ejecución en milisegundos |
| `NumeroIntento` | `int` NOT NULL | Número de intento en esta ejecución |
| `FechaRegistro` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Fecha/hora de la ejecución |

Índices: `IX_SyncJobLog_IdSyncControl` (IdSyncControl), `IX_SyncJobLog_Entidad_Fecha` (Entidad, FechaRegistro DESC).

**Tabla: AppSettings**

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdAppSettings` | `uniqueidentifier` PK DEFAULT NEWID() | Identificador único |
| `Clave` | `nvarchar(200)` NOT NULL UNIQUE | Clave de configuración |
| `Valor` | `nvarchar(max)` NOT NULL | Valor de la configuración |
| `Descripcion` | `nvarchar(500)` NULL | Descripción del parámetro |
| `FechaUltimaActualizacion` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Última modificación |

AppSettings iniciales: `LegacyBridge:MaxReintentos` (3), `LegacyBridge:BackoffSegundos` (60), `LegacyBridge:NotificarFallosPermanentes` (true), `LegacyBridge:NotificacionDestinatarios`.

**Vista: vw_SyncPendientes**

Expone registros de `SyncControl` donde `Estado IN ('Pendiente', 'Error')` y `NumeroIntentos < MaxReintentos`, ordenados por `FechaUltimoIntento ASC NULLS FIRST`. Es el origen de todos los recurring jobs de Hangfire.

### 8.2 Scaffolds en ProquifaDotNet y PConnect

El Scaffold de `ProquifaDotNetDbContext` y `PConnectDbContext` se construye de forma incremental conforme cada requisito funcional integra nuevas entidades. En la versión base (NO-FU-003) solo se genera el scaffold mínimo requerido para el `HealthCheckJob` y la conectividad inicial.

### 8.3 Cadena de conexión ProquifaDotNet

```
Data Source=RYNL010;Initial Catalog=ProquifaDotNet;Integrated Security=True;Persist Security Info=False;Pooling=False;MultipleActiveResultSets=False;Encrypt=False;TrustServerCertificate=True;Command Timeout=0
```

---

## 9. Estándares Transversales

| Aspecto | Estándar |
|---|---|
| Mensajes de error | Catálogo con códigos únicos, respuesta JSON `{errorCode, message, details}` |
| Logs | Serilog enriquecido con contexto: `{Entidad, IdRegistro, Job, NumeroIntento, Usuario}` |
| Autenticación | IdentityServer — misma infraestructura que Finanzas y Timbrado |
| Región | Perú nunca transfiere a Legacy. El corte se evalúa en el servicio, no en el job |
| Configuración runtime | MaxReintentos, BackoffSegundos, destinatarios y flags de activación por entidad desde `AppSettings` en BD |

---

## 10. Dependencias

| Dependencia | Descripción |
|---|---|
| RE-002 al RE-006 | Campos de Catálogo de Clientes que LegacyBridge transfiere |
| RE-008 | Buzón de Cobros y archivos adjuntos |
| RE-010, RE-011, RE-012 | Pedidos Crédito y variantes |
| RE-028 | ETL eventos E1/E2/E3/E6 (Buzón, Proforma, Factura, PDF) |
| RE-030 | Complemento de Pago — E4/E7 |
| RE-032 | Nota de Crédito México — E5/E8 |
| TPSC-NO-FU-001 | Brevo: patrón de notificación de fallos (referencia de integración) |
| ProquifaDotNet.Finanzas | Comparte patrón de arquitectura (Domain/Application/Infrastructure/API/Worker) |

---

## 11. Referencias

- `Soluciones Nuevas/ProquifaDotNet.LegacyBridge.md` — documento de solución con alcance funcional y estructura
- `TPSC-NO-FU-003-Tareas.md` — 10 tareas de implementación (BD → Domain → Application → Infrastructure → Testing)
- `Endpoints/Endpoints-LegacyBridge.md` — transferencias ETL E1–E8 documentadas
- `TPSC-NO-FU-001.md` — integración Brevo (patrón de referencia)
- `TPSC-RE-FU-028.md` — disparadores de eventos E1/E2/E3/E6
