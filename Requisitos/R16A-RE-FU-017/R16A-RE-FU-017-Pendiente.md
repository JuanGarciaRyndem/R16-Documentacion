# R16A-RE-FU-017 — Diseño y generación de Documentos: Proforma Perú

| Campo                   | Valor                                                                                                                                                                                                                                                                                                   |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ID**                  | R16A-RE-FU-017                                                                                                                                                                                                                                                                                          |
| **Título**              | Diseño y generación de Documentos: Proforma Perú                                                                                                                                                                                                                                                        |
| **Módulo / Épica**      | Tramitar Pedido                                                                                                                                                                                                                                                                                         |
| **Historia de Usuario** | Yo como ESAC, quiero que el sistema genere automáticamente el PDF de la Proforma adaptado a la normativa fiscal SUNAT al tramitar un pedido Prepago para clientes de Perú, para entregar al cliente un documento estandarizado y conforme a la regulación peruana que respalde el cobro por adelantado. |
| **Prioridad**           | Alta                                                                                                                                                                                                                                                                                                    |
| **Estado**              | Propuesto                                                                                                                                                                                                                                                                                               |
| **Requisito asociado**  | R16.1M-RE-FU-007                                                                                                                                                                                                                                                                                        |

---

## Requisito Funcional

El sistema debe generar un PDF de Proforma al tramitar un pedido Prepago sin Factura por Adelantado para clientes con Región Perú, con un diseño estandarizado equivalente al de la Proforma México pero adaptado a la normativa fiscal SUNAT. Dado que la única empresa emisora del grupo operando en Perú es Golocaer S.A.C., el branding del documento es único. El PDF se genera bajo demanda durante el flujo previo al envío al cliente y, al confirmarse el envío, se persiste como artefacto histórico inmutable accesible desde el módulo Validar Cobro.

---

## Alcance

### Aplica a

- Generación del PDF de Proforma al tramitar un pedido en modalidad Prepago sin factura por adelantado para clientes con Región Perú.
- Empresa emisora única: Golocaer S.A.C. (la única empresa del grupo PROQUIFA operando actualmente en Perú).
- Generación bajo demanda del PDF durante el flujo previo al envío de la Proforma al cliente.
- Persistencia del PDF en base de datos al recibir confirmación de envío exitoso del correo al cliente.
- Acceso al PDF histórico desde el módulo Validar Cobro una vez la Proforma fue enviada.
- Foliador global lineal PQF2 con prefijo PRF en la representación visual del documento (compartido con Proformas México: un solo contador global de Proformas para todo el grupo).
- Paginación automática cuando las partidas exceden el espacio de una página (comportamiento ya existente del sistema).
- Aplicación de catálogos fiscales SUNAT peruanos (RUC, IGV, CCI, moneda PEN, Reglamento de Comprobantes de Pago y Resolución de Superintendencia N° 097-2012/SUNAT).

### No aplica a

- Pedidos Crédito sin Factura por Adelantado ni Crédito/Prepago con Factura por Adelantado.
- Pedidos para clientes con Región México. Esa funcionalidad se documenta en requisito independiente.
- Otras empresas del grupo PROQUIFA. Solo Golocaer S.A.C. opera actualmente en Perú; las cuatro empresas del grupo México (Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V., Proveedora Quimico Farmaceutica S.A. de C.V.) no emiten proformas para clientes Perú.
- Régimen de Detracciones SUNAT (SPOT). ** Bajo análisis preliminar, los productos típicos de PROQUIFA NO están en los anexos sujetos a detracción de la R.S. 183-2004/SUNAT. La aplicabilidad final debe confirmarse con asesor contable peruano antes de habilitar el módulo productivamente. **
- Régimen de Percepciones SUNAT del IGV. ** Bajo análisis preliminar, Golocaer S.A.C. NO está designada por SUNAT como Agente de Percepción y sus productos no están en el Apéndice 1 de la Ley N° 29173. La aplicabilidad final debe confirmarse con asesor contable peruano antes de habilitar el módulo productivamente. **

---

## Reglas de Negocio

Regla 1 — Generación únicamente en pedidos Prepago sin Factura por Adelantado para clientes Perú
El sistema genera el PDF de Proforma con el diseño estandarizado para Perú únicamente cuando el pedido es en modalidad Prepago sin Factura por Adelantado y el cliente tiene Región = Perú. Los pedidos Crédito (con o sin Factura por Adelantado) y los pedidos Prepago con Factura por Adelantado no generan Proforma.

Regla 2 — Empresa emisora única Golocaer S.A.C.
La empresa emisora del documento para clientes Perú es siempre Golocaer S.A.C., única empresa del grupo PROQUIFA operando en Perú. No hay diferenciación por empresa emisora como en México. ** Pendiente confirmar con el cliente. **

Regla 3 — Foliador global con prefijo PRF compartido con Proformas México
El folio de la Proforma Perú usa el mismo foliador global lineal PQF2 que las Proformas México (un solo contador global para todo el grupo, sin segmentación por región ni por empresa), en formato MMDDAA-Consecutivo, con prefijo "PRF-" en la representación visual del documento. ** Pendiente confirmar si el prefijo se persiste también en el folio interno almacenado en base de datos. **

Regla 4 — Vigencia del documento
La Proforma calcula y muestra una fecha de vigencia en formato DD/MM/YYYY. ** La regla exacta del cálculo de la vigencia queda como duda formal del proyecto, pendiente de confirmar con el cliente. **

Regla 5 — Generación bajo demanda durante el flujo previo al envío
Al presionar "Tramitar" para un pedido Prepago sin Factura por Adelantado de cliente Perú, el sistema genera el PDF de la Proforma dinámicamente, leyendo los datos vigentes en ese momento desde las fuentes (Catálogo de Clientes, Pedido, Catálogo de Cuentas Bancarias Perú, Referencia Bancaria del Cliente), y lo muestra en previsualización. En esta etapa el PDF no se almacena en base de datos.

Regla 6 — Regeneración con datos actualizados si el usuario abandona la previsualización y reintenta
Si un usuario vio la previsualización pero abandonó el flujo sin enviar la Proforma, al volver al pedido y presionar "Tramitar" nuevamente el sistema regenera el PDF desde cero leyendo los datos vigentes en ese nuevo momento. Si entre intentos cambiaron datos fuente (razón social del cliente, dirección fiscal, precios, cuentas bancarias, etc.), la nueva versión los refleja. Esto aplica únicamente mientras la Proforma no haya sido enviada al cliente.

Regla 7 — Persistencia del PDF al recibir confirmación del envío exitoso del correo
Al confirmarse que el correo de la Proforma fue enviado exitosamente al cliente, el sistema persiste la versión final del PDF en base de datos como artefacto histórico inmutable, con los datos exactos enviados. El pendiente en Tramitar Pedido se cierra.

Regla 8 — Sin regeneración posterior al envío
Una Proforma enviada y persistida se entrega, al consultarse históricamente, desde el PDF almacenado en base de datos, sin regenerarlo desde los datos fuente actuales. El sistema no ofrece funcionalidad de reenvío.

Regla 9 — Consulta del PDF histórico desde Validar Cobro
Una Proforma enviada y persistida puede consultarse desde el módulo Validar Cobro para verificación y trazabilidad del cobro asociado, accediendo al PDF histórico.

Regla 10 — Disclaimer legal SUNAT
El documento muestra un texto fijo, equivalente bajo normativa SUNAT, que indica que es informativo previo a la emisión del Comprobante de Pago Electrónico (CPE) y carece de validez fiscal y tributaria conforme al Reglamento de Comprobantes de Pago. ** Texto propuesto: "ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DEL COMPROBANTE DE PAGO ELECTRÓNICO (CPE). CARECE DE VALIDEZ FISCAL Y TRIBUTARIA CONFORME AL REGLAMENTO DE COMPROBANTES DE PAGO Y RESOLUCIÓN DE SUPERINTENDENCIA N° 097-2012/SUNAT." Pendiente validación legal con asesor SUNAT antes de publicación productiva. **

Regla 11 — Paginación automática (comportamiento existente)
Cuando las partidas del pedido exceden el espacio disponible en una sola página, el sistema genera páginas adicionales con la misma cabecera y pie completo, mostrando la numeración "X/Y" en cada página. Este comportamiento ya existe en PQF2.

Regla 12 — Origen de los datos por sección
Los paneles del documento se arman desde las fuentes indicadas: datos de partidas (cantidad, descripción, precio unitario, importe) desde el Pedido; identificación del cliente, RUC y dirección fiscal desde el Catálogo de Clientes; moneda aplicada a los cálculos desde la moneda de facturación configurada en el Catálogo del cliente (no del pedido); Condiciones de Pago desde la configuración del cliente en el Catálogo; cuentas bancarias (Banca, Sucursal, Cuenta, CCI) desde el Catálogo de Cuentas Bancarias de Golocaer S.A.C. Perú; REF. CLIENTE de cada cuenta construida con la lógica de identificación de pagos peruana; Pedido interno, Parciales, Contacto y Lugar de entrega desde el Pedido; logo, color institucional, dirección y razón social legal generados por el sistema correspondientes a Golocaer S.A.C. Perú. ** El modelo de cuentas bancarias Perú y el modelo de Referencia Bancaria Perú no están definidos: brechas mayores documentadas en B1 y B2. **

---

## Riesgos

Riesgo 1 — Disclaimer legal SUNAT pendiente de validación
El disclaimer propuesto en el documento es una redacción aproximada basada en el marco normativo SUNAT. La redacción legal exacta debe ser validada por asesor contable o legal peruano antes de uso productivo. Un disclaimer mal redactado podría generar confusión en el cliente o exposición legal innecesaria.

Riesgo 2 — Régimen de Detracciones y Percepciones SUNAT con aplicabilidad pendiente de confirmar
Bajo análisis preliminar, los productos típicos de PROQUIFA (estándares químico-biológicos, cepas Microbiologics, sustancias controladas, columnas cromatográficas, equipos de laboratorio) no están en los anexos de Detracción de la R.S. 183-2004/SUNAT, y Golocaer S.A.C. no sería Agente de Percepción del IGV bajo la Ley N° 29173 para sus productos típicos. ** Confirmar formalmente con asesor contable peruano antes de habilitar Perú productivamente. Si algún producto cae en algún anexo de Detracción o si SUNAT designa a Golocaer S.A.C. como Agente de Percepción en el futuro, el sistema deberá adaptarse para reflejar el régimen aplicable. **

Riesgo 3 — Tipo de cambio inconsistente entre Proforma y validación de pago posterior
Si el tipo de cambio mostrado en la Proforma difiere del aplicado al recibir el pago en Validar Cobro, el cliente puede recibir documentos con montos distintos en moneda local generando confusión. La regla es la misma que en México: el tipo de cambio es el del día de generación de la Proforma.

---

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
Entonces deberá mostrar un texto que indique el carácter informativo del documento previo a la emisión del Comprobante de Pago Electrónico (CPE) bajo normativa SUNAT. ** Texto propuesto: "ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DEL COMPROBANTE DE PAGO ELECTRÓNICO (CPE). CARECE DE VALIDEZ FISCAL Y TRIBUTARIA CONFORME AL REGLAMENTO DE COMPROBANTES DE PAGO Y RESOLUCIÓN DE SUPERINTENDENCIA N° 097-2012/SUNAT." Pendiente validación legal con asesor SUNAT antes de publicación productiva. **

Criterio A3 — Título "Proforma"
Dado que el sistema renderiza la cabecera,
Cuando incluye el título del documento,
Entonces deberá mostrar el texto "Proforma". ** Pendiente confirmar si en Perú el título canónico comercial es "Proforma" o "Factura Proforma" — ambos términos se usan indistintamente en la práctica peruana. **

Criterio A4 — Folio con prefijo PRF
Dado que el sistema renderiza la cabecera,
Cuando incluye el folio del documento,
Entonces deberá mostrar el folio con formato "PRF-MMDDAA-Consecutivo". El consecutivo corresponde al foliador global lineal PQF2. ** El momento exacto en que se consume el folio (al previsualizar vs al confirmar envío) queda como duda técnica del proyecto. **

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
- "Sub-Total" con monto y moneda.
- "IGV" con tasa aplicable al pedido (18% según normativa SUNAT, salvo exoneraciones específicas que pudieran aplicar a productos puntuales — pendiente confirmar exoneraciones aplicables) y monto calculado.
- "Gran Total" con monto, suma de Sub-Total e IGV.
La moneda aplicada es la moneda de facturación del cliente desde el Catálogo (no la moneda del pedido). Para Perú las monedas típicas son PEN (Soles) y USD.

Criterio D2 — Monto del Gran Total expresado en letra
Dado que el sistema renderiza la conversión a letras del Gran Total,
Cuando incluye la leyenda monetaria,
Entonces deberá mostrar el monto en palabras según la moneda:
- Si moneda = soles peruanos: "(XXX SOLES XX/100)".
- Si moneda = dólares: "(XXX DOLARES XX/100)".
- Otras monedas: nomenclatura correspondiente.
** Pendiente confirmar la nomenclatura exacta esperada para SUNAT (algunas implementaciones usan "SOLES" otras "NUEVOS SOLES" pese a que la moneda oficial desde 2015 es solo "SOLES"). **

Criterio D3 — Tipo de Cambio (cuando aplica)
Dado que la moneda de facturación del cliente NO es soles peruanos,
Cuando el sistema renderiza la sección de pago,
Entonces deberá mostrar el tipo de cambio aplicado a la conversión. El tipo de cambio es el del día de generación. ** Pendiente confirmar si para Perú aplica el tipo de cambio SUNAT publicado (compra/venta) o un tipo de cambio interno corporativo. **

Criterio D4 — Condiciones de Pago
Dado que el sistema renderiza la sección de pago,
Cuando incluye las condiciones,
Entonces deberá mostrar las condiciones de pago aplicables al cliente (ejemplo: "PREPAGO 100%"), provenientes de la configuración del cliente en el Catálogo.

Criterio D5 — Leyenda de pago
Dado que el sistema renderiza el final de la sección de pago,
Cuando incluye la leyenda de pago,
Entonces ** pendiente definir la leyenda equivalente bajo normativa SUNAT. La normativa peruana clasifica las operaciones como Contado o Crédito sin el concepto "Pago en una sola exhibición" propio del SAT mexicano. Propuesta: omitir esta leyenda o reemplazarla con "OPERACIÓN AL CONTADO" cuando aplique. Pendiente confirmar. **

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
- RUC del cliente desde el Catálogo de Clientes (en lugar de RFC; etiqueta del campo "RUC").
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
- Contacto de entrega del pedido (si no existe, mostrar "NINGUNO"). ** Confirmar si es el contacto de entrega, contacto del cliente o contacto que realizó el pedido (misma duda que en México). **
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
Entonces deberá mostrar la razón social legal completa "Golocaer S.A.C." con su dirección legal completa en Perú. ** La dirección legal de Golocaer S.A.C. en Perú no está disponible en el sistema actual. Brecha pendiente: recopilar y capturar antes de habilitar Perú. **

Criterio H3 — Sellos de certificación y métodos de pago aceptados
Dado que el sistema renderiza el pie,
Cuando incluye certificaciones y métodos de pago aceptados,
Entonces deberá mostrar las certificaciones vigentes aplicables a Golocaer S.A.C. Perú. ** El sello NEEC (Nuevo Esquema de Empresas Certificadas) NO aplica para Perú por ser programa SAT exclusivo México. Pendiente confirmar si Golocaer Perú cuenta con certificación ISO 9001 o equivalente. Pendiente confirmar métodos de pago aceptados aplicables al mercado peruano (visa, mastercard, etc.). Brecha pendiente. **

Criterio H4 — Numeración de página
Dado que el sistema completa el documento,
Cuando incluye el contador de páginas,
Entonces deberá mostrar "X/Y" en el pie del documento, donde X es la página actual e Y es el total. Si el documento es de una sola página, se muestra "1/1".

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
Dado que un usuario presiona "Tramitar" en el módulo Tramitar Pedido para un pedido Perú,
Cuando el sistema procesa la acción,
Entonces deberá generar el PDF dinámicamente con los datos vigentes en ese momento y mostrarlo en previsualización al usuario. El PDF no se almacena en base de datos en esta etapa.

Criterio J2 — Regeneración con datos actualizados al reintentar
Dado que el usuario abandonó el flujo sin enviar la Proforma y vuelve a presionar "Tramitar",
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
Entonces el sistema no deberá ofrecer esa funcionalidad. El pendiente está cerrado y la Proforma original se conserva como registro permanente.

---

## Notas Adicionales

- Esta fila documenta el contenido y la generación del PDF de Proforma para clientes con Región Perú. La equivalente para Región México se documenta en requisito independiente.
- El requisito es un rediseño del documento de Proforma. La estructura visual específica (colores exactos, layout de bandas, tipografía, espaciados) es decisión del equipo de diseño UI; este requisito se enfoca en la información que debe contener cada sección del documento adaptada a la normativa fiscal peruana.
- Aplica exclusivamente a pedidos Prepago que NO seleccionaron Factura por Adelantado. Los pedidos Crédito (con o sin Factura por Adelantado) y los pedidos Prepago con Factura por Adelantado NO generan Proforma.
- La única empresa emisora del grupo PROQUIFA operando actualmente en Perú es Golocaer S.A.C. No hay diferenciación por empresa emisora como en México (donde son cuatro empresas: Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica).
- El PDF se genera bajo demanda en cada presión del botón "Tramitar" durante el flujo previo al envío de la Proforma. Si el usuario abandona el flujo sin enviar y vuelve a presionar "Tramitar", el PDF se regenera leyendo los datos fuente vigentes en ese nuevo momento.
- Cuando el sistema confirma que el correo de envío al cliente fue exitoso, el PDF final se persiste en base de datos como artefacto histórico inmutable. A partir de ese momento, la Proforma enviada queda como registro permanente del documento exacto que recibió el cliente. No se ofrece funcionalidad de reenvío posterior.
- El PDF histórico de la Proforma enviada se puede consultar desde el módulo Validar Cobro.
- Diferencias normativas SUNAT respecto al modelo México:
  - Impuesto: IGV 18% (Impuesto General a las Ventas) en lugar de IVA 16%.
  - Identificador fiscal del cliente: RUC en lugar de RFC.
  - Código bancario interbancario: CCI (Código de Cuenta Interbancario, 20 dígitos) en lugar de CLABE (18 dígitos).
  - Moneda local: PEN (Soles peruanos) en lugar de MXN (Pesos mexicanos).
  - Disclaimer legal: marco SUNAT (Reglamento de Comprobantes de Pago y R.S. 097-2012/SUNAT) en lugar de marco SAT (Art. 29 y 29A CFF).
  - Documento fiscal final: Comprobante de Pago Electrónico (CPE) en lugar de CFDI.
  - Régimen de Pago: SUNAT clasifica Contado/Crédito sin el concepto "Pago en una sola exhibición" del SAT.
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
Texto exacto del disclaimer validado por asesor legal peruano. Propuesta documentada: "ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DEL COMPROBANTE DE PAGO ELECTRÓNICO (CPE). CARECE DE VALIDEZ FISCAL Y TRIBUTARIA CONFORME AL REGLAMENTO DE COMPROBANTES DE PAGO Y RESOLUCIÓN DE SUPERINTENDENCIA N° 097-2012/SUNAT."

B5 — Régimen de Detracciones y Percepciones
Confirmación formal por asesor contable peruano de que los productos típicos de PROQUIFA NO están sujetos a Detracción (R.S. 183-2004/SUNAT) y de que Golocaer S.A.C. NO sería Agente de Percepción para sus productos (Ley N° 29173).

B7 — Certificaciones aplicables a Golocaer S.A.C. Perú
ISO 9001 o equivalente, métodos de pago aceptados (medios peruanos), y cualquier otra certificación de calidad vigente en el mercado peruano.

B8 — Catálogos farmacéuticos aplicables a Perú
Lista definitiva de logos del pie inferior aplicables a Golocaer S.A.C. Perú. Confirmados como aplicables: USP, EDQM, Microbiologics. NO aplica: FEUM. Otros logos (APACOR, CHATA Biosystems, Pharmaffiliates) pendientes de confirmar.

B10 — Título canónico del documento en Perú
Confirmar si el título es "Proforma" o "Factura Proforma" (ambos términos se usan indistintamente en la práctica comercial peruana).

B11 — Nomenclatura del monto en letra para soles peruanos
Confirmar si la nomenclatura aceptada es "SOLES" (oficial desde 2015) o "NUEVOS SOLES" (denominación previa que aún aparece en algunas implementaciones).

B12 — Tipo de cambio aplicado en Perú
Confirmar si para Perú aplica el tipo de cambio SUNAT publicado (compra/venta) o un tipo de cambio interno corporativo.

Mientras estas brechas no se resuelvan, el cliente Perú no puede recibir Proforma productiva con el formato adaptado. Se recomienda al cliente programar sesión específica para resolución integral del modelo Perú.


---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-06-10 | OBS-032 | Revisado en contexto de OBS-032 (pendientes Perú sin timbrado). Este requisito es sobre el PDF de Proforma Perú y no genera pendientes de Factura por Adelantado directamente; la corrección OBS-032 se aplica al módulo FU-018 (listado FxA). Sin cambios de contenido en este archivo. |
