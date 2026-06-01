# Mantenimiento de Catálogo de Clientes — Referencia de Pago y Código Validador

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-006 |
| **Nombre** | Mantenimiento de Catálogo de Clientes |
| **Catálogo** | Catálogo de Clientes |
| **Categoría** | Funcional |
| **Estatus** | Propuesto |
| **Referencia Legacy** | R16.3M-RE-FU-005 |

---

## Historia de Usuario

> Yo como usuario con acceso a la cartera de clientes, quiero asignar una o más cuentas bancarias del grupo PROQUIFA a cada cliente y capturar el Código Validador correspondiente a cada combinación cliente-cuenta, para que el sistema construya automáticamente la referencia bancaria que aparece en las proformas del cliente, permitiendo identificar correctamente los pagos recibidos en cada cuenta.

---

## Requisito

El sistema debe contar en la sección **Cobros** del Catálogo de Clientes con una sub-sección **“Referencia de Pago”** que permita asignar al cliente una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA, capturando para cada combinación cliente-cuenta un **Código Validador**. La referencia bancaria que aparece en cada proforma se reconstruye dinámicamente al generarla, aplicando reglas diferenciadas según el banco asociado a la cuenta seleccionada. La funcionalidad es nueva en PQF2 R16 y toma como referencia operativa el comportamiento equivalente del sistema Legacy actual.

---

## Alcance

### Aplica a

- Clientes de **México** en el Catálogo de Clientes.
- Pantalla **“Referencia de Pago”** como sub-sección dentro de la sección Cobros del cliente.
- Asignación de una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA al cliente.
- Captura manual del Código Validador para cada combinación cliente-cuenta.
- Reconstrucción dinámica de la referencia bancaria en cada generación de proforma asociada a la cuenta seleccionada.
- Replicación de la lógica documentada del sistema Legacy actual sobre Banamex (7 segmentos) y no-Banamex (nombre del cliente).

### No aplica a

- Clientes de Perú: queda fuera del alcance de este requisito en R16 hasta definir con el cliente el modelo bancario peruano de identificación de pagos. Levantada como duda formal del proyecto.
- Validación de formato o longitud del Código Validador (pendiente definir con el cliente si se requiere validación; el documento del cliente no especifica longitud máxima ni reglas de formato).
- Persistencia de la referencia armada (la referencia se reconstruye dinámicamente; no se almacena).
- Historial de cambios al Código Validador (al modificar el valor se sobrescribe el anterior sin trazabilidad de versiones).

---

## Reglas de Negocio

**Regla 1 — Pantalla “Referencia de Pago” en sección Cobros**
El Catálogo de Clientes cuenta con una sub-sección **“Referencia de Pago”** dentro de la sección Cobros, donde se gestionan la o las cuentas bancarias asignadas al cliente y su Código Validador correspondiente por combinación cliente-cuenta.

**Regla 2 — Asignación de una o más cuentas bancarias al cliente**
Un cliente puede tener asignadas una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA. Cada asignación se compone seleccionando primero el **Banco** (del catálogo de Bancos) y luego la **Cuenta** (del catálogo de cuentas del grupo, filtrado por el banco seleccionado). La **Sucursal** se hereda del dato de la cuenta seleccionada y es de solo lectura.

**Regla 3 — Código Validador por combinación cliente-cuenta**
Cada combinación cliente-cuenta tiene un Código Validador capturado manualmente por el usuario.

> **⚠️ Pendiente** — El documento del cliente no especifica longitud máxima ni reglas de formato del Código Validador. Queda como duda formal del proyecto antes del desarrollo.

**Regla 4 — Persistencia de la combinación cliente-cuenta-Código Validador**
La asignación persiste en la relación cliente-cuenta los datos: identificador de la cuenta bancaria, identificador del cliente y Código Validador capturado. La referencia bancaria armada que aparece en la proforma **no se almacena**.

**Regla 5 — Reconstrucción dinámica de la referencia en cada proforma**
La referencia bancaria que aparece en una proforma se reconstruye dinámicamente en el momento de generar la proforma, aplicando las reglas correspondientes según el banco de la cuenta. La referencia no se persiste; cada generación de proforma reconstruye el valor.

**Regla 6 — Referencia para bancos distintos de Banamex**
Para cuentas de bancos distintos de Banamex, la referencia bancaria es el **Nombre del cliente** como cadena directa, sin transformación adicional.

**Regla 7 — Referencia para Banamex (concatenación de 7 segmentos)**
Para cuentas de Banamex, la referencia bancaria se compone por la concatenación determinista de 7 segmentos:

| Segmento | Descripción | Fallback / Regla |
|---|---|---|
| 1 | Primera letra del nombre del cliente | “X” si no existe |
| 2 | Segunda letra del nombre del cliente | “X” si no existe |
| 3 | Tercera letra del nombre del cliente | “X” si no existe |
| 4 | Últimos 4 caracteres de la clave del cliente | Padding con ceros a la izquierda si la clave tiene menos de 4 caracteres |
| 5 | Código del banco (campo `Codigo` de la tabla `Bancos`) | — |
| 6 | Carácter de moneda | “P” si la primera letra del campo `Moneda` de la cuenta es “M” (peso); “D” en cualquier otro caso |
| 7 | Código Validador (campo `CodValidador` de la relación cliente-cuenta) | — |

**Regla 8 — Identificación del banco “Banamex”**
La determinación de si una cuenta pertenece a Banamex se realiza cruzando la cuenta contra la tabla de Empresas: el campo beneficiario de la cuenta debe coincidir con el prefijo de la empresa que factura, y la moneda debe cumplir una condición adicional.

> **⚠️ Pendiente** — La condición adicional de moneda aparece truncada en la documentación del cliente y no se ha clarificado. Queda como duda formal de desarrollo. Como simplificación alternativa, se propone para evaluación con desarrollo identificar Banamex directamente por nombre o ID del banco en la tabla `Bancos`.

**Regla 9 — Edición sin restricción de rol**
Cualquier usuario con acceso a la cartera del cliente puede asignar cuentas bancarias y capturar el Código Validador. La autorización proviene del acceso del usuario al cliente, no de un rol específico.

> **⚠️ Pendiente** — Validar con el cliente si esta funcionalidad debe restringirse a un rol específico (probablemente Coordinador de Tesorería) por sus implicaciones financieras.

---

## Riesgos

**Riesgo 1 — Inconsistencia entre proformas re-emitidas por reconstrucción dinámica**
Como la referencia bancaria se reconstruye en cada generación de proforma y no se almacena, si entre dos generaciones cambia el nombre del cliente, su clave, el Código Validador o cualquier dato fuente, la nueva proforma tendrá una referencia distinta a la original. Esto puede causar que un cliente pague con la referencia original (de una proforma anterior que conservó) y el sistema no identifique el pago correctamente al reconstruir la referencia con datos actualizados.

**Riesgo 2 — Código Validador sin validación de formato ni longitud**
Como el Código Validador es input manual sin validación, un usuario puede capturar un valor erróneo (espacios, caracteres especiales, longitud incompatible con el sistema bancario) que rompa la identificación de pagos.

**Riesgo 3 — Sin restricción de rol sobre asignación de cuentas bancarias**
La asignación de cuentas bancarias del grupo PROQUIFA a clientes y la captura del Código Validador tienen implicaciones financieras serias (errores en estos datos comprometen la identificación de pagos). Sin embargo, esta funcionalidad se modela inicialmente sin restricción de rol.

> **⚠️ Pendiente** — Validar con el cliente si debe restringirse a un rol específico (probablemente Coordinador de Tesorería).

**Riesgo 4 — Modelo Perú no definido**
La lógica documentada por el cliente corresponde exclusivamente a PROQUIFA México (Banamex, prefijos de empresas mexicanas, moneda en pesos/dólares). No existe documentación equivalente para el modelo bancario peruano de identificación de pagos. Mientras no se resuelva, los clientes Perú no podrán tener referencia bancaria armada por el sistema PQF2.

**Riesgo 5 — Pérdida de trazabilidad por sobrescritura del Código Validador**
Al modificar el Código Validador asignado a una combinación cliente-cuenta, el valor anterior se sobrescribe sin conservar historial. Esto puede generar problemas en auditorías o reconciliación de pagos pasados que fueron identificados con el valor anterior.

---

## Criterios de Aceptación

### SECCIÓN A — Acceso y Selección de Cuenta

**Criterio A1 — Acceso a pantalla “Referencia de Pago” desde sección Cobros**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta un cliente,
- **Cuando** navega a la sección Cobros,
- **Entonces** el sistema deberá ofrecer acceso a la sub-sección **“Referencia de Pago”** para gestionar las cuentas bancarias asignadas al cliente y sus Códigos Validadores.

**Criterio A2 — Selector de Banco desde catálogo de Bancos**
- **Dado** que un usuario en “Referencia de Pago” agrega una cuenta nueva,
- **Cuando** despliega el selector de Banco,
- **Entonces** el sistema deberá presentar las opciones del catálogo de Bancos del grupo PROQUIFA.

**Criterio A3 — Selector de Cuenta filtrado por Banco**
- **Dado** que el usuario seleccionó un Banco en “Referencia de Pago”,
- **Cuando** despliega el selector de Cuenta,
- **Entonces** el sistema deberá presentar únicamente las cuentas bancarias del catálogo del grupo PROQUIFA que correspondan al banco seleccionado.

**Criterio A4 — Sucursal autopoblada desde la cuenta seleccionada**
- **Dado** que el usuario seleccionó una Cuenta en “Referencia de Pago”,
- **Cuando** se renderiza el campo Sucursal,
- **Entonces** el sistema deberá autopopularlo con el valor del campo Sucursal de la cuenta, en modo **solo lectura** (no editable por el usuario en esta pantalla).

---

### SECCIÓN B — Código Validador y Persistencia

**Criterio B1 — Captura manual del Código Validador**
- **Dado** que el usuario configura una combinación cliente-cuenta,
- **Cuando** captura el campo Código Validador,
- **Entonces** el sistema deberá aceptar el valor como input manual del usuario sin aplicar validación de formato ni longitud en esta versión.

> **⚠️ Pendiente** — Definir reglas de validación con el cliente. Queda como duda formal del proyecto.

**Criterio B2 — Persistencia de la combinación cliente-cuenta-Código Validador**
- **Dado** que el usuario guarda una asignación nueva o modificada,
- **Cuando** el sistema procesa la operación,
- **Entonces** deberá persistir la combinación en la relación cliente-cuenta del modelo de datos. La referencia armada no se almacena.

**Criterio B3 — Múltiples cuentas asignables por cliente**
- **Dado** que un cliente ya tiene una o más cuentas bancarias asignadas,
- **Cuando** el usuario agrega una cuenta adicional,
- **Entonces** el sistema deberá permitir la asignación. No existe límite máximo de cuentas asignables por cliente en R16.

> **⚠️ Pendiente** — Validar con el cliente si se requiere algún tope máximo.

**Criterio B4 — Edición y eliminación de combinaciones**
- **Dado** que un cliente tiene una cuenta bancaria asignada con su Código Validador,
- **Cuando** el usuario edita o elimina la asignación,
- **Entonces** el sistema deberá permitir la operación. La edición sobrescribe el valor anterior sin conservar historial. La eliminación retira la combinación cliente-cuenta del sistema.

**Criterio B5 — Edición sin restricción de rol específica**
- **Dado** que cualquier usuario con acceso a la cartera del cliente abre la pantalla “Referencia de Pago”,
- **Cuando** intenta modificar la asignación de cuentas o el Código Validador,
- **Entonces** el sistema deberá permitir la edición sin requerir un rol específico.

> **⚠️ Pendiente** — Validar con el cliente si debe restringirse al rol Coordinador de Tesorería u otro por las implicaciones financieras.

---

### SECCIÓN C — Reconstrucción de la Referencia Bancaria

**Criterio C1 — Reconstrucción dinámica al generar la proforma**
- **Dado** que un módulo genera una proforma para el cliente con una cuenta bancaria asignada,
- **Cuando** se incorpora la referencia bancaria al PDF de la proforma,
- **Entonces** el sistema deberá reconstruir la referencia en ese momento aplicando las reglas según el banco de la cuenta, sin persistir el valor.

**Criterio C2 — Referencia para bancos distintos de Banamex**
- **Dado** que el banco de la cuenta asignada para una proforma no es Banamex,
- **Cuando** el sistema construye la referencia bancaria,
- **Entonces** la referencia será el **Nombre del cliente** como cadena directa, sin transformación adicional.

**Criterio C3 — Referencia para Banamex (7 segmentos)**
- **Dado** que el banco de la cuenta asignada para una proforma es Banamex,
- **Cuando** el sistema construye la referencia bancaria,
- **Entonces** deberá concatenar determinísticamente los 7 segmentos definidos en la Regla 7: tres primeras letras del nombre del cliente con fallback “X”, últimos 4 caracteres de la clave del cliente con padding de ceros, código del banco, carácter de moneda “P”/“D”, y Código Validador.

---

## Notas de Implementación

- Funcionalidad ubicada en la pantalla **“Referencia de Pago”**, sub-sección dentro de la sección Cobros del cliente en el Catálogo de Clientes.
- Funcionalidad **NUEVA en PQF2 R16**. Toma como referencia el comportamiento documentado del sistema Legacy actual de PROQUIFA, pero su implementación en PQF2 es desde cero.
- El modelo de datos involucra una relación **N:N** entre `Cliente` y `CuentaBanco` mediante una tabla cliente-cuenta que persiste el Código Validador por combinación.
- La referencia bancaria armada se reconstruye dinámicamente en cada generación de proforma; **NO se almacena en base de datos**. Esta decisión de diseño replica el comportamiento del Legacy y genera el riesgo documentado (Riesgo 1) de inconsistencia entre proformas re-emitidas.
- La lógica de Banamex (7 segmentos) y no-Banamex (nombre del cliente) replica 1:1 el comportamiento documentado por el cliente en el documento *“Especificación: Proceso para generar Referencia de Cliente (Código Validador)”* recibido el 2026-04-28.
- La funcionalidad provee insumos al módulo **Buzón de Pagos** (identificación automática de pagos entrantes contra la referencia armada). La integración entre ambos módulos se detalla en los requisitos correspondientes al Buzón de Pagos.

> **⚠️ Pendiente** — La condición de identificación de Banamex referente al campo moneda de la cuenta aparece truncada en el documento del cliente y no se ha clarificado. Queda como duda formal de desarrollo. Se propone evaluar con desarrollo identificar Banamex directamente por nombre o ID en la tabla `Bancos` como simplificación.

> **⚠️ Pendiente** — Longitud máxima y reglas de formato del Código Validador no especificadas por el cliente. El PMO del proyecto anunciaba que la información se enviaría pero no llegó. Queda como duda formal del proyecto.

> **⚠️ Pendiente** — Aplicabilidad de esta funcionalidad para clientes Perú. La documentación del cliente cubre exclusivamente PROQUIFA México. El modelo bancario peruano de identificación de pagos no está definido. Queda como duda formal del proyecto.

> **⚠️ Pendiente** — Validar con el cliente si la asignación de cuentas bancarias y captura del Código Validador debe restringirse al rol Coordinador de Tesorería u otro rol específico, considerando las implicaciones financieras.

> **⚠️ Pendiente** — Confirmar con el cliente si es posible asignar más de una cuenta bancaria por cliente y si se requiere tope máximo.
