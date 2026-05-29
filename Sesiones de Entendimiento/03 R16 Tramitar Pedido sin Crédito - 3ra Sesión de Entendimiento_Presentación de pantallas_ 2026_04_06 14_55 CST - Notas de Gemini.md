


6 abr 2026
R16 Tramitar Pedido sin Crédito -
3ra Sesión de
Entendimiento/Presentación de
pantallas
Invitados
Rose Ríos GómezJuan David García Cruzblcalvo@proquifa.net

Roberto Baez MuñozIrma Andrade AguadoFrancisco Uriel Guerrero Rivera

Jose Antonio Chavez Amadorssanchez@proquifa.netValdemar Farina Sanchez

Alan Fernandez Garcia
Archivos adjuntos
R16 Tramitar Pedido sin Crédito - 3ra Sesión de Entendimient...
Registros de la reunión
Grabación

Resumen
El equipo revisó los flujos para la opción “Entrega con remisión”,
acordando la exclusión mutua de esta y la función “Factura por
adelantado”, con discusiones sobre la persistencia de códigos de
verificación y la edición de datos financieros en pedidos.

Reglas de Entrega y Remisión
La opción "Entrega con remisión" anula las restricciones de entrega,
permitiendo fechas en días bloqueados, pero debe ser marcada
manualmente en cada pedido. Si está marcada, el sistema genera una
nota de remisión en lugar de una factura y requiere la misma
autorización que “Factura por adelantado”.

Diseño de Factura y Remisión
Se decidió cambiar el diseño de ambas opciones a un radio button para
forzar una selección mutuamente excluyente o ninguna. Se confirmó
que para pedidos con productos controlados, ambas funciones se

bloquearían completamente, manteniendo el flujo actual.

Persistencia de Código de Verificación
El equipo solicitó que el código de verificación tenga funcionalidad
persistente por 24 horas, permitiendo al usuario salir y regresar a la
pantalla para su ingreso. Rose Ríos Gómez indicó que esto aumentará el
esfuerzo de desarrollo y presentará la justificación a la Dirección General
para su evaluación.

Detalles
●
Comentarios iniciales sobre el proyecto y buenas prácticas: Rose Ríos
Gómez solicitó que el equipo considere buenas prácticas del mercado para el
nuevo proyecto, ya que no es un proyecto de mejora típico sino uno casi
totalmente nuevo. El objetivo es que el sistema tenga los controles
necesarios, pero también la flexibilidad, evitando que un exceso de controles
haga difícil cambiar las reglas de negocio. Las nuevas pantallas propuestas
en el área de *rinden* están diseñadas para ser más ágiles y mejorar el
rendimiento.
●
Discusión sobre el tema "Entrega con remisión": Roberto Baez Muñoz
presentó el módulo "Tramitar pedido" para discutir la inquietud inicial sobre la
opción de "Entrega con remisión" en el panel de facturar pedido, una función
que ya existe en el sistema. Valdemar Farina Sanchez explicó que esta opción
está en la sección de restricciones del cliente y se relaciona con la fecha
estimada de entrega.
●
Función de Entrega con remisión y restricciones de entrega: Valdemar Farina
Sanchez detalló que la opción de "Entrega con remisión" en el catálogo de
clientes anula las restricciones de entrega (como no entregar los últimos tres
días del mes por temas de facturación), permitiendo que la fecha de entrega
sí caiga en esos días. Si la opción de remisión está marcada, el sistema de
entrega generará una nota de remisión en lugar de una factura.
●
Aclaración sobre la omisión de días con restricciones de entrega: Sara
SAnchez solicitó una confirmación sobre la lógica, y Valdemar Farina Sanchez
aclaró que el sistema *no* se salta los días de restricción si la opción de
"Entrega con remisión" está marcada. Si el cliente tiene restricciones y el
campo *no* está marcado, la fecha se salta hasta el mes siguiente; si está

marcado, se coloca la fecha en los días restringidos y se envía el dato para
generar la remisión.
●
Relación del dato de Entrega con remisión a nivel de catálogo y de pedido:
Valdemar Farina Sanchez planteó una duda sobre cómo se mapea el dato de
restricción general del catálogo de clientes al pedido específico, preguntando
si el sistema debería marcar automáticamente la opción en el pedido.
Biridiana aclaró que la marca de "Entrega con remisión" debe ser establecida
manualmente por el usuario para cada pedido, ya que no todos los pedidos
tienen la misma autorización.
●
Afección de la fecha estimada de entrega por la marca manual: Biridiana
confirmó que la selección manual del usuario en el campo de remisión afecta
la fecha estimada de entrega, ya que si el *check* de remisión está presente,
el sistema puede arrojar los últimos días del mes; de lo contrario, bloquea
esos días. Larissa Calvo y Valdemar Farina Sanchez confirmaron que no
marcar el campo no obliga al usuario a marcarlo para continuar, siendo
opcional.
●
Relación de Entrega con remisión y Factura por adelantado: Valdemar Farina
Sanchez y Biridiana acordaron que las opciones "Entrega con remisión" y
"Factura por adelantado" son excluyentes, por lo que se elige una, la otra, o
ninguna. Biridiana también confirmó que para pedidos con productos
controlados, ambos campos ("Factura por adelantado" y "Entrega con
remisión") se bloquearían por completo.
●
Autorización para Entrega con remisión: Valdemar Farina Sanchez preguntó
si se requeriría una autorización para "Entrega con remisión", similar a
"Factura por adelantado", a lo que Biridiana confirmó que sí se solicitaría, con
el mismo rol de autorización.
●
Propuesta de diseño para Factura por adelantado y Remisión en el módulo
Tramitar pedido: Roberto Baez Muñoz presentó el diseño de pantalla para
clientes con crédito y sin productos controlados, señalando que la función de
"Factura por adelantado" y "Entrega con remisión" se cambiaría de una casilla
de verificación a un *radio button* para garantizar que solo se seleccione una
opción o ninguna. Sara SAnchez confirmó que la opción de no marcar
ninguna de las dos también estaría disponible.
●
Flujo para clientes con productos controlados: Para clientes con crédito y
productos controlados, se decidió que la opción de facturar por adelantado
estaría deshabilitada. Se mostraría un ícono informativo que explicaría al
usuario por qué la opción no puede seleccionarse, manteniendo el flujo de
tramitación de pedido igual al actual.

●
Flujo para clientes con Factura por adelantado y solicitud de código de
verificación: Para clientes con crédito sin productos controlados que
seleccionan "Factura por adelantado", se selecciona el *radio button* y, al
tramitar el pedido, se solicitaría un código de verificación. El pedido se
mostraría con un distintivo de que es factura por adelantado.
●
Duración y persistencia del código de verificación (R1): Biridiana preguntó
sobre el tiempo de validez del código de verificación, y Rose Ríos Gómez
explicó que, según el diseño actual desde R1, los códigos tienen una duración
de 24 horas. Sin embargo, si el usuario sale de la pantalla, la solicitud de
código se pierde, obligando a solicitar un nuevo código.
●
Solicitud de persistencia del código de verificación y aumento del esfuerzo
de desarrollo: Sara SAnchez y Larissa Calvo solicitaron que el código de
verificación tenga funcionalidad vigente en la pantalla, de modo que el
usuario pueda salir y regresar a la pantalla para ingresar el código dentro de
las 24 horas. Rose Ríos Gómez indicó que, si se implementa, el esfuerzo de
desarrollo aumentaría considerablemente, ya que implicaría un rediseño de la
solución. Rose Ríos Gómez dijo que se harán las dos estimaciones y se
presentarán a la Dirección General para su justificación, evaluando el impacto
en el negocio.
●
Evaluación de métodos alternativos de autorización: Valdemar Farina
Sanchez preguntó si existía alguna restricción para proponer un método de
autorización diferente al envío de códigos de cuatro dígitos por correo
electrónico. Larissa Calvo y Valdemar Farina Sanchez confirmaron que están
abiertos a caminos más sencillos, siempre que cumplan con la funcionalidad
esperada.
●
Restricciones para los nuevos mecanismos de autorización: Sara SAnchez
planteó una posible restricción si el mecanismo cambiara a un mensaje de
texto, ya que no todos los usuarios tienen un teléfono móvil de la empresa, lo
que podría generar rechazo. Valdemar Farina Sanchez agradeció esta
información, limitando las opciones a la plataforma, el correo o el equipo de
cómputo.
●
Flujo de pedidos prepago con productos controlados (Proforma): Roberto
Baez Muñoz explicó que para clientes prepago con productos controlados, no
habría opción de "Entrega con remisión" ni "Factura por adelantado". El botón
de tramitación cambiaría a "Generar Proforma", mostrando una
previsualización de la misma, y luego procediendo al envío del correo.
●
Definición del asunto del correo electrónico de Proforma: Roberto Baez
Muñoz solicitó la definición del asunto del correo electrónico para la

proforma. Biridiana y Sara SAnchez acordaron que el asunto debe ser
"Proforma" más el folio del pedido que se le dio, que es el mismo folio del
pedido interno. Valdemar Farina Sanchez solicitó agregar un pendiente para
que se proporcionen las reglas de foliado para las proformas, ya que parece
ser un contador independiente.
●
Ubicación de la generación de proformas y su foliador: Valdemar Farina
Sanchez preguntó si las proformas se generan en sistemas legados, lo que
podría afectar el foliador. Biridiana y Sara SAnchez confirmaron que, dado que
se planea dejar de usar el sistema *Legacy* para prepagos, el foliador de
proformas debe ser completamente nuevo y gestionado por este sistema.
●
Propuesta de plantillas de correo y notas específicas: Rose Ríos Gómez
propuso migrar los correos a plantillas en Brevo para facilitar el
mantenimiento y definir el diseño. En la pantalla de envío de correo, solo se
ingresarían las notas específicas que se agregarían a la plantilla ya definida.
●
Retención de pedidos prepago después de la generación de Proforma:
Roberto Baez Muñoz confirmó que una vez que se genera y envía la proforma,
el pendiente permanece en el módulo de "Tramitar pedido", pero bloqueado y
sin posibilidad de modificación hasta que se valide el cobro. Esto también
generaría un pendiente en "Validar Cobro".
●
Restricción de Entrega con remisión para clientes prepago: Biridiana aclaró
que "Entrega con remisión" no aplica a clientes prepago en ningún caso.
Explicó que la remisión es para clientes con crédito que no pueden aceptar
una factura por su cierre de mes, mientras que los prepagos siempre
requieren una factura para continuar con el proceso de compra.
●
Comportamiento de Pago contra entrega: Valdemar Farina Sanchez confirmó
que el flujo "Pago contra entrega" se comporta como crédito, por lo que sí
tendría la opción de marcar "Entrega con remisión".
●
Flujo para clientes prepago con Factura por adelantado: El último caso
abordado para pedidos prepago es con productos no controlados, donde solo
está la opción de "Factura por adelantado". Se solicita el código de
verificación, y una vez validado, el pedido permanece en estado bloqueado en
"Tramitar pedido" y se genera un pendiente en "Factura por adelantado".
●
Dudas sobre órdenes de compra internas y sin orden de compra: Roberto
Baez Muñoz preguntó cómo funcionarían los flujos de "Factura por
adelantado" y "Entrega con remisión" para pedidos con órdenes de compra
internas y pedidos sin orden de compra. Biridiana, Rose Ríos Gómez y

Roberto Baez Muñoz acordaron llevarse esta pregunta como pendiente para
evaluar los impactos en ambos flujos.
●
Programación de la discusión sobre Factura por adelantado: Larissa Calvo
propuso posponer la presentación detallada del módulo "Factura por
adelantado" (clientes crédito sin productos controlados) hasta que esté
presente un representante de Cuentas por Cobrar, ya que es un tema
financiero. Rose Ríos Gómez estuvo de acuerdo y solicitó gestionar la
próxima reunión lo antes posible para avanzar con los temas.
●
Funcionalidad existente de Editar Datos en Tramitar Pedido: Valdemar Farina
Sanchez planteó una duda sobre la opción "Editar datos" que ya existe en el
sistema para pedidos crédito, la cual permite al ESAC, con autorización de
Finanzas, modificar el uso de CFDI y el método de pago. Valdemar Farina
Sanchez preguntó si esta opción debe permanecer o eliminarse para el flujo
de Factura por adelantado, considerando que el área financiera es la
encargada de la facturación.
●
Opinión del área financiera sobre la edición de datos: Biridiana opinó que, por
conocimiento legal y normativas del SAT, solo el área financiera debería estar
facultada para editar datos como el método de pago o uso de CFDI, ya que la
responsabilidad del ESAC podría generar errores. Biridiana sugirió que la
edición de datos puede llevar a más errores, complejidad y mantenimiento,
por lo que se debe evaluar si la funcionalidad tiene valor para mantenerla o es
mejor eliminarla.
●
Reglas de negocio y campos automáticos para Factura por adelantado:
Biridiana sugirió la idea de que al activar la opción de "Factura por
adelantado", el sistema cambie automáticamente el método de pago de PUE
(pago único) a PPD (pago en parcialidades). Rose Ríos Gómez enfatizó que
este es el momento para aplicar mejores prácticas y simplificar el proceso,
posiblemente dejando la responsabilidad de la modificación en un área única
(Finanzas) para aligerar la codificación.
●
Relevancia de la edición de Razón Social y RFC: Sara SAnchez señaló que los
datos de Razón Social y RFC son importantes de poder editar, ya que los
problemas de timbrado a menudo surgen por estos campos, y las ESAC son
quienes reciben la Constancia de Situación Fiscal del cliente. Sin embargo,
Biridiana aclaró que la edición de Razón Social y RFC es a nivel de la plantilla
del cliente, no a nivel de pedido, donde la edición se limita al uso de CFDI o
método de pago.
●
Conclusión sobre la edición de datos: Valdemar Farina Sanchez propuso que
revisen de su lado si se mantienen los campos de edición, si se eliminan o si

la edición de ciertos datos debe realizarse a nivel del catálogo de clientes
para concluir en la siguiente sesión. Roberto Baez Muñoz recordó que
también se debe considerar el alcance para Perú en estos flujos, aunque no
se esté abordando en la sesión .

Pasos siguientes recomendados

[Irma Andrade Aguado] Agregar Pendiente: Agregar pendiente a la lista de
tareas para que el equipo proporcione las reglas de foliador de proformas.

[Sara SAnchez] Confirmar Foliador: Confirmar a nivel de código que el
contador de folios es lineal y no se maneja por empresa.

[Biridiana] Revisar Plantilla: Revisar la plantilla de correo existente que se
envía al cliente y proporcionarla al equipo de desarrollo.

[Rose Ríos Gómez] Estimar Desarrollo: Realizar estimación de desarrollo para
implementar la funcionalidad de vigencia del código de verificación de 24
horas. Esta estimación debe contemplar hacerlo como funciona normalmente
y la versión solicitada que permite regresar a la pantalla.

[Biridiana, Sara SAnchez, Larissa Calvo] Analizar Impacto: Revisar los
impactos que las órdenes de compra (internas o sin orden) pudieran tener en
los flujos de Factura por Adelantado y Entrega con Remisión.

[Larissa Calvo] Gestionar Sesión: Gestionar la programación de la próxima
sesión, asegurando la asistencia del área de Cuentas por Cobrar para revisar
el módulo de Factura por Adelantado.

[Biridiana, Sara SAnchez] Definir Edición: Revisar cuáles campos de datos del
cliente (Razón Social, RFC) pueden editarse en el trámite de pedido y cuáles
deben gestionarse a nivel catálogo para definir las reglas en la próxima
sesión.

Revisa las notas de Gemini para asegurarte de que sean correctas. Obtén consejos y
descubre cómo toma notas Gemini
Danos tu opinión sobre el uso de Gemini para tomar notas en una breve encuesta.
