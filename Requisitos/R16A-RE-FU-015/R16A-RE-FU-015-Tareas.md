# Tareas Back — R16A-RE-FU-015
**Requisito:** Tramitación de pedidos Prepago con Factura por Adelantado
**Total de tareas:** 9 (T2 obsoleta tras actualización de requisito — ver nota; T8 2026-06-16 — Validación VD; T9 2026-06-16 — Escenarios E2E flujo completo)

---

## T1 — [ R16A-RE-FU-015 ] [SERV-TRANSACT] Generación de pendiente FAA al tramitar con Factura por Adelantado

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Liberar
- L05.TramitarPedido\Facturas\Anticipos

### Consideraciones previas
- **Actualización de requisito:** este flujo ya no genera proforma, PDF ni correo (ver Alcance — "No aplica a" en `R16A-RE-FU-015.md`). El pendiente se genera directamente al confirmar la acción de tramitar, sin proforma previa
- INSERT en `tpProformaAdelanto` con datos del pedido/cliente/empresa/monto
- Debe ser atómico con la transacción de tramitación
- La entidad `tpProformaAdelanto` ya existe como CrudBO
- Como no se genera proforma, tampoco existe `MontoPendiente` que pudiera disparar un pendiente en Validar Cobro — no hay lógica de supresión que desarrollar (ver criterio de aceptación abajo, antes cubierto por T2, ahora obsoleta)

### Objetivo general
Implementar la generación del pendiente en el módulo Factura por Adelantado al tramitar un pedido Prepago con FAA activada.

### Objetivos específicos
- Modificar flujo en `tpPedidoTramitarTransaccionBO.cs` para detectar `FacturaPorAdelantado=1`
- INSERT en `tpProformaAdelanto` con: IdCliente, IdEmpresa, Monto, FolioPedido, DatosFacturación, Estado=Pendiente
- Asegurar atomicidad con la transacción de tramitación
- `IdCFDIGenerada = NULL` (pendiente de emisión por módulo FAA)
- Confirmar que no se genera ningún registro de proforma (`tpProformaPedido`) ni pendiente en Validar Cobro como efecto colateral

### Resultado esperado
Al tramitar un pedido Prepago con FAA=1, se genera automáticamente un pendiente en el módulo Factura por Adelantado con todos los datos necesarios para la emisión posterior de la factura, sin generar proforma, PDF, correo ni pendiente en Validar Cobro.

### Entregables
- Modificación de `tpPedidoTramitarTransaccionBO.cs`
- Lógica de INSERT en `tpProformaAdelanto`

### Criterios de aceptación
- Pendiente FAA se genera directamente al tramitar (sin proforma previa)
- Contiene todos los datos requeridos (cliente, empresa, monto, datos facturación)
- Es atómico con la transacción
- No se genera si FAA=0
- IdCFDIGenerada queda NULL (pendiente emisión)
- No se genera ningún registro de proforma (`tpProformaPedido`) ni pendiente en Validar Cobro al tramitar (criterio heredado de T2, ahora obsoleta)

---

## ~~T2 — [ R16A-RE-FU-015 ] [ALG-COMPLX-LOGIC] No generar pendiente Validar Cobro cuando FAA está activa~~ (OBSOLETA)

> ⚠️ **Obsoleta tras actualización del requisito.** El requisito ya no genera proforma en este flujo (ver Alcance — "No aplica a" en `R16A-RE-FU-015.md`), por lo que nunca existe un `tpProformaPedido.MontoPendiente` que pudiera disparar un pendiente en Validar Cobro — no hay nada que suprimir. La verificación de que no se genera pendiente VC queda como criterio de aceptación dentro de T1. Se conserva esta entrada solo para trazabilidad; no requiere desarrollo independiente.

---

## T3 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Eliminar código de autorización para Factura por Adelantado

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- Regla 3: la activación de FAA es directa, sin código de autorización
- Anteriormente se requería código para activar FAA
- Buscar y eliminar la validación si existe en el código actual

### Objetivo general
Eliminar la validación de código de autorización para la activación de Factura por Adelantado.

### Objetivos específicos
- Buscar en `tpPedidoTramitarTransaccionBO.cs` y archivos relacionados la validación de código de autorización para FAA
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

## T4 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Bloquear datos de facturación al activar FAA

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido
- WebApi.Logistica

### Consideraciones previas
- Regla 4: al activar FAA se fijan los datos de facturación del catálogo vigente del cliente
- Los datos fijados son: RFC, Razón Social, Uso CFDI, Método de Pago, Forma de Pago, Correo, Régimen Fiscal
- Rechazar edición posterior si FAA=1

### Objetivo general
Implementar el bloqueo y fijación de datos de facturación al activar Factura por Adelantado.

### Objetivos específicos
- Al tramitar con FAA=1: tomar datos de `DatosFacturacionCliente` vigente y fijarlos en el pedido/pendiente FAA (ya no en una proforma — este flujo no la genera)
- Agregar validación en endpoint de edición: si `FacturaPorAdelantado=1` -> rechazar con error
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

## T5 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Vinculación del pendiente FAA con módulo de facturación (RE-FU-018/019/020)

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Facturas\Anticipos

### Consideraciones previas
- La emisión de factura, CFDI y timbrado se desarrollan en RE-FU-018, RE-FU-019 y RE-FU-020
- Esta tarea asegura que el pendiente generado en T1 sea consumible por esos módulos
- Mismo patrón que RE-FU-012 T4
- A diferencia de RE-FU-012/013, en este flujo el pendiente se puebla directamente desde pedido/partidas/cliente, sin un objeto de proforma intermedio — confirmar que esto no genera diferencias de datos frente a lo que espera RE-FU-018

### Objetivo general
Vincular el proceso de generación del pendiente FAA con el flujo de facturación desarrollado en RE-FU-018/019/020.

### Objetivos específicos
- Verificar que la estructura de `tpProformaAdelanto` es compatible con lo esperado por RE-FU-018
- Ajustar campos/relaciones si es necesario para la integración
- Validar que el estado del pendiente se actualice correctamente cuando facturación lo consume
- Documentar contrato de datos entre pendiente y módulo de facturación

### Resultado esperado
El pendiente FAA generado en la tramitación es consumido correctamente por el módulo de facturación sin inconsistencias.

### Entregables
- Ajustes de integración (si aplican)
- Documentación del contrato de datos

### Criterios de aceptación
- El módulo FAA (RE-FU-018) puede leer y procesar el pendiente
- El estado del pendiente se actualiza al ser procesado
- No hay datos faltantes ni inconsistencias

---

---

## T6 — [ R16A-RE-FU-015 ] [UPDATE-TABL-CH] ⛔ BLOQUEANTE — Crear catálogo catEstatusPedido e IdEstatusPedido en tpPedido (OBS-027)

> **⛔ BLOQUEANTE — OBS-027**
>
> Esta tarea está **BLOQUEADA** hasta recibir la propuesta del cliente sobre el catálogo `catEstatusPedido`.
>
> **Pendiente del cliente:**
> - Definición de estados requeridos para `catEstatusPedido` (clave, descripción, estados terminales, transiciones permitidas).
> - Confirmación de si `IdEstatusPedido` aplica en `tpPedido`, `ppPedido`, o ambas.
>
> **No iniciar desarrollo hasta que se resuelva el BLOQUEANTE.**
>
> Una vez desbloqueada, esta tarea incluirá:
> - `CREATE TABLE dbo.catEstatusPedido` con los campos definidos por el cliente.
> - `ALTER TABLE dbo.tpPedido ADD IdEstatusPedido uniqueidentifier NULL FK → catEstatusPedido`.
> - Scripts DDL + DML (INSERT de catálogo) con los estados acordados.

---

## T7 — [ R16A-RE-FU-015 ] [ALG-COMPLX-LOGIC] ⛔ BLOQUEANTE — Lógica de transición de estados del pedido (OBS-027)

> **⛔ BLOQUEANTE — OBS-027**
>
> Esta tarea está **BLOQUEADA** hasta que T6 (catEstatusPedido + IdEstatusPedido) esté resuelta y desbloqueada.
>
> **Depende de:** T6 (catEstatusPedido definido por el cliente).
>
> **No iniciar desarrollo hasta que se resuelva el BLOQUEANTE.**
>
> Una vez desbloqueada, esta tarea incluirá:
> - Lógica de transición de estados en `tpPedidoTramitarTransaccionBO.cs` (o la clase que aplique) usando `IdEstatusPedido` → `catEstatusPedido`.
> - Validación de transiciones permitidas según el catálogo acordado.
> - Actualización de `tpPedido.IdEstatusPedido` en los puntos del flujo definidos.

---

## T8 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Validar flujo Venta Digital al tramitar pedido Prepago con FAA

### Aplicativos
- ProquifaDotNet
- Venta Digital

### Módulos
- L05.TramitarPedido\Liberar
- Extracto Venta Digital (`tpPedidoVD`, `tpPartidaPedidoVD`)

### Consideraciones previas
- El flujo de tramitación base (RE-FU-010) incluye como paso 12 el "Extracto Venta Digital": INSERT/UPDATE en `tpPedidoVD` y `tpPartidaPedidoVD`.
- RE-015 extiende ese flujo con la variante FAA (Prepago); se debe verificar que el paso de Venta Digital sigue ejecutándose correctamente independientemente de si FAA está activa o no.
- **TaskScheduler en Venta Digital:** existe un job programado (Windows Task Scheduler) que depende de los datos escritos en `tpPedidoVD`/`tpPartidaPedidoVD`. Ese job realiza dos acciones:
  1. **Procesamiento de Órdenes de Compra** — lee el extracto y genera/actualiza la Orden de Compra en Venta Digital.
  2. **Transferencia de PDFs a Legacy** — toma los PDFs asociados al pedido y los transfiere al directorio configurado de Legacy.
- Si el paso de Extracto Venta Digital se omite o genera datos incorrectos en el flujo FAA Prepago, el job de TaskScheduler falla silenciosamente, dejando la OC sin procesar y los PDFs sin transferir a Legacy.
- **Actualización de requisito:** este flujo ya no genera proforma, PDF ni correo en ningún punto de Tramitar Pedido (ver Alcance — "No aplica a" en `R16A-RE-FU-015.md`). La pregunta abierta deja de ser "cuál PDF está disponible" y pasa a ser **si existe algún documento en absoluto para transferir**, o si TaskScheduler debe operar sin ninguno para este flujo.

### Objetivo general
Verificar que el paso de Extracto Venta Digital (INSERT/UPDATE en `tpPedidoVD`/`tpPartidaPedidoVD`) se ejecuta correctamente en el flujo de tramitación Prepago con FAA, garantizando que el job de TaskScheduler de Venta Digital pueda procesar la Orden de Compra y transferir los PDFs a Legacy sin errores.

### Objetivos específicos
- Rastrear en `tpPedidoTramitarTransaccionBO.cs` dónde se invoca el extracto Venta Digital y confirmar que la condición FAA=1 en Prepago no lo omite ni lo cortocircuita.
- Verificar que los campos requeridos por Venta Digital (`tpPedidoVD.OrdenDeCompra`, `tpPedidoVD.IdPedido`, `tpPartidaPedidoVD.*`) se populan correctamente cuando FAA=1 en Prepago.
- Confirmar que el job de TaskScheduler puede leer y procesar el extracto generado (validar estructura de datos esperada vs. la que genera el flujo FAA Prepago).
- Clarificar si existe algún documento disponible para transferir a Legacy en el flujo Prepago con FAA: en RE-015 no se genera proforma, PDF ni correo, y el pendiente Validar Cobro tampoco se genera al tramitar (criterio heredado en T1) — confirmar si TaskScheduler debe operar sin documento en este caso, o si existe algún otro artefacto que deba transferir.
- Documentar cualquier ajuste necesario si se detecta que el flujo FAA Prepago altera el extracto o deja a TaskScheduler sin documento que transferir.

### Resultado esperado
El job de TaskScheduler de Venta Digital procesa correctamente los pedidos tramitados con FAA Prepago: la Orden de Compra queda registrada en Venta Digital y los PDFs se transfieren a Legacy sin diferencias respecto al flujo base.

### Entregables
- Reporte de validación: flujo FAA Prepago vs. flujo base en los datos escritos a `tpPedidoVD`/`tpPartidaPedidoVD`
- Aclaración de si existe algún documento disponible para transferencia a Legacy en el flujo Prepago con FAA (o si TaskScheduler debe operar sin ninguno)
- Ajustes en `tpPedidoTramitarTransaccionBO.cs` si se detecta alguna omisión (entregable condicional)
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
- En el flujo Prepago con FAA, T2 de RE-015 suprime el pendiente Validar Cobro al tramitar. Verificar que esta supresión no afecta la generación del extracto Venta Digital ni el PDF disponible para Legacy.

### Recursos
- `R16A-RE-FU-010-Back.md` — paso 12 (Extracto Venta Digital), tablas `tpPedidoVD`/`tpPartidaPedidoVD`
- `R16A-RE-FU-015-Back.md` — flujo tramitación Prepago con FAA
- `R16A-RE-FU-012-Tareas.md` — T8 (misma validación en variante Crédito, como referencia)
- `tpPedidoTramitarTransaccionBO.cs` — clase principal del flujo de tramitación

---

## Resumen de tareas

| # | Clave Catálogo | Título | Predecesora |
|---|----------------|--------|-------------|
| T1 | SERV-TRANSACT | Generación de pendiente FAA al tramitar con Factura por Adelantado (incluye criterios de T2) | — |
| T2 | ALG-COMPLX-LOGIC | ~~No generar pendiente Validar Cobro cuando FAA está activa~~ (OBSOLETA — ver T1) | — |
| T3 | ALG-BASIC-LOGIC | Eliminar código de autorización para Factura por Adelantado | — |
| T4 | ALG-BASIC-LOGIC | Bloquear datos de facturación al activar FAA | — |
| T5 | ALG-BASIC-LOGIC | Vinculación del pendiente FAA con módulo de facturación (RE-FU-018/019/020) | T1 |
| T6 | UPDATE-TABL-CH | ⛔ BLOQUEANTE — Crear catEstatusPedido + IdEstatusPedido en tpPedido (OBS-027) | — |
| T7 | ALG-COMPLX-LOGIC | ⛔ BLOQUEANTE — Lógica de transición de estados del pedido (OBS-027) | T6 |
| T8 | ALG-BASIC-LOGIC | Validar flujo Venta Digital al tramitar pedido Prepago con FAA (sin PDF disponible) | T1 |
| T9 | ALG-BASIC-LOGIC | Pruebas de flujo completo E2E — Tramitación Prepago con y sin FAA | T1,T3,T4,T8 |

---

## T9 — [ R16A-RE-FU-015 ] [ALG-BASIC-LOGIC] Pruebas de flujo completo E2E — Tramitación Prepago con y sin FAA

### Aplicativos
- ProquifaDotNet
- Venta Digital
- Legacy

### Módulos
- L05.TramitarPedido\Liberar
- L05.TramitarPedido\Facturas\Anticipos
- Extracto Venta Digital
- Validar Cobro

### Consideraciones previas
- Esta tarea valida impactos colaterales: cualquier modificación al flujo de tramitación Prepago debe demostrarse que no rompió el pipeline completo, incluyendo los módulos que dependen de él (Venta Digital, Legacy, Validar Cobro).
- El flujo Prepago tiene particularidades que lo diferencian de Crédito: el pendiente Validar Cobro se genera (o no) según FAA, y no hay transferencia directa a Legacy como en Crédito — confirmar qué sí transfiere TaskScheduler.
- Los escenarios cubren: flujo base Prepago sin FAA (no regresión RE-014), FAA activa, bloqueo de datos de facturación y reglas de negocio específicas de Prepago.
- Predecesoras: T1 (pendiente FAA, incluye verificación de no generación de proforma/VC), T3 (sin código autorización), T4 (bloqueo datos), T8 (validación VD).

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
| 2    | PDF de proforma generado (RE-016/017) — solo aplica al flujo base RE-014, ya no aplica a E2-E6 (actualización de requisito) |  ✓  |  —  |  —  |  —  |  —  |  —  |
| 3    | Correo de proforma enviado al cliente — solo aplica al flujo base RE-014, ya no aplica a E2-E6 (actualización de requisito) |  ✓  |  —  |  —  |  —  |  —  |  —  |
| 4    | Pendiente Validar Cobro generado (solo FAA=0)                              |  ✓  |  —  |  —  |  —  |  —  |  —  |
| 5    | Pendiente VC NO generado (FAA=1)                                           |  —  |  ✓  |  ✓  |  —  |  ✓  |  ✓  |
| 6    | Extracto Venta Digital: `tpPedidoVD` y `tpPartidaPedidoVD`                 |  ✓  |  ✓  |  ✓  |  —  |  ✓  |  ✓  |
| 7    | TaskScheduler VD: Orden de Compra procesada                                |  ✓  |  ✓  |  ✓  |  —  |  ✓  |  ✓  |
| 8    | TaskScheduler VD: documento transferido a Legacy (E2-E6: ⚠️ punto abierto — ya no hay PDF disponible, ver T8) |  ✓  |  ⚠️  |  —  |  —  |  —  |  —  |
| 9    | Pendiente FAA generado en `tpProformaAdelanto` (solo FAA=1)                |  —  |  ✓  |  ✓  |  —  |  ✓  |  ✓  |
| 10   | Datos de facturación fijados del catálogo vigente                          |  —  |  ✓  |  ✓  |  —  |  —  |  —  |
| 11   | Error al intentar editar datos con FAA activa (E4)                         |  —  |  —  |  —  |  ✓  |  —  |  —  |
| 12   | Activación sin solicitar código de autorización (E5)                       |  —  |  —  |  —  |  —  |  ✓  |  —  |
| 13   | Pendiente FAA consumible por módulo RE-018 (E6)                            |  —  |  —  |  —  |  —  |  —  |  ✓  |
| 14   | BitácoraCRUD registrada                                                    |  ✓  |  ✓  |  ✓  |  ✓  |  ✓  |  ✓  |

> **Nota (pasos 2, 3, 8):** tras la actualización de requisito, este flujo ya no genera proforma, PDF ni correo (ver Alcance — "No aplica a" en `R16A-RE-FU-015.md`). El punto abierto del paso 8 (E2-E6) se traslada a T8: confirmar si TaskScheduler debe operar sin ningún documento para transferir a Legacy en el flujo FAA Prepago.

### Objetivo general
Garantizar que las modificaciones introducidas en RE-015 (supresión pendiente VC, pendiente FAA, bloqueo datos, eliminación código autorización) no rompieron ningún paso del pipeline de tramitación Prepago ni los módulos dependientes.

### Resultado esperado
Todos los escenarios ejecutados con evidencia. Los pasos del pipeline marcados como aplicables funcionan correctamente. El flujo base Prepago sin FAA (E1) no presenta regresión respecto al comportamiento de RE-014.

### Entregables
- Tabla de evidencias por escenario (query o captura por paso verificado)
- Confirmación del PDF que TaskScheduler transfiere a Legacy en el flujo Prepago FAA (paso 8 — aclaración pendiente)
- Incidencias encontradas y su resolución
- Confirmación de que el pendiente FAA (paso 9) es consumible por RE-018 sin ajustes adicionales

### Criterios de aceptación
- [ ] E1 — Prepago base sin FAA: flujo completo RE-014 sin regresión (pasos 1-3, 4, 6-8, 14).
- [ ] E2 — FAA activa: pendiente VC suprimido (paso 5), pendiente FAA generado (paso 9), VD y TaskScheduler correctos.
- [ ] E3 — Datos fijados: los datos de facturación en `tpPedido`/`tpProformaAdelanto` corresponden al catálogo vigente del cliente al momento de tramitar.
- [ ] E4 — Bloqueo edición: el endpoint de edición de datos rechaza la operación con error claro cuando FAA=1.
- [ ] E5 — Sin código: el flujo de activación FAA no solicita ni valida código de autorización.
- [ ] E6 — Vinculación RE-018: el módulo de facturación puede leer y procesar el pendiente FAA sin inconsistencias de datos.
- [ ] Paso 8 aclarado: está documentado qué PDF transfiere TaskScheduler en el flujo Prepago FAA.
- [ ] PR aprobado por líder técnico con evidencias adjuntas.

### Más información de la tarea
- **Diferencia clave vs. RE-012:** en Prepago con FAA (tras la actualización de requisito) no se genera ni proforma ni confirmación de pedido en Tramitar Pedido — no hay ningún PDF disponible en ese momento. Confirmar si TaskScheduler debe operar sin documento para este flujo, o si debe esperar a un artefacto generado posteriormente por el módulo de facturación (RE-018/019/020).
- El paso 13 (E6) valida la integración hacia adelante: RE-015 pone el pendiente, RE-018 lo consume. Si el contrato de datos tiene diferencias, ambos requisitos se ven afectados.

### Recursos
- `R16A-RE-FU-014-Back.md` — flujo base Prepago (referencia para no regresión E1)
- `R16A-RE-FU-015-Back.md` — flujo Prepago con FAA
- `R16A-RE-FU-015-Tareas.md` — T8 (validación VD específica)
- `R16A-RE-FU-018-Back.md` — módulo de facturación que consume el pendiente FAA
- `R16A-RE-FU-012-Tareas.md` — T9 (referencia: misma estructura para variante Crédito)

---

## Dependencias con otros requisitos (NO incluidas como tareas)

| Requisito | Tarea relacionada | Relación |
|-----------|-------------------|----------|
| R16A-RE-FU-010 | Cancelación | Endpoint de cancelación se desarrolla en RE-FU-010 |
| R16A-RE-FU-013 | T6 (Perú) | Única parte del flujo base de RE-013 aún compartida — foliador, previsualización, envío de correo y vinculación PDF ya NO aplican (actualización de requisito) |
| R16A-RE-FU-014 | T1, T2 | Validación Remisión Prepago, datos facturación solo lectura |
| R16A-RE-FU-018/019/020 | Módulo FAA | Consume el pendiente generado en T1 |
| Venta Digital | T8 | TaskScheduler lee `tpPedidoVD`/`tpPartidaPedidoVD` para procesar OC y transferir documentos a Legacy — pendiente confirmar comportamiento sin PDF disponible |

> R16A-RE-FU-016/017 (generación de PDF/template de proforma) **ya no aplican** a este requisito — el requisito actualizado no genera proforma en este flujo.
