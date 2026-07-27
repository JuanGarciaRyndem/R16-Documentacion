# R16A-RE-FU-000 — Perfil Fiscal

## Información general

| Campo                             | Valor          |
| --------------------------------- | -------------- |
| **ID**                            | R16A-RE-FU-000 |
| **ID Padre**                      | —              |
| **Épica**                         | Perfil Fiscal  |
| **Módulo**                        | —              |
| **Tipo**                          | Funcional      |
| **Versión**                       | —              |
| **Status**                        | Propuesto      |
| **Fecha de Status**               | —              |
| **Prioridad**                     | —              |
| **Dueño del Requisito (cliente)** | —              |
| **Complejidad**                   | —              |
| **ID Cliente**                    | —              |

## Historia de Usuario

Yo como sistema PQF2, quiero contar con la configuración fiscal de los productos —la clave de producto/servicio del SAT, la clave de unidad y el perfil fiscal de IVA— definida a nivel Familia de productos, para que al armar y timbrar el CFDI mexicano cada partida reporte las claves correctas que exige el SAT sin configurar producto por producto.

## Requisito

El sistema debe contar con la configuración fiscal que cada producto necesita para timbrarse en México, compuesta por tres datos: la clave de producto/servicio del SAT, la clave de unidad de medida del SAT y el perfil fiscal de IVA (tasa 16%, tasa 0% o exento). Esta configuración se define a nivel Familia de productos: todos los productos de una misma familia comparten la misma configuración fiscal.

El perfil fiscal de IVA se apoya en un modelo de catálogos maestros del SAT (impuesto, tipo de factor y objeto de impuesto) que no se manipulan directamente; el perfil fiscal se resuelve como una opción acotada que referencia esos catálogos, sin exponer sus claves técnicas.

La gestión de esta configuración (asignación de claves y perfil fiscal a las familias, y carga de los catálogos maestros) no dispone de interfaz gráfica en R16: se realiza directamente en la base de datos por el área responsable.

## Alcance

### Aplica a

- Configuración fiscal de producto para México, compuesta por: clave de producto/servicio del SAT, clave de unidad de medida del SAT y perfil fiscal de IVA (16%, 0%, exento).
- Definición a nivel Familia de productos: todos los productos de una misma familia comparten la misma configuración fiscal.
- Perfil fiscal de IVA como catálogo de negocio acotado (IVA 16%, IVA 0%, Exento), donde cada opción referencia a los catálogos maestros del SAT correspondientes, sin exponer las claves técnicas del SAT.
- Catálogos maestros del SAT (impuesto, tipo de factor, objeto de impuesto) como datos oficiales precargados, que no se modifican.
- Gestión de la configuración (asignación de claves y perfil fiscal a las familias, carga de catálogos) realizada directamente en base de datos por el área responsable, sin interfaz gráfica en R16.
- Mapeo de familias a claves del SAT conforme a las definiciones entregadas por PROQUIFA (ver Observaciones).

### No aplica a

- Definición de la configuración fiscal a nivel Producto individual: no se contempla; toda la configuración vive a nivel Familia.
- Interfaz gráfica de usuario para la gestión de la configuración fiscal de producto (asignación de claves, perfil fiscal, catálogos maestros): no se desarrolla en R16; toda la gestión es por base de datos.
- Lógica del motor de facturación: la precedencia final de tasas y las reglas de exportación (por ejemplo, forzar tasa 0% cuando el receptor es extranjero) no forman parte de este catálogo; son responsabilidad del motor de facturación.
- Familia "Dispositivo médico": queda fuera del alcance; no se le define clave de producto/servicio y no podrá facturarse. Actualmente no hay productos en esa familia; si en el futuro se agregaran, su tratamiento se maneja de manera operativa (ver Riesgo 3).
- Configuración fiscal para Perú: fuera del alcance del timbrado de Perú.

## Criterios de Aceptación

### Reglas de Negocio

**Regla 1 — Tres datos fiscales por familia**
Cada familia de productos facturable en México cuenta con tres datos fiscales: clave de producto/servicio del SAT, clave de unidad del SAT y perfil fiscal de IVA.

**Regla 2 — Definición a nivel Familia**
La configuración fiscal se define a nivel Familia de productos. Todos los productos de una misma familia heredan la misma configuración; no existe configuración a nivel Producto individual.

**Regla 3 — Perfil fiscal como opción acotada**
El perfil fiscal de IVA corresponde a una opción acotada (IVA 16%, IVA 0%, Exento). No se capturan claves técnicas del SAT; cada perfil referencia internamente los catálogos maestros del SAT.

**Regla 4 — Regla de IVA por familia**
Se aplica IVA 16% a todos los productos, con excepción de las publicaciones, a las que se aplica IVA 0%.

**Regla 5 — Catálogos maestros del SAT no editables**
Los catálogos maestros del SAT (impuesto, tipo de factor, objeto de impuesto) se cargan como datos oficiales y no se modifican.

**Regla 6 — Mapeo de familia a clave del SAT**
Cada familia de productos tiene asignada su clave de producto/servicio del SAT y su clave de unidad conforme al mapeo definido por PROQUIFA:

Clave de producto/servicio del SAT por familia:

| Familia                      | Clave de producto/servicio del SAT    |
| ---------------------------- | ------------------------------------- |
| Biológico                    | 41116132                              |
| Estándares                   | 41116107                              |
| Reactivos                    | 41116105                              |
| Publicaciones                | 55101500                              |
| Capacitaciones               | 86101600                              |
| Labware                      | 41116100                              |
| Fletes                       | 78102205                              |
| Servicios                    | 85131701                              |
| Partidas de factura anticipo | 84111506                              |
| Dispositivo médico           | Sin clave asignada (fuera de alcance) |

Clave de unidad del SAT:

| Clave de unidad | Aplica a                    |
| --------------- | --------------------------- |
| E48             | Fletes y capacitaciones     |
| H87             | Todo lo que no entra en E48 |
| ACT             | Toda factura anticipo       |

**Regla 7 — Familia sin clave no facturable**
Una familia sin clave de producto/servicio asignada no puede facturarse; el intento de timbrado fallaría. Es el caso de la familia "Dispositivo médico", excluida del alcance y manejada operativamente (ver Riesgo 3).

**Regla 8 — Gestión sin interfaz gráfica en R16**
La gestión de la configuración fiscal de producto (asignación de claves y perfil fiscal a las familias, y carga de los catálogos maestros del SAT) no dispone de interfaz gráfica de usuario en R16. Es responsabilidad del área correspondiente mediante acceso directo a la base de datos del sistema.

### Criterios Funcionales

#### Sección A — Configuración fiscal de la familia

**Criterio A1 — Configuración fiscal de la familia en base de datos**
**Dado que** una familia de productos es facturable
**Cuando** se configura su información fiscal en base de datos
**Entonces** la familia queda con una clave de producto/servicio del SAT, una clave de unidad del SAT y un perfil fiscal de IVA asociados.

**Criterio A2 — Herencia hacia los productos**
**Dado que** un producto pertenece a una familia,
**Cuando** se factura ese producto,
**Entonces** el sistema toma la configuración fiscal de su familia para armar la partida del CFDI.

**Criterio A3 — Perfil fiscal acotado**
**Dado que** se configura la información fiscal de una familia en base de datos,
**Cuando** se le asigna el perfil fiscal de IVA,
**Entonces** corresponde a una de las opciones acotadas (IVA 16%, IVA 0%, Exento), que referencia internamente los catálogos maestros del SAT sin capturar sus claves como texto libre.

#### Sección B — Timbrado de la partida

**Criterio B1 — Claves en el CFDI**
**Dado que** se timbra una factura,
**Cuando** se arma cada partida,
**Entonces** el sistema reporta la clave de producto/servicio, la clave de unidad y la información de impuesto derivadas del perfil fiscal de la familia del producto.

**Criterio B2 — Familia sin clave**
**Dado que** un producto pertenece a una familia sin clave de producto/servicio asignada,
**Cuando** se intenta facturar,
**Entonces** el sistema no permite facturar ese producto.

### Riesgos

**Riesgo 1 — Escenarios de comercio exterior fuera de este release**
Existen operaciones al extranjero (importación/exportación) de muy bajo volumen, hoy gestionadas de forma manual y por fuera del sistema. Estos escenarios tienen implicaciones fiscales sobre la partida que este requisito no resuelve: en exportación el SAT exige tasa 0% para cliente extranjero (distinta del perfil fiscal de la familia), y en importación se requiere el pedimento. **Se atienden en un análisis aparte de comercio exterior; pendiente del cliente establecer los controles internos y compartirlos.**

**Riesgo 3 — Familia "Dispositivo médico" sin clave de servicio**
La familia "Dispositivo médico" existe en el sistema pero no tiene clave de producto/servicio definida (actualmente sin productos en el catálogo). Si en el futuro se agregan productos a esa familia y se intentan cotizar y facturar, el sistema no tendrá una definición fiscal para ellos y el timbrado fallará, deteniendo el flujo. Se manejará de manera operativa (no se maneja este tipo de familias).
