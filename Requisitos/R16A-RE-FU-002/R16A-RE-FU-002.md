# Mantenimiento de Catálogo de Clientes

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-002 |
| **Nombre** | Mantenimiento de Catálogo de Clientes |
| **Categoría** | Catálogo de Clientes |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-009 |

---

## Historia de Usuario

> Yo como **Coordinador de Tesorería o Gerente de Tesorería**, quiero **asignar un Cobrador a cada cliente** en el Catálogo de Clientes, para distribuir la carga operativa de cobranza entre los Gestores de Cobranza del equipo y habilitar la **visibilidad filtrada por cartera** en los módulos operativos.

---

## Requisito

El sistema debe contar con dos campos de asignación en la sección Datos Generales del Catálogo de Clientes: **"Cobrador"** (usuario con rol Gestor de Cobranza) y **"Coordinador de Tesorería"** (usuario con rol Coordinador de Tesorería). La edición de ambos campos está habilitada para los roles **Coordinador de Tesorería** y **Gerente de Tesorería**; para cualquier otro rol los campos se muestran visibles pero bloqueados. La asignación del Cobrador es la base operativa para que los Gestores de Cobranza vean en su bandeja únicamente los pendientes y pagos de los clientes que tienen asignados. Para clientes de **Región México** aplica en los módulos **Validar Cobro**, **Factura por Adelantado**, **Buzón de Pagos** y **Notas de Crédito**. Para clientes de **Región Perú** aplica únicamente en los módulos **Validar Cobro** y **Buzón de Pagos**, al no existir Factura por Adelantado ni Notas de Crédito en esa región.

---

## Alcance

### Aplica a

- Clientes de México y Perú en el Catálogo de Clientes.
- Dos campos de asignación en la sección Datos Generales del cliente:
  - **"Cobrador"** — selector que muestra usuarios con rol Gestor de Cobranza.
  - **"Coordinador de Tesorería"** — selector que muestra usuarios con rol Coordinador de Tesorería.
- Edición de ambos campos habilitada para los roles Coordinador de Tesorería y Gerente de Tesorería.
- Bloqueo de ambos campos (no editables, solo visibles) para cualquier rol distinto de Coordinador de Tesorería y Gerente de Tesorería.
- Aplicación de la asignación de Cobrador para filtrar la visibilidad de pendientes y pagos por cartera:
  - **Región México:** módulos Validar Cobro, Factura por Adelantado, Buzón de Pagos y Notas de Crédito.
  - **Región Perú:** módulos Validar Cobro y Buzón de Pagos (Factura por Adelantado y Notas de Crédito no aplican en Perú).
- Reasignación del Cobrador de un cliente: todos los pendientes y pagos del cliente se redistribuyen inmediatamente a la bandeja del nuevo Cobrador.
- Restricción de vaciado de ambos campos (Cobrador y Coordinador de Tesorería): una vez asignados por primera vez, los campos no pueden quedar vacíos; solo admiten reasignación a otro usuario del rol correspondiente.

### No aplica a

- Asignación múltiple de usuarios al mismo cliente en cualquiera de los dos campos: un solo Cobrador y un solo Coordinador de Tesorería por cliente.
- Filtrado por cartera del Cobrador en los módulos Factura por Adelantado y Notas de Crédito para clientes de Región Perú (esos módulos no existen en Perú).
- Campo Cobrador en el alta de cliente desde Cotizar lo Cotizable: ese alta está orientada exclusivamente a habilitar la cotización, no a la gestión del cliente. La asignación de Cobrador se realiza posteriormente en el Catálogo de Clientes.
- Preservación de historial de asignaciones previas en R16.

---

## Reglas de Negocio

**Regla 1 — Asignación de Cobrador por Coordinador o Gerente de Tesorería**
La edición del campo Cobrador en el Catálogo de Clientes es responsabilidad de los roles Coordinador de Tesorería y Gerente de Tesorería. Para los demás roles el campo es visible en consulta pero no editable.

**Regla 2 — Cobrador debe tener rol Gestor de Cobranza**
El usuario asignable como Cobrador de un cliente debe ser un usuario activo del sistema con rol Gestor de Cobranza. Usuarios con otros roles o inactivos no son asignables.

**Regla 3 — Un solo Cobrador por cliente**
Cada cliente tiene asignado a lo más un Cobrador en un momento dado. La asignación de un Cobrador nuevo reemplaza al previamente asignado.

**Regla 4 — Filtrado dinámico por cobrador asignado actualmente**
La bandeja del Gestor de Cobranza en los módulos operativos refleja siempre los pendientes y pagos de los clientes que tiene asignados en ese momento. La reasignación de Cobrador de un cliente redistribuye inmediatamente sus pendientes y pagos a la bandeja del nuevo Cobrador; la bandeja del Cobrador anterior deja de mostrarlos.

**Regla 5 — Cliente sin Cobrador asignado**
Los pendientes y pagos de un cliente sin Cobrador asignado se registran en el sistema pero no aparecen en la bandeja de ningún Gestor de Cobranza hasta que el Coordinador de Tesorería complete la asignación. En el Buzón de Cobros, los correos de clientes sin Cobrador asignado se gestionan desde la bandeja del Coordinador de Tesorería (ver R16A-RE-FU-008, Reglas 11 y 12).

**Regla 6 — El campo Cobrador no admite valor vacío tras la primera asignación**
Una vez que a un cliente se le asigna un Cobrador por primera vez, el campo Cobrador del Catálogo de Clientes no puede quedar vacío: solo puede reemplazarse por otro Cobrador (reasignación), nunca dejarse en blanco. Esto evita que un cliente que ya operaba con cartera asignada quede sin Cobrador y sus pendientes se vuelvan invisibles.

---

## Riesgos

**Riesgo 1 — Clientes sin Cobrador asignado quedan invisibles operativamente**
Si el Coordinador de Tesorería no asigna oportunamente un Cobrador a un cliente nuevo, los pendientes y pagos de ese cliente no llegarán a ninguna bandeja y la operación de cobranza sobre ese cliente quedará detenida hasta que se realice la asignación.

---

## Criterios de Aceptación

### Sección A — Visibilidad y edición del campo Cobrador

**Criterio A1 — Visibilidad del campo Cobrador para todos los roles**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta un cliente específico,
- **Cuando** se renderiza la sección Datos Generales,
- **Entonces** el sistema deberá mostrar el campo "Cobrador" con el valor actualmente asignado al cliente (o vacío si no hay asignación previa), visible para todos los roles que tengan acceso al Catálogo de Clientes.

**Criterio A2 — Edición habilitada para Coordinador o Gerente de Tesorería**
- **Dado** que el usuario que consulta el Catálogo de Clientes tiene rol Coordinador de Tesorería o Gerente de Tesorería,
- **Cuando** visualiza el campo Cobrador de un cliente,
- **Entonces** el campo deberá presentarse como editable, permitiendo desplegar el selector y elegir un Gestor de Cobranza.

**Criterio A3 — Campo bloqueado para roles distintos de Coordinador o Gerente de Tesorería**
- **Dado** que el usuario que consulta el Catálogo de Clientes tiene cualquier rol distinto de Coordinador de Tesorería y Gerente de Tesorería,
- **Cuando** visualiza el campo Cobrador de un cliente,
- **Entonces** el campo deberá presentarse en modo bloqueado: visible para consulta pero no editable, sin permitir desplegar el selector ni modificar el valor.

### Sección B — Selector de Cobrador

**Criterio B1 — Selector limitado a Gestores de Cobranza activos**
- **Dado** que el Coordinador de Tesorería despliega el selector del campo Cobrador,
- **Cuando** el sistema arma la lista de opciones,
- **Entonces** deberá incluir únicamente usuarios activos del sistema con rol Gestor de Cobranza. Usuarios inactivos o con otros roles no deben aparecer en el selector.

**Criterio B2 — Persistencia de la asignación**
- **Dado** que el Coordinador de Tesorería selecciona un Gestor de Cobranza para un cliente y guarda los cambios,
- **Cuando** el sistema procesa la operación,
- **Entonces** deberá registrar la asignación en la base de datos. La asignación queda disponible inmediatamente para los módulos Validar Cobro, Factura por Adelantado, Buzón de Pagos y Notas de Crédito.

**Criterio B3 — Reemplazo del Cobrador previamente asignado**
- **Dado** que un cliente ya tiene un Cobrador asignado,
- **Cuando** el Coordinador de Tesorería asigna un Cobrador distinto al mismo cliente y guarda,
- **Entonces** el sistema deberá reemplazar el Cobrador anterior por el nuevo.

### Sección C — Filtrado de bandeja por Cobrador asignado

**Criterio C1 — Bandeja filtrada por cobrador actualmente asignado**
- **Dado** que un Gestor de Cobranza accede a los módulos operativos según la región del cliente (Región México: Validar Cobro, Factura por Adelantado, Buzón de Pagos o Notas de Crédito; Región Perú: Validar Cobro o Buzón de Pagos),
- **Cuando** el sistema renderiza la bandeja de pendientes y pagos,
- **Entonces** deberá mostrar únicamente los registros correspondientes a clientes que tengan al Gestor de Cobranza actualmente asignado en su Catálogo de Clientes.

**Criterio C2 — Redistribución inmediata al reasignar Cobrador**
- **Dado** que un cliente tiene pendientes y pagos en la bandeja del Cobrador A en los módulos aplicables según la región del cliente (Región México: Validar Cobro, Factura por Adelantado, Buzón de Pagos y Notas de Crédito; Región Perú: Validar Cobro y Buzón de Pagos),
- **Cuando** el Coordinador de Tesorería reasigna ese cliente al Cobrador B y guarda,
- **Entonces** el sistema deberá mostrar los pendientes y pagos del cliente en la bandeja del Cobrador B en la siguiente consulta de la bandeja, y dejar de mostrarlos en la bandeja del Cobrador A. La reasignación aplica de manera dinámica sobre todos los pendientes y pagos vigentes del cliente, incluidos los que aún no han sido trabajados: todos pasan a la bandeja del nuevo Cobrador, sin distinción entre los previos a la reasignación y los nuevos.

**Criterio C3 — Cliente sin Cobrador asignado**
- **Dado** que un cliente no tiene un Cobrador asignado en su Catálogo,
- **Cuando** se generan pendientes o pagos asociados al cliente,
- **Entonces** el sistema deberá registrar los pendientes y pagos pero no mostrarlos en la bandeja de ningún Cobrador, hasta que el Coordinador de Tesorería complete la asignación.

**Criterio C4 — Trazabilidad del trabajo realizado por el Cobrador anterior**
- **Dado** que un cliente es reasignado de un Cobrador a otro,
- **Cuando** se realiza la reasignación,
- **Entonces** el sistema deberá conservar en el histórico el registro del trabajo ya realizado por el Cobrador anterior (pendientes y pagos gestionados antes del reemplazo), de modo que la reasignación no borre esa información. El cambio de visibilidad en la bandeja es dinámico, pero el historial de lo trabajado por cada Cobrador se preserva para trazabilidad.

---

## Notas de Implementación

- Funcionalidad ubicada en la sección **Datos Generales** del cliente dentro del Catálogo de Clientes.
- El campo se llama exactamente **"Cobrador"**.
- Roles relevantes para esta funcionalidad: **Coordinador de Tesorería** y **Gerente de Tesorería** (ambos pueden editar el campo Cobrador); **Gestor de Cobranza** (es el rol que aparece como opción en el selector y opera bandejas filtradas por cartera).
- La administración de roles del sistema (creación de roles, asignación usuario-rol) no tiene UI dedicada en R16. Los roles y las asignaciones usuario-rol se gestionan a nivel base de datos.
- La asignación cliente-Cobrador se utiliza posteriormente para filtrar la visibilidad de pendientes y pagos por cobrador asignado según la región del cliente: **Región México** — módulos Validar Cobro, Factura por Adelantado, Buzón de Pagos y Notas de Crédito; **Región Perú** — módulos Validar Cobro y Buzón de Pagos (Factura por Adelantado y Notas de Crédito no aplican en Perú).
- **Comportamiento del filtro de bandeja: dinámico.** Cuando el Coordinador o el Gerente de Tesorería reasigna el Cobrador de un cliente, todos los pendientes y pagos vigentes del cliente (incluidos los que aún no han sido trabajados) se reflejan inmediatamente en la bandeja del nuevo Cobrador y dejan de aparecer en la del Cobrador anterior. No hay distinción entre pendientes previos y nuevos respecto al filtro de visibilidad. Aparte del filtro dinámico de la bandeja, el sistema conserva en el histórico el registro del trabajo ya realizado por el Cobrador anterior, de modo que la reasignación no borra esa trazabilidad.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-07-27 | Alcance Perú | Se precisa que el filtrado por cartera para clientes de Región Perú opera únicamente en Validar Cobro y Buzón de Pagos (no aplica FAA ni NC). Se agrega el punto en Alcance "Aplica a" con los módulos aplicables por región. Se agrega exclusión en "No aplica a" del filtrado por cartera en FAA y NC para Perú. Se actualizan los Criterios C1 y C2 con la precisión por región. Se ajusta el bullet de módulos consumidores en Observaciones. |
| 2 | 2026-07-27 | Corrección de consistencia interna del Alcance | Se sustituyen los bullets que describían un campo único "Cobrador" por la descripción de los dos campos de asignación (Cobrador y Coordinador de Tesorería), con sus selectores y sus permisos de edición y bloqueo. La restricción de vaciado se extiende a ambos campos. La exclusión de asignación múltiple en "No aplica a" se extiende a ambos campos. El Alcance no se había actualizado al incorporarse el campo Coordinador de Tesorería, que ya estaba contemplado en el Requisito, las Reglas 7 a 9 y los Criterios D1 a D5. |
