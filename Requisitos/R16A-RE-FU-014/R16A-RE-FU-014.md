# Tramitación de pedidos Prepago sin controlados sin Factura por Adelantado

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-014 |
| **Nombre** | Tramitación de pedidos Prepago sin controlados sin Factura por Adelantado |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-001, R16.1M-RE-FU-006, R16.1M-RE-FU-007, R16.1M-RE-FU-008, R16.1M-RE-FU-015 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Prepago sin sustancias controladas y sin activar Factura por Adelantado, para que el sistema genere la proforma correspondiente, la envíe al cliente y deje el pedido listo para que cobranza valide el cobro en el módulo Validar Cobro siguiendo el flujo regular sin restricciones regulatorias adicionales.

---

## Requisito

El sistema debe permitir, en el módulo Tramitar Pedido, la tramitación de pedidos de clientes con condición de pago Prepago que no contienen sustancias controladas y no requieren Factura por Adelantado, en las operaciones de México y Perú. Para clientes prepago los datos de facturación no son editables en este módulo y se toman del catálogo del cliente vigente. Al tramitar, el sistema genera la proforma, la presenta al ESAC para validación visual y, tras confirmar el envío al cliente, genera el pendiente correspondiente en el módulo Validar Cobro y cierra el pendiente del pedido en la bandeja de Tramitar Pedido.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Prepago en las operaciones de México y Perú.
- Pedidos sin sustancias controladas (mundial, nacional, origen).
- Pedidos en los que el ESAC no activa la opción Factura por Adelantado durante la tramitación.
- Generación de la proforma al ejecutar la acción Tramitar en el módulo Tramitar Pedido.
- Asignación del folio interno del pedido conforme a la mecánica actual del sistema.
- Asignación del folio de la proforma desde el foliador lineal global de PQF2 (un solo contador para todas las proformas del sistema, sin segmentación por empresa o región).
- Flujo de envío con dos pasos: previsualización del PDF de la proforma + pantalla de datos de envío del correo.
- Generación automática del pendiente en el módulo Validar Cobro al confirmar el envío del correo.
- Cierre automático del pendiente del pedido en la bandeja del módulo Tramitar Pedido al completar la tramitación.
- Renderizado del radio button de Factura por Adelantado como opción disponible (no seleccionada en este flujo).
- Visualización en solo lectura de los datos de facturación del cliente (tomados del catálogo).
- Operación en Perú: el flujo opera idéntico al de México durante la tramitación en este módulo. Las diferencias para Perú se materializan posteriormente fuera de TP (no transfiere a Legacy tras la validación de cobro).

### No aplica a

- Pedidos de clientes con condición de pago Crédito (esos siguen los flujos descritos en los requisitos del bloque Crédito).
- Pedidos prepago con sustancias controladas (variante cubierta en requisito independiente del bloque Prepago).
- Pedidos prepago con activación de Factura por Adelantado (variante cubierta en requisito independiente del bloque Prepago).
- Renderizado del radio button de Entrega con Remisión, que no se muestra en el módulo Tramitar Pedido para clientes prepago en ninguna variante.
- Edición de los datos de facturación del cliente desde el módulo Tramitar Pedido para clientes prepago (los ajustes se gestionan en el Catálogo de Clientes).

---

## Reglas de Negocio

**Regla 1 — Renderizado de Factura por Adelantado para Prepago sin controlados**
Para pedidos de clientes Prepago sin sustancias controladas, el sistema renderiza el radio button de Factura por Adelantado como opción disponible. El ESAC decide si la activa o no; en este flujo no se activa.

**Regla 2 — No renderizado de Entrega con Remisión para Prepago**
Para pedidos de clientes Prepago (con o sin sustancias controladas), el sistema no renderiza el radio button de Entrega con Remisión. Esta opción no aplica para clientes prepago en ninguna variante.

**Regla 3 — Datos de facturación bloqueados en flujos Prepago**
Para pedidos de clientes con condición de pago Prepago, el sistema no permite editar los datos de facturación del cliente desde Tramitar Pedido. Los datos de facturación se toman del catálogo del cliente vigente y se muestran en modo solo lectura; cualquier ajuste se gestiona en el Catálogo de Clientes.

**Regla 4 — Generación automática de proforma al tramitar**
Al ejecutar la acción de tramitar un pedido prepago sin sustancias controladas y sin Factura por Adelantado, el sistema genera automáticamente una proforma en formato PDF.

**Regla 5 — Folio de la proforma desde foliador lineal global**
El folio de la proforma se toma del foliador lineal global de PQF2, que mantiene un solo contador para todas las proformas del sistema sin segmentación por empresa o región.

**Regla 6 — Folio del pedido interno conforme a mecánica actual**
El folio interno del pedido se asigna siguiendo la mecánica actual del sistema, sin cambios respecto a la versión vigente.

**Regla 7 — Previsualización del PDF de la proforma obligatoria**
Antes de avanzar a la pantalla de datos de envío, el sistema muestra la previsualización del PDF de la proforma. El ESAC debe aceptarla explícitamente para continuar.

**Regla 8 — Pantalla de datos de envío del correo de proforma**
La pantalla de datos de envío del correo de proforma presenta: Para (destinatario) con el contacto del cliente del pedido, editable, con default heredado del catálogo del cliente; CC con el ESAC asignado al cliente/pedido, editable, con default sugerido por el sistema; Asunto generado automáticamente según plantilla, no editable; Adjuntos con el PDF de la proforma, no editables; y Notas extras, un campo de texto editable opcional para texto adicional libre.

**Regla 9 — Composición del asunto del correo de proforma**
El asunto del correo de proforma se compone como la cadena "Proforma" seguida del folio del pedido interno.

**Regla 10 — Generación automática del pendiente Validar Cobro**
Al confirmarse el envío exitoso del correo de la proforma, el sistema genera automáticamente un pendiente en el módulo Validar Cobro asociado al pedido tramitado y la proforma emitida, sin requerir clic adicional del ESAC.

**Regla 11 — Cierre del pendiente de Tramitar Pedido al completar la acción**
Una vez completados el envío del correo de la proforma y la generación del pendiente en Validar Cobro, el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

**Regla 12 — Conservación del folio de proforma ante reintento de envío (DUDA-030, resuelta 2026-08-21)**
El folio de proforma ya asignado se CONSERVA (se consume) hasta que el envío del correo se complete exitosamente. Si el envío falla, el sistema debe reintentar con el MISMO folio de proforma; no se descarta el folio asignado ni se genera uno nuevo por cada intento fallido de envío. Esto evita huecos innecesarios en la numeración lineal global del foliador de proformas por simples reintentos de envío.

---

## Riesgos

**Riesgo 1 — Campos de información fiscal originalmente configurados para México**
Los campos de información fiscal del módulo Tramitar Pedido están actualmente configurados conforme a las normas fiscales de México. Al operar pedidos peruanos, el ESAC podría experimentar confusión sobre qué campos aplican o cómo interpretarlos en el contexto fiscal peruano. Se espera capacitación al equipo operativo para clarificar el manejo de los campos fiscales en pedidos de la región Perú.

---

## Criterios de Aceptación

### Sección A — Tramitación y opciones en pantalla

**Criterio A1 — Tramitación habilitada para Prepago sin controlados sin Factura por Adelantado**
- **Dado** que un pedido pertenece a un cliente Prepago en México o Perú, sin productos controlados, y el ESAC no activa la opción Factura por Adelantado,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación y, al ejecutarse, generar automáticamente la proforma asociada al pedido.

**Criterio A2 — Renderizado de Factura por Adelantado disponible**
- **Dado** que el pedido es de cliente Prepago sin sustancias controladas,
- **Cuando** el ESAC visualiza la pantalla del pedido,
- **Entonces** el radio button de Factura por Adelantado deberá renderizarse como opción disponible. El ESAC decide si lo activa o no.

**Criterio A3 — No renderizado de Entrega con Remisión para Prepago**
- **Dado** que el pedido es de cliente Prepago,
- **Cuando** el ESAC visualiza la pantalla del pedido,
- **Entonces** el radio button de Entrega con Remisión no deberá aparecer en la pantalla, dado que esta opción no aplica para clientes prepago en ninguna variante.

**Criterio A4 — Bloqueo de edición de datos de facturación para Prepago**
- **Dado** que el pedido es de cliente Prepago,
- **Cuando** el ESAC visualiza la pantalla del pedido,
- **Entonces** el botón "Editar Datos" para datos de facturación no debe aparecer disponible. Los datos de facturación se muestran en modo solo lectura tomados del catálogo del cliente.

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

> ~~Pendiente definir la política del folio de proforma ya asignado: si se conserva para el reintento o se descarta.~~
> **Resuelto (DUDA-030, 2026-08-21):** el folio de proforma se conserva/reintenta con el MISMO folio hasta que el envío se complete exitosamente; no se descarta ni se asigna uno nuevo en cada intento fallido (ver Regla 12).

**Criterio C2b — Reintento de envío conserva el mismo folio de proforma**
- **Dado** que el envío del correo de la proforma falla tras haberse asignado el folio de proforma,
- **Cuando** el ESAC o el sistema reintenta el envío,
- **Entonces** el sistema deberá reutilizar el mismo folio de proforma ya asignado, sin descartarlo ni generar uno nuevo, hasta que el envío se complete exitosamente.

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

### Sección D — Cierre y cancelación

**Criterio D1 — Desaparición del pendiente en bandeja Tramitar Pedido**
- **Dado** que el pedido se tramitó exitosamente (incluyendo el envío del correo de proforma y la generación del pendiente en Validar Cobro),
- **Cuando** se completa la tramitación,
- **Entonces** el pedido no deberá seguir apareciendo como pendiente en la bandeja del módulo Tramitar Pedido del ESAC. La consulta histórica del pedido sigue disponible desde los reportes correspondientes.

**Criterio D2 — Cancelación del pedido**
- **Dado** que un pedido tramitado tiene solicitud del cliente para cancelar,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

---

## Notas

- Variante prepago sin sustancias controladas y sin Factura por Adelantado del módulo Tramitar Pedido. Es el flujo más común dentro del bloque Prepago.
- Cubre tres requisitos del cliente: tramitación de pedidos prepago en México y Perú con emisión de Confirmación; generación y envío automático de proforma para prepago sin Factura por Adelantado; y cadena de pendientes tras la generación de la proforma.
- En este flujo, dado que el pedido no contiene sustancias controladas, la factura que se emita posteriormente en Validar Cobro será una factura normal (no factura anticipo).
- El radio button de Factura por Adelantado se renderiza disponible en este flujo porque el cliente sí podría solicitar esta variante.
- El radio button de Entrega con Remisión no se renderiza en el módulo Tramitar Pedido para clientes prepago en ninguna variante.
- Para clientes prepago, los datos de facturación nunca se pueden editar en Tramitar Pedido. El botón "Editar Datos" no aparece. Cualquier ajuste a los datos fiscales del cliente debe gestionarse en el Catálogo de Clientes.
- El foliador de la proforma es lineal global a PQF2 (un solo contador para todas las proformas del sistema). El folio del pedido interno conserva la mecánica actual del sistema.
- El asunto del correo de proforma se compone como "Proforma" más el folio del pedido interno.
- El flujo de envío del correo de proforma requiere dos pasos secuenciales en la UI: primero previsualizar y aceptar el PDF; después confirmar los datos de envío del correo.
- El pendiente del pedido en la bandeja del módulo Tramitar Pedido se cierra automáticamente al completarse la acción de tramitar.
- Aplicable a las operaciones de México y Perú. Las diferencias regionales (transferencia a Legacy) se materializan en módulos posteriores al de Tramitar Pedido.

## Cambios

| Fecha | Cambio | Referencia |
|---|---|---|
| 2026-08-21 | Se resuelve la política del folio de proforma ante reintento de envío fallido: se conserva/consume el mismo folio hasta el envío exitoso (no se descarta ni se reasigna). Se agrega Regla 12 y Criterio C2b; se cierra la nota pendiente bajo el Criterio C2. | DUDA-030 |
