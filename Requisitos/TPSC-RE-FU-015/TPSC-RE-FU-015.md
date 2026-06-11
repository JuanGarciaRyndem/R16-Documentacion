# Tramitación de pedidos Prepago con Factura por Adelantado

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-015 |
| **Nombre** | Tramitación de pedidos Prepago con Factura por Adelantado |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-001, R16.1M-RE-FU-002, R16.1M-RE-FU-003, R16.1M-RE-FU-004, R16.1M-RE-FU-007, R16.1M-RE-FU-015 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Prepago sin sustancias controladas activando la opción de Factura por Adelantado, para que el cliente reciba la proforma correspondiente para iniciar el cobro y, en paralelo, Finanzas gestione la emisión y timbrado de la factura necesaria para el flujo prepago anticipado.

---

## Requisito

El sistema debe permitir, en el módulo Tramitar Pedido, la tramitación de pedidos de clientes con condición de pago Prepago que no contienen sustancias controladas y que activan la opción Factura por Adelantado, en las operaciones de México y Perú. Para clientes prepago los datos de facturación no son editables en este módulo y se toman del catálogo del cliente vigente. Al tramitar, el sistema genera la proforma, la presenta al ESAC para validación visual y, tras confirmar el envío al cliente, genera el pendiente en el módulo Factura por Adelantado para que Finanzas gestione la emisión de la factura y cierra el pendiente del pedido en la bandeja de Tramitar Pedido. El pendiente en Validar Cobro se genera posteriormente, cuando la factura se emita.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Prepago en las operaciones de México y Perú.
- Pedidos sin sustancias controladas (mundial, nacional, origen).
- Pedidos en los que el ESAC activa la opción Factura por Adelantado durante la tramitación.
- Activación directa de la opción Factura por Adelantado sin código de autorización.
- Generación de la proforma al ejecutar la acción Tramitar en el módulo Tramitar Pedido.
- Asignación del folio interno del pedido conforme a la mecánica actual del sistema.
- Asignación del folio de la proforma desde el foliador lineal global de PQF2 (un solo contador para todas las proformas del sistema, sin segmentación por empresa o región).
- Flujo de envío con dos pasos: previsualización del PDF de la proforma + pantalla de datos de envío del correo.
- Generación automática del pendiente en el módulo Factura por Adelantado al confirmar el envío del correo.
- Cierre automático del pendiente del pedido en la bandeja del módulo Tramitar Pedido al completar la tramitación.
- Visualización en solo lectura de los datos de facturación del cliente (tomados del catálogo).
- Operación en Perú: el flujo opera idéntico al de México durante la tramitación en este módulo.

### No aplica a

- Pedidos de clientes con condición de pago Crédito (esos siguen los flujos descritos en los requisitos del bloque Crédito).
- Pedidos prepago con sustancias controladas (la combinación Factura por Adelantado + sustancias controladas no es permitida por regla regulatoria).
- Pedidos prepago sin activación de Factura por Adelantado (variante cubierta en requisito independiente del bloque Prepago).
- Renderizado del radio button de Entrega con Remisión, que no se muestra en el módulo Tramitar Pedido para clientes prepago en ninguna variante.
- Edición de los datos de facturación del cliente desde el módulo Tramitar Pedido para clientes prepago (los ajustes se gestionan en el Catálogo de Clientes).
- La generación del pendiente en Validar Cobro al tramitar (en este flujo el pendiente VC se generará posteriormente, al emitirse la factura en el módulo Factura por Adelantado).
- La emisión propiamente dicha de la factura ni la mecánica interna del módulo Factura por Adelantado.
- La validación del cobro de la factura, el timbrado fiscal, el cálculo de la FEE, la generación de la Confirmación de Pedido y la transferencia a Legacy.

---

## Reglas de Negocio

**Regla 1 — Renderizado de Factura por Adelantado para Prepago sin controlados**
Para pedidos de clientes Prepago sin sustancias controladas, el sistema renderiza el radio button de Factura por Adelantado como opción disponible para que el ESAC decida activarla o no.

**Regla 2 — No renderizado de Entrega con Remisión para Prepago**
Para pedidos de clientes Prepago (con o sin sustancias controladas), el sistema no renderiza el radio button de Entrega con Remisión. Esta opción no aplica para clientes prepago en ninguna variante.

**Regla 3 — Activación de Factura por Adelantado sin código de autorización**
La activación de Factura por Adelantado en Tramitar Pedido es directa, sin requerir código de autorización ni validación adicional.

**Regla 4 — Datos de facturación bloqueados cuando se activa Factura por Adelantado**
Al activar Factura por Adelantado para un pedido Prepago, el sistema no permite editar los datos de facturación del cliente (RFC, razón social, Uso CFDI, Método y Forma de Pago, correo de envío) desde Tramitar Pedido. Los datos de facturación quedan fijados con los valores del catálogo del cliente vigente al momento de la activación; cualquier ajuste posterior se gestiona en el módulo Factura por Adelantado o en el Catálogo de Clientes según corresponda.

**Regla 5 — Generación automática de proforma al tramitar**
Al ejecutar la acción de tramitar un pedido prepago sin sustancias controladas con Factura por Adelantado activada, el sistema genera automáticamente una proforma en formato PDF.

**Regla 6 — Folio de la proforma desde foliador lineal global**
El folio de la proforma se toma del foliador lineal global de PQF2, que mantiene un solo contador para todas las proformas del sistema sin segmentación por empresa o región.

**Regla 7 — Folio del pedido interno conforme a mecánica actual**
El folio interno del pedido se asigna siguiendo la mecánica actual del sistema, sin cambios respecto a la versión vigente.

**Regla 8 — Previsualización del PDF de la proforma obligatoria**
Antes de avanzar a la pantalla de datos de envío, el sistema muestra la previsualización del PDF de la proforma. El ESAC debe aceptarla explícitamente para continuar.

**Regla 9 — Pantalla de datos de envío del correo de proforma**
La pantalla de datos de envío del correo de proforma presenta: Para (destinatario) con el contacto del cliente del pedido, editable, con default heredado del catálogo del cliente; CC con el ESAC asignado al cliente/pedido, editable, con default sugerido por el sistema; Asunto generado automáticamente según plantilla, no editable; Adjuntos con el PDF de la proforma, no editables; y Notas extras, un campo de texto editable opcional para texto adicional libre.

**Regla 10 — Composición del asunto del correo de proforma**
El asunto del correo de proforma se compone como la cadena "Proforma" seguida del folio del pedido interno.

**Regla 11 — Generación automática del pendiente Factura por Adelantado**
Al confirmarse el envío exitoso del correo de la proforma para un pedido con Factura por Adelantado activada, el sistema genera automáticamente un pendiente en el módulo Factura por Adelantado asociado al folio del pedido, para que se gestione posteriormente la emisión y timbrado de la factura.

**Regla 12 — No generación del pendiente Validar Cobro al tramitar**
Al tramitar un pedido prepago con Factura por Adelantado activada, el sistema no genera el pendiente en el módulo Validar Cobro en ese momento. El pendiente Validar Cobro se generará posteriormente, cuando la factura se emita exitosamente en el módulo Factura por Adelantado.

**Regla 13 — Cierre del pendiente de Tramitar Pedido al completar la acción**
Una vez completados el envío del correo de la proforma y la generación del pendiente en Factura por Adelantado, el sistema cierra y elimina el pendiente del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC.

---

## Riesgos

**Riesgo 1 — Campos de información fiscal originalmente configurados para México**
Los campos de información fiscal del módulo Tramitar Pedido están actualmente configurados conforme a las normas fiscales de México. Al operar pedidos peruanos, el ESAC podría experimentar confusión sobre qué campos aplican o cómo interpretarlos en el contexto fiscal peruano. Se espera capacitación al equipo operativo para clarificar el manejo de los campos fiscales en pedidos de la región Perú.

---

## Criterios de Aceptación

### Sección A — Tramitación, activación y opciones en pantalla

**Criterio A1 — Tramitación habilitada para Prepago sin controlados con Factura por Adelantado activada**
- **Dado** que un pedido pertenece a un cliente Prepago en México o Perú, sin productos controlados, y el ESAC activa la opción Factura por Adelantado,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación y, al ejecutarse, generar automáticamente la proforma asociada al pedido.

**Criterio A2 — Activación de Factura por Adelantado desde Tramitar Pedido**
- **Dado** que un pedido pertenece a un cliente Prepago sin productos controlados,
- **Cuando** el ESAC visualiza el módulo Tramitar Pedido,
- **Entonces** el sistema deberá ofrecer la opción de activar Factura por Adelantado. La activación es directa y no requiere código de autorización.

**Criterio A3 — Bloqueo de edición de datos de facturación al activar Factura por Adelantado**
- **Dado** que el ESAC activó la opción Factura por Adelantado en Tramitar Pedido,
- **Cuando** se renderiza la pantalla del pedido,
- **Entonces** el botón "Editar Datos" para datos de facturación no debe aparecer disponible para este pedido. El sistema deberá mostrar los datos de facturación en modo solo lectura tomados del catálogo del cliente vigente al momento de la activación.

**Criterio A4 — No renderizado de Entrega con Remisión para Prepago**
- **Dado** que el pedido es de cliente Prepago,
- **Cuando** el ESAC visualiza la pantalla del pedido,
- **Entonces** el radio button de Entrega con Remisión no deberá aparecer en la pantalla, dado que esta opción no aplica para clientes prepago en ninguna variante.

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

### Sección D — Pendientes generados y cierre

**Criterio D1 — Generación automática del pendiente Factura por Adelantado**
- **Dado** que el correo de proforma se envió exitosamente y el pedido tiene Factura por Adelantado activada,
- **Cuando** se completa el envío,
- **Entonces** el sistema deberá generar automáticamente un pendiente en el módulo Factura por Adelantado asociado al folio del pedido, para que se ejecute posteriormente la emisión y timbrado de la factura.

**Criterio D2 — No generación del pendiente Validar Cobro al tramitar**
- **Dado** que el ESAC tramitó un pedido prepago con Factura por Adelantado activada,
- **Cuando** se completa la tramitación y el envío,
- **Entonces** el sistema no deberá generar pendiente en Validar Cobro en este momento. El pendiente Validar Cobro se generará posteriormente, cuando la factura se emita exitosamente desde el módulo Factura por Adelantado.

**Criterio D3 — Desaparición del pendiente en bandeja Tramitar Pedido**
- **Dado** que el pedido se tramitó exitosamente (incluyendo el envío del correo de proforma y la generación del pendiente en Factura por Adelantado),
- **Cuando** se completa la tramitación,
- **Entonces** el pedido no deberá seguir apareciendo como pendiente en la bandeja del módulo Tramitar Pedido del ESAC. La consulta histórica del pedido sigue disponible desde los reportes correspondientes.

**Criterio D4 — Cancelación del pedido**
- **Dado** que un pedido tramitado tiene solicitud del cliente para cancelar,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

---

## Notas

- Variante prepago sin sustancias controladas con activación de Factura por Adelantado del módulo Tramitar Pedido.
- Distinción clave respecto al flujo prepago sin Factura por Adelantado: en este flujo, al tramitar se genera el pendiente en Factura por Adelantado pero NO el pendiente en Validar Cobro. El pendiente Validar Cobro se generará después, cuando la factura PPD se emita exitosamente desde el módulo Factura por Adelantado.
- Cambio respecto al comportamiento actual: se elimina el código de autorización para activar Factura por Adelantado (antes lo requería). La activación ahora es directa.
- Para clientes prepago, los datos de facturación nunca se pueden editar en Tramitar Pedido. El botón "Editar Datos" no aparece.
- El radio button de Entrega con Remisión no se renderiza en el módulo Tramitar Pedido para clientes prepago en ninguna variante.
- El foliador de la proforma es lineal global a PQF2 (un solo contador para todas las proformas del sistema).
- El asunto del correo de proforma se compone como "Proforma" más el folio del pedido interno.
- El flujo de envío del correo de proforma requiere dos pasos secuenciales en la UI: primero previsualizar y aceptar el PDF; después confirmar los datos de envío del correo.
- El pendiente del pedido en la bandeja del módulo Tramitar Pedido se cierra automáticamente al completarse la acción de tramitar.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-06-10 | OBS-028 | El cliente confirma que el timbrado fiscal se genera en el módulo Factura por Adelantado. Confirmado/ya cubierto: el diseño vigente ya establece que el timbrado ocurre en el módulo Factura por Adelantado; FU-015 deja explícitamente fuera de su alcance la mecánica de facturación. Sin cambios estructurales. |
