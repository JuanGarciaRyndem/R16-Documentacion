R16A-RE-FU-035		"Diseño y generación de Documentos: NDC Perú
"	Notas de Crédito	Yo como ** operador del área de Tesorería (denominación pendiente de resolver) **, quiero que al confirmar el timbrado de una Nota de Crédito para un cliente con Región Perú el sistema genere el CPE tipo 07 (UBL 2.1) y su representación impresa (PDF) con el branding de Golocaer S.A.C. y los datos exigidos por SUNAT, para entregar al cliente un documento válido y verificable y conservar el XML conforme a la normativa.	El sistema debe generar una Nota de Crédito electrónica (CPE tipo 07, UBL 2.1) y su PDF representativo (representación impresa) al confirmar el timbrado desde el wizard del módulo Notas de Crédito de Perú (R16A-RE-FU-033), conforme a la normativa SUNAT vigente y con el branding de Golocaer S.A.C. El módulo de Notas de Crédito es el único disparador de la generación, y la NC referencia obligatoriamente el comprobante origen por serie-correlativo. El XML timbrado y el PDF se persisten en PQF2, se conservan por 5 años (R.S. 117-2017/SUNAT) y se envían al cliente con copia al ESAC tras el timbrado exitoso. Esta fila documenta la generación y la representación impresa; las funciones del módulo (consulta, wizard, modalidades, acoplamiento) están en R16A-RE-FU-033. La estructura se reutiliza de la de México (R16A-RE-FU-034) con diferencias significativas en lo fiscal SUNAT. ** Toda la mecánica fiscal SUNAT de esta fila está pendiente de validar con el asesor fiscal peruano. **			Propuesto					R16.4M-RE-FU-001	"## Aplica a
- Generación del CPE tipo 07 (Nota de Crédito Electrónica, UBL 2.1) y su PDF representativo al confirmar el timbrado desde el Paso 3 del wizard del módulo NC de Perú.
- Región Perú exclusivamente. Empresa emisora: Golocaer S.A.C. (RUC 20612772941).
- Estructura XML conforme a la NC electrónica SUNAT (UBL 2.1): cabecera con InvoiceTypeCode/tipo 07, Emisor (Golocaer S.A.C.), Receptor (cliente, datos heredados del catálogo de clientes), referencia obligatoria al comprobante afectado por serie-correlativo (cbc:ReferenceID + cac:BillingReference), cbc:ResponseCode (código del catálogo 09) y cbc:Description (sustento del motivo), líneas (cac:CreditNoteLine heredadas de partidas o concepto manual según modalidad), impuestos (IGV 18%) y totales (Valor Venta, IGV, Importe Total) en la moneda del comprobante origen.
- Multi-divisa: hereda la moneda del comprobante origen (no editable) y el tipo de cambio de la FECHA DE EMISIÓN de la factura origen (no el del día de la NC), conforme a SUNAT.
- Numeración: serie (prefijo según el comprobante modificado, ej. con F) y correlativo de 8 dígitos, consecutivo por Golocaer S.A.C.
- Timbrado del CPE ante SUNAT; el documento queda respaldado por la firma digital del emisor y el valor resumen/hash (no hay UUID ni sello SAT). ** Modalidad de emisión electrónica ante SUNAT pendiente de definir (ver R16A-RE-FU-029); no se da por hecho el uso de un OSE ni se reutiliza el PAC de México. **
- PDF representativo (representación impresa SUNAT) con identidad visual de Golocaer S.A.C. (logo, paleta corporativa), consistente con la Factura Perú (R16A-RE-FU-022) y demás documentos peruanos del proyecto.
- PDF con la información completa de la NC: datos del emisor (incl. RUC), datos del receptor (incl. RUC o documento), datos del comprobante (tipo 07, serie-correlativo, fecha), motivo (código y descripción del catálogo 09), referencia al comprobante origen (serie-correlativo), líneas o concepto manual con importes e IGV desglosado, totales (Valor Venta, IGV, Importe Total), código QR de verificación SUNAT y la leyenda obligatoria de representación impresa.
- Persistencia de XML timbrado y PDF en PQF2 con conservación de 5 años (R.S. 117-2017/SUNAT).
- Envío de la NC por correo al cliente con copia al ESAC tras timbrado exitoso, con PDF y XML adjuntos.
- Manejo de errores de timbrado consistente con la facturación Perú (R16A-RE-FU-029): mensaje al usuario; la NC no se persiste como vigente; reintento posterior.

## No aplica a
- Generación de NC para Región México: se documenta en R16A-RE-FU-034.
- Funciones del módulo NC (pantalla de consulta, wizard, modalidades de captura, acoplamiento con Validar Cobro): se documentan en R16A-RE-FU-033.
- Conceptos del SAT que no existen en SUNAT: TipoDeComprobante E, UsoCFDI G02, TipoRelacion 01, MetodoPago PUE, FormaPago, Folio Fiscal UUID, sello digital SAT, cadena complementaria, c_MotivoCancelacion.
- Cancelación SAT condicionada a ""totalidad + mismo mes"": en Perú la anulación se hace vía NC motivo 01 o Comunicación de Baja (ver R16A-RE-FU-033).
- PAC TurboPac: no se reutiliza la solución de México."	"## Reglas de Negocio

** Nota transversal: toda la mecánica fiscal SUNAT de esta fila (CPE tipo 07, campos UBL 2.1, representación impresa, QR, leyenda, tipo de cambio, conservación) está sujeta a validación con el asesor fiscal peruano de PROQUIFA antes de implementarse. **

Regla 1 — Detonante único módulo NC
La generación del CPE tipo 07 y su PDF se dispara exclusivamente al confirmar el timbrado desde el Paso 3 del wizard del módulo Notas de Crédito de Perú (R16A-RE-FU-033). No hay otro punto de generación.

Regla 2 — Tipo de comprobante 07 fijo
El documento se emite siempre como InvoiceTypeCode = 07 (Nota de Crédito Electrónica) en formato UBL 2.1. No se usa TipoDeComprobante E (concepto del SAT).

Regla 3 — Referencia obligatoria al comprobante afectado
El XML referencia obligatoriamente el comprobante origen por serie-correlativo (cbc:ReferenceID + cac:BillingReference), no por UUID. Una NC no puede emitirse sin comprobante de referencia.

Regla 4 — Motivo del catálogo 09 (código y sustento)
El XML consigna cbc:ResponseCode (código del catálogo 09) y cbc:Description (sustento del motivo), ambos a nivel documento. ** Origen del texto del sustento pendiente de validar (ver R16A-RE-FU-033). **

Regla 5 — Sin conceptos de método/forma de pago
La NC peruana no consigna Método de Pago ni Forma de Pago (conceptos del SAT). Solo aplica, cuando corresponde, la Condición de Pago heredada del comprobante origen.

Regla 6 — Moneda y tipo de cambio heredados de la factura origen
La NC hereda la moneda del comprobante origen (no editable). Cuando la moneda es extranjera, hereda el tipo de cambio de la FECHA DE EMISIÓN de la factura origen (no el del día de la NC), conforme a SUNAT (la NC es una modificación de la operación original).

Regla 7 — Líneas en modalidad por partidas
En modalidad por partidas, cada partida con Cant. NC > 0 genera una línea (cac:CreditNoteLine) heredando del concepto original (código de producto, descripción, unidad, valor unitario, afectación al IGV) y recalculando importes según la cantidad capturada.

Regla 8 — Concepto en modalidad manual
En modalidad manual, la NC lleva un único concepto con el monto y el sustento capturados, sin herencia de partidas.

Regla 9 — Impuestos calculados (IGV 18%)
El sistema calcula el IGV (18%) automáticamente sobre el valor de la NC. Cuando el comprobante origen tiene otros tributos (ISC u otros), se reflejan conforme a SUNAT solo si el comprobante modificado los tiene.

Regla 10 — Totales en moneda de la factura origen
Los totales (Valor Venta, IGV, Importe Total) se expresan en la moneda del comprobante origen, con la nomenclatura SUNAT.

Regla 11 — Foliado consecutivo por empresa
La NC se folia con serie (prefijo según el comprobante modificado) y correlativo de 8 dígitos, consecutivo por Golocaer S.A.C.

Regla 12 — Conservación XML 5 años
El XML timbrado, su constancia y su representación gráfica se conservan por 5 años (R.S. 117-2017/SUNAT), contados desde el 1 de enero del año siguiente al de emisión.

Regla 13 — Envío tras timbrado exitoso
Tras el timbrado exitoso, el sistema envía la NC por correo al cliente con copia al ESAC, con PDF y XML adjuntos.

Regla 14 — Identidad visual del PDF
El PDF usa el logo y la paleta corporativa de Golocaer S.A.C., consistente con la Factura Perú (R16A-RE-FU-022).

Regla 15 — Leyenda obligatoria y QR de la representación impresa
El PDF incluye el código QR de verificación SUNAT y la leyenda obligatoria de representación impresa (""Representación impresa de la Nota de Crédito Electrónica, consúltela en ...""). No lleva sello digital SAT ni cadena complementaria (conceptos del SAT); el respaldo es la firma digital del emisor y el valor resumen/hash.

Regla 16 — Manejo de errores de timbrado SUNAT
Cuando el timbrado falla, el sistema muestra el detalle del error, no persiste la NC como vigente y permite reintentar, mismo comportamiento que la facturación Perú (R16A-RE-FU-029).

Regla 17 — Anulación fuera de esta fila
La anulación de comprobantes (NC motivo 01 o Comunicación de Baja) se documenta en R16A-RE-FU-033; esta fila solo cubre la generación y representación impresa de la NC.

## Riesgos

Riesgo 1 — Mecánica fiscal SUNAT no validada
La estructura del CPE tipo 07, la representación impresa, el QR, la leyenda y el tipo de cambio están basados en investigación normativa pero no validados con el asesor fiscal peruano. Una desviación obligaría a rehacer la generación.

Riesgo 2 — Cálculo erróneo de impuestos al heredar/recalcular partidas
Si el recálculo de IGV por partida (modalidad por partidas) o la descomposición del monto manual no se hace correctamente, el documento timbrado podría salir con importes inconsistentes respecto al comprobante origen.

Riesgo 3 — Brecha de timbrado SUNAT no resuelta
La generación depende de la modalidad de emisión electrónica que defina Golocaer ante SUNAT, pendiente de definir (brecha compartida con R16A-RE-FU-029/033). Mientras no se resuelva, no es posible timbrar ni generar el PDF.

Riesgo 4 — Representación impresa no conforme
Si el PDF no incluye la leyenda obligatoria, el QR o la nomenclatura SUNAT correcta, la representación impresa podría no ser conforme y generar observaciones.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — DETONANTE
═══════════════════════════════════════════════════════════════

Criterio A1 — Disparo desde el módulo NC
Dado que el usuario confirma el timbrado en el Paso 3 del wizard del módulo NC de Perú,
Cuando se ejecuta la generación,
Entonces el sistema deberá generar el CPE tipo 07 (UBL 2.1) y su PDF representativo. No existe otro punto de disparo.

═══════════════════════════════════════════════════════════════
SECCIÓN B — CABECERA DEL XML
═══════════════════════════════════════════════════════════════

Criterio B1 — Campos fijos del comprobante
Dado que el sistema arma el XML,
Cuando consigna la cabecera,
Entonces deberá fijar InvoiceTypeCode = 07 y la versión UBL 2.1.

Criterio B2 — Serie, correlativo y fecha
Dado que el sistema arma el XML,
Cuando asigna la numeración,
Entonces deberá consignar serie (prefijo según el comprobante modificado), correlativo de 8 dígitos consecutivo por Golocaer S.A.C., y la fecha de emisión.

Criterio B3 — Moneda heredada del comprobante origen
Dado que el sistema arma el XML,
Cuando consigna la moneda,
Entonces deberá heredar la del comprobante origen (no editable).

Criterio B4 — Tipo de cambio de la factura origen cuando moneda extranjera
Dado que la NC es en moneda extranjera,
Cuando el sistema consigna el tipo de cambio,
Entonces deberá usar el de la FECHA DE EMISIÓN de la factura origen (no el del día de la NC), conforme a SUNAT.

═══════════════════════════════════════════════════════════════
SECCIÓN C — REFERENCIA AL COMPROBANTE AFECTADO
═══════════════════════════════════════════════════════════════

Criterio C1 — Referencia obligatoria por serie-correlativo
Dado que el sistema arma el XML,
Cuando referencia la factura origen,
Entonces deberá consignar el comprobante afectado por serie-correlativo (cbc:ReferenceID + cac:BillingReference). Sin referencia no se puede emitir.

Criterio C2 — Código y sustento del motivo
Dado que el sistema arma el XML,
Cuando consigna el motivo,
Entonces deberá incluir cbc:ResponseCode (catálogo 09) y cbc:Description (sustento), a nivel documento.

═══════════════════════════════════════════════════════════════
SECCIÓN D — EMISOR Y RECEPTOR
═══════════════════════════════════════════════════════════════

Criterio D1 — Datos del Emisor
Dado que el sistema arma el XML,
Cuando consigna el emisor,
Entonces deberá poner los datos de Golocaer S.A.C. (RUC, razón social, domicilio fiscal) y su Régimen Tributario (obtenido de la configuración de la empresa emisora).

Criterio D2 — Datos del Receptor
Dado que el sistema arma el XML,
Cuando consigna el receptor,
Entonces deberá heredar los datos del cliente del catálogo de clientes (RUC o documento, razón social). NO se consigna UsoCFDI (concepto del SAT).

═══════════════════════════════════════════════════════════════
SECCIÓN E — LÍNEAS MODALIDAD POR PARTIDAS
═══════════════════════════════════════════════════════════════

Criterio E1 — Una línea por partida con Cant. NC > 0
Dado que la NC es por partidas,
Cuando el sistema arma las líneas,
Entonces deberá generar una línea (cac:CreditNoteLine) por cada partida con Cant. NC > 0.

Criterio E2 — Herencia de datos del concepto original
Dado que se genera una línea por partida,
Cuando hereda los datos,
Entonces deberá tomar del concepto original el código de producto, descripción, unidad, valor unitario y afectación al IGV.

Criterio E3 — Recálculo según Cant. NC
Dado que se genera una línea por partida,
Cuando recalcula importes,
Entonces deberá ajustar valor de venta e IGV según la Cant. NC capturada.

═══════════════════════════════════════════════════════════════
SECCIÓN F — CONCEPTO MODALIDAD MANUAL
═══════════════════════════════════════════════════════════════

Criterio F1 — Único concepto en modalidad manual
Dado que la NC es manual,
Cuando el sistema arma las líneas,
Entonces deberá generar un único concepto con el monto y el sustento capturados, sin herencia de partidas.

═══════════════════════════════════════════════════════════════
SECCIÓN G — IMPUESTOS Y TOTALES
═══════════════════════════════════════════════════════════════

Criterio G1 — Cálculo automático de IGV
Dado que el sistema calcula impuestos,
Cuando arma el documento,
Entonces deberá calcular el IGV (18%) automáticamente; otros tributos (ISC u otros) solo si el comprobante origen los tiene.

Criterio G2 — Totales en moneda de la factura origen
Dado que el sistema arma los totales,
Cuando los expresa,
Entonces deberá usar la moneda del comprobante origen y la nomenclatura SUNAT (Valor Venta, IGV, Importe Total).

═══════════════════════════════════════════════════════════════
SECCIÓN H — TIMBRADO Y PERSISTENCIA
═══════════════════════════════════════════════════════════════

Criterio H1 — Timbrado ante SUNAT
Dado que el usuario confirma la emisión,
Cuando el sistema timbra la NC,
Entonces deberá timbrarla ante SUNAT y obtener la conformidad. El respaldo es la firma digital del emisor y el valor resumen/hash (no hay UUID ni sello SAT).

Criterio H2 — Persistencia XML y PDF
Dado que la NC se timbró,
Cuando el sistema la archiva,
Entonces deberá persistir el XML timbrado y el PDF en PQF2.

Criterio H3 — Manejo de errores de timbrado
Dado que el timbrado falla,
Cuando SUNAT retorna error,
Entonces el sistema deberá mostrar el detalle, no persistir la NC como vigente y permitir reintentar.

Criterio H4 — Conservación 5 años
Dado que la NC se persistió,
Cuando se archiva,
Entonces deberá conservarse por 5 años (R.S. 117-2017/SUNAT), desde el 1 de enero del año siguiente al de emisión.

═══════════════════════════════════════════════════════════════
SECCIÓN I — PDF: IDENTIDAD VISUAL E INFORMACIÓN
═══════════════════════════════════════════════════════════════

Criterio I1 — Identidad visual de Golocaer S.A.C.
Dado que el sistema genera el PDF,
Cuando lo renderiza,
Entonces deberá usar el logo y la paleta corporativa de Golocaer S.A.C., consistente con la Factura Perú (R16A-RE-FU-022).

Criterio I2 — Datos del emisor en el PDF
Dado que el sistema genera el PDF,
Cuando consigna el emisor,
Entonces deberá mostrar razón social, RUC y domicilio fiscal de Golocaer S.A.C.

Criterio I3 — Datos del receptor en el PDF
Dado que el sistema genera el PDF,
Cuando consigna el receptor,
Entonces deberá mostrar razón social y RUC/documento del cliente.

Criterio I4 — Datos del comprobante en el PDF
Dado que el sistema genera el PDF,
Cuando consigna el comprobante,
Entonces deberá mostrar tipo (07 – Nota de Crédito Electrónica), serie-correlativo y fecha.

Criterio I5 — Motivo y comprobante de referencia en el PDF
Dado que el sistema genera el PDF,
Cuando consigna el motivo,
Entonces deberá mostrar el motivo (código y descripción del catálogo 09) y el comprobante origen referenciado (serie-correlativo).

Criterio I6 — Líneas o concepto en el PDF
Dado que el sistema genera el PDF,
Cuando consigna el detalle,
Entonces deberá mostrar las líneas (modalidad por partidas) o el concepto (modalidad manual) con importes e IGV desglosado.

Criterio I7 — Totales en el PDF
Dado que el sistema genera el PDF,
Cuando consigna los totales,
Entonces deberá mostrar Valor Venta, IGV e Importe Total con la nomenclatura SUNAT, en la moneda del comprobante origen.

Criterio I8 — QR y leyenda obligatoria
Dado que el sistema genera el PDF,
Cuando lo finaliza,
Entonces deberá incluir el código QR de verificación SUNAT y la leyenda obligatoria de representación impresa. NO incluye sello digital SAT ni cadena complementaria (conceptos del SAT).

═══════════════════════════════════════════════════════════════
SECCIÓN J — ENVÍO POR CORREO
═══════════════════════════════════════════════════════════════

Criterio J1 — Envío tras timbrado
Dado que la NC se timbró exitosamente,
Cuando concluye la generación,
Entonces el sistema deberá enviar la NC por correo al cliente con PDF y XML adjuntos.

Criterio J2 — Destinatarios
Dado que el sistema arma el correo,
Cuando asigna destinatarios,
Entonces deberá poner como Para el contacto del cliente y como CC el ESAC asignado.

Criterio J3 — Asunto y cuerpo
Dado que el sistema arma el correo,
Cuando lo compone,
Entonces deberá pre-rellenar el asunto con el folio de la NC (serie-correlativo). ** Formato del asunto y plantilla del cuerpo para Perú pendientes de confirmar. **"								"- Fila para la generación y representación impresa (PDF) de la Nota de Crédito de Región Perú, contraparte de R16A-RE-FU-034 (México). Las funciones del módulo NC están en R16A-RE-FU-033.
- ** Nota transversal: toda la mecánica fiscal SUNAT de esta fila requiere validación con el asesor fiscal peruano de PROQUIFA antes de implementarse. **
- Empresa emisora única: Golocaer S.A.C. (RUC 20612772941). En México son cuatro emisoras; en Perú solo una.
- Cambios fiscales vs México (verificados con investigación normativa SUNAT, pendientes de validación): tipo E→07; CFDI 4.0→UBL 2.1; sin UsoCFDI/TipoRelacion/MetodoPago/FormaPago; referencia UUID→serie-correlativo; motivo SAT→catálogo 09 (cbc:ResponseCode + cbc:Description); IVA→IGV (18%).
- Representación impresa: en vez de sello digital SAT + cadena complementaria + QR del SAT, el PDF peruano lleva el QR de verificación SUNAT y la leyenda obligatoria de representación impresa; el respaldo legal es la firma digital del emisor y el valor resumen/hash.
- Tipo de cambio: hereda el de la factura origen (fecha de emisión del comprobante modificado), no el del día de la NC (igual criterio que R16A-RE-FU-033; diferencia intencional vs México).
- Conservación: 5 años (R.S. 117-2017/SUNAT), desde el 1 de enero del año siguiente al de emisión.
- ** Pendiente — modalidad de emisión electrónica ante SUNAT (ver R16A-RE-FU-029); no se da por hecho un OSE ni se reutiliza el PAC de México. **
- ** Pendiente — formato del asunto y plantilla del cuerpo del correo de envío para Perú. **
- ** Pendiente — origen del texto del sustento (cbc:Description); ver R16A-RE-FU-033. **
- ** Pendiente — maquetas del PDF de NC Perú no disponibles; el diseño se validará contra ellas cuando lleguen (esta fila se basó en el análisis del flujo mexicano y la representación impresa SUNAT). **"									