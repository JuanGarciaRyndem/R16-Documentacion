


abr 15, 2026
R16 Tramitar Pedido sin
Crédito - 7ma Sesión de
entendimiento
Invitado
barias@proquifa.netRose Ríos GómezJuan David García Cruz

blcalvo@proquifa.netmfranco@proquifa.netRoberto Baez Muñoz

Irma Andrade AguadoAlejandro Cervantes Florescmramirez@proquifa.net

ssanchez@proquifa.netValdemar Farina SanchezAlan Fernandez Garcia
Archivos adjuntos

R16 Tramitar Pedido sin Crédito - 7ma Sesión de entendimiento

R16 Tramitar Pedido sin Crédito - 7ma Sesión de entendimiento - 2026/04/15 1...
Registros de la reunión
Grabación 2Grabación


Resumen

El equipo definió reportes de facturas y pedidos junto con procesos de
cancelación y trazabilidad operativa.

Diseño de reportes operativos
El equipo optimizó la visualización de facturas eliminando columnas
innecesarias y estandarizando campos de fechas de pago. Se acordó
mantener el código universal de identificación y utilizar un semáforo
visual para gestionar el tiempo restante de crédito.

Protocolos de cancelación prepago
Se decidió que la cancelación de pedidos prepagados por falta de pago
será ejecutada manualmente por el personal de servicio al cliente. Esta
acción automatizará la anulación de proformas y facturas asociadas
dentro del mes vigente.


Trazabilidad de pedidos detallada
Se definió la estructura de trazabilidad por partida para pedidos,
integrando estados de compra, inspección y facturación con enlaces a
documentos. El sistema permitirá ordenar información para facilitar el
seguimiento operativo de los pedidos.


Próximos pasos

[Valdemar Farina Sanchez] Quitar Columnas: Analizar eliminación de
columnas innecesarias como WIP y MEC (Monto Estimado de Cobro).
Priorizar mantener la interfaz limpia.

[Valdemar Farina Sanchez] Revisar FEP: Analizar y proponer el mejor método
para registrar la Fecha Estimada de Pago (FEP). Revisar cómo establecer la
fecha para múltiples proformas o facturas en la lista.

[Larissa Calvo] Enviar Reglas: Proporcionar por escrito las reglas del
semáforo. Detallar los días exactos para los estados visuales (verde, amarillo,
rojo, morado).

[El grupo] Proponer Cancelación: Elaborar propuesta para que SAC pueda
cancelar pedido por falta de pago. Revisar cómo integrar cancelación
automática de proforma/factura.

[Sara SAnchez] Compartir Reporte Base: Compartir ejemplo de reporte actual
que se descarga para referencia del equipo de desarrollo.

[Cornel] Enviar Catálogo SAT: Enviar listado de medios de pago del catálogo
SAT al equipo para validación.

[El grupo] Agregar Fecha Pago: Incluir la fecha real de pago recibido en el
reporte detallado de facturas. Colocar el dato en la parte superior o junto al
folio de cobro.

[Larissa Calvo] Enviar Solicitud: Enviar correo a Valdemar para estimar la
solicitud de cambio sobre la regla de aceptación de parciales.

[Biridiana Arias] Compartir Video: Compartir video para detallar la información
visible por línea de pedido. Mostrar qué datos contiene cada partida individual
del sistema actual.

[Biridiana Arias] Confirmar Campo: Visualizar con equipo de compras el
nombre exacto del campo fecha de inicio en importación. Detallar esta
información en el video compartido.


[Sara SAnchez] Proporcionar Reglas: Compartir las reglas necesarias, a nivel
datos y documentos, desde los sistemas legados. Especificar la ubicación
para la conexión de extracción de datos.

[El grupo] Agendar Sesión: Revisar internamente los ajustes necesarios y
definir la propuesta de fecha para la última sesión de validación.

[Larissa Calvo, Sara SAnchez] Resolver Pendientes: Resolver las dudas y
pendientes listados en el archivo compartido. Asegurar el cierre de todos los
puntos críticos.


Detalles
●
Presentación del Módulo de Reportes de Facturas: Roberto Baez Muñoz
inició la sesión presentando el módulo de reportes, comenzando con el de
facturas, con la intención de revisar posteriormente el de cobros. La
propuesta inicial era mantener una pantalla principal similar a la actual,
mostrando un listado de facturas ya emitidas con columnas como folio,
nombre del cliente, estado (por cobrar o cobrada), fecha de facturación, tipo
de factura, folio de pedido interno, condiciones de pago, emisor y monto.
●
Revisión de Columnas de la Pantalla de Facturas: El equipo coincidió con
algunas columnas de la propuesta, pero buscaban confirmar si toda la
información propuesta era necesaria para los usuarios. Larissa Calvo indicó
que el equipo está "casado" con el formato actual y utiliza las bases de datos
en Excel para análisis robustos, sugiriendo que para la vista en pantalla
podrían eliminarse columnas como "UID" e incluso fechas, buscando
mantener una vista más limpia.
●
Búsqueda de Optimización de la Vista de Facturas: Valdemar Farina Sanchez
apoyó la idea de mantener la pantalla de consulta lo más limpia posible,
enfocándose en las columnas que permitan al usuario identificar rápidamente
la factura y acceder a su detalle. Se acordó eliminar algunas columnas
mencionadas por Larissa Calvo, mientras que el detalle completo de la
información estaría disponible al momento de exportar o al ver el detalle de la
factura.
●
Aclaración de los Campos FP y DRC: Valdemar Farina Sanchez solicitó
aclaraciones sobre los campos "FP" y "DRC", identificados como "Fecha
Estimada de Pago" y "Días Restantes de Crédito", respectivamente. Larissa
Calvo explicó que el FP es la fecha de pago programada por los analistas de

cobranza después de la confirmación del cliente, y el DRC indica cuántos días
faltan para cobrar la factura.
●
Proceso de Prepago y Fecha Estimada de Pago: Se discutió el proceso de
establecer la Fecha Estimada de Pago (FEP) para clientes de prepago, a
diferencia de los clientes a crédito, donde la FEP se establece tras la revisión
de la factura. Se mencionó que, para prepagos, la FEP la coloca el ejecutivo
de servicio a clientes y que, según el flujo actual, el analista de cuentas por
cobrar tendría 72 horas para validar el pago una vez recibido.
●
Gestión de Cobranza en Flujos de Prepago: Sara SAnchez aclaró que la
recepción y gestión del pago es responsabilidad del área de Cobros, no de
Servicio a Clientes (ESAC), lo que elimina la necesidad de una fecha estimada
de pago en el prepago. Sin embargo, Larissa Calvo y Valdemar Farina
Sanchez debatieron la existencia de una "ejecución de cobranza de prepago"
en los flujos conceptualizados, donde un analista contactaría al cliente si no
se recibe el pago de un pedido ya tramitado.
●
Decisión sobre la Gestión Proactiva de Cobranza de Prepago: El equipo
buscó definir si se debe implementar una gestión proactiva para el 50% de los
clientes de prepago que no pagan de inmediato o si se mantendrá el proceso
de esperar el pago, sin seguimiento de cobranza. Rose Ríos Gómez y
Valdemar Farina Sanchez sugirieron que, si es necesario un seguimiento, en
lugar de crear una nueva pantalla, se podría registrar una fecha compromiso o
fecha estimada de pago en el módulo actual para tener visibilidad.
●
Propuesta de Seguimiento sin Módulo Adicional: Biridiana Arias señaló que
el área de crédito no tiene visibilidad del seguimiento de pagos de prepago.
Larissa Calvo coincidió en que una pantalla adicional no sería necesaria, pero
propuso registrar una fecha de seguimiento de pago en la pantalla de
validación actual para evitar pérdidas de órdenes de compra, lo que permitiría
la visibilidad deseada.
●
Viabilidad Técnica de Registrar la Fecha de Seguimiento: Valdemar Farina
Sanchez indicó que el equipo de análisis debe revisar la mejor forma de
implementar la propuesta de registrar una fecha estimada de pago para cada
factura o proforma en la lista. Además, se planteó que la situación de clientes
que tardan en pagar incluso teniendo una factura por adelantado es poco
común (dos a tres casos al mes).
●
Aclaración sobre la Columna de Días Restantes de Crédito (DRC): Se
confirmó que, a pesar de que el término "crédito" puede ser confuso en
prepago, la columna DRC se reetiquetaría a "Días Restantes para Pago" para
prepagos, calculándose a partir de la Fecha Estimada de Pago que se haya

capturado. Si el cliente de prepago paga de inmediato y no se requiere
seguimiento, esta columna permanecería vacía.
●
Definición de la Fecha Estimada de Pago y sus Reglas: Larissa Calvo enfatizó
que la Fecha Estimada de Pago es la que el usuario captura y es diferente a la
fecha de facturación más los días de crédito, ya que considera el tiempo de
revisión de la factura por el cliente y las condiciones de pago habituales. Se
estableció que la fecha se coloca cuando se aprueba la factura o, en caso de
prepago, cuando el analista de cuentas por cobrar le habla al cliente.
●
Estados de la Factura y Reglas: Valdemar Farina Sanchez solicitó las reglas
para los estados de la factura. Larissa Calvo explicó que el estado es
"cerrado" cuando el pago se ha ejecutado y se ha hecho el complemento, y
"abierto" si no se ha cobrado.
●
Columnas a Omitir y Mantener: Se confirmó la intención de omitir algunas
columnas, aunque Larissa Calvo inicialmente mencionó que el UUID sí es útil
para la consulta y validación de facturas debido a problemas de duplicación
en el sistema. El equipo decidió mantener el UUID por el momento para luego
revisarlo. Se acordó eliminar el campo de "Monto Estimado de Cobro" (MS) y
mantener el "Monto de la Factura".
●
Uso y Reglas del Semáforo de Apoyo Visual (Banderitas): Larissa Calvo
explicó que la bandera roja en la columna DRC indica que una factura no ha
sido programada, y que el sistema utiliza un semáforo visual (verde, amarillo,
rojo, morado) para indicar el tiempo restante para el pago. Se acordó
mantener el diseño visual (aunque se propondrá una alternativa a las
"banderitas") porque será necesario para el flujo de crédito.
●
Efecto de la Reprogramación en el Semáforo: Se confirmó que los Días
Restantes de Crédito y el semáforo dependen de la Fecha Estimada de Pago,
por lo que si se establece una nueva fecha (debido a una reprogramación), el
semáforo y los días cambiarán. Larissa Calvo sugirió la idea de registrar una
fecha inicial y hasta dos fechas subsiguientes en futuros desarrollos.
●
Consideraciones Adicionales para Prepago (Vigencia de Proforma): Biridiana
Arias introdujo el factor de la vigencia de la Proforma (un mes), que rige el
proceso de pago en prepago, señalando que la fluctuación del tipo de cambio
puede afectar el monto y requerir una modificación si el cliente paga después
de la vigencia. Se aclaró que la Proforma tiene una vigencia de un mes para
evitar afectaciones por tipo de cambio o ajustes en precios de proveedores.
●
Vigencia de la Proforma y Cancelación de Pedidos: Larissa Calvo y Valdemar
Farina Sanchez discutieron la vigencia de la proforma, la cual se estableció en
un mes. Se acordó que si una proforma vence y el pago del cliente no se ha

recibido, el pedido asociado debe cancelarse, ya que la proforma ya no es
vigente. La cancelación del pedido implica automáticamente la cancelación
de la proforma y, si existiera, de la factura generada por adelantado, siempre
dentro del mismo mes.
●
Propuesta de Flujo de Cancelación: Sara SAnchez propuso que la acción de
cancelación se detone desde la cancelación del pedido en la plataforma de
tramitar pedido. La cancelación del pedido automáticamente cancelaría
cualquier documento asociado, como la proforma o la factura, para evitar
múltiples acciones de cancelación por diferentes áreas.
●
Determinación del Responsable de la Cancelación: La decisión de quién debe
iniciar la cancelación por falta de pago fue discutida, concluyendo que
debería ser el área de SAC (Servicio al Cliente). Se acordó que SAC cancelaría
el pedido desde la función de tramitar pedido, y el sistema cancelaría
automáticamente la proforma o factura relacionada.
●
Automatización vs. Intervención del Usuario en la Cancelación: Se debatió si
la cancelación debería ser automática al final del mes o si la acción debería
recaer en el usuario. Se llegó a la conclusión de que la acción de cancelar el
pedido por falta de pago debe ser ejecutada por el usuario (SAC). Esto
permite al usuario tener visibilidad y tomar decisiones, especialmente porque
en ciertos escenarios una proforma vencida aún podría ser cobrada.
●
Escenario de Facturación por Adelantado y Cancelación en Mes Posterior: Se
analizó el escenario de una factura generada por adelantado que no es
pagada ni cancelada dentro del mes corriente, lo cual implicaría generar una
nota de crédito en el mes posterior. Biridiana Arias señaló que los casos de
facturación por adelantado a clientes de prepago que no pagan son muy
bajos (menos del 1% de la cartera). Se sugirió no incorporar este escenario en
la sistematización inicial debido a su baja probabilidad, y que si llegara a
suceder, se aplicaría la nota de crédito.
●
Acuerdo sobre la Cancelación para el MVP: El equipo acordó que la
propuesta inicial se centrará en permitir que SAC cancele el pedido por falta
de pago, cubriendo tanto proformas como facturas por adelantado, solo para
pedidos de prepago, sin incluir el escenario de la nota de crédito para esta
primera entrega (MVP). Se recomienda tener el escenario de cancelación de
factura en mes posterior "en el radar" para futuras mejoras.
●
Reporte de Facturas - Listado y Funcionalidades: Se discutió el reporte de
facturas, el cual incluirá columnas con información detallada y la posibilidad
de descarga con selección múltiple. La pantalla de visualización permitirá
filtr ar por cliente, moneda y buscar por folio. Sara SAnchez solicitó que se
compartiera la lista de columnas que se mostrarán para su validación.

●
Reporte de Facturas - Detalle y Trazabilidad: El detalle de cada factura
mostrará información como el cliente, tipo de factura, fecha de facturación,
pedido interno asociado, condiciones de pago y monto. También se incluirá la
trazabilidad de documentos generados (proforma, complemento de pago y
nota de crédito). Se propuso la funcionalidad de reenvío del PDF y XML de la
factura por correo al cliente.
●
Navegación entre Reportes (Trazabilidad): Se propuso incluir links en los
reportes (facturas, cobros y pedidos) para permitir la navegación directa entre
ellos, asegurando la trazabilidad de la información sin saturar las pantallas.
Por ejemplo, el detalle de la factura incluirá un link al pedido interno
correspondiente y al cobro asociado.
●
Ajuste en la Visualización del Reporte de Facturas: Larissa Calvo solicitó que
se incluyera explícitamente la "fecha de pago" (la fecha en que el dinero
ingresó al banco) en el detalle de la factura, además de la fecha de timbrado
del complemento de pago, lo cual se acordó que se implementaría.
●
Reporte de Cobros - Listado y Campos: El reporte de cobros presentará un
listado con filtr os por cliente, cuenta de destino y factura. La vista principal
incluirá el folio del cobro, cliente, fecha de cobro, tipo de cambio, medio de
pago, cuentas de origen/destino y monto de cobro. Se aclaró que el monto de
cobro es lo que pagó el cliente, y el detalle de las facturas cubiertas se verá
en la siguiente pantalla.
●
Reporte de Cobros - Medios de Pago: El campo de medios de pago en la
pantalla de reporte de cobros debe alinearse con el catálogo del SAT. Sara
SAnchez acordó compartir la lista de medios de pago para que el equipo la
implemente.
●
Reporte de Cobros - Detalle y Relación con Facturas: El detalle del cobro
mostrará un resumen de la transacción y la relación de las facturas a las que
se aplicó el cobro. Este listado de facturas mostrará el folio, pedido interno,
cliente, fechas, montos y documentos ligados, como la proforma y el
complemento de pago.
●
Reporte de Pedidos - Estados y Definición de Pedido: El reporte de pedidos
tendrá un listado con filtr os y mostrará el estado del pedido interno. Se definió
que un "pedido" a efectos de este reporte es aquel que ya tiene un folio
asignado, cubriendo tanto crédito como prepago. Un "pedido confirmado" es
aquel para el que ya se ha generado la confirmación de pedido.
●
Reporte de Pedidos - Estados y Exclusiones: Se confirmó que los pedidos
que se muestran son aquellos que ya tienen conexión a compra y se
convirtieron en pedido (crédito o prepago). Los pedidos "intramitables" no se

reflejan en esta lista. Los estados a considerar en el reporte son "abierto"
(pendiente de entrega), "cerrado" (entregado) y se solicitó agregar el estado
"cancelado" (para folios que nunca continuaron).
●
Limitaciones de Cierre de Pedidos en el MVP: Biridiana Arias expuso un
problema donde los pedidos que tienen un reemplazo o backorder gestionado
por el proveedor se quedan en estado "abierto" a pesar de estar entregados.
Sara SAnchez aclaró que, para este MVP de pedidos prepago, el flujo de cierre
seguirá en Legacy y, por lo tanto, no habrá una conexión de regreso para
cerrarlos en Protifanet 2.
●
Transferencia de Datos y Visibilidad del Flujo: Sara SAnchez propuso que la
información deberá trasladarse a Legacy para que Biridiana Arias tenga
visibilidad completa del flujo de cierre. Valdemar Farina Sanchez sugirió que
la propuesta implicaría traer datos de la base de datos de Legacy para
completar los reportes en el sistema actual, lo cual depende de que el equipo
de Sara SAnchez proporcione las reglas de extracción. Sara SAnchez y
Valdemar Farina Sanchez confirmaron que habrá una conexión en ambos
sentidos para transferir pedidos a compras y traer información para reportes
actualizados.
●
Reglas de Cierre de Pedidos: Valdemar Farina Sanchez mencionó que el
marcado de pedidos como "cerrados" dependería de las reglas que Sara
SAnchez o el equipo SAP proporcionen. Si se proporcionan las reglas
correctas, el sistema puede marcar el pedido como cerrado en el escenario
mencionado por Biridiana Arias.
●
Funcionalidad de Agrupación/Ordenamiento en la Consulta: Biridiana Arias
preguntó si el sistema permite agrupar u ordenar la información por columna
(como fecha o cliente) para facilitar el análisis. Valdemar Farina Sanchez
confirmó que tienen considerada la funcionalidad de ordenar. Rose Ríos
Gómez aclaró la distinción entre agrupar y ordenar, confirmando que la
función prevista es ordenar.
●
Revisión del Detalle del Pedido (Pedidos sin Parciales): Roberto Baez Muñoz
presentó la vista de detalle de un pedido que no acepta parciales, mostrando
un resumen general con el folio, estatus, total, partidas, fecha de inicio, y el
estatus final de entrega. Se propuso un semáforo de estatus contra la fecha
estimada de entrega para indicar si el pedido se entregó a tiempo o con
retraso. Biridiana Arias confirmó que el semáforo debe ir contra la fecha
original, ya que los avisos de cambio podrían resultar en una entrega fuera del
tiempo comercial.
●
Manejo de Pedidos Programados en Clientes sin Parciales: Biridiana Arias
explicó que si un cliente que no acepta parciales envía una orden de compra

programada, la regla de "no parciales" se anula, y el pedido se convierte en
uno que sí acepta parciales. Sara SAnchez confirmó que la bandera de
"acepta parciales" del pedido se actualizaría antes de la transferencia de las
partidas si se realiza un cambio. Larissa Calvo mencionó que esta regla es
una mejora y no estaba dentro del alcance original del *release* (R16),
aunque se revisaría si se puede incluir en la estimación.
●
Información General del Pedido y Datos Financieros: Roberto Baez Muñoz
detalló el panel de información general que incluye datos del cliente, región,
comprador, tramitador, emisor, así como datos financieros como moneda,
condiciones de pago y fechas clave. Valdemar Farina Sanchez aclaró que los
campos de pagado y saldo dependen de si se incluirán pedidos a crédito, ya
que para prepago se acordó el pago completo.
●
Etiquetado del Campo "Comprador": Se discutió la etiqueta "Comprador," que
Roberto Baez Muñoz entendía como el contacto del cliente que envió la orden
de compra. Biridiana Arias explicó que la etiqueta debería coincidir con la
plantilla y que si el dato no está completo, se podría dejar como "usuario".
Valdemar Farina Sanchez confirmó que utilizarían el campo "puesto" y que, de
no tenerlo, se colocaría "usuario".
●
Trazabilidad de Partidas por Línea: Biridiana Arias enfatizó la necesidad de
ver la trazabilidad de la línea de tiempo (compra, importación, inspección,
facturación) por partida individualmente, incluso en pedidos que no aceptan
parciales. Esto se debe a que, aunque el pedido no acepte parciales, los
procesos como la compra e importación pueden ocurrir en momentos
diferentes para cada partida, dependiendo de los proveedores. Valdemar
Farina Sanchez reconoció que la compra, importación e inspección tienen
datos diferentes por partida, mientras que la recepción, tramitación,
facturación y envío son los mismos para todo el pedido que no acepta
parciales.
●
Modificaciones en la Facturación y Envío para Pedidos que No Aceptan
Parciales: Biridiana Arias explicó que, en un pedido de prepago que no acepta
parciales, si hay una línea en *back order*, el cliente puede solicitar una
modificación para abrir parciales, lo que detona el embalaje y la entrega de lo
disponible, aunque la factura original ya esté emitida. Valdemar Farina
Sanchez concluyó que se debe ver toda la trazabilidad de cada una de las
líneas, aunque algunos datos sean semejantes, y Biridiana Arias se ofreció a
enviar un video para detallar esta información.
●
Visualización para Pedidos que Aceptan Parciales: Roberto Baez Muñoz
presentó la vista para pedidos que aceptan parciales, donde la trazabilidad se
manejará por partida individual, incluyendo un resumen de entrega para cada

una. Se mencionó que se complementará la información de cada estado
(Recepción, Tramitación, Compra, Importación) basándose en un video que el
equipo de Biridiana Arias proporcionará.
●
Ajustes en el Detalle de Estatus de Pedido: Se hicieron varios ajustes en la
información mostrada en los estatus: en Recepción se acordó usar solo la
"fecha de recepción". En Tramitación, se mantuvo el uso del usuario del
sistema (ej. "Bilchis"). En Compra, se incluyeron detalles como proveedor,
comprador, guía de embarque y factura del proveedor, y Biridiana Arias
confirmó la necesidad de ver el PDF de la orden de compra. En Importación,
se agregó la fecha estimada de arribo y se decidió colocar el número de
pedimento, aunque el documento real solo es visible hasta la inspección en el
sistema actual.
●
Información y Documentación en Inspección y Facturación: En Inspección
Matriz, se incluyeron la fecha de inicio y fecha final de inspección, el nombre
del inspector, resultado y observaciones. Se solicitó incluir el certificado y la
hoja de seguridad como documentos para quien aplique. En Facturación, se
incluirán datos como el número de factura, el UID, el método de pago y los
documentos de facturación (factura y complemento de pago). Valdemar
Farina Sanchez sugirió que la información de carga al portal y comprobante
debe estar en el reporte de facturas, con solo un enlace en el detalle del
pedido.
●
Detalle de Envío y Trazabilidad por Partida: El estatus de Envío incluirá la
fecha de entrega/envío, número de guía, dirección y documentos como acuse
y documento de conformidad. Se confirmó que la visualización final de
partidas, tanto para clientes que aceptan como para los que no aceptan
parciales, será por partida individual, incluyendo detalles como cotización,
presentación y estatus.
●
Facturación y Cobranza (Visualización de Documentos): En la sección de
Facturación y Cobranza, Roberto Baez Muñoz explicó que se mostrarán
documentos como la Proforma, la Factura (con su estado SAT) y, para
pedidos a parciales, el desglose de partidas asociadas a cada factura.
Valdemar Farina Sanchez confirmó que se manejaría de manera similar a la
Proforma para los prepagos, incluyendo la remisión si se utilizó una para la
entrega.
●
Próximos Pasos y Cierre de la Preventa: El equipo acordó enviar el video
solicitado para complementar los ajustes de la trazabilidad y generar las
propuestas finales. Rose Ríos Gómez indicó que habrá una siguiente y
esperada última sesión para la validación final de los ajustes. Se enfatizó que
el equipo de Larissa Calvo y Sara SAnchez debe trabajar en paralelo para

resolver todos los pendientes que pudieran impactar en los ajustes
presentados.


Revisa las notas de Gemini para asegurarte de que sean precisas. Obtén sugerencias y
descubre cómo Gemini toma notas
Cómo es la calidad de estas notas específicas? Responde una breve encuesta para
darnos tu opinión; por ejemplo, cuán útiles te resultaron las notas.
