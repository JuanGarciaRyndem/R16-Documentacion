# R16A-RE-FU-007 — Tareas de Implementación Backend

| Campo | Valor |
|-------|-------|
| **Requisito** | R16A-RE-FU-007 |
| **Nombre** | Notificación Regulatoria al Cliente en Cotización Definitiva |
| **Total de tareas** | 4 |
| **Revisión aplicada** | R16A-RE-FU-007 Revision.md |
| **Última actualización** | 2026-06-10 — OBS-017, OBS-018 |

---

## Tarea 1

### R16A-RE-FU-007  [ BD-OBJ-M ] Modificar `fnEsProductoControlado` — agregar tipo de control `origen`

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Cotizar lo Cotizable — Generación de PDF de cotización definitiva

**Consideraciones previas:**
Esta tarea es **prerequisito de la Tarea 2**. La función debe detectar correctamente los tres tipos de control antes de que la lógica de backend los evalúe.
Verificar los módulos que consumen `fnEsProductoControlado` antes de ejecutar el ALTER, para descartar efectos colaterales. El cambio amplía la detección pero no modifica el significado de "es controlado".

**Descripción del problema:**
La función escalar `dbo.fnEsProductoControlado` evalúa si un producto pertenece a una familia clasificada como sustancia controlada. Actualmente el filtro solo incluye las claves `'mundiales'` y `'nacionales'`, dejando fuera la clave `'origen'`.
La Regla R2 del requisito establece que la leyenda regulatoria debe detonarse con **cualquiera de los tres tipos**: Mundial, Nacional **u Origen**. Sin este cambio, las cotizaciones que solo contienen partidas de tipo `origen` no incluirán la leyenda regulatoria.

**Archivo a modificar:**
`BD ProquifaDotNet — dbo.fnEsProductoControlado`

**Cambios requeridos:**

```sql
-- Verificar usos antes del ALTER
SELECT OBJECT_NAME(referencing_id), referencing_minor_id
FROM   sys.sql_expression_dependencies
WHERE  referenced_entity_name = 'fnEsProductoControlado';

-- ALTER: agregar 'origen' al filtro de claves de control
ALTER FUNCTION dbo.fnEsProductoControlado (@IdProducto uniqueidentifier)
RETURNS bit
AS
BEGIN
    DECLARE @resultado bit = 0;

    IF EXISTS (
        SELECT 1
        FROM   dbo.MarcaFamilia  mf
        JOIN   dbo.Familia       f   ON f.IdFamilia    = mf.IdFamilia
        JOIN   dbo.catControl    cc  ON cc.IdCatControl = f.IdCatControl
        WHERE  mf.IdMarca  = (SELECT IdMarca FROM dbo.catProductoOferta WHERE IdProducto = @IdProducto)
          AND  cc.Clave IN ('mundiales', 'nacionales', 'origen')  -- FU-007: se agrega 'origen'
    )
        SET @resultado = 1;

    RETURN @resultado;
END;
GO

-- Verificación
-- Reemplazar con un IdProducto real de tipo origen en el ambiente de pruebas
SELECT dbo.fnEsProductoControlado('<IdProducto-tipo-origen>') AS Resultado; -- esperado: 1
SELECT dbo.fnEsProductoControlado('<IdProducto-sin-control>') AS Resultado; -- esperado: 0
```

**Criterios de aceptación:**
- [ ] Usos de `fnEsProductoControlado` revisados — ningún módulo existente se ve afectado negativamente
- [ ] `fnEsProductoControlado` retorna `1` para un producto con `ControlClave = 'origen'`
- [ ] `fnEsProductoControlado` sigue retornando `1` para productos con `ControlClave = 'mundiales'` o `'nacionales'`
- [ ] `fnEsProductoControlado` retorna `0` para productos sin clasificación de control
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-01 del archivo `R16A-RE-FU-007-Back.md`
- Regla R2 del requisito: detonante de la leyenda cubre los tres tipos de control

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007.md`

---

## Tarea 2

### R16A-RE-FU-007  [SERV-TRANSACT] Agregar detección de controlados y leyenda regulatoria en `_CotizacionUnitaria` — ProquifaDotNet

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Logistica + Logic.Pqf.Catalogos

**Módulos:**
Cotizar lo Cotizable — Generación de PDF de cotización definitiva

**Consideraciones previas:**
**Depende de Tarea 1.** La función `fnEsProductoControlado` ya debe cubrir los tres tipos de control, aunque la detección en código C# usa `ControlClave` directo del contexto (sin invocar la función).
Confirmar Pendiente **P1**: texto definitivo de la leyenda para clientes México antes del UAT. Se implementa con texto provisional hasta que el cliente confirme.
Confirmar Pendiente **P4**: verificar la clave exacta de `Region.ClaveISO` para México en `catRegion` de la BD de QA antes de codificar el `switch`.
> **OBS-018** — Perú excluido: la leyenda regulatoria aplica **solo a Región México**. El `switch` ya no incluye rama `PER`. El Pendiente P2 (DIGEMID) queda cancelado.

**Descripción del problema:**
El método `_CotizacionUnitaria` en `ArchivoBOCotizacionUnitariaExtensions` genera el PDF de cotización definitiva, pero no evalúa si la cotización contiene partidas de productos controlados ni agrega ninguna leyenda regulatoria al modelo enviado al DocumentBuilder.
Se requieren tres cambios coordinados: (1) detectar si hay partidas controladas usando el contexto ya disponible, (2) obtener el texto de la leyenda según la Región del cliente, y (3) asignar el campo `RegulatoryNote` en `QuotationModel` antes de enviarlo al DocumentBuilder.

**Archivos a modificar:**
- `Logic.Pqf.Catalogos\Archivos\PDFs\Quotation\QuotationModel.cs`
- `Logic.Pqf.Logistica\PDF\Cotizacion\ArchivoBOCotizacionUnitariaExtensions.cs`

**Cambios requeridos:**

**Paso 1 — Agregar propiedad en `QuotationModel` (Logic.Pqf.Catalogos):**

```csharp
// Logic.Pqf.Catalogos\Archivos\PDFs\Quotation\QuotationModel.cs
// Agregar después de la propiedad FormatCode:

/// <summary>
/// Leyenda regulatoria para cotizaciones con sustancias controladas (FU-007).
/// Null cuando la cotización no tiene controlados o la región no tiene texto definido.
/// </summary>
public string RegulatoryNote { get; set; }
```

**Paso 2 — Agregar método privado `ObtenerLeyendaRegulatoria` en `ArchivoBOCotizacionUnitariaExtensions`:**

```csharp
// Logic.Pqf.Logistica\PDF\Cotizacion\ArchivoBOCotizacionUnitariaExtensions.cs
// Agregar como método privado estático en la clase:

private static string ObtenerLeyendaRegulatoria(Region region)
{
    if (region == null) return null;

    switch (region.ClaveISO?.Trim().ToUpperInvariant())
    {
        case "MEX":
            // ⚠️ PENDIENTE P1: Confirmar texto definitivo con el cliente antes de UAT
            return "Producto sujeto a regulación sanitaria. Para procesar el pedido " +
                   "se requiere: Licencia Sanitaria vigente y Aviso de Responsable Sanitario.";

        // OBS-018: Región Perú excluida — Sustancias Controladas no habilitadas en Perú (R16)
        default:
            return null;
    }
}
```

**Paso 3 — Agregar detección y asignación en `_CotizacionUnitaria`:**

```csharp
// Logic.Pqf.Logistica\PDF\Cotizacion\ArchivoBOCotizacionUnitariaExtensions.cs
// En el método _CotizacionUnitaria(), después de construir las listas de items
// y antes de armar QuotationModel:

// Detección de partidas controladas (GAP-02)
// Reutiliza ProductosById ya cargado en BuildQuotationContext — cero consultas extra
var clavesControladas = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
    { "mundiales", "nacionales", "origen" };

bool tienePartidasControladas = partidas.Any(p =>
{
    if (!ctx.OfertasById.TryGetValue(p.IdCotProductoOferta, out var oferta)) return false;
    ctx.ProductosById.TryGetValue(oferta.IdProducto, out var producto);
    var clave = producto?.ControlClave?.Trim() ?? string.Empty;
    return clavesControladas.Contains(clave);
});

// Texto de leyenda según región (GAP-03)
string leyendaRegulatoria = null;
if (tienePartidasControladas)
    leyendaRegulatoria = ObtenerLeyendaRegulatoria(ctx.Region);

// Asignación en QuotationModel (GAP-04b)
var quotationModel = new QuotationModel
{
    // ... propiedades existentes sin cambio ...
    FormatCode     = ctx.Cotizacion?.CodigoDeFormatoServicios ?? string.Empty,
    RegulatoryNote = leyendaRegulatoria,   // null si no aplica (R5)
};
```

**Criterios de aceptación:**
- [ ] Pendiente P1 (texto MEX) anotado como comentario en código hasta confirmación del cliente
- [ ] Pendiente P4 (clave ISO de Region México) verificado en BD de QA antes del merge
- [ ] `QuotationModel.RegulatoryNote` contiene el texto de leyenda cuando la cotización tiene ≥1 partida controlada (Mundial, Nacional u Origen)
- [ ] `QuotationModel.RegulatoryNote` es `null` cuando la cotización no tiene partidas controladas
- [ ] `QuotationModel.RegulatoryNote` es `null` en cotizaciones de investigación (no se toca `_CotizacionUnitariaInvestigacion`)
- [ ] El texto para región MEX referencia Licencia Sanitaria y Aviso de Responsable Sanitario
- [ ] Para región PER (y cualquier otra): `ObtenerLeyendaRegulatoria` retorna `null` — no se genera leyenda (OBS-018)
- [ ] La generación del PDF no lanza excepción cuando `RegulatoryNote` es null (R5)
- [ ] Los proyectos `Logic.Pqf.Catalogos` y `Logic.Pqf.Logistica` compilan sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-02, GAP-03 y GAP-04 del archivo `R16A-RE-FU-007-Back.md`
- Reglas R2, R3, R4 y R5 del requisito
- Criterios B1, B2, B4 y C1 del requisito (Criterio C2 Perú eliminado por OBS-018)

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007.md`

---

## Tarea 3

### R16A-RE-FU-007  [ ALG-BASIC-LOGIC ] Agregar `RegulatoryNote` en `QuotationModel` — DocumentBuilder

**Aplicativos:**
DocumentBuilder — Application

**Módulos:**
Motor de generación de PDF — Cotización Definitiva

**Consideraciones previas:**
**Depende de Tarea 2.** El campo `RegulatoryNote` debe existir en `QuotationModel` de ProquifaDotNet antes de sincronizar el DTO en DocumentBuilder.
El `QuotationModel` de DocumentBuilder es una clase **independiente** del de ProquifaDotNet — mismo contrato, distinto repositorio. El campo debe agregarse en ambos repos por separado.
`GenerateDocumentService.QuotationExtension` no requiere cambios: hace `JsonSerializer.Serialize(QuotationModel)` y pasa el JSON directamente a Scriban. Al agregar la propiedad al tipo C#, el campo queda automáticamente disponible como variable en los templates.

**Descripción del problema:**
El DTO `DocumentBuilder.Application.DTOs.Quotation.QuotationModel` no tiene la propiedad `RegulatoryNote`. Cuando ProquifaDotNet serializa el modelo y hace POST a `api/Report/quotation`, el campo llega en el JSON pero el deserializador de la API puede descartarlo al no encontrar la propiedad en el tipo destino.
Sin la propiedad en el tipo C# del DocumentBuilder, la variable `RegulatoryNote` no queda disponible en el contexto Scriban y los templates no pueden renderizarla.

**Archivo a modificar:**
`Application\DTOs\Quotation\QuotationModel.cs`

**Cambios requeridos:**

```csharp
// DocumentBuilder-\Application\DTOs\Quotation\QuotationModel.cs
// Agregar después de la propiedad FormatCode:

/// <summary>
/// Leyenda regulatoria para cotizaciones con sustancias controladas (FU-007).
/// Null cuando la cotización no tiene controlados o la región no tiene texto definido.
/// Disponible en templates Scriban como {{ RegulatoryNote }}.
/// </summary>
public string? RegulatoryNote { get; set; }
```

**Criterios de aceptación:**
- [ ] La propiedad `RegulatoryNote` existe en `QuotationModel` del DocumentBuilder
- [ ] El proyecto `DocumentBuilder.Application` compila sin errores
- [ ] Al enviar un request con `"RegulatoryNote": "Texto de prueba"`, el valor está disponible como variable Scriban `{{ RegulatoryNote }}` en el template
- [ ] Al enviar un request con `"RegulatoryNote": null`, el template no lanza error (el bloque `{{- if RegulatoryNote -}}` lo omite)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-05 del archivo `R16A-RE-FU-007-Back.md`
- La serialización y paso de datos a Scriban ocurre en `GenerateDocumentService.QuotationExtension.GenerateQuotationTemplate()`

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007-Back.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007.md`

---

## Tarea 4

### R16A-RE-FU-007  GAP-06  [ CREATE-PDF ] Agregar bloque Scriban `RegulatoryNote` en los 6 templates de cotización definitiva México — DocumentBuilder

**Aplicativos:**
DocumentBuilder — API Resources / Templates

**Módulos:**
Motor de generación de PDF — Templates HTML de cotización definitiva

**Consideraciones previas:**
**Depende de Tarea 3.** La propiedad `RegulatoryNote` debe existir en el DTO antes de referenciarla en los templates.
Condicionada al Pendiente **P3**: la ubicación exacta de la leyenda en el PDF (antes/después de la tabla de productos, sección dedicada o pie) debe ser definida por UX/Diseño antes de editar los templates. Este pendiente **bloquea** el inicio de esta tarea.
Los templates usan sintaxis **Scriban** — no Handlebars ni Razor. Confirmado en `PQF_MEX_COT_B.html`.
No se crean templates nuevos: los 7 templates COT ya están registrados en la tabla `DocumentTemplate` de la BD del DocumentBuilder.

**Descripción del problema:**
Los archivos `*_COT_B.html` (body) de los templates de cotización definitiva no incluyen ningún bloque que renderice `RegulatoryNote`. Aunque el campo llegue en el JSON, sin el bloque Scriban la leyenda nunca aparece en el PDF generado.
Se deben editar los 6 bodies de cotización definitiva México.

> **OBS-018** — El template `GOLPERU_PER_COT_B.html` queda **excluido**: la leyenda no aplica a Región Perú en R16.

**Archivos a modificar:**
```
API\Resources\Templates\PQF_MEX_COT\PQF_MEX_COT_B.html
API\Resources\Templates\GOL_MEX_COT\GOL_MEX_COT_B.html
API\Resources\Templates\MUN_MEX_COT\MUN_MEX_COT_B.html
API\Resources\Templates\PHS_MEX_COT\PHS_MEX_COT_B.html
API\Resources\Templates\PRO_MEX_COT\PRO_MEX_COT_B.html
API\Resources\Templates\ERVP_MEX_COT\ERVP_MEX_COT_B.html
```

**Cambios requeridos:**

Agregar el siguiente bloque Scriban en cada uno de los 7 bodies, en la posición que defina UX (PENDIENTE P3):

```html
{{- if RegulatoryNote && RegulatoryNote != "" -}}
<div class="regulatory-note">
    <p class="black-bold-9">{{ RegulatoryNote }}</p>
</div>
{{- end -}}
```

Agregar el estilo CSS correspondiente en la sección `<style>` de cada template (si no existe ya una clase `regulatory-note`):

```css
.regulatory-note {
    border-left: 3px solid #840083;
    margin-top: 12px;
    padding: 6px 10px;
}
```

> El color de borde (`#840083`) sigue el estilo `purple-bold-9` ya definido en los templates existentes.
> Ajustar diseño y posición según la decisión de UX del Pendiente P3.

**Criterios de aceptación:**
- [ ] Pendiente P3 (ubicación de la leyenda en el PDF) confirmado por UX/Diseño antes de iniciar
- [ ] Los 6 templates MEX `*_COT_B.html` incluyen el bloque Scriban `RegulatoryNote`
- [ ] Template `GOLPERU_PER_COT_B.html` **no** se modifica (Perú excluido — OBS-018)
- [ ] PDF generado para cotización definitiva MEX con controlados muestra la leyenda regulatoria visible
- [ ] PDF generado para cotización definitiva **sin** controlados no muestra la leyenda (bloque `{{- if -}}` la omite)
- [ ] PDF generado para cotización de investigación no muestra la leyenda regulatoria (template `*_COT_INV_*` no se modifica)
- [ ] La leyenda aparece una sola vez por documento (Criterio B3)
- [ ] El PDF se genera sin errores de render Scriban cuando `RegulatoryNote` es null
- [ ] PR aprobado por líder técnico y revisado visualmente con PDF de prueba

**Más información de la tarea:**
- GAP-06 del archivo `R16A-RE-FU-007-Back.md`
- Criterios B1, B2, B3 y B4 del requisito
- Sintaxis Scriban confirmada en `PQF_MEX_COT_B.html`: `{{ variable }}`, `{{- if condicion -}} ... {{- end -}}`
- Los templates `*_COT_H.html` (header) y `*_COT_F.html` (footer) no se modifican
- Los templates `*_COT_INV_*` no se modifican (cotizaciones de investigación fuera de alcance — Regla R1)

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-007/R16A-RE-FU-007.md`
