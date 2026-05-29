


8 abr 2026
R16 Tramitar Pedido sin Crédito -
Revisión de pantallas continuación
Invitados
Rose Ríos GómezJuan David García Cruzblcalvo@proquifa.net

mfranco@proquifa.netRoberto Baez MuñozIrma Andrade Aguado

Francisco Uriel Guerrero RiveraJose Antonio Chavez Amador

cmramirez@proquifa.netssanchez@proquifa.netAlan Fernandez Garcia

Valdemar Farina Sanchez
Archivos adjuntos
R16 Tramitar Pedido sin Crédito - Revisión de pantallas conti...
Registros de la reunión
Grabación

Resumen
Discusión de la facturación por adelantado abordó la moneda de la
Proforma, la visualización de datos clave para el cliente y la decisión de
campos editables.

Moneda de Proforma y Tipo de Cambio
Se determinó que la moneda de la Proforma debe reflejar la moneda de
facturación del cliente configurada en el catálogo. Se decidió que la
regla del tipo de cambio a utilizar al validar el pago (si el de la Proforma
o el del día del pago) sería revisada por el equipo.

Vista Resumen y Detalle
Se acordó que en la vista resumen de pedidos pendientes el monto total
se muestre en dólares para mejorar la usabilidad. Se confirmó que la
información del contacto del comprador debe aparecer por línea de
pedido, no en la cabecera, ya que puede haber múltiples compradores.

Campos de Factura por Adelantado
Se concluyó que la mayoría de los datos para la generación de la factura
por adelantado deben ser de solo lectura para la agilidad operativa. El

único campo que se mantendría editable sería el “Uso de CFDI”,
mientras que la forma de pago PPD se mostraría bloqueada.

Detalles
●
Abordaje de la facturación por adelantado y preguntas preliminares: Rose
Ríos Gómez dio inicio a la presentación del módulo de factura por adelantado,
y Roberto Baez Muñoz presentó preguntas preliminares sobre el tema de
facturación a clientes de prepago con productos controlados. Roberto Baez
Muñoz solicitó confirmación sobre si la moneda de facturación configurada
en el catálogo de clientes se aplicaría a la Proforma o si esta se podría
cambiar antes de la generación.
●
Definición de la moneda de la Proforma: Sara SAnchez y Larissa Calvo
confirmaron que la moneda utilizada en la Proforma es la que está
configurada como moneda de facturación en el catálogo de clientes. También
se aclaró que para prepagos de productos controlados, la factura generada es
una factura de anticipo, aunque sí se genera una Proforma.
●
Tipo de cambio y moneda del pedido: En caso de que el pedido tuviera una
moneda diferente a la de facturación, se preguntó si el tipo de cambio sería el
vigente al momento de generar la Proforma. Larissa Calvo indicó que el
pedido se realiza con una moneda y un tipo de cambio, y la Proforma debería
obedecer a la misma moneda del pedido para evitar incongruencias con la
factura.
●
Definición de la moneda a utilizar en la Proforma: Se determinó que, dado
que el documento Proforma es una representación de cómo debe emitirse la
factura, debería reflejar la moneda de facturación del cliente. Esta decisión se
basó en el hecho de que en el catálogo de clientes pueden existir la moneda
de la oferta (cotización) y la moneda de facturación, y la Proforma debe
indicar al cliente cómo se le cobrará.
●
Tipo de cambio para la conversión a pesos: Se preguntó sobre el tipo de
cambio a utilizar si la moneda de facturación fuera en pesos, ya que los
precios suelen estar dolarizados, y se concluyó que se debería usar el tipo de
cambio del día. Larissa Calvo y Sara SAnchez confirmaron que el sistema
debe registrar la trazabilidad de ese tipo de cambio, ya que será el mismo que
se utilizará para la emisión final de la factura después de la validación del
pago.

●
Revisión del uso del tipo de cambio en el tiempo: Rose Ríos Gómez señaló
que en las buenas prácticas del mercado, cada documento o transacción
lleva su propio tipo de cambio según la fecha, lo que puede llevar a ganancias
o pérdidas cambiarias. Sara SAnchez y Rose Ríos Gómez acordaron que la
regla de qué tipo de cambio se usaría al validar el pago (si el de la Proforma o
el del día del pago) sería revisada por el equipo para aprovechar el tiempo en
la sesión.
●
Presentación del módulo de Factura por Adelantado - Vista resumen:
Roberto Baez Muñoz presentó la pantalla principal del módulo "Factura por
Adelantado," que muestra un listado agrupado por cliente, incluyendo RFC,
facturas pendientes por generar y el monto total de estos pedidos. Esta
pantalla incluye un buscador inicial que permite buscar por cliente o RFC.
●
Presentación del módulo de Factura por Adelantado - Vista detalle: Al
seleccionar un cliente, la siguiente pantalla muestra los detalles fiscales del
cliente y una lista de los pedidos pendientes de facturación. La lista de
pedidos incluye el pedido interno, la fecha del pedido, las condiciones de
pago, quién factura y el monto total.
●
Propuestas de información en la vista detalle: Se debatió si la fecha
mostrada debería ser la del pedido o la de la generación del pendiente;
Valdemar Farina Sanchez sugirió la fecha de generación del pedido interno.
Mayra Franco propuso incluir en los datos del cliente el contacto del asesor y
su correo electrónico, y solicitó que en la lista de pedidos se visualizara el
subtotal, IVA y monto total por separado.
●
Definición de datos del cliente y buscador en la vista resumen: Se aclaró que
el contacto solicitado debería ser el comprador que envió la orden de compra.
Mayra Franco también sugirió buscar por el código de cliente interno en la
pantalla de resumen, aunque Sara SAnchez y Rose Ríos Gómez confirmaron
que dicho código no existe, siendo el RFC el equivalente.
●
Decisión sobre la moneda en la vista resumen: Se concluyó que en la vista
resumen (agrupada por cliente) es más útil para la usabilidad que el monto
total de los pedidos se muestre en una sola moneda, preferiblemente en
dólares para facilitar la comparación. Rose Ríos Gómez enfatizó que al entrar
en el detalle del pedido sí se debería ver la moneda correspondiente a la
transacción.
●
Funcionalidad del buscador de pedidos y visualización de datos de contacto:
Valdemar Farina Sanchez explicó que la vista detalle solo mostraría los
pedidos pendientes de facturación por adelantado para el cliente
seleccionado. Rose Ríos Gómez aclaró que la información del contacto del

comprador (quien genera la orden de compra) debe aparecer en cada línea de
pedido y no en la cabecera, ya que el cliente puede tener múltiples
compradores con diferentes contactos.
●
Ordenamiento y filtr os en la vista de pedidos pendientes: Se acordó que el
listado inicial de clientes con pendientes de facturación debe estar ordenado
por la antigüedad de la solicitud. Cornelio M Ramirez B sugirió que la
búsqueda por pedido interno en la pantalla inicial (resumen) sería muy útil
para atacar pedidos urgentes, lo cual es una mejora para la agilidad operativa.
●
Discusión sobre la inclusión de totales en la vista resumen: Cornelio M
Ramirez B solicitó agregar la totalización del número de facturas pendientes y
el monto total pendiente de facturación en la pantalla de resumen. Rose Ríos
Gómez instó a priorizar la agilidad y ligereza de las pantallas operativas,
reservando la analítica y totalización para los módulos de reportes que se
desarrollarán más adelante.
●
Presentación de la generación de factura y datos requeridos: Roberto Baez
Muñoz presentó el modal para generar la factura por adelantado, que incluye
el pedido interno, monto, condiciones de pago, datos fiscales del cliente (RFC,
razón social, correo) y detalles de facturación. Se confirmó que el CURP no es
un dato requerido.
●
Reglas de edición de campos y moneda en la factura: Cornelio M Ramirez B
afirmó que los datos de la factura por adelantado provienen del pedido y del
catálogo, por lo que no deberían ser editables si son consistentes. Valdemar
Farina Sanchez confirmó que el monto en esta pantalla de generación de
factura debe estar en la moneda de facturación.
●
Confirmación del método de pago PPD y uso de CFDI: Se confirmó que el
método de pago para todas las facturas por adelantado debe ser PPD (Pago
en Parcialidades o Diferido) y no editable, ya que es una regla de la autoridad
fiscal (SAT). Cornelio M Ramirez B y Sara SAnchez indicaron que el campo
Uso de CFDI sí debería ser seleccionable en esta pantalla, ya que puede variar
según la necesidad del cliente.
●
Discusión sobre la Empresa de Facturación y Refacturación: Se confirmó que
la misma empresa que genera la confirmación del pedido es la responsable
de la facturación. El equipo planteó la preocupación de que una refacturación
podría ser causada por un error en el sistema donde el pedido sale con la
empresa correcta, pero la factura no. Rose Ríos Gómez sugirió que el equipo
podría beneficiarse al tener la información de la empresa visible en el
sistema, pero solo como lectura, para reconfirmar la entidad correcta.

●
Análisis de Campos Editables y de Solo Lectura: Rose Ríos Gómez solicitó al
equipo que definier  a qué campos en la pantalla podrían cambiarse o si todos
debían ser de solo lectura. Cornelio M Ramirez B inicialmente pensó que la
moneda de facturación podría necesitar un cambio, aunque luego determinó
que, para fines de facturación, ya viene definida. La conclusión fue que la
mayoría de la información debería ser de solo lectura, con la posible
excepción de algunos campos.
●
Inclusión de la Forma de Pago en la Visualización: Sara SAnchez señaló que
el dato de la forma de pago era importante y no estaba visible. Cornelio M
Ramirez B explicó que si la forma de pago es PPD, se establece
automáticamente como "por definir  ," y esta es la regla estándar. Rose Ríos
Gómez y Cornelio M Ramirez B acordaron que la forma de pago (99% 'por
definir  ') debería ser visualizada en todo momento, aunque bloqueada y solo
de lectura.
●
Visualización del Importe en Pantalla: Mayra Franco sugirió que el importe
debería visualizarse rápidamente. Larissa Calvo explicó que era necesario
hacerlo más llamativo, posiblemente cambiando su ubicación en la interfaz,
ya que el importe no se veía claramente cerca de la opción de "generar
factura". Roberto Baez Muñoz confirmó que su objetivo era validar qué
información se mostraría, y Rose Ríos Gómez indicó que ellos ayudarían a
hacerlo más visual.
●
Evaluación de la Edición del Tipo de Cambio: Rose Ríos Gómez propuso
considerar si el campo del tipo de cambio podría ser editable, mencionando
que ha visto esto en otros sistemas para permitir ajustes en ciertas
negociaciones. Cornelio M Ramirez B indicó que cualquier cambio en el tipo
de cambio tendría que ser validado. Biridiana Arias señaló que el tipo de
cambio no suele modificarse para una factura por adelantado, pero sí podría
haber excepciones para validaciones en clientes de prepago.
●
Decisión Final sobre Campos Editables: El equipo acordó que, para fines de
agilidad y cobertura del 98% de los casos, la mayoría de los campos se
bloquearían. Mayra Franco estuvo de acuerdo en que el tipo de cambio debe
ser el que arroja el sistema por defecto. Se concluyó que solo el "uso de CFDI"
quedaría como editable, mientras que todos los demás campos serían de
solo lectura.
●
Planificación de Futuras Sesiones: Rose Ríos Gómez agradeció al equipo por
su participación y enfatizó la importancia de su presencia en futuras
sesiones, especialmente para la validación del pago. Se sugirió una duración
de dos horas para las siguientes reuniones.


Pasos siguientes recomendados

[Larissa Calvo] Revisar Tipo Cambio: Revisar regla de tipo de cambio al validar
pago. Reconfirmar si se usa el del día o el de generación de la Proforma.

[Valdemar Farina Sanchez] Agregar Contacto Pedido: Incluir el dato de
contacto del comprador y correo electrónico en cada línea de pedido
pendiente. Esto aplica en la vista de detalle del cliente.

[Valdemar Farina Sanchez] Configurar Buscador: Configurar funcionalidad de
búsqueda por coincidencia para pedidos. La búsqueda no debe ser exacta.

[Valdemar Farina Sanchez] Agregar Totalizadores: Incluir totalizadores en la
pantalla inicial de resumen. Mostrar total de clientes, total de facturas
pendientes y monto total.

[Valdemar Farina Sanchez] Agregar Razón Social: Agregar campo de razón
social de la empresa que factura en el modal de generación de factura. Esto
evitará refacturaciones.

[Roberto Baez Muñoz] Visualizar Pago: Poner la forma de pago. Asegurar que
esté bloqueado o de solo lectura.

[Roberto Baez Muñoz] Mejorar Visualización: Ayudar a hacer más visual el
importe. Cambiar la posición o destacarlo.

Revisa las notas de Gemini para asegurarte de que sean correctas. Obtén consejos y
descubre cómo toma notas Gemini
Danos tu opinión sobre el uso de Gemini para tomar notas en una breve encuesta.
