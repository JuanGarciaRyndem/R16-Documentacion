# Reglas de Negocio — ProquifaDotNet

## 1. Cotización

### RN-COT-01 — Contacto debe corresponder al cliente
El contacto de cliente asignado a una cotización **debe pertenecer al mismo cliente**
de la cotización. Si no coincide, se lanza un error de validación.

### RN-COT-02 — Datos de facturación se asignan automáticamente
Si la cotización no tiene datos de facturación, el sistema busca automáticamente
los datos de facturación activos del cliente y los asigna.

### RN-COT-03 — Empresa se hereda de datos de facturación
Si la cotización no tiene empresa asignada pero sí tiene datos de facturación,
la empresa se toma directamente de dichos datos.

### RN-COT-04 — Caducidad automática de cotizaciones
El sistema marca automáticamente como **caducadas** todas las cotizaciones activas
cuya `FechaCaducidad` sea menor o igual a la fecha actual.

### RN-COT-05 — Ajustes al cerrar oferta
Al cerrar una oferta se pueden solicitar los siguientes tipos de ajuste:
- Condiciones de pago
- Precios por partida
- Flete express
- Tiempo de entrega (ajuste "-2 días")

Todos los ajustes se ejecutan dentro de una **transacción con nivel ReadUncommitted**.

---

## 2. Pretramitar Pedido

### RN-PP-01 — Validaciones obligatorias antes de tramitar
Un pedido **no puede ser tramitado** si alguna de las siguientes condiciones no está cumplida:
- Condiciones de pago **no validadas**
- Contacto de cliente entrega **no validado**
- Dirección de cliente entrega **no validada**
- Razón social **no validada**
- Cliente **sin datos de facturación activos**
- Pedido **sin configuración** (`ppPedidoConfiguracion` ausente)

### RN-PP-02 — Todas las partidas deben estar configuradas
Cada partida marcada como tramitable debe tener:
- Configuración de partida (`ppPartidaPedidoConfiguracion`)
- Tiempo de entrega asignado (`IdValorConfiguracionTiempoEntrega`)

Si alguna partida no cumple, se bloquea el proceso completo.

### RN-PP-03 — Cancelación de pedido con dependencia bloqueada
Un pedido pre-tramitado **no puede cancelarse** si ya generó una parcialidad
en tramitar pedido (`tpPedido`). La cancelación también requiere que exista
la acción `"Cancelacion"` registrada en `catBitacoraAccion`.

---

## 3. Tramitar Pedido

### RN-TP-01 — Separación de partidas por tipo de producto
Al tramitar un pedido, las partidas se separan en **tres grupos independientes**,
cada uno genera un pedido (`tpPedido`) distinto:
1. **Productos controlados** (según región del cliente)
2. **Productos no controlados**
3. **Publicaciones** (solo si el cliente **no** factura publicaciones con la misma empresa)

### RN-TP-02 — Publicaciones facturan con empresa diferente
Si un producto es de tipo publicación y el cliente tiene
`MismaEmpresaFacturaPublicaciones = false`, se genera un pedido separado
facturado por la empresa designada para publicaciones (`EmpresaFacturaPublicaciones`).

### RN-TP-03 — Partidas ya tramitadas no se duplican
Antes de procesar cada partida, se valida que no exista ya un `tpPartidaPedido`
vinculado al mismo `IdPPPartidaPedido`. Si ya existe, se omite.

### RN-TP-04 — El cliente debe tener región asignada y coincidir con el pedido
Al liberar un pedido tramitado, se valida que:
- El cliente tenga región asignada.
- La región del cliente coincida con la región del pedido.

### RN-TP-05 — Condición de crédito/prepago determina el flujo de liberación
Al generar el correo de tramitación, se evalúa `catCondicionesDePago.SinCredito`
para distinguir entre pedidos a crédito y pedidos de prepago.

---

## 4. Orden de Compra

### RN-OC-01 — El proveedor debe tener configuración de logística y pagos
Una orden de compra **no puede guardarse** si el proveedor no tiene
`IdConfiguracionPagos` asignado. Se lanza una excepción si falta esta configuración.

### RN-OC-02 — Condiciones y medio de pago se heredan del proveedor
Si la OC no tiene condiciones de pago o medio de pago, se toman automáticamente
de la configuración de pagos del proveedor.

### RN-OC-03 — Fecha de compra por defecto es hoy
Si la OC no tiene `FechaCompra`, se asigna automáticamente la fecha actual.

### RN-OC-04 — Estado de confirmación calculado desde partidas
El estado `Confirmada` y `NoConfirmada` de la OC se recalcula en cada guardado,
basándose en si **alguna** o **ninguna** partida tiene `Confirmada = true`.

### RN-OC-05 — Estado de entrega calculado si tiene monto
La OC se marca como entregada (`Entregada`) solo si `TotalUSD > 0` y
todas sus partidas han sido entregadas.

### RN-OC-06 — Desactivar OC limpia referencias de partidas
Al desactivar una OC, todas sus partidas pierden sus referencias a:
- Factura de proveedor (`IdOcFacturaProveedor`)
- Pendiente de compra (`IdOCPendienteCompraProducto`)
- Packing list (`IdOcPackingList`)

---

## 5. Promesa de Compra

### RN-PC-01 — Fecha de última actualización se registra en cada guardado
Cada vez que se guarda una promesa de compra, se actualiza automáticamente
`FechaUltimaActualizacion` con la fecha actual (sin hora).

### RN-PC-02 — Desactivar en cascada
Al desactivar una promesa de compra, se desactivan en cascada:
- Todos los fletes asociados
- Todas las partidas asociadas
- Todos los archivos asociados

---

## 6. Productos (Catálogos)

### RN-PROD-01 — Número CAS con dígito verificador
El número CAS de un producto debe tener formato `XXXXXXX-XX-X` y su último
dígito debe ser el **dígito verificador** calculado como:
`(suma de dígitos × posición inversa) mod 10`.
Un CAS vacío o nulo se considera válido.

### RN-PROD-02 — Número de catálogo único por familia
No pueden existir dos productos activos con el mismo número de catálogo
dentro de la misma familia de marca (`IdMarcaFamilia`).

---

## 7. Contratos de Cliente

### RN-CONT-01 — No pueden existir contratos contemporáneos con la misma marca
Un contrato de cliente **no puede asignarse a una marca** si existe otro contrato
del mismo cliente cuyas fechas se solapan con la del contrato nuevo
y ya tiene asignada esa misma marca. Se validan fechas `FechaInicio` y `FechaFin`.

---

## 8. Configuración de Facturación

### RN-FAC-01 — Restricción mensual de facturación es única por cliente
Solo puede existir **un registro** de restricción mensual por datos de facturación
de cliente (`IdDatosFacturacionCliente`). Se controla con columna única en BD.

### RN-FAC-02 — Datos de facturación deben estar activos
Para poder tramitar o liberar un pedido, el cliente debe tener datos de facturación
con estado `Activo = true`. De lo contrario, el proceso se bloquea.