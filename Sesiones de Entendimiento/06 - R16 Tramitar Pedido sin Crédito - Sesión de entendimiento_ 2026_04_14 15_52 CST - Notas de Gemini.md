


abr 14, 2026
R16 Tramitar Pedido sin
Crédito - Sesión de
entendimiento
Invitado
barias@proquifa.netRose Ríos GómezJuan David García Cruz

blcalvo@proquifa.netmfranco@proquifa.netRoberto Baez Muñoz

Irma Andrade AguadoAlejandro Cervantes Florescmramirez@proquifa.net

ssanchez@proquifa.netValdemar Farina SanchezAlan Fernandez Garcia
Archivos adjuntos
R16 Tramitar Pedido sin Crédito - Sesión de entendimiento
Registros de la reunión
Grabación


Resumen

Revisión exhaustiva del módulo de notas de crédito y su funcionalidad,
determinando flujos de trabajo, ajustes de filtros y la necesidad de
diferenciar la devolución de dinero.

Diseño y listado de notas
Se revisó la propuesta visual del módulo de notas de crédito, acordando
que el folio mostrará el PDF y la columna XML permitirá la descarga. La
trazabilidad completa desde la nota de crédito no se consideró
prioritaria, ya que el punto de partida es el pedido.

Creación y selección de facturas
Se definió el flujo de creación de notas de crédito, donde se deben
mostrar todas las facturas de clientes de prepago con una antigüedad
menor a 5 años. Se tomó la decisión de añadir un filtro de moneda y
retirar la columna de 'saldo disponible' para evitar confusiones.


Reglas de cancelación y folios
Se acordó que las notas de crédito solo deben enfocarse en la
generación y no en la devolución de dinero, siendo sus motivos base:
devolución, bonificación o error de facturación. Se establecieron reglas
para la cancelación de facturas vía nota de crédito: debe ser total y
ocurrir dentro del mes de emisión.


Próximos pasos

[Larissa Calvo] Evaluar Trazabilidad: Discutir requerimiento de trazabilidad de
notas de crédito con Mayra Franco y Daniel. Priorizar funcionalidad inicial
antes de enriquecer vista.

[Mayra Franco] Revisar Requerimiento: Revisar sugerencia de trazabilidad
para evitar saltos entre módulos. Comentar observaciones al equipo de
desarrollo.

[Valdemar Farina Sanchez] Definir Estados: Acercarse a Cornelio M Ramirez B
y Mayra Franco para definir estados de notas de crédito. Determinar si son
estados del SAT o internos.

[Sara SAnchez, Larissa Calvo] Estandarizar Folios: Revisar y estandarizar la
estructura de foliado de los diferentes documentos. Asegurar un formato
consistente para notas y facturas.

[Sara SAnchez] Definir Reglas: Apoyar en la definición de las reglas
específicas para el consecutivo de folios. Incluir información sobre reinicio o
número máximo en el diccionario de datos.

[Sara SAnchez] Evaluar Folios: Evaluar reglas generación folios nuevos
prepagos. Verificar compatibilidad caracteres sistema Legacy.

[Larissa Calvo, Irma Andrade Aguado] Agendar Revisión: Agendar sesión
específica para revisar reportes pendientes. Coordinar disponibilidad 4 PM a
6 PM, 2 horas estimadas.


Detalles
●
Revisión de la Agenda y Módulo de Notas de Crédito: Roberto Baez Muñoz
comenzó la sesión confirmando la visibilidad de su presentación y describió

los temas de la agenda, que incluyen el módulo de notas de crédito y los
módulos de informes de pedidos, facturas y cobros. La sesión se centrará
primero en el módulo de notas de crédito, donde se implementaron funciones
basadas en una investigación sobre cómo operan estas notas.
●
Propuesta Visual y Funcionalidad del Módulo de Notas de Crédito: Roberto
Baez Muñoz presentó la propuesta visual del nuevo módulo, que mostrará
todas las notas de crédito generadas con la opción de crear nuevas. El
módulo incluye filtr os por rango de fechas, cliente, emisor, estado (aplicada o
disponible), y una función de búsqueda por factura o pedido interno.
●
Detalles del Listado de Notas de Crédito: El listado de notas de crédito es una
tabla clasificada por fecha, cliente, cobrador, folio, XML, emisor, monto,
factura asociada, pedido interno asociado y estado. Cornelio M Ramirez B
preguntó si la columna XML mostraría el PDF de la factura o el código XML, a
lo que Roberto Baez Muñoz respondió que se podría cambiar a PDF si fuera
prioritario.
●
Funcionalidad de XML y PDF en el Listado: Sara SAnchez sugirió que al hacer
clic en el folio de la nota de crédito se podría visualizar el PDF, y Cornelio M
Ramirez B preguntó si el XML permitiría la descarga del archivo. Se acordó
que el campo de nota de crédito (folio) permitirá ver el PDF, y la columna XML
permitirá descargar el archivo XML, ya que los clientes a menudo requieren
ambos.
●
Trazabilidad del Proceso desde la Nota de Crédito: Mayra Franco sugirió
añadir una funcionalidad en el módulo de notas de crédito para ver un "árbol
genealógico" o trazabilidad completa desde la orden de compra, la entrega de
mercancía y la factura. Valdemar Farina Sanchez explicó que la trazabilidad
completa se considera que parte del pedido, y que no se identificó un
escenario en el que la nota de crédito sea el punto de partida central para este
tipo de trazabilidad.
●
Diferencias en la Trazabilidad por Módulo: Mayra Franco argumentó que el
objetivo es facilitar el trabajo al evitar cambiar de módulo para buscar
información. Sara SAnchez indicó que la trazabilidad se puede obtener
utilizando los folios de factura y pedido interno proporcionados en la consulta
de notas de crédito y luego consultando el módulo de pedidos. Larissa Calvo
sugirió que este tema se discuta específicamente con Daniel, priorizando la
funcionalidad actual antes de enriquecerla.
●
Revisión de los Estados de las Notas de Crédito: Cornelio M Ramirez B
preguntó sobre los estados "por aplicar" y "vigente" que se muestran en la
pantalla. Valdemar Farina Sanchez mencionó que se acercarán a alguien de
finanzas, como Mayira o Cornelio M Ramirez B, para definir los estados de las

notas de crédito, ya sean del Servicio de Administración Tributaria (SAT) o
internos.
●
Proceso de Creación y Selección de Factura para Nota de Crédito: Roberto
Baez Muñoz presentó el proceso de creación de notas de crédito, el cual
comienza con un botón para avanzar a una pantalla de cuatro pasos, siendo
el primero la selección de una factura a la que se desea relacionar la nota de
crédito. La tabla de facturas incluye folio, UUID del SAT, RFC, cliente, razón
social, fecha, total, saldo disponible y estado.
●
Especificación de Moneda en la Tabla de Facturas: Sara SAnchez y Mayra
Franco solicitaron que el monto total de la factura incluya la moneda (MXN o
USD) para mejor referencia, ya que no todos los clientes facturan en dólares.
Valdemar Farina Sanchez coincidió en que la lista debe mostrar el total en la
moneda en que se facturó, ya que el objetivo principal es identificar la factura
a la que se generará la nota de crédito.
●
Propuesta de Filtro y Columna de Moneda: Mayra Franco sugirió añadir una
columna de moneda y un filtr o para seleccionar solo facturas en pesos o solo
en dólares, para facilitar la identificación y el manejo visual. Valdemar Farina
Sanchez acordó agregar el filtr o de moneda, junto con los datos de cliente,
RFC y fecha, que son los principales utilizados para buscar facturas.
●
Aclaración sobre la Columna "Saldo Disponible": Sara SAnchez preguntó por
la utilidad y significado de la columna "saldo disponible". Valdemar Farina
Sanchez explicó que este saldo se refier e a lo que el cliente aún debe, y se
agregó bajo el supuesto de que solo se pueden generar notas de crédito para
facturas que no han sido completamente pagadas.
●
Condiciones para Mostrar Facturas Candidatas a Nota de Crédito: Cornelio M
Ramirez B confirmó que el saldo disponible se refier e al saldo insoluto o el
saldo por pagar, como cuando se aplica un pago parcial a una factura.
Cornelio M Ramirez B aclaró que las notas de crédito nacen de cancelaciones
parciales o devoluciones de producto en clientes de prepago, lo que implica
que el saldo disponible no es necesariamente un factor limitante, ya que la
mayoría de los clientes ya han pagado.
●
Definición de Facturas Candidatas para Notas de Crédito: Valdemar Farina
Sanchez buscó definir la condición para que una factura se muestre como
candidata; Cornelio M Ramirez B indicó que se deben mostrar todas las
facturas de clientes de prepago. Se estableció que el límite de visualización
debe ser de 5 años por obligación mercantil.
●
Ajustes de Filtro y Retiro de Columna "Saldo Disponible": Se acordó que la
lista de facturas debería filtr arse primero por cliente para reducir la cantidad

de datos mostrados. Cornelio M Ramirez B sugirió retirar la columna "saldo
disponible" ya que podría causar confusión al tratarse de una deducción o
disminución del total de la factura.
●
Clarificación del Estado de la Factura y Facturas Canceladas: Sara SAnchez
preguntó sobre el significado de la columna "estado". Valdemar Farina
Sanchez explicó que es el estado de la factura según el SAT, y Cornelio M
Ramirez B confirmó que las facturas canceladas no pueden generar una nota
de crédito, pero sí pueden estar relacionadas a ellas.
●
Escenarios y Reglas de Negocio para Notas de Crédito y Devoluciones:
Cornelio M Ramirez B y Valdemar Farina Sanchez discutieron escenarios
donde se genera una nota de crédito, incluyendo cuando no es la totalidad de
la factura (la factura queda vigente) o cuando sí es la totalidad. Si la
cancelación es total, se puede optar por cancelar la factura y hacer una
devolución (dentro del mismo mes) o cancelar y hacer una nota de crédito
para que el cliente use el saldo en una compra futura.
●
Alcance del Módulo en el Lanzamiento Actual: Larissa Calvo y Valdemar
Farina Sanchez confirmaron que el alcance de este lanzamiento es
exclusivamente para clientes de prepago. Sara SAnchez sugirió considerar la
homologación para el concepto de crédito en el futuro.
●
Propuesta de Pantalla de Captura de Datos de la Nota de Crédito: Roberto
Baez Muñoz presentó la pantalla de captura de datos, la cual muestra la
información de la factura original y los campos para la nota de crédito. Los
campos de la nota de crédito incluyen motivo (catálogo del SAT), tipo de
relación (no editable), uso de CFDI y el monto de la nota de crédito.
●
Inclusión de la Opción de Devolución en la Pantalla de Captura: Valdemar
Farina Sanchez preguntó si, en el caso de que la nota sea por la totalidad de
la factura, se debe incluir la opción de hacer una devolución además de
generar la nota de crédito. Mayra Franco confirmó que es importante tener la
opción de devolución, aunque la prioridad sea retener al cliente mediante el
saldo a favor.
●
Proceso y Evidencia de Devolución: Mayra Franco describió el proceso de
devolución, el cual comienza con una solicitud de Servicio al Cliente (SAC)
donde se llena un formato con datos bancarios del cliente y la factura
referida, con un correo electrónico del cliente como evidencia de soporte.
Rose Ríos Gómez aclaró que una nota de crédito es el mecanismo principal
de Proquifa para evitar devolver dinero al cliente cuando hay una devolución
de producto.

●
Clarificación de Devolución de Producto vs. Devolución de Dinero: Rose Ríos
Gómez solicitó clarificar si el concepto de "devolución" se refería a la
devolución del producto o a la devolución del dinero al cliente. Valdemar
Farina Sanchez confirmó que la discusión inicial se centraba en la devolución
de dinero, debido a la confusión sobre si esta solo aplicaba a la totalidad de la
factura o si las opciones de nota de crédito o devolución de dinero estaban
disponibles en cualquier caso de devolución de dinero.
●
Determinación de Opciones de Devolución: El equipo discutió que el sistema
debería considerar ambos caminos (nota de crédito o devolución de dinero)
debido a que los clientes podrían solicitar la cancelación de la nota de crédito
y la devolución de su dinero. Cornelio M Ramirez B. señaló que, si bien la nota
de crédito es la opción preferida, el sistema debe contemplar ambas
opciones, ya sea por una devolución total o parcial de mercancía.
●
Notas de Crédito y Devolución de Dinero como Flujos Separados: Rose Ríos
Gómez enfatizó que la devolución de dinero no debería estar inmersa en el
módulo de notas de crédito, ya que son dos flujos alternos y diferentes. Rose
Ríos Gómez y Cornelio M Ramirez B. estuvieron de acuerdo en que el módulo
actual está destinado únicamente a la generación de notas de crédito,
independientemente del motivo que las origine.
●
Motivos para la Generación de Notas de Crédito: Rose Ríos Gómez identificó
tres entradas base para la generación de una nota de crédito: devolución de
producto (total o parcial), bonificación (o descuento), y error en la facturación.
El equipo acordó que estos tres escenarios deben ser el enfoque inicial para
el flujo de la nota de crédito, dejando el flujo de devolución de dinero como un
proceso alterno para su revisión posterior.
●
Alcance del Módulo de Notas de Crédito y su Aplicación a Prepago: Sara
SAnchez planteó una duda sobre el alcance de las notas de crédito,
preguntando si el módulo estaba limitado a facturas de prepago. Rose Ríos
Gómez recordó que el objetivo inicial del módulo era tener disponibles las
notas de crédito para asociarlas y aplicarlas a los prepagos que se van a
generar.
●
Manejo de la Cancelación de Facturas en el Flujo de Nota de Crédito:
Valdemar Farina Sanchez preguntó en qué escenarios una factura debe
cancelarse al generar una nota de crédito, ya que Rose Ríos Gómez sugirió
que la cancelación total de una factura y la generación de una nota de crédito
para esa factura son acciones excluyentes. Cornelio M Ramirez B. aclaró que
en Proquifa es posible generar una nota de crédito y cancelar la factura,
específicamente cuando la nota de crédito es por la totalidad de la factura y la
operación sucede dentro del mes corriente de la emisión.

●
Reglas de Negocio para la Cancelación de Facturas: Se definier  on las dos
reglas principales para que un usuario pueda optar por cancelar una factura al
generar una nota de crédito: debe ser por la totalidad de la factura y debe
ocurrir dentro del mes corriente en que se emitió la factura. Sara SAnchez
confirmó que el área de tesorería es quien toma la decisión de si se cancela o
no la factura bajo estas condiciones.
●
Información Requerida para la Cancelación de Facturas: Valdemar Farina
Sanchez preguntó si se requerían datos adicionales al cancelar la factura,
además del motivo de cancelación. Cornelio M Ramirez B. sugirió incluir una
nota adicional o un comentario para la nota de crédito como aclaración.
●
Revisión de la Necesidad de Autorización y Decimales: Mayra Franco sugirió
incorporar una firma de autorización para montos mayores a 50,000 pesos en
las notas de crédito, lo cual Larissa Calvo confirmó que el equipo está
revisando actualmente con las políticas existentes. También se discutió el
número de decimales a considerar, con Mayra Franco sugiriendo cuatro
decimales, y Cornelio M Ramirez B. señalando la necesidad de revisar las
reglas del SAT para el timbrado.
●
Detalle de Conceptos y Partidas en la Nota de Crédito: Cornelio M Ramirez B.
y Mayra Franco destacaron la necesidad de incluir más detalles en el
concepto de la nota de crédito, ya que solo decir "devolución" no proporciona
la materialidad necesaria. Rose Ríos Gómez solicitó incluir las partidas o
líneas específicas de la factura original que forman parte de la nota de
crédito, lo cual Valdemar Farina Sanchez confirmó que se agregará en el paso
de selección de factura.
●
Diseño del Concepto de la Nota de Crédito según el Motivo: Se definió que si
el motivo de la nota de crédito es "devolución de mercancía," se seleccionarán
las partidas y las piezas. Si el motivo es "descuento" o "bonificación," el
concepto quedaría como un campo abierto para que el usuario escriba la
descripción.
●
Envío y Foliado de la Nota de Crédito: Se confirmó que la nota de crédito debe
enviarse al cliente de primera instancia, con copia al área de crédito y al
analista de cuentas por cobrar. Cornelio M Ramirez B. sugirió incluir la orden
de compra del cliente o la factura relacionada en el asunto del correo para
referencia.
●
Foliado de la Nota de Crédito y Sistemas Legados: Se confirmó que las notas
de crédito manejan un folio consecutivo por empresa. Dado que Mayra Franco
confirmó que las notas de crédito se realizan por fuera actualmente, Sara
SAnchez sugirió que no habría conflict   o con los consecutivos de sistemas

legados, aunque podría ser necesario definir el último número consecutivo
para continuar la secuencia en Proquifanet 2.
●
Manejo de folios de factura para PQF2: Se discutió el uso de folios distintos
para las facturas generadas en el sistema PQF2, separados de los folios de
Legacy, para lo cual se puede añadir una serie diferente al folio. Cornelio M
Ramirez B. confirmó que es posible tener series distintas siempre que se siga
un consecutivo. El equipo acordó que se debe dar inicio al folio para los
prepagos, aunque para los créditos tendrían que traer el folio de Legacy
debido a que el flujo termina de ese lado.
●
Consideraciones sobre la transferencia de folios de crédito: Se identificó un
posible conflict   o en la transferencia de folios de crédito debido a los
caracteres admitidos en cada sistema, sugiriendo que se revisen las reglas de
unificación, especialmente si uno es numérico y el otro contiene letras. Rose
Ríos Gómez sugirió evaluar la posibilidad de utilizar un nuevo número de serie
distinto para las facturas por adelantado que se tramitarán en PQF2, con la
premisa de reducir el número máximo de conexiones entre sistemas. Sara
SAnchez mencionó la importancia de que Cornelio M Ramirez B. y Mayra
Franco revisen las pantallas de reportes y consultas para evaluar si se puede
evitar la transferencia de prepago al otro lado, ya que todo se cerraría del lado
de PQF2, ahorrando una transferencia.
●
Revisión del resumen final de la nota de crédito: Roberto Baez Muñoz
presentó el resumen final de la nota de crédito, que se muestra después de
enviar el correo, y mencionó que incluye el folio, el ID del SAT, el pack (Turbo
Pack), la certificación del SAT, el RFC y los datos específicos de la nota de
crédito. Se destacó que se mostrarían los saldos y los totales actualizados, y
que la información del XML se conservaría por un mínimo de 5 años para la
cancelación ante el SAT. Las acciones disponibles desde esta vista incluyen
descargar el XML, descargar el PDF, reenviar la nota por correo y cancelar la
nota de crédito, aunque la cancelación se puso como opcional dependiendo
del alcance.
●
Consulta de datos en el resumen de la nota de crédito: Cornelio M Ramirez B.
preguntó si el resumen de la nota de crédito incluye el folio y el folio fiscal
(UUID) de la factura original. Roberto Baez Muñoz confirmó que se incluirían
tanto el folio como el UUID. La información mostrada en el resumen permite
la consulta y la ejecución de acciones como la descarga o el envío por correo.
●
Reglas de cancelación para notas de crédito: Valdemar Farina Sanchez
consultó sobre las consideraciones o reglas para cancelar la nota de crédito,
y Cornelio M Ramirez B. explicó que una cancelación sigue las mismas reglas
del SAT que un CFDI: 24 horas para cancelar sin autorización, después de lo

cual se requiere la autorización del cliente. Se confirmó que la cancelación de
una nota de crédito que cancelaba una factura es posible. Cornelio M Ramirez
B. aclaró que al cancelar la nota de crédito, la cancelación de la factura que
se había realizado por medio de dicha nota queda inhabilitada y la factura
original se activa de nuevo, siempre y cuando esta no haya sido cancelada
ante el SAT.
●
Escenarios de cancelación de la nota de crédito: El equipo discutió que si la
factura original no está cancelada ante el SAT, la cancelación de la nota de
crédito inhabilita la cancelación que se había hecho por medio de esta, y se
podría expedir una nueva nota de crédito. Si la factura original ya ha sido
cancelada ante el SAT, y la nota de crédito que se había generado resulta
estar mal, ya no se podría generar una nueva nota de crédito; en ese caso, se
debe emitir una carta al cliente para que puedan ocupar su saldo. En ambos
escenarios (factura original cancelada o no), la acción dentro del sistema es
simplemente cancelar la nota de crédito, sin realizar ninguna acción adicional
sobre la factura original.
●
Programación de sesión para revisión de reportes: Robert mencionó que solo
falta revisar los reportes para concluir este módulo. El equipo acordó
programar una sesión específica para revisar los tres reportes pendientes,
estimando que se necesitarían aproximadamente dos horas. Cornelio M
Ramirez B. propuso la hora de 4 a 6 de la tarde para la sesión del día
siguiente, a lo cual el equipo de Rindem y Larissa Calvo estuvieron de
acuerdo.


Revisa las notas de Gemini para asegurarte de que sean precisas. Obtén sugerencias y
descubre cómo Gemini toma notas
Cómo es la calidad de estas notas específicas? Responde una breve encuesta para
darnos tu opinión; por ejemplo, cuán útiles te resultaron las notas.
