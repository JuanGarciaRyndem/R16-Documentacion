### Creación de solución **ProquifaDotNet.LegacyBridge**

**Descripción:** Se generará una solución denominada **ProquifaDotNet.LegacyBridge**, desarrollada en **.NET Core 10**, cuyo objetivo será centralizar y gestionar la transferencia de datos desde ProquifaDotNet hacia el sistema Legacy (PCconnect), reemplazando los paquetes SSIS existentes y proveyendo una plataforma extensible, auditada y con reintentos automáticos para la sincronización de entidades entre ambos sistemas.

**Alcance funcional:** La solución incluirá las siguientes funcionalidades:
- **Jobs de sincronización por entidad**: Pedidos, Buzón de Cobros, Facturas, Notas de Crédito, PDFs.
- **Control de estado de sincronización**: registro por entidad con estados Pendiente / EnProceso / Completado / Error.
- **Reintentos automáticos**: clasificación Transient (timeout, deadlock) vs. Permanent (error de negocio) con backoff configurable.
- **Log estructurado por ejecución**: snapshot JSON del payload enviado, duración, resultado por registro.
- **Sincronización de archivos**: transferencia de archivos adjuntos (PDFs, adjuntos de buzón) desde MinIO al directorio Legacy.
- **Notificaciones de fallos**: alertas vía Brevo cuando un job agota reintentos o encuentra error Permanent.
- **API de monitoreo**: endpoints para consultar estado de sincronización y forzar reintentos sin acceso directo a BD.
- **Dashboard Hangfire**: monitoreo operativo de recurring jobs en ambiente dev/QA.

**Relación con otros sistemas:**
La integración de LegacyBridge con el ecosistema ProquifaDotNet se realiza de la siguiente manera:
- **ProquifaDotNet** (origen): LegacyBridge lee entidades mediante `ProquifaDotNetDbContext` (EF Core Scaffold), sin modificar el modelo ni los endpoints existentes.
- **ProquifaDotNet.Finanzas**: dispara transferencias E1–E3 y E6 (Buzón, Proforma, Factura, PDF) al completar los pasos del wizard de Validar Cobro.
- **PCconnect (Legacy)**: destino de las transferencias. LegacyBridge escribe en `PConnectDbContext` (EF Core Scaffold de la BD Legacy).
- **PConnectProquifaDotNet**: base de datos propia de control operativo — SyncControl, SyncJobLog, AppSettings y tablas Hangfire.
- Solo México transfiere a Legacy. Perú nunca ejecuta transferencias (validado en el servicio, no en el job).

**Objetivo principal:**
- Proveer una **plataforma de integración extensible** que reemplace los paquetes SSIS de PCconnect, con trazabilidad completa y reintentos automáticos.
- Garantizar la **consistencia de datos entre ProquifaDotNet y Legacy** con log auditables y política de errores configurable.
- Facilitar la **incorporación de nuevas entidades** sin cambios estructurales a la solución — cada requisito funcional agrega su job concreto sobre la infraestructura base.

## 📂 Estructura de la solución **ProquifaDotNet.LegacyBridge**

### 1. Capas principales

- **Domain**
    - Entidades: `SyncControl`, `SyncJobLog`, `AppSettings`.
    - Interfaces de repositorios: `IRepository<T>`, `ISyncJobLog`, `IExceptionClassifier`, `IFileSyncService`, `INotificationService`, `ISyncJobBase`.
    - Enums: `SyncEstado` (Pendiente/EnProceso/Completado/Error), `ExceptionType` (Transient/Permanent), `SyncEntidad`.
    - Excepciones de dominio: `SyncPermanentException`, `SyncTransientException`.
- **Application**
    - Implementación de **CQRS** (Commands/Queries).
    - Commands base: `MarcarEnProcesoCommand`, `MarcarCompletadoCommand`, `MarcarErrorCommand`, `ForzarReintentoCommand`.
    - Queries base: `ObtenerPendientesQuery`, `ObtenerEstadoSyncQuery`, `ObtenerLogsJobQuery`.
    - `SyncControlService`: gestión de ciclo de vida y elegibilidad de reintentos.
    - DTOs: `SyncControlDto`, `SyncJobLogDto`, `SyncPendienteDto`, `SyncResultadoDto`.
- **Infrastructure**
    - Persistencia con EF Core — 3 contextos independientes:
        - `ProquifaDotNetDbContext` — lectura de entidades origen.
        - `PConnectDbContext` — escritura en base de datos Legacy.
        - `LegacyBridgeDbContext` — control operativo en `PConnectProquifaDotNet`.
    - **Hangfire** como motor de recurring jobs con storage en `PConnectProquifaDotNet`.
    - `SyncJobBase` abstracto — flujo estándar reutilizable por todos los jobs de entidad.
    - `ExceptionClassifier` — clasifica Transient/Permanent para determinar estrategia de reintento.
    - `SyncJobLogService` — persiste log estructurado con snapshot JSON del payload.
    - `FileSyncService` — transfiere archivos desde MinIO a directorio Legacy por entidad.
    - `BrevoNotificationService` — alertas de fallos al equipo de operaciones.
    - Configuración de **IdentityServer** (autenticación/autorización).
    - Manejo de **Logs** con Serilog.
    - Base de datos `PConnectProquifaDotNet` (SyncControl, SyncJobLog, AppSettings, tablas Hangfire).
- **API**
    - Endpoints RESTful de monitoreo: estado por entidad, pendientes, historial de intentos, reintento manual, estado de jobs Hangfire.
    - Autenticación IdentityServer — misma infraestructura que Finanzas y Timbrado.
    - Documentación Swagger/OpenAPI.
- **Worker.LegacyBridge**
    - Host de Hangfire que procesa recurring jobs de sincronización.
    - `HealthCheckJob` — valida conectividad a las 3 bases de datos cada 5 minutos.
    - Jobs por entidad (se agregan en cada requisito funcional): Clientes, Pedidos, BuzonCobros, Factura, NotaCredito, etc.
    - Reintentos configurables con backoff desde `AppSettings` en BD (sin redeployar).
    - Notificación automática vía Brevo al agotar reintentos o encontrar error Permanent.
- **Testing**
    - Pruebas unitarias: `ExceptionClassifier`, `SyncControlService`, `SyncJobBase` (con mocks).
    - Pruebas de integración QA: ciclo E2E de job, conectividad a 3 BDs, FileSync, notificación Brevo.

### 2. Contextos de base de datos

| Contexto EF Core          | Base de datos          | Propósito                                                                                            |
| ------------------------- | ---------------------- | ---------------------------------------------------------------------------------------------------- |
| `ProquifaDotNetDbContext` | ProquifaDotNet         | Lectura de entidades origen (Clientes, Pedidos, Facturas, etc.) — Scaffold incremental por requisito |
| `PConnectDbContext`       | PConnect               | Escritura en tablas Legacy destino — Scaffold de tablas receptoras                                   |
| `LegacyBridgeDbContext`   | PConnectProquifaDotNet | SyncControl, SyncJobLog, AppSettings, vistas Hangfire y vw_SyncPendientes                            |

### 3. Flujo funcional

1. **Requisito funcional** (ej. RE-028 Validar Cobro Paso 3) completa una acción → INSERT en `SyncControl` con `Estado='Pendiente'`.
2. **Hangfire** detecta el registro en `vw_SyncPendientes` en el próximo ciclo del recurring job.
3. **SyncJobBase** marca `EnProceso` → ejecuta `EjecutarSyncAsync` del job concreto → escribe en `PConnectDbContext` (Legacy).
4. Si éxito → marca `Completado` + INSERT `SyncJobLog (Estado='Exito')`.
5. Si error → `ExceptionClassifier` clasifica: `Transient` → Hangfire reintenta con backoff | `Permanent` → marca `Error`, INSERT `SyncJobLog (Estado='Error')`, `BrevoNotificationService.NotificarFalloAsync`.
6. **FileSync** (si aplica) — transfiere archivos asociados al registro desde MinIO al directorio Legacy configurado.
7. **API Monitoreo** — el equipo de operaciones puede consultar estado en cualquier momento y forzar `ForzarReintentoCommand` si el registro está en error reintentable.

### 4. Estándares transversales

- **Mensajes de error**: catálogo con códigos únicos y respuestas JSON (`errorCode`, `message`, `details`).
- **Logs**: Serilog enriquecido con contexto `{Entidad, IdRegistro, Job, NumeroIntento}`.
- **Seguridad**: autenticación/autorización con IdentityServer — misma infraestructura que Finanzas y Timbrado.
- **Región**: solo México transfiere a Legacy. La evaluación se realiza en el servicio de sincronización, no en la configuración del job.
- **Configuración en runtime**: intervalos, reintentos y destinatarios de notificación son modificables desde `PConnectProquifaDotNet.AppSettings` sin redeployar.
