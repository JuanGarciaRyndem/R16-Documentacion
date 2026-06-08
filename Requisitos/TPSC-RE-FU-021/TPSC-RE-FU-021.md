# TPSC-RE-FU-021 — Diseño y generación de Documentos: Factura México

| Campo | Valor |
|-------|-------|
| **ID** | TPSC-RE-FU-021 |
| **Título** | Diseño y generación de Documentos: Factura México |
| **Módulo / Épica** | Factura por Adelantado |
| **Historia de Usuario** | Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver) **, quiero que el sistema genere automáticamente el PDF de la Factura CFDI 4.0 con el branding de la empresa emisora al timbrarse exitosamente, para entregar al cliente la representación impresa del comprobante fiscal y conservarla como artefacto inmutable. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.1M-RE-FU-007 |

---

## Requisito Funcional

El sistema debe generar el PDF (representación impresa) de la Factura CFDI 4.0 al timbrarse exitosamente ante el PAC, para clientes con Región México, con un diseño estandarizado cuyo branding varía según la empresa emisora del pedido (Golocaer, Mungen, Proquifa o Proveedora Quimico Farmaceutica). El PDF se persiste en base de datos al timbrarse como artefacto fiscal inmutable. Esta funcionalidad aplica tanto a las Facturas generadas desde el módulo Factura por Adelantado como a las generadas desde Validar Cobro.

---

## Alcance

### Aplica a

- Generación del PDF de la Factura CFDI 4.0 al timbrarse exitosamente ante el PAC, para clientes con Región México.
- Facturas originadas en el módulo Factura por Adelantado (pedidos Crédito sin controlados y Prepago sin controlados).
- Facturas originadas en Validar Cobro (cobro recibido aplicado a Proforma de Prepago sin controlados).
- Cuatro empresas emisoras del grupo PROQUIFA México con branding propio: Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V. y Proveedora Quimico Farmaceutica S.A. de C.V.
- Branding diferenciado por empresa emisora (logo, colores y certificaciones).
- Foliador independiente por empresa emisora (consecutivo numérico, varchar 6).
- Paginación automática cuando las partidas exceden el espacio de una página (comportamiento ya existente del sistema).
- Persistencia del PDF (junto con el XML timbrado) en base de datos al timbrarse exitosamente.

### No aplica a

- Factura Anticipo (pedidos Prepago con productos controlados): se documenta en requisito independiente.
- Pedidos para clientes con Región Perú: la Factura Perú se documenta en requisito independiente.
- Generación del XML del CFDI (estructura técnica): es responsabilidad del PAC y se rige por el estándar CFDI 4.0 del SAT.
- Lógica de timbrado, integración con el PAC, manejo de errores: vive en los requisitos de los módulos que generan facturas.
- Cancelación de la Factura timbrada: se documenta en el módulo Notas de Crédito.
- Envío de la Factura al cliente por correo electrónico: vive en los requisitos de los módulos que generan facturas.
- Generación del Complemento de Pago: se documenta en requisito independiente del módulo Validar Cobro.

---

## Reglas de Negocio

Regla 1 — Generación al timbrado exitoso
Cuando el PAC retorna confirmación de timbrado exitoso, el sistema genera el PDF de la Factura y lo persiste en base de datos junto con el XML timbrado como artefacto fiscal inmutable.

Regla 2 — Diferenciación por empresa emisora
La Factura se diferencia según la empresa emisora del pedido (una de las cuatro del grupo PROQUIFA México) en logo, colores, certificaciones, lugar de expedición, RFC, dirección y razón social legal correspondiente.

Regla 3 — Folio independiente por empresa emisora
El folio de la Factura usa un contador consecutivo numérico independiente por empresa emisora. El folio es de tipo varchar(6) y se acompaña de la Serie (típicamente "A" en operación actual).

Regla 4 — Versión CFDI 4.0
El PDF refleja la versión 4.0 del estándar CFDI conforme normativa SAT vigente. El disclaimer del documento indica: "Representación impresa de un CFDI 4.0".

Regla 5 — Inmutabilidad de la Factura timbrada
Una vez timbrada exitosamente, el PDF de la Factura se entrega siempre desde el almacenado en base de datos sin posibilidad de modificación. La Factura timbrada es artefacto fiscal inmutable; cualquier corrección posterior requiere cancelación SAT o emisión de Nota de Crédito.

Regla 6 — Paginación automática (comportamiento existente)
Cuando las partidas del pedido exceden el espacio disponible en una sola página, el sistema genera páginas adicionales con la misma cabecera y pie completo, mostrando la numeración "X de Y" en cada página. Este comportamiento ya existe en PQF2.

Regla 7 — Origen de los datos por sección
El PDF se arma desde las fuentes indicadas: datos del Receptor (Razón Social, RFC, Uso CFDI, Código Postal, Régimen Fiscal) del CFDI timbrado (reflejo del Catálogo de Clientes al momento del timbrado); datos del Emisor (Razón Social, RFC, Lugar de Expedición, Régimen Fiscal, Dirección) del CFDI timbrado; datos del CFDI (Serie, Folio, Versión, Folio Fiscal UUID, Fecha y Hora de Emisión, Fecha y Hora de Certificación, Método de Pago, Forma de Pago, Condiciones de Pago, Tipo de Comprobante, Moneda, Tipo de Cambio) del CFDI timbrado; elementos técnicos SAT (Folio Fiscal UUID, Números de Serie de Certificados, Sellos Digitales, Cadena Original, Código QR) del TimbreFiscalDigital del XML generado por el PAC; branding (logo, colores, certificaciones) generado por el sistema según la empresa emisora. ** Pendiente clarificar el origen de los datos de partidas (descripción, Lote, Pedimento, Clave SAT del Producto/Servicio, Nº ID interno, Cantidad, Unidad de Medida, Clave Unidad SAT, Valor Unitario, Importe, desglose de impuestos): no se ha confirmado si provienen del Catálogo de Productos, del Pedido, de catálogos SAT o de configuración adicional. ** ** Pendiente confirmar si la construcción de la referencia bancaria del cliente sigue el mismo método que en la Proforma (lógica del Código Validador) o si tiene reglas propias para la Factura. **

---

## Riesgos

Riesgo 1 — Indisponibilidad del PAC al timbrar
Si el PAC está caído o responde con timeout, no se puede timbrar la Factura ni generar su PDF.

Riesgo 2 — Origen indefinido de los datos de partidas
Si no se clarifica el origen de cada dato de las partidas (Lote, Pedimento, Clave SAT, ID interno, Clave Unidad SAT, descripción, etc.) antes del desarrollo, hay riesgo de que algunos datos requeridos por el CFDI 4.0 no estén disponibles al momento de timbrar, generando errores SAT.

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — DATOS DEL EMISOR
═══════════════════════════════════════════════════════════════

Criterio A1 — Logo de la empresa emisora
Dado que el sistema renderiza el documento,
Cuando incluye el logo,
Entonces deberá mostrar el logo correspondiente a la empresa emisora del CFDI.

Criterio A2 — Datos institucionales del emisor
Dado que el sistema renderiza los datos del emisor,
Cuando incluye la información,
Entonces deberá mostrar:
- Nombre comercial del emisor.
- RFC del emisor.
- Lugar de Expedición (Código Postal del emisor).
- Dirección completa del emisor.
- Fecha y hora de expedición.

═══════════════════════════════════════════════════════════════
SECCIÓN B — DATOS DEL RECEPTOR (CLIENTE)
═══════════════════════════════════════════════════════════════

Criterio B1 — Identificación y datos fiscales del receptor
Dado que el sistema renderiza los datos del receptor,
Cuando incluye la información,
Entonces deberá mostrar:
- Razón Social del receptor.
- RFC del receptor.
- Uso de CFDI (clave SAT seleccionada al generar la Factura).
- Código Postal del receptor (Domicilio Fiscal del Receptor, obligatorio CFDI 4.0).
- Régimen Fiscal del receptor.

═══════════════════════════════════════════════════════════════
SECCIÓN C — DATOS DEL CFDI
═══════════════════════════════════════════════════════════════

Criterio C1 — Identificadores del CFDI
Dado que el sistema renderiza los datos del CFDI,
Cuando incluye los identificadores,
Entonces deberá mostrar:
- Serie (típicamente "A" en operación actual).
- Folio (consecutivo numérico por empresa emisora, varchar 6).
- Versión (4.0).
- Folio Fiscal (UUID de 36 caracteres asignado por el SAT al timbrar).

Criterio C2 — Fechas y horas del CFDI
Dado que el sistema renderiza el documento,
Cuando incluye las marcas temporales,
Entonces deberá mostrar:
- Fecha y Hora de Emisión (timestamp del momento de emisión de la factura por el sistema).
- Fecha y Hora de Certificación (timestamp del momento del timbrado por el PAC).

Criterio C3 — Datos fiscales del CFDI
Dado que el sistema renderiza los datos fiscales del CFDI,
Cuando incluye la información,
Entonces deberá mostrar:
- Método de Pago (PPD - Pago en parcialidades o diferido, valor forzado por la regla SAT).
- Forma de Pago (99 - Por definir, valor forzado por la regla SAT).
- Condiciones de Pago (texto descriptivo: PREPAGO 100%, 30 DIAS, 60 DIAS, 90 DIAS, etc.).
- Tipo de Comprobante (I - Ingreso).
- Régimen Fiscal del emisor (601 - General de Ley Personas Morales en operación actual).
- Moneda (de facturación, con código y nombre).
- Tipo de Cambio (del día de la generación).

Criterio C4 — Atributo de Exportación
Dado que el sistema renderiza los datos del CFDI,
Cuando incluye el atributo de Exportación (campo obligatorio del CFDI 4.0),
Entonces deberá mostrar el valor correspondiente del catálogo c_Exportacion del SAT (ejemplo: "01 - No aplica" para operaciones que no son de exportación, como se observa en la factura real de Mungen con "Exportación: No Aplica").

═══════════════════════════════════════════════════════════════
SECCIÓN D — REFERENCIAS BANCARIAS
═══════════════════════════════════════════════════════════════

Criterio D1 — Cuentas bancarias del grupo PROQUIFA México
Dado que el sistema renderiza las cuentas bancarias,
Cuando arma el contenido,
Entonces deberá mostrar las dos cuentas bancarias del grupo PROQUIFA México (una en MXN y una en USD), con los datos: Banco, Número de Cuenta, Moneda, Referencia del Cliente, CLABE, Sucursal.

Criterio D2 — Referencia bancaria del cliente
Dado que el sistema renderiza el campo Referencia de cada cuenta,
Cuando construye el valor,
Entonces ** pendiente confirmar si la construcción de la referencia sigue el mismo método que en la Proforma o si tiene reglas propias para la Factura. **

Criterio D3 — Referencia del pedido del cliente
Dado que el sistema renderiza la sección,
Cuando incluye el dato del pedido,
Entonces deberá mostrar el número de orden de compra del cliente o referencia equivalente proveniente del Pedido. ** Confirmar diferencia con Folio de Pedido Interno **

═══════════════════════════════════════════════════════════════
SECCIÓN E — PARTIDAS
═══════════════════════════════════════════════════════════════

Criterio E1 — Datos de cada partida
Dado que el sistema renderiza la tabla de partidas,
Cuando incluye los datos por cada partida,
Entonces deberá mostrar:
- Número consecutivo.
- Descripción del producto (incluye nombre del producto, marca, lote, catálogo y caducidad cuando aplique).
- Pedimento (cuando aplique a productos de importación).
- Clave SAT del Producto/Servicio (catálogo c_ClaveProdServ).
- Nº ID interno del producto.
- Cantidad.
- Unidad de Medida (descripción en texto).
- Clave Unidad SAT (catálogo c_ClaveUnidad).
- Valor Unitario con moneda.
- Importe con moneda (cantidad × valor unitario).
- Desglose de impuestos federales por partida: BASE, IMPUESTO (clave SAT), TIPO DE FACTOR, TASA/CUOTA, IMPORTE.

Criterio E2 — Origen de los datos de partidas
Dado que el sistema arma los datos de cada partida,
Cuando los obtiene,
Entonces ** Pendiente clarificar el origen operativo de TODOS los datos de las partidas: descripción del producto, Lote, Pedimento, Clave SAT del Producto/Servicio, Nº ID interno, Cantidad, Unidad de Medida, Clave Unidad SAT, Valor Unitario, Importe, desglose de impuestos. No se ha confirmado si provienen del Catálogo de Productos, del Pedido, de catálogos SAT o de configuración adicional. Duda general aplicable a toda la sección de partidas. **

═══════════════════════════════════════════════════════════════
SECCIÓN F — TOTALES E IMPUESTOS
═══════════════════════════════════════════════════════════════

Criterio F1 — Retenciones
Dado que el sistema renderiza el pie de totales,
Cuando incluye la sección de retenciones,
Entonces deberá mostrar las retenciones aplicables al CFDI (si no hay retenciones, la sección se muestra sin contenido).

Criterio F2 — Traslados (impuestos trasladados)
Dado que el sistema renderiza los impuestos trasladados,
Cuando incluye los datos,
Entonces deberá mostrar el desglose del IVA con: IMPUESTO, TIPO FACTOR, TASA/CUOTA, IMPORTE.

Criterio F3 — Total expresado en letra
Dado que el sistema renderiza el total,
Cuando incluye la conversión a letras,
Entonces deberá mostrar el monto en palabras según la moneda (ejemplo: "TREINTA Y UN MIL QUINIENTOS SETENTA DOLARES 00/100").

Criterio F4 — Moneda y Tipo de Cambio
Dado que el sistema renderiza los datos monetarios,
Cuando incluye la información,
Entonces deberá mostrar:
- Moneda con nombre completo.
- Tipo de Cambio al dia de la generación.

Criterio F5 — Subtotal, Impuestos Federales y Total
Dado que el sistema renderiza el bloque final,
Cuando incluye los montos,
Entonces deberá mostrar:
- Subtotal (suma de los importes de las partidas).
- Impuestos Federales (suma de los traslados de IVA).
- Total (Subtotal + Impuestos Federales).

═══════════════════════════════════════════════════════════════
SECCIÓN G — ELEMENTOS TÉCNICOS SAT
═══════════════════════════════════════════════════════════════

Criterio G1 — Elementos de certificación SAT
Dado que el sistema renderiza los elementos técnicos de certificación,
Cuando incluye los datos,
Entonces deberá mostrar:
- Código QR de validación.
- Número de Serie del Certificado del SAT.
- Número de Serie del CSD del Emisor.
- Sello Digital del SAT.
- Sello Digital del CFDI.
- Cadena Original del Complemento de Certificación Digital del SAT.

Criterio G2 — Origen técnico de los elementos SAT
Dado que el sistema renderiza la sección de elementos técnicos,
Cuando obtiene los valores,
Entonces deberá tomarlos del TimbreFiscalDigital del XML del CFDI timbrado por el PAC. El sistema NO calcula ni genera estos elementos; los recibe del PAC y los renderiza en el PDF.

Criterio G3 — Identificadores del pedido
Dado que el sistema renderiza la sección de identificadores,
Cuando incluye los datos,
Entonces deberá mostrar:
- Serie y Folio del CFDI.
- Folio del Pedido Interno (PI) del sistema PQF2.

═══════════════════════════════════════════════════════════════
SECCIÓN H — INFORMACIÓN INSTITUCIONAL DE LA EMPRESA EMISORA
═══════════════════════════════════════════════════════════════

Criterio H1 — Disclaimer de representación impresa
Dado que el sistema renderiza el pie del documento,
Cuando incluye el disclaimer,
Entonces deberá mostrar el texto fijo: "Representación impresa de un CFDI 4.0".

Criterio H2 — Certificaciones y métodos de pago aceptados
Dado que el sistema renderiza el pie,
Cuando incluye certificaciones y métodos de pago,
Entonces deberá mostrar las certificaciones vigentes (ISO 9001:2015, NEEC) y los métodos de pago aceptados aplicables. ** Validar con el cliente vigencia y diseño actualizado. **

Criterio H3 — Logos de catálogos farmacéuticos
Dado que el sistema renderiza el cierre del documento,
Cuando incluye los logos institucionales,
Entonces deberá mostrar los logos aplicables a la empresa emisora (EDQM, FEUM, USP, Microbiologics, APACOR, CHATA Biosystems, Pharmaffiliates). ** Validar vigencia con el cliente. **

Criterio H4 — Numeración de página
Dado que el sistema completa el documento,
Cuando incluye el contador de páginas,
Entonces deberá mostrar "X de Y" en el pie del documento.

═══════════════════════════════════════════════════════════════
SECCIÓN I — PAGINACIÓN AUTOMÁTICA
═══════════════════════════════════════════════════════════════

Criterio I1 — Múltiples páginas cuando las partidas exceden una página
Dado que el pedido tiene partidas que exceden el espacio disponible en una sola página,
Cuando el sistema renderiza el documento,
Entonces deberá generar páginas adicionales con la misma cabecera y pie completo. La numeración se actualiza (1 de 5, 2 de 5, ..., 5 de 5). Este comportamiento ya existe en PQF2.

═══════════════════════════════════════════════════════════════
SECCIÓN J — PERSISTENCIA
═══════════════════════════════════════════════════════════════

Criterio J1 — Persistencia al timbrado exitoso
Dado que el PAC confirma el timbrado exitoso de la Factura,
Cuando el sistema recibe la respuesta del PAC,
Entonces deberá persistir el PDF y el XML timbrado en base de datos como artefacto fiscal inmutable.

Criterio J2 — Sin regeneración posterior
Dado que la Factura fue timbrada y persistida,
Cuando un usuario consulta el PDF en cualquier momento posterior,
Entonces el sistema deberá entregar el PDF almacenado en base de datos sin regenerarlo. La Factura es artefacto fiscal inmutable. Si los datos fuente del cliente o del emisor cambian después del timbrado, la Factura histórica conserva los datos originales sin modificación.

═══════════════════════════════════════════════════════════════
SECCIÓN K — VARIANTE: FACTURA ANTICIPO (PRODUCTOS CONTROLADOS)
═══════════════════════════════════════════════════════════════

** Nota de la sección — NADA en esta sección está confirmado. PROQUIFA aún no tiene implementada la Factura Anticipo ni cuenta con un PDF de referencia; apenas se va a implementar. Todo el contenido siguiente son SUGERENCIAS basadas en la mecánica general del SAT y en la estructura de la factura normal de este requisito (Secciones A–J), y debe tratarse como pendiente de confirmar con el asesor comercial. No se sabe aún cómo debe quedar el documento. **

Contexto: La Factura Anticipo se emitiría para pedidos con productos controlados (Mundial, Nacional u Origen). Se entiende que PQF2 generaría únicamente el CFDI de Ingreso de PRIMERA EMISIÓN por el monto del anticipo (no la factura final de cierre con aplicación de anticipo / relación 07). Bajo ese entendido, las siguientes son sugerencias a validar.

Criterio K1 (sugerido) — Estructura base reutilizada
Dado que el sistema generaría el PDF de una Factura Anticipo,
Cuando arma el documento,
Entonces ** se sugiere reutilizar la misma estructura del PDF de la factura normal (Secciones A–J): datos del emisor, receptor, datos del CFDI, referencias bancarias, partidas, totales e impuestos, elementos técnicos SAT, información institucional, paginación y persistencia. Pendiente de confirmar con el asesor comercial. **

Criterio K2 (sugerido) — Concepto de la partida: anticipo
Dado que el sistema renderiza la sección de partidas de una Factura Anticipo,
Cuando arma el concepto,
Entonces ** se sugiere que el concepto corresponda al anticipo recibido y no al detalle de productos del pedido. El detalle exacto (clave SAT del producto/servicio para anticipos, descripción, importe, IVA) está sin definir. Pendiente de confirmar con el asesor comercial. **

Criterio K3 (sugerido) — Pedimento y CFDI relacionados
Dado que la Factura Anticipo sería un CFDI de primera emisión,
Cuando el sistema arma el documento,
Entonces ** se sugiere que NO incluya número de pedimento (N/A, igual que la factura normal de primera emisión) ni sección de CFDI Relacionados (tipo de relación 07), bajo el entendido de que PQF2 no genera la factura final de cierre. Pendiente de confirmar con el asesor comercial si este entendido es correcto. **

Criterio K4 (sugerido) — Método de Pago
Dado que el sistema renderiza los datos fiscales del CFDI de una Factura Anticipo,
Cuando incluye el Método de Pago,
Entonces ** conforme a la mecánica general del SAT, los anticipos suelen emitirse con Método de Pago PUE (Pago en una sola exhibición); es solo una sugerencia. Pendiente de confirmar con el asesor comercial. **

** Cierre de la sección — La estructura definitiva de la Factura Anticipo se definirá cuando PROQUIFA la implemente y/o el asesor comercial confirme el diseño. Mientras tanto, todo lo anterior es orientativo y no debe tomarse como especificación cerrada. **

---

## Notas Adicionales

- Esta fila documenta el contenido y la representación impresa (PDF) de la Factura CFDI 4.0 para clientes con Región México. La equivalente para Región Perú se documenta en requisito independiente.
- El requisito es un rediseño del documento de Factura. La estructura visual específica (colores exactos, layout, tipografía, espaciados) es decisión del equipo de diseño UI; este requisito se enfoca en la información que debe contener cada sección del documento.
- El diseño se valida contra cuatro CFDIs reales recibidos del cliente (folios 2374 Mungen, 7156 Golocaer, 20913 Proquifa, 143103 Proveedora Quimico Farmaceutica).
- Aplica a Facturas generadas desde el módulo Factura por Adelantado (pedidos Crédito sin controlados, Prepago sin controlados) y a Facturas generadas desde Validar Cobro (cobro recibido aplicado a Proforma de Prepago sin controlados).
- Las Facturas Anticipo (Prepago con productos controlados, generadas desde Validar Cobro) tienen reglas propias y se documentan en requisito independiente.
- El branding del documento varía por empresa emisora del pedido: logo, colores, certificaciones. Las cuatro empresas del grupo PROQUIFA México son Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V. y Proveedora Quimico Farmaceutica S.A. de C.V. El branding lo genera el sistema según la empresa emisora del pedido.
- El PDF se persiste en base de datos al timbrarse exitosamente la Factura ante el PAC. La Factura es artefacto fiscal inmutable.
- El PAC utilizado por PROQUIFA es TurboPac (Quadrum Tecnologías). Los elementos técnicos SAT del PDF (folio fiscal UUID, sellos digitales, cadena original, números de serie de certificados, código QR) son generados por el PAC al timbrar e incluidos en el TimbreFiscalDigital del XML. El sistema renderiza estos valores en el PDF, no los calcula por su cuenta.
- El folio de la Factura es un consecutivo numérico independiente por empresa emisora (varchar 6). Cada empresa mantiene su propio contador.
- El Folio Fiscal (UUID de 36 caracteres) es asignado por el SAT al timbrar el CFDI y es el identificador fiscal verdadero del documento ante la autoridad.
- En operación actual los CFDIs son emitidos con Régimen Fiscal del Emisor 601 (General de Ley Personas Morales) para las cuatro empresas del grupo.
- Los Usos CFDI observados en los ejemplos son: S01 (Sin efectos fiscales), G03 (Gastos en general). El catálogo completo lo define el SAT y es seleccionado por el usuario al generar la Factura desde el módulo correspondiente.
- El Tipo de Cambio aplicado a la Factura es el del día de la generación, no modificable por el usuario (regla SAT confirmada por el cliente). El Tipo de Cambio aplica cuando la moneda de la Factura difiere de la moneda de origen (por ejemplo, Proforma en EUR convertida a Factura en USD, u otras combinaciones).
- El disclaimer del documento es "Representación impresa de un CFDI 4.0".
- ** Pendiente clarificar el origen operativo de TODOS los datos de las partidas: descripción del producto, Lote, Pedimento, Clave SAT del Producto/Servicio (c_ClaveProdServ), Nº ID interno, Cantidad, Unidad de Medida, Clave Unidad SAT (c_ClaveUnidad), Valor Unitario, Importe, desglose de impuestos. No se ha confirmado si provienen del Catálogo de Productos, del Pedido, de catálogos SAT o de configuración adicional. Duda general aplicable a toda la sección de partidas. Requiere sesión específica con el cliente para mapear cada dato de partida a su fuente operativa. **
- ** Pendiente confirmar si la construcción de la referencia bancaria del cliente en la Factura sigue el mismo método que en la Proforma (lógica Banamex con concatenación de segmentos / lógica no-Banamex con nombre del cliente directo) o si tiene reglas propias. **
- ** Pendiente confirmar la diferencia entre el número de orden de compra del cliente (referencia externa proveniente del Pedido) y el Folio del Pedido Interno (PI) del sistema PQF2 que aparece en la sección de identificadores. Aclarar si son datos distintos o complementarios. **
- ** Pendiente validar con el cliente la vigencia y diseño actualizado de las certificaciones del pie del documento (ISO 9001:2015, NEEC) y de los métodos de pago aceptados. **
- ** Pendiente validar con el cliente la vigencia de los logos de catálogos farmacéuticos del pie del documento. **
- ** Pendiente decisión técnica: tipo de almacenamiento del PDF persistido en base de datos (archivo binario completo vs snapshot estructurado de datos). Decisión transversal con Proforma. **
- ** Pendiente — maquetas de Golocaer S.A.C. Perú no disponibles; criterios de detalle de modales se validarán contra ellas cuando lleguen. **
- ** Duda — Una factura electrónica real de Golocaer S.A.C. (E001-362, 08/05/2026) recibida como muestra fue emitida a Crédito (con cuotas y fecha de vencimiento), timbrada ante SUNAT. Esto contrasta con el requisito del cliente de que en Perú el flujo R16 aplica solo a Prepago. Pendiente confirmar: (1) si Golocaer factura habitualmente a crédito en su operación corriente (ajena a R16) o si el caso prepago es el único contemplado; (2) confirmar que el alcance de R16 para Perú se mantiene exclusivamente en Prepago. Mientras tanto, se conserva la decisión vigente: Perú = solo Prepago. **
