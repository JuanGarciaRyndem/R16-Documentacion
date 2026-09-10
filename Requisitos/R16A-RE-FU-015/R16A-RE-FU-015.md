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

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Prepago sin sustancias controladas activando la opción de Factura por Adelantado, para que el Analista de Cuentas por Cobrar (rol Gestor de Cobranza) gestione la emisión y timbrado de la factura que se enviará al cliente para iniciar el cobro.

---

## Requisito

El sistema debe permitir, en el módulo Tramitar Pedido, la tramitación de pedidos de clientes con condición de pago Prepago que no contienen sustancias controladas y que activan la opción Factura por Adelantado, en las operaciones de México y Perú. Para clientes prepago los datos de facturación no son editables en este módulo y se toman del catálogo del cliente vigente. Al tramitar con la opción Factura por Adelantado activada, el sistema genera el pendiente en el módulo Factura por Adelantado para que el Analista de Cuentas por Cobrar (rol Gestor de Cobranza) emita y timbre la factura, y cierra el pendiente operativo del pedido en la bandeja de Tramitar Pedido. El pendiente en Validar Cobro se genera posteriormente, cuando la factura se emita.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Prepago en las operaciones de México y Perú.
- Pedidos sin sustancias controladas (mundial, nacional, origen).
- Pedidos en los que el ESAC activa la opción Factura por Adelantado durante la tramitación.
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
Para pedidos de clientes Prepago sin sustancias controladas, el sistema muestra el radio button de Factura por Adelantado como opción disponible para que el ESAC decida activarla o no. La activación es directa, sin requerir código de autorización ni validación adicional.

**Regla 2 — No visualización de Entrega con Remisión para Prepago**
Para pedidos de clientes Prepago (con o sin sustancias controladas), el sistema no muestra el radio button de Entrega con Remisión. Esta opción no aplica para clientes prepago en ninguna variante.

**Regla 3 — Datos de facturación bloqueados cuando se activa Factura por Adelantado**
Al activar Factura por Adelantado para un pedido Prepago, el sistema no permite editar los datos de facturación del cliente desde Tramitar Pedido. Los datos de facturación quedan fijados con los valores del catálogo del cliente vigente al momento de la activación; cualquier ajuste posterior se gestiona en el módulo Factura por Adelantado o en el Catálogo de Clientes según corresponda.

**Regla 4 — Folio del pedido interno conforme a mecánica actual**
El folio interno del pedido se asigna siguiendo la mecánica actual del sistema, sin cambios respecto a la versión vigente.

**Regla 5 — Generación del pendiente Factura por Adelantado al tramitar**
Al ejecutar la acción de tramitar un pedido prepago sin sustancias controladas con Factura por Adelantado activada, el sistema genera automáticamente un pendiente en el módulo Factura por Adelantado asociado al folio del pedido, para que el Analista de Cuentas por Cobrar (rol Gestor de Cobranza) gestione la emisión y timbrado de la factura.

**Regla 6 — Momento de generación del pendiente Validar Cobro**
El pendiente en el módulo Validar Cobro se genera cuando la factura se emite exitosamente en el módulo Factura por Adelantado, no en el momento de tramitar.

**Regla 7 — Cierre del pendiente operativo de Tramitar Pedido al completar la acción**
Una vez generado el pendiente en Factura por Adelantado, el sistema cierra el pendiente operativo del pedido en la bandeja de Tramitar Pedido, de modo que el pedido ya no aparece como acción pendiente para el ESAC. Este cierre se refiere únicamente al pendiente operativo de esa bandeja: la acción a realizar en Tramitar Pedido finalizó y el pedido avanza al siguiente módulo (Factura por Adelantado y, después, Validar Cobro). El estatus real del pedido se consulta fuera de las bandejas de pendientes (ver Criterio D5).

---

## Riesgos

**Riesgo 1 — Panel de Información de Facturación sin diferenciación regional**
El panel de Información de Facturación de Tramitar Pedido muestra los mismos campos (conforme a las normas fiscales de México) para todos los clientes, sin diferenciar por Región, dado que el timbrado fiscal de Perú queda fuera del alcance de esta release. Es un riesgo operativo menor y no bloquea el desarrollo.

---

## Criterios de Aceptación

### Sección A — Tramitación, activación y opciones en pantalla

**Criterio A1 — Tramitación habilitada para Prepago sin controlados con Factura por Adelantado activada**
- **Dado** que un pedido pertenece a un cliente Prepago en México o Perú, sin productos controlados, y el ESAC activa la opción Factura por Adelantado,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación y, al ejecutarse, generar automáticamente el pendiente en el módulo Factura por Adelantado asociado al pedido.

**Criterio A2 — Activación de Factura por Adelantado desde Tramitar Pedido**
- **Dado** que un pedido pertenece a un cliente Prepago sin productos controlados,
- **Cuando** el ESAC visualiza el módulo Tramitar Pedido,
- **Entonces** el sistema deberá ofrecer la opción de activar Factura por Adelantado, de forma directa.

**Criterio A3 — Bloqueo de edición de datos de facturación al activar Factura por Adelantado**
- **Dado** que el ESAC activó la opción Factura por Adelantado en Tramitar Pedido,
- **Cuando** se muestra la pantalla del pedido,
- **Entonces** el botón "Editar Datos" para datos de facturación no debe aparecer disponible para este pedido. El sistema deberá mostrar los datos de facturación en modo solo lectura tomados del catálogo del cliente vigente al momento de la activación.

**Criterio A4 — No visualización de Entrega con Remisión para Prepago**
- **Dado** que el pedido es de cliente Prepago,
- **Cuando** el ESAC visualiza la pantalla del pedido,
- **Entonces** el radio button de Entrega con Remisión no deberá aparecer en la pantalla, dado que esta opción no aplica para clientes prepago en ninguna variante.

### Sección D — Pendientes generados y cierre

**Criterio D1 — Generación del pendiente Factura por Adelantado al tramitar**
- **Dado** un pedido prepago sin controlados con Factura por Adelantado activada,
- **Cuando** el ESAC ejecuta la acción Tramitar,
- **Entonces** el sistema deberá generar automáticamente un pendiente en el módulo Factura por Adelantado asociado al folio del pedido, para que el Analista de Cuentas por Cobrar (rol Gestor de Cobranza) gestione posteriormente la emisión y timbrado de la factura.

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

> **Resuelto (OBS-027):** El catálogo de estatus del pedido (`catEstadoPedido`) y su asignación al pedido (`PedidoEstadoActual`) quedaron diseñados y se documentan en el Impacto BD de este requisito. La visualización de estos estatus para coordinadores y gerencia se hará mediante tableros de Power BI, herramienta de referencia externa que NO forma parte del alcance funcional de PQF2 descrito en esta matriz.

---

## Notas

- Variante prepago sin sustancias controladas con activación de Factura por Adelantado del módulo Tramitar Pedido. En este flujo NO se genera proforma: el módulo de Tramitar Pedido es responsable únicamente de disparar el pendiente en Factura por Adelantado; la emisión, timbrado y envío de la factura PPD al cliente ocurren posteriormente en ese módulo.
- Cubre tres requisitos del cliente: tramitación de pedidos prepago en México y Perú con emisión de Confirmación; activación de Factura por Adelantado para prepago; y cadena de pendientes generada al tramitar.
- Distinción clave respecto al flujo prepago sin Factura por Adelantado: en este flujo, al tramitar se genera el pendiente en Factura por Adelantado pero NO el pendiente en Validar Cobro. El pendiente Validar Cobro se generará después, cuando la factura PPD se emita exitosamente desde el módulo Factura por Adelantado. Esto refleja que en este flujo el documento que se va a cobrar es la factura PPD, no una proforma.
- Para clientes prepago, los datos de facturación nunca se pueden editar en Tramitar Pedido (independientemente de si hay sustancias controladas o no, independientemente de si se activa Factura por Adelantado o no). El botón "Editar Datos" no aparece. Cualquier ajuste a los datos fiscales del cliente debe gestionarse en el Catálogo de Clientes.
- El radio button de Entrega con Remisión no se muestra en el módulo Tramitar Pedido para clientes prepago en ninguna variante.
- El pendiente del pedido en la bandeja del módulo Tramitar Pedido se cierra automáticamente al completarse la acción de tramitar. Esta mecánica evita que el ESAC vea pedidos ya gestionados en su bandeja de pendientes.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|-------------------------|
| 1 | 2026-09-04 | Verificación contra matriz | Se comparó este documento contra un nuevo extracto de la matriz de requisitos: el contenido ya coincide (incluye Regla 14 / Criterio A5 de panel regionalizado y la resolución de DUDA-030). La matriz traía reabierta la pregunta de DUDA-030 bajo el Criterio C2 y la de estatus del pedido bajo el Criterio D5; se conservó el cierre ya registrado el 2026-08-21 para DUDA-030 en vez de reabrirlo, y la pregunta de estatus del pedido sigue abierta en ambas versiones (sin cambio). |
| 2 | 2026-09-10 | Ajuste por retiro del timbrado de Perú / eliminación de la Sección C (proforma no aplica) / corrección de rol / cierre de estatus del pedido | Se elimina la Sección C completa (Criterios C1-C4 y la nota de DUDA-030 asociada), que contradecía el Alcance: este flujo (Prepago + Factura por Adelantado) **no genera proforma**. Se corrige el Criterio A1, que erróneamente indicaba generación de proforma; ahora refiere la generación del pendiente en Factura por Adelantado. Se elimina la Regla 3 (activación sin código de autorización, ya implícita en la Regla 1) y la Regla 14/Criterio A5 (panel regionalizado México/Perú, retirado por el fin del timbrado de Perú en esta release); las reglas restantes se renumeran consecutivamente (1-7). El Riesgo 1 se replantea: ya no es "confusión por campos fiscales de Perú" sino que el panel muestra los mismos campos (México) para todos los clientes sin diferenciación regional (riesgo operativo menor, no bloqueante). Se sustituye el término genérico "Finanzas" por "Analista de Cuentas por Cobrar (rol Gestor de Cobranza)" en Historia de Usuario, Requisito, Regla 5 y Criterio D1 (alineado con la resolución de DUDA-028 de los requisitos hermanos). Se resuelve el pendiente de definición del Criterio D5 (catálogo de estatus del pedido / Sesión Cliente 2): queda documentado como resuelto (OBS-027) vía `catEstadoPedido`/`PedidoEstadoActual`, diseñados en el Impacto BD de este requisito; se conserva únicamente la referencia a los tableros de Power BI como herramienta externa de visualización. Se depuran las Notas de referencias a proforma/foliador/envío de correo (no aplican a este flujo) y de la mención de "módulos de consulta" y capacitación al equipo operativo sobre campos fiscales de Perú (ya no aplica). |
