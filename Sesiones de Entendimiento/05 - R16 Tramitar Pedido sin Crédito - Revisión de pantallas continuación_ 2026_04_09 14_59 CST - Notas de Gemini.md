


9 abr 2026
R16 Tramitar Pedido sin Crédito -
Revisión de pantallas continuación
Invitados
Rose Ríos GómezJuan David García Cruzblcalvo@proquifa.net

mfranco@proquifa.netRoberto Baez MuñozIrma Andrade Aguado

Jose Antonio Chavez Amadorcmramirez@proquifa.netssanchez@proquifa.net

Valdemar Farina SanchezAlan Fernandez Garcia
Archivos adjuntos
R16 Tramitar Pedido sin Crédito - Revisión de pantallas conti...
Registros de la reunión
Grabación

Resumen
La discusión de la facturación anticipada y el flujo de validación de pago
para clientes prepago dominó la reunión, destacando la necesidad de
políticas claras para condonaciones y un estado de cuenta auxiliar.

Revisión Factura Anticipada Modal
El modal Generar Factura por Adelantado se modificó para que el total
de la factura sea más grande y se incluyan los datos fiscales del cliente.
Se evaluará generar un folio independiente para las facturas de prepago
en Proquifanet 2, separándolo de sistemas legados.

Discusión Módulo Validación Pago
Se presentó el módulo de validar pago para clientes prepago, el cual
sustituirá la validación de cobro en SAP, y requiere que Tesorería
proporcione listados de correos para entrenar la clasificación
automática. Se estableció la necesidad de definir un folio consecutivo e
independiente para cada transacción de cobro, además del folio interno.

Manejo Saldo a Favor/En Contra
No existe una política formal que permita la condonación de faltantes
mínimos en los pagos. Se acordó la necesidad de crear un estado de

cuenta auxiliar para que los clientes prepago puedan ver los saldos a
favor o en contra.

Detalles
●
Introducción y asistencia: Se confirmó la asistencia de Irma Andrade Aguado,
Mayra Franco, Rose Ríos Gómez y Larissa Calvo, mientras que Corne se uniría
de manera intermitente. El inicio de la sesión se retrasó brevemente debido a
que Brenda y Eric, del equipo de cuentas por cobrar, se retrasaron a causa de
la lluvia.
●
Discusión sobre la presentación de la factura por adelantado: Mayra Franco
sugirió mejorar la presentación del modal de "generar factura por adelantado"
para que se asemeje a una factura final, incluyendo el importe y los datos
fiscales. Rose Ríos Gómez indicó que en el flujo se podrá ver una
previsualización de la factura.
●
Revisión del modal "Generar factura por adelantado": Roberto Baez Muñoz
confirmó que se hicieron cambios en el modal, mostrando el total de la
factura más grande y visible. También se agregaron los datos de contacto y
los datos fiscales del cliente en modo de solo lectura, con excepción del uso
de CFDI que es editable.
●
Flujo de generación y timbrado de la factura: El flujo incluye la opción de
generar y previsualizar la factura antes del timbrado. Si el timbrado falla, se
mostraría un error (aunque el catálogo de errores aún está pendiente). Si la
generación es exitosa, se procede al modal de envío de la factura por correo.
●
Formato del asunto para el envío de la factura: Se consultó el formato
deseado para el asunto del correo de envío de la factura. Biridiana Arias y
Larissa Calvo confirmaron que el asunto debe ser el folio del pedido interno
más el folio de la factura.
●
Identificación y foliador de facturas internas: Valdemar Farina Sanchez
preguntó si las facturas generarán folios internos además del sello del SAT.
Sara Sanchez confirmó que sí se genera un folio interno, el cual es un
autoincrementable por empresa.
●
Reglas y estructura del folio interno de la factura: Valdemar Farina Sanchez
solicitó las reglas y el formato del folio interno. Sara Sanchez aclaró que es un
incremental sucesivo sin reinicio, aunque el campo en la base de datos podría
tener un límite máximo.

●
Consideración de folios independientes para facturas de prepago: Rose Ríos
Gómez propuso la evaluación de generar un folio específico e independiente
para las facturas de prepago en Proquifanet 2 para evitar conexiones con
sistemas legados. Sara Sanchez y el equipo de desarrollo aceptaron evaluar
esta propuesta, aunque señalaron que la factura por adelantado para clientes
de crédito también debería considerarse.
●
Manejo del pendiente de factura por adelantado en clientes crédito: Roberto
Baez Muñoz explicó que al completar la factura por adelantado, el pendiente
se eliminaría. Mayra Franco preguntó sobre los clientes de crédito, ya que
necesitan cargar la factura para revisión. Sara Sanchez clarificó que el
alcance para este lanzamiento solo cubre hasta la factura por adelantado en
Proquifanet 2, y el flujo posterior (como cargar la factura a revisión) se
continuará en los sistemas legados.
●
Introducción al módulo de validación de pago: Roberto Baez Muñoz presentó
el módulo de "validar pago," el cual aplica únicamente para clientes prepago.
La única modificación en el buzón de cobros existente será la clasificación,
que será automática, aunque se mantendrá la posibilidad de reclasificar.
●
Detalle de la pantalla de validación de pago (Prepago): La pantalla inicial
muestra un listado clasificado por clientes, incluyendo el RFC, los cobros
recibidos y las facturas/proformas pendientes por cobrar. Roberto Baez
Muñoz confirmó que este módulo sustituiría la validación de cobro en SAP
para clientes prepago.
●
Necesidad de una visualización coordinadora para tesorería: Larissa Calvo
solicitó una visualización similar a la que tienen los coordinadores de
pedidos, donde el coordinador de tesorería pueda ver todos los pagos
recibidos y el estado de procesamiento de su cartera de analistas. Valdemar
Farina Sanchez confirmó que se tiene considerada una pantalla para este fin,
más orientada a reporte.
●
Mecanismo de clasificación automática de cobros: Mayra Franco preguntó
cómo el sistema clasifica un correo como un cobro. Valdemar Farina Sanchez
explicó que un mecanismo lee los correos e identifica el tipo de correo
mediante palabras clave.
●
Solicitud de listados de correos representativos para entrenamiento: Sara
Sanchez y Valdemar Farina Sanchez acordaron la necesidad de que el equipo
de tesorería proporcione listados de correos representativos de cobros para
el entrenamiento del sistema de clasificación automática.

●
Detalle de la captura de cobro (Paso 1): Roberto Baez Muñoz presentó la
pantalla de captura del cobro, donde del lado izquierdo se ven los cobros
recibidos del buzón. Se planteó la necesidad de definir un folio interno de
cobro (propuesta: COB más secuencial) y la información necesaria para
identificar el cobro.
●
Recomendación de folio consecutivo para cobros: Rose Ríos Gómez enfatizó
que, como buena práctica, se debe considerar un folio consecutivo e
independiente para cada transacción de cobro, independientemente de su
asociación con un pedido o factura. Este folio es clave para una identificación
única.
●
Validación de documentos e inconsistencias en el cobro: Se discutió el
proceso de captura de datos donde se valida el comprobante de pago
recibido. Si existe una inconsistencia (ej. montos que no coinciden entre el
cuerpo del correo y el comprobante adjunto), se marca la inconsistencia y se
detiene el flujo en este paso.
●
Cambio de dinámica en la asignación de pagos a facturas (Prepago): Sara
Sanchez aclaró que en el nuevo flujo de prepago, el área de Tesorería será
responsable de llevar todo el proceso, incluyendo la revisión, validación y
relación del pago con la proforma o factura. Anteriormente, esto lo realizaban
los ESAC.
●
Flujo de pedidos de prepago y pendiente de tesorería: Larissa Calvo explicó
que cuando un ESAC tramita un pedido de un cliente prepago, se genera la
proforma y, en ese momento, aparece el pendiente de pago para el equipo de
Tesorería.
●
Discusión sobre la política de montos faltantes/remantes: Mayra Franco
preguntó si existe una política que permita validar un pago si el monto
faltante es mínimo, como 100 pesos o menos. Larissa Calvo y Rose Ríos
Gómez confirmaron que no existe tal política y que, si hay faltantes, el pago
se rechaza.
●
Asociación de facturas/proformas y escenarios de saldo (Paso 2): Roberto
Baez Muñoz presentó el segundo paso (asociación de facturas y proformas)
donde se pueden asociar facturas o proformas al cobro capturado en el paso
uno. Este paso mostrará la diferencia entre el monto capturado y el total a
cobrar, abordando escenarios de saldo a favor o de menos.
●
Manejo del saldo a favor/remanente: Se discutió el escenario en el que el
cliente deposita más del importe de la proforma, resultando en un "saldo
disponible". Mayra Franco mencionó que el cliente puede optar por un saldo a

favor para futuras compras. Biridiana Arias sugirió que el sistema emita una
carta al cliente para notificarles sobre este saldo a favor.
●
Necesidad de un estado de cuenta auxiliar y alcance del proyecto: Mayra
Franco solicitó que el sistema Proquifanet 2 genere un estado de cuenta
auxiliar donde el cliente pueda ver el saldo a favor/en contra. Rose Ríos
Gómez señaló que el tema del estado de cuenta se había considerado más
adelante, aunque hizo hincapié en la necesidad de considerar el porcentaje de
clientes prepago y evaluar qué escenarios conviene controlar dentro del
sistema.
●
Cuestionamiento sobre la legalidad de retener dinero excedente: Rose Ríos
Gómez preguntó si es legal y contablemente correcto para Proquifa recibir y
retener un excedente de dinero de un cliente contra un documento. Mayra
Franco proporcionó un ejemplo de un cliente que depositó 800,000 pesos de
más.
●
Retención de clientes y manejo de excedentes de pago: Mayra Franco
describió el proceso actual para manejar los pagos excedentes de los
clientes, donde la estrategia principal es retener al cliente y el dinero. La
primera opción es aplicar una factura de anticipo para futuras órdenes de
compra, y la otra es realizar una devolución si el cliente lo solicita. Rose Ríos
Gómez enfatizó la necesidad de definir escenarios concretos, incluyendo qué
sucede cuando un cliente paga de menos.
●
Homogeneización de clientes y la necesidad de un estado de cuenta: Mayra
Franco sugirió la necesidad de homogeneizar a todos los clientes para aplicar
el mismo método de prepago de contado, independientemente de su estatus
crediticio, mencionando el caso de Biores. La creación de un estado de
cuenta es considerada esencial para el seguimiento de los clientes, ya que
actualmente este proceso se realiza de forma manual.
●
Uso de notas de crédito vs. facturas de anticipo para saldos a favor: Sara
SAnchez propuso que, fiscalmente, lo correcto para el dinero sobrante podría
ser generar notas de crédito, que representan dinero a favor del cliente para
usar en facturas o proformas posteriores. Biridiana Arias aclaró que una nota
de crédito siempre ampara una factura, y la factura de anticipo es el
documento correcto cuando se tiene un saldo que se puede usar, pero no se
sabe a qué se alineará, y esta factura se cancelaría posteriormente con una
nota de crédito.
●
Manejo del saldo en contra (pago insuficiente): Se discutió el escenario en el
que un cliente deposita de menos, lo cual se entiende como un error.
Valdemar Farina Sanchez describió que, si el cliente deposita una cantidad

que no cubre la proforma, se les contacta para que depositen el faltante.
Mayra Franco mencionó la necesidad de establecer políticas claras para
determinar el monto a partir del cual se puede condonar el faltante o generar
una nota de crédito.
●
Proceso de asociación de múltiples pagos a una sola proforma: Si al cliente
le falta una cantidad significativa (ejemplo: más de $5000 pesos), se les debe
comunicar para que realicen un segundo depósito. Valdemar Farina Sanchez
y Mayra Franco confirmaron que en este escenario, los dos cobros (el pago
inicial y el faltante) se deben asociar a la misma proforma para saldar la
cantidad total. La lógica del sistema debería reflejar que el importe a pagar
sea cero después de aplicar ambos cobros.
●
Opciones en caso de pago insuficiente total y cancelación de pedido: Si un
cliente no cubre la totalidad del pago después de ser notificado, hay dos
caminos: el pedido se mantiene en espera hasta que liquiden el faltante, o el
cliente puede optar por cancelar el pedido y solicitar la devolución del dinero
depositado. Biridiana Arias y Larissa Calvo indicaron que, si hay cancelación,
el proceso necesitaría una salida específica en el flujo de validación de cobro
para la cancelación con devolución. Rose Ríos Gómez y Mayra Franco
sugirieron la adición de un botón de "cancelar pedido" para manejar estas
inconsistencias de cobro.
●
Autorización para condonación de montos menores y la necesidad de
políticas: Mayra Franco planteó que, para la condonación de montos menores
(como un centavo o hasta $100 pesos) si se llegara a un acuerdo, esta acción
requeriría la autorización del especialista de cuentas por cobrar y el
coordinador. Larissa Calvo indicó que no existe actualmente una política
formal para la condonación de montos pequeños, por lo que el equipo
revisará las políticas con base en los escenarios propuestos.
●
Consideración de notas de crédito en la validación de cobro: Se debatió si las
notas de crédito deberían considerarse en la pantalla de validación de cobro.
Biridiana Arias afirmó que sí se puede validar un cobro utilizando una nota de
crédito pendiente, siempre y cuando esta nota provenga de un historial
pasado donde hubo una factura que fue cancelada. Si el cobro es reciente, no
hay nota de crédito pendiente.
●
Gestión de la creación de notas de crédito y la limitación de Legacy: Sara
SAnchez informó que las notas de crédito se generan fuera del sistema actual
(Legacy), lo que dificulta su conexión y uso en el nuevo sistema. Rose Ríos
Gómez sugirió priorizar la creación de una pantalla que permita generar las
notas de crédito directamente en el nuevo sistema para evitar interfaces

complejas con sistemas Legacy y facilitar el proceso para los pedidos de
prepago. Se solicitó a Sara SAnchez compartir un ejemplo de una nota de
crédito (PDF y XML) para desarrollar esta propuesta.
●
Flujo de facturación y envío para proformas y facturas: Roberto Baez Muñoz
presentó el paso tres: facturación y envío, donde se configuran las proformas
y facturas para su emisión. Se confirmó que tanto PPD (pago en
parcialidades) como PUE (pago en una sola exhibición) están disponibles
para seleccionar como método de pago en las proformas. En el caso de PPD,
se requerirá la generación de un complemento de pago.
●
Estatus del envío y opciones de consulta de facturas: Se describieron los
estatus de las proformas/facturas (pendiente, factura/complemento
generado, enviado), destacando que una vez enviado, no se puede reenviar el
correo. Biridiana Arias preguntó si habría una opción para descargar y
reenviar facturas en caso de que el cliente no las reciba, a lo que Roberto
Baez Muñoz confirmó que se está proponiendo un módulo de facturas para la
visualización y descarga.
●
Integración de facturas con sistemas Legacy y planificación de reportes:
Sara SAnchez indicó que las facturas timbradas seguirían apareciendo en el
aplicativo "Contador" de Legacy, aunque podría haber un retraso en la
transferencia de datos. Valdemar Farina Sanchez reiteró la solicitud al equipo
de finanzas de compartir pantallazos de sus reportes actuales (como el de
"Contador") para asegurar que el nuevo módulo de reportes propuesto
(pedidos, facturas y cobros) satisfaga sus necesidades de datos.
●
Tramitación del pedido después de la validación del cobro: Una vez validado
el cobro, el pedido prepago se desbloquea en el módulo de tramitación de
pedidos. Roberto Baez Muñoz confirmó que este proceso implica el envío de
la factura de manera independiente, y luego el envío de la confirmación del
pedido una vez que se tramita, lo cual es consistente con la práctica actual.
●
Próximos pasos y sesión de seguimiento: Se acordó finalizar la sesión de
flujo y agendar una sesión de seguimiento, preferiblemente para el lunes, para
revisar los reportes y los pendientes restantes. Se solicitó al equipo de
desarrollo (Roberto Baez Muñoz) que compartiera el flujo de pantallas
presentado por correo electrónico para una revisión más detallada.


Pasos siguientes recomendados

[Sara SAnchez] Compartir Reglas Folio: Compartir reglas técnicas, detalles de
campo de base de datos. Enviar tabla y campos utilizados para folios por
empresa para correcta alineación.

[Sara SAnchez] Analizar Folio Prepago: Evaluar implementación de folio de
factura independiente para pagos anticipados. Minimizar dependencia con
sistemas legados. Discutir propuesta con Biridiana Arias y Mayra Franco.

[El grupo] Definir Inyección Legacy: Especificar punto de inserción en sistema
legado. Indicar dónde se debe inyectar factura por adelantado para continuar
flujo de proceso.

[Sara SAnchez, Biridiana Arias] Suministrar Correos Cobro: Proporcionar
listados de correos electrónicos representativos de cobros. Usar estas
muestras para entrenamiento del sistema de clasificación automática.

[El grupo] Evaluar Estado Cuenta: Evaluar incorporación de requisito para
generar estado de cuenta auxiliar. Analizar funcionalidad de emisión
automática de cartas de notificación por saldos a favor.

[El grupo] Crear Estado Cuenta: Homogeneizar todos los clientes por igual.
Crear un estado de cuenta que permita sacar el auxiliar.

[Mayra Franco] Definir Subpagos: Definir escenarios concretos para cuando
un cliente pague de menos. Utilizar esta información para construir
escenarios.

[El grupo] Diagrama Procesos: Generar mini diagrama de procesos fiscales
complejos, como notas de crédito y facturas anticipo. Asegurar que el
sistema pueda responder a estos flujos.

[Roberto Baez Muñoz] Proveer Escenarios: Proveer los escenarios de
validación actualmente discutidos al equipo.

[El grupo] Mapear Escenarios: Mapear internamente todos los escenarios
posibles de pago para cubrirlos.

[El grupo] Compartir Políticas: Compartir las políticas que definen criterios
para condonar pequeños faltantes o excedentes en los pagos.

[Larissa Calvo] Verificar Flujo: Verificar documentación del flujo de
cancelación para confirmar si regresa a tramitación de pedido o notifica
directamente. Informar a Valdemar los hallazgos.

[El grupo] Definir Catálogo: Acotar y amarrar el catálogo de motivos de
cancelación. Definir comportamiento esperado del sistema para la fábrica.


[El grupo] Revisar Políticas: Revisar conjuntamente las políticas de pago
respecto a condonación de montos. Proveer retroalimentación final al equipo
de fábrica.

[Sara SAnchez] Compartir Nota: Compartir una nota de crédito de muestra en
formato PDF y XML para referencia de diseño.

[El grupo] Ratificar Pago: Ratificar si los métodos de pago PPD y PUE deben
estar disponibles para Proformas.

[Sara SAnchez] Recopilar Reportes: Recopilar y compartir capturas de
pantalla de los reportes Legados (Pedidos y Contador). Proveer datos
valiosos necesarios.

[Roberto Baez Muñoz] Compartir Flujo: Compartir el flujo de pantallas
presentado durante la sesión por correo electrónico.

[Larissa Calvo] Agendar Sesión: Coordinar y agendar la próxima sesión de
seguimiento, apuntando a realizarla el día lunes.

Revisa las notas de Gemini para asegurarte de que sean correctas. Obtén consejos y
descubre cómo toma notas Gemini
Danos tu opinión sobre el uso de Gemini para tomar notas en una breve encuesta.
