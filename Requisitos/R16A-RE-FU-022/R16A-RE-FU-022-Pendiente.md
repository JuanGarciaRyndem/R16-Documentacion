# R16A-RE-FU-022 — Diseño y generación de Documentos: Factura Peru

| Campo                   | Valor                                                                                                                                                                                                                                                                                                                                                                                   |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ID**                  | R16A-RE-FU-022                                                                                                                                                                                                                                                                                                                                                                          |
| **Título**              | Diseño y generación de Documentos: Factura Peru                                                                                                                                                                                                                                                                                                                                         |
| **Módulo / Épica**      | Factura por Adelantado                                                                                                                                                                                                                                                                                                                                                                  |
| **Historia de Usuario** | Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente de resolver) **, quiero que el sistema genere automáticamente el PDF (representación impresa) de la factura electrónica de Golocaer S.A.C. al timbrarse exitosamente ante SUNAT, para entregar al cliente Perú el comprobante de pago electrónico y conservarlo como artefacto fiscal inmutable. |
| **Prioridad**           | Alta                                                                                                                                                                                                                                                                                                                                                                                    |
| **Estado**              | Propuesto                                                                                                                                                                                                                                                                                                                                                                               |
| **Requisito asociado**  | R16.1M-RE-FU-007                                                                                                                                                                                                                                                                                                                                                                        |

---

## Requisito Funcional

El sistema debe generar el PDF (representación impresa) de la factura electrónica (Comprobante de Pago Electrónico tipo 01, UBL 2.1) al timbrarse exitosamente ante SUNAT, para clientes con Región Perú, con un diseño estandarizado bajo el branding de Golocaer S.A.C. El PDF se persiste en base de datos junto con el XML timbrado como artefacto fiscal inmutable. Esta funcionalidad aplica a las facturas Prepago generadas desde el módulo Factura por Adelantado y desde Validar Cobro para la Región Perú.

---

## Alcance

### Aplica a

- Generación del PDF de la factura electrónica (CPE tipo 01, UBL 2.1) al timbrarse exitosamente ante SUNAT, para clientes con Región Perú.
- Facturas originadas en el módulo Factura por Adelantado (pedidos Prepago sin controlados — ver R16A-RE-FU-020).
- Facturas originadas en Validar Cobro (cobro recibido aplicado a Proforma de Prepago sin controlados, Región Perú).
- Empresa emisora: Golocaer S.A.C., con branding propio (logo, colores, certificaciones aplicables a Perú, lugar de expedición, RUC, dirección y razón social legal).
- Numeración con serie alfanumérica de 4 caracteres más correlativo de hasta 8 dígitos, consecutivo e independiente por serie.
- Paginación automática cuando las partidas exceden el espacio de una página (comportamiento ya existente del sistema).
- Persistencia del PDF (junto con el XML timbrado) en base de datos al timbrarse exitosamente.
- En el flujo de Factura por Adelantado (prepago), la factura se emite antes del despacho, por lo que NO referenciaría GRE. El campo de referencia a GRE es fiscalmente opcional en SUNAT. ** Pendiente de validar con el cliente la secuencia GRE↔factura en prepago. **

### No aplica a

- Factura Anticipo (pedidos Prepago con productos controlados): se documenta en requisito independiente.
- Facturas Crédito de la región Perú: la Factura por Adelantado no aplica a Crédito en Perú porque el timbrado peruano en R16 se limita a Prepago (ver R16A-RE-FU-012 y R16A-RE-FU-020).
- Clientes con Región México: el PDF de la Factura México se documenta en R16A-RE-FU-021.
- Generación del XML del CPE (estructura técnica UBL 2.1): es responsabilidad del proveedor de timbrado SUNAT/OSE y se rige por el estándar SUNAT.
- Lógica de timbrado, integración con SUNAT/OSE, manejo de errores: vive en los requisitos de los módulos que generan facturas (R16A-RE-FU-020 y Validar Cobro Perú).
- Cancelación de la factura timbrada: se documenta en el módulo Notas de Crédito Perú.
- Envío de la factura al cliente por correo electrónico: vive en los requisitos de los módulos que generan facturas.
- Complemento de Pago: no existe en Perú (SUNAT no tiene documento equivalente).
- Generación de la Guía de Remisión Electrónica (GRE): documento independiente que acompaña el despacho de mercancía, fuera del alcance de este requisito (ver Brecha 2 de R16A-RE-FU-005).

---

## Reglas de Negocio

Regla 1 — Generación al timbrado exitoso
Cuando el timbrado ante SUNAT/OSE se completa exitosamente, el sistema genera el PDF de la factura y lo persiste en base de datos junto con el XML timbrado como artefacto fiscal inmutable.

Regla 2 — Branding de Golocaer S.A.C.
El PDF aplica el branding de Golocaer S.A.C. (logo, colores, certificaciones aplicables, lugar de expedición, RUC, dirección y razón social legal). En la factura real de muestra el encabezado combina la marca PROQUIFA PERU con la razón social GOLOCAER S.A.C. ** Branding y datos legales de Golocaer S.A.C. Perú pendientes de recopilar y confirmar. **

Regla 3 — Numeración SUNAT por serie
El número de la factura usa serie alfanumérica de 4 caracteres más correlativo de hasta 8 dígitos, consecutivo e independiente por serie. ** Esquema de series de Golocaer S.A.C. pendiente de definir. **

Regla 4 — Formato UBL 2.1 y disclaimer
El PDF refleja un Comprobante de Pago Electrónico conforme al estándar UBL 2.1 de SUNAT. El disclaimer del documento indica que es la representación impresa de la factura electrónica generada en el sistema de SUNAT y que puede verificarse con la clave SOL.

Regla 5 — Inmutabilidad de la factura timbrada
Una vez timbrada exitosamente, el PDF se entrega siempre desde el almacenado en base de datos sin posibilidad de modificación. La factura timbrada es artefacto fiscal inmutable; cualquier corrección posterior requiere Nota de Crédito SUNAT.

Regla 6 — Paginación automática (comportamiento existente)
Cuando las partidas exceden el espacio de una página, el sistema genera páginas adicionales con la misma cabecera y pie completo, con numeración "X de Y". Este comportamiento ya existe en PQF2.

Regla 7 — Impuesto IGV en lugar de IVA
El PDF muestra el IGV (18%) en lugar del IVA mexicano, con el desglose de la afectación al IGV por partida.

Regla 8 — Origen de los datos por sección
El PDF se arma desde el CPE timbrado: datos del Receptor (Razón Social, RUC, dirección del receptor) y del Emisor (Golocaer S.A.C.: Razón Social, RUC, establecimiento del emisor) del comprobante timbrado; datos del comprobante (Serie, Correlativo, Tipo de Operación, Tipo de Comprobante, Moneda, Tipo de Cambio cuando aplique, Condición de Pago) del comprobante timbrado; elementos técnicos de certificación SUNAT (firma digital, código QR, valor resumen) de la respuesta de timbrado; branding generado por el sistema. ** El origen de los datos de partidas (descripción, código SUNAT del producto, unidad de medida SUNAT, afectación al IGV, cantidad, valor unitario, importe) depende de la captura de los datos fiscales SUNAT del producto, brecha bloqueante documentada en Brecha 1 de R16A-RE-FU-005. **

---

## Riesgos

Riesgo 1 — Datos fiscales del producto SUNAT inexistentes (bloqueante)
El catálogo de productos actual no contiene los datos fiscales SUNAT (código SUNAT, unidad de medida SUNAT, afectación al IGV por línea) que deben renderizarse en las partidas del PDF y son obligatorios en el XML UBL 2.1. Sin ellos no es posible generar la factura, por lo que esta brecha puede detener el desarrollo del flujo de facturación Perú si no se resuelve antes (ver Brecha 1 de R16A-RE-FU-005).

Riesgo 2 — Modelo bancario de Golocaer S.A.C. Perú no disponible
El PDF mexicano muestra cuentas bancarias y referencia del cliente; la factura real de Golocaer recibida como muestra NO muestra esa sección, y la definición para Perú no está disponible. La lógica mexicana no replica al contexto peruano (ver R16A-RE-FU-006, Riesgo 4).

Riesgo 3 — Branding, certificaciones y disclaimer de Golocaer S.A.C. Perú no confirmados
El logo, colores, certificaciones aplicables a Perú y el texto del disclaimer legal SUNAT de Golocaer S.A.C. no están confirmados. Si el PDF se publica con datos de México (certificaciones mexicanas, disclaimer SAT), genera incoherencia legal frente al cliente peruano.

Riesgo 4 — Brecha de timbrado SUNAT/OSE no resuelta
El PDF se genera al timbrarse exitosamente; mientras la integración con SUNAT/OSE no se resuelva (brecha mayor), no hay timbrado y por tanto no hay PDF (ver Brecha 5 de R16A-RE-FU-005).

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — DATOS DEL EMISOR (GOLOCAER S.A.C.)
═══════════════════════════════════════════════════════════════

Criterio A1 — Logo y branding de Golocaer S.A.C.
Dado que el sistema renderiza el documento,
Cuando incluye el branding,
Entonces deberá mostrar el logo y la identidad visual de Golocaer S.A.C. ** Branding de Golocaer S.A.C. Perú pendiente de confirmar. **

Criterio A2 — Datos institucionales del emisor
Dado que el sistema renderiza los datos del emisor,
Cuando incluye la información,
Entonces deberá mostrar: Razón Social de Golocaer S.A.C., RUC del emisor, establecimiento del emisor (dirección completa con distrito/provincia/departamento), y fecha y hora de emisión. ** Datos legales de Golocaer S.A.C. Perú pendientes de recopilar. **

═══════════════════════════════════════════════════════════════
SECCIÓN B — DATOS DEL RECEPTOR (CLIENTE)
═══════════════════════════════════════════════════════════════

Criterio B1 — Identificación y datos fiscales del receptor
Dado que el sistema renderiza los datos del receptor,
Cuando incluye la información,
Entonces deberá mostrar: Razón Social del receptor, RUC, y dirección del receptor de la factura. El identificador fiscal es RUC en lugar de RFC. (En la factura real de muestra no se muestran Uso CFDI ni Régimen del receptor, que son conceptos del SAT.)

═══════════════════════════════════════════════════════════════
SECCIÓN C — DATOS DEL COMPROBANTE
═══════════════════════════════════════════════════════════════

Criterio C1 — Identificadores del comprobante
Dado que el sistema renderiza los datos del comprobante,
Cuando incluye los identificadores,
Entonces deberá mostrar: Serie y Correlativo (ejemplo "E001-362"), Tipo de Comprobante (01 - Factura) y Tipo de Operación (catálogo 51). NO se muestra Folio Fiscal UUID (es un concepto del SAT que no existe en SUNAT).

Criterio C2 — Fechas del comprobante
Dado que el sistema renderiza el documento,
Cuando incluye las marcas temporales,
Entonces deberá mostrar la fecha y hora de emisión del comprobante.

Criterio C3 — Datos fiscales del comprobante
Dado que el sistema renderiza los datos fiscales del comprobante,
Cuando incluye la información,
Entonces deberá mostrar: Tipo de Comprobante (Factura), Condición de Pago (Contado/Crédito según corresponda — en la factura real de muestra apareció como "Forma de pago: Crédito"), Moneda (con código y nombre, ejemplo "DOLAR AMERICANO") y Tipo de Cambio del día cuando la moneda no sea PEN. NO se muestran Método de Pago PPD, Forma de Pago 99 ni Complemento de Pago (no aplican a Perú).

Criterio C4 — Referencia a Guía de Remisión en prepago
Dado que la factura se genera desde el flujo de Factura por Adelantado (prepago), antes del despacho de la mercancía,
Cuando el sistema arma el comprobante,
Entonces no debería incluir referencia a Guía de Remisión Electrónica, ya que al momento de facturar no existiría traslado (el campo es fiscalmente opcional en SUNAT). ** Pendiente de validar con el cliente la secuencia GRE↔factura en el flujo prepago de Perú. **

═══════════════════════════════════════════════════════════════
SECCIÓN D — REFERENCIAS BANCARIAS
═══════════════════════════════════════════════════════════════

Criterio D1 — Cuentas bancarias y referencia del cliente
Dado que el sistema renderiza el documento,
Cuando arma la sección de referencias bancarias,
Entonces ** pendiente de validar con el cliente: la factura real de Golocaer recibida como muestra NO muestra cuentas bancarias en el PDF. Confirmar si las facturas de Golocaer Perú incluyen sección de cuentas bancarias (Banco, Número de Cuenta, Moneda, CCI de 20 dígitos) y referencia bancaria del cliente, o si esa sección no aplica a Perú. La lógica mexicana (CLABE, referencia/Código Validador) no replica al contexto peruano (ver R16A-RE-FU-006, Riesgo 4). **

Criterio D2 — Referencia del pedido del cliente
Dado que el sistema renderiza la sección,
Cuando incluye el dato del pedido,
Entonces deberá mostrar el número de orden de compra del cliente o referencia equivalente proveniente del Pedido (en la factura real de muestra aparece como "ORDEN DE COMPRA" en la descripción de cada partida).

═══════════════════════════════════════════════════════════════
SECCIÓN E — PARTIDAS
═══════════════════════════════════════════════════════════════

Criterio E1 — Datos de cada partida
Dado que el sistema renderiza la tabla de partidas,
Cuando incluye los datos por cada partida,
Entonces deberá mostrar:
- Cantidad.
- Unidad de Medida según catálogo SUNAT.
- Código SUNAT del producto.
- Descripción del producto (incluye nombre, orden de compra, catálogo y lote cuando aplique).
- Valor Unitario con moneda.
- Afectación al IGV por línea (gravado, exonerado, inafecto, exportación, gratuito, etc.).
- ICBPER por línea cuando aplique (Impuesto al Consumo de Bolsas Plásticas).
- ** Tipo de Precio por línea (catálogo 16 SUNAT, ejemplo "01" precio unitario que incluye IGV): observado en la factura real de Golocaer; pendiente de validar si debe mostrarse. SAT no tiene un campo equivalente. **

Criterio E2 — Origen de los datos fiscales del producto
Dado que el sistema arma los datos de cada partida,
Cuando obtiene el código SUNAT del producto, la unidad de medida SUNAT y la afectación al IGV,
Entonces ** estos datos no existen en el catálogo de productos actual; brecha bloqueante que debe resolverse antes del desarrollo (ver Brecha 1 de R16A-RE-FU-005). **

═══════════════════════════════════════════════════════════════
SECCIÓN F — TOTALES E IMPUESTOS
═══════════════════════════════════════════════════════════════

Criterio F1 — Desglose de totales SUNAT
Dado que el sistema renderiza el bloque de totales,
Cuando incluye los montos,
Entonces deberá mostrar los conceptos del CPE SUNAT (mostrando en cero los que no apliquen a la operación):
- Sub Total Ventas.
- Anticipos.
- Descuentos.
- Valor de Venta de Operaciones Gratuitas.
- Valor Venta.
- ISC (Impuesto Selectivo al Consumo).
- IGV (18%).
- ICBPER (Impuesto al Consumo de Bolsas Plásticas).
- Otros Cargos.
- Otros Tributos.
- Monto de redondeo.
- Importe Total.

Criterio F2 — Total expresado en letra
Dado que el sistema renderiza el total,
Cuando incluye la conversión a letras,
Entonces deberá mostrar el monto en palabras según la moneda (ejemplo: "SON: DIECIOCHO MIL SETECIENTOS CUATRO Y 18/100 DOLAR AMERICANO").

Criterio F3 — Moneda y Tipo de Cambio
Dado que el sistema renderiza los datos monetarios,
Cuando incluye la información,
Entonces deberá mostrar la Moneda con nombre completo y el Tipo de Cambio del día de la generación cuando la moneda no sea PEN.

Criterio F4 — Información del crédito (cuando aplique la condición de pago Crédito)
Dado que la factura tiene condición de pago Crédito,
Cuando el sistema renderiza el bloque de crédito,
Entonces deberá mostrar el Monto neto pendiente de pago, el Total de Cuotas y el detalle por cuota (Número de cuota, Fecha de vencimiento, Monto). ** Observado en la factura real de muestra (a crédito); su aplicación en R16 está sujeta a la confirmación de que Perú es solo Prepago — ver duda en R16A-RE-FU-020. **

═══════════════════════════════════════════════════════════════
SECCIÓN G — ELEMENTOS TÉCNICOS SUNAT
═══════════════════════════════════════════════════════════════

Criterio G1 — Elementos de certificación SUNAT
Dado que el sistema renderiza los elementos técnicos de certificación,
Cuando incluye los datos,
Entonces deberá mostrar: el código QR de validación SUNAT, el valor resumen (hash) del comprobante y la firma digital, conforme al estándar UBL 2.1. NO se muestran los elementos del SAT (Folio Fiscal UUID, sellos digitales del SAT, cadena original) porque no existen en SUNAT.

Criterio G2 — Origen técnico de los elementos SUNAT
Dado que el sistema renderiza la sección de elementos técnicos,
Cuando obtiene los valores,
Entonces deberá tomarlos del XML del CPE firmado y de la respuesta de SUNAT/OSE. El sistema NO calcula los elementos de aceptación; los recibe del proceso de timbrado y los renderiza en el PDF.

Criterio G3 — Identificadores del pedido
Dado que el sistema renderiza la sección de identificadores,
Cuando incluye los datos,
Entonces deberá mostrar la Serie y Correlativo del comprobante y el Folio del Pedido Interno (PI) del sistema PQF2.

═══════════════════════════════════════════════════════════════
SECCIÓN H — INFORMACIÓN INSTITUCIONAL DE GOLOCAER S.A.C.
═══════════════════════════════════════════════════════════════

Criterio H1 — Disclaimer de representación impresa
Dado que el sistema renderiza el pie del documento,
Cuando incluye el disclaimer,
Entonces deberá mostrar el texto que identifica el documento como representación impresa de la factura electrónica generada en el sistema de SUNAT, verificable con la clave SOL. ** Texto exacto pendiente de validar con asesor legal peruano. **

Criterio H2 — Certificaciones aplicables a Golocaer S.A.C. Perú
Dado que el sistema renderiza el pie,
Cuando incluye certificaciones,
Entonces deberá mostrar las certificaciones vigentes aplicables a Golocaer S.A.C. Perú. ** Las certificaciones mexicanas (ISO 9001:2015, NEEC, FEUM) no aplican; las que apliquen a Perú quedan por validar con el cliente. **

Criterio H3 — Numeración de página
Dado que el sistema completa el documento,
Cuando incluye el contador de páginas,
Entonces deberá mostrar "X de Y" en el pie del documento.

═══════════════════════════════════════════════════════════════
SECCIÓN I — PAGINACIÓN AUTOMÁTICA
═══════════════════════════════════════════════════════════════

Criterio I1 — Múltiples páginas cuando las partidas exceden una página
Dado que el pedido tiene partidas que exceden el espacio disponible en una sola página,
Cuando el sistema renderiza el documento,
Entonces deberá generar páginas adicionales con la misma cabecera y pie completo, actualizando la numeración (1 de N, 2 de N, ...). Este comportamiento ya existe en PQF2.

═══════════════════════════════════════════════════════════════
SECCIÓN J — PERSISTENCIA
═══════════════════════════════════════════════════════════════

Criterio J1 — Persistencia al timbrado exitoso
Dado que SUNAT/OSE confirma el timbrado exitoso de la factura,
Cuando el sistema recibe la respuesta,
Entonces deberá persistir el PDF y el XML timbrado en base de datos como artefacto fiscal inmutable.

Criterio J2 — Sin regeneración posterior
Dado que la factura fue timbrada y persistida,
Cuando un usuario consulta el PDF en cualquier momento posterior,
Entonces el sistema deberá entregar el PDF almacenado en base de datos sin regenerarlo. Si los datos fuente del cliente o del emisor cambian después del timbrado, la factura histórica conserva los datos originales sin modificación.

---

## Notas Adicionales

- Fila para el PDF (representación impresa) de la factura electrónica de la Región Perú (solo Prepago), contraparte de R16A-RE-FU-021 (México). Estado depende de la resolución de las brechas Perú.
- El documento es la representación impresa de un Comprobante de Pago Electrónico SUNAT (CPE tipo 01, UBL 2.1), no un CFDI. Diferencias clave vs México: IGV en lugar de IVA; RUC en lugar de RFC; código QR, firma digital y valor resumen SUNAT en lugar de UUID/sellos/cadena original del SAT; sin Método de Pago PPD, Forma de Pago 99 ni Complemento de Pago; numeración por serie + correlativo; totales con conceptos SUNAT (ISC, ICBPER, Anticipos, Descuentos, Operaciones Gratuitas, Otros Tributos).
- Empresa emisora: Golocaer S.A.C. El encabezado de la factura real combina la marca PROQUIFA PERU con la razón social Golocaer S.A.C.
- ** Brecha bloqueante — Datos fiscales SUNAT del producto (código SUNAT, unidad de medida SUNAT, afectación al IGV por línea) inexistentes en el catálogo de productos actual; necesarios para renderizar las partidas. Puede detener el desarrollo del flujo de facturación Perú si no se resuelve antes (ver Brecha 1 de R16A-RE-FU-005). **
- ** Brecha — Cuentas bancarias y referencia del cliente: la factura real de Golocaer NO muestra sección bancaria. Pendiente validar si aplica a Perú y, de aplicar, su modelo (CCI, monedas PEN/USD). La lógica mexicana no replica a Perú (ver R16A-RE-FU-006, Riesgo 4). **
- ** Brecha — Branding, certificaciones aplicables y disclaimer legal SUNAT de Golocaer S.A.C. Perú. Las certificaciones mexicanas (NEEC, FEUM, ISO) no aplican. **
- ** Brecha — Integración SUNAT/OSE-PSE para timbrado (condiciona la generación del PDF). Ver Brecha 5 de R16A-RE-FU-005. **
- La generación de la GRE (Guía de Remisión Electrónica) es un proceso independiente del PDF de la factura, fuera del alcance de este requisito (ver Brecha 2 de R16A-RE-FU-005). En el flujo prepago la factura va antes del despacho, por lo que no referenciaría GRE. ** Pendiente de validar con el cliente la secuencia GRE↔factura en prepago. **
- Fundamento normativo: Reglamento de Comprobantes de Pago, R.S. 097-2012/SUNAT y modificatorias; formato UBL 2.1.
- ** Referencia — Se recibió como muestra una factura electrónica real de Golocaer S.A.C. (E001-362) con su XML UBL 2.1 y su Guía de Remisión Electrónica (EG07-00000350). Observaciones del ejemplo (un solo caso, NO confirmado como patrón general): serie "E001"; unidad de medida código "C62" (unidad/pieza); código de producto SUNAT único genérico ("41116107") para todas las líneas; la factura referencia la GRE que amparó el traslado; afectación al IGV "gravado" (18%); no muestra sección de cuentas bancarias; fue emitida a crédito con detalle de cuotas (ver duda relacionada en R16A-RE-FU-020). Estos datos deben validarse con el cliente antes de elevarse a regla. **
- ** Pendiente — maquetas de la factura de Golocaer S.A.C. Perú no disponibles; el detalle visual del PDF se validará contra ellas cuando lleguen. **
