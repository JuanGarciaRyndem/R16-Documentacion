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

> Yo como **ESAC**, quiero que el sistema genere automáticamente el PDF de la Proforma con el branding de la empresa emisora del pedido al tramitar un pedido Prepago sin Factura por Adelantado para clientes de México, para entregar al cliente un documento estandarizado y correcto que respalde el cobro por adelantado.

---

## Requisito

El sistema debe generar un PDF de Proforma al tramitar un pedido Prepago sin Factura por Adelantado para clientes con Región México, con un diseño estandarizado cuyo branding varía según la empresa emisora del pedido (Golocaer, Mungen, Proquifa o Proveedora Quimico Farmaceutica, las cuatro empresas del grupo PROQUIFA México en operación). El PDF se reconstruye dinámicamente en cada consulta tomando los valores aplicados al documento al momento de su generación inicial.

---

## Alcance

### Aplica a

- Generación del PDF de Proforma al tramitar un pedido en modalidad Prepago sin factura por adelantado para clientes con Región México.
- Cuatro empresas emisoras del grupo PROQUIFA México con branding propio: Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V. y Proveedora Quimico Farmaceutica S.A. de C.V.
- Branding diferenciado por empresa emisora.
- Generación bajo demanda del PDF durante el flujo previo al envío de la Proforma al cliente (cada vez que se tramita el pedido se previsualiza el PDF con datos vigentes).
- Almacenamiento del PDF al recibir confirmación de envío exitoso del correo al cliente.
- Acceso al PDF histórico desde el módulo Validar Cobro una vez la Proforma fue enviada.
- Foliador global PQF2 con prefijo PRF en la representación visual del documento.
- Paginación automática cuando las partidas exceden el espacio de una página (comportamiento ya existente del sistema).
- Aplicación de catálogos fiscales SAT mexicanos (RFC, IVA, CLABE, certificación NEEC, Art. 29 y 29A CFF).

### No aplica a

- Pedidos Crédito sin Factura por Adelantado ni Crédito/Prepago con Factura por Adelantado.
- Pedidos para clientes con Región Perú. Esa funcionalidad se documenta en requisito independiente.
- La construcción de la referencia bancaria del cliente (REF. CLIENTE): se documenta en el requisito de Referencia de Pago. Esta fila únicamente presenta el dato ya construido.

---

## Reglas de Negocio

**Regla 1 — Generación únicamente en pedidos Prepago sin Factura por Adelantado para clientes México**
El sistema genera el PDF de Proforma únicamente cuando el pedido es en modalidad Prepago sin Factura por Adelantado y el cliente tiene Región = México. Los pedidos Crédito (con o sin Factura por Adelantado) y los pedidos Prepago con Factura por Adelantado no generan Proforma.

**Regla 2 — Diferenciación por empresa emisora**
La Proforma se diferencia según la empresa emisora del pedido (una de las cuatro del grupo PROQUIFA México) en logo, color institucional, dirección y razón social legal correspondiente.

**Regla 3 — Foliador global con prefijo PRF**
El folio de la Proforma usa el foliador global PQF2 para Proformas (un solo contador sin segmentación por empresa) en formato MMDDAA-Consecutivo, con prefijo "PRF-" en la representación visual del documento (ejemplo: "PRF-031826-691"). El prefijo "PRF-" es **exclusivamente visual**: en base de datos se almacena únicamente el número de folio, sin el prefijo; la transferencia del dato a Legacy debe considerarse sin el prefijo. El folio se consume únicamente al confirmarse el envío exitoso del correo al cliente: se reintenta con el mismo folio hasta que el envío se complete exitosamente, sin descartarlo en cada intento fallido, de modo que las previsualizaciones abandonadas no generan huecos en la numeración.

**Regla 4 — Vigencia del documento**
La Proforma calcula y muestra una fecha de vigencia en formato DD/MM/YYYY. La vigencia es de **30 días naturales**, contados a partir del momento de la **generación** de la Proforma.

**Regla 5 — Generación bajo demanda durante el flujo previo al envío**
Al tramitar un pedido Prepago sin Factura por Adelantado de cliente México, el sistema genera el PDF de la Proforma dinámicamente, leyendo los datos vigentes en ese momento desde las fuentes (Catálogo de Clientes, Pedido, Catálogo de Cuentas Bancarias, Tabla de Empresas, Referencia Bancaria del Cliente), y lo muestra en previsualización. En esta etapa el PDF no se almacena.

**Regla 6 — Regeneración con datos actualizados si el usuario abandona la previsualización y reintenta**
Si un usuario vio la previsualización pero abandonó el flujo sin enviar la Proforma, al volver a tramitar el pedido el sistema regenera el PDF desde cero leyendo los datos vigentes en ese nuevo momento. Si entre intentos cambiaron datos fuente (razón social del cliente, dirección fiscal, precios, cuentas bancarias, etc.), la nueva versión los refleja. Esto aplica únicamente mientras la Proforma no haya sido enviada al cliente.

**Regla 7 — Almacenamiento del PDF al recibir confirmación del envío exitoso del correo**
Al confirmarse que el correo de la Proforma fue enviado exitosamente al cliente, el sistema almacena la versión final del PDF como artefacto histórico inmutable, con los datos exactos enviados; la Proforma enviada queda como registro permanente del documento exacto que recibió el cliente.

**Regla 8 — Sin regeneración posterior al envío**
Una Proforma enviada y almacenada se entrega, al consultarse históricamente, desde el PDF almacenado, sin regenerarlo desde los datos fuente actuales. El sistema no ofrece funcionalidad de reenvío. Si los datos fuente cambian después del envío, la Proforma histórica conserva los datos originales.

**Regla 9 — Consulta del PDF histórico desde Validar Cobro**
Una Proforma enviada y persistida puede consultarse desde el módulo Validar Cobro para verificación y trazabilidad del cobro asociado, accediendo al PDF histórico.

**Regla 10 — Disclaimer legal SAT obligatorio**
El documento muestra el texto fijo: "ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DE UN CFDI. CARECE DE VALIDEZ FISCAL SEGÚN ART.29 Y 29A CFF".

**Regla 11 — Paginación automática (comportamiento existente)**
Cuando las partidas del pedido exceden el espacio disponible en una sola página, el sistema genera páginas adicionales con la misma cabecera y pie completo, mostrando la numeración "X/Y" en cada página. Este comportamiento ya existe en PQF2.

**Regla 12 — Origen de los datos por sección**
Los paneles del documento se arman desde las fuentes indicadas: datos de partidas (cantidad, descripción, precio unitario, importe) desde el Pedido; identificación del cliente, RFC y dirección fiscal desde el Catálogo de Clientes; moneda aplicada a los cálculos desde la moneda de facturación configurada en el Catálogo del cliente (no del pedido); Condiciones de Pago desde la configuración del cliente en el Catálogo (sección Cobros); cuentas bancarias (Banca, Sucursal, Cuenta, CLABE) desde el Catálogo de Cuentas Bancarias del grupo PROQUIFA México; REF. CLIENTE de cada cuenta, presentada tal como se construye en el requisito de Referencia de Pago; Pedido interno, Parciales, Contacto y Lugar de entrega desde el Pedido; logo, color institucional, dirección y razón social legal generados por el sistema.

---

## Riesgos

**Riesgo 1 — Tipo de cambio inconsistente entre Proforma y validación de pago posterior**
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
- **Entonces** deberá mostrar el folio con formato "PRF-MMDDAA-Consecutivo" (ejemplo: "PRF-031826-691"). El consecutivo corresponde al foliador global PQF2. El prefijo "PRF-" es solo visual (en base de datos se almacena únicamente el número de folio) y el folio se consume únicamente al confirmarse el envío exitoso, sin huecos por intentos fallidos.

**Criterio A5 — Vigencia del documento**
- **Dado** que el sistema renderiza la cabecera,
- **Cuando** incluye el campo Vigencia,
- **Entonces** deberá mostrar la fecha de vigencia en formato DD/MM/YYYY, calculada como **30 días naturales** a partir de la fecha de generación de la Proforma.

### Sección B — Identificación del cliente

**Criterio B1 — Identificación del cliente**
- **Dado** que el sistema renderiza la sección Cliente,
- **Cuando** incluye el identificador del cliente,
- **Entonces** deberá mostrar la Razón Social del cliente desde el Catálogo de Clientes.

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
- **Entonces** deberá mostrar el texto "PAGO EN UNA SOLA EXHIBICIÓN". Esta leyenda es **fija para toda Proforma** (el esquema Prepago siempre asume PUE), independiente de la configuración de Método de Pago del cliente. El documento carece de validez fiscal (ver Regla 10); la leyenda no debe presentarse como una leyenda fiscal obligatoria del SAT.

### Sección E — Datos bancarios

**Criterio E1 — Cuentas activas más recientes de la empresa que factura**
- **Dado** que el sistema renderiza la sección de datos bancarios,
- **Cuando** arma el contenido,
- **Entonces** deberá mostrar las **dos cuentas activas más recientes** (según Fecha de última actualización) de la empresa que factura, independientemente de la moneda del pedido. Si solo existe una cuenta activa, se muestra únicamente esa; si hay más de dos registradas, se toman las dos más recientes. Este mismo mecanismo aplica también a Proformas de Perú. Los campos por cuenta son: Moneda, Banca, Sucursal, Cuenta, CLABE y REF. CLIENTE.

**Criterio E2 — Referencia bancaria del cliente (REF. CLIENTE)**
- **Dado** que el sistema renderiza la sección de datos bancarios,
- **Cuando** incluye la REF. CLIENTE de cada cuenta,
- **Entonces** deberá presentar el valor tal como se construye conforme al requisito de Referencia de Pago, que documenta la lógica de construcción de ese dato.

### Sección F — Datos de facturación

**Criterio F1 — RFC, Razón Social, Dirección fiscal**
- **Dado** que el sistema renderiza la sección de facturación,
- **Cuando** incluye los datos fiscales del cliente,
- **Entonces** deberá mostrar: RFC del cliente desde el Catálogo de Clientes; Razón Social del cliente desde el Catálogo de Clientes; Dirección fiscal completa del cliente (calle, número, colonia, ciudad, estado, país, CP) desde el Catálogo de Clientes.

### Sección G — Datos de entrega

**Criterio G1 — Pedido, Parciales, Contacto, Lugar**
- **Dado** que el sistema renderiza la sección de entrega,
- **Cuando** incluye los datos de entrega,
- **Entonces** deberá mostrar: Número de pedido interno; Parciales (SI/NO) según configuración del pedido; Contacto (Título+Contacto, con referencia a la tabla Pedidos en Legacy; si no existe, mostrar "NINGUNO"); Lugar de entrega completo (dirección).

### Sección H — Información legal de la empresa emisora

**Criterio H1 — Contacto PROQUIFA México**
- **Dado** que el sistema arma la información de contacto de PROQUIFA México,
- **Cuando** la incluye en el documento,
- **Entonces** deberá mostrar: Redes sociales: @PROQUIFA, /PROQUIFA_OFICIAL, PROQUIFA (LinkedIn); Teléfonos: Ciudad de México 55 1315 1498 y Guadalajara 01 (33) 4770 1170; Web: www.proquifa.com.mx; Correo: ventas@proquifa.com.mx.

**Criterio H2 — Razón social legal de la empresa emisora**
- **Dado** que el sistema arma la información legal de la empresa emisora,
- **Cuando** la incluye en el documento,
- **Entonces** deberá mostrar la razón social legal completa y dirección legal de la empresa emisora del pedido (Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V. o Proveedora Quimico Farmaceutica S.A. de C.V.).

**Criterio H3 — Sellos de certificación y métodos de pago**
- **Dado** que el sistema arma las certificaciones y métodos de pago aceptados,
- **Cuando** los incluye en el documento,
- **Entonces** deberá mostrar los sellos de las certificaciones vigentes ISO 9001:2015 y OEA (Operador Económico Autorizado). Los métodos de pago a mostrar quedan pendientes de confirmar con el cliente.

**Criterio H4 — Numeración de página**
- **Dado** que el sistema completa el documento,
- **Cuando** incluye el contador de páginas,
- **Entonces** deberá mostrar "X/Y", donde X es la página actual e Y es el total. Si el documento es de una sola página, se muestra "1/1".

**Criterio H5 — Sin logos de catálogos farmacéuticos ni de marcas**
- **Dado** que el sistema completa el documento,
- **Cuando** lo arma,
- **Entonces** no deberá incluir logos de catálogos farmacéuticos ni de marcas.

### Sección I — Paginación automática

**Criterio I1 — Múltiples páginas cuando las partidas exceden una página**
- **Dado** que el pedido tiene partidas que exceden el espacio disponible en una sola página,
- **Cuando** el sistema renderiza el documento,
- **Entonces** deberá generar páginas adicionales con la misma cabecera y pie completo. Las partidas continúan en las páginas adicionales. La numeración se actualiza (1/3, 2/3, 3/3). Este comportamiento ya existe en PQF2.

### Sección J — Persistencia y consulta post-envío

**Criterio J1 — Generación bajo demanda durante el flujo previo al envío**
- **Dado** que se ejecuta la acción de tramitar en el módulo Tramitar Pedido,
- **Cuando** el sistema procesa la acción,
- **Entonces** deberá generar el PDF dinámicamente con los datos vigentes en ese momento y mostrarlo en previsualización al usuario. El PDF no se almacena en esta etapa.

**Criterio J2 — Regeneración con datos actualizados al reintentar**
- **Dado** que el usuario abandonó el flujo sin enviar la Proforma y vuelve a tramitar el pedido,
- **Cuando** el sistema procesa la nueva acción,
- **Entonces** deberá regenerar el PDF desde cero con los datos fuente vigentes en ese nuevo momento. Si cambiaron datos entre intentos, el nuevo PDF los refleja.

**Criterio J3 — Almacenamiento del PDF al confirmar envío exitoso del correo**
- **Dado** que el sistema confirma que el correo de envío al cliente fue exitoso,
- **Cuando** se completa el envío,
- **Entonces** deberá almacenar el PDF final como artefacto histórico inmutable, como archivo/binario y no sufre regeneración: es el PDF generado originalmente el que se conserva y consulta, no se reconstruye a partir de datos.

**Criterio J4 — Consulta del PDF histórico desde Validar Cobro**
- **Dado** que una Proforma fue enviada y almacenada,
- **Cuando** un usuario consulta el módulo Validar Cobro para procesar el cobro asociado,
- **Entonces** el sistema deberá permitir acceder al PDF histórico de la Proforma. El PDF se entrega tal cual fue almacenado, sin regeneración desde datos fuente actuales.

**Criterio J5 — Sin reenvío posterior**
- **Dado** que una Proforma fue enviada y almacenada,
- **Cuando** un usuario intenta reenviarla desde el módulo Tramitar Pedido,
- **Entonces** el sistema no deberá ofrecer esa funcionalidad. La Proforma original se conserva como registro permanente.

---

## Notas

- Esta fila documenta el contenido y la generación del PDF de Proforma para pedidos Prepago sin Factura por Adelantado de clientes con Región México; la equivalente para Región Perú se documenta en requisito independiente.
- El requisito es un rediseño del documento de Proforma: se enfoca en la información que debe contener cada sección. La estructura visual específica (colores exactos, layout de bandas, tipografía, espaciados, posición de cada bloque en la página) es decisión del equipo de diseño UI.
- La construcción de la referencia bancaria del cliente (REF. CLIENTE) se documenta en el requisito de Referencia de Pago; esta fila únicamente presenta el dato.
- Pendiente: confirmar con el cliente los métodos de pago a mostrar en el pie del documento.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|-------------------------|
| 1 | 2026-09-11 | Cierre de dudas resueltas / retiro de definición duplicada / actualización del pie / correcciones de consistencia | Se limpia el marcado de tachado/duda usado para registrar el cierre de DUDA-031 a DUDA-039, conservando las conclusiones ya resueltas en las Reglas, Criterios y Notas correspondientes (consumo del folio al confirmar envío, prefijo "PRF-" solo visual, vigencia de 30 días naturales, Razón Social en Sección Cliente, leyenda "PAGO EN UNA SOLA EXHIBICIÓN" fija, dos cuentas bancarias activas más recientes, campo Contacto con Título+Contacto de la tabla Pedidos en Legacy). Se elimina el Riesgo 1 (consumo prematuro del folio), ya resuelto, y se renumera el riesgo restante. Se retira de esta fila la construcción de la referencia bancaria del cliente (REF. CLIENTE), que se documenta en el requisito de Referencia de Pago; esta fila conserva únicamente su presentación (Alcance, Regla 12, Criterio E2, Notas). Se actualiza el pie del documento: certificaciones vigentes ISO 9001:2015 y OEA, se retira NEEC, y se marcan como pendientes los métodos de pago a mostrar (Criterio H3); se establece que el documento no incluye logos de catálogos farmacéuticos ni de marcas (Criterio H5). Se cierra el pendiente del folio interno del pedido en la sección de Entrega. Correcciones de consistencia: se acota la Historia de Usuario a pedidos Prepago sin Factura por Adelantado; se retira la calificación de "lineal" del foliador (Alcance, Regla 3, Criterio A4); se retira la calificación de la leyenda "PAGO EN UNA SOLA EXHIBICIÓN" como fiscal obligatoria del SAT, que contradecía el disclaimer de la Regla 10 (Criterio D5); se retiran las referencias al cierre del pendiente en Tramitar Pedido de las Reglas 7/8 y Criterios J3/J5 (corresponde a otro requisito) y se sustituye la mención a la persistencia en base de datos por el almacenamiento del documento; se retiran las menciones al botón de pantalla "Tramitar" por corresponder a detalle de diseño (Alcance, Reglas 5/6, Criterios J1/J2, Notas); se retiran las referencias posicionales al "pie del documento" de la Sección H y los Criterios H1-H4, por corresponder a detalle de diseño (se renombra la sección a "Información legal de la empresa emisora"). Se reescriben las Notas completas, retirando los bullets que reproducían reglas y criterios ya documentados (incluido el desglose del origen de los datos por sección, cubierto íntegro por la Regla 12), conservando únicamente el encuadre de la fila, la delimitación entre requisito y diseño, las referencias cruzadas a otros requisitos y el único pendiente vigente (métodos de pago del pie). |
