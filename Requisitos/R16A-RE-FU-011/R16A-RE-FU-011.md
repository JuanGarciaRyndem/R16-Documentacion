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

El sistema debe permitir la tramitación de pedidos de clientes con condición de pago Crédito o Pago contra entrega cuando el pedido contiene sustancias controladas tipo Mundial, Nacional u Origen, reutilizando el flujo de tramitación de pedidos crédito existente en PQF2. Este flujo con Sustancias Controladas aplica únicamente a clientes de Región México; Región Perú no está soportada para el manejo de Sustancias Controladas en esta release (ver DUDA-027 en Notas). En este flujo el sistema no muestra las opciones de Factura por Adelantado ni de Entrega con Remisión, genera la Confirmación de Pedido al cliente y transfiere el pedido al sistema Legacy con la información necesaria para continuar el ciclo de venta.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Crédito en la operación de México.
- Pedidos con condición de pago Pago contra entrega.
- Pedidos que contienen al menos una sustancia controlada clasificada como Mundial, Nacional u Origen.
- Variante con sustancias controladas del flujo crédito preexistente en PQF2 (reuso del flujo regular, sin modificaciones funcionales salvo las restricciones específicas de productos controlados).

### No aplica a

- Pedidos de clientes con condición de pago Prepago (esos siguen un flujo distinto descrito en los requisitos del bloque Prepago).
- Pedidos sin sustancias controladas (variante cubierta en requisito independiente del bloque Crédito).
- Pedidos con activación de Factura por Adelantado (la combinación Factura por Adelantado + sustancias controladas no es permitida por regla regulatoria).
- Pedidos con marca de Entrega con Remisión (la combinación Remisión + sustancias controladas no es permitida por regla regulatoria).
- La validación de presencia de la Licencia Sanitaria y el Aviso de Responsable Sanitario del cliente: se documenta en su propio requisito, aunque se ejecute en este mismo módulo.
- La validación de pago de pedidos Pago contra entrega; esa validación la ejecuta Legacy.
- Región Perú: el manejo de Sustancias Controladas no está soportado en esta release; este flujo (Crédito con controlados) se acota a Región México. El soporte de controlados para Perú queda como riesgo operativo (ver Riesgo 2 y DUDA-027).

---

## Reglas de Negocio

**Regla 1 — Reuso del flujo crédito preexistente con restricciones regulatorias**
El módulo Tramitar Pedido aplica el flujo de tramitación de crédito existente en PQF2 para pedidos de clientes con condición de pago Crédito que contienen sustancias controladas tipo Mundial, Nacional u Origen, agregando las restricciones específicas que impone la presencia de sustancias controladas.

**Regla 2 — Factura por Adelantado y Entrega con Remisión no se ofrecen**
Para pedidos que contienen al menos una sustancia controlada tipo Mundial, Nacional u Origen, el sistema no ofrece las opciones de Factura por Adelantado ni de Entrega con Remisión, ya que su combinación con sustancias controladas no está permitida.

**Regla 3 — Pago contra entrega se comporta como crédito normal**
Los pedidos con condición de pago Pago contra entrega que contienen sustancias controladas se procesan siguiendo el mismo flujo de un pedido de Crédito con sustancias controladas. La detención por falta de validación de pago la ejecuta Legacy, no Tramitar Pedido.

**Regla 4 — Operación Perú sin transferencia a Legacy (referencia)**
Para pedidos de clientes Crédito de Región Perú, al concluir el flujo de tramitación en PQF2 el sistema no envía el pedido al sistema Legacy; la operación termina con la confirmación interna en PQF2. El manejo de Sustancias Controladas para Región Perú no está soportado en esta release (DUDA-027): no se construye flujo, validación ni bloqueo técnico para esa región en este requisito. Un pedido con controlados de un cliente de esa región podría, sin embargo, avanzar por este flujo; esto se asume como riesgo operativo comunicado al cliente (ver Riesgo 2). El comportamiento general de Crédito Perú sin controlados se documenta en las filas de Crédito correspondientes.

**Regla 5 — Cierre del pendiente de Tramitar Pedido al completar la acción**
Una vez ejecutada exitosamente la acción de tramitar, completado el envío del correo correspondiente al flujo y generados los pendientes derivados (si aplica), el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

---

## Riesgos

**Riesgo 1 — Clasificación incorrecta del producto en el catálogo**
Las restricciones de este flujo dependen de cómo esté clasificado cada producto en el catálogo. Si un producto controlado no está clasificado como tal, el sistema ofrecerá las opciones de Factura por Adelantado y Entrega con Remisión sobre un pedido que no debería tenerlas, incumpliendo la restricción regulatoria.

**Riesgo 2 — Avance de pedidos con controlados de Región Perú (riesgo operativo asumido, DUDA-027)**
El manejo de Sustancias Controladas para Región Perú no está soportado en esta release, pero el sistema no impide que un pedido con controlados de un cliente de esa región avance por el flujo de tramitación. El control es operativo, no de sistema. El cliente confirmó (DUDA-027) que Perú no soporta sustancias controladas en R16, por lo que no se desarrolla bloqueo técnico ni validación para esa región en este requisito; el riesgo se asume como operativo y se comunica al cliente.

---

## Criterios de Aceptación

### Sección A — Tramitación Crédito con controlados

**Criterio A1 — Tramitación habilitada para Crédito con controlados sin Factura por Adelantado sin Remisión**
- **Dado** que un pedido pertenece a un cliente Crédito de Región México, contiene al menos una sustancia controlada tipo Mundial, Nacional u Origen, y no requiere Factura por Adelantado ni Entrega con Remisión,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación siguiendo el flujo crédito existente, generar la Confirmación de Pedido al cliente y transferir el pedido a Legacy para continuar el ciclo de venta.

**Criterio A2 — Variante Pago contra entrega con controlados**
- **Dado** que un pedido pertenece a un cliente con condición de pago Pago contra entrega y contiene sustancias controladas,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá tramitarlo aplicando el mismo flujo de un pedido de Crédito con controlados.

**Criterio A3 — Transferencia a Legacy con marca de detención (México)**
- **Dado** que un pedido con condición de pago Pago contra entrega y con controlados se tramitó en PQF2 para un cliente de México,
- **Cuando** el sistema transfiere el pedido al sistema Legacy,
- **Entonces** deberá incluir en la transferencia la marca de detención que indica a Legacy que el pedido no debe entregarse hasta validar el pago.

### Sección B — Restricciones regulatorias

**Criterio B1 — No visualización de Factura por Adelantado y Entrega con Remisión**
- **Dado** que el pedido contiene al menos una sustancia controlada Mundial, Nacional u Origen,
- **Cuando** el ESAC visualiza las opciones disponibles en Tramitar Pedido,
- **Entonces** el sistema deberá ocultar las opciones Factura por Adelantado y Entrega con Remisión.

### Sección C — Cierre y transferencia

**Criterio C1 — Cancelación del pedido**
- **Dado** que un pedido tiene solicitud del cliente para cancelar,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

**Criterio C2 — Transferencia a Legacy de pedido tramitado (variante México)**
- **Dado** que un pedido Crédito con controlados se ha tramitado exitosamente en la operación de México,
- **Cuando** se completa la Confirmación de Pedido,
- **Entonces** el sistema deberá transferir automáticamente a Legacy toda la información necesaria del pedido para que el sistema legado continúe el ciclo de surtido, despacho y entrega.

---

## Notas

- Cubre dos requisitos del cliente sobre la tramitación bajo condición de pago Pago contra entrega y la transferencia a Legacy con marca de detención.
- A diferencia del flujo Prepago, en Crédito la Confirmación de Pedido se genera dentro del módulo Tramitar Pedido.
- La detención del pedido Pago contra entrega por falta de validación de pago es responsabilidad de Legacy.
- La validación de Licencia Sanitaria y Aviso de Responsable Sanitario del cliente ocurre antes de llegar a Tramitar Pedido (responsabilidad del módulo Pretramitar Pedido), por lo que no se incluye como criterio en este requisito.
- Aplicable únicamente a la operación de México. Perú no soporta sustancias controladas en R16.

**Resueltos (dudas cerradas):**
- **Alcance Perú (DUDA-027):** el cliente confirmó que Perú no soporta sustancias controladas en R16. No se desarrolla bloqueo técnico ni validación para esa región en este requisito; el avance de un controlado de Perú hacia facturación se asume como riesgo operativo, comunicado al cliente (control operativo, no de sistema). Ver Riesgo 2.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-08-21 | Cierre de duda | Se incorpora la resolución de DUDA-027: Perú no soporta sustancias controladas en R16 y no se construye bloqueo técnico ni validación para esa región; el riesgo se asume como operativo y comunicado al cliente. Se marcan como obsoletos (tachados, con motivo) los pasajes de Alcance, Regla 4 y Criterios A1/C3 que describían un flujo funcional para Perú, y se agrega Riesgo 2 documentando el riesgo operativo asumido. Se anota nota equivalente en `R16A-RE-FU-011-Back.md`. |
| 2 | 2026-09-04 | Sincronización matriz | Se reescribe el documento en limpio a partir de la matriz vigente, retirando el marcado de tachado/obsoleto usado para registrar el cierre de DUDA-027 (la conclusión se conserva en Regla 4, Riesgo 2 y en "Resueltos"). Se elimina formalmente el Criterio C3 (ya marcado como retirado). Se quita el prefijo "Crédito -" al referirse a la condición de pago Pago contra entrega. Requisito: se completa la redacción (la fuente de la matriz llegaba truncada) manteniendo la conclusión ya documentada de transferencia a Legacy. |
