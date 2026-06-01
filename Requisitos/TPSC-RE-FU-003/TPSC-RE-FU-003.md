# Mantenimiento de Catálogo de Clientes — Documentos Regulatorios

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-003 |
| **Nombre** | Mantenimiento de Catálogo de Clientes |
| **Catálogo** | Catálogo de Clientes |
| **Categoría** | Funcional |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.3M-RE-FU-004 |

---

## Historia de Usuario

> Yo como **usuario del área Regulatorios**, quiero **cargar y mantener los documentos regulatorios de cada cliente** (Licencia Sanitaria y Aviso de Responsable Sanitario) en el Catálogo de Clientes, para **habilitar la pretramitación de pedidos con sustancias controladas** y asegurar el cumplimiento normativo de PROQUIFA.

---

## Requisito

El sistema debe contar con una sección **“Documentos Regulatorios”** dentro del Catálogo de Clientes que permita al rol Regulatorios cargar, visualizar, reemplazar y eliminar los documentos PDF de **Licencia Sanitaria** y **Aviso de Responsable Sanitario** por cliente. Otros roles del sistema pueden visualizar la lista de documentos cargados pero no pueden modificarlos. El sistema no realiza validación de contenido, vigencia, firmas ni notificaciones de vencimiento. La sola presencia de ambos documentos cargados es lo que el módulo **Pretramitar Pedido** utiliza para validar y permitir la pretramitación de pedidos con sustancias controladas.

---

## Alcance

### Aplica a

- Clientes de México y Perú en el Catálogo de Clientes.
- Sección “Documentos Regulatorios” dentro de la pantalla del cliente.
- Dos documentos PDF por cliente: **Licencia Sanitaria** y **Aviso de Responsable Sanitario**.
- Operaciones disponibles para el rol Regulatorios sobre los documentos: carga (subida), visualización, reemplazo (subir nueva versión que sobrescribe la anterior) y eliminación.
- Visualización en modo bloqueado (solo consulta del listado de documentos cargados y de su contenido) para roles distintos de Regulatorios.
- Sobrescritura sin historial visible: al reemplazar un documento, la versión anterior se descarta del listado y solo permanece la versión vigente en pantalla.

### No aplica a

- Validación del contenido del PDF, firmas digitales, autenticidad del documento o vigencia regulatoria.
- Notificación automática de vencimiento o renovación de los documentos cargados.
- Conservación de historial de versiones de los documentos en el listado de pantalla (cada reemplazo descarta la versión anterior sin trazabilidad visual).

---

## Reglas de Negocio

**Regla 1 — Carga y modificación exclusivas del rol Regulatorios**
La carga, reemplazo y eliminación de documentos regulatorios del cliente son operaciones exclusivas del rol Regulatorios. Para los demás roles los controles correspondientes se presentan en modo bloqueado.

**Regla 2 — Formato exclusivo PDF**
Los documentos regulatorios se aceptan únicamente en formato PDF. Cualquier otro formato de archivo es rechazado al momento de la carga.

**Regla 3 — Una sola versión vigente por documento en pantalla**
Cada cliente mantiene a lo más una versión vigente de cada documento regulatorio (Licencia Sanitaria y Aviso de Responsable Sanitario) visible en la pantalla del Catálogo de Clientes. Al subir una nueva versión, la versión anterior deja de listarse en pantalla y solo se muestra la vigente.

**Regla 4 — Sin validación de vigencia ni contenido**
El sistema no valida el contenido, autenticidad, firmas digitales ni fecha de vigencia de los documentos cargados, ni notifica vencimientos. La responsabilidad de mantener los documentos vigentes y correctos recae fuera del sistema, en el rol Regulatorios.

**Regla 5 — Aplicación uniforme México y Perú**
La funcionalidad opera de manera idéntica para clientes de México y Perú. El equivalente documental peruano (denominaciones SUNAT/DIGEMID que correspondan) se carga bajo los mismos campos genéricos de Licencia Sanitaria y Aviso de Responsable Sanitario.

> ** Pendiente confirmar etiquetas de campos para Perú. **

---

## Riesgos

**Riesgo 1 — Documentos cargados pero no vigentes**
Como el sistema no valida la fecha de vigencia de los documentos cargados, un cliente podría tener Licencia Sanitaria o Aviso de Responsable Sanitario cargados pero vencidos legalmente. Operativamente esto permitiría tramitar pedidos con sustancias controladas sin documentación vigente, generando exposición regulatoria.
*Mitigación: protocolo de revisión periódica offline por parte del área Regulatorios.*

**Riesgo 2 — Pérdida de trazabilidad histórica en pantalla por reemplazo sin versionado visible**
Al sobrescribir un documento sin conservar la versión anterior en el listado de pantalla, se pierde la visibilidad en pantalla de qué Licencia Sanitaria estaba vigente en cada momento histórico. Esto puede generar problemas en auditorías regulatorias que requieran demostrar qué documento sustentaba un pedido pasado.

**Riesgo 3 — Eliminación accidental**
Como el rol Regulatorios puede eliminar documentos cargados, una eliminación accidental dejaría al cliente sin documentación, bloqueando inmediatamente cualquier tramitación con sustancias controladas hasta nueva carga.

**Riesgo 4 — Documentos peruanos con denominación distinta**
La nomenclatura de los documentos regulatorios en Perú (regulados por DIGEMID/SUNAT) puede no coincidir literalmente con “Licencia Sanitaria” y “Aviso de Responsable Sanitario” usados en México.

> ** Pendiente confirmar denominación exacta con el cliente y decidir si se modelan bajo los mismos campos genéricos o si requieren campos diferenciados por país. **

---

## Criterios de Aceptación

### Sección A — Visibilidad y acceso a la sección

**Criterio A1 — Sección Documentos Regulatorios visible en el Catálogo de Clientes**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta un cliente específico,
- **Cuando** se renderiza la pantalla del cliente,
- **Entonces** deberá presentarse la sección “Documentos Regulatorios” con el listado de los dos documentos posibles (Licencia Sanitaria y Aviso de Responsable Sanitario), indicando para cada uno si tiene archivo cargado o si está vacío.

**Criterio A2 — Modo bloqueado para roles distintos de Regulatorios**
- **Dado** que un usuario con rol distinto de Regulatorios consulta la sección Documentos Regulatorios de un cliente,
- **Cuando** visualiza la pantalla,
- **Entonces** el sistema deberá presentar el listado de documentos cargados en modo bloqueado: el usuario puede ver qué documentos están registrados y consultar su contenido, pero no puede cargar, reemplazar ni eliminar.

### Sección B — Carga y formato de los documentos

**Criterio B1 — Carga de PDF por rol Regulatorios**
- **Dado** que el usuario tiene rol Regulatorios y consulta la sección Documentos Regulatorios de un cliente,
- **Cuando** ejecuta la acción de cargar Licencia Sanitaria o Aviso de Responsable Sanitario,
- **Entonces** el sistema deberá permitir seleccionar un archivo PDF del equipo del usuario y, al confirmar, almacenarlo asociado al cliente y al tipo de documento correspondiente.

**Criterio B2 — Rechazo de formatos distintos a PDF**
- **Dado** que el rol Regulatorios intenta cargar un archivo en formato distinto a PDF,
- **Cuando** el sistema valida el archivo,
- **Entonces** deberá rechazar la operación, no almacenar el archivo y mostrar un mensaje claro indicando que solo se aceptan documentos en formato PDF.

### Sección C — Visualización de los documentos

**Criterio C1 — Entrega del PDF al navegador del usuario**
- **Dado** que un usuario consulta un documento previamente cargado,
- **Cuando** ejecuta la acción de visualizar,
- **Entonces** el sistema deberá entregar el archivo PDF al navegador del usuario. El comportamiento posterior de apertura (pestaña nueva, pestaña actual o descarga directa) depende de la configuración del navegador y de las preferencias del usuario, fuera del control del sistema.

### Sección D — Reemplazo y eliminación

**Criterio D1 — Reemplazo de documento existente**
- **Dado** que el cliente ya tiene cargada una Licencia Sanitaria o un Aviso de Responsable Sanitario y el rol Regulatorios carga una nueva versión del mismo documento,
- **Cuando** el sistema procesa la operación,
- **Entonces** deberá registrar la nueva versión como vigente. En el listado de pantalla únicamente se muestra la versión vigente; la versión anterior deja de listarse.

**Criterio D2 — Eliminación de documento cargado**
- **Dado** que el usuario tiene rol Regulatorios y un documento está cargado para un cliente,
- **Cuando** ejecuta la acción de eliminar,
- **Entonces** el sistema deberá solicitar confirmación explícita al usuario y, al confirmar, retirar el documento del catálogo del cliente. Tras la eliminación, el documento aparece como no cargado en el listado de pantalla.

### Sección E — Alcance de validación

**Criterio E1 — Sin validación de vigencia ni contenido**
- **Dado** que el rol Regulatorios cargó un documento PDF para un cliente,
- **Cuando** el sistema almacena el archivo,
- **Entonces** no deberá ejecutar ninguna validación de fecha de vigencia, contenido del documento ni autenticidad. El documento se considera vigente para fines del sistema por el solo hecho de estar cargado en el catálogo.

---

## Notas de Implementación

- Funcionalidad ubicada en la sección “Documentos Regulatorios” del cliente dentro del Catálogo de Clientes.
- Los dos documentos contemplados son exclusivamente: **Licencia Sanitaria** y **Aviso de Responsable Sanitario**. La carga de otros tipos de documentos del cliente no entra en este alcance.
- El rol Regulatorios es el único autorizado para cargar, reemplazar, eliminar y visualizar el contenido de los PDFs. Otros roles ven la sección en modo bloqueado.
- Comportamiento de visualización del PDF: el sistema entrega el archivo al navegador del usuario. El comportamiento posterior (apertura en pestaña nueva, en la pestaña actual o descarga directa) depende de la configuración del navegador. **Sin visor embebido dentro del módulo.**
- Comportamiento de eliminación / reemplazo a nivel de almacenamiento físico: al eliminar o reemplazar un documento desde la pantalla del Catálogo de Clientes, se elimina o reemplaza el **registro lógico** que apunta al archivo, pero los archivos físicos previos permanecen en el almacenamiento **MinIO** sin purga automática. En pantalla del listado solo se muestra la versión vigente.
- El sistema no realiza validación de contenido, autenticidad, firmas digitales ni fechas de vigencia.
- El sistema no notifica automáticamente vencimientos ni próximas renovaciones. La responsabilidad recae offline en el rol Regulatorios.
- Esta funcionalidad es habilitadora de la validación bloqueante en el módulo **Pretramitar Pedido** para pedidos con sustancias controladas (ver requisito **TPSC-RE-FU-009**).
- La administración de roles del sistema (creación del rol Regulatorios, asignación usuario-rol) se gestiona a nivel base de datos.

> ** Pendiente: definir si la denominación de los documentos en Perú difiere de “Licencia Sanitaria” y “Aviso de Responsable Sanitario” (regulados por DIGEMID/SUNAT) y, en caso afirmativo, si se modelan bajo los mismos campos genéricos o si requieren campos diferenciados por país. **
