R16A-RE-FU-033		Notas de Crédito: Perú	Notas de Crédito	Yo como ** operador del área de Tesorería (denominación pendiente de resolver) **, quiero contar con un módulo de Notas de Crédito para clientes con Región Perú, donde pueda consultar las NCs por cliente y generar nuevas NCs electrónicas (CPE tipo 07) sobre una factura origen, capturando el motivo del catálogo 09 de SUNAT y los datos en modalidad por partidas o manual, para documentar anulaciones, devoluciones, descuentos y bonificaciones de Golocaer S.A.C. conforme a la normativa SUNAT y dejarlas disponibles para su aplicación en Validar Cobro.	El sistema debe contar con un módulo independiente de Notas de Crédito para Región Perú, aplicable exclusivamente a clientes Prepago en este release y operado por el área de Tesorería. El módulo expone una pantalla de consulta de NCs agrupada por cliente y un wizard de generación que permite seleccionar la factura origen, capturar el motivo (catálogo 09 SUNAT) y los datos de la NC (en modalidad por partidas o manual), confirmar con previsualización del PDF y emitir el documento electrónico timbrado (CPE tipo 07, UBL 2.1). El módulo se acopla a Validar Cobro: las NCs generadas quedan disponibles para su aplicación durante la asociación de cobros. La estructura funcional se reutiliza de la de México (R16A-RE-FU-032); las diferencias son los catálogos SUNAT, los campos fiscales del CPE y la mecánica de anulación peruana (Nota de Crédito y, como caso excepcional, Comunicación de Baja). ** Toda la mecánica fiscal SUNAT de esta fila está pendiente de validar con el asesor fiscal peruano. **			Propuesto					R16.4M-RE-FU-001	"## Aplica a
- Módulo independiente Notas de Crédito en PQF2, separado de Validar Cobro (módulos distintos; NC alimenta a Validar Cobro pero no es sub-módulo), para Región Perú y operado por el área de Tesorería.
- Clientes prepago en R16 (crédito fuera de scope este release).
- Pantalla principal de Consulta de NCs agrupada por cliente, con drill-down al detalle por cliente.
- Wizard de generación de 4 pasos: Paso 1 Buscar Factura, Paso 2 Capturar Datos, Paso 3 Confirmar (con previsualización del PDF antes de timbrar), Paso 4 NC Emitida (misma vista que el detalle de cualquier NC ya generada).
- Selección obligatoria de cliente y de UNA factura electrónica vigente de prepago como origen de la NC.
- Motivo de la NC desde el catálogo 09 de SUNAT (11 motivos: 01 anulación, 02 anulación por error en el RUC, 03 corrección por error en la descripción, 04 descuento global, 05 descuento por ítem, 06 devolución total, 07 devolución por ítem, 08 bonificación, 09 disminución en el valor, 10 otros conceptos, 13 ajuste por crédito fiscal). ** Pendiente confirmar con el cliente qué motivos del catálogo 09 se habilitan en R16. **
- Dos modalidades de captura según el motivo: por partidas (devoluciones y descuento por ítem, con herencia de la tabla de partidas y cálculo automático del monto) y manual (descuento global, bonificación, disminución de valor, otros, con captura libre de monto y sustento).
- Anulación de una factura vía Nota de Crédito con motivo 01 (Anulación de la operación), plazo SUNAT de 10 días hábiles. La anulación vía NC deja la operación en cero.
- Comunicación de Baja como mecanismo alterno de anulación (caso excepcional): aplicable dentro de los 7 días calendario desde la emisión y recomendada para comprobantes no entregados al cliente. ** Pendiente confirmar con el cliente si se implementa la Comunicación de Baja en PQF2 para Perú o si la anulación se maneja únicamente vía Nota de Crédito. **
- Armado del XML conforme a la NC electrónica SUNAT (UBL 2.1): InvoiceTypeCode/tipo 07, serie F (hereda del comprobante modificado), referencia obligatoria al comprobante afectado (serie-correlativo) vía cbc:ReferenceID y cac:BillingReference, cbc:ResponseCode con el código del catálogo 09 y cbc:Description con el sustento del motivo. ** El origen del texto del sustento (cbc:Description) — auto-generado desde motivo/partidas o capturado por el usuario — queda pendiente de validar. **
- Timbrado del CPE ante SUNAT con feedback visual (en progreso / éxito / error) y previsualización del PDF antes de timbrar. ** La modalidad de emisión electrónica ante SUNAT está pendiente de definir (ver R16A-RE-FU-029); no se da por hecho el uso de un OSE ni se reutiliza el PAC de México. **
- Envío automático del correo al cliente al timbrar y opción de reenvío posterior, con PDF y XML adjuntos.
- Soporte multi-divisa: la NC hereda la moneda y el tipo de cambio de la factura origen (fecha de emisión del comprobante modificado), no el del día de la NC, conforme a SUNAT. La moneda base es PEN.
- Foliado por serie y correlativo SUNAT por la empresa emisora Golocaer S.A.C.
- Persistencia y conservación de los XML timbrados por 5 años (R.S. 117-2017/SUNAT).
- Acoplamiento uni-direccional con Validar Cobro: las NCs vigentes quedan disponibles automáticamente en el Paso 2 del wizard de Validar Cobro para aplicación a cobros nuevos del mismo cliente.

## No aplica a
- Módulo NC para Región México: se documenta en R16A-RE-FU-032.
- NCs para clientes de CRÉDITO (fuera de scope R16).
- Diseño del PDF representativo de la NC (layout, tipografía, secciones, branding, formato visual): se documenta en requisito independiente (R16A-RE-FU-035). Esta fila cubre solo las funciones y la información del módulo.
- Conceptos del SAT que no existen en SUNAT: TipoRelacion 01, UsoCFDI G02, Método de Pago PUE, Folio Fiscal UUID, cancelación SAT con c_MotivoCancelacion. En Perú la referencia al comprobante es por serie-correlativo y el motivo es del catálogo 09.
- Cancelación SAT condicionada a ""totalidad + mismo mes"" (mecánica mexicana): en Perú la anulación se hace vía NC motivo 01 (10 días hábiles) o Comunicación de Baja (7 días, no entregada).
- Flujo de Devolución de Dinero al cliente: no contemplado en R16.
- Generación o cancelación de NCs desde Validar Cobro (el acoplamiento es uni-direccional).
- Políticas de autorización por monto: ** pendiente (equivalente a PMO #54 de México). **"	"## Reglas de Negocio

** Nota transversal: toda la mecánica fiscal SUNAT de esta fila (catálogo 09, campos del CPE tipo 07, plazos, anulación vía NC y Comunicación de Baja, tipo de cambio, conservación) está sujeta a validación con el asesor fiscal peruano de PROQUIFA antes de implementarse. **

Regla 1 — Módulo independiente operado por Tesorería para prepago Perú
El módulo de Notas de Crédito es independiente de Validar Cobro y lo opera el área de Tesorería. En R16 aplica solo a clientes prepago de Región Perú. La empresa emisora es siempre Golocaer S.A.C.

Regla 2 — Política PROQUIFA: NC antes que devolución de dinero
Se mantiene la política de PROQUIFA de privilegiar la emisión de una Nota de Crédito antes que la devolución de dinero al cliente. En Perú, además, la NC es el mecanismo principal de anulación/corrección (la Comunicación de Baja es excepcional; ver Regla 8).

Regla 3 — Solo facturas vigentes prepago pueden originar NC
Solo una factura electrónica vigente (CPE tipo 01 con constancia de recepción aceptada por SUNAT) de un cliente prepago puede originar una NC. No se permite generar NC sobre comprobantes dados de baja, rechazados o ya anulados.

Regla 4 — Selección por partidas para devoluciones y descuento por ítem
Para los motivos del catálogo 09 que operan a nivel de línea (06 devolución total, 07 devolución por ítem, 05 descuento por ítem), la captura es por partidas: el sistema hereda la tabla de partidas del comprobante origen y el usuario edita la cantidad/monto por partida. El sistema calcula el monto de la NC automáticamente y arma las líneas (cac:CreditNoteLine) heredando del comprobante original.

Regla 5 — Modalidad manual para descuento global, bonificación y otros
Para los motivos que operan a nivel documento (04 descuento global, 08 bonificación, 09 disminución en el valor, 10 otros conceptos), la captura es manual: el usuario ingresa el monto total y un concepto/sustento de texto libre, sin tabla de partidas.

Regla 6 — Motivo desde el catálogo 09 de SUNAT
El motivo de la NC se selecciona del catálogo 09 de SUNAT y determina la modalidad de captura (por partidas o manual) y el código que viaja en el XML (cbc:ResponseCode). ** Pendiente confirmar con el cliente qué motivos del catálogo 09 se habilitan en R16 (de los 11 disponibles: 01, 02, 03, 04, 05, 06, 07, 08, 09, 10, 13). **

Regla 7 — Campos fiscales del XML (CPE tipo 07, UBL 2.1)
El XML de la NC se arma con: InvoiceTypeCode/tipo 07, serie F heredada del comprobante modificado, referencia obligatoria al comprobante afectado por serie-correlativo (cbc:ReferenceID + cac:BillingReference), cbc:ResponseCode (código del catálogo 09) y cbc:Description (sustento del motivo). NO se usan conceptos del SAT (TipoRelacion, UsoCFDI, MetodoPago, FormaPago, UUID). ** El origen del texto del sustento (cbc:Description) — auto-generado desde motivo/partidas o capturado por el usuario — queda pendiente de validar. La etiqueta y valores del Régimen Tributario (emisor Golocaer S.A.C. y receptor heredado del catálogo de clientes) quedan pendientes de validar. **

Regla 8 — Anulación: Nota de Crédito (principal) y Comunicación de Baja (excepcional)
La anulación de una factura se realiza preferentemente vía Nota de Crédito con motivo 01 (Anulación de la operación), plazo SUNAT de 10 días hábiles, que deja la operación en cero. Como mecanismo alterno y excepcional, la Comunicación de Baja aplica dentro de los 7 días calendario desde la emisión y está recomendada para comprobantes no entregados al cliente. En el flujo prepago la factura se envía al cliente en Validar Cobro, por lo que el camino natural es la NC; la Baja sería excepcional. ** Pendiente confirmar con el cliente si se implementa la Comunicación de Baja en PQF2 para Perú, o si la anulación se maneja únicamente vía Nota de Crédito. La obligatoriedad del criterio ""no entregada al cliente"" para la Baja debe confirmarse con el asesor fiscal peruano. **

Regla 9 — Foliado consecutivo por empresa con serie SUNAT
La NC se folia con serie (formato alfanumérico de 4 caracteres, ej. ""FC01"") y correlativo de 8 dígitos, consecutivo por la empresa emisora Golocaer S.A.C., conforme a la numeración SUNAT. La serie corresponde al tipo de comprobante modificado (factura → serie con prefijo F).

Regla 10 — Límite temporal para generar NC
La NC debe emitirse dentro del plazo SUNAT aplicable según el motivo: motivo 01 (anulación) hasta 10 días hábiles; motivos 02 y 03 (anulación por error en RUC, corrección de descripción) hasta 15 días hábiles. ** Plazos del resto de motivos del catálogo 09 a confirmar con el asesor fiscal peruano. **

Regla 11 — NCs históricas no se importan
Las NCs previas a la puesta en marcha del módulo no se importan; el módulo opera sobre comprobantes emitidos desde su habilitación.

Regla 12 — Inmutabilidad post-timbrado
Una vez timbrada la NC ante SUNAT, es inmutable: no se re-timbra ni se edita. Un error en una NC timbrada se corrige conforme a la normativa SUNAT aplicable.

Regla 13 — NCs disponibles para aplicación en Validar Cobro
Las NCs vigentes (timbradas y no aplicadas) quedan disponibles automáticamente en el Paso 2 del wizard de Validar Cobro para su aplicación a cobros nuevos del mismo cliente. El acoplamiento es uni-direccional: desde Validar Cobro no se generan ni cancelan NCs. ** Pendiente — aplicación de NC en moneda extranjera dentro de Validar Cobro: confirmar si una NC puede aplicarse a un documento de moneda distinta y, en ese caso, qué tipo de cambio se usa para la conversión en el momento de la aplicación. Si la NC y el documento destino son de la misma moneda, la aplicación es directa (sin conversión). **

Regla 14 — Conservación XML 5 años
Los XML timbrados de las NCs (junto con sus constancias y representaciones gráficas) se conservan por 5 años, conforme a la R.S. 117-2017/SUNAT, contados a partir del 1 de enero del año siguiente al de emisión.

Regla 15 — Destinatarios del correo de envío de la NC
El correo de envío de la NC presenta: CONTACTO (correo del contacto del cliente), CC (editable), ASUNTO pre-rellenado con el folio de la NC (serie-correlativo), adjuntos PDF y XML (no removibles) y un cuerpo de mensaje editable. El envío se dispara al timbrar y puede reenviarse después.

Regla 16 — Manejo de errores de timbrado SUNAT
Cuando el timbrado de la NC falla (validación, servicio indisponible, datos inválidos, error SUNAT), el sistema muestra un mensaje de error con el detalle, mismo comportamiento operativo que la facturación (R16A-RE-FU-029). ** La modalidad de emisión electrónica ante SUNAT está pendiente de definir; no se da por hecho el uso de un OSE ni se reutiliza el PAC de México. **

## Riesgos

Riesgo 1 — Mecánica fiscal SUNAT no validada
Toda la mecánica fiscal de la NC peruana (catálogo 09, campos del CPE tipo 07, plazos, anulación, tipo de cambio, conservación) está basada en investigación normativa pero no validada con el asesor fiscal peruano de PROQUIFA. Una desviación respecto a la práctica real de SUNAT obligaría a rehacer el módulo.

Riesgo 2 — Elección incorrecta del mecanismo de anulación
Usar Comunicación de Baja cuando correspondía NC (o viceversa), o fuera del plazo de 7 días, provoca rechazo de SUNAT y retrabajo. Si el sistema no controla el plazo y las condiciones, el operador podría intentar un mecanismo inválido.

Riesgo 3 — Brecha de timbrado SUNAT no resuelta
La emisión de NCs depende de la modalidad de emisión electrónica que defina Golocaer ante SUNAT, pendiente de definir (brecha mayor compartida con R16A-RE-FU-029). Mientras no se resuelva, no es posible timbrar NCs.

Riesgo 4 — Ajuste de declaraciones ya presentadas
Si la factura afectada por la NC ya fue incluida en una declaración mensual de IGV-Renta, podría requerirse ajuste/rectificación del período. Si el proceso no contempla esto, puede haber inconsistencia fiscal.

Riesgo 5 — Motivos del catálogo 09 no acotados
Si no se confirma qué motivos del catálogo 09 se habilitan, el módulo podría exponer motivos que PROQUIFA no usa (o faltar alguno necesario), generando NCs mal clasificadas.

Riesgo 6 — Tipo de cambio en aplicación de NC de distinta moneda
Si una NC en moneda extranjera se aplica en Validar Cobro a un documento de moneda distinta y no está definido qué tipo de cambio usar en la conversión, podría generarse una diferencia mal calculada en el saldo (ver Regla 13).

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — PANTALLA PRINCIPAL DE CONSULTA (AGRUPADA POR CLIENTE)
═══════════════════════════════════════════════════════════════

Criterio A1 — Consulta agrupada por cliente
Dado que el usuario entra al módulo de Notas de Crédito de Perú,
Cuando el sistema renderiza la pantalla principal,
Entonces deberá mostrar el título ""CONSULTA / NOTA DE CRÉDITO"", los filtros Cliente / Moneda / Fecha, el botón ""+ NUEVA NOTA DE CRÉDITO"" y una tabla agrupada por cliente con columnas: # / Cliente / Total NC / Vigentes / Por Aplicar / Monto Total / Moneda, más el pie con total de clientes y total de NCs.

Criterio A2 — Filtros de la consulta
Dado que el usuario usa los filtros,
Cuando selecciona Cliente, Moneda o Fecha,
Entonces el sistema deberá filtrar los resultados en consecuencia.

═══════════════════════════════════════════════════════════════
SECCIÓN B — DETALLE POR CLIENTE (DRILL-DOWN)
═══════════════════════════════════════════════════════════════

Criterio B1 — Drill-down al detalle del cliente
Dado que el usuario selecciona un cliente de la consulta,
Cuando abre su detalle,
Entonces el sistema deberá listar las NCs de ese cliente con sus datos (folio serie-correlativo, fecha, monto, estado vigente/aplicada, moneda) y el comprobante origen de cada una.

═══════════════════════════════════════════════════════════════
SECCIÓN C — WIZARD PASO 1: BUSCAR FACTURA
═══════════════════════════════════════════════════════════════

Criterio C1 — Selección de cliente y factura origen
Dado que el usuario presiona ""+ NUEVA NOTA DE CRÉDITO"",
Cuando entra al Paso 1 del wizard,
Entonces el sistema deberá permitir seleccionar el cliente y UNA factura electrónica vigente de prepago como origen, mostrando el listado de facturas relacionables con su folio (serie-correlativo), fecha, monto y moneda.

Criterio C2 — Barra de pasos del wizard
Dado que el usuario está en el wizard,
Cuando el sistema renderiza la barra de pasos,
Entonces deberá mostrar: 1. Buscar Factura, 2. Capturar Datos, 3. Confirmar, 4. NC Emitida, resaltando el paso activo.

═══════════════════════════════════════════════════════════════
SECCIÓN D — WIZARD PASO 2: CAPTURAR DATOS (COMÚN A AMBAS MODALIDADES)
═══════════════════════════════════════════════════════════════

Criterio D1 — Barra de FACTURA ORIGEN heredada
Dado que el usuario está en el Paso 2,
Cuando el sistema muestra los datos del comprobante origen,
Entonces deberá presentar: folio (serie-correlativo), Tipo CPE (01 – Factura), RUC, Razón Social, Fecha Emisión, Subtotal, IGV, Total y Estado. NO se muestra UUID (no existe en SUNAT).

Criterio D2 — Selección de Motivo (catálogo 09)
Dado que el usuario captura los datos de la NC,
Cuando selecciona el Motivo,
Entonces el sistema deberá ofrecer los motivos habilitados del catálogo 09 de SUNAT y, según el motivo, activar la modalidad por partidas o manual. NO se muestran los campos Uso CFDI ni Tipo Relación (conceptos del SAT). ** Motivos habilitados pendientes de confirmar. **

Criterio D3 — Campo de sustento del motivo (cbc:Description)
Dado que el usuario captura los datos de la NC,
Cuando el sistema arma el documento,
Entonces deberá registrar el sustento del motivo a nivel documento (cbc:Description, obligatorio en SUNAT). ** El origen del texto (auto-generado desde motivo/partidas o capturado por el usuario) queda pendiente de validar. **

Criterio D4 — Campo Observaciones
Dado que el usuario captura los datos,
Cuando completa el formulario,
Entonces el sistema deberá ofrecer un campo Observaciones opcional (texto libre).

═══════════════════════════════════════════════════════════════
SECCIÓN E — MODALIDAD POR PARTIDAS (DEVOLUCIONES / DESCUENTO POR ÍTEM)
═══════════════════════════════════════════════════════════════

Criterio E1 — Herencia de la tabla de partidas
Dado que el motivo es de devolución o descuento por ítem,
Cuando el sistema muestra la modalidad por partidas,
Entonces deberá heredar la tabla del comprobante origen con columnas: checkbox, Catálogo, Producto, Cant. Facturada, Cant. NC (editable), Precio Unitario, Subtotal, IGV, Total.

Criterio E2 — Edición de cantidad por partida
Dado que el usuario selecciona partidas,
Cuando captura la Cant. NC de cada una (0, parcial o total),
Entonces el sistema deberá calcular automáticamente Subtotal, IGV y Total por partida y el Total NC del documento. Las partidas con Cant. NC = 0 no se incluyen.

Criterio E3 — Composición de las líneas de la NC
Dado que el usuario confirma las partidas seleccionadas,
Cuando el sistema arma el XML,
Entonces cada partida con Cant. NC > 0 deberá generar una línea (cac:CreditNoteLine) heredando del concepto original (código de producto, descripción, unidad, valor unitario, afectación al IGV) y recalculando importes según la cantidad capturada.

═══════════════════════════════════════════════════════════════
SECCIÓN F — MODALIDAD MANUAL (DESCUENTO GLOBAL / BONIFICACIÓN / OTROS)
═══════════════════════════════════════════════════════════════

Criterio F1 — Captura de monto y concepto
Dado que el motivo es de descuento global, bonificación, disminución de valor u otros,
Cuando el sistema muestra la modalidad manual,
Entonces deberá ofrecer el campo Monto Total NC (editable) y el campo Concepto (obligatorio, texto libre que describe el motivo), sin tabla de partidas.

Criterio F2 — Cálculo de impuestos en modalidad manual
Dado que el usuario captura el Monto Total NC,
Cuando el sistema calcula el documento,
Entonces deberá descomponer el IGV (18%) correspondiente y reflejar Subtotal NC, IGV y TOTAL NC en el resumen.

═══════════════════════════════════════════════════════════════
SECCIÓN G — ANULACIÓN: NOTA DE CRÉDITO Y COMUNICACIÓN DE BAJA
═══════════════════════════════════════════════════════════════

Criterio G1 — Anulación vía NC con motivo 01
Dado que el usuario requiere anular una factura,
Cuando selecciona el motivo 01 (Anulación de la operación) dentro del plazo de 10 días hábiles,
Entonces el sistema deberá generar una NC que deje la operación en cero, heredando la totalidad de las partidas del comprobante origen.

Criterio G2 — Comunicación de Baja (excepcional)
Dado que se requiere anular un comprobante dentro de los 7 días calendario desde la emisión y (preferentemente) no entregado al cliente,
Cuando el usuario opta por la Comunicación de Baja,
Entonces el sistema deberá permitir comunicar la baja a SUNAT. ** Pendiente confirmar si la Comunicación de Baja se implementa en PQF2 para Perú; en caso afirmativo, validar las condiciones exactas (plazo, ""no entregada"") con el asesor fiscal peruano. **

Criterio G3 — Reemplazo de la cancelación SAT
Dado que en México la cancelación de la factura origen estaba condicionada a ""totalidad + mismo mes"",
Cuando se opera en Perú,
Entonces esa mecánica NO aplica: se reemplaza por la anulación vía NC motivo 01 (10 días hábiles) o la Comunicación de Baja (7 días, excepcional).

═══════════════════════════════════════════════════════════════
SECCIÓN H — CAMPOS FISCALES DEL XML (CPE TIPO 07, UBL 2.1)
═══════════════════════════════════════════════════════════════

Criterio H1 — Tipo de comprobante y versión
Dado que el sistema arma el XML,
Cuando asigna el tipo de documento,
Entonces deberá consignar InvoiceTypeCode = 07 (Nota de Crédito Electrónica) en formato UBL 2.1. En el Resumen Fiscal se muestra ""07 – Nota de Crédito Electrónica"" y ""UBL 2.1"" (no ""E – Egreso"" ni ""CFDI 4.0"").

Criterio H2 — Referencia al comprobante afectado
Dado que el sistema arma el XML,
Cuando referencia la factura origen,
Entonces deberá consignar el comprobante afectado por serie-correlativo (cbc:ReferenceID + cac:BillingReference), no por UUID.

Criterio H3 — Código y sustento del motivo
Dado que el sistema arma el XML,
Cuando consigna el motivo,
Entonces deberá incluir cbc:ResponseCode (código del catálogo 09) y cbc:Description (sustento), ambos a nivel documento.

Criterio H4 — Régimen Tributario obtenido de configuración y catálogo de clientes
Dado que el sistema arma el XML,
Cuando asigna el régimen,
Entonces el régimen del emisor se obtiene de la configuración de la empresa emisora (Golocaer S.A.C.) y el del receptor se hereda del catálogo de clientes; no se captura en el módulo de NC. ** Etiqueta y valores del Régimen Tributario peruano pendientes de validar. **

Criterio H5 — Impuestos calculados automáticamente (IGV)
Dado que el sistema arma el XML,
Cuando calcula impuestos,
Entonces deberá calcular el IGV (18%) automáticamente sobre el valor de la NC. NO aplican Uso CFDI, Método de Pago ni Forma de Pago (conceptos del SAT).

Criterio H6 — Moneda y tipo de cambio
Dado que el sistema arma el XML de una NC en moneda extranjera,
Cuando consigna el tipo de cambio,
Entonces deberá usar la moneda del comprobante origen y heredar el tipo de cambio de la FECHA DE EMISIÓN DE LA FACTURA ORIGEN (no el del día de la NC), conforme a la normativa SUNAT (la NC es una modificación de la operación original, no una operación nueva). Este criterio aplica a todos los motivos del catálogo 09 (anulación, descuentos, devoluciones, etc.). ** A validar con el asesor fiscal peruano. **

═══════════════════════════════════════════════════════════════
SECCIÓN I — WIZARD PASO 3: CONFIRMAR + PREVISUALIZACIÓN
═══════════════════════════════════════════════════════════════

Criterio I1 — Resumen fiscal y previsualización
Dado que el usuario avanza al Paso 3,
Cuando el sistema muestra la confirmación,
Entonces deberá presentar el Resumen Fiscal (Tipo Comprobante 07, UBL 2.1, Ref. Comprobante serie-correlativo, Régimen, Moneda, Subtotal NC, IGV, TOTAL NC) y permitir previsualizar el PDF representativo antes de timbrar.

Criterio I2 — Acción Regresar
Dado que el usuario está en el Paso 3 y aún no timbra,
Cuando presiona Regresar,
Entonces el sistema deberá permitir volver al Paso 2 a corregir.

═══════════════════════════════════════════════════════════════
SECCIÓN J — TIMBRADO Y FEEDBACK
═══════════════════════════════════════════════════════════════

Criterio J1 — Timbrado ante SUNAT con feedback
Dado que el usuario confirma la emisión,
Cuando el sistema timbra la NC ante SUNAT,
Entonces deberá mostrar feedback visual de progreso (loader ""en proceso""), y al concluir un modal de éxito (""¡Has generado una nota de crédito exitosamente!"") o de error con el detalle.

Criterio J2 — Manejo de error de timbrado
Dado que el timbrado falla,
Cuando SUNAT retorna error,
Entonces el sistema deberá mostrar el detalle y permitir reintentar, sin generar un documento timbrado inválido.

═══════════════════════════════════════════════════════════════
SECCIÓN K — PASO 4: NC EMITIDA (= VISTA DE DETALLE DE NC)
═══════════════════════════════════════════════════════════════

Criterio K1 — Vista de NC emitida
Dado que la NC se timbró exitosamente,
Cuando el sistema muestra el Paso 4,
Entonces deberá presentar la NC emitida con su folio (serie-correlativo) timbrado, sus datos fiscales y las acciones disponibles (descargar PDF/XML, enviar/reenviar). Esta vista es la misma que el detalle de cualquier NC ya generada.

═══════════════════════════════════════════════════════════════
SECCIÓN L — ENVÍO Y REENVÍO DE CORREO
═══════════════════════════════════════════════════════════════

Criterio L1 — Modal de envío
Dado que el usuario envía la NC,
Cuando el sistema abre el modal ""ENVIAR NOTA DE CRÉDITO"",
Entonces deberá mostrar: CONTACTO (correo del contacto del cliente), CC (editable), ASUNTO pre-rellenado con el folio de la NC, adjuntos PDF y XML (no removibles), cuerpo de mensaje editable, y botones CANCELAR / ENVIAR.

Criterio L2 — Reenvío posterior
Dado que una NC ya fue emitida,
Cuando el usuario solicita reenviarla,
Entonces el sistema deberá permitir reenviar el correo con los mismos adjuntos.

═══════════════════════════════════════════════════════════════
SECCIÓN M — ACOPLAMIENTO CON VALIDAR COBRO
═══════════════════════════════════════════════════════════════

Criterio M1 — NCs vigentes disponibles en Validar Cobro
Dado que existen NCs vigentes (timbradas, no aplicadas) de un cliente,
Cuando el usuario opera el Paso 2 de Validar Cobro para ese cliente,
Entonces el sistema deberá poner esas NCs a disposición para aplicación a cobros nuevos.

Criterio M2 — Acoplamiento uni-direccional
Dado que el acoplamiento entre NC y Validar Cobro es uni-direccional,
Cuando el usuario está en Validar Cobro,
Entonces NO deberá poder generar ni cancelar NCs desde ahí (solo aplicarlas).

Criterio M3 — Aplicación de NC en moneda extranjera
Dado que una NC vigente se aplica a un documento en Validar Cobro,
Cuando la NC y el documento destino son de la misma moneda,
Entonces la aplicación deberá ser directa (sin conversión). ** Si la NC y el documento destino son de monedas distintas, queda pendiente confirmar si se permite la aplicación y, en ese caso, qué tipo de cambio se usa para la conversión en el momento de aplicar (ver Regla 13). **

═══════════════════════════════════════════════════════════════
SECCIÓN N — MULTI-DIVISA, FOLIADO, DECIMALES
═══════════════════════════════════════════════════════════════

Criterio N1 — Multi-divisa
Dado que el comprobante origen está en una moneda distinta de PEN,
Cuando se genera la NC,
Entonces el sistema deberá preservar la moneda del comprobante origen y heredar su tipo de cambio (el de la fecha de emisión de la factura origen, ver Criterio H6).

Criterio N2 — Foliado SUNAT
Dado que se timbra una NC,
Cuando el sistema asigna el folio,
Entonces deberá usar serie (prefijo según el comprobante modificado) y correlativo consecutivo por la empresa emisora Golocaer S.A.C.

Criterio N3 — Conservación
Dado que una NC fue timbrada,
Cuando el sistema la archiva,
Entonces deberá conservar el XML, su constancia y su representación gráfica por 5 años (R.S. 117-2017/SUNAT), contados desde el 1 de enero del año siguiente al de emisión."								"- Fila para el módulo de Notas de Crédito de la Región Perú, contraparte de R16A-RE-FU-032 (México). El PDF representativo de la NC se documenta aparte en R16A-RE-FU-035.
- ** Nota transversal: toda la mecánica fiscal SUNAT de esta fila requiere validación con el asesor fiscal peruano de PROQUIFA antes de implementarse. **
- Estructura funcional idéntica a México: módulo independiente operado por Tesorería, consulta agrupada por cliente, wizard de 4 pasos, dos modalidades (por partidas / manual), acoplamiento uni-direccional con Validar Cobro, multi-divisa, foliado, conservación.
- Cambios fiscales vs México (verificados con investigación normativa SUNAT, pendientes de validación): tipo de comprobante E→07; versión CFDI 4.0→UBL 2.1; RFC→RUC; IVA→IGV (18%); referencia por UUID→serie-correlativo; motivo SAT→catálogo 09 SUNAT (11 motivos); MXN→PEN.
- Se ELIMINAN (conceptos del SAT): UUID, Uso CFDI, Tipo Relación, Método de Pago, Forma de Pago.
- Se AGREGAN (propios de SUNAT): cbc:ResponseCode (código del catálogo 09) y cbc:Description (sustento del motivo, obligatorio a nivel documento).
- Régimen Tributario: se obtiene de la configuración de la empresa emisora (Golocaer S.A.C.) y del catálogo de clientes (receptor), no se captura en el módulo (igual que México). ** Etiqueta/valores peruanos pendientes de validar. **
- Tipo de cambio (diferencia clave vs México): en Perú la NC en moneda extranjera HEREDA el tipo de cambio de la factura origen (fecha de emisión del comprobante modificado), NO el del día de la NC, porque fiscalmente es un ajuste a la operación original (Oficio SUNAT 024-2000). En México, en cambio, la NC usa el TC del día del timbrado (validado por el SAT contra el FIX) y la diferencia se reconoce como ganancia/pérdida cambiaria. Esta diferencia es intencional entre ambas regiones.
- Anulación: dos mecanismos — Nota de Crédito con motivo 01 (principal, 10 días hábiles) y Comunicación de Baja (excepcional, 7 días, no entregada). ** Pendiente confirmar si la Baja se implementa en PQF2 Perú. **
- Conservación: 5 años (R.S. 117-2017/SUNAT), desde el 1 de enero del año siguiente al de emisión (coincide en duración con México).
- ** Pendiente — qué motivos del catálogo 09 se habilitan en R16. **
- ** Pendiente — origen del texto del sustento (cbc:Description): auto-generado o capturado. **
- ** Pendiente — aplicación de NC en moneda extranjera en Validar Cobro: si se permite aplicar a documento de distinta moneda y qué TC se usa en la conversión. **
- ** Pendiente — modalidad de emisión electrónica ante SUNAT (ver R16A-RE-FU-029); no se da por hecho un OSE ni se reutiliza el PAC de México. **
- ** Pendiente — maquetas de Notas de Crédito Perú no disponibles; el detalle se validará contra ellas cuando lleguen (esta fila se basó en el análisis del flujo mexicano). **"									