# Tramitación de pedidos Prepago con sustancias controladas

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-013 |
| **Nombre** | Tramitación de pedidos Prepago con sustancias controladas |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-001, R16.1M-RE-FU-006, R16.1M-RE-FU-007, R16.1M-RE-FU-008, R16.1M-RE-FU-015 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Prepago que contienen sustancias controladas (Mundial, Nacional u Origen), para que el sistema genere la proforma correspondiente, la envíe al cliente y deje el pedido listo para que cobranza valide el cobro en el módulo Validar Cobro respetando las restricciones regulatorias del producto controlado.

---

## Requisito

El sistema debe permitir, en el módulo Tramitar Pedido, la tramitación de pedidos de clientes con condición de pago Prepago que contienen sustancias controladas tipo Mundial, Nacional u Origen, en las operaciones de México y Perú. Al tramitar, el sistema genera la proforma, la presenta al ESAC para validación visual y, tras confirmar el envío al cliente, genera el pendiente correspondiente en el módulo Validar Cobro asociado al pedido y la proforma. Por restricción regulatoria, en este flujo las opciones Factura por Adelantado y Entrega con Remisión no se renderizan en la pantalla.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Prepago en las operaciones de México y Perú.
- Pedidos que contienen al menos una sustancia controlada clasificada como Mundial, Nacional u Origen.
- Generación de la proforma al ejecutar la acción Tramitar en el módulo Tramitar Pedido.
- Asignación del folio interno del pedido conforme a la mecánica actual del sistema.
- Asignación del folio de la proforma desde el foliador lineal global de PQF2 (un solo contador para todas las proformas del sistema, sin segmentación por empresa o región).
- Flujo de envío con dos pasos: previsualización del PDF de la proforma + pantalla de datos de envío del correo.
- Generación automática del pendiente en el módulo Validar Cobro al confirmar el envío del correo.
- Operación en Perú: el flujo opera idéntico al de México durante la tramitación en este módulo. Las diferencias para Perú se materializan posteriormente fuera de Tramitar Pedido (no transfiere a Legacy tras la validación de cobro).

### No aplica a

- Pedidos de clientes con condición de pago Crédito (esos siguen los flujos descritos en los requisitos del bloque Crédito).
- Pedidos prepago sin sustancias controladas (variante cubierta en requisitos independientes del bloque Prepago).
- Pedidos con activación de Factura por Adelantado (no permitida por regla regulatoria cuando hay controlados; los radio buttons no aparecen).
- Pedidos con marca de Entrega con Remisión (no permitida por regla regulatoria cuando hay controlados; los radio buttons no aparecen).
- La validación de presencia de Licencia Sanitaria y Aviso de Responsable Sanitario del cliente, que ocurre en el módulo Pretramitar Pedido antes de llegar a Tramitar Pedido.
- La validación del cobro de la proforma, la emisión de la factura anticipo, el timbrado fiscal, el cálculo de la FEE y la generación de la Confirmación de Pedido. Todas esas acciones ocurren en el módulo Validar Cobro y se cubren en requisitos independientes.

---

## Reglas de Negocio

**Regla 1 — No renderización de Factura por Adelantado y Entrega con Remisión**
Para pedidos que contienen al menos una sustancia controlada tipo Mundial, Nacional u Origen, el sistema no renderiza los radio buttons de Factura por Adelantado ni de Entrega con Remisión. Estas opciones no aparecen en la pantalla.

**Regla 2 — Generación automática de proforma al tramitar**
Al ejecutar la acción de tramitar un pedido prepago con sustancias controladas, el sistema genera automáticamente una proforma en formato PDF.

**Regla 3 — Folio de la proforma desde foliador lineal global**
El folio de la proforma se toma del foliador lineal global de PQF2, que mantiene un solo contador para todas las proformas del sistema sin segmentación por empresa o región.

**Regla 4 — Folio del pedido interno conforme a mecánica actual**
El folio interno del pedido se asigna siguiendo la mecánica actual del sistema, sin cambios respecto a la versión vigente.

**Regla 5 — Previsualización del PDF de la proforma obligatoria**
Antes de avanzar a la pantalla de datos de envío, el sistema muestra la previsualización del PDF de la proforma. El ESAC debe aceptarla explícitamente para continuar.

**Regla 6 — Pantalla de datos de envío del correo de proforma**
La pantalla de datos de envío del correo de proforma presenta: Para (destinatario) con el contacto del cliente del pedido, editable, con default heredado del catálogo del cliente; CC con el ESAC asignado al cliente/pedido, editable, con default sugerido por el sistema; Asunto generado automáticamente según plantilla, no editable; Adjuntos con el PDF de la proforma, no editables; y Notas extras, un campo de texto editable opcional para texto adicional libre.

**Regla 7 — Composición del asunto del correo de proforma**
El asunto del correo de proforma se compone como la cadena "Proforma" seguida del folio del pedido interno.

**Regla 8 — Generación automática del pendiente Validar Cobro**
Al confirmarse el envío exitoso del correo de la proforma, el sistema genera automáticamente un pendiente en el módulo Validar Cobro asociado al pedido tramitado y la proforma emitida, sin requerir clic adicional del ESAC.

**Regla 9 — Datos de facturación bloqueados en flujos Prepago**
Para pedidos de clientes con condición de pago Prepago, el sistema no permite editar los datos de facturación del cliente desde Tramitar Pedido. Los datos de facturación se toman del catálogo del cliente vigente y se muestran en modo solo lectura; cualquier ajuste se gestiona en el Catálogo de Clientes.

**Regla 10 — Cierre del pendiente de Tramitar Pedido al completar la acción**
Una vez ejecutada exitosamente la acción de tramitar, completado el envío del correo correspondiente al flujo y generados los pendientes derivados (si aplica), el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

---

## Riesgos

**Riesgo 1 — Detección incorrecta de sustancias controladas**
La no renderización de las opciones Factura por Adelantado y Entrega con Remisión depende de que el sistema identifique correctamente la presencia de sustancias controladas en el pedido. Si la clasificación del producto en el catálogo es incorrecta o el sistema falla en detectar la presencia de un controlado, podrían aparecer opciones que violan la restricción regulatoria.

**Riesgo 2 — Campos de información fiscal originalmente configurados para México**
Los campos de información fiscal del módulo Tramitar Pedido están actualmente configurados conforme a las normas fiscales de México. Al operar pedidos peruanos, el ESAC podría experimentar confusión sobre qué campos aplican o cómo interpretarlos en el contexto fiscal peruano. Se espera capacitación al equipo operativo para clarificar el manejo de los campos fiscales en pedidos de la región Perú.

---

## Criterios de Aceptación

### Sección A — Tramitación y restricciones regulatorias

**Criterio A1 — Tramitación habilitada para Prepago con controlados**
- **Dado** que un pedido pertenece a un cliente Prepago en México o Perú y contiene al menos una sustancia controlada tipo Mundial, Nacional u Origen,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación y, al ejecutarse, generar automáticamente la proforma asociada al pedido.

**Criterio A2 — No renderización de opciones bloqueadas por controlados**
- **Dado** que el pedido contiene al menos una sustancia controlada,
- **Cuando** el ESAC visualiza la pantalla del pedido,
- **Entonces** los radio buttons de Factura por Adelantado y Entrega con Remisión no deberán aparecer en la pantalla.

### Sección B — Folios y generación de la proforma

**Criterio B1 — Asignación de folio interno al tramitar**
- **Dado** que el ESAC ejecuta la acción de tramitar,
- **Cuando** el sistema procesa la solicitud,
- **Entonces** deberá asignar el folio interno del pedido siguiendo la mecánica actual del sistema.

**Criterio B2 — Asignación de folio de proforma desde foliador lineal global**
- **Dado** que el sistema genera la proforma al tramitar,
- **Cuando** se asigna el folio del documento,
- **Entonces** el sistema deberá tomar el siguiente número del foliador lineal global de PQF2 (un solo contador compartido por todas las proformas del sistema).

**Criterio B3 — Generación del PDF de la proforma**
- **Dado** que el ESAC ejecuta la acción de tramitar,
- **Cuando** se completa la asignación de folios,
- **Entonces** el sistema deberá generar automáticamente el archivo PDF de la proforma con los datos del pedido, del cliente y los folios correspondientes.

### Sección C — Previsualización y envío de la proforma

**Criterio C1 — Previsualización obligatoria del PDF antes del envío**
- **Dado** que el PDF de la proforma se generó exitosamente,
- **Cuando** el sistema inicia el proceso de envío,
- **Entonces** deberá mostrar al ESAC la previsualización del PDF y requerir aceptación explícita antes de continuar a la pantalla de datos de envío.

**Criterio C2 — Cancelación desde la previsualización**
- **Dado** que el ESAC está viendo la previsualización del PDF de la proforma,
- **Cuando** el ESAC decide no continuar (cancela la previsualización),
- **Entonces** el sistema deberá permitir volver al pedido sin enviar la proforma.

> ** Pendiente definir la política del folio de proforma ya asignado: si se conserva para el reintento o se descarta. **

**Criterio C3 — Pantalla de datos de envío con CC editable y ESAC incluido**
- **Dado** que el usuario llegó al paso de envío del correo,
- **Cuando** el sistema renderiza la pantalla,
- **Entonces** deberá mostrar: Para con el contacto del cliente (editable, default heredado); CC con el ESAC asignado (editable, default sugerido); Asunto generado por sistema según plantilla (no editable); Adjuntos con el PDF de la proforma (no editables); y Notas extras como campo de texto editable opcional.

**Criterio C4 — Envío del correo de proforma**
- **Dado** que la pantalla de envío está completa,
- **Cuando** el usuario presiona Enviar,
- **Entonces** el sistema deberá enviar el correo al destinatario con CC al ESAC, asunto generado por sistema, adjunto del PDF de la proforma y las notas extras opcionales capturadas.

**Criterio C5 — Generación automática del pendiente Validar Cobro**
- **Dado** que el correo de proforma se envió exitosamente,
- **Cuando** se completa el envío,
- **Entonces** el sistema deberá generar automáticamente un pendiente en el módulo Validar Cobro asociado al pedido tramitado y la proforma emitida.

### Sección D — Cancelación

**Criterio D1 — Cancelación del pedido**
- **Dado** que un pedido tiene solicitud del cliente para cancelar,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

---

## Notas

- Variante prepago con sustancias controladas del módulo Tramitar Pedido. El módulo de Tramitar Pedido en este flujo es responsable de generar la proforma, gestionar la previsualización y envío al cliente, y disparar el pendiente en Validar Cobro. Lo que ocurre tras Validar Cobro (factura anticipo, timbrado, Confirmación, FEE, transferencia a Legacy en caso de México) está fuera del scope de este requisito y se cubre en requisitos del módulo Validar Cobro.
- Cubre tres requisitos del cliente: tramitación de pedidos prepago en México y Perú; generación y envío automático de proforma para prepago sin Factura por Adelantado; y cadena de pendientes tras la generación de la proforma.
- La presencia de sustancias controladas determina dos efectos en el módulo: no se renderizan las opciones de Factura por Adelantado ni Entrega con Remisión; y la factura que se emita posteriormente en Validar Cobro será del tipo factura anticipo (no factura normal) por falta de datos de pedimento y aduana.
- El foliador de la proforma es lineal global a PQF2 (un solo contador para todas las proformas del sistema). El folio del pedido interno conserva la mecánica actual del sistema.
- El asunto del correo de proforma se compone como "Proforma" más el folio del pedido interno.
- El flujo de envío del correo de proforma requiere dos pasos secuenciales en la UI: primero previsualizar y aceptar el PDF; después confirmar los datos de envío del correo.
- La validación de documentos regulatorios del cliente (Licencia Sanitaria y Aviso de Responsable Sanitario) ocurre antes de llegar a Tramitar Pedido (responsabilidad del módulo Pretramitar Pedido).
- Los campos de información fiscal del módulo están actualmente configurados conforme a las normas fiscales de México. Para pedidos peruanos se espera capacitación al equipo operativo para clarificar el manejo de estos campos en contexto peruano.
- Aplicable a las operaciones de México y Perú. Las diferencias regionales (transferencia a Legacy, foliador del pedido) se materializan en módulos posteriores al de Tramitar Pedido.
