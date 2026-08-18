# Liberación Paquete 01

| Campo | Valor |
| :---- | :---- |
| **FORMATO** | N/A |
| **PROYECTO** | R16 Tramitación sin Crédito |
| **REFERENCIA** | N/A |
| **VERSIÓN** | 1.0 |
| **FECHA** | 14/08/2026 |
| **AUTOR** | Juan David García Cruz |
| **REVISOR** | Valdemar Farina Sánchez |

---

## Alcance del Paquete 01

El Paquete 01 libera la primera entrega funcional del proyecto R16 — Tramitación sin Crédito, cubriendo los requisitos **001, 002, 003, 005, 006 y 007** (RE-FU-004 queda **cancelado**). Incluye:

- **PRs** en los 4 aplicativos del stack: ProquifaDotNet, DocumentBuilder, TaskScheduler e IdentityServer.
- **Integración de base de datos** con las estructuras liberadas en R18.
- **Scripts de base de datos** por requisito (DDL + DML + seeds de catálogos).
- **Manuales de usuario** y **listados de altas operativas** (usuarios, carteras, depuración de catálogos) que dependen del cliente.

---

## PRs a liberar (transversales al paquete)

- **ProquifaDotNet** — PR: _pendiente_ · Cambios agregados de los requisitos 001, 002, 003, 005, 006, 007 (.NET Framework 4.8)
- **DocumentBuilder** — PR: _pendiente_ · Cambios del requisito 007 (plantillas / servicios de generación de documentos)
- **TaskScheduler** — PR: _pendiente_ · Cambios asociados a tareas programadas del paquete (si aplica en 001–007)
- **IdentityServer** — PR: _pendiente_ · Cambios de autenticación / autorización que soporten los nuevos roles y funciones del paquete (002, 003)

> **Nota:** cada PR debe estar aprobado, con checks verdes en CI y adjuntar link a la evidencia de code review antes de ejecutar los scripts de BD.

---

## Base de datos

### Integración de Base de datos de R18

Antes de ejecutar los scripts del Paquete 01 debe estar aplicada la base de R18. Verificar que los objetos que R16 consume (tablas, catálogos, vistas, SPs de R18) existen en el ambiente destino con la versión esperada.

**Checklist de precondiciones:**

- [ ] Backup de la base productiva / de QA antes del despliegue.
- [ ] Verificación de que R18 está desplegado y estable en el ambiente destino.
- [ ] Ventana de mantenimiento coordinada con operaciones.

### Scripts de BD

Concentrador de scripts a ejecutar en orden (owner del listado: Ingeniería):

📄 **Sheet:** [R16 — Scripts de BD Paquete 01](https://docs.google.com/spreadsheets/d/1RWolqnXab7w0_IVr9cD34mkDw1zBnq9dOvaQLy4opOs/edit?usp=sharing)

El sheet debe indicar por script: número consecutivo, requisito, tipo (DDL/DML/Seed), objeto afectado, dependencias, checksum y estado de ejecución por ambiente (DEV / QA / PROD).

---

## Requisito 001

### Scripts de BD

- [ ] Ejecutar los scripts registrados para RE-FU-001 en el concentrador (sección correspondiente del sheet).
- [ ] Validar objetos creados/alterados post-ejecución (query de verificación en el mismo sheet).

### Liberación de PQF2

- **ProquifaDotNet** — PR: _pendiente_ · Estado: _pendiente_

### Manuales

📁 **Carpeta:** [Manuales RE-FU-001 — Google Drive](https://drive.google.com/drive/u/2/folders/1WcQ3MygdgDNFFhNApg04eXdgDrcRSG9I)

Confirmar que los manuales estén actualizados a la versión desplegada antes de la liberación.

---

## Requisito 002

### Scripts de BD

- [ ] Ejecutar los scripts registrados para RE-FU-002 en el concentrador.
- [ ] Verificar que el catálogo `catCobroEstatus` (u otros seeds del requisito) quede con las claves esperadas.

### Liberación de PQF2

- **ProquifaDotNet** — PR: _pendiente_ · Estado: _pendiente_

### Consideraciones para QA

Probar la generación de usuarios con las siguientes **Funciones**:

- CoordinadorDeTesoreria
- GerenteDeTesoreria
- AnalistaDeCuentasPorCobrar

### Carteras a generar

Generar las carteras correspondientes a las Funciones anteriores en el ambiente de QA/Producción según proceda.

### Listado de usuarios a registrar

| Nombre | Correo | Función | Cartera asignada | Región |
| :---- | :---- | :---- | :---- | :---- |
|  |  |  |  |  |
|  |  |  |  |  |

> **Pendiente del cliente:** entregar el listado definitivo antes de la ejecución.

---

## Requisito 003

### Scripts de BD

- [ ] Ejecutar los scripts registrados para RE-FU-003 en el concentrador.

### Liberación de PQF2

- **ProquifaDotNet** — PR: _pendiente_ · Estado: _pendiente_

### Generación de usuarios con roles nuevos

**Funciones a habilitar:**

- GestorDeAsuntosRegulatoriosYContenido
- AuxiliarDeAsuntosRegulatorios

**Roles nuevos:**

- Gestor de la Información
- Regulatorios

### Consideraciones para QA

Probar la generación de usuarios con las Funciones y Roles listados arriba, y verificar que los permisos derivados de los Roles nuevos operan como se espera en los módulos de asuntos regulatorios y gestión de contenido.

### Carteras a generar

Generar las carteras correspondientes a las Funciones anteriores.

### Listado de usuarios a registrar

| Nombre | Correo | Función | Rol | Cartera asignada | Región |
| :---- | :---- | :---- | :---- | :---- | :---- |
|  |  |  |  |  |  |
|  |  |  |  |  |  |

> **Pendiente del cliente:** entregar el listado definitivo antes de la ejecución.

---

## Requisito 004 — CANCELADO

Requisito **cancelado** — no forma parte del alcance del Paquete 01. Sin actividades de liberación.

---

## Requisito 005

### Scripts de BD

- [ ] Ejecutar los scripts registrados para RE-FU-005 en el concentrador.

### Liberación de PQF2

- **ProquifaDotNet** — PR: _pendiente_ · Estado: _pendiente_

### Depuración de catálogos por parte del cliente

📁 **Carpeta:** [Depuración de catálogos RE-FU-005 — Google Drive](https://drive.google.com/drive/folders/1vcReYUuk7xhT3UZ8d8jfbzWCBH_Fb7N_?usp=drive_link)

- [ ] Recibir del cliente el listado depurado de los catálogos que aplica RE-FU-005.
- [ ] Cotejar contra los catálogos en BD antes de la ejecución.
- [ ] Documentar en el sheet de scripts los INSERT/UPDATE de curación resultantes de la depuración.

> **Pendiente del cliente:** entrega del listado depurado.

---

## Requisito 006

### Scripts de BD

- [ ] Ejecutar los scripts registrados para RE-FU-006 en el concentrador.

### Liberación de PQF2

- **ProquifaDotNet** — PR: _pendiente_ · Estado: _pendiente_

---

## Requisito 007

### Liberación de aplicativos

- **ProquifaDotNet** — PR: _pendiente_ · Estado: _pendiente_
- **DocumentBuilder** — PR: _pendiente_ · Plantillas / servicios de generación de documentos requeridos por RE-FU-007 · Estado: _pendiente_

> Coordinar la liberación de DocumentBuilder con la de ProquifaDotNet — el aplicativo consume los servicios de DocumentBuilder al momento de generar los documentos.

---

## Orden de ejecución sugerido

1. Verificar despliegue previo de **R18** en el ambiente destino.
2. Backup de la base de datos.
3. **Merge y despliegue** de los PRs de IdentityServer, TaskScheduler, DocumentBuilder y ProquifaDotNet en el orden que la matriz de dependencias del paquete indique.
4. Ejecución de **scripts de BD** en el orden del sheet concentrador (por requisito).
5. **Post-deployment:** ejecución de seeds/depuración con los listados entregados por el cliente (002 usuarios, 003 usuarios + roles, 005 catálogos).
6. **Smoke tests** de QA en los flujos afectados por cada requisito.
7. Verificación final con el usuario funcional (aceptación).

---

## Punto de contacto por área

| Área | Responsable | Correo |
| :---- | :---- | :---- |
| Desarrollo Backend |  |  |
| Base de Datos |  |  |
| QA |  |  |
| PMO |  |  |
| Cliente (Tesorería, Regulatorios) |  |  |

---

## Riesgos y pendientes bloqueantes

| # | Riesgo / Pendiente | Requisito | Responsable | Estado |
| :---- | :---- | :---- | :---- | :---- |
| R1 | Listado de usuarios pendiente (Funciones Tesorería) | RE-FU-002 | Cliente |  |
| R2 | Listado de usuarios + Roles nuevos pendiente | RE-FU-003 | Cliente |  |
| R3 | Depuración de catálogos pendiente | RE-FU-005 | Cliente |  |
| R4 | Confirmar que R18 esté desplegado en cada ambiente antes del paquete | Transversal | DBA |  |
| R5 | Sincronización de PRs entre ProquifaDotNet y DocumentBuilder | RE-FU-007 | Desarrollo |  |

---

## Bitácora de despliegue

| Ambiente | Fecha | Hora inicio | Hora fin | Ejecutó | Resultado | Observaciones |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| DEV |  |  |  |  |  |  |
| QA |  |  |  |  |  |  |
| PROD |  |  |  |  |  |  |

---

## Control de versiones

| Versión | Fecha | Autor | Tipo de cambio | Descripción | Aprobó |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 1.0 | 14/08/2026 | Juan David García Cruz | Creación | Creación del documento. |  |
