# R16A-RE-FU-027 — Validar Cobro: Paso 2 Perú

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-027 |
| **Título** | Validar Cobro: Paso 2 Perú |
| **Módulo / Épica** | Validar Cobro |
| **Historia de Usuario** | Yo como **Gestor de Cobranza**, quiero contar con la segunda pantalla del wizard de Validar Cobro (Paso 2 - Asociación) para clientes con Región Perú, donde asocio los cobros capturados con las proformas y facturas pendientes del cliente y aplico opcionalmente sus Notas de Crédito, para conciliar lo recibido contra lo adeudado y dejar la asociación lista para el Paso 3. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.2M-RE-FU-002 |

---

> **Trazabilidad de dudas cerradas (2026-08-21):** DUDA-086 (Resuelta) — tolerancia de pago de menos para Perú: misma regla que México, configurable a nivel BD. DUDA-087 (Descartada) — se cancela la facturación de Perú en R16; no aplica desarrollar la mecánica de NC peruana (catálogo 09 SUNAT) ni su aplicación al adeudo en este Paso.

## Requisito Funcional

El sistema debe contar con la segunda pantalla del wizard de Validar Cobro (Paso 2 - Asociación) para clientes con Región Perú, donde el usuario asocia manualmente los cobros capturados con las proformas y facturas pendientes de cobrar del cliente, en relación de muchos a muchos, y puede aplicar opcionalmente las Notas de Crédito vigentes del cliente. El sistema calcula y muestra el saldo de la asociación y gobierna su cierre según los escenarios de pago exacto, sobrepago o pago de menos. La estructura UI de esta pantalla es la misma que la de México (R16A-RE-FU-026); las diferencias entre regiones son los catálogos y reglas fiscales específicas. A diferencia de México, en Perú la asociación NO tiene efecto fiscal: no genera Complemento de Pago ni se reporta a SUNAT (la factura peruana ya se emitió completa con su IGV); es un registro operativo interno de conciliación. Una vez cerrada la asociación, el flujo avanza al Paso 3.

---

## Alcance

### Aplica a

- Pantalla del Paso 2 del wizard de Validar Cobro: Asociación de cobros con Proformas/Facturas, para clientes con Región Perú.
- Estructura UI idéntica a la de México (R16A-RE-FU-026); las diferencias son exclusivamente catálogos y reglas fiscales.
- Cabecera del cliente (consistente con Paso 1: logo, Alias, etiquetas preexistentes de clasificación, RUC, razón social legal, moneda de facturación).
- Barra de pasos del wizard visible: 1-CAPTURAR COBRO (✓), 2-ASOCIAR FACTURA/PROFORMA (activo), 3-FACTURACIÓN Y ENVÍO.
- Listado de cobros del cliente capturados en el Paso 1 con selección múltiple (checkboxes) para usarse simultáneamente en la asociación.
- Multi-divisa: cuando la moneda del cobro difiere de la moneda de los documentos asociados, el sistema convierte los importes de los documentos a la moneda del cobro usando el TC capturado en el Paso 1 del cobro, expone el TC y la fórmula vía tooltip, y consolida los totales en moneda del cobro. La moneda base es PEN.
- Identificador visible "saldo a favor" en los cobros del listado del Paso 1 que tienen excedente disponible por aplicar.
- Header con el cobro o cobros seleccionados (folios COB-..., montos, fecha del cobro).
- Listado de Proformas y Facturas pendientes de cobrar del cliente (mezcladas, sin filtros adicionales por tipo, fecha u otros criterios). Empresa emisora única: Golocaer S.A.C. (no hay mezcla de empresas emisoras como en México).
- Asociación manual N a N: el usuario decide qué cobros aplica a qué documentos, sin distribución automática ni campo "Monto a aplicar" editable por línea.
- Aplicación OPCIONAL de Notas de Crédito vigentes del cliente al adeudo (cero, una o varias por documento). ~~La mecánica fiscal de la NC peruana (referencia SUNAT, catálogo 09) se desarrolla en las filas de Notas de Crédito Perú (R16A-RE-FU-033/035); en este Paso se contempla la aplicación operativa de NC al adeudo, pendiente de validar su mecánica de referencia para Perú.~~ **[DUDA-087 — Descartada: se cancela la facturación de Perú en R16; no aplica desarrollar la mecánica de NC peruana (catálogo 09 SUNAT) ni su aplicación al adeudo. La aplicación operativa de NC en este Paso queda fuera de alcance mientras no haya facturación Perú.]**
- Cálculo dinámico del saldo de la asociación (suma de cobros aplicados + suma de NCs aplicadas - suma de adeudos de los documentos seleccionados).
- Resolución de escenarios de pago: exacto, sobrepago (saldo a favor), pago de menos con tolerancia, pago de menos fuera de tolerancia (inconsistencia).
- Generación de saldo a favor cuando los cobros + NCs superan el adeudo: se refleja en el Estado de Cuenta/Auxiliar del cliente, queda disponible para futuras proformas/Factura por Adelantado desde Validar Cobro, sin generar documento fiscal adicional, y el cobro origen en el Paso 1 se marca con identificador visible "saldo a favor".
- Tolerancia de pago de menos: permite cerrar el adeudo cuando la diferencia faltante es menor o igual a un umbral; la diferencia se refleja como saldo pendiente en el Estado de Cuenta/Auxiliar del cliente sin bloquear el avance. **[DUDA-086 — Resuelta: se aplica la MISMA regla que México (tolerancia equivalente para Perú); los montos límite deben ser configurables a nivel BD.]**
- Marcado de inconsistencias del Paso 2 mediante modal: tipo de inconsistencia (combo) y comentario opcional. Tipos del Paso 2 incluyen los del Paso 1 más tipos contextuales (Pago Incompleto Vencido, Pago Insuficiente).
- Tipo "Pago Incompleto Vencido": habilita el marcado del pedido como "Pendiente de cancelación por falta de pago" (cancelación reactiva). El marcado NO ejecuta cancelación fiscal efectiva ni dispara devolución; transferencia para gestión externa.
- Auto-guardado del Paso 2 consistente con el Paso 1.
- Asociaciones del Paso 2 editables (modificables, deshacer y rehacer) mientras el usuario esté en el Paso 2. Una vez avanza al Paso 3, las asociaciones quedan fijas.
- Navegación: Regresar (vuelve al Paso 1) o Continuar (avanza al Paso 3 con la asociación cerrada).

### No aplica a

- Wizard de Validar Cobro para Región México: se documenta en R16A-RE-FU-026.
- Paso 1 (Captura del Cobro) y Paso 3 (Facturación y Envío) del wizard Perú: se documentan en R16A-RE-FU-025 y R16A-RE-FU-029.
- Generación de Complemento de Pago a partir de la asociación: NO aplica a Perú (SUNAT no tiene Complemento de Pago). La asociación cobro↔documento es operativa, no fiscal; no genera documento fiscal ni se reporta a SUNAT.
- Mezcla de empresas emisoras: en Perú el emisor es siempre Golocaer S.A.C.
- Generación de Notas de Crédito (emisión de nuevas NCs): se documenta en el módulo Notas de Crédito Perú (R16A-RE-FU-033/035).
- Mecánica fiscal de referencia de la NC peruana (catálogo 09 SUNAT): se desarrolla en R16A-RE-FU-033/035.
- Auxiliar de saldos remanentes de Notas de Crédito aplicadas parcialmente: queda fuera de scope R16 (consistente con México).
- Cancelación fiscal efectiva de proformas/facturas: NO contemplada en R16 fuera del módulo Notas de Crédito Perú. La cancelación del Paso 2 únicamente marca el pedido para gestión externa.
- Sistema de devoluciones de dinero al cliente: NO contemplado en R16.
- Catálogo de Tipos de Inconsistencia (definición de las opciones del combo): ** pendiente del lado de PROQUIFA (Tesorería). **

---

## Reglas de Negocio

**Regla 1 — Aplicabilidad solo a Región Perú**
El Paso 2 del wizard de Validar Cobro de este requisito opera exclusivamente sobre clientes con Región Perú. Los clientes con Región México son atendidos por el wizard equivalente de México (R16A-RE-FU-026). La UI es la misma; cambian los catálogos y las reglas fiscales.

**Regla 2 — La asociación cobro↔documento no tiene efecto fiscal en Perú**
La asociación que se realiza en este Paso es un registro operativo interno de conciliación: vincula qué cobro salda qué documento y actualiza saldos. A diferencia de México, NO genera Complemento de Pago ni se reporta a SUNAT, porque la factura peruana ya se emitió completa con su IGV. La utilidad de la asociación es de control de cobranza, no fiscal.

**Regla 3 — Cobros disponibles para asociar**
El listado de cobros del cliente muestra todos los cobros capturados en el Paso 1 disponibles para asociar (cobros confirmados que aún no han sido aplicados totalmente a documentos). Cada cobro muestra su folio COB-mmddaa-consecutivo, monto y fecha. Los cobros con excedente disponible (saldo a favor de cobros previos) se marcan visualmente con un identificador "saldo a favor".

**Regla 4 — Selección múltiple de cobros para asociación**
El usuario puede seleccionar uno o más cobros mediante checkboxes y aplicarlos a uno o más documentos en relación N a N. Combinaciones válidas: 1-1, 1-N, N-1, N-N.

**Regla 5 — Listado de Proformas y Facturas pendientes**
El listado central muestra todas las proformas y facturas pendientes de cobrar del cliente, mezcladas en el mismo listado, sin filtros adicionales por tipo, fecha u otros criterios. Cada documento muestra: tipo (Proforma o Factura), folio, Pedido Interno, importe total y saldo pendiente actual. La empresa emisora es siempre Golocaer S.A.C. (no hay mezcla de empresas como en México).

**Regla 6 — Asociación manual N a N con aplicación en orden de selección**
El sistema no ofrece un campo "Monto a aplicar" editable por línea. El usuario controla la asociación cobro↔documento mediante la selección de cuáles cobros aplica, cuáles documentos cubre y el orden de selección de los documentos, que determina la prioridad de aplicación del monto disponible del cobro. El sistema aplica el monto del cobro a los documentos en el orden seleccionado, hasta agotar el monto disponible. Si el cobro no alcanza para cubrir todos los documentos seleccionados, el último de la secuencia queda con saldo pendiente y la asociación queda pendiente del próximo cobro complementario.

**Regla 7 — Aplicación OPCIONAL de Notas de Crédito vigentes del cliente**
Cuando el cliente tiene una o más Notas de Crédito vigentes, el sistema muestra el catálogo de NCs vigentes y permite al usuario seleccionar cuáles aplica al documento (cero, una o varias). La aplicación es OPCIONAL; las NCs no seleccionadas siguen vigentes para sesiones futuras. Las NCs son por documento (cada NC se aplica completa a un solo documento). ~~La mecánica fiscal de referencia de la NC peruana (catálogo 09 SUNAT, forma de relacionarla al comprobante) se desarrolla en R16A-RE-FU-033/035 y queda pendiente de validar para Perú. En este Paso se contempla la aplicación operativa de la NC al adeudo.~~ **[DUDA-087 — Descartada: se cancela la facturación de Perú en R16, por lo que no aplica desarrollar la mecánica de NC peruana ni su aplicación al adeudo en este Paso.]**

**Regla 8 — Cálculo dinámico del saldo de la asociación**
El sistema calcula dinámicamente: suma del adeudo de los documentos seleccionados, suma de los cobros aplicados, suma de las NCs aplicadas, y saldo de la asociación = (cobros aplicados + NCs aplicadas) - adeudo total. El saldo se muestra con indicación clara del escenario resultante: cero (exacto), positivo (sobrepago → saldo a favor), negativo dentro de tolerancia (cierra con saldo pendiente en Estado de Cuenta), negativo fuera de tolerancia (requiere inconsistencia o asociación pendiente).

**Regla 9 — Escenario pago exacto**
Cuando la suma de cobros aplicados + NCs aplicadas iguala exactamente el adeudo total de los documentos seleccionados, el sistema permite avanzar al Paso 3 con la asociación cerrada.

**Regla 10 — Escenario sobrepago (saldo a favor)**
Cuando la suma de cobros aplicados + NCs aplicadas supera el adeudo total, el sistema permite avanzar al Paso 3 con la asociación que cubre el adeudo total; registra el excedente como saldo a favor del cliente; lo refleja en el Estado de Cuenta/Auxiliar del cliente; marca el cobro o cobros origen del excedente con identificador visible "saldo a favor" en el listado del Paso 1; deja el saldo a favor disponible para aplicarse a una proforma o factura nueva del mismo cliente desde Validar Cobro en sesiones futuras; y no genera documento fiscal adicional (no Nota de Crédito, no devolución).

**Regla 11 — Escenario pago de menos con tolerancia**
Cuando la suma de cobros aplicados + NCs aplicadas es menor al adeudo total, el sistema bifurca su comportamiento según el monto de la diferencia faltante: si es menor o igual al umbral de tolerancia, el sistema permite cerrar el documento con el monto recibido a través del registro manual del operador, refleja la diferencia como saldo pendiente en el Estado de Cuenta/Auxiliar del cliente (sin bloqueo) y permite avanzar al Paso 3; si supera el umbral, el sistema no permite avanzar y el usuario debe marcar inconsistencia o dejar la asociación pendiente del próximo cobro complementario. **[DUDA-086 — Resuelta: se aplica la MISMA regla de tolerancia que México (tolerancia equivalente para Perú); los montos límite deben ser configurables a nivel BD.]**

**Regla 12 — Marcado de inconsistencias del Paso 2**
Al presionar "Marcar Inconsistencia en Cobro", el sistema abre el modal "Inconsistencia de Pago" con los campos: Tipo de Inconsistencia (combo del catálogo de tipos aplicables al Paso 2) y Comentario adicional (opcional). El catálogo del Paso 2 incluye los tipos del Paso 1 (intrínsecos del cobro) más tipos contextuales que requieren conocer el documento a cobrar (Pago Incompleto Vencido, Pago Insuficiente). ** El catálogo completo está pendiente de definición por PROQUIFA (Tesorería). **

**Regla 13 — Tipo "Pago Incompleto Vencido" con marcado del pedido para cancelación externa**
Cuando el usuario selecciona el tipo "Pago Incompleto Vencido" (cliente realizó un pago parcial pero nunca completó el monto total y la operación se considera vencida) y confirma la inconsistencia, el sistema habilita adicionalmente la opción de marcar el pedido asociado al cobro como "Pendiente de cancelación por falta de pago". Este marcado no ejecuta la cancelación fiscal efectiva de las proformas o facturas asociadas ni dispara devolución del pago parcial recibido. El marcado únicamente notifica para gestión de cancelación efectiva externa y, si aplica, devolución del pago parcial. ** Mecanismo de transferencia del estado pendiente de definir. La cancelación fiscal en Perú se realizaría vía Nota de Crédito SUNAT — ver R16A-RE-FU-033/035. **

**Regla 14 — Tipo "Pago Insuficiente" (diferencia fuera de tolerancia)**
Cuando el usuario detecta que el cobro es insuficiente y la diferencia faltante supera la tolerancia, al confirmar la inconsistencia "Pago Insuficiente" el sistema registra la inconsistencia, mantiene la asociación pendiente y notifica al cliente que el cobro debe ser complementado. La operación queda en espera de que el cliente complete la diferencia o de instrucciones de Tesorería; no se marca el pedido para cancelación automáticamente.

**Regla 15 — Auto-guardado del Paso 2**
Cuando el usuario modifica asociaciones, aplica NCs o navega entre cobros y documentos, el sistema auto-guarda el estado del Paso 2 (asociaciones, NCs aplicadas, inconsistencias marcadas). No existe botón "Guardar" manual.

**Regla 16 — Asociaciones editables en el Paso 2**
Mientras el usuario esté en el Paso 2, el sistema permite deshacer asociaciones previas, reasociar cobros con otros documentos, y aplicar o remover NCs. Una vez avanza al Paso 3 con "Continuar", las asociaciones quedan fijas.

**Regla 17 — Navegación: Regresar y Continuar**
El Paso 2 ofrece dos acciones: Regresar (vuelve al Paso 1 sin perder lo capturado, auto-guardado activo) y Continuar (avanza al Paso 3 con la asociación cerrada). Solo se permite el avance si el escenario resultante es pago exacto, sobrepago (saldo a favor registrado) o pago de menos dentro de tolerancia.

**Regla 18 — Bloqueo de avance al Paso 3**
Cuando el escenario resultante es pago de menos con diferencia fuera de tolerancia sin inconsistencia marcada, al presionar "Continuar" el sistema bloquea el avance; el usuario debe marcar inconsistencia o dejar la asociación pendiente del próximo cobro complementario.

**Regla 19 — Totales en moneda del cobro con conversión por TC capturado**
Cuando los documentos seleccionados están en moneda distinta a la moneda del cobro, el sistema convierte el importe de cada documento a la moneda del cobro usando el TC capturado en el Paso 1 del cobro seleccionado; muestra todos los totales del panel (Adeudo Proformas y Facturas, NCs aplicadas, Cobros aplicados, Adeudo restante, Saldo a favor generado) en la moneda del cobro; muestra el TC aplicado de forma visible junto a cada documento convertido (tooltip con la fórmula: importe en moneda original × TC = importe en moneda del cobro); y valida que el monto del cobro sea suficiente para cubrir la suma convertida. Si la moneda del documento coincide con la del cobro, no aplica conversión. La moneda base es PEN. ** Fuente oficial del TC para Perú pendiente de definir (no aplica el DOF mexicano). **

**Regla 20 — NCs aplicadas en moneda del cobro**
Cuando una Nota de Crédito aplicada está en moneda distinta a la del cobro, el sistema convierte su monto aplicado a moneda del cobro usando el TC capturado en el Paso 1. El monto convertido es el que se descuenta del Adeudo restante.

---

## Riesgos

**Riesgo 1 — Tolerancia de pago de menos para Perú no definida** ~~(vigente)~~ **[DUDA-086 — Resuelta: se aplica la MISMA regla que México (tolerancia equivalente), montos configurables a nivel BD. Riesgo cerrado.]**
~~El umbral de tolerancia de pago de menos para Perú (equivalente a los 100 MXN de México) no está definido: monto, moneda y tratamiento cuando la facturación no es PEN. Sin él, el escenario de pago de menos no puede gobernarse automáticamente.~~

**Riesgo 2 — Mecánica fiscal de la NC peruana en la asociación pendiente** **[DUDA-087 — Descartada: se cancela la facturación de Perú en R16; no aplica desarrollar esta mecánica. Riesgo cerrado por no aplicabilidad.]**
~~La aplicación de Notas de Crédito al adeudo en el Paso 2 depende de la mecánica de referencia de la NC peruana (catálogo 09 SUNAT), que se define en R16A-RE-FU-033/035 y aún no está validada para Perú.~~

**Riesgo 3 — Fuente del tipo de cambio para Perú no definida**
La conversión multi-divisa del Paso 2 usa el TC capturado en el Paso 1, cuya fuente oficial para Perú no está definida (no aplica el DOF mexicano).

**Riesgo 4 — Cancelación del pedido sin cancelación fiscal ni devolución**
El tipo "Pago Incompleto Vencido" marca el pedido como "Pendiente de cancelación por falta de pago", pero R16 no ejecuta la cancelación fiscal efectiva ni dispara devolución del pago parcial. En Perú la cancelación fiscal se realizaría vía Nota de Crédito SUNAT (R16A-RE-FU-033/035). El mecanismo de transferencia del estado para gestión externa está pendiente de definir.

**Riesgo 5 — Catálogo de Tipos de Inconsistencia del Paso 2 pendiente**
El catálogo de tipos de inconsistencia del Paso 2 está pendiente de definición por PROQUIFA (Tesorería), transversal con México.

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA, BARRA DE PASOS Y COBROS DISPONIBLES
═══════════════════════════════════════════════════════════════

**Criterio A1 — Cabecera del cliente y barra de pasos**
Dado que el usuario entra al Paso 2 para un cliente con Región Perú,
Cuando el sistema renderiza la pantalla,
Entonces deberá mostrar la cabecera del cliente (logo, razón social, etiquetas, RUC, razón social legal, moneda de facturación) y la barra de pasos con "2 - ASOCIAR FACTURA/PROFORMA" activo y "1 - CAPTURAR COBRO" marcado como completado.

**Criterio A2 — Listado de cobros disponibles con selección múltiple**
Dado que el cliente tiene cobros capturados en el Paso 1,
Cuando el sistema renderiza el listado de cobros,
Entonces deberá mostrar cada cobro (folio COB-mmddaa-NNNN, monto, fecha) con checkbox de selección múltiple, marcando con identificador "saldo a favor" los que tengan excedente disponible.

═══════════════════════════════════════════════════════════════
SECCIÓN B — LISTADO DE DOCUMENTOS Y ASOCIACIÓN N A N
═══════════════════════════════════════════════════════════════

**Criterio B1 — Listado de Proformas y Facturas pendientes**
Dado que el cliente tiene proformas y facturas pendientes de cobrar,
Cuando el sistema renderiza el listado central,
Entonces deberá mostrar todos los documentos mezclados (tipo, folio, Pedido Interno, importe total, saldo pendiente), con emisor Golocaer S.A.C., sin filtros adicionales.

**Criterio B2 — Asociación manual N a N en orden de selección**
Dado que el usuario selecciona uno o más cobros y uno o más documentos,
Cuando el sistema procesa la asociación,
Entonces deberá aplicar el monto disponible del cobro a los documentos en el orden en que el usuario los seleccionó, hasta agotar el monto; si no alcanza, el último documento queda con saldo pendiente.

**Criterio B3 — Aplicación opcional de Notas de Crédito**
Dado que el cliente tiene Notas de Crédito vigentes,
Cuando el usuario decide aplicar NCs a un documento,
Entonces el sistema deberá permitir seleccionar cero, una o varias NCs por documento; las no seleccionadas siguen vigentes. ~~Mecánica fiscal de referencia de la NC peruana pendiente — ver R16A-RE-FU-033/035.~~ **[DUDA-087 — Descartada: se cancela la facturación de Perú en R16; no aplica desarrollar esta mecánica.]**

═══════════════════════════════════════════════════════════════
SECCIÓN C — CÁLCULO DEL SALDO Y ESCENARIOS DE PAGO
═══════════════════════════════════════════════════════════════

**Criterio C1 — Cálculo dinámico del saldo**
Dado que el usuario asocia cobros y NCs a documentos,
Cuando cambia cualquier asociación,
Entonces el sistema deberá recalcular y mostrar: adeudo total, cobros aplicados, NCs aplicadas y saldo de la asociación, con indicación del escenario resultante.

**Criterio C2 — Pago exacto**
Dado que cobros aplicados + NCs aplicadas igualan el adeudo total,
Cuando el usuario presiona Continuar,
Entonces el sistema deberá permitir avanzar al Paso 3 con la asociación cerrada.

**Criterio C3 — Sobrepago (saldo a favor)**
Dado que cobros aplicados + NCs aplicadas superan el adeudo total,
Cuando el usuario cierra la asociación,
Entonces el sistema deberá registrar el excedente como saldo a favor en el Estado de Cuenta/Auxiliar del cliente, marcar el cobro origen con "saldo a favor", dejarlo disponible para sesiones futuras y NO generar documento fiscal adicional.

**Criterio C4 — Pago de menos dentro de tolerancia**
Dado que la diferencia faltante es menor o igual al umbral de tolerancia,
Cuando el usuario cierra la asociación,
Entonces el sistema deberá permitir cerrar el documento con el monto recibido, reflejar la diferencia como saldo pendiente en el Estado de Cuenta/Auxiliar y permitir avanzar al Paso 3. **[DUDA-086 — Resuelta: umbral = misma regla que México, configurable a nivel BD.]**

**Criterio C5 — Pago de menos fuera de tolerancia**
Dado que la diferencia faltante supera el umbral de tolerancia y no hay inconsistencia marcada,
Cuando el usuario presiona Continuar,
Entonces el sistema deberá bloquear el avance; el usuario debe marcar inconsistencia o dejar la asociación pendiente del próximo cobro.

═══════════════════════════════════════════════════════════════
SECCIÓN D — MULTI-DIVISA
═══════════════════════════════════════════════════════════════

**Criterio D1 — Conversión de documentos a moneda del cobro**
Dado que los documentos seleccionados están en moneda distinta a la del cobro,
Cuando el sistema arma el panel de totales,
Entonces deberá convertir cada documento a la moneda del cobro usando el TC capturado en el Paso 1, mostrar el TC y la fórmula vía tooltip, y consolidar los totales en moneda del cobro. La moneda base es PEN. ** Fuente del TC para Perú pendiente de definir. **

**Criterio D2 — NCs en moneda del cobro**
Dado que una NC aplicada está en moneda distinta a la del cobro,
Cuando el sistema calcula el saldo,
Entonces deberá convertir el monto de la NC a moneda del cobro usando el TC capturado en el Paso 1 y descontarlo del adeudo restante.

═══════════════════════════════════════════════════════════════
SECCIÓN E — INCONSISTENCIAS
═══════════════════════════════════════════════════════════════

**Criterio E1 — Marcado de inconsistencia del Paso 2**
Dado que el usuario presiona "Marcar Inconsistencia en Cobro",
Cuando el sistema abre el modal "Inconsistencia de Pago",
Entonces deberá ofrecer Tipo de Inconsistencia (combo: tipos del Paso 1 + Pago Incompleto Vencido + Pago Insuficiente) y Comentario adicional opcional. ** Catálogo completo pendiente (Tesorería). **

**Criterio E2 — Pago Incompleto Vencido**
Dado que el usuario selecciona "Pago Incompleto Vencido" y confirma,
Cuando el sistema procesa la inconsistencia,
Entonces deberá habilitar la opción de marcar el pedido como "Pendiente de cancelación por falta de pago", sin ejecutar cancelación fiscal efectiva ni devolución; solo notifica para gestión externa. ~~La cancelación fiscal en Perú se realizaría vía Nota de Crédito SUNAT (R16A-RE-FU-033/035); mecanismo de transferencia pendiente.~~ **[DUDA-087 — Descartada: se cancela la facturación de Perú en R16; no aplica NC peruana como vía de cancelación fiscal. Mecanismo de transferencia del estado sigue pendiente por separado.]**

**Criterio E3 — Pago Insuficiente**
Dado que el usuario selecciona "Pago Insuficiente" (diferencia fuera de tolerancia) y confirma,
Cuando el sistema procesa la inconsistencia,
Entonces deberá registrarla, mantener la asociación pendiente y notificar al cliente que el cobro debe complementarse, sin marcar el pedido para cancelación automáticamente.

═══════════════════════════════════════════════════════════════
SECCIÓN F — AUTO-GUARDADO Y NAVEGACIÓN
═══════════════════════════════════════════════════════════════

**Criterio F1 — Auto-guardado del Paso 2**
Dado que el usuario modifica asociaciones, aplica NCs o navega,
Cuando ocurre el cambio,
Entonces el sistema deberá auto-guardar el estado del Paso 2 sin requerir acción manual.

**Criterio F2 — Asociaciones editables hasta avanzar**
Dado que el usuario está en el Paso 2,
Cuando modifica asociaciones (deshacer, reasociar, aplicar/remover NCs),
Entonces el sistema deberá permitirlo libremente; al avanzar al Paso 3 con Continuar, las asociaciones quedan fijas.

**Criterio F3 — Navegación Regresar y Continuar**
Dado que el usuario está en el Paso 2,
Cuando finaliza,
Entonces el sistema deberá ofrecer Regresar (vuelve al Paso 1, auto-guardado activo) y Continuar (avanza al Paso 3 solo si el escenario es exacto, sobrepago o pago de menos dentro de tolerancia).

---

## Notas Adicionales

- Fila para el Paso 2 (Asociación) del wizard de Validar Cobro de la Región Perú, contraparte de R16A-RE-FU-026 (México). Estado depende de la resolución de las brechas Perú.
- La estructura UI es idéntica a la de México; las diferencias son catálogos y reglas fiscales.
- Diferencia de fondo vs México — la asociación cobro↔documento NO tiene efecto fiscal en Perú: no genera Complemento de Pago ni se reporta a SUNAT (la factura peruana ya se emitió completa con su IGV). Es un registro operativo interno de conciliación de cobranza.
- Toda la lógica operativa (asociación N a N en orden de selección, escenarios de pago exacto/sobrepago/pago de menos, saldo a favor, multi-divisa con TC, inconsistencias) se mantiene igual que en México por ser operativa, no fiscal.
- Empresa emisora única: Golocaer S.A.C. (no hay mezcla de empresas emisoras como en México).
- **[DUDA-086 — Resuelta, 2026-08-21]** ~~Pendiente — tolerancia de pago de menos para Perú (monto, moneda, tratamiento cuando facturación no es PEN). En México es 100 MXN, Política Interna PROQUIFA.~~ Se aplica la MISMA regla que México (tolerancia equivalente para Perú); montos límite configurables a nivel BD.
- **[DUDA-087 — Descartada, 2026-08-21]** ~~Pendiente — mecánica fiscal de referencia de la NC peruana (catálogo 09 SUNAT) al aplicarla al adeudo; se desarrolla en R16A-RE-FU-033/035.~~ Se cancela la facturación de Perú en R16; no aplica desarrollar la mecánica de NC peruana ni su aplicación al adeudo en este Paso.
- ** Pendiente — fuente oficial del TC para Perú (no aplica el DOF mexicano). **
- ** Pendiente — catálogo de tipos de inconsistencia del Paso 2 (Tesorería), transversal con México. **
- ** Pendiente — mecanismo de transferencia del estado "Pendiente de cancelación por falta de pago" para gestión externa; en Perú la cancelación fiscal sería vía Nota de Crédito SUNAT (R16A-RE-FU-033/035). **
- Auxiliar de saldos remanentes de NC parcialmente aplicadas: fuera de scope R16 (consistente con México).
- ** Pendiente — maquetas de Validar Cobro Perú no disponibles; el detalle de la pantalla se validará contra ellas cuando lleguen. **
- **(Resuelto DUDA-047, 2026-08-21):** la denominación canónica del rol operativo en la documentación funcional es "Gestor de Cobranza"; se eliminan las menciones a "Analista de Cuentas por Cobrar" como nombre del rol.
