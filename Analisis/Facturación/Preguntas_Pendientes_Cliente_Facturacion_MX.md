# Preguntas Pendientes para el Cliente — Notas de Crédito y Facturas de Ingreso (México)

Consolidado de los puntos que quedaron pendientes de definir en el diseño del sistema de facturación. Cada punto incluye el contexto explicado en términos de negocio (sin tecnicismos), una propuesta concreta, y la pregunta puntual para que se apruebe o, en su defecto, se proponga algo distinto.

---

## A. Catálogos y datos maestros

### A.1 A qué nivel se configuran los datos fiscales de un producto

**Contexto:** cada producto necesita 2 tipos de información para poder facturarse correctamente:

1. **Qué tipo de producto es y en qué unidad se vende**, según los catálogos que exige el SAT (por ejemplo, si es "reactivo químico" o "libro/publicación", y si se vende por pieza, por caja, etc.). Sin este dato, el sistema no sabe qué clave reportarle al SAT al momento de facturar.
2. **Si ese producto paga IVA, a qué tasa (16% o 0%), o si está exento** — esto determina cuánto impuesto se cobra en la factura.

La pregunta es: ¿esta información se captura producto por producto, o se puede definir una sola vez para todo un grupo/familia de productos que se comporten igual?

**Propuesta:** definir esta información **a nivel Familia de productos**, con la posibilidad de capturar una excepción puntual a nivel de un producto específico si ese producto se comporta distinto al resto de su familia (por ejemplo, si la mayoría de una familia paga 16% pero un producto en particular está exento). Así no hay que configurar cada producto uno por uno cuando la mayoría comparte el mismo comportamiento.

**Pregunta:** ¿aprueban este esquema (definir por Familia, con excepción posible a nivel de un producto puntual)?

---

### A.2 Número de pedimento en mercancía importada

**Contexto:** cuando se vende, por primera vez, un producto que viene de importación, la ley exige incluir en la factura el número del trámite aduanal (pedimento) con el que ese producto entró al país. El problema es que, en los casos donde se factura por adelantado, la factura se genera **antes** de que el producto siquiera se haya comprado al proveedor — es decir, antes de que exista cualquier trámite de importación. En ese momento, ese número simplemente no existe todavía.

**Propuesta:** antes de decidir cómo resolver esto, necesitamos ver cómo se maneja hoy en la práctica — no queremos asumir una solución sin conocer el contexto real.

**Pregunta:** ¿nos pueden compartir 2 o 3 ejemplos reales de facturas ya emitidas hoy (tanto de facturación por adelantado como de Prepago normal), para revisar cómo se está resolviendo (o no) este dato actualmente?

---

## B. Complementos de Pago

### B.1 Un solo Complemento de Pago con varias formas de pago, o uno distinto por cada forma de pago

**Contexto:** cuando un cliente paga usando más de una forma de pago (por ejemplo, parte por transferencia y parte con tarjeta) dentro del mismo periodo, hay que generar un Complemento de Pago — el documento fiscal que declara ante el SAT que ese pago ocurrió. Ustedes pidieron originalmente que se genere un Complemento de Pago distinto por cada forma de pago. Sin embargo, el mecanismo que el SAT recomienda es distinto: **un solo Complemento de Pago, que registra por separado cada forma de pago dentro de él** — se obtiene el mismo nivel de detalle (se sabe exactamente cuánto se pagó con cada forma), pero con menos documentos generados.

**Propuesta:** usar el mecanismo que recomienda el SAT (un solo Complemento de Pago que desglosa cada forma de pago), en vez de generar documentos completamente separados.

**Pregunta:** ¿aprueban este mecanismo, o tienen una necesidad específica (por ejemplo, para conciliar cada pago contra el banco por separado) que requiera Complementos de Pago completamente independientes por cada forma de pago?

---

### B.2 Qué fecha se usa cuando se paga una factura con saldo a favor en vez de dinero

**Contexto:** cuando a un cliente se le queda un saldo a favor (por ejemplo, por una devolución) y ese saldo se usa después para pagar una factura nueva, hay que registrar una fecha para ese movimiento en el Complemento de Pago. El problema es que ese saldo a favor no es dinero que "llegó" un día en particular — ya existía de antes, desde que se generó.

**Propuesta:** usar la fecha del día en que efectivamente se aplica ese saldo a la nueva factura (no la fecha en que se originó el saldo a favor).

**Pregunta:** ¿aprueban esta interpretación? Este punto en particular también conviene confirmarlo con su contador, ya que es un caso que el SAT no contempla de forma explícita en su documentación.

---

## C. Monedas y tipos de cambio

### C.1 Qué tipo de cambio se le asigna a un pago, y cuál se usa cuando paga una factura en otra moneda

**Contexto:** cuando un cliente paga en una moneda distinta a la de su factura, hay 2 decisiones relacionadas que hay que tomar, una después de la otra:

1. **¿Con qué tipo de cambio se registra ese pago?** Existen 2 fechas posibles para fijarlo: el día en que el dinero efectivamente llega al banco, o el día en que el cliente avisa/envía su comprobante a PROQUIFA (que puede ser distinto, si avisa días después de haber pagado).
2. **Una vez registrado el pago con su propio tipo de cambio, ¿ese es el que se usa para convertir cuánto cubre de la factura, o se usa el tipo de cambio que ya tenía la factura desde que se generó?**

**Propuesta:**
1. Registrar el pago con el tipo de cambio del **día en que el dinero llega al banco** — es el mismo criterio que exige el SAT para este tipo de comprobantes.
2. Usar **ese mismo tipo de cambio del pago** (no el de la factura) para calcular cuánto cubre — refleja el valor real que efectivamente llegó ese día, en vez de un tipo de cambio que la factura pudo haber fijado semanas antes.

**Pregunta:** ¿aprueban ambos puntos (tipo de cambio del pago fijado por la fecha de llegada al banco, y ese mismo tipo de cambio usado para la conversión contra la factura)?

---

## D. Otros temas operativos

### D.1 Permitir cobrar una factura o proforma de Prepago aunque falte un monto pequeño por pagar

**Contexto:** se pidió poder "cobrar" una factura o proforma de Prepago (y con eso, permitir tramitar el pedido) aunque el comprobante de pago que envía el cliente sea menor al total de la factura por $100 pesos o menos — sin perdonarle esa diferencia al cliente. La forma en que se planteó originalmente (marcar la factura como "ya pagada" para que deje de aparecer pendiente) generaría un problema real: el sistema perdería el rastro de ese dinero pendiente, y no habría manera de cobrarlo después ni de generar el Complemento de Pago correspondiente cuando el cliente sí lo pague.

**Propuesta — una alternativa más simple, sin ese problema:** en vez de marcar la factura como pagada, se agrega una validación aparte que solo permite tramitar el pedido si lo que falta por pagar es menor a un monto límite (por ejemplo, $100 pesos) — pero la factura **sigue mostrándose como pendiente**, con su saldo real, hasta que el cliente efectivamente complete el pago. Esto logra el mismo objetivo (no detener el pedido por un faltante pequeño) sin perder el rastro del dinero ni generar inconsistencias.

**La regla de los $100 pesos aplica sobre el total de la operación de cobro, no sobre cada factura por separado.** Si en una sola operación se están cobrando varias facturas al mismo tiempo, lo que importa es cuánto falta en total al sumar todas — no cuánto le falta a cada una individualmente.

*Ejemplo:* se cobran 3 facturas a la vez, por $500, $300 y $200 (total $1,000). El comprobante de pago recibido es de $950. El faltante de la operación completa es $50 ($1,000 − $950), que está dentro del límite de $100 — se permite tramitar los 3 pedidos asociados. Si en cambio el comprobante hubiera sido de $850 (faltante de $150), no se permitiría tramitar ninguno de los 3, aunque el faltante estuviera "repartido" entre varias facturas y ninguna individualmente pareciera deber más de $100.

**Qué pasa cuando el cliente después paga el faltante:** el sistema ya cuenta con la regla que resuelve esto — antes de generar cualquier documento nuevo, siempre valida si ya existe una factura para ese pedido. Como la factura ya existe (se generó cuando se aprobó con el faltante), ese segundo pago simplemente genera un Complemento de Pago contra la factura ya existente — no se genera una factura nueva, ni se vuelve a tramitar el pedido, evitando procesar dos veces lo mismo.

**Pregunta:** ¿aprueban este mecanismo completo (el pedido avanza con el faltante pequeño evaluado sobre el total de la operación, la factura sigue pendiente con su saldo real, y el pago posterior genera su propio Complemento de Pago contra esa misma factura sin duplicar nada)?

**Puntos adicionales a confirmar si se aprueba:**
- ¿Debe existir un límite total acumulado por cliente a lo largo del tiempo, o no hace falta?
- **Cuando la operación no está en pesos (USD u otra moneda), ¿cómo se aplica el límite de $100 pesos?** Nuestra propuesta: convertir esos $100 pesos a la moneda de la operación, usando el tipo de cambio del propio comprobante de pago recibido (el mismo tipo de cambio que ya se registra para ese comprobante, sección C.1) — así el límite se ajusta automáticamente a la moneda de cada operación sin necesitar una tabla de límites distinta por moneda. ¿Aprueban este criterio de conversión?

---

### D.2 Cómo se entera ProquifaNet 2 de que una factura ya se cobró, cuando el cobro ocurre en sistemas legados

**Contexto:** en los casos de Crédito y Contra Entrega con factura por adelantado, la factura se genera en ProquifaNet 2, pero el cobro de esa factura se sigue gestionando en sistemas legados. Sin una forma de pasar ese dato de un sistema a otro, PQF2 nunca se enteraría de que esas facturas ya se cobraron, y las mostraría como pendientes de forma indefinida.

**Propuesta:** definir un mecanismo para pasar este dato de un sistema a otro — no necesita ser inmediato; podría ser un proceso que corra una vez al día o una vez por semana, actualizando el estatus de esas facturas específicas.

**Pregunta:** ¿se tiene planeado, y es deseable tener en ProquifaNet 2 el estatus de factura cobrada de las facturas de crédito y contra entrega? Si sí, ¿aprueban que se construya un proceso periódico para esto, o prefieren que esas facturas específicas simplemente no se les dé seguimiento en PQF2 una vez generadas?
