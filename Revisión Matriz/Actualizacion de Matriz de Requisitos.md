Fecha de Actualización: 10-Junio-2026

TPSC-RE-FU-001		Mantenimiento de catálogos del sistema	Bases de Datos	Yo como sistema PQF2, quiero contar con un Catálogo de Cuentas Bancarias del grupo PROQUIFA, que mantenga las cuentas vigentes de cada empresa emisora del grupo, para que los módulos que requieran información de cobro o pago dispongan de un origen único y consistente de las cuentas bancarias del grupo.	El sistema debe mantener un Catálogo de Cuentas Bancarias del grupo PROQUIFA con las cuentas vigentes de cada empresa emisora del grupo. El catálogo debe estar disponible como fuente única de consulta para los módulos del sistema que requieran información de las cuentas bancarias del grupo. El estado de cada cuenta debe poder distinguir entre cuentas que existen vigentes en sistema y cuentas que ya no existen vigentes (sin exponer el mecanismo técnico de borrado al consumidor). La baja de una cuenta es siempre lógica: el registro se conserva marcado como no vigente y nunca se elimina físicamente de la base de datos.			Propuesto					R16.3M-RE-FU-001	"## Aplica a

- Catálogo único de cuentas bancarias del grupo PROQUIFA, mantenido por el área de Soporte a la Producción.
- Empresas del grupo PROQUIFA México: Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica. Cada cuenta pertenece a una sola empresa emisora.
- Consulta del catálogo desde cualquier módulo del sistema que requiera información de cuentas bancarias del grupo (alta de cliente, emisión de proforma, pantallas de Tesorería, Validar Cobro, integraciones que consuman el catálogo).
- Estado de existencia vigente de cada cuenta: una cuenta existe vigente en sistema o no existe vigente. La baja es lógica: el registro de la cuenta se conserva marcado como no vigente y nunca se elimina físicamente de la base de datos.

## No aplica a

- Interfaz gráfica de usuario (UI) para alta, baja, modificación o consulta de cuentas bancarias: no se desarrolla en R16.
- Gestión de cuentas bancarias de clientes (clientes externos): aplica solo a las cuentas del grupo PROQUIFA emisor.
- Operaciones Perú: el catálogo de cuentas bancarias de Golocaer S.A.C. SÍ forma parte del alcance de R16 (es necesario para validar cobros en Perú), pero su modelo —estructura de la cuenta conforme a normativa SUNAT— aún no está definido y se mantiene como brecha por resolver (ver Riesgo 1). Por ello, hasta que se defina el modelo, las cuentas Perú no pueden poblarse."	"## Reglas de Negocio

Regla 1 — Catálogo único de cuentas bancarias del grupo
El sistema mantiene un Catálogo de Cuentas Bancarias del grupo PROQUIFA como fuente única de información sobre las cuentas bancarias de las empresas emisoras del grupo.

Regla 2 — Estado de existencia vigente
Cada cuenta bancaria tiene un estado que indica si existe vigente en sistema o no. Las cuentas que no existen vigentes no se ofrecen al usuario en ningún módulo consumidor, pero su información se conserva para trazabilidad histórica de operaciones previas.

Regla 3 — Gestión sin interfaz gráfica en R16
La gestión del catálogo (alta, baja, modificación) no dispone de interfaz gráfica de usuario en R16. La gestión es responsabilidad del área de Soporte a la Producción mediante acceso directo a la base de datos del sistema.

Regla 4 — Consumo desde módulos del sistema
Cualquier módulo del sistema que requiera información de cuentas bancarias del grupo PROQUIFA consulta el catálogo. No existen catálogos paralelos ni copias locales en otros módulos.

## Riesgos

Riesgo 1 — Modelo de cuentas bancarias Perú no definido
El catálogo de cuentas de Golocaer S.A.C. (Perú) forma parte del alcance de R16, pero su modelo —estructura de la cuenta conforme a normativa SUNAT, potencialmente distinta de la mexicana— aún no está definido. ** Mientras el modelo Perú no se defina, las cuentas de Golocaer S.A.C. no pueden poblarse en el catálogo, lo que bloquea la validación de cobros en Perú. Brecha prioritaria por resolver con el cliente. **

Riesgo 2 — Visibilidad restringida del catálogo no materializada en R16
Sin interfaz de usuario, los consumos al catálogo dependen completamente de la integridad de los datos cargados en BD. Errores manuales en BD impactan directamente todos los módulos consumidores sin posibilidad de validación por usuario final. 

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — DISPONIBILIDAD DEL CATÁLOGO
═══════════════════════════════════════════════════════════════

Criterio A1 — Consulta del catálogo desde módulos
Dado que un módulo del sistema requiere información de cuentas bancarias del grupo PROQUIFA,
Cuando consulta el Catálogo de Cuentas Bancarias,
Entonces deberá obtener las cuentas existentes vigentes que apliquen al contexto.

Criterio A2 — Filtrado por empresa emisora
Dado que un módulo solicita las cuentas bancarias asociadas a una empresa emisora específica del grupo (Golocaer, Mungen, Proquifa o Proveedora Quimico Farmaceutica),
Cuando consulta el catálogo,
Entonces deberá obtener las cuentas vigentes cuya empresa beneficiaria coincida con la solicitada.

═══════════════════════════════════════════════════════════════
SECCIÓN B — ESTADO DE EXISTENCIA DE LAS CUENTAS
═══════════════════════════════════════════════════════════════

Criterio B1 — Solo cuentas existentes vigentes ofrecidas al usuario
Dado que un módulo lista cuentas bancarias del grupo en una pantalla operativa,
Cuando renderiza el listado,
Entonces deberá incluir únicamente las cuentas que existen vigentes en sistema. Las cuentas que no existen vigentes no aparecen en ningún listado al usuario final.

Criterio B2 — Conservación histórica
Dado que una cuenta bancaria deja de existir vigente en el sistema,
Cuando una operación histórica previa referencia esa cuenta,
Entonces la información de la cuenta debe permanecer accesible para fines de trazabilidad histórica.

═══════════════════════════════════════════════════════════════
SECCIÓN C — GESTIÓN DEL CATÁLOGO
═══════════════════════════════════════════════════════════════

Criterio C1 — Sin interfaz de usuario en R16
Dado que un usuario operativo requiere agregar, modificar o dar de baja una cuenta bancaria del grupo,
Cuando intenta hacerlo desde la aplicación PQF2,
Entonces NO deberá encontrar pantalla disponible para esta operación.

Criterio C2 — Gestión por Soporte a la Producción
Dado que se requiere alta, modificación o baja de una cuenta bancaria del grupo,
Cuando se solicita la operación,
Entonces la solicitud deberá canalizarse al área de Soporte a la Producción (equipo SAP), que ejecuta la operación directamente en la base de datos del sistema. En el caso de una baja, ésta se realiza a nivel lógico: el registro de la cuenta no se elimina de la base de datos, sino que se marca como no vigente, conservando el registro para trazabilidad histórica."								"- Referencia técnica para implementación: el catálogo se materializa en la tabla CuentaBanco de la BD PConnect (referencia del sistema actual). Campo activo=1 identifica las cuentas vigentes; activo=0 las que ya no están vigentes pero se conservan para trazabilidad histórica de operaciones previas.
- Área responsable del mantenimiento del catálogo: Soporte a la Producción (PROQUIFA). El área ejecuta operaciones de alta, baja y modificación directamente en la base de datos del sistema.
- ** Estado del catálogo Perú: el catálogo de cuentas de Golocaer S.A.C. forma parte del alcance de R16, pero su modelo (estructura conforme a normativa SUNAT) aún no está definido y es una brecha prioritaria por resolver. Mientras no se defina, las cuentas Perú no pueden poblarse (ver Riesgo 1). **
La identificación automática de pagos entrantes con IA NO forma parte de R16: la asistencia con IA se limita a la captura de datos del cobro en Validar Cobro (ver TPSC-RE-FU-024)."									
TPSC-RE-FU-003		Mantenimiento de Catálogo de Clientes	Catálogo de Clientes	Yo como usuario del área Regulatorios, quiero cargar y mantener los documentos regulatorios de cada cliente (Licencia Sanitaria y Aviso de Responsable Sanitario) en el Catálogo de Clientes, para habilitar la pretramitación de pedidos con sustancias controladas y asegurar el cumplimiento normativo de PROQUIFA.	El sistema debe contar con una sección "Documentos Regulatorios" dentro del Catálogo de Clientes que permita al rol Regulatorios cargar, visualizar, reemplazar y eliminar los documentos PDF de Licencia Sanitaria y Aviso de Responsable Sanitario por cliente. Otros roles del sistema pueden visualizar la lista de documentos cargados pero no pueden modificarlos. El sistema no realiza validación de contenido, vigencia, firmas ni notificaciones de vencimiento. La sola presencia de ambos documentos cargados es lo que el módulo Pretramitar Pedido utiliza para validar y permitir la pretramitación de pedidos con sustancias controladas.			Propuesto					R16.3M-RE-FU-004	"## Aplica a

- Clientes de Región México en el Catálogo de Clientes (la sección de Documentos Regulatorios y su uso como condicionante regulatoria se construyen para México).
- Sección ""Documentos Regulatorios"" dentro de la pantalla del cliente.
- Dos documentos PDF por cliente: Licencia Sanitaria y Aviso de Responsable Sanitario.
- Operaciones disponibles para el rol Regulatorios sobre los documentos: carga (subida), visualización, reemplazo (subir nueva versión que sobrescribe la anterior) y eliminación.
- Visualización en modo bloqueado (solo consulta del listado de documentos cargados y de su contenido) para roles distintos de Regulatorios.
- Sobrescritura sin historial visible: al reemplazar un documento, la versión anterior se descarta del listado y solo permanece la versión vigente en pantalla.

## No aplica a

- Validación del contenido del PDF, firmas digitales, autenticidad del documento o vigencia regulatoria.
- Notificación automática de vencimiento o renovación de los documentos cargados.
- Conservación de historial de versiones de los documentos en el listado de pantalla (cada reemplazo descarta la versión anterior sin trazabilidad visual).
- Validaciones y condicionantes regulatorias de Sustancias Controladas para Región Perú: no se implementan en esta release. El sistema permite la existencia de familias controladas en cualquier región (no las restringe), pero para Perú no se construyen las validaciones regulatorias ni el bloqueo condicionante en Pretramitar Pedido; el manejo de controlados en Perú es operativo, no controlado por el sistema."	"## Reglas de Negocio

Regla 1 — Carga y modificación exclusivas del rol Regulatorios
La carga, reemplazo y eliminación de documentos regulatorios del cliente son operaciones exclusivas del rol Regulatorios. Para los demás roles los controles correspondientes se presentan en modo bloqueado.

Regla 2 — Formato exclusivo PDF
Los documentos regulatorios se aceptan únicamente en formato PDF. Cualquier otro formato de archivo es rechazado al momento de la carga.

Regla 3 — Una sola versión vigente por documento en pantalla
Cada cliente mantiene a lo más una versión vigente de cada documento regulatorio (Licencia Sanitaria y Aviso de Responsable Sanitario) visible en la pantalla del Catálogo de Clientes. Al subir una nueva versión, la versión anterior deja de listarse en pantalla y solo se muestra la vigente.

Regla 4 — Sin validación de vigencia ni contenido
El sistema no valida el contenido, autenticidad, firmas digitales ni fecha de vigencia de los documentos cargados, ni notifica vencimientos. La responsabilidad de mantener los documentos vigentes y correctos recae fuera del sistema, en el rol Regulatorios.

Regla 5 — Documentos Regulatorios construidos para Región México
La sección de Documentos Regulatorios y su uso como condicionante en Pretramitar Pedido se construyen para clientes de Región México. Para Región Perú no se implementan validaciones ni condicionantes regulatorias de Sustancias Controladas en esta release: el sistema no restringe la existencia de familias controladas, pero su manejo en Perú es operativo y no está soportado por validaciones del sistema.

## Riesgos

Riesgo 1 — Documentos cargados pero no vigentes
Como el sistema no valida la fecha de vigencia de los documentos cargados, un cliente podría tener Licencia Sanitaria o Aviso de Responsable Sanitario cargados pero vencidos legalmente. Operativamente esto permitiría tramitar pedidos con sustancias controladas sin documentación vigente, generando exposición regulatoria.

Riesgo 2 — Pérdida de trazabilidad histórica en pantalla por reemplazo sin versionado visible
Al sobrescribir un documento sin conservar la versión anterior en el listado de pantalla, se pierde la visibilidad en pantalla de qué Licencia Sanitaria estaba vigente en cada momento histórico. Esto puede generar problemas en auditorías regulatorias que requieran demostrar qué documento sustentaba un pedido pasado.

Riesgo 3 — Eliminación accidental
Como el rol Regulatorios puede eliminar documentos cargados, una eliminación accidental dejaría al cliente sin documentación, bloqueando inmediatamente cualquier tramitación con sustancias controladas hasta nueva carga.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — VISIBILIDAD Y ACCESO A LA SECCIÓN
═══════════════════════════════════════════════════════════════

Criterio A1 — Sección Documentos Regulatorios visible en el Catálogo de Clientes
Dado que un usuario abre el Catálogo de Clientes y consulta un cliente específico,
Cuando se renderiza la pantalla del cliente,
Entonces deberá presentarse la sección ""Documentos Regulatorios"" con el listado de los dos documentos posibles (Licencia Sanitaria y Aviso de Responsable Sanitario), indicando para cada uno si tiene archivo cargado o si está vacío.

Criterio A2 — Modo bloqueado para roles distintos de Regulatorios
Dado que un usuario con rol distinto de Regulatorios consulta la sección Documentos Regulatorios de un cliente,
Cuando visualiza la pantalla,
Entonces el sistema deberá presentar el listado de documentos cargados en modo bloqueado: el usuario puede ver qué documentos están registrados y consultar su contenido, pero no puede cargar, reemplazar ni eliminar.

═══════════════════════════════════════════════════════════════
SECCIÓN B — CARGA Y FORMATO DE LOS DOCUMENTOS
═══════════════════════════════════════════════════════════════

Criterio B1 — Carga de PDF por rol Regulatorios
Dado que el usuario tiene rol Regulatorios y consulta la sección Documentos Regulatorios de un cliente,
Cuando ejecuta la acción de cargar Licencia Sanitaria o Aviso de Responsable Sanitario,
Entonces el sistema deberá permitir seleccionar un archivo PDF del equipo del usuario y, al confirmar, almacenarlo asociado al cliente y al tipo de documento correspondiente.

Criterio B2 — Rechazo de formatos distintos a PDF
Dado que el rol Regulatorios intenta cargar un archivo en formato distinto a PDF,
Cuando el sistema valida el archivo,
Entonces deberá rechazar la operación, no almacenar el archivo y mostrar un mensaje claro indicando que solo se aceptan documentos en formato PDF.

═══════════════════════════════════════════════════════════════
SECCIÓN C — VISUALIZACIÓN DE LOS DOCUMENTOS
═══════════════════════════════════════════════════════════════

Criterio C1 — Entrega del PDF al navegador del usuario
Dado que un usuario consulta un documento previamente cargado,
Cuando ejecuta la acción de visualizar,
Entonces el sistema deberá entregar el archivo PDF al navegador del usuario. El comportamiento posterior de apertura (pestaña nueva, pestaña actual o descarga directa) depende de la configuración del navegador y de las preferencias del usuario, fuera del control del sistema.

═══════════════════════════════════════════════════════════════
SECCIÓN D — REEMPLAZO Y ELIMINACIÓN
═══════════════════════════════════════════════════════════════

Criterio D1 — Reemplazo de documento existente
Dado que el cliente ya tiene cargada una Licencia Sanitaria o un Aviso de Responsable Sanitario y el rol Regulatorios carga una nueva versión del mismo documento,
Cuando el sistema procesa la operación,
Entonces deberá registrar la nueva versión como vigente. En el listado de pantalla únicamente se muestra la versión vigente; la versión anterior deja de listarse.

Criterio D2 — Eliminación de documento cargado
Dado que el usuario tiene rol Regulatorios y un documento está cargado para un cliente,
Cuando ejecuta la acción de eliminar,
Entonces el sistema deberá solicitar confirmación explícita al usuario y, al confirmar, retirar el documento del catálogo del cliente. Tras la eliminación, el documento aparece como no cargado en el listado de pantalla.

═══════════════════════════════════════════════════════════════
SECCIÓN E — ALCANCE DE VALIDACIÓN
═══════════════════════════════════════════════════════════════

Criterio E1 — Sin validación de vigencia ni contenido
Dado que el rol Regulatorios cargó un documento PDF para un cliente,
Cuando el sistema almacena el archivo,
Entonces no deberá ejecutar ninguna validación de fecha de vigencia, contenido del documento ni autenticidad. El documento se considera vigente para fines del sistema por el solo hecho de estar cargado en el catálogo."								"- Funcionalidad ubicada en la sección ""Documentos Regulatorios"" del cliente dentro del Catálogo de Clientes.
- Los dos documentos contemplados son exclusivamente: Licencia Sanitaria y Aviso de Responsable Sanitario. La carga de otros tipos de documentos del cliente no entra en este alcance.
- El rol Regulatorios es el único autorizado para cargar, reemplazar, eliminar y visualizar el contenido de los PDFs. Otros roles ven la sección en modo bloqueado: pueden saber qué documentos están cargados y consultar su contenido pero no pueden modificarlos.
- Comportamiento de visualización del PDF: el sistema entrega el archivo al navegador del usuario. El comportamiento posterior (apertura en pestaña nueva, en la pestaña actual o descarga directa) depende de la configuración del navegador y de las preferencias del usuario, fuera del control del sistema. Sin visor embebido dentro del módulo.
- Comportamiento de eliminación / reemplazo a nivel de almacenamiento físico: el sistema mantiene el mecanismo actual del backend (almacenamiento en MinIO). Al eliminar o reemplazar un documento desde la pantalla del Catálogo de Clientes, se elimina o reemplaza el registro lógico que apunta al archivo, pero los archivos físicos previos permanecen en el almacenamiento MinIO sin purga automática. En pantalla del listado solo se muestra la versión vigente.
- El sistema no realiza validación de contenido, autenticidad, firmas digitales ni fechas de vigencia. La validación operativa que hace el módulo Pretramitar Pedido es exclusivamente sobre la presencia del registro, sin verificación adicional.
- El sistema no notifica automáticamente vencimientos ni próximas renovaciones. La responsabilidad de mantener los documentos vigentes recae offline en el rol Regulatorios.
- Esta funcionalidad es habilitadora de la validación bloqueante en el módulo Pretramitar Pedido para pedidos con sustancias controladas (ver requisito TPSC-RE-FU-009).
- La administración de roles del sistema (creación del rol Regulatorios, asignación usuario-rol) se gestiona a nivel base de datos."									
TPSC-RE-FU-005		Mantenimiento de Catálogo de Clientes	Catálogo de Clientes	Yo como usuario con acceso a la cartera de clientes, quiero capturar y mantener actualizada la configuración de cobros y facturación del cliente en el Catálogo de Clientes, contemplando las particularidades fiscales de México (SAT) y Perú (SUNAT), para que esos valores se utilicen como configuración default al generar proformas, facturas y comprobantes fiscales y se cumpla la normativa fiscal aplicable según la región del cliente.	El sistema debe contar en la sección Cobros del Catálogo de Clientes con los campos de configuración fiscal necesarios para que las proformas y facturas se emitan correctamente según la Región del cliente, presentando los campos correspondientes a México y los correspondientes a Perú según el caso. Los valores capturados funcionan como configuración default consumida por los módulos Factura por Adelantado y Validar Cobro, y son editables por cualquier usuario con acceso a la cartera del cliente.			Propuesto					(sin trazabilidad directa a la matriz original del cliente; emergente de sesiones y análisis fiscal Perú)	"## Aplica a

- Clientes de México y Perú en el Catálogo de Clientes.
- Sección Cobros dentro de la pantalla del cliente.
- Para Región México: tres campos (Forma de Pago, Uso de CFDI, Método de Pago) con los catálogos preexistentes del sistema.
- Para Región Perú: campos Condición de Pago (Contado/Crédito) y Tipo de Comprobante, con catálogos SUNAT. Comportamiento nuevo en R16.
- **Dos banderas tributarias para Región Perú, sujetas a confirmación de aplicabilidad con el cliente: Agente de Retención IGV (Sí/No) y Sujeto a Detracción (Sí/No con tasa cuando aplique).**
- Validación de obligatoriedad de los campos al guardar el cliente, según los aplicables por Región.
- Acceso libre a la edición por cualquier usuario con visibilidad sobre el cliente.
- Provisión de valores default consumidos por los módulos Factura por Adelantado y Validar Cobro al generar documentos fiscales.

## No aplica a

- Campo Forma de Pago (medio de pago) para Región Perú: la normativa SUNAT no exige declarar el medio de pago específico en el comprobante.
- Habilitación a nivel sistema y catálogo de productos requerida para emitir facturación electrónica SUNAT timbrada (datos fiscales del producto, código SUNAT del producto, tipo de afectación al IGV, unidad de medida SUNAT, integración con Operador de Servicios Electrónicos o sistema SEE-SOL de SUNAT, certificado digital del emisor Perú, Guía de Remisión Electrónica para despacho de mercancía). Estas son brechas reconocidas que se documentan en este requisito a nivel observación pero no son alcance de la presente fila.
- Otros campos de la sección Cobros distintos de los mencionados."	"## Reglas de Negocio

Regla 1 — Campos como configuración default del cliente
Los campos de la sección Cobros funcionan como configuración default del cliente. Estos valores se aplican automáticamente al generar proformas y facturas, salvo que se editen por operación en los módulos donde se permite.

Regla 2 — Catálogos de la sección Cobros diferenciados por Región
Los campos y catálogos de la sección Cobros se presentan en función de la Región del cliente. Para México se presentan Forma de Pago, Uso de CFDI y Método de Pago con los catálogos preexistentes del sistema. Para Perú se presentan Condición de Pago y Tipo de Comprobante con catálogos SUNAT, más las banderas tributarias aplicables.

Regla 3 — Condición de Pago como dimensión temporal del pago (Perú)
Para clientes Perú, el campo Condición de Pago (Contado/Crédito) expresa la dimensión temporal del pago conforme a la normativa SUNAT (Resolución de Superintendencia N° 193-2020/SUNAT). Es el equivalente conceptual del Método de Pago mexicano (PUE/PPD): Contado equivale a pago en una exhibición y Crédito a pago diferido.

Regla 4 — Método de Pago aplicable solo a México
Para clientes México, el Método de Pago se captura con el catálogo SAT de dos opciones: PUE (Pago en Una Exhibición) y PPD (Pago en Parcialidades o Diferido). Para clientes Perú este campo no se renderiza; la dimensión temporal del pago queda capturada en el campo Condición de Pago.

Regla 5 — Forma de Pago (medio) aplicable solo a México
Para clientes México, la Forma de Pago expresa el medio de pago con el catálogo preexistente del sistema. Para clientes Perú no se captura el medio de pago específico, ya que la normativa SUNAT no lo exige en el comprobante.

Regla 6 — Uso de CFDI y Tipo de Comprobante como campos independientes
El Uso de CFDI (México) y el Tipo de Comprobante (Perú) son conceptos fiscales distintos y se modelan como campos independientes con catálogos separados. El Uso de CFDI indica el uso fiscal que el receptor dará al comprobante; el Tipo de Comprobante indica la clase de documento emitido (Factura, Boleta o Recibo por Honorarios). Cada campo se renderiza según la Región del cliente.

Regla 7 — Agente de Retención IGV (Perú, sujeto a confirmación)
Para clientes Perú, el sistema contempla la bandera Agente de Retención IGV (Sí/No) que indica si el cliente está designado por SUNAT como Agente de Retención. Cuando el valor es Sí, las facturas emitidas a ese cliente deben contemplar la retención del IGV vigente (3%) y consignar la leyenda correspondiente; la lógica de cálculo y emisión vive en el módulo de facturación. La aplicabilidad de esta bandera a la cartera de PROQUIFA Perú está sujeta a confirmación con el cliente.

Regla 8 — Sujeto a Detracción (Perú, sujeto a confirmación)
Para clientes Perú, el sistema contempla la bandera Sujeto a Detracción (Sí/No). La detracción (SPOT) aplica por bien o servicio según los anexos de la R.S. 183-2004/SUNAT y solo a operaciones mayores a S/ 700; la ejecuta el comprador. La tasa aplicable se determina por el producto o servicio, no por el cliente. La aplicabilidad de esta bandera a los productos de PROQUIFA Perú está sujeta a confirmación con el cliente; la lógica de cálculo y emisión vive en el módulo de facturación.

Regla 9 — Edición sin restricción de rol
Cualquier usuario con acceso a la cartera del cliente puede modificar los campos de la sección Cobros. La autorización proviene del acceso del usuario al cliente, no de un rol específico.

## Riesgos

Riesgo 1 — Confusión por nomenclatura fiscal distinta entre México y Perú
Los campos Uso de CFDI (México) y Tipo de Comprobante (Perú) son conceptos fiscalmente distintos pero ocupan posiciones equivalentes en la sección Cobros. Igualmente, el Método de Pago (México) y la Condición de Pago (Perú) expresan la misma dimensión temporal con nomenclatura distinta. Esto puede generar confusión en usuarios que operen clientes de ambos países.

Riesgo 2 — Catálogos paramétricos desactualizados respecto a la normativa
Los catálogos de México (Forma de Pago, Uso de CFDI, Método de Pago) y SUNAT (Condición de Pago, Tipo de Comprobante) se actualizan periódicamente. Si no se mantienen sincronizados, los clientes podrían quedar con valores obsoletos que causen rechazo de timbrado.

Riesgo 3 — Brechas a nivel sistema y catálogo de productos para facturación electrónica Perú
La facturación electrónica SUNAT requiere capacidades que no están cubiertas a nivel sistema ni en el catálogo de productos actual de PROQUIFA. Aunque el catálogo de clientes capture la información fiscal correcta del cliente Perú, la emisión efectiva de un comprobante electrónico SUNAT timbrado depende de elementos adicionales que son brechas reconocidas en este punto del proyecto y que se detallan en las Observaciones.

Riesgo 4 — Detracciones y retenciones calculadas incorrectamente
La aplicación de detracciones (SPOT) y retenciones (Agente de Retención IGV) en la factura depende de que las banderas correspondientes estén correctamente capturadas y de que las reglas de cálculo se implementen correctamente en el módulo de facturación. Si se capturan mal, o si las reglas no se implementan, la factura puede emitirse con monto incorrecto generando problemas fiscales para PROQUIFA y para el cliente.

Riesgo 5 — Productos sujetos a Detracción no identificados en el catálogo de productos
La detracción aplica por producto/servicio (no por cliente). Un cliente podría estar marcado como Sujeto a Detracción cuando los productos que adquiere no aplican, o viceversa. Sin que el catálogo de productos identifique cada producto con su tasa de detracción aplicable, la factura puede calcular incorrectamente.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — VISUALIZACIÓN DE LOS CAMPOS SEGÚN REGIÓN
═══════════════════════════════════════════════════════════════

Criterio A1 — Visualización de los campos de Cobros según la Región del cliente
Dado que un usuario abre el Catálogo de Clientes y consulta un cliente,
Cuando se renderiza la sección Cobros,
Entonces el sistema deberá presentar para Región México los campos Forma de Pago, Uso de CFDI y Método de Pago; y para Región Perú los campos Condición de Pago, Tipo de Comprobante y las banderas tributarias aplicables (Agente de Retención IGV y Sujeto a Detracción). Los campos no aplicables a la Región no se renderizan.

═══════════════════════════════════════════════════════════════
SECCIÓN B — CAMPOS MÉXICO
═══════════════════════════════════════════════════════════════

Criterio B1 — Selector de Forma de Pago con catálogo del sistema
Dado que el cliente tiene Región = México,
Cuando el usuario despliega el selector de Forma de Pago,
Entonces el sistema deberá presentar el catálogo de Forma de Pago cargado en el sistema: Cheque, Depósito bancario, Efectivo, Tarjeta, Transferencia, Otros, Na y --ninguno--.

Criterio B2 — Selector de Uso de CFDI con catálogo del sistema
Dado que el cliente tiene Región = México,
Cuando el usuario despliega el selector de Uso de CFDI,
Entonces el sistema deberá presentar el catálogo de Uso de CFDI cargado en el sistema: G01 Adquisición de mercancías, G02 Devoluciones, descuentos o bonificaciones, G03 Gastos en general, S01 Sin efectos fiscales, Por definir y N/A.

Criterio B3 — Selector de Método de Pago con catálogo SAT
Dado que el cliente tiene Región = México,
Cuando el usuario despliega el selector de Método de Pago,
Entonces el sistema deberá presentar el catálogo SAT con dos opciones: PUE (Pago en Una Exhibición) y PPD (Pago en Parcialidades o Diferido).

═══════════════════════════════════════════════════════════════
SECCIÓN C — CAMPOS PERÚ (CATÁLOGOS SUNAT)
═══════════════════════════════════════════════════════════════

Criterio C1 — Selector de Condición de Pago con catálogo SUNAT
Dado que el cliente tiene Región = Perú,
Cuando el usuario despliega el selector de Condición de Pago,
Entonces el sistema deberá presentar el catálogo SUNAT con dos opciones: Contado y Crédito (conforme Resolución de Superintendencia N° 193-2020/SUNAT).

Criterio C2 — Selector de Tipo de Comprobante con catálogo SUNAT
Dado que el cliente tiene Región = Perú,
Cuando el usuario despliega el selector de Tipo de Comprobante,
Entonces el sistema deberá presentar el catálogo SUNAT con las opciones: Factura electrónica, Boleta de venta electrónica y Recibo por Honorarios electrónico. Este campo es independiente del Uso de CFDI mexicano y tiene su propio catálogo.

Criterio C3 — Método de Pago y Forma de Pago no renderizados para Perú
Dado que el cliente tiene Región = Perú,
Cuando se renderiza la sección Cobros,
Entonces los campos Método de Pago y Forma de Pago (medio) no deben aparecer en la pantalla. La dimensión temporal del pago queda capturada en el campo Condición de Pago.

═══════════════════════════════════════════════════════════════
SECCIÓN D — BANDERAS TRIBUTARIAS PERÚ
═══════════════════════════════════════════════════════════════

Criterio D1 — Bandera Agente de Retención IGV
Dado que el cliente tiene Región = Perú,
Cuando el usuario consulta la sección Cobros,
Entonces el sistema deberá presentar la bandera Agente de Retención IGV con opciones Sí y No (default propuesto: No). Si el valor es Sí, el sistema marca al cliente para que los módulos de facturación apliquen la retención del IGV vigente al emitir facturas. ** La aplicabilidad de esta bandera está sujeta a confirmación con el cliente sobre si su cartera Perú incluye clientes designados Agente de Retención por SUNAT. **

Criterio D2 — Bandera Sujeto a Detracción
Dado que el cliente tiene Región = Perú,
Cuando el usuario consulta la sección Cobros,
Entonces el sistema deberá presentar la bandera Sujeto a Detracción con opciones Sí y No (default propuesto: No). Si el valor es Sí, se habilita la captura de la tasa de detracción aplicable. ** La detracción aplica por bien o servicio según los anexos SUNAT, no por cliente, y solo a operaciones mayores a S/ 700; la aplicabilidad a los productos de PROQUIFA Perú está sujeta a confirmación con el cliente, así como si la tasa se captura a nivel cliente, a nivel producto o se determina desde el catálogo de productos. **

═══════════════════════════════════════════════════════════════
SECCIÓN E — OBLIGATORIEDAD Y PERSISTENCIA
═══════════════════════════════════════════════════════════════

Criterio E1 — Obligatoriedad al guardar (Región México)
Dado que el cliente tiene Región = México,
Cuando el usuario intenta guardar los datos del cliente,
Entonces el sistema deberá validar que los tres campos de Cobros (Forma de Pago, Uso de CFDI, Método de Pago) estén capturados. Si alguno está vacío, el guardado se bloquea.

Criterio E2 — Obligatoriedad al guardar (Región Perú)
Dado que el cliente tiene Región = Perú,
Cuando el usuario intenta guardar los datos del cliente,
Entonces el sistema deberá validar que estén capturados los campos Condición de Pago y Tipo de Comprobante, así como las banderas tributarias que se confirmen como aplicables. Si alguno de los campos obligatorios está vacío, el guardado se bloquea.

Criterio E3 — Edición sin restricción de rol
Dado que cualquier usuario con acceso al cliente abre la sección Cobros,
Cuando intenta modificar los campos,
Entonces el sistema deberá permitir la edición sin requerir un rol específico.

Criterio E4 — Persistencia de los valores como configuración default
Dado que el usuario guarda exitosamente los campos de Cobros del cliente,
Cuando el sistema procesa la operación,
Entonces deberá almacenar los valores asociados al cliente y dejarlos disponibles como configuración default para modulos posteriores."								"- Funcionalidad ubicada en la sección Cobros del cliente dentro del Catálogo de Clientes.
- Los campos para clientes México (Forma de Pago, Uso de CFDI, Método de Pago) ya existen en el sistema actual. R16 mantiene esa funcionalidad preexistente.
- Los campos para clientes Perú son nuevos en R16: Condición de Pago SUNAT (Contado/Crédito), Tipo de Comprobante SUNAT (Factura/Boleta/Recibo por Honorarios), ** Pendiente las banderas Agente de Retención IGV y Sujeto a Detracción. **
- El campo de dimensión temporal del pago para Perú se denomina Condición de Pago (Contado/Crédito) conforme a la Resolución de Superintendencia N° 193-2020/SUNAT. Es el equivalente conceptual del Método de Pago mexicano (PUE/PPD). Para Perú no se captura el medio de pago específico (Forma de Pago mexicana) porque la normativa SUNAT no lo exige en el comprobante.
- El Uso de CFDI (México) y el Tipo de Comprobante (Perú) se modelan como campos independientes con catálogos separados, dado que son conceptos fiscales distintos. No comparten campo en pantalla ni en base de datos, para evitar complicaciones al timbrar.
- Catálogo actual de Forma de Pago en PQF2 (México): Cheque, Depósito bancario, Efectivo, Tarjeta, Transferencia, Otros, Na, --ninguno--. Es un catálogo simplificado propio del sistema que no emplea la nomenclatura ni las claves del catálogo SAT c_FormaPago (01 Efectivo, 03 Transferencia electrónica, 04 Tarjeta de crédito, 99 Por definir, etc.). ** Pendiente validar si se requiere un mapeo de estas opciones a las claves del catálogo SAT para el timbrado del CFDI, dado que el XML del CFDI exige la clave SAT correspondiente. **
- Catálogo actual de Uso de CFDI en PQF2 (México): G01 Adquisición de mercancías, G02 Devoluciones/descuentos/bonificaciones, G03 Gastos en general, S01 Sin efectos fiscales, Por definir, N/A. Es un subconjunto del catálogo SAT c_UsoCFDI. Mantener el catálogo preexistente salvo indicación contraria del cliente.
- ** Pendiente confirmar con el cliente la denominación final del campo Condición de Pago para Perú. **
- Análisis de los mecanismos tributarios SUNAT relacionados con el IGV (Retención, Detracción, Percepción) y su aplicabilidad a la operación de venta de PROQUIFA Perú: ver archivo adjunto ""TPSC-RE-FU-005_Equivalencias_Cobros_MX_PE.xlsx"", hoja ""Mecanismos Tributarios PE"".
- ** Pendiente confirmar con el cliente si su cartera Perú incluye clientes designados Agente de Retención del IGV por SUNAT. Si no los tiene, la bandera Agente de Retención IGV es innecesaria y se evita su desarrollo. La retención (3% del IGV) la ejecuta el cliente comprador; PROQUIFA como vendedor solo consigna la leyenda y espera el pago con la retención aplicada. **
- ** Pendiente confirmar con el cliente si los productos o servicios de PROQUIFA Perú están sujetos a Detracción (SPOT) según los anexos de la R.S. 183-2004/SUNAT. La detracción aplica por bien o servicio (no por cliente), la ejecuta el comprador y solo aplica a operaciones mayores a S/ 700. Los productos químico-farmacéuticos no aparecen de forma evidente en los anexos de bienes sujetos; se requiere validar el catálogo exacto de productos. Si ninguno aplica, la bandera Sujeto a Detracción es innecesaria y se evita su desarrollo. Pendiente además definir si la tasa se captura a nivel cliente, a nivel producto o se determina desde el catálogo de productos. **
- ** Pendiente confirmar si PROQUIFA Perú (Golocaer) está designada por SUNAT como Agente de Percepción del IGV. La percepción es una condición del emisor (no del cliente); de aplicar, se configura a nivel emisor, no en el catálogo de clientes (ver Brecha 4). **
- Cualquier usuario con acceso a la cartera del cliente puede modificar los campos de la sección Cobros. No existe restricción de rol específica.
** Duda — Campo ""Tipo de Revisión"" (Digital / Física / Híbrida) en la sección de configuración fiscal del cliente: el campo existe en la pantalla actual de México (PQF2) pero no está documentado en ningún requisito de la matriz. Pendiente confirmar con el cliente: (1) si este campo entra dentro del alcance de R16 o queda fuera; (2) qué representa funcionalmente y qué reglas tiene (cómo afecta la generación o entrega del documento); (3) si aplica igual a Perú, considerando que es un dato operativo (no fiscal) y por tanto no dependería de SAT ni de SUNAT. **

═══════════════════════════════════════════════════════════════
BRECHAS RECONOCIDAS PARA HABILITAR FACTURACIÓN ELECTRÓNICA PERÚ
═══════════════════════════════════════════════════════════════

Este requisito captura en el catálogo del cliente la información fiscal necesaria del receptor del documento. Sin embargo, la emisión efectiva de una factura electrónica SUNAT timbrada requiere capacidades adicionales a nivel sistema y catálogo de productos que no están cubiertas en el alcance actual y se documentan a continuación como brechas reconocidas. Estas brechas se gestionan como Pendientes formales en el proyecto y se validarán con el cliente para acordar su tratamiento (alcance R16 completo, alcance progresivo o alcance reducido sin facturación electrónica integrada Perú).

Brecha 1 — Datos fiscales SUNAT en el catálogo de productos
Para emitir factura electrónica SUNAT, cada producto requiere campos fiscales del estándar SUNAT que no están modelados en el catálogo de productos actual de PROQUIFA: código SUNAT del producto, unidad de medida según catálogo SUNAT y tipo de afectación al IGV por línea. Son obligatorios en el XML UBL 2.1; sin ellos SUNAT rechaza la emisión. SAT no tiene equivalente a nivel línea. ** Patrón observado en una factura real de Golocaer (un solo caso, NO confirmado como general): Golocaer resuelve estos datos con valores genéricos únicos — un código de producto SUNAT genérico (""41116107"") para todas las líneas, unidad ""PIEZAS"" (código C62) y afectación al IGV ""gravado"" (18%). Pendiente validar con el cliente si este patrón genérico aplica siempre y, de ser así, si la solución para PQF2 es replicarlo en lugar de cargar datos SUNAT por producto. **

Brecha 2 — Guía de Remisión Electrónica para despacho de mercancía
Cuando PROQUIFA Perú despacha mercancía física al cliente, SUNAT requiere emitir la Guía de Remisión Electrónica (GRE) que acompaña al transporte. La factura referencia la guía. La GRE involucra datos del transportista, del vehículo, de la ruta y del receptor de la mercancía. Por qué es necesario: PROQUIFA Perú despacha mercancía (confirmado), por lo que cada operación de venta con entrega física requiere GRE además de la factura. Por qué se diferencia de México: SAT no exige el equivalente desde este flujo.

Brecha 3 — Tipo de Operación SUNAT (Catálogo 51) por factura
Cada factura electrónica Perú debe consignar un código de Tipo de Operación que identifica el contexto comercial (venta interna, exportación, anticipos, operación gratuita, etc.). Este código se captura por operación al emitir la factura, no en el catálogo del cliente. Por qué es necesario: campo obligatorio del XML UBL 2.1; SUNAT rechaza la emisión si falta o es incorrecto. Por qué se diferencia de México: SAT no tiene un campo equivalente; el contexto comercial se infiere de la combinación de otros campos del CFDI.

Brecha 4 — Régimen de Percepción del IGV de PROQUIFA como Agente de Percepción
Si PROQUIFA Perú está designada por SUNAT como Agente de Percepción, debe cobrar al cliente un porcentaje adicional al IGV (típicamente 2%) por ciertas operaciones. Es una condición del emisor (PROQUIFA), no del cliente. Por qué es necesario: si aplica, la factura debe emitirse considerando la percepción; omitirla genera contingencia fiscal para PROQUIFA. Por qué se diferencia de México: SAT no tiene un régimen equivalente. ** Pendiente confirmar si PROQUIFA Perú es Agente de Percepción designado por SUNAT. **

Brecha 5 — Configuración del emisor PROQUIFA Perú para facturación electrónica SUNAT
La emisión electrónica SUNAT requiere a nivel sistema: certificado digital vigente del emisor PROQUIFA Perú, designación/elección como emisor electrónico ante SUNAT, integración con SUNAT (vía Operador de Servicios Electrónicos OSE o vía SEE-SOL portal SUNAT) para envío de XMLs y recepción de Constancias de Recepción CDR, manejo de Resúmenes Diarios para boletas según volumen, y manejo de Comunicaciones de Baja para anulaciones. Por qué es necesario: sin esta infraestructura no se puede emitir un comprobante electrónico válido en Perú. Por qué se diferencia de México: en México la infraestructura está cubierta por la integración existente con TurboPac como PAC; en Perú se requiere infraestructura equivalente pero distinta (OSE o SEE-SOL).

═══════════════════════════════════════════════════════════════

- Para el detalle de los campos de la sección Cobros por Región, los catálogos y el análisis de los tres mecanismos tributarios peruanos, ver archivo adjunto ""TPSC-RE-FU-005_Equivalencias_Cobros_MX_PE.xlsx""."									
TPSC-RE-FU-006		Mantenimiento de Catálogo de Clientes	Catálogo de Clientes	Yo como usuario con acceso a la cartera de clientes, quiero asignar una o más cuentas bancarias del grupo PROQUIFA a cada cliente y capturar el Código Validador correspondiente a cada combinación cliente-cuenta, para que el sistema construya automáticamente la referencia bancaria que aparece en las proformas del cliente, permitiendo identificar correctamente los pagos recibidos en cada cuenta.	El sistema debe contar en la sección Cobros del Catálogo de Clientes con una sección "Referencia de Pago" que permita asignar al cliente una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA, capturando para cada combinación cliente-cuenta un Código Validador. La referencia bancaria que aparece en cada proforma se reconstruye dinámicamente al generarla, aplicando reglas diferenciadas según el banco asociado a la cuenta seleccionada. La funcionalidad es nueva en PQF2 R16 y toma como referencia operativa el comportamiento equivalente del sistema Legacy actual.			Propuesto					R16.3M-RE-FU-005	"## Aplica a
- Clientes de México en el Catálogo de Clientes.
- Pantalla ""Referencia de Pago"" como sub-sección dentro de la sección Cobros del cliente.
- Asignación de una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA al cliente. ** Pendiente de confirmar si es posible **
- Captura manual del Código Validador para cada combinación cliente-cuenta.
- Reconstrucción dinámica de la referencia bancaria en cada generación de proforma asociada a la cuenta seleccionada.
- Replicación de la lógica documentada del sistema Legacy actual sobre Banamex (7 segmentos) y no-Banamex (nombre del cliente).
- Migración de los datos actuales del sistema Legacy a ProQuiFaNet 2

## No aplica a
- Clientes de Perú: ** queda fuera del alcance de este requisito en R16 hasta definir con el cliente el modelo bancario peruano de identificación de pagos. Levantada como duda formal del proyecto. **
- Validación de formato o longitud del Código Validador (** pendiente definir con el cliente si se requiere validación; el documento del cliente no especifica longitud máxima ni reglas de formato **).
- Persistencia de la referencia armada (la referencia se reconstruye dinámicamente; no se almacena).
- Historial de cambios al Código Validador (al modificar el valor se sobrescribe el anterior sin trazabilidad de versiones)."	"## Reglas de Negocio

Regla 1 — Pantalla ""Referencia de Pago"" en sección Cobros
El Catálogo de Clientes cuenta con una sub-sección ""Referencia de Pago"" dentro de la sección Cobros, donde se gestionan la o las cuentas bancarias asignadas al cliente y su Código Validador correspondiente por combinación cliente-cuenta.

Regla 2 — Asignación de una o más cuentas bancarias al cliente
Un cliente puede tener asignadas una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA. Cada asignación se compone seleccionando primero el Banco (del catálogo de Bancos) y luego la Cuenta (del catálogo de cuentas del grupo, filtrado por el banco seleccionado). La Sucursal se hereda del dato de la cuenta seleccionada y es de solo lectura. ** Validar con el cliente si se requiere algún tope o nada mas es una**

Regla 3 — Código Validador por combinación cliente-cuenta
Cada combinación cliente-cuenta tiene un Código Validador capturado manualmente por el usuario. ** Pendiente: el documento del cliente no especifica longitud máxima ni reglas de formato del Código Validador; queda como duda formal del proyecto antes del desarrollo. **

Regla 4 — Persistencia de la combinación cliente-cuenta-Código Validador
La asignación persiste en la relación cliente-cuenta los datos: identificador de la cuenta bancaria, identificador del cliente y Código Validador capturado. La referencia bancaria armada que aparece en la proforma no se almacena.

Regla 5 — Reconstrucción dinámica de la referencia en cada proforma
La referencia bancaria que aparece en una proforma se reconstruye dinámicamente en el momento de generar la proforma, aplicando las reglas correspondientes según el banco de la cuenta. La referencia no se persiste; cada generación de proforma reconstruye el valor.

Regla 6 — Referencia para bancos distintos de Banamex
Para cuentas de bancos distintos de Banamex, la referencia bancaria es el Nombre del cliente como cadena directa, sin transformación adicional.

Regla 7 — Referencia para Banamex (concatenación de 7 segmentos)
Para cuentas de Banamex, la referencia bancaria se compone por la concatenación determinística de 7 segmentos:
1. Primera letra del nombre del cliente (fallback ""X"" si no existe).
2. Segunda letra del nombre del cliente (fallback ""X"" si no existe).
3. Tercera letra del nombre del cliente (fallback ""X"" si no existe).
4. Últimos 4 caracteres de la clave del cliente (padding con ceros a la izquierda si la clave tiene menos de 4 caracteres).
5. Código del banco (campo Codigo de la tabla Bancos).
6. Carácter ""P"" si la primera letra del campo Moneda de la cuenta es ""M"" (peso); carácter ""D"" en cualquier otro caso.
7. Código Validador (campo CodValidador de la relación cliente-cuenta).

Regla 8 — Identificación del banco ""Banamex""
La determinación de si una cuenta pertenece a Banamex se realiza cruzando la cuenta contra la tabla de Empresas: el campo beneficiario de la cuenta debe coincidir con el prefijo de la empresa que factura, y la moneda debe cumplir una condición adicional. ** La condición adicional de moneda aparece truncada en la documentación del cliente; queda como duda formal de desarrollo. Como simplificación alternativa, se propone para evaluación con desarrollo identificar Banamex directamente por nombre o ID del banco en la tabla Bancos. **

Regla 9 — Edición sin restricción de rol
Cualquier usuario con acceso a la cartera del cliente puede asignar cuentas bancarias y capturar el Código Validador. La autorización proviene del acceso del usuario al cliente, no de un rol específico. ** Pendiente validar con el cliente si esta funcionalidad debe restringirse a un rol específico (probablemente Coordinador de Tesorería) por sus implicaciones financieras. **

## Riesgos

Riesgo 1 — Inconsistencia entre proformas emitidas por reconstrucción dinámica
Como la referencia bancaria se reconstruye en cada generación de proforma y no se almacena, si entre dos generaciones cambia el nombre del cliente, su clave, el Código Validador o cualquier dato fuente, la nueva proforma tendrá una referencia distinta a la original. Esto puede causar que un cliente pague con la referencia original (de una proforma anterior que conservó) y el sistema no identifique el pago correctamente al reconstruir la referencia con datos actualizados.

Riesgo 2 — Código Validador sin validación de formato ni longitud
Como el Código Validador es input manual sin validación, un usuario puede capturar un valor erróneo (espacios, caracteres especiales, longitud incompatible con el sistema bancario) que rompa la identificación de pagos.

Riesgo 3 — Sin restricción de rol sobre asignación de cuentas bancarias
La asignación de cuentas bancarias del grupo PROQUIFA a clientes y la captura del Código Validador tienen implicaciones financieras serias (errores en estos datos comprometen la identificación de pagos). Sin embargo, esta funcionalidad se modela inicialmente sin restricción de rol (cualquier usuario con acceso al cliente puede modificar). ** Pendiente validar con el cliente si debe restringirse a un rol específico (probablemente Coordinador de Tesorería). **

Riesgo 4 — Modelo Perú no definido
La lógica documentada por el cliente corresponde exclusivamente a PROQUIFA México (Banamex, prefijos de empresas mexicanas, moneda en pesos/dólares). No existe documentación equivalente para el modelo bancario peruano de identificación de pagos. ** Queda como duda formal del proyecto antes de definir el alcance Perú para esta funcionalidad. Mientras no se resuelva, los clientes Perú no podrán tener referencia bancaria armada por el sistema PQF2. **

Riesgo 5 — Pérdida de trazabilidad por sobrescritura del Código Validador
Al modificar el Código Validador asignado a una combinación cliente-cuenta, el valor anterior se sobrescribe sin conservar historial. Esto puede generar problemas en auditorías o reconciliación de pagos pasados que fueron identificados con el valor anterior.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — ACCESO Y SELECCIÓN DE CUENTA
═══════════════════════════════════════════════════════════════

Criterio A1 — Acceso a pantalla ""Referencia de Pago"" desde sección Cobros
Dado que un usuario abre el Catálogo de Clientes y consulta un cliente,
Cuando navega a la sección Cobros,
Entonces el sistema deberá ofrecer acceso a la sub-sección ""Referencia de Pago"" para gestionar las cuentas bancarias asignadas al cliente y sus Códigos Validadores.

Criterio A2 — Selector de Banco desde catálogo de Bancos
Dado que un usuario en ""Referencia de Pago"" agrega una cuenta nueva,
Cuando despliega el selector de Banco,
Entonces el sistema deberá presentar las opciones del catálogo de Bancos del grupo PROQUIFA.

Criterio A3 — Selector de Cuenta filtrado por Banco
Dado que el usuario seleccionó un Banco en ""Referencia de Pago"",
Cuando despliega el selector de Cuenta,
Entonces el sistema deberá presentar únicamente las cuentas bancarias del catálogo del grupo PROQUIFA que correspondan al banco seleccionado.

Criterio A4 — Sucursal autopoblada desde la cuenta seleccionada
Dado que el usuario seleccionó una Cuenta en ""Referencia de Pago"",
Cuando se renderiza el campo Sucursal,
Entonces el sistema deberá autopopularlo con el valor del campo Sucursal de la cuenta, en modo solo lectura (no editable por el usuario en esta pantalla).

═══════════════════════════════════════════════════════════════
SECCIÓN B — CÓDIGO VALIDADOR Y PERSISTENCIA
═══════════════════════════════════════════════════════════════

Criterio B1 — Captura manual del Código Validador
Dado que el usuario configura una combinación cliente-cuenta,
Cuando captura el campo Código Validador,
Entonces el sistema deberá aceptar el valor como input manual del usuario sin aplicar validación de formato ni longitud en esta versión. ** Pendiente definir reglas de validación con el cliente; queda como duda formal del proyecto. **

Criterio B2 — Persistencia de la combinación cliente-cuenta-Código Validador
Dado que el usuario guarda una asignación nueva o modificada,
Cuando el sistema procesa la operación,
Entonces deberá persistir la combinación en la relación cliente-cuenta del modelo de datos. La referencia armada no se almacena.

Criterio B3 — Múltiples cuentas asignables por cliente
Dado que un cliente ya tiene una o más cuentas bancarias asignadas,
Cuando el usuario agrega una cuenta adicional,
Entonces el sistema deberá permitir la asignación. No existe límite máximo de cuentas asignables por cliente en R16. ** Validar con el cliente si se requiere algún tope. **

Criterio B4 — Edición y eliminación de combinaciones
Dado que un cliente tiene una cuenta bancaria asignada con su Código Validador,
Cuando el usuario edita o elimina la asignación,
Entonces el sistema deberá permitir la operación. La edición sobrescribe el valor anterior sin conservar historial. La eliminación retira la combinación cliente-cuenta del sistema.

Criterio B5 — Edición sin restricción de rol específica
Dado que cualquier usuario con acceso a la cartera del cliente abre la pantalla ""Referencia de Pago"",
Cuando intenta modificar la asignación de cuentas o el Código Validador,
Entonces el sistema deberá permitir la edición sin requerir un rol específico. ** Condición inicial propuesta; queda como duda formal del proyecto validar con el cliente si se debe restringir al rol Coordinador de Tesorería u otro por las implicaciones financieras. **

═══════════════════════════════════════════════════════════════
SECCIÓN C — RECONSTRUCCIÓN DE LA REFERENCIA BANCARIA
═══════════════════════════════════════════════════════════════

Criterio C1 — Reconstrucción dinámica al generar la proforma
Dado que un módulo genera una proforma para el cliente con una cuenta bancaria asignada,
Cuando se incorpora la referencia bancaria al PDF de la proforma,
Entonces el sistema deberá reconstruir la referencia en ese momento aplicando las reglas según el banco de la cuenta, sin persistir el valor.

Criterio C2 — Referencia para bancos distintos de Banamex
Dado que el banco de la cuenta asignada para una proforma no es Banamex,
Cuando el sistema construye la referencia bancaria,
Entonces la referencia será el Nombre del cliente como cadena directa, sin transformación adicional.

Criterio C3 — Referencia para Banamex (7 segmentos)
Dado que el banco de la cuenta asignada para una proforma es Banamex,
Cuando el sistema construye la referencia bancaria,
Entonces deberá concatenar determinísticamente los 7 segmentos definidos en la Regla 7 (tres primeras letras del nombre del cliente con fallback ""X"", últimos 4 caracteres de la clave del cliente con padding de ceros, código del banco, carácter de moneda ""P""/""D"", y Código Validador)."								"- Funcionalidad ubicada en la pantalla ""Referencia de Pago"", sub-sección dentro de la sección Cobros del cliente en el Catálogo de Clientes.
- Funcionalidad NUEVA en PQF2 R16. Toma como referencia el comportamiento documentado del sistema Legacy actual de PROQUIFA, pero su implementación en PQF2 es desde cero.
- Cubre el requisito del cliente sobre la generación de la Referencia de Cliente / Código Validador para identificación bancaria de pagos.
- El modelo de datos involucra una relación N:N entre Cliente y CuentaBanco mediante una tabla cliente-cuenta que persiste el Código Validador por combinación.
- La referencia bancaria armada se reconstruye dinámicamente en cada generación de proforma; NO se almacena en base de datos. ** Esta decisión de diseño replica el comportamiento del Legacy y genera el riesgo documentado (Riesgo 1) de inconsistencia entre proformas emitidas. **
- La lógica de Banamex (7 segmentos) y no-Banamex (nombre del cliente) replica 1:1 el comportamiento documentado por el cliente en el documento ""Especificación: Proceso para generar Referencia de Cliente (Código Validador)"" recibido el 2026-04-28.
- ** Pendiente: la condición de identificación de Banamex referente al campo moneda de la cuenta aparece truncada en el documento del cliente (""El campo moneda de la tabla CuentaBanco debe..."") y no se ha clarificado. Queda como duda formal de desarrollo. **
- ** Pendiente: longitud máxima y reglas de formato del Código Validador no especificadas por el cliente. El PMO del proyecto anunciaba que la información se enviaría pero no llegó. Queda como duda formal del proyecto. **
- ** Pendiente: aplicabilidad de esta funcionalidad para clientes Perú. La documentación del cliente cubre exclusivamente PROQUIFA México. El modelo bancario peruano de identificación de pagos no está definido. Queda como duda formal del proyecto. **
- ** Propuesta de simplificación para desarrollo: la identificación del banco Banamex actualmente requiere cruzar la cuenta contra la tabla Empresas (campo beneficiario vs prefijo) más una condición de moneda. Se propone evaluar con desarrollo identificar Banamex directamente por nombre o ID en la tabla Bancos, simplificando la mantenibilidad. **
- ** Pendiente: validar con el cliente si la asignación de cuentas bancarias y captura del Código Validador debe restringirse al rol Coordinador de Tesorería u otro rol específico, considerando las implicaciones financieras. La condición inicial propuesta es sin restricción de rol. **
- La funcionalidad provee insumos al módulo Buzón de Cobros (identificación automática de pagos entrantes contra la referencia armada). La integración entre ambos módulos se detalla en los requisitos correspondientes al Buzón de Cobros.
"									
TPSC-RE-FU-007		Notificación regulatoria al cliente	Cotizar lo Cotizable	Yo como Asesor Comercial, quiero que el sistema agregue automáticamente una leyenda regulatoria visible en el PDF de la cotización definitiva entregada al cliente cuando la cotización contiene al menos una partida de producto clasificado como Sustancia Controlada (Mundial, Nacional u Origen), para que el cliente conozca desde el primer artefacto comercial qué documentos regulatorios debe tener para que su pedido pueda procesarse, evitando re-trabajos y fricciones tardías en el flujo de pretramitación.	El sistema debe agregar automáticamente una leyenda regulatoria al PDF de la cotización definitiva generada por el módulo Cotizar lo Cotizable cuando la cotización contenga al menos una partida de producto clasificado como Sustancia Controlada tipo Mundial, Nacional u Origen. La leyenda aplica únicamente a las cotizaciones definitivas e informa al cliente que el pedido requiere, para su procesamiento, la entrega de los documentos regulatorios correspondientes (Licencia Sanitaria y Aviso de Responsable Sanitario para Región México. La leyenda no bloquea la generación del PDF: la cotización definitiva se entrega siempre.			Propuesto					(sin trazabilidad directa a las matrices originales del cliente; emergente del archivo PMO 5-may fila #68)	"## Aplica a

- Cotizaciones definitivas generadas en el módulo Cotizar lo Cotizable que contengan al menos una partida de producto clasificado como Sustancia Controlada tipo Mundial, Nacional u Origen.
- Adición de una leyenda regulatoria visible en el PDF de la cotización definitiva entregada al cliente.
- Aplicación a cotizaciones de clientes de Región México.
- Inclusión de la leyenda como información general del documento, sin desglose por partida.
- Comportamiento no bloqueante: el PDF de la cotización definitiva se genera y entrega al cliente independientemente de si ya tiene o no documentos regulatorios cargados en el Catálogo de Clientes.

## No aplica a

- Región Perú: el manejo de Sustancias Controladas no está soportado en este release, por lo que la leyenda regulatoria no aplica a cotizaciones de clientes de Perú. El soporte de controlados para Perú queda como riesgo operativo (ver pendiente en Observaciones).
- Cotizaciones de investigación: no incluyen la leyenda regulatoria. Las cotizaciones de investigación tienen su propia leyenda de trabajo en curso, que es funcionalidad preexistente del módulo y no es alcance de este requisito.
- Cotizaciones definitivas que NO contienen ninguna partida de producto clasificado como Sustancia Controlada (esas cotizaciones se generan sin la leyenda regulatoria).
- Validación de presencia de documentos regulatorios en el Catálogo de Clientes (esa validación bloqueante vive en el módulo Pretramitar Pedido, requisito TPSC-RE-FU-009).
- Otros cambios al diseño general del PDF de cotización (mantiene su layout, colores y estructura actuales; solo se agrega la leyenda regulatoria)."	"## Reglas de Negocio

Regla 1 — Leyenda regulatoria solo en cotizaciones definitivas
La leyenda regulatoria aplica exclusivamente a las cotizaciones definitivas del módulo Cotizar lo Cotizable. Las cotizaciones de investigación no incluyen la leyenda regulatoria.

Regla 2 — Detonante de la leyenda regulatoria
La leyenda regulatoria se agrega a la cotización definitiva cuando al menos una de sus partidas corresponde a un producto clasificado como Sustancia Controlada tipo Mundial, Nacional u Origen. Si ninguna partida es controlada, el PDF se genera sin la leyenda.

Regla 3 — Leyenda general a nivel documento
La leyenda regulatoria se incorpora una sola vez como nota general del documento, sin desglose por partida. El número de partidas controladas no afecta cuántas veces aparece: una sola aparición por PDF.

Regla 4 — Contenido de la leyenda (Región México)
El texto de la leyenda regulatoria referencia, para clientes de Región México, la Licencia Sanitaria y el Aviso de Responsable Sanitario (denominaciones COFEPRIS). La leyenda aplica únicamente a México; Región Perú no está soportada para el manejo de Sustancias Controladas en esta release.

Regla 5 — Leyenda no bloqueante
La leyenda regulatoria es informativa, no validativa. El sistema genera y entrega la cotización definitiva con la leyenda incorporada, sin bloquear la generación del PDF.

Regla 6 — Sin consulta al Catálogo de Clientes
La leyenda se agrega siempre que haya al menos una partida controlada en la cotización definitiva, sin importar si el cliente ya tiene documentos regulatorios cargados en el Catálogo. La leyenda funciona como recordatorio universal del requisito regulatorio para la operación con sustancias controladas.

## Riesgos

Riesgo 1 — Cliente confundido al recibir leyenda redundante
La leyenda aparece siempre que haya productos controlados, incluso si el cliente ya tiene cargados todos sus documentos regulatorios en el Catálogo. Esto puede generar fricción operativa: clientes recurrentes que reciben siempre la misma notificación y podrían interpretarla como un recordatorio innecesario.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — APLICABILIDAD A COTIZACIONES DEFINITIVAS
═══════════════════════════════════════════════════════════════

Criterio A1 — Leyenda regulatoria solo en cotizaciones definitivas
Dado que un usuario genera una cotización en el módulo Cotizar lo Cotizable,
Cuando la cotización es de investigación,
Entonces el sistema NO deberá agregar la leyenda regulatoria. La cotización de investigación conserva su comportamiento preexistente.

Criterio A2 — Detección de cotización definitiva con productos controlados
Dado que un usuario genera una cotización definitiva en el módulo Cotizar lo Cotizable,
Cuando el sistema procesa el PDF,
Entonces deberá inspeccionar las partidas de la cotización y determinar si al menos una corresponde a un producto clasificado como Sustancia Controlada tipo Mundial, Nacional u Origen.

═══════════════════════════════════════════════════════════════
SECCIÓN B — INCLUSIÓN DE LA LEYENDA
═══════════════════════════════════════════════════════════════

Criterio B1 — Inclusión de la leyenda regulatoria en cotización definitiva con controlados
Dado que una cotización definitiva contiene al menos una partida controlada,
Cuando el sistema genera el PDF,
Entonces deberá incluir la leyenda regulatoria visible en una posición claramente identificable del documento. ** La ubicación exacta queda como decisión de diseño UI: encabezado, sección dedicada antes de la firma, pie de página, etc. ** La leyenda informa al cliente que el pedido requiere la entrega de los documentos regulatorios para procesarse.

Criterio B2 — No inclusión de la leyenda en cotización definitiva sin controlados
Dado que una cotización definitiva NO contiene ninguna partida controlada,
Cuando el sistema genera el PDF,
Entonces NO deberá incluir la leyenda regulatoria. El PDF se genera con el layout estándar del módulo sin la adición.

Criterio B3 — Una sola aparición por documento
Dado que una cotización definitiva contiene múltiples partidas controladas,
Cuando el sistema agrega la leyenda al PDF,
Entonces deberá incluirla una sola vez como nota general del documento, sin repetirla por cada partida ni desglosarla por tipo de control.

Criterio B4 — Generación del PDF no bloqueada por la leyenda
Dado que una cotización definitiva contiene partidas controladas,
Cuando el sistema procesa la generación del PDF,
Entonces deberá completar la generación y entregar el documento al cliente, independientemente del estado del catálogo del cliente respecto a sus documentos regulatorios cargados. La leyenda es informativa, no bloquea ni interrumpe el flujo.

═══════════════════════════════════════════════════════════════
SECCIÓN C — TEXTO DE LA LEYENDA SEGÚN REGIÓN
═══════════════════════════════════════════════════════════════

Criterio C1 — Texto de la leyenda para clientes México
Dado que el cliente de la cotización definitiva tiene Región = México,
Cuando el sistema arma el texto de la leyenda,
Entonces deberá usar la denominación México: referenciar Licencia Sanitaria y Aviso de Responsable Sanitario como documentos requeridos para procesar el pedido. ** El texto exacto queda como decisión de UX/Marketing; propuesta base: ""Producto sujeto a regulación sanitaria. Para procesar el pedido se requiere: Licencia Sanitaria vigente y Aviso de Responsable Sanitario."" **"								"- Funcionalidad nueva en el módulo Cotizar lo Cotizable, que es un módulo preexistente en PQF2. Este es el único cambio funcional que R16 introduce al módulo; el resto del módulo opera conforme al sistema preexistente sin modificaciones.
- La leyenda regulatoria aplica únicamente a las cotizaciones definitivas. Las cotizaciones de investigación tienen su propia leyenda de trabajo en curso, que es funcionalidad preexistente del módulo y no forma parte del alcance de este requisito.
- La leyenda es general a nivel del PDF de la cotización definitiva: aparece una sola vez como nota informativa del documento, no por partida. El detonante es la presencia de al menos una partida de producto controlado (Mundial, Nacional u Origen) en la cotización.
- La leyenda NO bloquea la generación del PDF. La validación bloqueante para que un pedido pueda procesarse cuando el cliente tiene productos controlados vive en el módulo Pretramitar Pedido (requisito TPSC-RE-FU-009), que verifica que el cliente tenga registrados los documentos regulatorios en su Catálogo. La leyenda en cotización funciona como recordatorio proactivo previo a esa validación.
- ** Pendiente: ubicación exacta de la leyenda en el PDF (encabezado, sección dedicada, pie de página). Queda como decisión de diseño UI a definir en sprint de desarrollo. **
- ** Pendiente: texto exacto de la leyenda para clientes México. El texto definitivo es una decisión de UX/Marketing del cliente. **
- ** Propuesta abierta para evaluación con el cliente: variante dinámica de la leyenda que consulte el Catálogo de Clientes y solo agregue la nota cuando el cliente NO tenga cargados los documentos regulatorios. Esto evitaría ruido para clientes recurrentes que ya cumplieron con su documentación. La propuesta confirmada actual es la leyenda universal (siempre que haya productos controlados en una cotización definitiva). **
- La leyenda funciona como complemento al ciclo regulatorio del proyecto: carga de documentos regulatorios en el Catálogo de Clientes, validación bloqueante en Pretramitar Pedido. El cliente cumple su circuito completo: avisado en cotización definitiva, validado en pretrámite, soportado en el catálogo.
- El manejo de Sustancias Controladas para Región Perú no está soportado en esta release y no se restringe por código; por ello la leyenda regulatoria no aplica a cotizaciones de clientes Perú. El avance de un pedido con controlados de Perú hacia tramitación/facturación se asume como riesgo operativo (no de sistema); el riesgo formal está documentado en los Criterios de Aceptación de TPSC-RE-FU-011 y TPSC-RE-FU-013."									
TPSC-RE-FU-008		Buzón de Cobros	Buzones	Yo como Gestor de Cobranza, quiero contar con un Buzón de Cobros que reciba automáticamente los correos clasificados como cobros enviados por mis clientes asignados, genere un pendiente en Validar Cobro sin captura previa de datos y me permita reclasificar correos mal clasificados, para reducir el trabajo manual de identificación inicial de cobros.	El sistema debe contar con un módulo Buzón de Cobros nuevo en PQF2 que reciba automáticamente los correos clasificados como cobros por el Mailbot, sumando la clasificación "Cobro" a las clasificaciones existentes del sistema. Cada correo clasificado como cobro se refleja en el Buzón del Gestor de Cobranza correspondiente y dispara la generación automática de un pendiente en el módulo Validar Cobro. La visibilidad del Buzón es por cobrador asignado: cada Gestor de Cobranza ve únicamente los correos de los clientes que tiene asignados en su cartera. El Gestor de Cobranza puede reclasificar manualmente un correo cuando detecta que fue clasificado incorrectamente. Adicionalmente, el Buzón de Cobros cuenta con una bandeja del Coordinador de Tesorería que concentra los correos de cobro que no pueden enrutarse a un Gestor: los de clientes existentes sin Cobrador asignado y los de correos cuyo remitente no está dado de alta como contacto de ningún cliente. Desde esta bandeja, según el caso, se asigna el Cobrador (lo que mueve el correo a la bandeja de ese Cobrador) o se canaliza el alta del contacto por el flujo operativo existente.			Propuesto					 R16.2M-RE-FU-004	"## Aplica a

- Módulo en PQF2: Buzón de Cobros.
- Recepción de correos clasificados automáticamente como cobros por el Mailbot del sistema.
- Visibilidad de correos en el Buzón filtrada por la asignación del cobrador (cada Gestor de Cobranza ve los correos de los clientes que tiene asignados).
- Generación automática de un pendiente en Validar Cobro al clasificarse cada correo como cobro, sin captura previa de datos.
- Aplicación del mismo patrón de filtros, búsqueda y paginación que los Buzones preexistentes (Buzón de Requisiciones, Buzón de Pedidos).
- Cierre automático del pendiente del Buzón cuando el cobro se vincula a una proforma o factura en Validar Cobro.
- Eliminación automática del pendiente del Buzón cuando el cobro se marca como inconsistencia en Validar Cobro.
- Acción manual de reclasificación: el Gestor de Cobranza puede mover un correo a otro buzón si la clasificación automática fue incorrecta.
- Bandeja del Coordinador de Tesorería dentro del Buzón de Cobros: concentra los correos que no pueden enrutarse a un Gestor (cliente existente sin Cobrador asignado, o remitente no dado de alta como contacto), con la acción de asignar Cobrador o canalizar el alta del contacto según el caso.
- Aplicación a clientes de México y Perú.

## No aplica a

- Captura de datos del cobro (monto, cliente, banco emisor, cuenta origen, fecha del depósito, referencia bancaria). Esa captura ocurre la primera vez que se trabaja el pendiente en el módulo Validar Cobro.
- Eliminación directa de correos por parte del Gestor de Cobranza desde el Buzón de Cobros. La salida del correo del Buzón ocurre por los eventos del ciclo de vida (vinculación exitosa o inconsistencia en Validar Cobro) o por reclasificación manual hacia otro buzón.
- Definición del envío de notificación al cliente ante una inconsistencia. Ese comportamiento pertenece al módulo Validar Cobro (requisito independiente); en este requisito solo se contempla que el pendiente del Buzón se elimina cuando el cobro se marca como inconsistencia.
- Criterios configurables de clasificación por parte del usuario. El Mailbot se entrena con una base de conocimiento; los ajustes a la clasificación se realizan mediante un nuevo entrenamiento, no mediante parámetros configurables en la interfaz."	"## Reglas de Negocio

Regla 1 — Clasificación automática del correo por el Mailbot
El Mailbot del sistema clasifica automáticamente cada correo entrante en una de las categorías del modelo de clasificación: Cotización, Pedido, Cobro u Otros. R16 agrega la categoría Cobro al modelo existente; no modifica el resto del modelo. Solo los correos clasificados como Cobro entran al Buzón de Cobros.

Regla 2 — Clasificación basada en entrenamiento, sin criterios configurables
La clasificación del Mailbot se basa en una base de conocimiento entrenada. Los ajustes a la precisión de la clasificación se realizan mediante un nuevo entrenamiento del Mailbot, no mediante criterios configurables por el usuario en la interfaz.

Regla 3 — Reflejo del correo clasificado en el Buzón
Un correo clasificado como Cobro se refleja automáticamente en el Buzón de Cobros del Gestor de Cobranza que tenga asignado al cliente identificado en el correo. La visibilidad del correo en el Buzón es exclusiva del Gestor asignado al cliente.

Regla 4 — Generación automática de pendiente en Validar Cobro sin captura previa
Un correo clasificado como Cobro genera simultánea y automáticamente un pendiente en el módulo Validar Cobro asociado al correo, sin capturar previamente datos del cobro. La captura del monto y demás datos del cobro ocurre la primera vez que se trabaja el pendiente en Validar Cobro.

Regla 5 — Visibilidad por cobrador asignado y por región del cliente
La bandeja del Buzón de un Gestor de Cobranza muestra únicamente los correos asociados a clientes que tiene asignados en su cartera (campo Cobrador del Catálogo de Clientes). Los correos de clientes asignados a otros Gestores no aparecen en su Buzón. Adicionalmente, cada correo se refleja siempre en el contexto de la región del cliente que lo originó: un cliente de México no aparece en el contexto de cobros de Perú y viceversa.

Regla 6 — Cierre automático del pendiente del Buzón al vincular el cobro a proforma o factura
Cuando un cobro en Validar Cobro se vincula exitosamente a una proforma o factura, el sistema cierra y retira automáticamente el pendiente correspondiente del Buzón de Cobros. El correo deja de aparecer en la bandeja del Gestor; el ciclo de vida del pendiente del Buzón se considera completado.

Regla 7 — Eliminación automática del Buzón ante inconsistencia en Validar Cobro
Cuando un cobro en Validar Cobro se marca como inconsistencia, el sistema elimina automáticamente la entrada correspondiente del Buzón de Cobros. El tratamiento posterior de la inconsistencia (incluida cualquier notificación al cliente) pertenece al módulo Validar Cobro.

Regla 8 — Reclasificación manual hacia otro buzón
Cuando un Gestor de Cobranza identifica que un correo fue clasificado incorrectamente como Cobro, puede ejecutar la acción de reclasificarlo moviéndolo a otro buzón del sistema, incluido el buzón de Otros. No existe la opción ""marcar como no-cobro""; cualquier corrección de clasificación se realiza moviendo el correo a un buzón destino.

Regla 9 — Sin eliminación directa por el Gestor; eliminación reservada a ESAC desde Otros
El Gestor de Cobranza no dispone de una acción de eliminación directa del correo en el Buzón de Cobros. Si un correo no corresponde a un cobro, la salida es reclasificarlo al buzón de Otros; la eliminación del correo, de requerirse, la realiza el rol ESAC desde el buzón de Otros.

Regla 10 — Filtros, búsqueda y paginación equivalentes a Buzones preexistentes
El Buzón de Cobros ofrece los mismos mecanismos de filtros, búsqueda y paginación que los Buzones preexistentes del sistema, conservando consistencia de experiencia de usuario.

Regla 11 — Bandeja del Coordinador: correo de cliente existente sin Cobrador (Caso 1)
Cuando un correo de cobro proviene de un remitente dado de alta como contacto de un cliente existente, pero ese cliente no tiene Cobrador asignado, el correo se concentra en la bandeja del Coordinador de Tesorería dentro del Buzón de Cobros. Desde esa bandeja, el Coordinador de Tesorería puede asignar un Cobrador al cliente; al hacerlo, el correo desaparece de la bandeja del Coordinador y aparece en la bandeja del Cobrador asignado. La asignación también puede realizarse desde el Catálogo de Clientes, con el mismo efecto.

Regla 12 — Bandeja del Coordinador: correo de remitente no dado de alta (Caso 2)
Cuando un correo de cobro proviene de un remitente que no está dado de alta como contacto de ningún cliente existente, el correo se concentra en la bandeja del Coordinador de Tesorería. Este caso no se resuelve desde el Buzón: el flujo operativo es el existente, dar de alta el contacto (con ese correo) en el cliente correspondiente. Una vez dado de alta el contacto, si el cliente ya tiene Cobrador asignado el correo pasa a la bandeja de ese Cobrador; si el cliente no tiene Cobrador asignado, el correo permanece en la bandeja del Coordinador y se resuelve conforme a la Regla 11.

## Riesgos


## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CLASIFICACIÓN Y RECEPCIÓN
═══════════════════════════════════════════════════════════════

Criterio A1 — Recepción y clasificación automática de correo entrante
Dado que un correo entrante llega al sistema,
Cuando el Mailbot lo evalúa,
Entonces deberá clasificarlo en una de las categorías del modelo (Cotización, Pedido, Cobro u Otros) y, si lo clasifica como Cobro, enviarlo al Buzón de Cobros.

Criterio A2 — Clasificación sin criterios configurables por el usuario
Dado que se requiere ajustar la precisión de la clasificación del Mailbot,
Cuando se necesita modificar su comportamiento,
Entonces el ajuste deberá realizarse mediante un nuevo entrenamiento del Mailbot con su base de conocimiento, sin que existan criterios configurables por el usuario en la interfaz del Buzón.

═══════════════════════════════════════════════════════════════
SECCIÓN B — REFLEJO EN EL BUZÓN Y GENERACIÓN DE PENDIENTE
═══════════════════════════════════════════════════════════════

Criterio B1 — Reflejo del correo en el Buzón del Gestor asignado
Dado que un correo es clasificado como Cobro y el cliente identificado en el correo tiene un Gestor de Cobranza asignado en el Catálogo de Clientes,
Cuando el sistema procesa el reflejo en el Buzón,
Entonces deberá hacer visible el correo únicamente en la bandeja del Buzón del Gestor asignado. Otros Gestores no ven el correo en su bandeja.

Criterio B2 — Generación automática de pendiente en Validar Cobro
Dado que un correo es clasificado como Cobro,
Cuando el sistema procesa la clasificación,
Entonces deberá generar automáticamente un pendiente en el módulo Validar Cobro asociado al correo, sin pre-capturar datos del cobro. El pendiente queda disponible para que el Gestor lo trabaje en Validar Cobro.

Criterio B3 — Visibilidad filtrada por cobrador asignado
Dado que un Gestor de Cobranza accede al Buzón,
Cuando el sistema arma la bandeja,
Entonces deberá filtrar los correos para mostrar únicamente aquellos correspondientes a clientes que el Gestor tiene asignados en el Catálogo de Clientes (campo Cobrador).

═══════════════════════════════════════════════════════════════
SECCIÓN C — CICLO DE VIDA DEL PENDIENTE DEL BUZÓN
═══════════════════════════════════════════════════════════════

Criterio C1 — Cierre automático del pendiente del Buzón al vincular el cobro a proforma o factura
Dado que el Gestor de Cobranza vincula exitosamente el cobro a una proforma o factura desde el módulo Validar Cobro,
Cuando se completa la vinculación,
Entonces el sistema deberá retirar automáticamente el pendiente correspondiente del Buzón de Cobros, sin requerir acción manual adicional del Gestor. El correo deja de aparecer en su bandeja.

Criterio C2 — Eliminación automática del Buzón ante inconsistencia en Validar Cobro
Dado que un cobro en Validar Cobro se marca como inconsistencia y el correo origen estaba en el Buzón,
Cuando se procesa la inconsistencia en Validar Cobro,
Entonces el sistema deberá eliminar automáticamente la entrada correspondiente del Buzón de Cobros. El tratamiento posterior de la inconsistencia, incluida cualquier notificación al cliente, pertenece al módulo Validar Cobro.

═══════════════════════════════════════════════════════════════
SECCIÓN D — ACCIONES DEL GESTOR Y EXPERIENCIA DE USUARIO
═══════════════════════════════════════════════════════════════

Criterio D1 — Acción de reclasificación manual
Dado que un Gestor de Cobranza identifica un correo del Buzón que fue clasificado incorrectamente,
Cuando ejecuta la acción de reclasificar,
Entonces el sistema deberá ofrecer la opción de mover el correo a otro buzón del sistema (incluido el buzón de Otros). La acción se completa al elegir el buzón destino: el correo se retira del Buzón de Cobros y se refleja en el buzón seleccionado.

Criterio D2 — Sin eliminación directa por el Gestor; eliminación reservada a ESAC desde Otros
Dado que un Gestor de Cobranza opera el Buzón de Cobros,
Cuando consulta las acciones disponibles sobre un correo,
Entonces el sistema no deberá ofrecer una acción de eliminación individual y/o multiple del correo. Si el correo no corresponde a un cobro u otra clasificación, la salida es reclasificarlo al buzón de Otros; la eliminación del correo, de requerirse, la realiza el rol ESAC desde el buzón de Otros.

Criterio D3 — Filtros, búsqueda y paginación en el Buzón
Dado que un Gestor de Cobranza opera el Buzón,
Cuando consulta su bandeja,
Entonces el módulo deberá ofrecer las funcionalidades de filtros, búsqueda y paginación equivalentes al patrón aplicado en los Buzones preexistentes del sistema (Buzón de Requisiciones y Buzón de Pedidos).

═══════════════════════════════════════════════════════════════
SECCIÓN E — APLICABILIDAD REGIONAL
═══════════════════════════════════════════════════════════════

Criterio E1 — Aplicación a clientes México y Perú con segregación por región
Dado que el cliente del correo identificado tiene Región México o Región Perú,
Cuando el sistema procesa la clasificación y refleja el correo en el Buzón,
Entonces deberá operar con la misma mecánica para ambas regiones, respetando siempre la región del cliente: el correo de un cliente de México se refleja en el contexto de cobros de México y el de un cliente de Perú en el contexto de cobros de Perú. Un cliente de una región nunca aparece en el contexto de cobros de la otra; no hay cruce de correos entre regiones.

═══════════════════════════════════════
SECCIÓN F — BANDEJA DEL COORDINADOR
═══════════════════════════════════════

Criterio F1 — Concentración de correos no enrutables en la bandeja del Coordinador
Dado que un correo de cobro no puede enrutarse a un Gestor (cliente existente sin Cobrador asignado, o remitente no dado de alta como contacto),
Cuando el sistema procesa el correo,
Entonces deberá mostrarlo en la bandeja del Coordinador de Tesorería dentro del Buzón de Cobros, en lugar de dejarlo invisible.

Criterio F2 — Asignación de Cobrador desde la bandeja del Coordinador (Caso 1)
Dado que un correo de la bandeja del Coordinador corresponde a un cliente existente sin Cobrador asignado,
Cuando el Coordinador de Tesorería asigna un Cobrador (desde la bandeja o desde el Catálogo de Clientes),
Entonces el sistema deberá retirar el correo de la bandeja del Coordinador y reflejar en la bandeja del Cobrador asignado no solo ese correo, sino todos los correos y pendientes del cliente que se hayan generado previamente mientras no tenía Cobrador asignado (retroactividad completa).

Criterio F3 — Correo de remitente no dado de alta (Caso 2)
Dado que un correo de la bandeja del Coordinador proviene de un remitente no dado de alta como contacto de ningún cliente,
Cuando el usuario da de alta el contacto en el cliente correspondiente por el flujo operativo existente,
Entonces el sistema deberá enrutar el correo a la bandeja del Cobrador asignado si el cliente ya tiene Cobrador, o mantenerlo en la bandeja del Coordinador para asignación si el cliente aún no tiene Cobrador.

Criterio F4 — Visibilidad retroactiva de pendientes al asignar Cobrador
Dado que a un cliente que no tenía Cobrador asignado se le asigna uno (desde la bandeja del Coordinador o desde el Catálogo de Clientes),
Cuando se completa la asignación,
Entonces la bandeja del Cobrador asignado deberá mostrar todos los correos pendientes de ese cliente generados antes de la asignación, sin que ninguno quede excluido por haberse generado previamente."								"- Módulo en PQF2. El módulo se llama ""Buzón de Cobros"". El nombre refleja la perspectiva operativa: son cobros que PROQUIFA registra hacia el cliente, aunque el correo original sea el comprobante del pago que el cliente envía a PROQUIFA. Toda la terminología del módulo, su contenido y sus mensajes utiliza el término ""cobro"".
- Cubre el requisito del cliente sobre la integración del Buzón con Validar Cobro y la mecánica del ciclo de vida del correo en el Buzón.
- El Buzón de Cobros y el módulo Validar Cobro NO son el mismo módulo: el mismo correo es visible en ambos pero con funciones distintas. En el Buzón de Cobros únicamente se ve el correo clasificado, se opera la reclasificación si la clasificación automática fue incorrecta, y se observa cuándo el pendiente se cierra o elimina automáticamente. En Validar Cobro se capturan los datos del cobro, se vincula a proformas/facturas y se marca como inconsistencia si aplica.
- El ciclo de vida del pendiente del correo en el Buzón es estrictamente automático: el pendiente aparece al clasificarse el correo, se cierra automáticamente al completarse la vinculación a proforma/factura en Validar Cobro, o se elimina automáticamente al marcarse inconsistencia en Validar Cobro. No existe eliminación directa del correo por parte del Gestor; la única acción manual disponible en el Buzón es la reclasificación hacia otro buzón.
- La clasificación automática la realiza el Mailbot del sistema, que en R16 agrega la categoría Cobro a su modelo de clasificación existente (Cotización, Pedido, Otros). El Mailbot no usa criterios configurables por el usuario: se entrena con una base de conocimiento y los ajustes a la precisión de la clasificación se realizan mediante un nuevo entrenamiento. El entrenamiento inicial requiere un conjunto representativo de correos reales de PROQUIFA en producción para ajustar la tasa de acierto.
- La visibilidad de los correos en el Buzón está filtrada por el cobrador asignado al cliente (campo Cobrador del Catálogo de Clientes). Cada Gestor de Cobranza ve únicamente los correos correspondientes a su cartera.
- Los filtros, búsqueda y paginación del Buzón de Cobros aplican el mismo patrón que los Buzones preexistentes del sistema (Buzón de Requisiciones, Buzón de Pedidos), conservando consistencia de UX.
- La reclasificación manual del Gestor de Cobranza se realiza moviendo el correo a otro buzón del sistema, incluido el buzón de Otros. No existe la opción ""marcar como no-cobro""; cualquier corrección de clasificación incorrecta requiere mover el correo a un buzón destino específico.
- Manejo de la eliminación: a diferencia de los Buzones preexistentes que permiten eliminar pendientes, el Buzón de Cobros no ofrece eliminación directa al Gestor de Cobranza. La salida de un correo mal clasificado es reclasificarlo al buzón de Otros; desde Otros, el rol ESAC puede eliminarlo si corresponde.
- Cuando un cobro en Validar Cobro se marca como inconsistencia, el pendiente correspondiente del Buzón se elimina automáticamente. El tratamiento de la inconsistencia, incluido el envío de cualquier notificación al cliente, pertenece al módulo Validar Cobro y se documenta en su requisito, no en este.
- ** Pendiente: lógica para correos clasificados como cobro cuyo cliente no es identificable o no está registrado (correo de dominio genérico, referencia errónea, cliente nuevo no registrado). El equipo plantea: si el Mailbot clasifica como Cobro un correo de un cliente no registrado y el sistema lo asignaría a ""cliente nuevo"", ¿quién lo visualiza y atiende? Pendiente de confirmar con el cliente cómo se atienden estos correos y quién es responsable de ellos. Mientras no se resuelva, queda como duda formal del proyecto. **
- ** Propuesta complementaria abierta para evaluación con el cliente: incorporar IA al clasificador para que lea el documento adjunto del correo (comprobante de pago en PDF o imagen) y precargue automáticamente los datos en Validar Cobro (monto, banco emisor, cuenta origen, fecha del depósito, referencia bancaria, etc.). Esto reduciría el trabajo manual del Gestor de Cobranza al validar el cobro y aumentaría la eficiencia operativa del flujo completo. La propuesta queda como evolución futura del módulo, sujeta a validación de viabilidad y costo con el cliente. **
- Aplicable a clientes México y Perú con la misma mecánica funcional, respetando siempre la región del cliente: los correos se segregan por región, de modo que un cliente de México nunca aparece en el contexto de cobros de Perú ni viceversa.
- ** Pendiente confirmar con el cliente, para los correos de la bandeja del Coordinador (Caso 1): (1) si se permite reclasificar el correo a otro tipo de pendiente desde esa bandeja, y (2) en caso afirmativo, a qué destinos se permite reclasificar (la hipótesis operativa es que, de permitirse, sea únicamente a ""Otros""). **"									
TPSC-RE-FU-012		Tramitación de pedidos Crédito	Tramitar Pedido	Yo como ESAC, quiero tramitar pedidos de clientes con condición de pago Crédito sin sustancias controladas activando la opción de Factura por Adelantado, para que el cliente reciba la factura PPD timbrada antes de la entrega y pueda justificar fiscalmente el gasto, mientras el pedido continúa su flujo regular de tramitación sin bloqueos.	El sistema debe permitir, en el módulo Tramitar Pedido, la activación opcional de Factura por Adelantado para pedidos de clientes con condición de pago Crédito (incluyendo la variante Pago contra entrega) cuando el pedido no contiene sustancias controladas. Al tramitar el pedido con esta opción activa, el sistema genera un pendiente en el módulo Factura por Adelantado para que Finanzas gestione la emisión de la factura posteriormente. La tramitación del pedido continúa el flujo crédito regular hasta su Confirmación y transferencia a Legacy, de forma independiente al ciclo de emisión de la factura. Al activar la opción, los datos de facturación quedan bloqueados y se toman del catálogo del cliente vigente.			Propuesto					R16.1M-RE-FU-002, R16.1M-RE-FU-003, R16.1M-RE-FU-005, R16.1M-RE-FU-010, R16.1M-RE-FU-011, R16.1M-RE-FU-013, R16.1M-RE-FU-014	"## Aplica a
- Activación de Factura por Adelantado: pedidos de clientes con condición de pago Crédito en la operación de México exclusivamente. En Perú la Factura por Adelantado NO está disponible para pedidos Crédito, porque el timbrado fiscal en Perú aplica únicamente a pedidos Prepago en R16; un Crédito peruano no podría emitir la factura, por lo que la opción no se ofrece.
- Pedidos con condición Crédito - Pago contra entrega.
- Pedidos sin sustancias controladas (Mundial, Nacional, Origen).
- Activación opcional de la opción Factura por Adelantado desde el módulo Tramitar Pedido como punto de entrada único al flujo Factura por Adelantado.
- Generación de un pendiente en el módulo Factura por Adelantado al tramitar (la emisión y timbrado de la factura PPD ocurre posteriormente dentro de ese módulo, NO en Tramitar Pedido).
- Bloqueo de edición de los datos de facturación del pedido cuando se activa Factura por Adelantado.
- Operación en Perú: la tramitación del pedido Crédito (SIN Factura por Adelantado) opera idéntico a México pero NO transfiere a Legacy al concluir; la operación termina en la confirmación interna en PQF2. La opción Factura por Adelantado NO se ofrece para Crédito en Perú (ver punto anterior).

## No aplica a
- Pedidos de clientes con condición de pago Prepago (esos siguen un flujo distinto descrito en los requisitos del bloque Prepago).
- Pedidos con sustancias controladas (la combinación Factura por Adelantado + sustancias controladas no es permitida por regla regulatoria).
- La gestión del pendiente ""Relacionar facturas"" dentro de Legacy (responsabilidad de Legacy, no de PQF2).
- El conteo de los días de crédito del cliente (inicia con la emisión efectiva de la factura PPD, evento que ocurre fuera de Tramitar Pedido).
- Activación de Factura por Adelantado para pedidos Crédito de la región Perú: el timbrado fiscal peruano en R16 está limitado a Prepago, por lo que la factura anticipada de un Crédito peruano no podría emitirse."	"## Reglas de Negocio

Regla 1 — Factura por Adelantado opcional para Crédito
Para pedidos de clientes con condición de pago Crédito sin sustancias controladas, el módulo Tramitar Pedido ofrece la opción de activar Factura por Adelantado como mecanismo opcional. Si no se activa, el pedido sigue el flujo crédito regular sin cambios; si se activa, dispara la generación del pendiente correspondiente en el módulo Factura por Adelantado al tramitar.

Regla 2 — Tramitar Pedido como punto de entrada único al flujo Factura por Adelantado
La activación de Factura por Adelantado se realiza exclusivamente desde el módulo Tramitar Pedido, no desde Pretramitar Pedido ni otros módulos.

Regla 3 — Activación de Factura por Adelantado sin código de autorización
La activación de Factura por Adelantado en Tramitar Pedido es directa, sin requerir código de autorización ni validación adicional.

Regla 4 — Datos de facturación bloqueados cuando se activa Factura por Adelantado
Al activar Factura por Adelantado para un pedido Crédito, el sistema no permite editar los datos de facturación del cliente (RFC/RUC, razón social, y los campos fiscales según región: Uso CFDI y Método de Pago en México, Tipo de Operación y Condición de Pago en Perú) desde Tramitar Pedido. Los datos de facturación quedan fijados con los valores del catálogo del cliente vigente al momento de la activación; cualquier ajuste posterior se gestiona en el módulo Factura por Adelantado o en el Catálogo de Clientes según corresponda.

Regla 5 — Generación del pendiente Factura por Adelantado al tramitar
Al activar Factura por Adelantado y ejecutar la acción de tramitar, el sistema genera un pendiente en el módulo Factura por Adelantado con la información necesaria para que el rol correspondiente gestione posteriormente la emisión y timbrado de la factura PPD. La tramitación del pedido no espera a la emisión de la factura. ** Pendiente confirmar con el cliente qué rol gestiona la emisión y timbrado de la factura PPD (Finanzas o Coordinador de Planeación y Control). **

Regla 6 — Factura por Adelantado no bloquea la Confirmación de Pedido
La gestión del pendiente Factura por Adelantado es un proceso independiente. La Confirmación de Pedido se genera de inmediato y el pedido continúa su flujo crédito regular en paralelo a la futura emisión de la factura PPD.

Regla 7 — Operación Perú sin transferencia a Legacy
Para pedidos de clientes Crédito de la región Perú, al concluir el flujo de tramitación en PQF2 el sistema no envía el pedido al sistema Legacy. La operación termina con la confirmación interna en PQF2.

Regla 8 — Cierre del pendiente de Tramitar Pedido al completar la acción
Una vez ejecutada exitosamente la acción de tramitar, completado el envío del correo correspondiente al flujo y generados los pendientes derivados (si aplica), el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

Regla 9 — Composición regionalizada del panel de Información de Facturación
El panel de Información de Facturación de Tramitar Pedido es transversal a ambas regiones y muestra los datos del cliente tomados del catálogo, en modo solo lectura. Los campos comunes a México y Perú son: Razón Social, identificador fiscal (RFC para México / RUC para Perú), Moneda, Quién Factura (empresa emisora), Condiciones de Pago (plazo comercial; ej. ""60 Días"", ""Prepago 100%"") y Comentarios para la Facturación. Los campos fiscales se regionalizan según la Región del cliente: para México se muestran Uso CFDI y Método de Pago (catálogos SAT); para Perú estos se reemplazan por Tipo de Operación (catálogo 51 SUNAT, en lugar de Uso CFDI) y Condición de Pago SUNAT en singular, Contado/Crédito (en lugar de Método de Pago). La ""Condición de Pago"" SUNAT de Perú es un campo fiscal distinto de las ""Condiciones de Pago"" comerciales (plazo) y ambos coexisten en el panel para clientes Perú. Los campos Forma de Pago (medio) y correo de envío no se muestran en este panel en ninguna región.

## Riesgos

Riesgo 1 — Catálogo del cliente desactualizado al activar Factura por Adelantado
Como los datos de facturación se fijan al activar Factura por Adelantado sin opción de editar en Tramitar Pedido, si el catálogo del cliente tiene datos fiscales desactualizados (RFC incorrecto, correo viejo, Uso CFDI no aplicable a la operación), la factura PPD se emitirá con esos datos y requerirá cancelación fiscal y reemisión posterior.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — ACTIVACIÓN Y TRAMITACIÓN CON FACTURA POR ADELANTADO
═══════════════════════════════════════════════════════════════

Criterio A1 — Tramitación con Factura por Adelantado activada para Crédito México sin controlados
Dado que un pedido pertenece a un cliente Crédito en México, sin productos controlados, y el ESAC activa la opción Factura por Adelantado en el módulo Tramitar Pedido (en Región Perú la Factura por Adelantado no está disponible para pedidos Crédito, ver Regla y Criterio C6),
Cuando se ejecuta la acción de tramitar,
Entonces el sistema deberá tramitar el pedido siguiendo el flujo crédito regular, generar la Confirmación de Pedido al cliente y, en paralelo, generar el pendiente correspondiente en el módulo Factura por Adelantado para que el rol responsable emita y timbre la factura PPD.

Criterio A2 — Activación de Factura por Adelantado desde Tramitar Pedido
Dado que un pedido pertenece a un cliente Crédito sin productos controlados,
Cuando el ESAC visualiza el módulo Tramitar Pedido,
Entonces el sistema deberá ofrecer la opción de activar Factura por Adelantado. La activación es directa y no requiere código de autorización.

Criterio A3 — Variante Pago contra entrega con Factura por Adelantado
Dado que un pedido pertenece a un cliente con condición Crédito - Pago contra entrega sin controlados y el ESAC activa Factura por Adelantado,
Cuando el ESAC opera el módulo Tramitar Pedido,
Entonces el sistema deberá tramitarlo aplicando el mismo flujo de un Crédito normal con Factura por Adelantado. La detención por falta de validación de pago la ejecuta Legacy.

Criterio A4 — Composición regionalizada del panel de Información de Facturación
Dado que el ESAC visualiza el panel de Información de Facturación de un pedido en Tramitar Pedido,
Cuando el sistema renderiza el panel según la Región del cliente,
Entonces para clientes México deberá mostrar Uso CFDI y Método de Pago (catálogos SAT); para clientes Perú deberá mostrar Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago Contado/Crédito SUNAT en su lugar; en ambas regiones deberá mostrar los campos comunes (Razón Social, RFC/RUC, Moneda, Quién Factura, Condiciones de Pago comerciales y Comentarios) y NO deberá mostrar Forma de Pago ni correo de envío.

═══════════════════════════════════════════════════════════════
SECCIÓN B — BLOQUEO DE DATOS Y GENERACIÓN DEL PENDIENTE
═══════════════════════════════════════════════════════════════

Criterio B1 — Bloqueo de edición de datos de facturación al activar Factura por Adelantado
Dado que el ESAC activó la opción Factura por Adelantado en Tramitar Pedido,
Cuando se renderiza la pantalla del pedido,
Entonces el botón ""Editar Datos"" para datos de facturación no debe aparecer disponible para este pedido. El sistema deberá mostrar los datos de facturación en modo solo lectura tomados del catálogo del cliente vigente al momento de la activación.

Criterio B2 — Generación del pendiente en el módulo Factura por Adelantado
Dado que el ESAC tramitó exitosamente un pedido Crédito con la opción Factura por Adelantado activada,
Cuando se completa la tramitación del pedido,
Entonces el sistema deberá generar automáticamente un pendiente en el módulo Factura por Adelantado, asociado al pedido tramitado, para que el rol responsable ejecute posteriormente la emisión y timbrado de la factura PPD. ** Pendiente confirmar con el cliente qué rol gestiona la emisión y timbrado de la factura PPD (Finanzas o Coordinador de Planeación y Control). **

═══════════════════════════════════════════════════════════════
SECCIÓN C — CONFIRMACIÓN, FEE, CANCELACIÓN Y TRANSFERENCIA
═══════════════════════════════════════════════════════════════

Criterio C1 — Tramitación con Factura por Adelantado activada para Crédito sin controlados (México)
Dado que un pedido pertenece a un cliente Crédito de México, sin productos controlados, y el ESAC activa la opción Factura por Adelantado en el módulo Tramitar Pedido,
Cuando se ejecuta la acción de tramitar,
Entonces el sistema deberá generar la Confirmación de Pedido en formato PDF y permitir su envío al cliente, sin esperar a que se gestione el pendiente de Factura por Adelantado.

Criterio C2 — Cálculo de FEE al tramitar
Dado que el pedido se tramita exitosamente en Tramitar Pedido,
Cuando se genera la Confirmación de Pedido,
Entonces el sistema deberá calcular automáticamente la FEE correspondiente al pedido conforme a las reglas vigentes en el sistema.

Criterio C3 — Cancelación del pedido
Dado que un pedido tramitado tiene solicitud del cliente para cancelar,
Cuando el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
Entonces el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

Criterio C4 — Variante Pago contra entrega: transferencia a Legacy con marca de detención (México)
Dado que un pedido Crédito - Pago contra entrega con Factura por Adelantado se tramitó en PQF2 para un cliente de México,
Cuando el sistema transfiere el pedido al sistema Legacy,
Entonces deberá incluir en la transferencia la marca de detención que indica a Legacy que el pedido no debe entregarse hasta validar el pago.

Criterio C5 — Transferencia a Legacy de pedido tramitado (variante México)
Dado que un pedido Crédito con Factura por Adelantado se ha tramitado exitosamente en la operación de México,
Cuando se completa la Confirmación de Pedido,
Entonces el sistema deberá transferir automáticamente a Legacy toda la información necesaria del pedido para que el sistema legado continúe el ciclo de surtido, despacho y entrega.

Criterio C6 — Operación Perú sin transferencia a Legacy y sin Factura por Adelantado
Dado que un pedido Crédito se ha tramitado exitosamente para un cliente de la región Perú,
Cuando se completa la Confirmación de Pedido,
Entonces el sistema NO deberá ejecutar la transferencia a Legacy (la operación queda registrada únicamente en PQF2) y NO deberá ofrecer la opción Factura por Adelantado para el pedido, ya que el timbrado peruano en R16 aplica solo a Prepago."								"- Variante del flujo crédito preexistente del módulo Tramitar Pedido en PQF2 con la activación opcional del flujo Factura por Adelantado. El punto de entrada al flujo Factura por Adelantado se ubica exclusivamente en Tramitar Pedido (confirmado por el cliente; descartada la activación desde Pretramitar Pedido).
- Cubre tres requisitos del cliente: activación de Factura por Adelantado para clientes crédito generando pendiente y continuidad de flujo; tramitación bajo condición Crédito - Pago contra entrega apegada al flujo crédito existente; y transferencia a Legacy con marca de detención para Pago contra entrega.
- Al tramitar con Factura por Adelantado activada, el sistema NO emite la factura PPD en ese momento. Lo que hace es generar un pendiente en el módulo Factura por Adelantado para que el rol responsable gestione posteriormente la emisión y timbrado. La Confirmación del pedido se genera de inmediato sin esperar a la factura.
- Cambio respecto al comportamiento actual: se elimina el código de autorización para activar Factura por Adelantado (antes lo requería). La activación ahora es directa.
- Cambio respecto al comportamiento actual: cuando se activa Factura por Adelantado, los datos de facturación quedan fijados al momento de la activación y el botón ""Editar Datos"" se oculta para este pedido. Los datos de facturación se toman directamente del catálogo del cliente. Modificaciones posteriores requieren actualizar el catálogo o gestionar el ajuste en el módulo Factura por Adelantado. El botón ""Editar Datos"" sigue disponible para pedidos crédito que no activen Factura por Adelantado.
- Una vez emitida la factura por adelantado en el módulo Factura por Adelantado, el sistema debe generar un pendiente en el módulo ""Relacionar facturas"" del sistema Legacy. El mecanismo de comunicación PQF2 - Legacy para este pendiente queda pendiente de definir y NO es parte del scope de este requisito (corresponde al módulo Factura por Adelantado).
- A diferencia del flujo Prepago, en Crédito la Confirmación de Pedido se genera dentro del módulo Tramitar Pedido.
- La detención del pedido Pago contra entrega por falta de validación de pago es responsabilidad de Legacy.
- La tramitación del pedido Crédito es aplicable a México y Perú; en Perú no se transfiere a Legacy al concluir. La opción Factura por Adelantado, en cambio, solo aplica a Crédito de México: en Perú el timbrado fiscal en R16 se limita a Prepago, por lo que un Crédito peruano no puede emitir factura anticipada."									
TPSC-RE-FU-015		Tramitación de pedidos Prepago	Tramitar Pedido	"Yo como ESAC, quiero tramitar pedidos de clientes con condición de pago Prepago sin sustancias controladas activando la opción de Factura por Adelantado, para que el cliente reciba la proforma correspondiente para iniciar el cobro y, en paralelo, Finanzas gestione la emisión y timbrado de la factura necesaria para el flujo prepago anticipado.
"	El sistema debe permitir, en el módulo Tramitar Pedido, la tramitación de pedidos de clientes con condición de pago Prepago que no contienen sustancias controladas y que activan la opción Factura por Adelantado, en las operaciones de México y Perú. Para clientes prepago los datos de facturación no son editables en este módulo y se toman del catálogo del cliente vigente. Al tramitar, el sistema genera la proforma, la presenta al ESAC para validación visual y, tras confirmar el envío al cliente, genera el pendiente en el módulo Factura por Adelantado para que Finanzas gestione la emisión de la factura y cierra el pendiente del pedido en la bandeja de Tramitar Pedido. El pendiente en Validar Cobro se genera posteriormente, cuando la factura se emita.			Propuesto					R16.1M-RE-FU-001, R16.1M-RE-FU-002, R16.1M-RE-FU-003, R16.1M-RE-FU-004, R16.1M-RE-FU-007, R16.1M-RE-FU-015	"## Aplica a
- Pedidos de clientes con condición de pago Prepago en las operaciones de México y Perú.
- Pedidos sin sustancias controladas (mundial, nacional, origen).
- Pedidos en los que el ESAC activa la opción Factura por Adelantado durante la tramitación.
- Activación directa de la opción Factura por Adelantado sin código de autorización.
- Generación de la proforma al ejecutar la acción Tramitar en el módulo Tramitar Pedido.
- Asignación del folio interno del pedido conforme a la mecánica actual del sistema.
- Asignación del folio de la proforma desde el foliador lineal global de PQF2 (un solo contador para todas las proformas del sistema, sin segmentación por empresa o región).
- Flujo de envío con dos pasos: previsualización del PDF de la proforma + pantalla de datos de envío del correo.
- Generación automática del pendiente en el módulo Factura por Adelantado al confirmar el envío del correo.
- Cierre automático del pendiente del pedido en la bandeja del módulo Tramitar Pedido al completar la tramitación.
- Visualización en solo lectura de los datos de facturación del cliente (tomados del catálogo).
- Operación en Perú: el flujo opera idéntico al de México durante la tramitación en este módulo. Las diferencias para Perú se materializan posteriormente fuera de TP (no transfiere a Legacy tras la validación de cobro).

## No aplica a
- Pedidos de clientes con condición de pago Crédito (esos siguen los flujos descritos en los requisitos del bloque Crédito).
- Pedidos prepago con sustancias controladas (la combinación Factura por Adelantado + sustancias controladas no es permitida por regla regulatoria).
- Pedidos prepago sin activación de Factura por Adelantado (variante cubierta en requisito independiente del bloque Prepago).
- Renderizado del radio button de Entrega con Remisión, que no se muestra en el módulo Tramitar Pedido para clientes prepago en ninguna variante.
- Edición de los datos de facturación del cliente desde el módulo Tramitar Pedido para clientes prepago (los ajustes se gestionan en el Catálogo de Clientes).
- La generación del pendiente en Validar Cobro al tramitar (en este flujo el pendiente VC se generará posteriormente, al emitirse la factura en el módulo Factura por Adelantado).
- La emisión propiamente dicha de la factura ni la mecánica interna del módulo Factura por Adelantado.
- La validación del cobro de la factura, el timbrado fiscal, el cálculo de la FEE, la generación de la Confirmación de Pedido y la transferencia a Legacy. Todas esas acciones ocurren en módulos posteriores y se cubren en requisitos independientes."	"## Reglas de Negocio

Regla 1 — Renderizado de Factura por Adelantado para Prepago sin controlados
Para pedidos de clientes Prepago sin sustancias controladas, el sistema renderiza el radio button de Factura por Adelantado como opción disponible para que el ESAC decida activarla o no.

Regla 2 — No renderizado de Entrega con Remisión para Prepago
Para pedidos de clientes Prepago (con o sin sustancias controladas), el sistema no renderiza el radio button de Entrega con Remisión. Esta opción no aplica para clientes prepago en ninguna variante.

Regla 3 — Activación de Factura por Adelantado sin código de autorización
La activación de Factura por Adelantado en Tramitar Pedido es directa, sin requerir código de autorización ni validación adicional.

Regla 4 — Datos de facturación bloqueados cuando se activa Factura por Adelantado
Al activar Factura por Adelantado para un pedido Prepago, el sistema no permite editar los datos de facturación del cliente (RFC/RUC, razón social, y los campos fiscales según región: Uso CFDI y Método de Pago en México, Tipo de Operación y Condición de Pago en Perú) desde Tramitar Pedido. Los datos de facturación quedan fijados con los valores del catálogo del cliente vigente al momento de la activación; cualquier ajuste posterior se gestiona en el módulo Factura por Adelantado o en el Catálogo de Clientes según corresponda.

Regla 5 — Generación automática de proforma al tramitar
Al ejecutar la acción de tramitar un pedido prepago sin sustancias controladas con Factura por Adelantado activada, el sistema genera automáticamente una proforma en formato PDF.

Regla 6 — Folio de la proforma desde foliador lineal global
El folio de la proforma se toma del foliador lineal global de PQF2, que mantiene un solo contador para todas las proformas del sistema sin segmentación por empresa o región.

Regla 7 — Folio del pedido interno conforme a mecánica actual
El folio interno del pedido se asigna siguiendo la mecánica actual del sistema, sin cambios respecto a la versión vigente.

Regla 8 — Previsualización del PDF de la proforma obligatoria
Antes de avanzar a la pantalla de datos de envío, el sistema muestra la previsualización del PDF de la proforma. El ESAC debe aceptarla explícitamente para continuar.

Regla 9 — Pantalla de datos de envío del correo de proforma
La pantalla de datos de envío del correo de proforma presenta: Para (destinatario) con el contacto del cliente del pedido, editable, con default heredado del catálogo del cliente; CC con el ESAC asignado al cliente/pedido, editable, con default sugerido por el sistema; Asunto generado automáticamente según plantilla, no editable; Adjuntos con el PDF de la proforma, no editables; y Notas extras, un campo de texto editable opcional para texto adicional libre.

Regla 10 — Composición del asunto del correo de proforma
El asunto del correo de proforma se compone como la cadena ""Proforma"" seguida del folio del pedido interno.

Regla 11 — Generación automática del pendiente Factura por Adelantado
Al confirmarse el envío exitoso del correo de la proforma para un pedido con Factura por Adelantado activada, el sistema genera automáticamente un pendiente en el módulo Factura por Adelantado asociado al folio del pedido, para que se gestione posteriormente la emisión y timbrado de la factura.

Regla 12 — No generación del pendiente Validar Cobro al tramitar
Al tramitar un pedido prepago con Factura por Adelantado activada, el sistema no genera el pendiente en el módulo Validar Cobro en ese momento. El pendiente Validar Cobro se generará posteriormente, cuando la factura se emita exitosamente en el módulo Factura por Adelantado.

Regla 13 — Cierre del pendiente de Tramitar Pedido al completar la acción
Una vez completados el envío del correo de la proforma y la generación del pendiente en Factura por Adelantado, el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

Regla 14 — Composición regionalizada del panel de Información de Facturación
El panel de Información de Facturación de Tramitar Pedido es transversal a ambas regiones y muestra los datos del cliente tomados del catálogo, en modo solo lectura. Los campos comunes a México y Perú son: Razón Social, identificador fiscal (RFC para México / RUC para Perú), Moneda, Quién Factura (empresa emisora), Condiciones de Pago (plazo comercial; ej. ""60 Días"", ""Prepago 100%"") y Comentarios para la Facturación. Los campos fiscales se regionalizan según la Región del cliente: para México se muestran Uso CFDI y Método de Pago (catálogos SAT); para Perú estos se reemplazan por Tipo de Operación (catálogo 51 SUNAT, en lugar de Uso CFDI) y Condición de Pago SUNAT en singular, Contado/Crédito (en lugar de Método de Pago). La ""Condición de Pago"" SUNAT de Perú es un campo fiscal distinto de las ""Condiciones de Pago"" comerciales (plazo) y ambos coexisten en el panel para clientes Perú. Los campos Forma de Pago (medio) y correo de envío no se muestran en este panel en ninguna región.

## Riesgos

Riesgo 1 — Campos de información fiscal originalmente configurados para México
Los campos de información fiscal del módulo Tramitar Pedido están actualmente configurados conforme a las normas fiscales de México. Al operar pedidos peruanos, el ESAC podría experimentar confusión sobre qué campos aplican o cómo interpretarlos en el contexto fiscal peruano. Se espera capacitación al equipo operativo para clarificar el manejo de los campos fiscales en pedidos de la región Perú.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — TRAMITACIÓN, ACTIVACIÓN Y OPCIONES EN PANTALLA
═══════════════════════════════════════════════════════════════

Criterio A1 — Tramitación habilitada para Prepago sin controlados con Factura por Adelantado activada
Dado que un pedido pertenece a un cliente Prepago en México o Perú, sin productos controlados, y el ESAC activa la opción Factura por Adelantado,
Cuando el ESAC opera el módulo Tramitar Pedido,
Entonces el sistema deberá permitir la tramitación y, al ejecutarse, generar automáticamente la proforma asociada al pedido.

Criterio A2 — Activación de Factura por Adelantado desde Tramitar Pedido
Dado que un pedido pertenece a un cliente Prepago sin productos controlados,
Cuando el ESAC visualiza el módulo Tramitar Pedido,
Entonces el sistema deberá ofrecer la opción de activar Factura por Adelantado. La activación es directa y no requiere código de autorización.

Criterio A3 — Bloqueo de edición de datos de facturación al activar Factura por Adelantado
Dado que el ESAC activó la opción Factura por Adelantado en Tramitar Pedido,
Cuando se renderiza la pantalla del pedido,
Entonces el botón ""Editar Datos"" para datos de facturación no debe aparecer disponible para este pedido. El sistema deberá mostrar los datos de facturación en modo solo lectura tomados del catálogo del cliente vigente al momento de la activación.

Criterio A4 — No renderizado de Entrega con Remisión para Prepago
Dado que el pedido es de cliente Prepago,
Cuando el ESAC visualiza la pantalla del pedido,
Entonces el radio button de Entrega con Remisión no deberá aparecer en la pantalla, dado que esta opción no aplica para clientes prepago en ninguna variante.

Criterio A5 — Composición regionalizada del panel de Información de Facturación
Dado que el ESAC visualiza el panel de Información de Facturación de un pedido en Tramitar Pedido,
Cuando el sistema renderiza el panel según la Región del cliente,
Entonces para clientes México deberá mostrar Uso CFDI y Método de Pago (catálogos SAT); para clientes Perú deberá mostrar Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago Contado/Crédito SUNAT en su lugar; en ambas regiones deberá mostrar los campos comunes (Razón Social, RFC/RUC, Moneda, Quién Factura, Condiciones de Pago comerciales y Comentarios) y NO deberá mostrar Forma de Pago ni correo de envío.

═══════════════════════════════════════════════════════════════
SECCIÓN B — FOLIOS Y GENERACIÓN DE LA PROFORMA
═══════════════════════════════════════════════════════════════

Criterio B1 — Asignación de folio interno al tramitar
Dado que el ESAC ejecuta la acción de tramitar,
Cuando el sistema procesa la solicitud,
Entonces deberá asignar el folio interno del pedido siguiendo la mecánica actual del sistema.

Criterio B2 — Asignación de folio de proforma desde foliador lineal global
Dado que el sistema genera la proforma al tramitar,
Cuando se asigna el folio del documento,
Entonces el sistema deberá tomar el siguiente número del foliador lineal global de PQF2 (un solo contador compartido por todas las proformas del sistema).

Criterio B3 — Generación del PDF de la proforma
Dado que el ESAC ejecuta la acción de tramitar,
Cuando se completa la asignación de folios,
Entonces el sistema deberá generar automáticamente el archivo PDF de la proforma con los datos del pedido, del cliente y los folios correspondientes.

═══════════════════════════════════════════════════════════════
SECCIÓN C — PREVISUALIZACIÓN Y ENVÍO DE LA PROFORMA
═══════════════════════════════════════════════════════════════

Criterio C1 — Previsualización obligatoria del PDF antes del envío
Dado que el PDF de la proforma se generó exitosamente,
Cuando el sistema inicia el proceso de envío,
Entonces deberá mostrar al ESAC la previsualización del PDF y requerir aceptación explícita antes de continuar a la pantalla de datos de envío.

Criterio C2 — Cancelación desde la previsualización
Dado que el ESAC está viendo la previsualización del PDF de la proforma,
Cuando el ESAC decide no continuar (cancela la previsualización),
Entonces el sistema deberá permitir volver al pedido sin enviar la proforma. ** Pendiente definir la política del folio de proforma ya asignado: si se conserva para el reintento o se descarta. **

Criterio C3 — Pantalla de datos de envío con CC editable y ESAC incluido
Dado que el usuario llegó al paso de envío del correo,
Cuando el sistema renderiza la pantalla,
Entonces deberá mostrar: Para con el contacto del cliente (editable, default heredado); CC con el ESAC asignado (editable, default sugerido); Asunto generado por sistema según plantilla (no editable); Adjuntos con el PDF de la proforma (no editables); y Notas extras como campo de texto editable opcional.

Criterio C4 — Envío del correo de proforma
Dado que la pantalla de envío está completa,
Cuando el usuario presiona Enviar,
Entonces el sistema deberá enviar el correo al destinatario con CC al ESAC, asunto generado por sistema, adjunto del PDF de la proforma y las notas extras opcionales capturadas.

═══════════════════════════════════════════════════════════════
SECCIÓN D — PENDIENTES GENERADOS Y CIERRE
═══════════════════════════════════════════════════════════════

Criterio D1 — Generación automática del pendiente Factura por Adelantado
Dado que el correo de proforma se envió exitosamente y el pedido tiene Factura por Adelantado activada,
Cuando se completa el envío,
Entonces el sistema deberá generar automáticamente un pendiente en el módulo Factura por Adelantado asociado al folio del pedido, para que se ejecute posteriormente la emisión y timbrado de la factura.

Criterio D2 — No generación del pendiente Validar Cobro al tramitar
Dado que el ESAC tramitó un pedido prepago con Factura por Adelantado activada,
Cuando se completa la tramitación y el envío,
Entonces el sistema no deberá generar pendiente en Validar Cobro en este momento. El pendiente Validar Cobro se generará posteriormente, cuando la factura se emita exitosamente desde el módulo Factura por Adelantado.

Criterio D3 — Desaparición del pendiente en bandeja Tramitar Pedido
Dado que el pedido se tramitó exitosamente (incluyendo el envío del correo de proforma y la generación del pendiente en Factura por Adelantado),
Cuando se completa la tramitación,
Entonces el pedido no deberá seguir apareciendo como pendiente en la bandeja del módulo Tramitar Pedido del ESAC. La consulta histórica del pedido sigue disponible desde los reportes correspondientes.

Criterio D4 — Cancelación del pedido
Dado que un pedido tramitado tiene solicitud del cliente para cancelar,
Cuando el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
Entonces el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder."								"- Variante prepago sin sustancias controladas con activación de Factura por Adelantado del módulo Tramitar Pedido. El módulo de Tramitar Pedido en este flujo es responsable de generar la proforma, gestionar la previsualización y envío al cliente, y disparar el pendiente en Factura por Adelantado. La emisión y timbrado de la factura PPD ocurren posteriormente en el módulo Factura por Adelantado.
- Cubre tres requisitos del cliente: tramitación de pedidos prepago en México y Perú con emisión de Confirmación; generación y envío automático de proforma para prepago; y cadena de pendientes generada al tramitar.
- Distinción clave respecto al flujo prepago sin Factura por Adelantado: en este flujo, al tramitar se genera el pendiente en Factura por Adelantado pero NO el pendiente en Validar Cobro. El pendiente Validar Cobro se generará después, cuando la factura PPD se emita exitosamente desde el módulo Factura por Adelantado. Esto refleja que en este flujo el documento que se va a cobrar es la factura PPD, no la proforma.
- Cambio respecto al comportamiento actual: se elimina el código de autorización para activar Factura por Adelantado (antes lo requería). La activación ahora es directa.
- Para clientes prepago, los datos de facturación nunca se pueden editar en Tramitar Pedido (independientemente de si hay sustancias controladas o no, independientemente de si se activa Factura por Adelantado o no). El botón ""Editar Datos"" no aparece. Cualquier ajuste a los datos fiscales del cliente debe gestionarse en el Catálogo de Clientes.
- El radio button de Entrega con Remisión no se renderiza en el módulo Tramitar Pedido para clientes prepago en ninguna variante.
- El foliador de la proforma es lineal global a PQF2 (un solo contador para todas las proformas del sistema). El folio del pedido interno conserva la mecánica actual del sistema.
- El asunto del correo de proforma se compone como ""Proforma"" más el folio del pedido interno.
- El flujo de envío del correo de proforma requiere dos pasos secuenciales en la UI: primero previsualizar y aceptar el PDF; después confirmar los datos de envío del correo.
- El pendiente del pedido en la bandeja del módulo Tramitar Pedido se cierra automáticamente al completarse la acción de tramitar. La consulta del pedido tramitado sigue disponible desde módulos de consulta. Esta mecánica evita que el ESAC vea pedidos ya gestionados en su bandeja de pendientes.
- Los campos de información fiscal del módulo están actualmente configurados conforme a las normas fiscales de México. Para pedidos peruanos se espera capacitación al equipo operativo para clarificar el manejo de estos campos en contexto peruano."									
TPSC-RE-FU-017		"Diseño y generación de Documentos: Proforma Perú
"	Tramitar Pedido	Yo como ESAC, quiero que el sistema genere automáticamente el PDF de la Proforma adaptado a la normativa fiscal SUNAT al tramitar un pedido Prepago para clientes de Perú, para entregar al cliente un documento estandarizado y conforme a la regulación peruana que respalde el cobro por adelantado.	El sistema debe generar un PDF de Proforma al tramitar un pedido Prepago sin Factura por Adelantado para clientes con Región Perú, con un diseño estandarizado equivalente al de la Proforma México pero adaptado a la normativa fiscal SUNAT. Dado que la única empresa emisora del grupo operando en Perú es Golocaer S.A.C., el branding del documento es único. El PDF se genera bajo demanda durante el flujo previo al envío al cliente y, al confirmarse el envío, se persiste como artefacto histórico inmutable accesible desde el módulo Validar Cobro.			Propuesto					R16.1M-RE-FU-007	"## Aplica a
- Generación del PDF de Proforma al tramitar un pedido en modalidad Prepago sin factura por adelantado para clientes con Región Perú.
- Empresa emisora única: Golocaer S.A.C. (la única empresa del grupo PROQUIFA operando actualmente en Perú).
- Generación bajo demanda del PDF durante el flujo previo al envío de la Proforma al cliente.
- Persistencia del PDF en base de datos al recibir confirmación de envío exitoso del correo al cliente.
- Acceso al PDF histórico desde el módulo Validar Cobro una vez la Proforma fue enviada.
- Foliador global lineal PQF2 con prefijo PRF en la representación visual del documento (compartido con Proformas México: un solo contador global de Proformas para todo el grupo).
- Paginación automática cuando las partidas exceden el espacio de una página (comportamiento ya existente del sistema).
- Aplicación de catálogos fiscales SUNAT peruanos (RUC, IGV, CCI, moneda PEN, Reglamento de Comprobantes de Pago y Resolución de Superintendencia N° 097-2012/SUNAT).

## No aplica a
- Pedidos Crédito sin Factura por Adelantado ni Crédito/Prepago con Factura por Adelantado.
- Pedidos para clientes con Región México. Esa funcionalidad se documenta en requisito independiente.
- Otras empresas del grupo PROQUIFA. Solo Golocaer S.A.C. opera actualmente en Perú; las cuatro empresas del grupo México (Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V., Proveedora Quimico Farmaceutica S.A. de C.V.) no emiten proformas para clientes Perú.
- Régimen de Detracciones SUNAT (SPOT). ** Bajo análisis preliminar, los productos típicos de PROQUIFA NO están en los anexos sujetos a detracción de la R.S. 183-2004/SUNAT. La aplicabilidad final debe confirmarse con asesor contable peruano antes de habilitar el módulo productivamente. **
- Régimen de Percepciones SUNAT del IGV. ** Bajo análisis preliminar, Golocaer S.A.C. NO está designada por SUNAT como Agente de Percepción y sus productos no están en el Apéndice 1 de la Ley N° 29173. La aplicabilidad final debe confirmarse con asesor contable peruano antes de habilitar el módulo productivamente. **"	"## Reglas de Negocio

Regla 1 — Generación únicamente en pedidos Prepago sin Factura por Adelantado para clientes Perú
El sistema genera el PDF de Proforma con el diseño estandarizado para Perú únicamente cuando el pedido es en modalidad Prepago sin Factura por Adelantado y el cliente tiene Región = Perú. Los pedidos Crédito (con o sin Factura por Adelantado) y los pedidos Prepago con Factura por Adelantado no generan Proforma.

Regla 2 — Empresa emisora única Golocaer S.A.C.
La empresa emisora del documento para clientes Perú es siempre Golocaer S.A.C., única empresa del grupo PROQUIFA operando en Perú. No hay diferenciación por empresa emisora como en México. ** Pendiente confirmar con el cliente. **

Regla 3 — Foliador global con prefijo PRF compartido con Proformas México
El folio de la Proforma Perú usa el mismo foliador global lineal PQF2 que las Proformas México (un solo contador global para todo el grupo, sin segmentación por región ni por empresa), en formato MMDDAA-Consecutivo, con prefijo ""PRF-"" en la representación visual del documento. ** Pendiente confirmar si el prefijo se persiste también en el folio interno almacenado en base de datos. **

Regla 4 — Vigencia del documento
La Proforma calcula y muestra una fecha de vigencia en formato DD/MM/YYYY. ** La regla exacta del cálculo de la vigencia queda como duda formal del proyecto, pendiente de confirmar con el cliente. **

Regla 5 — Generación bajo demanda durante el flujo previo al envío
Al presionar ""Tramitar"" para un pedido Prepago sin Factura por Adelantado de cliente Perú, el sistema genera el PDF de la Proforma dinámicamente, leyendo los datos vigentes en ese momento desde las fuentes (Catálogo de Clientes, Pedido, Catálogo de Cuentas Bancarias Perú, Referencia Bancaria del Cliente), y lo muestra en previsualización. En esta etapa el PDF no se almacena en base de datos.

Regla 6 — Regeneración con datos actualizados si el usuario abandona la previsualización y reintenta
Si un usuario vio la previsualización pero abandonó el flujo sin enviar la Proforma, al volver al pedido y presionar ""Tramitar"" nuevamente el sistema regenera el PDF desde cero leyendo los datos vigentes en ese nuevo momento. Si entre intentos cambiaron datos fuente (razón social del cliente, dirección fiscal, precios, cuentas bancarias, etc.), la nueva versión los refleja. Esto aplica únicamente mientras la Proforma no haya sido enviada al cliente.

Regla 7 — Persistencia del PDF al recibir confirmación del envío exitoso del correo
Al confirmarse que el correo de la Proforma fue enviado exitosamente al cliente, el sistema persiste la versión final del PDF en base de datos como artefacto histórico inmutable, con los datos exactos enviados. El pendiente en Tramitar Pedido se cierra.

Regla 8 — Sin regeneración posterior al envío
Una Proforma enviada y persistida se entrega, al consultarse históricamente, desde el PDF almacenado en base de datos, sin regenerarlo desde los datos fuente actuales. El sistema no ofrece funcionalidad de reenvío.

Regla 9 — Consulta del PDF histórico desde Validar Cobro
Una Proforma enviada y persistida puede consultarse desde el módulo Validar Cobro para verificación y trazabilidad del cobro asociado, accediendo al PDF histórico.

Regla 10 — Disclaimer legal SUNAT
El documento muestra un texto fijo, equivalente bajo normativa SUNAT, que indica que es informativo previo a la emisión del Comprobante de Pago Electrónico (CPE) y carece de validez fiscal y tributaria conforme al Reglamento de Comprobantes de Pago. ** Texto propuesto: ""ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DEL COMPROBANTE DE PAGO ELECTRÓNICO (CPE). CARECE DE VALIDEZ FISCAL Y TRIBUTARIA CONFORME AL REGLAMENTO DE COMPROBANTES DE PAGO Y RESOLUCIÓN DE SUPERINTENDENCIA N° 097-2012/SUNAT."" Pendiente validación legal con asesor SUNAT antes de publicación productiva. **

Regla 11 — Paginación automática (comportamiento existente)
Cuando las partidas del pedido exceden el espacio disponible en una sola página, el sistema genera páginas adicionales con la misma cabecera y pie completo, mostrando la numeración ""X/Y"" en cada página. Este comportamiento ya existe en PQF2.

Regla 12 — Origen de los datos por sección
Los paneles del documento se arman desde las fuentes indicadas: datos de partidas (cantidad, descripción, precio unitario, importe) desde el Pedido; identificación del cliente, RUC y dirección fiscal desde el Catálogo de Clientes; moneda aplicada a los cálculos desde la moneda de facturación configurada en el Catálogo del cliente (no del pedido); Condiciones de Pago desde la configuración del cliente en el Catálogo; cuentas bancarias (Banca, Sucursal, Cuenta, CCI) desde el Catálogo de Cuentas Bancarias de Golocaer S.A.C. Perú; REF. CLIENTE de cada cuenta construida con la lógica de identificación de pagos peruana; Pedido interno, Parciales, Contacto y Lugar de entrega desde el Pedido; logo, color institucional, dirección y razón social legal generados por el sistema correspondientes a Golocaer S.A.C. Perú. ** El modelo de cuentas bancarias Perú y el modelo de Referencia Bancaria Perú no están definidos: brechas mayores documentadas en B1 y B2. **

## Riesgos

Riesgo 1 — Disclaimer legal SUNAT pendiente de validación
El disclaimer propuesto en el documento es una redacción aproximada basada en el marco normativo SUNAT. La redacción legal exacta debe ser validada por asesor contable o legal peruano antes de uso productivo. Un disclaimer mal redactado podría generar confusión en el cliente o exposición legal innecesaria.

Riesgo 2 — Régimen de Detracciones y Percepciones SUNAT con aplicabilidad pendiente de confirmar
Bajo análisis preliminar, los productos típicos de PROQUIFA (estándares químico-biológicos, cepas Microbiologics, sustancias controladas, columnas cromatográficas, equipos de laboratorio) no están en los anexos de Detracción de la R.S. 183-2004/SUNAT, y Golocaer S.A.C. no sería Agente de Percepción del IGV bajo la Ley N° 29173 para sus productos típicos. ** Confirmar formalmente con asesor contable peruano antes de habilitar Perú productivamente. Si algún producto cae en algún anexo de Detracción o si SUNAT designa a Golocaer S.A.C. como Agente de Percepción en el futuro, el sistema deberá adaptarse para reflejar el régimen aplicable. **

Riesgo 3 — Tipo de cambio inconsistente entre Proforma y validación de pago posterior
Si el tipo de cambio mostrado en la Proforma difiere del aplicado al recibir el pago en Validar Cobro, el cliente puede recibir documentos con montos distintos en moneda local generando confusión. La regla es la misma que en México: el tipo de cambio es el del día de generación de la Proforma.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL DOCUMENTO
═══════════════════════════════════════════════════════════════

Criterio A1 — Logo de Golocaer S.A.C.
Dado que el sistema renderiza la cabecera del documento,
Cuando incluye el logo,
Entonces deberá mostrar el logo de Golocaer S.A.C. correspondiente a la operación Perú.

Criterio A2 — Disclaimer legal SUNAT
Dado que el sistema renderiza la cabecera,
Cuando incluye el disclaimer legal,
Entonces deberá mostrar un texto que indique el carácter informativo del documento previo a la emisión del Comprobante de Pago Electrónico (CPE) bajo normativa SUNAT. ** Texto propuesto: ""ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DEL COMPROBANTE DE PAGO ELECTRÓNICO (CPE). CARECE DE VALIDEZ FISCAL Y TRIBUTARIA CONFORME AL REGLAMENTO DE COMPROBANTES DE PAGO Y RESOLUCIÓN DE SUPERINTENDENCIA N° 097-2012/SUNAT."" Pendiente validación legal con asesor SUNAT antes de publicación productiva. **

Criterio A3 — Título ""Proforma""
Dado que el sistema renderiza la cabecera,
Cuando incluye el título del documento,
Entonces deberá mostrar el texto ""Proforma"". ** Pendiente confirmar si en Perú el título canónico comercial es ""Proforma"" o ""Factura Proforma"" — ambos términos se usan indistintamente en la práctica peruana. **

Criterio A4 — Folio con prefijo PRF
Dado que el sistema renderiza la cabecera,
Cuando incluye el folio del documento,
Entonces deberá mostrar el folio con formato ""PRF-MMDDAA-Consecutivo"". El consecutivo corresponde al foliador global lineal PQF2. ** El momento exacto en que se consume el folio (al previsualizar vs al confirmar envío) queda como duda técnica del proyecto. **

Criterio A5 — Vigencia del documento
Dado que el sistema renderiza la cabecera,
Cuando incluye el campo Vigencia,
Entonces deberá mostrar la fecha de vigencia en formato DD/MM/YYYY. ** Regla exacta del cálculo pendiente confirmar. **

═══════════════════════════════════════════════════════════════
SECCIÓN B — IDENTIFICACIÓN DEL CLIENTE
═══════════════════════════════════════════════════════════════

Criterio B1 — Identificación del cliente
Dado que el sistema renderiza la sección Cliente,
Cuando incluye el identificador del cliente,
Entonces deberá mostrar el Alias del cliente desde el Catálogo de Clientes. ** Pendiente confirmar si el dato fuente correcto es Alias o Razón Social. **

═══════════════════════════════════════════════════════════════
SECCIÓN C — TABLA DE PARTIDAS
═══════════════════════════════════════════════════════════════

Criterio C1 — Datos de la tabla de partidas
Dado que el sistema renderiza la tabla de partidas,
Cuando incluye los datos por cada partida,
Entonces deberá mostrar: número consecutivo, cantidad, descripción (catálogo + descripción + marca), precio unitario con moneda, e importe calculado (cantidad × precio). Todos los datos provienen del Pedido.

═══════════════════════════════════════════════════════════════
SECCIÓN D — DATOS DE PAGO
═══════════════════════════════════════════════════════════════

Criterio D1 — Sub-Total, IGV y Gran Total
Dado que el sistema incluye los cálculos fiscales,
Cuando renderiza las líneas de monto,
Entonces deberá mostrar:
- ""Sub-Total"" con monto y moneda.
- ""IGV"" con tasa aplicable al pedido (18% según normativa SUNAT, salvo exoneraciones específicas que pudieran aplicar a productos puntuales — pendiente confirmar exoneraciones aplicables) y monto calculado.
- ""Gran Total"" con monto, suma de Sub-Total e IGV.
La moneda aplicada es la moneda de facturación del cliente desde el Catálogo (no la moneda del pedido). Para Perú las monedas típicas son PEN (Soles) y USD.

Criterio D2 — Monto del Gran Total expresado en letra
Dado que el sistema renderiza la conversión a letras del Gran Total,
Cuando incluye la leyenda monetaria,
Entonces deberá mostrar el monto en palabras según la moneda:
- Si moneda = soles peruanos: ""(XXX SOLES XX/100)"".
- Si moneda = dólares: ""(XXX DOLARES XX/100)"".
- Otras monedas: nomenclatura correspondiente.
** Pendiente confirmar la nomenclatura exacta esperada para SUNAT (algunas implementaciones usan ""SOLES"" otras ""NUEVOS SOLES"" pese a que la moneda oficial desde 2015 es solo ""SOLES""). **

Criterio D3 — Tipo de Cambio (cuando aplica)
Dado que la moneda de facturación del cliente NO es soles peruanos,
Cuando el sistema renderiza la sección de pago,
Entonces deberá mostrar el tipo de cambio aplicado a la conversión. El tipo de cambio es el del día de generación. ** Pendiente confirmar si para Perú aplica el tipo de cambio SUNAT publicado (compra/venta) o un tipo de cambio interno corporativo. **

Criterio D4 — Condiciones de Pago
Dado que el sistema renderiza la sección de pago,
Cuando incluye las condiciones,
Entonces deberá mostrar las condiciones de pago aplicables al cliente (ejemplo: ""PREPAGO 100%""), provenientes de la configuración del cliente en el Catálogo.

Criterio D5 — Leyenda de pago
Dado que el sistema renderiza el final de la sección de pago,
Cuando incluye la leyenda de pago,
Entonces ** pendiente definir la leyenda equivalente bajo normativa SUNAT. La normativa peruana clasifica las operaciones como Contado o Crédito sin el concepto ""Pago en una sola exhibición"" propio del SAT mexicano. Propuesta: omitir esta leyenda o reemplazarla con ""OPERACIÓN AL CONTADO"" cuando aplique. Pendiente confirmar. **

═══════════════════════════════════════════════════════════════
SECCIÓN E — DATOS BANCARIOS
═══════════════════════════════════════════════════════════════

Criterio E1 — Cuentas bancarias de Golocaer S.A.C. Perú
Dado que el sistema renderiza la sección de datos bancarios,
Cuando arma el contenido,
Entonces deberá mostrar las cuentas bancarias de Golocaer S.A.C. Perú. ** El modelo bancario Perú no está definido: pendiente confirmar (a) cuántas cuentas se muestran, (b) en qué monedas (solo PEN, o PEN + USD), (c) en qué bancos peruanos opera Golocaer S.A.C. (BCP, BBVA Continental, Interbank, Scotiabank Perú u otros), (d) si se muestran siempre las cuentas independientemente de la moneda del pedido (análogo a México) o solo la cuenta de la moneda aplicable. Brecha mayor del proyecto. **
Los campos por cuenta esperados son: Moneda, Banca, Sucursal, Cuenta, CCI (Código de Cuenta Interbancario de 20 dígitos, en lugar de CLABE) y REF. CLIENTE.

Criterio E2 — Referencia bancaria del cliente (REF. CLIENTE)
Dado que el sistema renderiza la REF. CLIENTE de cada cuenta,
Cuando construye el valor,
Entonces ** el modelo de Referencia Bancaria para Perú no está definido. La lógica usada en México (cuenta Banamex con 7 segmentos basados en nombre del cliente, clave, código del banco, moneda y CodValidador; cuenta no-Banamex con nombre del cliente directo) es exclusiva de PROQUIFA México y no aplica a Perú. Brecha mayor del proyecto. Pendiente definir antes de habilitar Perú productivamente. **

═══════════════════════════════════════════════════════════════
SECCIÓN F — DATOS DE FACTURACIÓN
═══════════════════════════════════════════════════════════════

Criterio F1 — RUC, Razón Social, Dirección fiscal
Dado que el sistema renderiza la sección de facturación,
Cuando incluye los datos fiscales del cliente,
Entonces deberá mostrar:
- RUC del cliente desde el Catálogo de Clientes (en lugar de RFC; etiqueta del campo ""RUC"").
- Razón Social del cliente desde el Catálogo de Clientes.
- Dirección fiscal completa del cliente (calle, número, distrito, provincia, departamento, país) desde el Catálogo de Clientes. El formato de dirección refleja las convenciones administrativas peruanas (distrito/provincia/departamento en lugar de colonia/ciudad/estado).

═══════════════════════════════════════════════════════════════
SECCIÓN G — DATOS DE ENTREGA
═══════════════════════════════════════════════════════════════

Criterio G1 — Pedido, Parciales, Contacto, Lugar
Dado que el sistema renderiza la sección de entrega,
Cuando incluye los datos de entrega,
Entonces deberá mostrar:
- Número de pedido interno. ** Aplica la misma duda de generación de folio interno que en México (momento de generación cuando el pedido aún no se ha enviado). **
- Parciales (SI/NO) según configuración del pedido.
- Contacto de entrega del pedido (si no existe, mostrar ""NINGUNO""). ** Confirmar si es el contacto de entrega, contacto del cliente o contacto que realizó el pedido (misma duda que en México). **
- Lugar de entrega completo (dirección).

═══════════════════════════════════════════════════════════════
SECCIÓN H — PIE LEGAL DE GOLOCAER S.A.C.
═══════════════════════════════════════════════════════════════

Criterio H1 — Contacto Golocaer S.A.C. Perú
Dado que el sistema renderiza el pie del documento,
Cuando incluye la información de contacto,
Entonces deberá mostrar los datos de contacto institucionales de Golocaer S.A.C. Perú: redes sociales aplicables, teléfonos de oficinas Perú, web y correo de ventas Perú. ** Datos pendientes de capturar en el sistema: no se cuenta actualmente con la información de contacto de Golocaer S.A.C. Perú (teléfonos, web institucional Perú, correo Perú, redes sociales Perú). Brecha pendiente. **

Criterio H2 — Razón social legal de Golocaer S.A.C.
Dado que el sistema renderiza el pie legal,
Cuando incluye la razón social legal,
Entonces deberá mostrar la razón social legal completa ""Golocaer S.A.C."" con su dirección legal completa en Perú. ** La dirección legal de Golocaer S.A.C. en Perú no está disponible en el sistema actual. Brecha pendiente: recopilar y capturar antes de habilitar Perú. **

Criterio H3 — Sellos de certificación y métodos de pago aceptados
Dado que el sistema renderiza el pie,
Cuando incluye certificaciones y métodos de pago aceptados,
Entonces deberá mostrar las certificaciones vigentes aplicables a Golocaer S.A.C. Perú. ** El sello NEEC (Nuevo Esquema de Empresas Certificadas) NO aplica para Perú por ser programa SAT exclusivo México. Pendiente confirmar si Golocaer Perú cuenta con certificación ISO 9001 o equivalente. Pendiente confirmar métodos de pago aceptados aplicables al mercado peruano (visa, mastercard, etc.). Brecha pendiente. **

Criterio H4 — Numeración de página
Dado que el sistema completa el documento,
Cuando incluye el contador de páginas,
Entonces deberá mostrar ""X/Y"" en el pie del documento, donde X es la página actual e Y es el total. Si el documento es de una sola página, se muestra ""1/1"".

Criterio H5 — Logos de catálogos farmacéuticos
Dado que el sistema renderiza la línea final del documento,
Cuando incluye los logos de catálogos y proveedores reconocidos,
Entonces deberá mostrar los logos aplicables a la operación Perú. ** El logo FEUM (Farmacopea de los Estados Unidos Mexicanos) NO aplica para Perú. Los logos USP (United States Pharmacopeia, internacional), EDQM (European Directorate for the Quality of Medicines, europeo) y Microbiologics típicamente sí aplican. Pendiente confirmar la lista exacta de logos aplicables a Golocaer S.A.C. Perú. Brecha pendiente. **

═══════════════════════════════════════════════════════════════
SECCIÓN I — PAGINACIÓN AUTOMÁTICA
═══════════════════════════════════════════════════════════════

Criterio I1 — Múltiples páginas cuando las partidas exceden una página
Dado que el pedido tiene partidas que exceden el espacio disponible en una sola página,
Cuando el sistema renderiza el documento,
Entonces deberá generar páginas adicionales con la misma cabecera y pie completo. Las partidas continúan en las páginas adicionales. La numeración se actualiza (1/3, 2/3, 3/3). Este comportamiento ya existe en PQF2.

═══════════════════════════════════════════════════════════════
SECCIÓN J — PERSISTENCIA Y CONSULTA POST-ENVÍO
═══════════════════════════════════════════════════════════════

Criterio J1 — Generación bajo demanda durante el flujo previo al envío
Dado que un usuario presiona ""Tramitar"" en el módulo Tramitar Pedido para un pedido Perú,
Cuando el sistema procesa la acción,
Entonces deberá generar el PDF dinámicamente con los datos vigentes en ese momento y mostrarlo en previsualización al usuario. El PDF no se almacena en base de datos en esta etapa.

Criterio J2 — Regeneración con datos actualizados al reintentar
Dado que el usuario abandonó el flujo sin enviar la Proforma y vuelve a presionar ""Tramitar"",
Cuando el sistema procesa la nueva acción,
Entonces deberá regenerar el PDF desde cero con los datos fuente vigentes en ese nuevo momento. Si cambiaron datos entre intentos, el nuevo PDF los refleja.

Criterio J3 — Persistencia del PDF al confirmar envío exitoso del correo
Dado que el sistema confirma que el correo de envío al cliente fue exitoso,
Cuando se completa el envío,
Entonces deberá persistir el PDF final en base de datos como artefacto histórico inmutable. El pendiente en Tramitar Pedido se cierra.

Criterio J4 — Consulta del PDF histórico desde Validar Cobro
Dado que una Proforma fue enviada y persistida,
Cuando un usuario consulta el módulo Validar Cobro para procesar el cobro asociado,
Entonces el sistema deberá permitir acceder al PDF histórico de la Proforma. El PDF se entrega tal cual fue almacenado, sin regeneración desde datos fuente actuales.

Criterio J5 — Sin reenvío posterior
Dado que una Proforma fue enviada y persistida,
Cuando un usuario intenta reenviarla desde el módulo Tramitar Pedido,
Entonces el sistema no deberá ofrecer esa funcionalidad. El pendiente está cerrado y la Proforma original se conserva como registro permanente."								"- Esta fila documenta el contenido y la generación del PDF de Proforma para clientes con Región Perú. La equivalente para Región México se documenta en requisito independiente.
- El requisito es un rediseño del documento de Proforma. La estructura visual específica (colores exactos, layout de bandas, tipografía, espaciados) es decisión del equipo de diseño UI; este requisito se enfoca en la información que debe contener cada sección del documento adaptada a la normativa fiscal peruana.
- Aplica exclusivamente a pedidos Prepago que NO seleccionaron Factura por Adelantado. Los pedidos Crédito (con o sin Factura por Adelantado) y los pedidos Prepago con Factura por Adelantado NO generan Proforma.
- La única empresa emisora del grupo PROQUIFA operando actualmente en Perú es Golocaer S.A.C. No hay diferenciación por empresa emisora como en México (donde son cuatro empresas: Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica).
- El PDF se genera bajo demanda en cada presión del botón ""Tramitar"" durante el flujo previo al envío de la Proforma. Si el usuario abandona el flujo sin enviar y vuelve a presionar ""Tramitar"", el PDF se regenera leyendo los datos fuente vigentes en ese nuevo momento.
- Cuando el sistema confirma que el correo de envío al cliente fue exitoso, el PDF final se persiste en base de datos como artefacto histórico inmutable. A partir de ese momento, la Proforma enviada queda como registro permanente del documento exacto que recibió el cliente. No se ofrece funcionalidad de reenvío posterior.
- El PDF histórico de la Proforma enviada se puede consultar desde el módulo Validar Cobro.
- Diferencias normativas SUNAT respecto al modelo México:
  - Impuesto: IGV 18% (Impuesto General a las Ventas) en lugar de IVA 16%.
  - Identificador fiscal del cliente: RUC en lugar de RFC.
  - Código bancario interbancario: CCI (Código de Cuenta Interbancario, 20 dígitos) en lugar de CLABE (18 dígitos).
  - Moneda local: PEN (Soles peruanos) en lugar de MXN (Pesos mexicanos).
  - Disclaimer legal: marco SUNAT (Reglamento de Comprobantes de Pago y R.S. 097-2012/SUNAT) en lugar de marco SAT (Art. 29 y 29A CFF).
  - Documento fiscal final: Comprobante de Pago Electrónico (CPE) en lugar de CFDI.
  - Régimen de Pago: SUNAT clasifica Contado/Crédito sin el concepto ""Pago en una sola exhibición"" del SAT.
  - Sello NEEC: NO aplica a Perú (programa SAT exclusivo México).
  - Logo FEUM: NO aplica a Perú (farmacopea mexicana).
- Régimen de Detracciones (SPOT) y Régimen de Percepciones del IGV: bajo análisis preliminar NO aplican a los productos típicos de Golocaer S.A.C. Perú. Sin embargo, esta validación debe ser confirmada por asesor contable peruano antes de habilitar Perú productivamente. Si en el futuro algún producto se incorpora a los anexos de Detracción o si SUNAT designa a Golocaer S.A.C. como Agente de Percepción, el sistema debe adaptarse.

═══════════════════════════════════════════════════════════════
BRECHAS PENDIENTES DE RESOLUCIÓN ANTES DE HABILITAR PERÚ
═══════════════════════════════════════════════════════════════

Las siguientes brechas se documentan formalmente y deben resolverse antes de habilitar la generación de Proformas para clientes Perú en producción:

B1 — Modelo de cuentas bancarias de Golocaer S.A.C. Perú
No se conocen los bancos peruanos donde Golocaer opera, ni la cantidad de cuentas que se muestran en la Proforma, ni las monedas (PEN únicamente o PEN + USD), ni el formato del CCI de 20 dígitos. Pendiente capturar y configurar.

B2 — Modelo de Referencia Bancaria Perú (Código Validador)
La lógica para construir la REF. CLIENTE que permite identificar pagos del cliente en cuentas peruanas no está definida. La lógica México (Banamex 7 segmentos / no-Banamex nombre directo) es exclusiva del Legacy mexicano y no replica al contexto peruano. Pendiente definir mecanismo de identificación de pagos por banco peruano.

B3 — Datos legales y de contacto de Golocaer S.A.C. Perú
Dirección legal, teléfonos, web institucional, correo de ventas y redes sociales de Golocaer Perú no están disponibles en el sistema actual. Pendiente recopilar y capturar.

B4 — Disclaimer legal SUNAT
Texto exacto del disclaimer validado por asesor legal peruano. Propuesta documentada: ""ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DEL COMPROBANTE DE PAGO ELECTRÓNICO (CPE). CARECE DE VALIDEZ FISCAL Y TRIBUTARIA CONFORME AL REGLAMENTO DE COMPROBANTES DE PAGO Y RESOLUCIÓN DE SUPERINTENDENCIA N° 097-2012/SUNAT.""

B5 — Régimen de Detracciones y Percepciones
Confirmación formal por asesor contable peruano de que los productos típicos de PROQUIFA NO están sujetos a Detracción (R.S. 183-2004/SUNAT) y de que Golocaer S.A.C. NO sería Agente de Percepción para sus productos (Ley N° 29173).

B6 — Certificaciones aplicables a Golocaer S.A.C. Perú
ISO 9001 o equivalente, métodos de pago aceptados (medios peruanos), y cualquier otra certificación de calidad vigente en el mercado peruano.

B7 — Catálogos farmacéuticos aplicables a Perú
Lista definitiva de logos del pie inferior aplicables a Golocaer S.A.C. Perú. Confirmados como aplicables: USP, EDQM, Microbiologics. NO aplica: FEUM. Otros logos (APACOR, CHATA Biosystems, Pharmaffiliates) pendientes de confirmar.

B8 — Título canónico del documento en Perú
Confirmar si el título es ""Proforma"" o ""Factura Proforma"" (ambos términos se usan indistintamente en la práctica comercial peruana).

B9 — Nomenclatura del monto en letra para soles peruanos
Confirmar si la nomenclatura aceptada es ""SOLES"" (oficial desde 2015) o ""NUEVOS SOLES"" (denominación previa que aún aparece en algunas implementaciones).

B10 — Tipo de cambio aplicado en Perú
Confirmar si para Perú aplica el tipo de cambio SUNAT publicado (compra/venta) o un tipo de cambio interno corporativo.

Mientras estas brechas no se resuelvan, el cliente Perú no puede recibir Proforma productiva con el formato adaptado. Se recomienda al cliente programar sesión específica para resolución integral del modelo Perú."									
TPSC-RE-FU-018		"Factura por Adelantado
"	Factura por Adelantado	Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente de resolver) **, quiero contar con una pantalla inicial del módulo Factura por Adelantado que liste agrupadamente los clientes que tienen al menos un pedido pendiente de facturar por adelantado, mostrando su Razón Social, identificador fiscal, número de facturas pendientes y monto total dolarizado, para identificar rápidamente qué clientes requieren atención y navegar al detalle de sus pedidos pendientes.	El sistema debe contar con una pantalla inicial en el módulo Factura por Adelantado que presente un listado, agrupado por cliente, de los pedidos pendientes de facturar por adelantado. El listado se ordena por antigüedad del pendiente (los más antiguos primero) y su visibilidad se filtra por la cartera del usuario operativo (campo Cobrador asignado en el Catálogo de Clientes). La pantalla permite al usuario identificar rápidamente qué clientes tienen facturas pendientes y navegar al detalle de cada uno para gestionarlas.			Propuesto					R16.2M-RE-FU-001	"## Aplica a
- Módulo NUEVO en PQF2 R16: Factura por Adelantado (módulo no existe en la versión actual del sistema).
- Pantalla inicial del módulo: listado agrupado por cliente con pedidos pendientes de facturar por adelantado.
- Pedidos elegibles: aquellos con estado ""pendiente de Factura por Adelantado"" generados desde el módulo Tramitar Pedido en cualquiera de los dos flujos que requieren Factura por Adelantado: Crédito con Factura por Adelantado y Prepago con Factura por Adelantado.
- Visualización agrupada por cliente con conteo de pedidos pendientes y monto total dolarizado.
- Buscador único por Razón Social, identificador fiscal o número de pedido interno.
- Ordenamiento por antigüedad del pendiente.
- Visibilidad filtrada por cartera del usuario operativo (campo Cobrador del Catálogo de Clientes).
- Paginación numerada con navegación anterior/siguiente.
- Estado vacío cuando no hay pendientes en la cartera del usuario.
- Aplicación a clientes de México y Perú.

## No aplica a
- Pedidos con Sustancias Controladas tipo Mundial, Nacional u Origen (regla del cliente que prohíbe Factura por Adelantado para controlados; no llegan a este módulo desde Tramitar Pedido).
- Filtros adicionales al buscador único (por moneda, empresa emisora, fecha, etc.). La pantalla privilegia ligereza operativa."	"## Reglas de Negocio

Regla 1 — Origen de los pedidos elegibles
El listado de pendientes considera exclusivamente pedidos en estado ""pendiente de Factura por Adelantado"" generados desde el módulo Tramitar Pedido, incluyendo los del flujo Crédito con Factura por Adelantado y los del flujo Prepago con Factura por Adelantado. Los pedidos sin pendiente de Factura por Adelantado o ya facturados y enviados no aparecen en el listado.

Regla 2 — Agrupación por cliente con conteo de pendientes
El listado se presenta agrupado por cliente. Cada fila representa un cliente con: Razón Social, identificador fiscal (RFC para México / RUC para Perú), número de Facturas Pendientes (conteo), Monto Total dolarizado, y la acción Ver Pedidos para navegar al detalle.

Regla 3 — Conteo de Facturas Pendientes incluye pedidos con factura generada pero no enviada
El conteo de Facturas Pendientes del cliente incluye los pedidos con factura por adelantado ya generada pero pendiente de envío al cliente. El conteo refleja todos los pedidos que aún requieren acción operativa del usuario (generar factura o enviar factura) y solo se retira del conteo cuando la factura se envía exitosamente al cliente.

Regla 4 — Monto Total dolarizado por cliente
Cuando los pedidos del cliente están en distintas monedas (MXN, USD, PEN, EUR u otras), el Monto Total del cliente se obtiene convirtiendo cada monto a USD y sumándolos. La visualización es siempre dolarizada para facilitar la comparación entre clientes. Cada documento se convierte a USD con el tipo de cambio de su propio documento origen (no con el tipo de cambio del día de la consulta ni con uno unificado) y los montos ya dolarizados se suman; no existe un tipo de cambio único del listado.

Regla 5 — Ordenamiento por antigüedad del pendiente
El listado de clientes se ordena por antigüedad del pendiente más antiguo del cliente. Los clientes con el pendiente más antiguo aparecen primero, priorizando la atención de los casos que más han esperado.

Regla 6 — Visibilidad filtrada por cartera del usuario
El listado muestra únicamente los clientes asignados a la cartera del usuario operativo (campo Cobrador del Catálogo de Clientes). Los clientes asignados a otros usuarios no aparecen en el listado.

Regla 7 — Buscador único por Razón Social, identificador fiscal o pedido interno
El buscador único de la pantalla busca por coincidencia parcial alfanumérica en: Razón Social, identificador fiscal (RFC o RUC) y número de Pedido Interno de los pedidos pendientes del cliente. El resultado de búsqueda son clientes (filas del listado agrupado), no pedidos individuales; si la búsqueda por pedido encuentra coincidencia, se retorna el cliente que contiene ese pedido. La búsqueda se ejecuta en tiempo real cuando el usuario deja de escribir (no requiere presionar Enter ni lupa) e ignora los espacios al inicio y al final del texto ingresado.

Regla 8 — Paginación
Cuando el listado tiene un número de clientes superior al de elementos visibles por página, la pantalla ofrece paginación con numeración de páginas y navegación anterior/siguiente. El número de elementos por página se define por la implementación del módulo.

Regla 9 — Estado vacío
Cuando el usuario no tiene clientes con pedidos pendientes de Factura por Adelantado en su cartera, la pantalla muestra una vista de estado vacío con mensaje informativo claro y elementos visuales que comuniquen la ausencia de trabajo pendiente. La pantalla no queda en blanco ni muestra la tabla sin filas.

## Riesgos

Riesgo 1 — Brecha de timbrado para Perú
Esta pantalla en sí es agnóstica al país, pero el módulo Factura por Adelantado completo depende de capacidad de timbrado fiscal por región. Para México existe integración con TurboPac. Para Perú la integración con OSE/SUNAT es una brecha mayor del proyecto documentada en TPSC-RE-FU-005 (Brecha 5). Mientras la brecha de facturación de Perú no esté habilitada, no deberían generarse pendientes de Perú ligados al timbrado, para evitar pendientes huérfanos que no puedan completarse y el consecuente ruido operativo. El alcance definitivo de Perú en esta release está pendiente de definición.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — VISUALIZACIÓN DEL LISTADO
═══════════════════════════════════════════════════════════════

Criterio A1 — Visualización del listado agrupado por cliente
Dado que el usuario operativo accede al módulo Factura por Adelantado y tiene clientes con pedidos pendientes en su cartera,
Cuando se renderiza la pantalla inicial,
Entonces el sistema deberá presentar un listado con una fila por cada cliente que tenga al menos un pedido pendiente, mostrando: Razón Social del cliente, identificador fiscal con etiqueta de columna ""RFC/RUC"", número de Facturas Pendientes (conteo), Monto Total dolarizado, y acción ""Ver Pedidos"".

Criterio A2 — Etiqueta de columna del identificador fiscal
Dado que los clientes en el listado pueden ser de México (RFC) o Perú (RUC),
Cuando el sistema renderiza la cabecera de la tabla,
Entonces la etiqueta de la columna del identificador fiscal deberá ser ""RFC/RUC"" como etiqueta dual estática. El valor mostrado en cada fila es el identificador correspondiente a la región del cliente.

Criterio A3 — Aplicación uniforme a clientes México y Perú
Dado que los clientes del listado pueden ser de México o Perú,
Cuando el sistema renderiza la pantalla,
Entonces la funcionalidad opera de manera idéntica para ambos países. La diferenciación por región afecta solo el contenido (RFC vs RUC, monedas distintas) y no la mecánica de la pantalla.

═══════════════════════════════════════════════════════════════
SECCIÓN B — CÁLCULO DE CONTEO Y MONTO
═══════════════════════════════════════════════════════════════

Criterio B1 — Origen de los pedidos elegibles para el conteo
Dado que el sistema cuenta los pedidos pendientes de Factura por Adelantado por cliente,
Cuando construye el conteo,
Entonces deberá considerar pedidos con pendiente de Factura por Adelantado originados desde Tramitar Pedido en ambos flujos: Crédito con Factura por Adelantado y Prepago con Factura por Adelantado. El conteo incluye pedidos sin factura generada y pedidos con factura generada pero pendiente de envío al cliente. Solo cuando la factura se envía exitosamente, el pedido sale del conteo.

Criterio B2 — Cálculo del Monto Total dolarizado
Dado que un cliente tiene pedidos pendientes en distintas monedas (MXN, USD, PEN, EUR u otras),
Cuando el sistema calcula el Monto Total del cliente,
Entonces deberá convertir cada monto del pedido a USD y sumar los resultados. El valor mostrado en la columna Monto Total es la sumatoria en USD. El tipo de cambio aplicado a cada conversión es el del propio documento origen de cada pedido (no el del día de consulta ni uno unificado).

═══════════════════════════════════════════════════════════════
SECCIÓN C — ORDEN, FILTRADO Y BÚSQUEDA
═══════════════════════════════════════════════════════════════

Criterio C1 — Ordenamiento por antigüedad
Dado que el listado contiene múltiples clientes con pendientes,
Cuando el sistema los ordena para visualización,
Entonces deberá ordenarlos por antigüedad del pendiente más antiguo del cliente. Los clientes con el pendiente que más tiempo lleva en estado de espera aparecen primero.

Criterio C2 — Visibilidad filtrada por cartera del usuario
Dado que el usuario accede al módulo,
Cuando el sistema arma el listado,
Entonces deberá mostrar únicamente los clientes asignados a la cartera del usuario operativo (campo Cobrador del Catálogo de Clientes). Los clientes asignados a otros usuarios no aparecen.

Criterio C3 — Buscador por Razón Social, RFC/RUC o pedido interno
Dado que el usuario ingresa texto en el buscador único de la pantalla,
Cuando el sistema procesa la búsqueda,
Entonces deberá realizar coincidencia parcial alfanumérica en los campos: Razón Social del cliente, identificador fiscal del cliente, y números de Pedido Interno de los pedidos pendientes del cliente, ignorando los espacios al inicio y al final del texto ingresado. El resultado son filas del listado agrupado por cliente. Si la coincidencia se da por número de pedido interno, se retorna el cliente que contiene ese pedido.

Criterio C4 — Paginación con numeración y navegación anterior/siguiente
Dado que el listado tiene más clientes que los visibles en una sola página,
Cuando el sistema renderiza la pantalla,
Entonces deberá ofrecer paginación con numeración de páginas y botones de navegación anterior/siguiente.

═══════════════════════════════════════════════════════════════
SECCIÓN D — NAVEGACIÓN Y ESTADO VACÍO
═══════════════════════════════════════════════════════════════

Criterio D1 — Acción Ver Pedidos para navegar al detalle
Dado que el usuario identifica un cliente en el listado del que quiere consultar el detalle de pedidos pendientes,
Cuando ejecuta la acción ""Ver Pedidos"",
Entonces el sistema deberá navegar a la pantalla de Detalle por cliente del módulo Factura por Adelantado, llevando el contexto del cliente seleccionado para que el detalle muestre los pedidos pendientes del cliente.

Criterio D2 — Estado vacío
Dado que el usuario operativo accede al módulo Factura por Adelantado y no tiene clientes con pedidos pendientes en su cartera,
Cuando el sistema renderiza la pantalla,
Entonces deberá mostrar una vista de estado vacío con mensaje informativo y elementos visuales apropiados que comuniquen la ausencia de trabajo pendiente. No debe mostrar la tabla sin filas ni dejar la pantalla en blanco."								"- Módulo NUEVO en PQF2 R16. El módulo Factura por Adelantado no existe en la versión actual del sistema y se incorpora en R16 para soportar el caso de negocio en el que un cliente solicita facturación anticipada antes del ingreso de la mercancía al almacén.
- Distinción conceptual importante (no confundir): la Factura por Adelantado es una factura de venta emitida anticipadamente como apoyo al cliente que solicita facturación antes del ingreso de la mercancía, y nunca se genera para Sustancias Controladas por su implicación regulatoria. Es un concepto distinto de la Factura Anticipo, que es exclusiva de pedidos Prepago con Sustancias Controladas (tipo Mundial, Nacional u Origen) y se genera desde Validar Cobro, no desde este módulo.
- Esta fila cubre exclusivamente la PANTALLA INICIAL del módulo: el listado agrupado por cliente con pedidos pendientes.
- Los pedidos elegibles para el conteo provienen de los flujos Crédito con Factura por Adelantado y Prepago con Factura por Adelantado, originados en Tramitar Pedido. Los pedidos con Sustancias Controladas no son elegibles para Factura por Adelantado por regla del cliente.
- El conteo de ""Facturas Pendientes"" del cliente incluye: (a) pedidos sin factura generada todavía y (b) pedidos con factura generada pero pendiente de envío al cliente. Solo sale del conteo cuando la factura se envía exitosamente.
- La conversión de los montos a USD permite comparar visualmente clientes con monedas distintas (MXN, USD, PEN, EUR u otras). El tipo de cambio aplicado para la conversión es el de cada documento origen (no el del día de consulta ni uno unificado): cada documento se dolariza con su propio tipo de cambio y los montos se suman.
- La etiqueta de la columna del identificador fiscal es ""RFC/RUC"" como etiqueta dual estática. El valor mostrado en cada fila corresponde a la región del cliente.
- El buscador único permite buscar por Razón Social, identificador fiscal o número de pedido interno. La búsqueda se ejecuta en tiempo real al dejar de escribir, con coincidencia parcial alfanumérica. Si la coincidencia se da por número de pedido interno, el resultado es el cliente que contiene ese pedido (el listado siempre se renderiza agrupado por cliente, no por pedido).
- La paginación aplica numeración de páginas más botones de navegación ""Anterior"" y ""Siguiente"".
- El ordenamiento por antigüedad del pendiente prioriza los casos más antiguos. No se implementan semáforos de prioridad ni alertas visuales adicionales (decisión de diseño: ligereza operativa sobre analítica visual).
- ** Pendiente: denominación canónica del rol operativo entre ""Gestor de Cobranza"" (matriz cliente y otros requisitos del proyecto) y ""Analista de Cuentas por Cobrar"" (sesión de revisión de pantallas Factura por Adelantado 8-abr). Resolver antes del desarrollo. **
- Tipo de cambio aplicado a la conversión de Monto Total a USD: el de cada documento origen (cada documento se dolariza con su propio tipo de cambio y se suman).
- La habilitación efectiva del módulo para clientes Perú depende de la resolución de la brecha de timbrado SUNAT (integración con OSE/SUNAT), documentada en TPSC-RE-FU-005 (Brecha 5). Mientras la brecha no se resuelva, esta pantalla podría mostrar pedidos de clientes Perú pero el flujo completo no podría cerrarse para ellos al timbrar.
- Aplicabilidad uniforme a clientes México y Perú en esta pantalla; las diferencias por región surgen al momento del timbrado de la factura."									
TPSC-RE-FU-019		Factura por Adelantado: Detalle México	Factura por Adelantado	Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver) **, quiero contar con una pantalla de detalle por cliente en Factura por Adelantado que me permita generar, timbrar y enviar la factura de cada pedido pendiente, para emitir oportunamente las facturas por adelantado y dar continuidad al flujo de cobro o transferencia a Legacy según el tipo de pedido.	El sistema debe contar con una pantalla de Detalle por cliente en el módulo Factura por Adelantado que muestre los datos del cliente y el listado de sus pedidos pendientes de facturar por adelantado, ofreciendo por cada pedido la acción de generar la factura o, si ya fue timbrada, de enviarla al cliente. El flujo de generación contempla la revisión de los datos fiscales, la previsualización del PDF y el timbrado ante el SAT; tras un envío exitoso el pedido sale del listado de pendientes. La salida operativa posterior depende del tipo de pedido: los de Crédito continúan por el flujo Crédito y los de Prepago generan el pendiente correspondiente en Validar Cobro. Esta funcionalidad no aplica a pedidos que contengan productos clasificados como Sustancias Controladas (tipo Mundial, Nacional u Origen).			Propuesto					R16.2M-RE-FU-001	"## Aplica a
- Pantalla de Detalle por cliente del módulo Factura por Adelantado.
- Listado de pedidos del cliente seleccionado con pendiente de generar o enviar Factura por Adelantado.
- Acciones contextuales por pedido: ""Generar Factura"" (pedido sin factura todavía) o ""Enviar Factura"" (pedido con factura ya generada pendiente de envío).
- Modal de Generación de Factura (revisión de datos fiscales del cliente, del emisor y del pedido; único campo editable: Uso CFDI).
- Modal de Previsualización del PDF de la Factura antes del timbrado SAT.
- Modal de Alerta SAT con descripción del error de validación o timbrado, cuando aplique.
- Modal de éxito de generación cuando el timbrado se completa exitosamente.
- Modal de Envío de Factura (destinatario, CC, asunto, adjuntos PDF + XML, notas del correo).
- Modal de éxito de envío cuando el correo se envía exitosamente.
- Cambio de estado del pedido en el listado conforme avanza el ciclo (pendiente generar → pendiente enviar → desaparece tras envío).
- Salida operativa del pedido tras envío exitoso: transferencia a Legacy para pedidos Crédito; generación del pendiente correspondiente en Validar Cobro para pedidos Prepago.
- Aplicación a clientes con Región México exclusivamente.

## No aplica a
- Pedidos que contengan productos clasificados como Sustancias Controladas tipo Mundial, Nacional u Origen. La Factura por Adelantado no es elegible para estos pedidos, independientemente del tipo (Crédito o Prepago).
- Pedidos para clientes con Región Perú. La habilitación para Perú depende de la resolución de la brecha de timbrado SUNAT/OSE y de las consideraciones fiscales peruanas; se documenta en requisito independiente cuando esté disponible.
- Generación del PDF de la Factura como artefacto (estructura visual, secciones, datos a renderizar): se documenta en requisito independiente, análogo al del PDF de Proforma.
- Reenvío de la Factura tras envío exitoso al cliente: no se ofrece funcionalidad de reenvío; la Factura queda persistida y consultable desde Validar Cobro.
- Edición de datos fiscales del cliente desde esta pantalla: esos datos se administran en el Catálogo de Clientes. Si hay error de validación SAT por datos del cliente (ejemplo: Código Postal), el usuario debe ir al Catálogo a corregir y volver a reintentar."	"## Reglas de Negocio

Regla 1 — Estados del pedido visibles en el listado
Cada pedido del cliente seleccionado muestra una acción contextual a su estado: ""Generar Factura"" si el pedido aún no tiene factura emitida (estado pendiente generar), o ""Enviar Factura"" si la factura ya fue generada y timbrada pero está pendiente de envío al cliente (estado pendiente enviar). Una vez la factura se envía exitosamente, el pedido desaparece del listado.

Regla 2 — Datos del cliente en cabecera del Detalle
La cabecera del Detalle del cliente muestra: Razón Social del cliente, identificador fiscal (RFC), moneda de facturación y la clasificación crediticia del cliente si está disponible. Son datos preexistentes en el sistema, en modo solo lectura, no editables desde este módulo.

Regla 3 — Datos del pedido visibles en el listado
Cada pedido del cliente en el listado muestra: Pedido Interno, Fecha del pedido, Condiciones de Pago (ejemplo: ""PREPAGO 100%"", ""30 DIAS"", ""60 DIAS"", ""90 DIAS""), Empresa Emisora del pedido, Subtotal, IVA y Monto Total en la moneda del pedido.

Regla 4 — Modal de Generación con Uso CFDI como único editable
Al presionar ""Generar Factura"" en un pedido, el modal de Generación de Factura muestra en modo solo lectura: cliente (Razón Social o Alias), Monto Total del pedido, Pedido Interno, Condiciones de Pago, datos del contacto del cliente (nombre, correo electrónico, teléfono), Datos Fiscales del Cliente (RFC, Razón Social, Código Postal, Régimen Fiscal, Correo electrónico, Moneda, Tipo de Cambio, Tipo de Comprobante, Método de Pago, Forma de Pago), Datos Fiscales del Emisor (RFC, Razón Social, Régimen Fiscal) y los Comentarios de Facturación. El único campo editable del modal es el Uso CFDI (combo de selección con catálogo SAT). ** Pendiente confirmar si el cliente se identifica por Razón Social o Alias. **

Regla 5 — Forma de Pago, Método de Pago y Tipo de Comprobante forzados por normativa SAT
Por ser la Factura por Adelantado una factura PPD, el modal de Generación presenta como valores forzados (solo lectura): Método de Pago = ""PPD - Pago en parcialidades o diferido"", Forma de Pago = ""99 - Por definir"", Tipo de Comprobante = ""I - Ingreso"". Estos valores son obligatorios por normativa SAT para facturas PPD y no son modificables por el usuario. La Forma de Pago real se captura posteriormente en el Complemento de Pago del módulo Validar Cobro.

Regla 6 — Tipo de Cambio del día de generación
Cuando el cliente factura en una moneda distinta a pesos mexicanos, el modal de Generación muestra el Tipo de Cambio aplicable al día de generación de la factura. El valor es solo lectura, no modificable por el usuario.

Regla 7 — Generación en dos pasos: revisar datos, previsualizar PDF, timbrar
Tras confirmar los datos en el modal de Generación de Factura, el sistema muestra el modal de Previsualización con el PDF de la Factura tal como saldrá al cliente. El usuario revisa el documento y confirma con ""Generar Factura"" desde el modal de Previsualización para proceder con el timbrado real ante el PAC SAT. La acción ""Cancelar"" en cualquiera de los dos modales aborta el flujo sin timbrar.

Regla 8 — Manejo de errores de validación SAT o de timbrado
Cuando el timbrado ante el PAC falla por un error de validación (ejemplo: el Código Postal del cliente no coincide con el registrado en el SAT) o por un error del propio PAC (errores de decimales en totales, indisponibilidad del servicio, otros), el sistema muestra el modal de Alerta SAT con la descripción específica del error y un botón ""Continuar"" que cierra el modal y devuelve al usuario al estado previo. El usuario debe corregir los datos en el módulo correspondiente (típicamente el Catálogo de Clientes para errores de datos fiscales del cliente) y reintentar la generación desde el principio. El sistema no ofrece edición directa de los datos del cliente desde este modal.

Regla 9 — Persistencia de la Factura al timbrarse exitosamente
Cuando el timbrado se completa exitosamente, el sistema persiste la Factura en base de datos como artefacto fiscal inmutable (incluye el documento timbrado completo: XML y representación visual PDF) y muestra al usuario el modal de confirmación de éxito de generación. A diferencia de la Proforma (que se persiste al envío), la Factura se persiste al timbrarse: una vez timbrada ya no se puede modificar.

Regla 10 — Folio de la Factura por empresa emisora
El folio de la Factura usa un contador consecutivo numérico independiente por empresa emisora (Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica). El folio de la Factura es de tipo varchar(6).

Regla 11 — Modal de Envío de Factura
Cuando la Factura fue timbrada exitosamente y el usuario presiona ""Enviar Factura"", el modal de Envío presenta: Para (destinatario) con el contacto del cliente del pedido, editable, con default heredado del flujo de tramitación; CC con el ESAC asignado al cliente/pedido, editable, con default sugerido por el sistema; Asunto generado automáticamente con los folios del documento, no editable; Adjuntos con el PDF y XML de la Factura timbrada, no editables; y Notas extras, un campo de texto editable opcional para texto adicional libre.

Regla 12 — Confirmación de envío exitoso y cierre del pendiente
Al confirmarse que el correo se envió exitosamente al cliente, el sistema muestra el modal de confirmación de éxito de envío, el pedido sale del listado del Detalle, y la factura queda persistida y consultable desde el módulo Validar Cobro.

Regla 13 — Salida operativa post-envío diferenciada por tipo de pedido
Tras el envío exitoso de la Factura por Adelantado, el sistema actúa diferenciadamente según el tipo del pedido: si es Crédito, la información de la Factura se transfiere a Legacy y el pedido continúa por el flujo Crédito en Legacy; si es Prepago, el sistema genera el pendiente correspondiente en el módulo Validar Cobro para que el equipo de Cobranza valide el pago del cliente contra esta Factura.

Regla 14 — Visibilidad filtrada por cartera del usuario
El acceso al Detalle de un cliente solo se permite si el cliente está asignado a la cartera del usuario (campo Cobrador del Catálogo de Clientes). Los clientes asignados a otros usuarios no son accesibles.

Regla 15 — Datos de la Factura provienen del Pedido
Los conceptos de la Factura se obtienen del Pedido: por cada partida, cantidad, descripción (catálogo + descripción + marca, sin lote), precio unitario e importe. La Factura por Adelantado se timbra sin lote ni pedimento: al emitirse antes del surtido del pedido (la asignación de lote en almacén ocurre después de cobrar y facturar), el lote no está disponible y no se consigna, igual que el pedimento, que se representa con ""N/A"". No se reserva lote anticipadamente ni se ajusta posteriormente con Nota de Crédito.

## Riesgos

Riesgo 1 — Indisponibilidad del PAC TurboPac
Si el PAC TurboPac (RFC QSO100827UB0, Quadrum Tecnologías) está caído o responde con timeout, no se puede timbrar la Factura.

Riesgo 2 — Solapamiento de denominación de rol
La denominación del rol que opera este módulo aparece como ""Gestor de Cobranza"" en la matriz del cliente y como ""Analista de Cuentas por Cobrar"" en sesiones de revisión de pantallas Factura por Adelantado. ** Pendiente resolver formalmente la denominación canónica antes del desarrollo. **

Riesgo 3 — Brecha de timbrado para Perú
La habilitación para Región Perú depende de la integración con OSE/SUNAT autorizado, brecha mayor no resuelta del proyecto documentada en TPSC-RE-FU-005 (Brecha 5). Mientras no se resuelva, los clientes Perú no pueden usar este módulo.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL CLIENTE Y LISTADO DE PEDIDOS
═══════════════════════════════════════════════════════════════

Criterio A1 — Datos del cliente en cabecera
Dado que el usuario navega al Detalle de un cliente desde el listado del módulo,
Cuando el sistema renderiza la cabecera,
Entonces deberá mostrar Razón Social del cliente, identificador fiscal (RFC), moneda de facturación, y la clasificación preexistente del cliente si está disponible.

Criterio A2 — Listado de pedidos pendientes
Dado que el cliente tiene facturas pendientes de generar o enviar Factura por Adelantado,
Cuando el sistema renderiza el listado,
Entonces deberá mostrar una fila por cada pedido pendiente con: Pedido Interno, Fecha del pedido, Condiciones de Pago, Empresa Emisora del pedido, Subtotal, IVA, Monto Total y acción contextual.

Criterio A3 — Acción contextual por estado del pedido
Dado que un pedido del cliente aparece en el listado,
Cuando el sistema renderiza la acción del pedido,
Entonces deberá mostrar:
- ""Generar Factura"" si el pedido NO tiene factura emitida (estado: pendiente generar).
- ""Enviar Factura"" si el pedido tiene factura ya timbrada pero pendiente de envío (estado: pendiente enviar).

═══════════════════════════════════════════════════════════════
SECCIÓN B — MODAL DE GENERACIÓN DE FACTURA
═══════════════════════════════════════════════════════════════

Criterio B1 — Apertura del modal al presionar ""Generar Factura""
Dado que el usuario presiona ""Generar Factura"" en un pedido del listado,
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal de Generación de Factura con los datos del pedido y del cliente cargados.

Criterio B2 — Datos del pedido en cabecera del modal
Dado que el modal de Generación se abre,
Cuando el sistema arma la cabecera del modal,
Entonces deberá mostrar: Razón Social del cliente, Monto Total del pedido (visualmente prominente), Pedido Interno, Condiciones de Pago.

Criterio B3 — Datos del contacto del cliente (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los datos del contacto del cliente,
Entonces deberá mostrar en modo solo lectura: nombre del contacto, correo electrónico, teléfono con extensión cuando aplique.

Criterio B4 — Datos Fiscales del Cliente (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los datos fiscales del cliente,
Entonces deberá mostrar en modo solo lectura: RFC, Razón Social, Código Postal, Régimen Fiscal, Correo electrónico, Moneda, Tipo de Cambio, Tipo de Comprobante (siempre ""I - Ingreso""), Método de Pago (siempre ""PPD - Pago en parcialidades o diferido""), Forma de Pago (siempre ""99 - Por definir"").

Criterio B5 — Datos Fiscales del Emisor (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los datos fiscales del emisor,
Entonces deberá mostrar en modo solo lectura los datos visibles al usuario:
- RFC del emisor.
- Emisor (identificador comercial: Golocaer, Mungen, Proquifa, Proveedora).
- Razón Social del emisor (razón social legal completa: Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V., Proveedora Quimico Farmaceutica S.A. de C.V.).

Criterio B6 — Comentarios de Facturación (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los comentarios de facturación,
Entonces deberá mostrar el texto del campo Comentarios de Facturación del pedido en modo solo lectura.

Criterio B7 — Uso CFDI como único campo editable
Dado que el modal de Generación está abierto,
Cuando el usuario interactúa con el modal,
Entonces el único campo modificable deberá ser Uso CFDI (combo de selección con el catálogo SAT correspondiente). Todos los demás datos son solo lectura.

Criterio B8 — Acciones del modal: Cancelar y Generar Factura
Dado que el usuario revisa los datos en el modal,
Cuando finaliza la revisión,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el flujo, cierra el modal sin generar factura) y Generar Factura (procede al siguiente paso: previsualización del PDF).

═══════════════════════════════════════════════════════════════
SECCIÓN C — MODAL DE PREVISUALIZACIÓN DEL PDF
═══════════════════════════════════════════════════════════════

Criterio C1 — Apertura del modal de Previsualización tras confirmar el modal de Generación
Dado que el usuario presiona ""Generar Factura"" en el modal de Generación,
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal de Previsualización mostrando el PDF de la Factura tal como saldrá al cliente, con todos los datos integrados. En este momento la Factura aún NO se ha timbrado ante el PAC.

Criterio C2 — Acciones del modal: Cancelar y Generar Factura
Dado que el usuario revisa el PDF previsualizado,
Cuando finaliza la revisión,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el flujo, cierra el modal sin timbrar) y Generar Factura (procede con el timbrado real ante el PAC SAT).

═══════════════════════════════════════════════════════════════
SECCIÓN D — MODAL DE ALERTA SAT (errores de validación o timbrado)
═══════════════════════════════════════════════════════════════

Criterio D1 — Aparición del modal de Alerta ante error
Dado que el sistema intenta timbrar la Factura y recibe respuesta de error del PAC SAT (errores de validación previa o errores del propio servicio de timbrado),
Cuando el error se identifica,
Entonces deberá mostrar el modal de Alerta SAT con la descripción específica del error (ejemplo: ""El código postal del cliente no coincide con el registrado en el SAT. Verifica y actualiza el código postal para poder timbrar"").

Criterio D2 — Acción del modal: Continuar
Dado que el modal de Alerta está abierto,
Cuando el usuario lo cierra,
Entonces el sistema deberá cerrar el modal con la acción ""Continuar"" y devolver al usuario al estado previo (típicamente el listado del Detalle). El sistema NO ofrece edición directa de los datos del cliente desde este modal: el usuario debe ir al módulo correspondiente (Catálogo de Clientes) a corregir y reintentar la generación desde el principio.

═══════════════════════════════════════════════════════════════
SECCIÓN E — MODAL DE ÉXITO DE GENERACIÓN (timbrado exitoso)
═══════════════════════════════════════════════════════════════

Criterio E1 — Loader durante el timbrado
Dado que el usuario confirmó el timbrado desde el modal de Previsualización,
Cuando el sistema envía la solicitud de timbrado al PAC y espera respuesta,
Entonces deberá mostrar un indicador de carga (loader) con mensaje informativo (ejemplo: ""Su solicitud está siendo atendida, por favor espere..."").

Criterio E2 — Modal de confirmación de éxito
Dado que el timbrado se completa exitosamente,
Cuando el PAC retorna la confirmación,
Entonces el sistema deberá mostrar el modal de confirmación de éxito (ejemplo: ""¡Has generado una factura Exitosamente!"") y persistir la Factura en base de datos. El pedido cambia de estado en el listado: ahora muestra acción ""Enviar Factura"" (verde) en lugar de ""Generar Factura"" (azul).

Criterio E3 — Persistencia inmediata al timbrado exitoso
Dado que el timbrado fue exitoso,
Cuando el sistema procesa la respuesta del PAC,
Entonces deberá persistir el artefacto fiscal completo (XML timbrado y PDF) en base de datos como registro permanente inmutable. La Factura no puede modificarse después de este momento.

═══════════════════════════════════════════════════════════════
SECCIÓN F — MODAL DE ENVÍO DE FACTURA
═══════════════════════════════════════════════════════════════

Criterio F1 — Apertura del modal de Envío al presionar ""Enviar Factura""
Dado que un pedido tiene Factura ya timbrada en estado pendiente enviar y el usuario presiona ""Enviar Factura"",
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal de Envío con los campos pre-rellenados.

Criterio F2 — Contacto destinatario pre-rellenado y editable
Dado que el modal de Envío está abierto,
Cuando el sistema muestra el campo destinatario,
Entonces deberá pre-rellenar el correo del contacto del cliente. El campo es editable por el usuario si fuese necesario modificar el destinatario.

Criterio F3 — CC editable con default ESAC
Dado el modal de envío, cuando el sistema renderiza el campo CC, entonces deberá pre-rellenarlo con el ESAC asignado al cliente/pedido como default, manteniéndolo editable por el usuario.

Criterio F4 — Asunto pre-rellenado con folios
Dado que el modal de Envío está abierto,
Cuando el sistema muestra el campo Asunto,
Entonces deberá pre-rellenar el asunto en un formato que incluya el folio de la Factura y el folio del Pedido Interno (formato canónico definido para correos de Factura de México). ** Para Perú, la lógica del asunto podría ser análoga, pero la numeración del folio fiscal es distinta (serie SUNAT F### + correlativo) y el formato final queda pendiente de confirmar con PROQUIFA y su asesor peruano. **

Criterio F5 — Adjuntos automáticos PDF y XML
Dado que el modal de Envío está abierto,
Cuando el sistema muestra los adjuntos,
Entonces deberá incluir automáticamente los archivos PDF y XML de la Factura timbrada. Los adjuntos NO son removibles por el usuario.

Criterio F6 — Notas extras editables
Dado el modal de envío, cuando el sistema renderiza el text area de notas extras, entonces deberá ser editable por el usuario para capturar texto adicional libre opcional.

Criterio F7 — Acciones del modal: Cancelar y Enviar
Dado que el usuario revisa el modal de Envío,
Cuando finaliza la edición,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el envío, cierra el modal; la Factura permanece timbrada y persistida con estado pendiente enviar) y Enviar (procede con el envío del correo al cliente).

═══════════════════════════════════════════════════════════════
SECCIÓN G — MODAL DE ÉXITO DE ENVÍO Y SALIDA OPERATIVA
═══════════════════════════════════════════════════════════════

Criterio G1 — Confirmación de envío exitoso
Dado que el sistema confirma que el correo de la Factura fue enviado exitosamente al cliente,
Cuando el envío se completa,
Entonces deberá mostrar el modal de confirmación de éxito (ejemplo: ""¡Has enviado una factura Exitosamente!"").

Criterio G2 — Salida del pedido del listado
Dado que el envío fue exitoso,
Cuando el sistema actualiza el estado del pedido,
Entonces el pedido deberá desaparecer del listado del Detalle del cliente. Si era el último pedido del cliente, el cliente puede salir también del listado agrupado de la pantalla inicial del módulo.

Criterio G3 — Salida operativa diferenciada por tipo de pedido
Dado que la Factura fue enviada exitosamente,
Cuando el sistema procesa la salida operativa,
Entonces deberá actuar diferenciadamente:
- Si el pedido es Crédito: la información de la Factura se transfiere a Legacy. El pedido continúa por el flujo Crédito en Legacy. NO se genera pendiente en Validar Cobro.
- Si el pedido es Prepago: el sistema genera el pendiente correspondiente en el módulo Validar Cobro asociado a esta Factura, para que el equipo de Cobranza valide el pago del cliente.

═══════════════════════════════════════════════════════════════
SECCIÓN H — ORDEN DEL FLUJO COMPLETO
═══════════════════════════════════════════════════════════════

Criterio H1 — Orden canónico del flujo
Dado que el usuario inicia el flujo completo de generar y enviar una Factura,
Cuando avanza paso a paso,
Entonces deberá seguirse el siguiente orden:
1. Usuario navega al Detalle del cliente y presiona ""Generar Factura"" en un pedido.
2. Sistema abre el modal de Generación con los datos del pedido y del cliente.
3. Usuario revisa, edita Uso CFDI si requiere, y presiona ""Generar Factura"" del modal.
4. Sistema abre el modal de Previsualización del PDF.
5. Usuario revisa el PDF previsualizado y presiona ""Generar Factura"" del modal de Previsualización.
6. Sistema envía la solicitud al PAC. Si hay error: muestra modal de Alerta SAT y termina aquí (el usuario corrige y vuelve a empezar). Si es exitoso: continúa.
7. Sistema muestra modal de loader durante el timbrado y luego modal de éxito de generación.
8. Sistema persiste la Factura en base de datos. El pedido cambia de estado a pendiente enviar.
9. Usuario regresa al listado y presiona ""Enviar Factura"" en el pedido.
10. Sistema abre el modal de Envío con datos pre-rellenados.
11. Usuario revisa, notas al correo si requiere, y presiona ""Enviar"".
12. Sistema envía el correo al cliente con adjuntos PDF y XML.
13. Sistema muestra modal de éxito de envío.
14. Pedido sale del listado. Salida operativa según tipo (Legacy para Crédito; pendiente en Validar Cobro para Prepago)."								"- Esta fila documenta la pantalla de Detalle por cliente del módulo Factura por Adelantado, junto con todos los modales del flujo: Generación de Factura, Previsualización del PDF, Alerta SAT, Éxito de generación, Envío de Factura y Éxito de envío. La pantalla inicial del módulo (listado agrupado por cliente) se documenta en requisito independiente.
- El módulo Factura por Adelantado es funcionalidad NUEVA en PQF2 R16. La incorporación de Factura por Adelantado atiende casos comerciales donde el cliente requiere la factura antes del ingreso de la mercancía.
- Aplica exclusivamente a pedidos SIN productos clasificados como Sustancias Controladas (tipo Mundial, Nacional u Origen). Los pedidos Prepago con controlados se atienden vía Factura Anticipo desde Validar Cobro (artefacto y flujo independientes). Los pedidos Crédito con controlados no son elegibles para Factura por Adelantado.
- Aplica a pedidos Crédito SIN controlados y Prepago SIN controlados. La diferenciación post-envío:
  - Crédito → transferencia a Legacy.
  - Prepago → pendiente en Validar Cobro.
- La generación de la Factura se realiza en DOS pasos: (1) modal de revisión de datos fiscales del cliente y del emisor con único editable Uso CFDI; (2) modal de previsualización del PDF antes del timbrado real ante el PAC SAT.
- El PAC utilizado por PROQUIFA es TurboPac (operado por Quadrum Tecnologías S.A. de C.V., RFC QSO100827UB0). El detalle técnico de la integración con el PAC se documenta en requisito independiente.
- El detalle del contenido y estructura del PDF de la Factura (qué información va, en qué secciones, branding por empresa emisora, etc.) se documenta en requisito independiente, análogo al del PDF de Proforma.
- Datos fiscales clave forzados por normativa SAT y no modificables:
  - Método de Pago = ""PPD - Pago en parcialidades o diferido"".
  - Forma de Pago = ""99 - Por definir"" (la Forma de Pago real se captura en el Complemento de Pago del módulo Validar Cobro).
  - Tipo de Comprobante = ""I - Ingreso"".
  - Tipo de Cambio = el del día de generación (regla SAT confirmada por el cliente, en factura no se permite modificar el TC).
- El folio de la Factura es un consecutivo numérico independiente POR empresa emisora (varchar 6). Esto contrasta con el folio de la Proforma que es global lineal para todo el grupo. Cada empresa del grupo PROQUIFA México (Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica) mantiene su propio contador.
- A diferencia de la Proforma (que se persiste al envío exitoso del correo), la Factura se persiste al timbrarse exitosamente ante el PAC. Una vez timbrada, la Factura es artefacto fiscal inmutable que no puede modificarse.
- Los errores más comunes reportados por PROQUIFA en el proceso de facturación electrónica actual y que deben manejarse en el modal de Alerta SAT son: (a) cambio de Código Postal del cliente respecto al registrado en el SAT (requiere corrección en el Catálogo de Clientes); (b) errores en decimales al calcular totales de la factura.
- El modal de Alerta SAT NO ofrece edición directa de los datos del cliente. El usuario debe corregir en el Catálogo de Clientes y reintentar la generación desde el principio. Esto preserva fuente única de verdad para los datos del cliente.
- El modal de Envío incluye copia automática a ESAC por regla canónica del proyecto.
- El asunto del correo de envío se pre-rellena con un formato canónico que incluye folio de la Factura y folio del Pedido Interno (México). ** Para Perú, el formato del asunto y la numeración del folio fiscal (serie SUNAT F### + correlativo, distinta del folio mexicano) quedan pendientes de confirmar con PROQUIFA y su asesor peruano cuando se habilite la región. **
- La sección Cliente del módulo (cabecera del Detalle) muestra etiquetas de clasificación del cliente que son datos preexistentes del Catálogo de Clientes y solo lectura desde este módulo.
- Cobertura geográfica de esta fila: clientes con Región México exclusivamente. La habilitación para Perú depende de la integración con OSE/SUNAT autorizado (brecha mayor del proyecto) y se documenta en requisito independiente cuando esté disponible.
- Origen del lote en la Factura por Adelantado (resuelto por el cliente): la Factura por Adelantado se timbra sin lote ni pedimento. Como se emite antes del surtido del pedido (la asignación de lote en almacén ocurre después de cobrar y facturar), el lote no está disponible al timbrar y no se consigna, de la misma forma que el pedimento se representa con ""N/A"". Se descartaron las opciones de reservar lote anticipadamente y de ajustar posteriormente con una Nota de Crédito.
- ** Pendiente confirmar la política ante indisponibilidad del PAC TurboPac (reintento automático, encolamiento, fallback). **
- ** Pendiente resolver formalmente la denominación canónica del rol operativo entre ""Gestor de Cobranza"" y ""Analista de Cuentas por Cobrar"". **"									
TPSC-RE-FU-023		Validar Cobro	Validar Cobro	Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver) **, quiero contar con la pantalla principal de Validar Cobro que liste los clientes de mi cartera con pendientes y me ofrezca la acción adecuada según su estado (realizar cobros o gestionar cobranza), para priorizar y dar seguimiento al cobro de cada cliente desde un único punto de entrada.	El sistema debe contar con una pantalla principal del módulo Validar Cobro que muestre el listado de clientes de la cartera del usuario que tengan pendientes (cobros recibidos pendientes de aplicar o proformas/facturas pendientes de cobrar). Por cada cliente, el sistema ofrece una acción contextual según su estado: cuando existen cobros recibidos pendientes de aplicar, conduce al wizard de tres pasos (Captura → Asociación → Facturación y Envío); cuando no hay cobros recibidos, abre la gestión de cobranza del cliente. La pantalla es estructuralmente la misma para Región México y Región Perú, con visibilidad filtrada por la cartera asignada al usuario, que opera clientes de una sola región. Las diferencias operativas entre regiones se manifiestan en las pantallas internas del wizard, documentadas en requisitos independientes.			Propuesto					R16.2M-RE-FU-002	"## Aplica a
- Pantalla principal del módulo Validar Cobro: listado de clientes con pendientes de cobranza.
- La pantalla aplica tanto a Región México como a Región Perú con estructura idéntica. La cartera del Cobrador es por región: cada usuario opera clientes de UNA sola región. El listado de cada usuario muestra únicamente clientes de su región y de su cartera (no se mezclan regiones en una misma vista).
- Listado agrupado por cliente con conteos de cobros recibidos pendientes de aplicar y de proformas/facturas pendientes de cobrar.
- Buscador único por nombre de cliente o identificador fiscal (RFC para México, RUC para Perú).
- Acción contextual por cliente según estado:
  - ""Realizar Cobros"": cuando el cliente tiene uno o más cobros recibidos pendientes de aplicar (correos del Buzón de Cobros del cliente).
  - ""Gestionar Cobranza"": cuando el cliente NO tiene cobros recibidos (cero cobros) pero sí tiene proformas/facturas pendientes de cobrar.
- Modal ""Gestionar Cobranza"" con listado de pedidos pendientes del cliente, datos de contacto, fecha estimada de pago editable por pedido y opción de cancelación de pedido por falta de pago.
- Visibilidad filtrada por cartera del usuario (campo Cobrador del Catálogo de Clientes) combinado con la región operativa del usuario.
- Tooltip explicativo sobre el estado del cliente cuando aplica (por ejemplo: cliente sin cobros recibidos en el Buzón).
- Salida hacia el wizard de tres pasos (Captura del Cobro → Asociación → Facturación y Envío) al presionar ""Realizar Cobros"".

## No aplica a
- Lógica de cancelación efectiva del pedido en Legacy o en el sistema de cumplimiento. La cancelación desde este módulo solo dispara el cambio de estado y la salida del pedido del listado de Validar Cobro."	"## Reglas de Negocio

Regla 1 — Aplicabilidad a ambas regiones con cartera por región
La pantalla principal del módulo Validar Cobro muestra exclusivamente clientes de la región del usuario activo (México o Perú), según la asignación de cartera. La estructura del listado, el buscador y los botones contextuales son idénticos para ambas regiones, pero un usuario individual no ve clientes de regiones distintas a la suya. Las diferencias operativas entre regiones se manifiestan posteriormente al entrar al wizard del cliente.

Regla 2 — Visibilidad filtrada por cartera y región del usuario
El listado muestra únicamente clientes asignados a la cartera del usuario (campo Cobrador del Catálogo de Clientes) y cuya Región del cliente coincida con la región del usuario. Los clientes asignados a otros usuarios o de regiones distintas no son visibles ni accesibles desde el listado.

Regla 3 — Inclusión de clientes con pendientes
El listado incluye clientes que cumplan al menos una de estas condiciones: tienen uno o más cobros recibidos en el Buzón de Cobros pendientes de aplicar a alguna proforma o factura del cliente; o tienen una o más proformas o facturas emitidas pendientes de cobrar (con saldo pendiente positivo). Los clientes sin pendientes no aparecen en el listado.

Regla 4 — Acción contextual según cobros recibidos
La acción de cada cliente en el listado es ""Realizar Cobros"" si el cliente tiene uno o más cobros recibidos en el Buzón de Cobros pendientes de aplicar (al presionar, el usuario navega al wizard de tres pasos del cliente), o ""Gestionar Cobranza"" si el cliente no tiene cobros recibidos pendientes (cero cobros en el Buzón) pero sí tiene proformas/facturas por cobrar (al presionar, se abre el modal ""Gestionar Cobranza"").

Regla 5 — Tooltip explicativo sobre estado sin cobros
Cuando un cliente tiene cero cobros recibidos en el Buzón, el listado muestra un tooltip explicativo en el conteo de cobros recibidos (ejemplo: ""No hay correos de cobro en el buzón de este cliente"") para que el usuario comprenda por qué la acción es ""Gestionar Cobranza"" en lugar de ""Realizar Cobros"".

Regla 6 — Buscador único por nombre de cliente o identificador fiscal
El buscador del listado filtra en tiempo real según coincidencias del texto en el nombre del cliente o en el identificador fiscal (RFC para México, RUC para Perú). El filtrado opera sin requerir botón de búsqueda. El buscador ignora los espacios al inicio y al final del texto ingresado (se recortan antes de filtrar), de modo que un texto con espacios sobrantes arroja los mismos resultados que el texto sin ellos.

Regla 7 — Modal ""Gestionar Cobranza"" al presionar la acción
Al presionar ""Gestionar Cobranza"" en un cliente, se abre el modal del cliente con: cabecera con nombre del cliente y Monto Total pendiente; listado de pedidos pendientes con datos por pedido (Pedido Interno, número de orden de compra o referencia del cliente, datos de contacto del cliente, fecha estimada de pago editable, botón Cancelar Pedido por pedido); y un botón ""Confirmar"" que guarda los cambios de fechas estimadas y cierra el modal.

Regla 8 — Edición de fecha estimada de pago
La fecha estimada de pago de un pedido se registra al presionar ""Confirmar"" en el modal ""Gestionar Cobranza"" y queda visible en el pedido para consulta posterior y en reportes operativos del módulo. Esta fecha es referencia operativa del equipo de Cobranza para seguimiento al cliente y no genera bloqueos automáticos en el sistema. Cada modificación de la fecha estimada de pago se registra en un histórico de cambios que conserva el valor anterior, el valor nuevo, el usuario que realizó el cambio y la fecha y hora (timestamp) en que se efectuó. El registro de este histórico está dentro del alcance de R16. ** Pendiente definir con el cliente si el histórico requiere una referencia visual / visualización en pantalla para el usuario o si se maneja únicamente en base de datos. **

Regla 9 — Cancelación de pedido por falta de pago desde Gestionar Cobranza
Al presionar ""Cancelar Pedido"" en un pedido específico del modal ""Gestionar Cobranza"", el sistema cancela el pedido por falta de pago: el pedido sale del listado de Validar Cobro y queda con estado ""Cancelado por falta de pago"" para trazabilidad histórica. La cancelación es decisión manual del operador y no está condicionada por el sistema a vigencias automáticas (la vigencia de proforma 30 días y de factura mes corriente es referencia operativa, no bloqueo técnico). La cancelación opera siempre a nivel de estado, nunca como eliminación en base de datos: el folio de pedido se cancela y no es reutilizable. La reversa fiscal depende del documento asociado: si el pedido tiene una factura emitida (timbrada), la cancelación detona además la cancelación fiscal de la factura ante el SAT; si el pedido solo tiene proforma, no aplica cancelación fiscal por no ser un documento fiscal, pero igualmente se cancela el folio de pedido existente. Este comportamiento aplica a la cancelación manual desde Gestionar Cobranza; el marcado del pedido por inconsistencia ""Pago Incompleto Vencido"" (ver TPSC-RE-FU-026/027) sigue su propio flujo y no está incluido en este comportamiento.

Regla 10 — Navegación al wizard al presionar ""Realizar Cobros""
Al presionar ""Realizar Cobros"" en un cliente, el sistema navega a la pantalla del wizard de tres pasos del cliente seleccionado (Paso 1 - Captura del Cobro como pantalla inicial). No es un modal: es una pantalla nueva con la cabecera del cliente y la barra de pasos del wizard.

## Riesgos

Riesgo 2 — Solapamiento de denominación de rol
La denominación del rol que opera el módulo aparece como ""Gestor de Cobranza"" en la matriz del cliente y como ""Analista de Cuentas por Cobrar"" en sesiones de revisión de pantallas. ** Pendiente resolver formalmente la denominación canónica antes del desarrollo. **

Riesgo 3 — Cliente Perú sin Buzón de Cobros equivalente operativo
Para clientes Perú, el Buzón de Cobros y su flujo de recepción de correos de cobro depende de configuración operativa específica (cuentas bancarias Perú, formato de comprobantes de cobro peruanos, etc.) que tiene brechas pendientes. Si la operación Perú no tiene un Buzón de Cobros poblado, los clientes Perú aparecerán siempre con cero cobros recibidos y solo podrá usarse ""Gestionar Cobranza"" hasta que se resuelvan las brechas de cobro Perú.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — ESTRUCTURA DEL LISTADO
═══════════════════════════════════════════════════════════════

Criterio A1 — Columnas del listado
Dado que el usuario accede al módulo Validar Cobro,
Cuando el sistema renderiza el listado de clientes,
Entonces deberá mostrar las siguientes columnas por cliente:
- Cliente (nombre del cliente).
- Identificador fiscal (RFC del cliente para Región México, RUC para Región Perú).
- Cobros recibidos (conteo de correos de cobro del Buzón pendientes de aplicar para este cliente).
- Factura / Proforma por Cobrar (conteo de proformas y facturas emitidas pendientes de cobrar para este cliente).
- Saldo Pendiente (monto total pendiente de cobro). Siempre se muestra dolarizado en USD: cuando los pedidos del cliente están en distintas monedas, cada monto se convierte a USD y se suman, de modo que el listado homogeneiza la comparación entre clientes en una sola moneda. Cada monto se convierte a USD con el tipo de cambio de su propio documento origen (no con el tipo de cambio del día de la consulta ni uno unificado); no existe un tipo de cambio único del listado.
- Acción contextual (botón ""Realizar Cobros"" o ""Gestionar Cobranza"" según estado).

Criterio A2 — Acción contextual visible por cliente
Dado que un cliente del listado tiene uno o más cobros recibidos pendientes de aplicar,
Cuando el sistema renderiza la acción del cliente,
Entonces deberá mostrar el botón ""Realizar Cobros"" (estilizado para destacar acción primaria del flujo).

Criterio A3 — Acción contextual cuando no hay cobros recibidos
Dado que un cliente del listado tiene cero cobros recibidos pendientes,
Cuando el sistema renderiza la acción del cliente,
Entonces deberá mostrar el botón ""Gestionar Cobranza"" en lugar de ""Realizar Cobros"".

Criterio A4 — Tooltip explicativo sobre cero cobros recibidos
Dado que un cliente tiene cero cobros recibidos pendientes,
Cuando el usuario consulta el conteo de cobros recibidos del cliente,
Entonces el sistema deberá mostrar un tooltip explicativo (ejemplo: ""No hay correos de cobro en el buzón de este cliente"").

Criterio A5 — Ordenamiento por defecto del listado
Dado que el usuario accede al módulo Validar Cobro,
Cuando el sistema renderiza el listado de clientes,
Entonces deberá ordenarlo por defecto según la antigüedad de los cobros recibidos pendientes de validar, mostrando primero los clientes cuyo cobro más antiguo lleva más tiempo sin validarse. Este ordenamiento atiende el SLA operativo de 72 horas con que cuenta el usuario para validar cada pago recibido; el vencimiento de dicho plazo no dispara ningún comportamiento automático en el sistema (alerta, marcado o escalamiento), es referencia operativa.

═══════════════════════════════════════════════════════════════
SECCIÓN B — BUSCADOR ÚNICO
═══════════════════════════════════════════════════════════════

Criterio B1 — Buscador en tiempo real
Dado que el usuario interactúa con el campo buscador del listado,
Cuando ingresa texto,
Entonces el sistema deberá filtrar el listado de clientes en tiempo real conforme el usuario escribe, según coincidencias con nombre de cliente o identificador fiscal (RFC o RUC según región del cliente). El filtrado opera sin requerir botón de búsqueda e ignora los espacios al inicio y al final del texto ingresado.

Criterio B2 — Sin filtros adicionales
Dado que el usuario opera el listado,
Cuando consulta los filtros disponibles,
Entonces el listado deberá ofrecer únicamente el buscador único. No hay filtros adicionales por estado, región, monto u otros criterios (decisión de diseño operativo: el listado es ligero y el usuario opera sobre los clientes que aparecen). ** Pendiente confirmar el orden por defecto del listado (alfabético por cliente, por antigüedad de cobros recibidos, por mayor saldo pendiente, u otro). **

═══════════════════════════════════════════════════════════════
SECCIÓN C — VISIBILIDAD POR CARTERA Y REGIÓN
═══════════════════════════════════════════════════════════════

Criterio C1 — Filtro por Cobrador asignado y región del usuario
Dado que el usuario accede al módulo,
Cuando el sistema arma el listado,
Entonces deberá incluir únicamente clientes que cumplan AMBAS condiciones: (a) el Cobrador asignado en el Catálogo de Clientes corresponde al usuario activo, y (b) la Región del cliente coincide con la región del usuario. Clientes asignados a otros Cobradores o de regiones distintas no aparecen en el listado.

═══════════════════════════════════════════════════════════════
SECCIÓN D — ACCIÓN ""REALIZAR COBROS"" Y NAVEGACIÓN AL WIZARD
═══════════════════════════════════════════════════════════════

Criterio D1 — Navegación al wizard al presionar ""Realizar Cobros""
Dado que el usuario presiona ""Realizar Cobros"" en un cliente del listado,
Cuando el sistema procesa la acción,
Entonces deberá navegar a la pantalla del wizard de tres pasos del cliente seleccionado, abriendo directamente el Paso 1 (Captura del Cobro). La navegación es a pantalla nueva (NO es un modal sobre la pantalla actual).

═══════════════════════════════════════════════════════════════
SECCIÓN E — MODAL ""GESTIONAR COBRANZA""
═══════════════════════════════════════════════════════════════

Criterio E1 — Apertura del modal al presionar ""Gestionar Cobranza""
Dado que el usuario presiona ""Gestionar Cobranza"" en un cliente del listado,
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal ""Gestionar Cobranza"" sobre la pantalla del listado.

Criterio E2 — Cabecera del modal
Dado que el modal ""Gestionar Cobranza"" está abierto,
Cuando el sistema arma la cabecera,
Entonces deberá mostrar: nombre del cliente y Monto Total pendiente del cliente (con moneda).

Criterio E3 — Listado de pedidos del cliente en el modal
Dado que el modal está abierto,
Cuando el sistema arma el listado de pedidos pendientes del cliente,
Entonces deberá mostrar por cada pedido:
- Pedido Interno.
- Referencia del pedido del cliente (número de orden de compra u otro identificador del cliente, si está capturado).
- Datos del contacto del cliente: nombre, correo electrónico, teléfono.
- Fecha estimada de pago (datepicker editable).
- Botón ""Cancelar Pedido"" (acción de cancelación directa de este pedido específico).

Criterio E4 — Edición de fecha estimada de pago y confirmación
Dado que el usuario modifica la fecha estimada de pago de uno o más pedidos en el modal,
Cuando presiona ""Confirmar"" abajo del modal,
Entonces el sistema deberá guardar las fechas estimadas actualizadas en los pedidos correspondientes, registrar en el histórico de cada pedido el cambio de fecha estimada (valor anterior, valor nuevo, usuario y fecha/hora) y cerrar el modal. ** Pendiente definir si el histórico de la fecha estimada se presenta visualmente al usuario o se maneja únicamente en base de datos. **

Criterio E5 — Cancelación de pedido por falta de pago
Dado que el usuario presiona ""Cancelar Pedido"" en un pedido del modal,
Cuando el sistema procesa la acción,
Entonces deberá cancelar el pedido por falta de pago: el pedido cambia a estado ""Cancelado por falta de pago"" y sale del listado de Validar Cobro y del modal ""Gestionar Cobranza"" del cliente. La cancelación queda registrada con trazabilidad de quién la ejecutó y cuándo. La cancelación opera a nivel de estado, no como eliminación: el folio de pedido se cancela y no es reutilizable. La reversa fiscal depende del documento asociado: si el pedido tiene factura emitida (timbrada), la cancelación detona la cancelación fiscal de la factura ante el SAT; si solo tiene proforma, no aplica cancelación fiscal por no ser documento fiscal, pero igualmente se cancela el folio de pedido. ** Pendiente confirmar si además propaga cancelación y/o transferencias a otros sistemas (Legacy). **

Criterio E6 — Cancelación sin restricción de vigencia automática
Dado que un pedido aparece en el modal ""Gestionar Cobranza"",
Cuando el usuario presiona ""Cancelar Pedido"" sobre ese pedido,
Entonces el sistema deberá permitir la cancelación sin restricción automática por vigencia (proforma 30 días, factura mes corriente). La vigencia es referencia operativa para el usuario, no bloqueo técnico del sistema. La decisión de cancelación es del operador.

Criterio E7 — Cierre del modal sin guardar cambios
Dado que el usuario abrió el modal y realizó cambios pero NO presionó ""Confirmar"",
Cuando cierra el modal con la X del modal o navega fuera,
Entonces el sistema deberá descartar los cambios de fecha estimada no confirmados. Las cancelaciones de pedido individuales son acciones inmediatas (no requieren ""Confirmar"" del modal)."								"- Esta fila documenta la pantalla principal del módulo Validar Cobro: listado de clientes con pendientes, buscador, botones contextuales y modal ""Gestionar Cobranza"".
- La pantalla principal aplica tanto a Región México como a Región Perú con estructura idéntica (mismas columnas, mismo buscador, mismos botones contextuales). Sin embargo, la cartera del Cobrador es por región: cada usuario opera clientes de UNA sola región (la que tenga asignada en su configuración como Cobrador). Por lo tanto, el listado que ve cada usuario muestra exclusivamente clientes de su región y de su cartera; NO se mezclan clientes México y Perú en una misma vista. Las diferencias regionales operativas (timbrado SAT vs SUNAT, monedas MXN/PEN, formato de identificador fiscal RFC/RUC, etc.) se manifiestan al entrar al wizard del cliente.
- El botón ""Realizar Cobros"" conduce al wizard de tres pasos (pantalla nueva, no modal): Paso 1 - Captura del Cobro, Paso 2 - Asociación a Proforma/Factura, Paso 3 - Facturación y Envío.
- El botón ""Gestionar Cobranza"" abre un modal sobre el listado con la información operativa del cliente y sus pedidos pendientes, permitiendo registrar fechas estimadas de pago y cancelar pedidos por falta de pago.
- La cancelación de pedido por falta de pago desde ""Gestionar Cobranza"" es la vía operativa preventiva (cliente que aún no ha pagado nada). La cancelación reactiva (cliente que pagó pero el pago es insuficiente o no cumple) se gestiona dentro del wizard como inconsistencia del cobro (no aplica en esta pantalla).
- La fecha estimada de pago capturada en el modal es referencia operativa del equipo de Cobranza para seguimiento al cliente. NO genera bloqueos automáticos en el sistema.
- La vigencia de proforma (30 días) y de factura (mes corriente) confirmada por el cliente para cancelaciones por falta de pago es referencia operativa, no bloqueo técnico del sistema. El operador puede cancelar antes o después según criterio.
- El buscador único filtra por nombre de cliente o identificador fiscal (RFC en México, RUC en Perú) en tiempo real. No hay filtros adicionales por estado, monto, región u otros criterios.
- ** Pendiente resolver formalmente la denominación canónica del rol operativo entre ""Gestor de Cobranza"" y ""Analista de Cuentas por Cobrar"". **
- ** Pendiente confirmar si la cancelación de pedido desde ""Gestionar Cobranza"" dispara cancelación de la proforma o factura asociada y si propaga cancelación y/o transferencias a otros sistemas (Legacy). **
- Moneda del listado de clientes (resuelto por el cliente): el Saldo Pendiente del listado siempre se muestra dolarizado en USD, para homogeneizar la comparación entre clientes. No se muestra en la moneda de facturación de cada cliente. El tipo de cambio aplicado para convertir a USD es el de cada documento origen (cada documento se dolariza con su propio tipo de cambio y los montos se suman), no el del día de la consulta ni uno unificado.
- Ordenamiento por defecto del listado (resuelto por el cliente): el listado de clientes se ordena por antigüedad de los cobros recibidos (los cobros más antiguos sin validar primero), bajo el entendido operativo de que el usuario cuenta con un SLA de 72 horas para validar cada pago recibido.
- ** Pendiente confirmar si la fecha estimada de pago registra historial de cambios con usuario y timestamp para trazabilidad de seguimiento. **
- ** Para clientes Región Perú, el correcto funcionamiento de esta pantalla depende del Buzón de Cobros Perú (con sus brechas pendientes de modelo bancario peruano y formatos de comprobantes). Si el Buzón Perú no está poblado, los clientes Perú siempre aparecerán con cero cobros recibidos hasta que se resuelvan las brechas correspondientes. **"									
TPSC-RE-FU-024		Validar Cobro: Paso 1 México	Validar Cobro	Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver) **, quiero contar con la primera pantalla del wizard de Validar Cobro (Paso 1 - Captura del Cobro) para revisar los correos del Buzón de Cobros del cliente y capturar los datos del cobro recibido o marcar sus inconsistencias, para registrar formalmente cada cobro y prepararlo para su asociación en el Paso 2.	El sistema debe contar con la primera pantalla del wizard de Validar Cobro (Paso 1 - Captura del Cobro) para clientes con Región México, donde el usuario revisa el correo y el comprobante de pago recibidos y captura los datos del cobro. El sistema contempla asistencia automatizada propuesta para el llenado del formulario a partir del análisis del correo y el comprobante, funcionalidad cuyo alcance técnico queda pendiente de definición **. Un cobro capturado y confirmado no puede editarse posteriormente. La estructura UI de esta pantalla se reutiliza idénticamente para clientes Región Perú; las diferencias entre regiones son exclusivamente los catálogos de opciones y se documentan en requisito independiente.			Propuesto					R16.2M-RE-FU-002	"## Aplica a
- Pantalla del Paso 1 del wizard de Validar Cobro: Captura del Cobro.
- Aplica a clientes con Región México exclusivamente (los catálogos y la operación Perú con la misma UI se documentan en requisito independiente).
- Cabecera del cliente (estructura consistente con Factura por Adelantado): logo del cliente, razón social, etiquetas preexistentes (BAJO, REST, etc.), RFC/RUC, razón social legal, moneda de facturación.
- Barra de pasos del wizard visible en cabecera: 1-CAPTURAR COBRO (activo), 2-ASOCIAR FACTURA/PROFORMA, 3-FACTURACIÓN Y ENVÍO.
- Listado de cobros del cliente: items capturados arriba (ordenados por Fecha del Cobro ascendente) e items sin capturar abajo (ordenados por fecha de llegada del correo al Buzón, ascendente).
- Detalle del correo seleccionado: asunto, cuerpo del correo, hora, fecha y adjuntos seleccionables con radio buttons (uno de los adjuntos se marca como comprobante de pago oficial).
- Formulario de Datos del Cobro: folio del cobro (formato COB-mmddaa-consecutivo donde la fecha corresponde al día efectivo del cobro), monto recibido, fecha del cobro, forma de pago (catálogo SAT c_FormaPago), cuenta origen (texto libre), cuenta destino (Catálogo de Cuentas Bancarias del grupo PROQUIFA México), moneda del cobro, tipo de cambio del día (vs moneda de facturación), notas del cobro.
- Asistencia automatizada propuesta (no comprometida): el sistema propondría el auto-completado de los campos del formulario a partir del análisis del correo y del comprobante de pago seleccionado. ** Alcance técnico pendiente de definición. **
- Selección obligatoria del comprobante de pago entre los adjuntos del correo antes de poder continuar.
- Captura de múltiples cobros del cliente en la misma sesión del Paso 1 antes de avanzar al Paso 2.
- Auto-guardado del Paso 1 para preservar lo capturado si el usuario sale o navega entre pantallas.
- Inmutabilidad del cobro una vez confirmado: no se puede editar posteriormente (alerta de confirmación antes del guardado definitivo).
- Marcado de inconsistencias del cobro mediante modal: tipo de inconsistencia (combo del catálogo de tipos de inconsistencia) y comentario opcional.
- Las inconsistencias del Paso 1 aplican únicamente al cobro como tal (datos incompletos, comprobante inválido, formato incorrecto, etc.). Las inconsistencias relativas a la relación entre el cobro y una proforma o factura (como ""Pago Incompleto Vencido"") se documentan en el Paso 2 cuando ya hay contexto del documento a cobrar.
- Navegación: Cancelar (regresa al listado principal de Validar Cobro) o Continuar (avanza al Paso 2 con los cobros capturados).

## No aplica a
- Wizard de Validar Cobro para Región Perú: se documenta en requisito independiente con la misma estructura UI pero catálogos específicos de Perú (Forma de Pago SUNAT, cuentas destino Golocaer Perú, fuente del TC SBS, etc.).
- Catálogo de Tipos de Inconsistencia (definición de las opciones del combo): ** pendiente del lado de PROQUIFA (Tesorería). **
- Detalle técnico de la asistencia automatizada propuesta (modelo de IA, motor de extracción de datos, integraciones técnicas): ** pendiente de definición, no está comprometido como alcance R16. **
- Edición posterior de un cobro ya capturado y confirmado: NO se permite."	"## Reglas de Negocio

Regla 1 — Aplicabilidad solo a Región México
El Paso 1 del wizard de Validar Cobro opera exclusivamente sobre clientes con Región México. Los clientes con Región Perú son atendidos por el wizard equivalente Perú, con la misma UI pero catálogos específicos de Perú (requisito independiente).

Regla 2 — Cabecera del cliente
La cabecera del Paso 1 muestra los datos del cliente seleccionado: logo (si existe), Alias, etiquetas de clasificación preexistentes del Catálogo de Clientes, identificador fiscal bajo la etiqueta unificada ""RFC/RUC"" (con el valor correspondiente según la región del cliente), razón social legal completa y moneda de facturación.

Regla 3 — Listado de cobros con identificación y orden según estado de captura
Cada item del listado de cobros del Paso 1 se muestra así: el item sin capturar (pre-captura) presenta un identificador temporal ""COB-N"" (consecutivo simple por sesión/cliente, ej. COB-1, COB-2, COB-3) sin datos adicionales hasta que se capture el cobro; el item capturado presenta folio definitivo ""COB-mmddaa-NNNN"" (donde mmddaa es la fecha efectiva del cobro), Fecha del cobro y ""Monto del cobro"" con su moneda. Si tras la asociación del Paso 2 quedó residual no aplicado, la etiqueta del monto se actualiza a ""Saldo a favor"". El orden del listado coloca primero los items capturados (ordenados por Fecha del Cobro de la más antigua a la más reciente) y al final los items sin capturar (ordenados por fecha de llegada del correo al Buzón, de la más antigua a la más reciente).

Regla 4 — Detalle del correo seleccionado
Al seleccionar un correo del listado, el detalle muestra: asunto del correo, cuerpo del correo, hora, fecha y los adjuntos del correo. Los adjuntos se presentan como opciones seleccionables (radio buttons) para identificar cuál corresponde al comprobante de pago oficial del cliente.

Regla 5 — Selección obligatoria del comprobante de pago
Para continuar al Paso 2, el usuario debe haber seleccionado uno de los adjuntos del correo como comprobante de pago. Si no hay selección, el sistema bloquea la continuación y muestra el error correspondiente.

Regla 6 — Datos del formulario del cobro
El formulario del cobro ofrece los siguientes campos: Folio del cobro (COB-mmddaa-consecutivo, generado por el sistema con la fecha efectiva del cobro); Monto recibido (obligatorio); Fecha del cobro (obligatorio, datepicker, día efectivo del cobro); Forma de pago (obligatorio, combo del catálogo SAT c_FormaPago: ejemplo ""01 - Efectivo"", ""02 - Cheque nominativo"", ""03 - Transferencia electrónica de fondos"", etc.); Cuenta origen (obligatorio, texto libre, es la cuenta del cliente); Cuenta destino (obligatorio, combo del Catálogo de Cuentas Bancarias del grupo PROQUIFA México); Moneda del cobro (obligatorio, combo); Tipo de Cambio (calculado automáticamente con el TC del día vs la moneda de facturación del cliente, no modificable por el usuario); Notas del cobro (opcional, texto libre).

Regla 7 — Tipo de cambio del día capturado con el cobro
Cuando la moneda del cobro y/o la moneda de facturación del cliente involucran una moneda distinta a MXN, el sistema calcula automáticamente el TC del día actual de esa moneda no-MXN respecto a MXN: si el cobro es en MXN con cliente de facturación en moneda extranjera, captura el TC de la moneda de facturación vs MXN; si el cobro es en moneda extranjera, captura el TC de la moneda del cobro vs MXN; si el cobro es en MXN con cliente de facturación en MXN, el TC es N/A. El valor es solo lectura, no modificable. Este TC capturado se conserva con el cobro y se utiliza en el Paso 2 (conversiones operativas) y en el Paso 3 (TipoCambio del CFDI fiscal cuando aplique).

Regla 8 — Asistencia automatizada propuesta para captura del cobro
El sistema ofrece asistencia automatizada propuesta (no comprometida) para proponer valores de auto-completado de los campos del formulario (monto, fecha, forma de pago, cuenta origen, moneda) extraídos del contenido del correo y del comprobante de pago seleccionado. El usuario siempre tiene la última palabra sobre los valores capturados; los datos sugeridos son editables antes de confirmar. ** El alcance técnico de esta asistencia (modelo de IA, motor de extracción, integraciones, precisión esperada, comportamiento ante baja confianza) queda pendiente de definición y no está comprometido como alcance R16. **

Regla 9 — Captura de múltiples cobros antes de avanzar
Cuando el cliente tiene múltiples correos pendientes de capturar, tras completar la captura de un cobro el usuario puede seleccionar otro correo del listado y capturar el siguiente cobro sin necesidad de avanzar al Paso 2. El Paso 2 se activa cuando el usuario explícitamente presiona ""Continuar"".

Regla 10 — Avance al Paso 2 con o sin captura nueva
Al presionar ""Continuar"" en el Paso 1, el sistema permite el avance al Paso 2 si existe al menos un cobro registrado (capturado en esta sesión o auto-guardado de sesión previa); no se requiere capturar un cobro nuevo en la sesión actual si ya hay cobros previos disponibles. El sistema bloquea el avance si no existe ningún cobro registrado para el cliente.

Regla 11 — Auto-guardado del Paso 1
Cuando el usuario avanza entre correos del listado, sale de la pantalla o navega a otra parte del sistema, el sistema auto-guarda el estado del Paso 1 (cobros capturados, selecciones de comprobantes, valores del formulario actual) para preservar el progreso. No existe botón de ""Guardar"" manual; el guardado es transparente al usuario. Si el usuario detiene la operación por cualquier motivo y posteriormente regresa al Paso 1 del mismo cliente, el sistema lo reanuda en el punto donde se quedó con el estado preservado, sin obligarlo a reiniciar la captura desde el inicio.

Regla 12 — Inmutabilidad del cobro confirmado
Un cobro capturado y confirmado no puede editarse posteriormente. Antes de confirmar el guardado definitivo del cobro, el sistema muestra una alerta de confirmación indicando que los datos no podrán modificarse después.

Regla 13 — Marcado de inconsistencia mediante modal
Al presionar ""Marcar Inconsistencia en Cobro"", el sistema abre el modal ""Inconsistencia de Pago"" con dos campos: Tipo de Inconsistencia (combo del catálogo de tipos de inconsistencia) y Comentario adicional (opcional, texto libre para describir detalle adicional al cliente). Los botones del modal son Cancelar y Confirmar Inconsistencia.

Regla 14 — Tipos de inconsistencia aplicables al Paso 1 (cobro como tal)
Al marcar una inconsistencia en el Paso 1, el sistema la registra contra el cobro como tal (sin contexto de pedido o documento por cobrar, ya que el cobro aún no se ha asociado a ninguna proforma o factura). Los tipos aplicables al Paso 1 corresponden a problemas intrínsecos del cobro recibido: datos incompletos, comprobante inválido, formato incorrecto, monto ilegible, etc. Las inconsistencias derivadas de la relación entre el cobro y un documento a cobrar (por ejemplo, ""Pago Incompleto Vencido"", ""Monto Incorrecto vs Proforma"") se detectan en el Paso 2 (Asociación), no aquí. ** Pendiente definir el catálogo completo de tipos de inconsistencia aplicables al Paso 1 (cobro como tal) del lado de PROQUIFA (Tesorería). **

Regla 15 — Navegación: Cancelar y Continuar
El Paso 1 ofrece dos acciones: Cancelar (botón al pie de la pantalla) regresa al listado principal de Validar Cobro, dejando el estado del Paso 1 auto-guardado para que el usuario pueda regresar posteriormente; Continuar avanza al Paso 2 (Asociación de Factura/Proforma) con todos los cobros capturados disponibles.

## Riesgos

Riesgo 1 — Catálogo de Tipos de Inconsistencia del Paso 1 pendiente
El catálogo de tipos de inconsistencia aplicables al Paso 1 (cobro como tal) está pendiente de definición por parte de PROQUIFA (Tesorería). Los tipos aplicables al Paso 1 corresponden a problemas intrínsecos del cobro recibido (datos incompletos, comprobante inválido, formato incorrecto, monto ilegible, etc.). El tipo ""Pago Incompleto Vencido"" no aplica al Paso 1; se marca en el Paso 2 cuando ya hay contexto del documento a cobrar. ** Pendiente solicitar el catálogo completo al cliente antes del desarrollo. **

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL CLIENTE Y BARRA DE PASOS
═══════════════════════════════════════════════════════════════

Criterio A1 — Cabecera del cliente
Dado que el usuario entra al Paso 1 del wizard desde ""Realizar Cobros"",
Cuando el sistema renderiza la cabecera del cliente,
Entonces deberá mostrar: logo del cliente (si existe), razón social, etiquetas preexistentes del cliente (clasificación), identificador fiscal bajo la etiqueta unificada ""RFC/RUC"" (con el valor correspondiente al cliente según su región), razón social legal completa y moneda de facturación.

Criterio A2 — Barra de pasos del wizard
Dado que el usuario está en el wizard,
Cuando el sistema renderiza la barra de pasos,
Entonces deberá mostrar los tres pasos del wizard: ""1 - CAPTURAR COBRO"" (activo), ""2 - ASOCIAR FACTURA/PROFORMA"", ""3 - FACTURACIÓN Y ENVÍO"".

═══════════════════════════════════════════════════════════════
SECCIÓN B — LISTADO DE COBROS DEL BUZÓN
═══════════════════════════════════════════════════════════════

Criterio B1 — Identificación del item en el listado según estado de captura
Dado el listado del Paso 1,
Cuando el sistema renderiza cada item,
Entonces si el cobro no ha sido capturado el item muestra únicamente ""COB-N"" (consecutivo simple pre-captura por sesión/cliente, ej. COB-1, COB-2); y si el cobro ya fue capturado el item muestra folio definitivo ""COB-mmddaa-NNNN"", Fecha del cobro y ""Monto del cobro"" con moneda. Si en el Paso 2 quedó residual no aplicado, la etiqueta del monto cambia a ""Saldo a favor"".

Criterio B2 — Orden mixto del listado según estado de captura
Dado el listado del Paso 1,
Cuando el sistema lo presenta al usuario,
Entonces deberá ordenarlo en dos bloques visualmente continuos: primero los items capturados (ordenados por Fecha del Cobro de la más antigua a la más reciente) y después los items sin capturar (COB-N, ordenados por fecha de llegada del correo al Buzón de la más antigua a la más reciente). Al confirmar la captura de un item, éste se desplaza al bloque de capturados según su Fecha del Cobro.

Criterio B3 — Modo lectura del item capturado
Dado un item del listado en estado capturado,
Cuando el sistema lo renderiza,
Entonces deberá mostrarlo en modo lectura solamente. El sistema no deberá ofrecer acción de edición sobre los datos del cobro capturado (consistente con la regla de inmutabilidad post-confirmación).

═══════════════════════════════════════════════════════════════
SECCIÓN C — DETALLE DEL CORREO Y SELECCIÓN DEL COMPROBANTE
═══════════════════════════════════════════════════════════════

Criterio C1 — Datos del correo seleccionado
Dado que el usuario selecciona un correo del listado,
Cuando el sistema renderiza el detalle,
Entonces deberá mostrar: contacto del cliente (nombre, correo electrónico, teléfono), asunto del correo, cuerpo del correo, fecha, hora y los adjuntos del correo.

Criterio C2 — Adjuntos seleccionables como comprobante de pago
Dado que el correo tiene uno o más adjuntos,
Cuando el sistema renderiza la sección de adjuntos,
Entonces deberá presentarlos como opciones seleccionables (radio buttons) para que el usuario identifique cuál corresponde al comprobante de pago oficial. El nombre del archivo de cada adjunto es visible para que el usuario decida.

Criterio C3 — Selección obligatoria del comprobante
Dado que el usuario intenta continuar al Paso 2,
Cuando el sistema valida la selección,
Entonces deberá bloquear la continuación si el usuario no ha seleccionado uno de los adjuntos como comprobante de pago.

═══════════════════════════════════════════════════════════════
SECCIÓN D — FORMULARIO DE DATOS DEL COBRO
═══════════════════════════════════════════════════════════════

Criterio D1 — Folio del cobro
Dado que el sistema renderiza el formulario,
Cuando muestra el folio,
Entonces deberá mostrar el folio formato COB-mmddaa-consecutivo, donde la fecha se construye con el día efectivo del cobro (campo Fecha del cobro del formulario). El folio se genera al confirmar la captura del cobro. ** Pendiente confirmar si el folio del cobro es por región (consecutivo independiente para México y Perú) o global. **

Criterio D2 — Monto recibido
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un campo numérico obligatorio para el monto recibido del cliente.

Criterio D3 — Fecha del cobro
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un datepicker obligatorio. La fecha capturada corresponde al día efectivo en que el cliente realizó el pago (no la fecha de llegada del correo al Buzón).

Criterio D4 — Forma de pago
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un combo obligatorio del catálogo SAT c_FormaPago (ejemplo de opciones: ""01 - Efectivo"", ""02 - Cheque nominativo"", ""03 - Transferencia electrónica de fondos"", etc.). No aplica la restricción ""99 - Por definir"" usada en facturas PPD; aquí se captura la forma efectiva del pago realizado.

Criterio D5 — Cuenta origen
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un campo de texto libre obligatorio para la cuenta origen del pago (cuenta bancaria del cliente desde la cual se realizó el pago).

Criterio D6 — Cuenta destino
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un combo obligatorio con las cuentas bancarias del Catálogo de Cuentas Bancarias del grupo PROQUIFA México (las cuentas operativas del grupo destinadas a recibir cobros).

Criterio D7 — Moneda del cobro
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un combo obligatorio para que el usuario seleccione la moneda en la que el cliente realizó el cobro.

Criterio D8 — Tipo de cambio del día capturado con el cobro
Dado que el usuario selecciona la moneda del cobro,
Cuando el sistema renderiza el campo Tipo de Cambio,
Entonces deberá comportarse según el caso: si la moneda del cobro es MXN y la moneda de facturación del cliente es MXN, el campo se muestra como N/A; si la moneda del cobro es MXN y la moneda de facturación del cliente es distinta a MXN, el sistema calcula automáticamente el TC del día de la moneda de facturación del cliente respecto a MXN, capturado anticipadamente para conversiones operativas en el Paso 2 con documentos en esa moneda; si la moneda del cobro es distinta a MXN, el sistema calcula automáticamente el TC del día de la moneda del cobro respecto a MXN. El valor del TC es solo lectura, no modificable por el usuario. El TC capturado se conserva con el cobro y se utiliza posteriormente en el Paso 2 (conversiones operativas) y en el Paso 3 (TipoCambio del CFDI fiscal cuando aplique). ** Pendiente confirmar la fuente oficial del TC (propuesta: TC FIX Banxico publicado en DOF). **

Criterio D9 — Notas del cobro
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un campo de texto libre opcional para notas adicionales del cobro.

═══════════════════════════════════════════════════════════════
SECCIÓN E — ASISTENCIA AUTOMATIZADA PROPUESTA
═══════════════════════════════════════════════════════════════

Criterio E1 — Propuesta de auto-completado del formulario
Dado que el usuario seleccionó un comprobante de pago del correo,
Cuando el sistema procesa la información del correo y del comprobante,
Entonces deberá ofrecer asistencia automatizada propuesta (no comprometida) para auto-completar los campos del formulario: monto recibido, fecha del cobro, forma de pago, cuenta origen, moneda. ** El alcance técnico de esta asistencia (motor de extracción, modelo IA, precisión esperada, integraciones) queda pendiente de definición y no está comprometido como alcance R16. **

Criterio E2 — Edición de los datos sugeridos
Dado que el sistema propuso valores de auto-completado,
Cuando el usuario revisa los datos,
Entonces deberá poder editar libremente cualquiera de los valores antes de confirmar. La asistencia es propuesta, no obligatoria. El usuario tiene la última palabra sobre los datos capturados.

═══════════════════════════════════════════════════════════════
SECCIÓN F — AUTO-GUARDADO E INMUTABILIDAD
═══════════════════════════════════════════════════════════════

Criterio F1 — Auto-guardado del Paso 1
Dado que el usuario captura datos en el formulario,
Cuando avanza entre correos del listado, sale de la pantalla o navega el sistema,
Entonces el sistema deberá auto-guardar el estado del Paso 1 (cobros capturados, selecciones de comprobantes, valores del formulario actual). No existe botón ""Guardar"" manual; el guardado es transparente. Al regresar al Paso 1 del mismo cliente tras una interrupción, el sistema deberá reanudar en el punto donde el usuario se quedó con el estado preservado, sin reiniciar el flujo.

Criterio F2 — Captura de múltiples cobros antes de avanzar
Dado que el cliente tiene varios correos pendientes,
Cuando el usuario finaliza la captura de un cobro,
Entonces el sistema deberá permitir al usuario seleccionar otro correo del listado y capturar el siguiente cobro. El Paso 2 se activa solo al presionar ""Continuar"".

Criterio F3 — Alerta de confirmación antes de guardar definitivamente un cobro
Dado que el usuario termina la captura de un cobro,
Cuando indica que desea confirmar,
Entonces el sistema deberá mostrar una alerta de confirmación indicando que los datos del cobro no podrán modificarse después. El usuario debe confirmar explícitamente para que el cobro se guarde como inmutable.

Criterio F4 — Inmutabilidad del cobro confirmado
Dado que un cobro fue capturado y confirmado por el usuario,
Cuando el usuario intenta editarlo posteriormente,
Entonces el sistema no deberá permitir la edición.

═══════════════════════════════════════════════════════════════
SECCIÓN G — MODAL DE INCONSISTENCIA DE PAGO
═══════════════════════════════════════════════════════════════

Criterio G1 — Apertura del modal
Dado que el usuario detecta inconsistencias en el cobro,
Cuando presiona ""Marcar Inconsistencia en Cobro"" en el pie de la pantalla,
Entonces el sistema deberá abrir el modal ""Inconsistencia de Pago"".

Criterio G2 — Campos del modal
Dado que el modal está abierto,
Cuando el sistema renderiza los campos,
Entonces deberá ofrecer: Tipo de Inconsistencia (obligatorio, combo del catálogo de tipos de inconsistencia aplicables al Paso 1) y Comentario adicional (opcional, texto libre para describir detalle adicional al cliente). ** El catálogo del Paso 1 está pendiente de definición por PROQUIFA. Los tipos aplicables al Paso 1 son inconsistencias intrínsecas del cobro (datos incompletos, comprobante inválido, formato incorrecto, etc.); no incluye ""Pago Incompleto Vencido"" que se marca en el Paso 2. **

Criterio G3 — Acciones del modal
Dado que el modal está abierto,
Cuando el usuario finaliza la captura,
Entonces deberá ofrecer dos acciones: Cancelar (cierra el modal sin guardar la inconsistencia) y Confirmar Inconsistencia (registra la inconsistencia en el cobro).

Criterio G4 — Alcance de las inconsistencias del Paso 1
Dado que el usuario marca una inconsistencia en el Paso 1,
Cuando confirma la inconsistencia,
Entonces el sistema deberá registrar la inconsistencia contra el cobro como tal. Las inconsistencias del Paso 1 no incluyen el caso ""Pago Incompleto Vencido"" ni otras inconsistencias que requieran conocer el documento contra el que se aplicaría el cobro (proforma o factura); esas inconsistencias se marcan en el Paso 2 (Asociación) cuando ya hay contexto del documento a cobrar. ** El catálogo específico de tipos de inconsistencia aplicables en cada paso queda pendiente de definición por PROQUIFA (Tesorería). **

═══════════════════════════════════════════════════════════════
SECCIÓN H — NAVEGACIÓN DEL PASO 1
═══════════════════════════════════════════════════════════════

Criterio H1 — Botón Cancelar
Dado que el usuario opera el Paso 1,
Cuando presiona ""Cancelar"" al pie de la pantalla,
Entonces el sistema deberá regresar al listado principal de Validar Cobro. El estado del Paso 1 queda auto-guardado para permitir retomar la captura posteriormente.

Criterio H2 — Botón Continuar: condiciones de habilitación
Dado el Paso 1,
Cuando el sistema evalúa el botón ""Continuar"",
Entonces si existe al menos un cobro registrado (capturado en esta sesión o auto-guardado de sesión previa) el botón está habilitado y al presionarlo avanza al Paso 2 con todos los cobros disponibles para asociación; si no existe ningún cobro registrado para el cliente el botón está deshabilitado. La captura nueva en la sesión actual no es obligatoria si ya hay cobros previos capturados."								"- Esta fila documenta el Paso 1 del wizard de Validar Cobro (Captura del Cobro) para Región México exclusivamente. La estructura UI de la pantalla se reutiliza idénticamente para clientes Región Perú; las únicas diferencias son los catálogos de opciones (Forma de Pago SAT vs SUNAT, cuentas bancarias grupo PROQUIFA México vs Golocaer Perú, fuente del TC, etc.) y se documentan en requisito independiente.
- El wizard se invoca desde la pantalla principal de Validar Cobro (Listado de clientes con pendientes) al presionar ""Realizar Cobros"" en un cliente. Es una pantalla nueva, no modal.
- La cabecera del cliente sigue la estructura consistente del proyecto (logo, Alias, etiquetas preexistentes de clasificación, identificador fiscal bajo etiqueta unificada ""RFC/RUC"", razón social legal, moneda de facturación), análoga a la cabecera de Factura por Adelantado.
- El listado de correos del Buzón de Cobros del cliente se ordena del más antiguo al más reciente, por fecha de llegada al Buzón.
- El folio del cobro tiene formato COB-mmddaa-consecutivo (confirmado por PMO #40), donde la fecha corresponde al día efectivo del cobro (no a la fecha de llegada del correo al Buzón). Esto genera una ambigüedad operativa en cómo identificar el correo en el listado antes de capturar el cobro (cuando aún no se conoce la fecha efectiva). ** Decisión pendiente. **
- Selección obligatoria del comprobante de pago: el correo del Buzón puede traer múltiples adjuntos (comprobante real + otros archivos como notas o referencias); el usuario debe identificar explícitamente cuál es el comprobante oficial mediante radio button.
- Forma de pago del cobro: catálogo SAT c_FormaPago (no aplica la restricción ""99 - Por definir"" que rige facturas PPD; aquí se captura la forma efectiva del pago realizado).
- Cuenta origen: texto libre (cuenta del cliente desde donde pagó).
- Cuenta destino: combo del Catálogo de Cuentas Bancarias del grupo PROQUIFA México (mismo catálogo usado en Proforma y Factura).
- Tipo de Cambio: del día, de la moneda no-MXN involucrada respecto a MXN (moneda base SAT, requisito del CFDI). El sistema decide qué moneda capturar según el caso: la moneda del cobro si es distinta a MXN, o la moneda de facturación del cliente si el cobro es MXN pero el cliente factura en otra moneda. Si ambas son MXN, el campo se muestra como N/A. El TC capturado sirve como referencia única para las conversiones del Paso 2 y para el TipoCambio del CFDI en el Paso 3. La fuente estándar fiscal mexicana es el TC FIX publicado por Banxico en el DOF.
- ** DUDA ABIERTA — Moneda base del TC capturado: actualmente se utiliza MXN como moneda base de todas las conversiones (consistente con la regla SAT del CFDI). Pendiente validar con asesor fiscal y PROQUIFA si esta es la opción correcta o si en algún escenario operativo conviene capturar el TC en sentido distinto (por ejemplo, vs moneda de facturación del cliente). **
- Asistencia automatizada propuesta: funcionalidad exploratoria para auto-completar los datos del formulario a partir del correo y el comprobante. No está comprometida como alcance R16; el alcance técnico queda pendiente de definición. El usuario siempre tiene la última palabra sobre los valores capturados.
- Auto-guardado del Paso 1: el progreso se preserva si el usuario sale o navega entre pantallas. No existe botón ""Guardar"" manual; el guardado es transparente al usuario.
- Inmutabilidad del cobro una vez confirmado: una vez que el usuario confirma la captura de un cobro (con alerta previa de confirmación), el cobro no puede editarse posteriormente.
- Captura de múltiples cobros antes de avanzar: el usuario puede capturar varios cobros del cliente en la misma sesión del Paso 1 sin avanzar al Paso 2 hasta que decida explícitamente.
- Modal ""Inconsistencia de Pago"" del Paso 1: permite marcar inconsistencias intrínsecas del cobro (datos incompletos, comprobante inválido, formato incorrecto, monto ilegible, etc.). Tipo de inconsistencia (combo) y comentario opcional. El catálogo de tipos del Paso 1 está pendiente de definición por PROQUIFA (Tesorería).
- Las inconsistencias del Paso 1 aplican únicamente al cobro como tal. Las inconsistencias derivadas de la relación entre el cobro y un documento a cobrar (proforma o factura), como ""Pago Incompleto Vencido"", se marcan en el Paso 2 (Asociación), no aquí.
- ** Pendiente confirmar si el folio del cobro (COB-mmddaa-consecutivo) es por región (consecutivo independiente para México y Perú) o global del grupo. **
- ** Pendiente clarificar la identificación visual del correo en el listado antes de la captura (cuando aún no se conoce la fecha efectiva del cobro). Propuestas: usar la fecha de llegada del correo, mostrar identificador temporal del Buzón, generar el folio definitivo solo al confirmar la captura, asignar folio provisional, u otra alternativa. **
- ** Pendiente definir el catálogo completo de Tipos de Inconsistencia aplicables al Paso 1 del lado de PROQUIFA (Tesorería). Tipos aplicables al Paso 1: inconsistencias intrínsecas del cobro (datos incompletos, comprobante inválido, formato incorrecto, etc.). El tipo ""Pago Incompleto Vencido"" no aplica al Paso 1 (se marca en el Paso 2 cuando ya hay contexto del documento a cobrar). **
- ** Pendiente definir el alcance técnico de la asistencia automatizada propuesta para auto-completar el formulario (modelo de IA o motor de extracción, integración, precisión esperada, comportamiento ante baja confianza). No comprometido como alcance R16. **
- ** Pendiente confirmar la fuente oficial del Tipo de Cambio del día para México (propuesta estándar fiscal: TC FIX publicado por Banxico en el DOF). Confirmar si PROQUIFA utiliza esta fuente o tiene una fuente propia. **
- ** Pendiente resolver formalmente la denominación canónica del rol operativo entre ""Gestor de Cobranza"" y ""Analista de Cuentas por Cobrar"" (transversal). **"									
TPSC-RE-FU-028		Validar Cobro: Paso 3 México	Validar Cobro	Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver) **, quiero contar con la tercera pantalla del wizard de Validar Cobro (Paso 3 - Facturación y Envío) para previsualizar, timbrar y enviar los documentos fiscales de cada línea de la asociación, para cerrar el ciclo operativo de cobranza con todos los artefactos fiscales y operativos del cliente entregados.	El sistema debe contar con la tercera pantalla del wizard de Validar Cobro (Paso 3 - Facturación y Envío) para clientes con Región México, donde el sistema determina por cada documento de la asociación el tipo de documento fiscal a generar (Factura, Factura Anticipo o Complemento de Pago) y el usuario previsualiza, timbra y envía cada uno de forma individual. Al enviar cada documento en operaciones México, el sistema dispara automáticamente el establecimiento de la Fecha Estimada de Entrega del pedido, la transferencia a Legacy y la generación de las Confirmaciones de Pedido. El usuario debe completar todas las líneas del cliente antes de cerrar el ciclo. La estructura UI de esta pantalla se reutiliza para Perú con diferencias importantes y se documenta en requisito independiente.			Propuesto					R16.2M-RE-FU-002	"## Aplica a
- Pantalla del Paso 3 del wizard de Validar Cobro: Facturación y Envío.
- Aplica a clientes con Región México exclusivamente (Perú se documenta en requisito independiente con diferencias significativas).
- Cabecera del cliente (estructura consistente con Paso 1 y Paso 2: logo, Alias, etiquetas preexistentes de clasificación, RFC/RUC, razón social legal, moneda de facturación).
- Barra de pasos del wizard: 1-CAPTURAR COBRO (✓), 2-ASOCIAR FACTURA/PROFORMA (✓), 3-FACTURACIÓN Y ENVÍO (activo).
- Listado de líneas a procesar, una por cada documento de la asociación cerrada en el Paso 2.
- Lógica condicional por línea según el tipo de documento origen del Paso 2:
  - Proforma normal (sin productos controlados) → genera Factura nueva (CFDI Ingreso).
  - Proforma con productos controlados → genera Factura Anticipo (CFDI Ingreso con tipo de relación 07 SAT — Aplicación de Anticipo).
  - Factura existente (de Prepago con Factura por Adelantado previa) con cobro asociado → genera Complemento de Pago (CFDI Pagos 2.0).
- Por línea, edición de Uso CFDI (combo del catálogo SAT c_UsoCFDI).
- Por línea, edición de Método de Pago únicamente para líneas que parten de proforma (PPD o PUE). Líneas de Complemento de Pago tienen Método de Pago fijo en PPD (no editable, conforme normativa SAT).
- Inclusión automática de las Notas de Crédito aplicadas en el Paso 2 dentro del nodo CFDIRelacionados del XML a timbrar (con UUID y monto correspondiente de cada NC, tipo de relación 01 o 07 según el caso).
- Estados por línea: Pendiente (estado inicial) → Factura Generada / Factura Anticipo Generada / Complemento Generado (después del timbrado exitoso) → Enviado (después del envío al cliente).
- Acciones por línea: Previsualizar PDF (modal con previsualización antes de timbrar), Generar/Timbrar (timbrado con PAC TurboPac), Enviar (modal de envío al cliente).
- Operación INDIVIDUAL por línea: no existen acciones masivas (no ""Timbrar todo"", no ""Enviar todo""). El usuario procesa una línea a la vez.
- Modal Previsualizar: muestra el PDF representativo del documento a timbrar para validación visual del usuario antes del timbrado.
- Modal éxito de generación: tras timbrado exitoso muestra confirmación con folio fiscal (UUID) timbrado.
- Modal Enviar: permite al usuario disparar el envío del documento al cliente.
- Destinatario del envío: el contacto del cliente que envió el pedido (heredado del flujo de tramitación del pedido).
- Cuerpo del correo de envío para facturas: misma plantilla que el correo de envío de Factura por Adelantado.
- Cuerpo del correo de envío para complementos de pago: ** Pendiente confirmar la plantilla; propuesta inicial ""<Folio Pedido Interno> - <Folio Factura>"" como asunto, cuerpo por definir. **
- Al ENVIAR cada documento, el sistema dispara automáticamente las acciones post-envío (solo México):
  - Establecimiento de la Fecha Estimada de Entrega (FEE) del pedido asociado.
  - Transferencia del pedido y documentos generados (factura/anticipo/complemento, NCs aplicadas, info del cobro) a Legacy.
  - Generación de la Confirmación de Pedido (documento que el cliente confirma, análogo al existente hoy en ProquifaNet para crédito, ahora extendido a prepago en R16).
- Auto-guardado del Paso 3 consistente con Paso 1 y Paso 2.
- Persistencia del estado del Paso 3: si el usuario sale del wizard antes de finalizar todas las líneas, al volver a entrar al mismo cliente desde Validar Cobro el sistema lo regresa directamente al Paso 3 (no se reinicia el wizard) hasta que todas las líneas estén en estado Enviado.
- Inmutabilidad post-timbrado: una vez una línea está en estado Factura Generada / Factura Anticipo Generada / Complemento Generado, no se permite re-timbrar ni editar el documento. La única vía de modificación post-timbrado fiscal es vía el módulo Notas de Crédito (módulo independiente, fuera del Paso 3 de Validar Cobro).
- Manejo de errores del PAC TurboPac (timbrado fallido, PAC indisponible, datos inválidos, errores SAT): mismo comportamiento operativo que en el módulo Factura por Adelantado (mensaje de error al usuario con detalle del problema).
- Cierre del wizard: el wizard se considera cerrado para el cliente cuando todas las líneas del Paso 3 estén en estado Enviado.

## No aplica a
- Wizard de Validar Cobro para Región Perú: se documenta en requisito independiente con diferencias significativas.
- Cancelación fiscal de documentos ya timbrados en el Paso 3 (vía CFDI de cancelación SAT): NO está contemplada en R16 dentro del Paso 3 ni en el módulo Validar Cobro.
- Generación masiva (Timbrar todo / Enviar todo): NO contemplada. La operación es individual por línea.
- Edición del Método de Pago en líneas de Complemento de Pago: NO permitida (PPD fijo conforme normativa SAT).
- Re-timbrado de un documento ya generado en el Paso 3: NO permitido. La línea queda inmutable post-timbrado."	"## Reglas de Negocio

Regla 1 — Aplicabilidad solo a Región México
El Paso 3 del wizard de Validar Cobro opera exclusivamente sobre clientes con Región México. Los clientes con Región Perú son atendidos por el wizard equivalente Perú, con diferencias significativas en catálogos y reglas (requisito independiente).

Regla 2 — Líneas a procesar derivadas del Paso 2
El sistema genera una línea por cada documento (proforma o factura) asociado en el Paso 2 que requiera emisión de un documento fiscal. Cada línea referencia: tipo de documento origen (Proforma normal, Proforma con controlados, Factura existente), folio del documento origen, Pedido Interno, empresa emisora del grupo, cobros asociados, y NCs aplicadas (si las hubo).

Regla 3 — Lógica condicional del tipo de documento fiscal a generar
El tipo de documento fiscal a generar por línea depende del tipo de documento origen: si la línea parte de una Proforma normal (sin productos controlados), se genera una Factura nueva (CFDI Ingreso); si parte de una Proforma con productos controlados, se genera una Factura Anticipo (CFDI Ingreso con tipo de relación 07 SAT — Aplicación de Anticipo); si parte de una Factura existente (de Prepago con Factura por Adelantado previo) con cobro asociado, se genera un Complemento de Pago (CFDI Pagos 2.0) que se relaciona al UUID de la factura existente. ** Pendiente confirmar el uso del tipo de relación 07 para la Factura Anticipo de proforma con controlados. **

Regla 4 — Comportamiento de la columna Uso CFDI según tipo de línea
La columna Uso CFDI es única y su comportamiento depende del tipo de documento de la línea. En líneas que generan Factura nueva o Factura Anticipo (parten de proforma), es un combo del catálogo SAT c_UsoCFDI (ejemplo: ""G01 - Adquisición de mercancías"", ""G03 - Gastos en general"", ""P01 - Por definir"", etc.) editable por el usuario antes del timbrado, con valor por defecto tomado del Uso CFDI configurado del cliente o capturado en el pedido original. En líneas que generan Complemento de Pago (la factura ya existe), la columna muestra en solo lectura el Uso CFDI con el que ya se timbró la factura origen, sin combo ni edición.

Regla 5 — Edición del Método de Pago según tipo de documento
El comportamiento del campo Método de Pago depende del tipo de documento a generar: si la línea genera Factura nueva o Factura Anticipo (parte de proforma), el Método de Pago es editable mediante radio button con dos opciones — PPD (Pago en parcialidades o diferido) o PUE (Pago en una sola exhibición); si la línea genera Complemento de Pago (parte de factura existente), el Método de Pago es siempre PPD no editable (conforme normativa SAT, los complementos solo aplican a CFDIs con método PPD).

Regla 6 — Inclusión de Notas de Crédito en el XML al timbrar
Cuando el usuario aplicó una o más NCs a la línea en el Paso 2, el sistema incluye las NCs aplicadas dentro del nodo CFDIRelacionados con: UUID de cada NC, monto correspondiente aplicado, y tipo de relación SAT (01 - Nota de crédito de los documentos relacionados, o 07 - CFDI por aplicación de anticipo, según el caso fiscal). Esto cumple con la normativa SAT (Apéndice 5 Anexo 20 versión 4.0).

Regla 7 — Flujo operativo por línea: previsualizar, timbrar, enviar
El flujo operativo por línea es: Previsualizar PDF (muestra el PDF representativo del documento que se generará, permite validar visualmente datos antes de timbrar); Generar/Timbrar (dispara el timbrado del documento con el PAC TurboPac: si la línea es Factura PUE desde proforma, timbra 1 CFDI; si es Factura PPD desde proforma, timbra en cascada 2 CFDIs — Factura PPD + Complemento de Pago asociado al cobro confirmado; si es Complemento de Pago desde factura existente, timbra 1 CFDI; si el timbrado es exitoso, la línea pasa al estado Generado correspondiente); y Enviar (abre el modal de envío con destinatario, CC al ESAC, asunto generado por sistema, adjuntos —PDF y XML de cada CFDI de la línea más Confirmación de Pedido del Pedido Prepago— y text area de notas extras opcional).

Regla 8 — Estados por línea
Los estados posibles de cada línea son: Pendiente (estado inicial al entrar al Paso 3, aún no se ha timbrado ni enviado); Generado (después del timbrado exitoso de los CFDIs que correspondan a la línea, 1 o 2 según aplique; los documentos están timbrados con UUID SAT pero aún no se enviaron al cliente); y Enviado (después del envío exitoso al cliente con todos los CFDIs adjuntos más Confirmación de Pedido; la línea está cerrada operativamente). Si el usuario timbra pero no envía en la misma sesión, la línea queda en estado intermedio Generado hasta el envío.

Regla 9 — Operación INDIVIDUAL por línea
No existe operación masiva en el Paso 3. La generación es individual (una línea a la vez) y el envío es individual (una línea a la vez). No hay botón ""Timbrar todo"" ni ""Enviar todo"".

Regla 10 — Generación en cascada según Método de Pago en líneas con proforma origen
Al confirmar el timbrado de una línea con proforma origen, el sistema actúa según el Método de Pago elegido: si es PUE, timbra únicamente la Factura PUE (1 CFDI), sin generar Complemento de Pago (conforme normativa SAT, las facturas PUE son fiscalmente autocontenidas); si es PPD, timbra primero la Factura PPD y, en cascada inmediata tras timbrado exitoso, timbra el Complemento de Pago asociado al cobro confirmado en el wizard (2 CFDIs).

Regla 11 — Acciones automáticas al enviar (solo México)
Cuando el usuario presiona Enviar en una línea y el envío es exitoso, el sistema dispara automáticamente (aplica únicamente a operaciones México): establecer la Fecha Estimada de Entrega (FEE) del pedido asociado al documento enviado, y transferir el pedido y los documentos generados (Factura, Complemento de Pago, NCs aplicadas, info del cobro) al sistema Legacy. La Confirmación de Pedido se genera al enviar (en el Paso 3 de Validar Cobro) y se adjunta en el mismo correo de envío de la línea. No se previsualiza: solo se genera y se muestra como archivo adjunto en el modal de envío. No existe candado bloqueante.

Regla 12 — Destinatarios del envío (Para y CC)
Al armar el modal de envío, el sistema determina los destinatarios: Para es el contacto del cliente del pedido (heredado del flujo de tramitación, editable por el usuario en el modal); CC es el ESAC asignado al cliente/pedido (sugerido por el sistema, editable por el usuario en el modal). ** Pendiente confirmar el comportamiento si el contacto del pedido no está disponible o si hay múltiples contactos del cliente. **

Regla 13 — Modal Previsualizar
Al presionar ""Previsualizar"", el sistema abre un modal con la previsualización del PDF representativo del documento a generar (incluye todos los datos que tendrá la factura/anticipo/complemento). El usuario puede cerrar el modal sin acción, regresar a editar campos del Paso 3, o proceder a Timbrar.

Regla 14 — Modal de éxito tras timbrado
Cuando el timbrado de una línea es exitoso y el PAC TurboPac responde con UUID timbrado, el sistema muestra un modal de éxito con el folio fiscal (UUID) del documento timbrado y confirmación de generación. La línea pasa al estado correspondiente (Generada).

Regla 15 — Modal Enviar
Cuando la línea está en estado Generado y el usuario presiona ""Enviar"", el sistema abre el modal de envío con: Destinatario (Para) con el contacto del cliente del pedido (editable, default heredado del flujo de tramitación); Copia (CC) con el ESAC asignado (editable, default sugerido por el sistema); Asunto generado automáticamente según plantilla por tipo de documento de la línea (no editable); Adjuntos con el PDF y XML de cada CFDI timbrado en la línea (1 o 2 según aplique) más la Confirmación de Pedido del Pedido Prepago (no editables); y Notas extras, text area editable por el usuario para capturar texto adicional libre opcional.

Regla 16 — Persistencia del Paso 3 y navegación atrás según estado de timbrado
Si el usuario sale del wizard antes de finalizar todas las líneas del Paso 3, al regresar al módulo Validar Cobro y seleccionar al mismo cliente el sistema lo redirige directamente al Paso 3 con el estado actual preservado (líneas pendientes, generadas, enviadas). La posibilidad de regresar a los Pasos 1 o 2 depende del estado de timbrado de cada línea: SÍ se permite regresar mientras la línea NO se haya timbrado (el documento está en borrador/corrección); NO se permite regresar una vez timbrada la línea (factura, anticipo o complemento), aunque falte enviarla, porque el documento timbrado es inmutable. La corrección posterior al timbrado es solo vía el módulo Notas de Crédito.

Regla 17 — Inmutabilidad post-timbrado
Cuando una línea está en estado Generado o Enviado, el sistema no permite editar el documento timbrado ni re-timbrarlo. La única vía de modificación post-timbrado fiscal es a través del módulo Notas de Crédito (módulo independiente). La cancelación fiscal vía CFDI de cancelación SAT no está contemplada en el Paso 3 ni en el módulo Validar Cobro.

Regla 18 — Manejo de errores del PAC TurboPac
Cuando el usuario presiona Timbrar y el PAC TurboPac responde con error, el sistema muestra el mensaje de error con el detalle del problema (PAC indisponible, datos inválidos, errores SAT, etc.) siguiendo el mismo comportamiento operativo que el módulo Factura por Adelantado. La línea permanece en estado Pendiente para que el usuario corrija (si aplica) e intente nuevamente.

Regla 19 — Auto-guardado del Paso 3
Cuando el usuario modifica Uso CFDI, Método de Pago, ejecuta acciones de previsualizar/timbrar/enviar, o navega, el sistema auto-guarda el estado del Paso 3 (estados de cada línea, valores editados, documentos timbrados, envíos confirmados). No existe botón ""Guardar"" manual.

Regla 20 — Cierre del wizard
Cuando todas las líneas del Paso 3 están en estado Enviado, al confirmar la finalización el sistema cierra el wizard de Validar Cobro para el cliente y retorna al listado principal de Validar Cobro. El cliente sale del listado de pendientes (al menos por los documentos procesados en esta sesión).

## Riesgos

Riesgo 1 — Bloqueo de navegación tras timbrado
Mientras una línea NO se haya timbrado, el usuario puede regresar a los Pasos 1 o 2 a corregir (el documento está en borrador). Una vez timbrada la línea, el documento es inmutable: no se puede regresar a los Pasos 1 o 2, solo resta enviarlo y, si hubo error, corregir vía el módulo Notas de Crédito. Esto implica que, si una línea quedó timbrada pero el ciclo necesita interrumpirse (cliente cancela a último minuto, error detectado tarde), no hay salida limpia del Paso 3 para esa línea sin pasar por Notas de Crédito.** Pendiente definir vía operativa de excepción para casos donde el usuario necesita salir del Paso 3 con líneas pendientes sin timbrar. **

# Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL CLIENTE Y BARRA DE PASOS
═══════════════════════════════════════════════════════════════

Criterio A1 — Cabecera del cliente
Dado que el usuario entra al Paso 3 del wizard,
Cuando el sistema renderiza la cabecera,
Entonces deberá mostrar: logo del cliente (si existe), Alias, etiquetas preexistentes de clasificación del Catálogo de Clientes, identificador fiscal bajo la etiqueta unificada ""RFC/RUC"", razón social legal completa y moneda de facturación.

Criterio A2 — Barra de pasos del wizard
Dado que el usuario está en el Paso 3,
Cuando el sistema renderiza la barra de pasos,
Entonces deberá mostrar los tres pasos: ""1 - CAPTURAR COBRO"" (✓), ""2 - ASOCIAR FACTURA/PROFORMA"" (✓), ""3 - FACTURACIÓN Y ENVÍO"" (activo).

═══════════════════════════════════════════════════════════════
SECCIÓN B — LISTADO DE LÍNEAS A PROCESAR
═══════════════════════════════════════════════════════════════

Criterio B1 — Una línea por cada documento de la asociación
Dado que el usuario avanzó al Paso 3 con la asociación cerrada en el Paso 2,
Cuando el sistema arma el listado del Paso 3,
Entonces deberá generar exactamente una línea por cada documento (proforma o factura) asociado en el Paso 2 que requiera emisión de un documento fiscal nuevo (factura, anticipo o complemento).

Criterio B2 — Datos visibles de cada línea
Dado que el sistema renderiza una línea del Paso 3,
Cuando muestra sus datos,
Entonces deberá presentar: tipo de documento origen (Proforma normal / Proforma con controlados / Factura existente), folio del documento origen, fecha, Pedido Interno, empresa emisora del grupo PROQUIFA México, monto total de la factura, tipo de cambio (cuando aplique), NCs aplicadas (si aplica), estado actual de la línea (Pendiente / Generado / Enviado), y los campos fiscales seleccionables de la línea según el caso (Uso CFDI y Método de Pago — ver Sección D).
═══════════════════════════════════════════════════════════════
SECCIÓN C — LÓGICA CONDICIONAL DEL TIPO DE DOCUMENTO FISCAL
═══════════════════════════════════════════════════════════════

Criterio C1 — Proforma normal → Factura nueva
Dado que una línea parte de una Proforma normal (sin productos controlados),
Cuando el sistema determina el tipo de documento fiscal a generar,
Entonces deberá configurar la línea para emitir una Factura nueva (CFDI Ingreso). La proforma se referencia como documento origen interno (no fiscal).

Criterio C2 — Proforma con productos controlados → Factura Anticipo
Dado que una línea parte de una Proforma con productos controlados (sustancias tipo Mundial, Nacional u Origen, conforme la clasificación de control usada en el resto del flujo),
Cuando el sistema determina el tipo de documento fiscal a generar,
Entonces deberá configurar la línea para emitir una Factura Anticipo (CFDI Ingreso con tipo de relación 07 SAT — Aplicación de Anticipo). Esto se debe a que el pago se recibe antes de la entrega física del producto controlado, conforme práctica operativa de PROQUIFA.

Criterio C3 — Factura existente con cobro → Complemento de Pago
Dado que una línea parte de una Factura existente (emitida previamente desde el flujo de Prepago con Factura por Adelantado) con un cobro asociado en el Paso 2,
Cuando el sistema determina el tipo de documento fiscal a generar,
Entonces deberá configurar la línea para emitir un Complemento de Pago (CFDI Pagos 2.0) que se relaciona al UUID de la factura existente para registrar el pago recibido conforme normativa SAT.

═══════════════════════════════════════════════════════════════
SECCIÓN D — EDICIÓN DE USO CFDI Y MÉTODO DE PAGO
═══════════════════════════════════════════════════════════════

Criterio D1 — Uso CFDI editable en líneas de Factura y Factura Anticipo
Dado que una línea genera Factura nueva o Factura Anticipo (parte de proforma),
Cuando el sistema renderiza la columna Uso CFDI,
Entonces deberá ofrecer un combo del catálogo SAT c_UsoCFDI (ejemplo: ""G01 - Adquisición de mercancías"", ""G03 - Gastos en general"", ""P01 - Por definir"", etc.), editable antes del timbrado. El valor por defecto corresponde al Uso CFDI configurado del cliente o capturado en el pedido original.

Criterio D2 — Método de Pago para líneas con proforma origen
Dado que una línea parte de una proforma (genera Factura o Factura Anticipo),
Cuando el sistema renderiza el campo Método de Pago,
Entonces deberá ofrecer un control radio button con dos opciones editables por el usuario antes del timbrado: PPD (Pago en parcialidades o diferido) y PUE (Pago en una sola exhibición). El usuario selecciona según corresponda al caso.

Criterio D3 — Método de Pago fijo PPD en Complementos
Dado que una línea genera Complemento de Pago (parte de factura existente),
Cuando el sistema renderiza el campo Método de Pago,
Entonces deberá mostrar PPD como valor fijo no editable. Esto cumple con la normativa SAT: los Complementos de Pago aplican únicamente a CFDIs con método PPD.

Criterio D4 — Uso CFDI en solo lectura en líneas de Complemento de Pago
Dado que una línea genera Complemento de Pago (la factura ya existe),
Cuando el sistema renderiza la columna Uso CFDI,
Entonces deberá mostrar, en modo solo lectura, el Uso CFDI con el que ya se timbró la factura origen, sin ofrecer combo ni permitir selección. El valor ya existe en la factura y solo se refleja como dato informativo del documento que se está pagando.

═══════════════════════════════════════════════════════════════
SECCIÓN E — INCLUSIÓN DE NOTAS DE CRÉDITO EN EL XML
═══════════════════════════════════════════════════════════════

Criterio E1 — NCs aplicadas en Paso 2 incluidas en CFDIRelacionados
Dado que la línea tiene una o más NCs aplicadas desde el Paso 2,
Cuando el sistema arma el XML del CFDI a timbrar,
Entonces deberá incluir cada NC dentro del nodo CFDIRelacionados del XML con: UUID timbrado de la NC, monto aplicado al documento, y tipo de relación SAT correspondiente (01 - Nota de crédito de los documentos relacionados, o 07 - CFDI por aplicación de anticipo, según el caso fiscal).

Criterio E2 — Visibilidad de NCs aplicadas en la línea del Paso 3
Dado que la línea tiene NCs aplicadas,
Cuando el sistema renderiza la línea,
Entonces deberá mostrar visualmente las NCs que se incluirán en el XML (folio NC, UUID, monto aplicado) para que el usuario valide antes del timbrado.

═══════════════════════════════════════════════════════════════
SECCIÓN F — FLUJO PREVISUALIZAR, TIMBRAR, ENVIAR
═══════════════════════════════════════════════════════════════

Criterio F1 — Acción Previsualizar
Dado que una línea está en estado Pendiente,
Cuando el usuario presiona ""Previsualizar"",
Entonces el sistema deberá abrir un modal con el PDF representativo del documento a generar. El modal incluye todos los datos que tendrá el CFDI (cliente, conceptos, montos, NCs aplicadas, Uso CFDI, Método de Pago, etc.). El usuario puede cerrar sin acción, regresar a editar o proceder a Timbrar.

Criterio F2 — Acción Timbrar (con generación en cascada cuando aplique)
Dado que el usuario validó la previsualización y procede a timbrar,
Cuando presiona ""Generar"" o ""Timbrar"",
Entonces el sistema deberá actuar según el caso: si la línea es Factura PUE desde proforma, timbrar la Factura PUE vía PAC TurboPac (línea pasa a estado Generado); si la línea es Factura PPD desde proforma, timbrar primero la Factura PPD y, si es exitoso, timbrar inmediatamente en cascada el Complemento de Pago asociado al cobro confirmado en el wizard (ambos CFDIs persistidos, línea pasa a estado Generado); si la línea es Complemento de Pago desde factura existente, timbrar el Complemento de Pago (línea pasa a estado Generado). Tras timbrado exitoso se muestra el modal de éxito.

Criterio F2.1 — Cascada Factura PPD + Complemento de Pago desde proforma
Dado que el usuario elige PPD para una línea con proforma origen y presiona Timbrar,
Cuando el sistema procesa,
Entonces deberá: timbrar primero la Factura PPD vía PAC TurboPac; si el timbrado de la Factura PPD es exitoso, disparar inmediatamente la generación y timbrado del Complemento de Pago asociado a esa Factura PPD (con el cobro confirmado en el wizard como Pago del Complemento); persistir ambos CFDIs timbrados (Factura PPD + Complemento de Pago) en PQF2; habilitar el envío al cliente desde el modal de envío del Paso 3 con destinatario (contacto del pedido, editable), CC al ESAC (editable), asunto generado por sistema según plantilla por tipo de documento (no editable), adjuntos (PDF y XML de la Factura PPD + PDF y XML del Complemento de Pago + Confirmación de Pedido del Pedido Prepago, no editables) y text area de notas extras opcional. Si el timbrado del Complemento de Pago falla tras Factura PPD exitosa, notificar al usuario y permitir reintento del Complemento; la Factura PPD permanece vigente.

Criterio F2.2 — Factura PUE sin generación de Complemento de Pago
Dado que el usuario elige PUE para una línea con proforma origen y presiona Timbrar,
Cuando el sistema procesa,
Entonces deberá timbrar únicamente la Factura PUE vía PAC TurboPac (conforme normativa SAT: las facturas PUE son fiscalmente autocontenidas, no requieren Complemento de Pago). El envío posterior al cliente desde el modal de envío del Paso 3 incluye: destinatario (contacto del pedido, editable), CC al ESAC (editable), asunto generado por sistema según plantilla por tipo de documento (no editable), adjuntos (PDF y XML de la Factura PUE + Confirmación de Pedido del Pedido Prepago, no editables) y text area de notas extras opcional. La línea queda cerrada con un único CFDI timbrado.

Criterio F3 — Acción Enviar
Dado que una línea está en estado Generado,
Cuando el usuario presiona ""Enviar"",
Entonces el sistema deberá abrir el modal de envío con: Destinatario (Para) con el contacto del cliente del pedido (editable, default heredado); Copia (CC) con el ESAC asignado (editable, default sugerido); Asunto generado por sistema según plantilla por tipo de documento (no editable); Adjuntos con el PDF y XML de cada CFDI timbrado en la línea más Confirmación de Pedido del Pedido Prepago (no editables); y Notas extras, text area editable opcional. Al confirmar el envío, si es exitoso, la línea pasa al estado Enviado.

Criterio F4 — Operación INDIVIDUAL por línea
Dado que el usuario opera el Paso 3,
Cuando ejecuta acciones de timbrado o envío,
Entonces deberá operar una línea a la vez. No existen botones de operación masiva (no ""Timbrar todo"", no ""Enviar todo"").

═══════════════════════════════════════════════════════════════
SECCIÓN G — ESTADOS POR LÍNEA
═══════════════════════════════════════════════════════════════

Criterio G1 — Estado inicial ""Pendiente""
Dado que el usuario entra al Paso 3,
Cuando el sistema renderiza el estado de cada línea,
Entonces el estado inicial de todas las líneas deberá ser ""Pendiente"".

Criterio G2 — Estado tras timbrado exitoso
Dado que el timbrado de una línea fue exitoso,
Cuando el sistema actualiza el estado,
Entonces la línea pasa a uno de los estados siguientes según el tipo de documento generado: ""Factura Generada"" (si se timbró Factura nueva), ""Factura Anticipo Generada"" (si se timbró Factura Anticipo) o ""Complemento Generado"" (si se timbró Complemento de Pago).

Criterio G3 — Estado tras envío exitoso
Dado que el envío de una línea al cliente fue exitoso,
Cuando el sistema actualiza el estado,
Entonces la línea pasa al estado ""Enviado"".

Criterio G4 — Estados intermedios persistidos si el usuario interrumpe
Dado que el usuario timbró una línea pero no la envió en la misma sesión,
Cuando regresa al Paso 3 (misma sesión o futura),
Entonces el sistema deberá conservar el estado Generado (Factura Generada / Factura Anticipo Generada / Complemento Generado) hasta que el usuario complete el envío.

═══════════════════════════════════════════════════════════════
SECCIÓN H — ACCIONES AUTOMÁTICAS AL ENVIAR (SOLO MÉXICO)
═══════════════════════════════════════════════════════════════

Criterio H1 — Establecimiento de Fecha Estimada de Entrega (FEE)
Dado que el envío de un documento fue exitoso,
Cuando el sistema confirma el envío,
Entonces deberá disparar automáticamente el establecimiento de la Fecha Estimada de Entrega (FEE) del pedido asociado al documento enviado. La FEE es una fecha calculada por el sistema según reglas operativas de PROQUIFA México (no es el envío de la entrega física, es el establecimiento del compromiso de fecha).

Criterio H2 — Transferencia a Legacy
Dado que el envío de un documento fue exitoso,
Cuando el sistema confirma el envío,
Entonces deberá transferir automáticamente al sistema Legacy de PROQUIFA: el pedido, el documento fiscal generado (factura/anticipo/complemento), las NCs aplicadas si las hubo, y la información del cobro asociado. Esta transferencia permite la continuidad operativa del pedido en Legacy (logística, surtido, entrega).

Criterio H3 — Confirmación de Pedido generada y adjunta al envío
Dado que el envío de una línea de un Pedido Prepago se realiza,
Cuando el sistema arma el correo,
Entonces deberá generar la Confirmación de Pedido del Pedido Prepago e incluirla como adjunto en el mismo correo, junto con los demás CFDIs timbrados de la línea (PDF y XML). La Confirmación de Pedido no se previsualiza: solo se genera y se muestra como archivo adjunto en el modal. No existe candado bloqueante previo al envío:

Criterio H4 — Aplica solo a operaciones México
Dado que las acciones automáticas H1, H2 y H3 se disparan al enviar,
Cuando el cliente es de Región México,
Entonces estas acciones aplicarán. Para clientes de Región Perú, estas acciones no aplican (Perú tiene flujos operativos distintos, documentados en requisito independiente).

═══════════════════════════════════════════════════════════════
SECCIÓN I — COMPOSICIÓN DEL MODAL DE ENVÍO
═══════════════════════════════════════════════════════════════

Criterio I1 — Destinatario del envío
Dado que el sistema arma el correo de envío,
Cuando determina el destinatario,
Entonces deberá usar el contacto del cliente que envió el pedido (heredado del flujo de tramitación del pedido).

Criterio I2 — Composición del modal de envío
Dado el modal de envío del Paso 3,
Cuando el sistema lo renderiza,
Entonces deberá mostrar: Para con el contacto del cliente del pedido (editable, default heredado del pedido); CC con el ESAC asignado (editable, default sugerido por el sistema); Asunto generado automáticamente por el sistema según plantilla por tipo de documento de la línea (no editable); Adjuntos con el PDF y XML de cada CFDI timbrado en la línea más Confirmación de Pedido del Pedido Prepago (no editables); y Notas extras, text area editable opcional para texto adicional libre.

Criterio I3 — Asunto del correo según tipo de documento
Dado el modal de envío del Paso 3,
Cuando el sistema arma el asunto del correo,
Entonces deberá generarlo automáticamente según los documentos de la línea y mostrarlo en el modal como no editable, diferenciando: línea con Factura PUE, línea con Factura PPD + Complemento de Pago (cascada), y línea con solo Complemento de Pago (desde factura existente). ** Plantilla del asunto pendiente de confirmar (PMO #31). Propuesta para los tres casos: ""<Folio Pedido Interno> - <Folio Factura>"". **

═══════════════════════════════════════════════════════════════
SECCIÓN J — PERSISTENCIA E INMUTABILIDAD
═══════════════════════════════════════════════════════════════

Criterio J1 — Auto-guardado del Paso 3
Dado que el usuario opera en el Paso 3,
Cuando modifica Uso CFDI, Método de Pago, ejecuta acciones o navega,
Entonces el sistema deberá auto-guardar el estado del Paso 3 (estados de cada línea, valores editados, documentos timbrados, envíos confirmados). No existe botón ""Guardar"" manual.

Criterio J2 — Persistencia del estado y navegación atrás según timbrado
Dado que el usuario sale del wizard antes de finalizar todas las líneas del Paso 3,
Cuando regresa al módulo Validar Cobro y selecciona al mismo cliente,
Entonces el sistema deberá redirigirlo directamente al Paso 3 con el estado preservado; permitir regresar a los Pasos 1 o 2 únicamente para las líneas que NO se hayan timbrado; y bloquear el regreso una vez la línea está timbrada (documento inmutable), aunque falte enviarla.

Criterio J3 — Inmutabilidad post-timbrado
Dado que una línea está en estado Generado o Enviado,
Cuando el usuario intenta editar el documento timbrado,
Entonces el sistema no deberá permitir la edición ni el re-timbrado. La única vía de modificación post-timbrado fiscal es a través del módulo Notas de Crédito (módulo independiente).

═══════════════════════════════════════════════════════════════
SECCIÓN K — MANEJO DE ERRORES Y CIERRE DEL WIZARD
═══════════════════════════════════════════════════════════════

Criterio K1 — Errores del PAC TurboPac
Dado que el usuario presiona Timbrar y el PAC TurboPac responde con error,
Cuando el sistema procesa la respuesta,
Entonces deberá mostrar el mensaje de error con el detalle del problema (PAC indisponible, datos inválidos, errores SAT, etc.) siguiendo el mismo comportamiento operativo que el módulo Factura por Adelantado. La línea permanece en estado Pendiente.

Criterio K2 — Cierre del wizard
Dado que todas las líneas del Paso 3 están en estado Enviado,
Cuando el usuario confirma la finalización,
Entonces el sistema deberá cerrar el wizard de Validar Cobro para el cliente y retornar al listado principal del módulo. El cliente sale del listado de pendientes en la medida en que los documentos procesados sean los únicos pendientes que tenía."								"- Esta fila documenta el Paso 3 del wizard de Validar Cobro (Facturación y Envío) para Región México exclusivamente. La estructura UI tiene diferencias significativas en Perú (sin Factura Anticipo SAT, sin Complemento de Pago 2.0, sin transferencia a Legacy ni Confirmaciones de Pedido, catálogos SUNAT en lugar de SAT) y se documenta en requisito independiente.
- El Paso 3 se invoca desde el Paso 2 al presionar ""Continuar"" con la asociación cerrada.
- Lógica condicional del tipo de documento fiscal a generar por línea:
  - Proforma normal → Factura nueva (CFDI Ingreso).
  - Proforma con productos controlados → Factura Anticipo (CFDI Ingreso con tipo de relación 07 SAT — Aplicación de Anticipo).
  - Factura existente (de Prepago con Factura por Adelantado previo) con cobro asociado → Complemento de Pago (CFDI Pagos 2.0).
- Productos controlados: PROQUIFA es distribuidor químico-biológico USP/EP/FEUM/Microbiologics. La práctica operativa exige pago anticipado para productos controlados, por lo que se factura como anticipo en lugar de factura estándar.
- Edición del Uso CFDI: combo del catálogo SAT c_UsoCFDI editable por línea antes del timbrado.
- Edición del Método de Pago: editable para proformas (PPD o PUE radio button); fijo PPD no editable para Complementos de Pago (conforme normativa SAT).
- Operación INDIVIDUAL por línea: no existen acciones masivas. Cada línea se previsualiza, timbra y envía una a la vez.
- Flujo operativo recomendado por línea: previsualizar PDF → timbrar (con PAC TurboPac) → enviar al cliente.
- Estados por línea: Pendiente (estado inicial), Factura Generada / Factura Anticipo Generada / Complemento Generado (después del timbrado), Enviado (después del envío al cliente). El estado intermedio (Generado pero no Enviado) persiste si el usuario interrumpe el flujo antes de enviar.
- Inclusión automática de NCs en el XML: las NCs aplicadas en el Paso 2 se incluyen en el nodo CFDIRelacionados del CFDI a timbrar, con UUID y monto correspondiente, tipo de relación 01 o 07 según el caso. Conforme Apéndice 5 Anexo 20 versión 4.0 SAT.
- Destinatario del envío: contacto del cliente que envió el pedido (heredado del flujo de tramitación).
- Cuerpo del correo: para Facturas y Facturas Anticipo, misma plantilla que el correo de envío de Factura por Adelantado. Para Complementos de Pago, plantilla pendiente de confirmación (propuesta inicial: asunto ""<Folio Pedido Interno> - <Folio Factura>"", cuerpo por definir).
- Al ENVIAR cada documento (no al timbrar), el sistema dispara automáticamente las acciones post-envío (SOLO MÉXICO):
  - FEE (Fecha Estimada de Entrega): no es un sistema, es el establecimiento de la fecha estimada de entrega del pedido asociado. El sistema calcula y establece la FEE conforme reglas operativas PROQUIFA México al momento del envío del documento.
  - Transferencia a Legacy: envío del pedido, documento fiscal, NCs aplicadas e info del cobro al sistema legado de PROQUIFA para continuidad operativa (logística, surtido, entrega).
  - Confirmación de Pedido: el concepto ya existe hoy en ProquifaNet para operaciones de crédito. R16 extiende esta práctica a operaciones de prepago vía Validar Cobro. Viaja como adjunto en el mismo correo de envío de la línea.
- Las tres acciones automáticas aplican únicamente a operaciones México. Perú tiene flujos distintos.
- Persistencia del Paso 3: si el usuario sale del wizard con líneas pendientes (Pendientes o Generadas sin Enviar), al volver a entrar al mismo cliente desde Validar Cobro el sistema lo redirige directamente al Paso 3 hasta cerrar todas las líneas. No se permite re-iniciar el wizard al Paso 1 o Paso 2 hasta que todas las líneas estén Enviadas. El sistema fuerza el cierre del ciclo del cliente.
- Inmutabilidad post-timbrado: una vez timbrado un documento (estado Generado o Enviado), no se permite re-timbrar ni editar. La cancelación fiscal vía CFDI de cancelación SAT no está contemplada en el Paso 3 ni en el módulo Validar Cobro.
- Manejo de errores del PAC TurboPac: mismo comportamiento operativo que en Factura por Adelantado (mensaje de error con detalle del problema; reintento manual del usuario).
- Cierre del wizard: cuando todas las líneas del Paso 3 están Enviadas, el sistema cierra el wizard y retorna al listado principal del módulo.
- ** Pendiente confirmar el uso del tipo de relación 07 SAT para la Factura Anticipo generada desde proforma con productos controlados. **
- ** Pendiente confirmar la plantilla del correo de envío para Complementos de Pago (asunto y cuerpo). Propuesta inicial: asunto ""<Folio Pedido Interno> - <Folio Factura>"". **
- ** Pendiente definir vía operativa de excepción para casos donde el usuario necesita salir del Paso 3 con líneas pendientes sin posibilidad de continuar (cliente cancela a último minuto, error operativo detectado tarde, etc.). **
- ** Pendiente definición formal de la política de indisponibilidad del PAC TurboPac (transversal con Factura por Adelantado). **
- ** Pendiente resolver formalmente la denominación canónica del rol operativo entre ""Gestor de Cobranza"" y ""Analista de Cuentas por Cobrar"" (transversal). **"									
