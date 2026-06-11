# Mantenimiento de catálogos del sistema

| Campo                 | Valor                                  |
| --------------------- | -------------------------------------- |
| **ID**                | TPSC-RE-FU-001                         |
| **Nombre**            | Mantenimiento de catálogos del sistema |
| **Categoría**         | Bases de Datos                         |
| **Estatus**           | Propuesto                              |
| **Referencia Legacy** | R16.3M-RE-FU-001                       |

---

## Historia de Usuario

> Yo como **sistema PQF2**, quiero contar con un **Catálogo de Cuentas Bancarias del grupo PROQUIFA**, que mantenga las cuentas vigentes de cada empresa emisora del grupo, para que los módulos que requieran información de cobro o pago dispongan de un **origen único y consistente** de las cuentas bancarias del grupo.

---

## Requisito

El sistema debe mantener un **Catálogo de Cuentas Bancarias del grupo PROQUIFA** con las cuentas vigentes de cada empresa emisora del grupo. El catálogo debe estar disponible como fuente única de consulta para los módulos del sistema que requieran información de las cuentas bancarias del grupo. El estado de cada cuenta debe poder distinguir entre cuentas que **existen vigentes** en sistema y cuentas que **ya no existen vigentes** (sin exponer el mecanismo técnico de borrado al consumidor). La baja de una cuenta es siempre **lógica**: el registro se conserva marcado como no vigente y nunca se elimina físicamente de la base de datos.

---

## Alcance

### Aplica a

- Catálogo único de cuentas bancarias del grupo PROQUIFA, mantenido por el área de Soporte a la Producción.
- Empresas del grupo PROQUIFA México: Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica. Cada cuenta pertenece a una sola empresa emisora.
- Consulta del catálogo desde cualquier módulo del sistema que requiera información de cuentas bancarias del grupo (alta de cliente, emisión de proforma, pantallas de Tesorería, Validar Cobro, integraciones que consuman el catálogo).
- Estado de existencia vigente de cada cuenta: una cuenta existe vigente en sistema o no existe vigente. La baja es lógica: el registro de la cuenta se conserva marcado como no vigente y nunca se elimina físicamente de la base de datos.

### No aplica a

- Interfaz gráfica de usuario (UI) para alta, baja, modificación o consulta de cuentas bancarias: no se desarrolla en R16.
- Gestión de cuentas bancarias de clientes (clientes externos): aplica solo a las cuentas del grupo PROQUIFA emisor.
- Operaciones Perú: el catálogo de cuentas de Golocaer S.A.C. (Perú) forma parte del alcance de R16 (es necesario para validar cobros en Perú), pero su modelo —estructura de la cuenta conforme a normativa SUNAT— aún no está definido y se mantiene como brecha por resolver (ver Riesgo 1). Por ello, hasta que se defina el modelo, las cuentas Perú no pueden poblarse.

---

## Reglas de Negocio

**Regla 1 — Catálogo único de cuentas bancarias del grupo**
El sistema mantiene un Catálogo de Cuentas Bancarias del grupo PROQUIFA como fuente única de información sobre las cuentas bancarias de las empresas emisoras del grupo.

**Regla 2 — Estado de existencia vigente**
Cada cuenta bancaria tiene un estado que indica si existe vigente en sistema o no. Las cuentas que no existen vigentes no se ofrecen al usuario en ningún módulo consumidor, pero su información se conserva para trazabilidad histórica de operaciones previas.

**Regla 3 — Gestión sin interfaz gráfica en R16**
La gestión del catálogo (alta, baja, modificación) no dispone de interfaz gráfica de usuario en R16. La gestión es responsabilidad del área de Soporte a la Producción mediante acceso directo a la base de datos del sistema.

**Regla 4 — Consumo desde módulos del sistema**
Cualquier módulo del sistema que requiera información de cuentas bancarias del grupo PROQUIFA consulta el catálogo. No existen catálogos paralelos ni copias locales en otros módulos.

---

## Riesgos

**Riesgo 1 — Modelo de cuentas bancarias Perú no definido**
El catálogo de cuentas de Golocaer S.A.C. (Perú) forma parte del alcance de R16, pero su modelo —estructura de la cuenta conforme a normativa SUNAT, potencialmente distinta de la mexicana— aún no está definido. Mientras el modelo Perú no se defina, las cuentas de Golocaer S.A.C. no pueden poblarse en el catálogo, lo que bloquea la validación de cobros en Perú. **Brecha prioritaria por resolver con el cliente.**

**Riesgo 2 — Visibilidad restringida del catálogo no materializada en R16**
Sin interfaz de usuario, los consumos al catálogo dependen completamente de la integridad de los datos cargados en BD. Errores manuales en BD impactan directamente todos los módulos consumidores sin posibilidad de validación por usuario final.

---

## Criterios de Aceptación

### Sección A — Disponibilidad del catálogo

**Criterio A1 — Consulta del catálogo desde módulos**
- **Dado** que un módulo del sistema requiere información de cuentas bancarias del grupo PROQUIFA,
- **Cuando** consulta el Catálogo de Cuentas Bancarias,
- **Entonces** deberá obtener las cuentas existentes vigentes que apliquen al contexto.

**Criterio A2 — Filtrado por empresa emisora**
- **Dado** que un módulo solicita las cuentas bancarias asociadas a una empresa emisora específica del grupo (Golocaer, Mungen, Proquifa o Proveedora Quimico Farmaceutica),
- **Cuando** consulta el catálogo,
- **Entonces** deberá obtener las cuentas vigentes cuya empresa beneficiaria coincida con la solicitada.

### Sección B — Estado de existencia de las cuentas

**Criterio B1 — Solo cuentas existentes vigentes ofrecidas al usuario**
- **Dado** que un módulo lista cuentas bancarias del grupo en una pantalla operativa,
- **Cuando** renderiza el listado,
- **Entonces** deberá incluir únicamente las cuentas que existen vigentes en sistema. Las cuentas que no existen vigentes no aparecen en ningún listado al usuario final.

**Criterio B2 — Conservación histórica**
- **Dado** que una cuenta bancaria deja de existir vigente en el sistema,
- **Cuando** una operación histórica previa referencia esa cuenta,
- **Entonces** la información de la cuenta debe permanecer accesible para fines de trazabilidad histórica.

### Sección C — Gestión del catálogo

**Criterio C1 — Sin interfaz de usuario en R16**
- **Dado** que un usuario operativo requiere agregar, modificar o dar de baja una cuenta bancaria del grupo,
- **Cuando** intenta hacerlo desde la aplicación PQF2,
- **Entonces** NO deberá encontrar pantalla disponible para esta operación.

**Criterio C2 — Gestión por Soporte a la Producción**
- **Dado** que se requiere alta, modificación o baja de una cuenta bancaria del grupo,
- **Cuando** se solicita la operación,
- **Entonces** la solicitud deberá canalizarse al área de Soporte a la Producción (equipo SAP), que ejecuta la operación directamente en la base de datos del sistema. En el caso de una baja, ésta se realiza a nivel lógico: el registro de la cuenta no se elimina de la base de datos, sino que se marca como no vigente, conservando el registro para trazabilidad histórica.

---

## Notas de Implementación

- Referencia técnica para implementación: el catálogo se materializa en la tabla `CuentaBanco` de la BD PConnect (referencia del sistema actual). Campo `activo=1` identifica las cuentas vigentes; `activo=0` las que ya no están vigentes pero se conservan para trazabilidad histórica de operaciones previas.
- Área responsable del mantenimiento del catálogo: **Soporte a la Producción (PROQUIFA)**. El área ejecuta operaciones de alta, baja y modificación directamente en la base de datos del sistema. **Nota:** «equipo SAP» y «Soporte a la Producción» son el mismo actor en la documentación del proyecto.
- **Estado del catálogo Perú:** el catálogo de cuentas de Golocaer S.A.C. forma parte del alcance de R16, pero su modelo (estructura conforme a normativa SUNAT) aún no está definido y es una brecha prioritaria por resolver. Mientras no se defina, las cuentas Perú no pueden poblarse (ver Riesgo 1).
- La identificación automática de pagos entrantes con IA **NO** forma parte de R16: la asistencia con IA se limita a la captura de datos del cobro en Validar Cobro (ver TPSC-RE-FU-024).

### Pendientes operativos

- Detalle completo de cuentas por empresa del grupo (banco, número, CLABE, moneda, códigos) — pendiente entrega completa por PROQUIFA.

---

## Cambios

| # | Fecha | Observación | Descripción del cambio |
|---|-------|-------------|------------------------|
| 1 | 2026-06-10 | OBS-001 | La baja de cuentas bancarias es siempre lógica. Se precisa en Requisito, Alcance "Aplica a" y Criterio C2: el registro se conserva marcado como no vigente; nunca se elimina físicamente. |
| 2 | 2026-06-10 | OBS-002 | Se registra el sinónimo del proyecto: «equipo SAP» = «Soporte a la Producción» (mismo actor). Agregado en Criterio C2 y Notas de Implementación. La documentación de BD del catálogo es el entregable dirigido a ese equipo. |
| 3 | 2026-06-10 | Revisión matriz | Alcance "No aplica a" Perú reformulado: el catálogo Golocaer S.A.C. SÍ es alcance R16 (necesario para cobros Perú), pero el modelo SUNAT no está definido — brecha prioritaria. Riesgo 1 actualizado en consecuencia. Pendiente de IA eliminado (no es alcance R16). |
