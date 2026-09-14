# R16A-RE-FU-021 — Diseño y generación de Documentos: Factura México

| Campo | Valor |
|-------|-------|
| **ID** | R16A-RE-FU-021 |
| **Título** | Diseño y generación de Documentos: Factura México |
| **Módulo / Épica** | Factura por Adelantado |
| **Historia de Usuario** | Yo como **Gestor de Cobranza** (función operativa: Analista de Cuentas por Cobrar), quiero que el sistema genere automáticamente el PDF de la Factura CFDI 4.0 con el branding de la empresa emisora al timbrarse exitosamente, para entregar al cliente la representación impresa del comprobante fiscal y conservarla como artefacto inmutable. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.1M-RE-FU-007 |

---

## Requisito Funcional

El sistema debe generar el PDF (representación impresa) de la Factura CFDI 4.0 al timbrarse exitosamente ante el PAC, para clientes con Región México, con un diseño estandarizado cuyo branding varía según la empresa emisora del pedido (Golocaer, Mungen, Proquifa o Proveedora Quimico Farmaceutica). El PDF se almacena al timbrarse como artefacto fiscal inmutable y no se regenera al consultarse posteriormente. Esta funcionalidad aplica tanto a las Facturas generadas desde el módulo Factura por Adelantado como a las generadas desde Validar Cobro.

---

## Alcance

### Aplica a

- Generación del PDF de la Factura CFDI 4.0 al timbrarse exitosamente ante el PAC, para clientes con Región México.
- Facturas originadas en el módulo Factura por Adelantado (pedidos Crédito sin controlados y Prepago sin controlados).
- Facturas originadas en Validar Cobro (cobro recibido aplicado a Proforma de Prepago sin controlados).
- Cuatro empresas emisoras del grupo PROQUIFA México con branding propio: Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V. y Proveedora Quimico Farmaceutica S.A. de C.V.
- Branding diferenciado por empresa emisora (logo, colores y certificaciones).
- Foliador independiente por empresa emisora (consecutivo numérico, varchar 6), serie "A2" distinta de la utilizada por el sistema anterior (Legacy), para evitar colisión de folios entre ambos.
- Paginación automática cuando las partidas exceden el espacio de una página (comportamiento ya existente del sistema).
- Almacenamiento del PDF (junto con el XML timbrado) al timbrarse exitosamente; no se regenera al consultarse.
- Factura Anticipo para pedidos Prepago con productos controlados (Sección K), reutilizando la estructura del documento normal con las particularidades de concepto único, ausencia de comprobantes relacionados, método y forma de pago, y herencia monetaria descritas en esa sección.

### No aplica a

- Pedidos para clientes con Región Perú: se cancela por completo la facturación de Perú en R16 (Decisión "Quitar Perú" 2026-07-17 / DUDA-052); no hay ningún desarrollo de facturación para Perú en este release, y el requisito independiente de Factura Perú (R16A-RE-FU-022) no se ejecuta.
- Generación del XML del CFDI (estructura técnica): es responsabilidad del PAC y se rige por el estándar CFDI 4.0 del SAT.
- Lógica de timbrado, integración con el PAC, manejo de errores: vive en los requisitos de los módulos que generan facturas.
- Cancelación de la Factura timbrada: se documenta en el módulo Notas de Crédito.
- Envío de la Factura al cliente por correo electrónico: vive en los requisitos de los módulos que generan facturas.
- Generación del Complemento de Pago: se documenta en requisito independiente del módulo Validar Cobro.

---

## Reglas de Negocio

Regla 1 — Generación al timbrado exitoso
Cuando el PAC retorna confirmación de timbrado exitoso, el sistema genera el PDF de la Factura y lo almacena junto con el XML timbrado como artefacto fiscal inmutable.

Regla 2 — Diferenciación por empresa emisora
La Factura se diferencia según la empresa emisora del pedido (una de las cuatro del grupo PROQUIFA México) en logo, colores, certificaciones, lugar de expedición, RFC, dirección y razón social legal correspondiente.

Regla 3 — Folio independiente por empresa emisora, serie "A2"
El folio de la Factura usa un contador consecutivo numérico independiente por empresa emisora, tomado del catálogo de consecutivos, de tipo varchar(6). La serie es "A2", distinta de la que usa el sistema anterior (Legacy), para evitar colisión de folios entre ambos. La Factura Anticipo (Sección K) toma su folio del mismo consecutivo que la Factura normal y se registra en el sistema anterior tanto como factura normal como en su propia tabla; su serie queda pendiente de confirmar.

Regla 4 — Versión CFDI 4.0
El PDF refleja la versión 4.0 del estándar CFDI conforme normativa SAT vigente. El disclaimer del documento indica: "Representación impresa de un CFDI 4.0".

Regla 5 — Método y Forma de Pago según el escenario de emisión
El Método de Pago y la Forma de Pago del CFDI se derivan del escenario que origina la Factura, con cuatro casos: (1) Factura por Adelantado de pedido Crédito y (2) Factura por Adelantado de pedido Prepago: al no conocerse aún la forma real de pago al momento de timbrar, el Método de Pago es "PPD - Pago en parcialidades o diferido" y la Forma de Pago es "99 - Por definir" (regla ya establecida en R16A-RE-FU-019, Regla 5); (3) Factura generada desde Validar Cobro por cobro recibido aplicado a Proforma de Prepago y (4) Factura Anticipo (Sección K): en ambos casos el pago ya fue recibido y validado al momento de timbrar, por lo que el Método de Pago es "PUE - Pago en una sola exhibición" y la Forma de Pago es la forma real del pago recibido. La normativa SAT obliga a que toda factura con Método de Pago diferido (PPD) use la Forma de Pago "99 - Por definir", y a que toda factura en una sola exhibición (PUE) use la forma de pago real.

Regla 6 — Herencia de moneda y tipo de cambio desde la Proforma
Cuando la Factura se origina a partir de una Proforma existente (Factura por Adelantado o Factura generada desde Validar Cobro sobre una Proforma de Prepago), hereda la moneda y el tipo de cambio de esa Proforma; no se recalculan al timbrar. Este tipo de cambio es distinto del tipo de cambio que valora un cobro recibido en Validar Cobro, correspondiente a un momento distinto del proceso.

Regla 7 — Inmutabilidad de la Factura timbrada
Una vez timbrada exitosamente, el PDF de la Factura se entrega siempre desde el almacenado sin posibilidad de modificación ni regeneración. La Factura timbrada es artefacto fiscal inmutable; cualquier corrección posterior requiere cancelación SAT o emisión de Nota de Crédito.

Regla 8 — Paginación automática (comportamiento existente)
Cuando las partidas del pedido exceden el espacio disponible en una sola página, el sistema genera páginas adicionales, mostrando la numeración "X de Y" en cada página. Este comportamiento ya existe en PQF2.

Regla 9 — Origen de los datos por sección
El PDF se arma desde las fuentes indicadas: datos del Receptor (Razón Social, RFC, Uso CFDI, Código Postal, Régimen Fiscal) del CFDI timbrado (reflejo del Catálogo de Clientes al momento del timbrado); datos del Emisor (Razón Social, RFC, Lugar de Expedición, Régimen Fiscal, Dirección) del CFDI timbrado; datos del CFDI (Serie, Folio, Versión, Folio Fiscal UUID, Fecha y Hora de Emisión, Fecha y Hora de Certificación, Método de Pago, Forma de Pago, Condiciones de Pago, Tipo de Comprobante, Moneda, Tipo de Cambio) del CFDI timbrado; elementos técnicos SAT (Folio Fiscal UUID, Números de Serie de Certificados, Sellos Digitales, Cadena Original, Código QR) del TimbreFiscalDigital del XML generado por el PAC; branding (logo, colores, certificaciones) generado por el sistema según la empresa emisora; y, por cada partida, la Clave SAT del Producto/Servicio, la Clave Unidad SAT y la información de impuesto (BASE, IMPUESTO, TIPO DE FACTOR, TASA/CUOTA, IMPORTE) desde la configuración fiscal de la Familia del producto, y la descripción, el Nº ID interno, la Unidad de Medida, la Cantidad, el Valor Unitario y el Importe desde el Pedido. La referencia bancaria del cliente se toma de la referencia vigente configurada en su Catálogo, conforme al requisito de Referencia de Pago; esta fila conserva únicamente su presentación en el documento.

Regla 10 — Referencia del Pedido vs Folio de Pedido Interno (PI)
La "Referencia" (orden de compra del cliente) y el "Folio de PI" son datos distintos y no intercambiables: la Referencia es la capturada en Buzones (orden de compra del cliente); el Folio de PI es el generado por PQF2 en la Confirmación de Pedido.

---

## Riesgos

Riesgo 1 — Indisponibilidad del PAC al timbrar
Si el PAC está caído o no responde, no se puede timbrar la Factura ni generar su PDF.

Riesgo 2 — Pedimento no aplicable y deducibilidad del comprobante
La Factura se emite antes del surtido del pedido, por lo que el número de pedimento se consigna como no aplicable (ver Criterio C5). Esto supone un riesgo para la deducibilidad del comprobante por parte del cliente receptor, cuando la normativa fiscal aplicable a su operación exija ese dato.

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
- Serie ("A2", distinta de la utilizada por el sistema anterior, para evitar colisión de folios).
- Folio (consecutivo numérico por empresa emisora, varchar 6).
- Versión (4.0).
- Folio Fiscal (UUID de 36 caracteres asignado por el SAT al timbrar).

Criterio C2 — Fechas y horas del CFDI
Dado que el sistema renderiza el documento,
Cuando incluye las fechas del comprobante,
Entonces deberá mostrar:
- Fecha y Hora de Emisión (momento de emisión de la factura por el sistema).
- Fecha y Hora de Certificación (momento del timbrado por el PAC).

Criterio C3 — Datos fiscales del CFDI
Dado que el sistema renderiza los datos fiscales del CFDI,
Cuando incluye la información,
Entonces deberá mostrar:
- Método de Pago y Forma de Pago, derivados del escenario que origina la Factura conforme a la Regla 5 (PPD / "99 - Por definir" para Factura por Adelantado de pedido Crédito o Prepago; PUE / forma real del pago para Factura generada desde Validar Cobro o Factura Anticipo).
- Condiciones de Pago (texto descriptivo: PREPAGO 100%, 30 DIAS, 60 DIAS, 90 DIAS, etc.).
- Tipo de Comprobante (I - Ingreso).
- Régimen Fiscal del emisor (601 - General de Ley Personas Morales en operación actual).
- Moneda y Tipo de Cambio, heredados de la Proforma de origen cuando la Factura proviene de una (Regla 6); si no proviene de una Proforma, corresponden a la moneda de facturación y al tipo de cambio del día de la generación.

Criterio C4 — Atributo de Exportación
Dado que el sistema renderiza los datos del CFDI,
Cuando incluye el atributo de Exportación (campo obligatorio del CFDI 4.0),
Entonces deberá mostrar el valor "01 - No aplica" del catálogo c_Exportacion del SAT, dado que las operaciones de exportación al extranjero quedan fuera del alcance de este release (como se observa en la factura real de Mungen con "Exportación: No Aplica").

Criterio C5 — Pedimento no aplicable
Dado que la Factura se emite antes del surtido del pedido,
Cuando el sistema arma el comprobante,
Entonces deberá consignar el número de pedimento como no aplicable (ver Criterio E1); esto supone un riesgo para la deducibilidad del comprobante por parte del cliente receptor (ver Riesgo 2), del cual el usuario emisor no es responsable.

═══════════════════════════════════════════════════════════════
SECCIÓN D — REFERENCIAS BANCARIAS
═══════════════════════════════════════════════════════════════

Criterio D1 — Cuentas bancarias de la empresa emisora
Dado que el sistema renderiza las cuentas bancarias,
Cuando arma el contenido,
Entonces deberá mostrar las dos cuentas activas más recientes de la empresa emisora que factura, con los datos que correspondan a cada cuenta: Banco, Número de Cuenta, Moneda, Referencia del Cliente, CLABE, Sucursal.

Criterio D2 — Referencia bancaria del cliente
Dado que el sistema renderiza el campo Referencia de cada cuenta,
Cuando construye el valor,
Entonces deberá tomarlo de la referencia vigente configurada en el Catálogo del cliente, conforme al requisito de Referencia de Pago; esta fila del documento conserva únicamente su presentación.

Criterio D3 — Referencia del pedido del cliente
Dado que el sistema renderiza la sección,
Cuando incluye el dato del pedido,
Entonces deberá mostrar el número de orden de compra del cliente o referencia equivalente proveniente del Pedido. ~~Confirmar diferencia con Folio de Pedido Interno~~ **[2026-08-21 · DUDA-060 · Resuelta]** Son datos distintos: esta "Referencia" es la capturada en Buzones (orden de compra del cliente), diferente del Folio de Pedido Interno (PI), generado por PQF2 en la Confirmación de Pedido (ver Criterio G3). No son intercambiables.

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
Entonces deberá tomar la Clave SAT del Producto/Servicio, la Clave Unidad SAT y el desglose de impuestos (BASE, IMPUESTO, TIPO DE FACTOR, TASA/CUOTA, IMPORTE) de la configuración fiscal de la Familia del producto; y la descripción, el Nº ID interno, la Unidad de Medida, la Cantidad, el Valor Unitario y el Importe, del Pedido (ver Regla 9).

═══════════════════════════════════════════════════════════════
SECCIÓN F — TOTALES E IMPUESTOS
═══════════════════════════════════════════════════════════════

Criterio F1 — Retenciones
Dado que el sistema renderiza los totales,
Cuando incluye la sección de retenciones,
Entonces deberá mostrar las retenciones aplicables al CFDI (si no hay retenciones, la sección se muestra sin contenido).

Criterio F2 — Traslados (impuestos trasladados)
Dado que el sistema renderiza los impuestos trasladados,
Cuando incluye los datos,
Entonces deberá mostrar el desglose de los impuestos trasladados aplicables según el Perfil Fiscal de cada partida (IVA tasa general, tasa cero o exento) con: IMPUESTO, TIPO FACTOR, TASA/CUOTA, IMPORTE.

Criterio F3 — Total expresado en letra
Dado que el sistema renderiza el total,
Cuando incluye la conversión a letras,
Entonces deberá mostrar el monto en palabras según la moneda (ejemplo: "TREINTA Y UN MIL QUINIENTOS SETENTA DOLARES 00/100").

Criterio F4 — Moneda y Tipo de Cambio
Dado que el sistema renderiza los datos monetarios,
Cuando incluye la información,
Entonces deberá mostrar la Moneda (con nombre completo) y el Tipo de Cambio, heredados de la Proforma de origen cuando la Factura proviene de una (Regla 6); si no proviene de una Proforma, el Tipo de Cambio corresponde al día de la generación. Este Tipo de Cambio es distinto del que valora un cobro recibido en Validar Cobro.

Criterio F5 — Subtotal, Impuestos Federales y Total
Dado que el sistema renderiza el bloque final,
Cuando incluye los montos,
Entonces deberá mostrar:
- Subtotal (suma de los importes de las partidas).
- Impuestos Federales (suma de los traslados aplicables según el Perfil Fiscal de cada partida: IVA, tasa cero o exento).
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
Entonces deberá tomarlos de la información entregada por el PAC al timbrar el CFDI. El sistema no calcula ni genera estos elementos; únicamente los recibe y los muestra en el documento.

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
Dado que el sistema renderiza el documento,
Cuando incluye el disclaimer,
Entonces deberá mostrar el texto fijo: "Representación impresa de un CFDI 4.0".

Criterio H2 — Certificaciones y métodos de pago aceptados
Dado que el sistema renderiza el documento,
Cuando incluye certificaciones y métodos de pago,
Entonces deberá mostrar las certificaciones vigentes (ISO 9001:2015 y OEA) y los métodos de pago aceptados aplicables.

Criterio H3 — Logos de catálogos farmacéuticos y de marcas
Dado que el sistema renderiza el documento,
Cuando arma el contenido,
Entonces NO deberá incluir logos de catálogos farmacéuticos ni de marcas (decisión confirmada por el cliente).

Criterio H4 — Numeración de página
Dado que el sistema completa el documento,
Cuando incluye el contador de páginas,
Entonces deberá mostrar "X de Y" en cada página.

═══════════════════════════════════════════════════════════════
SECCIÓN I — PAGINACIÓN AUTOMÁTICA
═══════════════════════════════════════════════════════════════

Criterio I1 — Múltiples páginas cuando las partidas exceden una página
Dado que el pedido tiene partidas que exceden el espacio disponible en una sola página,
Cuando el sistema renderiza el documento,
Entonces deberá generar páginas adicionales manteniendo el mismo formato del documento. La numeración se actualiza (1 de 5, 2 de 5, ..., 5 de 5). Este comportamiento ya existe en PQF2.

═══════════════════════════════════════════════════════════════
SECCIÓN J — ALMACENAMIENTO
═══════════════════════════════════════════════════════════════

Criterio J1 — Almacenamiento al timbrado exitoso
Dado que el PAC confirma el timbrado exitoso de la Factura,
Cuando el sistema recibe la respuesta del PAC,
Entonces deberá almacenar el PDF y el XML timbrado como artefacto fiscal inmutable.

Criterio J2 — Sin regeneración posterior
Dado que la Factura fue timbrada y almacenada,
Cuando un usuario consulta el PDF en cualquier momento posterior,
Entonces el sistema deberá entregar el PDF almacenado sin regenerarlo. La Factura es artefacto fiscal inmutable. Si los datos fuente del cliente o del emisor cambian después del timbrado, la Factura histórica conserva los datos originales sin modificación.

═══════════════════════════════════════════════════════════════
SECCIÓN K — VARIANTE: FACTURA ANTICIPO (PRODUCTOS CONTROLADOS)
═══════════════════════════════════════════════════════════════

Contexto: La Factura Anticipo se emite para pedidos Prepago con productos controlados (Mundial, Nacional u Origen), generada desde Validar Cobro por el cobro del anticipo recibido. PQF2 genera únicamente el CFDI de Ingreso de primera emisión por el monto del anticipo; no genera la factura final de cierre con aplicación de anticipo (relación 07).

Criterio K1 — Estructura base reutilizada
Dado que el sistema genera el PDF de una Factura Anticipo,
Cuando arma el documento,
Entonces deberá reutilizar la misma estructura del PDF de la Factura normal (Secciones A–J): datos del emisor, receptor, datos del CFDI, referencias bancarias, partidas, totales e impuestos, elementos técnicos SAT, información institucional, paginación y almacenamiento.

Criterio K2 — Concepto único de la partida: anticipo
Dado que el sistema renderiza la sección de partidas de una Factura Anticipo,
Cuando arma el concepto,
Entonces deberá mostrar una única partida correspondiente al anticipo recibido (no el detalle de productos del pedido), con su propia Clave SAT del Producto/Servicio y Clave Unidad SAT para anticipos, descripción, importe y desglose de impuestos conforme al Perfil Fiscal aplicable.

Criterio K3 — Ausencia de comprobantes relacionados
Dado que la Factura Anticipo es un CFDI de primera emisión,
Cuando el sistema arma el documento,
Entonces no deberá incluir número de pedimento (no aplicable, igual que la Factura normal, ver Criterio C5) ni sección de CFDI Relacionados (tipo de relación 07), dado que PQF2 no genera la factura final de cierre.

Criterio K4 — Método y Forma de Pago, y herencia monetaria
Dado que el sistema renderiza los datos fiscales del CFDI de una Factura Anticipo,
Cuando incluye el Método de Pago, la Forma de Pago, la Moneda y el Tipo de Cambio,
Entonces deberá mostrar Método de Pago "PUE - Pago en una sola exhibición" y la Forma de Pago real del pago recibido (Regla 5, caso 4); y la Moneda y el Tipo de Cambio heredados de la Proforma de origen (Regla 6).

---

## Notas Adicionales

- Esta fila documenta el contenido y la representación impresa (PDF) de la Factura CFDI 4.0 para clientes con Región México; Perú queda fuera de alcance de este release (ver Alcance).
- El requisito es un rediseño del documento de Factura. La estructura visual específica (colores exactos, layout, tipografía, espaciados) es decisión del equipo de diseño UI; este requisito se enfoca en la información que debe contener cada sección del documento.
- El diseño se valida contra cuatro CFDIs reales recibidos del cliente (folios 2374 Mungen, 7156 Golocaer, 20913 Proquifa, 143103 Proveedora Quimico Farmaceutica).
- El PAC utilizado por PROQUIFA es TurboPac (Quadrum Tecnologías).
- La Factura Anticipo (Sección K) se documenta dentro de este mismo requisito; no requiere requisito independiente.

---

## Cambios

| # | Fecha | Referencia | Descripción del cambio |
|---|-------|------------|------------------------|
| 1 | 2026-09-11 | DUDA-047/059/060 | Historia de Usuario: rol Gestor de Cobranza / función Analista de Cuentas por Cobrar. Regla 9 y Criterios D2, E1, E2: se cierra el origen de los datos de partidas (Familia del producto / Pedido) y de la referencia bancaria del cliente (Catálogo del cliente). Reglas 1, 7 y Criterios J1, J2: se cierra la decisión de almacenamiento del PDF (no persistencia regenerable). Criterio D1: dos cuentas activas más recientes de la empresa emisora. Criterio H2: certificaciones ISO 9001:2015 y OEA (se retira NEEC). Criterio H3: sin logos de catálogos farmacéuticos ni de marcas. |
| 2 | 2026-09-11 | Turno 4 | Regla 3 y Criterio C1: se define la serie "A2" para la Factura, distinta del sistema anterior. Regla 5 y Criterio C3: se incorpora el Método y Forma de Pago según el escenario de emisión (cuatro casos). Regla 6 y Criterios C3, F4: se incorpora la herencia de moneda y tipo de cambio desde la Proforma. Sección K: se sustituye el esbozo tentativo por la definición conforme al procedimiento de anticipos (estructura reutilizada, concepto único, ausencia de comprobantes relacionados, método/forma de pago y herencia monetaria). |
| 3 | 2026-09-11 | Turno 4 | Riesgo 2 y Criterio C5 (nuevo): se documenta el pedimento no aplicable por emitirse la Factura antes del surtido, y el riesgo de deducibilidad para el cliente receptor. Riesgo 1: se retira terminología técnica de la respuesta del PAC. Criterios F2 y F5: se generaliza el desglose de impuestos trasladados (IVA, tasa cero o exento) conforme al Perfil Fiscal. Criterios C2, F1, H1, H4, I1 y Sección G: se retiran referencias a pie/cabecera y terminología técnica de implementación. Criterio C4: se precisa el atributo de Exportación como "No aplica" dado que las operaciones al extranjero quedan fuera de alcance. Notas Adicionales reescritas, retirando duplicados de reglas/criterios, pendientes ya resueltos y notas de Perú (fuera de alcance). |
