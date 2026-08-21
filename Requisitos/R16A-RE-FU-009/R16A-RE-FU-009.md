# Validación regulatoria

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-009 |
| **Nombre** | Validación regulatoria |
| **Módulo** | Tramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.3M-RE-FU-004 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero que el sistema valide automáticamente que el cliente cuente con la documentación regulatoria registrada en su catálogo cuando el pedido contiene sustancias controladas, para que el pedido completo permanezca retenido hasta que la documentación esté registrada y se procese solo cuando el cumplimiento normativo esté garantizado.

---

## Requisito

El sistema debe validar automáticamente, como **último paso del módulo Tramitar Pedido**, que el cliente del pedido tenga registrados en el Catálogo de Clientes los documentos regulatorios requeridos cuando el pedido contiene sustancias controladas tipo Mundial, Nacional u Origen (Licencia Sanitaria y Aviso de Responsable Sanitario para Región México). Si alguno de los documentos requeridos no está registrado, el sistema debe **retener el pedido completo** e indicar al ESAC que revise la documentación regulatoria en la configuración del cliente. El pedido se procesa únicamente cuando la documentación queda registrada. La validación se realiza sobre la presencia del registro, sin verificación de vigencia. El pedido se procesa siempre como una unidad completa, sin separación de partidas por su clasificación regulatoria.

---

## Alcance

### Aplica a

- Pedidos en el módulo **Tramitar Pedido** que contienen sustancias controladas clasificadas como Mundial, Nacional u Origen.
- Validación de presencia (registro) en el Catálogo de Clientes de los documentos regulatorios requeridos según la Región del cliente: para México, Licencia Sanitaria y Aviso de Responsable Sanitario.
- **Retención del pedido completo** cuando la validación falla — el pedido no se procesa hasta que la documentación quede registrada.
- **Tramitación del pedido completo en su folio original** una vez que la documentación se registra, sin generar folios adicionales.
- Notificación al ESAC indicando que el cliente no cuenta con la documentación regulatoria requerida.
- La validación se ejecuta **como último paso de Tramitar Pedido**, después de cualquier ajuste operativo (fletes, direcciones, ajustes a la OC, aceptación de OC Interna, tramitación con errores). Todos los caminos de entrada a Tramitar Pedido pasan por esta validación antes de procesar el pedido.

### No aplica a

- Pedidos sin sustancias controladas (la validación no se invoca; el pedido avanza sin verificación regulatoria adicional).
- Validación de fecha de vigencia o validez de los documentos registrados (solo se valida que estén registrados, no que estén vigentes).
- Separación del pedido por documentación regulatoria: el pedido se procesa siempre como una unidad completa, sin fragmentar partidas controladas y no controladas en pedidos separados con folios distintos.
- Facturación por adelantado sobre partidas no controladas mientras las controladas esperan documentación: la facturación por adelantado aplica al pedido completo cuando corresponde, no a partidas fragmentadas del pedido.
- Región Perú: las validaciones regulatorias y su condicionante en Tramitar Pedido se construyen únicamente para México (ver R16A-RE-FU-003).

---

## Reglas de Negocio

**Regla 1 — Validación regulatoria solo aplica cuando hay sustancias controladas**
La validación regulatoria del módulo Tramitar Pedido se ejecuta únicamente cuando el pedido contiene sustancias controladas clasificadas como Mundial, Nacional u Origen. Los pedidos sin sustancias controladas no invocan esta validación y siguen su flujo normal.

**Regla 2 — Documentos regulatorios a validar (Región México)**
Los documentos regulatorios que la validación verifica en el Catálogo de Clientes son, para Región México: Licencia Sanitaria y Aviso de Responsable Sanitario. Para Región Perú no se construye validación regulatoria en esta release (ver R16A-RE-FU-003).

**Regla 3 — Validación sobre presencia del registro, no sobre vigencia ni contenido**
La validación comprueba únicamente que los documentos estén registrados como información asociada al cliente. No valida fechas de vigencia, estados de validez, contenido del archivo ni firmas digitales. Un documento registrado satisface la validación para ese documento.

**Regla 4 — Retención del pedido completo ante validación fallida**
Cuando al menos uno de los documentos requeridos no está registrado, el sistema **retiene el pedido completo** en Tramitar Pedido y muestra al ESAC un mensaje que lo invita a revisar la documentación regulatoria en la configuración del cliente. El pedido no se procesa mientras la documentación no quede registrada, independientemente de si el cliente acepta o no entregas parciales y de si el pedido tiene además partidas no controladas.

**Regla 5 — Tramitación en el folio original al registrarse la documentación**
Una vez que la documentación regulatoria queda registrada en el Catálogo de Clientes, el pedido retenido se procesa en su **folio original** cuando el ESAC ejecuta nuevamente el avance. El sistema no genera un pedido nuevo con folio distinto para las partidas controladas ni para el pedido resultante — es el mismo pedido continuando su flujo.

**Regla 6 — Documentación regulatoria a nivel cliente**
La documentación regulatoria (Licencia Sanitaria y Aviso de Responsable Sanitario) se registra a **nivel cliente** en el Catálogo de Clientes y aplica a todos los pedidos del cliente que contengan sustancias controladas. No se registra por pedido ni por partida.

**Regla 7 — Validación como último paso de Tramitar Pedido**
La validación regulatoria se ejecuta como el **último paso del módulo Tramitar Pedido**, después de cualquier ajuste operativo previo (fletes, direcciones, ajustes a la OC, aceptación de OC Interna del cliente, tramitación con errores desde Gestionar Intramitable, etc.). Cualquier camino que lleve un pedido a Tramitar Pedido pasa por esta validación antes de procesar.

---

## Riesgos

**Riesgo 1 — Documentos registrados pero no vigentes**
Como la validación solo verifica la presencia del registro (no la vigencia), un cliente podría tener Licencia Sanitaria o Aviso de Responsable Sanitario registrados pero vencidos legalmente. El sistema permitiría procesar el pedido sin detectar esta situación.

**Riesgo 2 — Documentos registrados con archivos incorrectos**
Como la validación solo verifica que el documento esté registrado (sin validar contenido), un cliente podría tener archivos registrados como Licencia Sanitaria pero que en realidad contengan información incorrecta o sean documentos distintos. El sistema lo daría por válido.

**Riesgo 3 — Pedido completo retenido por falta de actualización del catálogo**
Si el cliente solicita un pedido con sustancias controladas y su catálogo no tiene los documentos registrados, el pedido completo queda retenido en Tramitar Pedido y el ESAC debe esperar a que el área responsable actualice el Catálogo de Clientes antes de poder procesar el pedido. Esto puede generar fricción operativa y demoras en la atención al cliente, incluso sobre las partidas no controladas que forman parte del mismo pedido.

---

## Criterios de Aceptación

### Sección A — Determinación de aplicabilidad

**Criterio A1 — Detección de pedido con sustancias controladas**
- **Dado** que el ESAC opera un pedido en el módulo Tramitar Pedido,
- **Cuando** el sistema evalúa el contenido del pedido,
- **Entonces** deberá identificar si contiene sustancias controladas clasificadas como Mundial, Nacional u Origen para determinar si debe ejecutar la validación regulatoria.

**Criterio A2 — Validación no se ejecuta para pedidos sin controlados**
- **Dado** que el pedido no contiene sustancias controladas,
- **Cuando** el ESAC intenta procesar el pedido,
- **Entonces** el sistema no deberá ejecutar la validación regulatoria. El pedido continúa el flujo normal sin verificación de documentos regulatorios.

### Sección B — Ejecución de la validación y resultado

**Criterio B1 — Validación regulatoria automática como último paso de Tramitar Pedido**
- **Dado** que el pedido contiene sustancias controladas y el ESAC intenta procesar el pedido en Tramitar Pedido,
- **Cuando** se ejecuta el último paso de Tramitar Pedido,
- **Entonces** el sistema deberá consultar automáticamente el Catálogo de Clientes y verificar la presencia de los documentos regulatorios correspondientes (Licencia Sanitaria y Aviso de Responsable Sanitario para Región México) registrados en el catálogo del cliente.

**Criterio B2 — Validación solo sobre presencia del registro**
- **Dado** que el sistema ejecuta la validación regulatoria,
- **Cuando** consulta el Catálogo de Clientes,
- **Entonces** deberá considerar el documento como válido si está registrado, independientemente de su fecha de carga, fecha de vigencia o cualquier otro atributo. No valida vigencia, contenido del archivo ni firmas digitales en este release.

**Criterio B3 — Retención del pedido completo cuando falta uno o ambos documentos**
- **Dado** que la validación detecta que al menos uno de los documentos requeridos no está registrado en el Catálogo de Clientes,
- **Cuando** el ESAC intenta procesar el pedido,
- **Entonces** el sistema deberá **retener el pedido completo** en Tramitar Pedido, sin procesarlo. Ninguna partida del pedido (controladas ni no controladas) avanza mientras la documentación no quede registrada.

**Criterio B4 — Mensaje al ESAC al retener el pedido**
- **Dado** que el pedido fue retenido por validación regulatoria fallida,
- **Cuando** se notifica al ESAC,
- **Entonces** el sistema deberá mostrar un mensaje indicando que no es posible procesar el pedido porque el cliente no cuenta con la documentación regulatoria requerida para productos controlados, e invitándolo a revisar la sección de documentos regulatorios en la configuración del cliente.

**Criterio B5 — Tramitación permitida cuando los documentos requeridos están registrados**
- **Dado** que la validación confirma que los documentos regulatorios están registrados en el Catálogo de Clientes,
- **Cuando** el ESAC ejecuta la acción de procesar,
- **Entonces** el sistema deberá permitir la tramitación del pedido completo en su folio original, sin interrupción ni mensaje adicional.

**Criterio B6 — Tramitación del pedido retenido en su folio original al registrarse la documentación**
- **Dado** que un pedido fue retenido en Tramitar Pedido por falta de documentación regulatoria y el área responsable registró posteriormente los documentos requeridos en el Catálogo de Clientes,
- **Cuando** el ESAC ejecuta nuevamente el avance del pedido en Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación del pedido completo en su **folio original**, sin generar folios adicionales ni fragmentar el pedido en partidas separadas.

---

## Notas

- Este es el único cambio funcional que R16 introduce al módulo Tramitar Pedido en materia regulatoria. El resto del flujo del módulo opera conforme al sistema preexistente en PQF2 sin modificaciones funcionales.
- La validación regulatoria se ejecuta **como último paso de Tramitar Pedido**. Todos los caminos que pueden llevar un pedido a Tramitar Pedido (avance desde Pretramitar Pedido, OC corregida validada, tramitación con errores desde Gestionar Intramitable, aceptación de OC Interna del cliente) pasan por esta validación antes de procesar.
- Cubre el requisito del cliente sobre verificación previa a la tramitación de que el cliente cuente con los documentos regulatorios registrados, cuando el pedido contenga sustancias controladas.
- La validación es estrictamente sobre presencia del registro en el Catálogo de Clientes, no sobre vigencia ni contenido. La responsabilidad de mantener los documentos vigentes y correctos recae en el proceso de actualización del Catálogo de Clientes.
- El módulo donde se cargan o actualizan los documentos es Catálogo de Clientes, no Tramitar Pedido. Si la validación retiene un pedido, el ESAC debe coordinar con el área responsable del Catálogo de Clientes para registrar la documentación antes de poder procesar el pedido.
- El pedido se procesa siempre como una **unidad completa** — el sistema no separa partidas controladas y no controladas en pedidos con folios distintos. La documentación regulatoria se registra a **nivel cliente** y aplica a todo el pedido.
- Aplicable a **Región México**. Para Región Perú no se construye validación regulatoria en esta release (ver R16A-RE-FU-003).

**Resueltos (dudas cerradas):**
- **Puntos de entrada a Tramitar Pedido** (DUDA-024): la validación se ubica como último paso de Tramitar Pedido, con lo que todos los caminos de entrada (avance desde Pretramitar, OC corregida, tramitación con errores desde Gestionar Intramitable, aceptación de OC Interna) pasan por ella antes de procesar.
- **Cambio de familia del producto a controlado** (DUDA-025): deja de ser un problema al validarse en el último paso de Tramitar Pedido — cualquier reclasificación previa queda cubierta por esta validación final.
- **Pedidos que mezclan partidas controladas y no controladas:** confirmado por el cliente que el escenario **no ocurre** en la operación real. En consecuencia, se retira la mecánica de separación que lo presuponía; el pedido se procesa siempre como unidad completa.
- **Alcance Perú** (DUDA-027 / DUDA-029 / DUDA-051): el cliente confirmó que Perú no soporta sustancias controladas en R16; el riesgo de tramitar/facturar un controlado de Perú se asume como riesgo operativo comunicado al cliente, sin bloqueo técnico ni validación regulatoria por sistema para esa región.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-08-14 | Retiro de separación de partidas | Historia, Requisito, Alcance y Reglas 4–6 reescritos para retención y tramitación del pedido completo (sin separación por documentación regulatoria). Se retira la tramitación de partidas no controladas sin esperar documentación de las controladas. Se retira la generación de folio nuevo distinto. Regla 6 pasa a "documentación regulatoria a nivel cliente". Riesgo 3 reescrito en términos del pedido completo retenido. Criterios B3 y B6 reescritos. Criterios A1, B1 y B4 ajustan la redacción de "al menos una sustancia controlada" a "sustancias controladas". |
| 2 | 2026-08-14 | Cierres de dudas | Punto de entrada a Tramitar Pedido: validación ubicada como último paso de Tramitar Pedido (cubre todos los caminos: avance desde Pretramitar, OC corregida, tramitación con errores, aceptación OC Interna). Cambio de familia del producto a controlado: deja de ser problema al validarse en el último paso. Pedido mezcla controlados/no controlados: confirmado que no ocurre en operación real; se retira mecánica de separación. |
| 3 | 2026-08-14 | Correcciones de consistencia | Se corrigen las referencias al módulo Pretramitar Pedido, que quedaron desactualizadas al moverse la validación regulatoria a Tramitar Pedido. El campo "Módulo" pasa de Pretramitar Pedido a Tramitar Pedido. |
| 4 | 2026-08-21 | Trazabilidad con dudas a cliente | Se añaden referencias a DUDA-024, DUDA-025, DUDA-027/029/051 en "Resueltos" para trazabilidad. Se actualizan `R16A-RE-FU-009-Back.md` y `R16A-RE-FU-009_BD.md` marcando como obsoletos/retirados los objetos de BD y la lógica de bifurcación por entregas parciales (`AceptaEntregasParciales`, `IdPedidoOrigenControlado`, pedido hijo) y los pendientes de documentación regulatoria para Perú, consistente con las decisiones ya reflejadas en este documento. Se anota el cierre de los hallazgos H-01, H-02 y H-07 en `R16A-RE-FU-009_DIS-SOL_Revision.md`. |
