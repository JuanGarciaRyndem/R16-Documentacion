# Impacto en Back — R16A-RE-FU-023
**Requisito:** Validar Cobro — Pantalla Principal
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)
**Módulo:** Validar Cobro — Pantalla Principal
**Impacto:** ✅ fccFolioPagoCliente y fccPagoCliente ya existen en BD (verificado) — ALTER fccPagoCliente x5 ❌ pendiente — ALTER tpPedido x2 ❌ pendiente — ProquifaDotNet.Finanzas: Scaffold EF Core (fccFolioPagoCliente, fccPagoCliente, tpProformaPedido) + CQRS + Endpoints REST — tpPedido cancelación vía endpoint dedicado ProquifaDotNet

---

## Resumen

Este requisito implementa la **pantalla principal del módulo Validar Cobro** en **ProquifaDotNet.Finanzas**.
Las dos tablas base del flujo Validar Cobro **ya existen en ProquifaDotNet**:

- `fccFolioPagoCliente` — pendiente generado al clasificar un correo como Cobro en el Buzón.
- `fccPagoCliente` — registro del cobro capturado en el wizard.

El **CRUD de ambas tablas reside en ProquifaDotNet.Finanzas** (.NET Core 10), no en ProquifaDotNet.
Finanzas accede directamente via **EF Core Scaffold** de la base de datos ProquifaDotNet.
`tpProformaPedido` también se integra al modelo de Finanzas vía Scaffold (no via API).
Las operaciones sobre `tpPedido` (cancelación) se realizan mediante un **endpoint dedicado en ProquifaDotNet**.

La lógica de INSERT en `fccFolioPagoCliente` **ya existe** en `CorreoRecibidoClienteToPagoBO.cs`
(Mailbot RE-FU-008) y no debe duplicarse.

### Hallazgos en código fuente

| Archivo                                                        | Relevancia                                                                                                                                                                                                                |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `L11.MailBot/Procesos/Pagos/CorreoRecibidoClienteToPagoBO.cs`  | **Ya hace INSERT en `fccFolioPagoCliente`** al clasificar correo como cobro. Lógica: itera referencias, crea registro con Consecutivo, FechaRecepcion, SubtotalMailBot, IvaMailBot, TotalMailBot, Folio. **No duplicar.** |
| `CorreoRecibidoClienteToPagoBO.ActualizarBuzonPagoLegacy`      | Llama `spActualizarBuzonPagoLegacyLegacy`. Confirmar si aplica al cancelar pedido desde Validar Cobro.                                                                                                                    |
| `L05.TramitarPedido/Dashboard/FacturasPendientesClienteObj.cs` | Patrón de cálculo multi-divisa con `ConversorDivisas`. Base técnica para el motor de saldo del Paso 2 (RE-FU-026).                                                                                                        |
| `Logic.MailXslt/Cobranza/CorreoFCCPagoCliente.cs`              | Plantilla de correo para `fccPagoClienteCorreoEnviado`. Confirmar si aplica a la confirmación del cobro (RE-FU-024).                                                                                                      |

### Distribución de responsabilidades

| Capa | Aplicativo | Responsabilidad |
| ---- | ---------- | --------------- |
| BD — tablas base | ProquifaDotNet | ✅ `fccFolioPagoCliente` (existente), ✅ `fccPagoCliente` (existente), ❌ ALTER `fccPagoCliente` x5 (pendiente), ❌ ALTER `tpPedido` x2 (pendiente) |
| INSERT `fccFolioPagoCliente` | ProquifaDotNet — Mailbot (existente) | `CorreoRecibidoClienteToPagoBO.cs` — no duplicar |
| CRUD `fccFolioPagoCliente` | ProquifaDotNet.Finanzas — Scaffold + CQRS | Scaffold EF Core + Query pendientes + Command cerrar pendiente |
| CRUD `fccPagoCliente` | ProquifaDotNet.Finanzas — Scaffold + CQRS | Scaffold EF Core + Query cobro + Command INSERT borrador / UPDATE borrador |
| Listado pedidos / fecha pago | ProquifaDotNet.Finanzas — Scaffold + CQRS | `tpProformaPedido` vía Scaffold + Query listado + Command actualizar fecha |
| Cancelar pedido | ProquifaDotNet — endpoint dedicado | `PUT /api/pedidos/{id}/cancelar-falta-pago` — UPDATE tpProformaPedido + UPDATE tpPedido |
| Lógica pantalla principal | ProquifaDotNet.Finanzas | Listado clientes QueryInfo, conteos, acción contextual, modal Gestionar Cobranza |

### Infraestructura creada en este requisito

| Componente                           | Estado                              | Consumidor                                                   |
| ------------------------------------ | ----------------------------------- | ------------------------------------------------------------ |
| `fccFolioPagoCliente` tabla          | ✅ Existente (ProquifaDotNet)        | Mailbot (ya escribe), Finanzas (Scaffold EF Core)            |
| `fccPagoCliente` tabla               | ✅ Existente (ProquifaDotNet)        | Finanzas — wizard RE-FU-024 a 029                            |
| ALTER `fccPagoCliente` x5 campos     | ❌ Pendiente                         | Finanzas — wizard RE-FU-024 a 029                            |
| ALTER `tpPedido` x2 campos           | ❌ Pendiente                         | Modal Gestionar Cobranza — trazabilidad cancelación          |
| Scaffold `fccFolioPagoCliente`       | EF Core en Finanzas.Infrastructure  | Queries/Commands del módulo Validar Cobro                    |
| Scaffold `fccPagoCliente`            | EF Core en Finanzas.Infrastructure  | Queries/Commands del módulo Validar Cobro                    |
| Scaffold `tpProformaPedido`          | EF Core en Finanzas.Infrastructure  | Listado modal + actualizar fecha pago + cancelación          |
| `GetPendingPaymentFoliosQuery` | Finanzas.Application — CQRS Query   | Conteo `PendingPaymentsReceived` en listado principal      |
| `ClosePaymentFolioCommand`      | Finanzas.Application — CQRS Command | Cierra pendiente (`Activo=0`) al confirmar cobro (RE-FU-024) |
| `GetClientPaymentQuery`                | Finanzas.Application — CQRS Query   | Lee cobro por Id / por cliente                               |
| `UpsertClientPaymentCommand`           | Finanzas.Application — CQRS Command | INSERT borrador Paso 1 / UPDATE borrador                     |
| `GetClientQuoteQuery`      | Finanzas.Application — CQRS Query   | Listado pedidos pendientes modal Gestionar Cobranza          |
| `UpdatePromiseDateCommand`      | Finanzas.Application — CQRS Command | Actualiza FechaPromesaPagoMonitoreoCobros por pedido         |
| `CancelOrderForNonPaymentCommand`     | ProquifaDotNet — endpoint dedicado  | UPDATE tpProformaPedido + UPDATE tpPedido (trazabilidad)     |

---

## Parte A — Base de Datos (ProquifaDotNet)

### A1 — ALTER TABLE fccPagoCliente — Campos Wizard Validar Cobro

> **❌ PENDIENTE de ejecutar.** Los campos NO existen aún en fccPagoCliente (verificado).
> Estos campos son requeridos a partir de RE-FU-024 pero se crean aquí como parte de la
> infraestructura base del módulo Validar Cobro.

```sql
-- ❌ PENDIENTE — ejecutar en ProquifaDotNet
ALTER TABLE dbo.fccPagoCliente
    ADD Confirmado bit NOT NULL CONSTRAINT DF_fccPagoCliente_Confirmado DEFAULT (0);

ALTER TABLE dbo.fccPagoCliente
    ADD FechaConfirmacion datetime NULL;

ALTER TABLE dbo.fccPagoCliente
    ADD IdUsuarioConfirmacion uniqueidentifier NULL;

ALTER TABLE dbo.fccPagoCliente
    ADD Notas varchar(500) NULL;

ALTER TABLE dbo.fccPagoCliente
    ADD IdCatMoneda uniqueidentifier NULL;

ALTER TABLE dbo.fccPagoCliente
    ADD CONSTRAINT FK_fccPagoCliente_catMoneda
        FOREIGN KEY (IdCatMoneda) REFERENCES dbo.catMoneda (IdCatMoneda);
```

### A2 — ALTER TABLE tpPedido — Trazabilidad de Cancelación + CFDI (OBS-042)

> **❌ PENDIENTE de ejecutar.** Los campos NO existen aún en tpPedido (verificado).

```sql
-- ❌ PENDIENTE — ejecutar en ProquifaDotNet
-- Trazabilidad cancelación por falta de pago
ALTER TABLE dbo.tpPedido
    ADD FechaCancelacionPorFaltaPago datetime NULL;

ALTER TABLE dbo.tpPedido
    ADD IdUsuarioCancelacion uniqueidentifier NULL;

-- OBS-042: trazabilidad cancelación CFDI ante el SAT
ALTER TABLE dbo.tpPedido
    ADD FechaSolicitudCancelacion datetime NULL;

ALTER TABLE dbo.tpPedido
    ADD EstadoCancelacionCFDI varchar(50) NULL;
```

> **OBS-042:** `FechaSolicitudCancelacion` y `EstadoCancelacionCFDI` se populan al ejecutar la cancelación del CFDI ante el SAT desde el modal Gestionar Cobranza (complementan `FechaCancelacionPorFaltaPago` que registra la cancelación del pedido). Son conceptos distintos: la cancelación del pedido y la cancelación del CFDI ante el SAT ocurren como pasos separados.

---

## Parte B — ProquifaDotNet.Finanzas: Scaffold + CQRS tablas base

Las tablas `fccFolioPagoCliente`, `fccPagoCliente` y `tpProformaPedido` se acceden desde
Finanzas mediante **EF Core Scaffold** de la BD ProquifaDotNet, siguiendo el patrón estándar
de la solución: Infrastructure genera las entidades scaffoldeadas; Application implementa los
Queries y Commands en CQRS.

### B1 — Scaffold EF Core: fccFolioPagoCliente + fccPagoCliente + tpProformaPedido

**Capa:** `ProquifaDotNet.Finanzas.Infrastructure`

Las tablas `fccFolioPagoCliente`, `fccPagoCliente` y `tpProformaPedido` se agregan al scaffold
existente de la BD ProquifaDotNet en el proyecto Finanzas. Esto genera las entidades EF Core
que los Queries y Commands usan directamente.

```bash
# Comando referencial — ajustar connection string y output al proyecto real:
dotnet ef dbcontext scaffold "Server=RYNL010;Database=ProquifaDotNet;Integrated Security=True;..." Microsoft.EntityFrameworkCore.SqlServer \
  --table fccFolioPagoCliente \
  --table fccPagoCliente \
  --table tpProformaPedido \
  --output-dir Infrastructure/Persistence/Scaffold \
  --context-dir Infrastructure/Persistence \
  --no-onconfiguring
```

> El scaffold de `fccPagoCliente` se ejecuta **después** de que los ALTER de la Tarea A1
> estén aplicados en la BD `ProquifaDotNet`.

### B2 — CQRS: fccFolioPagoCliente — Query pendientes + Command cerrar

**Capa:** `ProquifaDotNet.Finanzas.Application`

**Query — `GetPendingPaymentFoliosQuery`:**
Retorna los pendientes del Buzón activos por cliente. Consumido por el listado principal
(Parte C, sección C1) para calcular `PendingPaymentsReceived`.

```
Filtro: fccFolioPagoCliente.Activo = true
        JOIN CorreoRecibidoCliente WHERE IdCliente = @idCliente
Retorna: List<PendingPaymentFolioDto> (Id, Folio, TotalMailBot, FechaRecepcion)
```

**Command — `ClosePaymentFolioCommand`:**
Marca `Activo=0` en el pendiente del Buzón al confirmar el cobro. Se invoca desde
el servicio de confirmación de RE-FU-024.

```
Input: IdFCCFolioPagoCliente
Operación: UPDATE fccFolioPagoCliente SET Activo=0, FechaUltimaActualizacion=UtcNow
           WHERE IdFCCFolioPagoCliente = @Id
```

**Endpoints expuestos:**

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| `GET` | `/api/v1/validate-collection/mailbox/{idCliente}/pending` | Lista pendientes activos del cliente |
| `PUT` | `/api/v1/validate-collection/mailbox/{id}/close` | Cierra pendiente (`Activo=0`) |

### B3 — CQRS: fccPagoCliente — Query cobro + Command upsert

**Capa:** `ProquifaDotNet.Finanzas.Application`

**Query — `GetClientPaymentQuery`:**
Obtiene un cobro por Id o lista cobros activos por cliente.

```
Por Id:      fccPagoCliente WHERE IdFCCPagoCliente = @id AND Activo = true
Por cliente: fccPagoCliente WHERE IdCliente = @idCliente AND Activo = true
Retorna: ClientPaymentDto (todos los campos base + campos de RE-FU-023: Confirmado,
         FechaConfirmacion, IdUsuarioConfirmacion, Notas, IdCatMoneda)
```

**Command — `UpsertClientPaymentCommand`:**
INSERT si `IdFCCPagoCliente` es vacío / UPDATE si ya existe. Maneja el borrador del
Paso 1. El borrador se identifica por `Confirmado = 0`.

```
Input: ClientPaymentDto
INSERT: nuevo registro con Activo=1, Confirmado=0 (borrador)
UPDATE: actualiza campos editables del formulario Paso 1 (Monto, FechaPago, TipoDeCambio,
        IdCatMedioDePago, IdDatosBancarios, MXN, USD, CuentaOrdenante, ReferenciaBancaria)
```

> La lógica de confirmación del cobro (generar Folio, poblar `Confirmado=1`, `FechaConfirmacion`,
> `IdUsuarioConfirmacion`) corresponde a RE-FU-024 y extiende este comando o crea
> un `ConfirmClientPaymentCommand`.

**Endpoints expuestos:**

| Método | Ruta                                       | Descripción                       |
| ------ | ------------------------------------------ | --------------------------------- |
| `GET`  | `/api/v1/validate-collection/payment/{id}`           | Obtener cobro por Id              |
| `GET`  | `/api/v1/validate-collection/payment?idCliente={id}` | Lista cobros activos del cliente  |
| `PUT`  | `/api/v1/validate-collection/payment`                | INSERT borrador / UPDATE borrador |

### B4 — CQRS: tpProformaPedido — Query listado + Commands fecha y cancelación

**Capa:** `ProquifaDotNet.Finanzas.Application`

**Query — `GetClientQuoteQuery`:**
Lista pedidos pendientes de un cliente para el modal Gestionar Cobranza.

```
Filtro: tpProformaPedido WHERE IdCliente = @idCliente AND MontoPendiente > 0 AND Cancelada = 0
JOIN:   tpPedido (PedidoInterno, ReferenciaPedidoCliente)
JOIN:   ContactoCliente (Nombre, Correo, Telefono)
Retorna: QuoteDto (IdTpProformaPedido, IdTpPedido, PedidoInterno,
         ReferenciaPedidoCliente, MontoPendiente, FechaPromesaPagoMonitoreoCobros,
         ContactoNombre, ContactoCorreo, ContactoTelefono)
Cabecera: MontoTotalPendiente = SUM(MontoPendiente)
```

**Command — `UpdatePromiseDateCommand`:**
Actualiza la fecha estimada de pago de uno o más pedidos desde el modal Gestionar Cobranza.

```
Input: List<(IdTpProformaPedido, FechaPromesaPagoMonitoreoCobros, IdUsuarioCambio, Motivo?)>
Operación (por cada ítem):
  1. Leer FechaPromesaPagoMonitoreoCobros actual de tpProformaPedido
  2. UPDATE tpProformaPedido SET FechaPromesaPagoMonitoreoCobros = @fechaNueva
           WHERE IdTpProformaPedido = @id
  3. INSERT fccFechaEstimadaPagoHistorial (IdTpProformaPedido, FechaEstimadaPagoAnterior,
           FechaEstimadaPagaNueva, FechaCambio=GETDATE(), IdUsuarioCambio, Motivo)
```

> **OBS-044 (10/07):** El histórico de modificaciones de FechaEstimadaPago está **dentro del alcance R16** — confirmado por el cliente (Riesgo 1 eliminado al registrarse el histórico en BD). El comando NO sobreescribe silenciosamente: cada cambio queda registrado con valor anterior, valor nuevo, usuario y timestamp.
>
> ⚠️ **Sub-duda VIVA (solo presentación / desempeño):**
> - ¿El histórico requiere **visualización en pantalla** o basta con el **registro en BD** (bitácora general del sistema)?
> - ¿Se conserva **bitácora completa** (append-only en `fccFechaEstimadaPagoHistorial`) o **solo el último cambio** (2 campos en `tpProformaPedido`: FechaEstimadaAnterior + IdUsuarioCambio/FechaCambio) por desempeño?
>
> **Propuesta del equipo:** solo el último cambio por desempeño (misma naturaleza que el historial del Código Validador FU-006 / OBS-014, donde solo se conservan "actual" + "inmediatamente anterior"). El diseño actual (`fccFechaEstimadaPagoHistorial` append-only, ver A3 en _BD.md) queda condicionado a la resolución de esta sub-duda.

**Endpoints expuestos:**

| Método | Ruta                                          | Descripción                                  |
| ------ | --------------------------------------------- | -------------------------------------------- |
| `GET`  | `/api/v1/validate-collection/quote?idCliente={id}` | Lista pedidos pendientes del cliente (modal) |
| `PUT`  | `/api/v1/validate-collection/quote/promiseDate`  | Actualiza fecha estimada de pago             |

---

## Parte C — ProquifaDotNet.Finanzas: Pantalla Principal

### C1 — Listado de clientes con acción contextual

**Patrón:** `QueryInfo` (mismo que Punchout) — `[HttpPost]` recibe `[FromBody] QueryInfo`. Retorna `QueryResultDto<PaymentValidationClientDto>` con `TotalResults` + `Results`.

**Tablas (vía Scaffold Finanzas):** `ClienteCartera`, `ClienteCarteraCliente`, `Cliente`,
`DatosFacturacionCliente`, `tpProformaPedido`, `fccFolioPagoCliente`

**Endpoint:**
```
POST /api/v1/validate-collection/client/search
Body: QueryInfo { SortField, SortDirection, Filters, PageSize, DesiredPage }
Response: QueryResultDto<PaymentValidationClientDto>
```

**Filtros soportados vía `QueryInfo.Filters`:**

| `FieldName` | Tipo | Descripción |
| ----------- | ---- | ----------- |
| `textoBusqueda` | string | Nombre o RFC/RUC del cliente (LIKE) — trim automático aplicado al texto ingresado antes de ejecutar el filtrado (OBS-041) |

> **Nota:** No hay filtros adicionales por región u otros criterios (Criterio B2 del requisito).
> La región del usuario se resuelve del token de autenticación (IdentityServer), no como filtro explícito.
> La cartera del usuario (campo Cobrador en Catálogo de Clientes) también se resuelve del contexto del usuario autenticado.

**Campos calculados en `PaymentValidationClientDto`:**

| Campo calculado | Cálculo |
| --------------- | ------- |
| `PendingPaymentsReceived` | COUNT `fccFolioPagoCliente.Activo=1` por cliente (Scaffold Finanzas) |
| `PendingQuoteInvoices` | COUNT `tpProformaPedido` con `MontoPendiente > 0` y `Cancelada=0` (Scaffold Finanzas) |
| `TotalPendingBalance` | SUM por cliente de `tpProformaPedido.MontoPendiente` **convertido a USD documento por documento**, usando el **tipo de cambio del propio documento origen** (proforma / factura) — no el TC del día de consulta ni un TC unificado del listado. Los montos ya dolarizados se suman. Reutiliza `ConversorDivisas` existente (`FacturasPendientesClienteObj`). Decisión confirmada OBS-046 — 10/07 |
| `ContextualAction` | `PROCESS_PAYMENTS` si `PendingPaymentsReceived > 0`; `MANAGE_COLLECTIONS` si no |
| `OldestPendingPayment` | MIN `fccFolioPagoCliente.FechaRecepcion` donde `Activo=1` por cliente — para ordenamiento por antigüedad (OBS-047) |
| `HasSlaExpired` | `true` si `OldestPendingPayment` lleva más de 72 horas sin procesar; se muestra indicador visual de alerta en UI (OBS-047) |

**Componentes capa Finanzas (patrón Punchout):**
- `Domain.Interfaces` → `IPaymentValidationClientQueryRepository` con `GetFilteredAsync(QueryInfo)`
- `Application.Interfaces` → `IPaymentValidationClientService`
- `Application.Services` → `PaymentValidationClientService`
- `Application.DTOs` → `PaymentValidationClientDto` + `QueryResultDto<PaymentValidationClientDto>`
- `Infrastructure.Repository` → `PaymentValidationClientRepository` — usa `QueryableExtensions` (`ToPagedList`, `ApplyFilter`)
- `API.Controllers` → `[HttpPost]` `/api/v1/validate-collection/client/search` con `[FromBody] QueryInfo`

### C2 — Listado de pedidos para Modal Gestionar Cobranza

**Capa:** `ProquifaDotNet.Finanzas` — vía Scaffold `tpProformaPedido` (B4).

Filtra `tpProformaPedido` del cliente con `MontoPendiente > 0` y `Cancelada=0` directamente
desde Finanzas via Scaffold EF Core. La cabecera del modal incluye `MontoTotalPendiente` (SUM).

**Endpoint:** `GET /api/v1/validate-collection/quote?idCliente={id}` (definido en B4)

### C3 — Actualizar fecha estimada de pago (OBS-044)

**Capa:** `ProquifaDotNet.Finanzas` — vía Scaffold `tpProformaPedido` (B4).

Actualiza `tpProformaPedido.FechaPromesaPagoMonitoreoCobros` directamente desde Finanzas
via Scaffold EF Core mediante `UpdatePromiseDateCommand`.

> **OBS-044:** El `UpdatePromiseDateCommand` guarda el **historial completo** de cambios en `fccFechaEstimadaPagoHistorial` (tabla nueva, ver A3 en _BD.md). Cada cambio genera un INSERT con el valor anterior y el nuevo — no se sobreescribe el registro. El endpoint debe exponer `IdUsuarioCambio` y opcionalmente `Motivo`.

**Endpoint:** `PUT /api/v1/validate-collection/quote/promiseDate` (definido en B4)

### C4 — Cancelar pedido por falta de pago + cancelación CFDI (OBS-042)

**Capa:** `ProquifaDotNet` — endpoint dedicado en `tpPedidoController`.

Requiere un endpoint nuevo en ProquifaDotNet para ejecutar la transacción de cancelación,
ya que `tpPedido` no está en el Scaffold de Finanzas y contiene los campos de trazabilidad
`FechaCancelacionPorFaltaPago`, `IdUsuarioCancelacion`, `FechaSolicitudCancelacion` y
`EstadoCancelacionCFDI` (ALTER A2, pendiente de ejecutar).

**Nuevo endpoint ProquifaDotNet:**

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| `PUT` | `/api/pedidos/{idTpPedido}/cancelar-falta-pago` | Cancela pedido por falta de pago |

**Transacción en ProquifaDotNet:**
```sql
UPDATE tpProformaPedido SET Cancelada = 1 WHERE IdTpPedido = @idTpPedido
UPDATE tpPedido SET FechaCancelacionPorFaltaPago = GETDATE(),
                    IdUsuarioCancelacion = @idUsuario
    WHERE IdTpPedido = @idTpPedido
```

> **OBS-042:** Adicionalmente, si el pedido tiene un CFDI asociado (`CFDIGenerada` con estado `Timbrada`), la cancelación del pedido debe:
> 1. Cambiar el estado de `CFDIGenerada` a `CancelacionSolicitada` (o el estado que corresponda).
> 2. Actualizar `tpPedido.FechaSolicitudCancelacion = GETDATE()` y `tpPedido.EstadoCancelacionCFDI = 'Pendiente'`.
> 3. Enviar la solicitud de cancelación al SAT vía PAC (ProquifaDotNet.Timbrado).
> 4. Actualizar `tpPedido.EstadoCancelacionCFDI` con el resultado del SAT (`'Cancelado'`, `'Rechazado'`, etc.).
>
> El flujo de cancelación CFDI puede ser asíncrono (CFDI 4.0 requiere aceptación del receptor). Considerar estados intermedios.

> ⚠️ Pendiente confirmar si la cancelación dispara `spActualizarBuzonPagoLegacyLegacy`
> u otras transferencias a Legacy.

---

## Brechas

> ✅ **BRECHA — Moneda del Saldo Pendiente total — Resuelta OBS-046 (criterio TC cerrado 10/07)**
> Decisión OBS-046: el listado siempre se muestra dolarizado en USD. **Criterio de tipo de cambio:** cada documento se convierte a USD con el **TC de su propio documento origen** (proforma/factura), NO con el TC del día de consulta ni un TC unificado del listado. Los montos ya dolarizados se suman. Reutilizar `ConversorDivisas` existente en ProquifaDotNet (`FacturasPendientesClienteObj`).
>
> ⚠️ **Pendiente operativo NO bloqueante:** documentar los tipos de cambio del proceso (proforma, cobro, factura) y su trazabilidad/origen por documento.

> ⚠️ **BRECHA — spActualizarBuzonPagoLegacyLegacy al cancelar pedido**
> Confirmar si la cancelación desde Validar Cobro dispara el SP de sincronización Legacy.

> ⚠️ **BRECHA — CorreoFCCPagoCliente.xslt**
> `Logic.MailXslt/Cobranza/CorreoFCCPagoCliente.cs` ya existe. Confirmar si aplica a la
> confirmación del cobro (RE-FU-024) o a otro flujo.

> ✅ **BRECHA — Orden por defecto del listado — Resuelta OBS-047**
> Decisión OBS-047: el listado se ordena por antigüedad del cobro recibido más antiguo del cliente (MIN `fccFolioPagoCliente.FechaRecepcion`, ASC). Clientes sin cobros pendientes se ubican al final. Indicador SLA 72h cuando el cobro más antiguo lleva más de 72 horas sin procesar.

> ⚠️ **BRECHA — Moneda del Saldo Pendiente Perú**
> Para clientes Región Perú, confirmar denominación monetaria del `TotalPendingBalance`
> (MXN, USD, PEN). El ConversorDivisas existe en ProquifaDotNet.

---

**Gen