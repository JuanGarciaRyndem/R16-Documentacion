# R16A-RE-FU-017 — Diseño y generación de Documentos: Proforma Perú

| Campo                   | Valor                                                                                                                                                                                                                                                                                                   |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ID**                  | R16A-RE-FU-017                                                                                                                                                                                                                                                                                          |
| **Título**              | Diseño y generación de Documentos: Proforma Perú                                                                                                                                                                                                                                                        |
| **Módulo / Épica**      | Tramitar Pedido                                                                                                                                                                                                                                                                                         |
| **Historia de Usuario** | Yo como ESAC, quiero que el sistema genere automáticamente el PDF de la Proforma adaptado a las convenciones fiscales y administrativas peruanas al tramitar un pedido Prepago sin Factura por Adelantado para clientes de Perú, para entregar al cliente un documento estandarizado y conforme a esas convenciones que respalde el cobro por adelantado. |
| **Prioridad**           | Alta                                                                                                                                                                                                                                                                                                    |
| **Estado**              | Propuesto                                                                                                                                                                                                                                                                                               |
| **Requisito asociado**  | R16.1M-RE-FU-007                                                                                                                                                                                                                                                                                        |

---

## Requisito Funcional

El sistema debe generar un PDF de Proforma al tramitar un pedido Prepago sin Factura por Adelantado para clientes con Región Perú, con un diseño estandarizado equivalente al de la Proforma México pero adaptado a las convenciones fiscales y administrativas peruanas. Dado que la única empresa emisora del grupo operando en Perú es Golocaer S.A.C., el branding del documento es único. El PDF se genera bajo demanda durante el flujo previo al envío al cliente y, al confirmarse el envío, se almacena y queda accesible desde el módulo Validar Cobro.

> **~~⚠️ Precondición (OBS-032)~~** ~~La generación de Proforma Perú está condicionada a que la facturación / timbrado Perú esté habilitada productivamente. Mientras la facturación Perú no esté habilitada, no se genera Proforma Perú.~~ **[Actualizado — Decisión "Quitar Perú" 2026-07-17]** El cliente canceló Facturación y Timbrado de Perú; esto **no reduce el alcance de RE-FU-017**. La Proforma Perú se genera íntegra (generación, foliado, persistencia, consulta histórica). Lo que cambia es el estado final: la Proforma Perú nunca transiciona a `Facturada` — cierra en `CompletadaSinFactura`, responsabilidad de RE-FU-029. OBS-032 ya **no es bloqueante** para este requisito.

---

## Alcance

### Aplica a

- Generación del PDF de Proforma al tramitar un pedido en modalidad Prepago sin factura por adelantado para clientes con Región Perú.
- Empresa emisora única: Golocaer S.A.C. (la única empresa del grupo PROQUIFA operando actualmente en Perú).
- Generación bajo demanda del PDF durante el flujo previo al envío de la Proforma al cliente.
- Almacenamiento del PDF al recibir confirmación de envío exitoso del correo al cliente.
- Acceso al PDF histórico desde el módulo Validar Cobro una vez la Proforma fue enviada.
- Foliador global PQF2 con prefijo PRF en la representación visual del documento (compartido con Proformas México: un solo contador global de Proformas para todo el grupo).
- Paginación automática cuando las partidas exceden el espacio de una página (comportamiento ya existente del sistema).
- Aplicación de las convenciones fiscales y administrativas peruanas que el documento adopta (RUC, IGV, CCI, moneda PEN), y del texto de disclaimer basado en el Reglamento de Comprobantes de Pago y la Resolución de Superintendencia N° 097-2012/SUNAT.

### No aplica a

- Pedidos Crédito sin Factura por Adelantado ni Crédito/Prepago con Factura por Adelantado.
- Pedidos para clientes con Región México. Esa funcionalidad se documenta en requisito independiente.
- Otras empresas del grupo PROQUIFA. Solo Golocaer S.A.C. opera actualmente en Perú; las cuatro empresas del grupo México (Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V., Proveedora Quimico Farmaceutica S.A. de C.V.) no emiten proformas para clientes Perú.
- Los regímenes de Detracciones y Percepciones SUNAT: quedan sin objeto, dado que no se emiten comprobantes fiscales desde el sistema para clientes Perú (el IGV se conserva en el documento como referencia informativa; la facturación se realiza fuera de ProquifaNet).
- Generación de Proforma Perú condicionada a que la facturación / timbrado Perú esté habilitada productivamente (OBS-032): el cliente canceló Facturación y Timbrado de Perú; la Proforma Perú se genera sin esa precondición.
- La construcción de la referencia bancaria del cliente (REF. CLIENTE): se documenta en el requisito de Referencia de Pago. Esta fila únicamente presenta el dato ya construido.

---

## Reglas de Negocio

~~Regla 0 — Precondición: facturación Perú habilitada (OBS-032)~~ **[Anulada — Decisión "Quitar Perú" 2026-07-17]**
~~La generación de Proforma Perú depende de que la facturación Perú (timbrado SUNAT y catálogos asociados) esté habilitada productivamente. Mientras esa precondición no se cumpla, ningún flujo del sistema invoca este requisito y no se generan Proformas Perú ni pendientes derivados.~~

> **Decisión "Quitar Perú" (2026-07-17):** El cliente canceló Facturación y Timbrado de Perú. La Proforma Perú se genera sin bloqueante. El flujo cierra en `CompletadaSinFactura` (RE-FU-029); nunca transiciona a `Facturada`.

Regla 1 — Generación únicamente en pedidos Prepago sin Factura por Adelantado para clientes Perú
Cumplida la Regla 0, el sistema genera el PDF de Proforma con el diseño estandarizado para Perú únicamente cuando el pedido es en modalidad Prepago sin Factura por Adelantado y el cliente tiene Región = Perú. Los pedidos Crédito (con o sin Factura por Adelantado) y los pedidos Prepago con Factura por Adelantado no generan Proforma.

Regla 2 — Empresa emisora única Golocaer S.A.C.
La empresa emisora del documento para clientes Perú es siempre Golocaer S.A.C., única empresa del grupo PROQUIFA operando en Perú. No hay diferenciación por empresa emisora como en México.

Regla 3 — Foliador global con prefijo PRF compartido con Proformas México
El folio de la Proforma Perú usa el mismo foliador global PQF2 que las Proformas México (un solo contador global para todo el grupo, sin segmentación por región ni por empresa), en formato MMDDAA-Consecutivo, con prefijo "PRF-" en la representación visual del documento. El prefijo "PRF-" es exclusivamente visual: en base de datos se almacena únicamente el número de folio. El folio se consume únicamente al confirmarse el envío exitoso del correo al cliente.

Regla 4 — Vigencia del documento
La Proforma calcula y muestra una fecha de vigencia en formato DD/MM/YYYY. La vigencia es de **30 días naturales**, contados a partir del momento de la generación de la Proforma.

Regla 5 — Generación bajo demanda durante el flujo previo al envío
Al tramitar un pedido Prepago sin Factura por Adelantado de cliente Perú, el sistema genera el PDF de la Proforma dinámicamente, leyendo los datos vigentes en ese momento desde las fuentes (Catálogo de Clientes, Pedido, Catálogo de Cuentas Bancarias Perú, Referencia Bancaria del Cliente), y lo muestra en previsualización. En esta etapa el PDF no se almacena.

Regla 6 — Regeneración con datos actualizados si el usuario abandona la previsualización y reintenta
Si un usuario vio la previsualización pero abandonó el flujo sin enviar la Proforma, al volver a tramitar el pedido el sistema regenera el PDF desde cero leyendo los datos vigentes en ese nuevo momento. Si entre intentos cambiaron datos fuente (razón social del cliente, dirección fiscal, precios, cuentas bancarias, etc.), la nueva versión los refleja. Esto aplica únicamente mientras la Proforma no haya sido enviada al cliente.

Regla 7 — Almacenamiento del PDF al recibir confirmación del envío exitoso del correo
Al confirmarse que el correo de la Proforma fue enviado exitosamente al cliente, el sistema almacena la versión final del PDF como artefacto histórico inmutable, con los datos exactos enviados.

Regla 8 — Sin regeneración posterior al envío
Una Proforma enviada y almacenada se entrega, al consultarse históricamente, desde el PDF almacenado, sin regenerarlo desde los datos fuente actuales. El sistema no ofrece funcionalidad de reenvío.

Regla 9 — Consulta del PDF histórico desde Validar Cobro
Una Proforma enviada y almacenada puede consultarse desde el módulo Validar Cobro para verificación y trazabilidad del cobro asociado, accediendo al PDF histórico.

Regla 10 — Disclaimer legal SUNAT
El documento muestra el texto fijo, aprobado por el cliente junto con los diseños de los documentos: "ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DEL COMPROBANTE DE PAGO ELECTRÓNICO (CPE). CARECE DE VALIDEZ FISCAL Y TRIBUTARIA CONFORME AL REGLAMENTO DE COMPROBANTES DE PAGO Y RESOLUCIÓN DE SUPERINTENDENCIA N° 097-2012/SUNAT."

Regla 11 — Paginación automática (comportamiento existente)
Cuando las partidas del pedido exceden el espacio disponible en una sola página, el sistema genera páginas adicionales completas, mostrando la numeración "X/Y" en cada página. Este comportamiento ya existe en PQF2.

Regla 12 — Origen de los datos por sección
Los paneles del documento se arman desde las fuentes indicadas: datos de partidas (cantidad, descripción, precio unitario, importe) desde el Pedido; identificación del cliente, RUC y dirección fiscal desde el Catálogo de Clientes; moneda aplicada a los cálculos desde la moneda de facturación configurada en el Catálogo del cliente (no del pedido); Condiciones de Pago desde la configuración del cliente en el Catálogo; cuentas bancarias (Banca, Cuenta, CCI) desde el Catálogo de Cuentas Bancarias de Golocaer S.A.C. Perú, mostrando las dos cuentas activas más recientes con los mismos campos que en México, salvo la Sucursal, que no aplica; REF. CLIENTE de cada cuenta, presentada tal como se construye en el requisito de Referencia de Pago; Pedido interno, Parciales, Contacto y Lugar de entrega desde el Pedido; logo, color institucional, dirección y razón social legal generados por el sistema correspondientes a Golocaer S.A.C. Perú.

---

## Riesgos

Riesgo 1 — Tipo de cambio inconsistente entre Proforma y validación de pago posterior
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
Entonces deberá mostrar el texto aprobado por el cliente: "ESTE ES UN DOCUMENTO INFORMATIVO PREVIO A LA EMISIÓN DEL COMPROBANTE DE PAGO ELECTRÓNICO (CPE). CARECE DE VALIDEZ FISCAL Y TRIBUTARIA CONFORME AL REGLAMENTO DE COMPROBANTES DE PAGO Y RESOLUCIÓN DE SUPERINTENDENCIA N° 097-2012/SUNAT."

Criterio A3 — Título "Proforma"
Dado que el sistema renderiza la cabecera,
Cuando incluye el título del documento,
Entonces deberá mostrar el texto "Proforma".

Criterio A4 — Folio con prefijo PRF
Dado que el sistema renderiza la cabecera,
Cuando incluye el folio del documento,
Entonces deberá mostrar el folio con formato "PRF-MMDDAA-Consecutivo". El consecutivo corresponde al foliador global PQF2. El prefijo "PRF-" es solo visual y el folio se consume únicamente al confirmarse el envío exitoso.

Criterio A5 — Vigencia del documento
Dado que el sistema renderiza la cabecera,
Cuando incluye el campo Vigencia,
Entonces deberá mostrar la fecha de vigencia en formato DD/MM/YYYY, calculada como 30 días naturales a partir de la fecha de generación de la Proforma.

═══════════════════════════════════════════════════════════════
SECCIÓN B — IDENTIFICACIÓN DEL CLIENTE
═══════════════════════════════════════════════════════════════

Criterio B1 — Identificación del cliente
Dado que el sistema renderiza la sección Cliente,
Cuando incluye el identificador del cliente,
Entonces deberá mostrar la Razón Social del cliente desde el Catálogo de Clientes.

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
- "IGV" con tasa aplicable al pedido (18%) y monto calculado.
- "Gran Total" con monto, suma de Sub-Total e IGV.
La moneda aplicada es la moneda de facturación del cliente desde el Catálogo (no la moneda del pedido). Para Perú las monedas típicas son PEN (Soles) y USD.

Criterio D2 — Monto del Gran Total expresado en letra
Dado que el sistema renderiza la conversión a letras del Gran Total,
Cuando incluye la leyenda monetaria,
Entonces deberá mostrar el monto en palabras según la moneda:
- Si moneda = soles peruanos: "(XXX SOLES XX/100)". La nomenclatura oficial desde 2015 es "SOLES"; no se usa "NUEVOS SOLES".
- Si moneda = dólares: "(XXX DOLARES XX/100)".
- Otras monedas: nomenclatura correspondiente.

Criterio D3 — Tipo de Cambio (cuando aplica)
Dado que la moneda de facturación del cliente NO es soles peruanos,
Cuando el sistema renderiza la sección de pago,
Entonces deberá mostrar el tipo de cambio aplicado a la conversión. El tipo de cambio es el del día de generación: el mismo que el sistema ya utiliza para los pedidos que no están en dólares. No se requiere una fuente nueva o distinta.

Criterio D4 — Condiciones de Pago
Dado que el sistema renderiza la sección de pago,
Cuando incluye las condiciones,
Entonces deberá mostrar las condiciones de pago aplicables al cliente (ejemplo: "PREPAGO 100%"), provenientes de la configuración del cliente en el Catálogo.

Criterio D5 — Leyenda de pago
Dado que el sistema renderiza el final de la sección de pago,
Cuando incluye la leyenda de pago,
Entonces deberá mostrar el texto "OPERACIÓN AL CONTADO". Esta leyenda es fija para toda Proforma Perú, sustituye a la leyenda mexicana "Pago en una sola exhibición" y cumple con la clasificación de la normativa peruana (el escenario Prepago en Perú es siempre al contado).

═══════════════════════════════════════════════════════════════
SECCIÓN E — DATOS BANCARIOS
═══════════════════════════════════════════════════════════════

Criterio E1 — Cuentas bancarias de Golocaer S.A.C. Perú
Dado que el sistema renderiza la sección de datos bancarios,
Cuando arma el contenido,
Entonces deberá mostrar las dos cuentas activas más recientes de Golocaer S.A.C. Perú, independientemente de la moneda del pedido (mismo criterio que México). Las cuentas se obtienen de `EmpresaDatosBancarios` filtradas por `IdRegion = PER`.
Los campos por cuenta son los mismos que en México, salvo la Sucursal, que no aplica: Moneda, Banca, Cuenta, CCI (Código de Cuenta Interbancario de 20 dígitos, en lugar de CLABE) y REF. CLIENTE.

Criterio E2 — Referencia bancaria del cliente (REF. CLIENTE)
Dado que el sistema renderiza la sección de datos bancarios,
Cuando incluye la REF. CLIENTE de cada cuenta,
Entonces deberá presentar el valor tal como se construye conforme al requisito de Referencia de Pago: para Perú, al no existir mecanismo de identificación de pagos mediante Código Validador, la REF. CLIENTE es la Razón Social del cliente (mismo camino que bancos distintos de Banamex, RE-FU-006 Regla 6-PER).

═══════════════════════════════════════════════════════════════
SECCIÓN F — DATOS DE FACTURACIÓN
═══════════════════════════════════════════════════════════════

Criterio F1 — RUC, Razón Social, Dirección fiscal
Dado que el sistema renderiza la sección de facturación,
Cuando incluye los datos fiscales del cliente,
Entonces deberá mostrar:
- RUC del cliente desde el Catálogo de Clientes (en lugar de RFC; etiqueta del campo "RUC").
- Razón Social del cliente desde el Catálogo de Clientes.
- Dirección fiscal del cliente tal como esté capturada en el Catálogo de Clientes, que no diferencia la estructura de la dirección por región.

═══════════════════════════════════════════════════════════════
SECCIÓN G — DATOS DE ENTREGA
═══════════════════════════════════════════════════════════════

Criterio G1 — Pedido, Parciales, Contacto, Lugar
Dado que el sistema renderiza la sección de entrega,
Cuando incluye los datos de entrega,
Entonces deberá mostrar:
- Número de pedido interno.
- Parciales (SI/NO) según configuración del pedido.
- Contacto (Título+Contacto, con referencia a la tabla Pedidos en Legacy; si no existe, mostrar "NINGUNO").
- Lugar de entrega completo (dirección).

═══════════════════════════════════════════════════════════════
SECCIÓN H — INFORMACIÓN LEGAL DE GOLOCAER S.A.C.
═══════════════════════════════════════════════════════════════

Criterio H1 — Contacto Golocaer S.A.C. Perú
Dado que el sistema arma la información de contacto de Golocaer S.A.C. Perú,
Cuando la incluye en el documento,
Entonces deberá mostrar los datos de contacto institucionales de Golocaer S.A.C. Perú: redes sociales aplicables, teléfonos de oficinas Perú, web y correo de ventas Perú. ** Brecha pendiente (B3): no se cuenta actualmente con la información de contacto de Golocaer S.A.C. Perú (teléfonos, web institucional Perú, correo Perú, redes sociales Perú). **

Criterio H2 — Razón social legal de Golocaer S.A.C.
Dado que el sistema arma la información legal de Golocaer S.A.C.,
Cuando la incluye en el documento,
Entonces deberá mostrar la razón social legal completa "Golocaer S.A.C." con su dirección legal completa en Perú. ** Brecha pendiente (B3): la dirección legal de Golocaer S.A.C. en Perú no está disponible en el sistema actual. **

Criterio H3 — Sellos de certificación y métodos de pago aceptados
Dado que el sistema arma las certificaciones y métodos de pago aceptados,
Cuando los incluye en el documento,
Entonces deberá mostrar las certificaciones vigentes aplicables a Golocaer S.A.C. Perú. El sello NEEC no aplica para Perú (programa SAT exclusivo México). ** Brecha pendiente (B6): confirmar si Golocaer Perú cuenta con certificación ISO 9001 o equivalente, y los métodos de pago aceptados aplicables al mercado peruano. **

Criterio H4 — Numeración de página
Dado que el sistema completa el documento,
Cuando incluye el contador de páginas,
Entonces deberá mostrar "X/Y", donde X es la página actual e Y es el total. Si el documento es de una sola página, se muestra "1/1".

Criterio H5 — Sin logos de catálogos farmacéuticos ni de marcas
Dado que el sistema completa el documento,
Cuando lo arma,
Entonces no deberá incluir logos de catálogos farmacéuticos ni de marcas.

═══════════════════════════════════════════════════════════════
SECCIÓN I — PAGINACIÓN AUTOMÁTICA
═══════════════════════════════════════════════════════════════

Criterio I1 — Múltiples páginas cuando las partidas exceden una página
Dado que el pedido tiene partidas que exceden el espacio disponible en una sola página,
Cuando el sistema renderiza el documento,
Entonces deberá generar páginas adicionales completas. Las partidas continúan en las páginas adicionales. La numeración se actualiza (1/3, 2/3, 3/3). Este comportamiento ya existe en PQF2.

═══════════════════════════════════════════════════════════════
SECCIÓN J — ALMACENAMIENTO Y CONSULTA POST-ENVÍO
═══════════════════════════════════════════════════════════════

Criterio J1 — Generación bajo demanda durante el flujo previo al envío
Dado que se ejecuta la acción de tramitar en el módulo Tramitar Pedido para un pedido Perú,
Cuando el sistema procesa la acción,
Entonces deberá generar el PDF dinámicamente con los datos vigentes en ese momento y mostrarlo en previsualización al usuario. El PDF no se almacena en esta etapa.

Criterio J2 — Regeneración con datos actualizados al reintentar
Dado que el usuario abandonó el flujo sin enviar la Proforma y vuelve a tramitar el pedido,
Cuando el sistema procesa la nueva acción,
Entonces deberá regenerar el PDF desde cero con los datos fuente vigentes en ese nuevo momento. Si cambiaron datos entre intentos, el nuevo PDF los refleja.

Criterio J3 — Almacenamiento del PDF al confirmar envío exitoso del correo
Dado que el sistema confirma que el correo de envío al cliente fue exitoso,
Cuando se completa el envío,
Entonces deberá almacenar el PDF final como artefacto histórico inmutable.

Criterio J4 — Consulta del PDF histórico desde Validar Cobro
Dado que una Proforma fue enviada y almacenada,
Cuando un usuario consulta el módulo Validar Cobro para procesar el cobro asociado,
Entonces el sistema deberá permitir acceder al PDF histórico de la Proforma. El PDF se entrega tal cual fue almacenado, sin regeneración desde datos fuente actuales.

Criterio J5 — Sin reenvío posterior
Dado que una Proforma fue enviada y almacenada,
Cuando un usuario intenta reenviarla desde el módulo Tramitar Pedido,
Entonces el sistema no deberá ofrecer esa funcionalidad. La Proforma original se conserva como registro permanente.

---

## Notas Adicionales

- Esta fila documenta el contenido y la generación del PDF de Proforma para clientes con Región Perú. La equivalente para Región México se documenta en R16A-RE-FU-016.
- El requisito es un rediseño del documento de Proforma. La estructura visual específica (colores, layout, tipografía, espaciados) es decisión del equipo de diseño UI; este requisito se enfoca en la información que debe contener cada sección, adaptada a las convenciones fiscales y administrativas peruanas (IGV, RUC, CCI, PEN, disclaimer SUNAT).
- La construcción de la REF. CLIENTE de cada cuenta se documenta en el requisito de Referencia de Pago (ver también RE-FU-006 Regla 6-PER).
- OBS-032 ya no bloquea este requisito (Decisión "Quitar Perú" 2026-07-17): la Proforma Perú se genera íntegra; el ciclo de vida del pedido cierra en `CompletadaSinFactura` (RE-FU-029) en lugar de transicionar a `Facturada`.

═══════════════════════════════════════════════════════════════
BRECHAS PENDIENTES DE RESOLUCIÓN ANTES DE HABILITAR PERÚ
═══════════════════════════════════════════════════════════════

B1 — Datos bancarios de Golocaer S.A.C. Perú (DML)
El criterio de visualización está resuelto (mismo mecanismo que México: dos cuentas activas más recientes). Pendiente restante: insertar los registros reales en `EmpresaDatosBancarios` + `DatosBancarios` para los bancos peruanos de Golocaer S.A.C. (BCP, BBVA Continental u otros). Ver RE-FU-017_BD.md brecha B1.

B3 — Datos legales y de contacto de Golocaer S.A.C. Perú
Dirección legal, teléfonos, web institucional, correo de ventas y redes sociales de Golocaer Perú no están disponibles en el sistema actual. Pendiente recopilar y capturar. Ver Criterios H1 y H2.

B6 — Certificaciones y métodos de pago aplicables a Golocaer S.A.C. Perú
Confirmar si Golocaer Perú cuenta con certificación ISO 9001 o equivalente, y los métodos de pago aceptados aplicables al mercado peruano. Ver Criterio H3.

Brechas resueltas: B2 — referencia bancaria (REF. CLIENTE = Razón Social por default, RE-FU-006 Regla 6-PER); B4 — disclaimer legal SUNAT (texto fijo aprobado por el cliente, Regla 10 / Criterio A2); B5 — Detracciones y Percepciones (ambos regímenes quedan sin objeto: no se emiten comprobantes fiscales desde el sistema para clientes Perú); B7 — catálogos farmacéuticos (el documento no incluye logos de catálogos farmacéuticos ni de marcas, Criterio H5); B8 — título del documento ("Proforma", Criterio A3); B9 — nomenclatura del monto en letra ("SOLES", Criterio D2); B10 — tipo de cambio (el mismo que ya usa el sistema para pedidos no-USD, Criterio D3).

Mientras B1, B3 y B6 no se resuelvan, el cliente Perú no puede recibir Proforma productiva completa con el formato adaptado.


---

## Cambios

| #   | Fecha      | Observación | Descripción del cambio                                                                                                                                                                                                                                                                   |
| --- | ---------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | 2026-06-10 | OBS-032     | Revisado inicialmente en contexto de OBS-032: la corrección principal se aplicó en FU-018 (listado FxA). Sin cambios de contenido en este archivo en esa primera pasada. |
| 2   | 2026-06-25 | OBS-032     | Incorporación explícita de la precondición OBS-032 en el cuerpo del requisito: se agrega nota de precondición en "Requisito Funcional", nueva viñeta en Alcance / No aplica a, y Regla 0 — la generación de Proforma Perú depende de que la facturación / timbrado Perú esté habilitada productivamente; mientras no lo esté, no se generan Proformas Perú ni pendientes asociados, evitando ruido operativo. Renumeración de brechas a secuencia continua B1–B10. |
| 3   | 2026-07-17 | Duda FU-006/FU-017 | Resolución referencia bancaria Perú (B2): Perú no tiene mecanismo de Código Validador; REF. CLIENTE usa Razón Social por default (mismo camino que bancos no-Banamex, RE-FU-006 Regla 6-PER). Criterio E2 y Brecha B2 actualizados. |
| 4   | 2026-07-21 | DIS-SOL v1.2 / Decisión "Quitar Perú" 2026-07-17 | Actualización integral tras validación del Diseño de Solución v1.2: (1) OBS-032 anulada como bloqueante — la Proforma Perú procede íntegramente; ciclo cierra en `CompletadaSinFactura` (RE-FU-029). Regla 0 y precondición en Requisito Funcional marcadas como anuladas. (2) Criterio A3: título "Proforma" confirmado (DUDA-041). (3) Criterio B1: dato fuente confirmado como Razón Social (DIS-SOL v1.1). (4) Criterio D2: nomenclatura "SOLES" confirmada (DUDA-042). (5) Criterio D5: leyenda Contado/Crédito como texto fijo en plantilla GOLPERU_PER_PRO, no campo DTO (DUDA-043). (6) Criterio E1: se muestran las dos cuentas activas más recientes (DUDA-118/036). (7) Brechas B8 y B9 cerradas (DUDA-041, DUDA-042). (8) Brecha B1 parcialmente resuelta en criterio de visualización; DML pendiente. |
| 5   | 2026-08-21 | DUDA-009 / DUDA-018 / DUDA-041 / DUDA-042 / DUDA-043 / DUDA-044 / DUDA-054 | Revisión de trazabilidad de dudas cerradas de la hoja "R16 - Dudas a Cliente": (1) DUDA-009 — Régimen de Detracciones (SPOT) cerrado de forma definitiva (NO aplica); se separa de Percepciones del IGV, que sigue pendiente. Actualizados Alcance/No aplica a, Riesgo 2, Notas Adicionales y Brecha B5. (2) DUDA-044 — se agrega cita cruzada en Criterio E1 y Brecha B1 (mismo mecanismo que México, ya reflejado vía DUDA-118/036). (3) DUDA-054 — Criterio D3 y Brecha B10 cerrados: el tipo de cambio de Perú ya existe (el usado para pedidos no-USD), no se requiere fuente nueva. (4) DUDA-018, DUDA-041, DUDA-042 y DUDA-043 verificadas: ya estaban correctamente reflejadas en el documento, sin cambios de contenido. |
| 6   | 2026-09-11 | Cierre de dudas resueltas / Correcciones de consistencia / Retiro de la proforma del flujo (Tramitar) / Ajuste por el retiro del timbrado de Perú | Pasada integral de limpieza y consistencia: (1) Historia de Usuario y Requisito Funcional depurados de lenguaje de UI ("botón", "presionar Tramitar") y de "persiste"→"almacena"; Alcance/No aplica a reescrito para reflejar que Detracciones y Percepciones SUNAT quedan sin objeto (no se emiten comprobantes fiscales desde el sistema para Perú), cerrando así Brecha B5 en su totalidad. (2) Reglas 3, 5–8, 10 y 12 limpiadas de lenguaje de UI y de persistencia en BD; Regla 10 y Criterio A2 fijan el disclaimer SUNAT como texto final aprobado (cierra Brecha B4). (3) Eliminado el Riesgo de validación del disclaimer y el Riesgo de Percepciones SUNAT (ya cubierto por el cambio de Alcance); renumerado el Riesgo de tipo de cambio a Riesgo 1. (4) Criterio D5: leyenda de pago fijada a texto final "OPERACIÓN AL CONTADO" (ya no template propuesto). (5) Criterio E1/E2, F1 y G1 limpiados de lenguaje de UI/BD y de pendientes ya cerrados. (6) Sección H renombrada "Información legal de Golocaer S.A.C."; Criterio H5 reescrito para establecer que el documento NO incluye logos de catálogos farmacéuticos ni de marcas (cierra Brecha B7); Criterios H1–H3 se dejan reescritos pero con sus brechas (B3, B6) explícitamente abiertas, a la espera de datos reales del cliente. (7) Sección J renombrada "Almacenamiento y consulta post-envío" y sus criterios limpiados de lenguaje de UI/BD/pendiente Tramitar Pedido. (8) Notas Adicionales reescritas por completo, retirando los bullets que reproducían reglas y criterios ya documentados; sección de Brechas condensada a los tres pendientes vigentes (B1 datos DML, B3 contacto/dirección legal Perú, B6 certificaciones), con las brechas B2, B4, B5, B7, B8, B9 y B10 documentadas como resueltas. Nota: los bloques de instrucción sobre "configuración fiscal de producto", "retiro de Región Perú del alcance" (roles Analista de Cuentas por Cobrar / Gestor de Cobranza, aprobación de exportaciones) y "ordenamiento por columna" no se aplicaron a este documento por no corresponder a su alcance (Proforma Perú); quedan pendientes de reasignación al requisito correcto, probablemente FU-018. |
