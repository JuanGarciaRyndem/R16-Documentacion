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

Yo como **Gestor de Cobranza** (función operativa: Analista de Cuentas por Cobrar), quiero contar con el listado principal de Validar Cobro que muestre los clientes de mi cartera con pendientes y me ofrezca la acción adecuada según su estado (realizar cobros o gestionar cobranza), para priorizar y dar seguimiento al cobro de cada cliente desde un único punto de entrada.

**Descripción:**
El sistema debe mostrar el listado de clientes de la cartera del usuario que tengan pendientes (cobros recibidos pendientes de aplicar, generados por la clasificación automática del correo, o proformas/facturas pendientes de cobrar). Por cada cliente, el sistema ofrece una acción contextual según su estado: cuando existen cobros recibidos pendientes de aplicar, conduce al flujo de tres pasos (Captura → Asociación → Facturación y Envío); cuando no hay cobros recibidos, abre la gestión de cobranza del cliente. El listado es estructuralmente el mismo para Región México y Región Perú, con visibilidad filtrada por la cartera asignada al usuario, que opera clientes de una sola región. Las diferencias operativas entre regiones se manifiestan dentro del flujo de tres pasos, documentadas en requisitos independientes.

---

## Alcance

### Aplica a

- Listado principal del módulo Validar Cobro: clientes con pendientes de cobranza.
- El listado aplica tanto a Región México como a Región Perú con estructura idéntica. La cartera del Cobrador es por región: cada usuario opera clientes de UNA sola región. El listado de cada usuario muestra únicamente clientes de su región y de su cartera (no se mezclan regiones en una misma vista).
- Listado agrupado por cliente con conteos de cobros recibidos pendientes de aplicar y de proformas/facturas pendientes de cobrar.
- Buscador único por nombre de cliente o identificador fiscal (RFC para México, RUC para Perú).
- Ordenamiento del listado por columna (Cliente, Factura o Proforma por Cobrar, Saldo Pendiente), aplicado por el usuario de una columna a la vez (ver Regla 13).
- Acción contextual por cliente según estado:
  - "Realizar Cobros": cuando el cliente tiene uno o más cobros recibidos pendientes de aplicar (generados por la clasificación automática del correo).
  - "Gestionar Cobranza": cuando el cliente NO tiene cobros recibidos (cero cobros) pero sí tiene proformas/facturas pendientes de cobrar.
- Gestión de cobranza del cliente: listado de sus pedidos pendientes con el documento asociado (Proforma o Factura) y su monto, datos de contacto, fecha estimada de pago editable por pedido, y cancelación de pedido por falta de pago cuando el pedido la admite (ver Regla 10).
- Notificación por correo electrónico al contacto del pedido al cancelarlo, y advertencia al usuario, al confirmar la cancelación, de que se solicitará la cancelación fiscal del comprobante ante el SAT cuando el pedido tiene factura emitida (ver Regla 11).
- Visibilidad filtrada por cartera del usuario (campo Cobrador del Catálogo de Clientes) combinado con la región operativa del usuario.
- Información explicativa sobre el estado del cliente cuando aplica (por ejemplo: cliente sin cobros recibidos pendientes).
- Salida hacia el flujo de tres pasos (Captura del Cobro → Asociación → Facturación y Envío) al elegir "Realizar Cobros".

### No aplica a

- Lógica de cancelación efectiva del pedido en Legacy o en el sistema de cumplimiento. La cancelación desde este módulo solo dispara el cambio de estado y la salida del pedido del listado de Validar Cobro.
- Seguimiento del resultado de la solicitud de cancelación fiscal de la factura ante el SAT: se solicita al cancelar el pedido, pero su resultado y los reintentos ante un eventual rechazo del receptor se gestionan fuera del sistema.

---

## Reglas de Negocio

**Regla 1 — Aplicabilidad a ambas regiones con cartera por región**
La pantalla principal del módulo Validar Cobro muestra exclusivamente clientes de la región del usuario activo (México o Perú), según la asignación de cartera. La estructura del listado, el buscador y las acciones contextuales son idénticas para ambas regiones, pero un usuario individual no ve clientes de regiones distintas a la suya. Las diferencias operativas entre regiones se manifiestan posteriormente al entrar al flujo del cliente.

**Regla 2 — Visibilidad filtrada por cartera y región del usuario**
El listado muestra únicamente clientes asignados a la cartera del usuario (campo Cobrador del Catálogo de Clientes) y cuya Región del cliente coincida con la región del usuario. Los clientes asignados a otros usuarios o de regiones distintas no son visibles ni accesibles desde el listado.

**Regla 3 — Inclusión de clientes con pendientes**
El listado incluye clientes que cumplan al menos una de estas condiciones: tienen uno o más cobros recibidos pendientes de aplicar a alguna proforma o factura del cliente (generados por la clasificación automática del correo); o tienen una o más proformas o facturas emitidas pendientes de cobrar (con saldo pendiente positivo). Los clientes sin pendientes no aparecen en el listado.

**Regla 4 — Acción contextual según cobros recibidos**
La acción de cada cliente en el listado es "Realizar Cobros" si el cliente tiene uno o más cobros recibidos pendientes de aplicar (al elegirla, el usuario avanza al flujo de tres pasos del cliente), o "Gestionar Cobranza" si el cliente no tiene cobros recibidos pendientes (cero cobros) pero sí tiene proformas/facturas por cobrar (al elegirla, se abre la gestión de cobranza del cliente).

**Regla 5 — Información explicativa sobre estado sin cobros**
Cuando un cliente tiene cero cobros recibidos pendientes, el listado muestra información explicativa junto al conteo de cobros recibidos (ejemplo: "No hay cobros recibidos pendientes para este cliente") para que el usuario comprenda por qué la acción es "Gestionar Cobranza" en lugar de "Realizar Cobros".

**Regla 6 — Buscador único por nombre de cliente o identificador fiscal**
El buscador del listado filtra en tiempo real según coincidencias del texto en el nombre del cliente o en el identificador fiscal (RFC para México, RUC para Perú), sin requerir una acción explícita de búsqueda. El sistema ignora los espacios al inicio y al final del texto ingresado antes de ejecutar el filtrado (trim automático). Ver OBS-041.

**Regla 7 — Gestión de cobranza del cliente al elegir esa acción**
Al elegir "Gestionar Cobranza" para un cliente, el sistema presenta el nombre del cliente, su Monto Total pendiente, y el listado de sus pedidos pendientes con, por cada pedido: el documento asociado (Proforma o Factura) y su monto, el Pedido Interno, el número de orden de compra o referencia del cliente, los datos de contacto del cliente, la fecha estimada de pago (editable), y la opción de cancelar el pedido cuando admite cancelación (ver Regla 10). Confirmar guarda los cambios de fechas estimadas.

**Regla 8 — Edición de fecha estimada de pago**
La fecha estimada de pago de un pedido se registra al confirmar los cambios en la gestión de cobranza del cliente y queda visible en el pedido para consulta posterior y en reportes operativos del módulo. Esta fecha es referencia operativa del equipo de Cobranza para seguimiento al cliente y no genera bloqueos automáticos en el sistema. El sistema conserva en base de datos el valor vigente y el inmediatamente anterior de la fecha estimada, cada uno con su usuario y fecha de registro — no es bitácora completa — sin vista en pantalla dedicada; se integra a la bitácora general de movimientos del sistema.

**Regla 9 — Cancelación de pedido por falta de pago desde Gestionar Cobranza**
Al elegir cancelar un pedido específico en la gestión de cobranza del cliente, el sistema cancela el pedido por falta de pago: el pedido sale del listado de Validar Cobro y queda con estado "Cancelado por falta de pago" para trazabilidad histórica. La cancelación es decisión manual del operador y no está condicionada por el sistema a vigencias automáticas (la vigencia de proforma 30 días y de factura mes corriente es referencia operativa, no bloqueo técnico). La cancelación no propaga hacia otros sistemas (Legacy), ya que hasta este punto del flujo no se ha transferido información del pedido ni del cobro. Cuando el pedido tiene una factura emitida, la cancelación además solicita la cancelación fiscal de la factura ante el SAT; el sistema no rastrea el resultado de esa solicitud. Si el receptor rechaza la cancelación fiscal, los reintentos se gestionan fuera del sistema, sin pantalla dedicada.

**Regla 10 — Alcance de la cancelación según avance del pedido**
La opción de cancelar un pedido solo se ofrece mientras el pedido no se haya tramitado y su factura no tenga ningún cobro aplicado. Deja de ofrecerse cuando el pedido ya se tramitó con un faltante dentro del monto aplazable, caso en el que la factura queda parcialmente cobrada y solo resta que el cliente complete su pago; en ese escenario el seguimiento de cobro continúa por la vía normal del flujo de tres pasos, no por cancelación.

**Regla 11 — Aviso de la cancelación al cliente y advertencia de cancelación fiscal**
Al cancelar un pedido, el sistema envía una notificación por correo electrónico al contacto del pedido, con destinatario y copias editables por el usuario, como en los demás envíos del sistema. Cuando el pedido tiene una factura emitida, el sistema advierte al usuario, al confirmar la cancelación, que se solicitará la cancelación del comprobante ante el SAT y que el seguimiento de esa solicitud se realiza fuera del sistema.

**Regla 12 — Acceso al flujo de tres pasos al elegir "Realizar Cobros"**
Al elegir "Realizar Cobros" para un cliente, el sistema conduce al flujo de tres pasos del cliente seleccionado, iniciando en el Paso 1 - Captura del Cobro, con el contexto del cliente disponible durante todo el flujo.

**Regla 13 — Ordenamiento por columna**
El usuario puede ordenar el listado por columna: Cliente, Factura o Proforma por Cobrar, o Saldo Pendiente. El ordenamiento se aplica de una columna a la vez y sustituye temporalmente al orden predeterminado por antigüedad de los cobros recibidos pendientes (Criterio B2) mientras esté activo.

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
- Cobros recibidos (conteo de cobros recibidos pendientes de aplicar para este cliente, generados por la clasificación automática del correo).
- Factura / Proforma por Cobrar (conteo de proformas y facturas emitidas pendientes de cobrar para este cliente).
- Saldo Pendiente (monto total pendiente de cobro en USD). El listado siempre se muestra dolarizado en USD para homogeneizar la comparación entre clientes. Decisión confirmada por el cliente — OBS-046.
- Acción contextual ("Realizar Cobros" o "Gestionar Cobranza" según estado).

**Criterio A2 — Acción contextual visible por cliente**
Dado que un cliente del listado tiene uno o más cobros recibidos pendientes de aplicar,
Cuando el sistema presenta la acción del cliente,
Entonces deberá ofrecer al usuario la acción "Realizar Cobros" como la acción principal para ese cliente.

**Criterio A3 — Acción contextual cuando no hay cobros recibidos**
Dado que un cliente del listado tiene cero cobros recibidos pendientes,
Cuando el sistema presenta la acción del cliente,
Entonces deberá ofrecer la acción "Gestionar Cobranza" en lugar de "Realizar Cobros".

**Criterio A4 — Información explicativa sobre cero cobros recibidos**
Dado que un cliente tiene cero cobros recibidos pendientes,
Cuando el usuario consulta el conteo de cobros recibidos del cliente,
Entonces el sistema deberá mostrar información explicativa (ejemplo: "No hay cobros recibidos pendientes para este cliente").

**Criterio A5 — Indicador de SLA 72 horas**
Dado que un cliente tiene uno o más cobros recibidos pendientes de aplicar,
Cuando el cobro más antiguo del cliente lleva más de 72 horas sin ser procesado (SLA de atención de cobros),
Entonces el sistema deberá señalizar al cliente en el listado el vencimiento del SLA. Decisión confirmada por el cliente — OBS-047.

**Criterio A6 — Ordenamiento del listado por columna**
Dado que el usuario opera el listado,
Cuando interactúa con una columna ordenable (Cliente, Factura o Proforma por Cobrar, Saldo Pendiente),
Entonces el sistema deberá reordenar el listado según esa columna, en orden ascendente o descendente.

**Criterio A7 — Ordenamiento de una columna a la vez, sustituye el orden por antigüedad**
Dado que el usuario aplica el ordenamiento por columna,
Cuando selecciona una columna distinta a la que tiene el orden activo,
Entonces el sistema deberá aplicar el nuevo ordenamiento a una columna a la vez, sustituyendo el orden predeterminado por antigüedad de los cobros recibidos (Criterio B2) mientras el ordenamiento por columna esté activo.

═══════════════════════════════════════════════════════════════
SECCIÓN B — BUSCADOR ÚNICO
═══════════════════════════════════════════════════════════════

**Criterio B1 — Buscador en tiempo real con trim automático**
Dado que el usuario interactúa con el campo buscador del listado,
Cuando ingresa texto,
Entonces el sistema deberá filtrar el listado de clientes en tiempo real conforme el usuario escribe, según coincidencias con nombre de cliente o identificador fiscal (RFC o RUC según región del cliente), sin requerir una acción explícita de búsqueda. El sistema ignorará los espacios al inicio y al final del texto ingresado antes de ejecutar el filtrado (trim automático). Ver OBS-041.

**Criterio B2 — Sin filtros adicionales, ordenamiento por antigüedad de cobros**
Dado que el usuario opera el listado,
Cuando consulta los filtros disponibles,
Entonces el listado deberá ofrecer únicamente el buscador único. No hay filtros adicionales por estado, región, monto u otros criterios. El listado se ordena por defecto por antigüedad de los cobros recibidos pendientes de aplicar (el cliente con el cobro recibido más antiguo aparece primero). Decisión confirmada por el cliente — OBS-047. El usuario puede sustituir este orden aplicando el ordenamiento por columna (Regla 13, Criterios A6–A7).

═══════════════════════════════════════════════════════════════
SECCIÓN C — VISIBILIDAD POR CARTERA Y REGIÓN
═══════════════════════════════════════════════════════════════

**Criterio C1 — Filtro por Cobrador asignado y región del usuario**
Dado que el usuario accede al módulo,
Cuando el sistema arma el listado,
Entonces deberá incluir únicamente clientes que cumplan AMBAS condiciones: (a) el Cobrador asignado en el Catálogo de Clientes corresponde al usuario activo, y (b) la Región del cliente coincide con la región del usuario. Clientes asignados a otros Cobradores o de regiones distintas no aparecen en el listado.

═══════════════════════════════════════════════════════════════
SECCIÓN D — ACCIÓN "REALIZAR COBROS" Y ACCESO AL FLUJO
═══════════════════════════════════════════════════════════════

**Criterio D1 — Acceso al flujo al elegir "Realizar Cobros"**
Dado que el usuario elige "Realizar Cobros" para un cliente del listado,
Cuando el sistema procesa la acción,
Entonces deberá conducir al flujo de tres pasos del cliente seleccionado, iniciando en el Paso 1 (Captura del Cobro), manteniendo el contexto del cliente durante todo el flujo.

═══════════════════════════════════════════════════════════════
SECCIÓN E — GESTIÓN DE COBRANZA DEL CLIENTE
═══════════════════════════════════════════════════════════════

**Criterio E1 — Acceso a la gestión de cobranza al elegir esa acción**
Dado que el usuario elige "Gestionar Cobranza" para un cliente del listado,
Cuando el sistema procesa la acción,
Entonces deberá presentar la gestión de cobranza de ese cliente sobre el listado.

**Criterio E2 — Información inicial del cliente**
Dado que el usuario está en la gestión de cobranza del cliente,
Cuando el sistema presenta la información inicial,
Entonces deberá mostrar el nombre del cliente y su Monto Total pendiente, dolarizado en USD, de forma consistente con el Saldo Pendiente del listado (Criterio A1).

**Criterio E3 — Listado de pedidos del cliente**
Dado que el usuario está en la gestión de cobranza del cliente,
Cuando el sistema arma el listado de sus pedidos pendientes,
Entonces deberá mostrar por cada pedido:
- Documento asociado (Proforma o Factura) y su monto.
- Pedido Interno.
- Referencia del pedido del cliente (número de orden de compra u otro identificador del cliente, si está capturado).
- Datos del contacto del cliente: nombre, correo electrónico, teléfono.
- Fecha estimada de pago, editable por el usuario.
- La opción de cancelar ese pedido, cuando admite cancelación (ver Regla 10 / Criterio E7).

**Criterio E4 — Edición de fecha estimada de pago y confirmación**
Dado que el usuario modifica la fecha estimada de pago de uno o más pedidos en la gestión de cobranza del cliente,
Cuando confirma los cambios,
Entonces el sistema deberá guardar las fechas estimadas actualizadas en los pedidos correspondientes, conservando en base de datos el valor vigente y el inmediatamente anterior de cada fecha (con su usuario y fecha de registro, sin vista en pantalla dedicada).

**Criterio E5 — Cancelación de pedido por falta de pago**
Dado que el usuario cancela un pedido en la gestión de cobranza del cliente,
Cuando el sistema procesa la acción,
Entonces deberá cancelar el pedido por falta de pago: el pedido cambia a estado "Cancelado por falta de pago" y sale del listado de Validar Cobro y de la gestión de cobranza del cliente. La cancelación queda registrada con trazabilidad de quién la ejecutó y cuándo. La cancelación no propaga hacia otros sistemas (Legacy), dado que no se ha transferido previamente información del pedido ni del cobro. Cuando el pedido tiene una factura emitida, el sistema solicita adicionalmente la cancelación fiscal de la factura ante el SAT, sin rastrear el resultado de esa solicitud; si el receptor la rechaza, los reintentos se gestionan fuera del sistema.

**Criterio E6 — Cancelación sin restricción de vigencia automática**
Dado que un pedido admite cancelación conforme a su avance (Regla 10, Criterio E7),
Cuando el usuario cancela ese pedido,
Entonces el sistema deberá permitir la cancelación sin restricción automática por vigencia (proforma 30 días, factura mes corriente). La vigencia es referencia operativa para el usuario, no bloqueo técnico del sistema. La decisión de cancelación es del operador.

**Criterio E7 — Disponibilidad de la cancelación según avance del pedido**
Dado que un pedido está en la gestión de cobranza del cliente,
Cuando el pedido no se ha tramitado y su factura no tiene ningún cobro aplicado,
Entonces el sistema deberá ofrecer al usuario la opción de cancelar ese pedido.

**Criterio E8 — Cancelación no disponible para pedido tramitado con faltante aplazable**
Dado que un pedido ya se tramitó con un faltante dentro del monto aplazable (su factura queda parcialmente cobrada),
Cuando el sistema arma el listado de pedidos del cliente,
Entonces NO deberá ofrecer la opción de cancelar ese pedido; el seguimiento de cobro continúa por la vía normal del flujo de tres pasos.

**Criterio E9 — Notificación al cliente por correo al cancelar**
Dado que el usuario cancela un pedido desde la gestión de cobranza del cliente,
Cuando el sistema procesa la cancelación,
Entonces deberá enviar una notificación por correo electrónico al contacto del pedido, con destinatario y copias editables por el usuario antes del envío, igual que en los demás envíos del sistema.

**Criterio E10 — Advertencia de cancelación fiscal cuando hay factura emitida**
Dado que el pedido a cancelar tiene una factura emitida,
Cuando el usuario confirma la cancelación,
Entonces el sistema deberá advertir al usuario que se solicitará la cancelación del comprobante ante el SAT y que el seguimiento de esa solicitud se realiza fuera del sistema.

**Criterio E11 — Descarte de cambios no confirmados**
Dado que el usuario realizó cambios de fecha estimada en la gestión de cobranza del cliente pero no los confirmó,
Cuando sale de esa vista sin confirmar,
Entonces el sistema deberá descartar los cambios de fecha estimada no confirmados. Las cancelaciones de pedido son acciones inmediatas y no requieren confirmación adicional.

---

## Notas Adicionales

- Esta fila documenta el listado principal del módulo Validar Cobro: clientes con pendientes, buscador, ordenamiento, acciones contextuales y gestión de cobranza.
- El listado aplica tanto a Región México como a Región Perú con estructura idéntica (mismas columnas, mismo buscador, mismas acciones contextuales). La cartera del Cobrador es por región: cada usuario opera clientes de una sola región, por lo que su listado muestra exclusivamente clientes de su región y de su cartera. Las diferencias regionales operativas (timbrado SAT vs SUNAT, monedas MXN/PEN, formato de identificador fiscal RFC/RUC, etc.) se manifiestan dentro del flujo de tres pasos.
- "Realizar Cobros" conduce al flujo de tres pasos: Paso 1 - Captura del Cobro, Paso 2 - Asociación a Proforma/Factura, Paso 3 - Facturación y Envío.
- "Gestionar Cobranza" presenta la información operativa del cliente y sus pedidos pendientes, permitiendo registrar fechas estimadas de pago y cancelar los pedidos que admiten cancelación.
- La cancelación por falta de pago desde "Gestionar Cobranza" es la vía preventiva (cliente que aún no ha pagado nada). La cancelación reactiva (cliente que pagó, pero de forma insuficiente o que no cumple) se gestiona dentro del flujo de tres pasos como inconsistencia del cobro; no aplica en este listado. La devolución de dinero tampoco aplica aquí, ya que la gestión de cobranza solo alcanza a clientes sin cobros pendientes de aplicar.
- La cancelación de un pedido puede originarse en dos flujos — este listado y el paso de asociación del flujo de cobro — y ambos llegan al mismo estado ("Cancelado por falta de pago").
- La fecha estimada de pago es referencia operativa del equipo de Cobranza; no genera bloqueos automáticos. La vigencia de proforma (30 días) y de factura (mes corriente) es igualmente referencia operativa, no bloqueo técnico.
- El buscador único filtra por nombre de cliente o identificador fiscal (RFC en México, RUC en Perú) en tiempo real; no hay filtros adicionales por estado, monto, región u otros criterios.
- **Decisión OBS-046:** el Saldo Pendiente y el Monto Total pendiente del cliente se muestran siempre dolarizados en USD. **Trazabilidad (DUDA-067):** la conversión usa el tipo de cambio de cada documento origen (proforma/factura), no el del día de consulta ni uno unificado — detalle técnico en `_BD.md` / `-Back.md`.
- **Decisión OBS-047:** el orden por defecto del listado es por antigüedad de los cobros recibidos pendientes de aplicar (el cliente con el cobro más antiguo aparece primero); los clientes sin cobros recibidos se muestran al final. El usuario puede sustituir este orden aplicando el ordenamiento por columna (Regla 13). Los clientes cuyo cobro más antiguo supera las 72 horas de SLA reciben una señal de alerta (Criterio A5).

---

## Cambios

| # | Fecha | Referencia | Descripción del cambio |
|---|-------|------------|------------------------|
| 1 | 2026-06-10 | OBS-041 | Regla 6: trim automático agregado al buscador. Criterio B1: actualizado con trim automático. |
| 2 | 2026-06-10 | OBS-046 | Criterio A1: Saldo Pendiente siempre en USD. Pendiente de moneda cerrado. |
| 3 | 2026-06-10 | OBS-047 | Criterio B2: ordenamiento por defecto por antigüedad de cobros recibidos. Criterio A5 agregado: indicador visual SLA 72 horas. Pendiente de orden cerrado. |
| 4 | 2026-08-21 | DUDA-066 | Riesgo 1 y Notas Adicionales: histórico de fecha estimada de pago cerrado — solo 2 valores en BD (actual + anterior), no bitácora completa, sin pantalla dedicada. |
| 5 | 2026-08-21 | DUDA-068 | Criterio E5 y Notas Adicionales: cerrada la pregunta sobre propagación a Legacy al cancelar pedido — no aplica porque no hay transferencia previa a Legacy. |
| 6 | 2026-08-21 | DUDA-069 | Riesgo 3 y Notas Adicionales: brechas de Buzón de Cobros Perú cerradas (ver DUDA-001, DUDA-018) — ya llegan cobros de Perú al listado. |
| 7 | 2026-08-21 | DUDA-047 | Riesgo 2 y Notas Adicionales: homologada la denominación del rol operativo a "Gestor de Cobranza"; se eliminan las menciones a "Analista de Cuentas por Cobrar" como nombre del rol. |
| 8 | 2026-09-11 | Turno 4 | Alineación con el rediseño del manejo de correos: se retiran las referencias al Buzón de Cobros en Alcance, Reglas 3-5 y Criterios A1/A4; los cobros recibidos llegan como pendientes generados por la clasificación automática del correo. Se elimina el Riesgo 3 (Buzón Perú), sin objeto. Se elimina el Riesgo 2 (solapamiento de denominación): la Historia de Usuario ya define rol y función operativa. Se elimina el Riesgo 1 (historial de fecha estimada), incorporado a la Regla 8 / Criterio E4 (dos valores en BD, sin vista en pantalla). Se retira la sección de Riesgos, sin contenido tras estas resoluciones. |
| 9 | 2026-09-11 | Turno 4 | Regla 9 y Criterio E5: la cancelación no propaga hacia Legacy (nada transferido aún); cuando el pedido tiene factura emitida, además solicita su cancelación fiscal ante el SAT sin rastrear el resultado, y los reintentos ante un rechazo del receptor se gestionan fuera del sistema. Alcance ("No aplica a"): se agrega el no seguimiento de esa solicitud. |
| 10 | 2026-09-11 | Turno 4 | Nueva Regla 10 (alcance de la cancelación según avance del pedido) y Criterios E7-E8: la cancelación deja de ofrecerse cuando el pedido ya se tramitó con un faltante dentro del monto aplazable. Nueva Regla 11 (avisos de la cancelación) y Criterios E9-E10: notificación por correo al contacto del pedido y advertencia de cancelación fiscal SAT cuando hay factura. La antigua Regla 10 (navegación al flujo) se renumera a Regla 12; el antiguo Criterio E7 (cierre sin guardar) se renumera a Criterio E11. Criterio E6 acotado a pedidos que admiten cancelación. |
| 11 | 2026-09-11 | Turno 4 | Nueva Regla 13 y Criterios A6-A7: ordenamiento del listado por columna (Cliente, Factura o Proforma por Cobrar, Saldo Pendiente), de una columna a la vez, que sustituye temporalmente el orden por antigüedad (Criterio B2). Regla 7 y Criterio E3: se incorpora el documento asociado y su monto al listado de pedidos pendientes. Criterio E2: Monto Total pendiente dolarizado, consistente con el listado de clientes. |
| 12 | 2026-09-11 | Turno 4 | Correcciones de consistencia: se reescriben Requisito, Alcance, Reglas, Criterios y Notas Adicionales retirando el lenguaje de pantallas y controles (modal, wizard, botones, tooltip, datepicker, cabecera), describiendo en su lugar la información presentada y las acciones del usuario. Se corrige la numeración de reglas y criterios resultante de las inserciones anteriores. |
