# Mantenimiento de Catálogo de Clientes — Actualización de catálogos Forma de Pago y Uso de CFDI

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-005 |
| **Nombre** | Actualización de catálogos Forma de Pago y Uso de CFDI |
| **Catálogo** | Catálogo de Clientes |
| **Categoría** | Funcional |
| **Estatus** | Propuesto |
| **Referencia Legacy** | Sin trazabilidad directa a la matriz original del cliente; emergente de sesiones |

---

## Historia de Usuario

> Yo como **usuario con acceso a la cartera de clientes**, quiero que los catálogos de opciones de **Forma de Pago** y **Uso de CFDI** estén actualizados y regionalizados según corresponda, para que al capturar o consultar la configuración de cobros del cliente los selectores muestren la lista vigente confirmada por el cliente con los formatos y reglas de obligatoriedad correctos por región.

---

## Requisito

El sistema debe **actualizar los catálogos de opciones** de los campos **Forma de Pago** y **Uso de CFDI** utilizados en la sección **Cobros** del Catálogo de Clientes. La actualización comprende **altas**, **bajas** y **modificaciones** de opciones, ejecutadas directamente en base de datos con base en el listado consolidado en el documento *R16 - Catálogos Fiscales*. El sistema únicamente despliega el listado resultante en los selectores correspondientes.

**Formato de despliegue:**
- **Forma de Pago:** clave y descripción de la opción, separadas por guion (ej. `03 - Transferencia electrónica de fondos`), reflejando la clave del comprobante fiscal.
- **Uso de CFDI:** descripción de la opción, sin guion ni clave adicional.

**Regionalización:**
- **Forma de Pago** se regionaliza: catálogo propio para **México** (claves SAT) y catálogo propio para **Perú**. El campo es **obligatorio para Región México** y **no obligatorio para Región Perú**.
- **Uso de CFDI** se despliega **sin diferenciación por región** — la misma lista aplica a todos los clientes.

Los campos de la sección Cobros ya existen en el sistema; este release cubre exclusivamente la actualización de los catálogos de opciones y las reglas de regionalización/obligatoriedad. No se implementa interfaz gráfica para administrar los catálogos: la gestión se ejecuta directamente en base de datos.

---

## Alcance

### Aplica a

- **Actualización del catálogo de Forma de Pago** utilizado en el campo Forma de Pago de la sección Cobros del Catálogo de Clientes.
- **Actualización del catálogo de Uso de CFDI** utilizado en el campo Uso de CFDI de la misma sección.
- **Tipos de actualización:** altas, bajas y modificaciones de opciones ejecutadas directamente en base de datos.
- **Formato de despliegue:**
  - Forma de Pago: `Clave - Descripción` (con guion como separador, reflejando la clave del comprobante fiscal).
  - Uso de CFDI: descripción de la opción (sin guion ni clave adicional).
- **Regionalización del catálogo de Forma de Pago:** listas propias para México (claves SAT) y para Perú.
- **Obligatoriedad del campo Forma de Pago por región:** obligatorio para clientes de Región México; no obligatorio para clientes de Región Perú.
- **Despliegue único del catálogo de Uso de CFDI** — la misma lista se muestra a todos los clientes, sin diferenciación por región.
- **Origen de la actualización:** documento *R16 - Catálogos Fiscales* consolidado por el cliente.

### No aplica a

- **Región Perú fuera del catálogo de Forma de Pago:** no se implementan campos, catálogos ni banderas tributarias exclusivos de Perú (Condición de Pago, Tipo de Comprobante, Agente de Retención de IGV, Sujeto a Detracción) en este release.
- **Regionalización del catálogo de Uso de CFDI:** la lista es única para todos los clientes.
- **Campos preexistentes de la sección Cobros:** ya existen en el sistema y no se modifican en este release; el requisito cubre exclusivamente la actualización de los catálogos.
- **Interfaz gráfica para administrar los catálogos** desde la aplicación: la gestión se ejecuta directamente en base de datos.
- **Tratamiento de clientes con valores dados de baja del catálogo:** el cliente entrega el listado depurado y ejecuta la curaduría (ver Regla 9 y Riesgo 2).

---

## Reglas de Negocio

**Regla 1 — Alcance de la actualización**
La actualización aplica exclusivamente a los catálogos de **Forma de Pago** y **Uso de CFDI** utilizados en la sección Cobros del Catálogo de Clientes. Ningún otro catálogo se modifica en este requisito.

**Regla 2 — Gestión del catálogo sin interfaz gráfica**
La gestión del catálogo (altas, bajas y modificaciones de opciones) no dispone de interfaz gráfica de usuario en R16. La gestión es responsabilidad del área de Soporte a la Producción (equipo SAP) mediante acceso directo a la base de datos del sistema.

**Regla 3 — Tipos de actualización**
Los tipos de actualización comprenden altas, bajas y modificaciones de opciones del catálogo, ejecutadas directamente en base de datos con base en el listado entregado por el cliente.

**Regla 4 — Formato de despliegue del Uso de CFDI**
El catálogo de Uso de CFDI se despliega en los selectores mostrando únicamente la **descripción** de la opción, sin clave ni guion.

**Regla 5 — Regionalización del catálogo de Forma de Pago**
El catálogo de Forma de Pago se regionaliza: existe una lista propia para clientes de **Región México** (alineada a las claves SAT) y una lista propia para clientes de **Región Perú**. El sistema muestra la lista correspondiente a la Región del cliente al desplegar el selector.

**Regla 6 — Obligatoriedad del campo Forma de Pago por región**
El campo Forma de Pago es **obligatorio** para clientes de **Región México** — el sistema no permite guardar la configuración de cobros del cliente sin un valor seleccionado. Para clientes de **Región Perú** el campo **no es obligatorio** — el sistema permite guardar la configuración sin selección.

**Regla 7 — Formato de despliegue de la Forma de Pago con la clave del comprobante fiscal**
Cada opción del catálogo de Forma de Pago se despliega en el selector con el formato `Clave - Descripción` separado por guion, donde la clave corresponde a la del comprobante fiscal de la región (SAT para México, catálogo SUNAT para Perú).

**Regla 8 — Origen de la actualización**
La lista de opciones a agregar, dar de baja o modificar proviene del documento *R16 - Catálogos Fiscales* consolidado por el cliente. Ese documento es la fuente única de la actualización.

**Regla 9 — Tratamiento de clientes afectados por bajas**
Cuando la actualización incluye bajas de opciones que hoy están asignadas a clientes existentes, el cliente ejecuta una **curaduría** para reasignar esos clientes a las opciones vigentes. La curaduría es responsabilidad del cliente y se coordina con el equipo de Soporte a la Producción.

---

## Riesgos

**Riesgo 1 — Catálogos desactualizados frente al SAT**
El catálogo de Forma de Pago para México se alinea al catálogo publicado por el SAT. Si el SAT actualiza su catálogo después del despliegue de R16 (nuevas opciones, cambios de descripción o retiros), el catálogo interno quedará desalineado hasta que se ejecute una nueva actualización manual. **Mitigación:** protocolo de revisión periódica del catálogo SAT vigente por parte del área fiscal de PROQUIFA.

**Riesgo 2 — Clientes con opciones dadas de baja del catálogo**
Cuando la actualización incluye bajas, los clientes que hoy tienen asignadas opciones que serán dadas de baja quedan con un valor inválido tras la actualización si no se ejecuta la curaduría previamente. **Resuelto:** el cliente entrega el listado depurado y ejecuta la curaduría reasignando esos clientes a las opciones vigentes antes de la baja (ver Regla 9).

---

## Criterios de Aceptación

### Sección A — Despliegue de los catálogos actualizados

**Criterio A1 — Alta, baja o modificación reflejada en el selector**
- **Dado** que se ejecutó una alta, baja o modificación en el catálogo directamente en base de datos,
- **Cuando** un usuario despliega el selector correspondiente en el sistema,
- **Entonces** deberá ver la lista vigente resultante de la operación (opciones nuevas visibles, opciones dadas de baja fuera del selector, descripciones o claves modificadas actualizadas).

**Criterio A2 — Selector de Forma de Pago regionalizado**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta la sección Cobros de un cliente,
- **Cuando** despliega el selector de Forma de Pago,
- **Entonces** el sistema deberá mostrar la lista correspondiente a la **Región del cliente** (catálogo México si el cliente es MEX; catálogo Perú si el cliente es PER), cada opción con el formato `Clave - Descripción` separado por guion.

**Criterio A3 — Selector de Uso de CFDI con despliegue único**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta la sección Cobros de un cliente,
- **Cuando** despliega el selector de Uso de CFDI,
- **Entonces** el sistema deberá mostrar la lista completa del catálogo vigente, cada opción únicamente con su **descripción**, sin diferenciar por región.

**Criterio A4 — Bajas reflejadas en el selector**
- **Dado** que se dio de baja una opción del catálogo directamente en base de datos,
- **Cuando** un usuario despliega el selector correspondiente,
- **Entonces** el sistema **no deberá mostrar** la opción dada de baja en la lista. Los clientes que ya la tenían asignada mantienen el valor a nivel de datos hasta que se ejecute la curaduría.

### Sección B — Obligatoriedad del campo Forma de Pago

**Criterio B1 — Forma de Pago obligatoria para Región México**
- **Dado** que un usuario captura la configuración de cobros de un cliente de **Región México**,
- **Cuando** intenta guardar sin seleccionar Forma de Pago,
- **Entonces** el sistema **no deberá permitir el guardado** y deberá notificar que Forma de Pago es un campo obligatorio.

**Criterio B2 — Forma de Pago no obligatoria para Región Perú**
- **Dado** que un usuario captura la configuración de cobros de un cliente de **Región Perú**,
- **Cuando** intenta guardar sin seleccionar Forma de Pago,
- **Entonces** el sistema **deberá permitir el guardado** sin exigir valor en el campo.

---

## Notas de Implementación

- Este requisito cubre exclusivamente la actualización de los catálogos de **Forma de Pago** y **Uso de CFDI** utilizados en la sección Cobros del Catálogo de Clientes. Los campos ya existen en el sistema y no se modifican.
- El área responsable de ejecutar las altas, bajas y modificaciones en base de datos es **Soporte a la Producción (equipo SAP)** de PROQUIFA.
- La fuente única del listado a aplicar es el documento *R16 - Catálogos Fiscales* consolidado por el cliente.
- El catálogo de **Forma de Pago** se regionaliza (México y Perú tienen listas propias). El de **Uso de CFDI** se despliega sin diferenciación por región.
- La **obligatoriedad** del campo Forma de Pago se determina por la Región del cliente: obligatorio para México, no obligatorio para Perú.
- Cuando la actualización incluye bajas, la curaduría de clientes que tienen la opción asignada se coordina con el cliente antes de ejecutar la baja en base de datos.

**Resueltos (dudas cerradas):**
- **Mapeo del catálogo de Forma de Pago a las claves del SAT:** cerrado con la actualización del catálogo al formato `Clave - Descripción` (Regla 7).
- **Campo "Tipo de Revisión" (DUDA-013):** no tiene impacto en el alcance de R16 porque R16 opera bajo esquema **prepago** (proforma → pago → factura por adelantado), sin el paso de aceptación de factura a crédito al que corresponde este campo. Es obligatorio en el catálogo de clientes pero no se considera en el flujo de prepago; no impacta ni en México ni en Perú.
- **Bandera Sujeto a Detracción / SPOT (DUDA-009):** no aplica Detracción SPOT para la operación de PROQUIFA — sin objeto al retirarse Perú del alcance de las banderas tributarias.
- **Bandera Agente de Retención de IGV (DUDA-008), Agente de Percepción (DUDA-010) y denominación del campo Condición de Pago (DUDA-012):** descartadas; no se desarrollan en el contexto de la cancelación de la facturación de Región Perú.
- **Catálogos fiscales específicos por producto para Perú — código de producto SUNAT, unidad, IGV, TipoAfectacionIGV, Tipo de Operación (DUDA-014):** descartados por la cancelación de facturación Perú (17/07); los catálogos fiscales vigentes son únicamente los de México.
- **Confirmación del cliente sobre las opciones de los catálogos:** cerrada con el documento *R16 - Catálogos Fiscales*.
- **Tratamiento de clientes con opciones dadas de baja:** cerrado con la Regla 9 (curaduría del cliente).

### Documentos de referencia del cliente

- Documento *R16 - Catálogos Fiscales* — listado consolidado de opciones vigentes de Forma de Pago (por región) y Uso de CFDI.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-07-31 | Retiro de Región Perú del alcance | Historia, Requisito, Alcance y Criterios de Aceptación reescritos completos retirando las particularidades fiscales de Perú (SUNAT, Condición de Pago, Tipo de Comprobante, Agente de Retención IGV, Sujeto a Detracción, presentación diferenciada por región). Se eliminan reglas de catálogos diferenciados por región, MetodoPago aplicable solo a México, banderas tributarias. Se eliminan riesgos de nomenclatura fiscal MEX/PER, brechas facturación electrónica Perú, detracciones/retenciones, productos sujetos a detracción. Se sustituyen las secciones A a E por una sección única de despliegue del catálogo actualizado. Se retira el bloque de brechas y todos los bullets relativos a Perú. |
| 2 | 2026-07-31 | Reenfoque del requisito | El requisito pasa de configuración de cobros y facturación del cliente a cubrir únicamente el despliegue del catálogo actualizado de Forma de Pago. Los campos de la sección Cobros ya existen y no se modifican. Se precisa que el alta/baja/modificación se ejecuta directamente en BD y en el sistema solo se despliega el listado resultante. Se agrega exclusión de interfaz gráfica en "No aplica a". Regla 2 de gestión sin UI. Reglas 1, 3–6 con alcance, formato, correspondencia con clave del comprobante y origen. Riesgo 2 nuevo (clientes con valores dados de baja). Criterios A3 y A4 redactados en función del reflejo en el sistema de altas y bajas. |
| 3 | 2026-07-31 | Cierres de dudas | Se cierra el pendiente sobre el mapeo del catálogo de Forma de Pago a las claves del SAT con la actualización al formato clave y concepto. Se cierra la duda del campo "Tipo de Revisión" (corresponde al flujo de crédito, sin impacto R16). Se cierra la bandera Sujeto a Detracción (SPOT no aplica a PROQUIFA). Se cierran los pendientes de bandera Agente de Retención de IGV, Agente de Percepción y denominación del campo Condición de Pago (sin objeto al retirarse Perú del alcance). |
| 4 | 2026-08-07 | Incorporación del catálogo Uso de CFDI | Historia, Requisito y Alcance: se incorpora el catálogo de opciones del campo Uso de CFDI, que se actualiza junto con el de Forma de Pago. Alcance "Aplica a" agrega el catálogo de Uso de CFDI y precisa los dos formatos de despliegue (con guion para Forma de Pago, sin guion para Uso de CFDI). Regla 4 nueva (formato Uso de CFDI). Criterio A3 nuevo (selector de Uso de CFDI). Observaciones incorporan los bullets correspondientes. |
| 5 | 2026-08-07 | Regionalización y obligatoriedad Forma de Pago | Historia, Requisito y Alcance: se incorpora la regionalización del catálogo de Forma de Pago (México y Perú con listas propias) y la obligatoriedad únicamente para clientes de Región México. Alcance "Aplica a" agrega los puntos de regionalización y obligatoriedad por región; acota el despliegue único al catálogo de Uso de CFDI. Regla 5 nueva (regionalización Forma de Pago). Regla 6 nueva (obligatoriedad por región). Criterio A2 nuevo (selector Forma de Pago regionalizado). Sección B nueva con B1 (Región México) y B2 (Región Perú). Observaciones agregan bullets de regionalización y obligatoriedad. |
| 6 | 2026-08-07 | Cierre del pendiente del listado de catálogos | Regla 8 cierra el pendiente de confirmación del cliente con el documento *R16 - Catálogos Fiscales*. Documentos de referencia del cliente lo incluyen con su enlace. Observaciones cierran el pendiente. Regla 9 nueva (tratamiento de clientes afectados mediante curaduría). |
| 7 | 2026-08-21 | Trazabilidad de dudas cerradas (batch "R16 - Dudas a Cliente") | Sección "Resueltos" actualizada para citar explícitamente DUDA-008 (Agente de Retención IGV, Descartada), DUDA-009 (Detracción SPOT, Resuelta — no aplica), DUDA-010 (Agente de Percepción, Descartada) y DUDA-012 (denominación Condición de Pago, Descartada — no hay facturación Perú); todas ya reflejadas en "No aplica a". Se precisa el motivo de cierre de "Tipo de Revisión" (DUDA-013): no impacta R16 por ser esquema prepago, sin paso de aceptación de factura a crédito. Se agrega el cierre de DUDA-014 (catálogos fiscales SUNAT específicos por producto), descartados por la cancelación de facturación Perú del 17/07. No se modifican Requisito, Alcance, Reglas ni Criterios de Aceptación: ya excluían estas funcionalidades desde el cambio #1. |
