# Tareas BackEnd — R16A-RE-FU-010
**Requisito:** Tramitacion de pedidos Credito
**Modulo:** L05.TramitarPedido

---

## Tarea 1

**Titulo:** [ R16A-RE-FU-010 ] [IMP-EXIST-SERVICE] Verificacion del flujo de tramitacion Credito existente

**Aplicativos:** ProquifaDotNet

**Modulos:** Logic.Pqf.Logistica\L05.TramitarPedido\Liberar

**Consideraciones previas:**
- El flujo de tramitacion Credito ya esta implementado en `tpPedidoTramitarTransaccionBO.GenerarCorreoTramitarPedido()`
- El endpoint ya existe en `tpPedidoTramitarController.cs`

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

**Criterios de aceptacion:**
- El pedido se tramita exitosamente con folio generado
- El PDF se sube a MinIO correctamente
- El correo se envia al cliente
- Se crean registros en `ocPendienteCompraProducto`, `tpPedidoCorreoEnviado`, `tpPartidaPedidoSeguimiento`

**Mas informacion de la tarea:**
- Clase: `tpPedidoTramitarTransaccionBO.cs`
- Endpoint: `tpPedidoTramitarController.cs`
- Criterio de aceptacion del requisito: A1

**Recursos:**
- R16A-RE-FU-010.md
- R16A-RE-FU-010-Back.md (Seccion A)

---

## Tarea 2

**Titulo:** [ R16A-RE-FU-010 ] [IMP-EXIST-SERVICE] Verificacion de variante Pago contra entrega con marca de detencion en Legacy

**Aplicativos:** ProquifaDotNet

**Modulos:** Logic.Pqf.Logistica\L05.TramitarPedido\Liberar

**Consideraciones previas:**
- Los pedidos con condicion Credito - Pago contra entrega se procesan con el mismo flujo Credito
- La marca de detencion debe transferirse a Legacy para que detenga el pedido en fase de entrega
- SP relacionados: `spActualizarBuzonPedidoLegacy`, `spActualizarBuzonPedidoLegacyEncolar`

**Objetivo general:**
Verificar que la variante Pago contra entrega genera correctamente la marca de detencion en la transferencia a Legacy.

**Objetivos especificos:**
1. Identificar como se determina Pago contra entrega (`catCondicionesDePago.Clave = 'pagocontraentrega'`)
2. Validar que la transferencia a Legacy incluye la marca de detencion
3. Verificar que Legacy recibe y procesa correctamente la marca

**Resultado esperado:**
Al tramitar un pedido con condicion Pago contra entrega, el sistema transfiere al Legacy la informacion con la marca de detencion que impide la entrega hasta validar el pago.

**Entregables:**
- Evidencia de transferencia a Legacy con marca de detencion
- Documentacion del flujo de datos hacia Legacy

**Criterios de aceptacion:**
- El pedido Pago contra entrega se tramita como Credito normal
- La transferencia a Legacy incluye la marca de detencion
- Legacy detiene el pedido hasta validacion de pago

**Mas informacion de la tarea:**
- SP: `spActualizarBuzonPedidoLegacy`
- Tabla: `catCondicionesDePago` (SinCredito=1, Clave='pagocontraentrega')
- Criterios de aceptacion del requisito: A2, A3

**Recursos:**
- R16A-RE-FU-010.md
- R16A-RE-FU-010_BD.md
- R16A-RE-FU-010-Back.md (Seccion Transferencia a Legacy)

---

## Tarea 3

**Titulo:** [ R16A-RE-FU-010 ] [SERV-TRANSACT] Crear endpoint de Cancelacion de pedido desde Tramitar Pedido

**Aplicativos:** ProquifaDotNet

**Modulos:** WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido, Logic.Pqf.Logistica\L05.TramitarPedido

**Consideraciones previas:**
- Actualmente solo existe `[HttpDelete] tpPedido` que ejecuta `Desactivar()` (solo marca Activo=false)
- La logica de cancelacion completa ya existe en `ppPedidoCancelacionBO.CancelacionOrdenDeCompra()`
- Se necesita un endpoint especifico que ejecute el flujo completo de cancelacion
- El endpoint debe ser invocado tras confirmacion explicita del ESAC (modal en Front)

**Objetivo general:**
Crear un nuevo endpoint de cancelacion de pedido en el modulo Tramitar Pedido que ejecute el flujo transaccional completo: validaciones, cancelacion de pedido y partidas, inactivacion de tpPedido, y registro en bitacora.

**Objetivos especificos:**
1. Crear controller `tpPedidoCancelacionController.cs` (o agregar al existente) con ruta `tpPedido/cancelar`
2. Crear BO `tpPedidoCancelacionBO.cs` que reutilice/extienda la logica de `ppPedidoCancelacionBO`
3. Implementar flujo transaccional:
   - Validar existencia y estado del pedido
   - Marcar `ppPedido.Cancelada = true`
   - Marcar cada `ppPartidaPedido.Cancelada = true`
   - Inactivar `tpPedido` asociado (Activo=false)
   - Registrar en `BitacoraCRUD` (pedido + partidas)
   - Eliminar/cerrar pendiente de bandeja Tramitar Pedido
4. Retornar Response con resultado de la operacion

**Resultado esperado:**
Nuevo endpoint `[HttpPut] tpPedido/cancelar` que recibe `IdPPPedido` y ejecuta la cancelacion completa del pedido con todas sus validaciones y registros en bitacora.

**Entregables:**
- `WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\tpPedidoCancelacionController.cs`
- `Logic.Pqf.Logistica\L05.TramitarPedido\Cancelacion\tpPedidoCancelacionBO.cs`
- Pruebas unitarias del flujo de cancelacion

**Criterios de aceptacion:**
- El endpoint responde correctamente con Response (true/false)
- Valida que el pedido no este previamente cancelado
- Valida que no tenga dependencia de parcialidad en tpPedido
- Marca `ppPedido.Cancelada = true` y `ppPartidaPedido.Cancelada = true`
- Inactiva el `tpPedido` asociado
- Registra en `BitacoraCRUD` la accion de cancelacion (1 registro por pedido + 1 por partida)
- El pedido desaparece de la bandeja de Tramitar Pedido
- La operacion es transaccional (rollback en caso de error)

**Mas informacion de la tarea:**
- Referencia de logica existente: `ppPedidoCancelacionBO.CancelacionOrdenDeCompra()`
- Entidades escritura: `ppPedido`, `ppPartidaPedido`, `tpPedido`, `BitacoraCRUD`
- Entidades lectura: `catBitacoraAccion`, `ContactoCliente`, `Cliente`, `Producto`
- Criterio de aceptacion del requisito: B1

**Recursos:**
- R16A-RE-FU-010.md (Seccion B - Cancelacion)
- R16A-RE-FU-010-Back.md (Seccion B)
- `Logic.Pqf.Logistica\L04.PretramitarPedido\GestionarIntramitables\ppPedidoCancelacionBO.cs`

---

## Tarea 4

**Titulo:** [ R16A-RE-FU-010 ] [IMP-EXIST-SERVICE] Verificacion del cierre de pendiente en bandeja de Tramitar Pedido

**Aplicativos:** ProquifaDotNet

**Modulos:** Logic.Pqf.Logistica\L05.TramitarPedido

**Consideraciones previas:**
- Regla 4 del requisito: al completar la tramitacion exitosa, se debe cerrar y eliminar el pendiente del pedido en la bandeja
- Esto aplica tanto para tramitacion exitosa como para cancelacion

**Objetivo general:**
Verificar que tras la tramitacion exitosa o la cancelacion del pedido, el pendiente se cierra/elimina de la bandeja de Tramitar Pedido y el pedido ya no aparece como accion pendiente para el ESAC.

**Objetivos especificos:**
1. Identificar la tabla/entidad que gestiona los pendientes de la bandeja de Tramitar Pedido
2. Verificar que el flujo de tramitacion Credito cierra el pendiente al finalizar
3. Verificar que el nuevo flujo de cancelacion (Tarea 3) tambien cierra el pendiente
4. Validar que el pedido no aparece en la bandeja tras ambas acciones

**Resultado esperado:**
Tras tramitar o cancelar un pedido, el registro de pendiente se cierra y el pedido desaparece de la bandeja del ESAC.

**Entregables:**
- Evidencia de que el pendiente se cierra en ambos flujos
- Ajuste al BO de cancelacion si el cierre no esta incluido

**Criterios de aceptacion:**
- Al tramitar exitosamente, el pedido desaparece de la bandeja
- Al cancelar, el pedido desaparece de la bandeja
- No quedan registros huerfanos de pendientes

**Mas informacion de la tarea:**
- Regla de negocio 4 del requisito
- Criterio de aceptacion del requisito: A1, B1

**Recursos:**
- R16A-RE-FU-010.md (Regla 4)
- R16A-RE-FU-010-Back.md

---

## Resumen de Tareas

| # | Clave Catalogo | Descripcion | Estado |
|---|----------------|-------------|--------|
| T1 | IMP-EXIST-SERVICE | Verificacion flujo tramitacion Credito existente | Verificacion |
| T2 | IMP-EXIST-SERVICE | Verificacion variante Pago contra entrega + marca detencion Legacy | Verificacion |
| T3 | SERV-TRANSACT | Crear endpoint de Cancelacion de pedido | Desarrollo nuevo |
| T4 | IMP-EXIST-SERVICE | Verificacion cierre de pendiente en bandeja | Verificacion |
