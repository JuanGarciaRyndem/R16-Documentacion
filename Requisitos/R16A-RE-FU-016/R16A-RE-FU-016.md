# Diseño y generación de Documentos: Proforma México

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-016 |
| **Nombre** | Diseño y generación de Documentos: Proforma México |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-007 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero que el sistema genere automáticamente el PDF de la Proforma con el branding de la empresa emisora del pedido al tramitar un pedido Prepago para clientes de México, para entregar al cliente un documento estandarizado y correcto que respalde el cobro por adelantado.

---

## Requisito

El sistema debe generar un PDF de Proforma al tramitar un pedido Prepago sin Factura por Adelantado para clientes con Región México, con un diseño estandarizado cuyo branding varía según la empresa emisora del pedido (Golocaer, Mungen, Proquifa o Proveedora Quimico Farmaceutica, las cuatro empresas del grupo PROQUIFA México en operación). El PDF se reconstruye dinámicamente en cada consulta tomando los valores aplicados al documento al momento de su generación inicial.

---

## Alcance

### Aplica a

- Generación del PDF de Proforma al tramitar un pedido en modalidad Prepago sin factura por adelantado para clientes con Región México.
- Cuatro empresas emisoras del grupo PROQUIFA México con branding propio: Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V. y Proveedora Quimico Farmaceutica S.A. de C.V.
- Branding diferenciado por empresa emisora.
- Generación bajo demanda del PDF durante el flujo previo al envío de la Proforma al cliente (cada vez que el usuario presiona "Tramitar" se previsualiza el PDF con datos vigentes).
- Persistencia del PDF en base de datos al recibir confirmación de envío exitoso del correo al cliente.
- Acceso al PDF histórico desde el módulo Validar Cobro una vez la Proforma fue enviada.
- Foliador global lineal PQF2 con prefijo PRF en la representación visual del documento.
- Paginación automática cuando las partidas exceden el espacio de una página (comportamiento ya existente del sistema).
- Aplicación de catálogos fiscales SAT mexicanos (RFC, IVA, CLABE, certificación NEEC, Art. 29 y 29A CFF).

### No aplica a

- Pedidos Crédito sin Factura por Adelantado ni Crédito/Prepago con Factura por Adelantado.
- Pedidos para clientes con Región Perú. Esa funcionalidad se documenta en requisito independiente.

---

## Reglas de Negocio

**Regla 1 — Generación únicamente en pedidos Prepago sin Factura por Adelantado para clientes México**
El sistema genera el PDF de Proforma únicamente cuando el pedido es en modalidad Prepago sin Factura por Adelantado y el cliente tiene Región = México. Los pedidos Crédito (con o sin Factura por Adelantado) y los pedidos Prepago con Factura por Adelantado no generan Proforma.

**Regla 2 — Diferenciación por empresa emisora**
La Proforma se diferencia según la empresa emisora del pedido (una de las cuatro del grupo PROQUIFA México) en logo, color institucional, dirección y razón social legal correspondiente.

**Regla 3 — Foliador global con prefijo PRF**
El folio de la Proforma usa el foliador global lineal PQF2 para Proformas (un solo contador sin segmentación por empresa) en formato MMDDAA-Consecutivo, con prefijo "PRF-" en la representación visual del documento (ejemplo: "PRF-031826-691").

> ** Pendiente confirmar si el prefijo se persiste también en el folio interno almacenado en base de datos. **

**Regla 4 — Vigencia del documento**
La Proforma calcula y muestra una fecha de vigencia en formato DD/MM/YYYY.

> ** La regla exacta del cálculo de la vigencia queda como duda formal del proyecto, pendiente de confirmar con el cliente. **

**Regla 5 — Generación bajo demanda durante el flujo previo al envío**
Al presionar "Tramitar" para un pedido Prepago sin Factura por Adelantado de cliente México, el sistema genera el PDF de la Proforma dinámicamente, leyendo los datos vigentes en ese momento desde las fuentes (Catálogo de Clientes, Pedido, Catálogo de Cuentas Bancarias, Tabla de Empresas, Referencia Bancaria del Cliente), y lo muestra en previsualización. En esta etapa el PDF no se almacena en base de datos.

**Regla 6 — Regeneración con datos actualizados si el usuario abandona la previsualización y reintenta**
Si un usuario vio la previsualización pero abandonó el flujo sin enviar la Proforma, al volver al pedido y presionar "Tramitar" nuevamente el sistema regenera el PDF desde cero leyendo los datos vigentes en ese nuevo momento. Si entre intentos cambiaron datos fuente (razón social del cliente, dirección fiscal, precios, cuentas bancarias, etc.), la nueva versión los refleja. Esto aplica únicamente mientras la Proforma no haya sido enviada al cliente.

**Regla 7 — Persistencia del PDF al recibir confirmación del envío exitoso del correo**
Al confirmarse que el correo de la Proforma fue enviado exitosamente al cliente, el sistema persiste la versión final del PDF en base de datos como artefacto histórico inmutable, con los datos exactos enviados. El pendiente en Tramitar Pedido se cierra y la Proforma enviada queda como registro permanente del documento exacto que recibió el cliente.

**Regla 8 — Sin regeneración posterior al envío**
Una Proforma enviada y persistida se entrega, al consultarse históricamente, desde el PDF almacenado en base de datos, sin regenerarlo desde los datos fuente actuales. El sistema no ofrece funcionalidad de reenvío. Si los datos fuente cambian después del envío, la Proforma histórica conserva los datos originales.

**Regla 9 — Consulta del PDF histórico desde Validar Cobro**
Una Proforma enviada y persistida puede consultarse desde el módulo Validar Cobro para verificación y trazabilidad del cobro asociado, accediendo al PDF histórico.

**Regla 10 — Disclaimer legal SAT obligatorio**
El documento muestra el texto fijo: "ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DE UN CFDI. CARECE DE VALIDEZ FISCAL SEGÚN ART.29 Y 29A CFF".

**Regla 11 — Paginación automática (comportamiento existente)**
Cuando las partidas del pedido exceden el espacio disponible en una sola página, el sistema genera páginas adicionales con la misma cabecera y pie completo, mostrando la numeración "X/Y" en cada página. Este comportamiento ya existe en PQF2.

**Regla 12 — Origen de los datos por sección**
Los paneles del documento se arman desde las fuentes indicadas: datos de partidas (cantidad, descripción, precio unitario, importe) desde el Pedido; identificación del cliente, RFC y dirección fiscal desde el Catálogo de Clientes; moneda aplicada a los cálculos desde la moneda de facturación configurada en el Catálogo del cliente (no del pedido); Condiciones de Pago desde la configuración del cliente en el Catálogo (sección Cobros); cuentas bancarias (Banca, Sucursal, Cuenta, CLABE) desde el Catálogo de Cuentas Bancarias del grupo PROQUIFA México; REF. CLIENTE de cada cuenta construida dinámicamente con la lógica del Código Validador; Pedido interno, Parciales, Contacto y Lugar de entrega desde el Pedido; logo, color institucional, dirección y razón social legal generados por el sistema.

---

## Riesgos

**Riesgo 1 — Consumo prematuro del folio en la previsualización**
Si el sistema reserva un folio del foliador global PQF2 al generar el PDF en la previsualización pero el usuario abandona sin enviar, ese folio quedaría huérfano generando huecos en la numeración consecutiva. La numeración consecutiva sin huecos puede ser un requisito fiscal o de auditoría.

> ** Pendiente decisión técnica: definir si el folio se reserva al generar el PDF de previsualización (consume folio aunque no se envíe) o exclusivamente al confirmar el envío exitoso (sin huecos). **

**Riesgo 2 — Tipo de cambio inconsistente entre Proforma y validación de pago posterior**
Si el tipo de cambio mostrado en la Proforma difiere del aplicado al recibir el pago en Validar Cobro, el cliente puede recibir documentos con montos distintos en moneda local generando confusión.

---

## Criterios de Aceptación

### Sección A — Cabecera del documento

**Criterio A1 — Logo de la empresa emisora**
- **Dado** que el sistema renderiza la cabecera del documento,
- **Cuando** incluye el logo,
- **Entonces** deberá mostrar el logo correspondiente a la empresa emisora del pedido.

**Criterio A2 — Disclaimer legal SAT**
- **Dado** que el sistema renderiza la cabecera,
- **Cuando** incluye el disclaimer legal,
- **Entonces** deberá mostrar el texto fijo: "ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DE UN CFDI. CARECE DE VALIDEZ FISCAL SEGÚN ART.29 Y 29A CFF".

**Criterio A3 — Título "Proforma"**
- **Dado** que el sistema renderiza la cabecera,
- **Cuando** incluye el título del documento,
- **Entonces** deberá mostrar el texto "Proforma".

**Criterio A4 — Folio con prefijo PRF**
- **Dado** que el sistema renderiza la cabecera,
- **Cuando** incluye el folio del documento,
- **Entonces** deberá mostrar el folio con formato "PRF-MMDDAA-Consecutivo" (ejemplo: "PRF-031826-691"). El consecutivo corresponde al foliador global lineal PQF2.

> ** El momento exacto en que se consume el folio (al previsualizar vs al confirmar envío) queda como duda técnica del proyecto. **

**Criterio A5 — Vigencia del documento**
- **Dado** que el sistema renderiza la cabecera,
- **Cuando** incluye el campo Vigencia,
- **Entonces** deberá mostrar la fecha de vigencia en formato DD/MM/YYYY.

> ** Regla exacta del cálculo pendiente confirmar. **

### Sección B — Identificación del cliente

**Criterio B1 — Identificación del cliente**
- **Dado** que el sistema renderiza la sección Cliente,
- **Cuando** incluye el identificador del cliente,
- **Entonces** deberá mostrar el Alias del cliente desde el Catálogo de Clientes.

> ** Pendiente confirmar si el dato fuente correcto es Alias o Razón Social. **

### Sección C — Tabla de partidas

**Criterio C1 — Datos de la tabla de partidas**
- **Dado** que el sistema renderiza la tabla de partidas,
- **Cuando** incluye los datos por cada partida,
- **Entonces** deberá mostrar: número consecutivo, cantidad, descripción (catálogo + descripción + marca), precio unitario con moneda, e importe calculado (cantidad × precio). Todos los datos provienen del Pedido.

### Sección D — Datos de pago

**Criterio D1 — Sub-Total, IVA y Gran Total**
- **Dado** que el sistema incluye los cálculos fiscales,
- **Cuando** renderiza las líneas de monto,
- **Entonces** deberá mostrar: "Sub-Total" con monto y moneda; "IVA" con tasa aplicable al pedido (0%, 16%, etc.) y monto calculado; "Gran Total" con monto, suma de Sub-Total e IVA. La moneda aplicada es la moneda de facturación del cliente desde el Catálogo (no la moneda del pedido).

**Criterio D2 — Monto del Gran Total expresado en letra**
- **Dado** que el sistema renderiza la conversión a letras del Gran Total,
- **Cuando** incluye la leyenda monetaria,
- **Entonces** deberá mostrar el monto en palabras según la moneda: si moneda = pesos mexicanos: "(XXX PESOS XX/100 M.N.)"; si moneda = dólares: "(XXX DOLARES XX/100)"; otras monedas: nomenclatura correspondiente.

**Criterio D3 — Tipo de Cambio (cuando aplica)**
- **Dado** que la moneda de facturación del cliente NO es pesos mexicanos,
- **Cuando** el sistema renderiza la sección de pago,
- **Entonces** deberá mostrar el tipo de cambio aplicado a la conversión. El tipo de cambio es el del día de generación.

**Criterio D4 — Condiciones de Pago**
- **Dado** que el sistema renderiza la sección de pago,
- **Cuando** incluye las condiciones,
- **Entonces** deberá mostrar las condiciones de pago aplicables al cliente (ejemplo: "PREPAGO 100%"), provenientes de la configuración del cliente en el Catálogo.

**Criterio D5 — Leyenda "Pago en una sola exhibición"**
- **Dado** que el sistema renderiza el final de la sección de pago,
- **Cuando** incluye la leyenda de exhibición,
- **Entonces** deberá mostrar el texto "PAGO EN UNA SOLA EXHIBICIÓN" como leyenda fiscal obligatoria SAT.

> ** Confirmar si siempre es PUE. **

### Sección E — Datos bancarios

**Criterio E1 — Doble cuenta visible siempre**
- **Dado** que el sistema renderiza la sección de datos bancarios,
- **Cuando** arma el contenido,
- **Entonces** deberá mostrar SIEMPRE las dos cuentas (una en M.N. y una en DLS) del grupo PROQUIFA México, independientemente de la moneda del pedido. Los campos por cuenta son: Moneda, Banca, Sucursal, Cuenta, CLABE y REF. CLIENTE.

> ** Confirmar si siempre se muestran M.N y DLS o puede variar, ej. EUR. **

**Criterio E2 — Referencia bancaria del cliente (REF. CLIENTE)**
- **Dado** que el sistema renderiza la REF. CLIENTE de cada cuenta,
- **Cuando** construye el valor,
- **Entonces** deberá aplicar la lógica documentada del Código Validador: cuenta Banamex con concatenación de 7 segmentos basados en nombre del cliente, clave, código del banco, moneda y CodValidador; cuenta no-Banamex con nombre del cliente directo.

### Sección F — Datos de facturación

**Criterio F1 — RFC, Razón Social, Dirección fiscal**
- **Dado** que el sistema renderiza la sección de facturación,
- **Cuando** incluye los datos fiscales del cliente,
- **Entonces** deberá mostrar: RFC del cliente desde el Catálogo de Clientes; Razón Social del cliente desde el Catálogo de Clientes; Dirección fiscal completa del cliente (calle, número, colonia, ciudad, estado, país, CP) desde el Catálogo de Clientes.

### Sección G — Datos de entrega

**Criterio G1 — Pedido, Parciales, Contacto, Lugar**
- **Dado** que el sistema renderiza la sección de entrega,
- **Cuando** incluye los datos de entrega,
- **Entonces** deberá mostrar: Número de pedido interno; Parciales (SI/NO) según configuración del pedido; Contacto de entrega del pedido (si no existe, mostrar "NINGUNO"); Lugar de entrega completo (dirección).

> ** Aplica la misma duda de generación de folio interno, ya que no se ha enviado el pedido aún. **

> ** Confirmar si es el contacto de entrega, contacto del cliente o contacto que realizó el pedido. **

### Sección H — Pie legal de la empresa emisora

**Criterio H1 — Contacto PROQUIFA México**
- **Dado** que el sistema renderiza el pie del documento,
- **Cuando** incluye la información de contacto,
- **Entonces** deberá mostrar: Redes sociales: @PROQUIFA, /PROQUIFA_OFICIAL, PROQUIFA (LinkedIn); Teléfonos: Ciudad de México 55 1315 1498 y Guadalajara 01 (33) 4770 1170; Web: www.proquifa.com.mx; Correo: ventas@proquifa.com.mx.

**Criterio H2 — Razón social legal de la empresa emisora**
- **Dado** que el sistema renderiza el pie legal,
- **Cuando** incluye la razón social legal,
- **Entonces** deberá mostrar la razón social legal completa y dirección legal de la empresa emisora del pedido (Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V. o Proveedora Quimico Farmaceutica S.A. de C.V.).

**Criterio H3 — Sellos de certificación y métodos de pago**
- **Dado** que el sistema renderiza el pie,
- **Cuando** incluye certificaciones y métodos de pago aceptados,
- **Entonces** deberá mostrar: sello ISO 9001:2015, sello NEEC (Nuevo Esquema de Empresas Certificadas, programa SAT exclusivo México), y los métodos de pago aceptados (American Express / Tarjetas Bienvenidas).

> ** Confirmar con el cliente si estas certificaciones siguen vigentes, así como su diseño. **

**Criterio H4 — Numeración de página**
- **Dado** que el sistema completa el documento,
- **Cuando** incluye el contador de páginas,
- **Entonces** deberá mostrar "X/Y" en el pie del documento, donde X es la página actual e Y es el total. Si el documento es de una sola página, se muestra "1/1".

**Criterio H5 — Logos de catálogos farmacéuticos**
- **Dado** que el sistema renderiza la línea final del documento,
- **Cuando** incluye los logos de catálogos y proveedores reconocidos,
- **Entonces** deberá mostrar los logos aplicables a la empresa emisora del pedido: EDQM, FEUM, USP, Microbiologics, APACOR, CHATA Biosystems, Pharmaffiliates (varían según empresa emisora).

> ** Confirmar si esta info sigue vigente. **

### Sección I — Paginación automática

**Criterio I1 — Múltiples páginas cuando las partidas exceden una página**
- **Dado** que el pedido tiene partidas que exceden el espacio disponible en una sola página,
- **Cuando** el sistema renderiza el documento,
- **Entonces** deberá generar páginas adicionales con la misma cabecera y pie completo. Las partidas continúan en las páginas adicionales. La numeración se actualiza (1/3, 2/3, 3/3). Este comportamiento ya existe en PQF2.

### Sección J — Persistencia y consulta post-envío

**Criterio J1 — Generación bajo demanda durante el flujo previo al envío**
- **Dado** que un usuario presiona "Tramitar" en el módulo Tramitar Pedido,
- **Cuando** el sistema procesa la acción,
- **Entonces** deberá generar el PDF dinámicamente con los datos vigentes en ese momento y mostrarlo en previsualización al usuario. El PDF no se almacena en base de datos en esta etapa.

**Criterio J2 — Regeneración con datos actualizados al reintentar**
- **Dado** que el usuario abandonó el flujo sin enviar la Proforma y vuelve a presionar "Tramitar",
- **Cuando** el sistema procesa la nueva acción,
- **Entonces** deberá regenerar el PDF desde cero con los datos fuente vigentes en ese nuevo momento. Si cambiaron datos entre intentos, el nuevo PDF los refleja.

**Criterio J3 — Persistencia del PDF al confirmar envío exitoso del correo**
- **Dado** que el sistema confirma que el correo de envío al cliente fue exitoso,
- **Cuando** se completa el envío,
- **Entonces** deberá persistir el PDF final en base de datos como artefacto histórico inmutable. El pendiente en Tramitar Pedido se cierra.

> ** Pendiente decisión técnica del tipo de almacenamiento del PDF (binario completo vs snapshot estructurado). **

**Criterio J4 — Consulta del PDF histórico desde Validar Cobro**
- **Dado** que una Proforma fue enviada y persistida,
- **Cuando** un usuario consulta el módulo Validar Cobro para procesar el cobro asociado,
- **Entonces** el sistema deberá permitir acceder al PDF histórico de la Proforma. El PDF se entrega tal cual fue almacenado, sin regeneración desde datos fuente actuales.

**Criterio J5 — Sin reenvío posterior**
- **Dado** que una Proforma fue enviada y persistida,
- **Cuando** un usuario intenta reenviarla desde el módulo Tramitar Pedido,
- **Entonces** el sistema no deberá ofrecer esa funcionalidad. El pendiente está cerrado y la Proforma original se conserva como registro permanente.

---

## Notas

- Esta fila documenta el contenido y la generación del PDF de Proforma para clientes con Región México. La equivalente para Región Perú se documenta en requisito independiente.
- El requisito es un rediseño del documento de Proforma. La estructura visual específica (colores exactos, layout de bandas, tipografía, espaciados) es decisión del equipo de diseño UI; este requisito se enfoca en la información que debe contener cada sección del documento.
- Aplica exclusivamente a pedidos Prepago que NO seleccionaron Factura por Adelantado.
- El branding del documento varía por empresa emisora del pedido (logo, color institucional, dirección y razón social legal).
- El PDF se genera bajo demanda en cada presión del botón "Tramitar" durante el flujo previo al envío de la Proforma. En esta etapa el PDF NO se almacena en base de datos.
- Cuando el sistema confirma que el correo de envío al cliente fue exitoso, el PDF final se persiste en base de datos como artefacto histórico inmutable.
- El PDF histórico de la Proforma enviada se puede consultar desde el módulo Validar Cobro para verificación y trazabilidad del cobro asociado.
- La moneda aplicada a los cálculos del documento es la moneda de facturación configurada en el Catálogo del cliente, no la moneda del pedido.
- El tipo de cambio aplicado a la conversión es el del día de generación de la Proforma.
- Las condiciones de pago provienen de la configuración del cliente en el Catálogo.
- El foliador es global lineal PQF2 para Proformas (un solo contador sin segmentación por empresa).
- La paginación automática del PDF cuando las partidas exceden una página es comportamiento ya existente en PQF2.

> ** Pendiente confirmar la regla exacta de Vigencia del documento. **

> ** Pendiente confirmar si el prefijo "PRF-" del folio aplica solo a la representación visual del PDF o también se persiste en el folio interno almacenado en base de datos. **

> ** Pendiente confirmar si la sección Cliente muestra el Alias del cliente o la Razón Social. **

> ** Pendiente confirmar si los pedidos Prepago siempre son Método de Pago PUE. **

> ** Pendiente confirmar si la sección de Datos Bancarios siempre muestra las dos cuentas M.N. y DLS. **

> ** Pendiente respecto al folio interno del pedido que aparece en la sección de Entrega. **

> ** Pendiente definir qué dato fuente corresponde al campo "Contacto" de la sección de Entrega. **

> ** Pendiente validar con el cliente la vigencia de las certificaciones del pie del documento (ISO 9001:2015, NEEC). **

> ** Pendiente validar con el cliente la vigencia de los logos de catálogos farmacéuticos. **

> ** Pendiente decisión técnica: tipo de almacenamiento del PDF persistido en base de datos. **

> ** Pendiente decisión técnica del momento de consumo del folio del foliador PQF2. **
