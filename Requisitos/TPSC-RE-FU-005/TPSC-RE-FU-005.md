# Mantenimiento de Catálogo de Clientes — Configuración de Cobros y Facturación

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-005 |
| **Nombre** | Mantenimiento de Catálogo de Clientes |
| **Catálogo** | Catálogo de Clientes |
| **Categoría** | Funcional |
| **Estatus** | Propuesto |
| **Referencia Legacy** | Sin trazabilidad directa a la matriz original del cliente; emergente de sesiones y análisis fiscal Perú |

---

## Historia de Usuario

> Yo como usuario con acceso a la cartera de clientes, quiero capturar y mantener actualizada la configuración de cobros y facturación del cliente en el Catálogo de Clientes, contemplando las particularidades fiscales de México (SAT) y Perú (SUNAT), para que esos valores se utilicen como configuración default al generar proformas, facturas y comprobantes fiscales y se cumpla la normativa fiscal aplicable según la región del cliente.

---

## Requisito

El sistema debe contar en la sección **Cobros** del Catálogo de Clientes con los campos de configuración necesarios para que las proformas y facturas se emitan correctamente según la **Región** del cliente.

- Para clientes de **México** el sistema debe presentar los campos **Forma de Pago**, **Uso de CFDI** y **Método de Pago** (mantiene la mecánica preexistente del sistema).
- Para clientes de **Perú** el sistema debe presentar los campos **Condición de Pago** con catálogo SUNAT (Contado/Crédito conforme Resolución de Superintendencia N° 193-2020/SUNAT) y **Tipo de Comprobante** con catálogo SUNAT (Factura electrónica, Boleta de venta electrónica, Recibo por Honorarios electrónico), modelado como campo independiente del Uso de CFDI mexicano.
- Adicionalmente, para Perú el sistema debe contemplar dos banderas asociadas a mecanismos tributarios SUNAT: **Agente de Retención IGV** y **Sujeto a Detracción**, sujetas a confirmación de aplicabilidad con el cliente.

Los valores capturados funcionan como configuración default consumida por los módulos **Factura por Adelantado** y **Validar Cobro**. Cualquier usuario con acceso a la cartera del cliente puede modificar los campos. La habilitación efectiva de la facturación electrónica Perú depende adicionalmente de capacidades a nivel sistema y catálogo de productos que no son alcance de este requisito.

---

## Alcance

### Aplica a

- Clientes de México y Perú en el Catálogo de Clientes.
- Sección **Cobros** dentro de la pantalla del cliente.
- Para **Región México**: tres campos (Forma de Pago, Uso de CFDI, Método de Pago) con los catálogos preexistentes del sistema.
- Para **Región Perú**: campos Condición de Pago (Contado/Crédito) y Tipo de Comprobante, con catálogos SUNAT. Comportamiento nuevo en R16.
- **Dos banderas tributarias para Región Perú, sujetas a confirmación de aplicabilidad con el cliente: Agente de Retención IGV (Sí/No) y Sujeto a Detracción (Sí/No con tasa cuando aplique).**
- Validación de obligatoriedad de los campos al guardar el cliente, según los aplicables por Región.
- Acceso libre a la edición por cualquier usuario con visibilidad sobre el cliente.
- Provisión de valores default consumidos por los módulos Factura por Adelantado y Validar Cobro al generar documentos fiscales.

### No aplica a

- Campo Forma de Pago (medio de pago) para Región Perú: la normativa SUNAT no exige declarar el medio de pago específico en el comprobante.
- Habilitación a nivel sistema y catálogo de productos requerida para emitir facturación electrónica SUNAT timbrada (datos fiscales del producto, código SUNAT del producto, tipo de afectación al IGV, unidad de medida SUNAT, integración con Operador de Servicios Electrónicos o sistema SEE-SOL de SUNAT, certificado digital del emisor Perú, Guía de Remisión Electrónica para despacho de mercancía). Estas son brechas reconocidas que se documentan en este requisito a nivel observación pero no son alcance de la presente fila.
- Otros campos de la sección Cobros distintos de los mencionados.

---

## Reglas de Negocio

**Regla 1 — Campos como configuración default del cliente**
Los campos de la sección Cobros funcionan como configuración default del cliente. Estos valores se aplican automáticamente al generar proformas y facturas, salvo que se editen por operación en los módulos donde se permite.

**Regla 2 — Catálogos de la sección Cobros diferenciados por Región**
Los campos y catálogos de la sección Cobros se presentan en función de la Región del cliente. Para México se presentan Forma de Pago, Uso de CFDI y Método de Pago con los catálogos preexistentes del sistema. Para Perú se presentan Condición de Pago y Tipo de Comprobante con catálogos SUNAT, más las banderas tributarias aplicables.

**Regla 3 — Condición de Pago como dimensión temporal del pago (Perú)**
Para clientes Perú, el campo Condición de Pago (Contado/Crédito) expresa la dimensión temporal del pago conforme a la normativa SUNAT (Resolución de Superintendencia N° 193-2020/SUNAT). Es el equivalente conceptual del Método de Pago mexicano (PUE/PPD): Contado equivale a pago en una exhibición y Crédito a pago diferido.

**Regla 4 — Método de Pago aplicable solo a México**
Para clientes México, el Método de Pago se captura con el catálogo SAT de dos opciones: PUE (Pago en Una Exhibición) y PPD (Pago en Parcialidades o Diferido). Para clientes Perú este campo no se renderiza; la dimensión temporal del pago queda capturada en el campo Condición de Pago.

**Regla 5 — Forma de Pago (medio) aplicable solo a México**
Para clientes México, la Forma de Pago expresa el medio de pago con el catálogo preexistente del sistema. Para clientes Perú no se captura el medio de pago específico, ya que la normativa SUNAT no lo exige en el comprobante.

**Regla 6 — Uso de CFDI y Tipo de Comprobante como campos independientes**
El Uso de CFDI (México) y el Tipo de Comprobante (Perú) son conceptos fiscales distintos y se modelan como campos independientes con catálogos separados. El Uso de CFDI indica el uso fiscal que el receptor dará al comprobante; el Tipo de Comprobante indica la clase de documento emitido (Factura, Boleta o Recibo por Honorarios). Cada campo se renderiza según la Región del cliente.

**Regla 7 — Agente de Retención IGV (Perú, sujeto a confirmación)**
Para clientes Perú, el sistema contempla la bandera **Agente de Retención IGV** (Sí/No) que indica si el cliente está designado por SUNAT como Agente de Retención. Cuando el valor es Sí, las facturas emitidas a ese cliente deben contemplar la retención del IGV vigente (3%) y consignar la leyenda correspondiente; la lógica de cálculo y emisión vive en el módulo de facturación. La aplicabilidad de esta bandera a la cartera de PROQUIFA Perú está sujeta a confirmación con el cliente.

**Regla 8 — Sujeto a Detracción (Perú, sujeto a confirmación)**
Para clientes Perú, el sistema contempla la bandera **Sujeto a Detracción** (Sí/No). La detracción (SPOT) aplica por bien o servicio según los anexos de la R.S. 183-2004/SUNAT y solo a operaciones mayores a S/ 700; la ejecuta el comprador. La tasa aplicable se determina por el producto o servicio, no por el cliente. La aplicabilidad de esta bandera a los productos de PROQUIFA Perú está sujeta a confirmación con el cliente; la lógica de cálculo y emisión vive en el módulo de facturación.

**Regla 9 — Edición sin restricción de rol**
Cualquier usuario con acceso a la cartera del cliente puede modificar los campos de la sección Cobros. La autorización proviene del acceso del usuario al cliente, no de un rol específico.

---

## Riesgos

**Riesgo 1 — Confusión por nomenclatura fiscal distinta entre México y Perú**
Los campos Uso de CFDI (México) y Tipo de Comprobante (Perú) son conceptos fiscalmente distintos pero ocupan posiciones equivalentes en la sección Cobros. Igualmente, el Método de Pago (México) y la Condición de Pago (Perú) expresan la misma dimensión temporal con nomenclatura distinta. Esto puede generar confusión en usuarios que operen clientes de ambos países.

**Riesgo 2 — Catálogos paramétricos desactualizados respecto a la normativa**
Los catálogos de México (Forma de Pago, Uso de CFDI, Método de Pago) y SUNAT (Condición de Pago, Tipo de Comprobante) se actualizan periódicamente. Si no se mantienen sincronizados, los clientes podrían quedar con valores obsoletos que causen rechazo de timbrado.

**Riesgo 3 — Brechas a nivel sistema y catálogo de productos para facturación electrónica Perú**
La facturación electrónica SUNAT requiere capacidades que no están cubiertas a nivel sistema ni en el catálogo de productos actual de PROQUIFA. Aunque el catálogo de clientes capture la información fiscal correcta del cliente Perú, la emisión efectiva de un comprobante electrónico SUNAT timbrado depende de elementos adicionales que son brechas reconocidas en este punto del proyecto y que se detallan en las Observaciones.

**Riesgo 4 — Detracciones y retenciones calculadas incorrectamente**
La aplicación de detracciones (SPOT) y retenciones (Agente de Retención IGV) en la factura depende de que las banderas correspondientes estén correctamente capturadas y de que las reglas de cálculo se implementen correctamente en el módulo de facturación. Si se capturan mal, o si las reglas no se implementan, la factura puede emitirse con monto incorrecto generando problemas fiscales para PROQUIFA y para el cliente.

**Riesgo 5 — Productos sujetos a Detracción no identificados en el catálogo de productos**
La detracción aplica por producto/servicio (no por cliente). Un cliente podría estar marcado como Sujeto a Detracción cuando los productos que adquiere no aplican, o viceversa. Sin que el catálogo de productos identifique cada producto con su tasa de detracción aplicable, la factura puede calcular incorrectamente.

---

## Criterios de Aceptación

### SECCIÓN A — Visualización de los campos según Región

**Criterio A1 — Visualización de los campos de Cobros según la Región del cliente**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta un cliente,
- **Cuando** se renderiza la sección Cobros,
- **Entonces** el sistema deberá presentar para Región México los campos Forma de Pago, Uso de CFDI y Método de Pago; y para Región Perú los campos Condición de Pago, Tipo de Comprobante y las banderas tributarias aplicables (Agente de Retención IGV y Sujeto a Detracción). Los campos no aplicables a la Región no se renderizan.

---

### SECCIÓN B — Campos México

**Criterio B1 — Selector de Forma de Pago con catálogo del sistema**
- **Dado** que el cliente tiene Región = México,
- **Cuando** el usuario despliega el selector de Forma de Pago,
- **Entonces** el sistema deberá presentar el catálogo de Forma de Pago cargado en el sistema: Cheque, Depósito bancario, Efectivo, Tarjeta, Transferencia, Otros, Na y —ninguno—.

**Criterio B2 — Selector de Uso de CFDI con catálogo del sistema**
- **Dado** que el cliente tiene Región = México,
- **Cuando** el usuario despliega el selector de Uso de CFDI,
- **Entonces** el sistema deberá presentar el catálogo de Uso de CFDI cargado en el sistema: G01 Adquisición de mercancías, G02 Devoluciones, descuentos o bonificaciones, G03 Gastos en general, S01 Sin efectos fiscales, Por definir y N/A.

**Criterio B3 — Selector de Método de Pago con catálogo SAT**
- **Dado** que el cliente tiene Región = México,
- **Cuando** el usuario despliega el selector de Método de Pago,
- **Entonces** el sistema deberá presentar el catálogo SAT con dos opciones: PUE (Pago en Una Exhibición) y PPD (Pago en Parcialidades o Diferido).

---

### SECCIÓN C — Campos Perú (Catálogos SUNAT)

**Criterio C1 — Selector de Condición de Pago con catálogo SUNAT**
- **Dado** que el cliente tiene Región = Perú,
- **Cuando** el usuario despliega el selector de Condición de Pago,
- **Entonces** el sistema deberá presentar el catálogo SUNAT con dos opciones: Contado y Crédito (conforme Resolución de Superintendencia N° 193-2020/SUNAT).

**Criterio C2 — Selector de Tipo de Comprobante con catálogo SUNAT**
- **Dado** que el cliente tiene Región = Perú,
- **Cuando** el usuario despliega el selector de Tipo de Comprobante,
- **Entonces** el sistema deberá presentar el catálogo SUNAT con las opciones: Factura electrónica, Boleta de venta electrónica y Recibo por Honorarios electrónico. Este campo es independiente del Uso de CFDI mexicano y tiene su propio catálogo.

**Criterio C3 — Método de Pago y Forma de Pago no renderizados para Perú**
- **Dado** que el cliente tiene Región = Perú,
- **Cuando** se renderiza la sección Cobros,
- **Entonces** los campos Método de Pago y Forma de Pago (medio) no deben aparecer en la pantalla. La dimensión temporal del pago queda capturada en el campo Condición de Pago.

---

### SECCIÓN D — Banderas Tributarias Perú

**Criterio D1 — Bandera Agente de Retención IGV**
- **Dado** que el cliente tiene Región = Perú,
- **Cuando** el usuario consulta la sección Cobros,
- **Entonces** el sistema deberá presentar la bandera **Agente de Retención IGV** con opciones Sí y No (default propuesto: No). Si el valor es Sí, el sistema marca al cliente para que los módulos de facturación apliquen la retención del IGV vigente al emitir facturas.

> **⚠️ Pendiente** — La aplicabilidad de esta bandera está sujeta a confirmación con el cliente sobre si su cartera Perú incluye clientes designados Agente de Retención por SUNAT. Si no los tiene, la bandera es innecesaria y se evita su desarrollo.

**Criterio D2 — Bandera Sujeto a Detracción**
- **Dado** que el cliente tiene Región = Perú,
- **Cuando** el usuario consulta la sección Cobros,
- **Entonces** el sistema deberá presentar la bandera **Sujeto a Detracción** con opciones Sí y No (default propuesto: No). Si el valor es Sí, se habilita la captura de la tasa de detracción aplicable.

> **⚠️ Pendiente** — La detracción aplica por bien o servicio según los anexos SUNAT, no por cliente, y solo a operaciones mayores a S/ 700. Pendiente confirmar aplicabilidad a los productos de PROQUIFA Perú, así como si la tasa se captura a nivel cliente, a nivel producto o se determina desde el catálogo de productos.

---

### SECCIÓN E — Obligatoriedad y Persistencia

**Criterio E1 — Obligatoriedad al guardar (Región México)**
- **Dado** que el cliente tiene Región = México,
- **Cuando** el usuario intenta guardar los datos del cliente,
- **Entonces** el sistema deberá validar que los tres campos de Cobros (Forma de Pago, Uso de CFDI, Método de Pago) estén capturados. Si alguno está vacío, el guardado se bloquea.

**Criterio E2 — Obligatoriedad al guardar (Región Perú)**
- **Dado** que el cliente tiene Región = Perú,
- **Cuando** el usuario intenta guardar los datos del cliente,
- **Entonces** el sistema deberá validar que estén capturados los campos Condición de Pago y Tipo de Comprobante, así como las banderas tributarias que se confirmen como aplicables. Si alguno de los campos obligatorios está vacío, el guardado se bloquea.

**Criterio E3 — Edición sin restricción de rol**
- **Dado** que cualquier usuario con acceso al cliente abre la sección Cobros,
- **Cuando** intenta modificar los campos,
- **Entonces** el sistema deberá permitir la edición sin requerir un rol específico.

**Criterio E4 — Persistencia de los valores como configuración default**
- **Dado** que el usuario guarda exitosamente los campos de Cobros del cliente,
- **Cuando** el sistema procesa la operación,
- **Entonces** deberá almacenar los valores asociados al cliente y dejarlos disponibles como configuración default para módulos posteriores.

---

## Notas de Implementación

- Funcionalidad ubicada en la sección **Cobros** del cliente dentro del Catálogo de Clientes.
- Los campos para clientes México (Forma de Pago, Uso de CFDI, Método de Pago) ya existen en el sistema actual. R16 mantiene esa funcionalidad preexistente.
- Los campos para clientes Perú son nuevos en R16: Condición de Pago SUNAT (Contado/Crédito), Tipo de Comprobante SUNAT (Factura/Boleta/Recibo por Honorarios). **Pendiente de confirmación:** las banderas Agente de Retención IGV y Sujeto a Detracción (sujetas a validación de aplicabilidad con el cliente antes de desarrollarse).
- El campo de dimensión temporal del pago para Perú se denomina **Condición de Pago** (Contado/Crédito) conforme a la Resolución de Superintendencia N° 193-2020/SUNAT. Es el equivalente conceptual del Método de Pago mexicano (PUE/PPD). Para Perú no se captura el medio de pago específico (Forma de Pago mexicana) porque la normativa SUNAT no lo exige en el comprobante.
- El Uso de CFDI (México) y el Tipo de Comprobante (Perú) se modelan como **campos independientes** con catálogos separados. No comparten campo en pantalla ni en base de datos, para evitar complicaciones al timbrar.
- Catálogo actual de Forma de Pago en PQF2 (México): Cheque, Depósito bancario, Efectivo, Tarjeta, Transferencia, Otros, Na, —ninguno—. Es un catálogo simplificado propio del sistema que no emplea la nomenclatura ni las claves del catálogo SAT c_FormaPago.
- Catálogo actual de Uso de CFDI en PQF2 (México): G01 Adquisición de mercancías, G02 Devoluciones/descuentos/bonificaciones, G03 Gastos en general, S01 Sin efectos fiscales, Por definir, N/A. Mantener el catálogo preexistente salvo indicación contraria del cliente.
- Cualquier usuario con acceso a la cartera del cliente puede modificar los campos de la sección Cobros. No existe restricción de rol específica.
- Para el detalle de los campos de la sección Cobros por Región, los catálogos y el análisis de los mecanismos tributarios peruanos, ver archivo adjunto `TPSC-RE-FU-005_Equivalencias_Cobros_MX_PE.xlsx`.

> **⚠️ Duda** — Campo "Tipo de Revisión" (Digital / Física / Híbrida) en la sección de configuración fiscal del cliente: el campo existe en la pantalla actual de México (PQF2) pero no está documentado en ningún requisito de la matriz. Pendiente confirmar con el cliente: (1) si este campo entra dentro del alcance de R16 o queda fuera; (2) qué representa funcionalmente y qué reglas tiene; (3) si aplica igual a Perú.

> **⚠️ Pendiente** — Confirmar con el cliente la denominación final del campo Condición de Pago para Perú.

> **⚠️ Pendiente** — Validar si se requiere un mapeo de las opciones de Forma de Pago del sistema a las claves del catálogo SAT c_FormaPago para el timbrado del CFDI, dado que el XML del CFDI exige la clave SAT correspondiente.

> **⚠️ Pendiente** — Confirmar con el cliente si su cartera Perú incluye clientes designados Agente de Retención del IGV por SUNAT. Si no los tiene, la bandera Agente de Retención IGV es innecesaria y se evita su desarrollo.

> **⚠️ Pendiente** — Confirmar con el cliente si los productos o servicios de PROQUIFA Perú están sujetos a Detracción (SPOT) según los anexos de la R.S. 183-2004/SUNAT. Pendiente además definir si la tasa se captura a nivel cliente, a nivel producto o se determina desde el catálogo de productos.

> **⚠️ Pendiente** — Confirmar si PROQUIFA Perú (Golocaer) está designada por SUNAT como Agente de Percepción del IGV. La percepción es una condición del emisor (no del cliente); de aplicar, se configura a nivel emisor (ver Brecha 4).

---

## Brechas Reconocidas — Facturación Electrónica Perú

Este requisito captura en el catálogo del cliente la información fiscal necesaria del receptor del documento. Sin embargo, la emisión efectiva de una factura electrónica SUNAT timbrada requiere capacidades adicionales a nivel sistema y catálogo de productos que no están cubiertas en el alcance actual. Estas brechas se gestionan como Pendientes formales en el proyecto.

**Brecha 1 — Datos fiscales SUNAT en el catálogo de productos**
Para emitir factura electrónica SUNAT cada producto/servicio requiere campos fiscales propios del estándar SUNAT que no existen en el catálogo de productos actual: código SUNAT del producto, unidad de medida según catálogo SUNAT (KGM, MTR, LTR, NIU, etc.) y tipo de afectación al IGV por línea (gravado, exonerado, inafecto, exportación, gratuito, etc.). Son obligatorios en el XML UBL 2.1; sin ellos SUNAT rechaza la emisión. Estos campos no aplican a México porque el modelo SAT no clasifica los productos por código SUNAT ni por afectación al IGV a nivel línea de comprobante.

> **⚠️ Pendiente** — Patrón observado en una factura real de Golocaer (un solo caso, NO confirmado como general): Golocaer resuelve estos datos con valores genéricos únicos — un código de producto SUNAT genérico ("41116107") para todas las líneas, unidad "PIEZAS" (código C62) y afectación al IGV "gravado" (18%). Pendiente validar con el cliente si este patrón genérico aplica siempre y, de ser así, si la solución para PQF2 es replicarlo en lugar de cargar datos SUNAT por producto.

**Brecha 2 — Guía de Remisión Electrónica para despacho de mercancía**
Cuando PROQUIFA Perú despacha mercancía física al cliente, SUNAT requiere emitir la Guía de Remisión Electrónica (GRE) que acompaña al transporte. La GRE involucra datos del transportista, del vehículo, de la ruta y del receptor. *Por qué es necesario:* PROQUIFA Perú despacha mercancía (confirmado), por lo que cada operación de venta con entrega física requiere GRE además de la factura. *Por qué se diferencia de México:* SAT no exige el equivalente desde este flujo.

**Brecha 3 — Tipo de Operación SUNAT (Catálogo 51) por factura**
Cada factura electrónica Perú debe consignar un código de Tipo de Operación que identifica el contexto comercial (venta interna, exportación, anticipos, operación gratuita, etc.). Este código se captura por operación al emitir la factura, no en el catálogo del cliente. *Por qué es necesario:* campo obligatorio del XML UBL 2.1; SUNAT rechaza la emisión si falta o es incorrecto. *Por qué se diferencia de México:* SAT no tiene un campo equivalente; el contexto comercial se infiere de la combinación de otros campos del CFDI.

**Brecha 4 — Régimen de Percepción del IGV de PROQUIFA como Agente de Percepción**
Si PROQUIFA Perú está designada por SUNAT como Agente de Percepción, debe cobrar al cliente un porcentaje adicional al IGV (típicamente 2%) por ciertas operaciones. Es una condición del emisor (PROQUIFA), no del cliente. *Por qué es necesario:* si aplica, la factura debe emitirse considerando la percepción; omitirla genera contingencia fiscal para PROQUIFA. *Por qué se diferencia de México:* SAT no tiene un régimen equivalente.

> **⚠️ Pendiente** — Confirmar si PROQUIFA Perú es Agente de Percepción designado por SUNAT.

**Brecha 5 — Configuración del emisor PROQUIFA Perú para facturación electrónica SUNAT**
La emisión electrónica SUNAT requiere a nivel sistema: certificado digital vigente del emisor PROQUIFA Perú, designación como emisor electrónico ante SUNAT, integración con SUNAT (vía Operador de Servicios Electrónicos OSE o vía SEE-SOL portal SUNAT) para envío de XMLs y recepción de Constancias de Recepción CDR, manejo de Resúmenes Diarios para boletas según volumen, y manejo de Comunicaciones de Baja para anulaciones. *Por qué es necesario:* sin esta infraestructura no se puede emitir un comprobante electrónico válido en Perú — es la brecha mayor bloqueante. *Por qué se diferencia de México:* en México la infraestructura está cubierta por la integración existente con TurboPac como PAC; en Perú se requiere infraestructura equivalente pero distinta (OSE o SEE-SOL).

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-06-10 | OBS-009 | Brechas de facturación electrónica Perú confirmadas como ya cubiertas. Se amplía el detalle de cada brecha con contexto "Por qué es necesario" / "Por qué se diferencia de México". Brecha 1: se agrega el patrón genérico observado en factura real de Golocaer (pendiente de validación). Notas: banderas tributarias marcadas como pendientes de confirmación; nueva duda sobre campo "Tipo de Revisión"; referencia al archivo adjunto de equivalencias MX-PE. |
