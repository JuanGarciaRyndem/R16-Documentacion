# Mantenimiento de Catálogo de Clientes — Referencia de Pago y Código Validador

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-006 |
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

El sistema debe contar en la sección **Cobros** del Catálogo de Clientes con una sub-sección **“Referencia de Pago”** que permita asignar al cliente una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA, capturando para cada combinación cliente-cuenta un **Código Validador**. La referencia bancaria se arma y se persiste como **referencia vigente del cliente** al configurar su cuenta, aplicando reglas diferenciadas según el banco asociado; al generar el PDF de una proforma, la referencia vigente se **casa al documento** y las proformas ya emitidas conservan su referencia. La funcionalidad es nueva en ProquifaDotNet R16 y toma como referencia operativa el comportamiento equivalente del sistema Legacy actual.

---

## Alcance

### Aplica a

- Clientes de **México** en el Catálogo de Clientes.
- Pantalla **“Referencia de Pago”** como sub-sección dentro de la sección Cobros del cliente.
- Asignación de una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA al cliente.
- Captura manual del Código Validador para cada combinación cliente-cuenta.
- Generación de la **referencia bancaria vigente** al configurar/actualizar la cuenta del cliente y **casado** de esa referencia al PDF de cada proforma en firme.
- Replicación de la lógica documentada del sistema Legacy actual sobre Banamex (7 segmentos) y no-Banamex (nombre del cliente).
- Migración de los datos actuales del sistema Legacy a ProquifaDotNet: cuentas bancarias asignadas por cliente y sus Códigos Validadores (OBS-010).

### No aplica a

- ~~Clientes de Perú~~ **[Resuelto — Duda FU-006/FU-017]** — La pantalla "Referencia de Pago" y la captura de Código Validador **no aplican** a clientes de Perú (no existe mecanismo de identificación de pagos por código validador en Perú). Sin embargo, la generación de referencia bancaria **sí aplica**: para clientes Perú se genera por default con la **Razón Social** del cliente, utilizando el mismo camino que bancos distintos de Banamex (ver Regla 6-PER).
- Validación de formato o longitud del Código Validador (pendiente definir con el cliente si se requiere validación; el documento del cliente no especifica longitud máxima ni reglas de formato).
- Recálculo de la referencia ya casada a una proforma emitida (el PDF cae en firme; las proformas históricas conservan su referencia y no se regeneran al consultarse — OBS-015).
- Vista en pantalla del historial del Código Validador: el historial se conserva en ProquifaDotNet.BitacoraCambios para auditoría, sin componente de UI en R16 (la consulta se hace desde ese aplicativo).

---

## Reglas de Negocio

**Regla 1 — Pantalla “Referencia de Pago” en sección Cobros**
El Catálogo de Clientes cuenta con una sub-sección **“Referencia de Pago”** dentro de la sección Cobros, donde se gestionan la o las cuentas bancarias asignadas al cliente y su Código Validador correspondiente por combinación cliente-cuenta.

**Regla 2 — Asignación de una o más cuentas bancarias al cliente**
Un cliente puede tener asignadas una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA. Cada asignación se compone seleccionando primero el **Banco** (del catálogo de Bancos) y luego la **Cuenta** (del catálogo de cuentas del grupo, filtrado por el banco seleccionado). La **Sucursal** se hereda del dato de la cuenta seleccionada y es de solo lectura.

**Regla 3 — Código Validador por combinación cliente-cuenta**
Cada combinación cliente-cuenta tiene un Código Validador capturado manualmente por el usuario.

> **Resuelto** — El Código Validador es **numérico, siempre 2 dígitos con cero a la izquierda** (rango `01`–`99`). El Front enviará exactamente 2 caracteres; la columna en BD puede tener mayor capacidad pero el valor nunca la excederá.

**Regla 4 — Persistencia en dos niveles: referencia vigente del cliente y referencia casada a la proforma**
La referencia bancaria se persiste en dos niveles:

1. **Referencia vigente del cliente:** la combinación cliente-cuenta persiste en el modelo de datos el identificador de la cuenta bancaria, el identificador del cliente, el Código Validador capturado y la **referencia bancaria armada vigente**. Esta referencia se genera una sola vez al configurar la cuenta del cliente (Catálogo de Clientes → Cobros → Referencia de Pago) y **solo se regenera si cambia un dato fuente** (banco, cuenta, Código Validador o datos del cliente que la componen — Nombre y Clave).
2. **Referencia casada a la proforma:** al generarse el PDF de una proforma, la referencia vigente en ese momento queda casada al documento; las proformas ya emitidas conservan su referencia y no se ven afectadas por regeneraciones posteriores de la referencia vigente del cliente.

Adicionalmente, cada guardado o modificación del Código Validador se registra en el Aplicativo Nuevo **ProquifaDotNet.BitacoraCambios** (Reglas al diseñar — regla 8), que conserva el **historial completo** de cambios: valor anterior, valor nuevo, autor y fecha. Sustituye al esquema de rotación de un nivel de OBS-014; `ClienteDatosBancarios` ya no lleva columnas de historial propias.

**Regla 5 — Generación de la referencia y casado al PDF en firme**
La referencia bancaria se arma al configurar/actualizar la cuenta del cliente, aplicando las reglas según el banco de la cuenta (ver Reglas 6 y 7), y se persiste como referencia vigente del cliente. Al generar el PDF de una proforma, el sistema toma la **referencia vigente del cliente y la casa al documento**: el PDF cae en firme y, al consultarse después, no se reconstruye ni recalcula la referencia (refleja los datos casados al momento de la emisión, equivalente al comportamiento del Legacy — OBS-015).

**Regla 6 — Referencia para bancos distintos de Banamex**
Para cuentas de bancos distintos de Banamex, la referencia bancaria es el **Nombre del cliente** como cadena directa, sin transformación adicional.

**Regla 6-PER — Referencia para clientes de Perú (sin Código Validador)** **[Resuelto — Duda FU-006/FU-017]**
Para clientes de la Región Perú, no existe un mecanismo de identificación de pagos mediante Código Validador. La referencia bancaria se genera por default con la **Razón Social** del cliente como cadena directa, sin transformación adicional. No se requiere captura de Código Validador para estos clientes. Este mecanismo sigue el mismo camino que la Regla 6 (no-Banamex), usando Razón Social en lugar de Nombre.

**Regla 7 — Referencia para Banamex (concatenación de 7 segmentos)**
Para cuentas de Banamex, la referencia bancaria se compone por la concatenación determinista de 7 segmentos:

| Segmento | Descripción                                                           | Fallback / Regla                                                                                  |
| -------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1        | Primera letra del nombre del cliente (ignorando espacios)             | “X” si no existe                                                                                  |
| 2        | Segunda letra del nombre del cliente (ignorando espacios)             | “X” si no existe                                                                                  |
| 3        | Tercera letra del nombre del cliente (ignorando espacios)             | “X” si el nombre sin espacios tiene menos de 3 caracteres (ej. “GP” → “GPX”)                     |
| 4        | Últimos 4 caracteres de la clave del cliente                          | Padding con ceros a la izquierda si la clave tiene menos de 4 caracteres                          |
| 5        | Código del banco (campo `Codigo` de la tabla `Bancos`)                | —                                                                                                 |
| 6        | Carácter de moneda                                                    | “P” si la primera letra del campo `Moneda` de la cuenta es “M” (peso); “D” en cualquier otro caso |
| 7        | Código Validador (campo `CodValidador` de la relación cliente-cuenta) | —                                                                                                 |

**Regla 8 — Identificación del banco “Banamex”**
La determinación de si una cuenta pertenece a Banamex se realiza cruzando la cuenta contra la tabla de Empresas: el campo beneficiario de la cuenta debe coincidir con el prefijo de la empresa que factura, y la moneda debe cumplir una condición adicional.

> **⚠️ Pendiente** — La condición adicional de moneda aparece truncada en la documentación del cliente y no se ha clarificado. Queda como duda formal de desarrollo. Como simplificación alternativa, se propone para evaluación con desarrollo identificar Banamex directamente por nombre o ID del banco en la tabla `Bancos`.

**Regla 9 — Edición sin restricción de rol**
Cualquier usuario con acceso a la cartera del cliente puede asignar cuentas bancarias y capturar el Código Validador. La autorización proviene del acceso del usuario al cliente, no de un rol específico.

> **⚠️ Pendiente** — Validar con el cliente si esta funcionalidad debe restringirse a un rol específico (probablemente Coordinador de Tesorería) por sus implicaciones financieras.

---

## Riesgos

**Riesgo 1 — Código Validador sin validación de formato ni longitud**
Como el Código Validador es input manual sin validación, un usuario puede capturar un valor erróneo (espacios, caracteres especiales, longitud incompatible con el sistema bancario) que rompa la identificación de pagos.

**Riesgo 2 — Sin restricción de rol sobre asignación de cuentas bancarias**
La asignación de cuentas bancarias del grupo PROQUIFA a clientes y la captura del Código Validador tienen implicaciones financieras serias (errores en estos datos comprometen la identificación de pagos). Sin embargo, esta funcionalidad se modela inicialmente sin restricción de rol.

> **⚠️ Pendiente** — Validar con el cliente si debe restringirse a un rol específico (probablemente Coordinador de Tesorería).

~~**Riesgo 3 — Modelo Perú no definido**~~ **[Resuelto — Duda FU-006/FU-017]**
~~La lógica documentada por el cliente corresponde exclusivamente a PROQUIFA México (Banamex, prefijos de empresas mexicanas, moneda en pesos/dólares). No existe documentación equivalente para el modelo bancario peruano de identificación de pagos. Mientras no se resuelva, los clientes Perú no podrán tener referencia bancaria armada por el sistema ProquifaDotNet.~~

> **Resolución:** Para clientes de Perú no existe mecanismo de Código Validador. La referencia bancaria se genera por default con la **Razón Social** del cliente (mismo camino que bancos no-Banamex, ver Regla 6-PER). No se requiere lógica adicional específica para Perú.

> **Riesgos retirados** — El riesgo de inconsistencia entre proformas re-emitidas por reconstrucción dinámica (antiguo Riesgo 1) queda descartado por OBS-013 (persistencia en dos niveles + snapshot inmutable en PDF). El riesgo de pérdida de trazabilidad por sobrescritura del Código Validador (antiguo Riesgo 5) queda descartado por el registro de cada cambio en ProquifaDotNet.BitacoraCambios (historial completo — actualización 2026-07-07, sustituye a OBS-014).

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

**Criterio B2 — Persistencia de la combinación cliente-cuenta-Código Validador y referencia vigente**
- **Dado** que el usuario guarda una asignación nueva o modificada,
- **Cuando** el sistema procesa la operación,
- **Entonces** deberá persistir la combinación en la relación cliente-cuenta del modelo de datos, incluida la **referencia bancaria armada vigente del cliente**, que solo se regenera ante un cambio de un dato fuente (banco, cuenta, Código Validador o datos del cliente). Al modificar el Código Validador, el sistema registra el cambio (valor anterior, valor nuevo, autor y fecha) en ProquifaDotNet.BitacoraCambios, conservando el historial completo. Este registro refleja únicamente cambios hechos desde el sistema y se conserva a nivel de datos, sin requerir visualización en pantalla en R16.

**Criterio B3 — Múltiples cuentas asignables por cliente**
- **Dado** que un cliente ya tiene una o más cuentas bancarias asignadas,
- **Cuando** el usuario agrega una cuenta adicional,
- **Entonces** el sistema deberá permitir la asignación. No existe límite máximo de cuentas asignables por cliente en R16.

> **⚠️ Pendiente** — Validar con el cliente si se requiere algún tope máximo.

**Criterio B4 — Edición y eliminación de combinaciones**
- **Dado** que un cliente tiene una cuenta bancaria asignada con su Código Validador,
- **Cuando** el usuario edita o elimina la asignación,
- **Entonces** el sistema deberá permitir la operación. Al editar el Código Validador, el sistema registra el cambio en ProquifaDotNet.BitacoraCambios conforme al Criterio B2 (historial completo). La eliminación retira la combinación cliente-cuenta del sistema y también se registra en BitacoraCambios.

**Criterio B5 — Edición sin restricción de rol específica**
- **Dado** que cualquier usuario con acceso a la cartera del cliente abre la pantalla “Referencia de Pago”,
- **Cuando** intenta modificar la asignación de cuentas o el Código Validador,
- **Entonces** el sistema deberá permitir la edición sin requerir un rol específico.

> **⚠️ Pendiente** — Validar con el cliente si debe restringirse al rol Coordinador de Tesorería u otro por las implicaciones financieras.

---

### SECCIÓN C — Generación y Casado de la Referencia Bancaria

**Criterio C1 — Casado de la referencia vigente al generar la proforma**
- **Dado** que un módulo genera una proforma para el cliente con una cuenta bancaria asignada,
- **Cuando** se incorpora la referencia bancaria al PDF de la proforma,
- **Entonces** el sistema deberá **tomar la referencia vigente del cliente y casarla al documento**: el PDF cae en firme y, al consultarse después, no se recalcula la referencia (equivalente al comportamiento del Legacy / Drobo — OBS-015).

**Criterio C2 — Referencia para bancos distintos de Banamex**
- **Dado** que el banco de la cuenta asignada para una proforma no es Banamex,
- **Cuando** el sistema construye la referencia bancaria,
- **Entonces** la referencia será el **Nombre del cliente** como cadena directa, sin transformación adicional.

**Criterio C3 — Referencia para Banamex (7 segmentos)**
- **Dado** que el banco de la cuenta asignada para una proforma es Banamex,
- **Cuando** el sistema construye la referencia bancaria,
- **Entonces** deberá concatenar determinísticamente los 7 segmentos definidos en la Regla 7: tres primeras letras del nombre del cliente **ignorando espacios** (ej. “BP Farmaceutica” → “BPF”) con fallback “X”, últimos 4 caracteres de la clave del cliente con padding de ceros, código del banco, carácter de moneda “P”/”D”, y Código Validador.

---

## Notas de Implementación

- Funcionalidad ubicada en la pantalla **“Referencia de Pago”**, sub-sección dentro de la sección Cobros del cliente en el Catálogo de Clientes.
- Funcionalidad **NUEVA en ProquifaDotNet R16**. Toma como referencia el comportamiento documentado del sistema Legacy actual de PROQUIFA, pero su implementación en ProquifaDotNet es desde cero.
- El modelo de datos involucra una relación **N:N** entre `Cliente` y `DatosBancarios` mediante una tabla cliente-cuenta que persiste el Código Validador y la referencia bancaria vigente por combinación.
- La referencia bancaria armada se persiste en **dos niveles** (confirmado en Sesión Cliente 1 — OBS-013): (1) como referencia **vigente del cliente** en la tabla `ClienteDatosBancarios`, regenerada solo ante cambio de un dato fuente; y (2) como referencia **casada al PDF** de cada proforma en firme (snapshot inmutable). Las proformas ya emitidas conservan su referencia y no se recalculan al consultarse, equivalente al Legacy (Drobo — OBS-015). Esto descarta el antiguo riesgo de inconsistencia entre proformas.
- La lógica de Banamex (7 segmentos) y no-Banamex (nombre del cliente) replica 1:1 el comportamiento documentado por el cliente en el documento *“Especificación: Proceso para generar Referencia de Cliente (Código Validador)”* recibido el 2026-04-28.
- La funcionalidad provee insumos al módulo **Buzón de Cobros** (identificación automática de pagos entrantes contra la referencia armada). La integración entre ambos módulos se detalla en los requisitos correspondientes al Buzón de Cobros.

> **⚠️ Pendiente** — La condición de identificación de Banamex referente al campo moneda de la cuenta aparece truncada en el documento del cliente y no se ha clarificado. Queda como duda formal de desarrollo. Se propone evaluar con desarrollo identificar Banamex directamente por nombre o ID en la tabla `Bancos` como simplificación.

> **Resuelto** — El Código Validador es **numérico, siempre 2 dígitos con cero a la izquierda** (rango `01`–`99`). El Front enviará exactamente 2 caracteres; la columna en BD puede tener mayor capacidad pero el valor nunca la excederá.

> **⚠️ Pendiente** — Aplicabilidad de esta funcionalidad para clientes Perú. La documentación del cliente cubre exclusivamente PROQUIFA México. El modelo bancario peruano de identificación de pagos no está definido. Queda como duda formal del proyecto.

> **⚠️ Pendiente** — Validar con el cliente si la asignación de cuentas bancarias y captura del Código Validador debe restringirse al rol Coordinador de Tesorería u otro rol específico, considerando las implicaciones financieras.

> **⚠️ Pendiente** — Confirmar con el cliente si es posible asignar más de una cuenta bancaria por cliente y si se requiere tope máximo.

---

## Cambios

| #   | Fecha      | Observación | Descripción del cambio                                                                                                                                                                   |
| --- | ---------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | 2026-06-10 | OBS-010     | Se agrega viñeta en Alcance "Aplica a": migración de datos actuales del sistema Legacy (cuentas bancarias y Códigos Validadores por cliente) como parte de los entregables de R16.       |
| 2   | 2026-06-10 | OBS-012     | Confirmado / ya cubierto: la proforma solo se genera en Tramitar Pedido, no en Validar Cobro. Diseño vigente ya era correcto; sin cambios en la matriz.                                  |
| 3   | 2026-06-10 | OBS-016     | Entregable: el cliente solicita mockups/pantallas del Catálogo de Clientes (sección Cobros, código validador). No genera cambio en la matriz. Pendiente producir el diseño de pantallas. |
| 4   | 2026-06-09 | OBS-013     | Persistencia aplicada en dos niveles: (1) referencia vigente del cliente en `ClienteDatosBancarios`, regenerada solo ante cambio de dato fuente; (2) referencia casada al PDF de cada proforma en firme (snapshot inmutable). Tocado: Requisito, Alcance, Reglas 4 y 5, Criterios B2 y C1, título Sección C, Riesgos (retira antiguo Riesgo 1), Notas de Implementación. |
| 5   | 2026-06-10 | OBS-014     | Historial de un nivel del Código Validador: al modificar, se conservan el valor actual y el inmediatamente anterior con autor y fecha; el “anterior” previo se sobrescribe. Solo a nivel de datos, sin UI. Tocado: Alcance (No aplica a), Regla 4, Criterios B2 y B4, Riesgos (retira antiguo Riesgo 5). |
| 6   | 2026-07-07 | BitacoraCambios | El historial del Código Validador se integra al Aplicativo Nuevo ProquifaDotNet.BitacoraCambios (regla 8): cada cambio se registra con valor anterior/nuevo, autor y fecha (historial completo). Sustituye la rotación de un nivel de OBS-014 y elimina las columnas `CodigoValidadorAnterior`/`FechaModificacionAnterior`/`IdUsuarioModificacionAnterior` de `ClienteDatosBancarios`. Tocado: Alcance (No aplica a), Regla 4, Criterios B2 y B4, Riesgos. |
| 7   | 2026-06-09 | OBS-015     | Confirmado: el PDF de la proforma se almacena y al consultarlo NO se reconstruye la referencia (equivalente al Legacy/Drobo). Se elimina el término "proformas re-emitidas". Tocado: Alcance (No aplica a), Reglas 4 y 5, Criterio C1, Notas de Implementación. |
| 8   | 2026-08-05 | BUG-001     | Corrección de bug en Regla 7: los segmentos 1-3 deben extraerse ignorando espacios en el nombre del cliente. Ejemplo: "BP Farmaceutica" debe producir "BPF", no "BP ". Tocado: Regla 7 (tabla de segmentos), Criterio C3. |
