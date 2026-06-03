# Validación regulatoria

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-009 |
| **Nombre** | Validación regulatoria |
| **Módulo** | Pretramitar Pedido |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.3M-RE-FU-004 |

---

## Historia de Usuario

> Yo como **ESAC**, quiero que el sistema valide automáticamente que un cliente cuente con la documentación regulatoria registrada en su catálogo antes de permitir que un pedido con sustancias controladas avance del módulo Pretramitar Pedido al módulo Tramitar Pedido, para asegurar el cumplimiento normativo y evitar tramitar pedidos sin el soporte documental requerido.

---

## Requisito

El sistema debe validar automáticamente, en el módulo Pretramitar Pedido, que el cliente del pedido tenga registrados en el Catálogo de Clientes los documentos regulatorios requeridos cuando el pedido contiene al menos una sustancia controlada tipo Mundial, Nacional u Origen (Licencia Sanitaria y Aviso de Responsable Sanitario para Región México; documentos equivalentes según normativa DIGEMID para Región Perú **). Si alguno de los documentos requeridos no está registrado, el sistema debe bloquear el avance del pedido hacia Tramitar Pedido e indicar al ESAC que revise la documentación regulatoria en la configuración del cliente. La validación se realiza únicamente sobre la presencia del registro, sin verificación de vigencia.

---

## Alcance

### Aplica a

- Pedidos en el módulo Pretramitar Pedido que contienen al menos una sustancia controlada clasificada como Mundial, Nacional u Origen.
- Validación de presencia (registro) en el Catálogo de Clientes de los documentos regulatorios requeridos según la Región del cliente: para México, Licencia Sanitaria y Aviso de Responsable Sanitario; para Perú, los documentos equivalentes según normativa DIGEMID.
- Bloqueo del avance del pedido al módulo Tramitar Pedido cuando la validación falla.
- Notificación al ESAC indicando qué documento(s) específicos no están registrados en el catálogo del cliente.
- Operaciones México y Perú: la validación aplica con la misma mecánica en ambas regiones, verificando en cada caso los documentos regulatorios correspondientes a la Región del cliente.

### No aplica a

- Pedidos sin sustancias controladas (la validación no se invoca para estos pedidos; avanzan al flujo de Tramitar Pedido sin verificación regulatoria adicional).
- Validación de fecha de vigencia o validez de los documentos registrados (solo se valida que estén registrados, no que estén vigentes).

---

## Reglas de Negocio

**Regla 1 — Validación regulatoria solo aplica cuando hay sustancias controladas**
La validación regulatoria del módulo Pretramitar Pedido se ejecuta únicamente cuando el pedido contiene al menos una sustancia controlada clasificada como Mundial, Nacional u Origen. Los pedidos sin sustancias controladas no invocan esta validación y siguen su flujo normal.

**Regla 2 — Documentos regulatorios a validar según Región**
Los documentos regulatorios que la validación verifica en el Catálogo de Clientes dependen de la Región del cliente del pedido. Para Región México son dos: Licencia Sanitaria y Aviso de Responsable Sanitario. Para Región Perú son los documentos regulatorios equivalentes según normativa DIGEMID.

> ** Denominación exacta de los documentos para Perú pendiente de confirmar con el cliente. **

**Regla 3 — Validación sobre presencia del registro, no sobre vigencia ni contenido**
La validación comprueba únicamente que los documentos estén registrados como información asociada al cliente. No valida fechas de vigencia, estados de validez, contenido del archivo ni firmas digitales. Un documento registrado satisface la validación para ese documento.

**Regla 4 — Bloqueo del avance ante validación fallida**
Cuando al menos uno de los documentos requeridos no está registrado, el sistema bloquea el avance del pedido del módulo Pretramitar Pedido al módulo Tramitar Pedido y muestra al ESAC un mensaje genérico que lo invita a revisar la documentación regulatoria en la configuración del cliente. El mensaje no especifica cuál documento falta.

**Regla 5 — Avance permitido cuando los documentos requeridos están registrados**
Cuando los documentos regulatorios correspondientes a la Región del cliente están registrados en el Catálogo de Clientes, el sistema permite el avance del pedido al módulo Tramitar Pedido, dando por satisfecha la validación regulatoria.

---

## Riesgos

**Riesgo 1 — Documentos registrados pero no vigentes**
Como la validación solo verifica la presencia del registro (no la vigencia), un cliente podría tener Licencia Sanitaria o Aviso de Responsable Sanitario registrados pero vencidos legalmente. El sistema permitiría avanzar el pedido a Tramitar Pedido sin detectar esta situación.

**Riesgo 2 — Documentos registrados con archivos incorrectos**
Como la validación solo verifica que el documento esté registrado (sin validar contenido), un cliente podría tener archivos registrados como Licencia Sanitaria pero que en realidad contengan información incorrecta o sean documentos distintos. El sistema lo daría por válido.

**Riesgo 3 — Cliente bloqueado en operación por falta de actualización del catálogo**
Si el cliente solicita un pedido con controlados y su catálogo no tiene los documentos registrados, el ESAC quedará bloqueado en Pretramitar y deberá esperar a que el área responsable actualice el Catálogo de Clientes antes de poder avanzar. Esto puede generar fricción operativa y demoras en la atención al cliente.

**Riesgo 4 — Denominación regulatoria Perú no definida**
Para clientes con Región Perú, la denominación exacta de los documentos regulatorios equivalentes (regulados por DIGEMID en lugar de COFEPRIS) no está confirmada por el cliente.

> ** Si el desarrollo arranca sin esta definición, la validación para clientes Perú podría verificar documentos con denominaciones incorrectas. Pendiente clarificar con el cliente la nomenclatura exacta para Perú antes del desarrollo. **

**Riesgo 5 — Puntos de entrada a Tramitar Pedido que no pasan por la validación regulatoria de Pretramitar**
La validación regulatoria se ejecuta solo en el avance de Pretramitar Pedido a Tramitar Pedido. Existen otros caminos por los que un pedido con sustancias controladas puede llegar a Tramitar Pedido sin pasar por Pretramitar (gestión de pedido intramitable con OC corregida validada en Validar ajustes a la OC, decisión de "tramitar con errores", y aceptación de OC Interna por el cliente). Si la validación no se re-ejecuta en esos puntos, un pedido con sustancias controladas podría tramitarse sin el soporte documental regulatorio requerido.

> ** Comportamiento de la validación en esos puntos de entrada pendiente de confirmar con el cliente (transversal México y Perú). **

**Riesgo 6 — Documentos regulatorios retirados después de la validación**
La validación regulatoria comprueba la presencia de los documentos en el momento de avanzar de Pretramitar Pedido a Tramitar Pedido. Si después de esa validación los documentos se retiran o eliminan del Catálogo de Clientes, y en Tramitar Pedido no se vuelve a validar, el pedido podría tramitarse con sustancias controladas sin el soporte documental regulatorio que en su momento sí existía.

> ** Comportamiento pendiente de confirmar con el cliente: ¿se re-valida la presencia de documentos en Tramitar Pedido, o la validación de Pretramitar es definitiva? (transversal México y Perú). **

**Riesgo 7 — Cambio de familia del producto a controlado después de la validación**
La validación regulatoria solo se invoca cuando el pedido contiene al menos una sustancia controlada (Mundial, Nacional u Origen). Si en Pretramitar Pedido el producto no era controlado (por lo que no se exigió documentación) y posteriormente se reclasifica a una familia controlada ya en Tramitar Pedido, el pedido podría quedar con sustancias controladas sin haber pasado nunca por la validación regulatoria.

> ** Comportamiento pendiente de confirmar con el cliente: ¿un cambio de familia a controlado después de Pretramitar dispara una nueva validación regulatoria? (transversal México y Perú). **

---

## Criterios de Aceptación

### Sección A — Determinación de aplicabilidad

**Criterio A1 — Detección de pedido con sustancias controladas**
- **Dado** que el ESAC opera un pedido en el módulo Pretramitar Pedido,
- **Cuando** el sistema evalúa el contenido del pedido,
- **Entonces** deberá identificar si contiene al menos una sustancia controlada clasificada como Mundial, Nacional u Origen para determinar si debe ejecutar la validación regulatoria.

**Criterio A2 — Validación no se ejecuta para pedidos sin controlados**
- **Dado** que el pedido no contiene sustancias controladas,
- **Cuando** el ESAC intenta avanzar el pedido al módulo Tramitar Pedido,
- **Entonces** el sistema no deberá ejecutar la validación regulatoria. El pedido avanza al flujo normal sin verificación de documentos regulatorios.

### Sección B — Ejecución de la validación y resultado

**Criterio B1 — Validación regulatoria automática al intentar avanzar**
- **Dado** que el pedido contiene al menos una sustancia controlada y el ESAC intenta avanzar el pedido al módulo Tramitar Pedido,
- **Cuando** se ejecuta la acción de avance,
- **Entonces** el sistema deberá consultar automáticamente el Catálogo de Clientes y verificar la presencia de los documentos regulatorios correspondientes a la Región del cliente del pedido (para México, Licencia Sanitaria y Aviso de Responsable Sanitario; para Perú, los equivalentes según DIGEMID) registrados en el Catálogo de Clientes.

**Criterio B2 — Validación solo sobre presencia del registro**
- **Dado** que el sistema ejecuta la validación regulatoria,
- **Cuando** consulta el Catálogo de Clientes,
- **Entonces** deberá considerar el documento como válido si está registrado, independientemente de su fecha de carga, fecha de vigencia o cualquier otro atributo. No valida vigencia, contenido del archivo ni firmas digitales en este release.

**Criterio B3 — Avance bloqueado cuando falta uno o ambos documentos**
- **Dado** que la validación detecta que al menos uno de los documentos requeridos no está registrado en el Catálogo de Clientes,
- **Cuando** el ESAC intenta avanzar el pedido,
- **Entonces** el sistema no deberá permitir el avance al módulo Tramitar Pedido. El pedido permanece en el módulo Pretramitar Pedido.

**Criterio B4 — Mensaje genérico al ESAC al bloquear el avance**
- **Dado** que el avance fue bloqueado por validación regulatoria fallida,
- **Cuando** se notifica al ESAC,
- **Entonces** el sistema deberá mostrar un mensaje genérico indicando que no es posible procesar el pedido porque el cliente no cuenta con la documentación regulatoria requerida para productos controlados, e invitándolo a revisar la sección de documentos regulatorios en la configuración del cliente. El mensaje no especifica cuál documento falta ni distingue por región.

**Criterio B5 — Avance permitido cuando los documentos requeridos están registrados**
- **Dado** que la validación confirma que los documentos regulatorios correspondientes a la Región del cliente están registrados en el Catálogo de Clientes,
- **Cuando** el ESAC ejecuta la acción de avance,
- **Entonces** el sistema deberá permitir el avance del pedido al módulo Tramitar Pedido sin interrupción ni mensaje adicional.

---

## Notas

- Este es el único cambio funcional que R16 introduce al módulo Pretramitar Pedido. El resto del flujo del módulo (validación de orden de compra, agregar partidas de productos, modificación de dirección de entrega, ajuste de fletes y demás operaciones del módulo) opera conforme al sistema preexistente en PQF2 sin modificaciones funcionales.
- Cubre el requisito del cliente sobre verificación previa a la tramitación de que el cliente cuente con los documentos regulatorios registrados, cuando el pedido contenga sustancias controladas.
- La validación es estrictamente sobre presencia del registro en el Catálogo de Clientes, no sobre vigencia ni contenido. La responsabilidad de mantener los documentos vigentes y correctos recae en el proceso de actualización del Catálogo de Clientes.
- El módulo donde se cargan o actualizan los documentos es Catálogo de Clientes, no Pretramitar Pedido. Si la validación bloquea un pedido, el ESAC debe coordinar con el área responsable del Catálogo de Clientes para registrar la documentación antes de poder avanzar el pedido.
- Aplicable a las operaciones de México y Perú; la validación regulatoria opera con la misma mecánica en ambas regiones, verificando en cada caso los documentos correspondientes a la Región del cliente.

> ** Pendiente: denominación exacta de los documentos regulatorios equivalentes para clientes con Región Perú según normativa DIGEMID. Es el mismo pendiente que en los requisitos TPSC-RE-FU-003 (carga de documentos regulatorios) y TPSC-RE-FU-007 (leyenda regulatoria en cotización). Debe resolverse de forma unificada para las tres filas. **

> ** Duda — La validación de documentos regulatorios se ejecuta únicamente al avanzar de Pretramitar Pedido a Tramitar Pedido. Existen otros caminos a Tramitar Pedido que no pasan por esa validación: (1) Gestionar Pedido Intramitable → OC corregida del cliente → Validar ajustes a la OC → avanza a Tramitar si resuelve, o se cicla a Intramitable si no; (2) "tramitar con errores" desde Gestionar Intramitable (avanzar pese a inconsistencias no resueltas); (3) Tramitar con OC Interna, donde desde Pretramitar el pedido se envía siempre a Gestionar Intramitable solo para solicitar al cliente la aceptación de la OC Interna, y al aceptarla avanza a Tramitar. Pendiente confirmar con el cliente cómo se comporta la validación regulatoria en cada punto de entrada: ¿se re-ejecuta antes de tramitar, o el pedido podría llegar a Tramitar sin validar los documentos regulatorios? Aplica a México y Perú. **
