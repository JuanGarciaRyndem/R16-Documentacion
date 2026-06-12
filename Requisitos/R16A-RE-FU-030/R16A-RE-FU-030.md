# R16A-RE-FU-030 — Diseño y generación de Documentos: CDP México

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-030 |
| **Título** | Diseño y generación de Documentos: CDP México |
| **Módulo / Épica** | Validar Cobro |
| **Historia de Usuario** | Yo como **Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver)**, quiero que el sistema genere automáticamente el Complemento de Pago y su PDF al confirmar un cobro contra facturas PPD en Validar Cobro, para cumplir con la obligación fiscal SAT de documentar el pago y entregar el comprobante al cliente sin operar un módulo aparte. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.2M-RE-FU-002 |

---

## Requisito Funcional

El sistema debe generar un Complemento de Pago (CFDI tipo P) y su PDF representativo al confirmar un cobro en Validar Cobro contra una o más facturas con Método de Pago PPD, conforme a la normativa SAT vigente y con el branding de la empresa emisora del grupo PROQUIFA México. No existe pantalla ni módulo independiente para generarlo: el único disparador es la confirmación del cobro en el Paso 3 del wizard de Validar Cobro. El XML timbrado y el PDF se persisten en PQF2, se conservan por el plazo fiscal exigido y se envían automáticamente al cliente con copia al ESAC y al analista de Cuentas por Cobrar. La estructura se reutiliza para Perú si SUNAT exige documento equivalente; se documenta en requisito independiente.

---

## Alcance

### Aplica a

- Generación automática del CFDI tipo P (Complemento de Pago / Recibo Electrónico de Pago) y su PDF representativo al confirmar un cobro en el Paso 3 del wizard de Validar Cobro contra una o más facturas vigentes del cliente con MetodoPago=PPD.
- Región México exclusivamente. Empresas emisoras: Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica.
- Política de un Complemento de Pago por factura: si un cobro cubre N facturas PPD, se generan N Complementos de Pago independientes (cada Complemento de Pago con un único Pago y un único DoctoRelacionado).
- Estructura XML conforme a CFDI 4.0 Pagos20 v2.0 (Apéndice 6 Anexo 20 SAT vigente): cabecera fija, Emisor, Receptor con UsoCFDI=CP01, Concepto único fijo, Complemento Pagos20 con Totales, Pago y un Documento Relacionado con impuestos desglosados cuando aplique.
- Multi-divisa con EquivalenciaDR (cuando MonedaDR ≠ MonedaP) y TipoCambioP (cuando MonedaP ≠ MXN).
- Monto del Pago = ImpPagado del DoctoRelacionado (porción del cobro aplicada a la factura específica del Complemento de Pago).
- Timbrado vía PAC TurboPac con UUID asignado por el SAT.
- Cascada de timbrado según Método de Pago elegido en líneas con proforma origen:
  - PUE: genera únicamente la Factura PUE (1 CFDI).
  - PPD: genera la Factura PPD y, en cascada inmediata, el Complemento de Pago asociado al cobro confirmado (2 CFDIs).
- Líneas con factura origen (factura PPD ya existente): siempre generan únicamente el Complemento de Pago.
- Modal de envío al cliente con destinatario (contacto del pedido, editable), CC al ESAC asignado (editable), asunto generado por sistema según plantilla por tipo de documento (no editable), adjuntos (PDF y XML de cada CFDI timbrado en la línea + Confirmación de Pedido del Pedido Prepago, no editables) y text area de notas extras opcional.
- Para Pedidos Prepago: la Confirmación de Pedido se incluye siempre como adjunto del correo. No existe candado previo que bloquee o difiera su envío.
- PDF representativo con identidad visual de la empresa emisora (logo, paleta corporativa, iconografía de certificaciones del giro químico-farmacéutico).
- PDF con la información completa del Complemento de Pago: emisor, receptor, datos del comprobante, sección Concepto estructural, Totales del Complemento, datos del Pago, datos del Documento Relacionado con saldos y parcialidad, impuestos desglosados cuando aplique, sellos y trazabilidad SAT, código QR de verificación.
- Persistencia de XML timbrado y PDF en PQF2 con conservación obligatoria de mínimo 5 años (Art. 30 CFF).
- Envío automático del Complemento de Pago por correo al cliente con copia al ESAC y al analista de Cuentas por Cobrar tras timbrado exitoso.
- Manejo de errores PAC consistente con Factura por Adelantado y Validar Cobro (mensaje al usuario; Complemento de Pago no se persiste como vigente; reintento posterior).

### No aplica a

- Generación manual o standalone de Complementos de Pago fuera del flujo de Validar Cobro (no existe módulo independiente).
- Facturas con MetodoPago=PUE (regla SAT inmutable: Complemento de Pago solo aplica a PPD).
- Complementos de Pago que agrupen múltiples DoctosRelacionados en un único comprobante (la política operativa de R16 es un Complemento de Pago por factura).
- Cancelación de Complementos de Pago (fuera del scope de R16, consistente con NC y cancelación standalone de facturas).
- Inclusión de NCs aplicadas en el cobro como DoctoRelacionado del Complemento de Pago (las NCs son CFDI de Egreso y se relacionan a la factura origen desde la propia NC).
- Generación del Complemento de Pago para Perú (requisito independiente si SUNAT exige documento equivalente).
- Plantilla del cuerpo del correo de envío del Complemento de Pago (** PMO #31 transversal **).
- Política formal de reintento ante fallo de timbrado PAC (** transversal con Factura por Adelantado, Notas de Crédito y Validar Cobro **).
- Convención de hora en FechaPago (12:00:00 fija vs hora real del cobro) (** pendiente validar con asesor fiscal **).
- Validación del esquema definitivo del foliador con serie "P" (** pendiente **).

---

## Reglas de Negocio

**Regla 1 — Detonante único Validar Cobro**
La generación del Complemento de Pago se dispara automáticamente cuando el usuario confirma, en el Paso 3 del wizard de Validar Cobro, un cobro que cubre facturas PPD. Validar Cobro es el único disparador; no existe módulo ni pantalla independiente para generar Complementos de Pago.

**Regla 2 — Un Complemento de Pago por factura cubierta**
Cuando un cobro cubre N facturas PPD, el sistema emite N Complementos de Pago, uno por cada factura cubierta. Cada Complemento de Pago contiene exactamente un Pago con un único DoctoRelacionado referenciando a la factura específica.

**Regla 3 — Solo facturas PPD generan Complemento de Pago**
El sistema emite Complemento de Pago únicamente para las facturas asociadas al cobro con MetodoPago=PPD. Las facturas PUE no requieren Complemento de Pago.

**Regla 4 — Campos fijos de cabecera SAT**
Los campos fijos del comprobante root del Complemento de Pago son: Version=4.0, TipoDeComprobante=P, Exportacion=01, SubTotal=0, Total=0, Moneda=XXX.

**Regla 5 — Concepto único fijo**
El nodo Conceptos del Complemento de Pago contiene un único Concepto con: ClaveProdServ=84111506, Cantidad=1, ClaveUnidad=ACT, Descripcion=Pago, ValorUnitario=0, Importe=0, ObjetoImp=01.

**Regla 6 — UsoCFDI receptor CP01 fijo**
El UsoCFDI del receptor en el Complemento de Pago es CP01 fijo, según norma SAT.

**Regla 7 — FormaDePagoP real, no 99**
El FormaDePagoP del nodo Pago es la forma real en que entró el dinero (típicamente 03 Transferencia). No se usa 99 Por definir.

**Regla 8 — Monto del Pago igual a ImpPagado del DR**
El Monto del nodo Pago es igual al ImpPagado del DoctoRelacionado de ese Complemento de Pago (porción del cobro aplicada a la factura específica del Complemento de Pago).

**Regla 9 — TipoCambioP cuando moneda extranjera**
Cuando el nodo Pago tiene MonedaP ≠ MXN, el sistema incluye TipoCambioP con el TC del día del cobro respecto a MXN.

**Regla 10 — Multi-divisa con EquivalenciaDR**
Cuando un DoctoRelacionado tiene MonedaDR ≠ MonedaP, el sistema incluye EquivalenciaDR como factor numérico de conversión. Si las monedas coinciden, EquivalenciaDR = 1.

**Regla 11 — Impuestos trasladados solo si ObjetoImpDR=02**
Cuando un DoctoRelacionado tiene ObjetoImpDR=02, el sistema incluye el sub-nodo ImpuestosDR/TrasladosDR. Si ObjetoImpDR=01, el sub-nodo se omite.

**Regla 12 — Foliado consecutivo por empresa**
El folio interno del Complemento de Pago es consecutivo continuo por empresa emisora, con serie "P" propuesta. El UUID lo asigna el SAT. ** Esquema del foliador con serie "P" pendiente de validar. **

**Regla 13 — Conservación XML 5 años**
El XML timbrado del Complemento de Pago se conserva por un mínimo de 5 años (Art. 30 CFF).

**Regla 14 — Envío tras timbrado exitoso**
Tras el timbrado exitoso del Complemento de Pago, el sistema envía el correo al cliente con el PDF y el XML adjuntos.

**Regla 15 — NCs no son DoctoRelacionado del Complemento de Pago**
Cuando un cobro aplicó NCs, el sistema no incluye las NCs como DoctoRelacionado del Complemento de Pago. Las NCs son CFDI de Egreso y se relacionan a la factura origen desde la propia NC.

**Regla 16 — Identidad visual del PDF por empresa emisora**
El PDF del Complemento de Pago aplica el logo y la paleta corporativa de la empresa emisora del grupo PROQUIFA México.

---

## Riesgos

**Riesgo 1 — Cálculo erróneo de EquivalenciaDR y TipoCambioP en multi-divisa**
Si el sistema calcula incorrectamente los factores de conversión, el SAT puede rechazar el timbrado o el cliente puede tener problemas para acreditar el IVA.

**Riesgo 2 — Convención de hora 12:00:00 en FechaPago**
Los ejemplos reales analizados tienen hora 12:00:00 fija en FechaPago, lo que sugiere una convención heredada del sistema legacy. Si el SAT requiere la hora real del cobro, podría haber una observación fiscal. ** Pendiente validar con asesor fiscal la convención de hora en FechaPago. **

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — DETONANTE
═══════════════════════════════════════════════════════════════

**Criterio A1 — Disparo automático desde Validar Cobro**
Dado un cobro que cubre al menos una factura PPD,
Cuando el usuario confirma el cobro en el Paso 3 del wizard de Validar Cobro,
Entonces el sistema deberá generar automáticamente los Complementos de Pago correspondientes.

**Criterio A2 — No genera Complemento de Pago si todas las facturas son PUE**
Dado un cobro que cubre solo facturas PUE,
Cuando el usuario confirma el cobro,
Entonces el sistema no deberá generar Complemento de Pago.

**Criterio A3 — Un Complemento de Pago por factura cubierta**
Dado un cobro confirmado que cubre N facturas PPD,
Cuando el sistema procesa la generación,
Entonces deberá emitir N Complementos de Pago, uno por cada factura, cada uno con un único DoctoRelacionado referenciando a la factura específica.

═══════════════════════════════════════════════════════════════
SECCIÓN B — CABECERA Y CONCEPTO DEL XML
═══════════════════════════════════════════════════════════════

**Criterio B1 — Campos fijos del comprobante**
Dado la generación del Complemento de Pago,
Cuando el sistema arma el comprobante,
Entonces los campos serán: Version=4.0, TipoDeComprobante=P, Exportacion=01, SubTotal=0, Total=0, Moneda=XXX.

**Criterio B2 — Serie, Folio, Fecha, LugarExpedicion**
Dado la generación del Complemento de Pago,
Cuando el sistema arma el comprobante,
Entonces deberá incluir Serie="P", Folio consecutivo por empresa, Fecha del timbrado y LugarExpedicion = CP de la empresa emisora.

**Criterio B3 — Concepto único fijo**
Dado la generación del Complemento de Pago,
Cuando el sistema arma el nodo Conceptos,
Entonces deberá contener un único Concepto con: ClaveProdServ=84111506, Cantidad=1, ClaveUnidad=ACT, Descripcion=Pago, ValorUnitario=0, Importe=0, ObjetoImp=01.

═══════════════════════════════════════════════════════════════
SECCIÓN C — EMISOR Y RECEPTOR
═══════════════════════════════════════════════════════════════

**Criterio C1 — Datos del Emisor**
Dado la generación del Complemento de Pago,
Cuando el sistema arma el nodo Emisor,
Entonces deberá incluir RFC, Nombre/Razón Social y RegimenFiscal=601 de la empresa emisora del grupo.

**Criterio C2 — Datos del Receptor**
Dado la generación del Complemento de Pago,
Cuando el sistema arma el nodo Receptor,
Entonces deberá incluir RFC, Nombre/Razón Social, DomicilioFiscalReceptor y RegimenFiscalReceptor del cliente.

**Criterio C3 — UsoCFDI CP01 fijo**
Dado la generación del Complemento de Pago,
Cuando el sistema asigna UsoCFDI del receptor,
Entonces deberá ser CP01 (Pagos) fijo.

═══════════════════════════════════════════════════════════════
SECCIÓN D — NODO PAGO
═══════════════════════════════════════════════════════════════

**Criterio D1 — FechaPago del cobro**
Dado el nodo Pago,
Cuando el sistema lo arma,
Entonces deberá incluir FechaPago en formato ISO 8601 datetime. ** Pendiente validar si la hora es 12:00:00 fija o la hora real del cobro. **

**Criterio D2 — FormaDePagoP real**
Dado el nodo Pago,
Cuando el sistema asigna FormaDePagoP,
Entonces deberá usar la forma real (catálogo c_FormaPago, típicamente 03). No se usa 99.

**Criterio D3 — MonedaP y Monto por Complemento de Pago**
Dado el nodo Pago de cada Complemento de Pago,
Cuando el sistema lo arma,
Entonces deberá incluir MonedaP (moneda del cobro recibido) y Monto = ImpPagado del DoctoRelacionado de ese Complemento de Pago (porción del cobro aplicada a la factura específica del Complemento de Pago).

**Criterio D4 — TipoCambioP cuando moneda extranjera**
Dado el nodo Pago con MonedaP ≠ MXN,
Cuando el sistema lo arma,
Entonces deberá incluir TipoCambioP con el TC del día respecto a MXN.

═══════════════════════════════════════════════════════════════
SECCIÓN E — NODO DOCTORELACIONADO
═══════════════════════════════════════════════════════════════

**Criterio E1 — Un único DoctoRelacionado por Complemento de Pago**
Dado el armado del Complemento de Pago,
Cuando el sistema construye el nodo Pago,
Entonces deberá generar un único nodo DoctoRelacionado referenciando a la factura cubierta por ese Complemento de Pago.

**Criterio E2 — Campos del DoctoRelacionado**
Dado un DoctoRelacionado,
Cuando el sistema lo arma,
Entonces deberá incluir: IdDocumento (UUID), Serie, Folio, MonedaDR, EquivalenciaDR, NumParcialidad, ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto, ObjetoImpDR.

**Criterio E3 — Cálculo del ImpSaldoInsoluto**
Dado un DoctoRelacionado,
Cuando el sistema calcula ImpSaldoInsoluto,
Entonces deberá ser ImpSaldoAnt - ImpPagado.

**Criterio E4 — NumParcialidad consecutivo por factura**
Dado un DoctoRelacionado,
Cuando el sistema asigna NumParcialidad,
Entonces deberá ser el consecutivo de pagos aplicados a esa factura (1, 2, 3...).

**Criterio E5 — EquivalenciaDR cuando monedas difieren**
Dado un DoctoRelacionado con MonedaDR ≠ MonedaP,
Cuando el sistema asigna EquivalenciaDR,
Entonces deberá ser el factor numérico de conversión. Si las monedas coinciden, EquivalenciaDR = 1.

═══════════════════════════════════════════════════════════════
SECCIÓN F — IMPUESTOS DEL DOCTORELACIONADO
═══════════════════════════════════════════════════════════════

**Criterio F1 — Sub-nodo solo si ObjetoImpDR=02**
Dado un DoctoRelacionado con ObjetoImpDR=02,
Cuando el sistema arma el XML,
Entonces deberá incluir el sub-nodo ImpuestosDR/TrasladosDR.

**Criterio F2 — Sin sub-nodo si ObjetoImpDR=01**
Dado un DoctoRelacionado con ObjetoImpDR=01,
Cuando el sistema arma el XML,
Entonces no deberá incluir el sub-nodo ImpuestosDR.

**Criterio F3 — Campos del TrasladoDR**
Dado un TrasladoDR,
Cuando el sistema lo arma,
Entonces deberá incluir: BaseDR, ImpuestoDR=002, TipoFactorDR=Tasa, TasaOCuotaDR (0.160000), ImporteDR. ** Confirmar soporte para tasas distintas a 16%. **

═══════════════════════════════════════════════════════════════
SECCIÓN G — NODO TOTALES
═══════════════════════════════════════════════════════════════

**Criterio G1 — MontoTotalPagos igual al Monto del único Pago en MXN**
Dado el nodo Totales,
Cuando el sistema calcula MontoTotalPagos,
Entonces deberá ser igual al Monto del único Pago del Complemento de Pago convertido a MXN.

**Criterio G2 — TotalTrasladosBaseIVA16 cuando aplica**
Dado el nodo Totales,
Cuando el DoctoRelacionado tiene ObjetoImpDR=02 y tasa 16%,
Entonces deberá incluir TotalTrasladosBaseIVA16 y TotalTrasladosImpuestoIVA16 en MXN.

**Criterio G3 — Omitir Totales de IVA si DR no es objeto**
Dado el nodo Totales,
Cuando el DoctoRelacionado tiene ObjetoImpDR=01,
Entonces los campos de IVA del nodo Totales se omitirán.

═══════════════════════════════════════════════════════════════
SECCIÓN H — TIMBRADO Y PERSISTENCIA
═══════════════════════════════════════════════════════════════

**Criterio H1 — Timbrado vía PAC TurboPac**
Dado el Complemento de Pago armado,
Cuando el sistema lo envía a timbrar,
Entonces deberá usar el PAC TurboPac.

**Criterio H2 — UUID asignado por SAT**
Dado el timbrado exitoso,
Cuando el sistema recibe la respuesta,
Entonces deberá registrar el UUID asignado por el SAT en el XML timbrado.

**Criterio H3 — Persistencia XML y PDF**
Dado el timbrado exitoso,
Cuando el sistema confirma,
Entonces deberá persistir el XML timbrado y el PDF representativo en PQF2.

**Criterio H4 — Manejo de errores PAC**
Dado un timbrado fallido,
Cuando el sistema procesa la respuesta,
Entonces el Complemento de Pago no se persiste como vigente; el usuario puede reintentar el timbrado posteriormente.

**Criterio H5 — Conservación 5 años**
Dado el timbrado exitoso,
Cuando el sistema persiste el documento,
Entonces deberá conservar el XML por un mínimo de 5 años (Art. 30 CFF).

═══════════════════════════════════════════════════════════════
SECCIÓN I — PDF: IDENTIDAD VISUAL E INFORMACIÓN
═══════════════════════════════════════════════════════════════

**Criterio I1 — Logo y paleta corporativa de la empresa emisora**
Dado la generación del PDF,
Cuando el sistema arma el documento,
Entonces deberá incluir el logo y aplicar la paleta de colores corporativa de la empresa emisora (Golocaer naranja, Mungen verde, Proquifa cyan, Proveedora Quimico Farmaceutica cyan).

**Criterio I2 — Iconografía de certificaciones del giro**
Dado la generación del PDF,
Cuando el sistema arma el documento,
Entonces deberá incluir la iconografía de certificaciones del giro químico-farmacéutico consistente con la factura. ** Confirmar vigencia de las certificaciones. **

**Criterio I3 — Consistencia con Factura y NC México**
Dado la generación del PDF del Complemento de Pago,
Cuando el sistema arma el documento,
Entonces el branding, tipografía e identidad visual deberán ser consistentes con Factura México y NC México.

**Criterio I4 — Datos del Emisor en el PDF**
Dado la generación del PDF,
Cuando el sistema arma el encabezado,
Entonces deberá incluir: Razón Social del emisor, RFC, Lugar de Expedición, Fecha y Hora de Expedición.

**Criterio I5 — Datos del Receptor en el PDF**
Dado la generación del PDF,
Cuando el sistema arma la sección Cliente,
Entonces deberá incluir: Razón Social del receptor, RFC, Domicilio Fiscal, Uso CFDI (CP01).

**Criterio I6 — Datos del Comprobante en el PDF**
Dado la generación del PDF,
Cuando el sistema arma la sección Pago,
Entonces deberá incluir: Serie, Folio interno, Versión CFDI (4.0), Folio Fiscal (UUID), Fecha y Hora de Certificación, Fecha y Hora de Emisión, Tipo de Comprobante (P-Pago), Régimen Fiscal del emisor.

**Criterio I7 — Sección Concepto en el PDF**
Dado la generación del PDF,
Cuando el sistema arma la sección Concepto,
Entonces deberá incluir: ClaveProdServ 84111506, Cantidad 1, ClaveUnidad ACT, Descripción "Pago", Valor Unitario 0.00, Importe 0.00, Subtotal 0.00, Total 0.00, Moneda XXX, Total en letra "CERO XXX 00/100".

**Criterio I8 — Sección Totales del Complemento en el PDF**
Dado la generación del PDF,
Cuando el sistema arma la sección Totales,
Entonces deberá incluir Monto Total de Pagos, y cuando aplique TotalTrasladosBaseIVA16 y TotalTrasladosImpuestoIVA16.

**Criterio I9 — Sección Pago en el PDF**
Dado la generación del PDF,
Cuando el sistema arma la sección Pago,
Entonces deberá incluir: Versión (2.0), Fecha de Pago, Forma de Pago P, Moneda P, Monto, Tipo de Cambio P cuando aplique.

**Criterio I10 — Sección Documento Relacionado en el PDF**
Dado la generación del PDF,
Cuando el sistema arma la sección del Documento Relacionado,
Entonces deberá incluir: ID del Documento (UUID), Serie, Folio, Moneda DR, Equivalencia DR, Número de Parcialidad, Importe Saldo Anterior, Importe Pagado, Importe Saldo Insoluto, Método de Pago DR (PPD), Tipo de Cambio DR cuando aplique.

**Criterio I11 — Sub-sección Impuestos DR cuando aplica**
Dado un DoctoRelacionado con ObjetoImpDR=02,
Cuando el sistema arma el PDF,
Entonces deberá incluir la tabla de Traslados DR con Base DR, Impuesto DR (002), Tipo Factor DR (Tasa), Tasa o Cuota DR (0.160000), Importe DR.

**Criterio I12 — Resumen de impuestos a nivel pago cuando aplica**
Dado un Complemento de Pago con impuestos desglosados,
Cuando el sistema arma el PDF,
Entonces deberá incluir el resumen de Traslados a nivel pago con Base P, Impuesto P, Tipo Factor P, Tasa o Cuota P, Importe P.

**Criterio I13 — Sello y trazabilidad SAT en el PDF**
Dado la generación del PDF,
Cuando el sistema arma el pie del documento,
Entonces deberá incluir: Número de Serie del Certificado del SAT, Número de Serie del CSD del Emisor, Sello Digital del SAT, Sello Digital del CFDI, Cadena Original del Complemento de Certificación Digital del SAT.

**Criterio I14 — Código QR de verificación SAT**
Dado la generación del PDF,
Cuando el sistema arma el documento,
Entonces deberá incluir el código QR estándar SAT que encodea la URL de verificación con UUID + RFC emisor + RFC receptor + Total + últimos 8 caracteres del sello.

═══════════════════════════════════════════════════════════════
SECCIÓN J — ENVÍO POR CORREO
═══════════════════════════════════════════════════════════════

**Criterio J1 — Envío tras timbrado**
Dado el timbrado exitoso,
Cuando el sistema confirma,
Entonces deberá enviar el correo al cliente con el PDF y el XML del Complemento de Pago adjuntos.

**Criterio J2 — Destinatarios**
Dado el envío,
Cuando el sistema arma los destinatarios,
Entonces Para = contacto del cliente vinculado a la factura, CC = ESAC asignado + analista de Cuentas por Cobrar.

**Criterio J3 — Asunto y cuerpo plantilla**
Dado el envío,
Cuando el sistema arma el correo,
Entonces el asunto y cuerpo seguirán la plantilla definida. ** Plantilla pendiente de confirmar (PMO #31 transversal). **

---

## Notas Adicionales

- Fila documenta la generación y diseño del Complemento de Pago (CFDI tipo P / Recibo Electrónico de Pago) para México. Análoga a Factura México: una sola fila cubre estructura XML + diseño del PDF.
- El Complemento de Pago no es un módulo independiente: se dispara desde el Paso 3 del wizard de Validar Cobro al confirmar el cobro contra facturas PPD.
- Política operativa de R16: un Complemento de Pago por factura. Si un cobro cubre N facturas PPD se generan N Complementos de Pago independientes, cada uno con un único Pago y un único DoctoRelacionado.
- Monto del Pago = ImpPagado del DoctoRelacionado: cada Complemento de Pago refleja la porción del cobro aplicada a la factura específica.
- Solo facturas PPD requieren Complemento de Pago (regla SAT inmutable). En R16 prepago aplica a un subconjunto reducido, pero la infraestructura queda lista para releases futuros (R17 crédito).
- Cumplimiento fiscal SAT vigente CFDI 4.0 Pagos20 v2.0 (Apéndice 6 Anexo 20).
- NCs aplicadas en el cobro no se incluyen como DoctoRelacionado del Complemento de Pago (las NCs son CFDI de Egreso y se relacionan a la factura origen desde la propia NC).
- Convención del sistema legacy en los ejemplos analizados: FechaPago con hora fija 12:00:00. Pendiente validar si se mantiene o se captura hora real.
- Cancelación de Complemento de Pago no incluida en este release (consistente con NC y cancelación standalone de facturas).
- Foliado consecutivo continuo por empresa emisora del grupo PROQUIFA México con serie "P" propuesta (pendiente validar esquema final).
- Diseño del PDF: identidad visual por empresa emisora (logo + paleta corporativa), con iconografía de certificaciones del giro químico-farmacéutico en el pie. Esta fila documenta qué información debe contener el PDF, no la disposición visual literal (cintas, posicionamiento, dimensiones).

### Pendientes

- ** Pendiente: convención de FechaPago (hora fija 12:00:00 vs hora real del cobro). **
- ** Pendiente: soporte para tasas de IVA distintas a 16% (frontera 8%, 0%) según escenarios del cliente. **
- ** Pendiente: vigencia de la iconografía de certificaciones del giro químico-farmacéutico (ISO, NEEC, edQM, FELUM, USP, Microbiologics, APACOR, CHATA, Pharmaffiliates, Amex). **
- ** Pendiente: mecanismo de reintento de timbrado en caso de error PAC (transversal con Factura por Adelantado, Notas de Crédito y Validar Cobro). **
- ** Pendiente: plantilla del cuerpo del correo de envío (PMO #31, transversal Proforma/Factura/NC/Complemento/Inconsistencia). **
- ** Pendiente: validación de la serie "P" en el foliador final. **
- ** Para Perú: SUNAT no tiene equivalente directo del Complemento de Pago. Validar si aplica una fila Perú o no requiere este documento. **
