# Mantenimiento de Catálogo de Clientes — Documentos Regulatorios

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-003 |
| **Nombre** | Mantenimiento de Catálogo de Clientes |
| **Catálogo** | Catálogo de Clientes |
| **Categoría** | Funcional |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.3M-RE-FU-004 |

---

## Historia de Usuario

> Yo como usuario con las funciones **Gestor de Asuntos Regulatorios y Contenido** o **Auxiliar de Asuntos Regulatorios** (rol **Gestor de la Información, Regulatorios**), quiero **cargar y mantener los documentos regulatorios de cada cliente** (Licencia Sanitaria y Aviso de Responsable Sanitario) en el Catálogo de Clientes, para **habilitar la pretramitación de pedidos con sustancias controladas** y asegurar el cumplimiento normativo de PROQUIFA.

---

## Requisito

El sistema debe contar con una sección **"Documentos Regulatorios"** dentro del Catálogo de Clientes que permita a los usuarios con función **Gestor de Asuntos Regulatorios y Contenido** o **Auxiliar de Asuntos Regulatorios** (rol **Gestor de la Información, Regulatorios**) cargar, visualizar, reemplazar y eliminar los documentos PDF de **Licencia Sanitaria** y **Aviso de Responsable Sanitario** por cliente. Cualquier otro usuario con acceso al catálogo puede visualizar la lista de documentos cargados y consultar su contenido, pero no puede cargarlos, reemplazarlos ni eliminarlos. El sistema no realiza validación de contenido, vigencia, firmas ni notificaciones de vencimiento. La sola presencia de ambos documentos cargados es lo que el módulo **Pretramitar Pedido** utiliza para validar y permitir la pretramitación de pedidos con sustancias controladas.

---

## Alcance

### Aplica a

- Clientes de Región México en el Catálogo de Clientes (la sección de Documentos Regulatorios y su uso como condicionante regulatoria se construyen para México).
- Sección “Documentos Regulatorios” dentro de la pantalla del cliente.
- Dos documentos PDF por cliente: **Licencia Sanitaria** y **Aviso de Responsable Sanitario**.
- Operaciones disponibles para las funciones **Gestor de Asuntos Regulatorios y Contenido** y **Auxiliar de Asuntos Regulatorios** (rol **Gestor de la Información, Regulatorios**) sobre los documentos: carga (subida), visualización, reemplazo (subir nueva versión que sobrescribe la anterior) y eliminación.
- Visualización en modo bloqueado (solo consulta del listado de documentos cargados y de su contenido) para cualquier otro usuario con acceso al catálogo.
- Sobrescritura sin historial visible: al reemplazar un documento, la versión anterior se descarta del listado y solo permanece la versión vigente en pantalla.

### No aplica a

- Validación del contenido del PDF, firmas digitales, autenticidad del documento o vigencia regulatoria.
- Notificación automática de vencimiento o renovación de los documentos cargados.
- Conservación de historial de versiones de los documentos en el listado de pantalla (cada reemplazo descarta la versión anterior sin trazabilidad visual).
- Validaciones y condicionantes regulatorias de Sustancias Controladas para Región Perú: no se implementan en esta release. El sistema permite la existencia de familias controladas en cualquier región (no las restringe), pero para Perú no se construyen las validaciones regulatorias ni el bloqueo condicionante en Pretramitar Pedido; el manejo de controlados en Perú es operativo, no controlado por el sistema.

---

## Reglas de Negocio

**Regla 1 — Carga y modificación exclusivas de las funciones autorizadas**
La carga, reemplazo y eliminación de documentos regulatorios del cliente son operaciones exclusivas de las funciones **Gestor de Asuntos Regulatorios y Contenido** y **Auxiliar de Asuntos Regulatorios** (rol **Gestor de la Información, Regulatorios**). Para cualquier otro usuario con acceso al catálogo los controles correspondientes se presentan en modo bloqueado.

**Regla 2 — Formato exclusivo PDF**
Los documentos regulatorios se aceptan únicamente en formato PDF. Cualquier otro formato de archivo es rechazado al momento de la carga.

**Regla 3 — Una sola versión vigente por documento en pantalla**
Cada cliente mantiene a lo más una versión vigente de cada documento regulatorio (Licencia Sanitaria y Aviso de Responsable Sanitario) visible en la pantalla del Catálogo de Clientes. Al subir una nueva versión, la versión anterior deja de listarse en pantalla y solo se muestra la vigente.

**Regla 4 — Sin validación de vigencia ni contenido**
El sistema no valida el contenido, autenticidad, firmas digitales ni fecha de vigencia de los documentos cargados, ni notifica vencimientos. La responsabilidad de mantener los documentos vigentes y correctos recae fuera del sistema, en el área de Regulatorios (funciones Gestor de Asuntos Regulatorios y Contenido y Auxiliar de Asuntos Regulatorios).

**Regla 5 — Documentos Regulatorios construidos para Región México**
La sección de Documentos Regulatorios y su uso como condicionante en Pretramitar Pedido se construyen para clientes de Región México. Para Región Perú no se implementan validaciones ni condicionantes regulatorias de Sustancias Controladas en esta release: el sistema no restringe la existencia de familias controladas, pero su manejo en Perú es operativo y no está soportado por validaciones del sistema.

---

## Riesgos

**Riesgo 1 — Documentos cargados pero no vigentes**
Como el sistema no valida la fecha de vigencia de los documentos cargados, un cliente podría tener Licencia Sanitaria o Aviso de Responsable Sanitario cargados pero vencidos legalmente. Operativamente esto permitiría tramitar pedidos con sustancias controladas sin documentación vigente, generando exposición regulatoria.
*Mitigación: protocolo de revisión periódica offline por parte del área Regulatorios.*

**Riesgo 2 — Pérdida de trazabilidad histórica en pantalla por reemplazo sin versionado visible**
Al sobrescribir un documento sin conservar la versión anterior en el listado de pantalla, se pierde la visibilidad en pantalla de qué Licencia Sanitaria estaba vigente en cada momento histórico. Esto puede generar problemas en auditorías regulatorias que requieran demostrar qué documento sustentaba un pedido pasado.

**Riesgo 3 — Eliminación accidental**
Como los usuarios con función Gestor de Asuntos Regulatorios y Contenido o Auxiliar de Asuntos Regulatorios pueden eliminar documentos cargados, una eliminación accidental dejaría al cliente sin documentación, bloqueando inmediatamente cualquier tramitación con sustancias controladas hasta nueva carga.

---

## Criterios de Aceptación

### Sección A — Visibilidad y acceso a la sección

**Criterio A1 — Sección Documentos Regulatorios visible en el Catálogo de Clientes**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta un cliente específico,
- **Cuando** se renderiza la pantalla del cliente,
- **Entonces** deberá presentarse la sección “Documentos Regulatorios” con el listado de los dos documentos posibles (Licencia Sanitaria y Aviso de Responsable Sanitario), indicando para cada uno si tiene archivo cargado o si está vacío.

**Criterio A2 — Modo bloqueado para usuarios sin función autorizada**
- **Dado** que un usuario sin función Gestor de Asuntos Regulatorios y Contenido ni Auxiliar de Asuntos Regulatorios consulta la sección Documentos Regulatorios de un cliente,
- **Cuando** visualiza la pantalla,
- **Entonces** el sistema deberá presentar el listado de documentos cargados en modo bloqueado: el usuario puede ver qué documentos están registrados y consultar su contenido, pero no puede cargar, reemplazar ni eliminar.

### Sección B — Carga y formato de los documentos

**Criterio B1 — Carga de PDF por función autorizada**
- **Dado** que el usuario tiene función Gestor de Asuntos Regulatorios y Contenido o Auxiliar de Asuntos Regulatorios y consulta la sección Documentos Regulatorios de un cliente,
- **Cuando** ejecuta la acción de cargar Licencia Sanitaria o Aviso de Responsable Sanitario,
- **Entonces** el sistema deberá permitir seleccionar un archivo PDF del equipo del usuario y, al confirmar, almacenarlo asociado al cliente y al tipo de documento correspondiente.

**Criterio B2 — Rechazo de formatos distintos a PDF**
- **Dado** que un usuario con función autorizada (Gestor de Asuntos Regulatorios y Contenido o Auxiliar de Asuntos Regulatorios) intenta cargar un archivo en formato distinto a PDF,
- **Cuando** el sistema valida el archivo,
- **Entonces** deberá rechazar la operación, no almacenar el archivo y mostrar un mensaje claro indicando que solo se aceptan documentos en formato PDF.

### Sección C — Visualización de los documentos

**Criterio C1 — Entrega del PDF al navegador del usuario**
- **Dado** que un usuario consulta un documento previamente cargado,
- **Cuando** ejecuta la acción de visualizar,
- **Entonces** el sistema deberá entregar el archivo PDF al navegador del usuario. El comportamiento posterior de apertura (pestaña nueva, pestaña actual o descarga directa) depende de la configuración del navegador y de las preferencias del usuario, fuera del control del sistema.

### Sección D — Reemplazo y eliminación

**Criterio D1 — Reemplazo de documento existente**
- **Dado** que el cliente ya tiene cargada una Licencia Sanitaria o un Aviso de Responsable Sanitario y un usuario con función Gestor de Asuntos Regulatorios y Contenido o Auxiliar de Asuntos Regulatorios carga una nueva versión del mismo documento,
- **Cuando** el sistema procesa la operación,
- **Entonces** deberá registrar la nueva versión como vigente. En el listado de pantalla únicamente se muestra la versión vigente; la versión anterior deja de listarse.

**Criterio D2 — Eliminación de documento cargado**
- **Dado** que el usuario tiene función Gestor de Asuntos Regulatorios y Contenido o Auxiliar de Asuntos Regulatorios y un documento está cargado para un cliente,
- **Cuando** ejecuta la acción de eliminar,
- **Entonces** el sistema deberá solicitar confirmación explícita al usuario y, al confirmar, retirar el documento del catálogo del cliente. Tras la eliminación, el documento aparece como no cargado en el listado de pantalla.

### Sección E — Alcance de validación

**Criterio E1 — Sin validación de vigencia ni contenido**
- **Dado** que un usuario con función Gestor de Asuntos Regulatorios y Contenido o Auxiliar de Asuntos Regulatorios cargó un documento PDF para un cliente,
- **Cuando** el sistema almacena el archivo,
- **Entonces** no deberá ejecutar ninguna validación de fecha de vigencia, contenido del documento ni autenticidad. El documento se considera vigente para fines del sistema por el solo hecho de estar cargado en el catálogo.

---

## Notas de Implementación

- Funcionalidad ubicada en la sección “Documentos Regulatorios” del cliente dentro del Catálogo de Clientes.
- Los dos documentos contemplados son exclusivamente: **Licencia Sanitaria** y **Aviso de Responsable Sanitario**. La carga de otros tipos de documentos del cliente no entra en este alcance.
- Las funciones **Gestor de Asuntos Regulatorios y Contenido** y **Auxiliar de Asuntos Regulatorios** (rol **Gestor de la Información, Regulatorios**) son las únicas autorizadas para cargar, reemplazar y eliminar los PDFs. Cualquier otro usuario con acceso al catálogo puede visualizar la lista y consultar el contenido de los documentos, pero no puede modificarlos.
- Comportamiento de visualización del PDF: el sistema entrega el archivo al navegador del usuario. El comportamiento posterior (apertura en pestaña nueva, en la pestaña actual o descarga directa) depende de la configuración del navegador. **Sin visor embebido dentro del módulo.**
- Comportamiento de eliminación / reemplazo a nivel de almacenamiento físico: al eliminar o reemplazar un documento desde la pantalla del Catálogo de Clientes, se elimina o reemplaza el **registro lógico** que apunta al archivo, pero los archivos físicos previos permanecen en el almacenamiento **MinIO** sin purga automática. En pantalla del listado solo se muestra la versión vigente.
- El sistema no realiza validación de contenido, autenticidad, firmas digitales ni fechas de vigencia.
- El sistema no notifica automáticamente vencimientos ni próximas renovaciones. La responsabilidad recae offline en el área de Regulatorios (funciones Gestor de Asuntos Regulatorios y Contenido y Auxiliar de Asuntos Regulatorios).
- Esta funcionalidad es habilitadora de la validación bloqueante en el módulo **Pretramitar Pedido** para pedidos con sustancias controladas (ver requisito **R16A-RE-FU-009**).
- La administración de roles del sistema (creación del rol Gestor de la Información, Regulatorios, y asignación de funciones a usuarios) se gestiona a nivel base de datos.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-06-10 | OBS-007 | Alcance restringido a Región México. Regla 5 actualizada: los Documentos Regulatorios y su uso como condicionante en Pretramitar Pedido se construyen solo para México. Para Región Perú no se implementan validaciones ni condicionantes regulatorias en esta release. Se agrega bullet en "No aplica a" para Perú y se elimina Riesgo 4 (denominación Perú) ya no relevante. |
| 2 | 2026-08-05 | Cierre de pendientes de rol y función | Se definen las funciones autorizadas (Gestor de Asuntos Regulatorios y Contenido, Auxiliar de Asuntos Regulatorios) y el rol Gestor de la Información, Regulatorios. Se precisa en Historia de Usuario, Requisito, Alcance, Regla 1, Regla 4, Riesgo 3, Criterios A2/B1/B2/D1/D2/E1 y Observaciones. Se corrige el bullet de permisos que listaba "visualizar contenido" como operación exclusiva — la visualización del contenido está disponible para cualquier usuario con acceso al catálogo. Se cierra el pendiente de confirmación del cliente sobre la propuesta de permisos: consulta abierta y alta/modificación/eliminación restringidas a las dos funciones. |
