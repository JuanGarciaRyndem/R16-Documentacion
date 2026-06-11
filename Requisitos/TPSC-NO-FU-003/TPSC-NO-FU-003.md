# TPSC-NO-FU-003 — Creación de Solución Base ProquifaDotNet.LegacyBridge

**Tipo:** No Funcional
**Estado:** Análisis
**Fecha:** 2026-06-11

---

## 1. Objetivo

Crear la solución base **ProquifaDotNet.LegacyBridge** (.NET Core), cuyo propósito es centralizar y gestionar la transferencia de datos desde ProquifaDotNet hacia el sistema Legacy (PCconnect), reemplazando los paquetes SSIS existentes y proveyendo una plataforma extensible para comunicación bidireccional futura entre ambos sistemas.

La solución actúa como puente de integración entre ProquifaDotNet y Legacy, con soporte para reintentos, logs auditables, notificaciones de fallos y monitoreo operativo.

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
| Buzón Cobros ETL (E1) | RE-028 | Solo México |
| Proforma (E2) | RE-028 | Solo México |
| Factura + PDF (E3 + E6) | RE-028 | Solo México |

---

## 3. Solución Propuesta — ProquifaDotNet.LegacyBridge

### 3.1 Stack tecnológico

| Componente | Tecnología |
|---|---|
| Framework | .NET Core (versión alineada con ProquifaDotNet.Finanzas y ProquifaDotNet.Timbrado) |
| API | RESTful con ASP.NET Core |
| ORM | Entity Framework Core |
| Jobs asíncronos | Hangfire |
| Notificaciones | Brevo (mismo servicio que el resto de ProquifaDotNet) |
| Autenticación | IdentityServer |
| Logs | Serilog |

### 3.2 Contextos de base de datos

| Contexto EF Core | Base de datos | Propósito |
|---|---|---|
| `ProquifaDotNetContext` | ProquifaDotNet | Lectura de entidades origen (Clientes, Pedidos, Facturas, etc.) |
| `PConnectContext` | PConnect | Lectura/escritura en base de datos Legacy |
| `PConnectProquifaDotNetContext` | PConnectProquifaDotNet | Tablas de control de sincronización y logs propios de LegacyBridge |

---

## 4. Estructura de la Solución

```
ProquifaDotNet.LegacyBridge/
├── LegacyBridge.Domain/
│   ├── Entities/               # Entidades de dominio (SyncJobLog, SyncControl, etc.)
│   ├── Enums/                  # ExceptionType (Transient/Permanent), SyncStatus
│   └── Interfaces/             # IRepository<T>, ISyncJobLog, IExceptionClassifier
│
├── LegacyBridge.Application/
│   ├── CQRS/                   # Commands y Queries por entidad
│   ├── DTOs/                   # Payloads de transferencia por entidad
│   ├── Services/               # ISyncService, IFileSyncService, INotificationService
│   └── Validators/             # Validaciones de negocio antes de transferir
│
├── LegacyBridge.Infrastructure/
│   ├── Persistence/
│   │   ├── ProquifaDotNetContext/   # Scaffold tablas origen
│   │   ├── PConnectContext/         # Scaffold tablas Legacy destino
│   │   └── PConnectProquifaDotNetContext/  # Tablas de control LegacyBridge
│   ├── Repositories/           # Repositorio genérico CRUD + queries SQL
│   ├── Jobs/                   # Configuración Hangfire: recurring jobs base
│   ├── FileSync/               # Mecanismo de sincronización de archivos
│   ├── Notifications/          # Integración Brevo para notificación de fallos
│   └── Logging/                # SyncJobLog — logs con snapshots JSON
│
├── LegacyBridge.API/
│   ├── Controllers/
│   │   ├── MonitoreoController     # Consulta estado de jobs, reintentos manuales
│   │   └── SyncController          # Disparar sincronizaciones bajo demanda
│   └── Program.cs / appsettings
│
├── LegacyBridge.Worker/
│   └── Workers/                # Procesos asíncronos que revisan colas Hangfire
│
└── LegacyBridge.Testing/
    ├── Unit/                   # Pruebas unitarias: builders, clasificador, repositorio
    └── Integration/            # Pruebas integración: flujos E2E en QA
```

---

## 5. Componentes Clave

### 5.1 Repositorio Genérico

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

### 5.2 Sistema de Control de Sincronización

Tablas en `PConnectProquifaDotNet` que registran el estado de sincronización por entidad. Incluye vistas de pendientes para que Hangfire identifique qué registros están sin sincronizar o con error.

| Tabla | Propósito |
|---|---|
| `SyncControl_<Entidad>` | Estado por registro: Pendiente / EnProceso / Completado / Error |
| `vw_SyncPendientes_<Entidad>` | Vista que expone registros en estado Pendiente o Error reintentable |

### 5.3 SyncJobLog

Log estructurado por ejecución de job. Cada ejecución registra:
- Entidad sincronizada, cantidad de registros procesados, cantidad con error
- Snapshot JSON del payload enviado a Legacy
- Resultado por registro (éxito / error + detalle)
- Duración de la ejecución

### 5.4 ExceptionClassifier

Clasifica las excepciones ocurridas durante la transferencia para determinar la estrategia de reintento:

| Tipo | Descripción | Acción |
|---|---|---|
| `Transient` | Error temporal (timeout, conexión, lock) | Reintento automático con backoff |
| `Permanent` | Error de datos o negocio (violación FK, valor inválido) | No reintento — notificación inmediata |

### 5.5 Servicio de Notificación de Fallos (Brevo)

Cuando un job agota sus reintentos o encuentra un error permanente, envía notificación vía Brevo al equipo de operaciones con:
- Entidad afectada, identificador del registro fallido
- Tipo de error y mensaje
- Número de intentos realizados
- Enlace al endpoint de monitoreo para reintento manual

### 5.6 Jobs Hangfire Base

Configuración de recurring jobs para cada entidad sincronizable. En la versión base se configura la infraestructura y el job genérico; las implementaciones por entidad se agregan en cada requisito funcional.

```csharp
// Ejemplo de registro de job recurring
RecurringJob.AddOrUpdate<ISyncJobBase>(
    "sync-clientes",
    job => job.ExecuteAsync("Clientes"),
    Cron.Minutely(5));
```

### 5.7 API de Monitoreo

Endpoints RESTful para consultar el estado de sincronización y disparar reintentos sin necesidad de acceso directo a la base de datos:

| Endpoint | Método | Descripción |
|---|---|---|
| `/sync/status/{entidad}` | GET | Estado actual de sincronización por entidad |
| `/sync/pendientes/{entidad}` | GET | Lista de registros pendientes o con error |
| `/sync/reintentar/{idRegistro}` | POST | Forzar reintento de un registro específico |
| `/sync/jobs` | GET | Estado de todos los recurring jobs de Hangfire |

### 5.8 Sincronización de Archivos

Mecanismo para transferir archivos relacionados con las entidades sincronizadas (ej. archivos adjuntos de Buzón de Cobros, PDFs de Factura). La sincronización de archivos se ejecuta en el mismo job que la entidad propietaria, con log independiente por archivo y política de reintento propia.

---

## 6. Estándares Transversales

| Aspecto | Estándar |
|---|---|
| Mensajes de error | Catálogo con códigos únicos, respuesta JSON `{errorCode, message, details}` |
| Logs | Serilog enriquecido con contexto: `{Entidad, IdRegistro, Job, Usuario}` |
| Autenticación | IdentityServer — misma infraestructura que Finanzas y Timbrado |
| Región | Perú nunca transfiere a Legacy. El corte se evalúa en el servicio, no en el job |
| Reintentos | Configurables por AppSettings: `LegacyBridge:MaxReintentos`, `LegacyBridge:BackoffSegundos` |

---

## 7. Dependencias

| Dependencia             | Descripción                                                                    |
| ----------------------- | ------------------------------------------------------------------------------ |
| RE-002 al RE-006        | Campos de Catálogo de Clientes que LegacyBridge transfiere                     |
| RE-008                  | Buzón de Cobros y archivos adjuntos                                            |
| RE-010, RE-011, RE-012  | Pedidos Crédito y variantes (pago contra entrega, controlados, FAA)            |
| RE-028                  | ETL eventos E1/E2/E3/E6 (Buzón, Proforma, Factura, PDF)                        |
| TPSC-NO-FU-001          | Brevo: patrón de notificación de fallos (referencia de integración)            |
| ProquifaDotNet.Finanzas | Comparte patrón de arquitectura (Domain/Application/Infrastructure/API/Worker) |
