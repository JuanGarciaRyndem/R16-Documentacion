# Tareas BackEnd — TPSC-NO-FU-003
**Requisito:** Creación de Solución Base ProquifaDotNet.LegacyBridge
**Aplicativos:** ProquifaDotNet.LegacyBridge (.NET Core)

---

> **Orden de ejecución sugerido:** BD PConnectProquifaDotNet — tablas control + log (T1) → Scaffold solución + Domain (T2) → Application CQRS base (T3) → Infrastructure: 3 contextos EF Core + repositorio genérico (T4) → SyncJobLog + ExceptionClassifier (T5) → Hangfire + Worker.LegacyBridge (T6) → Brevo notificaciones fallos (T7) → FileSync mecanismo archivos (T8) → API Monitoreo (T9) → Testing base (T10).
>
> **Dependencias externas:** Acceso a servidor RYNL010 para crear `PConnectProquifaDotNet`. Scaffold de `ProquifaDotNet` y `PConnect` requiere acceso a ambas bases. Credenciales Brevo para notificaciones. Configuración IdentityServer para la nueva solución. Hangfire requiere base de datos propia (puede ser `PConnectProquifaDotNet`).
>
> **Contexto:** LegacyBridge reemplaza los paquetes SSIS de PCconnect que actualmente transfieren datos de ProquifaDotNet hacia Legacy. La solución provee la infraestructura base reutilizable; cada requisito funcional (RE-006, RE-008, RE-010/012, RE-028) agrega sus jobs y builders específicos sobre esta base.

---

## Resumen de tareas

| #   | Clave               | Título simple                                                                                    | Tipo | Aplicativo                  |
| --- | ------------------- | ------------------------------------------------------------------------------------------------ | ---- | --------------------------- |
| 1   | CREATE-TABL-M       | Crear BD PConnectProquifaDotNet — tablas SyncControl, SyncJobLog, AppSettings, vistas pendientes | BD   | PConnectProquifaDotNet      |
| 2   | CREATE-SOLUTION     | Scaffold solución ProquifaDotNet.LegacyBridge — estructura de proyectos + capa Domain            | Back | ProquifaDotNet.LegacyBridge |
| 3   | ALG-COMPLX-LOGIC    | Implementar Application layer — CQRS base, DTOs, interfaces de servicios                         | Back | ProquifaDotNet.LegacyBridge |
| 4   | SERV-TRANSACT       | Implementar Infrastructure — 3 contextos EF Core + repositorio genérico CRUD                     | Back | ProquifaDotNet.LegacyBridge |
| 5   | ALG-BASIC-LOGIC     | Implementar SyncJobLog + ExceptionClassifier (Transient/Permanent)                               | Back | ProquifaDotNet.LegacyBridge |
| 6   | CREATE-WORKER       | Implementar Hangfire + Worker.LegacyBridge — recurring jobs base + worker host                   | Back | ProquifaDotNet.LegacyBridge |
| 7   | SERV-TRANSACT       | Implementar servicio de notificaciones Brevo — alertas de fallos de integración                  | Back | ProquifaDotNet.LegacyBridge |
| 8   | SERV-TRANSACT       | Implementar mecanismo de sincronización de archivos (FileSync)                                   | Back | ProquifaDotNet.LegacyBridge |
| 9   | CREATE-API-ENDPOINT | Implementar API de monitoreo — endpoints consulta de estado y reintento                          | Back | ProquifaDotNet.LegacyBridge |
| 10  | QUERY-M             | Testing base — pruebas unitarias e integración de infraestructura LegacyBridge                   | Back | ProquifaDotNet.LegacyBridge |

---

## TAREA 1

**[ NO-FU-003 ] [ CREATE-TABL-M ] Crear BD PConnectProquifaDotNet — tablas de control, SyncJobLog y AppSettings**

**Aplicativos:**
ProquifaDotNet.LegacyBridge — Base de datos PConnectProquifaDotNet

**Módulos:**
Infraestructura de control de sincronización

**Consideraciones previas:**
- Base de datos nueva `PConnectProquifaDotNet` en el servidor `RYNL010`, dedicada al control operativo de LegacyBridge.
- No almacena datos de negocio — solo estado de sincronización, logs y configuración de la solución.
- `SyncControl` es la tabla genérica de control por entidad (Clientes, Pedidos, Buzón, etc.). Cada requisito funcional que se integre a LegacyBridge tendrá una fila por registro a sincronizar.
- `SyncJobLog` registra el resultado de cada ejecución de job con snapshot JSON del payload.
- `AppSettings` reemplaza la configuración de `appsettings.json` para valores modificables en runtime (reintentos, intervalos, flags de activación por entidad).
- Hangfire puede usar esta misma base de datos para sus tablas internas (confirmar en T6).
- Las vistas `vw_SyncPendientes` exponen los registros en estado `Pendiente` o `Error` con menos de `MaxReintentos` intentos — son el origen de los jobs Hangfire.

**Objetivo general:**
Crear la base de datos `PConnectProquifaDotNet` con las tablas de control, logs y configuración que sustentan la operación de LegacyBridge.

**Objetivos específicos:**
- Crear la base de datos `PConnectProquifaDotNet` en `RYNL010`.
- Ejecutar DDL para `SyncControl`, `SyncJobLog`, `AppSettings` con sus constraints, defaults e índices.
- Crear vista `vw_SyncPendientes` que expone registros en estado `Pendiente` o `Error` elegibles para reintento.
- Insertar AppSettings iniciales: `LegacyBridge:MaxReintentos` (default 3), `LegacyBridge:BackoffSegundos` (default 60), `LegacyBridge:NotificarFallosPermanentes` (default true).
- Verificar que todos los constraints, defaults, índices y registros iniciales son correctos.

**Resultado esperado:**
Base de datos `PConnectProquifaDotNet` creada y funcional. Los workers de LegacyBridge pueden leer pendientes desde `vw_SyncPendientes` y escribir resultados en `SyncJobLog` inmediatamente después de ejecutar esta tarea.

**Entregables:**
Scripts SQL:
- `CREATE DATABASE PConnectProquifaDotNet`
- `CREATE TABLE SyncControl` + índices
- `CREATE TABLE SyncJobLog` + índices
- `CREATE TABLE AppSettings` + índices
- `CREATE VIEW vw_SyncPendientes`
- `INSERT` AppSettings iniciales
- Script de validación

**Diccionario de datos:**

**Tabla: SyncControl**

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdSyncControl` | `uniqueidentifier` PK DEFAULT NEWSEQUENTIALID() | Identificador único |
| `Entidad` | `nvarchar(100)` NOT NULL | Nombre de la entidad (ej. `Clientes`, `PedidosCredito`, `BuzonCobros`) |
| `IdRegistroOrigen` | `uniqueidentifier` NOT NULL | PK del registro en ProquifaDotNet |
| `Estado` | `nvarchar(20)` NOT NULL DEFAULT `'Pendiente'` | `Pendiente` / `EnProceso` / `Completado` / `Error` |
| `NumeroIntentos` | `int` NOT NULL DEFAULT 0 | Cantidad de intentos realizados |
| `MaxReintentos` | `int` NOT NULL DEFAULT 3 | Máximo de reintentos configurado al crear el registro |
| `FechaUltimoIntento` | `datetime2(7)` NULL | Fecha del último intento de sincronización |
| `FechaCompletado` | `datetime2(7)` NULL | Fecha en que el estado llegó a `Completado` |
| `MotivoError` | `nvarchar(max)` NULL | Detalle del último error (mensaje de excepción) |
| `TipoError` | `nvarchar(20)` NULL | `Transient` / `Permanent` — clasificación del error |
| `FechaRegistro` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Fecha de creación del registro de control |
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
| `Estado` | `nvarchar(20)` NOT NULL | `Exito` / `Error` |
| `PayloadJson` | `nvarchar(max)` NULL | Snapshot JSON del payload enviado a Legacy |
| `RespuestaJson` | `nvarchar(max)` NULL | Respuesta de Legacy (si aplica) |
| `MensajeError` | `nvarchar(max)` NULL | Detalle del error (si Estado = `Error`) |
| `TipoError` | `nvarchar(20)` NULL | `Transient` / `Permanent` |
| `DuracionMs` | `int` NULL | Duración de la ejecución en milisegundos |
| `NumeroIntento` | `int` NOT NULL | Número de intento en esta ejecución |
| `FechaRegistro` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Fecha/hora de la ejecución |

Índices: `IX_SyncJobLog_IdSyncControl` (IdSyncControl), `IX_SyncJobLog_Entidad_Fecha` (Entidad, FechaRegistro DESC).

**Tabla: AppSettings**

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdAppSettings` | `uniqueidentifier` PK DEFAULT NEWID() | Identificador único |
| `Clave` | `nvarchar(200)` NOT NULL UNIQUE | Clave de configuración (ej. `LegacyBridge:MaxReintentos`) |
| `Valor` | `nvarchar(max)` NOT NULL | Valor de la configuración |
| `Descripcion` | `nvarchar(500)` NULL | Descripción del parámetro |
| `FechaUltimaActualizacion` | `datetime2(7)` NOT NULL DEFAULT SYSUTCDATETIME() | Última modificación |

**Vista: vw_SyncPendientes**

Expone registros de `SyncControl` donde `Estado IN ('Pendiente', 'Error')` y `NumeroIntentos < MaxReintentos`, ordenados por `FechaUltimoIntento ASC NULLS FIRST`.

**Criterios de aceptación:**
- [ ] Base de datos `PConnectProquifaDotNet` creada en `RYNL010`.
- [ ] Tablas `SyncControl`, `SyncJobLog` y `AppSettings` creadas con todos los constraints e índices.
- [ ] Vista `vw_SyncPendientes` creada y retorna registros elegibles correctamente.
- [ ] AppSettings iniciales insertados (`MaxReintentos`, `BackoffSegundos`, `NotificarFallosPermanentes`).
- [ ] Script de validación ejecutado sin errores.

**Recursos:**
- `TPSC-NO-FU-003.md` — secciones 5.2 (Sistema de Control) y 5.3 (SyncJobLog)
- Servidor: `RYNL010`

---

## TAREA 2

**[ NO-FU-003 ] [ CREATE-SOLUTION ] Scaffold solución ProquifaDotNet.LegacyBridge — estructura de proyectos y capa Domain**

**Aplicativos:**
ProquifaDotNet.LegacyBridge

**Módulos:**
Infraestructura base — Scaffold y Domain

**Consideraciones previas:**
- La solución sigue el mismo arquetipo de `ProquifaDotNet.Finanzas` y `ProquifaDotNet.Timbrado`: Domain / Application / Infrastructure / API / Worker / Testing.
- El proyecto `LegacyBridge.Domain` no tiene dependencias externas — solo tipos del framework y abstracciones propias.
- Las entidades de dominio son: `SyncControl`, `SyncJobLog`, `AppSettings` (espejo de BD), más interfaces `IRepository<T>`, `ISyncJobLog`, `IExceptionClassifier`, `IFileSyncService`, `INotificationService`.
- Los enums: `SyncEstado` (Pendiente/EnProceso/Completado/Error), `ExceptionType` (Transient/Permanent), `SyncEntidad` (Clientes/PedidosCredito/BuzonCobros/etc.).
- Crear el repositorio de Git para la solución y el pipeline de CI base.

**Objetivo general:**
Crear la solución `ProquifaDotNet.LegacyBridge` con la estructura de proyectos completa y la capa Domain implementada.

**Objetivos específicos:**
- Crear la solución en Visual Studio con los 6 proyectos: `LegacyBridge.Domain`, `LegacyBridge.Application`, `LegacyBridge.Infrastructure`, `LegacyBridge.API`, `LegacyBridge.Worker`, `LegacyBridge.Testing`.
- Establecer referencias entre proyectos (Domain ← Application ← Infrastructure ← API/Worker).
- Implementar `LegacyBridge.Domain`:
  - Entidades: `SyncControl`, `SyncJobLog`, `AppSettings`
  - Interfaces: `IRepository<T>`, `ISyncJobLog`, `IExceptionClassifier`, `IFileSyncService`, `INotificationService`, `ISyncJobBase`
  - Enums: `SyncEstado`, `ExceptionType`, `SyncEntidad`
  - Excepciones de dominio: `SyncPermanentException`, `SyncTransientException`
- Configurar `global.json` con versión de SDK, `.editorconfig`, `.gitignore`.
- Verificar que la solución compila sin errores.

**Resultado esperado:**
Solución `ProquifaDotNet.LegacyBridge` creada, con los 6 proyectos referenciados correctamente y la capa Domain implementada y compilando.

**Entregables:**
- Solución `.sln` con 6 proyectos
- `LegacyBridge.Domain` implementado (entidades, interfaces, enums, excepciones)
- Compilación sin errores en todos los proyectos
- Repositorio Git creado con pipeline CI base

**Criterios de aceptación:**
- [ ] Los 6 proyectos de la solución existen con las referencias correctas.
- [ ] `LegacyBridge.Domain` compila sin dependencias externas (solo framework).
- [ ] Interfaces `IRepository<T>`, `ISyncJobLog`, `IExceptionClassifier`, `IFileSyncService`, `INotificationService`, `ISyncJobBase` definidas en Domain.
- [ ] Enums `SyncEstado`, `ExceptionType`, `SyncEntidad` definidos.
- [ ] Excepciones `SyncPermanentException` y `SyncTransientException` definidas.
- [ ] La solución compila sin errores ni warnings críticos.

**Recursos:**
- `TPSC-NO-FU-003.md` — sección 4 (Estructura de la solución)
- Referencia arquitectónica: `ProquifaDotNet.Finanzas`

---

## TAREA 3

**[ NO-FU-003 ] [ ALG-COMPLX-LOGIC ] Implementar Application layer — CQRS base, DTOs y interfaces de servicios**

**Aplicativos:**
ProquifaDotNet.LegacyBridge

**Módulos:**
LegacyBridge.Application

**Consideraciones previas:**
- La capa Application implementa CQRS usando el patrón Command/Query con MediatR (o el equivalente definido por el equipo).
- Los Commands y Queries base son genéricos; cada entidad (Clientes, Pedidos, etc.) los extenderá en su requisito funcional correspondiente.
- Los DTOs base incluyen: `SyncControlDto`, `SyncJobLogDto`, `SyncPendienteDto`, `SyncResultadoDto`.
- Los servicios de aplicación son interfaces que la capa Infrastructure implementa — Application no tiene dependencias de Infrastructure.
- Incluir `SyncControlService` base: gestión de estado (marcar EnProceso, Completado, Error), incremento de intentos, evaluación de elegibilidad para reintento.

**Objetivo general:**
Implementar la capa Application de LegacyBridge con la infraestructura CQRS base y los DTOs necesarios para orquestar sincronizaciones.

**Objetivos específicos:**
- Configurar MediatR (o patrón equivalente) en `LegacyBridge.Application`.
- Implementar Commands base:
  - `MarcarEnProcesoCommand` — marca un `SyncControl` como EnProceso.
  - `MarcarCompletadoCommand` — marca como Completado, registra log de éxito.
  - `MarcarErrorCommand` — marca como Error, clasifica tipo, registra log, evalúa si es reintentable.
  - `ForzarReintentoCommand` — resetea estado a Pendiente para reintento manual desde API.
- Implementar Queries base:
  - `ObtenerPendientesQuery` — lista registros elegibles de `vw_SyncPendientes` por entidad.
  - `ObtenerEstadoSyncQuery` — estado actual de un registro específico.
  - `ObtenerLogsJobQuery` — historial de intentos de un registro.
- Implementar `SyncControlService` con lógica de transición de estados y validación de reintentos.
- Definir DTOs: `SyncControlDto`, `SyncJobLogDto`, `SyncPendienteDto`, `SyncResultadoDto`.

**Resultado esperado:**
La capa Application tiene los Commands, Queries y servicios base que permiten a Workers y API orquestar el ciclo de vida de una sincronización sin conocer detalles de Infrastructure.

**Entregables:**
- Commands: `MarcarEnProcesoCommand`, `MarcarCompletadoCommand`, `MarcarErrorCommand`, `ForzarReintentoCommand`
- Queries: `ObtenerPendientesQuery`, `ObtenerEstadoSyncQuery`, `ObtenerLogsJobQuery`
- `SyncControlService` con lógica de estados y reintentos
- DTOs: `SyncControlDto`, `SyncJobLogDto`, `SyncPendienteDto`, `SyncResultadoDto`
- `LegacyBridge.Application` compila sin errores

**Criterios de aceptación:**
- [ ] MediatR (o equivalente) configurado en Application.
- [ ] Los 4 Commands implementados con sus handlers.
- [ ] Las 3 Queries implementadas con sus handlers.
- [ ] `SyncControlService` implementa las transiciones de estado correctamente (Pendiente → EnProceso → Completado/Error).
- [ ] La lógica de elegibilidad para reintento (`NumeroIntentos < MaxReintentos` y `TipoError != Permanent`) está encapsulada en `SyncControlService`.
- [ ] `LegacyBridge.Application` compila sin referencias a Infrastructure.
- [ ] PR aprobado por líder técnico.

**Recursos:**
- `TPSC-NO-FU-003.md` — sección 5.2 (Sistema de Control)
- Tarea 1 (estructura de tablas `SyncControl` y `SyncJobLog`)

---

## TAREA 4

**[ NO-FU-003 ] [ SERV-TRANSACT ] Implementar Infrastructure — 3 contextos EF Core y repositorio genérico**

**Aplicativos:**
ProquifaDotNet.LegacyBridge

**Módulos:**
LegacyBridge.Infrastructure — Persistencia

**Consideraciones previas:**
- Los 3 contextos EF Core acceden a bases de datos distintas: `ProquifaDotNet` (origen), `PConnect` (Legacy destino), `PConnectProquifaDotNet` (control LegacyBridge).
- El Scaffold de `ProquifaDotNet` y `PConnect` genera las entidades necesarias. Se incluyen solo las tablas que LegacyBridge necesita leer o escribir — no el modelo completo.
- El Scaffold se actualiza incrementalmente conforme cada requisito funcional integra nuevas entidades.
- El repositorio genérico `Repository<T>` implementa `IRepository<T>` del Domain y es reutilizable para cualquier contexto.
- La cadena de conexión de `ProquifaDotNet` usa: `Data Source=RYNL010;Initial Catalog=ProquifaDotNet;Integrated Security=True;Persist Security Info=False;Pooling=False;MultipleActiveResultSets=False;Encrypt=False;TrustServerCertificate=True;Command Timeout=0`
- Registrar los 3 contextos y el repositorio genérico en el contenedor DI de la solución.

**Objetivo general:**
Implementar la capa de persistencia de LegacyBridge con los 3 contextos EF Core y un repositorio genérico reutilizable por todos los jobs y servicios.

**Objetivos específicos:**
- Configurar EF Core con los 3 contextos:
  - `ProquifaDotNetDbContext` — lectura de entidades origen (tablas a definir por cada requisito funcional).
  - `PConnectDbContext` — escritura en Legacy destino.
  - `LegacyBridgeDbContext` — lectura/escritura en `PConnectProquifaDotNet` (SyncControl, SyncJobLog, AppSettings, vw_SyncPendientes).
- Generar Scaffold inicial de `ProquifaDotNetDbContext` y `PConnectDbContext` con las tablas base (se amplía en cada requisito).
- Implementar `Repository<T>` genérico que implemente `IRepository<T>`:
  - `GetByIdAsync(Guid id)`
  - `QueryAsync(string sql, object? parameters)` — consultas SQL parametrizadas
  - `SaveOrUpdateAsync(T entity)`
  - `DeactivateAsync(Guid id)`
- Implementar `AppSettingsRepository` para leer valores de configuración desde BD en runtime.
- Registrar todo en DI (Program.cs / Startup).
- Verificar conectividad a las 3 bases de datos desde el ambiente de desarrollo.

**Resultado esperado:**
Los 3 contextos EF Core están configurados y el repositorio genérico está disponible para inyectarse en cualquier job o servicio de LegacyBridge.

**Entregables:**
- `ProquifaDotNetDbContext`, `PConnectDbContext`, `LegacyBridgeDbContext`
- Scaffold inicial de tablas base en cada contexto
- `Repository<T>` implementando `IRepository<T>`
- `AppSettingsRepository` con lectura en runtime
- Registro DI de los 3 contextos y repositorios
- Verificación de conectividad a las 3 bases de datos

**Criterios de aceptación:**
- [ ] Los 3 contextos EF Core conectan correctamente a sus bases de datos respectivas.
- [ ] `Repository<T>` funciona con `LegacyBridgeDbContext` para `SyncControl` y `SyncJobLog`.
- [ ] `AppSettingsRepository` lee valores de `AppSettings` en runtime sin reiniciar la aplicación.
- [ ] Scaffold inicial generado para `ProquifaDotNetDbContext` y `PConnectDbContext`.
- [ ] Todos los contextos y repositorios registrados en DI.
- [ ] PR aprobado por líder técnico.

**Recursos:**
- `TPSC-NO-FU-003.md` — sección 3.2 (Contextos de base de datos)
- Tarea 1 (DDL de `PConnectProquifaDotNet`)
- Cadena de conexión ProquifaDotNet: `Data Source=RYNL010;Initial Catalog=ProquifaDotNet;Integrated Security=True;...`

---

## TAREA 5

**[ NO-FU-003 ] [ ALG-BASIC-LOGIC ] Implementar SyncJobLog y ExceptionClassifier**

**Aplicativos:**
ProquifaDotNet.LegacyBridge

**Módulos:**
LegacyBridge.Infrastructure — Logging y clasificación de errores

**Consideraciones previas:**
- `SyncJobLogService` implementa `ISyncJobLog` del Domain. Persiste cada ejecución de job en la tabla `SyncJobLog` con snapshot JSON del payload.
- El snapshot JSON debe ser el payload exacto enviado a Legacy — no un resumen. Permite auditar qué datos se transfirieron en cada intento.
- `ExceptionClassifier` implementa `IExceptionClassifier` del Domain. La clasificación determina si se reintenta automáticamente (Transient) o se notifica y detiene (Permanent).
- La clasificación Transient/Permanent debe ser configurable vía `AppSettings` para agregar nuevos tipos sin redeployar.
- Serilog ya está configurado en la solución (desde T2). `SyncJobLogService` complementa Serilog con persistencia estructurada en BD.

**Objetivo general:**
Implementar el servicio de log estructurado por ejecución (`SyncJobLog`) y el clasificador de excepciones (`ExceptionClassifier`) que determinan el comportamiento de reintento ante fallos.

**Objetivos específicos:**
- Implementar `SyncJobLogService` (implementa `ISyncJobLog`):
  - `RegistrarExitoAsync(idSyncControl, nombreJob, payloadJson, respuestaJson, duracionMs, numeroIntento)`
  - `RegistrarErrorAsync(idSyncControl, nombreJob, payloadJson, excepcion, tipoError, duracionMs, numeroIntento)`
  - Persiste en tabla `SyncJobLog` vía `Repository<SyncJobLog>`.
- Implementar `ExceptionClassifier` (implementa `IExceptionClassifier`):
  - Clasifica por tipo de excepción: `TimeoutException`, `SqlException` (deadlock/timeout) → `Transient`; `InvalidOperationException`, violaciones de constraints, errores de negocio → `Permanent`.
  - Lista de tipos Transient configurable desde `AppSettings` (clave `LegacyBridge:ExcepcionesTransient`).
  - Método: `ExceptionType Clasificar(Exception ex)`
- Registrar `SyncJobLogService` y `ExceptionClassifier` en DI.
- Pruebas unitarias de `ExceptionClassifier` cubriendo casos Transient, Permanent y tipos no clasificados (default Permanent).

**Resultado esperado:**
Cada ejecución de job queda auditada en `SyncJobLog` con snapshot del payload. Las excepciones son clasificadas automáticamente para determinar la estrategia de reintento.

**Entregables:**
- `SyncJobLogService` implementando `ISyncJobLog`
- `ExceptionClassifier` implementando `IExceptionClassifier`
- Registro DI de ambos servicios
- Pruebas unitarias de `ExceptionClassifier` (Transient, Permanent, default)

**Criterios de aceptación:**
- [ ] `SyncJobLogService.RegistrarExitoAsync` persiste el log con payload JSON completo en `SyncJobLog`.
- [ ] `SyncJobLogService.RegistrarErrorAsync` persiste el log con excepción y tipo de error.
- [ ] `ExceptionClassifier.Clasificar` retorna `Transient` para timeout y deadlock SQL.
- [ ] `ExceptionClassifier.Clasificar` retorna `Permanent` para errores de negocio y violaciones de constraints.
- [ ] La lista de excepciones Transient es configurable desde `AppSettings` sin redeployar.
- [ ] Pruebas unitarias del clasificador aprobadas.
- [ ] PR aprobado por líder técnico.

**Recursos:**
- `TPSC-NO-FU-003.md` — secciones 5.3 (SyncJobLog) y 5.4 (ExceptionClassifier)
- Tarea 1 (estructura tabla `SyncJobLog`)
- Tarea 4 (Repository genérico)

---

## TAREA 6

**[ NO-FU-003 ] [ CREATE-WORKER ] Implementar Hangfire + Worker.LegacyBridge — recurring jobs base**

**Aplicativos:**
ProquifaDotNet.LegacyBridge

**Módulos:**
LegacyBridge.Infrastructure (Hangfire) + LegacyBridge.Worker

**Consideraciones previas:**
- Hangfire gestiona la ejecución y reintento de jobs. El `Worker.LegacyBridge` es el proceso host que ejecuta los jobs de Hangfire.
- Hangfire puede usar `PConnectProquifaDotNet` como base de datos de sus tablas internas (confirmar con DBA antes de implementar).
- El job base genérico `SyncJobBase` implementa `ISyncJobBase` e incluye el flujo estándar: leer pendientes → marcar EnProceso → ejecutar → registrar resultado → marcar Completado/Error.
- Cada entidad (Clientes, Pedidos, etc.) hereda de `SyncJobBase` e implementa solo el método `EjecutarSyncAsync(SyncPendienteDto pendiente)`.
- Los recurring jobs se configuran con intervalos desde `AppSettings` para no requerir redeployar al cambiar frecuencias.
- En esta tarea se implementa el job base y se registra un job de ejemplo/prueba — los jobs reales por entidad se agregan en cada requisito funcional.
- El Dashboard de Hangfire se habilita solo en ambiente de desarrollo/QA, protegido por autenticación en producción.

**Objetivo general:**
Configurar Hangfire como motor de jobs asíncronos e implementar el `Worker.LegacyBridge` con el job base genérico reutilizable por todas las entidades.

**Objetivos específicos:**
- Instalar y configurar Hangfire en `LegacyBridge.Infrastructure` y `LegacyBridge.Worker`:
  - Storage: `PConnectProquifaDotNet` (SQL Server).
  - Dashboard protegido por autenticación en producción.
  - Política de reintentos: usar clasificación de `ExceptionClassifier` para determinar si Hangfire reintenta o descarta.
- Implementar `SyncJobBase` (abstracto, implementa `ISyncJobBase`):
  - `ExecuteAsync(string entidad)` — flujo estándar: obtener pendientes → procesar cada uno → registrar en `SyncJobLog` → actualizar `SyncControl`.
  - Método abstracto `EjecutarSyncAsync(SyncPendienteDto pendiente)` — implementado por cada entidad concreta.
  - Manejo de errores: clasificar con `ExceptionClassifier` → si Transient, Hangfire reintenta; si Permanent, marcar Error definitivo + notificar.
- Registrar un recurring job de prueba (`HealthCheckJob`) con ejecución cada 5 minutos que valide conectividad a las 3 bases de datos.
- Configurar el host `Worker.LegacyBridge` como Windows Service o proceso .NET Worker.
- Verificar que Hangfire Dashboard es accesible y muestra el `HealthCheckJob` ejecutándose.

**Resultado esperado:**
Hangfire configurado y ejecutando. `SyncJobBase` disponible para que cada requisito funcional extienda su job concreto. `HealthCheckJob` corriendo en ciclo y reportando estado de conectividad.

**Entregables:**
- Hangfire configurado con storage `PConnectProquifaDotNet`
- `SyncJobBase` abstracto con flujo estándar implementado
- `HealthCheckJob` recurring job de prueba/validación
- `Worker.LegacyBridge` configurado como host ejecutable
- Dashboard Hangfire habilitado (dev/QA) con autenticación
- Instrucciones de despliegue del Worker como Windows Service

**Criterios de aceptación:**
- [ ] Hangfire usa `PConnectProquifaDotNet` como storage y sus tablas internas fueron creadas.
- [ ] `SyncJobBase` implementa el flujo estándar: pendientes → EnProceso → ejecución → log → Completado/Error.
- [ ] Errores Transient: Hangfire reintenta automáticamente con backoff configurable.
- [ ] Errores Permanent: se marca Error definitivo en `SyncControl` y se dispara notificación (T7).
- [ ] `HealthCheckJob` ejecuta cada 5 minutos y valida conectividad a las 3 BDs.
- [ ] Dashboard de Hangfire accesible en dev/QA y protegido en producción.
- [ ] PR aprobado por líder técnico.

**Recursos:**
- `TPSC-NO-FU-003.md` — sección 5.6 (Jobs Hangfire Base)
- Tarea 1 (BD PConnectProquifaDotNet)
- Tarea 4 (Repository genérico)
- Tarea 5 (SyncJobLog + ExceptionClassifier)

---

## TAREA 7

**[ NO-FU-003 ] [ SERV-TRANSACT ] Implementar servicio de notificaciones Brevo — alertas de fallos de integración**

**Aplicativos:**
ProquifaDotNet.LegacyBridge

**Módulos:**
LegacyBridge.Infrastructure — Notificaciones

**Consideraciones previas:**
- El servicio implementa `INotificationService` del Domain. Se invoca desde `SyncJobBase` cuando un error es clasificado como Permanent o cuando un registro agota sus reintentos.
- Usa Brevo (mismo proveedor que el resto de ProquifaDotNet) con la configuración de credenciales desde `AppSettings` o `PConnectProquifaDotNet.AppSettings`.
- El correo de alerta incluye: entidad afectada, `IdRegistroOrigen`, número de intentos, tipo de error, mensaje de excepción y enlace al endpoint de monitoreo para reintento manual (T9).
- La lista de destinatarios es configurable desde `AppSettings` (`LegacyBridge:NotificacionDestinatarios`).
- Si el envío de la notificación falla, se loguea en Serilog pero NO se lanza excepción — la notificación es best-effort para no bloquear el flujo del job.

**Objetivo general:**
Implementar el servicio de notificación de fallos de integración vía Brevo, que alerta al equipo de operaciones cuando una sincronización falla definitivamente.

**Objetivos específicos:**
- Implementar `BrevoNotificationService` (implementa `INotificationService`):
  - Método `NotificarFalloAsync(SyncResultadoDto resultado)` — envía correo de alerta vía Brevo API.
  - Incluir en el cuerpo: entidad, `IdRegistroOrigen`, número de intentos, tipo de error, mensaje, enlace al endpoint de reintento.
  - Leer destinatarios desde `AppSettings` (`LegacyBridge:NotificacionDestinatarios`).
  - Leer credenciales Brevo desde `AppSettings` (`LegacyBridge:Brevo:ApiKey`, `LegacyBridge:Brevo:CorreoEmisor`).
- Implementar plantilla HTML del correo de alerta (inline o desde `AppSettings`).
- Manejo de fallo en envío: loguear en Serilog + continuar, sin propagar excepción.
- Registrar `BrevoNotificationService` en DI.
- Prueba de integración: enviar notificación de prueba a destinatario de QA y verificar recepción.

**Resultado esperado:**
Cuando un job marca un error Permanent o agota reintentos, el equipo de operaciones recibe automáticamente un correo vía Brevo con el detalle del fallo y el enlace para reintento manual.

**Entregables:**
- `BrevoNotificationService` implementando `INotificationService`
- Plantilla HTML del correo de alerta
- Registro DI del servicio
- Prueba de integración: correo enviado y recibido en QA

**Criterios de aceptación:**
- [ ] `BrevoNotificationService.NotificarFalloAsync` envía correo via Brevo con entidad, IdRegistro, intentos, error y enlace de reintento.
- [ ] Los destinatarios se leen desde `AppSettings` — no hardcodeados.
- [ ] Un fallo en el envío del correo no propaga excepción al job (best-effort).
- [ ] El fallo en notificación queda registrado en Serilog.
- [ ] Prueba de integración exitosa: correo recibido en destinatario de QA.
- [ ] PR aprobado por líder técnico.

**Recursos:**
- `TPSC-NO-FU-003.md` — sección 5.5 (Servicio de Notificación de Fallos)
- `TPSC-NO-FU-001.md` — referencia de integración Brevo en ProquifaDotNet
- Tarea 6 (SyncJobBase — punto de invocación de la notificación)

---

## TAREA 8

**[ NO-FU-003 ] [ SERV-TRANSACT ] Implementar mecanismo de sincronización de archivos (FileSync)**

**Aplicativos:**
ProquifaDotNet.LegacyBridge

**Módulos:**
LegacyBridge.Infrastructure — FileSync

**Consideraciones previas:**
- Algunas entidades sincronizadas tienen archivos relacionados: Buzón de Cobros tiene archivos adjuntos en MinIO (`ArchivoCorreoRecibido`), Factura tiene PDFs, etc.
- El mecanismo FileSync es genérico y reutilizable — no está acoplado a ninguna entidad específica.
- Los archivos origen pueden estar en: MinIO (bucket `mailbot` u otros) o en la base de datos ProquifaDotNet.
- El destino de los archivos en Legacy es un directorio de red configurado por entidad en `AppSettings`.
- FileSync tiene su propio log por archivo dentro del `SyncJobLog` del registro padre — un archivo fallido no cancela los demás ni el registro de datos.
- Política de reintento de archivos: independiente de la política del registro padre. Configurable via `AppSettings` (`LegacyBridge:FileSync:MaxReintentos`).
- La interfaz `IFileSyncService` ya está definida en Domain (T2).

**Objetivo general:**
Implementar el servicio de sincronización de archivos relacionados con las entidades que LegacyBridge transfiere, con log independiente por archivo y política de reintento propia.

**Objetivos específicos:**
- Implementar `FileSyncService` (implementa `IFileSyncService`):
  - `SincronizarArchivosAsync(IEnumerable<ArchivoSyncDto> archivos, string directorioDestino)` — descarga archivos desde MinIO y los copia al directorio Legacy configurado.
  - `ArchivoSyncDto`: `{ IdArchivo, NombreArchivo, UrlOrigen, DirectorioDestino }`.
  - Descarga desde MinIO: usando `WebClient` o `HttpClient` con la URL del archivo.
  - Copia al directorio destino: `File.Copy` o equivalente al path de red Legacy.
  - Por cada archivo: registrar resultado (éxito/error) en `SyncJobLog` como entrada individual.
  - Un archivo fallido no cancela los demás archivos ni el registro de datos del job padre.
- Implementar `MinIOFileResolver`: obtiene la URL de descarga de un archivo dado su `IdArchivo` desde `ProquifaDotNetDbContext`.
- Configurar directorio destino por entidad en `AppSettings` (ej. `LegacyBridge:FileSync:DirectorioClientes`, `LegacyBridge:FileSync:DirectorioBuzonCobros`).
- Política de reintento por archivo: `MaxReintentos` desde `AppSettings`, independiente del registro padre.
- Registrar `FileSyncService` y `MinIOFileResolver` en DI.

**Resultado esperado:**
Los archivos relacionados con entidades sincronizadas se transfieren al directorio Legacy correspondiente. Un archivo fallido queda registrado en el log sin bloquear la sincronización de los demás archivos ni del registro de datos.

**Entregables:**
- `FileSyncService` implementando `IFileSyncService`
- `MinIOFileResolver` para obtener URLs de archivos desde `ProquifaDotNetDbContext`
- `ArchivoSyncDto`
- Configuración de directorios destino en `AppSettings`
- Registro DI de ambos servicios

**Criterios de aceptación:**
- [ ] `FileSyncService.SincronizarArchivosAsync` descarga archivos desde MinIO y los copia al directorio Legacy configurado.
- [ ] Cada archivo tiene su propio log en `SyncJobLog` con resultado individual.
- [ ] Un archivo con error no cancela los demás archivos ni el registro de datos del job padre.
- [ ] El directorio destino es configurable por entidad desde `AppSettings`.
- [ ] `MinIOFileResolver` obtiene la URL del archivo desde `ProquifaDotNetDbContext.Archivo`.
- [ ] PR aprobado por líder técnico.

**Recursos:**
- `TPSC-NO-FU-003.md` — sección 5.8 (Sincronización de Archivos)
- `TPSC-RE-FU-008-Back.md` — referencia: `ArchivoCorreoRecibido`, `ArchivoBO`, `ArchivoDetalle.Url`
- Tarea 4 (ProquifaDotNetDbContext)
- Tarea 5 (SyncJobLog para log por archivo)

---

## TAREA 9

**[ NO-FU-003 ] [ CREATE-API-ENDPOINT ] Implementar API de monitoreo — endpoints de consulta de estado y reintento**

**Aplicativos:**
ProquifaDotNet.LegacyBridge

**Módulos:**
LegacyBridge.API — Monitoreo

**Consideraciones previas:**
- La API de monitoreo es la interfaz operativa de LegacyBridge: permite consultar estado de sincronización y forzar reintentos sin acceso directo a la base de datos.
- Los endpoints usan los Commands y Queries de Application (T3) — la API no accede directamente a la BD.
- La autenticación es via IdentityServer (mismo que ProquifaDotNet.Finanzas y ProquifaDotNet.Timbrado).
- El endpoint `/sync/reintentar/{idSyncControl}` solo funciona si el registro está en estado `Error` con `TipoError != Permanent`. Si es Permanent, retorna 400 con mensaje explicativo.
- Documentar la API con Swagger/OpenAPI.

**Objetivo general:**
Implementar la API RESTful de LegacyBridge que expone endpoints de monitoreo para consultar el estado de sincronizaciones y forzar reintentos manuales.

**Objetivos específicos:**
- Implementar `MonitoreoController` con los siguientes endpoints:
  - `GET /sync/status/{entidad}` — resumen de estado por entidad: total Pendiente, EnProceso, Completado, Error.
  - `GET /sync/pendientes/{entidad}` — lista paginada de registros en estado Pendiente o Error con detalles.
  - `GET /sync/log/{idSyncControl}` — historial de intentos de un registro específico.
  - `POST /sync/reintentar/{idSyncControl}` — fuerza reintento de un registro en error (dispatch `ForzarReintentoCommand`).
  - `GET /sync/jobs` — estado de todos los recurring jobs de Hangfire (nombre, última ejecución, próxima ejecución, estado).
- Configurar autenticación IdentityServer en la API.
- Documentar con Swagger/OpenAPI (anotaciones XML y esquemas de respuesta).
- Configurar CORS y rate limiting básico.

**Resultado esperado:**
Los endpoints de monitoreo son accesibles y funcionales. El equipo de operaciones puede consultar el estado de cualquier entidad y forzar reintentos desde la API sin acceso directo a BD.

**Entregables:**
- `MonitoreoController` con 5 endpoints implementados
- Autenticación IdentityServer configurada
- Documentación Swagger/OpenAPI generada
- CORS y rate limiting configurados

**Criterios de aceptación:**
- [ ] `GET /sync/status/{entidad}` retorna conteo por estado correctamente.
- [ ] `GET /sync/pendientes/{entidad}` retorna lista paginada de registros elegibles.
- [ ] `GET /sync/log/{idSyncControl}` retorna historial completo de intentos.
- [ ] `POST /sync/reintentar/{idSyncControl}` resetea el estado a Pendiente y Hangfire lo procesa en el siguiente ciclo.
- [ ] `POST /sync/reintentar` retorna 400 si el registro es `Permanent` con mensaje explicativo.
- [ ] `GET /sync/jobs` retorna estado de todos los recurring jobs de Hangfire.
- [ ] La API requiere autenticación IdentityServer — requests sin token retornan 401.
- [ ] Swagger documentado y accesible en dev/QA.
- [ ] PR aprobado por líder técnico.

**Recursos:**
- `TPSC-NO-FU-003.md` — sección 5.7 (API de Monitoreo)
- Tarea 3 (Commands y Queries de Application)
- Tarea 6 (Hangfire — estado de jobs)

---

## TAREA 10

**[ NO-FU-003 ] [ QUERY-M ] Testing base — pruebas unitarias e integración de infraestructura LegacyBridge**

**Aplicativos:**
ProquifaDotNet.LegacyBridge

**Módulos:**
LegacyBridge.Testing

**Consideraciones previas:**
- Las pruebas de esta tarea validan la infraestructura base — no los jobs específicos por entidad (esos se prueban en cada requisito funcional).
- Pruebas unitarias: `ExceptionClassifier`, `SyncControlService` (transiciones de estado), `SyncJobBase` (flujo genérico con mocks).
- Pruebas de integración: conectividad a las 3 BDs, ciclo completo de un job de prueba en QA, envío de notificación, FileSync con un archivo de prueba.
- Las pruebas de integración requieren acceso a ambiente QA — no se ejecutan en CI en cada commit, solo en pipeline de validación de release.

**Objetivo general:**
Validar mediante pruebas unitarias e integración que la infraestructura base de LegacyBridge funciona correctamente antes de que los requisitos funcionales empiecen a agregar jobs específicos.

**Objetivos específicos:**
- Pruebas unitarias:
  - `ExceptionClassifierTests`: casos Transient (timeout, deadlock), Permanent (negocio, constraint), tipo no clasificado (default Permanent).
  - `SyncControlServiceTests`: transiciones de estado válidas e inválidas, elegibilidad de reintento.
  - `SyncJobBaseTests`: flujo estándar con mock de ejecución exitosa y con error Transient/Permanent.
- Pruebas de integración (ambiente QA):
  - Conectividad a `ProquifaDotNet`, `PConnect`, `PConnectProquifaDotNet`.
  - Ciclo completo de `HealthCheckJob`: ejecución → log en `SyncJobLog` → estado en `SyncControl`.
  - `FileSyncService`: descarga de archivo de prueba desde MinIO → copia a directorio Legacy de prueba.
  - Notificación Brevo: envío de correo de alerta de prueba → verificar recepción.
  - API Monitoreo: `GET /sync/jobs` retorna estado del `HealthCheckJob`.
- Documentar resultados de pruebas de integración con evidencias.

**Resultado esperado:**
Todas las pruebas unitarias pasan. Las pruebas de integración en QA confirman que la infraestructura base de LegacyBridge está operativa y lista para recibir los jobs de requisitos funcionales.

**Entregables:**
- Suite de pruebas unitarias: `ExceptionClassifierTests`, `SyncControlServiceTests`, `SyncJobBaseTests`
- Suite de pruebas de integración QA con evidencias
- Informe de resultados con cobertura de casos

**Criterios de aceptación:**
- [ ] Todas las pruebas unitarias pasan sin errores.
- [ ] Conectividad a las 3 bases de datos validada en QA.
- [ ] Ciclo completo de `HealthCheckJob` verificado: log en `SyncJobLog` y estado en `SyncControl` correctos.
- [ ] `FileSyncService` transfiere archivo de prueba desde MinIO a directorio Legacy en QA.
- [ ] Notificación Brevo recibida en destinatario de prueba.
- [ ] `GET /sync/jobs` retorna estado del `HealthCheckJob` desde la API.
- [ ] Informe de resultados documentado y aprobado por el líder técnico.

**Recursos:**
- Todos los entregables de T1 a T9
- Acceso a ambiente QA con las 3 bases de datos
- Acceso a MinIO y directorio Legacy en QA
