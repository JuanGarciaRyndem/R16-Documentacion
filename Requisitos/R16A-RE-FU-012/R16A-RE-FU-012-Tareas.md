# Tareas Back — R16A-RE-FU-012
**Requisito:** Tramitacion de pedidos Credito con Factura por Adelantado
**Total de tareas:** 9 (T5, T6, T7 agregadas 2026-06-11 — Migración SSIS; T8 2026-06-16 — Validación VD; T9 2026-06-16 — Escenarios E2E flujo completo)

---

## T1 — [ R16A-RE-FU-012 ] [ALG-BASIC-LOGIC] Revision de codigo existente tpProformaAdelanto para aprovechabilidad

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Facturas\Anticipos
- L05.TramitarPedido\Facturas\Generadores\Anticipo

### Consideraciones previas
- El flujo anterior de Factura por Adelantado NO se reutilizara directamente
- **Actualización (06/07/2026):** el destino final de datos ya NO es `tpProformaAdelanto` — es `fccFactura` (+ `fccFacturaPartida` + `fccFacturaReferenciaBancaria`), esquema definido y creado en RE-FU-015, propiedad de `ProquifaDotNet.Finanzas`. Esta revisión de código legacy sirve para identificar lógica de negocio reutilizable, no estructura de datos
- El desarrollador debe analizar que logica/entidades son aprovechables

### Objetivo general
Revisar el codigo existente relacionado con tpProformaAdelanto y determinar que logica es reutilizable para el nuevo flujo FAA sobre `fccFactura`.

### Objetivos especificos
- Revisar `tpProformaAdelantoBO.cs` y sus extensiones (lógica de negocio, no estructura)
- Revisar `tpProformaAdelantoToCFDIGeneradaBO.cs` (codigo comentado, guía para el mapeo hacia `CFDIGenerada`)
- Revisar `CFDIGeneradaConceptoAnticipoFactory.cs`
- Revisar tablas BD legacy: `tpProformaAdelanto`, `tpProformaAdelantoProformaPedido` (ya no aplican como destino) y `fccPagoFacturaAdelanto` (vigente, FK migrada a `IdFccFactura`)
- Documentar que logica es aprovechable hacia `fccFactura` y que debe crearse desde cero

### Resultado esperado
Documento de analisis con decision de aprovechabilidad del codigo legacy, mapeado contra el esquema `fccFactura`.

### Entregables
- Documento de analisis tecnico

### Criterios de aceptacion
- Se identifican claramente los componentes aprovechables vs los que requieren nuevo desarrollo
- Se valida contra el modelo de datos `fccFactura` de `R16A-RE-FU-015_BD.md` (no contra `tpProformaAdelanto`)

---

## T2 — [ R16A-RE-FU-012 ] [SERV-TRANSACT] Generacion de pendiente FAA en transaccion de tramitacion

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- Depende de T1 (revision de codigo existente)
- **Actualización (06/07/2026):** el INSERT del pendiente ya NO es en `tpProformaAdelanto` — es en `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria` (esquema de RE-FU-015, `ProquifaDotNet.Finanzas`), en paralelo a `tpProformaPedido` dentro de la misma transacción
- El INSERT del pendiente debe ser atomico con la tramitacion del pedido y con la Confirmación de Pedido
- Solo aplica cuando `tpPedido.FacturaPorAdelantado = 1`
- Solo region Mexico
- `fccFactura.IdTPProformaPedido` debe poblarse con el Id de la Confirmación de Pedido (`tpProformaPedido`) recién insertada — es la diferencia clave frente al flujo Prepago (RE-015), donde ese campo queda NULL
- **Campos fiscales regionalizados (Regla 9 — sincronización matriz):** Los datos de facturación que se fijan varían por región. Para México: RFC, RazonSocial, UsoCFDI, MetodoPago, RegimenFiscal. Para Perú (fuera de alcance R16 FAA-Crédito): RUC, RazonSocial, TipoOperacion, CondicionPago SUNAT. La Forma de Pago y el correo de envío NO se incluyen en el Panel de Información de Facturación. Ver Back.md Sección C.

### Objetivo general
Implementar la generacion del pendiente FAA en `fccFactura` dentro de la transaccion de tramitacion del pedido cuando FAA esta activa.

### Objetivos especificos
- Modificar `tpPedidoTramitarTransaccionBO.GenerarCorreoTramitarPedido()` para detectar FAA=1
- INSERT atómico en `fccFactura` (`IdTPPedido`, `IdTPProformaPedido`, `EsFacturaPorAdelantado=1`, `IdCliente`, `IdEmpresa`, `MontoTotal`, `IdCatMoneda`, `IdCFDIGenerada=NULL`, `Enviada=0`) + `fccFacturaPartida` (snapshot de partidas) + `fccFacturaReferenciaBancaria` (cuentas del grupo)
- Fijar datos de facturacion del catalogo vigente del cliente como snapshot en `fccFactura`
- Asegurar atomicidad (misma transaccion que tramitacion y que el INSERT de `tpProformaPedido`)
- Generar confirmacion de pedido inmediatamente (no espera factura), en paralelo al pendiente FAA

### Resultado esperado
Al tramitar un pedido con FAA=1 en Mexico, se genera un registro pendiente FAA en `fccFactura` (+ detalle) con todos los datos necesarios para que el modulo de facturacion lo consuma posteriormente vía `vfccFactura`.

### Entregables
- Modificacion de `tpPedidoTramitarTransaccionBO.cs`
- Llamada al servicio de Finanzas para INSERT del pendiente FAA en `fccFactura`

### Criterios de aceptacion
- El pendiente se genera atomicamente con la tramitacion y con la Confirmación de Pedido
- Contiene todos los datos requeridos (cliente, empresa, monto, datos facturacion, moneda, partidas, referencias bancarias)
- `fccFactura.IdTPProformaPedido` queda poblado con la Confirmación de Pedido generada
- No se genera pendiente si FAA=0
- No se genera pendiente si region != Mexico
- La confirmacion de pedido se genera sin esperar factura

---

## T3 — [ R16A-RE-FU-012 ] [ALG-BASIC-LOGIC] Validaciones Back para Factura por Adelantado

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- Se implementan dentro del flujo de tramitacion
- FAA no es compatible con productos controlados (validacion de RE-FU-011)

### Objetivo general
Implementar las validaciones de negocio requeridas para la activacion de FAA.

### Objetivos especificos
- Validar que FAA solo aplica para region Mexico (rechazar si FAA=1 y region != Mexico)
- Eliminar validacion de codigo de autorizacion si existe (Regla 3: activacion directa)
- Validar que datos de facturacion del cliente esten vigentes antes de fijarlos
- Bloquear edicion de datos de facturacion cuando `tpPedido.FacturaPorAdelantado = 1`

### Resultado esperado
El sistema rechaza tramitaciones con FAA activa que no cumplan las reglas de negocio.

### Entregables
- Validaciones en flujo de tramitacion
- Validacion en endpoint de edicion de datos facturacion

### Criterios de aceptacion
- Error claro si FAA=1 y region != Mexico
- No se solicita codigo de autorizacion para activar FAA
- Error si datos de facturacion no estan vigentes
- Error si se intenta editar datos facturacion con FAA activa

---

## T4 — [ R16A-RE-FU-012 ] [ALG-BASIC-LOGIC] Vinculacion del pendiente FAA con modulo de facturacion (RE-FU-018/019/020)

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Facturas

### Consideraciones previas
- La emision de factura, CFDI y timbrado se desarrollan en RE-FU-018, RE-FU-019 y RE-FU-020
- Esta tarea asegura que el pendiente generado en T2 sea consumible por esos modulos

### Objetivo general
Vincular el proceso de generacion del pendiente FAA con el flujo de facturacion desarrollado en los requisitos RE-FU-018/019/020.

### Objetivos especificos
- Verificar que `fccFactura` (con `IdTPProformaPedido` poblado) es compatible con lo que consulta RE-FU-018 vía `vfccFactura`
- Ajustar campos/relaciones si es necesario para la integracion
- Validar que el estado del pendiente (`EstadoFAA` calculado en `vfccFactura`) se actualice correctamente cuando facturacion lo consume (UPDATE de `IdCFDIGenerada`/`Enviada` sobre el mismo registro, no INSERT duplicado — RT-10)
- Documentar contrato de datos entre pendiente y modulo de facturacion

### Resultado esperado
El pendiente FAA generado en la tramitacion es consumido correctamente por el modulo de facturacion sin inconsistencias.

### Entregables
- Ajustes de integracion (si aplican)
- Documentacion del contrato de datos entre modulos

### Criterios de aceptacion
- El modulo de facturacion (RE-FU-018) puede leer y procesar el pendiente FAA
- El estado del pendiente se actualiza al ser procesado
- No hay datos faltantes ni inconsistencias entre ambos modulos

---

## T8 — [ R16A-RE-FU-012 ] [ALG-BASIC-LOGIC] Validar flujo Venta Digital al tramitar pedido Crédito con FAA

### Aplicativos
- ProquifaDotNet
- Venta Digital

### Módulos
- L05.TramitarPedido\Liberar
- Extracto Venta Digital (`tpPedidoVD`, `tpPartidaPedidoVD`)

### Consideraciones previas
- El flujo de tramitación base (RE-FU-010) incluye como paso 12 el "Extracto Venta Digital": INSERT/UPDATE en `tpPedidoVD` y `tpPartidaPedidoVD`.
- RE-012 extiende ese flujo con la variante FAA; se debe verificar que el paso de Venta Digital sigue ejecutándose correctamente independientemente de si FAA está activa o no.
- **TaskScheduler en Venta Digital:** existe un job programado (Windows Task Scheduler) que depende de los datos escritos en `tpPedidoVD`/`tpPartidaPedidoVD`. Ese job realiza dos acciones:
  1. **Procesamiento de Órdenes de Compra** — lee el extracto y genera/actualiza la Orden de Compra en Venta Digital.
  2. **Transferencia de PDFs a Legacy** — toma los PDFs asociados al pedido y los transfiere al directorio configurado de Legacy.
- Si el paso de Extracto Venta Digital se omite o genera datos incorrectos en el flujo FAA, el job de TaskScheduler falla silenciosamente, dejando la OC sin procesar y los PDFs sin transferir a Legacy.

### Objetivo general
Verificar que el paso de Extracto Venta Digital (INSERT/UPDATE en `tpPedidoVD`/`tpPartidaPedidoVD`) se ejecuta correctamente en el flujo de tramitación Crédito con FAA, garantizando que el job de TaskScheduler de Venta Digital pueda procesar la Orden de Compra y transferir los PDFs a Legacy sin errores.

### Objetivos específicos
- Rastrear en `tpPedidoTramitarTransaccionBO.cs` dónde se invoca el extracto Venta Digital y confirmar que la condición FAA=1 no lo omite ni lo cortocircuita.
- Verificar que los campos requeridos por Venta Digital (`tpPedidoVD.OrdenDeCompra`, `tpPedidoVD.IdPedido`, `tpPartidaPedidoVD.*`) se populan correctamente cuando FAA=1.
- Confirmar que el job de TaskScheduler puede leer y procesar el extracto generado (validar estructura de datos esperada vs. la que genera el flujo FAA).
- Verificar que la transferencia de PDFs a Legacy funciona correctamente: el PDF de confirmación de pedido se genera antes de la factura (ya que FAA genera la confirmación inmediatamente, sin esperar la factura).
- Documentar cualquier ajuste necesario si se detecta que el flujo FAA altera el extracto o el PDF de alguna forma.

### Resultado esperado
El job de TaskScheduler de Venta Digital procesa correctamente los pedidos tramitados con FAA: la Orden de Compra queda registrada en Venta Digital y los PDFs se transfieren a Legacy sin diferencias respecto al flujo base (RE-010).

### Entregables
- Reporte de validación: flujo FAA vs. flujo base en los datos escritos a `tpPedidoVD`/`tpPartidaPedidoVD`
- Ajustes en `tpPedidoTramitarTransaccionBO.cs` si se detecta alguna omisión (entregable condicional)
- Evidencia de que el job de TaskScheduler procesa correctamente el extracto FAA (queries o capturas)

### Criterios de aceptación
- [ ] El extracto Venta Digital se genera en `tpPedidoVD`/`tpPartidaPedidoVD` al tramitar con FAA=1 (igual que con FAA=0).
- [ ] Los campos requeridos por el job de TaskScheduler están presentes y correctos.
- [ ] El job de TaskScheduler procesa la Orden de Compra sin errores cuando el pedido tiene FAA=1.
- [ ] Los PDFs se transfieren a Legacy correctamente (el PDF de confirmación de pedido se genera antes que la factura en el flujo FAA).
- [ ] No se genera diferencia funcional entre FAA=1 y FAA=0 en los datos entregados a Venta Digital.

### Más información de la tarea
- El flujo base de Extracto Venta Digital se define en RE-FU-010, paso 12 (tablas `tpPedidoVD`, `tpPartidaPedidoVD`).
- El job de TaskScheduler en Venta Digital opera sobre datos de `ppPedidoVD`/`ppPartidaPedidoVD` (pretramitado) y `tpPedidoVD`/`tpPartidaPedidoVD` (tramitado). Tiene dos responsabilidades: procesar la OC y transferir PDFs a Legacy.
- En el flujo FAA la confirmación de pedido se genera inmediatamente (paso 6 en RE-012 Back.md), sin esperar la factura — esto afecta qué PDF está disponible para transferir a Legacy en el momento en que TaskScheduler se ejecuta.

### Recursos
- `R16A-RE-FU-010-Back.md` — paso 12 (Extracto Venta Digital), tablas `tpPedidoVD`/`tpPartidaPedidoVD`
- `R16A-RE-FU-012-Back.md` — flujo tramitación Crédito con FAA, paso 7 (Transferencia Legacy)
- `tpPedidoTramitarTransaccionBO.cs` — clase principal del flujo de tramitación

---

## T9 — [ R16A-RE-FU-012 ] [ALG-BASIC-LOGIC] Pruebas de flujo completo E2E — Tramitación Crédito con y sin FAA

### Aplicativos
- ProquifaDotNet
- Venta Digital
- Legacy

### Módulos
- L05.TramitarPedido\Liberar
- Extracto Venta Digital
- Transferencia Legacy

### Consideraciones previas
- Esta tarea valida impactos colaterales: cualquier modificación al flujo de tramitación Crédito debe demostrarse que no rompió el pipeline completo, desde el trigger hasta el último paso downstream.
- No es suficiente validar solo la funcionalidad específica del requisito (pendiente FAA). El developer debe ejecutar todos los escenarios antes de solicitar revisión del PR.
- Los escenarios cubren: flujo base sin FAA (no regresión RE-010), variantes de RE-011 (controlados, Perú, PCE) y la variante FAA de este requisito (RE-012).
- Predecesoras: T1 (análisis código), T2 (pendiente FAA), T3 (validaciones), T4 (vinculación facturación), T8 (validación VD).

### Escenarios de prueba

| # | Escenario | FAA | Región | Controlados | PCE |
|---|-----------|:---:|--------|:-----------:|:---:|
| E1 | Crédito base — no regresión RE-010 | ✗ | MEX | ✗ | ✗ |
| E2 | Crédito con FAA activa | ✓ | MEX | ✗ | ✗ |
| E3 | Crédito FAA + región Perú (debe rechazarse) | ✓ | PER | ✗ | ✗ |
| E4 | Crédito Pago Contra Entrega — no regresión RE-010 | ✗ | MEX | ✗ | ✓ |
| E5 | Crédito con sustancias controladas — no regresión RE-011 | ✗ | MEX | ✓ | ✗ |
| E6 | Crédito Perú base (sin transferencia Legacy) | ✗ | PER | ✗ | ✗ |
| E7 | Cancelación de pedido tramitado | — | MEX | — | — |

### Pipeline a verificar por escenario

Para cada escenario aplicable, verificar que cada paso del pipeline se ejecutó correctamente:

| Paso | Qué verificar | E1 | E2 | E3 | E4 | E5 | E6 | E7 |
|------|--------------|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| 1 | Folio de pedido asignado | ✓ | ✓ | — | ✓ | ✓ | ✓ | — |
| 2 | PDF de confirmación generado correctamente | ✓ | ✓ | — | ✓ | ✓ | ✓ | — |
| 3 | Correo de confirmación enviado al cliente | ✓ | ✓ | — | ✓ | ✓ | ✓ | — |
| 4 | Extracto Venta Digital: registro en `tpPedidoVD` y `tpPartidaPedidoVD` | ✓ | ✓ | — | ✓ | ✓ | ✓ | — |
| 5 | TaskScheduler VD: Orden de Compra procesada en Venta Digital | ✓ | ✓ | — | ✓ | ✓ | ✓ | — |
| 6 | TaskScheduler VD: PDFs transferidos a Legacy | ✓ | ✓ | — | ✓ | ✓ | — | — |
| 7 | Transferencia Legacy: Pedido + Partidas enviados | ✓ | ✓ | — | ✓ | ✓ | — | — |
| 8 | Pendiente FAA generado (solo con FAA=1) | — | ✓ | — | — | — | — | — |
| 9 | No se genera pendiente FAA (cuando FAA=0) | ✓ | — | — | ✓ | ✓ | ✓ | — |
| 10 | BitácoraCRUD registrada por operación | ✓ | ✓ | — | ✓ | ✓ | ✓ | ✓ |
| 11 | E3: error claro al intentar FAA en Perú | — | — | ✓ | — | — | — | — |
| 12 | E4: PCE se traduce como crédito en Legacy (OBS-024) | — | — | — | ✓ | — | — | — |
| 13 | E5: correo a Regulatory Affairs enviado | — | — | — | — | ✓ | — | — |
| 14 | E6: ningún dato en Legacy para región Perú | — | — | — | — | — | ✓ | — |
| 15 | E7: pedido cancelado no aparece en bandeja ESAC | — | — | — | — | — | — | ✓ |
| 16 | E7: `tpPedidoVD` inactivado o marcado al cancelar | — | — | — | — | — | — | ✓ |

### Objetivo general
Garantizar que las modificaciones introducidas en RE-012 (pendiente FAA, validaciones, bloqueo datos facturación) no rompieron ningún paso del pipeline de tramitación Crédito en ninguna de sus variantes.

### Resultado esperado
Todos los escenarios ejecutados con evidencia. Los pasos del pipeline marcados como aplicables funcionan correctamente. No hay regresión en el flujo base ni en las variantes RE-010/RE-011.

### Entregables
- Documento o tabla de evidencias por escenario (query o captura por paso verificado)
- Incidencias encontradas y su resolución
- Confirmación explícita de que E7 (cancelación) actualiza correctamente `tpPedidoVD`

### Criterios de aceptación
- [ ] E1 — Flujo base Crédito sin FAA: los 10 pasos generales se completan sin errores.
- [ ] E2 — FAA activa: pasos 1-10 completos + pendiente FAA generado (paso 8).
- [ ] E3 — FAA en Perú: el sistema rechaza con error claro sin generar ningún registro downstream.
- [ ] E4 — PCE: Legacy recibe la marca de crédito (no prepago) según OBS-024.
- [ ] E5 — Controlados: correo a Regulatory Affairs enviado además del correo estándar.
- [ ] E6 — Perú base: ningún dato llega a Legacy ni a TaskScheduler para transferencia.
- [ ] E7 — Cancelación: pedido desaparece de bandeja ESAC; `tpPedidoVD` refleja el estado cancelado.
- [ ] PR aprobado por líder técnico con evidencias adjuntas.

### Más información de la tarea
- El paso 16 (E7: impacto cancelación en `tpPedidoVD`) es una brecha identificada: no está documentado si el flujo de cancelación actualiza esta tabla. El developer debe confirmar el comportamiento actual y corregirlo si es necesario.
- El paso 6 (TransferenciaPDFs vía TaskScheduler) depende de que el PDF esté disponible antes de que el job se ejecute — confirmar timing en ambiente de QA.

### Recursos
- `R16A-RE-FU-010-Back.md` — flujo base + paso 12 Extracto VD + Sección B Cancelación
- `R16A-RE-FU-011-Back.md` — variantes controlados y Perú
- `R16A-RE-FU-012-Back.md` — flujo FAA Crédito
- `R16A-RE-FU-012-Tareas.md` — T8 (validación VD específica)

---

## Resumen de tareas

| # | Clave Catalogo | Titulo | Predecesora |
|---|----------------|--------|-------------|
| T1 | ALG-BASIC-LOGIC | Revision de codigo existente tpProformaAdelanto para aprovechabilidad | — |
| T2 | SERV-TRANSACT | Generacion de pendiente FAA en transaccion de tramitacion | T1 |
| T3 | ALG-BASIC-LOGIC | Validaciones Back para Factura por Adelantado | T1 |
| T4 | ALG-BASIC-LOGIC | Vinculacion del pendiente FAA con modulo de facturacion (RE-FU-018/019/020) | T2 |
| T5 | QUERY-G | Análisis — Migración de transferencia Pedidos Crédito de SSIS a LegacySync | — |
| T6 | SERV-COMPLEX-TRANSACT | Implementación — PedidoCreditoSyncJob en LegacySync + disparador SyncControl | T5 |
| T7 | SERV-TRANSACT | Integración — Validación E2E en LegacySync y desactivación del SSIS de Pedidos Crédito | T5, T6 |
| T8 | ALG-BASIC-LOGIC | Validar flujo Venta Digital al tramitar pedido Crédito con FAA | T2 |
| T9 | ALG-BASIC-LOGIC | Pruebas de flujo completo E2E — Tramitación Crédito con y sin FAA | T1,T2,T3,T4,T8 |

---

## Dependencias con otros requisitos (NO incluidas como tareas)

| Requisito | Tarea relacionada | Relacion |
|-----------|-------------------|----------|
| R16A-RE-FU-010 | T3 (Cancelacion) | Endpoint de cancelacion se desarrolla en RE-FU-010 |
| R16A-RE-FU-011 | Validacion controlados | fnEsProductoControlado se desarrolla en RE-FU-011 |
| R16A-RE-FU-018 | Generacion de factura | Consume el pendiente FAA generado en T2 |
| R16A-RE-FU-019 | Generacion de CFDI | Proceso posterior a factura |
| R16A-RE-FU-020 | Timbrado fiscal (PAC) | Proceso posterior a CFDI |
| Venta Digital | T8 | TaskScheduler lee `tpPedidoVD`/`tpPartidaPedidoVD` para procesar OC y transferir PDFs a Legacy |
| R16A-RE-FU-008-Legacy | T5, T6, T7 | Infraestructura base de ProquifaDotNet.LegacySync (SyncControl, SyncJobLog, SyncJobBase, Hangfire, ExceptionClassifier, Brevo) — se crea allá, aquí solo se consume |



---

## T5 — [ R16A-RE-FU-012 ] [ QUERY-G ] Análisis — Migración de transferencia Pedidos Crédito de SSIS a LegacySync

### Aplicativos
- ProquifaDotNet.LegacySync / ETLs SSIS / ProquifaDotNet

### Módulos
- LegacySync — Migración transferencia Pedidos Crédito → Legacy (Análisis)

### Consideraciones previas
- La transferencia de Pedidos Crédito a Legacy actualmente se realiza mediante un paquete SSIS en PCconnect. El objetivo de esta migración es **trasladar esa lógica a la solución ProquifaDotNet.LegacySync** (ver `Soluciones Nuevas/ProquifaDotNet.LegacySync.md` y `R16A-RE-FU-008-Legacy.md`), bajo el modelo `SyncControl` + recurring job de Hangfire — NO como servicio post-commit dentro del aplicativo ProquifaDotNet.
- La migración debe contemplar las tres variantes del flujo de Pedido Crédito en R16: RE-010 (base + Pago contra entrega), RE-011 (Sustancias controladas, Perú sin transferencia), RE-012 (FAA paralelo).
- **Perú no transfiere a Legacy** — conforme a la regla de LegacySync, la evaluación de región se realiza en el servicio de sincronización, no en la configuración del job ni en el SSIS.
- El único cambio en ProquifaDotNet es el disparador: INSERT en `SyncControl` (BD `PConnectProquifaDotNet`) post-commit de la tramitación. Toda la lógica de transferencia, reintentos y notificación vive en LegacySync.
- Esta tarea analiza el SSIS existente para entender qué lógica debe migrarse y cómo mapearla al patrón `SyncJobBase` de LegacySync. Es prerequisito bloqueante para T6 y T7.
- La infraestructura base de LegacySync (solución, `SyncControl`, `SyncJobLog`, `SyncJobBase`, Hangfire, `ExceptionClassifier`, `BrevoNotificationService`) se crea en **R16A-RE-FU-008-Legacy** — este requisito solo agrega el job de la entidad Pedidos Crédito; no duplicar esas tareas aquí.

### Descripción del problema
El paquete SSIS de PCconnect realiza la transferencia de Pedidos Crédito a Legacy. Al migrar a LegacySync, esta lógica debe replicarse como un job de sincronización por entidad (patrón `SyncJobBase` + Hangfire), incluyendo las variantes nuevas de R16 que el SSIS actual no contempla (pago contra entrega, controlados, FAA, corte Perú).

### Objetivos específicos
- Analizar el paquete SSIS existente de Pedidos Crédito: identificar qué tablas lee, qué datos envía a Legacy (Pedido, Partidas), qué SPs invoca (`spActualizarBuzonPedidoLegacy` / `spActualizarBuzonPedidoLegacyEncolar`) y bajo qué condiciones.
- Definir el modelo de migración a LegacySync:
  - Job a crear: `PedidoCreditoSyncJob` (hereda de `SyncJobBase`, Worker.LegacySync) con su servicio `PedidoCreditoSyncService` (`EjecutarSyncAsync`).
  - Builder del payload: `PedidoCreditoPayloadBuilder` que construya el objeto a escribir en PCconnect (vía `PConnectDbContext` o los SPs existentes — definir en este análisis).
  - Disparador: INSERT en `SyncControl` (`Entidad='PedidoCredito'`, `IdRegistro=IdTPPedido`, `Estado='Pendiente'`) post-commit en `tpPedidoTramitarController` (o el punto de tramitación definitivo).
  - Lectura de origen: `tpPedido` + partidas vía `ProquifaDotNetDbContext` (Scaffold incremental — agregar las tablas que falten).
- Mapear las variantes R16 en el nuevo job:
  - **RE-010 base:** Pedido + Partidas estándar.
  - **RE-010 Pago contra entrega:** marca de detención derivada de `catCondicionesDePago.Clave = 'pagocontraentrega'`.
  - **RE-011 Controlados (México):** incluir flag de sustancia controlada si Legacy lo requiere.
  - **RE-011 Perú:** `PedidoCreditoSyncService` evalúa la región y marca el registro `Completado` sin transferir — sin error, sin notificación Brevo.
  - **RE-012 FAA:** transferencia idéntica al flujo base (FAA es paralelo, no altera el payload del pedido).
- Definir la escritura destino en PCconnect: reutilizar los SPs Legacy existentes o escribir directo en las tablas receptoras vía `PConnectDbContext` (Scaffold).
- Confirmar si el SSIS debe desactivarse tras la migración o si ambos pueden coexistir temporalmente (riesgo de doble inserción).
- Documentar el diseño del job como insumo para T6.

### Resultado esperado
Documento de análisis con: lógica del SSIS existente documentada, diseño del `PedidoCreditoSyncJob` en LegacySync (job, servicio, builder, disparador SyncControl, destino PCconnect), mapeo de variantes RE-010/011/012, y plan de desactivación del SSIS.

### Entregables
- Documento de análisis de migración con:
  - Lógica del SSIS existente (tablas, columnas, SPs, condiciones)
  - Diseño del job LegacySync: `PedidoCreditoSyncJob`, `PedidoCreditoSyncService`, `PedidoCreditoPayloadBuilder`, punto de INSERT en `SyncControl`
  - Tablas a agregar al Scaffold de `ProquifaDotNetDbContext` y `PConnectDbContext`
  - Mapeo de variantes RE-010 (base + pago contra entrega), RE-011 (controlados + Perú), RE-012 (FAA)
  - Decisión SPs Legacy vs escritura directa vía `PConnectDbContext`
  - Plan de coexistencia o desactivación del SSIS

### Criterios de aceptación
- [ ] La lógica del SSIS existente está documentada (tablas, SPs, condiciones).
- [ ] El diseño del job en LegacySync está definido (job, servicio, builder, disparador SyncControl, destino).
- [ ] Las variantes RE-010, RE-011 y RE-012 están mapeadas en el nuevo diseño.
- [ ] La decisión SPs Legacy vs escritura directa está confirmada con arquitectura.
- [ ] El plan de desactivación/coexistencia del SSIS está acordado.
- [ ] El documento está aprobado como prerequisito para T6 y T7.

### Más información de la tarea
- Patrón de referencia: `SyncJobBase` + `SyncControl` + Hangfire de ProquifaDotNet.LegacySync (`R16A-RE-FU-008-Legacy.md`).
- SPs Legacy a reemplazar o reutilizar: `spActualizarBuzonPedidoLegacy` / `spActualizarBuzonPedidoLegacyEncolar`.
- Controlador de tramitación (punto del INSERT en SyncControl): `WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\Liberar\tpPedidoTramitarController.cs`
- La infraestructura base de LegacySync NO se crea en este requisito (pertenece a RE-008-Legacy) — solo se referencia como dependencia.

### Recursos
- `Soluciones Nuevas/ProquifaDotNet.LegacySync.md` — arquitectura de la solución
- `R16A-RE-FU-008-Legacy.md` / `R16A-RE-FU-008-Legacy-Tareas.md` — infraestructura base (SyncControl, SyncJobBase, Hangfire)
- `R16A-RE-FU-010-Back.md`, `R16A-RE-FU-011-Back.md`, `R16A-RE-FU-012-Back.md` — secciones Transferencia a Legacy
- Paquete SSIS existente de Pedidos Crédito en PCconnect

---

## T6 — [ R16A-RE-FU-012 ] [ SERV-COMPLEX-TRANSACT ] Implementación — PedidoCreditoSyncJob en LegacySync + disparador SyncControl

### Aplicativos
- ProquifaDotNet.LegacySync / ProquifaDotNet

### Módulos
- LegacySync — Migración transferencia Pedidos Crédito → Legacy (Implementación)

### Consideraciones previas
- **Predecesora: T5.** El diseño del job debe estar aprobado antes de implementar.
- **Predecesora externa:** infraestructura base de LegacySync operativa (RE-008-Legacy: `SyncControl`, `SyncJobLog`, `SyncJobBase`, Hangfire, `ExceptionClassifier`, `BrevoNotificationService`) — solo se referencia, no se duplica aquí.
- La lógica de transferencia vive íntegramente en LegacySync; en ProquifaDotNet solo se agrega el INSERT en `SyncControl` post-commit de la tramitación. El flujo de tramitación NO se bloquea por la transferencia: si LegacySync está caído, el registro queda `Pendiente` y el job lo procesa en el siguiente ciclo.
- **Perú no transfiere:** `PedidoCreditoSyncService` evalúa la región antes de ejecutar cualquier lógica (regla de LegacySync: la evaluación de región vive en el servicio de sincronización, no en la configuración del job).

### Descripción del problema
La lógica de transferencia de Pedidos Crédito a Legacy vive en el SSIS de PCconnect. Al migrar a LegacySync, se crea el job de la entidad Pedidos Crédito (patrón `SyncJobBase` + Hangfire) que reemplaza esa lógica, incluyendo las variantes R16, con reintentos automáticos y notificación de fallos vía Brevo.

### Objetivos específicos
- **ProquifaDotNet (disparador):** agregar en el punto de tramitación definido en T5 (`tpPedidoTramitarController` o equivalente) el INSERT post-commit en `SyncControl` (`Entidad='PedidoCredito'`, `IdRegistro=IdTPPedido`, `Estado='Pendiente'`).
- **LegacySync (Worker):** crear `PedidoCreditoSyncJob` heredando de `SyncJobBase`, registrado como recurring job de Hangfire.
- **LegacySync (Application):** crear `PedidoCreditoSyncService` con `EjecutarSyncAsync`:
  - Lee `tpPedido` + partidas vía `ProquifaDotNetDbContext` (agregar tablas al Scaffold si faltan).
  - Evalúa región: si `region.Clave != MEX`, marca el registro `Completado` sin transferir — sin error, sin notificación.
  - Escribe en PCconnect vía `PConnectDbContext` o SPs Legacy (según decisión de T5).
- Crear `PedidoCreditoPayloadBuilder` que construya el objeto a enviar a Legacy:
  - Campos base: Pedido (folio, cliente, montos), Partidas (producto, piezas, precios).
  - Variante Pago contra entrega: incluir marca de detención si `catCondicionesDePago.Clave = 'pagocontraentrega'`.
  - **OBS-024:** PCE (`catCondicionesDePago.Clave = 'pagocontraentrega'`) se traduce como **crédito** en el payload de Legacy, **NO como prepago**. Legacy procesa PCE como flujo de crédito independientemente del nombre de la condición.
  - Variante Controlados (RE-011): incluir flag si Legacy lo requiere según análisis T5.
  - Variante FAA (RE-012): payload idéntico al base (sin campos adicionales, FAA es paralelo).
- **OBS-025:** LegacySync solo inserta datos planos en Legacy. La lógica de "Relacionar facturas" (asociación de facturas al pedido dentro de Legacy) es responsabilidad del proceso interno de Legacy — **el `PedidoCreditoPayloadBuilder` no implementa esta lógica**.
- Manejo de errores con el patrón estándar de LegacySync: `ExceptionClassifier` (Transient → reintento Hangfire con backoff; Permanent → `Error` + `SyncJobLog` + notificación Brevo).
- Registrar cada ejecución en `SyncJobLog` con snapshot JSON del payload.
- Pruebas unitarias del builder para cada variante (base, pago contra entrega, controlados, FAA) y del corte de región en `PedidoCreditoSyncService`.

### Resultado esperado
El `PedidoCreditoSyncJob` procesa los registros `Pendiente` de `SyncControl` y transfiere Pedidos Crédito a PCconnect para las variantes RE-010/011/012, con reintentos, log estructurado y corte regional. La tramitación en ProquifaDotNet solo inserta el evento y no se bloquea.

### Entregables
- INSERT en `SyncControl` post-commit en el controlador de tramitación (ProquifaDotNet)
- `PedidoCreditoSyncJob` — recurring job Hangfire (Worker.LegacySync)
- `PedidoCreditoSyncService` — servicio de sincronización con corte regional
- `PedidoCreditoPayloadBuilder` — builder con variantes RE-010/011/012
- Scaffold incremental: tablas origen en `ProquifaDotNetDbContext` y destino en `PConnectDbContext`
- Registro del job en la configuración de Hangfire y en `AppSettings` (cron, reintentos, backoff)
- Pruebas unitarias del builder (por variante) y del corte de región

### Criterios de aceptación
- [ ] Al tramitar un Pedido Crédito se inserta el registro en `SyncControl` post-commit (México y Perú) sin bloquear la tramitación.
- [ ] El job procesa registros `Pendiente` y los marca `EnProceso` → `Completado`/`Error` según el patrón `SyncJobBase`.
- [ ] Para Perú: el servicio marca `Completado` sin transferir — sin error, sin notificación Brevo.
- [ ] El builder genera el payload correcto para el flujo base (RE-010).
- [ ] El builder incluye la marca de detención para Pago contra entrega (RE-010).
- [ ] **OBS-024:** El builder traduce PCE (`catCondicionesDePago.Clave = 'pagocontraentrega'`) como **crédito** en Legacy, no como prepago.
- [ ] **OBS-025:** El builder no implementa lógica de "Relacionar facturas" — esa responsabilidad recae en Legacy.
- [ ] El builder contempla la variante de controlados (RE-011) según el análisis T5.
- [ ] El builder genera el payload FAA idéntico al base (RE-012).
- [ ] Errores Transient reintentan con backoff; errores Permanent notifican vía Brevo y quedan en `SyncJobLog`.
- [ ] Pruebas unitarias del builder aprobadas para todas las variantes.
- [ ] PR aprobado por líder técnico.

### Más información de la tarea
- Patrón de referencia: `SyncJobBase` + jobs por entidad de LegacySync (`R16A-RE-FU-008-Legacy.md`); jobs análogos de RE-028 (Buzón Cobros/Proforma/Factura).
- La desactivación del SSIS y la validación E2E se cierran en T7.

### Recursos
- `Soluciones Nuevas/ProquifaDotNet.LegacySync.md` — arquitectura (SyncJobBase, ExceptionClassifier, Hangfire)
- Documento de análisis de T5
- `tpPedidoTramitarController.cs` — punto del INSERT en SyncControl
- `R16A-RE-FU-010-Back.md`, `R16A-RE-FU-011-Back.md`, `R16A-RE-FU-012-Back.md`

---

## T7 — [ R16A-RE-FU-012 ] [ SERV-TRANSACT ] Integración — Validación E2E en LegacySync y desactivación del SSIS de Pedidos Crédito

### Aplicativos
- ProquifaDotNet.LegacySync / ETLs SSIS / ProquifaDotNet / PCconnect

### Módulos
- LegacySync — Migración transferencia Pedidos Crédito → Legacy (Integración y cierre)

### Consideraciones previas
- **Predecesoras: T5 y T6.** El `PedidoCreditoSyncJob` debe estar implementado y desplegado en QA antes de esta tarea.
- El canal de transferencia queda resuelto por LegacySync (lectura `ProquifaDotNetDbContext` → escritura `PConnectDbContext`/SPs Legacy) — la antigua Brecha B3 de canal no aplica a este flujo.
- Una vez validada la transferencia vía LegacySync, el paquete SSIS de PCconnect debe desactivarse o retirarse según el plan acordado en T5, evitando el periodo de doble inserción.
- **Perú no transfiere:** validar en pruebas que ningún pedido de región Perú genera registros en Legacy (el registro de SyncControl queda `Completado` sin transferencia).
- **OBS-025:** La integración E2E debe confirmar que "Relacionar facturas" **no es responsabilidad de LegacySync** — es un proceso interno de Legacy. No replicar ni validar esta lógica en el job.

### Descripción del problema
Con el job funcionando en QA, esta tarea cierra la migración: valida E2E contra Legacy real (tramitación → SyncControl → Hangfire → PCconnect), verifica el monitoreo y los reintentos, y desactiva el SSIS de PCconnect para eliminar la fuente doble.

### Objetivos específicos
- Ejecutar pruebas de integración E2E en QA/staging cubriendo:
  - **RE-010 base:** Pedido + Partidas → Legacy sin errores.
  - **RE-010 Pago contra entrega:** marca de detención presente en Legacy.
  - **RE-011 Controlados (México):** transferencia correcta a Legacy.
  - **RE-011 Perú:** ningún registro llega a Legacy; SyncControl queda `Completado` sin transferencia.
  - **RE-012 FAA:** transferencia idéntica al flujo base.
- Comparar directamente ProquifaDotNet ↔ Legacy: Pedido (folio, cliente, montos), Partidas (producto, piezas, precios), marca de detención (si aplica).
- Validar el ciclo de reintentos: simular fallo Transient (Legacy no disponible) → reintento con backoff; fallo Permanent → `Error` + `SyncJobLog` + notificación Brevo.
- Validar el monitoreo vía API de LegacySync: estado por entidad, pendientes, historial de intentos y `ForzarReintentoCommand` para registros en error reintentable.
- Verificar no regresión del flujo base (pedidos anteriores a R16).
- Coordinar con el equipo PCconnect/Legacy la desactivación del paquete SSIS una vez validada la transferencia vía LegacySync.
- Documentar evidencias, incidencias, resolución y el cierre de la migración.

### Resultado esperado
La transferencia de Pedidos Crédito a Legacy se realiza íntegramente vía ProquifaDotNet.LegacySync para las variantes RE-010/011/012, con reintentos y monitoreo operativo. El SSIS de PCconnect está desactivado o retirado. Las evidencias están documentadas y aprobadas para producción.

### Entregables
- Documento de resultados de integración:
  - Escenarios ejecutados (RE-010 base, pago contra entrega, RE-011 controlados, RE-011 Perú, RE-012 FAA)
  - Comparación ProquifaDotNet ↔ Legacy por variante
  - Verificación de corte Perú (sin registros en Legacy)
  - Validación de reintentos (Transient/Permanent) y notificación Brevo
  - Validación del monitoreo API y reintento manual
  - No regresión del flujo base
  - Incidencias y resolución
- Evidencias de validación (queries o capturas por escenario, registros SyncControl/SyncJobLog)
- Confirmación de desactivación del SSIS en PCconnect
- Aprobación para paso a producción

### Criterios de aceptación
- [ ] RE-010 base: Pedido y Partidas en Legacy con valores correctos.
- [ ] RE-010 Pago contra entrega: marca de detención presente en Legacy.
- [ ] RE-011 Controlados (México): transferencia correcta a Legacy.
- [ ] RE-011 Perú: ningún registro de región Perú en Legacy; SyncControl `Completado` sin transferencia.
- [ ] RE-012 FAA: transferencia idéntica al flujo base.
- [ ] Reintentos Transient con backoff y notificación Brevo en errores Permanent, verificados con evidencia en `SyncJobLog`.
- [ ] Monitoreo API operativo (estado, pendientes, historial, reintento manual).
- [ ] Flujo base (pedidos previos a R16) sin regresión.
- [ ] El paquete SSIS de PCconnect está desactivado o retirado según el plan.
- [ ] Las pruebas están documentadas con evidencias y aprobadas por el líder técnico.
- [ ] Aprobado para paso a producción.

### Más información de la tarea
Esta tarea cierra la migración completa del ETL de Pedidos Crédito — de SSIS a ProquifaDotNet.LegacySync — para las variantes RE-010, RE-011 y RE-012.

### Recursos
- `Soluciones Nuevas/ProquifaDotNet.LegacySync.md` — flujo funcional, API de monitoreo
- Documento de análisis de T5 e implementación de T6
- Acceso a Legacy en ambiente QA/staging
- Equipo PCconnect para desactivación del SSIS
