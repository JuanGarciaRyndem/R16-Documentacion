# Tramitación de pedidos Crédito

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-010 |
| **Nombre** | Tramitación de pedidos Crédito |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-010, R16.1M-RE-FU-011, R16.1M-RE-FU-013, R16.1M-RE-FU-014 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Crédito (incluyendo la variante Pago contra entrega) sin sustancias controladas y sin Factura por Adelantado, para emitir la Confirmación de Pedido y transferir el pedido al sistema Legacy donde continúa el ciclo regular de surtido, despacho y entrega.

---

## Requisito

El sistema debe permitir la tramitación de pedidos de clientes con condición de pago Crédito (incluyendo la variante Pago contra entrega) cuando el pedido no contiene sustancias controladas y no requiere Factura por Adelantado; es el flujo de tramitación de pedidos crédito existente en PQF2. Al tramitar, el sistema debe asignar el folio interno de pedido, generar la Confirmación de Pedido al cliente, calcular la FEE correspondiente y transferir el pedido al sistema Legacy con la información necesaria para continuar el proceso de compra.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Crédito en la operación de México y Perú.
- Pedidos con condición Crédito - Pago contra entrega (se comportan idénticamente al crédito normal en el módulo Tramitar Pedido; la detención por falta de validación de pago la ejecuta Legacy).
- Pedidos sin sustancias controladas (Mundial, Nacional, Origen).
- Pedidos sin activación de la opción Factura por Adelantado.

### No aplica a

- Pedidos de clientes con condición de pago Prepago (esos siguen un flujo distinto descrito en los requisitos del bloque Prepago).
- Pedidos con sustancias controladas (variante cubierta en requisito independiente del bloque Crédito).
- Pedidos con activación de la opción Factura por Adelantado (variante cubierta en requisito independiente del bloque Crédito).
- La validación de pago de pedidos Pago contra entrega; esa validación la ejecuta Legacy, no el módulo Tramitar Pedido.

---

## Reglas de Negocio

**Regla 1 — Reuso del flujo crédito preexistente**
El módulo Tramitar Pedido aplica el flujo de tramitación de crédito existente en PQF2 para pedidos de clientes con condición de pago Crédito sin sustancias controladas y sin Factura por Adelantado, sin introducir comportamientos nuevos respecto a la versión actual del sistema.

**Regla 2 — Pago contra entrega se comporta como crédito normal**
Los pedidos con condición de pago Crédito - Pago contra entrega se procesan siguiendo el mismo flujo de un pedido Crédito normal. La detención por falta de validación de pago no se realiza en Tramitar Pedido; esa responsabilidad recae en el sistema Legacy.

**Regla 3 — Transferencia a Legacy con marca de detención para Pago contra entrega**
Al transferir a Legacy un pedido con condición Crédito - Pago contra entrega, la transferencia incluye la información necesaria para que Legacy detenga el pedido en la fase de entrega si aún no se tiene la validación del pago.

**Regla 4 — Cierre del pendiente de Tramitar Pedido al completar la acción**
Una vez ejecutada exitosamente la acción de tramitar, completado el envío del correo correspondiente al flujo y generados los pendientes derivados (si aplica), el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

---

## Criterios de Aceptación

### Sección A — Tramitación Crédito

**Criterio A1 — Tramitación habilitada para Crédito sin controlados sin Factura por Adelantado**
- **Dado** que un pedido pertenece a un cliente Crédito en México o Perú, sin productos controlados y sin activación de Factura por Adelantado,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación siguiendo el flujo crédito existente, generar la Confirmación de Pedido al cliente y transferir el pedido a Legacy para continuar el ciclo de surtido.

**Criterio A2 — Variante Pago contra entrega**
- **Dado** que un pedido pertenece a un cliente con condición de pago Crédito - Pago contra entrega, sin productos controlados y sin Factura por Adelantado,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá tramitarlo aplicando el mismo flujo de un Crédito normal.

**Criterio A3 — Transferencia a Legacy con marca de detención**
- **Dado** que el pedido Crédito - Pago contra entrega ha sido tramitado en PQF2,
- **Cuando** el sistema transfiere el pedido al sistema Legacy,
- **Entonces** deberá incluir en la transferencia la marca de detención que indica a Legacy que el pedido no debe entregarse hasta validar el pago.

### Sección B — Cancelación

**Criterio B1 — Cancelación del pedido desde Tramitar Pedido**
- **Dado** que un cliente solicita cancelar un pedido,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

---

## Notas

- Flujo preexistente del módulo Tramitar Pedido en PQF2. Este requisito documenta el alcance heredado sin cambios funcionales respecto a la versión actual del sistema, salvo por la convivencia con las nuevas capacidades transversales (cancelación).
- Cubre dos requisitos del cliente: tramitación bajo condición Crédito - Pago contra entrega apegada al flujo crédito existente, y transferencia a Legacy con marca de detención para Pago contra entrega.
- La detención del pedido Pago contra entrega por falta de validación de pago es responsabilidad de Legacy, no del módulo Tramitar Pedido en PQF2.
- A diferencia del flujo Prepago, en Crédito la Confirmación de Pedido se genera dentro del módulo Tramitar Pedido (no se posterga a Validar Cobro porque el flujo Crédito no pasa por Validar Cobro).
