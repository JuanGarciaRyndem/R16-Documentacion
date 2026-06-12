# R16A-RE-FU-010

**Estatus:** 🔲 No revisado

---

## Observaciones

> Sin observaciones.

## Notas adicionales

- En este requisito se debe analizar y generar un **catálogo explícito o implícito de estados de pedidos** (Orden de compra, pedido en curso, pedido confirmado, etc.) para tener claridad de en qué momento es una OC, en qué momento es un pedido con folio interno pero no confirmado, y en qué momento ya es un pedido confirmado. Esto dará claridad a los distintos módulos (facturación, reportes, etc.).
- Considerar la atención del ticket [DTP2-86](https://newryndem.atlassian.net/browse/DTP2-86) como parte de este requisito.
Ticket DTP2-86:
La transacción de Tramitar pedido y generación de PDF de confirmación de pedido no llevan un flujo correcto, ya que por un lado la transacción de tramitación del pedido recibe datos desde el front que debe guardar en el pedido y en otros objetos, sin embargo la generación del PDF ocurre antes de que la transacción se cierre, además de que consulta datos del pedido en BD en lugar de usar los que están en memoria (vienen desde front o son calculados en la propia transacción) lo cual hace que el PDF tenga condiciones diferentes al pedido final guardado. Actualmente para evitar estas diferencias se agregaron algunos parches (como que la generación del pdf calcule nuevamente datos que ya calculó la transacción y tiene en memoria), sin embargo el flujo natural debiera ser: datos llegan desde front, proceso de tramitación de pedido toma esos datos, hace cálculos que debe hacer (ej. FEE) y proceso de generación de PDF solo recibe esos parametros como input sin hacer recalculos, sin consultar desde BD, así se asegura que el PDF siempre se genere de manera idéntica al pedido guardado en BD.

## Resumen de cambios aplicados

> Sin cambios registrados.
