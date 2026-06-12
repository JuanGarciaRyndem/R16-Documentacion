# R16A-RE-FU-023 — Validar Cobro

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-023 |
| **Módulo** | Validar Cobro |
| **Submódulo** | Validar Cobro |
| **Estado** | Propuesto |
| **Épica / Relación** | R16.2M-RE-FU-002 |

---

## Requisito Funcional

Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver) **, quiero contar con la pantalla principal de Validar Cobro que liste los clientes de mi cartera con pendientes y me ofrezca la acción adecuada según su estado (realizar cobros o gestionar cobranza), para priorizar y dar seguimiento al cobro de cada cliente desde un único punto de entrada.

**Descripción:**
El sistema debe contar con una pantalla principal del módulo Validar Cobro que muestre el listado de clientes de la cartera del usuario que tengan pendientes (cobros recibidos pendientes de aplicar o proformas/facturas pendientes de cobrar). Por cada cliente, el sistema ofrece una acción contextual según su estado: cuando existen cobros recibidos pendientes de aplicar, conduce al wizard de tres pasos (Captura → Asociación → Facturación y Envío); cuando no hay cobros recibidos, abre la gestión de cobranza del cliente. La pantalla es estructuralmente la misma para Región México y Región Perú, con visibilidad filtrada por la cartera asignada al usuario, que opera clientes de una sola región. Las diferencias operativas entre regiones se manifiestan en las pantallas internas del wizard, documentadas en requisitos independientes.

---

## Alcance

### Aplica a

- Pantalla principal del módulo Validar Cobro: listado de clientes con pendientes de cobranza.
- La pantalla aplica tanto a Región México como a Región Perú con estructura idéntica. La cartera del Cobrador es por región: cada usuario opera clientes de UNA sola región. El listado de cada usuario muestra únicamente clientes de su región y de su cartera (no se mezclan regiones en una misma vista).
- Listado agrupado por cliente con conteos de cobros recibidos pendientes de aplicar y de proformas/facturas pendientes de cobrar.
- Buscador único por nombre de cliente o identificador fiscal (RFC para México, RUC para Perú).
- Acción contextual por cliente según estado:
  - "Realizar Cobros": cuando el cliente tiene uno o más cobros recibidos pendientes de aplicar (correos del Buzón de Cobros del cliente).
  - "Gestionar Cobranza": cuando el cliente NO tiene cobros recibidos (cero cobros) pero sí tiene proformas/facturas pendientes de cobrar.
- Modal "Gestionar Cobranza" con listado de pedidos pendientes del cliente, datos de contacto, fecha estimada de pago editable por pedido y opción de cancelación de pedido por falta de pago.
- Visibilidad filtrada por cartera del usuario (campo Cobrador del Catálogo de Clientes) combinado con la región operativa del usuario.
- Tooltip explicativo sobre el estado del cliente cuando aplica (por ejemplo: cliente sin cobros recibidos en el Buzón).
- Salida hacia el wizard de tres pasos (Captura del Cobro → Asociación → Facturación y Envío) al presionar "Realizar Cobros".

### No aplica a

- Lógica de cancelación efectiva del pedido en Legacy o en el sistema de cumplimiento. La cancelación desde este módulo solo dispara el cambio de estado y la salida del pedido del listado de Validar Cobro.

---

## Reglas de Negocio

**Regla 1 — Aplicabilidad a ambas regiones con cartera por región**
La pantalla principal del módulo Validar Cobro muestra exclusivamente clientes de la región del usuario activo (México o Perú), según la asignación de cartera. La estructura del listado, el buscador y los botones contextuales son idénticos para ambas regiones, pero un usuario individual no ve clientes de regiones distintas a la suya. Las diferencias operativas entre regiones se manifiestan posteriormente al entrar al wizard del cliente.

**Regla 2 — Visibilidad filtrada por cartera y región del usuario**
El listado muestra únicamente clientes asignados a la cartera del usuario (campo Cobrador del Catálogo de Clientes) y cuya Región del cliente coincida con la región del usuario. Los clientes asignados a otros usuarios o de regiones distintas no son visibles ni accesibles desde el listado.

**Regla 3 — Inclusión de clientes con pendientes**
El listado incluye clientes que cumplan al menos una de estas condiciones: tienen uno o más cobros recibidos en el Buzón de Cobros pendientes de aplicar a alguna proforma o factura del cliente; o tienen una o más proformas o facturas emitidas pendientes de cobrar (con saldo pendiente positivo). Los clientes sin pendientes no aparecen en el listado.

**Regla 4 — Acción contextual según cobros recibidos**
La acción de cada cliente en el listado es "Realizar Cobros" si el cliente tiene uno o más cobros recibidos en el Buzón de Cobros pendientes de aplicar (al presionar, el usuario navega al wizard de tres pasos del cliente), o "Gestionar Cobranza" si el cliente no tiene cobros recibidos pendientes (cero cobros en el Buzón) pero sí tiene proformas/facturas por cobrar (al presionar, se abre el modal "Gestionar Cobranza").

**Regla 5 — Tooltip explicativo sobre estado sin cobros**
Cuando un cliente tiene cero cobros recibidos en el Buzón, el listado muestra un tooltip explicativo en el conteo de cobros recibidos (ejemplo: "No hay correos de cobro en el buzón de este cliente") para que el usuario comprenda por qué la acción es "Gestionar Cobranza" en lugar de "Realizar Cobros".

**Regla 6 — Buscador único por nombre de cliente o identificador fiscal**
El buscador del listado filtra en tiempo real según coincidencias del texto en el nombre del cliente o en el identificador fiscal (RFC para México, RUC para Perú). El filtrado opera sin requerir botón de búsqueda. El sistema ignora los espacios al inicio y al final del texto ingresado antes de ejecutar el filtrado (trim automático). Ver OBS-041.

**Regla 7 — Modal "Gestionar Cobranza" al presionar la acción**
Al presionar "Gestionar Cobranza" en un cliente, se abre el modal del cliente con: cabecera con nombre del cliente y Monto Total pendiente; listado de pedidos pendientes con datos por pedido (Pedido Interno, número de orden de compra o referencia del cliente, datos de contacto del cliente, fecha estimada de pago editable, botón Cancelar Pedido por pedido); y un botón "Confirmar" que guarda los cambios de fechas estimadas y cierra el modal.

**Regla 8 — Edición de fecha estimada de pago**
La fecha estimada de pago de un pedido se registra al presionar "Confirmar" en el modal "Gestionar Cobranza" y queda visible en el pedido para consulta posterior y en reportes operativos del módulo. Esta fecha es referencia operativa del equipo de Cobranza para seguimiento al cliente y no genera bloqueos automáticos en el sistema.

**Regla 9 — Cancelación de pedido por falta de pago desde Gestionar Cobranza**
Al presionar "Cancelar Pedido" en un pedido específico del modal "Gestionar Cobranza", el sistema cancela el pedido por falta de pago: el pedido sale del listado de Validar Cobro y queda con estado "Cancelado por falta de pago" para trazabilidad histórica. La cancelación es decisión manual del operador y no está condicionada por el sistema a vigencias automáticas (la vigencia de proforma 30 días y de factura mes corriente es referencia operativa, no bloqueo técnico). ** Pendiente confirmar si la cancelación dispara reversa de la proforma o factura asociada. **

**Regla 10 — Navegación al wizard al presionar "Realizar Cobros"**
Al presionar "Realizar Cobros" en un cliente, el sistema navega a la pantalla del wizard de tres pasos del cliente seleccionado (Paso 1 - Captura del Cobro como pantalla inicial). No es un modal: es una pantalla nueva con la cabecera del cliente y la barra de pasos del wizard.

---

## Riesgos

**Riesgo 1 — Modificación de fecha estimada sin trazabilidad de quién y cuándo**
La fecha estimada de pago es un campo de uso operativo que cambia frecuentemente. Si no se registra el historial de cambios, se pierde información de seguimiento. ** Pendiente confirmar si el registro del histórico de modificaciones de la fecha estimada (con usuario y timestamp) está dentro del alcance R16 o se trata como tema operativo posterior. **

**Riesgo 2 — Solapamiento de denominación de rol**
La denominación del rol que opera el módulo aparece como "Gestor de Cobranza" en la matriz del cliente y como "Analista de Cuentas por Cobrar" en sesiones de revisión de pantallas. ** Pendiente resolver formalmente la denominación canónica antes del desarrollo. **

**Riesgo 3 — Cliente Perú sin Buzón de Cobros equivalente operativo**
Para clientes Perú, el Buzón de Cobros y su flujo de recepción de correos de cobro depende de configuración operativa específica (cuentas bancarias Perú, formato de comprobantes de cobro peruanos, etc.) que tiene brechas pendientes. Si la operación Perú no tiene un Buzón de Cobros poblado, los clientes Perú aparecerán siempre con cero cobros recibidos y solo podrá usarse "Gestionar Cobranza" hasta que se resuelvan las brechas de cobro Perú.

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — ESTRUCTURA DEL LISTADO
═══════════════════════════════════════════════════════════════

**Criterio A1 — Columnas del listado**
Dado que el usuario accede al módulo Validar Cobro,
Cuando el sistema renderiza el listado de clientes,
Entonces deberá mostrar las siguientes columnas por cliente:
- Cliente (nombre del cliente).
- Identificador fiscal (RFC del cliente para Región México, RUC para Región Perú).
- Cobros recibidos (conteo de correos de cobro del Buzón pendientes de aplicar para este cliente).
- Factura / Proforma por Cobrar (conteo de proformas y facturas emitidas pendientes de cobrar para este cliente).
- Saldo Pendiente (monto total pendiente de cobro en USD). El listado siempre se muestra dolarizado en USD para homogeneizar la comparación entre clientes. Decisión confirmada por el cliente — OBS-046.
- Acción contextual (botón "Realizar Cobros" o "Gestionar Cobranza" según estado).

**Criterio A2 — Acción contextual visible por cliente**
Dado que un cliente del listado tiene uno o más cobros recibidos pendientes de aplicar,
Cuando el sistema renderiza la acción del cliente,
Entonces deberá mostrar el botón "Realizar Cobros" (estilizado para destacar acción primaria del flujo).

**Criterio A3 — Acción contextual cuando no hay cobros recibidos**
Dado que un cliente del listado tiene cero cobros recibidos pendientes,
Cuando el sistema renderiza la acción del cliente,
Entonces deberá mostrar el botón "Gestionar Cobranza" en lugar de "Realizar Cobros".

**Criterio A4 — Tooltip explicativo sobre cero cobros recibidos**
Dado que un cliente tiene cero cobros recibidos pendientes,
Cuando el usuario consulta el conteo de cobros recibidos del cliente,
Entonces el sistema deberá mostrar un tooltip explicativo (ejemplo: "No hay correos de cobro en el buzón de este cliente").

**Criterio A5 — Indicador de SLA 72 horas**
Dado que un cliente tiene uno o más cobros recibidos pendientes de aplicar,
Cuando el cobro más antiguo del cliente lleva más de 72 horas sin ser procesado (SLA de atención de cobros),
Entonces el sistema deberá mostrar un indicador visual de alerta (por ejemplo, ícono o resaltado) sobre el cliente en el listado para señalizar el vencimiento del SLA. Decisión confirmada por el cliente — OBS-047.

═══════════════════════════════════════════════════════════════
SECCIÓN B — BUSCADOR ÚNICO
═══════════════════════════════════════════════════════════════

**Criterio B1 — Buscador en tiempo real con trim automático**
Dado que el usuario interactúa con el campo buscador del listado,
Cuando ingresa texto,
Entonces el sistema deberá filtrar el listado de clientes en tiempo real conforme el usuario escribe, según coincidencias con nombre de cliente o identificador fiscal (RFC o RUC según región del cliente). El filtrado opera sin requerir botón de búsqueda. El sistema ignorará los espacios al inicio y al final del texto ingresado antes de ejecutar el filtrado (trim automático). Ver OBS-041.

**Criterio B2 — Sin filtros adicionales, ordenamiento por antigüedad de cobros**
Dado que el usuario opera el listado,
Cuando consulta los filtros disponibles,
Entonces el listado deberá ofrecer únicamente el buscador único. No hay filtros adicionales por estado, región, monto u otros criterios. El listado se ordena por defecto por antigüedad de los cobros recibidos pendientes de aplicar (el cliente con el cobro recibido más antiguo aparece primero). Decisión confirmada por el cliente — OBS-047.

═══════════════════════════════════════════════════════════════
SECCIÓN C — VISIBILIDAD POR CARTERA Y REGIÓN
═══════════════════════════════════════════════════════════════

**Criterio C1 — Filtro por Cobrador asignado y región del usuario**
Dado que el usuario accede al módulo,
Cuando el sistema arma el listado,
Entonces deberá incluir únicamente clientes que cumplan AMBAS condiciones: (a) el Cobrador asignado en el Catálogo de Clientes corresponde al usuario activo, y (b) la Región del cliente coincide con la región del usuario. Clientes asignados a otros Cobradores o de regiones distintas no aparecen en el listado.

═══════════════════════════════════════════════════════════════
SECCIÓN D — ACCIÓN "REALIZAR COBROS" Y NAVEGACIÓN AL WIZARD
═══════════════════════════════════════════════════════════════

**Criterio D1 — Navegación al wizard al presionar "Realizar Cobros"**
Dado que el usuario presiona "Realizar Cobros" en un cliente del listado,
Cuando el sistema procesa la acción,
Entonces deberá navegar a la pantalla del wizard de tres pasos del cliente seleccionado, abriendo directamente el Paso 1 (Captura del Cobro). La navegación es a pantalla nueva (NO es un modal sobre la pantalla actual).

═══════════════════════════════════════════════════════════════
SECCIÓN E — MODAL "GESTIONAR COBRANZA"
═══════════════════════════════════════════════════════════════

**Criterio E1 — Apertura del modal al presionar "Gestionar Cobranza"**
Dado que el usuario presiona "Gestionar Cobranza" en un cliente del listado,
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal "Gestionar Cobranza" sobre la pantalla del listado.

**Criterio E2 — Cabecera del modal**
Dado que el modal "Gestionar Cobranza" está abierto,
Cuando el sistema arma la cabecera,
Entonces deberá mostrar: nombre del cliente y Monto Total pendiente del cliente (con moneda).

**Criterio E3 — Listado de pedidos del cliente en el modal**
Dado que el modal está abierto,
Cuando el sistema arma el listado de pedidos pendientes del cliente,
Entonces deberá mostrar por cada pedido:
- Pedido Interno.
- Referencia del pedido del cliente (número de orden de compra u otro identificador del cliente, si está capturado).
- Datos del contacto del cliente: nombre, correo electrónico, teléfono.
- Fecha estimada de pago (datepicker editable).
- Botón "Cancelar Pedido" (acción de cancelación directa de este pedido específico).

**Criterio E4 — Edición de fecha estimada de pago y confirmación**
Dado que el usuario modifica la fecha estimada de pago de uno o más pedidos en el modal,
Cuando presiona "Confirmar" abajo del modal,
Entonces el sistema deberá guardar las fechas estimadas actualizadas en los pedidos correspondientes y cerrar el modal.

**Criterio E5 — Cancelación de pedido por falta de pago**
Dado que el usuario presiona "Cancelar Pedido" en un pedido del modal,
Cuando el sistema procesa la acción,
Entonces deberá cancelar el pedido por falta de pago: el pedido cambia a estado "Cancelado por falta de pago" y sale del listado de Validar Cobro y del modal "Gestionar Cobranza" del cliente. La cancelación queda registrada con trazabilidad de quién la ejecutó y cuándo. ** Pendiente confirmar si la cancelación dispara cancelación de la proforma o factura asociada y si propaga cancelación y/o transferencias a otros sistemas (Legacy). **

**Criterio E6 — Cancelación sin restricción de vigencia automática**
Dado que un pedido aparece en el modal "Gestionar Cobranza",
Cuando el usuario presiona "Cancelar Pedido" sobre ese pedido,
Entonces el sistema deberá permitir la cancelación sin restricción automática por vigencia (proforma 30 días, factura mes corriente). La vigencia es referencia operativa para el usuario, no bloqueo técnico del sistema. La decisión de cancelación es del operador.

**Criterio E7 — Cierre del modal sin guardar cambios**
Dado que el usuario abrió el modal y realizó cambios pero NO presionó "Confirmar",
Cuando cierra el modal con la X del modal o navega fuera,
Entonces el sistema deberá descartar los cambios de fecha estimada no confirmados. Las cancelaciones de pedido individuales son acciones inmediatas (no requieren "Confirmar" del modal).

---

## Notas Adicionales

- Esta fila documenta la pantalla principal del módulo Validar Cobro: listado de clientes con pendientes, buscador, botones contextuales y modal "Gestionar Cobranza".
- La pantalla principal aplica tanto a Región México como a Región Perú con estructura idéntica (mismas columnas, mismo buscador, mismos botones contextuales). Sin embargo, la cartera del Cobrador es por región: cada usuario opera clientes de UNA sola región (la que tenga asignada en su configuración como Cobrador). Por lo tanto, el listado que ve cada usuario muestra exclusivamente clientes de su región y de su cartera; NO se mezclan clientes México y Perú en una misma vista. Las diferencias regionales operativas (timbrado SAT vs SUNAT, monedas MXN/PEN, formato de identificador fiscal RFC/RUC, etc.) se manifiestan al entrar al wizard del cliente.
- El botón "Realizar Cobros" conduce al wizard de tres pasos (pantalla nueva, no modal): Paso 1 - Captura del Cobro, Paso 2 - Asociación a Proforma/Factura, Paso 3 - Facturación y Envío.
- El botón "Gestionar Cobranza" abre un modal sobre el listado con la información operativa del cliente y sus pedidos pendientes, permitiendo registrar fechas estimadas de pago y cancelar pedidos por falta de pago.
- La cancelación de pedido por falta de pago desde "Gestionar Cobranza" es la vía operativa preventiva (cliente que aún no ha pagado nada). La cancelación reactiva (cliente que pagó pero el pago es insuficiente o no cumple) se gestiona dentro del wizard como inconsistencia del cobro (no aplica en esta pantalla).
- La fecha estimada de pago capturada en el modal es referencia operativa del equipo de Cobranza para seguimiento al cliente. NO genera bloqueos automáticos en el sistema.
- La vigencia de proforma (30 días) y de factura (mes corriente) confirmada por el cliente para cancelaciones por falta de pago es referencia operativa, no bloqueo técnico del sistema. El operador puede cancelar antes o después según criterio.
- El buscador único filtra por nombre de cliente o identificador fiscal (RFC en México, RUC en Perú) en tiempo real. No hay filtros adicionales por estado, monto, región u otros criterios.
- ** Pendiente resolver formalmente la denominación canónica del rol operativo entre "Gestor de Cobranza" y "Analista de Cuentas por Cobrar". **
- ** Pendiente confirmar si la cancelación de pedido desde "Gestionar Cobranza" dispara cancelación de la proforma o factura asociada y si propaga cancelación y/o transferencias a otros sistemas (Legacy). **
- **Decisión OBS-046:** el Saldo Pendiente del listado se muestra siempre dolarizado en USD para homogeneizar la comparación entre clientes, independientemente de la moneda de facturación de cada cliente.
- **Decisión OBS-047:** el orden por defecto del listado es por antigüedad de los cobros recibidos pendientes de aplicar (el cliente con el cobro más antiguo aparece primero). Los clientes sin cobros recibidos (acción "Gestionar Cobranza") se muestran al final del listado. Adicionalmente, los clientes cuyo cobro más antiguo supera las 72 horas de SLA reciben un indicador visual de alerta (Criterio A5).
- ** Pendiente confirmar si la fecha estimada de pago registra historial de cambios con usuario y timestamp para trazabilidad de seguimiento. **
- ** Para clientes Región Perú, el correcto funcionamiento de esta pantalla depende del Buzón de Cobros Perú (con sus brechas pendientes de modelo bancario peruano y formatos de comprobantes). Si el Buzón Perú no está poblado, los clientes Perú siempre aparecerán con cero cobros recibidos hasta que se resuelvan las brechas correspondientes. **

---

## Cambios

| # | Fecha | Referencia | Descripción del cambio |
|---|-------|------------|------------------------|
| 1 | 2026-06-10 | OBS-041 | Regla 6: trim automático agregado al buscador. Criterio B1: actualizado con trim automático. |
| 2 | 2026-06-10 | OBS-046 | Criterio A1: Saldo Pendiente siempre en USD. Pendiente de moneda cerrado. |
| 3 | 2026-06-10 | OBS-047 | Criterio B2: ordenamiento por defecto por antigüedad de cobros recibidos. Criterio A5 agregado: indicador visual SLA 72 horas. Pendiente de orden cerrado. |
