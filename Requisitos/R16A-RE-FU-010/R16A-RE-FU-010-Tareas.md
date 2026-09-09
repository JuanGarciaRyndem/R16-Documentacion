# Tareas BackEnd — R16A-RE-FU-010
**Requisito:** Tramitacion de pedidos Credito
**Modulo:** L05.TramitarPedido

---

## Tarea 1

**Titulo:** [ R16A-RE-FU-010 ] [IMP-EXIST-SERVICE] Verificacion del flujo de tramitacion Credito existente

**Aplicativos:** ProquifaDotNet

**Modulos:** Logic.Pqf.Logistica\L05.TramitarPedido\Liberar

**Ticket:** R16A-1478 (corregido 2026-09-09: T1 y T2 se agrupan en R16A-1478; T3 y T4 en R16A-1477 — asi empezo Sebas la organizacion, agrupando por tema: verificacion credito vs. cancelacion/cierre de pendiente)

**Estado (actualizado 2026-09-08):** Codigo construido, sin Pull Request abierto. Rama `feature/r16-phase-02-R16A-RE-FU-010`, commit `c59d51f12` ("feat(R16A-RE-FU-010): flujo por condicion de pago y ETL por region"), publicada en origin sobre base `project/r16-phase-02`. Falta abrir el PR y levantar la evidencia (T1 es un requisito de verificacion; el codigo que sostiene la verificacion ya existe: `CondicionPagoFlujoHelper` + su prueba unitaria sin BD).

**Consideraciones previas:**
- El flujo de tramitacion Credito ya esta implementado en `tpPedidoTramitarTransaccionBO.GenerarCorreoTramitarPedido()`
- El endpoint ya existe en `tpPedidoTramitarController.cs`
- Desde el commit `c59d51f12`, la decision `SinCredito` ya no se lee directamente en el BO: pasa por `Logic.Pqf.Logistica\L05.TramitarPedido\Helpers\CondicionPagoFlujoHelper.cs` (nuevo, con prueba unitaria propia sin BD)

**Objetivo general:**
Verificar que el flujo preexistente de tramitacion de pedidos Credito (sin sustancias controladas, sin Factura por Adelantado) funciona correctamente para clientes con condicion de pago Credito en Mexico y Peru.

**Objetivos especificos:**
1. Validar que el flujo genera correctamente el folio interno de pedido
2. Validar que se genera el PDF de Confirmacion de Pedido
3. Validar que se crean los pendientes de compra (`ocPendienteCompraProducto`)
4. Validar que se envia el correo de confirmacion via RabbitMQ/SendInBlue
5. Validar que se actualiza `tpPedido.Tramitado = 1`

**Resultado esperado:**
El flujo de tramitacion Credito ejecuta correctamente todos los pasos: folio, PDF, partidas, pendientes de compra, correo y commit.

**Entregables:**
- Evidencia de pruebas del flujo completo en ambiente de desarrollo
- Documentacion de casos de prueba
- **Pendiente:** abrir el Pull Request de la rama `feature/r16-phase-02-R16A-RE-FU-010` (publicada desde el 5 de septiembre, sin PR)

**Criterios de aceptacion:**
- El pedido se tramita exitosamente con folio generado
- El PDF se sube a MinIO correctamente
- El correo se envia al cliente
- Se crean registros en `ocPendienteCompraProducto`, `tpPedidoCorreoEnviado`, `tpPartidaPedidoSeguimiento`

**Mas informacion de la tarea:**
- Clase: `tpPedidoTramitarTransaccionBO.cs`
- Endpoint: `tpPedidoTramitarController.cs`
- Helper nuevo: `CondicionPagoFlujoHelper.cs`
- Criterio de aceptacion del requisito: A1

**Recursos:**
- R16A-RE-FU-010.md
- R16A-RE-FU-010-Back.md (Seccion A)

---

## Tarea 2

**Titulo:** [ R16A-RE-FU-010 ] [IMP-EXIST-SERVICE] Verificacion de variante Pago contra entrega con marca de detencion en Legacy

**Aplicativos:** ProquifaDotNet

**Modulos:** Logic.Pqf.Logistica\L05.TramitarPedido\Liberar

**Ticket:** R16A-1478 (corregido 2026-09-09: agrupada con T1 en R16A-1478)

**Estado (actualizado 2026-09-08):** Misma rama y commit que Tarea 1 (`c59d51f12`), sin PR abierto — pero ese commit solo cubre el lado ProquifaDotNet (que el pedido se tramite como Credito via `CondicionPagoFlujoHelper`, y el condicional `region.ProcesarEnETL` alrededor del ETL de buzon a Legacy para Regla 5 / CA-1: Peru no transfiere). El SP de transferencia en si (`spActualizarBuzonPedidoLegacy`) **no fue modificado**.

**La transferencia real a Legacy (flujo de datos + marca de detencion) no vive en ProquifaDotNet: vive en el repositorio SSIS `interfaces-proquifanet2`** (paquetes `Pedidos.dtsx` / `PedidosDatos.dtsx`). Se reviso ese repositorio (2026-09-08): existe una rama creada para R16 (`project/r16-pedidos-sin-credito`), pero **no tiene ningun commit propio** — su contenido es identico al punto donde se separo de `main`, y no hay ninguna referencia a "Contra Entrega" en todo el proyecto SSIS. **Esta parte de la tarea (traducir Contra Entrega a flujo de Credito en la transferencia, y la marca de detencion) sigue pendiente de construir.**

**Consideraciones previas:**
- Los pedidos con condicion Credito - Pago contra entrega se procesan con el mismo flujo Credito, ahora via `CondicionPagoFlujoHelper.EsCondicionCredito()`
- La marca de detencion debe transferirse a Legacy para que detenga el pedido en fase de entrega
- SP relacionados: `spActualizarBuzonPedidoLegacy`, `spActualizarBuzonPedidoLegacyEncolar`
- El ETL hacia Legacy ahora se omite completo para regiones con `Region.ProcesarEnETL = false` (Peru)
- **Catalogo `catCondicionesDePago`:** Pago Contra Entrega (`B104EAFE-7ECE-434B-B6CE-3B63448422FE`, clave `PAGO CONTRA ENTREGA`) tiene `SinCredito = 1` — el mismo valor que Prepago 100% (`665027FC-E87C-42C5-AB53-28D5CE25ED04`). A nivel de este catalogo, hoy Contra Entrega y Prepago son indistinguibles.
- **Confirmado en `interfaces-proquifanet2`:** los pasos de la ETL de credito en `PedidosDatos.dtsx` (16 puntos, entre ellos la tarea "Pedidos Nuevos Pendientes Credito") filtran explicitamente `catcp.SinCredito = 0` — hoy excluyen a Contra Entrega, que sigue el mismo tratamiento que Prepago (consistente con la nota ya conocida: "hoy estos pedidos operan como prepago; en R16 deben traducirse a Credito").
- Para que Contra Entrega se transfiera como Credito hay que actualizar ese filtro (o la logica que separa los flujos) en `interfaces-proquifanet2`; a la fecha no se encontro ese cambio en ninguna rama del repo.
- La marca de detencion en si tampoco esta implementada ahi — no aparece ninguna referencia a ella en todo el proyecto SSIS (bloqueante **P1**, ver `R16A-RE-FU-010-DIS-SOL-Back.md` §1.4).

**Objetivo general:**
Verificar que la variante Pago contra entrega genera correctamente la marca de detencion en la transferencia a Legacy.

**Objetivos especificos:**
1. Identificar como se determina Pago contra entrega (`CondicionPagoFlujoHelper.EsPagoContraEntrega`, clave `pagocontraentrega`)
2. Validar que la transferencia a Legacy incluye la marca de detencion
3. Verificar que Legacy recibe y procesa correctamente la marca
4. **Nuevo:** confirmar que Peru (region con `ProcesarEnETL=false`) efectivamente omite el ETL y queda rastro en log
5. **Nuevo — pendiente de construir:** actualizar la ETL de credito en `interfaces-proquifanet2` (`PedidosDatos.dtsx`) para que incluya a Contra Entrega (hoy excluido por el filtro `catcp.SinCredito = 0`)
6. **Nuevo — pendiente de construir:** definir e implementar la marca de detencion hacia Legacy (bloqueante P1, sin resolver a la fecha)

**Resultado esperado:**
Al tramitar un pedido con condicion Pago contra entrega, el sistema transfiere al Legacy la informacion con la marca de detencion que impide la entrega hasta validar el pago. Para Peru, el ETL se omite.

**Entregables:**
- Evidencia de transferencia a Legacy con marca de detencion
- Documentacion del flujo de datos hacia Legacy
- **Pendiente:** abrir el Pull Request de la rama `feature/r16-phase-02-R16A-RE-FU-010`

**Criterios de aceptacion:**
- El pedido Pago contra entrega se tramita como Credito normal
- La transferencia a Legacy incluye la marca de detencion
- Legacy detiene el pedido hasta validacion de pago
- Peru no ejecuta el ETL de buzon (verificado en log)

**Mas informacion de la tarea:**
- SP: `spActualizarBuzonPedidoLegacy`
- Tabla: `catCondicionesDePago` (SinCredito=1, Clave='pagocontraentrega')
- Helper: `CondicionPagoFlujoHelper.cs`
- **Repositorio adicional (SSIS):** `interfaces-proquifanet2`, paquetes `Pedidos.dtsx` y `PedidosDatos.dtsx`
- **Rama creada para R16 en ese repo:** `project/r16-pedidos-sin-credito` (sin commits propios a la fecha)
- Criterios de aceptacion del requisito: A2, A3

**Recursos:**
- R16A-RE-FU-010.md
- R16A-RE-FU-010_BD.md
- R16A-RE-FU-010-Back.md (Seccion Transferencia a Legacy)
- `R16A-RE-FU-010-DIS-SOL-Back.md` (§1.4, bloqueante P1)

---

## Tarea 3

**Titulo:** [ R16A-RE-FU-010 ] [SERV-TRANSACT] Crear endpoint de Cancelacion de pedido desde Tramitar Pedido

**Aplicativos:** ProquifaDotNet

**Modulos:** WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\Cancelacion, Logic.Pqf.Logistica\L04.PretramitarPedido\GestionarIntramitables

**Ticket:** R16A-1477 (corregido 2026-09-09: T3 y T4 se agrupan en R16A-1477; T1 y T2 en R16A-1478 — asi empezo Sebas la organizacion, agrupando por tema: verificacion credito vs. cancelacion/cierre de pendiente)

**Estado (actualizado 2026-09-08): CONSTRUIDO Y PROBADO.** Viaja en el Pull Request [#269](https://github.com/ryndem/ProquifaDotNet/pull/269) de ProquifaDotNet, junto con R16A-RE-FU-015 (T7), por la dependencia con el catalogo `catMotivoCancelacion` que esa tarea entrega. La desviacion queda declarada en el cuerpo del PR.

**Consideraciones previas:**
- Actualmente existe `[HttpDelete] tpPedido` que ejecuta `Desactivar()` (solo marca Activo=false) — sigue existiendo, sin cambios, en paralelo al endpoint nuevo
- La logica de cancelacion completa **ya estaba** en `ppPedidoCancelacionBO.CancelacionOrdenDeCompra(Guid)` (usada por Gestionar Intramitables) — se le agrego un **overload** de 3 parametros en vez de crear un BO nuevo
- El endpoint debe ser invocado tras confirmacion explicita del ESAC (modal en Front) — el endpoint no valida esto, es responsabilidad del Front

**Objetivo general:**
Crear un nuevo endpoint de cancelacion de pedido en el modulo Tramitar Pedido que ejecute el flujo transaccional completo: validaciones, cancelacion de pedido y partidas, inactivacion de tpPedido, y registro en bitacora con motivo.

**Lo que realmente se construyo (vs. lo planeado):**

| Planeado | Construido |
|---|---|
| Ruta `[HttpPut] tpPedido/cancelar` | Ruta real: `[HttpPut] PUT /api/tpPedido/Cancelar` |
| BO nuevo `tpPedidoCancelacionBO.cs` | **No se creo BO nuevo** — overload `CancelacionOrdenDeCompra(idppPedido, motivoCancelacionClave, inactivarPedidoTramitado)` sobre `ppPedidoCancelacionBO` existente; el overload de 1 parametro sigue igual para Gestionar Intramitables |
| Recibe solo `IdPPPedido` | Recibe `IdPPPedido` **y** `MotivoCancelacionClave` (modelo `GMtpPedidoCancelacion`) — el motivo se valida contra `catMotivoCancelacion` (catalogo nuevo de FU-015 T7) |
| — | Se corrigieron 2 defectos previos en el codigo compartido (pedido inexistente producia `NullReferenceException`; Id vacio no se validaba antes de abrir el contexto) |
| — | Se detecto y **no** se corrigio un 3er defecto: `Rollback()` sobre transaccion ya desechada por el `using`, que enmascara la excepcion original |

**Objetivos especificos (estado real):**
1. ~~Crear controller `tpPedidoCancelacionController.cs`~~ -> **Hecho**, con ruta `tpPedido/Cancelar` (no `tpPedido/cancelar`)
2. ~~Crear BO `tpPedidoCancelacionBO.cs`~~ -> **No se creo**; se reutilizo `ppPedidoCancelacionBO` via overload (mas simple, menos codigo duplicado)
3. Implementar flujo transaccional -> **Hecho**, incluye ademas el motivo de cancelacion (no estaba en el plan original de este documento)
4. Retornar Response con resultado de la operacion -> **Hecho**, con mapeo HTTP 200/400/409

**Resultado esperado:**
Endpoint `PUT /api/tpPedido/Cancelar` que recibe `IdPPPedido` y `MotivoCancelacionClave`, y ejecuta la cancelacion completa del pedido con todas sus validaciones y registros en bitacora.

**Entregables:**
- `WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\Cancelacion\tpPedidoCancelacionController.cs` — construido
- `Logic.Pqf.Logistica\L05.TramitarPedido\Models\GMtpPedidoCancelacion.cs` — construido (modelo de solicitud)
- Overload en `ppPedidoCancelacionBO.cs` — construido
- Pruebas del flujo de cancelacion — construidas (automatizadas + HTTP real sin token)
- **Pendiente:** prueba end-to-end con token valido (BitacoraCRUD exige `IdUsuario` del usuario autenticado)
- **Pendiente:** decision sobre el defecto de Rollback enmascarado (codigo compartido con Gestionar Intramitables, se deja aparte a proposito)

**Criterios de aceptacion:**
- El endpoint responde correctamente con Response (200/400/409)
- Valida que el pedido no este previamente cancelado (409)
- Valida que no tenga dependencia de parcialidad en tpPedido (409)
- Valida la clave de motivo contra `catMotivoCancelacion` (400 si no existe)
- Marca `ppPedido.Cancelada = true` y `ppPartidaPedido.Cancelada = true`
- Inactiva el `tpPedido` asociado (si existe)
- Registra en `BitacoraCRUD` la accion de cancelacion (1 registro por pedido + 1 por partida), con el motivo en el detalle
- El pedido desaparece de la bandeja de Tramitar Pedido (vTramitarPedido filtra Activo=1)
- La operacion es transaccional (rollback en caso de error) — **con la salvedad del defecto #3 no corregido**

**Mas informacion de la tarea:**
- Referencia de logica existente: `ppPedidoCancelacionBO.CancelacionOrdenDeCompra()`
- Entidades escritura: `ppPedido`, `ppPartidaPedido`, `tpPedido`, `BitacoraCRUD`
- Entidades lectura: `catBitacoraAccion`, `catMotivoCancelacion`, `ContactoCliente`, `Cliente`, `Producto`
- Criterio de aceptacion del requisito: B1

**Recursos:**
- R16A-RE-FU-010.md (Seccion B - Cancelacion)
- R16A-RE-FU-010-Back.md (Seccion B)
- `Logic.Pqf.Logistica\L04.PretramitarPedido\GestionarIntramitables\ppPedidoCancelacionBO.cs`
- PR [#269](https://github.com/ryndem/ProquifaDotNet/pull/269) (ProquifaDotNet), compartido con R16A-RE-FU-015

---

## Tarea 4

**Titulo:** [ R16A-RE-FU-010 ] [IMP-EXIST-SERVICE] Verificacion del cierre de pendiente en bandeja de Tramitar Pedido

**Aplicativos:** ProquifaDotNet

**Modulos:** Logic.Pqf.Logistica\L05.TramitarPedido

**Ticket:** R16A-1477 (agrupada con T3)

**Estado (actualizado 2026-09-08): VERIFICADO POR CONSTRUCCION, sin desarrollo adicional.** No existe ninguna entidad de "pendiente" que cerrar. La bandeja de Tramitar Pedido es la vista `vTramitarPedido`, y su definicion filtra por `tpPedido.Activo = 1`. Tanto la tramitacion exitosa (Regla 4, `Tramitado=1`) como la cancelacion (T3, `tpPedido.Activo=false`) retiran al pedido de esa vista dentro de su propia transaccion — no hay un paso 10 separado de "cerrar pendiente": es la misma accion que inactivar/marcar el pedido.

**Consideraciones previas:**
- Regla 4 del requisito: al completar la tramitacion exitosa, se debe cerrar y eliminar el pendiente del pedido en la bandeja
- Esto aplica tanto para tramitacion exitosa como para cancelacion
- La bandeja es la vista `vTramitarPedido` (`WHERE tpPedido.Activo = 1`), no una tabla de pendientes separada

**Objetivo general:**
Verificar que tras la tramitacion exitosa o la cancelacion del pedido, el pendiente se cierra/elimina de la bandeja de Tramitar Pedido y el pedido ya no aparece como accion pendiente para el ESAC.

**Objetivos especificos:**
1. ~~Identificar la tabla/entidad que gestiona los pendientes de la bandeja~~ -> **Resuelto**: es la vista `vTramitarPedido`, filtrada por `tpPedido.Activo = 1`
2. Verificar que el flujo de tramitacion Credito cierra el pendiente al finalizar -> pendiente de evidencia (junto con T1)
3. ~~Verificar que el nuevo flujo de cancelacion (Tarea 3) tambien cierra el pendiente~~ -> **Verificado por construccion**: el paso de inactivar `tpPedido` en T3 es exactamente esto
4. Validar que el pedido no aparece en la bandeja tras ambas acciones -> pendiente de evidencia end-to-end

**Resultado esperado:**
Tras tramitar o cancelar un pedido, el registro se marca (Tramitado=1 o Activo=false) y el pedido desaparece de `vTramitarPedido`.

**Entregables:**
- Evidencia de que el pendiente se cierra en ambos flujos (pendiente, junto con la evidencia end-to-end de T1/T3)
- ~~Ajuste al BO de cancelacion si el cierre no esta incluido~~ -> no aplico, ya esta incluido

**Criterios de aceptacion:**
- Al tramitar exitosamente, el pedido desaparece de la bandeja
- Al cancelar, el pedido desaparece de la bandeja
- No quedan registros huerfanos de pendientes

**Mas informacion de la tarea:**
- Regla de negocio 4 del requisito
- Criterio de aceptacion del requisito: A1, B1
- Vista: `vTramitarPedido`

**Recursos:**
- R16A-RE-FU-010.md (Regla 4)
- R16A-RE-FU-010-Back.md (Seccion B)

---

## Resumen de Tareas

| # | Clave Catalogo | Ticket | Descripcion | Estado (2026-09-08) |
|---|----------------|--------|-------------|--------|
| T1 | IMP-EXIST-SERVICE | R16A-1478 | Verificacion flujo tramitacion Credito existente | Codigo listo, rama sin PR, falta evidencia |
| T2 | IMP-EXIST-SERVICE | R16A-1478 | Verificacion variante Pago contra entrega + marca detencion Legacy | Lado ProquifaDotNet listo (rama sin PR); lado SSIS/Legacy (traducir Contra Entrega a flujo Credito + marca de detencion) **pendiente de construir**, sin cambios en `interfaces-proquifanet2` a la fecha |
| T3 | SERV-TRANSACT | R16A-1477 | Crear endpoint de Cancelacion de pedido | Construido y probado (PR #269); falta prueba end-to-end con token |
| T4 | IMP-EXIST-SERVICE | R16A-1477 | Verificacion cierre de pendiente en bandeja | Verificado por construccion |

**Nota (corregido 2026-09-09):** la agrupacion por ticket de Jira es: **R16A-1477** = T3 + T4 (cancelacion y cierre de pendiente); **R16A-1478** = T1 + T2 (verificacion del flujo credito). Asi es como Sebas empezo a organizar el trabajo (por tema), no por el orden T1-T4 del documento.

## Punto abierto para seguimiento

**Relacion con `pedidoEstadoActual` (R16A-RE-FU-015):** el endpoint de Cancelacion de este requisito (T3) no actualiza la tabla `pedidoEstadoActual` que introduce R16A-RE-FU-015 (T7/T8). Ese requisito tiene su propio endpoint generico `PUT /api/orders/status`, que segun lo construido si acepta un estado `cancelado` con motivo obligatorio. Falta decidir si la cancelacion de este requisito debe tambien reportar a `pedidoEstadoActual`, para que el estatus consolidado del pedido no quede desactualizado tras una cancelacion via Tramitar Pedido. Ver R16A-RE-FU-010-Back.md y R16A-RE-FU-010_BD.md (Gap #4).

**Repositorio y ciclo de vida de la transferencia a Legacy (Tarea 2):** hoy la logica pendiente de Tarea 2 (traducir Contra Entrega a flujo de Credito en la transferencia + marca de detencion) le corresponderia construirse en el repositorio SSIS `interfaces-proquifanet2` (paquetes `Pedidos.dtsx`/`PedidosDatos.dtsx`), que es donde vive hoy toda la logica de transferencia a Legacy. Sin embargo, segun R16A-RE-FU-015 esta logica de integracion con Legacy se va a migrar a un repositorio nuevo, `LegacySync`. Falta confirmar con el equipo de FU-015 la secuencia entre ambos requisitos antes de iniciar el desarrollo en el SSIS actual, para no construir algo que de inmediato haya que volver a migrar.
