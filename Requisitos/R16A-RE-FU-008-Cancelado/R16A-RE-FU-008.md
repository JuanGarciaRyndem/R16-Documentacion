# Buzón de Cobros

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-008 |
| **Nombre** | Buzón de Cobros |
| **Módulo** | Buzones |
| **Categoría** | Funcional |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.2M-RE-FU-004 |

---

## Historia de Usuario

> Yo como Gestor de Cobranza, quiero contar con un Buzón de Cobros que reciba automáticamente los correos clasificados como cobros enviados por mis clientes asignados, genere un pendiente en Validar Cobro sin captura previa de datos y me permita reclasificar correos mal clasificados, para reducir el trabajo manual de identificación inicial de cobros.

---

## Requisito

El sistema debe contar con un módulo **Buzón de Cobros** nuevo en PQF2 que reciba automáticamente los correos clasificados como cobros por el Mailbot, sumando la clasificación “Cobro” a las clasificaciones existentes del sistema. Cada correo clasificado como cobro se refleja en el Buzón del Gestor de Cobranza correspondiente y dispara la generación automática de un pendiente en el módulo **Validar Cobro**. La visibilidad del Buzón es por cobrador asignado: cada Gestor de Cobranza ve únicamente los correos de los clientes que tiene asignados en su cartera. El Gestor de Cobranza puede reclasificar manualmente un correo cuando detecta que fue clasificado incorrectamente. Adicionalmente, el Buzón de Cobros cuenta con una **bandeja del Coordinador de Tesorería** que concentra los correos de cobro que no pueden enrutarse a un Gestor: los de clientes existentes sin Cobrador asignado y los de correos cuyo remitente no está dado de alta como contacto de ningún cliente. Desde esta bandeja, según el caso, se asigna el Cobrador —lo que mueve el correo y todos los pendientes previos del cliente a la bandeja de ese Cobrador (retroactividad)— o se canaliza el alta del contacto por el flujo operativo existente.

---

## Alcance

### Aplica a

- Módulo en PQF2: **Buzón de Cobros**.
- Recepción de correos clasificados automáticamente como cobros por el Mailbot del sistema.
- Visibilidad de correos en el Buzón filtrada por la asignación del cobrador (cada Gestor de Cobranza ve los correos de los clientes que tiene asignados).
- Generación automática de un pendiente en Validar Cobro al clasificarse cada correo como cobro, sin captura previa de datos.
- Aplicación del mismo patrón de filtros, búsqueda y paginación que los Buzones preexistentes (Buzón de Requisiciones, Buzón de Pedidos).
- Cierre automático del pendiente del Buzón cuando el cobro se vincula a una proforma o factura en Validar Cobro.
- Eliminación automática del pendiente del Buzón cuando el cobro se marca como inconsistencia en Validar Cobro.
- Acción manual de reclasificación: el Gestor de Cobranza puede mover un correo a otro buzón si la clasificación automática fue incorrecta.
- **Bandeja del Coordinador de Tesorería** dentro del Buzón de Cobros: concentra los correos que no pueden enrutarse a un Gestor (cliente existente sin Cobrador asignado, o remitente no dado de alta como contacto), con las acciones de asignar Cobrador o canalizar el alta del contacto según el caso.
- Aplicación a clientes de México y Perú.

### No aplica a

- Captura de datos del cobro (monto, cliente, banco emisor, cuenta origen, fecha del depósito, referencia bancaria). Esa captura ocurre la primera vez que se trabaja el pendiente en el módulo Validar Cobro.
- Eliminación directa de correos por parte del Gestor de Cobranza desde el Buzón de Cobros. La salida del correo del Buzón ocurre por los eventos del ciclo de vida (vinculación exitosa o inconsistencia en Validar Cobro) o por reclasificación manual hacia otro buzón.
- Definición del envío de notificación al cliente ante una inconsistencia. Ese comportamiento pertenece al módulo Validar Cobro (requisito independiente); en este requisito solo se contempla que el pendiente del Buzón se elimina cuando el cobro se marca como inconsistencia.
- Criterios configurables de clasificación por parte del usuario. El Mailbot se entrena con una base de conocimiento; los ajustes a la clasificación se realizan mediante un nuevo entrenamiento, no mediante parámetros configurables en la interfaz.

---

## Reglas de Negocio

**Regla 1 — Clasificación automática del correo por el Mailbot**
El Mailbot del sistema clasifica automáticamente cada correo entrante en una de las categorías del modelo de clasificación: Cotización, Pedido, Cobro u Otros. R16 agrega la categoría **Cobro** al modelo existente; no modifica el resto del modelo. Solo los correos clasificados como Cobro entran al Buzón de Cobros.

**Regla 2 — Clasificación basada en entrenamiento, sin criterios configurables**
La clasificación del Mailbot se basa en una base de conocimiento entrenada. Los ajustes a la precisión de la clasificación se realizan mediante un nuevo entrenamiento del Mailbot, no mediante criterios configurables por el usuario en la interfaz.

**Regla 3 — Reflejo del correo clasificado en el Buzón**
Un correo clasificado como Cobro se refleja automáticamente en el Buzón de Cobros del Gestor de Cobranza que tenga asignado al cliente identificado en el correo. La visibilidad del correo en el Buzón es exclusiva del Gestor asignado al cliente.

**Regla 4 — Generación automática de pendiente en Validar Cobro sin captura previa**
Un correo clasificado como Cobro genera simultánea y automáticamente un pendiente en el módulo Validar Cobro asociado al correo, sin capturar previamente datos del cobro. La captura del monto y demás datos del cobro ocurre la primera vez que se trabaja el pendiente en Validar Cobro.

**Regla 5 — Visibilidad por cobrador asignado y por región del cliente**
La bandeja del Buzón de un Gestor de Cobranza muestra únicamente los correos asociados a clientes que tiene asignados en su cartera (campo Cobrador del Catálogo de Clientes). Los correos de clientes asignados a otros Gestores no aparecen en su Buzón. Adicionalmente, cada correo se refleja siempre en el contexto de la región del cliente que lo originó: un cliente de México no aparece en el contexto de cobros de Perú y viceversa.

**Regla 6 — Cierre automático del pendiente del Buzón al vincular el cobro a proforma o factura**
Cuando un cobro en Validar Cobro se vincula exitosamente a una proforma o factura, el sistema cierra y retira automáticamente el pendiente correspondiente del Buzón de Cobros. El correo deja de aparecer en la bandeja del Gestor; el ciclo de vida del pendiente del Buzón se considera completado.

**Regla 7 — Eliminación automática del Buzón ante inconsistencia en Validar Cobro**
Cuando un cobro en Validar Cobro se marca como inconsistencia, el sistema elimina automáticamente la entrada correspondiente del Buzón de Cobros. El tratamiento posterior de la inconsistencia (incluida cualquier notificación al cliente) pertenece al módulo Validar Cobro.

**Regla 8 — Reclasificación manual hacia otro buzón**
Cuando un Gestor de Cobranza identifica que un correo fue clasificado incorrectamente como Cobro, puede ejecutar la acción de reclasificarlo moviéndolo a otro buzón del sistema, incluido el buzón de Otros. No existe la opción “marcar como no-cobro”; cualquier corrección de clasificación se realiza moviendo el correo a un buzón destino.

**Regla 9 — Sin eliminación directa por el Gestor; eliminación reservada a ESAC desde Otros**
El Gestor de Cobranza no dispone de una acción de eliminación directa del correo en el Buzón de Cobros. Si un correo no corresponde a un cobro, la salida es reclasificarlo al buzón de Otros; la eliminación del correo, de requerirse, la realiza el rol ESAC desde el buzón de Otros.

**Regla 10 — Filtros, búsqueda y paginación equivalentes a Buzones preexistentes**
El Buzón de Cobros ofrece los mismos mecanismos de filtros, búsqueda y paginación que los Buzones preexistentes del sistema, conservando consistencia de experiencia de usuario.

**Regla 11 — Bandeja del Coordinador: correo de cliente existente sin Cobrador (Caso 1)**
Cuando un correo de cobro proviene de un remitente dado de alta como contacto de un cliente existente, pero ese cliente no tiene Cobrador asignado, el correo se concentra en la bandeja del Coordinador de Tesorería dentro del Buzón de Cobros. Desde esa bandeja, el Coordinador de Tesorería puede asignar un Cobrador al cliente; al hacerlo, el correo desaparece de la bandeja del Coordinador y aparece en la bandeja del Cobrador asignado. La asignación también puede realizarse desde el Catálogo de Clientes, con el mismo efecto. Al asignarse el Cobrador, la bandeja de ese Cobrador muestra retroactivamente todos los correos y pendientes del cliente generados mientras no tenía Cobrador asignado.

**Regla 12 — Bandeja del Coordinador: correo de remitente no dado de alta (Caso 2)**
Cuando un correo de cobro proviene de un remitente que no está dado de alta como contacto de ningún cliente existente, el correo se concentra en la bandeja del Coordinador de Tesorería. Este caso no se resuelve desde el Buzón: el flujo operativo es el existente, dar de alta el contacto (con ese correo) en el cliente correspondiente. Una vez dado de alta el contacto, si el cliente ya tiene Cobrador asignado el correo pasa a la bandeja de ese Cobrador; si el cliente no tiene Cobrador asignado, el correo permanece en la bandeja del Coordinador y se resuelve conforme a la Regla 11.

---

## Riesgos

> El caso de cliente sin Cobrador asignado (riesgo anterior) queda cubierto por la **bandeja del Coordinador de Tesorería** (Reglas 11 y 12, SECCIÓN F de Criterios). Ver OBS-021.

---

## Criterios de Aceptación

### SECCIÓN A — Clasificación y Recepción

**Criterio A1 — Recepción y clasificación automática de correo entrante**
- **Dado** que un correo entrante llega al sistema,
- **Cuando** el Mailbot lo evalúa,
- **Entonces** deberá clasificarlo en una de las categorías del modelo (Cotización, Pedido, Cobro u Otros) y, si lo clasifica como Cobro, enviarlo al Buzón de Cobros.

**Criterio A2 — Clasificación sin criterios configurables por el usuario**
- **Dado** que se requiere ajustar la precisión de la clasificación del Mailbot,
- **Cuando** se necesita modificar su comportamiento,
- **Entonces** el ajuste deberá realizarse mediante un nuevo entrenamiento del Mailbot con su base de conocimiento, sin que existan criterios configurables por el usuario en la interfaz del Buzón.

---

### SECCIÓN B — Reflejo en el Buzón y Generación de Pendiente

**Criterio B1 — Reflejo del correo en el Buzón del Gestor asignado**
- **Dado** que un correo es clasificado como Cobro y el cliente identificado en el correo tiene un Gestor de Cobranza asignado en el Catálogo de Clientes,
- **Cuando** el sistema procesa el reflejo en el Buzón,
- **Entonces** deberá hacer visible el correo únicamente en la bandeja del Buzón del Gestor asignado. Otros Gestores no ven el correo en su bandeja.

**Criterio B2 — Generación automática de pendiente en Validar Cobro**
- **Dado** que un correo es clasificado como Cobro,
- **Cuando** el sistema procesa la clasificación,
- **Entonces** deberá generar automáticamente un pendiente en el módulo Validar Cobro asociado al correo, sin pre-capturar datos del cobro. El pendiente queda disponible para que el Gestor lo trabaje en Validar Cobro.

**Criterio B3 — Visibilidad filtrada por cobrador asignado**
- **Dado** que un Gestor de Cobranza accede al Buzón,
- **Cuando** el sistema arma la bandeja,
- **Entonces** deberá filtrar los correos para mostrar únicamente aquellos correspondientes a clientes que el Gestor tiene asignados en el Catálogo de Clientes (campo Cobrador).

---

### SECCIÓN C — Ciclo de Vida del Pendiente del Buzón

**Criterio C1 — Cierre automático del pendiente del Buzón al vincular el cobro a proforma o factura**
- **Dado** que el Gestor de Cobranza vincula exitosamente el cobro a una proforma o factura desde el módulo Validar Cobro,
- **Cuando** se completa la vinculación,
- **Entonces** el sistema deberá retirar automáticamente el pendiente correspondiente del Buzón de Cobros, sin requerir acción manual adicional del Gestor. El correo deja de aparecer en su bandeja.

**Criterio C2 — Eliminación automática del Buzón ante inconsistencia en Validar Cobro**
- **Dado** que un cobro en Validar Cobro se marca como inconsistencia y el correo origen estaba en el Buzón,
- **Cuando** se procesa la inconsistencia en Validar Cobro,
- **Entonces** el sistema deberá eliminar automáticamente la entrada correspondiente del Buzón de Cobros. El tratamiento posterior de la inconsistencia, incluida cualquier notificación al cliente, pertenece al módulo Validar Cobro.

---

### SECCIÓN D — Acciones del Gestor y Experiencia de Usuario

**Criterio D1 — Acción de reclasificación manual**
- **Dado** que un Gestor de Cobranza identifica un correo del Buzón que fue clasificado incorrectamente,
- **Cuando** ejecuta la acción de reclasificar,
- **Entonces** el sistema deberá ofrecer la opción de mover el correo a otro buzón del sistema (incluido el buzón de Otros). La acción se completa al elegir el buzón destino: el correo se retira del Buzón de Cobros y se refleja en el buzón seleccionado.

**Criterio D2 — Sin eliminación directa por el Gestor; eliminación reservada a ESAC desde Otros**
- **Dado** que un Gestor de Cobranza opera el Buzón de Cobros,
- **Cuando** consulta las acciones disponibles sobre un correo,
- **Entonces** el sistema no deberá ofrecer una acción de eliminación individual y/o múltiple del correo. Si el correo no corresponde a un cobro u otra clasificación, la salida es reclasificarlo al buzón de Otros; la eliminación del correo, de requerirse, la realiza el rol ESAC desde el buzón de Otros.

**Criterio D3 — Filtros, búsqueda y paginación en el Buzón**
- **Dado** que un Gestor de Cobranza opera el Buzón,
- **Cuando** consulta su bandeja,
- **Entonces** el módulo deberá ofrecer las funcionalidades de filtros, búsqueda y paginación equivalentes al patrón aplicado en los Buzones preexistentes del sistema (Buzón de Requisiciones y Buzón de Pedidos).

---

### SECCIÓN E — Aplicabilidad Regional

**Criterio E1 — Aplicación a clientes México y Perú con segregación por región**
- **Dado** que el cliente del correo identificado tiene Región México o Región Perú,
- **Cuando** el sistema procesa la clasificación y refleja el correo en el Buzón,
- **Entonces** deberá operar con la misma mecánica para ambas regiones, respetando siempre la región del cliente: el correo de un cliente de México se refleja en el contexto de cobros de México y el de un cliente de Perú en el contexto de cobros de Perú. Un cliente de una región nunca aparece en el contexto de cobros de la otra; no hay cruce de correos entre regiones.

---

### SECCIÓN F — Bandeja del Coordinador

**Criterio F1 — Concentración de correos no enrutables en la bandeja del Coordinador**
- **Dado** que un correo de cobro no puede enrutarse a un Gestor (cliente existente sin Cobrador asignado, o remitente no dado de alta como contacto),
- **Cuando** el sistema procesa el correo,
- **Entonces** deberá mostrarlo en la bandeja del Coordinador de Tesorería dentro del Buzón de Cobros, en lugar de dejarlo invisible operativamente.

**Criterio F2 — Asignación de Cobrador desde la bandeja del Coordinador (Caso 1)**
- **Dado** que un correo de la bandeja del Coordinador corresponde a un cliente existente sin Cobrador asignado,
- **Cuando** el Coordinador de Tesorería asigna un Cobrador (desde la bandeja o desde el Catálogo de Clientes),
- **Entonces** el sistema deberá retirar el correo de la bandeja del Coordinador y reflejar en la bandeja del Cobrador asignado no solo ese correo, sino todos los correos y pendientes del cliente que se hayan generado previamente mientras no tenía Cobrador asignado (retroactividad completa).

**Criterio F3 — Correo de remitente no dado de alta (Caso 2)**
- **Dado** que un correo de la bandeja del Coordinador proviene de un remitente no dado de alta como contacto de ningún cliente,
- **Cuando** el usuario da de alta el contacto en el cliente correspondiente por el flujo operativo existente,
- **Entonces** el sistema deberá enrutar el correo a la bandeja del Cobrador asignado si el cliente ya tiene Cobrador, o mantenerlo en la bandeja del Coordinador para asignación si el cliente aún no tiene Cobrador.

**Criterio F4 — Visibilidad retroactiva de pendientes al asignar Cobrador**
- **Dado** que a un cliente que no tenía Cobrador asignado se le asigna uno (desde la bandeja del Coordinador o desde el Catálogo de Clientes),
- **Cuando** se completa la asignación,
- **Entonces** la bandeja del Cobrador asignado deberá mostrar todos los correos pendientes de ese cliente generados antes de la asignación, sin que ninguno quede excluido por haberse generado previamente.

---

## Notas de Implementación

- El módulo se llama **“Buzón de Cobros”**. El nombre refleja la perspectiva operativa: son cobros que PROQUIFA registra hacia el cliente, aunque el correo original sea el comprobante del pago que el cliente envía a PROQUIFA. Toda la terminología del módulo, su contenido y sus mensajes utiliza el término “cobro”.
- El **Buzón de Cobros** y el módulo **Validar Cobro** NO son el mismo módulo. El mismo correo es visible en ambos pero con funciones distintas:
  - En el **Buzón de Cobros**: se ve el correo clasificado, se opera la reclasificación si la clasificación automática fue incorrecta, y se observa cuándo el pendiente se cierra o elimina automáticamente.
  - En **Validar Cobro**: se capturan los datos del cobro, se vincula a proformas/facturas y se marca como inconsistencia si aplica.
- El ciclo de vida del pendiente del correo en el Buzón es estrictamente automático: aparece al clasificarse el correo, se cierra automáticamente al completarse la vinculación en Validar Cobro, o se elimina automáticamente al marcarse inconsistencia. No existe eliminación directa del correo por parte del Gestor; la única acción manual disponible en el Buzón es la reclasificación.
- La clasificación la realiza el Mailbot, que en R16 agrega la categoría Cobro a su modelo existente (Cotización, Pedido, Otros). El entrenamiento inicial requiere un conjunto representativo de correos reales de PROQUIFA en producción para ajustar la tasa de acierto.
- La visibilidad de los correos está filtrada por el cobrador asignado al cliente (campo Cobrador del Catálogo de Clientes). Cada Gestor de Cobranza ve únicamente los correos correspondientes a su cartera.
- Los filtros, búsqueda y paginación aplican el mismo patrón que los Buzones preexistentes (Buzón de Requisiciones, Buzón de Pedidos), conservando consistencia de UX.
- La reclasificación manual se realiza moviendo el correo a otro buzón del sistema, incluido el buzón de Otros. No existe la opción “marcar como no-cobro”.
- Aplicable a clientes México y Perú con la misma mecánica funcional, respetando siempre la región del cliente.

> **⚠️ Pendiente** — Confirmar con el cliente la lógica para correos clasificados como cobro cuyo cliente no es identificable o no está registrado (cliente nuevo no registrado, referencia errónea, correo desde dominio genérico). ¿Quién lo visualiza y atiende? Queda como duda formal del proyecto.

> **⚠️ Pendiente confirmar con el cliente** — Para los correos de la bandeja del Coordinador (Caso 1): (1) si se permite reclasificar el correo a otro tipo de pendiente desde esa bandeja, y (2) en caso afirmativo, a qué destinos se permite reclasificar (hipótesis operativa: únicamente a "Otros").

> **⚠️ Propuesta abierta** — Incorporar IA al clasificador para que lea el documento adjunto del correo (comprobante de pago en PDF o imagen) y precargue automáticamente los datos en Validar Cobro (monto, banco emisor, cuenta origen, fecha del depósito, referencia bancaria, etc.). Esto reduciría el trabajo manual del Gestor de Cobranza al validar el cobro. Queda como evolución futura del módulo, sujeta a validación de viabilidad y costo con el cliente.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-06-10 | OBS-021 | Bandeja del Coordinador de Tesorería formalizada como requisito. Riesgo 1 (cliente sin Cobrador) resuelto por la mecánica de la bandeja. Cambios: Requisito ampliado con descripción de la bandeja; Alcance "Aplica a" con nueva viñeta; Regla 11 y Regla 12 agregadas; Riesgos reemplazados por nota de resolución; SECCIÓN F — Criterios F1–F4 (bandeja del Coordinador y retroactividad al asignar Cobrador) agregada. |
