## Descripción

Se incorpora la configuración fiscal de producto para México, definida a nivel Familia de productos, como base para el armado y timbrado del CFDI. Cada familia facturable cuenta con tres datos fiscales: clave de producto/servicio del SAT, clave de unidad de medida del SAT y perfil fiscal de IVA (IVA 16%, IVA 0%, Exento).

El perfil fiscal de IVA es un catálogo de negocio acotado que referencia internamente los catálogos maestros del SAT (impuesto, tipo de factor, objeto de impuesto), sin exponer sus claves técnicas. Los catálogos maestros se precargan como datos oficiales y no se modifican.

La gestión de la configuración (asignación de claves y perfil fiscal a las familias, carga de catálogos) NO dispone de interfaz gráfica en R16: se realiza directamente en base de datos por el área responsable.

## Alcance

- Configuración fiscal por Familia: clave de producto/servicio del SAT, clave de unidad del SAT y perfil fiscal de IVA.
    
- Herencia hacia los productos: al facturar, cada partida del CFDI toma la configuración fiscal de la familia del producto.
    
- Catálogos maestros del SAT precargados y no editables.
    
- Mapeo de familias a claves conforme a definición de PROQUIFA (Biológico 41116132, Estándares 41116107, Reactivos 41116105, Publicaciones 55101500, Capacitaciones 86101600, Labware 41116100, Fletes 78102205, Servicios 85131701, Partidas de factura anticipo 84111506; unidades: E48 fletes/capacitaciones, H87 resto, ACT factura anticipo).
    
- Familia sin clave de producto/servicio asignada: no facturable.
    

## Fuera de alcance

- Configuración a nivel Producto individual (toda la configuración vive a nivel Familia).
    
- Interfaz gráfica para la gestión de la configuración fiscal (toda la gestión es por base de datos).
    
- Lógica del motor de facturación (precedencia final de tasas y reglas de exportación).
    
- Configuración fiscal y timbrado para Perú.
    
- Familia "Dispositivo médico" (sin clave, no facturable, manejo operativo).
    

## Criterios de aceptación (resumen)

- La familia facturable queda configurada en BD con clave de producto/servicio, clave de unidad y perfil fiscal de IVA (opción acotada: IVA 16%, IVA 0%, Exento).
    
- Al timbrar, cada partida reporta las claves y la información de impuesto derivadas del perfil fiscal de la familia del producto.
    
- Regla de IVA: 16% a todos los productos, excepto publicaciones (0%).
    
- Un producto de una familia sin clave de producto/servicio no puede facturarse.
    

## Riesgos

- Comercio exterior (importación/exportación) fuera de este release — se atiende en análisis aparte; pendiente del cliente compartir controles internos.
    
- Familia "Dispositivo médico" sin clave — si a futuro recibe productos, el timbrado fallará; manejo operativo.