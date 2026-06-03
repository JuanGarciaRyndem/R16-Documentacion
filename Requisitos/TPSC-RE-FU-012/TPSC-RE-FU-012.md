# Tramitación de pedidos Crédito con Factura por Adelantado

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-012 |
| **Nombre** | Tramitación de pedidos Crédito con Factura por Adelantado |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-002, R16.1M-RE-FU-003, R16.1M-RE-FU-005, R16.1M-RE-FU-010, R16.1M-RE-FU-011, R16.1M-RE-FU-013, R16.1M-RE-FU-014 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Crédito sin sustancias controladas activando la opción de Factura por Adelantado, para que el cliente reciba la factura PPD timbrada antes de la entrega y pueda justificar fiscalmente el gasto, mientras el pedido continúa su flujo regular de tramitación sin bloqueos.

---

## Requisito

El sistema debe permitir, en el módulo Tramitar Pedido, la activación opcional de Factura por Adelantado para pedidos de clientes con condición de pago Crédito (incluyendo la variante Pago contra entrega) cuando el pedido no contiene sustancias controladas. Al tramitar el pedido con esta opción activa, el sistema genera un pendiente en el módulo Factura por Adelantado para que Finanzas gestione la emisión de la factura posteriormente. La tramitación del pedido continúa el flujo crédito regular hasta su Confirmación y transferencia a Legacy, de forma independiente al ciclo de emisión de la factura. Al activar la opción, los datos de facturación quedan bloqueados y se toman del catálogo del cliente vigente.

---

## Alcance

### Aplica a

- Activación de Factura por Adelantado: pedidos de clientes con condición de pago Crédito en la operación de México exclusivamente. En Perú la Factura por Adelantado NO está disponible para pedidos Crédito, porque el timbrado fiscal en Perú aplica únicamente a pedidos Prepago en R16; un Crédito peruano no podría emitir la factura, por lo que la opción no se ofrece.
- Pedidos con condición Crédito - Pago contra entrega.
- Pedidos sin sustancias controladas (Mundial, Nacional, Origen).
- Activación opcional de la opción Factura por Adelantado desde el módulo Tramitar Pedido como punto de entrada único al flujo Factura por Adelantado.
- Generación de un pendiente en el módulo Factura por Adelantado al tramitar (la emisión y timbrado de la factura PPD ocurre posteriormente dentro de ese módulo, NO en Tramitar Pedido).
- Bloqueo de edición de los datos de facturación del pedido cuando se activa Factura por Adelantado.
- Operación en Perú: la tramitación del pedido Crédito (SIN Factura por Adelantado) opera idéntico a México pero NO transfiere a Legacy al concluir; la operación termina en la confirmación interna en PQF2. La opción Factura por Adelantado NO se ofrece para Crédito en Perú (ver punto anterior).

### No aplica a

- Pedidos de clientes con condición de pago Prepago (esos siguen un flujo distinto descrito en los requisitos del bloque Prepago).
- Pedidos con sustancias controladas (la combinación Factura por Adelantado + sustancias controladas no es permitida por regla regulatoria).
- La gestión del pendiente "Relacionar facturas" dentro de Legacy (responsabilidad de Legacy, no de PQF2).
- El conteo de los días de crédito del cliente (inicia con la emisión efectiva de la factura PPD, evento que ocurre fuera de Tramitar Pedido).
- Activación de Factura por Adelantado para pedidos Crédito de la región Perú: el timbrado fiscal peruano en R16 está limitado a Prepago, por lo que la factura anticipada de un Crédito peruano no podría emitirse.

---

## Reglas de Negocio

**Regla 1 — Factura por Adelantado opcional para Crédito**
Para pedidos de clientes con condición de pago Crédito sin sustancias controladas, el módulo Tramitar Pedido ofrece la opción de activar Factura por Adelantado como mecanismo opcional. Si no se activa, el pedido sigue el flujo crédito regular sin cambios; si se activa, dispara la generación del pendiente correspondiente en el módulo Factura por Adelantado al tramitar.

**Regla 2 — Tramitar Pedido como punto de entrada único al flujo Factura por Adelantado**
La activación de Factura por Adelantado se realiza exclusivamente desde el módulo Tramitar Pedido, no desde Pretramitar Pedido ni otros módulos.

**Regla 3 — Activación de Factura por Adelantado sin código de autorización**
La activación de Factura por Adelantado en Tramitar Pedido es directa, sin requerir código de autorización ni validación adicional.

**Regla 4 — Datos de facturación bloqueados cuando se activa Factura por Adelantado**
Al activar Factura por Adelantado para un pedido Crédito, el sistema no permite editar los datos de facturación del cliente (RFC, razón social, Uso CFDI, Método y Forma de Pago, correo de envío) desde Tramitar Pedido. Los datos de facturación quedan fijados con los valores del catálogo del cliente vigente al momento de la activación; cualquier ajuste posterior se gestiona en el módulo Factura por Adelantado o en el Catálogo de Clientes según corresponda.

**Regla 5 — Generación del pendiente Factura por Adelantado al tramitar**
Al activar Factura por Adelantado y ejecutar la acción de tramitar, el sistema genera un pendiente en el módulo Factura por Adelantado con la información necesaria para que el rol correspondiente gestione posteriormente la emisión y timbrado de la factura PPD. La tramitación del pedido no espera a la emisión de la factura.

> ** Pendiente confirmar con el cliente qué rol gestiona la emisión y timbrado de la factura PPD (Finanzas o Coordinador de Planeación y Control). **

**Regla 6 — Factura por Adelantado no bloquea la Confirmación de Pedido**
La gestión del pendiente Factura por Adelantado es un proceso independiente. La Confirmación de Pedido se genera de inmediato y el pedido continúa su flujo crédito regular en paralelo a la futura emisión de la factura PPD.

**Regla 7 — Operación Perú sin transferencia a Legacy**
Para pedidos de clientes Crédito de la región Perú, al concluir el flujo de tramitación en PQF2 el sistema no envía el pedido al sistema Legacy. La operación termina con la confirmación interna en PQF2.

**Regla 8 — Cierre del pendiente de Tramitar Pedido al completar la acción**
Una vez ejecutada exitosamente la acción de tramitar, completado el envío del correo correspondiente al flujo y generados los pendientes derivados (si aplica), el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

---

## Riesgos

**Riesgo 1 — Catálogo del cliente desactualizado al activar Factura por Adelantado**
Como los datos de facturación se fijan al activar Factura por Adelantado sin opción de editar en Tramitar Pedido, si el catálogo del cliente tiene datos fiscales desactualizados (RFC incorrecto, correo viejo, Uso CFDI no aplicable a la operación), la factura PPD se emitirá con esos datos y requerirá cancelación fiscal y reemisión posterior.

---

## Criterios de Aceptación

### Sección A — Activación y tramitación con Factura por Adelantado

**Criterio A1 — Tramitación con Factura por Adelantado activada para Crédito sin controlados**
- **Dado** que un pedido pertenece a un cliente Crédito en México, sin productos controlados, y el ESAC activa la opción Factura por Adelantado en el módulo Tramitar Pedido,
- **Cuando** se ejecuta la acción de tramitar,
- **Entonces** el sistema deberá tramitar el pedido siguiendo el flujo crédito regular, generar la Confirmación de Pedido al cliente y, en paralelo, generar el pendiente correspondiente en el módulo Factura por Adelantado para que el rol responsable emita y timbre la factura PPD.

**Criterio A2 — Activación de Factura por Adelantado desde Tramitar Pedido**
- **Dado** que un pedido pertenece a un cliente Crédito sin productos controlados,
- **Cuando** el ESAC visualiza el módulo Tramitar Pedido,
- **Entonces** el sistema deberá ofrecer la opción de activar Factura por Adelantado. La activación es directa y no requiere código de autorización.

**Criterio A3 — Variante Pago contra entrega con Factura por Adelantado**
- **Dado** que un pedido pertenece a un cliente con condición Crédito - Pago contra entrega sin controlados y el ESAC activa Factura por Adelantado,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá tramitarlo aplicando el mismo flujo de un Crédito normal con Factura por Adelantado. La detención por falta de validación de pago la ejecuta Legacy.

### Sección B — Bloqueo de datos y generación del pendiente

**Criterio B1 — Bloqueo de edición de datos de facturación al activar Factura por Adelantado**
- **Dado** que el ESAC activó la opción Factura por Adelantado en Tramitar Pedido,
- **Cuando** se renderiza la pantalla del pedido,
- **Entonces** el botón "Editar Datos" para datos de facturación no debe aparecer disponible para este pedido. El sistema deberá mostrar los datos de facturación en modo solo lectura tomados del catálogo del cliente vigente al momento de la activación.

**Criterio B2 — Generación del pendiente en el módulo Factura por Adelantado**
- **Dado** que el ESAC tramitó exitosamente un pedido Crédito con la opción Factura por Adelantado activada,
- **Cuando** se completa la tramitación del pedido,
- **Entonces** el sistema deberá generar automáticamente un pendiente en el módulo Factura por Adelantado, asociado al pedido tramitado, para que el rol responsable ejecute posteriormente la emisión y timbrado de la factura PPD.

> ** Pendiente confirmar con el cliente qué rol gestiona la emisión y timbrado de la factura PPD (Finanzas o Coordinador de Planeación y Control). **

### Sección C — Confirmación, FEE, cancelación y transferencia

**Criterio C1 — Generación de Confirmación de Pedido sin esperar factura**
- **Dado** que un pedido pertenece a un cliente Crédito de México, sin productos controlados, y el ESAC activa la opción Factura por Adelantado en el módulo Tramitar Pedido,
- **Cuando** se ejecuta la acción de tramitar,
- **Entonces** el sistema deberá generar la Confirmación de Pedido en formato PDF y permitir su envío al cliente, sin esperar a que se gestione el pendiente de Factura por Adelantado.

**Criterio C2 — Cálculo de FEE al tramitar**
- **Dado** que el pedido se tramita exitosamente en Tramitar Pedido,
- **Cuando** se genera la Confirmación de Pedido,
- **Entonces** el sistema deberá calcular automáticamente la FEE correspondiente al pedido conforme a las reglas vigentes en el sistema.

**Criterio C3 — Cancelación del pedido**
- **Dado** que un pedido tramitado tiene solicitud del cliente para cancelar,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

**Criterio C4 — Variante Pago contra entrega: transferencia a Legacy con marca de detención (México)**
- **Dado** que un pedido Crédito - Pago contra entrega con Factura por Adelantado se tramitó en PQF2 para un cliente de México,
- **Cuando** el sistema transfiere el pedido al sistema Legacy,
- **Entonces** deberá incluir en la transferencia la marca de detención que indica a Legacy que el pedido no debe entregarse hasta validar el pago.

**Criterio C5 — Transferencia a Legacy de pedido tramitado (variante México)**
- **Dado** que un pedido Crédito con Factura por Adelantado se ha tramitado exitosamente en la operación de México,
- **Cuando** se completa la Confirmación de Pedido,
- **Entonces** el sistema deberá transferir automáticamente a Legacy toda la información necesaria del pedido para que el sistema legado continúe el ciclo de surtido, despacho y entrega.

**Criterio C6 — Operación Perú sin transferencia a Legacy y sin Factura por Adelantado**
- **Dado** que un pedido Crédito se ha tramitado exitosamente para un cliente de la región Perú,
- **Cuando** se completa la Confirmación de Pedido,
- **Entonces** el sistema NO deberá ejecutar la transferencia a Legacy (la operación queda registrada únicamente en PQF2) y NO deberá ofrecer la opción Factura por Adelantado para el pedido, ya que el timbrado peruano en R16 aplica solo a Prepago.

---

## Notas

- Variante del flujo crédito preexistente del módulo Tramitar Pedido en PQF2 con la activación opcional del flujo Factura por Adelantado. El punto de entrada al flujo Factura por Adelantado se ubica exclusivamente en Tramitar Pedido (confirmado por el cliente; descartada la activación desde Pretramitar Pedido).
- Cubre tres requisitos del cliente: activación de Factura por Adelantado para clientes crédito generando pendiente y continuidad de flujo; tramitación bajo condición Crédito - Pago contra entrega apegada al flujo crédito existente; y transferencia a Legacy con marca de detención para Pago contra entrega.
- Al tramitar con Factura por Adelantado activada, el sistema NO emite la factura PPD en ese momento. Lo que hace es generar un pendiente en el módulo Factura por Adelantado para que el rol responsable gestione posteriormente la emisión y timbrado. La Confirmación del pedido se genera de inmediato sin esperar a la factura.
- Cambio respecto al comportamiento actual: se elimina el código de autorización para activar Factura por Adelantado (antes lo requería). La activación ahora es directa.
- Cambio respecto al comportamiento actual: cuando se activa Factura por Adelantado, los datos de facturación quedan fijados al momento de la activación y el botón "Editar Datos" se oculta para este pedido. Los datos de facturación se toman directamente del catálogo del cliente. Modificaciones posteriores requieren actualizar el catálogo o gestionar el ajuste en el módulo Factura por Adelantado. El botón "Editar Datos" sigue disponible para pedidos crédito que no activen Factura por Adelantado.
- Una vez emitida la factura por adelantado en el módulo Factura por Adelantado, el sistema debe generar un pendiente en el módulo "Relacionar facturas" del sistema Legacy. El mecanismo de comunicación PQF2 - Legacy para este pendiente queda pendiente de definir y NO es parte del scope de este requisito (corresponde al módulo Factura por Adelantado).
- A diferencia del flujo Prepago, en Crédito la Confirmación de Pedido se genera dentro del módulo Tramitar Pedido.
- La detención del pedido Pago contra entrega por falta de validación de pago es responsabilidad de Legacy.
- La tramitación del pedido Crédito es aplicable a México y Perú; en Perú no se transfiere a Legacy al concluir. La opción Factura por Adelantado, en cambio, solo aplica a Crédito de México: en Perú el timbrado fiscal en R16 se limita a Prepago, por lo que un Crédito peruano no puede emitir factura anticipada.
