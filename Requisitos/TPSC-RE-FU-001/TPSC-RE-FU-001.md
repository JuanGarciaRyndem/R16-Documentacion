# Mantenimiento de catálogos del sistema

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-001 |
| **Nombre** | Mantenimiento de catálogos del sistema |
| **Categoría** | Bases de Datos |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.3M-RE-FU-001 |

---

## Historia de Usuario

> Yo como **sistema PQF2**, quiero contar con un **Catálogo de Cuentas Bancarias del grupo PROQUIFA**, que mantenga las cuentas vigentes de cada empresa emisora del grupo, para que los módulos que requieran información de cobro o pago dispongan de un **origen único y consistente** de las cuentas bancarias del grupo.

---

## Requisito

El sistema debe mantener un **Catálogo de Cuentas Bancarias del grupo PROQUIFA** con las cuentas vigentes de cada empresa emisora del grupo. El catálogo debe estar disponible como fuente única de consulta para los módulos del sistema que requieran información de las cuentas bancarias del grupo. El estado de cada cuenta debe poder distinguir entre cuentas que **existen vigentes** en sistema y cuentas que **ya no existen vigentes** (sin exponer el mecanismo técnico de borrado al consumidor).

---

## Alcance

### Aplica a

- Catálogo único de cuentas bancarias del grupo PROQUIFA, mantenido por el área de Soporte a la Producción.
- Empresas del grupo PROQUIFA México: Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica. Cada cuenta pertenece a una sola empresa emisora.
- Consulta del catálogo desde cualquier módulo del sistema que requiera información de cuentas bancarias del grupo (alta de cliente, emisión de proforma, pantallas de Tesorería, Validar Cobro, integraciones que consuman el catálogo).
- Estado de existencia vigente de cada cuenta: una cuenta existe vigente en sistema o no existe vigente.
- ** Identificación automática de pagos entrantes desde Buzón de Pagos: pendiente confirmar si esta funcionalidad aplica como alcance R16 (depende de la propuesta de identificación automática con IA aún por confirmar con el cliente). **

### No aplica a

- Interfaz gráfica de usuario (UI) para alta, baja, modificación o consulta de cuentas bancarias: no se desarrolla en R16.
- Gestión de cuentas bancarias de clientes (clientes externos): aplica solo a las cuentas del grupo PROQUIFA emisor.
- Operaciones Perú: el modelo de cuentas bancarias para Golocaer S.A.C. no está definido en R16 (ver Riesgo 1).

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
El catálogo R16 está diseñado para las empresas mexicanas del grupo. Golocaer S.A.C. (Perú) opera bajo normativa SUNAT con estructura potencialmente distinta. Mitigación: el modelo Perú queda fuera del alcance R16 y debe definirse en un release posterior.

> ** Validar con el cliente cuándo se aborda el catálogo Perú. **

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
- **Entonces** la solicitud deberá canalizarse al área de Soporte a la Producción, que ejecuta la operación directamente en la base de datos del sistema.

---

## Notas de Implementación

- Referencia técnica: el catálogo se materializa en la tabla `CuentaBanco` de la BD PConnect. Campo `activo=1` identifica las cuentas vigentes; `activo=0` las que ya no están vigentes pero se conservan para trazabilidad histórica.
- Área responsable del mantenimiento: **Soporte a la Producción (PROQUIFA)**. Ejecuta operaciones de alta, baja y modificación directamente en la base de datos del sistema.

> ** Buzón de Pagos como consumidor del catálogo: el catálogo se incluyó como input del Buzón de Pagos pensando en la propuesta de identificación automática de pagos entrantes (mapeo de cuenta destino a empresa emisora vía IA). Si la propuesta de IA no se materializa como alcance R16, esta relación debe revisarse y posiblemente eliminarse del alcance. **

> ** Estado del catálogo Perú: el modelo de cuentas bancarias para Golocaer S.A.C. queda fuera del alcance R16 y debe definirse en un release posterior con normativa SUNAT (ver Riesgo 1). **

### Pendientes operativos

- Detalle completo de cuentas por empresa del grupo (banco, número, CLABE, moneda, códigos) — pendiente entrega completa por PROQUIFA.
- Decisión sobre asignación manual vs automática (propuesta IA leyendo correos del Buzón de Pagos).
