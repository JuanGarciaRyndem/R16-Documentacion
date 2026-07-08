# Tareas Back — R16A-RE-FU-015
**Requisito:** Tramitación de pedidos Prepago con Factura por Adelantado
**Total de tareas:** 10 (adopta DIS-SOL v1.0 — reemplaza tpProformaAdelanto por fccFactura; T3 obsoleta; T9 2026-06-16 — Validación VD; T10 2026-06-16 — Escenarios E2E flujo completo)

---

## T1 — [ R16A-RE-FU-015 ] [CREATE-TABL-M] Crear catálogo catFacturaEstado y tablas fccFactura, fccFacturaPartida y fccFacturaReferenciaBancaria

### Aplicativos
- ProquifaDotNet.Finanzas

### Módulos
- Finanzas.Infrastructure (Scaffold EF Core)

### Consideraciones previas
- Decisión de diseño del DIS-SOL v1.0: el pendiente FAA **no reutiliza `tpProformaAdelanto`**. Se modela con tres tablas nuevas propiedad de `ProquifaDotNet.Finanzas`.
- `fccFactura` es tabla **única** para la FAA y la factura final, diferenciadas por `EsFacturaPorAdelantado` (RT-10). Los campos fiscales del timbrado SAT van `NULL` en la FAA.
- Ver diccionario de datos completo (columnas, relaciones, índices, consideraciones especiales) en `R16A-RE-FU-015_BD.md`.
- ⚠️ H-01 (`R16A-RE-FU-015_DIS-SOL_Revision.md`): `fccFactura` no modela aún columnas para Tipo de Operación / Condición de Pago SUNAT (Perú) — pendiente de resolver antes de cerrar desarrollo para Perú, no bloquea esta tarea.
- Esta tarea es predecesora de T2 (INSERT del pendiente FAA) — la estructura debe existir antes de implementar la lógica de servicio.

### Objetivo general
Crear en la base de datos `ProquifaDotNet` las tres tablas nuevas requeridas para el pendiente de Factura por Adelantado, e incorporarlas al Scaffold EF Core de `Finanzas.Infrastructure`.

### Objetivos específicos
- `CREATE TABLE catFacturaEstado` + seed (7 estados: POR_GENERAR, ERROR_TIMBRADO, GENERADA, ENVIADA, PAGADA_PARCIAL, PAGADA, CANCELADA — ver `R16A-RE-FU-015_BD.md` v2.1)
- `CREATE TABLE fccFactura` (cabecera) con PK `IdFccFactura`, FK `IdTPPedido` → `tpPedido`, FK `IdCatFacturaEstado` → `catFacturaEstado`, bandera `EsFacturaPorAdelantado`, datos de moneda/montos, snapshot de datos del receptor y campos fiscales del timbrado SAT (nullable)
- `CREATE TABLE fccFacturaPartida` (detalle 1:N) con PK `IdFccFacturaPartida`, FK `IdFccFactura` → `fccFactura`
- `CREATE TABLE fccFacturaReferenciaBancaria` (detalle 1:N) con PK `IdFccFacturaReferenciaBancaria`, FK `IdFccFactura` → `fccFactura`
- Crear índices no clusterizados sobre las FK (`IdTPPedido`, `IdFccFactura`) y sobre `FolioPedidoInterno`
- Agregar las tres tablas al Scaffold EF Core en `Finanzas.Infrastructure`

### Resultado esperado
Las tres tablas existen en `ProquifaDotNet` con la estructura definida en el DIS-SOL v1.0 y son accesibles desde `Finanzas.Infrastructure` vía EF Core.

### Entregables
- Script DDL `CREATE TABLE catFacturaEstado` + seed
- Script DDL `CREATE TABLE fccFactura`
- Script DDL `CREATE TABLE fccFacturaPartida`
- Script DDL `CREATE TABLE fccFacturaReferenciaBancaria`
- Actualización del Scaffold EF Core en `Finanzas.Infrastructure`

### Criterios de aceptación
- Las tres tablas existen con las columnas, tipos y FK documentados en `R16A-RE-FU-015_BD.md`
- Los índices sobre FK y `FolioPedidoInterno` existen
- `fccFactura` permite `EsFacturaPorAdelantado = 1` con todos los campos fiscales del timbrado en `NULL`
- `catFacturaEstado` queda poblado con los 7 estados y `fccFactura.IdCatFacturaEstado` se asigna a POR_GENERAR al crear el registro
- Las tablas son visibles y operables desde el Scaffold EF Core de `Finanzas.Infrastructure`

### Más información de la tarea
- Diccionario de datos completo: `R16A-RE-FU-015_BD.md`
- `ClaveProductoServicioSAT` en `fccFacturaPartida` reutiliza el catálogo `catClaveProdServSAT` poblado en RE-FU-021

### Recursos
- `[R16A-RE-FU-015][DIS-SOL] Diseño de la solución.pdf` — Diseño de Modelo de Datos
- `R16A-RE-FU-015_BD.md`

---

## T2 — [ R16A-RE-FU-015 ] [SERV-TRANSACT] Generación de pendiente FAA al tramitar con Factura por Adelantado

### Aplicativos
- ProquifaDotNet
- ProquifaDotNet.Finanzas

### Módulos
- L05.TramitarPedido (endpoint orquestador, nuevo)
- ProquifaDotNet.Finanzas — servicio de creación del pendiente FAA (nuevo)

### Consideraciones previas
- Este flujo no genera proforma, PDF ni correo (RT-01). El pendiente se genera directamente al confirmar la acción de tramitar, sin proforma previa
- Arquitectura orquestador → Finanzas (ya validada en RE-013/014):
  - `ProquifaDotNet` — `POST /v1/api/invoices/advance-invoice/{orderId}`: valida, fija datos de facturación, genera `FolioPedidoInterno` y lo asigna a `tpPedido` (commit)
  - `ProquifaDotNet.Finanzas` — `POST /v1/api/invoices/advance-invoice/create/{orderId}` (interno): lee `FolioPedidoInterno`, INSERT atómico en `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`
- Predecesora: T1 (estructura de las tres tablas debe existir)
- Idempotencia (RT-03): si `FolioPedidoInterno` ya existe en `tpPedido` al reintentar, se reutiliza sin regenerar
- Atomicidad (RT-04): las tres inserciones (cabecera + partidas + referencias bancarias) son atómicas
- Como no se genera proforma, tampoco existe `MontoPendiente` que pudiera disparar un pendiente en Validar Cobro (RT-06) — no hay lógica de supresión que desarrollar (ver criterio de aceptación abajo, antes cubierto por T3, ahora obsoleta)
- `tpPedidoTramitarTransaccionBO.cs` **no se utiliza** en este flujo

### Objetivo general
Implementar el endpoint orquestador en `ProquifaDotNet` y el servicio de creación del pendiente en `ProquifaDotNet.Finanzas` para generar el pendiente FAA al tramitar un pedido Prepago con FAA activada.

### Objetivos específicos
- `ProquifaDotNet`: implementar `POST /v1/api/invoices/advance-invoice/{orderId}` — valida `SinCredito=1`, `TieneControlados()=false`, `FAA=1`, `Remisión=0`; fija datos de facturación; genera y commitea `FolioPedidoInterno` en `tpPedido`; si el guardado falla, rollback y no se llama a Finanzas; si el folio ya existe (reintento), se reutiliza
- `ProquifaDotNet`: llamar `POST /v1/api/invoices/advance-invoice/create/{orderId}` en Finanzas tras el commit del folio; si esta llamada falla, no hacer rollback (el reintento es seguro)
- `ProquifaDotNet.Finanzas`: implementar el endpoint interno — lee `FolioPedidoInterno` desde `tpPedido`, INSERT atómico de `fccFactura` (`EsFacturaPorAdelantado=1`, campos fiscales timbrado `NULL`), `fccFacturaPartida` (una por partida) y `fccFacturaReferenciaBancaria` (cuentas M.N./DLS + `ReferenciaCliente` por Código Validador, RE-006)
- Confirmar que no se genera ningún registro de proforma (`tpProformaPedido`) ni pendiente en Validar Cobro como efecto colateral

### Resultado esperado
Al tramitar un pedido Prepago con FAA=1, se genera automáticamente el pendiente FAA (`fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`) con todos los datos necesarios para la emisión posterior de la factura, sin generar proforma, PDF, correo ni pendiente en Validar Cobro.

### Entregables
- Nuevo endpoint `POST /v1/api/invoices/advance-invoice/{orderId}` en `ProquifaDotNet`
- Nuevo endpoint `POST /v1/api/invoices/advance-invoice/create/{orderId}` en `ProquifaDotNet.Finanzas`
- Lógica de INSERT atómico en `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`

### Criterios de aceptación
- Pendiente FAA se genera directamente al tramitar (sin proforma previa)
- Contiene todos los datos requeridos (cliente, empresa, monto, partidas, datos facturación, referencias bancarias)
- Las tres inserciones son atómicas (rollback conjunto si falla alguna)
- No se genera si FAA=0
- Campos fiscales del timbrado quedan `NULL` en `fccFactura` (pendiente emisión)
- El reintento tras fallo de Finanzas no regenera el `FolioPedidoInterno` ni duplica registros
- No se genera ningún registro de proforma (`tpProformaPedido`) ni pendiente en Validar Cobro al tramitar (criterio heredado de T3, ahora obsoleta)

---

## ~~T3 — [ R16A-RE-FU-015 ] [ALG-COMPLX-LOGIC] No generar pendiente Validar Cobro cuando FAA está activa~~ (OBSOLETA)

> ⚠️ **Obsoleta.** El requisito no genera proforma en este flujo, por lo que nunca existe un `tpProformaPedido.MontoPendiente` que pudiera disparar un pendiente en Validar Cobro (RT-06) — no hay nada que suprimir. La verificación de que no se genera pendiente VC queda como criterio de aceptación dentro de T2. Se conserva esta entrada solo para trazabilidad; no requiere desarrollo independiente.

---

## T4 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Eliminar código de autorización para Factura por Adelantado

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido

### Consideraciones previas
- RT-07 / Regla 3: la activación de FAA es directa, sin código de autorización
- Anteriormente se requería código para activar FAA
- Buscar y eliminar la validación si existe en el código actual

### Objetivo general
Eliminar la validación de código de autorización para la activación de Factura por Adelantado.

### Objetivos específicos
- Buscar en el flujo orquestador FAA (T2) y archivos relacionados la validación de código de autorización
- Eliminar o deshabilitar la validación
- Confirmar que la activación es directa (Front envía `FacturaPorAdelantado=1` sin campo de código)

### Resultado esperado
La activación de FAA no requiere código de autorización.

### Entregables
- Eliminación de validación de código de autorización

### Criterios de aceptación
- No se solicita código al activar FAA
- La tramitación procede con FAA=1 sin validación adicional
- No se afectan otros flujos que pudieran usar códigos de autorización

---

## T5 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Bloquear datos de facturación al activar FAA

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido
- WebApi.Logistica

### Consideraciones previas
- Regla 4: al activar FAA se fijan los datos de facturación del catálogo vigente del cliente
- Los datos fijados se persisten como snapshot en `fccFactura`: RFC, Razón Social, CP, Régimen Fiscal, Uso CFDI, Método de Pago, Forma de Pago
- Rechazar edición posterior si FAA=1

### Objetivo general
Implementar el bloqueo y fijación de datos de facturación al activar Factura por Adelantado.

### Objetivos específicos
- Al tramitar con FAA=1: tomar datos de `DatosFacturacionCliente` vigente y fijarlos como snapshot en `fccFactura` (T2)
- Agregar validación en endpoint de edición: si `FacturaPorAdelantado=1` → rechazar con error
- Los datos fijados quedan inmutables desde Tramitar Pedido

### Resultado esperado
Los datos de facturación se fijan al activar FAA y no pueden editarse posteriormente desde Tramitar Pedido.

### Entregables
- Lógica de fijación de datos al tramitar
- Validación en endpoint de edición

### Criterios de aceptación
- Datos se fijan del catálogo vigente al tramitar con FAA=1
- Error claro si se intenta editar datos con FAA activa
- Los datos fijados son los correctos (RFC, Razón Social, Uso CFDI, etc.)

---

## T6 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Vinculación del pendiente FAA con módulo de facturación (RE-FU-018/019/020)

### Aplicativos
- ProquifaDotNet.Finanzas

### Módulos
- Módulo Factura por Adelantado (RE-FU-018/019/020)

### Consideraciones previas
- La emisión de factura, CFDI y timbrado se desarrollan en RE-FU-018, RE-FU-019 y RE-FU-020
- Esta tarea asegura que el pendiente generado en T2 (`fccFactura` + detalle) sea consumible por esos módulos
- RT-10: `fccFactura` es tabla única — el módulo FAA debe hacer UPDATE del mismo registro (`EsFacturaPorAdelantado: 1 → 0` y llenar campos fiscales del timbrado), no crear un registro nuevo
- A diferencia de RE-FU-012/013, en este flujo el pendiente se puebla directamente desde pedido/partidas/cliente, sin un objeto de proforma intermedio — confirmar que esto no genera diferencias de datos frente a lo que espera RE-FU-018

### Objetivo general
Vincular el proceso de generación del pendiente FAA con el flujo de facturación desarrollado en RE-FU-018/019/020.

### Objetivos específicos
- Verificar que la estructura de `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria` es compatible con lo esperado por RE-FU-018
- Confirmar el contrato de UPDATE que usará RE-FU-018/019/020 sobre `fccFactura` (RT-10) al timbrar la factura final
- Validar que el estado del pendiente se actualice correctamente cuando facturación lo consume
- Documentar contrato de datos entre pendiente y módulo de facturación

### Resultado esperado
El pendiente FAA generado en la tramitación es consumido correctamente por el módulo de facturación sin inconsistencias.

### Entregables
- Ajustes de integración (si aplican)
- Documentación del contrato de datos (INSERT en T2 → UPDATE en RE-018/019/020)

### Criterios de aceptación
- El módulo FAA (RE-FU-018) puede leer y procesar el pendiente
- El estado del pendiente se actualiza (UPDATE de `fccFactura`, no INSERT duplicado) al ser procesado
- No hay datos faltantes ni inconsistencias

---

## T7 — [ R16A-RE-FU-015 ] [UPDATE-TABL-CH] ⛔ BLOQUEANTE — Crear catálogo CatEstadoTpPedido e IdCatEstadoTpPedido en tpPedido (OBS-027)

> **⛔ BLOQUEANTE — OBS-027**
>
> Esta tarea está **BLOQUEADA** hasta recibir la propuesta del cliente sobre el catálogo `CatEstadoTpPedido`.
>
> **Pendiente del cliente:**
> - Definición de estados requeridos para `CatEstadoTpPedido` (clave, descripción, estados terminales, transiciones permitidas).
> - Confirmación de si `IdCatEstadoTpPedido` aplica en `tpPedido`, `ppPedido`, o ambas.
>
> **No iniciar desarrollo hasta que se resuelva el BLOQUEANTE.**
>
> Una vez desbloqueada, esta tarea incluirá:
> - `CREATE TABLE dbo.CatEstadoTpPedido` con los campos definidos por el cliente.
> - `ALTER TABLE dbo.tpPedido ADD IdCatEstadoTpPedido uniqueidentifier NULL FK → CatEstadoTpPedido`.
> - Scripts DDL + DML (INSERT de catálogo) con los estados acordados.

---

## T8 — [ R16A-RE-FU-015 ] [ALG-COMPLX-LOGIC] ⛔ BLOQUEANTE — Lógica de transición de estados del pedido (OBS-027)

> **⛔ BLOQUEANTE — OBS-027**
>
> Esta tarea está **BLOQUEADA** hasta que T7 (`CatEstadoTpPedido` + `IdCatEstadoTpPedido`) esté resuelta y desbloqueada.
>
> **Depende de:** T7 (`CatEstadoTpPedido` definido por el cliente).
>
> **No iniciar desarrollo hasta que se resuelva el BLOQUEANTE.**
>
> Una vez desbloqueada, esta tarea incluirá:
> - Lógica de asignación de `IdCatEstadoTpPedido` en `ProquifaDotNet.Finanzas` al cerrar el pendiente Tramitar Pedido (paso 3i del flujo, RT-05).
> - Validación de transiciones permitidas según el catálogo acordado.
> - Actualización de `tpPedido.IdCatEstadoTpPedido` en los puntos del flujo definidos.

---

## T9 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Validar flujo Venta Digital al tramitar pedido Prepago con FAA

### Aplicativos
- ProquifaDotNet
- Venta Digital

### Módulos
- L05.TramitarPedido
- Extracto Venta Digital (`tpPedidoVD`, `tpPartidaPedidoVD`)

### Consideraciones previas
- El flujo de tramitación base (RE-FU-010) incluye como paso 12 el "Extracto Venta Digital": INSERT/UPDATE en `tpPedidoVD` y `tpPartidaPedidoVD`.
- RE-015 extiende ese flujo con la variante FAA (Prepago); se debe verificar que el paso de Venta Digital sigue ejecutándose correctamente independientemente de si FAA está activa o no.
- **TaskScheduler en Venta Digital:** existe un job programado (Windows Task Scheduler) que depende de los datos escritos en `tpPedidoVD`/`tpPartidaPedidoVD`. Ese job realiza dos acciones:
  1. **Procesamiento de Órdenes de Compra** — lee el extracto y genera/actualiza la Orden de Compra en Venta Digital.
  2. **Transferencia de PDFs a Legacy** — toma los PDFs asociados al pedido y los transfiere al directorio configurado de Legacy.
- Si el paso de Extracto Venta Digital se omite o genera datos incorrectos en el flujo FAA Prepago, el job de TaskScheduler falla silenciosamente, dejando la OC sin procesar y los PDFs sin transferir a Legacy.
- Este flujo no genera proforma, PDF ni correo en ningún punto de Tramitar Pedido. La pregunta abierta deja de ser "cuál PDF está disponible" y pasa a ser **si existe algún documento en absoluto para transferir**, o si TaskScheduler debe operar sin ninguno para este flujo.

### Objetivo general
Verificar que el paso de Extracto Venta Digital (INSERT/UPDATE en `tpPedidoVD`/`tpPartidaPedidoVD`) se ejecuta correctamente en el flujo de tramitación Prepago con FAA, garantizando que el job de TaskScheduler de Venta Digital pueda procesar la Orden de Compra y transferir los PDFs a Legacy sin errores.

### Objetivos específicos
- Rastrear en el flujo orquestador FAA (T2) dónde se invoca el extracto Venta Digital y confirmar que la condición FAA=1 en Prepago no lo omite ni lo cortocircuita.
- Verificar que los campos requeridos por Venta Digital (`tpPedidoVD.OrdenDeCompra`, `tpPedidoVD.IdPedido`, `tpPartidaPedidoVD.*`) se populan correctamente cuando FAA=1 en Prepago.
- Confirmar que el job de TaskScheduler puede leer y procesar el extracto generado (validar estructura de datos esperada vs. la que genera el flujo FAA Prepago).
- Clarificar si existe algún documento disponible para transferir a Legacy en el flujo Prepago con FAA: en RE-015 no se genera proforma, PDF ni correo, y el pendiente Validar Cobro tampoco se genera al tramitar (criterio heredado en T2) — confirmar si TaskScheduler debe operar sin documento en este caso, o si existe algún otro artefacto que deba transferir.
- Documentar cualquier ajuste necesario si se detecta que el flujo FAA Prepago altera el extracto o deja a TaskScheduler sin documento que transferir.

### Resultado esperado
El job de TaskScheduler de Venta Digital procesa correctamente los pedidos tramitados con FAA Prepago: la Orden de Compra queda registrada en Venta Digital y los PDFs se transfieren a Legacy sin diferencias respecto al flujo base.

### Entregables
- Reporte de validación: flujo FAA Prepago vs. flujo base en los datos escritos a `tpPedidoVD`/`tpPartidaPedidoVD`
- Aclaración de si existe algún documento disponible para transferencia a Legacy en el flujo Prepago con FAA (o si TaskScheduler debe operar sin ninguno)
- Ajustes en el endpoint orquestador FAA (T2) si se detecta alguna omisión (entregable condicional)
- Evidencia de que el job de TaskScheduler procesa correctamente el extracto FAA Prepago

### Criterios de aceptación
- [ ] El extracto Venta Digital se genera en `tpPedidoVD`/`tpPartidaPedidoVD` al tramitar Prepago con FAA=1 (igual que con FAA=0).
- [ ] Los campos requeridos por el job de TaskScheduler están presentes y correctos.
- [ ] El job de TaskScheduler procesa la Orden de Compra sin errores cuando el pedido tiene FAA=1 (Prepago).
- [ ] Está aclarado si TaskScheduler transfiere algún documento a Legacy en el flujo FAA Prepago, o si opera sin ninguno (ya no se genera proforma/PDF en este flujo).
- [ ] No se genera diferencia funcional entre FAA=1 y FAA=0 en los datos entregados a Venta Digital (fuera de la ausencia de documento).

### Más información de la tarea
- El flujo base de Extracto Venta Digital se define en RE-FU-010, paso 12 (tablas `tpPedidoVD`, `tpPartidaPedidoVD`).
- El job de TaskScheduler en Venta Digital opera sobre `ppPedidoVD`/`ppPartidaPedidoVD` (pretramitado) y `tpPedidoVD`/`tpPartidaPedidoVD` (tramitado). Tiene dos responsabilidades: procesar la OC y transferir PDFs a Legacy.
- En el flujo Prepago con FAA, T3 (obsoleta) suprime el pendiente Validar Cobro al tramitar. Verificar que esta supresión no afecta la generación del extracto Venta Digital ni el PDF disponible para Legacy.

### Recursos
- `R16A-RE-FU-010-Back.md` — paso 12 (Extracto Venta Digital), tablas `tpPedidoVD`/`tpPartidaPedidoVD`
- `R16A-RE-FU-015-Back.md` — flujo tramitación Prepago con FAA
- `R16A-RE-FU-012-Tareas.md` — T8 (misma validación en variante Crédito, como referencia)

---

## Resumen de tareas

| # | Clave Catálogo | Título | Predecesora |
|---|----------------|--------|-------------|
| T1 | CREATE-TABL-M | Crear tablas fccFactura, fccFacturaPartida y fccFacturaReferenciaBancaria | — |
| T2 | SERV-TRANSACT | Generación de pendiente FAA al tramitar con Factura por Adelantado (incluye criterios de T3) | T1 |
| T3 | ALG-COMPLX-LOGIC | ~~No generar pendiente Validar Cobro cuando FAA está activa~~ (OBSOLETA — ver T2) | — |
| T4 | ALG-BASIC-LOGIC | Eliminar código de autorización para Factura por Adelantado | — |
| T5 | ALG-BASIC-LOGIC | Bloquear datos de facturación al activar FAA | — |
| T6 | ALG-BASIC-LOGIC | Vinculación del pendiente FAA con módulo de facturación (RE-FU-018/019/020) | T2 |
| T7 | UPDATE-TABL-CH | ⛔ BLOQUEANTE — Crear CatEstadoTpPedido + IdCatEstadoTpPedido en tpPedido (OBS-027) | — |
| T8 | ALG-COMPLX-LOGIC | ⛔ BLOQUEANTE — Lógica de transición de estados del pedido (OBS-027) | T7 |
| T9 | ALG-BASIC-LOGIC | Validar flujo Venta Digital al tramitar pedido Prepago con FAA (sin PDF disponible) | T2 |
| T10 | ALG-BASIC-LOGIC | Pruebas de flujo completo E2E — Tramitación Prepago con y sin FAA | T1,T2,T4,T5,T9 |

---

## T10 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Pruebas de flujo completo E2E — Tramitación Prepago con y sin FAA

### Aplicativos
- ProquifaDotNet
- ProquifaDotNet.Finanzas
- Venta Digital
- Legacy

### Módulos
- L05.TramitarPedido
- Extracto Venta Digital
- Validar Cobro

### Consideraciones previas
- Esta tarea valida impactos colaterales: cualquier modificación al flujo de tramitación Prepago debe demostrarse que no rompió el pipeline completo, incluyendo los módulos que dependen de él (Venta Digital, Legacy, Validar Cobro).
- El flujo Prepago tiene particularidades que lo diferencian de Crédito: el pendiente Validar Cobro se genera (o no) según FAA, y no hay transferencia directa a Legacy como en Crédito — confirmar qué sí transfiere TaskScheduler.
- Los escenarios cubren: flujo base Prepago sin FAA (no regresión RE-014), FAA activa, bloqueo de datos de facturación y reglas de negocio específicas de Prepago.
- Predecesoras: T1 (tablas `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria`), T2 (pendiente FAA, incluye verificación de no generación de proforma/VC), T4 (sin código autorización), T5 (bloqueo datos), T9 (validación VD).

### Escenarios de prueba

| # | Escenario | FAA | Pendiente VC esperado |
|---|-----------|:---:|:---------------------:|
| E1 | Prepago base — no regresión RE-014 | ✗ | ✓ Se genera |
| E2 | Prepago con FAA activa | ✓ | ✗ No se genera |
| E3 | Prepago FAA: datos de facturación se fijan y bloquean | ✓ | — |
| E4 | Prepago FAA: intento de editar datos facturación (debe rechazarse) | ✓ | — |
| E5 | Prepago FAA: activación sin código de autorización | ✓ | — |
| E6 | Prepago FAA: vinculación con módulo de facturación RE-018 | ✓ | — |

### Pipeline a verificar por escenario

Para cada escenario aplicable, verificar que cada paso del pipeline se ejecutó correctamente:

| Paso | Qué verificar                                                              | E1  | E2  | E3  | E4  | E5  | E6  |
| ---- | -------------------------------------------------------------------------- | :-: | :-: | :-: | :-: | :-: | :-: |
| 1    | Folio de pedido asignado                                                   |  ✓  |  ✓  |  ✓  |  —  |  ✓  |  ✓  |
| 2    | PDF de proforma generado (RE-016/017) — solo aplica al flujo base RE-014, ya no aplica a E2-E6 |  ✓  |  —  |  —  |  —  |  —  |  —  |
| 3    | Correo de proforma enviado al cliente — solo aplica al flujo base RE-014, ya no aplica a E2-E6 |  ✓  |  —  |  —  |  —  |  —  |  —  |
| 4    | Pendiente Validar Cobro generado (solo FAA=0)                              |  ✓  |  —  |  —  |  —  |  —  |  —  |
| 5    | Pendiente VC NO generado (FAA=1)                                           |  —  |  ✓  |  ✓  |  —  |  ✓  |  ✓  |
| 6    | Extracto Venta Digital: `tpPedidoVD` y `tpPartidaPedidoVD`                 |  ✓  |  ✓  |  ✓  |  —  |  ✓  |  ✓  |
| 7    | TaskScheduler VD: Orden de Compra procesada                                |  ✓  |  ✓  |  ✓  |  —  |  ✓  |  ✓  |
| 8    | TaskScheduler VD: documento transferido a Legacy (E2-E6: ⚠️ punto abierto — ya no hay PDF disponible, ver T9) |  ✓  |  ⚠️  |  —  |  —  |  —  |  —  |
| 9    | Pendiente FAA generado (`fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`, solo FAA=1) |  —  |  ✓  |  ✓  |  —  |  ✓  |  ✓  |
| 10   | Datos de facturación fijados del catálogo vigente (snapshot en `fccFactura`) |  —  |  ✓  |  ✓  |  —  |  —  |  —  |
| 11   | Error al intentar editar datos con FAA activa (E4)                         |  —  |  —  |  —  |  ✓  |  —  |  —  |
| 12   | Activación sin solicitar código de autorización (E5)                       |  —  |  —  |  —  |  —  |  ✓  |  —  |
| 13   | Pendiente FAA consumible por módulo RE-018 (E6) — UPDATE de `fccFactura`, no INSERT duplicado |  —  |  —  |  —  |  —  |  —  |  ✓  |
| 14   | BitácoraCRUD registrada                                                    |  ✓  |  ✓  |  ✓  |  ✓  |  ✓  |  ✓  |

> **Nota (pasos 2, 3, 8):** este flujo ya no genera proforma, PDF ni correo (RT-01). El punto abierto del paso 8 (E2-E6) se traslada a T9: confirmar si TaskScheduler debe operar sin ningún documento para transferir a Legacy en el flujo FAA Prepago.

### Objetivo general
Garantizar que las modificaciones introducidas en RE-015 (estructura `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria`, supresión pendiente VC, pendiente FAA, bloqueo datos, eliminación código autorización) no rompieron ningún paso del pipeline de tramitación Prepago ni los módulos dependientes.

### Resultado esperado
Todos los escenarios ejecutados con evidencia. Los pasos del pipeline marcados como aplicables funcionan correctamente. El flujo base Prepago sin FAA (E1) no presenta regresión respecto al comportamiento de RE-014.

### Entregables
- Tabla de evidencias por escenario (query o captura por paso verificado)
- Confirmación del documento que TaskScheduler transfiere a Legacy en el flujo Prepago FAA (paso 8 — aclaración pendiente)
- Incidencias encontradas y su resolución
- Confirmación de que el pendiente FAA (paso 9) es consumible por RE-018 sin ajustes adicionales

### Criterios de aceptación
- [ ] E1 — Prepago base sin FAA: flujo completo RE-014 sin regresión (pasos 1-3, 4, 6-8, 14).
- [ ] E2 — FAA activa: pendiente VC suprimido (paso 5), pendiente FAA generado (paso 9) en las tres tablas, VD y TaskScheduler correctos.
- [ ] E3 — Datos fijados: los datos de facturación en `fccFactura` corresponden al catálogo vigente del cliente al momento de tramitar.
- [ ] E4 — Bloqueo edición: el endpoint de edición de datos rechaza la operación con error claro cuando FAA=1.
- [ ] E5 — Sin código: el flujo de activación FAA no solicita ni valida código de autorización.
- [ ] E6 — Vinculación RE-018: el módulo de facturación puede leer y procesar el pendiente FAA (UPDATE de `fccFactura`) sin inconsistencias de datos.
- [ ] Paso 8 aclarado: está documentado qué documento transfiere TaskScheduler en el flujo Prepago FAA.
- [ ] PR aprobado por líder técnico con evidencias adjuntas.

### Más información de la tarea
- **Diferencia clave vs. RE-012:** en Prepago con FAA no se genera ni proforma ni confirmación de pedido en Tramitar Pedido — no hay ningún PDF disponible en ese momento. Confirmar si TaskScheduler debe operar sin documento para este flujo, o si debe esperar a un artefacto generado posteriormente por el módulo de facturación (RE-018/019/020).
- El paso 13 (E6) valida la integración hacia adelante: RE-015 pone el pendiente en `fccFactura`, RE-018 lo actualiza (UPDATE, no INSERT). Si el contrato de datos tiene diferencias, ambos requisitos se ven afectados.

### Recursos
- `R16A-RE-FU-014-Back.md` — flujo base Prepago (referencia para no regresión E1)
- `R16A-RE-FU-015-Back.md` — flujo Prepago con FAA
- `R16A-RE-FU-015_BD.md` — diccionario de datos `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria`
- `R16A-RE-FU-018-Back.md` — módulo de facturación que consume el pendiente FAA
- `R16A-RE-FU-012-Tareas.md` — T9 (referencia: misma estructura para variante Crédito)

---

## Dependencias con otros requisitos (NO incluidas como tareas)

| Requisito | Tarea relacionada | Relación |
|-----------|-------------------|----------|
| R16A-RE-FU-006 | T2, T6 | Código Validador → `fccFacturaReferenciaBancaria.ReferenciaCliente` |
| R16A-RE-FU-010 | Cancelación | Endpoint de cancelación se desarrolla en RE-FU-010 |
| R16A-RE-FU-013 | Arquitectura orquestador→Finanzas | Patrón ya validado ahí; única parte del flujo base de RE-013 aún compartida — foliador, previsualización, envío de correo y vinculación PDF ya NO aplican |
| R16A-RE-FU-014 | T1 (RE-014), T2 (RE-014) | Validación Remisión Prepago, datos facturación solo lectura |
| R16A-RE-FU-016 | Criterio E1 | Dos cuentas bancarias del grupo PROQUIFA México, reutilizadas en `fccFacturaReferenciaBancaria` |
| R16A-RE-FU-018/019/020 | Módulo FAA | Consume (UPDATE) el pendiente generado en T2 |
| R16A-RE-FU-021 | `catClaveProdServSAT` | Reutilizado por `fccFacturaPartida.ClaveProductoServicioSAT` |
| Venta Digital | T9 | TaskScheduler lee `tpPedidoVD`/`tpPartidaPedidoVD` para procesar OC y transferir documentos a Legacy — pendiente confirmar comportamiento sin PDF disponible |

> R16A-RE-FU-016/017 (generación de PDF/template de proforma) **ya no aplican** a este requisito — el requisito actualizado no genera proforma en este flujo.
