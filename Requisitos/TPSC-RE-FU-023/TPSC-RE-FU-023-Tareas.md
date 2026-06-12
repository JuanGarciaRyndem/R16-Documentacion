# Tareas — TPSC-RE-FU-023
**Requisito:** Validar Cobro — Pantalla Principal
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)

---

> **Orden de ejecución:** BD (Tareas 1–2) → Scaffold + CQRS Finanzas (Tareas 3–5) → Endpoints Finanzas (Tareas 6–9)
> **Prerrequisito externo:** `CorreoRecibidoCliente` ya existe (RE-FU-008).
> `fccFolioPagoCliente` y `fccPagoCliente` ya existen en BD (verificado RYNL010).
> La Tarea 1 (ALTER fccPagoCliente) debe ejecutarse antes del Scaffold (Tarea 3) para que
> las entidades generadas incluyan los campos del wizard.
>
> **Lógica existente a NO duplicar:**
> `CorreoRecibidoClienteToPagoBO.cs` (ProquifaDotNet, Mailbot) ya hace INSERT en
> `fccFolioPagoCliente`. El CQRS de Finanzas solo hace lectura y cierre del pendiente.

---

## Resumen de tareas

| # | Clave | Título simple | Tipo | Aplicativo | Estado |
|---|-------|--------------|------|-----------|--------|
| 1 | UPDATE-TABL-CH | Agregar campos del wizard a fccPagoCliente | BD | ProquifaDotNet | ❌ Pendiente |
| 2 | UPDATE-TABL-CH | Agregar trazabilidad de cancelación + CFDI a tpPedido (OBS-042) | BD | ProquifaDotNet | ❌ Pendiente |
| 3 | IMP-EXIST-SERVICE | Scaffold EF Core fccFolioPagoCliente, fccPagoCliente y tpProformaPedido en Finanzas | Back | ProquifaDotNet.Finanzas | ❌ Pendiente |
| 4 | SIMPLE-CRUD | CQRS fccFolioPagoCliente — Query pendientes + Command cerrar pendiente | Back | ProquifaDotNet.Finanzas | ❌ Pendiente |
| 5 | SIMPLE-CRUD | CQRS fccPagoCliente — Query cobro + Command upsert borrador | Back | ProquifaDotNet.Finanzas | ❌ Pendiente |
| 6 | LIST-PAG-MULT-FILTER | Listado de clientes de Validar Cobro con acción contextual | Back | ProquifaDotNet.Finanzas | ❌ Pendiente |
| 7 | LIST-NO-FILTER | Listado de pedidos pendientes para modal Gestionar Cobranza | Back | ProquifaDotNet.Finanzas | ❌ Pendiente |
| 8 | SERV-SIMPLE-PUT | Actualizar fecha estimada de pago con historial (OBS-044) | Back | ProquifaDotNet.Finanzas | ❌ Pendiente |
| 9 | SERV-TRANSACT | Cancelar pedido + CFDI por falta de pago (OBS-042) | Back | ProquifaDotNet + ProquifaDotNet.Finanzas | ❌ Pendiente |
| 10 | UPDATE-TABL-CH | Crear tabla fccFechaEstimadaPagoHistorial (OBS-044) | BD | ProquifaDotNet | ❌ Pendiente |

---

## TAREA 1

**[ RE-FU-023 ] [UPDATE-TABL-CH] Agregar campos del wizard Validar Cobro a fccPagoCliente**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro

**Consideraciones previas:**
- `fccPagoCliente` ya existe en BD (verificado RYNL010).
- Estos campos son requeridos a partir de RE-FU-024 pero se crean aquí como infraestructura base
  del módulo, para que el Scaffold de Finanzas (Tarea 3) los incluya desde el primer ciclo.
- `Confirmado` identifica si el cobro fue confirmado (`1`) o es borrador (`0`). El borrador se
  identifica por `Confirmado=0` (no por `Folio=NULL`).
- `IdCatMoneda` requiere FK a `catMoneda`.
- Los campos deben ser NULL (excepto `Confirmado` con DEFAULT 0) para no romper registros existentes.
- Prerrequisito de la Tarea 3 (Scaffold) — ejecutar antes de regenerar el scaffold.
- Los campos `FechaConfirmacion`, `IdUsuarioConfirmacion` los poblará RE-FU-024 al confirmar.

**Objetivo general:**
Agregar los 5 campos del wizard a `fccPagoCliente` para completar el esquema
que usará el Scaffold de Finanzas y los requisitos RE-FU-024 en adelante.

**Objetivos específicos:**
- `ALTER TABLE dbo.fccPagoCliente ADD Confirmado bit NOT NULL CONSTRAINT DF_fccPagoCliente_Confirmado DEFAULT (0)`
- `ALTER TABLE dbo.fccPagoCliente ADD FechaConfirmacion datetime2 NULL`
- `ALTER TABLE dbo.fccPagoCliente ADD IdUsuarioConfirmacion uniqueidentifier NULL`
- `ALTER TABLE dbo.fccPagoCliente ADD Notas varchar(500) NULL`
- `ALTER TABLE dbo.fccPagoCliente ADD IdCatMoneda uniqueidentifier NULL`
- `ALTER TABLE dbo.fccPagoCliente ADD CONSTRAINT FK_fccPagoCliente_catMoneda FOREIGN KEY (IdCatMoneda) REFERENCES dbo.catMoneda (IdCatMoneda)`
- Verificar registros existentes: todos con `Confirmado=0`, resto en NULL.

**Resultado esperado:**
`fccPagoCliente` con los 5 campos nuevos disponibles. El Scaffold de la Tarea 3
generará entidades con todos los campos del wizard incluidos.

**Entregables:**
- Script DDL: 5 ALTER TABLE + 1 ADD CONSTRAINT FK
- Script de validación: SELECT columnas + verificar FK catMoneda + verificar DEFAULT Confirmado

**Criterios de aceptación:**
- Los 5 campos existen en `fccPagoCliente`.
- `Confirmado` tiene DEFAULT 0 y NOT NULL.
- FK a `catMoneda` válida.
- Registros existentes tienen `Confirmado=0` y el resto NULL.

**Más información de la tarea:**
Ver sección *"A1"* en `TPSC-RE-FU-023-Back.md` y sección *"3"* en `TPSC-RE-FU-023_BD.md`.

**Recursos:**
- `TPSC-RE-FU-023-Back.md` — Parte A, A1
- `TPSC-RE-FU-023_BD.md` — Sección 3: ALTER fccPagoCliente
- `TPSC-RE-FU-024_BD.md` — Contexto de uso de estos campos en el Paso 1 del wizard

---

## TAREA 2

**[ RE-FU-023 ] [UPDATE-TABL-CH] Agregar campos de trazabilidad de cancelación a tpPedido**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro

**Consideraciones previas:**
- `tpProformaPedido.Cancelada` (bit) ya existe — no requiere ALTER.
- Los campos `FechaCancelacionPorFaltaPago` e `IdUsuarioCancelacion` registran quién y cuándo canceló el pedido desde el modal Gestionar Cobranza.
- **OBS-042:** Adicionalmente se agregan `FechaSolicitudCancelacion` (datetime2 NULL) y `EstadoCancelacionCFDI` (varchar(50) NULL) para trazabilidad de la cancelación del CFDI ante el SAT. Son campos distintos de la cancelación del pedido.
- Todos los campos deben ser NULL para no romper registros existentes.
- Prerrequisito de la Tarea 9 (endpoint de cancelación en ProquifaDotNet).
- Esta tarea es independiente de la Tarea 1 — puede ejecutarse en paralelo.

**Objetivo general:**
Agregar los 4 campos de trazabilidad a `tpPedido`: cancelación por falta de pago (`FechaCancelacionPorFaltaPago`, `IdUsuarioCancelacion`) y cancelación CFDI ante el SAT (`FechaSolicitudCancelacion`, `EstadoCancelacionCFDI` — OBS-042).

**Objetivos específicos:**
- `ALTER TABLE dbo.tpPedido ADD FechaCancelacionPorFaltaPago datetime2 NULL`
- `ALTER TABLE dbo.tpPedido ADD IdUsuarioCancelacion uniqueidentifier NULL`
- `ALTER TABLE dbo.tpPedido ADD FechaSolicitudCancelacion datetime2 NULL` (OBS-042)
- `ALTER TABLE dbo.tpPedido ADD EstadoCancelacionCFDI varchar(50) NULL` (OBS-042)
- Verificar SPs, vistas y triggers dependientes de `tpPedido`.

**Resultado esperado:**
`tpPedido` con los 4 campos NULL disponibles para el endpoint de cancelación (Tarea 9).

**Entregables:**
- Script DDL: 4 ALTER TABLE tpPedido
- Script de validación y checklist de dependencias

**Criterios de aceptación:**
- Los 4 campos existen como nullable en `tpPedido`.
- Sin errores en objetos dependientes tras el ALTER.
- Registros existentes con NULL en los nuevos campos.
- **OBS-042:** `FechaSolicitudCancelacion` y `EstadoCancelacionCFDI` incluidos en el script.

**Más información de la tarea:**
Ver sección *"A2"* en `TPSC-RE-FU-023-Back.md` y secciones *"4"* y *"5"* en `TPSC-RE-FU-023_BD.md`.

**Recursos:**
- `TPSC-RE-FU-023_BD.md` — Sección 4 y Sección 5 (OBS-042): ALTER tpPedido
- `TPSC-RE-FU-023-Back.md` — Parte A, A2

---

## TAREA 3

**[ RE-FU-023 ] [IMP-EXIST-SERVICE] Scaffold EF Core fccFolioPagoCliente, fccPagoCliente y tpProformaPedido en Finanzas**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Infraestructura — Validar Cobro

**Consideraciones previas:**
- La Tarea 1 debe estar completada: `fccPagoCliente` con los 5 campos del wizard incluidos.
  El Scaffold debe ejecutarse **después** de la Tarea 1 para que `Confirmado`, `FechaConfirmacion`,
  `IdUsuarioConfirmacion`, `Notas` e `IdCatMoneda` queden en las entidades generadas.
- `fccFolioPagoCliente` y `fccPagoCliente` ya existen en BD — solo agregar al Scaffold.
- `tpProformaPedido` ya existe en BD — se agrega al Scaffold de Finanzas para que las Tareas
  7, 8 y 9 puedan accederla directamente sin llamadas a la API de ProquifaDotNet.
- Seguir el patrón estándar de Finanzas: scaffold en `Infrastructure/Persistence/Scaffold`,
  configuración con `IEntityTypeConfiguration<T>` si se requieren ajustes, DbSets en el DbContext.
- No modificar manualmente las entidades generadas — usar Fluent API en archivos de configuración.

**Objetivo general:**
Generar las entidades EF Core para `fccFolioPagoCliente`, `fccPagoCliente` y `tpProformaPedido`
en `ProquifaDotNet.Finanzas.Infrastructure` para que los Queries y Commands de las Tareas 4-9
puedan consumirlas directamente.

**Objetivos específicos:**
- Ejecutar `dotnet ef dbcontext scaffold` con las 3 tablas apuntando a la BD `ProquifaDotNet`:
  ```
  --table fccFolioPagoCliente
  --table fccPagoCliente
  --table tpProformaPedido
  ```
- Verificar tipos de datos en las entidades generadas:
  - Decimales con precisión correcta (Monto, SubtotalMailBot, TipoDeCambio).
  - `Confirmado` como `bool` (bit → bool en EF Core).
  - Propiedades nullable correctamente mapeadas.
- Agregar `DbSet<fccFolioPagoCliente>`, `DbSet<fccPagoCliente>`, `DbSet<tpProformaPedido>`
  al DbContext de Finanzas.Infrastructure.
- Crear configuraciones `IEntityTypeConfiguration<T>` solo si los tipos generados requieren ajustes.

**Resultado esperado:**
Entidades `fccFolioPagoCliente`, `fccPagoCliente` y `tpProformaPedido` disponibles en el
`DbContext` de Finanzas.Infrastructure, listas para las Tareas 4-9.

**Entregables:**
- Entidades scaffoldeadas: `FccFolioPagoCliente.cs`, `FccPagoCliente.cs`, `TpProformaPedido.cs`
- `DbSet<>` de las 3 entidades en el DbContext
- Configuraciones `IEntityTypeConfiguration<T>` si se requieren ajustes
- El proyecto Finanzas compila sin errores tras el scaffold

**Criterios de aceptación:**
- `context.FccFolioPagoCliente`, `context.FccPagoCliente`, `context.TpProformaPedido` accesibles.
- Entidad `FccPagoCliente` incluye los 5 campos del wizard: `Confirmado`, `FechaConfirmacion`,
  `IdUsuarioConfirmacion`, `Notas`, `IdCatMoneda`.
- Los tipos de datos coinciden con la definición DDL.
- El scaffold no modifica entidades ya existentes en el DbContext.
- El proyecto `ProquifaDotNet.Finanzas` compila sin errores.

**Más información de la tarea:**
Ver sección *"B1"* en `TPSC-RE-FU-023-Back.md`.

**Recursos:**
- `TPSC-RE-FU-023-Back.md` — Parte B, B1
- `TPSC-RE-FU-023_BD.md` — Diccionario de datos de fccFolioPagoCliente y fccPagoCliente

---

## TAREA 4

**[ RE-FU-023 ] [SIMPLE-CRUD] CQRS fccFolioPagoCliente — Query pendientes + Command cerrar pendiente**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Application (CQRS)

**Consideraciones previas:**
- La Tarea 3 (Scaffold) debe estar completada.
- El INSERT en `fccFolioPagoCliente` lo hace el **Mailbot en ProquifaDotNet** —
  este CQRS es solo para **lectura** (Query pendientes por cliente) y **cierre** (Command Activo=0).
- El Query es consumido por la Tarea 6 (listado principal) para calcular `CobrosRecibidosPendientes`.
- El Command de cierre es invocado por el servicio de confirmación de cobro (RE-FU-024).
- No crear Command de INSERT — el INSERT lo gestiona ProquifaDotNet/Mailbot.
- Seguir la estructura CQRS estándar de Finanzas: `Query` + `QueryHandler`, `Command` + `CommandHandler`.

**Objetivo general:**
Implementar en ProquifaDotNet.Finanzas el Query para obtener los pendientes del Buzón por
cliente y el Command para cerrar un pendiente, consumiendo el scaffold de la Tarea 3.

**Objetivos específicos:**
- **Query `GetFolioPagoClientePendientesQuery`:**
  - Input: `Guid idCliente`
  - Handler: filtra `fccFolioPagoCliente WHERE Activo=true` y hace JOIN con
    `CorreoRecibidoCliente WHERE IdCliente=@idCliente`.
  - Output: `List<FolioPagoClientePendienteDto>` con `IdFCCFolioPagoCliente`, `Folio`,
    `TotalMailBot`, `SubtotalMailBot`, `IvaMailBot`, `FechaRecepcion`, `MxnMailBot`, `UsdMailBot`.
- **Command `CerrarFolioPagoClienteCommand`:**
  - Input: `Guid idFCCFolioPagoCliente`
  - Handler: `UPDATE fccFolioPagoCliente SET Activo=false, FechaUltimaActualizacion=UtcNow`.
  - Validar que el registro existe y está `Activo=true` antes de cerrar.
- Exponer vía endpoints en el Controller de Validar Cobro en Finanzas.API:
  - `GET /api/validar-cobro/buzon/{idCliente}/pendientes`
  - `PUT /api/validar-cobro/buzon/{id}/cerrar`
- Registrar en Serilog ante errores.

**Resultado esperado:**
`GetFolioPagoClientePendientesQuery` retorna los pendientes activos del Buzón por cliente.
`CerrarFolioPagoClienteCommand` marca `Activo=0` en el registro indicado.

**Entregables:**
- `GetFolioPagoClientePendientesQuery` + Handler en Application
- `FolioPagoClientePendienteDto` en Application/DTOs
- `CerrarFolioPagoClienteCommand` + Handler en Application
- Endpoints en Finanzas.API
- Pruebas unitarias de ambos Handlers

**Criterios de aceptación:**
- `GET /api/validar-cobro/buzon/{idCliente}/pendientes` retorna solo registros `Activo=true`.
- `PUT /api/validar-cobro/buzon/{id}/cerrar` pone `Activo=false` y actualiza `FechaUltimaActualizacion`.
- Retorna 404 si el registro no existe o ya está cerrado.

**Más información de la tarea:**
Ver sección *"B2"* en `TPSC-RE-FU-023-Back.md`.

**Recursos:**
- `TPSC-RE-FU-023-Back.md` — Parte B, B2
- `ProquifaDotNet-R14\L11.MailBot\Procesos\Pagos\CorreoRecibidoClienteToPagoBO.cs` — lógica INSERT referencia

---

## TAREA 5

**[ RE-FU-023 ] [SIMPLE-CRUD] CQRS fccPagoCliente — Query cobro + Command upsert borrador**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Application (CQRS)

**Consideraciones previas:**
- La Tarea 3 (Scaffold) debe estar completada, incluyendo los 5 campos del wizard en la entidad.
- Este CQRS es la **puerta de entrada del wizard**: Finanzas inserta el cobro como borrador
  al auto-guardar en el Paso 1 (RE-FU-024) y lo actualiza con cada cambio.
- El borrador se identifica por `Confirmado=0` (campo disponible desde la Tarea 1).
- El Command de confirmación definitiva (generar Folio, poblar `Confirmado=1`,
  `FechaConfirmacion`, `IdUsuarioConfirmacion`) se implementa en **RE-FU-024**.
- Seguir la estructura CQRS estándar de Finanzas.

**Objetivo general:**
Implementar el Query para obtener un cobro por Id o listar cobros activos por cliente,
y el Command para insertar o actualizar el borrador del cobro en `fccPagoCliente`.

**Objetivos específicos:**
- **Query `GetPagoClienteQuery`:**
  - Por Id: `fccPagoCliente WHERE IdFCCPagoCliente=@id AND Activo=true`.
  - Por cliente: `fccPagoCliente WHERE IdCliente=@idCliente AND Activo=true`.
  - Output: `PagoClienteDto` con todos los campos incluyendo `Confirmado`, `FechaConfirmacion`,
    `IdUsuarioConfirmacion`, `Notas`, `IdCatMoneda`.
- **Command `UpsertPagoClienteCommand`:**
  - Si `IdFCCPagoCliente` es `Guid.Empty` o no existe → INSERT con `Activo=1`, `Confirmado=0`.
  - Si ya existe → UPDATE de los campos editables del formulario Paso 1:
    `Monto`, `FechaPago`, `TipoDeCambio`, `IdCatMedioDePago`, `IdDatosBancarios`,
    `IdCatBanco`, `CuentaOrdenante`, `ReferenciaBancaria`, `MXN`, `USD`,
    `FechaUltimaActualizacion`.
  - Retorna `IdFCCPagoCliente` (nuevo o existente).
- Exponer vía endpoints en el Controller de Validar Cobro en Finanzas.API:
  - `GET /api/validar-cobro/cobros/{id}`
  - `GET /api/validar-cobro/cobros?idCliente={id}`
  - `PUT /api/validar-cobro/cobros`
- Registrar en Serilog ante errores.

**Resultado esperado:**
`GetPagoClienteQuery` retorna el cobro o lista de cobros del cliente con todos los campos.
`UpsertPagoClienteCommand` crea o actualiza el borrador del cobro sin confirmarlo.

**Entregables:**
- `GetPagoClienteQuery` + Handler en Application
- `PagoClienteDto` en Application/DTOs (con los 5 campos del wizard)
- `UpsertPagoClienteCommand` + Handler en Application
- Endpoints en Finanzas.API
- Pruebas unitarias de ambos Handlers

**Criterios de aceptación:**
- `GET /api/validar-cobro/cobros/{id}` retorna el cobro con `Activo=true` e incluye `Confirmado`.
- `PUT /api/validar-cobro/cobros` (nuevo) inserta con `Confirmado=0` y retorna el nuevo Id.
- `PUT /api/validar-cobro/cobros` (existente) actualiza solo los campos editables.
- `Confirmado`, `FechaConfirmacion`, `IdUsuarioConfirmacion`, `Notas`, `IdCatMoneda` presentes en el DTO.
- El proyecto compila y las pruebas unitarias pasan.

**Más información de la tarea:**
Ver sección *"B3"* en `TPSC-RE-FU-023-Back.md`.

**Recursos:**
- `TPSC-RE-FU-023-Back.md` — Parte B, B3
- `TPSC-RE-FU-023_BD.md` — Estructura `fccPagoCliente`
- `TPSC-RE-FU-024_BD.md` — Contexto de confirmación que extiende este Command

---

## TAREA 6

**[ RE-FU-023 ] [LIST-PAG-MULT-FILTER] Listado de clientes de Validar Cobro con acción contextual**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Pantalla Principal

**Consideraciones previas:**
- Las Tareas 3 y 4 deben completarse: el Scaffold y `GetFolioPagoClientePendientesQuery` disponibles.
- **Patrón QueryInfo** (mismo que Punchout): el endpoint es `[HttpPost]` y recibe `[FromBody] QueryInfo`.
  Retorna `QueryResultDto<ClienteValidarCobroDto>`.
- `QueryInfo` encapsula: `SortField`, `SortDirection`, `Filters` (`List<FilterItem>`), `PageSize`, `DesiredPage`.
- `QueryableExtensions` aplica ordenamiento, paginación y filtros sobre `IQueryable<T>`.
  Para filtros LIKE (`textoBusqueda`) se implementa el case en el repositorio.
- **Sin filtro `idRegion`**: según Criterio B2 del requisito, no hay filtros adicionales.
  La región y cartera del usuario se resuelven del token de autenticación (IdentityServer).
- `AccionContextual` es calculado en el Service: `REALIZAR_COBROS` si `CobrosRecibidosPendientes > 0`,
  `GESTIONAR_COBRANZA` si no.
- Todas las tablas (`ClienteCartera`, `tpProformaPedido`, `fccFolioPagoCliente`) se leen vía
  Scaffold Finanzas — sin llamadas a la API de ProquifaDotNet.
- Solo muestra clientes de la cartera del usuario: `ClienteCartera.IdUsuarioCobrador = @IdUsuarioActivo`.
- **OBS-041:** Aplicar trim automático al valor de `textoBusqueda` antes de ejecutar el filtrado (ignorar espacios al inicio y al final).
- **OBS-046:** `SaldoPendienteTotal` siempre en USD — convertir `MontoPendiente` a USD usando el `ConversorDivisas` existente en ProquifaDotNet.
- **OBS-047:** El listado se ordena por antigüedad del cobro recibido más antiguo del cliente (`MIN fccFolioPagoCliente.FechaRecepcion`, ASC). Clientes sin cobros pendientes se ubican al final. Agregar campo `TieneSlaVencido` (bool): `true` si el cobro más antiguo lleva más de 72 horas sin procesar.

**Objetivo general:**
`POST /api/validar-cobro/clientes` con `[FromBody] QueryInfo` que retorna
`QueryResultDto<ClienteValidarCobroDto>` paginado con `AccionContextual` calculada,
siguiendo el patrón QueryInfo del repo Punchout.

**Objetivos específicos:**
- **`Domain.Interfaces`** → `IClienteValidarCobroQueryRepository` con `GetFilteredAsync(QueryInfo)`.
- **`Application.Interfaces`** → `IClienteValidarCobroService` con `GetAsync(QueryInfo)`.
- **`Application.Services`** → `ClienteValidarCobroService`:
  - Llama al repositorio con `QueryInfo`.
  - Calcula `CobrosRecibidosPendientes` (COUNT `fccFolioPagoCliente.Activo=true` por cliente).
  - Calcula `ProformasFacturasPendientes` (COUNT `tpProformaPedido` con `MontoPendiente > 0`, `Cancelada=0`).
  - Calcula `SaldoPendienteTotal` (SUM `tpProformaPedido.MontoPendiente`).
  - Calcula `AccionContextual`.
  - Mapea a `ClienteValidarCobroDto`.
- **`Application.DTOs`** → `ClienteValidarCobroDto` con: `IdCliente`, `Alias`, `RFC`,
  `CobrosRecibidosPendientes`, `ProformasFacturasPendientes`, `SaldoPendienteTotalUsd` (en USD, OBS-046),
  `FechaCobroMasAntiguo` (nullable, para ordenamiento OBS-047), `TieneSlaVencido` (bool, OBS-047),
  `AccionContextual`.
- **`Infrastructure.Repository`** → `ClienteValidarCobroRepository`:
  - Filtro soportado vía `QueryInfo.Filters`:
    - `textoBusqueda` → trim automático del valor ingresado, luego `WHERE TRIM(Alias) LIKE % OR TRIM(RFC) LIKE %` (case custom en switch). OBS-041.
  - Aplica filtro de cartera/región desde claims del usuario autenticado.
  - Usa `ToPagedList(queryInfo, out totalCount)` de `QueryableExtensions`.
- **`API.Controllers`** → `ValidarCobroController` con `[HttpPost] POST /api/validar-cobro/clientes`
  y `[FromBody] QueryInfo`.

**Resultado esperado:**
`POST /api/validar-cobro/clientes` retorna `QueryResultDto<ClienteValidarCobroDto>` paginado
con `AccionContextual` correcto por cliente.

**Entregables:**
- `IClienteValidarCobroQueryRepository` en Domain.Interfaces
- `ClienteValidarCobroRepository` en Infrastructure.Repository (con `QueryInfo` + `QueryableExtensions`)
- `IClienteValidarCobroService` + `ClienteValidarCobroService` en Application
- `ClienteValidarCobroDto` + reutilización de `QueryResultDto<T>` en Application.DTOs
- Endpoint `POST /api/validar-cobro/clientes` en API.Controllers
- Pruebas unitarias (2 escenarios de `AccionContextual`, 1 escenario paginación)

**Criterios de aceptación:**
- Request `POST` con `QueryInfo` vacío retorna primera página (PageSize=50 por defecto).
- Filtro `textoBusqueda` aplica trim automático al valor ingresado antes de ejecutar el LIKE (OBS-041).
- `TotalResults` refleja el total antes de paginar.
- `CobrosRecibidosPendientes` = COUNT real `fccFolioPagoCliente.Activo=true` del cliente.
- `AccionContextual` = `REALIZAR_COBROS` si pendientes > 0, `GESTIONAR_COBRANZA` si no.
- Solo retorna clientes de la cartera del usuario activo (por claims del token).
- `SaldoPendienteTotalUsd` se expresa siempre en USD, aplicando conversión de moneda (OBS-046).
- El listado se ordena por `FechaCobroMasAntiguo` ASC (cobro recibido más antiguo primero). Clientes sin cobros pendientes al final (OBS-047).
- `TieneSlaVencido = true` cuando `FechaCobroMasAntiguo` supera las 72 horas sin procesar (OBS-047).

**Más información de la tarea:**
Ver sección *"C1"* en `TPSC-RE-FU-023-Back.md`. Patrón de referencia: `BrandController`,
`BrandService`, `BrandRepository`, `QueryableExtensions` en repo Punchout.

**Recursos:**
- `TPSC-RE-FU-023-Back.md` — Parte C, C1
- `TPSC-RE-FU-023_BD.md` — Cadena de datos y lógica AccionContextual
- `proquifa-punchout-backend/Domain/Common/QueryInfo.cs`
- `proquifa-punchout-backend/Infrastructure/Extensions/QueryableExtensions.cs`
- `proquifa-punchout-backend/API/Controllers/BrandController.cs`
- `proquifa-punchout-backend/Application/Services/BrandService.cs`
- `proquifa-punchout-backend/Infrastructure/Persistence/Repository/BrandRepository.cs`

---

## TAREA 7

**[ RE-FU-023 ] [LIST-NO-FILTER] Listado de pedidos pendientes para modal Gestionar Cobranza**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Modal Gestionar Cobranza

**Consideraciones previas:**
- La Tarea 3 (Scaffold tpProformaPedido) debe estar completada.
- Solo se invoca cuando `AccionContextual = GESTIONAR_COBRANZA`.
- `tpProformaPedido` se accede directamente vía Scaffold Finanzas — no vía API ProquifaDotNet.
- La cabecera del response incluye `MontoTotalPendiente` (SUM MontoPendiente del cliente).
- Validar que el cliente pertenece a la cartera del usuario activo (por claims del token).

**Objetivo general:**
`GET /api/validar-cobro/proformas?idCliente={idCliente}` en Finanzas que alimenta
el modal Gestionar Cobranza con los pedidos pendientes del cliente y datos de contacto.

**Objetivos específicos:**
- Query + Handler: `GetProformaPedidoClienteQuery`.
  - Filtro: `tpProformaPedido WHERE IdCliente=@idCliente AND MontoPendiente > 0 AND Cancelada=0`.
  - JOIN `tpPedido` para `PedidoInterno` y `ReferenciaPedidoCliente`.
  - JOIN `ContactoCliente` para nombre, correo y teléfono del contacto.
  - Calcula `MontoTotalPendiente` = SUM `MontoPendiente`.
- DTO: `ProformaPedidoDto` con: `IdTpProformaPedido`, `IdTpPedido`, `PedidoInterno`,
  `ReferenciaPedidoCliente`, `MontoPendiente`, `FechaPromesaPagoMonitoreoCobros` (nullable),
  `ContactoNombre`, `ContactoCorreo`, `ContactoTelefono`.
- Response incluye `MontoTotalPendiente` en cabecera.

**Resultado esperado:**
Endpoint alimenta el modal con los pedidos pendientes del cliente y `MontoTotalPendiente`.

**Entregables:**
- Endpoint `GET /api/validar-cobro/proformas?idCliente={id}`
- `GetProformaPedidoClienteQuery` + Handler en Application
- `ProformaPedidoDto` + DTO de respuesta con `MontoTotalPendiente`
- Pruebas unitarias del Handler

**Criterios de aceptación:**
- Solo retorna proformas con `MontoPendiente > 0` y `Cancelada=0`.
- `MontoTotalPendiente` en respuesta es el SUM correcto.
- Retorna 404 si el cliente no pertenece a la cartera del usuario activo.
- `FechaPromesaPagoMonitoreoCobros` puede ser null (campo editable en el modal).

**Más información de la tarea:**
Ver sección *"B4"* y *"C2"* en `TPSC-RE-FU-023-Back.md`.

**Recursos:**
- `TPSC-RE-FU-023-Back.md` — Parte B, B4 y Parte C, C2

---

## TAREA 8

**[ RE-FU-023 ] [SERV-SIMPLE-PUT] Actualizar fecha estimada de pago desde modal Gestionar Cobranza**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Modal Gestionar Cobranza

**Consideraciones previas:**
- La Tarea 3 (Scaffold tpProformaPedido) debe estar completada.
- La Tarea 10 (CREATE TABLE fccFechaEstimadaPagoHistorial) debe completarse antes de esta tarea.
- `tpProformaPedido.FechaPromesaPagoMonitoreoCobros` ya existe en la tabla — solo es un UPDATE.
- Se actualiza directamente vía Scaffold Finanzas — no vía API ProquifaDotNet.
- Puede actualizar varios pedidos a la vez al presionar "Confirmar" en el modal.
- La fecha puede enviarse como `null` para limpiarla.
- Validar que los pedidos pertenecen a la cartera del usuario activo (por claims del token).
- **OBS-044:** Cada cambio de fecha genera un INSERT en `fccFechaEstimadaPagoHistorial` con el valor anterior y el nuevo. No se sobreescribe el historial — es append-only.

**Objetivo general:**
`PUT /api/validar-cobro/proformas/fecha-promesa` en Finanzas que persiste
`FechaPromesaPagoMonitoreoCobros` en `tpProformaPedido` y guarda el historial completo de cambios en `fccFechaEstimadaPagoHistorial` (OBS-044).

**Objetivos específicos:**
- Command + Handler: `UpdateFechaPromesaPagoCommand`.
  - Input: `List<(Guid IdTpProformaPedido, DateTime? FechaPromesaPago, Guid IdUsuarioCambio, string? Motivo)>`.
  - Handler (por cada ítem):
    1. Leer `FechaPromesaPagoMonitoreoCobros` actual.
    2. UPDATE `tpProformaPedido.FechaPromesaPagoMonitoreoCobros = @fechaNueva`.
    3. INSERT `fccFechaEstimadaPagoHistorial` (anterior, nueva, timestamp, usuario, motivo). (OBS-044)
  - Permitir fecha `null` para limpiar (registrar en historial como `FechaEstimadaPagaNueva = null`).
- Agregar `fccFechaEstimadaPagoHistorial` al Scaffold de Finanzas (después de Tarea 10).
- Validar pertenencia de los pedidos a la cartera del usuario activo.
- Registrar en Serilog con contexto usuario/módulo/operación.

**Resultado esperado:**
`FechaPromesaPagoMonitoreoCobros` actualizada y fila de historial insertada en `fccFechaEstimadaPagoHistorial` por cada cambio (OBS-044).

**Entregables:**
- Endpoint `PUT /api/validar-cobro/proformas/fecha-promesa`
- `UpdateFechaPromesaPagoCommand` + Handler en Application (con historial OBS-044)
- DTO: `ActualizarFechaPromesaPagoRequestDto`
- Pruebas unitarias del Handler (incluyendo que el historial se inserta)

**Criterios de aceptación:**
- `FechaPromesaPagoMonitoreoCobros` actualizada por cada pedido enviado.
- Se permite `null` para limpiar la fecha.
- Valida que pedidos pertenecen a la cartera del usuario activo.
- Registra en Serilog con contexto.
- **OBS-044:** Por cada cambio de fecha se inserta una fila en `fccFechaEstimadaPagoHistorial` con `FechaEstimadaPagoAnterior`, `FechaEstimadaPagaNueva`, `FechaCambio` (UTC), `IdUsuarioCambio` y `Motivo` opcional.
- **OBS-044:** Si la fecha es `null` (limpiar), se registra `FechaEstimadaPagaNueva = null` en el historial.
- **OBS-044:** El historial es append-only — nunca UPDATE ni DELETE sobre `fccFechaEstimadaPagoHistorial`.

**Más información de la tarea:**
Ver sección *"B4"* y *"C3"* en `TPSC-RE-FU-023-Back.md`. Sección *"6"* en `TPSC-RE-FU-023_BD.md`.

**Recursos:**
- `TPSC-RE-FU-023-Back.md` — Parte B, B4 y Parte C, C3 (OBS-044)
- `TPSC-RE-FU-023_BD.md` — Sección 6 (OBS-044): CREATE TABLE fccFechaEstimadaPagoHistorial

---

## TAREA 9

**[ RE-FU-023 ] [SERV-TRANSACT] Cancelar pedido por falta de pago desde modal Gestionar Cobranza**

**Aplicativos:** ProquifaDotNet + ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Modal Gestionar Cobranza

**Consideraciones previas:**
- La Tarea 2 debe completarse (4 campos nuevos en `tpPedido` incluyendo OBS-042).
- La cancelación requiere un **endpoint nuevo en ProquifaDotNet** (`tpPedidoController`) ya que
  `tpPedido` no está en el Scaffold de Finanzas y la transacción toca ambas tablas
  (`tpProformaPedido` y `tpPedido`) en la misma BD.
- Finanzas llama al endpoint de ProquifaDotNet vía API con el Id del pedido y el Id del usuario.
- Acción inmediata al presionar "Cancelar Pedido" — no requiere confirmar el modal general.
- El pedido cancelado sale del listado del modal y del listado principal (C1 y C2).
- **OBS-042:** Si el pedido tiene un CFDI en estado `Timbrada`, la cancelación debe además: (1) cambiar el estado de `CFDIGenerada` a `CancelacionSolicitada`, (2) enviar solicitud de cancelación al SAT vía ProquifaDotNet.Timbrado, (3) actualizar `tpPedido.FechaSolicitudCancelacion` y `tpPedido.EstadoCancelacionCFDI`. Este flujo puede ser asíncrono (CFDI 4.0 requiere aceptación del receptor).
- ⚠️ Confirmar si debe disparar `spActualizarBuzonPagoLegacyLegacy`
  (ver `CorreoRecibidoClienteToPagoBO.ActualizarBuzonPagoLegacy`).
- Registrar en Serilog con contexto usuario/módulo/operación.

**Objetivo general:**
Crear el endpoint `PUT /api/pedidos/{idTpPedido}/cancelar-falta-pago` en ProquifaDotNet
y el Command en Finanzas que lo invoca, para cancelar el pedido con trazabilidad completa (incluyendo cancelación CFDI ante el SAT — OBS-042).

**Objetivos específicos:**
- **ProquifaDotNet — endpoint nuevo:**
  - `PUT /api/pedidos/{idTpPedido}/cancelar-falta-pago` en `tpPedidoController`.
  - Transacción:
    ```sql
    UPDATE tpProformaPedido SET Cancelada = 1 WHERE IdTpPedido = @idTpPedido
    UPDATE tpPedido SET FechaCancelacionPorFaltaPago = SYSUTCDATETIME(),
                        IdUsuarioCancelacion = @idUsuario
        WHERE IdTpPedido = @idTpPedido
    ```
  - **OBS-042:** Si el pedido tiene CFDI en estado `Timbrada`:
    - UPDATE `CFDIGenerada.Estado = 'CancelacionSolicitada'`
    - UPDATE `tpPedido.FechaSolicitudCancelacion = SYSUTCDATETIME(), EstadoCancelacionCFDI = 'Pendiente'`
    - Invocar API ProquifaDotNet.Timbrado para enviar cancelación al SAT.
    - Actualizar `tpPedido.EstadoCancelacionCFDI` con el resultado (`'Cancelado'`, `'Rechazado'`, etc.)
  - Validar que el pedido existe y no está ya cancelado.
- **ProquifaDotNet.Finanzas — Command:**
  - Command + Handler: `CancelarPedidoPorFaltaDePagoCommand`.
  - Input: `Guid idTpPedido`, `Guid idUsuario`.
  - Llama al endpoint de ProquifaDotNet vía API HTTP.
  - Valida pertenencia del pedido a la cartera del usuario activo.
  - Registra en Serilog con contexto usuario/módulo/operación.
- Endpoint Finanzas: `PUT /api/validar-cobro/pedidos/{idTpPedido}/cancelar-falta-pago`.

**Resultado esperado:**
Pedido cancelado con trazabilidad completa: `tpProformaPedido.Cancelada=1`, `tpPedido.FechaCancelacionPorFaltaPago` e `IdUsuarioCancelacion` poblados. Si aplica, CFDI cancelado ante el SAT con estado registrado en `tpPedido.FechaSolicitudCancelacion` y `EstadoCancelacionCFDI` (OBS-042).

**Entregables:**
- Endpoint nuevo en ProquifaDotNet: `PUT /api/pedidos/{idTpPedido}/cancelar-falta-pago`
- Lógica OBS-042: integración con ProquifaDotNet.Timbrado para cancelación CFDI
- `CancelarPedidoPorFaltaDePagoCommand` + Handler en Finanzas.Application
- Endpoint en Finanzas.API: `PUT /api/validar-cobro/pedidos/{id}/cancelar-falta-pago`
- Pruebas unitarias del Handler (incluyendo error en llamada a ProquifaDotNet)

**Criterios de aceptación:**
- `tpProformaPedido.Cancelada=1` tras la cancelación.
- `tpPedido.FechaCancelacionPorFaltaPago` e `IdUsuarioCancelacion` quedan poblados.
- El pedido no aparece en el listado del modal ni en el listado principal.
- Errores registrados en Serilog con contexto usuario/módulo/operación.
- Retorna error si el pedido no existe o ya está cancelado.
- **OBS-042:** Si el pedido tiene CFDI en estado `Timbrada`, se envía solicitud de cancelación al SAT y se actualiza `tpPedido.FechaSolicitudCancelacion` y `EstadoCancelacionCFDI`.
- **OBS-042:** `CFDIGenerada.Estado` pasa a `CancelacionSolicitada` cuando se inicia la cancelación.

**Más información de la tarea:**
Ver sección *"C4"* en `TPSC-RE-FU-023-Back.md`.

**Recursos:**
- `TPSC-RE-FU-023-Back.md` — Parte C, C4 (OBS-042)
- `TPSC-RE-FU-023_BD.md` — Sección 5 (OBS-042): ALTER tpPedido cancelación CFDI
- `ProquifaDotNet-R14\L11.MailBot\Procesos\Pagos\CorreoRecibidoClienteToPagoBO.cs` — método `ActualizarBuzonPagoLegacy`

---

## TAREA 10

**[ RE-FU-023 ] [UPDATE-TABL-CH] Crear tabla fccFechaEstimadaPagoHistorial (OBS-044)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro

**Consideraciones previas:**
- Tabla nueva — no existe en BD (OBS-044).
- Es append-only: solo INSERT, nunca UPDATE ni DELETE.
- Prerrequisito de la Tarea 8 (`UpdateFechaPromesaPagoCommand` con historial).
- Independiente de las Tareas 1 y 2 — puede ejecutarse en paralelo.

**Objetivo general:**
Crear la tabla `fccFechaEstimadaPagoHistorial` para guardar el historial completo de cambios de `FechaPromesaPagoMonitoreoCobros` (FechaEstimadaPago) en `tpProformaPedido`. Cada cambio genera una fila nueva.

**Objetivos específicos:**
- Ejecutar script DDL:
  ```sql
  CREATE TABLE dbo.fccFechaEstimadaPagoHistorial (
      IdFccFechaEstimadaPagoHistorial uniqueidentifier NOT NULL
          CONSTRAINT PK_fccFechaEstimadaPagoHistorial PRIMARY KEY
          CONSTRAINT DF_fccFechaEstimadaPagoHistorial_Id DEFAULT (NEWID()),
      IdTpProformaPedido             uniqueidentifier NOT NULL
          CONSTRAINT FK_fccFechaEstimadaPagoHistorial_ProformaPedido
              FOREIGN KEY REFERENCES dbo.tpProformaPedido(IdTpProformaPedido),
      FechaEstimadaPagoAnterior      datetime2        NULL,
      FechaEstimadaPagaNueva         datetime2        NULL,
      FechaCambio                    datetime2        NOT NULL
          CONSTRAINT DF_fccFechaEstimadaPagoHistorial_FechaCambio DEFAULT (SYSUTCDATETIME()),
      IdUsuarioCambio                uniqueidentifier NOT NULL,
      Motivo                         varchar(300)     NULL
  );

  CREATE NONCLUSTERED INDEX IX_fccFechaEstimadaPagoHistorial_ProformaPedido
      ON dbo.fccFechaEstimadaPagoHistorial (IdTpProformaPedido, FechaCambio DESC);
  ```
- Verificar FK a `tpProformaPedido` vigente en BD.

**Resultado esperado:**
Tabla `fccFechaEstimadaPagoHistorial` creada en ProquifaDotNet, lista para ser consumida por el Scaffold de Finanzas (Tarea 3/8).

**Entregables:**
- Script DDL: CREATE TABLE + índice
- Script de validación post-creación

**Criterios de aceptación:**
- Tabla existe en BD con los 7 campos definidos.
- FK a `tpProformaPedido` activa y sin errores.
- Índice `IX_fccFechaEstimadaPagoHistorial_ProformaPedido` creado.
- Registros de prueba pueden insertarse correctamente (sin UPDATE ni DELETE).

**Más información de la tarea:**
Ver sección *"6"* en `TPSC-RE-FU-023_BD.md` y OBS-044.

**Recursos:**
- `TPSC-RE-FU-023_BD.md` — Sección 6 (OBS-044): CREATE TABLE fccFechaEstimadaPagoHistorial