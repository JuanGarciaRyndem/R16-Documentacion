# TPSC-RE-FU-007 — Análisis de Impacto Backend
## Leyenda Regulatoria en Cotización Definitiva

> **Módulo:** Cotizar lo Cotizable  
> **Tipo de cambio:** Funcionalidad nueva sobre módulo preexistente  
> **Alcance:** Solo cotizaciones definitivas con al menos una partida de Sustancia Controlada

---

## 1. Estructura Actual del Proyecto

El FU-007 impacta **dos repositorios independientes**:

```
Repositorio 1 — ProquifaDotNet-R14  (Backend principal)
  Logic.Pqf.Catalogos\Archivos\PDFs\Quotation\
    QuotationModel.cs                       <-- DTO enviado al DocumentBuilder API

  Logic.Pqf.Logistica\PDF\Cotizacion\
    ArchivoBOCotizacionExtensions.cs         <-- Entry point: definitiva vs investigación
    ArchivoBOCotizacionUnitariaExtensions.cs <-- Genera el PDF de cotización definitiva
    ArchivoBOControladosNacionales.cs        <-- Carta controlados por producto (independiente)

  BD (SQL Server)
    dbo.fnEsProductoControlado               <-- Función escalar que detecta controlados
    dbo.catControl                           <-- Catálogo de tipos: mundiales/nacionales/origen

Repositorio 2 — DocumentBuilder-R14  (Motor de renderizado PDF)
  Application\DTOs\Quotation\
    QuotationModel.cs                       <-- DTO recibido por la API (clase espejo)

  Application\Validators\
    DocumentGenerateQuoationDtoFluentValidator.cs  <-- Validaciones FluentValidation

  Application\Services\
    GenerateDocumentService.QuotationExtension.cs  <-- Orquesta render + PDF merge

  API\Resources\Templates\  (templates Scriban HTML por empresa/región)
    PQF_MEX_COT\  PQF_MEX_COT_B.html / _H.html / _F.html
    GOLPERU_PER_COT\  GOLPERU_PER_COT_B.html / _H.html / _F.html
    GOL_MEX_COT\  MUN_MEX_COT\  PHS_MEX_COT\  PRO_MEX_COT\  ERVP_MEX_COT\
    (un directorio por clave TemplateKey = {EMPRESA}_{ISO}_{COT})
```

### Flujo actual de generación del PDF de cotización definitiva

```
ProquifaDotNet-R14
  ArchivoBO.Cotizacion(idCotCotizacion)
    └── ArchivoBOCotizacionExtensions._Cotizacion()
          ├── Si CotizacionDeInvestigacion=true
          │     └── _CotizacionUnitariaInvestigacion()   [sin leyenda]
          └── Si cotización definitiva guardada
                ├── BuildQuotationContext()               [carga region, partidas, productos]
                └── _CotizacionUnitaria(ctx, publicaciones)
                      ├── Construye items / itemsCDU / itemsService
                      ├── Arma QuotationModel
                      └── POST api/Report/quotation  ──────────────────────────────────┐
                                                                                         │
DocumentBuilder-R14                                                                     │
  GenerateDocumentService.GenerateQuotationTemplate(DocumentGenerateQuoationDto) ◄──────┘
    ├── GetTemplate(TemplateKey)  →  DocumentTemplate de BD
    ├── JsonSerializer.Serialize(QuotationModel)  →  jsonString
    ├── RenderDocumentFromJson(body/header/footer con Scriban)
    ├── ConvertDocumentToPdf (x2: sin/con totales)
    ├── Si ItemsControlledSubstances.Any()  →  UsageLetter()
    ├── PdfMergeService.MergeManyPdfs()
    └── retorna PDF bytes
```

---

## 2. Mapeo Reglas de Negocio → Código

| Regla | Descripción | Estado actual | Ubicación |
|-------|-------------|---------------|-----------|
| R1 | Leyenda solo en cotizaciones definitivas | ✅ Existe | `ArchivoBOCotizacionExtensions._Cotizacion()` ya separa definitiva de investigación |
| R2 | Detonante: ≥1 partida controlada (Mundial, Nacional u Origen) | ❌ Falta | Sin lógica de detección en `_CotizacionUnitaria`; `fnEsProductoControlado` no cubre `origen` |
| R3 | Leyenda aparece una sola vez por documento | ❌ Falta | `QuotationModel` (ambos repos) no tiene campo de leyenda |
| R4 | Texto según Región del cliente (MEX / PER) | ❌ Falta | Sin lógica de texto por región |
| R5 | Leyenda no bloqueante — PDF siempre se genera | ✅ Existe | Ninguna validación FluentValidation bloquea si el campo está ausente |
| R6 | Sin consulta al Catálogo de Clientes para la leyenda | ✅ Por diseño | La implementación no requiere consulta al catálogo de clientes |

---

## 3. GAPs de Implementación

### GAP-01 — BD: `fnEsProductoControlado` no detecta Origen

**Problema:** La función escalar solo evalúa `'mundiales'` y `'nacionales'`.
La Regla 2 exige también 'origen'.

```sql
-- ANTES (incompleto)
WHERE cc.Clave IN ('mundiales', 'nacionales')

-- DESPUÉS — ALTER FUNCTION dbo.fnEsProductoControlado
WHERE cc.Clave IN ('mundiales', 'nacionales', 'origen')
```

> Verificar usos de `fnEsProductoControlado` en otros módulos antes del ALTER.
> La ampliación no rompe el significado de "es controlado".

---

### GAP-02 — ProquifaDotNet: Detección de partidas controladas en `_CotizacionUnitaria`

**Problema:** `_CotizacionUnitaria` no evalúa si hay partidas de productos controlados.
`vProducto.ControlClave` ya está disponible en `ctx.ProductosById` (precargado en
`BuildQuotationContext`) — el mismo campo ya se usa en la misma función para `itemsCDU`.

```csharp
// ArchivoBOCotizacionUnitariaExtensions._CotizacionUnitaria()
// — después de construir la lista de items, antes de armar QuotationModel —

var clavesControladas = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
    { "mundiales", "nacionales", "origen" };

bool tienePartidasControladas = partidas.Any(p =>
{
    if (!ctx.OfertasById.TryGetValue(p.IdCotProductoOferta, out var oferta)) return false;
    ctx.ProductosById.TryGetValue(oferta.IdProducto, out var producto);
    var clave = producto?.ControlClave?.Trim() ?? string.Empty;
    return clavesControladas.Contains(clave);
});
```

> **Por qué no usar `fnEsProductoControlado` aquí:** `ProductosById` ya está
> precargado en el contexto con cero consultas adicionales. El campo `ControlClave`
> es exactamente lo que la función evalúa internamente.

---

### GAP-03 — ProquifaDotNet: Texto de leyenda por Región

**Problema:** No existe método ni constante que provea el texto de la leyenda
regulatoria según la Región del cliente.

```csharp
// Método privado en ArchivoBOCotizacionUnitariaExtensions
private static string ObtenerLeyendaRegulatoria(Region region)
{
    if (region == null) return null;

    switch (region.ClaveISO?.Trim().ToUpperInvariant())
    {
        case "MEX":
            // ⚠️ PENDIENTE P1: Texto definitivo a confirmar con el cliente
            return "Producto sujeto a regulación sanitaria. Para procesar el pedido " +
                   "se requiere: Licencia Sanitaria vigente y Aviso de Responsable Sanitario.";

        case "PER":
            // ⚠️ PENDIENTE P2: Denominación DIGEMID a confirmar con el cliente
            return "[PENDIENTE — Texto regulatorio DIGEMID para Perú]";

        default:
            return null;  // Región sin leyenda: campo omitido en el PDF
    }
}
```

---

### GAP-04 — ProquifaDotNet: Campo `RegulatoryNote` en `QuotationModel`

**Problema:** La clase `QuotationModel` de `Logic.Pqf.Catalogos` no tiene campo
para la leyenda regulatoria. Sin él, el DocumentBuilder no recibe el texto.

**4a — Agregar propiedad en `QuotationModel` (ProquifaDotNet):**

```csharp
// Logic.Pqf.Catalogos\Archivos\PDFs\Quotation\QuotationModel.cs
public string FormatCode { get; set; }

/// <summary>
/// Leyenda regulatoria para cotizaciones con sustancias controladas (FU-007).
/// Null cuando la cotización no tiene controlados o la región no tiene texto.
/// </summary>
public string RegulatoryNote { get; set; }   // NUEVO — FU-007
```

**4b — Asignar el campo en `_CotizacionUnitaria`:**

```csharp
// En la construcción del quotationModel, después de FormatCode:
string leyendaRegulatoria = null;
if (tienePartidasControladas)                 // flag de GAP-02
    leyendaRegulatoria = ObtenerLeyendaRegulatoria(ctx.Region);  // GAP-03

var quotationModel = new QuotationModel
{
    // ... propiedades existentes ...
    FormatCode      = ctx.Cotizacion?.CodigoDeFormatoServicios ?? string.Empty,
    RegulatoryNote  = leyendaRegulatoria,   // NUEVO — null si no aplica
};
```

---

### GAP-05 — DocumentBuilder: Campo `RegulatoryNote` en `QuotationModel` del repo espejo

**Problema:** El `QuotationModel` del DocumentBuilder-R14 es una clase independiente
(mismo nombre, mismo contrato, distinto repo). Si ProquifaDotNet serializa
`RegulatoryNote` en el JSON pero DocumentBuilder no tiene la propiedad, el campo
llega al JSON pero **no queda disponible como variable Scriban** en los templates.

```csharp
// DocumentBuilder-R14\Application\DTOs\Quotation\QuotationModel.cs
public string? FormatCode { get; set; }

/// <summary>
/// Leyenda regulatoria. Null cuando no aplica.
/// Disponible en templates Scriban como {{ RegulatoryNote }}.
/// </summary>
public string? RegulatoryNote { get; set; }  // NUEVO — FU-007
```

> **Por qué es necesario:** `GenerateDocumentService.QuotationExtension` hace
> `JsonSerializer.Serialize(requestGenerateReport.QuotationModel)` y pasa el JSON
> como contexto de datos a Scriban. Si el tipo C# no tiene la propiedad, el
> deserializador puede descartar el campo al recibir el request.

---

### GAP-06 — DocumentBuilder: Templates HTML Scriban de cotización definitiva

**Problema:** Los templates `*_COT_B.html` (body) de cada empresa/región no renderizan
`RegulatoryNote`. Sin este cambio el campo existe en el JSON pero nunca aparece en el PDF.

**Templates COT afectados** (uno por empresa con cotizaciones definitivas):

| TemplateKey | Archivo Body | Región |
|-------------|-------------|--------|
| `PQF_MEX_COT` | `PQF_MEX_COT_B.html` | MEX |
| `GOL_MEX_COT` | `GOL_MEX_COT_B.html` | MEX |
| `MUN_MEX_COT` | `MUN_MEX_COT_B.html` | MEX |
| `PHS_MEX_COT` | `PHS_MEX_COT_B.html` | MEX |
| `PRO_MEX_COT` | `PRO_MEX_COT_B.html` | MEX |
| `ERVP_MEX_COT` | `ERVP_MEX_COT_B.html` | MEX |
| `GOLPERU_PER_COT` | `GOLPERU_PER_COT_B.html` | PER |

**Snippet Scriban a agregar en cada body** (en la posición que defina UX — PENDIENTE P3):

```html
{{- if RegulatoryNote && RegulatoryNote != "" -}}
<div class="regulatory-note">
    <p class="black-bold-9">{{ RegulatoryNote }}</p>
</div>
{{- end -}}
```

> Los templates usan sintaxis **Scriban** (no Handlebars): `{{ variable }}`,
> `{{- if condicion -}} ... {{- end -}}`. Confirmado en `PQF_MEX_COT_B.html`.

**Script SQL de registro de templates (si se agregan templates nuevos):**

> Los templates existentes ya están en `DocumentTemplate` de la BD del DocumentBuilder.
> No se crean nuevos templates para FU-007 — solo se editan los bodies existentes.
> No se requiere script SQL adicional.

---

## 4. Entidades de Base de Datos Involucradas

### ProquifaDotNet DB

| Entidad | Rol en FU-007 | Cambio requerido |
|---------|---------------|-----------------|
| `cotCotizacion` | Cotización origen del PDF | Sin cambio |
| `cotPartidaCotizacion` | Partidas evaluadas para detectar controlados | Sin cambio |
| `cotProductoOferta` | Relaciona partida con producto | Sin cambio |
| `vProducto` | Vista que expone `ControlClave` | Sin cambio |
| `catControl` | Catálogo de tipos: mundiales / nacionales / origen | Sin cambio en datos |
| `Familia` + `MarcaFamilia` | Jerarquía producto-familia-control | Sin cambio |
| `dbo.fnEsProductoControlado` | Función escalar de detección | **ALTER** agregar `'origen'` |
| `Region` | Determina el texto de la leyenda (MEX / PER) | Sin cambio |

### DocumentBuilder DB

| Entidad | Rol en FU-007 | Cambio requerido |
|---------|---------------|-----------------|
| `DocumentTemplate` | Registro de templates por `TemplateKey` | Sin cambio — templates COT ya existen |

---

## 5. Archivos a Modificar

### Repositorio ProquifaDotNet-R14

| # | Archivo | Tipo de cambio | GAP |
|---|---------|----------------|-----|
| 1 | `BD: dbo.fnEsProductoControlado` | `ALTER FUNCTION` — agregar `'origen'` | GAP-01 |
| 2 | `Logic.Pqf.Catalogos\Archivos\PDFs\Quotation\QuotationModel.cs` | + propiedad `RegulatoryNote` | GAP-04a |
| 3 | `Logic.Pqf.Logistica\PDF\Cotizacion\ArchivoBOCotizacionUnitariaExtensions.cs` | + detección controlados + texto región + asignación en QuotationModel | GAP-02, GAP-03, GAP-04b |

### Repositorio DocumentBuilder-R14

| # | Archivo | Tipo de cambio | GAP |
|---|---------|----------------|-----|
| 4 | `Application\DTOs\Quotation\QuotationModel.cs` | + propiedad `RegulatoryNote` | GAP-05 |
| 5 | `API\Resources\Templates\PQF_MEX_COT\PQF_MEX_COT_B.html` | + bloque Scriban `RegulatoryNote` | GAP-06 |
| 6 | `API\Resources\Templates\GOL_MEX_COT\GOL_MEX_COT_B.html` | + bloque Scriban `RegulatoryNote` | GAP-06 |
| 7 | `API\Resources\Templates\MUN_MEX_COT\MUN_MEX_COT_B.html` | + bloque Scriban `RegulatoryNote` | GAP-06 |
| 8 | `API\Resources\Templates\PHS_MEX_COT\PHS_MEX_COT_B.html` | + bloque Scriban `RegulatoryNote` | GAP-06 |
| 9 | `API\Resources\Templates\PRO_MEX_COT\PRO_MEX_COT_B.html` | + bloque Scriban `RegulatoryNote` | GAP-06 |
| 10 | `API\Resources\Templates\ERVP_MEX_COT\ERVP_MEX_COT_B.html` | + bloque Scriban `RegulatoryNote` | GAP-06 |
| 11 | `API\Resources\Templates\GOLPERU_PER_COT\GOLPERU_PER_COT_B.html` | + bloque Scriban `RegulatoryNote` | GAP-06 |

---

## 6. Archivos Sin Cambio

| Archivo                                         | Motivo                                                                               |
| ----------------------------------------------- | ------------------------------------------------------------------------------------ |
| `ArchivoBOCotizacionExtensions.cs`              | Separación definitiva/investigación ya existe (R1 ✅)                                 |
| `ArchivoBOControladosNacionales.cs`             | Genera carta por producto individual — scope distinto                                |
| `BuildQuotationContext()`                       | Ya carga `Region`, `OfertasById`, `ProductosById` — sin cambio                       |
| `_CotizacionUnitariaInvestigacion()`            | Las cotizaciones de investigación no llevan leyenda (R1)                             |
| `DocumentGenerateQuoationDtoFluentValidator.cs` | `RegulatoryNote` es opcional — no requiere validación obligatoria                    |
| `GenerateDocumentService.QuotationExtension.cs` | El JSON serializado incluye automáticamente el nuevo campo — sin cambio en la lógica |
| Templates `*_COT_H.html` y `*_COT_F.html`       | Header y footer no muestran la leyenda regulatoria                                   |
| Templates `*_COT_INV_*`                         | Cotizaciones de investigación no llevan leyenda (R1)                                 |
| `DocumentTemplate` (BD DocumentBuilder)         | Templates COT ya existen — no se crean nuevos                                        |
| `catControl`, `Familia`, `MarcaFamilia` (datos) | Datos de control ya existen en BD                                                    |

---

## 7. Pendientes Externos al Backend

| ID | Pendiente | Bloqueante | Responsable |
|----|-----------|------------|-------------|
| P1 | Texto definitivo de la leyenda para clientes **México** | No — implementar con texto provisional hasta UAT | Cliente / UX |
| P2 | Texto y denominación regulatoria para clientes **Perú** (DIGEMID) | No — implementar placeholder hasta confirmación | Cliente |
| P3 | Ubicación de la leyenda en el PDF (antes/después de tabla de productos, pie, sección dedicada) | **Sí** — bloquea la edición de los templates HTML (GAP-06) | UX / Diseño |

---

## 8. Criterios de Aceptación Backend

| Criterio | Cómo verificar |
|----------|----------------|
| PDF de cotización definitiva con ≥1 partida controlada incluye leyenda | Inspeccionar el campo `RegulatoryNote` en el request JSON que llega al DocumentBuilder — no debe ser null ni vacío |
| PDF de cotización definitiva sin partidas controladas NO incluye leyenda | `RegulatoryNote` debe ser null en el request |
| Leyenda aparece una sola vez (campo string, no lista) | `QuotationModel.RegulatoryNote` es `string` — un valor por documento |
| Cotizaciones de investigación no reciben `RegulatoryNote` | `_CotizacionUnitariaInvestigacion` no modifica `QuotationModel` con leyenda |
| `fnEsProductoControlado` retorna 1 para productos con control tipo `origen` | `SELECT dbo.fnEsProductoControlado(@idProductoConOrigen)` → resultado = 1 |
| El PDF se renderiza sin error cuando `RegulatoryNote` es null | Generar PDF de cotización sin controlados → PDF generado correctamente, sin excepción |
| El texto MEX referencia Licencia Sanitaria y Aviso de Responsable Sanitario | Inspección del valor de `RegulatoryNote` en cotización de cliente MEX con controlados |
| El texto PER contiene el texto DIGEMID (o placeholder hasta confirmación) | Inspección del valor de `RegulatoryNote` en cotización de cliente PER con controlados |

---

## 9. Resumen de Cambios

```
REPOSITORIO         ARCHIVOS NUEVOS   ARCHIVOS MODIFICADOS
─────────────────── ─────────────── ──────────────────────────────────────────
ProquifaDotNet-R14       0            3  (fnEsProductoControlado BD,
                                          QuotationModel.cs,
                                          ArchivoBOCotizacionUnitariaExtensions.cs)
DocumentBuilder-R14      0            8  (QuotationModel.cs,
                                          7 templates *_COT_B.html)
─────────────────── ─────────────── ──────────────────────────────────────────
TOTAL                    0            11

CONSULTAS BD EXTRA:     0  (reutiliza ProductosById ya cargado en BuildQuotationContext)
CAMBIOS BLOQUEANTES:    0  (R5 garantizado — PDF siempre se genera)
SCRIPTS SQL NUEVOS:     1  (ALTER FUNCTION fnEsProductoControlado)
```
