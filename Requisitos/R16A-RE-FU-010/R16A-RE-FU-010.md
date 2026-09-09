# Tramitación de pedidos Crédito (Sin controlados, sin FxA)

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-010 |
| **Nombre** | Tramitación de pedidos Crédito (Sin controlados, sin FxA) |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-010, R16.1M-RE-FU-011, R16.1M-RE-FU-013, R16.1M-RE-FU-014 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Crédito (incluyendo la variante Pago contra entrega) sin sustancias controladas y sin Factura por Adelantado, para emitir la Confirmación de Pedido y transferir el pedido al sistema Legacy donde continúa el ciclo regular de surtido, despacho y entrega.

---

## Requisito

El sistema debe permitir la tramitación de pedidos de clientes con condición de pago Crédito (incluyendo la variante Pago contra entrega) cuando el pedido no contiene sustancias controladas y no requiere Factura por Adelantado; es el flujo de tramitación de pedidos crédito existente en PQF2. Al tramitar, el sistema debe asignar el folio interno de pedido, generar la Confirmación de Pedido al cliente, calcular la FEE correspondiente y, salvo para Perú, transferir el pedido al sistema Legacy con la información necesaria para continuar el proceso de compra. Para clientes de Región Perú no se transfiere a Legacy: la operación termina con la confirmación interna en PQF2.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Crédito en la operación de México y Perú.
- Pedidos con condición de pago Pago contra entrega (se comportan idénticamente al crédito normal en el módulo Tramitar Pedido; la detención por falta de validación de pago la ejecuta Legacy).
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
Los pedidos con condición de Pago contra entrega se procesan siguiendo el mismo flujo de un pedido Crédito normal. La detención por falta de validación de pago no se realiza en Tramitar Pedido; esa responsabilidad recae en el sistema Legacy.

**Regla 3 — Transferencia a Legacy con marca de detención para Pago contra entrega**
Al transferir a Legacy un pedido con condición Pago contra entrega, la transferencia incluye la información necesaria para que Legacy detenga el pedido en la fase de entrega si aún no se tiene la validación del pago.

**Regla 4 — Cierre del pendiente de Tramitar Pedido al completar la acción**
Una vez ejecutada exitosamente la acción de tramitar, completado el envío del correo correspondiente al flujo y generados los pendientes derivados (si aplica), el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

**Regla 5 — Operación Perú sin transferencia a Legacy**
Para pedidos de clientes Crédito de la región Perú, al concluir el flujo de tramitación en PQF2 el sistema no envía el pedido al sistema Legacy. La operación termina con la confirmación interna en PQF2.

---

## Criterios de Aceptación

### Sección A — Tramitación Crédito

**Criterio A1 — Tramitación habilitada para Crédito sin controlados sin Factura por Adelantado**
- **Dado** que un pedido pertenece a un cliente Crédito en México o Perú, sin productos controlados y sin activación de Factura por Adelantado,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación siguiendo el flujo crédito existente, generar la Confirmación de Pedido al cliente y, salvo para Perú, transferir el pedido a Legacy para continuar el ciclo de surtido. Para clientes de Región Perú no se transfiere a Legacy: la operación termina con la confirmación interna en PQF2.

**Criterio A2 — Variante Pago contra entrega**
- **Dado** que un pedido pertenece a un cliente con condición de Pago contra entrega, sin productos controlados y sin Factura por Adelantado,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá tramitarlo aplicando el mismo flujo de un Crédito normal.

**Criterio A3 — Transferencia a Legacy con marca de detención**
- **Dado** que el pedido Pago contra entrega ha sido tramitado en PQF2,
- **Cuando** el sistema transfiere el pedido al sistema Legacy,
- **Entonces** deberá incluir en la transferencia la marca de detención que indica a Legacy que el pedido no debe entregarse hasta validar el pago.

> Nota de referencia: el pedido de Pago contra entrega debe transferirse a Legacy con tratamiento de Crédito. Hoy, a nivel de código, estos pedidos operan como prepago (se transfieren con ese tratamiento para darles salida por el flujo de prepago); en R16 deben traducirse a Crédito. **Pendiente de verificar en la documentación de transferencia que esta traducción a crédito quede contemplada.**

### Sección B — Cancelación

**Criterio B1 — Cancelación del pedido desde Tramitar Pedido**
- **Dado** que un cliente solicita cancelar un pedido,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

---

## Notas

- Flujo preexistente del módulo Tramitar Pedido en PQF2. Este requisito documenta el alcance heredado sin cambios funcionales respecto a la versión actual del sistema, salvo por la convivencia con las nuevas capacidades transversales (cancelación).
- Cubre dos requisitos del cliente: tramitación bajo condición Pago contra entrega apegada al flujo crédito existente, y transferencia a Legacy con marca de detención para Pago contra entrega.
- La detención del pedido Pago contra entrega por falta de validación de pago es responsabilidad de Legacy, no del módulo Tramitar Pedido en PQF2.
- A diferencia del flujo Prepago, en Crédito la Confirmación de Pedido se genera dentro del módulo Tramitar Pedido (no se posterga a Validar Cobro porque el flujo Crédito no pasa por Validar Cobro).
- **Decisión de scope pendiente — Autorización en "Entrega con Remisión":** era requisito del cliente que al seleccionar "Entrega con Remisión" se pidiera un código de autorización. Hoy NO está implementado. La función de autorización se reconstruirá para el release; pendiente confirmar con el cliente si debe incluir el caso de "Entrega con Remisión". Impacto: si no se confirma su necesidad, conviene acotar el scope y dejar ese desarrollo para un release futuro o de mejoras, en lugar de construirlo ahora sin certeza de que se usará.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|---|---|---|
| 1 | 2026-09-04 | Sincronización matriz | Requisito, Regla 1→2 (Pago contra entrega), Criterio A1 y A2: se elimina el prefijo "Crédito -" al referirse a la condición de pago Pago contra entrega, dejándola como condición propia. Requisito y Criterio A1: se agrega la excepción de Perú (no se transfiere a Legacy; la operación termina con la confirmación interna en PQF2). Regla 5 agregada: Operación Perú sin transferencia a Legacy. |
| 2 | 2026-09-04 | Corrección de redacción | Criterio A3: se agrega nota de referencia sobre la traducción a tratamiento de Crédito de los pedidos Pago contra entrega transferidos a Legacy (hoy se transfieren con tratamiento de prepago a nivel de código); queda pendiente verificar que esta traducción esté contemplada en la documentación de transferencia. |
| 3 | 2026-09-08 | Sincronización matriz | Título/Nombre: se actualiza a "Tramitación de pedidos Crédito (Sin controlados, sin FxA)" para reflejar el campo Épica vigente en la matriz de requisitos (disambigua frente a las variantes controlados/FxA del bloque Crédito). Resto del contenido (Historia de Usuario, Requisito, Alcance, Reglas, Criterios A1-A3/B1, Notas) verificado sin cambios frente a la matriz. |
