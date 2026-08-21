# Tramitación de pedidos Crédito con sustancias controladas

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-011 |
| **Nombre** | Tramitación de pedidos Crédito con sustancias controladas |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-010, R16.1M-RE-FU-011, R16.1M-RE-FU-013, R16.1M-RE-FU-014 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Crédito que contienen sustancias controladas (Mundial, Nacional u Origen) sin opción de Factura por Adelantado ni Entrega con Remisión, para procesarlos por el flujo crédito regular respetando las restricciones regulatorias del producto controlado.

---

## Requisito

El sistema debe permitir la tramitación de pedidos de clientes con condición de pago Crédito (incluyendo la variante Pago contra entrega) cuando el pedido contiene sustancias controladas tipo Mundial, Nacional u Origen, reutilizando el flujo de tramitación de pedidos crédito existente en PQF2. En este flujo el sistema deshabilita las opciones de Factura por Adelantado y Entrega con Remisión, genera la Confirmación de Pedido al cliente y transfiere el pedido al sistema Legacy con la información necesaria para continuar el ciclo de surtido, despacho y entrega.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Crédito en la operación de México.
- Pedidos con condición Crédito - Pago contra entrega.
- Pedidos que contienen al menos una sustancia controlada clasificada como Mundial, Nacional u Origen.
- Variante con sustancias controladas del flujo crédito preexistente en PQF2 (reuso del flujo regular, sin modificaciones funcionales salvo las restricciones específicas de productos controlados).
- ~~Pedidos de clientes con condición de pago Crédito en las operaciones de México y Perú.~~ ~~Operación en Perú: el flujo funcional opera idéntico al de México pero NO transfiere a Legacy al concluir; la operación termina en la confirmación interna en PQF2.~~ *(Obsoleto — ver DUDA-027 en Notas/Resueltos: el cliente confirmó que Perú NO soporta sustancias controladas en R16; este requisito no construye flujo ni bloqueo técnico para Perú. El avance de un controlado de Perú se asume como riesgo operativo comunicado al cliente.)*

### No aplica a

- Pedidos de clientes con condición de pago Prepago (esos siguen un flujo distinto descrito en los requisitos del bloque Prepago).
- Pedidos sin sustancias controladas (variante cubierta en requisito independiente del bloque Crédito).
- Pedidos con activación de Factura por Adelantado (la combinación Factura por Adelantado + sustancias controladas no es permitida por regla regulatoria).
- Pedidos con marca de Entrega con Remisión (la combinación Remisión + sustancias controladas no es permitida por regla regulatoria).
- La validación de presencia de Licencia Sanitaria y Aviso de Responsable Sanitario del cliente, que ocurre en el módulo Pretramitar Pedido antes de llegar a Tramitar Pedido.
- La validación de pago de pedidos Pago contra entrega; esa validación la ejecuta Legacy.
- Pedidos de clientes de la región Perú: Perú no soporta sustancias controladas en R16 (DUDA-027); este requisito no construye flujo ni validación para esa región. El riesgo de que se tramite/facture un controlado de Perú se asume como riesgo operativo comunicado al cliente, sin bloqueo técnico.

---

## Reglas de Negocio

**Regla 1 — Reuso del flujo crédito preexistente con restricciones regulatorias**
El módulo Tramitar Pedido aplica el flujo de tramitación de crédito existente en PQF2 para pedidos de clientes con condición de pago Crédito que contienen sustancias controladas tipo Mundial, Nacional u Origen, agregando las restricciones específicas que impone la presencia de sustancias controladas.

**Regla 2 — Bloqueo de Factura por Adelantado y Entrega con Remisión**
Para pedidos que contienen al menos una sustancia controlada tipo Mundial, Nacional u Origen, el sistema oculta las opciones de Factura por Adelantado y Entrega con Remisión, ya que su combinación con sustancias controladas no está permitida por regla regulatoria.

**Regla 3 — Pago contra entrega se comporta como crédito normal**
Los pedidos con condición de pago Crédito - Pago contra entrega que contienen sustancias controladas se procesan siguiendo el mismo flujo de un pedido Crédito normal con sustancias controladas. La detención por falta de validación de pago la ejecuta Legacy, no Tramitar Pedido.

**Regla 4 — ~~Operación Perú sin transferencia a Legacy~~ (Obsoleta, ver DUDA-027)**
~~Para pedidos de clientes Crédito de la región Perú, al concluir el flujo de tramitación en PQF2 el sistema no envía el pedido al sistema Legacy. La operación termina con la confirmación interna en PQF2.~~
El cliente confirmó (DUDA-027) que Perú no soporta sustancias controladas en R16: no se construye flujo, validación ni bloqueo técnico para esa región en este requisito. El avance de un pedido con controlado de Perú hacia facturación se asume como riesgo operativo comunicado al cliente, y no se controla por sistema.

**Regla 5 — Cierre del pendiente de Tramitar Pedido al completar la acción**
Una vez ejecutada exitosamente la acción de tramitar, completado el envío del correo correspondiente al flujo y generados los pendientes derivados (si aplica), el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

---

## Riesgos

**Riesgo 1 — Detección incorrecta de sustancias controladas**
La habilitación o bloqueo de las opciones Factura por Adelantado y Entrega con Remisión depende de que el sistema identifique correctamente la presencia de sustancias controladas en el pedido. Si la clasificación del producto en el catálogo es incorrecta o el sistema falla en detectar la presencia de un controlado dentro del pedido, podrían habilitarse opciones que violan la restricción regulatoria.

**Riesgo 2 — Riesgo operativo asumido: sustancias controladas en Perú (OBS-007 / DUDA-027)**
Perú no soporta sustancias controladas en R16 y este requisito no construye bloqueo técnico ni validación para esa región. Existe el riesgo de que, por error operativo, se tramite y/o facture un pedido con sustancia controlada en la operación de Perú. Este riesgo se asume como riesgo operativo comunicado al cliente (control operativo, no de sistema), conforme a la resolución de DUDA-027.

---

## Criterios de Aceptación

### Sección A — Tramitación Crédito con controlados

**Criterio A1 — Tramitación habilitada para Crédito con controlados sin Factura por Adelantado sin Remisión**
- **Dado** que un pedido pertenece a un cliente Crédito en México, contiene al menos una sustancia controlada tipo Mundial, Nacional u Origen, y no requiere Factura por Adelantado ni Entrega con Remisión,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación siguiendo el flujo crédito existente, generar la Confirmación de Pedido al cliente y transferir el pedido a Legacy para continuar el ciclo de venta.
- *(~~y, salvo para Perú, transferir~~ — redacción obsoleta: este requisito no aplica a Perú, ver DUDA-027 en Notas/Resueltos.)*

**Criterio A2 — Variante Pago contra entrega con controlados**
- **Dado** que un pedido pertenece a un cliente con condición Crédito - Pago contra entrega y contiene sustancias controladas,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá tramitarlo aplicando el mismo flujo de un Crédito normal con controlados, sin candado en Tramitar Pedido.

**Criterio A3 — Transferencia a Legacy con marca de detención (México)**
- **Dado** que un pedido Crédito - Pago contra entrega con controlados se tramitó en PQF2 para un cliente de México,
- **Cuando** el sistema transfiere el pedido al sistema Legacy,
- **Entonces** deberá incluir en la transferencia la marca de detención que indica a Legacy que el pedido no debe entregarse hasta validar el pago.

### Sección B — Restricciones regulatorias

**Criterio B1 — No visualización de Factura por Adelantado y Entrega con Remisión**
- **Dado** que el pedido contiene al menos una sustancia controlada Mundial, Nacional u Origen,
- **Cuando** el ESAC visualiza las opciones disponibles en Tramitar Pedido,
- **Entonces** el sistema deberá ocultar las opciones Factura por Adelantado y Entrega con Remisión.

### Sección C — Cierre y transferencia

**Criterio C1 — Cancelación del pedido**
- **Dado** que un pedido tramitado tiene solicitud del cliente para cancelar,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

**Criterio C2 — Transferencia a Legacy de pedido tramitado (variante México)**
- **Dado** que un pedido Crédito con controlados se ha tramitado exitosamente en la operación de México,
- **Cuando** se completa la Confirmación de Pedido,
- **Entonces** el sistema deberá transferir automáticamente a Legacy toda la información necesaria del pedido para que el sistema legado continúe el ciclo de surtido, despacho y entrega.

**Criterio C3 — ~~Operación Perú sin transferencia a Legacy~~ (Obsoleto, ver DUDA-027)**
- ~~**Dado** que un pedido Crédito con controlados se ha tramitado exitosamente para un cliente de la región Perú,~~
- ~~**Cuando** se completa la Confirmación de Pedido,~~
- ~~**Entonces** el sistema NO deberá ejecutar la transferencia a Legacy. La operación queda registrada únicamente en PQF2.~~
- *Retirado: Perú no soporta sustancias controladas en R16 (DUDA-027); este requisito no aplica a esa región y no se construye criterio de aceptación para ella. El riesgo se asume como riesgo operativo (ver Riesgo 2).*

---

## Notas

- Cubre dos requisitos del cliente sobre tramitación bajo condición Crédito - Pago contra entrega y la transferencia a Legacy con marca de detención.
- La validación de Licencia Sanitaria y Aviso de Responsable Sanitario del cliente ocurre antes de llegar a Tramitar Pedido (responsabilidad del módulo Pretramitar Pedido), por lo que no se incluye como criterio en este requisito.
- A diferencia del flujo Prepago, en Crédito la Confirmación de Pedido se genera dentro del módulo Tramitar Pedido.
- La detención del pedido Pago contra entrega por falta de validación de pago es responsabilidad de Legacy.
- ~~Aplicable a las operaciones de México y Perú. En Perú no se transfiere a Legacy al concluir.~~ *(Obsoleto, ver DUDA-027 abajo.)* Aplicable únicamente a la operación de México.

**Resueltos (dudas cerradas):**
- **Alcance Perú** (DUDA-027): el cliente confirmó que Perú no soporta sustancias controladas en R16. En consecuencia, no se desarrolla bloqueo técnico ni validación para esa región en este requisito; el avance de un controlado de Perú hacia facturación se asume como riesgo operativo, comunicado al cliente (control operativo, no de sistema). Ver Riesgo 2.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-08-21 | Cierre de duda | Se incorpora la resolución de DUDA-027: Perú no soporta sustancias controladas en R16 y no se construye bloqueo técnico ni validación para esa región; el riesgo se asume como operativo y comunicado al cliente. Se marcan como obsoletos (tachados, con motivo) los pasajes de Alcance, Regla 4 y Criterios A1/C3 que describían un flujo funcional para Perú, y se agrega Riesgo 2 documentando el riesgo operativo asumido. Se anota nota equivalente en `R16A-RE-FU-011-Back.md`. |
