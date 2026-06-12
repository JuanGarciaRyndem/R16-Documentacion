# Tareas Back — R16A-RE-FU-012
**Requisito:** Tramitacion de pedidos Credito con Factura por Adelantado
**Total de tareas:** 7 (T5, T6, T7 agregadas 2026-06-11 — Migración transferencia Pedidos Crédito de SSIS a aplicativo, variantes RE-010/011/012)

---

## T1 — [ R16A-RE-FU-012 ] [ALG-BASIC-LOGIC] Revision de codigo existente tpProformaAdelanto para aprovechabilidad

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Facturas\Anticipos
- L05.TramitarPedido\Facturas\Generadores\Anticipo

### Consideraciones previas
- El flujo anterior de Factura por Adelantado NO se reutilizara directamente
- El desarrollador debe analizar que logica/entidades son aprovechables

### Objetivo general
Revisar el codigo existente relacionado con tpProformaAdelanto y determinar que es reutilizable para el nuevo flujo FAA.

### Objetivos especificos
- Revisar `tpProformaAdelantoBO.cs` y sus extensiones
- Revisar `tpProformaAdelantoToCFDIGeneradaBO.cs` (codigo comentado)
- Revisar `CFDIGeneradaConceptoAnticipoFactory.cs`
- Revisar tablas BD: `tpProformaAdelanto`, `tpProformaAdelantoProformaPedido`, `fccPagoFacturaAdelanto`
- Documentar que es aprovechable y que debe crearse desde cero

### Resultado esperado
Documento de analisis con decision de aprovechabilidad del codigo/tablas existentes.

### Entregables
- Documento de analisis tecnico

### Criterios de aceptacion
- Se identifican claramente los componentes aprovechables vs los que requieren nuevo desarrollo
- Se valida contra el nuevo modelo de datos de BD (R16A-RE-FU-012_BD)

---

## T2 — [ R16A-RE-FU-012 ] [SERV-TRANSACT] Generacion de pendiente FAA en transaccion de tramitacion

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- Depende de T1 (revision de codigo existente)
- El INSERT del pendiente debe ser atomico con la tramitacion del pedido
- Solo aplica cuando `tpPedido.FacturaPorAdelantado = 1`
- Solo region Mexico
- **Campos fiscales regionalizados (Regla 9 — sincronización matriz):** Los datos de facturación que se fijan varían por región. Para México: RFC, RazonSocial, UsoCFDI, MetodoPago, RegimenFiscal. Para Perú (fuera de alcance R16 FAA-Crédito): RUC, RazonSocial, TipoOperacion, CondicionPago SUNAT. La Forma de Pago y el correo de envío NO se incluyen en el Panel de Información de Facturación. Ver Back.md Sección C.

### Objetivo general
Implementar la generacion del pendiente FAA dentro de la transaccion de tramitacion del pedido cuando FAA esta activa.

### Objetivos especificos
- Modificar `tpPedidoTramitarTransaccionBO.GenerarCorreoTramitarPedido()` para detectar FAA=1
- INSERT pendiente FAA con datos: Cliente, Empresa, Monto, OrdenCompra, DatosFacturacion, Region, Moneda, Estado=Pendiente
- Fijar datos de facturacion del catalogo vigente del cliente
- Asegurar atomicidad (misma transaccion que tramitacion)
- Generar confirmacion de pedido inmediatamente (no espera factura)

### Resultado esperado
Al tramitar un pedido con FAA=1 en Mexico, se genera un registro pendiente FAA con todos los datos necesarios para que el modulo de facturacion lo consuma posteriormente.

### Entregables
- Modificacion de `tpPedidoTramitarTransaccionBO.cs`
- BO/metodo para INSERT del pendiente FAA

### Criterios de aceptacion
- El pendiente se genera atomicamente con la tramitacion
- Contiene todos los datos requeridos (cliente, empresa, monto, datos facturacion, moneda)
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
- Verificar que la estructura del pendiente FAA es compatible con lo esperado por RE-FU-018
- Ajustar campos/relaciones si es necesario para la integracion
- Validar que el estado del pendiente se actualice correctamente cuando facturacion lo consume
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

## Resumen de tareas

| # | Clave Catalogo | Titulo | Predecesora |
|---|----------------|--------|-------------|
| T1 | ALG-BASIC-LOGIC | Revision de codigo existente tpProformaAdelanto para aprovechabilidad | — |
| T2 | SERV-TRANSACT | Generacion de pendiente FAA en transaccion de tramitacion | T1 |
| T3 | ALG-BASIC-LOGIC | Validaciones Back para Factura por Adelantado | T1 |
| T4 | ALG-BASIC-LOGIC | Vinculacion del pendiente FAA con modulo de facturacion (RE-FU-018/019/020) | T2 |

---

## Dependencias con otros requisitos (NO incluidas como tareas)

| Requisito | Tarea relacionada | Relacion |
|-----------|-------------------|----------|
| R16A-RE-FU-010 | T3 (Cancelacion) | Endpoint de cancelacion se desarrolla en RE-FU-010 |
| R16A-RE-FU-011 | Validacion controlados | fnEsProductoControlado se desarrolla en RE-FU-011 |
| R16A-RE-FU-018 | Generacion de factura | Consume el pendiente FAA generado en T2 |
| R16A-RE-FU-019 | Generacion de CFDI | Proceso posterior a factura |
| R16A-RE-FU-020 | Timbrado fiscal (PAC) | Proceso posterior a CFDI |



---

## T5 — [ R16A-RE-FU-012 ] [ QUERY-G ] Análisis — Migración de transferencia Pedidos Crédito de SSIS a aplicativo

### Aplicativos
- ProquifaDotNet.AplicativoLegacy / ETLs SSIS

### Módulos
- ETL — Migración transferencia Pedidos Crédito → Legacy (Análisis)

### Consideraciones previas
- La transferencia de Pedidos Crédito a Legacy actualmente se realiza mediante un paquete SSIS en PCconnect. El objetivo de esta migración es **trasladar esa lógica al aplicativo ProquifaDotNet**, siguiendo el patrón `IEtlLegacyTransferenciaService` establecido en RE-028.
- La migración debe contemplar las tres variantes del flujo de Pedido Crédito en R16: RE-010 (base + Pago contra entrega), RE-011 (Sustancias controladas, Perú sin transferencia), RE-012 (FAA paralelo).
- **Perú no transfiere a Legacy** — la condición de región debe estar en el aplicativo, no en el SSIS.
- Esta tarea analiza el SSIS existente para entender qué lógica debe migrarse y cómo mapearla al patrón de aplicativo. Es prerequisito bloqueante para T6 y T7.
- Coordinar con el equipo de arquitectura para confirmar si el canal de transferencia es el mismo que RE-028 (Brecha B3) o uno ya definido para pedidos.

### Descripción del problema
El paquete SSIS de PCconnect realiza la transferencia de Pedidos Crédito a Legacy. Al migrar al aplicativo, esta lógica debe replicarse en ProquifaDotNet como un servicio que se invoca post-commit de tramitación, incluyendo las variantes nuevas de R16 que el SSIS actual no contempla (pago contra entrega, controlados, FAA, corte Perú).

### Objetivos específicos
- Analizar el paquete SSIS existente de Pedidos Crédito: identificar qué tablas lee, qué datos envía a Legacy (Pedido, Partidas), qué SPs invoca (`spActualizarBuzonPedidoLegacy` / `spActualizarBuzonPedidoLegacyEncolar`) y bajo qué condiciones.
- Definir el modelo de migración al aplicativo:
  - Servicio/interfaz a crear (siguiendo patrón `IEtlLegacyTransferenciaService` de RE-028).
  - Builder del payload: `EtlPedidoCreditoPayloadBuilder` (o equivalente) que construya el objeto a enviar a Legacy.
  - Momento de invocación: post-commit en `tpPedidoTramitarController` (o el punto de tramitación definitivo).
- Mapear las variantes R16 en el nuevo servicio:
  - **RE-010 base:** Pedido + Partidas estándar.
  - **RE-010 Pago contra entrega:** marca de detención derivada de `catCondicionesDePago.Clave = 'pagocontraentrega'`.
  - **RE-011 Controlados (México):** incluir flag de sustancia controlada si Legacy lo requiere.
  - **RE-011 Perú:** el servicio no ejecuta si `region.Clave != MEX` — sin error, sin log de fallo.
  - **RE-012 FAA:** transferencia idéntica al flujo base (FAA es paralelo, no altera el payload del pedido).
- Confirmar el canal de transferencia definitivo (mismo B3 de RE-028, o SP directo ya funcional).
- Confirmar si el SSIS debe desactivarse tras la migración o si ambos pueden coexistir temporalmente.
- Documentar el diseño del servicio como insumo para T6.

### Resultado esperado
Documento de análisis con: lógica del SSIS existente documentada, diseño del servicio aplicativo equivalente, mapeo de variantes RE-010/011/012, canal de transferencia confirmado, plan de desactivación del SSIS.

### Entregables
- Documento de análisis de migración con:
  - Lógica del SSIS existente (tablas, columnas, SPs, condiciones)
  - Diseño del servicio aplicativo: interfaz, builder, DTO payload, punto de invocación
  - Mapeo de variantes RE-010 (base + pago contra entrega), RE-011 (controlados + Perú), RE-012 (FAA)
  - Canal de transferencia confirmado
  - Plan de coexistencia o desactivación del SSIS

### Criterios de aceptación
- [ ] La lógica del SSIS existente está documentada (tablas, SPs, condiciones).
- [ ] El diseño del servicio aplicativo equivalente está definido (interfaz, builder, DTO, punto de invocación).
- [ ] Las variantes RE-010, RE-011 y RE-012 están mapeadas en el nuevo diseño.
- [ ] El canal de transferencia definitivo está confirmado con arquitectura.
- [ ] El plan de desactivación/coexistencia del SSIS está acordado.
- [ ] El documento está aprobado como prerequisito para T6 y T7.

### Más información de la tarea
- Patrón de referencia: `IEtlLegacyTransferenciaService` + `EtlLegacyTransferenciaServiceStub` de RE-028.
- SPs Legacy a reemplazar o mantener: `spActualizarBuzonPedidoLegacy` / `spActualizarBuzonPedidoLegacyEncolar`.
- Controlador de tramitación: `WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\Liberar	pPedidoTramitarController.cs`

### Recursos
- `R16A-RE-FU-028-Back.md` — patrón `IEtlLegacyTransferenciaService`, Brecha B3
- `R16A-RE-FU-028-Tareas.md` — T17/T18/T19 (referencia del patrón de migración)
- `R16A-RE-FU-010-Back.md`, `R16A-RE-FU-011-Back.md`, `R16A-RE-FU-012-Back.md` — secciones Transferencia a Legacy
- Paquete SSIS existente de Pedidos Crédito en PCconnect

---

## T6 — [ R16A-RE-FU-012 ] [ SERV-COMPLEX-TRANSACT ] Implementación — Servicio aplicativo de transferencia Pedidos Crédito a Legacy

### Aplicativos
- ProquifaDotNet.LegacyBridge  / ETLs SSIS

### Módulos
- ETL — Migración transferencia Pedidos Crédito → Legacy (Implementación)

### Consideraciones previas
- **Predecesora: T5.** El diseño del servicio debe estar aprobado antes de implementar.
- Seguir el patrón `IEtlLegacyTransferenciaService` / `EtlLegacyTransferenciaServiceStub` establecido en RE-028: crear primero el stub (logea `ETL_PENDIENTE` via Serilog) para no bloquear el flujo de tramitación mientras se define el canal definitivo.
- El stub permite inyectar la implementación real via DI sin cambiar los callers — misma estrategia que RE-028.
- **Perú no transfiere:** el servicio debe evaluar la región antes de ejecutar cualquier lógica.

### Descripción del problema
La lógica de transferencia de Pedidos Crédito a Legacy vive en el SSIS de PCconnect. Al migrar al aplicativo, se crea un servicio en ProquifaDotNet que reemplaza esa lógica, incluyendo las variantes R16. El stub permite avanzar el desarrollo sin bloquear en el canal definitivo.

### Objetivos específicos
- Crear la interfaz `IEtlPedidoCreditoLegacyService` con el método `EnviarAsync(tpPedido pedido, Region region)` (o los parámetros que determine el análisis de T5).
- Crear `EtlPedidoCreditoLegacyServiceStub` que implemente la interfaz y registre `ETL_PENDIENTE` via Serilog con contexto (IdPedido, Folio, Región, FechaIntento).
- Crear `EtlPedidoCreditoPayloadBuilder` que construya el objeto a enviar a Legacy:
  - Campos base: Pedido (folio, cliente, montos), Partidas (producto, piezas, precios).
  - Variante Pago contra entrega: incluir marca de detención si `catCondicionesDePago.Clave = 'pagocontraentrega'`.
  - **OBS-024:** PCE (`catCondicionesDePago.Clave = 'pagocontraentrega'`) se traduce como **crédito** en el payload de Legacy, **NO como prepago**. Legacy procesa PCE como flujo de crédito independientemente del nombre de la condición.
  - Variante Controlados (RE-011): incluir flag si Legacy lo requiere según análisis T5.
  - Variante FAA (RE-012): payload idéntico al base (sin campos adicionales, FAA es paralelo).
- **OBS-025:** PQF2 solo inserta datos planos en Legacy. La lógica de "Relacionar facturas" (asociación de facturas al pedido dentro de Legacy) es responsabilidad del proceso interno de Legacy — **el `EtlPedidoCreditoPayloadBuilder` no implementa esta lógica**.
- Registrar el corte regional en el servicio: si `region.Clave != MEX`, retornar sin ejecutar (sin error, sin log de fallo innecesario).
- Invocar el servicio post-commit en el punto de tramitación definido en T5 (`tpPedidoTramitarController` o equivalente), usando inyección de dependencias.
- Registrar el stub en el contenedor de DI del proyecto.
- Pruebas unitarias del builder para cada variante (base, pago contra entrega, controlados, FAA) y del corte de región.

### Resultado esperado
El servicio stub está integrado en el flujo de tramitación de Pedidos Crédito. El builder construye el payload correcto para cada variante. El flujo de tramitación no se bloquea por la transferencia.

### Entregables
- `IEtlPedidoCreditoLegacyService` — interfaz
- `EtlPedidoCreditoLegacyServiceStub` — stub con log `ETL_PENDIENTE`
- `EtlPedidoCreditoPayloadBuilder` — builder con variantes RE-010/011/012
- `EtlPedidoCreditoPayload` — DTO del payload
- Registro DI del stub en el proyecto
- Invocación post-commit en el controlador de tramitación
- Pruebas unitarias del builder (por variante) y del corte de región

### Criterios de aceptación
- [ ] El servicio stub se invoca post-commit al tramitar un Pedido Crédito (México y Perú).
- [ ] Para Perú: el servicio retorna sin ejecutar — sin error, sin log de fallo.
- [ ] El builder genera el payload correcto para el flujo base (RE-010).
- [ ] El builder incluye la marca de detención para Pago contra entrega (RE-010).
- [ ] **OBS-024:** El builder traduce PCE (`catCondicionesDePago.Clave = 'pagocontraentrega'`) como **crédito** en Legacy, no como prepago.
- [ ] **OBS-025:** El builder no implementa lógica de "Relacionar facturas" — esa responsabilidad recae en Legacy.
- [ ] El builder contempla la variante de controlados (RE-011) según el análisis T5.
- [ ] El builder genera el payload FAA idéntico al base (RE-012).
- [ ] El stub logea `ETL_PENDIENTE` con contexto suficiente (IdPedido, Folio, Región).
- [ ] Pruebas unitarias del builder aprobadas para todas las variantes.
- [ ] PR aprobado por líder técnico.

### Más información de la tarea
- Patrón de referencia: RE-028 T18 (`EtlBuzonCobrosPayloadBuilder`, `IEtlBuzonCobrosLegacyService`, `EtlLegacyTransferenciaServiceStub`).
- El stub se reemplaza por la implementación real en T7 via DI, sin modificar el caller.

### Recursos
- `R16A-RE-FU-028-Tareas.md` — T18 (referencia del patrón builder + stub)
- Documento de análisis de T5
- `tpPedidoTramitarController.cs` — punto de invocación post-commit
- `R16A-RE-FU-010-Back.md`, `R16A-RE-FU-011-Back.md`, `R16A-RE-FU-012-Back.md`

---

## T7 — [ R16A-RE-FU-012 ] [ SERV-TRANSACT ] Integración — Canal definitivo y desactivación del SSIS de Pedidos Crédito

### Aplicativos
- ProquifaDotNet.LegacyBridge  / ETLs SSIS / ProquifaDotNet / PCconnect

### Módulos
- ETL — Migración transferencia Pedidos Crédito → Legacy (Integración y cierre)

### Consideraciones previas
- **Predecesoras: T5 y T6.** El stub debe estar integrado y el canal definitivo confirmado antes de esta tarea.
- Esta tarea reemplaza el stub por la implementación real del canal (misma Brecha B3 de RE-028, o SP directo si ya está resuelto).
- La implementación real se inyecta via DI sin cambiar el caller — esa es la ventaja del patrón stub.
- Una vez validada la transferencia via aplicativo, el paquete SSIS de PCconnect debe desactivarse o retirarse según el plan acordado en T5.
- **Perú no transfiere:** validar en pruebas que ningún pedido de región Perú genera registros en Legacy.
- **OBS-025:** La integración E2E debe confirmar que "Relacionar facturas" **no es responsabilidad del aplicativo ProquifaDotNet** — es un proceso interno de Legacy. No replicar ni validar esta lógica en el canal real.

### Descripción del problema
Con el stub funcionando en tramitación, esta tarea cierra la migración: implementa el canal real de transferencia, valida E2E en QA/staging contra Legacy real, y desactiva el SSIS de PCconnect para eliminar la fuente doble.

### Objetivos específicos
- Implementar `EtlPedidoCreditoLegacyService` (implementación real) usando el canal definitivo confirmado en T5 (SP directo, RabbitMQ, o API Legacy).
- Reemplazar el stub por la implementación real en el registro DI — sin cambios al controlador ni al builder.
- Ejecutar pruebas de integración E2E en QA/staging cubriendo:
  - **RE-010 base:** Pedido + Partidas → Legacy sin errores.
  - **RE-010 Pago contra entrega:** marca de detención presente en Legacy.
  - **RE-011 Controlados (México):** transferencia correcta a Legacy.
  - **RE-011 Perú:** ningún registro llega a Legacy.
  - **RE-012 FAA:** transferencia idéntica al flujo base.
- Comparar directamente ProquifaDotNet ↔ Legacy: Pedido (folio, cliente, montos), Partidas (producto, piezas, precios), marca de detención (si aplica).
- Verificar no regresión del flujo base (pedidos anteriores a R16).
- Coordinar con el equipo PCconnect/Legacy la desactivación del paquete SSIS una vez validada la transferencia por aplicativo.
- Documentar evidencias, incidencias, resolución y el cierre de la migración.

### Resultado esperado
La transferencia de Pedidos Crédito a Legacy se realiza íntegramente desde el aplicativo ProquifaDotNet para las variantes RE-010/011/012. El SSIS de PCconnect está desactivado o retirado. Las evidencias están documentadas y aprobadas para producción.

### Entregables
- `EtlPedidoCreditoLegacyService` — implementación real del canal
- Registro DI actualizado (implementación real reemplaza stub)
- Documento de resultados de integración:
  - Escenarios ejecutados (RE-010 base, pago contra entrega, RE-011 controlados, RE-011 Perú, RE-012 FAA)
  - Comparación ProquifaDotNet ↔ Legacy por variante
  - Verificación de corte Perú (sin registros en Legacy)
  - No regresión del flujo base
  - Incidencias y resolución
- Evidencias de validación (queries o capturas por escenario)
- Confirmación de desactivación del SSIS en PCconnect
- Aprobación para paso a producción

### Criterios de aceptación
- [ ] La implementación real del canal sustituye el stub via DI — sin cambios al controlador ni al builder.
- [ ] RE-010 base: Pedido y Partidas en Legacy con valores correctos.
- [ ] RE-010 Pago contra entrega: marca de detención presente en Legacy.
- [ ] RE-011 Controlados (México): transferencia correcta a Legacy.
- [ ] RE-011 Perú: ningún registro de región Perú en Legacy.
- [ ] RE-012 FAA: transferencia idéntica al flujo base.
- [ ] Flujo base (pedidos previos a R16) sin regresión.
- [ ] El paquete SSIS de PCconnect está desactivado o retirado según el plan.
- [ ] Las pruebas están documentadas con evidencias y aprobadas por el líder técnico.
- [ ] Aprobado para paso a producción.

### Más información de la tarea
Esta tarea cierra la migración completa del ETL de Pedidos Crédito — de SSIS a aplicativo — para las variantes RE-010, RE-011 y RE-012. La Brecha B3 (canal de transferencia) debe estar resuelta antes de implementar el canal real.

### Recursos
- `R16A-RE-FU-028-Tareas.md` — T19 (referencia del patrón integración canal real + sustitución stub)
- Documento de análisis de T5
- Implementación stub de T6
- Acceso a Legacy en ambiente QA/staging
- Equipo PCconnect para desactivación del SSIS
