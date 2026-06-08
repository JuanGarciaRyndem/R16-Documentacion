# TPSC-RE-FU-020 — Factura por Adelantado: Detalle Peru

| Campo                   | Valor                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ID**                  | TPSC-RE-FU-020                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **Título**              | Factura por Adelantado: Detalle Peru                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Módulo / Épica**      | Factura por Adelantado                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| **Historia de Usuario** | Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente de resolver) **, quiero contar con la pantalla de Detalle por cliente en Factura por Adelantado que me permita generar, timbrar ante SUNAT y enviar la factura electrónica de cada pedido Prepago pendiente de clientes con Región Perú, para emitir oportunamente las facturas por adelantado de Golocaer S.A.C. y dar continuidad al flujo de cobro en Validar Cobro, cumpliendo la normativa SUNAT. |
| **Prioridad**           | Alta                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Estado**              | Propuesto                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **Requisito asociado**  | R16.2M-RE-FU-001                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

---

## Requisito Funcional

El sistema debe contar con una pantalla de Detalle por cliente en el módulo Factura por Adelantado que muestre los datos del cliente y el listado de sus pedidos Prepago pendientes de facturar por adelantado de clientes con Región Perú, ofreciendo por cada pedido la acción de generar la factura o, si ya fue timbrada, de enviarla al cliente. El flujo de generación contempla la revisión de los datos fiscales, la previsualización del PDF y el timbrado ante SUNAT; tras un envío exitoso el pedido sale del listado de pendientes. La salida operativa posterior genera el pendiente correspondiente en Validar Cobro. Esta funcionalidad aplica únicamente a pedidos Prepago (los pedidos Crédito de Perú no usan este flujo) y no aplica a pedidos que contengan productos clasificados como Sustancias Controladas (tipo Mundial, Nacional u Origen).

---

## Alcance

### Aplica a

- Pantalla de Detalle por cliente del módulo Factura por Adelantado para pedidos Prepago de clientes con Región Perú.
- Empresa Emisora del pedido, mostrada en el listado y en los datos del pedido (igual que en México). El emisor en Perú es Golocaer S.A.C.
- Listado de pedidos del cliente con acción contextual según estado: "Generar Factura" (pendiente generar) o "Enviar Factura" (timbrada pendiente de envío).
- Modal de Generación de Factura con los datos del pedido, del cliente y del emisor en modo solo lectura, y el Tipo de Operación SUNAT (catálogo 51) como dato de la operación.
- Modal de Previsualización del PDF antes del timbrado real ante SUNAT/OSE.
- Modal de Alerta ante error de validación o de timbrado de SUNAT/OSE.
- Modal de éxito de generación tras el timbrado exitoso.
- Modal de Envío al cliente (destinatario, CC al ESAC, asunto, adjuntos PDF + XML, notas extras).
- Modal de éxito de envío y salida del pedido del listado.
- Emisión como Comprobante de Pago Electrónico tipo 01 (Factura), formato UBL 2.1, con IGV 18%.
- Numeración con serie alfanumérica de 4 caracteres más correlativo de hasta 8 dígitos, consecutivo e independiente por serie.
- Persistencia del XML timbrado y su representación PDF como artefacto fiscal inmutable al timbrarse.
- Salida operativa post-envío: generación del pendiente correspondiente en Validar Cobro (todos los pedidos de esta fila son Prepago).
- Visibilidad del Detalle del cliente filtrada por la cartera del usuario (campo Cobrador del Catálogo de Clientes).

### No aplica a

- Pedidos Crédito de la región Perú: la Factura por Adelantado no se ofrece para Crédito en Perú, porque el timbrado fiscal peruano en R16 se limita a Prepago. Decisión de alcance del proyecto (ver TPSC-RE-FU-012).
- Pedidos con Sustancias Controladas tipo Mundial, Nacional u Origen: en México se excluyen de la Factura por Adelantado y se atienden vía Factura Anticipo desde Validar Cobro, porque al ser importados el CFDI de venta de primera mano exige el dato del pedimento aduanero. ** En Perú este impedimento no existe (la factura de venta SUNAT no exige el dato aduanero; la DUA es trámite separado) y la Ley del IGV permite facturar por el monto cobrado anticipadamente, por lo que fiscalmente los controlados SÍ podrían facturarse por adelantado con factura normal en Perú. Pendiente confirmar con el cliente si desea habilitar la facturación por adelantado de controlados en Perú o mantener la exclusión por política operativa interna. **
- Clientes con Región México (cubiertos en TPSC-RE-FU-019).
- Otras empresas del grupo PROQUIFA: solo Golocaer S.A.C. emite en Perú.
- Esquema fiscal mexicano: Método de Pago PPD, Forma de Pago 99, Tipo de Comprobante "I-Ingreso" del SAT y Complemento de Pago NO aplican a Perú; en el modal de Generación los campos Método de Pago y Forma de Pago no se muestran.
- Transferencia a Legacy: de Perú nada se transfiere a Legacy.
- Generación del PDF de la factura como artefacto (estructura visual): se documenta en TPSC-RE-FU-022.
- Régimen de Detracciones (SPOT) y Percepciones del IGV: bajo análisis preliminar NO aplican a los productos típicos de Golocaer S.A.C. ** Pendiente confirmar con asesor contable peruano antes de habilitar Perú productivamente. **
- Guía de Remisión Electrónica (GRE): en el flujo de Factura por Adelantado la factura se emite antes del despacho, por lo que al facturar aún no existiría traslado ni GRE, y la factura no la referenciaría. ** Pendiente de validar con el cliente la secuencia GRE↔factura en prepago. **

---

## Reglas de Negocio

Regla 1 — Estados del pedido visibles en el listado
Cada pedido del cliente muestra una acción contextual según su estado: "Generar Factura" si aún no tiene factura timbrada (pendiente generar), o "Enviar Factura" si ya fue timbrada pero está pendiente de envío al cliente (pendiente enviar). Una vez la factura se envía exitosamente, el pedido desaparece del listado.

Regla 2 — Solo aplica a Prepago
La Factura por Adelantado de Perú aplica únicamente a pedidos Prepago. Los pedidos Crédito peruanos no usan este flujo, porque el timbrado fiscal peruano en R16 se limita a Prepago (ver TPSC-RE-FU-012). ** Una factura electrónica real de Golocaer S.A.C. recibida como muestra (E001-362, 08/05/2026) fue emitida a Crédito; pendiente confirmar con el cliente si Golocaer factura habitualmente a crédito en su operación corriente (ajena a R16) o si el alcance Perú se mantiene solo en Prepago. Se conserva la decisión vigente: Perú = solo Prepago. **

Regla 3 — Datos del cliente en cabecera del Detalle
La cabecera del Detalle del cliente muestra: Razón Social del cliente, RUC, moneda de facturación y la clasificación del cliente si está disponible. Son datos preexistentes en el sistema, en modo solo lectura, no editables desde este módulo.

Regla 4 — Datos del pedido visibles en el listado
Cada pedido del cliente en el listado muestra: Pedido Interno, Fecha del pedido, Condición de Pago, Empresa Emisora del pedido, Subtotal, IGV y Monto Total en la moneda del pedido.

Regla 5 — Modal de Generación de Factura
Al presionar "Generar Factura" en un pedido, el modal de Generación de Factura muestra en modo solo lectura: cliente (Razón Social), Monto Total del pedido, Pedido Interno, Condición de Pago, datos del contacto del cliente (nombre, correo electrónico, teléfono), Datos Fiscales del Cliente (RUC, Razón Social, dirección fiscal, Régimen Tributario, Correo electrónico, Moneda, Tipo de Cambio cuando aplique, Tipo de Comprobante, Condición de Pago) y Datos Fiscales del Emisor (RUC, Emisor, Razón Social de Golocaer S.A.C.) y los Comentarios de Facturación.

Regla 6 — Sin Método de Pago ni Forma de Pago; sin Complemento de Pago
El modal de Generación de Perú NO muestra Método de Pago ni Forma de Pago (son campos del SAT que no aplican a SUNAT). El Tipo de Comprobante se fija en "01 - Factura". No se genera Complemento de Pago en ningún momento; la factura se emite completa con su IGV desde la emisión.

Regla 7 — Tipo de Operación SUNAT (catálogo 51) en la factura
Cada factura consigna un Tipo de Operación del catálogo 51 SUNAT (campo obligatorio del XML UBL 2.1; en la factura real de muestra de Golocaer el valor fue "0101 - Venta interna"), equivalente funcional al Uso CFDI mexicano. ** Pendiente definir para la maqueta si el Tipo de Operación es un campo configurable que el operador selecciona en el modal (como el Uso CFDI en México) o si el sistema lo fija siempre en "0101 - Venta interna". La evidencia de la factura real apunta a un valor único, pero un solo documento no confirma el patrón. Ver Brecha 3 de TPSC-RE-FU-005. **

Regla 8 — Tipo de Cambio del día de generación
Cuando el cliente factura en una moneda distinta a soles (PEN), el modal de Generación muestra el Tipo de Cambio aplicable al día de generación de la factura, en modo solo lectura. ** Fuente oficial del Tipo de Cambio para Perú pendiente de definir (no aplica el DOF mexicano). **

Regla 9 — Generación en dos pasos: revisar datos, previsualizar PDF, timbrar
Tras confirmar los datos en el modal de Generación, el sistema muestra el modal de Previsualización con el PDF de la Factura tal como saldrá al cliente. El usuario revisa y confirma con "Generar Factura" desde la Previsualización para proceder con el timbrado real ante SUNAT/OSE. La acción "Cancelar" en cualquiera de los dos modales aborta el flujo sin timbrar.

Regla 10 — Manejo de errores de validación o de timbrado SUNAT/OSE
Cuando el timbrado falla por un error de validación o por un error del servicio de timbrado, el sistema muestra un modal de Alerta con la descripción específica del error y un botón "Continuar" que cierra el modal y devuelve al usuario al estado previo. El usuario debe corregir los datos en el módulo correspondiente (típicamente el Catálogo de Clientes) y reintentar la generación desde el principio. El sistema no ofrece edición directa de los datos del cliente desde este modal.

Regla 11 — Persistencia de la Factura al timbrarse exitosamente
Cuando el timbrado se completa exitosamente, el sistema persiste la Factura en base de datos como artefacto fiscal inmutable (XML timbrado y representación PDF) y muestra el modal de confirmación de éxito de generación. La Factura se persiste al timbrarse: una vez timbrada ya no se puede modificar.

Regla 12 — Numeración SUNAT por serie
El número de la Factura usa serie alfanumérica de 4 caracteres más correlativo de hasta 8 dígitos, consecutivo e independiente por serie. El emisor es Golocaer S.A.C. ** Esquema de series de Golocaer S.A.C. pendiente de definir. **

Regla 13 — Modal de Envío de Factura
Cuando la Factura fue timbrada exitosamente y el usuario presiona "Enviar Factura", el modal de Envío presenta: Para (destinatario) con el contacto del cliente del pedido, editable; CC con el ESAC asignado al cliente/pedido, editable; Asunto generado automáticamente con los folios del documento, no editable; Adjuntos con el PDF y XML de la Factura timbrada, no editables; y Notas extras, campo de texto editable opcional.

Regla 14 — Confirmación de envío exitoso y cierre del pendiente
Al confirmarse que el correo se envió exitosamente, el sistema muestra el modal de confirmación de éxito de envío, el pedido sale del listado del Detalle, y la factura queda persistida y consultable desde Validar Cobro.

Regla 15 — Salida operativa post-envío a Validar Cobro
Tras el envío exitoso de la Factura por Adelantado, el sistema genera el pendiente correspondiente en Validar Cobro para que el equipo de Cobranza concilie el pago del cliente contra esta Factura. No hay transferencia a Legacy (de Perú nada va a Legacy) y, al ser todos los pedidos de esta fila Prepago, no existe rama de Crédito.

Regla 16 — Visibilidad filtrada por cartera del usuario
El acceso al Detalle de un cliente solo se permite si el cliente está asignado a la cartera del usuario (campo Cobrador del Catálogo de Clientes). Los clientes asignados a otros usuarios no son accesibles.

Regla 17 — Datos de la Factura provienen del Pedido
Los conceptos de la Factura se obtienen del Pedido: por cada partida, cantidad, descripción (catálogo + descripción + marca + lote + caducidad cuando aplique), valor unitario e importe. Además, cada partida debe consignar los datos fiscales SUNAT del producto (código SUNAT, unidad de medida SUNAT, afectación al IGV por línea). ** Estos datos fiscales del producto no existen en el catálogo de productos actual; brecha bloqueante documentada en Brecha 1 de TPSC-RE-FU-005. **

---

## Riesgos

Riesgo 1 — Datos fiscales del producto SUNAT inexistentes (bloqueante)
El catálogo de productos actual no contiene el código SUNAT del producto, la unidad de medida SUNAT ni la afectación al IGV por línea, datos obligatorios en el XML UBL 2.1. Sin ellos no es posible timbrar la factura, por lo que esta brecha puede detener el desarrollo del flujo de facturación Perú si no se resuelve antes (ver Brecha 1 de TPSC-RE-FU-005).

Riesgo 2 — Brecha de timbrado SUNAT/OSE no resuelta
La habilitación para Región Perú depende de la integración con un OSE/PSE autorizado por SUNAT, brecha mayor no resuelta del proyecto (ver Brecha 5 de TPSC-RE-FU-005). Mientras no se resuelva, los clientes Perú no pueden completar el timbrado.

Riesgo 3 — Datos legales de Golocaer S.A.C. no disponibles
El RUC del emisor, domicilio fiscal, ubigeo, certificado digital y series de facturación de Golocaer S.A.C. no están registrados en el sistema, lo que impide emitir el CPE hasta que se capturen.

Riesgo 4 — Solapamiento de denominación de rol
La denominación del rol que opera este módulo aparece como "Gestor de Cobranza" en la matriz del cliente y como "Analista de Cuentas por Cobrar" en las sesiones de revisión de pantallas, sin una denominación canónica resuelta.

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL CLIENTE Y LISTADO DE PEDIDOS
═══════════════════════════════════════════════════════════════

Criterio A1 — Datos del cliente en cabecera
Dado que el usuario navega al Detalle de un cliente con Región Perú desde el listado del módulo,
Cuando el sistema renderiza la cabecera,
Entonces deberá mostrar Razón Social del cliente, RUC, moneda de facturación, y la clasificación crediticia preexistente del cliente si está disponible.

Criterio A2 — Listado de pedidos pendientes
Dado que el cliente tiene pedidos pendientes de generar o enviar Factura por Adelantado,
Cuando el sistema renderiza el listado,
Entonces deberá mostrar una fila por cada pedido pendiente con: Pedido Interno, Fecha del pedido, Condición de Pago, Empresa Emisora del pedido, Subtotal, IGV, Monto Total y acción contextual.

Criterio A3 — Acción contextual por estado del pedido
Dado que un pedido del cliente aparece en el listado,
Cuando el sistema renderiza la acción del pedido,
Entonces deberá mostrar:
- "Generar Factura" si el pedido NO tiene factura timbrada (estado: pendiente generar).
- "Enviar Factura" si el pedido tiene factura ya timbrada pero pendiente de envío (estado: pendiente enviar).

Criterio A4 — Crédito Perú fuera de alcance
Dado un pedido Crédito de un cliente con Región Perú,
Cuando se evalúa la elegibilidad para Factura por Adelantado,
Entonces el sistema NO deberá ofrecer la Factura por Adelantado, ya que el timbrado peruano en R16 aplica solo a Prepago. ** Pendiente de confirmar con el cliente a la luz de una factura real de Golocaer emitida a crédito (ver Regla 2). **

═══════════════════════════════════════════════════════════════
SECCIÓN B — MODAL DE GENERACIÓN DE FACTURA
═══════════════════════════════════════════════════════════════

Criterio B1 — Apertura del modal al presionar "Generar Factura"
Dado que el usuario presiona "Generar Factura" en un pedido del listado,
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal de Generación de Factura con los datos del pedido y del cliente cargados.

Criterio B2 — Datos del pedido en cabecera del modal
Dado que el modal de Generación se abre,
Cuando el sistema arma la cabecera del modal,
Entonces deberá mostrar: Razón Social del cliente, Monto Total del pedido (visualmente prominente), Pedido Interno y Condición de Pago.

Criterio B3 — Datos del contacto del cliente (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los datos del contacto del cliente,
Entonces deberá mostrar en modo solo lectura: nombre del contacto, correo electrónico, teléfono con extensión cuando aplique.

Criterio B4 — Datos Fiscales del Cliente (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los datos fiscales del cliente,
Entonces deberá mostrar en modo solo lectura: RUC, Razón Social, dirección fiscal, Régimen Tributario, Correo electrónico, Moneda, Tipo de Cambio (cuando la moneda no sea PEN), Tipo de Comprobante (siempre "01 - Factura") y Condición de Pago (Contado/Crédito según el pedido). NO se muestran Método de Pago ni Forma de Pago (no existen en Perú).

Criterio B5 — Datos Fiscales del Emisor (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los datos fiscales del emisor,
Entonces deberá mostrar en modo solo lectura:
- RUC del emisor.
- Emisor (identificador comercial: Golocaer).
- Razón Social del emisor (razón social legal completa: Golocaer S.A.C.).

Criterio B6 — Comentarios de Facturación (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los comentarios de facturación,
Entonces deberá mostrar el texto del campo Comentarios de Facturación del pedido en modo solo lectura.

Criterio B7 — Tipo de Operación SUNAT en la factura
Dado que el modal de Generación está abierto,
Cuando el sistema arma la factura,
Entonces deberá consignar el Tipo de Operación del catálogo 51 SUNAT. ** Pendiente definir para la maqueta si el operador lo selecciona (combo editable, como el Uso CFDI de México) o si el sistema lo fija siempre en "0101 - Venta interna". Si es seleccionable, este sería el único campo configurable del modal; si es fijo, el modal es de solo lectura. Ver Regla 7 y Brecha 3 de TPSC-RE-FU-005. **

Criterio B8 — Acciones del modal: Cancelar y Generar Factura
Dado que el usuario revisa los datos en el modal,
Cuando finaliza la revisión,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el flujo, cierra el modal sin generar factura) y Generar Factura (procede al siguiente paso: previsualización del PDF).

═══════════════════════════════════════════════════════════════
SECCIÓN C — MODAL DE PREVISUALIZACIÓN DEL PDF
═══════════════════════════════════════════════════════════════

Criterio C1 — Apertura del modal de Previsualización tras confirmar el modal de Generación
Dado que el usuario presiona "Generar Factura" en el modal de Generación,
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal de Previsualización mostrando el PDF de la Factura tal como saldrá al cliente, con todos los datos integrados. En este momento la Factura aún NO se ha timbrado ante SUNAT/OSE.

Criterio C2 — Acciones del modal: Cancelar y Generar Factura
Dado que el usuario revisa el PDF previsualizado,
Cuando finaliza la revisión,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el flujo, cierra el modal sin timbrar) y Generar Factura (procede con el timbrado real ante SUNAT/OSE).

═══════════════════════════════════════════════════════════════
SECCIÓN D — MODAL DE ALERTA SUNAT (errores de validación o timbrado)
═══════════════════════════════════════════════════════════════

Criterio D1 — Aparición del modal de Alerta ante error
Dado que el sistema intenta timbrar la Factura y recibe respuesta de error de SUNAT/OSE (errores de validación previa o errores del propio servicio de timbrado),
Cuando el error se identifica,
Entonces deberá mostrar el modal de Alerta SUNAT con la descripción específica del error (ejemplo: "El RUC del cliente no es válido o no está activo ante SUNAT. Verifica y actualiza el RUC para poder timbrar").

Criterio D2 — Acción del modal: Continuar
Dado que el modal de Alerta está abierto,
Cuando el usuario lo cierra,
Entonces el sistema deberá cerrar el modal con la acción "Continuar" y devolver al usuario al estado previo (típicamente el listado del Detalle). El sistema NO ofrece edición directa de los datos del cliente desde este modal: el usuario debe ir al módulo correspondiente (Catálogo de Clientes) a corregir y reintentar la generación desde el principio.

═══════════════════════════════════════════════════════════════
SECCIÓN E — MODAL DE ÉXITO DE GENERACIÓN (timbrado exitoso)
═══════════════════════════════════════════════════════════════

Criterio E1 — Loader durante el timbrado
Dado que el usuario confirmó el timbrado desde el modal de Previsualización,
Cuando el sistema envía la solicitud de timbrado a SUNAT/OSE y espera respuesta,
Entonces deberá mostrar un indicador de carga (loader) con mensaje informativo (ejemplo: "Su solicitud está siendo atendida, por favor espere...").

Criterio E2 — Modal de confirmación de éxito
Dado que el timbrado se completa exitosamente,
Cuando SUNAT/OSE retorna la confirmación,
Entonces el sistema deberá mostrar el modal de confirmación de éxito (ejemplo: "¡Has generado una factura Exitosamente!") y persistir la Factura en base de datos. El pedido cambia de estado en el listado: ahora muestra acción "Enviar Factura" (verde) en lugar de "Generar Factura" (azul).

Criterio E3 — Persistencia inmediata al timbrado exitoso
Dado que el timbrado fue exitoso,
Cuando el sistema procesa la respuesta de SUNAT/OSE,
Entonces deberá persistir el artefacto fiscal completo (XML timbrado y PDF) en base de datos como registro permanente inmutable. La Factura no puede modificarse después de este momento.

═══════════════════════════════════════════════════════════════
SECCIÓN F — MODAL DE ENVÍO DE FACTURA
═══════════════════════════════════════════════════════════════

Criterio F1 — Apertura del modal de Envío al presionar "Enviar Factura"
Dado que un pedido tiene Factura ya timbrada en estado pendiente enviar y el usuario presiona "Enviar Factura",
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
Entonces deberá pre-rellenar el asunto en un formato que incluya la Serie y Correlativo de la Factura y el folio del Pedido Interno. ** Formato del asunto para Perú pendiente de confirmar con el cliente. **

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
Entonces deberá mostrar el modal de confirmación de éxito (ejemplo: "¡Has enviado una factura Exitosamente!").

Criterio G2 — Salida del pedido del listado
Dado que el envío fue exitoso,
Cuando el sistema actualiza el estado del pedido,
Entonces el pedido deberá desaparecer del listado del Detalle del cliente. Si era el último pedido del cliente, el cliente puede salir también del listado agrupado de la pantalla inicial del módulo.

Criterio G3 — Salida operativa a Validar Cobro
Dado que la Factura fue enviada exitosamente,
Cuando el sistema procesa la salida operativa,
Entonces deberá generar el pendiente correspondiente en el módulo Validar Cobro asociado a esta Factura, para que el equipo de Cobranza concilie el pago del cliente. NO se ejecuta transferencia a Legacy. Al ser todos los pedidos de esta fila Prepago, no existe rama de Crédito.

═══════════════════════════════════════════════════════════════
SECCIÓN H — ORDEN DEL FLUJO COMPLETO
═══════════════════════════════════════════════════════════════

Criterio H1 — Orden canónico del flujo
Dado que el usuario inicia el flujo completo de generar y enviar una Factura,
Cuando avanza paso a paso,
Entonces deberá seguirse el siguiente orden:
1. Usuario navega al Detalle del cliente y presiona "Generar Factura" en un pedido.
2. Sistema abre el modal de Generación con los datos del pedido y del cliente.
3. Usuario revisa los datos (y define el Tipo de Operación si resultara configurable) y presiona "Generar Factura" del modal.
4. Sistema abre el modal de Previsualización del PDF.
5. Usuario revisa el PDF previsualizado y presiona "Generar Factura" del modal de Previsualización.
6. Sistema envía la solicitud a SUNAT/OSE. Si hay error: muestra modal de Alerta SUNAT y termina aquí (el usuario corrige y vuelve a empezar). Si es exitoso: continúa.
7. Sistema muestra modal de loader durante el timbrado y luego modal de éxito de generación.
8. Sistema persiste la Factura en base de datos. El pedido cambia de estado a pendiente enviar.
9. Usuario regresa al listado y presiona "Enviar Factura" en el pedido.
10. Sistema abre el modal de Envío con datos pre-rellenados.
11. Usuario revisa, agrega notas al correo si requiere, y presiona "Enviar".
12. Sistema envía el correo al cliente con adjuntos PDF y XML.
13. Sistema muestra modal de éxito de envío.
14. Pedido sale del listado. Salida operativa: pendiente en Validar Cobro (Prepago).

** Detalle visual de los modales (campos exactos, loaders, textos) pendiente de validar contra las maquetas de Golocaer S.A.C. Perú, aún no disponibles. **

---

## Notas Adicionales

- Fila para el detalle/generación de la Factura por Adelantado de la Región Perú (solo Prepago), contraparte de TPSC-RE-FU-019 (México). Estado depende de la resolución de las brechas Perú.
- La Factura por Adelantado es una factura electrónica NORMAL emitida anticipadamente (CPE tipo 01 SUNAT), no una factura de anticipo. En México existe además la "Factura Anticipo" para pedidos con productos controlados (generada desde Validar Cobro), cuyo origen es que el CFDI de venta de primera mano exige el dato del pedimento aduanero. ** En Perú ese esquema de anticipo pierde fundamento porque la factura de venta SUNAT no exige el dato aduanero; pendiente confirmar con el cliente si los controlados se facturan por adelantado directamente (ver duda en Alcance). **
- Decisión de alcance: la Factura por Adelantado NO aplica a Crédito en Perú porque el timbrado peruano en R16 se limita a Prepago (ver TPSC-RE-FU-012). De Perú nada se transfiere a Legacy.
- Diferencia fiscal vs México: en Perú no existe el esquema PPD + Forma de Pago 99 + Complemento de Pago del SAT; el CPE se emite completo con IGV desde la emisión.
- Dato configurable por factura: el equivalente al Uso CFDI mexicano es el Tipo de Operación SUNAT (catálogo 51). ** Pendiente definir si es editable por el operador o lo fija el sistema (ver Brecha 3 de TPSC-RE-FU-005). **
- ** Brecha bloqueante — Datos fiscales SUNAT del producto (código SUNAT, unidad de medida SUNAT, afectación al IGV por línea) inexistentes en el catálogo de productos actual; obligatorios para emitir el CPE. Puede detener el desarrollo del flujo de facturación Perú si no se resuelve antes (ver Brecha 1 de TPSC-RE-FU-005). **
- Fundamento normativo: Reglamento de Comprobantes de Pago, R.S. 097-2012/SUNAT y modificatorias; formato UBL 2.1; Anexo 8 catálogos SUNAT; plazo de envío del ejemplar (R.S. 117-2022/SUNAT).
- El RUC del cliente se valida en el Catálogo de Clientes (TPSC-RE-FU-004); este módulo lo consume ya validado, no lo revalida.
- ** Brecha — Integración SUNAT/OSE-PSE para timbrado del CPE. El proveedor de timbrado para Perú está pendiente de definir (no se reutiliza el PAC de México). Brecha mayor documentada en Brecha 5 de TPSC-RE-FU-005. **
- ** Brecha — Datos legales de Golocaer S.A.C. Perú: RUC, domicilio fiscal, ubigeo, certificado digital, series de facturación. Pendiente recopilar y configurar. **
- ** Brecha — Régimen de Detracciones (SPOT) y Percepciones del IGV: confirmar con asesor contable peruano. **
- ** Pendiente — denominación canónica del rol operativo, transversal con México. **
- ** Pendiente — régimen tributario de Golocaer S.A.C. (en Perú no aplica el código 601 del SAT). **
- ** Pendiente — maquetas de Golocaer S.A.C. Perú no disponibles; criterios de detalle de modales se validarán contra ellas cuando lleguen. **
- ** Duda — Secuencia GRE↔factura en prepago: en la operación corriente de Golocaer la GRE se emite antes del traslado y la factura la referencia (observado en la muestra recibida). En el flujo de Factura por Adelantado la factura va antes del despacho, por lo que la GRE no existiría al facturar. Pendiente confirmar con el cliente cómo opera la GRE en el flujo prepago de R16 y si la factura por adelantado lleva o no referencia a GRE. **
