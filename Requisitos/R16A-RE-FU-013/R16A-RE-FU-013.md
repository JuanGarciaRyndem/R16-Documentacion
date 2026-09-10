# Tramitación de pedidos Prepago — Con controlados, no aplica FxA

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-013 |
| **Nombre** | Tramitación de pedidos Prepago (Con controlados, no aplica FxA) |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-001, R16.1M-RE-FU-006, R16.1M-RE-FU-007, R16.1M-RE-FU-008, R16.1M-RE-FU-015 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero tramitar pedidos de clientes con condición de pago Prepago que contienen sustancias controladas (Mundial, Nacional u Origen), para que el sistema genere la proforma correspondiente, la envíe al cliente y deje el pedido listo para que cobranza valide el cobro en el módulo Validar Cobro respetando las restricciones regulatorias del producto controlado.

---

## Requisito

El sistema debe permitir, en el módulo Tramitar Pedido, la tramitación de pedidos de clientes con condición de pago Prepago que contienen sustancias controladas tipo Mundial, Nacional u Origen, en la operación de México. Región Perú no está soportada para el manejo de Sustancias Controladas en esta release. Al tramitar, el sistema genera la proforma, la presenta al ESAC para validación visual y, tras confirmar el envío al cliente, genera el pendiente correspondiente en el módulo Validar Cobro asociado al pedido y la proforma. En este flujo las opciones Factura por Adelantado y Entrega con Remisión no están disponibles: ambas quedan excluidas por restricción regulatoria cuando el pedido contiene sustancias controladas.

---

## Alcance

### Aplica a

- Pedidos de clientes con condición de pago Prepago en la operación de México.
- Pedidos que contienen al menos una sustancia controlada clasificada como Mundial, Nacional u Origen.
- Generación de la proforma al ejecutar la acción Tramitar en el módulo Tramitar Pedido.
- Asignación del folio interno del pedido conforme a la mecánica actual del sistema.
- Asignación del folio de la proforma desde el foliador lineal global de PQF2 (un solo contador para todas las proformas del sistema, sin segmentación por empresa o región).
- Flujo de envío con dos pasos: previsualización del PDF de la proforma + pantalla de datos de envío del correo.
- Generación automática del pendiente en el módulo Validar Cobro al confirmar el envío del correo.

### No aplica a

- Pedidos de clientes con condición de pago Crédito (esos siguen los flujos descritos en los requisitos del bloque Crédito).
- Pedidos prepago sin sustancias controladas (variante cubierta en requisitos independientes del bloque Prepago).
- Pedidos con activación de Factura por Adelantado: no se permite con sustancias controladas (restricción regulatoria).
- Pedidos con marca de Entrega con Remisión (no permitida por regla regulatoria cuando hay controlados).
- Región Perú: el manejo de Sustancias Controladas no está contemplado dentro del alcance de esta release para Perú (confirmado por el cliente — Duda 061). Este flujo (Prepago con controlados) se acota a Región México. El sistema no restringe el avance por código; el control es operativo (ver Riesgo 3 y DUDA-027).
- Región Perú — Factura por Adelantado con controlados: confirmado (DUDA-029) que, igual que en México, la exclusión de Factura por Adelantado con productos controlados es también el criterio esperado para Perú. El tratamiento es el mismo que el de DUDA-027: manejo operativo, no restringido por código; no hay una validación de sistema distinta por región para este caso.
- La validación de presencia de Licencia Sanitaria y Aviso de Responsable Sanitario del cliente, que ocurre en el módulo Pretramitar Pedido antes de llegar a Tramitar Pedido.
- La validación del cobro de la proforma, la emisión de la factura anticipo, el timbrado fiscal, el cálculo de la FEE y la generación de la Confirmación de Pedido. Todas esas acciones ocurren en el módulo Validar Cobro y se cubren en requisitos independientes.

---

## Reglas de Negocio

**Regla 1 — No disponibilidad de Factura por Adelantado y Entrega con Remisión**
Para pedidos que contienen al menos una sustancia controlada tipo Mundial, Nacional u Origen, el sistema no ofrece las opciones de Factura por Adelantado ni de Entrega con Remisión: ambas quedan excluidas por restricción regulatoria. Esta exclusión aplica de forma consistente para México y Perú (DUDA-029): en México es una restricción de sistema; en Perú, dado que el manejo de controlados no está en alcance de esta release, la exclusión de Factura por Adelantado con controlados queda como el mismo criterio esperado a nivel operativo, no como una validación de sistema adicional.

**Regla 2 — Generación automática de proforma al tramitar**
Al ejecutar la acción de tramitar un pedido prepago con sustancias controladas, el sistema genera automáticamente una proforma en formato PDF.

**Regla 3 — Folio de la proforma desde foliador lineal global**
El folio de la proforma se toma del foliador lineal global de PQF2, que mantiene un solo contador para todas las proformas del sistema sin segmentación por empresa o región. El folio se consume en el momento en que el envío del correo de la proforma se completa exitosamente; un intento de envío fallido no consume folio (ver Regla 11 y Criterio C2bis).

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

**Regla 11 — Conservación del folio de proforma ante reintento de envío (DUDA-030)**
El folio de proforma asignado al tramitar se consume (se conserva) hasta que el envío del correo se complete exitosamente. Si el envío falla y el ESAC reintenta, el sistema reutiliza el MISMO folio ya asignado; no se descarta el folio ni se asigna uno nuevo en cada intento fallido.

---

## Riesgos

**Riesgo 1 — Clasificación incorrecta del producto en el catálogo**
La no visualización de las opciones Factura por Adelantado y Entrega con Remisión depende de cómo esté clasificado cada producto en el catálogo. El sistema actúa correctamente sobre el dato que recibe; el riesgo está en que ese dato de clasificación sea incorrecto: si un producto controlado no está clasificado como tal, el sistema podría ofrecer opciones que violan la restricción regulatoria.

**Riesgo 2 — Avance de pedidos con controlados de Región Perú (riesgo operativo asumido)**
El manejo de Sustancias Controladas para Región Perú no está contemplado en el alcance de esta release (confirmado por el cliente — Duda 061). El sistema no impide que un pedido con controlados de un cliente Perú avance por el flujo de tramitación; el control es operativo, no de sistema (DUDA-027).

---

## Criterios de Aceptación

### Sección A — Tramitación y restricciones regulatorias

**Criterio A1 — Tramitación habilitada para Prepago con controlados (Región México)**
- **Dado** que un pedido pertenece a un cliente Prepago de Región México y contiene al menos una sustancia controlada tipo Mundial, Nacional u Origen,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación y, al ejecutarse, generar automáticamente la proforma asociada al pedido.

**Criterio A2 — No disponibilidad de opciones bloqueadas por controlados**
- **Dado** que un pedido contiene al menos una sustancia controlada (escenario soportado únicamente para Región México),
- **Cuando** el ESAC visualiza la pantalla del pedido,
- **Entonces** las opciones de Factura por Adelantado y Entrega con Remisión no deberán estar disponibles, por restricción regulatoria.

### Sección B — Folios y generación de la proforma

**Criterio B1 — Asignación de folio interno al tramitar**
- **Dado** que el ESAC ejecuta la acción de tramitar,
- **Cuando** el sistema procesa la solicitud,
- **Entonces** deberá asignar el folio interno del pedido siguiendo la mecánica actual del sistema.

**Criterio B2 — Generación del PDF de la proforma**
- **Dado** que el ESAC ejecuta la acción de tramitar,
- **Cuando** el sistema genera la proforma asociada al pedido,
- **Entonces** el sistema deberá generar automáticamente el archivo PDF de la proforma con los datos del pedido, del cliente y el folio correspondiente, tomado del foliador lineal global de PQF2 (un solo contador compartido por todas las proformas del sistema; el folio se consume al completarse el envío exitoso del correo, ver Regla 3 y Criterio C6).

### Sección C — Previsualización y envío de la proforma

**Criterio C1 — Previsualización obligatoria del PDF antes del envío**
- **Dado** que el PDF de la proforma se generó exitosamente,
- **Cuando** el sistema inicia el proceso de envío,
- **Entonces** deberá mostrar al ESAC la previsualización del PDF y requerir aceptación explícita antes de continuar a la pantalla de datos de envío.

**Criterio C2 — Cancelación desde la previsualización**
- **Dado** que el ESAC está viendo la previsualización del PDF de la proforma,
- **Cuando** el ESAC decide no continuar (cancela la previsualización),
- **Entonces** el sistema deberá permitir volver al pedido sin enviar la proforma.

> ~~**Pendiente definir** la política del folio de proforma ya asignado: si se conserva para el reintento o se descarta.~~ **Resuelto (DUDA-030):** el folio se consume/conserva hasta el envío exitoso; ver Criterio C2bis y Regla 11.

**Criterio C2bis — Conservación del folio de proforma en reintento de envío**
- **Dado** que la proforma ya tiene folio asignado y el envío del correo falla o el ESAC cancela la previsualización,
- **Cuando** el ESAC reintenta el envío,
- **Entonces** el sistema deberá reutilizar el mismo folio de proforma ya asignado, sin descartarlo ni generar uno nuevo, hasta que el envío se complete exitosamente (DUDA-030).

**Criterio C3 — Pantalla de datos de envío con CC editable y ESAC incluido**
- **Dado** que el usuario llegó al paso de envío del correo,
- **Cuando** el sistema muestra la pantalla,
- **Entonces** deberá mostrar: Para con el contacto del cliente (editable, default heredado); CC con el ESAC asignado (editable, default sugerido); Asunto generado por sistema según plantilla (no editable); Adjuntos con el PDF de la proforma (no editables); y Notas extras como campo de texto editable opcional.

**Criterio C4 — Envío del correo de proforma**
- **Dado** que la pantalla de envío está completa,
- **Cuando** el usuario presiona Enviar,
- **Entonces** el sistema deberá enviar el correo al destinatario con CC al ESAC, asunto generado por sistema, adjunto del PDF de la proforma y las notas extras opcionales capturadas.

**Criterio C5 — Generación automática del pendiente Validar Cobro**
- **Dado** que el correo de proforma se envió exitosamente,
- **Cuando** se completa el envío,
- **Entonces** el sistema deberá generar automáticamente un pendiente en el módulo Validar Cobro asociado al folio del pedido y la proforma emitida.

**Criterio C6 — Consumo del folio de proforma al completar el envío**
- **Dado** que el correo de la proforma se envía exitosamente,
- **Cuando** se confirma el envío,
- **Entonces** el sistema deberá consumir de forma definitiva el folio de proforma asignado; un intento de envío fallido no consume el folio (ver Criterio C2bis y Regla 11).

### Sección D — Cancelación

**Criterio D1 — Cancelación del pedido**
- **Dado** que un pedido tiene solicitud del cliente para cancelar,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder.

---

## Notas

- Variante prepago con sustancias controladas del módulo Tramitar Pedido. El módulo de Tramitar Pedido en este flujo es responsable de generar la proforma, gestionar la previsualización y envío al cliente, y disparar el pendiente en Validar Cobro. Lo que ocurre tras Validar Cobro (factura anticipo, timbrado, Confirmación, FEE, transferencia a Legacy en caso de México) está fuera del scope de este requisito y se cubre en requisitos del módulo Validar Cobro.
- Cubre tres requisitos del cliente: tramitación de pedidos prepago en México y Perú; generación y envío automático de proforma para prepago sin Factura por Adelantado; y cadena de pendientes tras la generación de la proforma.
- La presencia de sustancias controladas determina dos efectos en el módulo: no están disponibles las opciones de Factura por Adelantado ni Entrega con Remisión (restricción regulatoria); y la factura que se emita posteriormente en Validar Cobro será del tipo factura anticipo (no factura normal).
- El avance de un pedido con controlados de Región Perú por el flujo de tramitación se asume como riesgo operativo; ver Riesgo 2.
- El foliador de la proforma es lineal global a PQF2 (un solo contador para todas las proformas del sistema). El folio del pedido interno conserva la mecánica actual del sistema. El folio de la proforma se consume al completarse exitosamente el envío del correo; un intento fallido no lo consume (Regla 3, Regla 11, Criterio C2bis, Criterio C6).
- El asunto del correo de proforma se compone como "Proforma" más el folio del pedido interno.
- El flujo de envío del correo de proforma requiere dos pasos secuenciales en la UI: primero previsualizar y aceptar el PDF; después confirmar los datos de envío del correo.
- La validación de documentos regulatorios del cliente (Licencia Sanitaria y Aviso de Responsable Sanitario) ocurre antes de llegar a Tramitar Pedido (responsabilidad del módulo Pretramitar Pedido).
- Aplicable únicamente a la operación de México. Región Perú no está en el alcance de esta release para el manejo de sustancias controladas (Duda 061 / DUDA-027).
- **Trazabilidad (2026-08-21):** DUDA-027 confirma que el avance de un controlado de Perú por el flujo de tramitación no tiene bloqueo técnico y se asume como riesgo operativo comunicado al cliente (Riesgo 2). DUDA-029 confirma que la exclusión de Factura por Adelantado con controlados aplica igual para México y Perú, con control operativo y no por código (Regla 1, Alcance). DUDA-030 resuelve que el folio de proforma se conserva y reintenta con el mismo folio hasta el envío exitoso, sin descartarlo (Regla 11, Criterio C2bis, Criterio C6).

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|-------------------------|
| 1 | 2026-09-04 | Verificación contra matriz | Se comparó este documento contra un nuevo extracto de la matriz de requisitos. El extracto corresponde a una versión más antigua que no refleja las resoluciones ya cerradas aquí (DUDA-027, DUDA-029, DUDA-030, Duda 061; Regla 12; Criterio C2bis); no se aplicaron esos cambios para no reabrir preguntas ya resueltas. Diferencias menores de redacción del extracto, sin impacto en el contenido, no se incorporaron. |
| 2 | 2026-09-10 | Ajuste por retiro del timbrado de Perú / corrección del foliador / consistencia | Se retira la justificación fiscal-aduanera de México (pedimento) del Requisito, Alcance, Regla 1 y Notas; la restricción de Factura por Adelantado/Entrega con Remisión con controlados queda fundamentada únicamente en la restricción regulatoria-sanitaria. Se elimina la Regla 11 (panel regionalizado México/Perú); la antigua Regla 12 se renumera a **Regla 11** y se le agrega la precisión del momento de consumo del folio (se consume al completarse el envío del correo de la proforma; un envío fallido no consume folio, ver Criterio C2bis y C6). Se elimina el Riesgo 2 (confusión de campos fiscales, ligado al panel regionalizado eliminado); el antiguo Riesgo 3 se renumera a **Riesgo 2**, se retira la referencia a la "decisión acordada entre Osmar y Robert" y queda fundamentado en Duda 061/DUDA-027. Se elimina el Criterio A3 (composición regionalizada del panel) y se simplifica A2 (se retira detalle de UI y justificación por pedimento). Se elimina el Criterio B2 (asignación de folio al tramitar, ya no aplica); el antiguo Criterio B3 se renumera a **Criterio B2** y su redacción sustituye "se completa la asignación de folios" por el nuevo criterio de consumo al envío. Se agrega el nuevo **Criterio C6 — Consumo del folio de proforma al completar el envío**, referenciado desde Regla 11 y Criterio C2bis. En Notas se corrige la contradicción de aplicabilidad regional ("México y Perú" → únicamente México), se elimina la nota de capacitación al equipo operativo sobre campos fiscales de Perú (ya no aplica, el timbrado de Perú se retira de esta release), y se actualizan las referencias cruzadas Regla 12→11 y Riesgo 3→2 en toda la Trazabilidad y el resto del documento. |
