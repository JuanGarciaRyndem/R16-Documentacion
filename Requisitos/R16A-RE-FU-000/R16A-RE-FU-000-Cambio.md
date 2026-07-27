# R16A-RE-FU-000 — Solicitud de Cambio al Proyecto

## Información del cambio

| Campo | Valor |
| --- | --- |
| **Proyecto** | R16 — ProquifaDotNet |
| **Requisito asociado** | R16A-RE-FU-000 — Perfil Fiscal |
| **Tipo de cambio** | Incorporación de nuevo requisito funcional |
| **Fecha de solicitud** | 2026-07-27 |
| **Solicitante** | Equipo de Análisis R16 |
| **Estado** | Propuesto |

## Descripción del cambio

Se incorpora al alcance del proyecto R16 la **configuración fiscal de producto para México**, definida a nivel Familia de productos, como base para el armado y timbrado del CFDI. El cambio consiste en que el sistema cuente con tres datos fiscales por familia: la clave de producto/servicio del SAT, la clave de unidad de medida del SAT y el perfil fiscal de IVA (16%, 0% o exento).

El perfil fiscal de IVA se resuelve como un catálogo de negocio acotado (IVA 16%, IVA 0%, Exento) que referencia internamente los catálogos maestros del SAT (impuesto, tipo de factor y objeto de impuesto), sin exponer sus claves técnicas. Los catálogos maestros se precargan como datos oficiales y no se modifican.

## Justificación

Sin esta configuración, el sistema no puede reportar en cada partida del CFDI las claves que exige el SAT para el timbrado en México. Definirla a nivel Familia evita la captura producto por producto: todos los productos de una familia heredan la misma configuración fiscal, conforme al mapeo de claves entregado por PROQUIFA (9 familias con clave asignada; "Dispositivo médico" queda fuera de alcance, sin clave y no facturable).

## Alcance del cambio

- Estructura en base de datos ProquifaDotNet para los catálogos maestros del SAT y el catálogo de perfil fiscal, y su asignación a nivel Familia de productos.
- Precarga de los catálogos maestros del SAT y del mapeo de familias a claves (producto/servicio y unidad) definido por PROQUIFA.
- Consumo de la configuración por el armado de la partida del CFDI: cada partida reporta clave de producto/servicio, clave de unidad e información de impuesto derivadas de la familia del producto.
- Restricción de facturación para familias sin clave de producto/servicio asignada.

## Fuera del alcance del cambio

- Interfaz gráfica para la gestión de la configuración fiscal: toda la gestión (asignación de claves, perfil fiscal y carga de catálogos) se realiza directamente en base de datos por el área responsable.
- Configuración fiscal a nivel Producto individual: la configuración vive exclusivamente a nivel Familia.
- Lógica del motor de facturación: precedencia final de tasas y reglas de exportación son responsabilidad del motor de facturación, no de este catálogo.
- Configuración fiscal y timbrado para Perú.
- Familia "Dispositivo médico": sin clave asignada, no facturable, manejo operativo.

## Impacto

| Área | Impacto |
| --- | --- |
| **Base de datos** | Alta — nuevas tablas de catálogos maestros del SAT, catálogo de perfil fiscal y configuración fiscal a nivel Familia, con seed de datos oficiales y del mapeo PROQUIFA |
| **Facturación / Timbrado** | Media — el armado de la partida del CFDI consume la configuración de la familia; validación de familia sin clave |
| **Interfaz de usuario** | Nula — sin desarrollo de pantallas en R16 |
| **Operación** | Baja — el área responsable gestiona la configuración por acceso directo a base de datos |

## Riesgos asociados

1. **Comercio exterior fuera de este release** — las operaciones de importación/exportación tienen implicaciones fiscales sobre la partida que este cambio no resuelve; se atienden en un análisis aparte, pendiente de que el cliente establezca y comparta los controles internos.
2. **Familia "Dispositivo médico" sin clave** — si en el futuro se agregan productos a esa familia e intentan facturarse, el timbrado fallará; se maneja de manera operativa.

## Referencias

- `R16A-RE-FU-000.md` — especificación completa del requisito (reglas de negocio, criterios funcionales, riesgos).
- `Analisis\Guia_Tecnica_Perfil_Fiscal_IVA_MX.md` — reglas de negocio del IVA (México).
- `Analisis\Facturación\Diseño_PerfilFiscal_Unificado_MX_PE.md` — diseño de la estructura de base de datos.

---

# Descripción para JIRA

**Título sugerido:** [ R16A-RE-FU-000 ] Perfil Fiscal — Configuración fiscal de producto a nivel Familia (México)

```
h2. Descripción

Se incorpora la configuración fiscal de producto para México, definida a nivel Familia de productos, como base para el armado y timbrado del CFDI. Cada familia facturable cuenta con tres datos fiscales: clave de producto/servicio del SAT, clave de unidad de medida del SAT y perfil fiscal de IVA (IVA 16%, IVA 0%, Exento).

El perfil fiscal de IVA es un catálogo de negocio acotado que referencia internamente los catálogos maestros del SAT (impuesto, tipo de factor, objeto de impuesto), sin exponer sus claves técnicas. Los catálogos maestros se precargan como datos oficiales y no se modifican.

La gestión de la configuración (asignación de claves y perfil fiscal a las familias, carga de catálogos) NO dispone de interfaz gráfica en R16: se realiza directamente en base de datos por el área responsable.

h2. Alcance

* Configuración fiscal por Familia: clave de producto/servicio del SAT, clave de unidad del SAT y perfil fiscal de IVA.
* Herencia hacia los productos: al facturar, cada partida del CFDI toma la configuración fiscal de la familia del producto.
* Catálogos maestros del SAT precargados y no editables.
* Mapeo de familias a claves conforme a definición de PROQUIFA (Biológico 41116132, Estándares 41116107, Reactivos 41116105, Publicaciones 55101500, Capacitaciones 86101600, Labware 41116100, Fletes 78102205, Servicios 85131701, Partidas de factura anticipo 84111506; unidades: E48 fletes/capacitaciones, H87 resto, ACT factura anticipo).
* Familia sin clave de producto/servicio asignada: no facturable.

h2. Fuera de alcance

* Configuración a nivel Producto individual (toda la configuración vive a nivel Familia).
* Interfaz gráfica para la gestión de la configuración fiscal (toda la gestión es por base de datos).
* Lógica del motor de facturación (precedencia final de tasas y reglas de exportación).
* Configuración fiscal y timbrado para Perú.
* Familia "Dispositivo médico" (sin clave, no facturable, manejo operativo).

h2. Criterios de aceptación (resumen)

* La familia facturable queda configurada en BD con clave de producto/servicio, clave de unidad y perfil fiscal de IVA (opción acotada: IVA 16%, IVA 0%, Exento).
* Al timbrar, cada partida reporta las claves y la información de impuesto derivadas del perfil fiscal de la familia del producto.
* Regla de IVA: 16% a todos los productos, excepto publicaciones (0%).
* Un producto de una familia sin clave de producto/servicio no puede facturarse.

h2. Riesgos

* Comercio exterior (importación/exportación) fuera de este release — se atiende en análisis aparte; pendiente del cliente compartir controles internos.
* Familia "Dispositivo médico" sin clave — si a futuro recibe productos, el timbrado fallará; manejo operativo.

h2. Referencias

* Especificación: R16A-RE-FU-000.md
* Guía técnica: Guia_Tecnica_Perfil_Fiscal_IVA_MX.md
* Diseño de BD: Diseño_PerfilFiscal_Unificado_MX_PE.md
```
