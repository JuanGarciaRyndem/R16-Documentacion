# Propuesta — Almacenamiento genérico de Addendas para `fccFactura`

## Objetivo

Diseñar un modelo **agnóstico al receptor** que permita almacenar cualquier addenda CFDI (Liverpool, Walmart, cliente genérico, etc.) sin modificar el esquema cada vez que se agregue un nuevo formato, manteniendo trazabilidad, reconstrucción del XML original y consulta estructurada.

## Contexto

Las **Addendas** son un nodo opcional del CFDI (`<cfdi:Addenda>`) que contiene información **comercial o administrativa** requerida por el receptor. El SAT no valida su contenido, pero cada cadena comercial (Liverpool, Walmart, Ford, IMSS, etc.) publica su propio esquema (XSD).

Un enfoque naive sería crear una tabla por cada tipo de addenda (`fccAddendaLiverpool`, `fccAddendaWalmart`, etc.), lo cual es **inviable** por mantenimiento. Esta propuesta plantea un modelo genérico.

## Estrategia recomendada — Híbrida (XML crudo + EAV normalizado)

Se propone un enfoque **híbrido** que combina lo mejor de tres mundos:

1. **Persistir el XML original completo** (fidelidad y reconstrucción exacta).
2. **Catálogo de plantillas** por tipo de addenda/cliente (metadatos y esquema).
3. **Tabla EAV (Entity-Attribute-Value)** para consulta estructurada de nodos/atributos clave.

Esto evita el antipatrón de crear una tabla por cada cliente y a la vez permite consultar campos específicos sin parsear el XML cada vez.

---

## Diccionario de datos

### Tabla 1 — `fccAddendaTipo` (Catálogo de tipos de addenda)

**Descripción:** Catálogo maestro de tipos de addenda soportados por el sistema. Cada registro representa un formato distinto (por cliente/cadena).

| Columna         | Tipo                  | Descripción                                                          |
| --------------- | --------------------- | -------------------------------------------------------------------- |
| `IdAddendaTipo` | `UNIQUEIDENTIFIER` PK | Identificador del tipo de addenda (GUID hardcodeado en script)       |
| `Clave`         | `VARCHAR(50)` UK      | Clave única (ej. `ADD-CLIENTE-GEN`, `ADD-LIVERPOOL`, `ADD-WALMART`)  |
| `Nombre`        | `VARCHAR(200)`        | Nombre descriptivo del tipo de addenda                               |
| `Namespace`     | `VARCHAR(500)`        | XML namespace del receptor (`http://www.cliente.com.mx/addenda`)     |
| `Prefijo`       | `VARCHAR(20)`         | Prefijo del namespace (`cliente`)                                    |
| `Version`       | `VARCHAR(20)`         | Versión del esquema (`1.0`)                                          |
| `RutaXsd`       | `VARCHAR(500)` NULL   | URL/ruta al XSD del receptor para validación (opcional)              |
| `PlantillaXslt` | `NVARCHAR(MAX)` NULL  | XSLT o plantilla Razor para generar el XML desde datos estructurados |
| `Activo`        | `BIT`                 | Habilitado/deshabilitado                                             |
| `FechaCreacion` | `DATETIME2`           | Auditoría                                                            |

**Relaciones:** 1 → N con `fccFacturaAddenda`.

**Índices:**
- `PK_fccAddendaTipo` (Clustered en `IdAddendaTipo`).
- `UK_fccAddendaTipo_Clave` (Unique sobre `Clave`).

**Consideraciones:** Catálogo con `IdAddendaTipo` GUID hardcodeado (estándar R16 para catálogos) y UK en `Clave`.

### Tabla 2 — `fccFacturaAddenda` (Addenda por factura — cabecera)

**Descripción:** Cabecera transaccional que asocia una factura con su addenda correspondiente y almacena el XML íntegro.

| Columna             | Tipo                                     | Descripción                                                     |
| ------------------- | ---------------------------------------- | --------------------------------------------------------------- |
| `IdFacturaAddenda`  | `UNIQUEIDENTIFIER` PK                    | GUID generado como UUIDv7 desde la aplicación (sin default en BD) |
| `IdFactura`         | `UNIQUEIDENTIFIER` FK → `fccFactura`     | Factura asociada                                                |
| `IdAddendaTipo`     | `UNIQUEIDENTIFIER` FK → `fccAddendaTipo` | Tipo de addenda aplicada                                        |
| `XmlOriginal`       | `XML`                                    | XML completo del nodo `<cfdi:Addenda>` tal como se envió/timbró |
| `Hash`              | `VARCHAR(64)`                            | SHA-256 del XML para integridad                                 |
| `EstadoIntegracion` | `TINYINT`                                | 0=Pendiente, 1=Aplicada, 2=Rechazada                            |
| `FechaGeneracion`   | `DATETIME2`                              | Cuándo se generó                                                |
| `IdUsuarioCreacion` | `UNIQUEIDENTIFIER`                       | Auditoría                                                       |

**Relaciones:** N → 1 con `fccFactura`; N → 1 con `fccAddendaTipo`; 1 → N con `fccFacturaAddendaDetalle`.

**Índices:**
- `PK_fccFacturaAddenda` (Clustered en `IdFacturaAddenda`).
- `IX_fccFacturaAddenda_IdFactura` (para consulta por factura).
- `IX_fccFacturaAddenda_IdAddendaTipo` (para reportes por tipo).

**Consideraciones:** Se recomienda relación 1:1 con `fccFactura` (una addenda por CFDI). El campo `XmlOriginal` como tipo `XML` de SQL Server permite consultas XPath nativas.

### Tabla 3 — `fccFacturaAddendaDetalle` (EAV — nodos/atributos indexables)

**Descripción:** Almacena los valores clave que el negocio necesita consultar/reportar sin parsear el XML. Modelo Entity-Attribute-Value.

| Columna                   | Tipo                                        | Descripción                                                                                 |
| ------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `IdFacturaAddendaDetalle` | `UNIQUEIDENTIFIER` PK                       | GUID generado como UUIDv7 desde la aplicación (sin default en BD)                           |
| `IdFacturaAddenda`        | `UNIQUEIDENTIFIER` FK → `fccFacturaAddenda` | Cabecera                                                                                    |
| `NodoRuta`                | `VARCHAR(300)`                              | Ruta XPath simplificada (`OrdenCompra`, `Entrega/almacenDestino`, `Referencias/Referencia`) |
| `Atributo`                | `VARCHAR(100)`                              | Nombre del atributo (`numero`, `folioRemision`, `valor`)                                    |
| `Valor`                   | `NVARCHAR(1000)`                            | Valor del atributo                                                                          |
| `TipoDato`                | `VARCHAR(20)`                               | `string`, `date`, `decimal`, `int` (útil para casting)                                      |
| `Orden`                   | `INT`                                       | Orden dentro del padre (para nodos repetidos)                                               |

**Relaciones:** N → 1 con `fccFacturaAddenda`.

**Índices:**
- `PK_fccFacturaAddendaDetalle` (Clustered en `IdFacturaAddendaDetalle`).
- `IX_Detalle_IdFacturaAddenda`.
- `IX_Detalle_NodoRuta_Atributo` (para búsquedas por campo).

**Consideraciones:** Este modelo permite reportes como "todas las facturas con `OrdenCompra` que empieza con `OC-2026-`" sin necesidad de deserializar el XML.

---

## Ejemplo — Descomposición de la Addenda

**XML de entrada:**

```xml
<cfdi:Addenda>
  <cliente:AddendaCliente xmlns:cliente="http://www.cliente.com.mx/addenda" version="1.0">
    <cliente:OrdenCompra numero="OC-2026-9876" fecha="2026-08-01"/>
    <cliente:Entrega folioRemision="REM-4567" almacenDestino="CEDIS-NORTE" fechaEntrega="2026-08-10"/>
    <cliente:Contacto nombre="Juan Pérez" correo="compras@cliente.com.mx" telefono="5555123456"/>
    <cliente:Referencias>
      <cliente:Referencia tipo="CentroCostos" valor="CC-4400"/>
      <cliente:Referencia tipo="Proyecto" valor="PRY-INV-2026"/>
    </cliente:Referencias>
    <cliente:Observaciones>Entregar en horario de 8:00 a 14:00 hrs.</cliente:Observaciones>
  </cliente:AddendaCliente>
</cfdi:Addenda>
```

**Filas generadas en `fccFacturaAddendaDetalle`:**

| NodoRuta | Atributo | Valor | TipoDato | Orden |
|---|---|---|---|---|
| `OrdenCompra` | `numero` | `OC-2026-9876` | string | 0 |
| `OrdenCompra` | `fecha` | `2026-08-01` | date | 0 |
| `Entrega` | `folioRemision` | `REM-4567` | string | 0 |
| `Entrega` | `almacenDestino` | `CEDIS-NORTE` | string | 0 |
| `Entrega` | `fechaEntrega` | `2026-08-10` | date | 0 |
| `Contacto` | `nombre` | `Juan Pérez` | string | 0 |
| `Contacto` | `correo` | `compras@cliente.com.mx` | string | 0 |
| `Contacto` | `telefono` | `5555123456` | string | 0 |
| `Referencias/Referencia` | `tipo` | `CentroCostos` | string | 0 |
| `Referencias/Referencia` | `valor` | `CC-4400` | string | 0 |
| `Referencias/Referencia` | `tipo` | `Proyecto` | string | 1 |
| `Referencias/Referencia` | `valor` | `PRY-INV-2026` | string | 1 |
| `Observaciones` | `#text` | `Entregar en horario de 8:00 a 14:00 hrs.` | string | 0 |

---

## Metodología de guardado (flujo paso a paso)

### 1. Recepción/Generación de la Addenda

El servicio `AddendaService` recibe:

- Un DTO estructurado (`AddendaClienteDto` con `OrdenCompra`, `Entrega`, `Contacto`, `Referencias`, `Observaciones`), **o**
- Un XML pre-armado por el cliente.

### 2. Resolución del tipo

```csharp
var tipo = await _repo.GetAddendaTipoPorClaveAsync("ADD-CLIENTE-GEN");
```

### 3. Serialización a XML

Si viene DTO → aplicar `PlantillaXslt` o serializador XML tipado del tipo correspondiente para producir el nodo `<cfdi:Addenda>` con el namespace correcto.

### 4. Inserción de cabecera

```csharp
var addenda = new FacturaAddenda {
    IdFacturaAddenda = UuidV7.NewGuid(),  // UUIDv7 generado en app
    IdFactura = idFactura,
    IdAddendaTipo = tipo.IdAddendaTipo,
    XmlOriginal = xmlAddenda,
    Hash = Sha256(xmlAddenda),
    EstadoIntegracion = EstadoIntegracion.Pendiente,
    FechaGeneracion = DateTime.UtcNow
};
```

### 5. Extracción de detalle (EAV)

Un `AddendaParser` recorre el XML con XPath y proyecta los pares nodo/atributo/valor a `fccFacturaAddendaDetalle`.

### 6. Inyección en el CFDI

Al momento de generar el XML final del CFDI (antes del timbrado si el receptor lo requiere firmado, o después si solo lo conserva el PAC), el `CfdiBuilder` concatena `XmlOriginal` dentro del nodo `<cfdi:Addenda>` del comprobante.

### 7. Post-timbrado

Actualizar `EstadoIntegracion = Aplicada` y persistir el CFDI final ya con addenda incluida en `fccFactura.XmlTimbrado`.

---

## Arquitectura de código (ProquifaDotNet.Finanzas)

**Domain:**

- Entidades: `AddendaTipo`, `FacturaAddenda`, `FacturaAddendaDetalle`.
- Interfaz: `IAddendaBuilder` (una implementación por tipo cuando se requiera lógica especial).

**Application (CQRS):**

- `CrearFacturaAddendaCommand` — recibe `IdFactura` + `ClaveAddendaTipo` + payload (DTO o XML).
- `GetFacturaAddendaQuery` — por `IdFactura`.
- `AddendaFactory` — resuelve el `IAddendaBuilder` correcto por `Clave`.

**Infrastructure:**

- `AddendaXmlParser` — extrae detalle EAV vía `XDocument`/XPath.
- `AddendaXmlSerializer` — genérico basado en plantilla XSLT/Razor almacenada en BD.
- Repositorios EF Core scaffolded para las 3 tablas (`fccAddendaTipo`, `fccFacturaAddenda`, `fccFacturaAddendaDetalle`).

**Patrón:** Strategy + Factory. Cada tipo nuevo de addenda solo requiere:

1. Insertar registro en `fccAddendaTipo` (con plantilla).
2. Opcionalmente, un `IAddendaBuilder` específico si el receptor tiene reglas complejas (validaciones, mapeos condicionales).

---

## Ventajas de este enfoque

- **Extensible:** agregar Liverpool/Walmart/etc. es un registro en catálogo + plantilla, sin cambios de esquema ni migraciones.
- **Fiel:** `XmlOriginal` permite reconstruir exactamente lo timbrado.
- **Consultable:** el EAV permite reportes estructurados sin parsear XML.
- **Auditable:** hash, fechas, usuario, estado de integración.
- **Compatible con transferencia a Legacy:** el XML crudo puede enviarse tal cual al ETL.
- **Alineado con la arquitectura R16:** encaja en el scaffold de EF Core de `ProquifaDotNet.Finanzas` y respeta el estándar UUIDv7 para PKs.

## Consideraciones adicionales

- Si el volumen de addendas por tipo crece mucho y se requieren reportes intensivos, se puede materializar una vista tipada por tipo (vista indexada sobre `fccFacturaAddendaDetalle` filtrada por `IdAddendaTipo`).
- Validar contra XSD antes de guardar (usar `RutaXsd` del catálogo) para rechazar addendas malformadas.
- El campo `XmlOriginal` como tipo `XML` de SQL Server permite consultas XPath nativas como fallback cuando el EAV no tenga el dato requerido.
- Definir política de retención: el `XmlOriginal` puede archivarse en Minio después de N años, dejando solo hash y detalle EAV en BD.

## Próximos pasos sugeridos

1. Aprobación de la propuesta con el arquitecto y equipo de Finanzas.
2. Generar script DDL con el estándar UUIDv7 de R16.
3. Levantar el catálogo inicial con al menos `ADD-CLIENTE-GEN` (addenda genérica de ejemplo).
4. Definir tareas de estimación en la Guía de Estimación (WBS Backend) para el requisito correspondiente.
