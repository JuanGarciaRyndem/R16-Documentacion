# Tramitación de pedidos Prepago

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-015 |
| **Nombre** | Tramitación de pedidos Prepago |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-001, R16.1M-RE-FU-002, R16.1M-RE-FU-003, R16.1M-RE-FU-004, R16.1M-RE-FU-007, R16.1M-RE-FU-015 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Prepago sin sustancias controladas activando la opción de Factura por Adelantado, para que Finanzas gestione la emisión y timbrado de la factura que se enviará al cliente para iniciar el cobro.

---

## Requisito

El sistema debe permitir, en el módulo Tramitar Pedido, la tramitación de pedidos de clientes con condición de pago Prepago que no contienen sustancias controladas y que activan la opción Factura por Adelantado, en las operaciones de México y Perú. Para clientes prepago los datos de facturación no son editables en este módulo y se toman del catálogo del cliente vigente. Al tramitar con la opción Factura por Adelantado activada, el sistema genera el pendiente en el módulo Factura por Adelantado para que Finanzas emita y timbre la factura, y cierra el pendiente operativo del pedido en la bandeja de Tramitar Pedido. El pendiente en Validar Cobro se genera posteriormente, cuando la factura se emita.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Prepago en las operaciones de México y Perú.
- Pedidos sin sustancias controladas (mundial, nacional, origen).
- Pedidos en los que el ESAC activa la opción Factura por Adelantado durante la tramitación.
- Activación directa de la opción Factura por Adelantado sin código de autorización.
- Asignación del folio interno del pedido conforme a la mecánica actual del sistema.
- Generación del pendiente en el módulo Factura por Adelantado al ejecutar la acción Tramitar.
- Cierre del pendiente operativo del pedido en la bandeja del módulo Tramitar Pedido al completar la acción.
- Visualización en solo lectura de los datos de facturación del cliente (tomados del catálogo).
- Operación en Perú: el flujo opera idéntico al de México durante la tramitación en este módulo. Las diferencias para Perú se materializan posteriormente fuera de TP (no transfiere a Legacy tras la validación de cobro).

### No aplica a

- Pedidos de clientes con condición de pago Crédito (esos siguen los flujos descritos en los requisitos del bloque Crédito).
- Pedidos prepago con sustancias controladas (la combinación Factura por Adelantado + sustancias controladas no es permitida por regla regulatoria).
- Pedidos prepago sin activación de Factura por Adelantado (variante cubierta en requisito independiente del bloque Prepago).
- Visualización del radio button de Entrega con Remisión, que no se muestra en el módulo Tramitar Pedido para clientes prepago en ninguna variante.
- Edición de los datos de facturación del cliente desde el módulo Tramitar Pedido para clientes prepago (los ajustes se gestionan en el Catálogo de Clientes).
- La generación del pendiente en Validar Cobro al tramitar (en este flujo el pendiente VC se generará posteriormente, al emitirse la factura en el módulo Factura por Adelantado).
- La emisión propiamente dicha de la factura ni la mecánica interna del módulo Factura por Adelantado.
- La validación del cobro de la factura, el timbrado fiscal, el cálculo de la FEE, la generación de la Confirmación de Pedido y la transferencia a Legacy. Todas esas acciones ocurren en módulos posteriores y se cubren en requisitos independientes.
- La generación de proforma: en este escenario (prepago con Factura por Adelantado) no se genera proforma.
- La previsualización o el envío de cualquier documento al cliente desde el módulo Tramitar Pedido (el envío de la factura ocurre en el módulo Factura por Adelantado).

---

## Reglas de Negocio

**Regla 1 — Visualización de Factura por Adelantado para Prepago sin controlados**
Para pedidos de clientes Prepago sin sustancias controladas, el sistema muestra el radio button de Factura por Adelantado como opción disponible para que el ESAC decida activarla o no.

**Regla 2 — No visualización de Entrega con Remisión para Prepago**
Para pedidos de clientes Prepago (con o sin sustancias controladas), el sistema no muestra el radio button de Entrega con Remisión. Esta opción no aplica para clientes prepago en ninguna variante.

**Regla 3 — Activación de Factura por Adelantado sin código de autorización**
La activación de Factura por Adelantado en Tramitar Pedido es directa, sin requerir código de autorización ni validación adicional.

**Regla 4 — Datos de facturación bloqueados cuando se activa Factura por Adelantado**
Al activar Factura por Adelantado para un pedido Prepago, el sistema no permite editar los datos de facturación del cliente (RFC/RUC, razón social, y los campos fiscales según región: Uso CFDI y Método de Pago en México, Tipo de Operación y Condición de Pago en Perú) desde Tramitar Pedido. Los datos de facturación quedan fijados con los valores del catálogo del cliente vigente al momento de la activación; cualquier ajuste posterior se gestiona en el módulo Factura por Adelantado o en el Catálogo de Clientes según corresponda.

**Regla 7 — Folio del pedido interno conforme a mecánica actual**
El folio interno del pedido se asigna siguiendo la mecánica actual del sistema, sin cambios respecto a la versión vigente.

**Regla 11 — Generación del pendiente Factura por Adelantado al tramitar**
Al ejecutar la acción de tramitar un pedido prepago sin sustancias controladas con Factura por Adelantado activada, el sistema genera automáticamente un pendiente en el módulo Factura por Adelantado asociado al folio del pedido, para que Finanzas gestione la emisión y timbrado de la factura.

**Regla 12 — Momento de generación del pendiente Validar Cobro**
El pendiente en el módulo Validar Cobro se genera cuando la factura se emite exitosamente en el módulo Factura por Adelantado, no en el momento de tramitar.

**Regla 13 — Cierre del pendiente operativo de Tramitar Pedido al completar la acción**
Una vez generado el pendiente en Factura por Adelantado, el sistema cierra y elimina el pendiente operativo del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC. Este cierre se refiere únicamente al pendiente operativo de esa bandeja: la acción a realizar en Tramitar Pedido finalizó y el pedido avanza al siguiente módulo (Factura por Adelantado y, después, Validar Cobro). El estatus real del pedido se consulta fuera de las bandejas de pendientes (ver Criterio D5).

**Regla 14 — Composición regionalizada del panel de Información de Facturación**
El panel de Información de Facturación de Tramitar Pedido es transversal a ambas regiones y muestra los datos del cliente tomados del catálogo, en modo solo lectura. Los campos comunes a México y Perú son: Razón Social, identificador fiscal (RFC para México / RUC para Perú), Moneda, Quién Factura (empresa emisora), Condiciones de Pago (plazo comercial; ej. "60 Días", "Prepago 100%") y Comentarios para la Facturación. Los campos fiscales se regionalizan según la Región del cliente: para México se muestran Uso CFDI y Método de Pago (catálogos SAT); para Perú estos se reemplazan por Tipo de Operación (catálogo 51 SUNAT, en lugar de Uso CFDI) y Condición de Pago SUNAT en singular, Contado/Crédito (en lugar de Método de Pago). La "Condición de Pago" SUNAT de Perú es un campo fiscal distinto de las "Condiciones de Pago" comerciales (plazo) y ambos coexisten en el panel para clientes Perú. Los campos Forma de Pago (medio) y correo de envío no se muestran en este panel en ninguna región.

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
- **Cuando** se muestra la pantalla del pedido,
- **Entonces** el botón "Editar Datos" para datos de facturación no debe aparecer disponible para este pedido. El sistema deberá mostrar los datos de facturación en modo solo lectura tomados del catálogo del cliente vigente al momento de la activación.

**Criterio A4 — No visualización de Entrega con Remisión para Prepago**
- **Dado** que el pedido es de cliente Prepago,
- **Cuando** el ESAC visualiza la pantalla del pedido,
- **Entonces** el radio button de Entrega con Remisión no deberá aparecer en la pantalla, dado que esta opción no aplica para clientes prepago en ninguna variante.

**Criterio A5 — Composición regionalizada del panel de Información de Facturación**
- **Dado** que el ESAC visualiza el panel de Información de Facturación de un pedido en Tramitar Pedido,
- **Cuando** el sistema muestra el panel según la Región del cliente,
- **Entonces** para clientes México deberá mostrar Uso CFDI y Método de Pago (catálogos SAT); para clientes Perú deberá mostrar Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago Contado/Crédito SUNAT en su lugar; en ambas regiones deberá mostrar los campos comunes (Razón Social, RFC/RUC, Moneda, Quién Factura, Condiciones de Pago comerciales y Comentarios) y NO deberá mostrar Forma de Pago ni correo de envío.

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
- **Cuando** el sistema muestra la pantalla,
- **Entonces** deberá mostrar: Para con el contacto del cliente (editable, default heredado); CC con el ESAC asignado (editable, default sugerido); Asunto generado por sistema según plantilla (no editable); Adjuntos con el PDF de la proforma (no editables); y Notas extras como campo de texto editable opcional.

**Criterio C4 — Envío del correo de proforma**
- **Dado** que la pantalla de envío está completa,
- **Cuando** el usuario presiona Enviar,
- **Entonces** el sistema deberá enviar el correo al destinatario con CC al ESAC, asunto generado por sistema, adjunto del PDF de la proforma y las notas extras opcionales capturadas.

### Sección D — Pendientes generados y cierre

**Criterio D1 — Generación del pendiente Factura por Adelantado al tramitar**
- **Dado** un pedido prepago sin controlados con Factura por Adelantado activada,
- **Cuando** el ESAC ejecuta la acción Tramitar,
- **Entonces** el sistema deberá generar automáticamente un pendiente en el módulo Factura por Adelantado asociado al folio del pedido, para que Finanzas gestione posteriormente la emisión y timbrado de la factura.

**Criterio D2 — Momento de generación del pendiente Validar Cobro**
- **Dado** que el ESAC tramitó un pedido prepago con Factura por Adelantado activada,
- **Cuando** se completa la tramitación,
- **Entonces** el pendiente en Validar Cobro se generará posteriormente, cuando la factura se emita exitosamente desde el módulo Factura por Adelantado.

**Criterio D3 — Desaparición del pendiente operativo en bandeja Tramitar Pedido**
- **Dado** que la acción de Tramitar Pedido se completó (con la generación del pendiente en Factura por Adelantado),
- **Cuando** el pendiente operativo finaliza,
- **Entonces** el pedido no deberá seguir apareciendo como pendiente en la bandeja del módulo Tramitar Pedido del ESAC, entendiéndose que el pendiente operativo de esa bandeja terminó y el pedido avanzó al siguiente módulo, no que el pedido quedó tramitado en su totalidad (la Confirmación de Pedido de prepago se genera al validar el cobro). La trazabilidad del estatus del pedido se consulta fuera de las bandejas de pendientes (ver Criterio D5).

**Criterio D4 — Cancelación del pedido**
- **Dado** que un pedido tramitado tiene solicitud del cliente para cancelar,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

**Criterio D5 — Estatus del pedido a lo largo del flujo**
- **Dado** que el sistema opera por pendientes (que aparecen y desaparecen de cada bandeja a medida que se trabajan) y que esos pendientes no reflejan por sí solos el estatus global del pedido,
- **Cuando** un pedido avanza por las distintas etapas del flujo (orden recibida, pretramitación, inconsistencias, por tramitar, tramitado, con folio en espera de validación de cobro, confirmado, etc.),
- **Entonces** el sistema deberá mantener un estatus del pedido que refleje su punto en el flujo, de modo que ese estatus pueda consultarse para la trazabilidad del pedido fuera de las bandejas de pendientes.

> ** Pendiente de definición: el catálogo de estatus del pedido será propuesto por el cliente (equipo de procesos) y validado con el equipo (Sesión Cliente 2). La visualización de estos estatus para coordinadores y gerencia se hará mediante tableros de Power BI, que es una herramienta de referencia externa y NO forma parte del alcance funcional de PQF2 descrito en esta matriz. **

---

## Notas

- Variante prepago sin sustancias controladas con activación de Factura por Adelantado del módulo Tramitar Pedido. El módulo de Tramitar Pedido en este flujo es responsable de generar la proforma, gestionar la previsualización y envío al cliente, y disparar el pendiente en Factura por Adelantado. La emisión y timbrado de la factura PPD ocurren posteriormente en el módulo Factura por Adelantado.
- Cubre tres requisitos del cliente: tramitación de pedidos prepago en México y Perú con emisión de Confirmación; generación y envío automático de proforma para prepago; y cadena de pendientes generada al tramitar.
- Distinción clave respecto al flujo prepago sin Factura por Adelantado: en este flujo, al tramitar se genera el pendiente en Factura por Adelantado pero NO el pendiente en Validar Cobro. El pendiente Validar Cobro se generará después, cuando la factura PPD se emita exitosamente desde el módulo Factura por Adelantado. Esto refleja que en este flujo el documento que se va a cobrar es la factura PPD, no la proforma.
- Cambio respecto al comportamiento actual: se elimina el código de autorización para activar Factura por Adelantado (antes lo requería). La activación ahora es directa.
- Para clientes prepago, los datos de facturación nunca se pueden editar en Tramitar Pedido (independientemente de si hay sustancias controladas o no, independientemente de si se activa Factura por Adelantado o no). El botón "Editar Datos" no aparece. Cualquier ajuste a los datos fiscales del cliente debe gestionarse en el Catálogo de Clientes.
- El radio button de Entrega con Remisión no se muestra en el módulo Tramitar Pedido para clientes prepago en ninguna variante.
- El foliador de la proforma es lineal global a PQF2 (un solo contador para todas las proformas del sistema). El folio del pedido interno conserva la mecánica actual del sistema.
- El asunto del correo de proforma se compone como "Proforma" más el folio del pedido interno.
- El flujo de envío del correo de proforma requiere dos pasos secuenciales en la UI: primero previsualizar y aceptar el PDF; después confirmar los datos de envío del correo.
- El pendiente del pedido en la bandeja del módulo Tramitar Pedido se cierra automáticamente al completarse la acción de tramitar. La consulta del pedido tramitado sigue disponible desde módulos de consulta. Esta mecánica evita que el ESAC vea pedidos ya gestionados en su bandeja de pendientes.
- Los campos de información fiscal del módulo están actualmente configurados conforme a las normas fiscales de México. Para pedidos peruanos se espera capacitación al equipo operativo para clarificar el manejo de estos campos en contexto peruano.
