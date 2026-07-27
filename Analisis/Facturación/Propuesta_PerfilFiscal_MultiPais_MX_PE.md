# Propuesta — PerfilFiscal multi-país (México + Perú)

**Estado:** CERRADO — decisión tomada: se integran las 2 entidades en una sola tabla `PerfilFiscal` (IVA + IGV) separada por `IdRegion`, con catálogos de apoyo neutrales por región y Perú como catálogo cerrado (no atributo por producto). Ver el diseño final en `Diseño_PerfilFiscal_Unificado_MX_PE.md` — sustituye la recomendación de la sección 3 (Opción B).

**Contexto original de la propuesta:** propuesta arquitectónica, no decisión tomada. Se levanta a partir de la conversación de revisión del `PerfilFiscal` actual (diseño 100% SAT/México) y el análisis paralelo de facturación Perú que ya vive en `Guia_Tecnica_Facturacion_Peru.md`.

**Documentos base a leer antes de este:**
- `../Guia_Tecnica_Perfil_Fiscal_IVA_MX.md` — diseño actual del `PerfilFiscal` mexicano (catálogos SAT `SatImpuesto`, `SatTipoFactor`, `SatObjetoImp` + `PerfilFiscal`).
- `./Guia_Tecnica_Facturacion_Peru.md` — reglas de negocio de facturación Perú (IGV, Retención, Percepción, Detracción) y borrador de `PerfilFiscalPeruProducto` (sección 5.2).

---

## 0. Alcance del documento

**Este documento NO redefine reglas de negocio de Perú.** Todas las reglas de IGV, Retención, Percepción, Detracción, catálogos SUNAT (05, 07, 09, 10, 17, 25), y decisiones ya cerradas con el cliente (ej. Detracción excluida del alcance) ya viven en `Guia_Tecnica_Facturacion_Peru.md` — este documento se apoya en esas decisiones y no las repite.

**Lo que este documento sí aporta** es una decisión arquitectónica todavía abierta: cuando llegue el momento de habilitar Perú en ProquifaDotNet.Finanzas, ¿se resuelve con una **tabla `PerfilFiscal` única multi-país**, o con **tablas separadas por país** (`PerfilFiscal` para México, `PerfilFiscalPeruProducto` para Perú, como está borrado hoy)?

La decisión condiciona:
- Estructura de `Domain` en ProquifaDotNet.Finanzas.
- Estructura del scaffold EF Core sobre ProquifaDotNet.
- Contrato de la interfaz que emite el XML del comprobante (CFDI vs UBL 2.1).
- Cómo se comparte —o no— la configuración fiscal de un mismo producto entre México y Perú.

---

## 1. Punto de partida — dos diseños ya escritos, incompatibles entre sí

### 1.1 Diseño México (ya documentado, sección 3.2 del `Guia_Tecnica_Perfil_Fiscal_IVA_MX.md`)

```sql
CREATE TABLE PerfilFiscal (
    PerfilFiscalId INT IDENTITY(1,1) PRIMARY KEY,
    Nombre NVARCHAR(100) NOT NULL,
    SatImpuestoCode CHAR(3) NOT NULL REFERENCES SatImpuesto(Code),
    SatTipoFactorCode VARCHAR(10) NOT NULL REFERENCES SatTipoFactor(Code),
    SatObjetoImpCode CHAR(2) NOT NULL REFERENCES SatObjetoImp(Code),
    TasaOCuota DECIMAL(6,6) NULL,
    Fundamento NVARCHAR(200) NULL,
    CONSTRAINT CK_TasaOCuota_Exento CHECK (
        (SatTipoFactorCode = 'Exento' AND TasaOCuota IS NULL) OR
        (SatTipoFactorCode <> 'Exento' AND TasaOCuota IS NOT NULL)
    )
);
```

3-4 filas iniciales (`IVA General 16%`, `IVA Tasa 0%`, `Exento`), asignadas a producto/familia con precedencia Producto → Familia.

### 1.2 Diseño Perú (borrador, sección 5.2 del `Guia_Tecnica_Facturacion_Peru.md`)

```sql
CREATE TABLE PerfilFiscalPeruProducto (
    ProductoId INT PRIMARY KEY,
    TipoAfectacionIGVCode CHAR(2) NOT NULL REFERENCES SatTipoAfectacionIGV(Code),
    TasaIGV DECIMAL(5,2) NULL,
    SujetoPercepcion BIT NOT NULL DEFAULT 0,
    TasaPercepcion DECIMAL(5,2) NULL,
    CONSTRAINT CK_TasaIGV_Gravado CHECK (
        (TipoAfectacionIGVCode = '10' AND TasaIGV IS NOT NULL) OR
        (TipoAfectacionIGVCode <> '10' AND TasaIGV IS NULL)
    ),
    CONSTRAINT CK_TasaPercepcion CHECK (
        (SujetoPercepcion = 0 AND TasaPercepcion IS NULL) OR
        (SujetoPercepcion = 1 AND TasaPercepcion IS NOT NULL)
    )
);
```

Asignado directamente por producto/familia con precedencia Producto → Familia.

### 1.3 Por qué no son "el mismo diseño con distinto catálogo"

La forma habitual de pensarlo —"Perú es solo México con otra tasa y otras claves de catálogo"— **no funciona**, por 3 razones estructurales:

1. **México necesita 3 catálogos SAT combinados** (`SatImpuesto` + `SatTipoFactor` + `SatObjetoImp`) porque el CFDI exige los tres nodos en el XML. **Perú resuelve todo con un solo catálogo SUNAT** (Catálogo 07 — `TipoAfectacionIGV`) — el UBL 2.1 no exige la triple descomposición. Ver `Guia_Tecnica_Facturacion_Peru.md` sección 1.2 para la lista completa de 18 valores del Catálogo 07.
2. **México NO tiene un mecanismo equivalente a la Percepción** — es un dato adicional por producto (`SujetoPercepcion` + `TasaPercepcion`) que no existe en el `PerfilFiscal` mexicano y no tiene ningún reflejo en el CFDI.
3. **La granularidad del catálogo es distinta.** En México, `PerfilFiscal` es un catálogo cerrado de 3-4 filas reutilizables entre miles de productos. En Perú, el borrador actual asigna `TipoAfectacionIGV` **por producto directamente** (no via un catálogo intermedio reutilizable). Esto es una diferencia de modelo, no solo de datos.

Traducción: no hay un supertipo natural que colapse ambos diseños sin perder información. Cualquier intento de "una tabla para los dos" o bien deja columnas nullable dependientes de país (mal olor de diseño), o bien introduce un mapeo indirecto que oculta el modelo real de cada país.

---

## 2. Las 3 opciones sobre la mesa

### 2.1 Opción A — Una tabla `PerfilFiscal` con discriminador `PaisCode`

```sql
ALTER TABLE PerfilFiscal ADD PaisCode CHAR(2) NOT NULL DEFAULT 'MX';
-- Columnas de SAT quedan nullable para filas PE
-- Columnas nuevas de SUNAT quedan nullable para filas MX
-- Percepción se agrega como 2 columnas nullable, solo válidas para PE
```

**A favor:**
- Un solo lugar para responder "¿qué tratamiento fiscal tiene este producto?".
- Simplifica la relación Producto → PerfilFiscal (una sola FK, no una por país).
- Es fácil ver la operación cruzada de un producto vendido en ambos países.

**En contra:**
- Muchas columnas nullable condicionales al `PaisCode` — genera CHECKs largos y frágiles, y el modelo se lee mal.
- El scaffold EF Core queda con una entidad "gordita" que mezcla conceptos de dos jurisdicciones.
- Cada vez que uno de los dos países agregue un impuesto nuevo (ej. ISC en Perú, o IEPS más granular en México), impacta a la tabla común y a todas sus validaciones.
- Los mappers a CFDI y a UBL 2.1 igual tienen que ramificar por `PaisCode`, así que la "unificación" es cosmética.

### 2.2 Opción B — Tablas separadas por país, sin supertipo (situación actual del borrador)

`PerfilFiscal` (MX) y `PerfilFiscalPeruProducto` (PE), como están descritas en la sección 1.

**A favor:**
- Cada tabla refleja limpiamente el modelo fiscal real de su país — sin nullables condicionales.
- Los CHECKs son locales al país y fáciles de leer.
- Los mappers son independientes: `MxCfdiMapper` lee de `PerfilFiscal`, `PeUblMapper` lee de `PerfilFiscalPeruProducto`.
- Un cambio en un catálogo de un país no afecta al otro.

**En contra:**
- Duplica la relación Producto → PerfilFiscal (una tabla para MX, otra para PE) — un producto vendido en ambos países vive en dos configuraciones distintas.
- No hay un "lugar único" para responder preguntas de reporting cruzado país.
- Si mañana entra Colombia o Chile, se agrega otra tabla más — el patrón escala por copia, no por composición.

### 2.3 Opción C — Supertipo `PerfilFiscal` neutro + subtipos por país

```sql
CREATE TABLE PerfilFiscal (
    PerfilFiscalId INT IDENTITY(1,1) PRIMARY KEY,
    PaisCode CHAR(2) NOT NULL,
    Nombre NVARCHAR(100) NOT NULL,
    -- Sin columnas fiscales — solo identidad y país
);

CREATE TABLE PerfilFiscalMx (
    PerfilFiscalId INT PRIMARY KEY REFERENCES PerfilFiscal(PerfilFiscalId),
    SatImpuestoCode CHAR(3) NOT NULL REFERENCES SatImpuesto(Code),
    SatTipoFactorCode VARCHAR(10) NOT NULL REFERENCES SatTipoFactor(Code),
    SatObjetoImpCode CHAR(2) NOT NULL REFERENCES SatObjetoImp(Code),
    TasaOCuota DECIMAL(6,6) NULL,
    Fundamento NVARCHAR(200) NULL
);

CREATE TABLE PerfilFiscalPe (
    PerfilFiscalId INT PRIMARY KEY REFERENCES PerfilFiscal(PerfilFiscalId),
    TipoAfectacionIGVCode CHAR(2) NOT NULL REFERENCES SunatTipoAfectacionIGV(Code),
    TasaIGV DECIMAL(5,2) NULL,
    SujetoPercepcion BIT NOT NULL DEFAULT 0,
    TasaPercepcion DECIMAL(5,2) NULL
);
```

**A favor:**
- La Opción A y B combinadas — Producto tiene una sola FK a `PerfilFiscal`, pero cada país tiene su tabla con solo sus columnas propias y sus CHECKs limpios.
- Extender a un tercer país es agregar `PerfilFiscalCo` sin tocar nada de lo existente.
- Los mappers hacen `JOIN` directo a su tabla-país, no inspeccionan discriminadores.

**En contra:**
- Es el diseño más "correcto" en modelo relacional, pero introduce complejidad de mantenimiento en EF Core (herencia TPT — Table Per Type) que hoy no existe en la solución.
- Requiere un patrón de repositorio consciente del país al insertar (crear fila en padre + subtabla en una transacción).
- Overkill si Perú nunca entra al alcance — se paga complejidad por una eventualidad.

---

## 3. Recomendación

**Opción B — mantener tablas separadas por país — mientras Perú siga en "pendiente de decisión de alcance"** (ver `Guia_Tecnica_Facturacion_Peru.md` sección 0).

Razonamiento:
- No hay costo de refactor todavía — la tabla `PerfilFiscal` mexicana ya existe con ese nombre y ese modelo, y así se documentó al cliente.
- La tabla `PerfilFiscalPeruProducto` ya está borrada, con nombre distinto — no hay colisión conceptual.
- Los mappers a CFDI y a UBL 2.1 son código distinto de todos modos — no se pierde reuso real por tener tablas separadas.
- **La Opción A se descarta** — los nullables condicionales por país no compensan la ilusión de unificación.
- **La Opción C se pospone** — solo tiene sentido si aparece un tercer país. Mientras solo sean MX y PE, la Opción B es más barata y más legible.

### 3.1 Lo único que sí conviene abstraer desde ya

Aunque las tablas se queden separadas, la capa de aplicación en ProquifaDotNet.Finanzas sí debe tener un contrato único para no acoplar el consumidor al país:

```csharp
public interface IPerfilFiscalResolver
{
    // Devuelve el tratamiento fiscal listo para el mapper del país
    // El resolver internamente sabe si consulta PerfilFiscal (MX) o PerfilFiscalPeruProducto (PE)
    // según el país de la operación
    Task<TratamientoFiscal> ResolverAsync(int productoId, string paisCode);
}
```

Y dos implementaciones del mapper a XML, una por país:

```csharp
public interface IComprobanteFiscalMapper
{
    string PaisCode { get; }
    XmlDocument Generar(Comprobante c);
}

public class MxCfdiMapper : IComprobanteFiscalMapper { public string PaisCode => "MX"; /* ... */ }
public class PeUblMapper  : IComprobanteFiscalMapper { public string PaisCode => "PE"; /* ... */ }
```

El registro por DI (`services.AddKeyedScoped<IComprobanteFiscalMapper, MxCfdiMapper>("MX")`) permite que el orquestador de emisión elija el mapper por país de la operación sin `if`/`switch` regados por el código.

### 3.2 Lo que NO se hace hoy

- No se crea la tabla `PerfilFiscalPeruProducto` en la base de datos hasta que se cierre la decisión de alcance de Perú (sección 0 del guía Perú).
- No se cargan catálogos SUNAT (Cat 05, 07, 09, 10, 17) al scaffold EF Core hasta la misma condición.
- No se implementa `PeUblMapper` — se deja como stub o interfaz sin implementación, para que el compilador no se queje si otro código lo referencia condicionalmente.

---

## 4. Impacto en ProquifaDotNet.Finanzas cuando Perú entre al alcance

Cuando el cliente confirme que Perú se construye, el trabajo se reduce a:

1. **Base de datos ProquifaDotNet:** crear `PerfilFiscalPeruProducto` (más su tabla hermana a nivel Familia, ver `Guia_Tecnica_Facturacion_Peru.md` sección 5.2), y los catálogos SUNAT (Cat 05, 07, 09, 10, 17) como seed.
2. **`Finanzas.Infrastructure`:** agregar esas tablas al scaffold EF Core.
3. **`Finanzas.Domain`:** entidad `PerfilFiscalPe` + `Percepcion` (valor de Cobro, no de emisión — ver guía Perú secciones 3 y 4).
4. **`Finanzas.Application`:** implementación de `PerfilFiscalResolver` para el caso `PE`, y del cálculo de Retención/Percepción a nivel Cobro.
5. **Nueva solución (o módulo dentro de Finanzas):** implementación del `PeUblMapper` — armado UBL 2.1, firma digital con certificado del emisor peruano, envío al PSE peruano.
6. **Configuración de cliente:** agregar `EsAgenteRetencion` a la ficha de cliente (Datos Legales), como se define en guía Perú sección 3.

Nada de esto invalida el diseño mexicano actual de `PerfilFiscal` — coexisten sin colisión.

---

## 5. Preguntas abiertas — se responden con el cliente, no arquitectónicamente

Estas viven en `Guia_Tecnica_Facturacion_Peru.md` (secciones 0, 1.2, 3, 4, 5.3, 8) y este documento **no las duplica**. Se listan aquí solo como referencia de lo que sigue bloqueado:

- Decisión de alcance — ¿Perú se construye en R16 o queda fuera?
- ¿Algún producto del catálogo PROQUIFA está Exonerado/Inafecto en Perú?
- ¿Golocaer S.A.C. es Agente de Percepción designado por SUNAT?
- ¿Algún cliente de la cartera Perú es Agente de Retención designado por SUNAT?
- ¿Los atributos `TipoAfectacionIGV` y `SujetoPercepcion` se configuran al mismo nivel (Producto o Familia), o pueden vivir en niveles distintos?

Ninguna de estas afecta la decisión de tablas-separadas-vs-tabla-unica de la sección 3 — con o sin esas respuestas, la recomendación se mantiene.

---

## 6. Resumen ejecutivo

- El `PerfilFiscal` actual es 100% mexicano por diseño; Perú necesita otro modelo, no un ajuste al mismo.
- Se descarta la opción de "una tabla con `PaisCode`" — mezcla ejes fiscales distintos con nullables condicionales.
- Se recomienda mantener tablas separadas por país (`PerfilFiscal` MX, `PerfilFiscalPeruProducto` PE) y abstraer el consumo detrás de `IPerfilFiscalResolver` + un `IComprobanteFiscalMapper` por país (registrado como keyed service en DI).
- Nada se implementa de Perú hasta que se cierre la decisión de alcance — pero el diseño MX no queda bloqueado por esta decisión.
