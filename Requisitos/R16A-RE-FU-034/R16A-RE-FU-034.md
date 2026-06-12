R16A-RE-FU-034		Diseño y generación de Documentos: NDC México	Notas de Crédito	Yo como ** Usuario de Tesorería de PROQUIFA México **, quiero que el sistema genere automáticamente el CFDI de Egreso de la Nota de Crédito y su PDF al confirmar el timbrado en el módulo de Notas de Crédito, para entregar al cliente la representación impresa del comprobante y conservarla como artefacto fiscal inmutable.	El sistema debe generar una Nota de Crédito (CFDI tipo E / Egreso) y su PDF representativo al confirmar el timbrado desde el wizard del módulo Notas de Crédito (R16A-RE-FU-032), conforme a la normativa SAT vigente y con el branding de la empresa emisora del grupo PROQUIFA México. El módulo de Notas de Crédito es el único disparador de la generación, y la NC referencia obligatoriamente la factura origen. El XML timbrado y el PDF se persisten en PQF2, se conservan por el plazo fiscal exigido y se envían al cliente con copia al ESAC y al analista de Cuentas por Cobrar tras el timbrado exitoso. La estructura se reutiliza para Perú con diferencias significativas y se documenta en requisito independiente.			Propuesto					R16.4M-RE-FU-001	"## Aplica a
- Generación del CFDI tipo E (Nota de Crédito / Egreso) y su PDF representativo al confirmar el timbrado desde el Paso 3 del wizard del módulo NC.
- Región México exclusivamente. Empresas emisoras: Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica.
- Estructura XML conforme a CFDI 4.0 (Apéndice 5 Anexo 20 SAT vigente): cabecera con TipoDeComprobante=E, Emisor, Receptor con UsoCFDI=G02, CfdiRelacionados con TipoRelacion=01 + UUID factura origen, Conceptos (heredados de partidas o concepto manual según modalidad), Impuestos trasladados, Totales reales (Subtotal, Descuento si aplica, Total) en moneda de la factura origen.
- MetodoPago=PUE fijo (regla SAT inmutable: NCs siempre PUE).
- FormaPago heredada de la factura origen pagada (típicamente 03 Transferencia).
- Multi-divisa: hereda Moneda de la factura origen (no editable), preserva TipoCambio del día del timbrado.
- Cancelación condicional de la factura origen en el mismo flujo: cuando la NC es por totalidad + mismo mes calendario, el sistema dispara cancelación SAT de la factura origen con motivo seleccionado del catálogo c_MotivoCancelacion.
- Timbrado vía PAC TurboPac con UUID asignado por el SAT.
- PDF representativo con identidad visual de la empresa emisora (logo, paleta corporativa, iconografía de certificaciones del giro químico-farmacéutico), consistente con Factura México y Complemento de Pago México.
- PDF con la información completa de la NC: emisor, receptor, datos del comprobante, motivo y tipo de relación, referencia a factura origen, partidas o concepto manual con importes e impuestos desglosados, totales, sellos y trazabilidad SAT, código QR de verificación.
- Persistencia de XML timbrado y PDF en PQF2 con conservación obligatoria de mínimo 5 años (Art. 30 CFF).
- Envío de la NC por correo al cliente con copia al ESAC y al analista de Cuentas por Cobrar tras timbrado exitoso.
- Manejo de errores PAC consistente con Factura por Adelantado, Validar Cobro y Complemento de Pago (mensaje al usuario; la NC no se persiste como vigente; reintento posterior).

## No aplica a
- Generación de NC para Perú."	"## Reglas de Negocio

Regla 1 — Detonante único módulo NC
La generación del CFDI tipo E se dispara cuando el usuario confirma el timbrado en el Paso 3 del wizard del módulo Notas de Crédito. El módulo NC es el único disparador.

Regla 2 — TipoDeComprobante E fijo
El TipoDeComprobante del comprobante root de la NC es E (Egreso) fijo.

Regla 3 — CfdiRelacionados obligatorio TipoRelacion 01
El XML de la NC incluye obligatoriamente el nodo CfdiRelacionados con TipoRelacion=01 (Nota de crédito de los documentos relacionados) y el UUID de la factura origen.

Regla 4 — UsoCFDI receptor G02 default
El UsoCFDI del receptor de la NC es G02 (Devoluciones, descuentos o bonificaciones) por default. ** Validar si debe ser editable según el receptor; la política PQF2 mantiene G02. **

Regla 5 — MetodoPago PUE fijo
El MetodoPago de la NC es PUE (Pago en Una Exhibición) fijo, no editable, conforme regla SAT inmutable.

Regla 6 — FormaPago heredada de factura origen
El FormaPago de la NC se hereda de la factura origen pagada (típicamente 03 Transferencia electrónica). ** Validar el comportamiento cuando la factura origen no esté pagada. **

Regla 7 — Moneda y TipoCambio heredados
La Moneda de la NC se hereda de la factura origen (no editable) y el TipoCambio se captura del día del timbrado para monedas extranjeras.

Regla 8 — Conceptos en modalidad por partidas
En modalidad por partidas (motivo Devolución de mercancía), el nodo Conceptos del XML contiene un nodo Concepto por cada partida con Cant. NC > 0, heredando ClaveProdServ, ClaveUnidad, NoIdentificacion, ValorUnitario, Descripción y configuración de impuestos del concepto original, y recalculando importes con la Cant. NC.

Regla 9 — Conceptos en modalidad manual
En modalidad manual (motivo Descuento o bonificación), el nodo Conceptos contiene un único Concepto con ClaveProdServ=84111506 (Servicios de facturación) y ClaveUnidad=ACT (Actividad) por default, Cantidad=1, Descripcion = concepto capturado por el usuario, ValorUnitario e Importe = Monto Total NC capturado, y ObjetoImp según corresponda al producto/servicio facturado en la factura origen. ** Confirmar el ObjetoImp aplicable en modalidad manual. **

Regla 10 — Impuestos trasladados calculados
El sistema calcula automáticamente el IVA al 16% o la tasa correspondiente al producto, agregando el nodo Impuestos con los TrasladosTotales sumados.

Regla 11 — Totales reales en moneda de factura origen
El SubTotal y el Total del comprobante root de la NC son los valores reales en la moneda de la factura origen (a diferencia del CFDI tipo P, donde son 0). ** Pendiente validar con asesor fiscal: el patrón canónico del proyecto indica que PROQUIFA opera la NC sin campo Descuento explícito (el ajuste se refleja directamente en SubTotal/Total), pero esta fila lo redactó como ""Descuento si aplica"". Confirmar si el campo Descuento del comprobante root debe poblarse o debe omitirse en las NC de PROQUIFA. **

Regla 12 — Cancelación condicional de factura origen
Cuando la NC se generó con la opción de cancelar la factura origen activa (NC por totalidad + mismo mes calendario), al procesar el timbrado exitoso de la NC el sistema dispara la cancelación SAT de la factura origen vía TurboPac con el motivo del catálogo c_MotivoCancelacion seleccionado en el Paso 2 del wizard.

Regla 13 — Foliado consecutivo por empresa
El folio interno de la NC es consecutivo continuo por empresa emisora, con serie distintiva propuesta. El UUID lo asigna el SAT. ** Esquema del foliador y serie distintiva pendiente de validar. **

Regla 14 — Conservación XML 5 años
El XML timbrado de la NC se conserva por un mínimo de 5 años (Art. 30 CFF).

Regla 15 — Envío tras timbrado exitoso
Tras el timbrado exitoso de la NC, el sistema envía el correo al cliente con el PDF y el XML adjuntos.

Regla 16 — Identidad visual del PDF por empresa emisora
El PDF de la NC aplica el logo y la paleta corporativa de la empresa emisora del grupo PROQUIFA México.

Regla 17 — Manejo de errores PAC
Cuando el timbrado falla, la NC no se persiste como vigente; el usuario puede reintentar posteriormente según la política transversal de manejo de errores del PAC.

## Riesgos

Riesgo 1 — Cálculo erróneo de impuestos al heredar/recalcular partidas
Si el sistema calcula incorrectamente los importes y los impuestos trasladados al recalcular partidas con Cant. NC, el SAT puede rechazar el timbrado o generar inconsistencias entre la NC y la factura origen.

Riesgo 2 — Cancelación de factura origen fallida tras NC timbrada
Si la NC se timbra exitosamente pero la cancelación SAT de la factura origen falla (caso totalidad + mismo mes con opción activa), queda una inconsistencia operativa: la NC existe vigente pero la factura origen también.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — DETONANTE
═══════════════════════════════════════════════════════════════

Criterio A1 — Disparo desde el módulo NC
Dado el armado de una NC en el wizard del módulo Notas de Crédito,
Cuando el usuario confirma el timbrado en el Paso 3,
Entonces el sistema deberá generar el CFDI tipo E correspondiente.

═══════════════════════════════════════════════════════════════
SECCIÓN B — CABECERA DEL XML
═══════════════════════════════════════════════════════════════

Criterio B1 — Campos fijos del comprobante root
Dado la generación de la NC,
Cuando el sistema arma el comprobante,
Entonces los campos fijos son: Version=4.0, TipoDeComprobante=E, Exportacion=01.

Criterio B2 — Serie, Folio, Fecha, LugarExpedicion
Dado la generación de la NC,
Cuando el sistema arma el comprobante,
Entonces deberá incluir Serie distintiva, Folio consecutivo por empresa, Fecha del timbrado y LugarExpedicion = CP de la empresa emisora. ** Serie distintiva pendiente de validar. **

Criterio B3 — Moneda heredada de factura origen
Dado la generación de la NC,
Cuando el sistema asigna Moneda,
Entonces deberá heredar la Moneda de la factura origen (no editable).

Criterio B4 — TipoCambio del día cuando moneda extranjera
Dado la NC con Moneda ≠ MXN,
Cuando el sistema arma el comprobante,
Entonces deberá incluir TipoCambio del día del timbrado.

Criterio B5 — MetodoPago PUE fijo
Dado la generación de la NC,
Cuando el sistema asigna MetodoPago,
Entonces deberá ser PUE fijo.

Criterio B6 — FormaPago heredada
Dado la generación de la NC,
Cuando el sistema asigna FormaPago,
Entonces deberá heredar de la factura origen (típicamente 03 Transferencia).

═══════════════════════════════════════════════════════════════
SECCIÓN C — CFDIRELACIONADOS
═══════════════════════════════════════════════════════════════

Criterio C1 — Nodo CfdiRelacionados obligatorio
Dado la generación de la NC,
Cuando el sistema arma el XML,
Entonces deberá incluir obligatoriamente el nodo CfdiRelacionados con TipoRelacion=01 y un nodo CfdiRelacionado con el UUID de la factura origen.

═══════════════════════════════════════════════════════════════
SECCIÓN D — EMISOR Y RECEPTOR
═══════════════════════════════════════════════════════════════

Criterio D1 — Datos del Emisor
Dado la generación de la NC,
Cuando el sistema arma el nodo Emisor,
Entonces deberá incluir RFC, Nombre/Razón Social y RegimenFiscal=601 de la empresa emisora del grupo.

Criterio D2 — Datos del Receptor
Dado la generación de la NC,
Cuando el sistema arma el nodo Receptor,
Entonces deberá incluir RFC, Nombre/Razón Social, DomicilioFiscalReceptor y RegimenFiscalReceptor del cliente.

Criterio D3 — UsoCFDI G02 default
Dado la generación de la NC,
Cuando el sistema asigna UsoCFDI del receptor,
Entonces deberá ser G02 por default.

═══════════════════════════════════════════════════════════════
SECCIÓN E — CONCEPTOS MODALIDAD POR PARTIDAS
═══════════════════════════════════════════════════════════════

Criterio E1 — Un Concepto por partida con Cant. NC > 0
Dado modalidad por partidas,
Cuando el sistema arma el nodo Conceptos,
Entonces deberá generar un nodo Concepto por cada partida con Cant. NC > 0 (las partidas con Cant. NC = 0 no se incluyen).

Criterio E2 — Herencia de datos fiscales del concepto original
Dado un Concepto en modalidad por partidas,
Cuando el sistema lo arma,
Entonces deberá heredar del concepto original de la factura origen: ClaveProdServ, ClaveUnidad, NoIdentificacion, ValorUnitario, Descripción y configuración de impuestos.

Criterio E3 — Recalculo de importes según Cant. NC
Dado un Concepto en modalidad por partidas,
Cuando el sistema calcula importes,
Entonces Cantidad = Cant. NC capturada, Importe = Cant. NC × ValorUnitario, e impuestos trasladados recalculados sobre el nuevo Importe.

═══════════════════════════════════════════════════════════════
SECCIÓN F — CONCEPTOS MODALIDAD MANUAL
═══════════════════════════════════════════════════════════════

Criterio F1 — Único Concepto en modalidad manual
Dado modalidad manual,
Cuando el sistema arma el nodo Conceptos,
Entonces deberá contener un único Concepto.

Criterio F2 — Campos del Concepto manual
Dado el Concepto en modalidad manual,
Cuando el sistema lo arma,
Entonces los campos serán: ClaveProdServ=84111506, ClaveUnidad=ACT, Cantidad=1, Descripcion = concepto capturado por el usuario, ValorUnitario e Importe = Monto Total NC capturado, ObjetoImp según corresponda.

═══════════════════════════════════════════════════════════════
SECCIÓN G — IMPUESTOS Y TOTALES
═══════════════════════════════════════════════════════════════

Criterio G1 — Cálculo automático de impuestos trasladados
Dado la generación de la NC,
Cuando el sistema calcula impuestos,
Entonces deberá calcular automáticamente IVA al 16% o la tasa correspondiente al producto, sumando los traslados a nivel comprobante.

Criterio G2 — Totales reales en moneda de factura origen
Dado la generación de la NC,
Cuando el sistema arma el comprobante root,
Entonces SubTotal y Total deberán ser los valores reales en la moneda de la factura origen. ** Campo Descuento del comprobante root pendiente de validar con asesor fiscal (ver Regla 11): definir si se puebla o se omite según la operación de PROQUIFA. **

═══════════════════════════════════════════════════════════════
SECCIÓN H — CANCELACIÓN CONDICIONAL DE FACTURA ORIGEN
═══════════════════════════════════════════════════════════════

Criterio H1 — Cancelación SAT de factura origen tras timbrado de NC
Dado que la NC se confirmó con la opción de cancelar la factura origen activa (NC por totalidad + mismo mes calendario),
Cuando el timbrado de la NC es exitoso,
Entonces el sistema deberá disparar la cancelación SAT de la factura origen vía TurboPac con el motivo c_MotivoCancelacion seleccionado.

Criterio H2 — Manejo de error en cancelación de factura origen
Dado que la cancelación SAT de la factura origen falla,
Cuando el sistema procesa la respuesta,
Entonces deberá notificar al usuario del fallo y permitir reintento posterior según la política transversal. La NC ya timbrada permanece vigente.

═══════════════════════════════════════════════════════════════
SECCIÓN I — TIMBRADO Y PERSISTENCIA
═══════════════════════════════════════════════════════════════

Criterio I1 — Timbrado vía PAC TurboPac
Dado la NC armada,
Cuando el sistema la envía a timbrar,
Entonces deberá usar el PAC TurboPac.

Criterio I2 — UUID asignado por SAT
Dado timbrado exitoso,
Cuando el sistema recibe la respuesta,
Entonces deberá registrar el UUID asignado por el SAT en el XML timbrado.

Criterio I3 — Persistencia XML y PDF
Dado timbrado exitoso,
Cuando el sistema confirma,
Entonces deberá persistir el XML timbrado y el PDF representativo en PQF2.

Criterio I4 — Manejo de errores PAC
Dado timbrado fallido,
Cuando el sistema procesa la respuesta,
Entonces la NC no se persiste como vigente; el usuario puede reintentar posteriormente según la política transversal.

Criterio I5 — Conservación 5 años
Dado timbrado exitoso,
Cuando el sistema persiste el documento,
Entonces deberá conservar el XML por un mínimo de 5 años (Art. 30 CFF).

═══════════════════════════════════════════════════════════════
SECCIÓN J — PDF: IDENTIDAD VISUAL E INFORMACIÓN
═══════════════════════════════════════════════════════════════

Criterio J1 — Logo y paleta corporativa de la empresa emisora
Dado la generación del PDF,
Cuando el sistema arma el documento,
Entonces deberá incluir el logo y aplicar la paleta de colores corporativa de la empresa emisora (Golocaer naranja, Mungen verde, Proquifa cyan, Proveedora Quimico Farmaceutica cyan).

Criterio J2 — Iconografía de certificaciones del giro
Dado la generación del PDF,
Cuando el sistema arma el documento,
Entonces deberá incluir la iconografía de certificaciones del giro químico-farmacéutico consistente con factura y Complemento de Pago. ** Confirmar vigencia de las certificaciones. **

Criterio J3 — Consistencia con Factura y Complemento de Pago México
Dado la generación del PDF de la NC,
Cuando el sistema arma el documento,
Entonces el branding, tipografía e identidad visual deberán ser consistentes con Factura México y Complemento de Pago México.

Criterio J4 — Datos del Emisor en el PDF
Dado la generación del PDF,
Cuando el sistema arma el encabezado,
Entonces deberá incluir: Razón Social del emisor, RFC, Lugar de Expedición, Fecha y Hora de Expedición, Régimen Fiscal.

Criterio J5 — Datos del Receptor en el PDF
Dado la generación del PDF,
Cuando el sistema arma la sección Cliente,
Entonces deberá incluir: Razón Social del receptor, RFC, Domicilio Fiscal, Régimen Fiscal del receptor, Uso CFDI (G02).

Criterio J6 — Datos del Comprobante en el PDF
Dado la generación del PDF,
Cuando el sistema arma la cabecera del comprobante,
Entonces deberá incluir: Serie, Folio interno, Versión CFDI (4.0), Folio Fiscal (UUID), Fecha y Hora de Certificación, Fecha y Hora de Emisión, Tipo de Comprobante (E-Egreso), Moneda, Tipo de Cambio cuando aplique, Método de Pago (PUE), Forma de Pago.

Criterio J7 — Sección Motivo y CFDI relacionado en el PDF
Dado la generación del PDF,
Cuando el sistema arma la sección de relación,
Entonces deberá incluir: Motivo de la NC, Tipo de Relación SAT (01), Folio y UUID de la factura origen relacionada.

Criterio J8 — Sección Partidas en modalidad por partidas
Dado modalidad por partidas,
Cuando el sistema arma el PDF,
Entonces deberá incluir una tabla con las partidas con Cant. NC > 0: Clave Producto/Servicio, NoIdentificacion (código interno), Descripción, Cantidad (Cant. NC), Clave Unidad, Valor Unitario, Importe, Impuesto Traslado por línea.

Criterio J9 — Sección Concepto en modalidad manual
Dado modalidad manual,
Cuando el sistema arma el PDF,
Entonces deberá incluir: Clave Producto/Servicio (84111506), Cantidad (1), Clave Unidad (ACT), Descripción (concepto capturado), Valor Unitario, Importe, Impuesto Traslado.

Criterio J10 — Sección Totales del comprobante en el PDF
Dado la generación del PDF,
Cuando el sistema arma la sección Totales,
Entonces deberá incluir: Subtotal, Impuestos Trasladados (IVA), Total, todos en la moneda de la factura origen. Adicionalmente, Total en letra.

Criterio J11 — Sello y trazabilidad SAT en el PDF
Dado la generación del PDF,
Cuando el sistema arma el pie del documento,
Entonces deberá incluir: Número de Serie del Certificado del SAT, Número de Serie del CSD del Emisor, Sello Digital del SAT, Sello Digital del CFDI, Cadena Original del Complemento de Certificación Digital del SAT.

Criterio J12 — Código QR de verificación SAT
Dado la generación del PDF,
Cuando el sistema arma el documento,
Entonces deberá incluir el código QR estándar SAT que encodea la URL de verificación con UUID + RFC emisor + RFC receptor + Total + últimos 8 caracteres del sello.

═══════════════════════════════════════════════════════════════
SECCIÓN K — ENVÍO POR CORREO
═══════════════════════════════════════════════════════════════

Criterio K1 — Envío tras timbrado
Dado timbrado exitoso,
Cuando el sistema confirma,
Entonces deberá enviar el correo al cliente con el PDF y el XML de la NC adjuntos.

Criterio K2 — Destinatarios
Dado el envío,
Cuando el sistema arma los destinatarios,
Entonces Para = contacto del cliente vinculado a la factura origen, CC = ESAC asignado + analista de Cuentas por Cobrar.

Criterio K3 — Asunto y cuerpo plantilla
Dado el envío del correo,
Cuando el sistema lo arma,
Entonces el asunto y cuerpo seguirán la plantilla definida. ** Plantilla pendiente de confirmar (PMO #31 transversal). **"								"- Fila documenta la generación y diseño del CFDI tipo E (Nota de Crédito / Egreso) para México. Análoga a Factura México y Complemento de Pago México: una sola fila cubre estructura XML + diseño del PDF.
- La NC no es un módulo independiente: se dispara desde el Paso 3 del wizard del módulo Notas de Crédito al confirmar el timbrado.
- Estructura del CFDI tipo E validada contra el ejemplo real B-128 entregado por el cliente: TipoDeComprobante=E, CfdiRelacionados TipoRelacion=01 con UUID padre 66099124-f173-a87d-55d9-336a3b6ad3f6, Emisor PROQUIFA RFC PRO970821ML3 Régimen 601, Receptor NATURES TOUCH MEXICO RFC NTM230915DQ8 Régimen 601, Moneda USD, TipoCambio 19.17, SubTotal 48.00, Total 55.68 (sin campo Descuento), FormaPago=03 (Transferencia), MetodoPago=PUE, Concepto ClaveProdServ 41116132 ClaveUnidad H87, IVA 16%, RfcProvCertif QSO100827UB0 (TurboPac).
- Cumplimiento fiscal SAT vigente CFDI 4.0 (Apéndice 5 Anexo 20).
- Cancelación condicional de factura origen en el mismo flujo cuando la NC es totalidad + mismo mes, con motivo del catálogo c_MotivoCancelacion seleccionado en el Paso 2 del wizard. Tras timbrado exitoso de la NC, el sistema dispara la cancelación SAT de la factura origen vía TurboPac. Si la cancelación falla, la NC ya timbrada permanece vigente y se permite reintento.
- Foliado consecutivo continuo por empresa emisora del grupo PROQUIFA México con serie distintiva propuesta (pendiente validar esquema final).
- Diseño del PDF: identidad visual por empresa emisora (logo + paleta corporativa: Golocaer naranja, Mungen verde, Proquifa cyan, Proveedora Quimico Farmaceutica cyan), consistente con Factura México y Complemento de Pago México, con iconografía de certificaciones del giro químico-farmacéutico en el pie. Esta fila documenta qué información debe contener el PDF, no la disposición visual literal (cintas, posicionamiento, dimensiones).
- ** Pendiente (duda fiscal): FormaPago en modalidad manual, comportamiento si la factura origen no estuviera pagada (escenario raro en R16 prepago). Validar con asesor fiscal. **
- ** Pendiente (duda fiscal): ClaveProdServ y ClaveUnidad en modalidad manual, confirmar que 84111506/ACT son apropiados para todos los casos de descuento/bonificación. Validar con asesor fiscal. **
- ** Pendiente (duda fiscal): UsoCFDI receptor G02 fijo. En el ejemplo B-128 aparece G03 (discrepancia legacy vs SAT); la política PQF2 fija G02 por default. Validar si debe ser editable según el receptor. **
- ** Pendiente: validación de la serie del foliador final. **
- ** Pendiente: vigencia de la iconografía de certificaciones del giro químico-farmacéutico (ISO, NEEC, edQM, FELUM, USP, Microbiologics, APACOR, CHATA, Pharmaffiliates, Amex). **
- ** Pendiente: PMO #31, plantilla del cuerpo del correo de envío (transversal Proforma / Factura / NC / Complemento de Pago / Inconsistencia de Pago). **"									