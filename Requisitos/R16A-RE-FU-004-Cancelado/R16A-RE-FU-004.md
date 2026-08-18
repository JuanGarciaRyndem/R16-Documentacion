# Mantenimiento de Catálogo de Clientes — Actualización de catálogos Régimen Fiscal y Tipo de Sociedad Mercantil

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-004 |
| **Nombre** | Actualización de catálogos Régimen Fiscal y Tipo de Sociedad Mercantil |
| **Catálogo** | Catálogo de Clientes |
| **Categoría** | Funcional |
| **Estatus** | Propuesto |
| **Referencia Legacy** | Sin trazabilidad directa a la matriz original del cliente; emergente de sesiones |

---

## Historia de Usuario

> Yo como **usuario con acceso a la cartera de clientes**, quiero que los catálogos de opciones de **Régimen Fiscal** y **Tipo de Sociedad Mercantil** estén actualizados en el sistema, para que al capturar o consultar la información fiscal del cliente los selectores muestren la lista vigente confirmada por el cliente, sin diferenciación por región.

---

## Requisito

El sistema debe **actualizar los catálogos de opciones** de los campos **Régimen Fiscal** y **Tipo de Sociedad Mercantil** utilizados en la sección **Entrega y Facturación** del Catálogo de Clientes. La actualización comprende **altas**, **bajas** y **modificaciones** de opciones, ejecutadas directamente en base de datos con base en el listado consolidado en el documento *R16 - Catálogos Fiscales*. El sistema únicamente despliega el listado resultante en los selectores correspondientes.

**Formato de despliegue:**
- **Régimen Fiscal:** clave y descripción de la opción, separadas por guion (ej. `601 - General de Ley Personas Morales`).
- **Tipo de Sociedad Mercantil:** descripción de la opción sin guion ni clave adicional.

Los catálogos se despliegan **sin diferenciación por región** — la misma lista aplica a todos los clientes, independientemente de la Región. La captura y validación del identificador fiscal del cliente, las etiquetas del campo, los bloqueos de guardado y la persistencia son funcionalidad **ya existente** en el sistema y **no se modifican** en este release.

---

## Alcance

### Aplica a

- **Actualización del catálogo de Régimen Fiscal** utilizado en el campo Régimen Fiscal de la sección Entrega y Facturación del Catálogo de Clientes.
- **Actualización del catálogo de Tipo de Sociedad Mercantil** utilizado en el campo Tipo de Sociedad Mercantil de la misma sección.
- **Tipos de actualización:** altas, bajas y modificaciones de opciones ejecutadas directamente en base de datos.
- **Formato de despliegue:**
  - Régimen Fiscal: `Clave - Descripción` (con guion como separador).
  - Tipo de Sociedad Mercantil: descripción de la opción (sin guion ni clave adicional).
- **Despliegue único sin diferenciación por región** — la misma lista se muestra a todos los clientes.
- **Origen de la actualización:** documento *R16 - Catálogos Fiscales* consolidado por el cliente.

### No aplica a

- **Regionalización de los campos ni de sus catálogos:** los catálogos se despliegan igual para todos los clientes independientemente de la Región.
- **Captura y validación del identificador fiscal del cliente (RFC/RUC):** funcionalidad ya existente en el sistema, no se modifica en este release.
- **Etiquetas dinámicas por región de los campos.**
- **Bloqueos de guardado y persistencia** de la información fiscal: funcionalidad ya existente.
- **Obligatoriedad y validaciones de formato** del identificador fiscal: funcionalidad ya existente.
- **Interfaz gráfica para administrar los catálogos** desde la aplicación: la gestión se ejecuta directamente en base de datos.
- **Tratamiento de clientes con valores dados de baja del catálogo:** el cliente entrega el listado depurado y ejecuta la curaduría (ver Regla 7 y Riesgo 2).

---

## Reglas de Negocio

**Regla 1 — Alcance de la actualización**
La actualización aplica exclusivamente a los catálogos de **Régimen Fiscal** y **Tipo de Sociedad Mercantil** utilizados en la sección Entrega y Facturación del Catálogo de Clientes. Ningún otro catálogo se modifica en este requisito.

**Regla 2 — Tipos de actualización**
Los tipos de actualización comprenden **altas**, **bajas** y **modificaciones** de opciones del catálogo, ejecutadas directamente en base de datos por el equipo de Soporte a la Producción (equipo SAP) con base en el listado entregado por el cliente.

**Regla 3 — Formato de despliegue del Régimen Fiscal**
El catálogo de Régimen Fiscal se despliega en los selectores con el formato `Clave - Descripción`, separados por guion (por ejemplo, `601 - General de Ley Personas Morales`).

**Regla 4 — Formato de despliegue del Tipo de Sociedad Mercantil**
El catálogo de Tipo de Sociedad Mercantil se despliega en los selectores mostrando únicamente la **descripción** de la opción, sin guion ni clave adicional.

**Regla 5 — Despliegue igual para todos los clientes (sin diferenciación por región)**
Los dos catálogos se despliegan con la misma lista para todos los clientes, independientemente de la Región (México o Perú). El sistema no filtra ni diferencia el catálogo por región.

**Regla 6 — Origen de la actualización**
La lista de opciones a agregar, dar de baja o modificar proviene del documento *R16 - Catálogos Fiscales* consolidado por el cliente. Ese documento es la fuente única de la actualización.

**Regla 7 — Tratamiento de clientes afectados por bajas**
Cuando la actualización incluye bajas de opciones que hoy están asignadas a clientes existentes, el cliente ejecuta una **curaduría** para reasignar esos clientes a las opciones vigentes. La curaduría es responsabilidad del cliente y se coordina con el equipo de Soporte a la Producción.

---

## Riesgos

**Riesgo 1 — Catálogos desactualizados frente al SAT**
El catálogo de Régimen Fiscal se alinea al catálogo publicado por el SAT. Si el SAT actualiza su catálogo después del despliegue de R16 (nuevas opciones, cambios de descripción o retiros), el catálogo interno quedará desalineado hasta que se ejecute una nueva actualización manual. **Mitigación:** protocolo de revisión periódica del catálogo SAT vigente por parte del área fiscal de PROQUIFA.

**Riesgo 2 — Clientes con opciones dadas de baja del catálogo**
Cuando la actualización incluye bajas, los clientes que hoy tienen asignadas opciones que serán dadas de baja quedan con un valor inválido tras la actualización si no se ejecuta la curaduría previamente. **Resuelto:** el cliente entrega el listado depurado y ejecuta la curaduría reasignando esos clientes a las opciones vigentes antes de la baja (ver Regla 7).

---

## Criterios de Aceptación

### Sección A — Despliegue de los catálogos actualizados

**Criterio A1 — Régimen Fiscal muestra la lista vigente con formato Clave - Descripción**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta la sección Entrega y Facturación,
- **Cuando** despliega el selector de Régimen Fiscal,
- **Entonces** el sistema deberá mostrar todas las opciones vigentes del catálogo actualizado, cada una con el formato `Clave - Descripción` separado por guion.

**Criterio A2 — Tipo de Sociedad Mercantil muestra la lista vigente por descripción**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta la sección Entrega y Facturación,
- **Cuando** despliega el selector de Tipo de Sociedad Mercantil,
- **Entonces** el sistema deberá mostrar todas las opciones vigentes del catálogo actualizado, cada una únicamente con su **descripción** (sin clave ni guion).

**Criterio A3 — Altas reflejadas en el selector**
- **Dado** que se agregó una opción nueva al catálogo directamente en base de datos,
- **Cuando** un usuario despliega el selector correspondiente en el sistema,
- **Entonces** deberá ver la opción nueva incluida en la lista, con el formato establecido para ese catálogo.

**Criterio A4 — Bajas reflejadas en el selector**
- **Dado** que se dio de baja una opción del catálogo directamente en base de datos,
- **Cuando** un usuario despliega el selector correspondiente en el sistema,
- **Entonces** el sistema **no deberá mostrar** la opción dada de baja en la lista. Los clientes que ya la tenían asignada mantienen el valor a nivel de datos hasta que se ejecute la curaduría.

**Criterio A5 — Despliegue igual para todos los clientes sin diferenciación por región**
- **Dado** que un usuario consulta la sección Entrega y Facturación de dos clientes en regiones distintas (México y Perú),
- **Cuando** despliega los selectores de Régimen Fiscal y Tipo de Sociedad Mercantil en cada uno,
- **Entonces** el sistema deberá mostrar la **misma lista** de opciones en ambos casos, sin filtrar por región.

---

## Notas de Implementación

- Este requisito cubre exclusivamente la actualización de los catálogos de **Régimen Fiscal** y **Tipo de Sociedad Mercantil**. La captura del identificador fiscal, las etiquetas del campo, los bloqueos de guardado y la persistencia son funcionalidad ya existente y no se modifican.
- El área responsable de ejecutar las altas, bajas y modificaciones en base de datos es **Soporte a la Producción (equipo SAP)** de PROQUIFA.
- La fuente única del listado a aplicar es el documento *R16 - Catálogos Fiscales* consolidado por el cliente, junto con el archivo de equivalencias que se levantó en el análisis previo para mapear el estado actual con el estado objetivo.
- Cuando la actualización incluye bajas, la curaduría de clientes que tienen la opción asignada se coordina con el cliente antes de ejecutar la baja en base de datos.
- La validación de formato del RFC ya está implementada en el sistema y no se modifica en este release.

**Resueltos (dudas cerradas):**
- **Confirmación del cliente sobre las opciones de los catálogos:** cerrada con el documento *R16 - Catálogos Fiscales* que contiene el listado consolidado. Se conserva el registro de las discrepancias detectadas durante el análisis previo (ver archivo de equivalencias).
- **Tratamiento de clientes con opciones dadas de baja:** cerrado con la Regla 7 — el cliente ejecuta la curaduría reasignando esos clientes a opciones vigentes antes de la baja.
- **Validación del RUC:** sin objeto — no se regionalizan los campos ni sus catálogos, la validación del identificador fiscal es funcionalidad ya existente.
- **Denominación "Sociedad por Acciones Cerrada Simplificada":** sin objeto — no se regionalizan los catálogos.
- **Categoría tributaria "Régimen para Personas Naturales":** sin objeto — no se regionalizan los catálogos.

### Documentos de referencia del cliente

- Documento *R16 - Catálogos Fiscales* — listado consolidado de opciones vigentes de Régimen Fiscal y Tipo de Sociedad Mercantil.
- Archivo de equivalencias del análisis previo — mapeo entre el estado actual del catálogo en el sistema y el estado objetivo confirmado por el cliente.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-07-29 | Reenfoque del requisito | Historia, Requisito, Alcance y Criterios de Aceptación reescritos completos. El requisito pasa a cubrir únicamente la actualización de los catálogos de Régimen Fiscal y Tipo de Sociedad Mercantil, sin regionalización. Se retira la captura y validación del identificador fiscal, las etiquetas dinámicas por región, la validación del RFC/RUC (Módulo 11), el bloqueo de guardado y la persistencia — todos son funcionalidad ya existente en el sistema. Nuevas Reglas 1–6 con alcance, tipos de actualización, formatos de despliegue por catálogo, despliegue único sin diferenciación por región y origen de la actualización. Nueva Sección A con 5 criterios de despliegue. Riesgo 1 acotado al catálogo del SAT; Riesgo 2 nuevo sobre clientes con valores dados de baja (marcado como pendiente en su momento). Se cierran los pendientes de validación del RUC, denominación "Sociedad por Acciones Cerrada Simplificada" y categoría tributaria "Régimen para Personas Naturales". |
| 2 | 2026-08-07 | Cierres de pendientes | Regla 7 y Observaciones: se cierra el pendiente de confirmación del cliente sobre las opciones de los catálogos con el documento *R16 - Catálogos Fiscales*. Se agrega la Regla 7 de tratamiento de clientes afectados mediante curaduría del cliente. Riesgo 2 y Observaciones: se cierra el pendiente del tratamiento de los clientes con opciones dadas de baja. Documentos de referencia del cliente: se referencia el documento *R16 - Catálogos Fiscales* junto con el archivo de equivalencias del análisis previo. |
