


20 mar 2026
R16 Adquisiciones - 1ra sesión de
entendimiento
Invitados
Rose Ríos GómezJuan David García Cruzblcalvo@proquifa.net

Roberto Baez MuñozIrma Andrade Aguadossanchez@proquifa.net

Valdemar Farina SanchezAlan Fernandez Garcia
Archivos adjuntos
R16 Adquisiciones - 1ra sesión de entendimiento
Registros de la reunión
Grabación

Resumen
La revisión de la matriz de requisitos se centró en la claridad del
lenguaje, con una decisión clave sobre la gestión de términos del
proyecto y el alcance regional.

Clarificación de la Nomenclatura del Módulo
La matriz de requisitos inició una discusión sobre el nombre del módulo
'Validar Pago', y el equipo de Proquifa acordó utilizar el término 'validar
cobro' para el proceso que realiza el cliente. Valdemar Farina Sanchez no
estuvo de acuerdo, lo que llevó al equipo a dejar pendiente la definición
final del nombre del módulo internamente.

Acuerdo sobre Términos Regionales
Se discutió la necesidad de un glosario de términos específicos,
resolviéndose con la propuesta de compartir las políticas de Tesorería
para la cobranza en lugar de crear un glosario formal. Se confirmó que
Perú está en el alcance de R16, y se decidió investigar con el área de
finanzas si el proveedor de timbrado actual puede realizar facturación
electrónica en Perú.

Propuesta de Flujo de Factura Adelantada
Se acordó que el equipo de Valdemar Farina Sanchez revisará el flujo de

'factura por adelantado' para clientes de crédito y presentará una
contrapropuesta para ubicar la opción en el módulo 'tramitar pedido'. Se
confirmaron las reglas de facturación, incluyendo que la factura por
adelantado no se generará para pedidos con sustancias controladas.

Detalles
●
Inicio de Sesión y Compartición de Documentos: La sesión inicial tuvo como
objetivo el entendimiento del proyecto R16es, centrado en adquisiciones y
pagos, con la guía de Roberto Baez Muñoz. Se comenzó la revisión de la
matriz de requisitos enfocándose en la nomenclatura y conceptos clave, en
lugar de ir requisito por requisito, para evitar confusiones. Roberto Baez
Muñoz compartió el documento de requisitos para la revisión detallada de las
preguntas.
●
Clarificación del Módulo "Validar Pago": Surgió una duda sobre el nombre del
módulo "validar pago" (requisito 2 de la matriz 2), ya que el sistema podría
manejar procesos de cobro (hacia clientes) y de pago (hacia proveedores), lo
que podría generar confusión en el futuro. Se aclaró que todos los requisitos y
las matrices compartidas se enfocan en la parte de cobros por parte del
cliente.
●
Discusión sobre la Nomenclatura de Cobros vs. Pagos: Se confirmó la
necesidad de establecer un nombre final para el módulo y se sugirió ampliar
el nombre a "validar pago del cliente" para mayor especificidad. Rose Ríos
Gómez instó a tomar una decisión rápida entre "validar cobro" o "validar pago
del cliente". El equipo de Proquifa estuvo de acuerdo en que el término
"validar cobro" podría ser más adecuado, ya que se refier e al cobro que se
realiza al cliente, aunque el pago sea efectuado por el cliente.
●
Discrepancia en el Término "Validar Cobro": Valdemar Farina Sanchez no
estuvo completamente de acuerdo con "validar cobro", argumentando que la
validación es del pago que el cliente ya realizó. Si el módulo unifica la gestión
de cobranza y la validación del pago, Valdemar Farina Sanchez sugirió que el
problema podría ser la palabra "validar", proponiendo alternativas como
"ejecutar cobranza" o "gestionar cobranza". El equipo de Proquifa acordó
llevarse como tarea pendiente la definición interna del nombre del módulo
con las áreas correspondientes.
●
Definición de Términos del Proyecto: Roberto Baez Muñoz planteó la
necesidad de definir conceptos específicos como "monto aplicado", "saldo

disponible", y "saldo a favor" para asegurar la comprensión de todo el equipo y
evitar malinterpretaciones de la matriz de requisitos. Se solicitó un pequeño
glosario o una definición clara de estos términos. Rose Ríos Gómez enfatizó
la importancia de establecer acuerdos sobre los compromisos de Proquifa
respecto a la estandarización de términos.
●
Acuerdo sobre el Glosario de Términos: Sara SAnchez expresó que la
elaboración de un glosario no sería factible debido a la alta carga de trabajo,
ya que los conceptos están ligados a la funcionalidad. Larissa Calvo propuso
que, en lugar de un glosario formal, se podrían compartir las políticas de
Tesorería relativas a la cobranza, y que las dudas muy específicas se podrían
consultar con personal de Tesorería en sesiones futuras. Rose Ríos Gómez
aceptó esta propuesta para evitar cargar de tiempo al equipo de Proquifa,
acordando que se partirá de conceptos genéricos de la industria y que las
dudas puntuales se resolverán más adelante.
●
Alcance del Proyecto R16es para la Región Perú: Se abordó el alcance y la
configuración por región, ya que el requisito 1 menciona el trámite de pedidos
para clientes prepago tanto en México como en Perú, pero los documentos de
la matriz 2 (CFDI, SAT, Turbopac) son específicos para México. Larissa Calvo
confirmó que Perú sigue siendo parte del alcance de R16. La pregunta central
es si la región Perú compartirá el flujo de facturación de México, y si no, qué
documentos y validaciones aplicarían, una información que se acordó llevar
con el área de finanzas.
●
Facturación Electrónica y Proveedores de Servicio: Sara SAnchez explicó que
la facturación electrónica en México seguirá utilizando al proveedor Turbo
Pack, y se investigará si este proveedor puede realizar timbrados en Perú. Si
no es posible, se evaluará un nuevo proveedor con Finanzas. Valdemar Farina
Sanchez confirmó que se incluirá la facturación electrónica en Panet 2 y
solicitó la documentación existente sobre el proceso de timbrado para
agilizar el desarrollo.
●
Compromiso y Documentación de Timbrado: Sara SAnchez indicó que no
existe documentación formal avanzada, pero que el conocimiento podría
transmitirse en sesiones, y se podría elaborar un documento de alto nivel. El
compromiso es entregar a más tardar el martes 24 un documento de alto
nivel sobre el sistema de timbrado de México para que el equipo de desarrollo
pueda dimensionar el esfuerzo de facturación.
●
Transferencia de Datos para Perú (Legacy): Se abordó la pregunta sobre si el
Release 16 incluiría la transferencia de datos de Perú a la plataforma Legacy,
ya que actualmente no se realiza. Larissa Calvo aclaró que las operaciones de

la región Perú no tendrán ningún tipo de conexión con ninguna plataforma
Legacy. Respecto al flujo de Perú, Valdemar Farina Sanchez preguntó si se
mantendría el reporte de pedidos tramitados y si se actualizaría para reflejar
la cobranza, a lo que Sara SAnchez respondió que se considera parte de los
reportes en la matriz de requisitos 3.
●
Validaciones del RUC en Perú: Se señaló que el RUC (equivalente al RFC en
México) en Perú no tiene ninguna validación en el sistema actual, a diferencia
del RFC en México. Sara SAnchez consideró que sí se deben proporcionar los
parámetros para la validación del RUC. Además, Valdemar Farina Sanchez y
Sara SAnchez acordaron que, dado que se incluirá la facturación para Perú,
los datos fiscales (como tipo de sociedad mercantil y régimen fiscal) deben
ser separados para regionalizarlos en un catálogo propio de Perú.
●
Flujo de Factura por Adelantado (Clientes de Crédito): Roberto Baez Muñoz
preguntó qué sucede con los clientes de crédito que pasan por el flujo de
factura por adelantado. Sara SAnchez aclaró que, a diferencia de los
prepagos, el cliente de crédito avanzaría sin ningún bloqueo hacia el módulo
de tramitar pedido, y se generarían dos pendientes, uno de los cuales podría
ser la factura por adelantado.
●
Restricciones de Factura por Adelantado: Se consultó si un cliente de crédito
en el flujo de factura por adelantado aplicaría las mismas restricciones (por
ejemplo, por sustancias controladas o validación fiscal) que un cliente
prepago. Sara SAnchez indicó que el deber ser es que sí deberían aplicar
todas las reglas, ya que se genera una factura por adelantado, pero solicitó
más información sobre las restricciones específicas para confirmar.
●
Conceptualización del Flujo de Factura por Adelantado: Valdemar Farina
Sanchez solicitó aclarar el flujo de "factura por adelantado" para entender el
contexto, diferenciando el proceso normal del de prepago. Se confirmó que en
el flujo de prepago, la confirmación de pedido se emite después de recibir y
validar el pago, y se usa la factura ya generada en lugar de facturar en
almacén. El objetivo de la factura por adelantado en clientes de crédito es
justificar fiscalmente el gasto de presupuesto, asumiendo que los días de
crédito comienzan a correr al generarse la factura.
●
Momento de Solicitud de Factura por Adelantado (Crédito): Biridiana Arias
explicó que la solicitud de factura por adelantado puede ocurrir en cualquier
punto del proceso (desde la cotización hasta un aviso de cambio en la
compra), especialmente si la entrega se retrasará y se necesita justificación
fiscal. Actualmente, este proceso no está sistematizado y requiere levantar un
ticket.

●
Propuesta para el Botón de Factura por Adelantado: Sara SAnchez mencionó
que se habilitaría un botón de "factura por adelantado" en el módulo de
pretramitar, con la restricción de que no se habilite para pedidos con
productos controlados. Valdemar Farina Sanchez sugirió que el botón debería
estar en el módulo de "tramitar pedido" en lugar de "pretramitar",
argumentando que este último está diseñado para atender inconsistencias, y
la colocación ahí generaría complejidad y más escenarios a manejar.
●
Acuerdo para la Revisión de la Propuesta de Flujo: Sara SAnchez estuvo de
acuerdo en que la idea de Valdemar Farina Sanchez de ubicar la opción en
"tramitar pedido" es buena, pero debe ajustarse a las reglas establecidas en la
matriz, como el cálculo de la fecha estimada de entrega. Se acordó que el
equipo de Valdemar Farina Sanchez revisará los requisitos de cumplimiento y
presentará una propuesta para este flujo.
●
Flujo para Clientes de Crédito sin Factura Adelantada: Roberto Baez Muñoz
preguntó qué sucede con los clientes de crédito que no seleccionan la opción
de "factura por adelantado". Sara SAnchez confirmó que no sufrirían ningún
cambio y continuarían con el flujo normal (directamente a tramitar pedido).
●
Clarificación de "Folio de Pedido": Se preguntó a qué se refier e el "folio de
pedido" (requisito 15, matriz 1) que se generaría automáticamente para
clientes prepago. Sara SAnchez y Valdemar Farina Sanchez confirmaron que
se trata del "pedido interno". Rose Ríos Gómez determinó que las preguntas
relacionadas con la ubicación (pretramitar o tramitar) de la generación de
este folio quedarían pendientes hasta que se revise la contrapropuesta de
flujo, con el objetivo de evitar complejidades innecesarias.
●
Requisito de Folio de Pedido Interno: Rose Ríos Gómez comentó que la
necesidad inmediata es una solución que funcione bien y pronto. Valdemar
Farina Sanchez consultó la razón para generar un folio de pedido interno, ya
que este folio se plasma en la proforma y en la factura. Biridiana Arias aclaró
que en la factura actualmente no aparece, pero al emitir la proforma o factura
por adelantado, el folio ya existe y permite la visualización en la consulta de
pedidos.
●
Utilidad del Folio para Trazabilidad y Cobranza: Sara SAnchez y Larissa Calvo
explicaron que el folio actúa como un control y un medio de comunicación
esencial entre el área de Servicio al Cliente (SAC) y Cobranzas, ayudando a
identificar el trámite y seguir el proceso de cobranza. También es crucial para
la trazabilidad y consulta de pedidos para ver el flujo en el que se encuentra el
pedido. Valdemar Farina Sanchez planteó si el número de orden de compra

podría servir, a lo que Sara SAnchez respondió negativamente, reafirmando
que el folio es el dato de referencia.
●
Definición de "Pedido en Firme": Valdemar Farina Sanchez distinguió entre
generar un folio y tener un pedido en firme, indicando que un pedido en firme
implica la confirmación del pedido. Sara SAnchez solicitó definir claramente
qué se considera un pedido para cumplir con la consulta de requisitos, que
debe visualizar la trazabilidad de cualquier pedido (prepago o crédito)
mediante el folio ya generado. Valdemar Farina Sanchez propuso aclarar si se
incluyen pedidos confirmados, pedidos en curso, o todos los estados,
reconociendo que un alcance más amplio tardaría más.
●
Trazabilidad de Pedidos Abiertos y Confirmados: Biridiana Arias señaló que
actualmente existe una confirmación de pedido, y en el caso de prepagos sin
pago validado, la única diferencia es la falta de una fecha estimada de
entrega, aunque se monitorea como un pedido abierto. Valdemar Farina
Sanchez aceptó la sugerencia de manejar las categorías "Pedidos abiertos" y
"Pedidos confirmados" para avanzar en la claridad.
●
Cambio de Condición de Prepago a Crédito: Roberto Baez Muñoz introdujo la
pregunta sobre el requisito de permitir el cambio de un pedido de prepago a
crédito, con la obligatoriedad de ingresar un código de autorización de
Tesorería. Biridiana Arias confirmó que el cambio solo es unidireccional (de
prepago a crédito) y es exclusivo a nivel trámite. Rose Ríos Gómez y Roberto
Baez Muñoz confirmaron que este cambio solo implica modificar el tipo de
cobro a nivel pedido.
●
Datos Solicitados en el Cambio de Condición: Roberto Baez Muñoz preguntó
si al cambiar a crédito se seguirían las reglas de configuración del cliente,
como solicitar forma y número de cuenta. Valdemar Farina Sanchez y Sara
SAnchez coincidieron en que solo se requeriría la forma de pago, no el
número de cuenta. Biridiana Arias mencionó que, según los análisis de
Cornelio, la forma de pago cambia automáticamente de PUE a PPD.
●
Transferencia de Datos de Forma de Pago para Facturación: Valdemar Farina
Sanchez planteó la necesidad de transferir los datos de la forma de pago a
*legacy* o verificar si se pueden capturar en la factura, a lo que Sara SAnchez
indicó que se tendrían que transferir al no tener forma de capturarlos. Se
acordó que al cambiar de prepago a crédito, se solicitará la forma de pago
para la factura. Valdemar Farina Sanchez señaló que, debido a la
transferencia de datos en el pedido, será necesario ajustar la transferencia
para que tome la forma de pago desde el pedido y no del catálogo de clientes.

●
Visualización y Modificación del Destinatario en la Proforma: Roberto Baez
Muñoz preguntó si la copia del destinatario de Servicio al Cliente (ESAC) que
va en el correo de la proforma se mostraría en pantalla y si sería editable.
Sara SAnchez confirmó que sí debería mostrarse en pantalla y que se podría
modificar o eliminar de acuerdo con las funcionalidades actuales del sistema.
●
Restricciones de Factura Anticipada para Sustancias Controladas: Roberto
Baez Muñoz notó una contradicción entre los requisitos: el módulo de
facturación por adelantado bloquea la generación de facturas para sustancias
controladas, mientras que el módulo de validar pago contempla la emisión de
una factura anticipada si no hay una factura para sustancias controladas.
Sara SAnchez explicó que son dos conceptos diferentes: para sustancias
controladas no debe habilitarse la opción de "factura por adelantado".
●
Flujo de Pago con Sustancias Controladas (Prepago): Sara SAnchez detalló
que si un pedido prepago tiene sustancias controladas, se debe ir por el canal
de la proforma, validar el pago y, al validar el pago, generar una factura
anticipo. Larissa Calvo y Sara SAnchez explicaron que una factura normal no
es viable porque aún no se tienen datos fiscales obligatorios como el
pedimento o la aduana.
●
Resumen de Flujos de Facturación y Cobranza: Valdemar Farina Sanchez
resumió los flujos: para clientes de crédito sin controlados se genera factura
por adelantado si se solicita, pero si tiene controlados, no se genera y solo
sale la confirmación de pedido. Para prepagos sin controlados, se genera
proforma, se recibe el pago y se genera una factura normal. Para prepagos
con controlados, se genera proforma, se recibe el pago y se genera una
factura anticipo.
●
Generación de Complemento de Pago: Sara SAnchez aclaró que cuando
existe una factura por adelantado y luego se valida el pago, se emite un
Comprobante Fiscal Digital por Internet (CFDI) o complemento de pago. Se
especificó que un complemento de pago se genera solo si ya existe una
factura previa; de lo contrario, se genera la factura (normal o anticipo,
dependiendo del caso). Valdemar Farina Sanchez y Sara SAnchez
confirmaron que, en el caso de prepago, el pago debe ser completo.
●
Validación de Datos Fiscales para Facturación: Roberto Baez Muñoz preguntó
qué campos exactos constituyen la "información fiscal completa" para
generar la factura. Valdemar Farina Sanchez sugirió que este punto se
aclararía cuando Sara SAnchez compartiera la información requerida para la
facturación.

●
Manejo de Fallas en el Timbrado con Turbo PAC: Ante una falla de timbrado
con el proveedor Turbo PAC, Sara SAnchez indicó que el sistema debe
mostrar el error devuelto por el proveedor para que el usuario pueda corregir y
volver a intentarlo. Valdemar Farina Sanchez solicitó a Sara SAnchez
compartir el catálogo de restricciones (por ejemplo, combinaciones
incompatibles de métodos y formas de pago) para implementar validaciones
desde el sistema.
●
Aplazamiento de Revisión del Flujo de Validar Pago: Debido a la complejidad
del tema de pagos, saldos disponibles y facturas, Valdemar Farina Sanchez
sugirió manejar este punto en una sesión técnica aparte para no extender
demasiado la reunión. Sara SAnchez estuvo de acuerdo en revisar todas las
validaciones relacionadas con pagos en una sesión dedicada.
●
Roles y Funciones en el Flujo de Cobranza: Valdemar Farina Sanchez
consultó si la función de cobranza (factura, cobro, validación de cobro) era
ejecutada por el mismo rol o por funciones distintas. Larissa Calvo y Sara
SAnchez confirmaron que es el mismo rol (gestor de pagos) y mencionaron
dos funciones que ejecutan las mismas actividades: especialista de cuentas
por cobrar y analista de cuentas por cobrar (senior y junior), con diferencias
solo en el grado de complejidad de la cartera. También se confirmó que se
manejarán carteras.
●
Diseño y Estructura de los Documentos PDF: Valdemar Farina Sanchez
preguntó sobre el diseño de los documentos (factura normal, factura anticipo,
complemento de pago). Rose Ríos Gómez clarificó que se necesitan cuatro
documentos: proforma, factura normal, factura anticipo y complemento de
pago, y que la propuesta es que la estructura sea la misma, cambiando solo el
*branding* (imagen corporativa) por empresa y por región. Sara SAnchez
propuso que el diseño sea desarrollado por la fábrica de *software*.
●
Inconsistencias en Pagos: Roberto Baez Muñoz preguntó sobre el manejo de
inconsistencias en el pago, si existe un catálogo de inconsistencias, y si el
pago con inconsistencia se elimina, dejando algún registro de auditoría. Rose
Ríos Gómez sugirió que lo ideal sería tener un catálogo de inconsistencias
para mejorar la comunicación con el cliente. Sara SAnchez aclaró que un
nuevo pago correcto reemplazaría al pago con inconsistencia, y que el área
de Pagos será responsable de alinear el pago con las facturas o proformas
correspondientes.
●
Manejo de Notas de Crédito: Roberto Baez Muñoz preguntó sobre el origen y
aplicación de las notas de crédito. Sara SAnchez indicó que el alcance inicial
es solo para las notas generadas en *Prokifanet 2*, excluyendo las de

*legacy*. Rose Ríos Gómez sugirió profundizar en este tema con el área
financiera, ya que la cancelación no es el único mecanismo para generar una
nota de crédito en la industria, y advirtió sobre el riesgo de no visualizar las
notas de crédito provenientes de sistemas *legacy*. Larissa Calvo se
comprometió a revisar el tema de la generación de notas de crédito con
Finanzas.
●
Captura de Datos en el Buzón de Pagos: Respecto al buzón de pagos, Roberto
Baez Muñoz consultó qué datos deben capturarse del correo para procesar el
pendiente en "validar pago". Sara SAnchez explicó que, dado que el pendiente
se generaría automáticamente sin datos (similar al flujo de cotizaciones),
solo se consideró el monto. Se acordó que la captura del monto del pago se
realizaría en la primera vez que se use el pago, dentro del módulo de
cobranza.
●
**Programación de Próxima Sesión y Cambio de Nombre del *Release***:
Rose Ríos Gómez sugirió cerrar la agenda debido a la restricción de tiempo y
programar una sesión técnica para el día martes, si es posible, para revisar
los pendientes. Rose Ríos Gómez también sugirió cambiar el nombre del
*release* de "R16 Adquisiciones" a algo que represente mejor el alcance,
como "Tramitar Pedido Sin Crédito", nombre que ya está siendo utilizado
internamente.

Pasos siguientes recomendados

Larissa Calvo definir  á completamente el nombre de la pantalla que llevará a
cabo la ejecución de la cobranza y la validación de pago, ya que esto es parte
del argot del área.

Larissa Calvo y Sara SAnchez compartirán las políticas que tiene el área de
tesorería respecto a la cobranza, en lugar de crear un glosario de términos.

Sara SAnchez investigará si TurboPac tiene la facultad para ejecutar
timbrados en Perú y, si no, evaluará al proveedor con el área de Finanzas.

Sara SAnchez entregará la documentación de alto nivel del proceso de
timbrado en México (incluyendo la documentación Swagger) a más tardar el
martes 24 al final del día.

Sara SAnchez se llevará la anotación para obtener y proporcionar la data de
las opciones que corresponden a los catálogos del tipo de sociedad mercantil
y el régimen fiscal para Perú.


Sara SAnchez hará el análisis con el área de finanzas para proporcionar solo
las opciones de usos aplicables dentro del negocio para México y Perú.

Valdemar Farina Sanchez revisará los puntos de la matriz de requisitos que se
necesitan cumplir y analizará para hacer una propuesta sobre la ubicación del
botón de factura por adelantado.

Sara SAnchez investigará las restricciones de incompatibilidad de métodos
de pago y forma de pago definidas por el SAT, o aquellas que han
experimentado, para compartirlas con el equipo.

Larissa Calvo revisará con tesorería el posible catálogo de inconsistencias de
pago.

Larissa Calvo volverá a tratar con el área de finanzas el tema de la generación
de notas de crédito para afinar su visualización.

Larissa Calvo y Sara SAnchez gestionarán que se muestre al equipo Ringdem
cómo funcionaba el buzón de pagos anteriormente en SAP para mayor
claridad.

Revisa las notas de Gemini para asegurarte de que sean correctas. Obtén consejos y
descubre cómo toma notas Gemini
Danos tu opinión sobre el uso de Gemini para tomar notas en una breve encuesta.
