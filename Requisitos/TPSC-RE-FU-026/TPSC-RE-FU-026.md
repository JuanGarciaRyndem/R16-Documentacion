# TPSC-RE-FU-026 — Validar Cobro: Paso 2 México

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-026 |
| **Título** | Validar Cobro: Paso 2 México |
| **Módulo / Épica** | Validar Cobro |
| **Historia de Usuario** | Yo como **Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver)**, quiero contar con la segunda pantalla del wizard de Validar Cobro (Paso 2 - Asociación) para asociar los cobros capturados con las proformas y facturas pendientes del cliente y aplicar opcionalmente sus Notas de Crédito, para conciliar lo recibido contra lo adeudado y dejar la asociación lista para emitir los documentos fiscales. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.2M-RE-FU-002 |

---

## Requisito Funcional

El sistema debe contar con la segunda pantalla del wizard de Validar Cobro (Paso 2 - Asociación) para clientes con Región México, donde el usuario asocia manualmente los cobros capturados con las proformas y facturas pendientes de cobrar del cliente, en relación de muchos a muchos, y puede aplicar opcionalmente las Notas de Crédito vigentes del cliente. El sistema calcula y muestra el saldo de la asociación y gobierna su cierre según los escenarios de pago exacto, sobrepago o pago de menos. Una vez cerrada la asociación, el flujo avanza al Paso 3 para emitir los documentos fiscales correspondientes. La estructura UI de esta pantalla se reutiliza idénticamente para clientes Región Perú; las diferencias entre regiones son los catálogos y reglas específicas y se documentan en requisito independiente.

---

## Alcance

### Aplica a

- Pantalla del Paso 2 del wizard de Validar Cobro: Asociación de cobros con Proformas/Facturas.
- Aplica a clientes con Región México exclusivamente (los catálogos y la operación Perú con la misma UI se documentan en requisito independiente).
- Cabecera del cliente (estructura consistente con Paso 1: logo, Alias, etiquetas preexistentes de clasificación, RFC/RUC, razón social legal, moneda de facturación).
- Barra de pasos del wizard visible: 1-CAPTURAR COBRO (✓), 2-ASOCIAR FACTURA/PROFORMA (activo), 3-FACTURACIÓN Y ENVÍO.
- Listado de cobros del cliente capturados en el Paso 1 con selección múltiple (checkboxes) para usarse simultáneamente en la asociación.
- Multi-divisa: cuando la moneda del cobro difiere de la moneda de los documentos asociados, el sistema convierte los importes de los documentos a la moneda del cobro usando el TC capturado en el Paso 1 del cobro, expone el TC y la fórmula vía tooltip, y consolida los totales en moneda del cobro.
- Identificador visible "saldo a favor" en los cobros del listado del Paso 1 que tienen excedente disponible por aplicar.
- Header con el cobro o cobros seleccionados (folios COB-..., montos, fecha del cobro).
- Listado de Proformas y Facturas pendientes de cobrar del cliente (mezcladas, sin filtros adicionales por tipo, empresa emisora, fecha u otros criterios). Puede incluir documentos de diferentes empresas emisoras del grupo PROQUIFA México en una misma sesión.
- Asociación manual N a N: el usuario decide qué cobros aplica a qué documentos. No hay distribución automática del sistema, ni campo "Monto a aplicar" editable por línea: el usuario marca asociaciones completas cobro↔documento.
- Aplicación OPCIONAL de Notas de Crédito vigentes del cliente al adeudo: el usuario puede aplicar cero, una o varias NCs por documento (NC por documento) según decida. NO se obliga a aplicar todas las NCs vigentes del cliente. Las NCs no aplicadas siguen vigentes para sesiones futuras. Las NCs referencian su UUID timbrado previamente del catálogo de NCs vigentes del cliente.
- Cálculo dinámico del saldo de la asociación (suma de cobros aplicados + suma de NCs aplicadas - suma de adeudos de los documentos seleccionados).
- Resolución de escenarios de pago: exacto, sobrepago (saldo a favor), pago de menos con tolerancia 100 MXN, pago de menos > 100 MXN (inconsistencia).
- Generación de saldo a favor cuando los cobros + NCs superan el adeudo: se refleja en el Estado de Cuenta/Auxiliar del cliente, queda disponible para futuras proformas/Factura por Adelantado desde Validar Cobro, sin generar documento fiscal adicional, y el cobro origen en el Paso 1 se marca con identificador visible "saldo a favor".
- Tolerancia 100 MXN: permite cerrar el adeudo cuando la diferencia faltante es ≤ 100 MXN (inclusiva del límite); la diferencia se refleja como saldo pendiente en el Estado de Cuenta/Auxiliar del cliente sin bloquear el avance.
- Marcado de inconsistencias del Paso 2 mediante modal: tipo de inconsistencia (combo) y comentario opcional. Tipos del Paso 2 incluyen los del Paso 1 más tipos contextuales (Pago Incompleto Vencido, Pago Insuficiente).
- Tipo "Pago Incompleto Vencido": habilita el marcado del pedido como "Pendiente de cancelación por falta de pago" (cancelación reactiva). El marcado NO ejecuta cancelación fiscal efectiva ni dispara devolución; transferencia a Finanzas para gestión externa (cancelación fiscal vía Notas de Crédito y devolución por mecanismo fuera de R16).
- Auto-guardado del Paso 2 consistente con el Paso 1.
- Asociaciones del Paso 2 editables (modificables, deshacer y rehacer) mientras el usuario esté en el Paso 2. Una vez avanza al Paso 3, las asociaciones quedan fijas para la emisión de documentos fiscales.
- Navegación: Regresar (vuelve al Paso 1) o Continuar (avanza al Paso 3 con la asociación cerrada).

### No aplica a

- Paso 1 (Captura del Cobro) y Paso 3 (Facturación y Envío) del wizard: se documentan en requisitos independientes.
- Wizard de Validar Cobro para Región Perú: se documenta en requisito independiente con la misma estructura UI pero catálogos y reglas específicas de Perú.
- Catálogo de Tipos de Inconsistencia (definición de las opciones del combo del Paso 2): pendiente del lado de PROQUIFA (Tesorería). Tipos confirmados a la fecha: "Pago Incompleto Vencido" y "Pago Insuficiente".
- Generación de Notas de Crédito (emisión de nuevas NCs): se documenta en el módulo Notas de Crédito (módulo independiente).
- Auxiliar de saldos remanentes de Notas de Crédito aplicadas parcialmente: queda fuera de scope R16 (decisión cliente). El sistema NO gestiona saldos contables internos de NCs parcialmente aplicadas como entidad re-aplicable.
- Cancelación fiscal efectiva de proformas/facturas (vía CFDI de cancelación SAT): NO está contemplada en R16 fuera del módulo Notas de Crédito. La cancelación del Paso 2 únicamente marca el pedido para gestión externa por Finanzas.
- Sistema de devoluciones de dinero al cliente: NO contemplado en R16. La devolución del pago parcial recibido tras cancelación es operación externa a PQF2.
- Mecanismo de transferencia del estado "Pendiente de cancelación por falta de pago" al área de Finanzas: pendiente de definición.

---

## Reglas de Negocio

**Regla 1 — Aplicabilidad solo a Región México**
El Paso 2 del wizard de Validar Cobro opera exclusivamente sobre clientes con Región México. Los clientes con Región Perú son atendidos por el wizard equivalente Perú, con la misma UI pero catálogos y reglas específicas (requisito independiente).

**Regla 2 — Cobros disponibles para asociar**
El listado de cobros del cliente muestra todos los cobros capturados en el Paso 1 disponibles para asociar (cobros confirmados que aún no han sido aplicados totalmente a documentos). Cada cobro muestra su folio COB-mmddaa-consecutivo, monto y fecha. Los cobros con excedente disponible por aplicar (saldo a favor de cobros previos) se marcan visualmente con un identificador "saldo a favor" para que el usuario pueda identificarlos rápidamente.

**Regla 3 — Selección múltiple de cobros para asociación**
El usuario puede seleccionar uno o más cobros mediante checkboxes y aplicarlos a uno o más documentos en relación N a N. Combinaciones válidas: 1 cobro a 1 documento (1-1), 1 cobro a N documentos (1-N), N cobros a 1 documento (N-1), N cobros a N documentos (N-N).

**Regla 4 — Listado de Proformas y Facturas pendientes**
El listado central muestra todas las proformas y facturas pendientes de cobrar del cliente, mezcladas en el mismo listado, sin filtros adicionales por tipo, empresa emisora, fecha u otros criterios. Cada documento muestra: tipo (Proforma o Factura), folio, Pedido Interno, empresa emisora del grupo, importe total y saldo pendiente actual. Pueden mezclarse documentos de diferentes empresas emisoras del grupo PROQUIFA México (Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica) en la misma sesión; cada documento se procesará independientemente en el Paso 3.

**Regla 5 — Asociación manual N a N con aplicación en orden de selección**
El sistema no ofrece un campo "Monto a aplicar" editable por línea. El usuario controla la asociación cobro↔documento mediante: la selección de cuáles cobros aplica, la selección de cuáles documentos cubre, y el orden de selección de los documentos, que determina la prioridad de aplicación del monto disponible del cobro. El sistema aplica el monto del cobro a los documentos en el orden en que el usuario los seleccionó: el primer documento seleccionado se cubre primero, el segundo después, y así sucesivamente, hasta agotar el monto disponible del cobro. Si el cobro no alcanza para cubrir todos los documentos seleccionados, el último documento de la secuencia queda con saldo pendiente y la asociación queda pendiente del próximo cobro complementario.

**Regla 6 — Aplicación OPCIONAL de Notas de Crédito vigentes del cliente**
Cuando el cliente tiene una o más Notas de Crédito vigentes (emitidas previamente con UUID timbrado SAT), el sistema muestra el catálogo de NCs vigentes y permite al usuario seleccionar cuáles aplica al documento (cero, una o varias). La aplicación de NCs es OPCIONAL: el usuario no está obligado a aplicar todas las NCs vigentes del cliente; selecciona libremente cuáles aplica a cada documento. Las NCs no seleccionadas siguen vigentes y disponibles para sesiones futuras del módulo. Las NCs son por documento (cada NC se aplica completa a un solo documento, no se distribuye entre múltiples documentos en la misma sesión del Paso 2). El sistema referencia el UUID timbrado de cada NC aplicada para su uso en el Paso 3 (al emitir el documento fiscal correspondiente, las NCs aplicadas se relacionan al CFDI mediante el nodo CFDIRelacionados con su UUID y monto correspondiente, conforme normativa SAT).

**Regla 7 — Cálculo dinámico del saldo de la asociación**
El sistema calcula dinámicamente la suma del adeudo de los documentos seleccionados, la suma de los cobros seleccionados aplicados, la suma de las NCs aplicadas, y el saldo de la asociación = (cobros aplicados + NCs aplicadas) - adeudo total. El saldo se muestra al usuario con indicación clara del escenario resultante: cero (exacto), positivo (sobrepago → saldo a favor), negativo dentro de tolerancia (≤ 100 MXN, cierra con saldo pendiente en Estado de Cuenta), negativo fuera de tolerancia (> 100 MXN, requiere inconsistencia o asociación pendiente).

**Regla 8 — Escenario pago exacto**
Cuando la suma de cobros aplicados + NCs aplicadas iguala exactamente el adeudo total de los documentos seleccionados, el sistema permite avanzar al Paso 3 con la asociación cerrada para emisión de documentos fiscales.

**Regla 9 — Escenario sobrepago (saldo a favor)**
Cuando la suma de cobros aplicados + NCs aplicadas supera el adeudo total de los documentos seleccionados, el sistema permite al usuario avanzar al Paso 3 con la asociación que cubre el adeudo total; registra el excedente como saldo a favor del cliente; lo refleja en el Estado de Cuenta/Auxiliar del cliente; marca el cobro o cobros origen del excedente con identificador visible "saldo a favor" en el listado del Paso 1 para reconocimiento futuro del usuario; deja el saldo a favor disponible para aplicarse a una proforma o factura nueva del mismo cliente desde Validar Cobro en sesiones futuras; y no genera documento fiscal adicional (no Nota de Crédito, no devolución).

**Regla 10 — Escenario pago de menos con tolerancia 100 MXN**
Cuando la suma de cobros aplicados + NCs aplicadas es menor al adeudo total de los documentos seleccionados, el sistema bifurca su comportamiento según el monto de la diferencia faltante: si es ≤ 100 MXN (inclusiva del límite, Política Interna PROQUIFA), el sistema permite cerrar el documento con el monto recibido a través del registro manual del operador, refleja la diferencia como saldo pendiente en el Estado de Cuenta/Auxiliar del cliente (sin bloqueo) y permite avanzar al Paso 3; si es > 100 MXN, el sistema no permite avanzar al Paso 3 con esta asociación y el usuario debe marcar inconsistencia (tipo "Pago Insuficiente" o "Pago Incompleto Vencido" según corresponda) o dejar la asociación pendiente del próximo cobro complementario. La tolerancia 100 MXN es una Política Interna PROQUIFA aplicable a operaciones con moneda de facturación en MXN. ** Pendiente confirmar el tratamiento de la tolerancia cuando la moneda de facturación es USD u otra distinta a MXN (¿se convierte la tolerancia al equivalente del día?, ¿aplica únicamente a operaciones MXN?). **

**Regla 11 — Duda fiscal sobre el monto de la factura en tolerancia 100 MXN**
Cuando el usuario aplica la tolerancia 100 MXN para cerrar un documento con monto recibido menor al adeudo y el sistema avanza al Paso 3 para emitir la factura correspondiente: ** Pendiente confirmar con PROQUIFA y su asesor fiscal el tratamiento de la factura: (a) se timbra por el monto total de la proforma original con la diferencia reflejada como saldo pendiente en el Estado de Cuenta del cliente (postura conservadora alineada con regla SAT de operación comercial completa), o (b) se timbra por el monto efectivamente recibido del cliente (más limpio operativamente pero no corresponde al monto comercial original). Decisión pendiente de validación fiscal. **

**Regla 12 — Marcado de inconsistencias del Paso 2**
Al presionar "Marcar Inconsistencia en Cobro", el sistema abre el modal "Inconsistencia de Pago" con los campos: Tipo de Inconsistencia (combo del catálogo de tipos aplicables al Paso 2) y Comentario adicional (opcional, texto libre para describir detalle adicional al cliente). El catálogo del Paso 2 incluye los tipos del Paso 1 (intrínsecos del cobro) más tipos contextuales que requieren conocer el documento a cobrar (Pago Incompleto Vencido, Pago Insuficiente). ** El catálogo completo está pendiente de definición por PROQUIFA (Tesorería). Tipos confirmados a la fecha: "Pago Incompleto Vencido" y "Pago Insuficiente". **

**Regla 13 — Tipo "Pago Incompleto Vencido" con marcado del pedido para cancelación externa**
Cuando el usuario selecciona el tipo "Pago Incompleto Vencido" (escenario aplicable cuando el cliente realizó un pago parcial pero nunca completó el monto total y la operación se considera vencida) y confirma la inconsistencia, el sistema habilita adicionalmente la opción de marcar el pedido asociado al cobro como "Pendiente de cancelación por falta de pago". Este marcado no ejecuta la cancelación fiscal efectiva de las proformas o facturas asociadas (la cancelación fiscal vía CFDI de cancelación SAT no está contemplada en R16 fuera del módulo Notas de Crédito) ni dispara devolución del pago parcial recibido al cliente (no hay sistema de devoluciones contemplado en R16). El marcado únicamente notifica al área de Finanzas que el pedido debe gestionarse para cancelación efectiva externa y, si aplica, devolución del pago parcial. ** Pendiente definir el mecanismo de transferencia del estado y de la información del cobro parcial al área de Finanzas. **

**Regla 14 — Tipo "Pago Insuficiente" (diferencia > 100 MXN)**
Cuando el usuario detecta que el cobro es insuficiente y la diferencia faltante supera la tolerancia 100 MXN, al confirmar la inconsistencia "Pago Insuficiente" el sistema registra la inconsistencia, mantiene la asociación pendiente y notifica al cliente que el cobro debe ser complementado. La operación queda en espera de que el cliente complete la diferencia o de instrucciones de Tesorería; no se marca el pedido para cancelación automáticamente (la cancelación requiere el tipo "Pago Incompleto Vencido" explícito por parte del usuario).

**Regla 15 — Auto-guardado del Paso 2**
Cuando el usuario modifica asociaciones, aplica NCs o navega entre cobros y documentos, el sistema auto-guarda el estado del Paso 2 (asociaciones cobros↔documentos, NCs aplicadas, inconsistencias marcadas) para preservar el progreso. No existe botón "Guardar" manual; el guardado es transparente.

**Regla 16 — Asociaciones editables en el Paso 2**
Mientras el usuario esté en el Paso 2, el sistema permite deshacer asociaciones previas, reasociar cobros con otros documentos, y aplicar o remover NCs. Una vez el usuario avanza al Paso 3 con "Continuar", las asociaciones quedan fijas para la emisión de documentos fiscales.

**Regla 17 — Navegación: Regresar y Continuar**
El Paso 2 ofrece dos acciones: Regresar (botón al pie de la pantalla) vuelve al Paso 1 (Captura del Cobro) sin perder lo capturado (auto-guardado activo); Continuar avanza al Paso 3 (Facturación y Envío) con la asociación cerrada. Solo se permite el avance si el escenario resultante es pago exacto, pago con sobrepago (saldo a favor registrado) o pago de menos dentro de tolerancia 100 MXN.

**Regla 18 — Bloqueo de avance al Paso 3**
Cuando el escenario resultante de la asociación es pago de menos con diferencia > 100 MXN sin inconsistencia marcada, al presionar "Continuar" el sistema bloquea el avance; el usuario debe marcar inconsistencia o dejar la asociación pendiente del próximo cobro complementario.

**Regla 19 — Totales en moneda del cobro con conversión por TC capturado**
Cuando los documentos seleccionados (proformas o facturas) están en moneda distinta a la moneda del cobro, el sistema convierte el importe de cada documento a la moneda del cobro usando el tipo de cambio capturado en el Paso 1 del cobro seleccionado; muestra todos los totales del panel (Adeudo Proformas y Facturas, NCs aplicadas, Cobros aplicados, Adeudo restante, Saldo a favor generado) en la moneda del cobro; muestra el tipo de cambio aplicado de forma visible junto a cada documento convertido (tooltip al hacer hover o click sobre el importe convertido, indicando la fórmula de conversión: importe en moneda original × TC = importe en moneda del cobro); y valida que el monto del cobro (en su moneda) sea suficiente para cubrir la suma convertida de los documentos seleccionados. Si la moneda del documento coincide con la moneda del cobro, no aplica conversión y el importe se muestra tal cual.

**Regla 20 — NCs aplicadas en moneda del cobro**
Cuando una Nota de Crédito aplicada al documento está en moneda distinta a la moneda del cobro, el sistema convierte su monto aplicado a moneda del cobro usando el TC capturado en el Paso 1 del cobro seleccionado. El monto convertido es el que se descuenta del Adeudo restante en el panel de Totales.

---

## Riesgos

**Riesgo 1 — Tratamiento fiscal de tolerancia 100 MXN**
La regla permite cerrar el documento con monto recibido cuando la diferencia faltante es ≤ 100 MXN, reflejando la diferencia en el Estado de Cuenta del cliente. Sin embargo, no se ha definido formalmente si la factura se timbra por el monto total de la proforma original (manteniendo descuadre fiscal con el saldo en Estado de Cuenta) o por el monto efectivamente pagado (alineando factura con cobro pero perdiendo el monto comercial original). Esta es una decisión fiscal que tiene implicaciones contables y de cumplimiento SAT.

**Riesgo 2 — Saldo remanente de Notas de Crédito aplicadas parcialmente**
Cuando se aplica una NC del cliente y su valor excede el adeudo del documento cubierto, el saldo remanente de la NC no se gestiona en R16 (fuera de scope). Fiscalmente en México no existe mecanismo nativo SAT para tratar el saldo remanente de una NC como entidad re-aplicable a futuras facturas (una NC nueva no puede relacionarse a una compra futura como "saldo de crédito previo"). ** Pendiente confirmar el tratamiento operativo del saldo remanente: ¿se controla en algún auxiliar interno no fiscal?, ¿se descarta operativamente?, ¿se notifica al cliente y se gestiona fuera del sistema? La limitación se documenta y se comunica a Tesorería que el control del saldo remanente de NCs queda como auxiliar interno fuera del sistema. **

**Riesgo 3 — Cancelación del pedido sin cancelación fiscal ni devolución**
El tipo de inconsistencia "Pago Incompleto Vencido" marca el pedido como "Pendiente de cancelación por falta de pago", pero R16 no ejecuta la cancelación fiscal efectiva de las proformas o facturas asociadas (vía CFDI de cancelación SAT) ni dispara devolución del pago parcial recibido. Estas operaciones quedan como trabajo externo del área de Finanzas. Si el mecanismo de transferencia a Finanzas no está claramente definido, hay riesgo de pedidos huérfanos sin gestión efectiva. ** Pendiente definir el mecanismo de transferencia del estado y de la información del cobro parcial al área de Finanzas (transferencia a Legacy con flag, notificación por correo, reporte específico, módulo de consulta, otro mecanismo). **

---

## Notas Adicionales

- Esta fila documenta el Paso 2 del wizard de Validar Cobro (Asociación de cobros con Proformas/Facturas) para Región México exclusivamente. La estructura UI de la pantalla se reutiliza idénticamente para clientes Región Perú; las únicas diferencias son los catálogos y reglas específicas (tolerancia en USD, tipos de NC SUNAT, fuente del TC, etc.) y se documentan en requisito independiente.
- El Paso 2 se invoca desde el Paso 1 al presionar "Continuar" con al menos un cobro capturado.
- Asociación N a N: 1↔1, 1↔N, N↔1, N↔N. Documentos mezclables (proformas y facturas del mismo cliente, de diferentes empresas emisoras del grupo). Cada documento se procesa independientemente en el Paso 3.
- La aplicación del monto del cobro a los documentos seleccionados sigue el ORDEN DE SELECCIÓN del usuario: el primer documento seleccionado se cubre primero, el segundo después, y así sucesivamente, hasta agotar el monto disponible del cobro. No hay campo "Monto a aplicar" editable por línea; el usuario controla la prioridad mediante el orden en que selecciona los documentos.
- Si un cobro no cubre todos los documentos seleccionados, el último documento de la secuencia queda con saldo pendiente y la asociación queda pendiente del próximo cobro complementario.
- Visualización del correo de origen como referencia contextual: el icono de correo (📩) en cada item del listado de cobros abre un pop-up modal con los datos del correo origen (asunto, fecha, hora, cuerpo, comprobante de pago identificado en Paso 1). El pop-up es de solo lectura y se cierra mediante acción explícita del usuario, sin alterar la asociación en curso. La visualización persistente del correo en el área central no aplica en el Paso 2.
- Notas de Crédito: el cliente puede tener una o más NCs vigentes; el usuario selecciona libremente cuáles aplica a cada documento. La aplicación de NCs es OPCIONAL — el usuario puede aplicar cero, una o varias NCs por documento, según decida. Las NCs no aplicadas en la sesión actual siguen vigentes para sesiones futuras (el sistema no descarta NCs no usadas). Cada NC se aplica completa a un solo documento (NCs son por documento, no se distribuyen entre múltiples documentos en la misma sesión). El UUID timbrado de cada NC aplicada se preserva para uso en el Paso 3.
- Multi-divisa: la conversión se realiza al TC capturado en el Paso 1 del cobro. Todos los totales del panel se muestran en moneda del cobro. Cuando hay documentos en moneda distinta, el TC aplicado se expone vía tooltip (hover o click) junto a cada importe convertido con su fórmula de cálculo.
- Saldo a favor por sobrepago: se gestiona como abono en Estado de Cuenta + disponible para proforma/Factura por Adelantado futura del mismo cliente desde Validar Cobro. No se genera documento adicional. El cobro origen se marca con identificador visible "saldo a favor" en el Paso 1.
- Tolerancia 100 MXN: bifurca el comportamiento del sistema. ≤ 100 MXN inclusiva del límite cierra con monto recibido + saldo pendiente en Estado de Cuenta. > 100 MXN bloquea avance, requiere inconsistencia. Política Interna PROQUIFA validada por Tesorería y Contabilidad. Aplicable a operaciones MXN; pendiente definición para otras monedas.
- ** DUDA FISCAL PENDIENTE — TOLERANCIA 100 MXN: si la factura se timbra por el total de la proforma original (postura conservadora, mantiene descuadre con saldo en Estado de Cuenta) o por el monto efectivamente recibido (más limpio operativamente, pero la factura no refleja el monto comercial original). Decisión fiscal pendiente con asesor de PROQUIFA. **
- Inconsistencias del Paso 2 = tipos del Paso 1 (intrínsecos del cobro) + tipos contextuales del Paso 2 ("Pago Incompleto Vencido", "Pago Insuficiente"). Catálogo completo pendiente Tesorería.
- "Pago Incompleto Vencido": cliente pagó parcial y nunca completó (operación vencida). Marca pedido como "Pendiente de cancelación por falta de pago" o similar, pero NO ejecuta cancelación fiscal NI devolución (ambos fuera de R16). Transferencia a Finanzas para gestión externa.
- "Pago Insuficiente": cliente pagó menos pero la operación sigue activa (esperando complemento). Asociación queda pendiente del próximo cobro. No marca pedido para cancelación.
- IMPORTANTE — Sobre Notas de Crédito y saldo remanente: Fiscalmente en México (SAT) no existe mecanismo nativo para tratar el saldo remanente de una NC como entidad re-aplicable a futuras facturas. Una NC se emite por su valor total y se distribuye a uno o más CFDIs de Ingreso al momento de timbrarse vía el nodo CFDIRelacionados. El SAT no contempla "saldo de NC" como entidad que pueda aplicarse a una compra futura (cada venta es operación independiente y no debe relacionarse a NCs previas mediante tipo de relación). En consecuencia: si una NC aplicada en el Paso 2 excede el adeudo del documento cubierto, el saldo remanente queda como auxiliar contable interno de PROQUIFA, FUERA DE SCOPE R16 (consistente con decisión cliente).
- IMPORTANTE adicional — Sobre cancelación y devolución: R16 no contempla cancelación fiscal efectiva de facturas (vía CFDI de cancelación SAT) fuera del módulo Notas de Crédito. R16 tampoco contempla un sistema de devoluciones de dinero al cliente. Cuando se marca un pedido como "Pendiente de cancelación por falta de pago", la cancelación fiscal y la devolución (si aplica del pago parcial recibido) son operaciones EXTERNAS gestionadas por el área de Finanzas. PQF2 R16 únicamente deja registro del estado para que Finanzas opere.
- Auto-guardado del Paso 2: estado preservado al navegar o salir. Asociaciones editables mientras el usuario esté en el Paso 2. Una vez avanza al Paso 3, las asociaciones quedan fijas.
- Multi-divisa en el panel de Totales: las conversiones a moneda del cobro se realizan con el TC capturado en el Paso 1 del cobro (TC del día de la moneda no-MXN involucrada vs MXN). Todos los totales del panel se muestran en moneda del cobro. El TC aplicado se expone vía tooltip (hover o click) junto a cada importe convertido con su fórmula de cálculo.
- ** DUDA ABIERTA — Moneda base del TC capturado: actualmente se utiliza MXN como moneda base para todas las conversiones (consistente con la regla SAT del CFDI). Pendiente validar con asesor fiscal y PROQUIFA si esta es la opción correcta o si en algún escenario operativo conviene capturar el TC vs moneda distinta. Misma duda que en TPSC-RE-FU-024. **
- ** Pendiente definir el comportamiento operativo cuando el cobro y el documento están en monedas extranjeras distintas entre sí (cobro EUR + documento USD, por ejemplo). Escenario poco común para PROQUIFA. **
- ** Pendiente definir el catálogo completo de Tipos de Inconsistencia aplicables al Paso 2. Tipos confirmados: "Pago Incompleto Vencido", "Pago Insuficiente". **
- ** Pendiente confirmar el tratamiento de la tolerancia 100 MXN cuando la moneda de facturación del cliente es USD u otra distinta a MXN (¿se convierte al equivalente del día?, ¿aplica únicamente a operaciones MXN?). **
- ** Pendiente definir el tratamiento operativo del saldo remanente de Notas de Crédito aplicadas parcialmente (auxiliar interno no fiscal, descarte operativo, vinculación al cliente fuera del sistema, otro). Fuera de scope R16 confirmado, pero requiere decisión operativa. **
- ** Pendiente definir el mecanismo de transferencia del estado "Pendiente de cancelación por falta de pago" y de la información del cobro parcial al área de Finanzas (Legacy con flag, correo automático, reporte, módulo de consulta, otro). **
- ** Pendiente definir el comportamiento exacto del modal de inconsistencia cuando el tipo es "Pago Incompleto Vencido" (¿muestra opción "Marcar para cancelación" al confirmar?, ¿abre modal adicional?, ¿marca automáticamente al confirmar este tipo?). **
- ** Pendiente verificar la vigencia de Notas de Crédito en el sistema: ¿las NCs tienen fecha de caducidad?, ¿cómo se determina que una NC sigue "vigente" para ser ofrecida en el catálogo de aplicables?, ¿hay reglas de expiración? **
- ** Pendiente confirmar el cuerpo del correo de notificación al cliente cuando se marca inconsistencia "Pago Insuficiente": qué información lleva, formato, plantilla, mecanismo de envío. **
- ** Pendiente confirmar la fuente oficial del Tipo de Cambio del día para conversiones en el Paso 2 (propuesta estándar fiscal: TC FIX Banxico DOF — transversal con Paso 1). **
- ** Pendiente resolver formalmente la denominación canónica del rol operativo entre "Gestor de Cobranza" y "Analista de Cuentas por Cobrar" (transversal). **
