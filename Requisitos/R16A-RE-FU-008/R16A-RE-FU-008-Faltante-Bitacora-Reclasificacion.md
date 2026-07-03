# R16A-RE-FU-008 — Faltante respecto al Canvas (Bitácora + Reclasificación + Acciones Masivas)

| Campo | Valor |
|---|---|
| **Requisito base** | R16A-RE-FU-008 — Buzón de Cobros (+ buzones transversales) |
| **Insumo** | Canvas `10 - Flujo Mailbot Buzones y Modulos Consumidores.canvas` |
| **Fecha de análisis** | 2026-06-29 |
| **Autor** | Juan David García Cruz |
| **Documentos comparados** | `R16A-RE-FU-008_BD.md` v1.0, `R16A-RE-FU-008-P2-Back.md` v1.0, `R16A-RE-FU-008-v2_Propuesta2.md` |
| **Dominio dueño** | **Mailbot** — toda la lógica de Buzones, Bitácora Mailbot y reclasificación vive en la solución `MailbotWorker` (.NET 10), no se reparte entre los módulos consumidores |
| **Decisiones cerradas** | 2026-06-29 — Pendientes Q1–Q8 cerrados (sección 9 al final del documento) |

---

## 0. Confirmación de dominio y catálogo de buzones

### 0.1 Todo el dominio en Mailbot

Todo lo relacionado con **Buzones** (clasificación, reclasificación, acciones masivas, vincular OC, cambiar cliente, eliminar uno o muchos, spam) y la **Bitácora del Mailbot** (correos procesados / no procesados / Otros, traza de vínculo a buzón / cliente / contacto) vive en la solución **`MailbotWorker`** (.NET 10), no se reparte entre los módulos consumidores (Cotización, Atender PC, Validar Pago, Validar Ajuste).

Los módulos consumidores solo:
- **Consultan** los pendientes generados (lectura desde su propia tabla — `fccFolioPagoCliente`, `tpPedido`, `tpCotizacion`, etc.).
- **Invocan** los endpoints del Mailbot para reclasificar o eliminar el pendiente cuando el usuario operativo así lo decide desde la pantalla del módulo.

Esto centraliza la lógica transversal en un solo aplicativo, evita duplicación y permite que la auditoría unificada (Bitácora) sea fuente única.

### 0.2 Catálogo de buzones — estado en BD

Revisé `C:\Users\juan.garcia\Documents\R16-Documentacion\Database\ProquifaDotNet.sql`. **No existe una tabla `catBuzon` ni equivalente.** Los buzones se infieren hoy de la combinación:

- `catProceso` (línea 29630) — procesos de negocio (Cobros, Cotización, Pedido, Requisición, Otros, etc.).
- `catClasificacionCorreoRecibido` (línea 29030) — clasificación + flags por rol (`EVI`, `EVE`, `ESAC`, `AnalistaDeCuentasPorCobrar`, `CoordinadorDeServicioAlCliente`) + `Posicion`.
- `RegionConfiguracionMailBot` (línea 24425) — configuración de la cuenta de correo y etiquetas por región.

**Problema actual:** la noción de "buzón" es implícita. El Mailbot debe tener lógica hardcodeada para saber:
- Qué tipo de pendiente genera cada buzón (Cobros → `fccFolioPagoCliente`; Atender PC → `tpPedido`; Cotización → `tpCotizacion`; …).
- Qué módulo consumidor lo recibe.
- Qué acciones soporta cada buzón (reclasificar, eliminar pendiente, masivas, vincular OC, etc.).
- Qué política de cancelación aplicar al pendiente origen al reclasificar.

### 0.3 ~~Propuesta — Crear `catBuzon` (catálogo nuevo)~~ — **DESCARTADA (2026-06-29, Q7/Q8)**

> **Decisión final:** NO se crea `catBuzon`. Se reutiliza el catálogo existente `catClasificacionCorreoRecibido`, que ya contiene las 7 clasificaciones operativas: **Queja, Pago, Otros, Requisición, Pedido, Solicitud de información, Sugerencia**. Las metadatos que iba a tener `catBuzon` (`TablaPendiente`, `PoliticaCancelacion`, banderas `Permite*`) se resuelven en código dentro de cada estrategia de cancelación (`IPendienteCanceladorStrategy`), una por proceso de negocio. El frontend conoce las acciones disponibles por convención del buzón, no por configuración en BD.
>
> La propuesta de DDL para `catBuzon` que aparecía a continuación queda como **referencia histórica** y NO debe ejecutarse. Las Tareas que la implementaban (T13 catBuzon, T19 endpoint QueryResult) también quedan **canceladas**.

#### Propuesta histórica (DESCARTADA — solo referencia)

```sql
CREATE TABLE dbo.catBuzon (
    IdCatBuzon                       uniqueidentifier NOT NULL
        CONSTRAINT PK_catBuzon PRIMARY KEY CLUSTERED
        CONSTRAINT DF_catBuzon_Id DEFAULT (NEWID()),

    Clave                            varchar(80)      NOT NULL,    -- 'cobros' | 'cotizacion' | 'atenderpc' | 'validarajuste' | 'otros' | ...
    Nombre                           varchar(150)     NOT NULL,    -- 'Buzón de Cobros' | 'Buzón de Cotización' | ...
    Descripcion                      varchar(500)     NULL,

    IdCatProceso                     uniqueidentifier NOT NULL
        CONSTRAINT FK_catBuzon_Proceso
            FOREIGN KEY REFERENCES dbo.catProceso(IdCatProceso),
    IdCatClasificacionCorreoRecibido uniqueidentifier NOT NULL
        CONSTRAINT FK_catBuzon_Clasificacion
            FOREIGN KEY REFERENCES dbo.catClasificacionCorreoRecibido(IdCatClasificacionCorreoRecibido),

    -- Pendiente que genera este buzón (apunta a la tabla del módulo consumidor)
    TablaPendiente                   varchar(80)      NULL,    -- 'fccFolioPagoCliente' | 'tpPedido' | 'tpCotizacion' | etc.
    GeneraPendienteAutomatico        bit              NOT NULL  -- 1 si al clasificar se crea pendiente automáticamente
        CONSTRAINT DF_catBuzon_GeneraAuto DEFAULT (1),

    -- Acciones soportadas por el buzón (banderas)
    PermiteReclasificar              bit              NOT NULL
        CONSTRAINT DF_catBuzon_Reclasificar DEFAULT (1),
    PermiteEliminarPendiente         bit              NOT NULL
        CONSTRAINT DF_catBuzon_Eliminar DEFAULT (1),
    PermiteAccionMasiva              bit              NOT NULL
        CONSTRAINT DF_catBuzon_Masiva DEFAULT (0),
    PermiteVincularOC                bit              NOT NULL
        CONSTRAINT DF_catBuzon_VincularOC DEFAULT (0),
    PermiteCambiarCliente            bit              NOT NULL
        CONSTRAINT DF_catBuzon_CambiarCliente DEFAULT (1),

    -- Política de cancelación del pendiente al reclasificar (Q1 — pendiente confirmar)
    PoliticaCancelacion              varchar(40)      NOT NULL  -- 'ActivoCero' | 'EstadoCancelado' | 'NoCancela'
        CONSTRAINT DF_catBuzon_PoliticaCancel DEFAULT ('ActivoCero'),

    Orden                            int              NOT NULL
        CONSTRAINT DF_catBuzon_Orden DEFAULT (100),
    Activo                           bit              NOT NULL
        CONSTRAINT DF_catBuzon_Activo DEFAULT (1),
    FechaRegistro                    datetime         NOT NULL
        CONSTRAINT DF_catBuzon_FechaRegistro DEFAULT (GETDATE()),
    FechaUltimaActualizacion         datetime         NOT NULL
        CONSTRAINT DF_catBuzon_FechaActualizacion DEFAULT (GETDATE())
);

CREATE UNIQUE NONCLUSTERED INDEX UIX_catBuzon_Clave
    ON dbo.catBuzon (Clave) WHERE Activo = 1;
CREATE NONCLUSTERED INDEX IX_catBuzon_Clasificacion
    ON dbo.catBuzon (IdCatClasificacionCorreoRecibido);
```

**Datos iniciales propuestos:**

| Clave | Nombre | TablaPendiente | GeneraAuto | Reclasif | Eliminar | Masiva | VincularOC | CambiarCliente | PolíticaCancel |
|---|---|---|---|---|---|---|---|---|---|
| `cobros` | Buzón de Cobros | `fccFolioPagoCliente` | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | `ActivoCero` |
| `cotizacion` | Buzón de Cotización | `tpCotizacion` | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ | `EstadoCancelado` |
| `atenderpc` | Buzón de Atender PC | `tpPedido` | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ | `EstadoCancelado` |
| `validarajuste` | Buzón de Validar Ajuste | `tpPedido` | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ | `EstadoCancelado` |
| `otros` | Buzón Otros | (ninguno) | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ | `NoCancela` |
| `requisicion` | Buzón de Requisición | (a confirmar) | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ | `ActivoCero` |
| `solicituddeinformacion` | Buzón de Información | (ninguno) | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ | `NoCancela` |
| `queja` | Buzón de Queja | (ninguno) | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ | `NoCancela` |
| `sugerencia` | Buzón de Sugerencia | (ninguno) | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ | `NoCancela` |

**Beneficios de tener `catBuzon` explícito:**

1. **Mailbot deja de hardcodear** qué pendiente genera cada buzón — consulta `catBuzon.TablaPendiente`.
2. **El frontend de cada buzón** consulta `catBuzon.Permite*` para mostrar/ocultar acciones, sin redeploy.
3. **`BuzonReclasificacionBO`** lee `catBuzon.PoliticaCancelacion` y aplica la estrategia correspondiente sin saber el detalle del proceso (Q1 se resuelve por configuración, no por código).
4. **`BitacoraCorreoMailbot.IdCatClasificacionCorreoRecibido`** + `catBuzon` permite consultas tipo "todos los eventos del buzón Cobros independientemente del proceso interno".
5. **Onboarding de un nuevo buzón** se vuelve un INSERT + configuración, no un cambio de código.

> **Decisión recomendada:** crear `catBuzon` como tabla de configuración. La alternativa sería extender `catClasificacionCorreoRecibido` con las 9 columnas nuevas, pero esa tabla ya está cargada con flags de rol y mezclarle config de buzón degrada su semántica. Mejor tabla nueva con FK a la clasificación.

---

## 1. Síntesis del faltante

El requisito 008 cubre hoy bien la captura de correos (Mailbot.Api + Worker + IA) y el Buzón de Cobros como módulo consumidor (Validar Cobro). Pero el canvas muestra que **Buzones es un hub transversal** que alimenta también a Cotización, Atender PC, Validar Pago y Validar Ajuste, y que se necesitan **5 capacidades nuevas** que hoy no están documentadas:

| # | Capacidad faltante | Estado actual en 008 |
|---|---|---|
| F-1 | Endpoint en Mailbot para reclasificación con **cancelación atómica del pendiente anterior** | Solo UPDATE de clasificación; no cancela el pendiente origen |
| F-2 | Bitácora de correos procesados con **vínculo buzón ↔ cliente ↔ contacto** | Existen `MailbotEventoCorreo` y `MailbotClasificacionLog` (IA), pero no la traza operativa unificada |
| F-3 | **Vista** de bitácora consolidada para consulta | No existe |
| F-4 | Sección **"Otros"** con reclasificar/inactivar individual y masivo | Solo individual |
| F-5 | Reclasificar/eliminar pendiente desde **cada buzón consumidor** (no solo Cobros) | Solo Cobros documentado |

---

## 2. Impacto en Base de Datos

### 2.1 Tabla nueva — `BitacoraCorreoMailbot`

**Propósito:** Registrar cada evento operativo de un correo procesado: a qué buzón se integró, qué cliente y contacto se vincularon, y el estado del pendiente generado. Es la traza unificada que faltaba.

```sql
CREATE TABLE dbo.BitacoraCorreoMailbot (
    IdBitacoraCorreoMailbot      uniqueidentifier NOT NULL
        CONSTRAINT PK_BitacoraCorreoMailbot PRIMARY KEY CLUSTERED
        CONSTRAINT DF_BitacoraCorreoMailbot_Id DEFAULT (NEWID()),

    IdCorreoRecibido             uniqueidentifier NOT NULL
        CONSTRAINT FK_BitacoraCorreoMailbot_CorreoRecibido
            FOREIGN KEY REFERENCES dbo.CorreoRecibido(IdCorreoRecibido),
    IdCorreoRecibidoCliente      uniqueidentifier NULL
        CONSTRAINT FK_BitacoraCorreoMailbot_CorreoRecibidoCliente
            FOREIGN KEY REFERENCES dbo.CorreoRecibidoCliente(IdCorreoRecibidoCliente),

    -- Vínculo con el buzón / clasificación destino
    IdCatClasificacionCorreoRecibido uniqueidentifier NOT NULL
        CONSTRAINT FK_BitacoraCorreoMailbot_Clasificacion
            FOREIGN KEY REFERENCES dbo.catClasificacionCorreoRecibido(IdCatClasificacionCorreoRecibido),
    IdCatProceso                 uniqueidentifier NOT NULL
        CONSTRAINT FK_BitacoraCorreoMailbot_Proceso
            FOREIGN KEY REFERENCES dbo.catProceso(IdCatProceso),

    -- Cliente y contacto vinculados (nullable: puede ser correo no identificable)
    IdCliente                    uniqueidentifier NULL
        CONSTRAINT FK_BitacoraCorreoMailbot_Cliente
            FOREIGN KEY REFERENCES dbo.Cliente(IdCliente),
    IdContactoCliente            uniqueidentifier NULL
        CONSTRAINT FK_BitacoraCorreoMailbot_ContactoCliente
            FOREIGN KEY REFERENCES dbo.ContactoCliente(IdContactoCliente),

    -- Pendiente generado (polimórfico — guid sin FK rígida)
    IdPendienteGenerado          uniqueidentifier NULL,
    TablaPendiente               varchar(80)      NULL,    -- 'fccFolioPagoCliente' | 'tpPedido' | 'tpCotizacion' | etc.

    -- Estado operativo
    EstadoProcesamiento          varchar(40)      NOT NULL,    -- 'Procesado' | 'NoProcesado' | 'Reclasificado' | 'Inactivado' | 'Recuperado'
    MotivoNoProcesado            varchar(400)     NULL,
    EventoOrigen                 varchar(60)      NOT NULL,    -- 'MailbotInicial' | 'ReclasificacionManual' | 'AccionMasiva' | 'EliminacionPendiente'

    IdUsuarioOperacion           uniqueidentifier NULL
        CONSTRAINT FK_BitacoraCorreoMailbot_Usuario
            FOREIGN KEY REFERENCES dbo.Usuario(IdUsuario),
    IdRegion                     uniqueidentifier NOT NULL
        CONSTRAINT FK_BitacoraCorreoMailbot_Region
            FOREIGN KEY REFERENCES dbo.Region(IdRegion),

    FechaEvento                  datetime         NOT NULL
        CONSTRAINT DF_BitacoraCorreoMailbot_FechaEvento DEFAULT (GETDATE()),
    FechaRegistro                datetime         NOT NULL
        CONSTRAINT DF_BitacoraCorreoMailbot_FechaRegistro DEFAULT (GETDATE()),
    Activo                       bit              NOT NULL
        CONSTRAINT DF_BitacoraCorreoMailbot_Activo DEFAULT (1)
);

CREATE NONCLUSTERED INDEX IX_BitacoraCorreoMailbot_Correo
    ON dbo.BitacoraCorreoMailbot (IdCorreoRecibido, FechaEvento DESC);
CREATE NONCLUSTERED INDEX IX_BitacoraCorreoMailbot_Cliente
    ON dbo.BitacoraCorreoMailbot (IdCliente, EstadoProcesamiento, FechaEvento DESC);
CREATE NONCLUSTERED INDEX IX_BitacoraCorreoMailbot_Buzon
    ON dbo.BitacoraCorreoMailbot (IdCatClasificacionCorreoRecibido, IdRegion, EstadoProcesamiento);
```

**Notas:**
- `IdPendienteGenerado` + `TablaPendiente` es referencia polimórfica (no FK física) porque el pendiente puede vivir en distintas tablas según el proceso (fccFolioPagoCliente para Cobros, tpPedido para Atender PC, tpCotizacion para Cotización, etc.).
- `EventoOrigen` distingue entre clasificación inicial del Mailbot, reclasificación manual del usuario, acción masiva, o eliminación del pendiente — necesario para auditar la trazabilidad solicitada por el usuario.

### 2.2 Vista nueva — `vBitacoraCorreoMailbot`

**Propósito:** Consulta legible para el área operativa: muestra correo, buzón destino, cliente, contacto y pendiente generado en una sola fila.

```sql
CREATE VIEW dbo.vBitacoraCorreoMailbot AS
SELECT
    bcm.IdBitacoraCorreoMailbot,
    bcm.FechaEvento,
    bcm.EstadoProcesamiento,
    bcm.EventoOrigen,
    bcm.MotivoNoProcesado,

    cr.Asunto,
    cr.CorreoEmisor,
    cr.FechaRecepcion,
    r.ClaveISO                              AS Region,

    cat.ClasificacionCorreoRecibido         AS Buzon,
    cat.Clave                               AS BuzonClave,
    proc.Proceso                            AS Proceso,

    c.IdCliente,
    c.Nombre                                AS Cliente,
    cc.IdContactoCliente,
    cc.NombreCompleto                       AS Contacto,

    bcm.IdPendienteGenerado,
    bcm.TablaPendiente,

    u.IdUsuario                             AS IdUsuarioOperacion,
    u.NombreCompleto                        AS UsuarioOperacion
FROM dbo.BitacoraCorreoMailbot bcm
INNER JOIN dbo.CorreoRecibido                  cr   ON bcm.IdCorreoRecibido           = cr.IdCorreoRecibido
INNER JOIN dbo.catClasificacionCorreoRecibido  cat  ON bcm.IdCatClasificacionCorreoRecibido = cat.IdCatClasificacionCorreoRecibido
INNER JOIN dbo.catProceso                      proc ON bcm.IdCatProceso              = proc.IdCatProceso
INNER JOIN dbo.Region                          r    ON bcm.IdRegion                  = r.IdRegion
LEFT  JOIN dbo.Cliente                         c    ON bcm.IdCliente                 = c.IdCliente
LEFT  JOIN dbo.ContactoCliente                 cc   ON bcm.IdContactoCliente         = cc.IdContactoCliente
LEFT  JOIN dbo.Usuario                         u    ON bcm.IdUsuarioOperacion        = u.IdUsuario
WHERE bcm.Activo = 1;
```

### 2.3 Diccionario de datos — Resumen

| Tabla / Vista | Tipo | Estado | Descripción |
|---|---|---|---|
| `catBuzon` | Catálogo | ✨ NUEVO R16 | Catálogo explícito de buzones con su `TablaPendiente`, banderas de acciones permitidas y política de cancelación (sección 0.3) |
| `BitacoraCorreoMailbot` | Tabla | ✨ NUEVO R16 | Bitácora unificada de eventos operativos de correos (F-2) |
| `vBitacoraCorreoMailbot` | Vista | ✨ NUEVO R16 | Vista consolidada para consulta operativa (F-3) |
| `CorreoRecibidoCliente` | Tabla | ✅ Existente — sin cambios | Sigue siendo la fuente de la clasificación vigente |
| `MailbotEventoCorreo` | Tabla | ✅ Existente (Propuesta 2) | Trazabilidad técnica del push Pub/Sub (no se mezcla con la bitácora operativa) |
| `MailbotClasificacionLog` | Tabla | ✅ Existente (Propuesta 2) | Auditoría IA (clasificación inicial, confianza, tokens) |
| `fccFolioPagoCliente`, `tpPedido`, `tpCotizacion`, … | Tablas | ✅ Existentes | Referenciadas por `BitacoraCorreoMailbot.IdPendienteGenerado` polimórficamente |

---

## 3. Impacto en Back

### 3.1 Endpoints nuevos (todos hospedados en `Mailbot.Api`)

> **Actualización 2026-06-29 (Q7):** E-0 (catálogo de buzones) **eliminado**. Se reutiliza el endpoint existente de `catClasificacionCorreoRecibido` para obtener la lista de buzones (Queja, Pago, Otros, Requisición, Pedido, Solicitud de información, Sugerencia).

| #   | Método | Ruta                                                              | Capacidad                                                                                                                                                                                                                                                                                                                                                            |
| --- | ------ | ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| E-1 | `POST` | `/api/mailbot/correo/reclasificar`                                | **F-1**: reclasifica un correo a otra clasificación, **inactiva atómicamente el correo + pendiente + tablas relacionadas** (cotización/partidas, ocOrdenDeCompra/partidas, fccFolioPagoCliente) **y crea el nuevo pendiente + datos del proceso destino** en la misma transacción. **Alimenta al agente IA** con los nuevos parámetros para clasificaciones futuras. |
| E-2 | `POST` | `/api/mailbot/correo/reclasificar-masivo`                         | **F-4**: reclasificación masiva (máximo 100 elementos por request, Q4) a una clasificación destino + cancelación masiva de pendientes                                                                                                                                                                                                                                |
| E-3 | `POST` | `/api/mailbot/correo/inactivar-masivo`                            | **F-4**: inactivación masiva (máximo 100 elementos por request, Q4) de correos en "Otros"                                                                                                                                                                                                                                                                            |
| E-4 | `POST` | `/api/bitacora-correo-mailbot/QueryResult`                        | **F-3**: consulta paginada de `vBitacoraCorreoMailbot` con `QueryInfo` (filtros, paginación, orden)                                                                                                                                                                                                                                                                  |
| E-5 | `PUT`  | `/api/buzon/{clave}/{idCorreoRecibidoCliente}/eliminar-pendiente` | **F-5**: cada buzón consumidor (cotizacion, atenderpc, cobros, otros) elimina el pendiente generado y deja una entrada en bitácora                                                                                                                                                                                                                                   |
| E-6 | `PUT`  | `/api/buzon/{clave}/{idCorreoRecibidoCliente}/reclasificar`       | **F-5**: cada buzón puede reclasificar a otro proceso (atajo del E-1 con la clave del buzón en la ruta)                                                                                                                                                                                                                                                              |

> **Todos los endpoints viven en `Mailbot.Api` / `MailbotWorker`.** Los controllers de los módulos consumidores (Cotización, Atender PC, Validar Pago, Validar Ajuste) NO duplican esta lógica — invocan al Mailbot vía HTTP cuando el usuario operativo dispara la acción desde la pantalla del módulo.
>
> **`{clave}` corresponde a `catClasificacionCorreoRecibido.Clave`** (no a una clave de catálogo nuevo): `cobros` (renombrar de `pago`), `cotizacion`, `pedido`, `requisicion`, `solicituddeinformacion`, `queja`, `sugerencia`, `otros`.

### 3.2 Detalle — `POST /api/mailbot/correo/reclasificar` (F-1)

**Request:**

```json
{
  "idCorreoRecibidoCliente": "guid",
  "idCatClasificacionDestino": "guid",
  "motivo": "string opcional",
  "cancelarPendienteOrigen": true
}
```

**Lógica del BO `BuzonReclasificacionBO`:**

1. Cargar `CorreoRecibidoCliente` actual con su clasificación origen.
2. Identificar el **pendiente generado** asociado (consulta a `BitacoraCorreoMailbot` por `IdCorreoRecibidoCliente` + `EventoOrigen = 'MailbotInicial'` o equivalente).
3. **Transacción:**
   - Si `cancelarPendienteOrigen = true` y existe pendiente:
     - Aplicar baja lógica al pendiente según `TablaPendiente`:
       - `fccFolioPagoCliente` → `Activo = 0`
       - `tpPedido` → según política de cancelación del proceso (puede ser estado `Cancelado` en vez de Activo=0)
       - `tpCotizacion` → según política
     - Insertar fila en `BitacoraCorreoMailbot` con `EventoOrigen = 'EliminacionPendiente'`, `EstadoProcesamiento = 'Inactivado'`.
   - `UPDATE CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido = idCatClasificacionDestino`.
   - **Si la clasificación destino genera pendiente** (regla por proceso), invocar al BO del proceso destino para crear el nuevo pendiente (p. ej. `FolioPagoClienteBO.Crear` para Cobros, `PedidoPretramitadoBO.Crear` para Atender PC).
   - Insertar fila en `BitacoraCorreoMailbot` con `EventoOrigen = 'ReclasificacionManual'`, `EstadoProcesamiento = 'Reclasificado'`, capturando el nuevo `IdPendienteGenerado` y `TablaPendiente`.
4. Commit transacción.

**Atomicidad:** todo en una sola transacción para que no quede inconsistente el correo reclasificado sin el pendiente cancelado.

### 3.3 Detalle — Endpoints masivos (F-4)

```json
POST /api/mailbot/correo/reclasificar-masivo
{
  "idsCorreoRecibidoCliente": ["guid", "guid", ...],
  "idCatClasificacionDestino": "guid",
  "cancelarPendienteOrigen": true
}

POST /api/mailbot/correo/inactivar-masivo
{
  "idsCorreoRecibidoCliente": ["guid", "guid", ...],
  "motivo": "string"
}
```

Implementación: iterar sobre la lista, aplicar lógica E-1 / inactivación por elemento, agrupar errores y retornar resumen `{ procesados, errores: [{id, mensaje}] }`. No fallar el batch completo si un elemento falla (reportar por elemento).

### 3.4 Detalle — `POST /api/bitacora-correo-mailbot/QueryResult` (F-3)

Sigue el patrón estándar `QueryInfo` / `QueryResult<T>` del proyecto.

**Filtros típicos:**

| Caso de uso | Filtros |
|---|---|
| Bitácora de un correo | `IdCorreoRecibido = guid` |
| Eventos de un cliente | `IdCliente = guid` + orden `FechaEvento DESC` |
| Correos en buzón "Otros" no procesados | `BuzonClave = 'otros'` + `EstadoProcesamiento = 'NoProcesado'` |
| Reclasificaciones por usuario | `EventoOrigen = 'ReclasificacionManual'` + `IdUsuarioOperacion = guid` |
| Bitácora por región | `Region = 'MEX'` o `'PER'` |

### 3.5 Detalle — Endpoints por buzón (F-5)

Cada buzón consumidor recibe los mismos dos endpoints en su namespace, delegando al BO transversal:

| Buzón | Ruta |
|---|---|
| Cotización | `PUT /api/buzon/cotizacion/{id}/eliminar-pendiente` y `.../reclasificar` |
| Atender PC | `PUT /api/buzon/atenderpc/{id}/eliminar-pendiente` y `.../reclasificar` |
| Validar Pago / Cobros | `PUT /api/buzon/cobros/{id}/eliminar-pendiente` y `.../reclasificar` |
| Validar Ajuste | `PUT /api/buzon/validarajuste/{id}/eliminar-pendiente` y `.../reclasificar` |
| Otros | `PUT /api/buzon/otros/{id}/eliminar-pendiente` y `.../reclasificar` |

Internamente todos delegan a `BuzonReclasificacionBO` y `BuzonPendienteBO`. La diferencia entre buzones es la **política de cancelación** del pendiente (cada proceso decide si su pendiente se desactiva con `Activo = 0`, se marca como `Cancelado`, o requiere validación adicional).

### 3.6 Capa lógica nueva (solución `MailbotWorker` / .NET 10)

| Archivo                                                                       | Tipo  | Descripción                                                                              |
| ----------------------------------------------------------------------------- | ----- | ---------------------------------------------------------------------------------------- |
| `Mailbot.Application\UseCases\ReclasificarCorreoUseCase.cs`                   | NUEVO | Orquesta reclasificación + cancelación de pendiente origen + creación de nuevo pendiente (E-1) |
| `Mailbot.Application\UseCases\ReclasificarMasivoUseCase.cs`                   | NUEVO | Operaciones masivas (E-2, E-3) con reporte por elemento                                  |
| `Mailbot.Application\UseCases\EliminarPendienteUseCase.cs`                    | NUEVO | Elimina pendiente del buzón consumidor (E-5)                                             |
| `Mailbot.Domain\Buzones\IPendienteCanceladorStrategy.cs`                      | NUEVO | Interfaz para implementar política de cancelación por proceso (Chain of Strategies)      |
| `Mailbot.Infrastructure\Buzones\Estrategias\PendienteCobroCancelador.cs`      | NUEVO | Implementa cancelación para `fccFolioPagoCliente`                                        |
| `Mailbot.Infrastructure\Buzones\Estrategias\PendientePedidoCancelador.cs`     | NUEVO | Implementa cancelación para `tpPedido` (pendiente Atender PC)                            |
| `Mailbot.Infrastructure\Buzones\Estrategias\PendienteCotizacionCancelador.cs` | NUEVO | Implementa cancelación para `tpCotizacion`                                               |
| `Mailbot.Infrastructure\Buzones\Estrategias\PendienteValidarAjusteCancelador.cs` | NUEVO | Implementa cancelación para ajuste                                                    |
| `Mailbot.Infrastructure\Buzones\PoliticaCancelacionResolver.cs`               | NUEVO | Resuelve la estrategia a aplicar leyendo `catBuzon.PoliticaCancelacion`                  |
| `Mailbot.Application\UseCases\ConsultarBitacoraUseCase.cs`                    | NUEVO | `QueryResult` sobre `vBitacoraCorreoMailbot`                                             |
| `Mailbot.Application\UseCases\ConsultarCatalogoBuzonesUseCase.cs`             | NUEVO | `QueryResult` sobre `catBuzon` (E-0)                                                     |
| `Mailbot.Infrastructure\Persistence\Repositories\CatBuzonRepository.cs`       | NUEVO | Acceso a `catBuzon` desde el Scaffold EF Core                                            |
| `Mailbot.Infrastructure\Persistence\Repositories\BitacoraRepository.cs`       | NUEVO | Persistencia / consulta de `BitacoraCorreoMailbot`                                       |

### 3.7 Capa API (`Mailbot.Api` / Minimal API .NET 10)

| Archivo                                             | Tipo                                                  |
| --------------------------------------------------- | ----------------------------------------------------- |
| `Mailbot.Api\Endpoints\CatBuzonEndpoints.cs`        | NUEVO — endpoint E-0 `POST /api/catbuzon/QueryResult` |
| `Mailbot.Api\Endpoints\ReclasificacionEndpoints.cs` | NUEVO — endpoints E-1, E-2, E-3, E-6                  |
| `Mailbot.Api\Endpoints\PendienteBuzonEndpoints.cs`  | NUEVO — endpoint E-5                                  |
| `Mailbot.Api\Endpoints\BitacoraCorreoEndpoints.cs`  | NUEVO — endpoint E-4                                  |

> En **ProquifaDotNet** (.NET Framework 4.8) NO se crean BOs ni controllers para esta funcionalidad. Las pantallas Angular de cada buzón consumidor invocan directamente los endpoints del Mailbot vía HTTP (mismo patrón que el resto de la integración Finanzas/Timbrado).

### 3.8 Cambios en el Mailbot.Worker (.NET 10)

| Caso de uso | Cambio |
|---|---|
| `ProcesarCorreoUseCase` | Después de crear el pendiente (cuando aplique), **insertar fila en `BitacoraCorreoMailbot`** con `EventoOrigen = 'MailbotInicial'`, `EstadoProcesamiento = 'Procesado'` o `'NoProcesado'` según resultado IA. |
| `ProcesarCorreoUseCase` (caso no procesable) | Si la IA no puede clasificar, insertar bitácora con `EstadoProcesamiento = 'NoProcesado'` y `MotivoNoProcesado` con el detalle (clasificación dudosa, datos faltantes, etc.). |

---

## 4. Tareas nuevas propuestas

Manteniendo la convención de numeración del 008 (Tareas 1–12 actuales en el archivo cancelado, propuesta-2 con más tareas), propongo agregar **Tareas T13–T19** (continuación) con la siguiente distribución:

> **Renumeración 2026-06-29:** Tras descartar `catBuzon` (Q7), las tareas se renumeran a **T13–T19** (antes T13–T21). Se eliminan T13 (catBuzon) y T19 (endpoint catBuzon). Las estimaciones de tiempo siguen el catálogo `Catalogo BackEnd.md` (códigos entre corchetes).

| #       | Clave                         | Tarea                                                                                                                                                                                                                                                                                                                                                                      | GAP       | Capacidad                |           Horas |
| ------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------ | --------------: |
| **T13** | `CREATE-TABL-G` + `BD-OBJ-CH` | `CREATE TABLE BitacoraCorreoMailbot` (17 columnas, índices, constraints) + `CREATE VIEW vBitacoraCorreoMailbot`                                                                                                                                                                                                                                                            | F-2 + F-3 | BD                       | **30** (24 + 6) |
| **T14** | `SERV-COMPLEX-TRANSACT`       | `ReclasificarCorreoUseCase` — orquesta: inactiva correo + tablas relacionadas del proceso origen (Q1), crea nuevo pendiente + datos del proceso destino (Q2), alimenta al agente IA con nuevos parámetros (Q2), registra evento en bitácora                                                                                                                                | F-1       | Mailbot (.NET 10)        |          **32** |
| **T15** | `ALG-COMPLX-LOGIC`            | Estrategias polimórficas de cancelación (`IPendienteCanceladorStrategy` + 3 implementaciones: `PendienteCobroCancelador` → `fccFolioPagoCliente`; `PendienteOrdenCompraCancelador` → `ocOrdenDeCompra` + partidas; `PendienteCotizacionCancelador` → `cotCotizacion` + partidas) — Q1/Q3 cierran que `ppPedido`/`tpPedido` están más adelante en el flujo de venta interna | F-1 + F-5 | Mailbot (.NET 10)        |          **32** |
| **T16** | `SERV-COMPLEX-TRANSACT`       | `ReclasificarMasivoUseCase` (reclasificar-masivo + inactivar-masivo) con **tope 100 elementos por request** (Q4) y reporte por elemento                                                                                                                                                                                                                                    | F-4       | Mailbot (.NET 10)        |          **32** |
| **T17** | `LIST-PAG-MULT-FILTER`        | `ConsultarBitacoraUseCase` + endpoint `POST /api/bitacora-correo-mailbot/QueryResult` con `QueryInfo` (filtros por correo / cliente / contacto / buzón / estado / región / usuario)                                                                                                                                                                                        | F-3       | Mailbot (.NET 10)        |          **40** |
| **T18** | `IMP-EXIST-SERVICE`           | `EliminarPendienteUseCase` + endpoints E-5 y E-6 por buzón (`/api/buzon/{clave}/{id}/eliminar-pendiente` y `.../reclasificar`). Atajos sobre T14 con la `clave` de `catClasificacionCorreoRecibido` en la ruta                                                                                                                                                             | F-5       | Mailbot (.NET 10)        |          **12** |
| **T19** | `IMP-EXIST-SERVICE`           | Modificar `Mailbot.Worker.ProcesarCorreoUseCase` para insertar fila en `BitacoraCorreoMailbot` tras procesar (`Procesado` / `NoProcesado` + `MotivoNoProcesado`)                                                                                                                                                                                                           | F-2       | Mailbot (.NET 10 Worker) |          **12** |
|         |                               | **Total estimado**                                                                                                                                                                                                                                                                                                                                                         |           |                          |       **190 h** |

**Distribución por categoría:**

| Categoría | Tareas | Horas |
|---|---|---:|
| Base de datos | T13 | 30 |
| Backend Mailbot — orquestación atómica | T14 | 32 |
| Backend Mailbot — estrategias de cancelación | T15 | 32 |
| Backend Mailbot — acciones masivas | T16 | 32 |
| Backend Mailbot — consulta de bitácora | T17 | 40 |
| Backend Mailbot — endpoints por buzón | T18 | 12 |
| Backend Mailbot — registro de procesamiento IA | T19 | 12 |
| **Total** | **7 tareas** | **190 h** |

**Predecesoras:**

- **T13** (BitacoraCorreoMailbot + vista) es prerequisito de T14, T16, T17, T18, T19.
- **T15** (estrategias) es prerequisito de T14, T16, T18 (las estrategias se inyectan en los use cases vía DI).
- **T14** (orquestador base) es prerequisito de T16 y T18.
- **T17** y **T19** pueden ejecutarse en paralelo con el resto una vez T13 esté listo.

**Ruta crítica (camino más largo):** T13 (30 h) → T15 (32 h) → T14 (32 h) → T16 (32 h) = **126 h** en serie.
**Trabajo paralelo:** T17 (40 h) + T18 (12 h) + T19 (12 h) = 64 h corren en paralelo con la ruta crítica.

---

## 5. Criterios de aceptación nuevos (sugeridos para incorporar al requisito)

| ID | Criterio |
|---|---|
| CA-F1 | Al reclasificar un correo desde Buzones, si el correo origen tenía un pendiente generado, el sistema **cancela atómicamente** el pendiente origen y registra el evento en `BitacoraCorreoMailbot` |
| CA-F2 | Cada procesamiento del Mailbot genera **exactamente una** fila en `BitacoraCorreoMailbot` con `EventoOrigen = 'MailbotInicial'`, vinculada al buzón destino, cliente y contacto (si están identificados) |
| CA-F3 | La vista `vBitacoraCorreoMailbot` retorna correo + buzón + cliente + contacto + pendiente + usuario operación en una sola fila, consultable por `QueryResult` |
| CA-F4 | El usuario puede seleccionar varios correos del buzón "Otros" y reclasificarlos a otro buzón **o** inactivarlos masivamente, recibiendo reporte por elemento (procesado / error con detalle) |
| CA-F5 | Cada buzón consumidor (Cotización, Atender PC, Cobros, Validar Ajuste, Otros) expone endpoints para reclasificar o eliminar el pendiente de cada ítem, dejando traza en bitácora |
| CA-F6 | Si el correo no es procesable por el Mailbot (IA no logra clasificar o faltan datos), se registra en `BitacoraCorreoMailbot` con `EstadoProcesamiento = 'NoProcesado'` y `MotivoNoProcesado` |
| CA-F7 | Existe `catBuzon` con un registro por cada buzón operativo. Frontend consume `POST /api/catbuzon/QueryResult` para renderizar dinámicamente acciones (`PermiteReclasificar`, `PermiteEliminarPendiente`, `PermiteAccionMasiva`, `PermiteVincularOC`, `PermiteCambiarCliente`) sin hardcodeo |
| CA-F8 | La lógica de buzones, reclasificación, bitácora y acciones masivas vive **exclusivamente en la solución `MailbotWorker`** (.NET 10). Los módulos consumidores (Cotización, Atender PC, Validar Pago/Cobros, Validar Ajuste) NO duplican esta lógica — invocan los endpoints del Mailbot |

---

## 6. Pendientes / Decisiones cerradas (2026-06-29)

| #   | Pendiente | Decisión |
| --- | --- | --- |
| Q1  | Política de cancelación por proceso | **CERRADO** — Inactivar (`Activo = 0`) las entidades relacionadas por proceso:<br>• Cotización: correo + `cotCotizacion` + partidas<br>• Pedidos (buzón): correo + `ocOrdenDeCompra` + partidas (`ppPedido` y `tpPedido` quedan más adelante en venta interna — ver Q3)<br>• Cobros: correo + pendiente de Validar Cobro (`fccFolioPagoCliente`) |
| Q2  | ¿Reclasificación crea automáticamente el nuevo pendiente? | **CERRADO — SÍ.** Al reclasificar: (1) se crea el nuevo pendiente del proceso destino con registros en tablas relacionadas, (2) se **alimenta al agente IA** con los nuevos parámetros para clasificaciones futuras, (3) al quitar la clasificación origen se **inactivan** los registros del proceso anterior. |
| Q3  | Dueño de la lógica de cancelación para pedidos / cotización | **CERRADO** — Para el ámbito del Buzón se cancela **`ocOrdenDeCompra`** (no `tpPedido` / `ppPedido`, que viven más adelante en venta interna) y **`cotCotizacion`** para cotizaciones. Estrategias: `PendienteOrdenCompraCancelador` y `PendienteCotizacionCancelador`. |
| Q4  | Tope máximo de elementos por request masivo | **CERRADO** — Se integra el tope sugerido: **100 elementos por request**. Validación en `Mailbot.Api` con HTTP 400 si se supera. |
| Q5  | Purga de `BitacoraCorreoMailbot` | **CERRADO** — **NO automática.** Purga manual por usuario. No hay job programado. |
| Q6  | Acción "Recuperar" del buzón "Otros" | **CERRADO — NO se implementa en R16.** No hay flujo para revertir inactivación. Se sugiere documentar **guía de resolución manual** para Soporte a la Producción (no entra como tarea de desarrollo). |
| Q7  | Crear `catBuzon` vs. extender `catClasificacionCorreoRecibido` | **CERRADO — Se descarta `catBuzon`.** Se reutiliza `catClasificacionCorreoRecibido` que ya contiene las 7 clasificaciones operativas: Queja, Pago, Otros, Requisición, Pedido, Solicitud de información, Sugerencia. La metadata de cancelación vive en código (estrategias `IPendienteCanceladorStrategy`), no en BD. |
| Q8  | `TablaPendiente` de Cotización y Atender PC | **CERRADO** — Cotización → `cotCotizacion`; Atender PC → `ocOrdenDeCompra` (no `tpPedido`). Ver Q3. |

---

## 7. Resumen accionable (actualizado 2026-06-29)

**2 cambios de BD (todos NUEVO R16):**
1. Crear `BitacoraCorreoMailbot` con índices (17 columnas).
2. Crear `vBitacoraCorreoMailbot`.

*(Sin cambios estructurales en tablas existentes. **`catBuzon` descartado (Q7)** — se reutiliza `catClasificacionCorreoRecibido` con las 7 clasificaciones existentes: Queja, Pago, Otros, Requisición, Pedido, Solicitud de información, Sugerencia.)*

**6 endpoints nuevos (todos en `Mailbot.Api` / .NET 10):**
- E-1 reclasificación atómica (inactiva proceso origen + crea proceso destino + alimenta IA)
- E-2/E-3 acciones masivas con **tope de 100 elementos por request** (Q4)
- E-4 `POST /api/bitacora-correo-mailbot/QueryResult`
- E-5/E-6 acciones por buzón usando `catClasificacionCorreoRecibido.Clave`

**11 archivos nuevos en `MailbotWorker`** (capa Application + Domain + Infrastructure) + **4 endpoints Minimal API** + **modificación de `ProcesarCorreoUseCase`**.

**0 archivos nuevos en ProquifaDotNet** (.NET Framework 4.8) — toda la lógica vive en Mailbot; los frontends de los módulos consumidores invocan los endpoints del Mailbot vía HTTP.

**7 tareas nuevas (T13–T19)** con estimación total de **190 horas** según `Catalogo BackEnd.md`:

| Tarea | Clave catálogo | Horas |
|---|---|---:|
| T13 | `CREATE-TABL-G` (24) + `BD-OBJ-CH` (6) | **30** |
| T14 | `SERV-COMPLEX-TRANSACT` | **32** |
| T15 | `ALG-COMPLX-LOGIC` | **32** |
| T16 | `SERV-COMPLEX-TRANSACT` | **32** |
| T17 | `LIST-PAG-MULT-FILTER` | **40** |
| T18 | `IMP-EXIST-SERVICE` | **12** |
| T19 | `IMP-EXIST-SERVICE` | **12** |
| **Total** | | **190 h** |

**Ruta crítica:** T13 → T15 → T14 → T16 = **126 h en serie.** T17, T18, T19 (64 h) corren en paralelo con la ruta crítica.

**8 criterios de aceptación nuevos** (CA-F1 a CA-F8) para añadir al requisito.

**0 pendientes abiertos** — Q1–Q8 cerrados en sesión del 2026-06-29 (sección 6). Listo para construcción.

### Entregable adicional sugerido (no de desarrollo)

Documentar una **guía de resolución manual para Soporte a la Producción** sobre cómo revertir una inactivación de pendiente cuando sea necesario (cubre Q6 sin necesidad de implementar flujo "Recuperar" en R16).

---

## 8. Referencias

- Canvas: `Diagramas/Canvas/10 - Flujo Mailbot Buzones y Modulos Consumidores.canvas`
- BD vigente: `Requisitos/R16A-RE-FU-008/R16A-RE-FU-008_BD.md` v1.0
- Back (Propuesta 2): `Requisitos/R16A-RE-FU-008/R16A-RE-FU-008-P2-Back.md` v1.0
- Solución .NET 10: `Requisitos/R16A-RE-FU-008/R16A-RE-FU-008-v2_Propuesta2.md` v1.0
- Tareas vigentes: `Requisitos/R16A-RE-FU-008-Cancelado/R16A-RE-FU-008-Tareas.md` (12 tareas; este faltante agrega T13–T21)
- Esquema BD revisado: `Database/ProquifaDotNet.sql` — verificación de ausencia de `catBuzon` y existencia de `catProceso` (línea 29630), `catClasificacionCorreoRecibido` (línea 29030), `RegionConfiguracionMailBot` (línea 24425)

---

## 10. Actualización 2026-07-02 — Integración de pantallas Ryndem Studio + flujo Miro

> **Insumos nuevos:**
> - **Pantallas de UX Ryndem Studio** (14 mockups): pantalla principal `CORREOS RECIBIDOS`, tabs `Pendientes Recibidos` / `Otros`, dropdown de clasificación con checkboxes, modal Vincular OC, modales de confirmación (clasificar, eliminar, spam), pantalla `COTIZAR LO COTIZABLE`, pantalla `ATENDER PROMESA DE COMPRA`.
> - **Flujo Miro** (pizarrón compartido por producto): decisión canónica de reclasificación via «Otros», bitácora sin filtro por cartera, regla de "1ª clasificación", `pedido regenerado (intramitable)`.
>
> Estos insumos son **posteriores y prevalecen** sobre las secciones 0–9. Donde haya conflicto, la sección 10 gana. Las secciones anteriores quedan como historial de decisiones.

### 10.1 Delta conceptual detectado

| # | Concepto | Estado en Faltante v1 (secc. 0–9) | Nuevo — pantallas + Miro (secc. 10) | Impacto |
|---|---|---|---|---|
| Δ-1 | Multi-clasificación | E-1 reclasifica 1 origen → 1 destino | Un correo puede tener **N clasificaciones simultáneas** (checkboxes multi-selección: ej. `Cotización, Pedido`). Genera N pendientes activos al mismo tiempo, uno por módulo. | 🔴 Cambio de modelo de datos: 1:1 → N:N |
| Δ-2 | Camino canónico de reclasificación | Origen → Destino directo | *"Reclasificar mal clasificado → «Otros» (camino canónico; NO se mueve directo a otro módulo)"*. Desde Otros se re-clasifica al módulo correcto. | 🔴 Cambio en E-1 y T14 |
| Δ-3 | Clasificaciones visibles en UI | 7 (Queja, Pago, Otros, Requisición, Pedido, Solicitud de información, Sugerencia) | **4 principales**: Cotización, Pedido, Cobro, Otros (+ tab dedicado Otros). En Atender Promesa aparece adicional **OC Ajustada**. Queja/Sugerencia/Solicitud no aparecen — caen en Otros (Q10 pendiente). | 🟡 Reducción de clasificaciones operativas |
| Δ-4 | Nombre de módulos | Cotización, Atender PC, Validar Pago, Validar Ajuste, Otros | **Cotizar lo cotizable** (antes Cotización), **Atender Promesa de Compra** (antes Atender PC / Atender Promesa), Validar Cobro, Validar Ajuste | 🟢 Renombre — actualizar labels |
| Δ-5 | OC Vinculada | No existía | Estado adicional tras vincular archivos recibidos con OCs pendientes de ajuste. Se muestra badge "OC Vinculada" junto a "OC Ajustada" y al confirmar pasa a Validar Ajuste. | 🔴 Nuevo campo + flujo |
| Δ-6 | Modal Vincular OC | No existía | Modal 2 columnas: **ARCHIVOS RECIBIDOS** ↔ **OC PENDIENTES DE AJUSTE**. Selección 1:1 por archivo. Endpoint dedicado. | 🔴 Nuevo endpoint + UI |
| Δ-7 | Botón Confirmar Clasificación | Siempre disponible | *"El botón se habilita únicamente cuando hay un [cambio en la selección]"*. Estado deshabilitado por defecto. | 🟢 Regla de frontend |
| Δ-8 | Botón Eliminar Pendiente | No existía como botón contextual | Aparece **solo cuando la selección resulta en `Otros`** (checkbox único). Color rojo. Reemplaza al botón Confirmar Clasificación. | 🟡 Regla contextual UI |
| Δ-9 | Modal de eliminación + Spam | Solo confirmación simple | Checkbox opcional: *"¿Deseas también marcar este contacto como Spam? Si marcas este contacto como spam, no volverás a recibir sus correos. Ten en cuenta que esta acción es irreversible."* | 🔴 Nuevo endpoint + tabla `ContactoSpam` |
| Δ-10 | Tab Otros con acciones masivas | Solo endpoints masivos genéricos | Botones específicos: **Vaciar bandeja** (elimina todos), **Eliminar seleccionados** (checkbox por fila), filtro Cliente | 🟢 Confirma alcance de F-4 |
| Δ-11 | Regla "1ª clasificación" en Bitácora | No documentada | *"Las reclasificaciones posteriores NO cuentan como nuevas — salvo cuando un correo pasa de «Otros» a clasificación válida (ahí sí cuenta como 1ª)"* | 🟡 Nueva columna `EsPrimeraClasificacion` en `BitacoraCorreoMailbot` |
| Δ-12 | Bitácora sin filtro por cartera | Vista con filtros amplios | *"General: todos ven todo, sin filtro por cartera (PENDIENTE confirmar con cliente)"* | 🟢 Ya cubierto — Q9 confirmar |
| Δ-13 | Pedido regenerado (intramitable) | Concepto no definido | Al reclasificar Validar Ajuste → Pedido: inactiva Validar Ajuste + **regenera pedido "intramitable"** + genera nuevo pendiente en APC | 🔴 Nueva estrategia de cancelación |
| Δ-14 | Autocompletado IA por módulo | Genérico | Cada módulo tiene su propio detalle: `Cotizar lo cotizable` (pre-arma cotización), `Validar Cobro` (llena datos pago), `Atender Promesa` (llena referencia/totales), `Validar Ajuste` (preselecciona adjunto) | 🟢 Documentar expectativa |
| Δ-15 | Menú lateral de Pendientes | No documentado | Pantalla muestra sidebar completo de pendientes de venta interna: Correos Recibidos (30,163), Cotizar lo cotizable (467), Atender investigación técnico comercial (582), Determinar familia (487), Configurar familia — Costo/Tiempo/Rentabilidad (19/0/14), Atender cierre (2), Junta diaria (1), Resumen general (1), Cerrar oferta (22), Ajustar oferta (0), Atender seguimiento a promesa de compra (121) | 🟡 Buzones adicionales no cubiertos por 008 — requieren mapeo con otros requisitos |

### 10.2 Ajustes al modelo de datos

#### 10.2.1 Multi-clasificación (Δ-1) — Tabla intermedia N:N

Hoy `CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido` es 1:1 (una única clasificación por correo). Los mockups exigen que un mismo correo tenga varias clasificaciones activas simultáneamente.

**Opción recomendada:** tabla intermedia dedicada.

```sql
CREATE TABLE dbo.CorreoRecibidoClienteClasificacion (
    IdCorreoRecibidoClienteClasificacion  uniqueidentifier NOT NULL
        CONSTRAINT PK_CorreoRecibidoClienteClasificacion PRIMARY KEY CLUSTERED
        CONSTRAINT DF_CRCC_Id DEFAULT (NEWID()),

    IdCorreoRecibidoCliente          uniqueidentifier NOT NULL
        CONSTRAINT FK_CRCC_CorreoRecibidoCliente
            FOREIGN KEY REFERENCES dbo.CorreoRecibidoCliente(IdCorreoRecibidoCliente),
    IdCatClasificacionCorreoRecibido uniqueidentifier NOT NULL
        CONSTRAINT FK_CRCC_Clasificacion
            FOREIGN KEY REFERENCES dbo.catClasificacionCorreoRecibido(IdCatClasificacionCorreoRecibido),

    -- Trazabilidad
    EsPrimeraClasificacion           bit              NOT NULL   -- Δ-11: TRUE solo la 1ª vez o cuando viene de "Otros"
        CONSTRAINT DF_CRCC_EsPrimera DEFAULT (0),
    OrigenClasificacion              varchar(40)      NOT NULL   -- 'Mailbot' | 'UsuarioManual' | 'ReclasificacionDesdeOtros'
        CONSTRAINT DF_CRCC_Origen DEFAULT ('Mailbot'),

    -- Pendiente que generó esta clasificación (polimórfico)
    IdPendienteGenerado              uniqueidentifier NULL,
    TablaPendiente                   varchar(80)      NULL,     -- 'cotCotizacion' | 'ocOrdenDeCompra' | 'fccFolioPagoCliente' | 'tpPedidoIntramitable' | NULL para 'Otros'

    Activo                           bit              NOT NULL
        CONSTRAINT DF_CRCC_Activo DEFAULT (1),
    FechaRegistro                    datetime         NOT NULL
        CONSTRAINT DF_CRCC_FechaRegistro DEFAULT (GETDATE()),
    IdUsuarioClasificacion           uniqueidentifier NULL       -- NULL si la clasificó el Mailbot
        CONSTRAINT FK_CRCC_Usuario
            FOREIGN KEY REFERENCES dbo.Usuario(IdUsuario)
);

CREATE UNIQUE NONCLUSTERED INDEX UIX_CRCC_CorreoClasificacionActiva
    ON dbo.CorreoRecibidoClienteClasificacion (IdCorreoRecibidoCliente, IdCatClasificacionCorreoRecibido)
    WHERE Activo = 1;
CREATE NONCLUSTERED INDEX IX_CRCC_Pendiente
    ON dbo.CorreoRecibidoClienteClasificacion (IdPendienteGenerado, TablaPendiente);
```

**Regla de migración:** para correos existentes (1:1), poblar un solo registro por correo con `EsPrimeraClasificacion = 1`. El campo `CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido` se marca como **DEPRECATED** (mantener por compatibilidad, no leer para nueva lógica).

#### 10.2.2 OC Vinculada (Δ-5, Δ-6) — Columna dedicada + tabla de vínculo

`OC Ajustada` es una clasificación (fila en `catClasificacionCorreoRecibido` que hay que insertar). **`OC Vinculada`** NO es clasificación: es un **estado adicional** que solo tiene sentido para clasificación `OC Ajustada`.

```sql
-- INSERT nueva clasificación para Atender Promesa
INSERT INTO dbo.catClasificacionCorreoRecibido (IdCatClasificacionCorreoRecibido, Clave, ClasificacionCorreoRecibido, Posicion, Activo)
VALUES (NEWID(), 'ocajustada', 'OC Ajustada', 55, 1);

-- Tabla de vínculo archivo-recibido ↔ OC pendiente de ajuste
CREATE TABLE dbo.VinculoArchivoOCAjustada (
    IdVinculoArchivoOCAjustada       uniqueidentifier NOT NULL
        CONSTRAINT PK_VinculoArchivoOCAjustada PRIMARY KEY CLUSTERED
        CONSTRAINT DF_VAOCA_Id DEFAULT (NEWID()),

    IdCorreoRecibidoClienteClasificacion  uniqueidentifier NOT NULL
        CONSTRAINT FK_VAOCA_Clasificacion
            FOREIGN KEY REFERENCES dbo.CorreoRecibidoClienteClasificacion(IdCorreoRecibidoClienteClasificacion),

    IdArchivoRecibido                uniqueidentifier NOT NULL,   -- FK a la tabla de adjuntos del correo (existente)
    IdOrdenCompra                    uniqueidentifier NOT NULL
        CONSTRAINT FK_VAOCA_OrdenCompra
            FOREIGN KEY REFERENCES dbo.ocOrdenDeCompra(IdOrdenCompra),

    FechaVinculo                     datetime         NOT NULL
        CONSTRAINT DF_VAOCA_Fecha DEFAULT (GETDATE()),
    IdUsuarioVincula                 uniqueidentifier NOT NULL
        CONSTRAINT FK_VAOCA_Usuario
            FOREIGN KEY REFERENCES dbo.Usuario(IdUsuario),
    Activo                           bit              NOT NULL
        CONSTRAINT DF_VAOCA_Activo DEFAULT (1)
);

CREATE UNIQUE NONCLUSTERED INDEX UIX_VAOCA_ClasifOC
    ON dbo.VinculoArchivoOCAjustada (IdCorreoRecibidoClienteClasificacion, IdOrdenCompra)
    WHERE Activo = 1;
```

El estado "OC Vinculada" del UI se deriva por consulta: `EXISTS(SELECT 1 FROM VinculoArchivoOCAjustada WHERE IdCorreoRecibidoClienteClasificacion = @id AND Activo = 1)`. No requiere columna booleana adicional.

#### 10.2.3 Contactos marcados como Spam (Δ-9)

```sql
CREATE TABLE dbo.ContactoSpam (
    IdContactoSpam           uniqueidentifier NOT NULL
        CONSTRAINT PK_ContactoSpam PRIMARY KEY CLUSTERED
        CONSTRAINT DF_ContactoSpam_Id DEFAULT (NEWID()),

    CorreoEmisor             varchar(320)     NOT NULL,  -- email en minúsculas normalizado
    IdContactoCliente        uniqueidentifier NULL       -- si el contacto existe en Cliente/ContactoCliente
        CONSTRAINT FK_ContactoSpam_Contacto
            FOREIGN KEY REFERENCES dbo.ContactoCliente(IdContactoCliente),
    Motivo                   varchar(400)     NULL,
    FechaMarcado             datetime         NOT NULL
        CONSTRAINT DF_ContactoSpam_Fecha DEFAULT (GETDATE()),
    IdUsuarioMarca           uniqueidentifier NOT NULL
        CONSTRAINT FK_ContactoSpam_Usuario
            FOREIGN KEY REFERENCES dbo.Usuario(IdUsuario),
    IdRegion                 uniqueidentifier NOT NULL
        CONSTRAINT FK_ContactoSpam_Region
            FOREIGN KEY REFERENCES dbo.Region(IdRegion),
    Activo                   bit              NOT NULL
        CONSTRAINT DF_ContactoSpam_Activo DEFAULT (1)
);

CREATE UNIQUE NONCLUSTERED INDEX UIX_ContactoSpam_Email
    ON dbo.ContactoSpam (CorreoEmisor, IdRegion) WHERE Activo = 1;
```

**Regla de aplicación:** el Mailbot consulta `ContactoSpam` antes de procesar un correo entrante. Si `EXISTS` con `Activo = 1`, se descarta (registro en `BitacoraCorreoMailbot` con `EstadoProcesamiento = 'DescartadoSpam'`) y NO llega a Buzones.

#### 10.2.4 Ajuste a `BitacoraCorreoMailbot`

Se agregan/ajustan columnas para reflejar los cambios:

```sql
-- Δ-11: identificar la 1ª clasificación (para métricas de correos "nuevos")
ALTER TABLE dbo.BitacoraCorreoMailbot
    ADD EsPrimeraClasificacion   bit  NOT NULL
        CONSTRAINT DF_BitacoraCorreoMailbot_EsPrimera DEFAULT (0);

-- Δ-2: distinguir eventos de "camino canónico via Otros"
ALTER TABLE dbo.BitacoraCorreoMailbot
    ADD PasoIntermedio           varchar(40) NULL;   -- 'ViaOtros' | 'DirectoModuloDestino' | NULL

-- Δ-9: registrar cuando la eliminación incluyó marcar spam
ALTER TABLE dbo.BitacoraCorreoMailbot
    ADD MarcadoSpam              bit  NOT NULL
        CONSTRAINT DF_BitacoraCorreoMailbot_Spam DEFAULT (0);
```

Valores nuevos para `EventoOrigen`: `'ClasificacionAdicional'` (multi-clasificación), `'VinculoOC'`, `'DescartadoSpam'`, `'ReclasificacionCanonicaViaOtros'`.

#### 10.2.5 Diccionario de datos — Delta 2026-07-02

| Tabla / Vista                                            | Tipo     | Estado                 | Descripción                                                                                                |
| -------------------------------------------------------- | -------- | ---------------------- | ---------------------------------------------------------------------------------------------------------- |
| `CorreoRecibidoClienteClasificacion`                     | Tabla    | ✨ NUEVO R16 (Δ-1)      | Relación N:N correo ↔ clasificación con pendiente polimórfico                                              |
| `VinculoArchivoOCAjustada`                               | Tabla    | ✨ NUEVO R16 (Δ-5, Δ-6) | Vínculo archivo recibido ↔ OC pendiente de ajuste                                                          |
| `ContactoSpam`                                           | Tabla    | ✨ NUEVO R16 (Δ-9)      | Lista negra por correo emisor + región                                                                     |
| `BitacoraCorreoMailbot`                                  | Tabla    | 🔄 Alterado            | +3 columnas (`EsPrimeraClasificacion`, `PasoIntermedio`, `MarcadoSpam`) + nuevos valores en `EventoOrigen` |
| `catClasificacionCorreoRecibido`                         | Catálogo | 🔄 INSERT              | Fila nueva: `OC Ajustada` (clave `ocajustada`)                                                             |
| `CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido` | Columna  | ⚠️ DEPRECATED          | Se mantiene por compatibilidad; nueva lógica lee `CorreoRecibidoClienteClasificacion`                      |

### 10.3 Ajustes a endpoints

| Ref             | Endpoint                                                      | Cambio                                                                                                                                                                                                                                                                     |
| --------------- | ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| E-1 (existente) | `POST /api/mailbot/correo/reclasificar`                       | 🔴 Cambia de `idCatClasificacionDestino` (guid único) a `idsCatClasificacionDestino: [guid]` (array). Δ-1.                                                                                                                                                                 |
| E-1 (existente) | mismo                                                         | 🔴 Regla nueva: si el destino es distinto al origen y no incluye `otros`, la reclasificación **debe pasar primero por `otros`** (transacción interna 2 pasos). Δ-2.                                                                                                        |
| E-2/E-3         | Endpoints masivos                                             | 🟢 Sin cambio estructural; validar tope 100 sigue igual                                                                                                                                                                                                                    |
| **E-7** ✨       | `POST /api/mailbot/correo/{id}/agregar-clasificacion`         | Agrega una clasificación adicional conservando las existentes. Δ-1.                                                                                                                                                                                                        |
| **E-8** ✨       | `POST /api/mailbot/oc-ajustada/vincular`                      | Vincula archivos recibidos con OCs pendientes de ajuste. Body: `{idCorreoRecibidoClienteClasificacion, vinculos: [{idArchivoRecibido, idOrdenCompra}]}`. Al confirmar, la clasificación pasa a estado "OC Vinculada" y se activa el pendiente en Validar Ajuste. Δ-5, Δ-6. |
| **E-9** ✨       | `POST /api/mailbot/otros/vaciar-bandeja`                      | Vacía la bandeja Otros de la región del usuario. Requiere confirmación. Δ-10.                                                                                                                                                                                              |
| **E-10** ✨      | `POST /api/mailbot/contacto-spam/marcar`                      | Marca `CorreoEmisor + IdRegion` como spam. Δ-9.                                                                                                                                                                                                                            |
| **E-11** ✨      | `PUT /api/mailbot/correo/{id}/eliminar-con-spam`              | Elimina el pendiente y opcionalmente marca al contacto como spam. Body: `{marcarSpam: true/false}`. Δ-9.                                                                                                                                                                   |
| **E-12** ✨      | `POST /api/mailbot/validar-ajuste/{id}/reclasificar-a-pedido` | Reclasifica Validar Ajuste → Pedido: inactiva Validar Ajuste + regenera pedido intramitable + genera pendiente en APC. Δ-13.                                                                                                                                               |

### 10.4 Ajustes a tareas (T20–T24)

Se agregan **5 tareas nuevas** al tramo T13–T19 vigente. La ruta crítica cambia porque T13' (BD ampliada) es prerequisito duro de T14/T15/T16.

| #        | Clave                   | Tarea                                                                                                                                                                                 | GAP / Δ             |                             Horas |
| -------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- | --------------------------------: |
| **T13'** | `BD-OBJ-CH`             | **Modificación** de T13: agregar `CorreoRecibidoClienteClasificacion`, `VinculoArchivoOCAjustada`, `ContactoSpam` + INSERT `OC Ajustada` + ALTER `BitacoraCorreoMailbot` (3 columnas) | Δ-1, Δ-5, Δ-9, Δ-11 | **+12** (T13 pasa de 30 a **42**) |
| **T14'** | `SERV-COMPLEX-TRANSACT` | **Modificación** de T14: soportar array de clasificaciones destino + camino canónico via Otros                                                                                        | Δ-1, Δ-2            |  **+8** (T14 pasa de 32 a **40**) |
| **T20**  | `IMP-EXIST-SERVICE`     | `AgregarClasificacionUseCase` + endpoint E-7 (multi-clasificación aditiva conservando existentes)                                                                                     | Δ-1                 |                            **12** |
| **T21**  | `SERV-COMPLEX-TRANSACT` | `VincularOCAjustadaUseCase` + endpoint E-8 (modal 2 columnas, activación de Validar Ajuste) + `PendienteValidarAjusteRegeneradorStrategy` para E-12 (Δ-13)                            | Δ-5, Δ-6, Δ-13      |                            **32** |
| **T22**  | `IMP-EXIST-SERVICE`     | `MarcarSpamUseCase` + endpoints E-10, E-11 + consulta previa en `ProcesarCorreoUseCase` (Δ-9)                                                                                         | Δ-9                 |                            **12** |
| **T23**  | `IMP-EXIST-SERVICE`     | `VaciarBandejaOtrosUseCase` + endpoint E-9 (variante especializada de E-3)                                                                                                            | Δ-10                |                            **12** |
| **T24**  | `IMP-EXIST-SERVICE`     | Marcar `EsPrimeraClasificacion` en `BitacoraCorreoMailbot` + `CorreoRecibidoClienteClasificacion` según regla Δ-11 (solo la 1ª o desde Otros a válida)                                | Δ-11                |                            **12** |

**Total actualizado del Faltante:** ~~190 h~~ → **230 h**

| Categoría | Tareas | Horas |
|---|---|---:|
| BD | T13' | 42 |
| Backend Mailbot — orquestación atómica | T14' | 40 |
| Backend Mailbot — estrategias de cancelación | T15 | 32 |
| Backend Mailbot — acciones masivas | T16 | 32 |
| Backend Mailbot — consulta de bitácora | T17 | 40 |
| Backend Mailbot — endpoints por buzón | T18 | 12 |
| Backend Mailbot — registro de procesamiento IA | T19 | 12 |
| Backend Mailbot — agregar clasificación (Δ-1) | T20 | 12 |
| Backend Mailbot — vincular OC + regenerar (Δ-5, Δ-6, Δ-13) | T21 | 32 |
| Backend Mailbot — spam (Δ-9) | T22 | 12 |
| Backend Mailbot — vaciar bandeja (Δ-10) | T23 | 12 |
| Backend Mailbot — 1ª clasificación (Δ-11) | T24 | 12 |
| **Total** | **12 tareas** | **300 h** |

> **Corrección:** el total real considerando T13' (42) + T14' (40) + resto original (T15 32 + T16 32 + T17 40 + T18 12 + T19 12 = 128) + nuevas (T20 12 + T21 32 + T22 12 + T23 12 + T24 12 = 80) = **290 h**. Antes eran 190 h, incremento neto **+100 h**.

**Nueva ruta crítica:** T13' (42 h) → T15 (32 h) → T14' (40 h) → T21 (32 h) = **146 h en serie**. T16, T17, T18, T19, T20, T22, T23, T24 (144 h agregadas) corren en paralelo.

### 10.5 Pendientes nuevos (Q9–Q13)

| # | Pendiente | Detalle | Prioridad |
|---|---|---|---|
| Q9 | Bitácora sin filtro por cartera | *"PENDIENTE confirmar con cliente"* — ¿la vista de Bitácora muestra todos los correos sin importar la cartera del usuario, o filtra por cartera asignada? | 🔴 Bloqueante para T17 |
| Q10 | Quejas/Sugerencias/Solicitud de información | ¿Se mantienen como clasificaciones separadas en `catClasificacionCorreoRecibido` (7 clasificaciones actuales) o se subsumen en Otros? Las pantallas solo muestran 4 (Cot/Ped/Cob/Otros) + OC Ajustada. | 🟡 Impacta seed de catálogo |
| Q11 | Asociación cuenta ↔ clasificación | *"PENDIENTE: definir asociación cuenta ↔ clasificación (¿segmentar por cuenta o todas reciben todo?)"* — la nota del Miro. ¿ventas PROQUIFA solo recibe cotizaciones/pedidos, cobros@ solo recibe cobros, etc.? | 🟡 Impacta reglas del Mailbot |
| Q12 | Pedido intramitable (Δ-13) | ¿Qué tabla/estado representa un "pedido intramitable" tras la regeneración? ¿Nueva columna en `tpPedido` o tabla dedicada `tpPedidoIntramitable`? | 🔴 Bloqueante para T21 |
| Q13 | Permiso de eliminar según correo visto | *"DUDA: Permiso de eliminar según si el correo fue visto"* — ¿el usuario que aún no abrió el correo puede eliminarlo? ¿Se marca `Visto = 1` en `BitacoraCorreoMailbot`? | 🟢 Regla de UX, no bloqueante |

### 10.6 Criterios de aceptación nuevos (CA-F9–CA-F13)

| ID | Criterio |
|---|---|
| CA-F9 | Un correo puede tener **N clasificaciones activas simultáneamente**. Cada clasificación genera su propio pendiente activo. Al desmarcar una, se cancela solo ese pendiente. Al reclasificar solo a `Otros`, se cancelan todas las demás. |
| CA-F10 | Reclasificar desde un módulo a otro **pasa siempre por el estado intermedio `Otros`** en la misma transacción (2 eventos en Bitácora: `ReclasificacionCanonicaViaOtros` + `ReclasificacionManual`). |
| CA-F11 | En Atender Promesa, la acción "Vincular OC" abre un modal de 2 columnas (archivos ↔ OCs pendientes). Al vincular al menos 1 par y confirmar, la clasificación queda como `OC Ajustada + OC Vinculada` y se activa el pendiente en Validar Ajuste. |
| CA-F12 | Al eliminar un pendiente clasificado como `Otros`, el usuario puede opcionalmente marcar al contacto emisor como Spam. Si lo hace, el Mailbot descarta futuros correos de ese emisor en la región del usuario. |
| CA-F13 | `BitacoraCorreoMailbot.EsPrimeraClasificacion` es `1` únicamente cuando: (a) el Mailbot clasificó el correo por primera vez, o (b) el correo pasó de `Otros` a una clasificación válida. Métricas de "correos nuevos" filtran por esta bandera. |

### 10.7 Resumen accionable actualizado 2026-07-02

**BD (7 cambios):**
1. `CREATE TABLE CorreoRecibidoClienteClasificacion` (N:N) ✨
2. `CREATE TABLE VinculoArchivoOCAjustada` ✨
3. `CREATE TABLE ContactoSpam` ✨
4. `CREATE TABLE BitacoraCorreoMailbot` (17 columnas base — de sección 2)
5. `CREATE VIEW vBitacoraCorreoMailbot` (de sección 2)
6. `ALTER TABLE BitacoraCorreoMailbot` +3 columnas
7. `INSERT catClasificacionCorreoRecibido` fila `OC Ajustada`

**Endpoints (12 total):**
- E-1 modificado (multi-destino + camino canónico via Otros)
- E-2, E-3 (masivos — sin cambio)
- E-4 (Bitácora QueryResult — sin cambio)
- E-5, E-6 (por buzón — sin cambio)
- E-7 agregar clasificación ✨
- E-8 vincular OC ✨
- E-9 vaciar bandeja Otros ✨
- E-10 marcar spam ✨
- E-11 eliminar con opción spam ✨
- E-12 Validar Ajuste → Pedido (regenerar intramitable) ✨

**Tareas: 12 (era 7) — 290 h totales (era 190 h)**

**Pendientes abiertos: 5 (Q9–Q13)** — bloquean T17, T21, y afinan seed de catálogo.

**Nombres actualizados en UI:**
- Módulo Cotización → **Cotizar lo cotizable**
- Módulo Atender PC → **Atender Promesa de Compra**
- Buzón principal: pantalla `CORREOS RECIBIDOS` con tabs `Pendientes Recibidos` / `Otros`

---

## 11. Referencias (2026-07-02)

- Pantallas Ryndem Studio (14 mockups): `CORREOS RECIBIDOS`, `COTIZAR LO COTIZABLE`, `ATENDER PROMESA DE COMPRA`, modales de clasificación / eliminación / spam / vincular OC.
- Flujo Miro: pizarrón compartido por producto — camino canónico via Otros + bitácora sin cartera + regla 1ª clasificación + pedido intramitable.
- Documento base Faltante: secciones 0–9 (histórico de decisiones).
- Anotaciones del diseñador (Ryndem Studio hace ~6 h): configuración Con OC / Sin OC, reglas del botón Confirmar Clasificación, modales de eliminación con Spam.
