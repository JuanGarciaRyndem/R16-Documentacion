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

> Yo como usuario con acceso a la cartera de clientes, quiero asignar a cada cliente una o más combinaciones de **Empresa → Banco → Cuenta bancaria** del grupo PROQUIFA y capturar el **Código Validador** cuando la cuenta es de Banamex, para que el sistema construya automáticamente la **referencia bancaria** que aparece en las **proformas** y **facturas** emitidas al cliente, permitiendo identificar correctamente los pagos recibidos en cada cuenta.

Para clientes de la **Región Perú** —donde no existe mecanismo de Código Validador— la referencia se genera con el **nombre del cliente** por default y se muestra igualmente en los documentos emitidos.

---

## Requisito

El sistema debe contar en la sección **Cobros** del Catálogo de Clientes con una sub-sección **"Referencia de Pago"** que permita asignar al cliente una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA. La selección arranca en la **Empresa** emisora del grupo, que determina los **Bancos** disponibles, y estos a su vez las **Cuentas** seleccionables. Cada combinación cliente-cuenta es **única**: la misma cuenta no puede asignarse dos veces al mismo cliente.

El **Código Validador** es obligatorio únicamente cuando la cuenta pertenece a **Banamex**. Para cualquier otro banco el campo permanece bloqueado y el sistema establece la **referencia bancaria** automáticamente con la **razón social del cliente**.

La referencia bancaria se arma y persiste como **referencia vigente del cliente** al configurar su cuenta y se **casa al documento** al generar el PDF de una **proforma** o **factura** emitida al cliente; los documentos ya emitidos conservan su referencia. La funcionalidad es nueva en ProquifaDotNet R16 y toma como referencia operativa el comportamiento equivalente del sistema Legacy actual.

**Región Perú:** no existe Código Validador. La referencia se genera por default con la **razón social del cliente** y se muestra en los documentos emitidos siguiendo el mismo camino que las cuentas de bancos distintos de Banamex.

---

## Alcance

### Aplica a

- Clientes de **México** y **Perú** en el Catálogo de Clientes, con mecanismos diferenciados:
  - **México:** cuentas de Banamex requieren Código Validador; cuentas de otros bancos usan la razón social del cliente.
  - **Perú:** sin Código Validador; referencia por default con razón social del cliente.
- Pantalla **"Referencia de Pago"** como sub-sección dentro de la sección Cobros del cliente.
- **Encadenamiento de selectores desde la Empresa emisora:** selección de Empresa del grupo → filtra los Bancos disponibles → filtra las Cuentas del catálogo del grupo. La **Moneda** y la **Sucursal** se heredan como datos de solo lectura de la cuenta seleccionada.
- Asignación de una o más cuentas bancarias del catálogo de cuentas del grupo PROQUIFA al cliente, con **unicidad garantizada** por combinación cliente-cuenta (una misma cuenta no puede asignarse dos veces al mismo cliente).
- Captura del **Código Validador** obligatoria **únicamente** cuando la cuenta es de Banamex. Para cualquier otro banco el campo permanece bloqueado y el sistema establece la referencia bancaria automáticamente con la razón social del cliente.
- Generación de la **referencia bancaria vigente** al configurar/actualizar la cuenta del cliente y **casado** de esa referencia al PDF de cada **proforma** y **factura** en firme emitida al cliente.
- Replicación de la lógica documentada del sistema Legacy actual sobre Banamex (7 segmentos) y no-Banamex (razón social).
- Migración de los datos actuales del sistema Legacy a ProquifaDotNet: cuentas bancarias asignadas por cliente y sus Códigos Validadores (OBS-010).

### No aplica a

- Captura del Código Validador para clientes de Perú ni para cuentas de bancos distintos de Banamex (el campo permanece bloqueado y la referencia se establece automáticamente con la razón social del cliente).
- Asignación repetida de una misma cuenta al mismo cliente (la combinación cliente-cuenta es única).
- Recálculo de la referencia ya casada a una proforma o factura emitida (el PDF cae en firme; los documentos históricos conservan su referencia y no se regeneran al consultarse — OBS-015).
- Vista en pantalla del historial del Código Validador: se conservan el **valor actual (vigente)** y el **inmediatamente anterior** con autor y fecha; ante una nueva modificación el "actual" pasa a "anterior" y el anterior previo se sobrescribe. Solo registra cambios hechos desde el sistema y se conserva a nivel de datos, sin componente de UI en R16 (Riesgo previo de pérdida de trazabilidad eliminado).
- Restricción por rol de la asignación de cuentas bancarias y captura del Código Validador (queda fuera del alcance de R16).

---

## Reglas de Negocio

**Regla 1 — Pantalla "Referencia de Pago" en sección Cobros**
El Catálogo de Clientes cuenta con una sub-sección **"Referencia de Pago"** dentro de la sección Cobros, donde se gestionan las combinaciones Empresa-Banco-Cuenta asignadas al cliente y su Código Validador correspondiente (cuando aplica).

**Regla 2 — Encadenamiento de selectores desde Empresa y unicidad cliente-cuenta**
Cada asignación se compone seleccionando primero la **Empresa** emisora del grupo PROQUIFA (que determina los Bancos disponibles), luego el **Banco** (del catálogo de Bancos filtrado por empresa) y finalmente la **Cuenta** (del catálogo de cuentas del grupo filtrado por banco). La **Moneda** y la **Sucursal** se heredan como datos de solo lectura de la cuenta seleccionada. La combinación cliente-cuenta es **única**: una misma cuenta bancaria no puede asignarse dos veces al mismo cliente.

**Regla 3 — Código Validador condicionado al banco (solo Banamex)**
La captura del Código Validador aplica exclusivamente cuando la cuenta seleccionada pertenece a **Banamex**. En ese caso el campo es obligatorio. Para cuentas de cualquier otro banco (México no-Banamex o Perú) el campo permanece **bloqueado** y el sistema no solicita ni acepta captura del Código Validador.

> **Resuelto (DUDA-015)** — El Código Validador es **alfanumérico, con longitud máxima de 3 caracteres**, sin acentos ni espacios en blanco (restricción de negocio/Frontend). A nivel de base de datos el campo se reserva con longitud máxima de 50 caracteres para compatibilidad y futuras extensiones, sin cambiar esta regla de captura (ver `R16A-RE-FU-006_BD.md` y `DIS_INT_006.md`).

**Regla 4 — Persistencia en dos niveles: referencia vigente del cliente y referencia casada al PDF de proformas y facturas**
La referencia bancaria se persiste en dos niveles:

1. **Referencia vigente del cliente:** la combinación cliente-cuenta persiste en el modelo de datos el identificador de la cuenta bancaria, el identificador del cliente, el Código Validador (cuando aplica) y la **referencia bancaria armada vigente** construida con la **razón social del cliente**. Esta referencia se genera una sola vez al configurar la cuenta del cliente (Catálogo de Clientes → Cobros → Referencia de Pago) y **solo se regenera si cambia un dato fuente** (empresa, banco, cuenta, Código Validador, razón social o clave del cliente).
2. **Referencia casada al PDF de proformas y facturas:** al generarse el PDF de una **proforma** o **factura** emitida al cliente, la referencia vigente en ese momento queda casada al documento; los documentos ya emitidos conservan su referencia y no se ven afectados por regeneraciones posteriores de la referencia vigente del cliente.

Adicionalmente, al modificar el Código Validador el sistema conserva a nivel de datos el valor **actual (vigente)** y el **inmediatamente anterior** con autor y fecha (ante una nueva modificación el "actual" pasa a "anterior" y el anterior previo se sobrescribe). Este historial se registra a nivel de datos y no requiere visualización en pantalla en R16.

**Regla 5 — Generación de la referencia y casado al PDF en firme de proformas y facturas**
La referencia bancaria se arma al configurar/actualizar la cuenta del cliente, aplicando las reglas según el banco de la cuenta (ver Reglas 6, 7 y 9), y se persiste como referencia vigente del cliente. Al generar el PDF de una **proforma** o de una **factura** emitida al cliente, el sistema toma la **referencia vigente del cliente y la casa al documento**: el PDF cae en firme y, al consultarse después, no se reconstruye ni recalcula la referencia (equivalente al comportamiento del Legacy — OBS-015).

**Regla 6 — Referencia automática para bancos distintos de Banamex**
Para cuentas de bancos distintos de Banamex (México no-Banamex), el sistema establece automáticamente la referencia bancaria con la **razón social del cliente** como cadena directa, sin transformación adicional. El Código Validador no se solicita para estas cuentas.

**Regla 7 — Referencia para Banamex (concatenación de 7 segmentos)**
Para cuentas de Banamex, la referencia bancaria se compone por la concatenación determinista de 7 segmentos:

| Segmento | Descripción                                                                                                       | Fallback / Regla                                                                                  |
| -------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1        | Primera letra de la razón social del cliente (ignorando espacios)                                                 | "X" si no existe                                                                                  |
| 2        | Segunda letra de la razón social del cliente (ignorando espacios)                                                 | "X" si no existe                                                                                  |
| 3        | Tercera letra de la razón social del cliente (ignorando espacios)                                                 | "X" si la razón social sin espacios tiene menos de 3 caracteres (ej. "GP" → "GPX")                |
| 4        | Últimos 4 caracteres del campo **Clave** de la tabla de clientes                                                  | Padding con ceros a la izquierda si la Clave tiene menos de 4 caracteres (ej. Clave `42` → `0042`; Clave `7` → `0007`) |
| 5        | Código del banco (campo `Codigo` de la tabla `Bancos`)                                                            | —                                                                                                 |
| 6        | Carácter de moneda                                                                                                | "P" si la primera letra del campo `Moneda` de la cuenta es "M" (peso); "D" en cualquier otro caso |
| 7        | Código Validador (campo `CodValidador` de la relación cliente-cuenta)                                             | —                                                                                                 |

**Regla 8 — Identificación de cuentas de Banamex y alcance del encadenamiento**
La determinación de si una cuenta pertenece a Banamex parte del encadenamiento **Empresa → Banco → Cuenta**: la empresa emisora del grupo determina los bancos disponibles, y el banco elegido (Banamex u otro) activa o bloquea la captura del Código Validador. El filtrado de cuentas seleccionables aplica las tres condiciones confirmadas por el cliente: (1) la cuenta debe pertenecer a la empresa seleccionada; (2) la cuenta debe pertenecer al banco seleccionado; (3) la moneda de la cuenta debe corresponder a la condición operativa vigente (**DUDA-016**: la condición de moneda para identificar Banamex, antes truncada en la documentación recibida del cliente, quedó completada y cerrada por el cliente en documento aparte).

**Regla 9 — Referencia para clientes de Región Perú (sin Código Validador)**
Para clientes de la Región Perú no existe mecanismo de Código Validador. El sistema establece automáticamente la referencia bancaria con la **razón social del cliente** como cadena directa, sin transformación adicional (mismo camino que la Regla 6 para bancos no-Banamex). La referencia armada se casa al PDF de las proformas y facturas emitidas al cliente siguiendo la Regla 5.

---

## Riesgos

**Riesgo 1 — Código Validador sin validación estricta puede romper la identificación de pagos**
Aunque el Código Validador queda acotado a 3 caracteres alfanuméricos sin acentos ni espacios, el input manual siempre expone a errores humanos que rompan la identificación de pagos. Mitigación: capacitación operativa y trazabilidad por auditoría (valor anterior conservado).

**Riesgo 2 — Sin restricción de rol sobre la asignación de cuentas bancarias**
La asignación de cuentas del grupo PROQUIFA a clientes y la captura del Código Validador tienen implicaciones financieras. En R16 esta funcionalidad se implementa sin restricción de rol. **Resuelto (DUDA-017)** — el cliente desestimó esta restricción de rol (p. ej. Coordinador de Tesorería) para R16: este tipo de control de roles corresponde al alcance del release **R7**, no de R16.

> **Riesgos retirados** — El riesgo de inconsistencia entre proformas re-emitidas por reconstrucción dinámica queda descartado por OBS-013 (persistencia en dos niveles + snapshot inmutable en PDF). El riesgo del modelo Perú no definido queda cerrado por la respuesta del cliente: Perú usa razón social por default sin Código Validador (Regla 9). El riesgo de pérdida de trazabilidad por sobrescritura del Código Validador queda descartado por el historial de dos niveles (valor actual + inmediatamente anterior con autor y fecha).

---

## Criterios de Aceptación

### SECCIÓN A — Acceso y Encadenamiento de Selectores

**Criterio A1 — Acceso a pantalla "Referencia de Pago" desde sección Cobros**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta un cliente,
- **Cuando** navega a la sección Cobros,
- **Entonces** el sistema deberá ofrecer acceso a la sub-sección **"Referencia de Pago"** para gestionar las cuentas bancarias asignadas al cliente y sus Códigos Validadores.

**Criterio A2 — Selector de Empresa emisora**
- **Dado** que un usuario en "Referencia de Pago" agrega una cuenta nueva,
- **Cuando** despliega el selector de Empresa,
- **Entonces** el sistema deberá presentar las opciones del catálogo de Empresas del grupo PROQUIFA que emiten cobros al cliente. La selección de Empresa determina los Bancos disponibles en el siguiente selector.

**Criterio A3 — Selector de Banco filtrado por Empresa**
- **Dado** que el usuario seleccionó una Empresa en "Referencia de Pago",
- **Cuando** despliega el selector de Banco,
- **Entonces** el sistema deberá presentar únicamente los Bancos del catálogo que tengan al menos una cuenta asociada a la Empresa seleccionada.

**Criterio A4 — Selector de Cuenta filtrado por Empresa y Banco**
- **Dado** que el usuario seleccionó una Empresa y un Banco en "Referencia de Pago",
- **Cuando** despliega el selector de Cuenta,
- **Entonces** el sistema deberá presentar únicamente las cuentas bancarias del catálogo del grupo PROQUIFA que correspondan a la combinación Empresa-Banco y que no estén ya asignadas al cliente. Al seleccionar la Cuenta, la **Moneda** y la **Sucursal** se heredan como datos de solo lectura.

**Criterio A5 — Moneda y Sucursal autopobladas desde la cuenta seleccionada**
- **Dado** que el usuario seleccionó una Cuenta en "Referencia de Pago",
- **Cuando** se renderizan los campos Moneda y Sucursal,
- **Entonces** el sistema deberá autopoblarlos con los valores de los campos correspondientes de la cuenta, en modo **solo lectura** (no editables por el usuario en esta pantalla).

---

### SECCIÓN B — Código Validador, Unicidad y Persistencia

**Criterio B1 — Captura obligatoria del Código Validador para cuentas de Banamex**
- **Dado** que el usuario configura una combinación cliente-cuenta y la cuenta pertenece a **Banamex**,
- **Cuando** captura el campo Código Validador,
- **Entonces** el sistema deberá aceptar el valor como input manual del usuario (alfanumérico, máximo 3 caracteres, sin acentos ni espacios en blanco). El campo es obligatorio para guardar la asignación.

**Criterio B2 — Código Validador bloqueado para cuentas de otros bancos**
- **Dado** que el usuario configura una combinación cliente-cuenta y la cuenta pertenece a un banco **distinto de Banamex** (México no-Banamex o Perú),
- **Cuando** se renderiza el campo Código Validador,
- **Entonces** el sistema deberá presentarlo en modo **bloqueado** (no editable) y establecerá automáticamente la referencia bancaria con la razón social del cliente al guardar.

**Criterio B3 — Persistencia de la combinación cliente-cuenta-Código Validador y referencia vigente**
- **Dado** que el usuario guarda una asignación nueva o modificada,
- **Cuando** el sistema procesa la operación,
- **Entonces** deberá persistir la combinación en la relación cliente-cuenta del modelo de datos, incluida la **referencia bancaria armada vigente del cliente**, que solo se regenera ante un cambio de un dato fuente (empresa, banco, cuenta, Código Validador, razón social o clave del cliente). Al modificar el Código Validador el sistema conserva el valor anterior (autor y fecha) siguiendo el patrón de dos niveles descrito en la Regla 4.

**Criterio B4 — Asignación de múltiples cuentas distintas por cliente**
- **Dado** que un cliente ya tiene una o más cuentas bancarias asignadas,
- **Cuando** el usuario agrega una cuenta adicional distinta de las ya asignadas,
- **Entonces** el sistema deberá permitir la asignación. No existe límite máximo de cuentas asignables por cliente en R16.

**Criterio B5 — Unicidad de la combinación cliente-cuenta**
- **Dado** que un cliente tiene una cuenta asignada,
- **Cuando** el usuario intenta asignar la misma cuenta al mismo cliente por segunda vez,
- **Entonces** el sistema deberá **bloquear la operación** y notificar al usuario que la combinación cliente-cuenta ya existe. El selector de Cuenta (Criterio A4) filtra las cuentas ya asignadas para prevenir el intento.

**Criterio B6 — Edición y eliminación de combinaciones**
- **Dado** que un cliente tiene una cuenta bancaria asignada,
- **Cuando** el usuario edita o elimina la asignación,
- **Entonces** el sistema deberá permitir la operación. Al editar el Código Validador se aplica el patrón de historial de dos niveles (Regla 4). La eliminación retira la combinación cliente-cuenta del sistema y **libera la cuenta** para poder volver a asignarse al mismo cliente en el futuro.

**Criterio B7 — Edición sin restricción de rol específica**
- **Dado** que cualquier usuario con acceso a la cartera del cliente abre la pantalla "Referencia de Pago",
- **Cuando** intenta modificar la asignación de cuentas o el Código Validador,
- **Entonces** el sistema deberá permitir la edición sin requerir un rol específico (la restricción por rol queda fuera del alcance de R16).

---

### SECCIÓN C — Generación y Casado de la Referencia Bancaria

**Criterio C1 — Casado de la referencia vigente al generar la proforma o factura**
- **Dado** que un módulo genera una **proforma** o una **factura** para el cliente con una cuenta bancaria asignada,
- **Cuando** se incorpora la referencia bancaria al PDF del documento,
- **Entonces** el sistema deberá **tomar la referencia vigente del cliente y casarla al documento**: el PDF cae en firme y, al consultarse después, no se recalcula la referencia (equivalente al comportamiento del Legacy / Drobo — OBS-015).

**Criterio C2 — Referencia para bancos distintos de Banamex (México)**
- **Dado** que el banco de la cuenta asignada para una proforma o factura no es Banamex,
- **Cuando** el sistema construye la referencia bancaria,
- **Entonces** la referencia será la **razón social del cliente** como cadena directa, sin transformación adicional.

**Criterio C3 — Referencia para Banamex (7 segmentos)**
- **Dado** que el banco de la cuenta asignada para una proforma o factura es Banamex,
- **Cuando** el sistema construye la referencia bancaria,
- **Entonces** deberá concatenar determinísticamente los 7 segmentos definidos en la Regla 7: tres primeras letras de la razón social del cliente **ignorando espacios** (ej. "BP Farmaceutica" → "BPF") con fallback "X", últimos 4 caracteres del campo **Clave** del cliente con padding de ceros (ej. Clave `42` → `0042`), código del banco, carácter de moneda "P"/"D", y Código Validador.

**Criterio C4 — Referencia para clientes de Región Perú**
- **Dado** que el cliente es de la Región Perú,
- **Cuando** el sistema construye la referencia bancaria para su proforma o factura,
- **Entonces** la referencia será la **razón social del cliente** como cadena directa, sin transformación adicional. No se captura Código Validador para estos clientes (Regla 9).

---

## Notas de Implementación

- Funcionalidad ubicada en la pantalla **"Referencia de Pago"**, sub-sección dentro de la sección Cobros del cliente en el Catálogo de Clientes.
- Funcionalidad **NUEVA en ProquifaDotNet R16**. Toma como referencia el comportamiento documentado del sistema Legacy actual de PROQUIFA, pero su implementación en ProquifaDotNet es desde cero.
- El modelo de datos involucra una relación **N:N** entre `Cliente` y `DatosBancarios` mediante una tabla cliente-cuenta que persiste el Código Validador (cuando aplica) y la referencia bancaria vigente por combinación, con **restricción de unicidad** sobre la combinación cliente-cuenta.
- **Encadenamiento de selectores:** la pantalla arranca en el selector de **Empresa** del grupo, que determina los **Bancos** disponibles; el Banco elegido determina las **Cuentas** seleccionables; al seleccionar la Cuenta se autopopulan **Moneda** y **Sucursal** como datos de solo lectura.
- El **Código Validador** aplica exclusivamente a Banamex. Para cualquier otra cuenta (México no-Banamex o Perú) el campo permanece bloqueado y el sistema genera automáticamente la referencia con la razón social del cliente.
- La referencia bancaria armada se persiste en **dos niveles** (confirmado en Sesión Cliente 1 — OBS-013): (1) como referencia **vigente del cliente** en la tabla `ClienteDatosBancarios`, regenerada solo ante cambio de un dato fuente; y (2) como referencia **casada al PDF** de cada **proforma** y **factura** en firme (snapshot inmutable). Los documentos ya emitidos conservan su referencia y no se recalculan al consultarse, equivalente al Legacy (Drobo — OBS-015).
- **Historial del Código Validador:** al modificar, se conservan el **valor actual (vigente)** y el **inmediatamente anterior** con autor y fecha; ante una nueva modificación el "actual" pasa a "anterior" y el anterior previo se sobrescribe. Solo registra cambios hechos desde el sistema y se conserva a nivel de datos, sin componente de UI en R16.
- **Unicidad de la combinación cliente-cuenta:** el modelo impone que una misma cuenta bancaria no puede asignarse dos veces al mismo cliente. El selector de Cuenta (Criterio A4) filtra las cuentas ya asignadas para prevenir intentos; el guardado también rechaza duplicados.
- La lógica de Banamex (7 segmentos) replica 1:1 el comportamiento documentado por el cliente en el documento *"Especificación: Proceso para generar Referencia de Cliente (Código Validador)"* recibido el 2026-04-28.
- La funcionalidad provee insumos al módulo **Buzón de Cobros** (identificación automática de pagos entrantes contra la referencia armada). La integración entre ambos módulos se detalla en los requisitos correspondientes al Buzón de Cobros.

**Resueltos (dudas cerradas):**
- **Código Validador (DUDA-015):** longitud máxima 3 caracteres, alfanumérico, sin acentos ni espacios en blanco (regla de negocio/Frontend); a nivel de base de datos el campo se reserva con longitud máxima de 50 caracteres para compatibilidad y futuras extensiones.
- **Condición de moneda para identificación de Banamex (DUDA-016):** cerrada con las tres condiciones confirmadas por el cliente (empresa, banco, moneda — ver Regla 8); el texto original recibido del cliente sobre esta condición estaba truncado y quedó completado.
- **Restricción por rol (DUDA-017):** fuera del alcance de R16 — corresponde al release R7.
- **Aplicabilidad Perú (DUDA-018):** sin Código Validador; referencia por default con razón social del cliente (Regla 9).
- **Tope de cuentas asignables por cliente (DUDA-118):** no se limita.
- **Trazabilidad del Código Validador (DUDA-120):** se conservan a nivel de datos el valor actual (vigente) y el inmediatamente anterior con autor y fecha; no se requiere bitácora completa ni vista en pantalla en R16.
- **Clave del cliente (DUDA-122):** se mantienen las claves del sistema Legacy; los clientes Legacy ya se vinculan con PQF2 en la base de datos intermedia y existe un procedimiento (`spObtenerClienteLegacyId`, desarrollado por Armando) para obtener la Clave Legacy a partir del ID de Cliente de PQF2 — corresponde al campo `Clave`/`ClienteLegacy` usado en el segmento 4 de la referencia Banamex, con relleno de ceros a la izquierda si tiene menos de 4 caracteres (Regla 7 segmento 4).
- **Razón social del cliente:** dato empleado al construir la referencia bancaria (Reglas 4, 6, 7 y 9; Criterios B2, B3, C2, C3 y C4).

---

## Cambios

| #   | Fecha      | Observación | Descripción del cambio                                                                                                                                                                   |
| --- | ---------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | 2026-06-10 | OBS-010     | Se agrega viñeta en Alcance "Aplica a": migración de datos actuales del sistema Legacy (cuentas bancarias y Códigos Validadores por cliente) como parte de los entregables de R16.       |
| 2   | 2026-06-10 | OBS-012     | Confirmado / ya cubierto: la proforma solo se genera en Tramitar Pedido, no en Validar Cobro. Diseño vigente ya era correcto; sin cambios en la matriz.                                  |
| 3   | 2026-06-10 | OBS-016     | Entregable: el cliente solicita mockups/pantallas del Catálogo de Clientes (sección Cobros, código validador). No genera cambio en la matriz. Pendiente producir el diseño de pantallas. |
| 4   | 2026-06-09 | OBS-013     | Persistencia aplicada en dos niveles: (1) referencia vigente del cliente en `ClienteDatosBancarios`, regenerada solo ante cambio de dato fuente; (2) referencia casada al PDF de cada proforma en firme (snapshot inmutable). |
| 5   | 2026-06-10 | OBS-014     | Historial de un nivel del Código Validador: al modificar, se conservan el valor actual y el inmediatamente anterior con autor y fecha; el "anterior" previo se sobrescribe. Solo a nivel de datos, sin UI. |
| 6   | 2026-06-09 | OBS-015     | Confirmado: el PDF de la proforma se almacena y al consultarlo NO se reconstruye la referencia (equivalente al Legacy/Drobo). Se elimina el término "proformas re-emitidas". |
| 7   | 2026-08-05 | BUG-001     | Corrección de bug en Regla 7: los segmentos 1-3 deben extraerse ignorando espacios en el nombre del cliente. Ejemplo: "BP Farmaceutica" debe producir "BPF", no "BP ". |
| 8   | 2026-08-05 | Incorporación de Perú | Se incorpora el mecanismo de Región Perú: referencia con razón social del cliente sin Código Validador. Alcance pasa a aplicar a México y Perú con mecanismos diferenciados. Se agrega Regla 9 (Perú) y Criterio C4. Se elimina el riesgo del modelo Perú no definido; los riesgos restantes se renumeran como Riesgo 1 y Riesgo 2. Se cierra el pendiente de aplicabilidad Perú. |
| 9   | 2026-08-05 | Alcance a proformas y facturas | Historia, Requisito, Alcance y Observaciones: se precisa que la referencia bancaria se muestra tanto en proformas como en facturas emitidas al cliente. Reglas 4, 5 y 9 extienden el casado al PDF de la factura. Criterios C1–C4 extendidos a la generación de facturas. |
| 10  | 2026-08-05 | Encadenamiento desde Empresa | Se incorpora la selección de Empresa como punto de partida del encadenamiento (Empresa → Banco → Cuenta) con herencia de Moneda y Sucursal. Regla 2 reescrita con el encadenamiento completo. Regla 8 reescrita para reflejar que el encadenamiento parte de la empresa con tres condiciones confirmadas. Sección A agrega A2 (Empresa) y A3 (Banco filtrado por empresa); A4 y A5 renumerados con Moneda heredada. |
| 11  | 2026-08-05 | CV condicionado a Banamex | Alcance precisa que el Código Validador se solicita y es obligatorio únicamente cuando la cuenta es de Banamex, bloqueado para otros bancos. Regla 3 reescrita para condicionar la captura al banco. Regla 6 precisa el establecimiento automático de referencia para no-Banamex. Sección B desdoblada: B1 Banamex obligatorio, B2 otros bancos bloqueado; criterios siguientes renumerados a partir de B3. |
| 12  | 2026-08-05 | Unicidad cliente-cuenta | Alcance precisa la unicidad de la combinación cliente-cuenta y excluye la asignación repetida en "No aplica a". Regla 2 incorpora la unicidad. Criterio B4 acotado a cuentas distintas. Criterio B5 nuevo (unicidad); criterios renumerados como B6 y B7. Criterio B6 precisa que la eliminación libera la cuenta para volver a asignarse. Observaciones agregan el bullet de unicidad. |
| 13  | 2026-08-05 | Cierres de dudas | Se cierran los pendientes de: longitud CV (3 caracteres alfanumérico sin acentos ni espacios), tope de cuentas asignables por cliente (no se limita), restricción de rol (fuera de R16), condición de moneda truncada (tres condiciones confirmadas). Se corrige la exclusión del historial del CV en "No aplica a": se conservan el valor vigente y el inmediatamente anterior con autor y fecha. Regla 7 segmento 4 precisa que la clave del cliente corresponde al campo `Clave` de la tabla y se ejemplifica el relleno de ceros. |
| 14  | 2026-08-07 | Cierres de dudas | Reglas 4, 6, 7 y 9 y Criterios B2, B3, C2, C3 y C4: se define la **razón social del cliente** como el dato empleado al construir la referencia bancaria. Regla 7 segmento 4 y Reglas 4 y 9: se precisa que el cliente cuenta con una clave que forma parte de la composición de la referencia. Observaciones alineadas. |
| 15  | 2026-08-21 | Trazabilidad DUDA-015/016/017/018/118/120/122 | Se citan explícitamente en Regla 3, Regla 8, Riesgo 2 y "Resueltos" los números de duda cerrada correspondientes a: longitud/formato del Código Validador (DUDA-015, incluida precisión de longitud 50 en BD vs. 3 en captura), condición de moneda para Banamex (DUDA-016), restricción de rol diferida a R7 (DUDA-017), aplicabilidad Perú (DUDA-018), tope de cuentas por cliente (DUDA-118), trazabilidad de dos niveles del Código Validador (DUDA-120) y mantenimiento de la clave de cliente Legacy vía procedimiento de vinculación (DUDA-122). Sin cambios de contenido funcional; se corrigen además inconsistencias de formato del Código Validador encontradas en `DIS_INT_006.md` y `R16A-RE-FU-006-Cambio.md` (decían "numérico, 2 dígitos, 01–99"). |
