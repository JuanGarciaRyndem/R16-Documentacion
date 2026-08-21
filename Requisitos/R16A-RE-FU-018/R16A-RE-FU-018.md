# R16A-RE-FU-018 — Factura por Adelantado

| Campo                   | Valor                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ID**                  | R16A-RE-FU-018                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Título**              | Factura por Adelantado                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Módulo / Épica**      | Factura por Adelantado                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Historia de Usuario** | Yo como **Gestor de Cobranza**, quiero contar con una pantalla inicial del módulo Factura por Adelantado que liste agrupadamente los clientes que tienen al menos un pedido pendiente de facturar por adelantado, mostrando su Razón Social, identificador fiscal, número de facturas pendientes y monto total dolarizado, para identificar rápidamente qué clientes requieren atención y navegar al detalle de sus pedidos pendientes. |
| **Prioridad**           | Alta                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| **Estado**              | Propuesto                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **Requisito asociado**  | R16.2M-RE-FU-001                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |

---

## Requisito Funcional

El sistema debe contar con una pantalla inicial en el módulo Factura por Adelantado que presente un listado, agrupado por cliente, de los pedidos pendientes de facturar por adelantado. El listado se ordena por antigüedad del pendiente (los más antiguos primero) y su visibilidad se filtra por la cartera del usuario operativo (campo Cobrador asignado en el Catálogo de Clientes). La pantalla permite al usuario identificar rápidamente qué clientes tienen facturas pendientes y navegar al detalle de cada uno para gestionarlas.

---

## Alcance

### Aplica a

- Módulo NUEVO en PQF2 R16: Factura por Adelantado (módulo no existe en la versión actual del sistema).
- Pantalla inicial del módulo: listado agrupado por cliente con pedidos pendientes de facturar por adelantado.
- Pedidos elegibles: aquellos con estado "pendiente de Factura por Adelantado" generados desde el módulo Tramitar Pedido en cualquiera de los dos flujos que requieren Factura por Adelantado: Crédito con Factura por Adelantado y Prepago con Factura por Adelantado.
- Visualización agrupada por cliente con conteo de pedidos pendientes y monto total dolarizado.
- Buscador único por Razón Social, identificador fiscal o número de pedido interno.
- Ordenamiento por antigüedad del pendiente.
- Visibilidad filtrada por cartera del usuario operativo (campo Cobrador del Catálogo de Clientes).
- Paginación numerada con navegación anterior/siguiente.
- Estado vacío cuando no hay pendientes en la cartera del usuario.
- Aplicación a clientes de México y Perú.

### No aplica a

- Pedidos con Sustancias Controladas tipo Mundial, Nacional u Origen (regla del cliente que prohíbe Factura por Adelantado para controlados; no llegan a este módulo desde Tramitar Pedido).
- Filtros adicionales al buscador único (por moneda, empresa emisora, fecha, etc.). La pantalla privilegia ligereza operativa.

---

## Reglas de Negocio

Regla 1 — Origen de los pedidos elegibles
El listado de pendientes considera exclusivamente pedidos en estado "pendiente de Factura por Adelantado" generados desde el módulo Tramitar Pedido, incluyendo los del flujo Crédito con Factura por Adelantado y los del flujo Prepago con Factura por Adelantado. Los pedidos sin pendiente de Factura por Adelantado o ya facturados y enviados no aparecen en el listado.

Regla 2 — Agrupación por cliente con conteo de pendientes
El listado se presenta agrupado por cliente. Cada fila representa un cliente con: Razón Social, identificador fiscal (RFC para México / RUC para Perú), número de Facturas Pendientes (conteo), Monto Total dolarizado, y la acción Ver Pedidos para navegar al detalle.

Regla 3 — Conteo de Facturas Pendientes incluye pedidos con factura generada pero no enviada
El conteo de Facturas Pendientes del cliente incluye los pedidos con factura por adelantado ya generada pero pendiente de envío al cliente. El conteo refleja todos los pedidos que aún requieren acción operativa del usuario (generar factura o enviar factura) y solo se retira del conteo cuando la factura se envía exitosamente al cliente.

Regla 4 — Monto Total dolarizado por cliente
Cuando los pedidos del cliente están en distintas monedas (MXN, USD, PEN, EUR u otras), el Monto Total del cliente se obtiene convirtiendo cada monto a USD y sumándolos. La visualización es siempre dolarizada para facilitar la comparación entre clientes. ~~El tipo de cambio aplicado a la conversión queda como duda formal del proyecto: opciones a evaluar son (a) tipo de cambio del día de generación de la proforma original de cada pedido (histórico estable), (b) tipo de cambio del día actual de la consulta del listado (dinámico). Pendiente definir con el cliente.~~ (Resuelto DUDA-046, ver Cambios). El monto de CADA PEDIDO se dolariza individualmente, usando el tipo de cambio de su propio pedido/documento origen, y LUEGO se suman los montos ya dolarizados para obtener el Monto Total mostrado en el listado. No se aplica un tipo de cambio único del día de consulta a todo el listado.

Regla 5 — Ordenamiento por antigüedad del pendiente
El listado de clientes se ordena por antigüedad del pendiente más antiguo del cliente. Los clientes con el pendiente más antiguo aparecen primero, priorizando la atención de los casos que más han esperado.

Regla 6 — Visibilidad filtrada por cartera del usuario
El listado muestra únicamente los clientes asignados a la cartera del usuario operativo (campo Cobrador del Catálogo de Clientes). Los clientes asignados a otros usuarios no aparecen en el listado.

Regla 7 — Buscador único por Razón Social, identificador fiscal o pedido interno
El buscador único de la pantalla busca por coincidencia parcial alfanumérica en: Razón Social, identificador fiscal (RFC o RUC) y número de Pedido Interno de los pedidos pendientes del cliente. El resultado de búsqueda son clientes (filas del listado agrupado), no pedidos individuales; si la búsqueda por pedido encuentra coincidencia, se retorna el cliente que contiene ese pedido. La búsqueda se ejecuta en tiempo real cuando el usuario deja de escribir (no requiere presionar Enter ni lupa). **El buscador ignora espacios al inicio y al final del texto ingresado (trim automático).**

Regla 8 — Paginación
Cuando el listado tiene un número de clientes superior al de elementos visibles por página, la pantalla ofrece paginación con numeración de páginas y navegación anterior/siguiente. El número de elementos por página se define por la implementación del módulo.

Regla 9 — Estado vacío
Cuando el usuario no tiene clientes con pedidos pendientes de Factura por Adelantado en su cartera, la pantalla muestra una vista de estado vacío con mensaje informativo claro y elementos visuales que comuniquen la ausencia de trabajo pendiente. La pantalla no queda en blanco ni muestra la tabla sin filas.

---

## Riesgos

Riesgo 1 — Brecha de timbrado para Perú
Esta pantalla en sí es agnóstica al país, pero el módulo Factura por Adelantado completo depende de capacidad de timbrado fiscal por región. Para México existe integración con TurboPac. Para Perú la integración con OSE/SUNAT es una brecha mayor del proyecto documentada en R16A-RE-FU-005 (Brecha 5). Mientras la brecha no esté habilitada, el sistema no debe generar pendientes de Factura por Adelantado para clientes Perú; hacerlo generaría pendientes huérfanos e inoperables que representan ruido operativo. Ver OBS-032/033.

> Nota: el riesgo anterior de "cliente sin Cobrador invisible" (Riesgo 1 previo) quedó mitigado por FU-002 Regla 6: el campo Cobrador no puede quedar vacío tras la primera asignación. Eliminado por OBS-036.

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — VISUALIZACIÓN DEL LISTADO
═══════════════════════════════════════════════════════════════

Criterio A1 — Visualización del listado agrupado por cliente
Dado que el usuario operativo accede al módulo Factura por Adelantado y tiene clientes con pedidos pendientes en su cartera,
Cuando se renderiza la pantalla inicial,
Entonces el sistema deberá presentar un listado con una fila por cada cliente que tenga al menos un pedido pendiente, mostrando: Razón Social del cliente, identificador fiscal con etiqueta de columna "RFC/RUC", número de Facturas Pendientes (conteo), Monto Total dolarizado, y acción "Ver Pedidos".

Criterio A2 — Etiqueta de columna del identificador fiscal
Dado que los clientes en el listado pueden ser de México (RFC) o Perú (RUC),
Cuando el sistema renderiza la cabecera de la tabla,
Entonces la etiqueta de la columna del identificador fiscal deberá ser "RFC/RUC" como etiqueta dual estática. El valor mostrado en cada fila es el identificador correspondiente a la región del cliente.

Criterio A3 — Aplicación uniforme a clientes México y Perú
Dado que los clientes del listado pueden ser de México o Perú,
Cuando el sistema renderiza la pantalla,
Entonces la funcionalidad opera de manera idéntica para ambos países. La diferenciación por región afecta solo el contenido (RFC vs RUC, monedas distintas) y no la mecánica de la pantalla.

═══════════════════════════════════════════════════════════════
SECCIÓN B — CÁLCULO DE CONTEO Y MONTO
═══════════════════════════════════════════════════════════════

Criterio B1 — Origen de los pedidos elegibles para el conteo
Dado que el sistema cuenta los pedidos pendientes de Factura por Adelantado por cliente,
Cuando construye el conteo,
Entonces deberá considerar pedidos con pendiente de Factura por Adelantado originados desde Tramitar Pedido en ambos flujos: Crédito con Factura por Adelantado y Prepago con Factura por Adelantado. El conteo incluye pedidos sin factura generada y pedidos con factura generada pero pendiente de envío al cliente. Solo cuando la factura se envía exitosamente, el pedido sale del conteo.

Criterio B2 — Cálculo del Monto Total dolarizado
Dado que un cliente tiene pedidos pendientes en distintas monedas (MXN, USD, PEN, EUR u otras),
Cuando el sistema calcula el Monto Total del cliente,
Entonces deberá convertir cada monto del pedido a USD individualmente, usando el tipo de cambio del propio pedido/documento origen, y luego sumar los montos ya dolarizados. El valor mostrado en la columna Monto Total es la sumatoria de esos montos dolarizados. ~~El tipo de cambio aplicado queda pendiente de definición formal con el cliente.~~ (Resuelto DUDA-046).

═══════════════════════════════════════════════════════════════
SECCIÓN C — ORDEN, FILTRADO Y BÚSQUEDA
═══════════════════════════════════════════════════════════════

Criterio C1 — Ordenamiento por antigüedad
Dado que el listado contiene múltiples clientes con pendientes,
Cuando el sistema los ordena para visualización,
Entonces deberá ordenarlos por antigüedad del pendiente más antiguo del cliente. Los clientes con el pendiente que más tiempo lleva en estado de espera aparecen primero.

Criterio C2 — Visibilidad filtrada por cartera del usuario
Dado que el usuario accede al módulo,
Cuando el sistema arma el listado,
Entonces deberá mostrar únicamente los clientes asignados a la cartera del usuario operativo (campo Cobrador del Catálogo de Clientes). Los clientes asignados a otros usuarios no aparecen.

Criterio C3 — Buscador por Razón Social, RFC/RUC o pedido interno
Dado que el usuario ingresa texto en el buscador único de la pantalla,
Cuando el sistema procesa la búsqueda,
Entonces deberá realizar coincidencia parcial alfanumérica en los campos: Razón Social del cliente, identificador fiscal del cliente, y números de Pedido Interno de los pedidos pendientes del cliente. El resultado son filas del listado agrupado por cliente. Si la coincidencia se da por número de pedido interno, se retorna el cliente que contiene ese pedido. El sistema ignorará los espacios al inicio y al final del texto ingresado antes de ejecutar la búsqueda (trim automático). Ver OBS-041.

Criterio C4 — Paginación con numeración y navegación anterior/siguiente
Dado que el listado tiene más clientes que los visibles en una sola página,
Cuando el sistema renderiza la pantalla,
Entonces deberá ofrecer paginación con numeración de páginas y botones de navegación anterior/siguiente.

═══════════════════════════════════════════════════════════════
SECCIÓN D — NAVEGACIÓN Y ESTADO VACÍO
═══════════════════════════════════════════════════════════════

Criterio D1 — Acción Ver Pedidos para navegar al detalle
Dado que el usuario identifica un cliente en el listado del que quiere consultar el detalle de pedidos pendientes,
Cuando ejecuta la acción "Ver Pedidos",
Entonces el sistema deberá navegar a la pantalla de Detalle por cliente del módulo Factura por Adelantado, llevando el contexto del cliente seleccionado para que el detalle muestre los pedidos pendientes del cliente.

Criterio D2 — Estado vacío
Dado que el usuario operativo accede al módulo Factura por Adelantado y no tiene clientes con pedidos pendientes en su cartera,
Cuando el sistema renderiza la pantalla,
Entonces deberá mostrar una vista de estado vacío con mensaje informativo y elementos visuales apropiados que comuniquen la ausencia de trabajo pendiente. No debe mostrar la tabla sin filas ni dejar la pantalla en blanco.

---

## Notas Adicionales

- Módulo NUEVO en PQF2 R16. El módulo Factura por Adelantado no existe en la versión actual del sistema y se incorpora en R16 para soportar el caso de negocio en el que un cliente solicita facturación anticipada antes del ingreso de la mercancía al almacén.
- Esta fila cubre exclusivamente la PANTALLA INICIAL del módulo: el listado agrupado por cliente con pedidos pendientes.
- Los pedidos elegibles para el conteo provienen de los flujos Crédito con Factura por Adelantado y Prepago con Factura por Adelantado, originados en Tramitar Pedido. Los pedidos con Sustancias Controladas no son elegibles para Factura por Adelantado por regla del cliente.
- El conteo de "Facturas Pendientes" del cliente incluye: (a) pedidos sin factura generada todavía y (b) pedidos con factura generada pero pendiente de envío al cliente. Solo sale del conteo cuando la factura se envía exitosamente.
- La conversión de los montos a USD permite comparar visualmente clientes con monedas distintas (MXN, USD, PEN, EUR u otras). ~~El tipo de cambio aplicado para la conversión queda como duda formal del proyecto: tipo de cambio histórico de la proforma original vs tipo de cambio del día actual de consulta. Pendiente definir con el cliente.~~ (Resuelto DUDA-046): cada pedido se dolariza individualmente con el tipo de cambio de su propio pedido/documento origen, y luego se suman los montos ya dolarizados. No se usa un tipo de cambio único del día de consulta.
- La etiqueta de la columna del identificador fiscal es "RFC/RUC" como etiqueta dual estática. El valor mostrado en cada fila corresponde a la región del cliente.
- El buscador único permite buscar por Razón Social, identificador fiscal o número de pedido interno. La búsqueda se ejecuta en tiempo real al dejar de escribir, con coincidencia parcial alfanumérica. Si la coincidencia se da por número de pedido interno, el resultado es el cliente que contiene ese pedido (el listado siempre se renderiza agrupado por cliente, no por pedido).
- La paginación aplica numeración de páginas más botones de navegación "Anterior" y "Siguiente".
- El ordenamiento por antigüedad del pendiente prioriza los casos más antiguos. No se implementan semáforos de prioridad ni alertas visuales adicionales (decisión de diseño: ligereza operativa sobre analítica visual).
- **(Resuelto DUDA-047, 2026-08-21):** la denominación canónica del rol operativo en la documentación funcional es "Gestor de Cobranza". Se elimina la mención a "Analista de Cuentas por Cobrar" como nombre del rol (ese nombre corresponde, según la resolución, a la Función/puesto de RRHH, no al rol funcional documentado).
- ~~Pendiente: tipo de cambio aplicado a la conversión de Monto Total a USD. Definir con el cliente.~~ (Resuelto DUDA-046: dolarización por pedido individual, no por TC único del día de consulta. Ver Regla 4 y Criterio B2).
- La habilitación efectiva del módulo para clientes Perú depende de la resolución de la brecha de timbrado SUNAT (integración con OSE/SUNAT), documentada en R16A-RE-FU-005 (Brecha 5). Mientras la brecha no esté habilitada, **no se deben generar pendientes de Factura por Adelantado para clientes Perú**; generarlos produciría pendientes huérfanos que no podrían cerrarse y representarían ruido operativo (OBS-032/033).
- Aplicabilidad uniforme a clientes México y Perú en esta pantalla una vez habilitada la facturación para Perú; las diferencias por región surgen al momento del timbrado de la factura.
- **Distinción FxA vs Factura Anticipo (OBS-037):** La Factura por Adelantado es un apoyo al cliente y nunca aplica a pedidos con Sustancias Controladas (por implicación regulatoria). La Factura Anticipo es exclusiva de pedidos de clientes prepago con Sustancias Controladas y se genera desde el módulo Validar Cobro. Ambos términos no son intercambiables.

---

## Cambios

| #   | Fecha      | Observación | Descripción del cambio                                                                                                                                                                                                                                                                            |
| --- | ---------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | 2026-06-10 | OBS-032/033 | Riesgo Perú corregido: premisa "aparecen aunque no puedan cerrarse" eliminada. Ahora: mientras la brecha de timbrado Perú (OSE/SUNAT) no esté habilitada, no se generan pendientes de Perú para evitar huérfanos y ruido. Actualizado en Riesgo 1 (antes Riesgo 2) y Nota al final del documento. |
| 2   | 2026-06-10 | OBS-036     | Riesgo 1 anterior ("cliente sin Cobrador invisible") eliminado: mitigado por FU-002 Regla 6 (Cobrador no puede quedar vacío tras primera asignación). Riesgo 2 renumerado a Riesgo 1.                                                                                                             |
| 3   | 2026-06-10 | OBS-037     | Nota agregada: distinción explícita entre Factura por Adelantado (apoyo al cliente, nunca con controlados) y Factura Anticipo (exclusiva prepago con controlados, desde Validar Cobro).                                                                                                           |
| 4   | 2026-06-10 | OBS-041     | Regla 7 y Criterio C3: buscador aplica trim automático (ignora espacios al inicio y al final del texto ingresado).                                                                                                                                                                                |
| 5   | 2026-08-21 | DUDA-047    | Homologada la denominación del rol operativo a "Gestor de Cobranza"; se eliminan las menciones a "Analista de Cuentas por Cobrar" como nombre del rol.                                                                                                                                            |
| 5   | 2026-08-21 | DUDA-046    | Cerrado el pendiente de tipo de cambio para dolarización (Regla 4, Criterio B2, Notas Adicionales, equivalente a OBS-052): cada pedido se dolariza individualmente con el tipo de cambio de su propio pedido/documento origen, y luego se suman los montos ya dolarizados. No se usa un TC único del día de consulta para todo el listado.                                                                                                                                                                                |

