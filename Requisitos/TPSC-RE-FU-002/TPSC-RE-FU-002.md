# Mantenimiento de Catálogo de Clientes

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-002 |
| **Nombre** | Mantenimiento de Catálogo de Clientes |
| **Categoría** | Catálogo de Clientes |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.1M-RE-FU-009 |

---

## Historia de Usuario

> Yo como **Coordinador de Tesorería**, quiero **asignar un Cobrador a cada cliente** en el Catálogo de Clientes, para distribuir la carga operativa de cobranza entre los Gestores de Cobranza del equipo y habilitar la **visibilidad filtrada por cartera** en los módulos operativos.

---

## Requisito

El sistema debe contar con un campo **"Cobrador"** en la sección Datos Generales del Catálogo de Clientes que permita asignar a cada cliente un usuario con rol Gestor de Cobranza. La edición del campo es exclusiva del rol Coordinador de Tesorería; para cualquier otro rol el campo se muestra visible pero bloqueado. La asignación es la base operativa para que los Gestores de Cobranza vean en su bandeja únicamente los pendientes y pagos de los clientes que tienen asignados en los módulos **Validar Cobro**, **Factura por Adelantado** y **Buzón de Pagos**.

---

## Alcance

### Aplica a

- Clientes de México y Perú en el Catálogo de Clientes.
- Campo único "Cobrador" en la sección Datos Generales del cliente.
- Edición del campo exclusiva por el rol Coordinador de Tesorería.
- Selector que muestra usuarios con rol Gestor de Cobranza.
- Bloqueo del campo (no editable, solo visible) para cualquier rol distinto de Coordinador de Tesorería.
- Aplicación de la asignación en los módulos Validar Cobro, Factura por Adelantado y Buzón de Pagos para filtrar la visibilidad de pendientes y pagos por cobrador asignado.
- Reasignación del Cobrador de un cliente: todos los pendientes y pagos del cliente se redistribuyen inmediatamente a la bandeja del nuevo Cobrador.

### No aplica a

- Asignación múltiple de Cobradores al mismo cliente (un solo Gestor de Cobranza por cliente).
- Campo Cobrador en el alta de cliente desde Cotizar lo Cotizable: ese alta está orientada exclusivamente a habilitar la cotización, no a la gestión del cliente. La asignación de Cobrador se realiza posteriormente en el Catálogo de Clientes.
- Preservación de historial de asignaciones previas en R16.

---

## Reglas de Negocio

**Regla 1 — Asignación de Cobrador exclusiva del Coordinador de Tesorería**
La edición del campo Cobrador en el Catálogo de Clientes es responsabilidad y derecho exclusivo del rol Coordinador de Tesorería. Para los demás roles el campo es visible en consulta pero no editable.

**Regla 2 — Cobrador debe tener rol Gestor de Cobranza**
El usuario asignable como Cobrador de un cliente debe ser un usuario activo del sistema con rol Gestor de Cobranza. Usuarios con otros roles o inactivos no son asignables.

**Regla 3 — Un solo Cobrador por cliente**
Cada cliente tiene asignado a lo más un Cobrador en un momento dado. La asignación de un Cobrador nuevo reemplaza al previamente asignado.

**Regla 4 — Filtrado dinámico por cobrador asignado actualmente**
La bandeja del Gestor de Cobranza en los módulos operativos refleja siempre los pendientes y pagos de los clientes que tiene asignados en ese momento. La reasignación de Cobrador de un cliente redistribuye inmediatamente sus pendientes y pagos a la bandeja del nuevo Cobrador; la bandeja del Cobrador anterior deja de mostrarlos.

**Regla 5 — Cliente sin Cobrador asignado**
Los pendientes y pagos de un cliente sin Cobrador asignado se registran en el sistema pero no aparecen en la bandeja de ningún Gestor de Cobranza hasta que el Coordinador de Tesorería complete la asignación.

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

**Criterio A2 — Edición habilitada solo para Coordinador de Tesorería**
- **Dado** que el usuario que consulta el Catálogo de Clientes tiene rol Coordinador de Tesorería,
- **Cuando** visualiza el campo Cobrador de un cliente,
- **Entonces** el campo deberá presentarse como editable, permitiendo desplegar el selector y elegir un Gestor de Cobranza.

**Criterio A3 — Campo bloqueado para roles distintos de Coordinador de Tesorería**
- **Dado** que el usuario que consulta el Catálogo de Clientes tiene cualquier rol distinto de Coordinador de Tesorería,
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
- **Entonces** deberá registrar la asignación en la base de datos. La asignación queda disponible inmediatamente para los módulos Validar Cobro, Factura por Adelantado y Buzón de Pagos.

**Criterio B3 — Reemplazo del Cobrador previamente asignado**
- **Dado** que un cliente ya tiene un Cobrador asignado,
- **Cuando** el Coordinador de Tesorería asigna un Cobrador distinto al mismo cliente y guarda,
- **Entonces** el sistema deberá reemplazar el Cobrador anterior por el nuevo.

### Sección C — Filtrado de bandeja por Cobrador asignado

**Criterio C1 — Bandeja filtrada por cobrador actualmente asignado**
- **Dado** que un Gestor de Cobranza accede a los módulos Validar Cobro, Factura por Adelantado o Buzón de Pagos,
- **Cuando** el sistema renderiza la bandeja de pendientes y pagos,
- **Entonces** deberá mostrar únicamente los registros correspondientes a clientes que tengan al Gestor de Cobranza actualmente asignado en su Catálogo de Clientes.

**Criterio C2 — Redistribución inmediata al reasignar Cobrador**
- **Dado** que un cliente tiene pendientes y pagos en la bandeja del Cobrador A,
- **Cuando** el Coordinador de Tesorería reasigna ese cliente al Cobrador B y guarda,
- **Entonces** el sistema deberá mostrar los pendientes y pagos del cliente en la bandeja del Cobrador B en la siguiente consulta de la bandeja, y dejar de mostrarlos en la bandeja del Cobrador A. La reasignación aplica de manera dinámica sobre todos los pendientes y pagos vigentes del cliente, sin distinción entre los previos a la reasignación y los nuevos.

**Criterio C3 — Cliente sin Cobrador asignado**
- **Dado** que un cliente no tiene un Cobrador asignado en su Catálogo,
- **Cuando** se generan pendientes o pagos asociados al cliente,
- **Entonces** el sistema deberá registrar los pendientes y pagos pero no mostrarlos en la bandeja de ningún Cobrador, hasta que el Coordinador de Tesorería complete la asignación.

---

## Notas de Implementación

- Funcionalidad ubicada en la sección **Datos Generales** del cliente dentro del Catálogo de Clientes.
- El campo se llama exactamente **"Cobrador"**.
- Roles relevantes para esta funcionalidad:
  - **Coordinador de Tesorería**: puede editar el campo Cobrador.
  - **Gestor de Cobranza**: es el rol que aparece como opción en el selector y opera bandejas filtradas por cartera.
- La administración de roles del sistema (creación de roles, asignación usuario-rol) no tiene UI dedicada en R16. Los roles y las asignaciones usuario-rol se gestionan a nivel base de datos.
- La asignación cliente-Cobrador se utiliza posteriormente en los módulos Validar Cobro, Factura por Adelantado y Buzón de Pagos para filtrar la visibilidad de pendientes y pagos por cobrador asignado. Esa funcionalidad de filtrado se documenta en los requisitos correspondientes a esos módulos.
- **Comportamiento del filtro de bandeja: dinámico.** Cuando el Coordinador de Tesorería reasigna el Cobrador de un cliente, todos los pendientes y pagos del cliente (vigentes en el momento del cambio) se reflejan inmediatamente en la bandeja del nuevo Cobrador y dejan de aparecer en la bandeja del Cobrador anterior. No hay distinción entre pendientes previos y nuevos respecto al filtro.
