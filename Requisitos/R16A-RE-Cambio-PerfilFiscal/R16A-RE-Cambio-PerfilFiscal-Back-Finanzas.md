# Impacto en ProquifaDotNet.Finanzas — R16A-RE-Cambio-PerfilFiscal

**Cambio:** Perfil Fiscal — Configuración fiscal por Familia de Producto (MX + PE)
**Aplicativo:** ProquifaDotNet.Finanzas (.NET Core 10)
**Módulo:** Facturación — construcción de Conceptos CFDI y documentos de Proforma/Factura
**Impacto:** Los campos fiscales (`TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat`) son resueltos por ProquifaDotNet en sus vistas (`vProducto`, `vFlete`) y llegan a Finanzas ya calculados. El impacto en Finanzas es **solo en DTOs y en la construcción de conceptos CFDI** — eliminación de hard-codes.

---

## Contexto

ProquifaDotNet (.NET Framework 4.8) es el dueño de las tablas de Perfil Fiscal (`PerfilFiscal`, `PerfilFiscalConfiguracionFamilia`, catálogos SAT). Esas tablas se consultan desde las vistas `vProducto` y `vFlete` mediante JOINs, y los campos fiscales resueltos (`TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat`) se exponen directamente en la vista.

**Finanzas no hace Scaffold de las tablas de Perfil Fiscal.** Los datos fiscales llegan a través de:
- Los DTOs de partida que ProquifaDotNet transmite vía API inter-sistema (partidas de pedido ya contienen los campos fiscales).
- Si Finanzas necesita consulta directa de producto/flete, puede hacer Scaffold de `vProducto` o `vMarcaFamilia` (las vistas ya exponen los campos resueltos).

Los hard-codes que se eliminan con este cambio (en ProquifaDotNet — ver `Back.md`):

| Archivo (ProquifaDotNet .NET 4.8) | Hard-code eliminado |
|---|---|
| `tpProformaPartidaPedidoToCFDIGeneradaConceptoBO.cs` | `ClaveUnidad = "H87"` → `tpPartida.ClaveUnidadSat` |
| `tpProformaPartidaPedidoToCFDIGeneradaConceptoBO.cs` | `familia.ClaveProductoServicioCFDI` → `tpPartida.ClaveProdServSat` |
| `tpProformaPartidaPedidoToCFDIGeneradaConceptoBO.cs` | `if (producto.GravaIVA)` → `if (tpPartida.TipoFactor != "Exento")` |
| `CFDIGeneradaConceptoToCFDIGeneradaImpuestoIVABO.cs` | `TasaOCuota = 0.16M` → tasa como parámetro |
| `CFDIGeneradaConceptoToCFDIGeneradaImpuestoIVABO.cs` | `Importe = 0.16M * source.Importe` → `tasa * source.Importe` |

---

## 1. Proforma Partidas (RE-016)

### 1.1 `ProformaPartidaDto` — campos fiscales nuevos

Los campos fiscales se agregan al DTO de partida de proforma. Finanzas los recibe de ProquifaDotNet (ya resueltos desde `vProducto`) y los usa al construir el PDF y al pre-armar conceptos CFDI:

```csharp
namespace ProquifaDotNet.Finanzas.Application.DTOs;

public class ProformaPartidaDto
{
    // -- Campos existentes (RE-016) --
    public Guid    IdTPProformaPedido { get; set; }
    public Guid    IdProducto         { get; set; }
    public string  Descripcion        { get; set; } = null!;
    public decimal NumeroDePiezas     { get; set; }
    public decimal PrecioUnitario     { get; set; }
    public decimal Importe            { get; set; }
    public decimal IVA                { get; set; }
    public decimal Total              { get; set; }

    // -- Campos fiscales nuevos (este cambio — resueltos por ProquifaDotNet) --
    /// <summary>Clave c_ClaveProdServ del SAT. Solo MX; null para PE.</summary>
    public string? ClaveProdServSat   { get; set; }

    /// <summary>Clave c_ClaveUnidad del SAT (ej. "H87"). Solo MX; null para PE.</summary>
    public string? ClaveUnidadSat     { get; set; }

    /// <summary>"Tasa" | "Cuota" | "Exento" — de catTipoFactorSat.</summary>
    public string  TipoFactor         { get; set; } = null!;

    /// <summary>Tasa del impuesto (ej. 0.160000). Null cuando TipoFactor = "Exento".</summary>
    public decimal? TasaOCuota        { get; set; }
}
```

### 1.2 Uso en `ProformaService`

Los campos fiscales llegan ya calculados en los datos de partida de ProquifaDotNet. El `ProformaService` solo los propaga al DTO:

```csharp
// Dentro de ProformaService — construcción de ProformaPartidaDto
// Los campos TipoFactor, TasaOCuota, ClaveProdServSat, ClaveUnidadSat
// ya vienen resueltos en la partida recibida de ProquifaDotNet

var importe = p.PrecioUnitario * p.NumeroDePiezas;
var gravaIVA = p.TipoFactor != "Exento";
var iva = gravaIVA ? decimal.Round(importe * (p.TasaOCuota ?? 0m), 6) : 0m;

return new ProformaPartidaDto
{
    IdTPProformaPedido = p.IdTPProformaPedido,
    IdProducto         = p.IdProducto,
    Descripcion        = p.Descripcion,
    NumeroDePiezas     = p.NumeroDePiezas,
    PrecioUnitario     = p.PrecioUnitario,
    Importe            = importe,
    IVA                = iva,
    Total              = importe + iva,
    ClaveProdServSat   = p.ClaveProdServSat,
    ClaveUnidadSat     = p.ClaveUnidadSat,
    TipoFactor         = p.TipoFactor,
    TasaOCuota         = p.TasaOCuota
};
```

---

## 2. Factura Partidas y CFDI Conceptos (RE-019, RE-028)

### 2.1 `FacturaConceptoDto` — campos fiscales nuevos

```csharp
namespace ProquifaDotNet.Finanzas.Application.DTOs;

public class FacturaConceptoDto
{
    // -- Campos ya existentes (RE-019) --
    public decimal Cantidad      { get; set; }
    public string  Descripcion   { get; set; } = null!;   // "catálogo + descripción + marca" (OBS-039)
    public decimal ValorUnitario { get; set; }
    public decimal Importe       { get; set; }

    // -- Campos fiscales — recibidos de ProquifaDotNet (ya resueltos en vProducto/vFlete) --
    /// <summary>Clave c_ClaveProdServ. Solo MX.</summary>
    public string? ClaveProdServSat { get; set; }

    /// <summary>Clave c_ClaveUnidad (ej. "H87"). Solo MX.</summary>
    public string? ClaveUnidadSat   { get; set; }

    /// <summary>"Tasa" | "Cuota" | "Exento".</summary>
    public string  TipoFactor       { get; set; } = null!;

    /// <summary>Tasa del impuesto. Null cuando Exento.</summary>
    public decimal? TasaOCuota      { get; set; }
}
```

### 2.2 Factura por Adelantada — perfil `factura-anticipo`

El concepto de anticipo en el CFDI de la Factura por Adelantada (RE-019) usa un perfil fiscal propio, identificado en ProquifaDotNet con `ClaveTipoEntidad = 'factura-anticipo'` (entrada en `PerfilFiscalConfiguracionFamilia` con `IdFamilia IS NULL`).

ProquifaDotNet resuelve este perfil (en el BO generador de la Factura por Adelantada — L05) y envía los datos fiscales del concepto de anticipo a Finanzas. Finanzas los usa para armar el nodo CFDI correspondiente:

```csharp
// Datos fiscales del concepto de anticipo — recibidos de ProquifaDotNet
// (resueltos con ClaveTipoEntidad = 'factura-anticipo')
var conceptoAnticipo = new FacturaConceptoDto
{
    Cantidad         = 1,
    Descripcion      = "Anticipo",
    ValorUnitario    = montoAnticipo,
    Importe          = montoAnticipo,
    ClaveProdServSat = anticipoFiscal.ClaveProdServSat,   // perfil factura-anticipo
    ClaveUnidadSat   = anticipoFiscal.ClaveUnidadSat,
    TipoFactor       = anticipoFiscal.TipoFactor,
    TasaOCuota       = anticipoFiscal.TasaOCuota
};
```

### 2.3 Construcción de conceptos en `AdvanceInvoiceService.GenerateAsync`

Los campos fiscales llegan en la partida de ProquifaDotNet. Solo se mapean al DTO de concepto:

```csharp
// Paso de armar Conceptos — los campos fiscales ya vienen en la partida
var conceptos = partidas.Select(p =>
{
    var importe = p.PrecioUnitario * p.NumeroDePiezas;
    var gravaIVA = p.TipoFactor != "Exento";

    return new FacturaConceptoDto
    {
        Cantidad         = p.NumeroDePiezas,
        Descripcion      = BuildDescripcion(p),    // catálogo + descripción + marca (OBS-039)
        ValorUnitario    = p.PrecioUnitario,
        Importe          = importe,
        ClaveProdServSat = p.ClaveProdServSat,
        ClaveUnidadSat   = p.ClaveUnidadSat,
        TipoFactor       = p.TipoFactor,
        TasaOCuota       = gravaIVA ? p.TasaOCuota : null
    };
}).ToList();
```

### 2.3 `CfdiTrasladoDto` — nodo Traslado dinámico

```csharp
namespace ProquifaDotNet.Finanzas.Application.DTOs;

public class CfdiTrasladoDto
{
    public decimal  Base       { get; set; }
    public string   Impuesto   { get; set; } = null!;  // "002" (IVA MX)
    public string   TipoFactor { get; set; } = null!;  // "Tasa" | "Exento"
    public decimal? TasaOCuota { get; set; }           // 0.160000 — null si Exento
    public decimal? Importe    { get; set; }           // Base * TasaOCuota — null si Exento
}
```

**Construcción dinámica — elimina el hard-code `0.16M`:**

```csharp
private static CfdiTrasladoDto? BuildTraslado(FacturaConceptoDto concepto, decimal importe)
{
    if (concepto.TipoFactor == "Exento")
        return null;

    return new CfdiTrasladoDto
    {
        Base       = importe,
        Impuesto   = "002",                                         // IVA MX — mapear si hay IGV PE
        TipoFactor = concepto.TipoFactor,
        TasaOCuota = concepto.TasaOCuota,
        Importe    = decimal.Round(importe * concepto.TasaOCuota!.Value, 6)
    };
}
```

> **Nota MX — CFDI 4.0:** conceptos exentos requieren nodo `Traslado` con `TipoFactor="Exento"`, `TasaOCuota="0.000000"`, `Importe="0.000000"`. Verificar con el PAC antes de implementar.

---

## 3. Mapeo hacia Timbrado (`AdvanceInvoiceItemDto`)

```csharp
var advanceItems = conceptos.Select(c => new AdvanceInvoiceItemDto
{
    Cantidad       = c.Cantidad,
    Descripcion    = c.Descripcion,
    PrecioUnitario = c.ValorUnitario,
    Importe        = c.Importe,
    ClaveUnidad    = c.ClaveUnidadSat ?? "H87",      // fallback temporal durante migración
    ClaveProdServ  = c.ClaveProdServSat ?? string.Empty,
    Traslados      = BuildTraslado(c, c.Importe) is { } t
                        ? new[] { t }
                        : Array.Empty<CfdiTrasladoDto>()
}).ToList();
```

> Eliminar el fallback `"H87"` una vez que `PerfilFiscalConfiguracionFamilia` esté poblada para todas las familias activas en MX.

---

## 4. Base de datos — ALTER TABLE (ProquifaDotNet.Finanzas)

Las tablas de Finanzas que persisten partidas deben recibir los 4 campos fiscales para almacenar los datos resueltos por ProquifaDotNet:

### `tpProformaPartidaPedido`

```sql
ALTER TABLE [dbo].[tpProformaPartidaPedido]
    ADD [ClaveProdServSat] VARCHAR(10)   NULL,   -- Clave c_ClaveProdServ SAT. Solo MX.
        [ClaveUnidadSat]   VARCHAR(10)   NULL,   -- Clave c_ClaveUnidad SAT (ej. "H87"). Solo MX.
        [TipoFactor]       VARCHAR(10)   NULL,   -- "Tasa" | "Cuota" | "Exento"
        [TasaOCuota]       DECIMAL(8,6)  NULL;   -- NULL cuando TipoFactor = "Exento"
```

### `fccFacturaPartida`

```sql
ALTER TABLE [dbo].[fccFacturaPartida]
    ADD [ClaveProdServSat] VARCHAR(10)   NULL,   -- Clave c_ClaveProdServ SAT. Solo MX.
        [ClaveUnidadSat]   VARCHAR(10)   NULL,   -- Clave c_ClaveUnidad SAT (ej. "H87"). Solo MX.
        [TipoFactor]       VARCHAR(10)   NULL,   -- "Tasa" | "Cuota" | "Exento"
        [TasaOCuota]       DECIMAL(8,6)  NULL;   -- NULL cuando TipoFactor = "Exento"
```

Ambas columnas `ClaveProdServSat` y `ClaveUnidadSat` son `NULL` para filas de Perú (PE no emite CFDI SAT). `TipoFactor` es `NULL` en registros anteriores al cambio — tratar como `"Tasa"` con la tasa de la región al leer datos históricos si fuera necesario.

---

## 5. Resumen de impacto por módulo

| Módulo / Flujo | Cambio |
|---|---|
| **Proforma (RE-016)** | `ProformaPartidaDto`: agregar 4 campos fiscales. `ProformaService`: calcular IVA desde `TipoFactor`/`TasaOCuota` en lugar de lógica fija. |
| **Factura por Adelantado (RE-019)** | `AdvanceInvoiceService`: mapear campos fiscales de partida → `FacturaConceptoDto`. `CfdiTrasladoDto` dinámico (eliminar `0.16M`). |
| **Factura de Ingreso (RE-028)** | Mismo patrón que RE-019. |
| **CFDI Concepto builder** | `BuildTraslado` construido desde `TasaOCuota`/`TipoFactor` — sin hard-codes. |

---

## Consideraciones

| # | Consideración | Detalle |
|---|---|---|
| 1 | Sin Scaffold de PerfilFiscal | Finanzas no accede a `PerfilFiscal`, `catImpuesto`, `PerfilFiscalConfiguracionFamilia` directamente — esas tablas pertenecen a ProquifaDotNet. |
| 2 | Datos fiscales via API | Los campos `TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat` llegan a Finanzas dentro de los DTOs de partida recibidos desde ProquifaDotNet. |
| 3 | vProducto / vMarcaFamilia opcional | Si Finanzas necesita consulta directa de datos de producto (no siempre el caso), puede hacer Scaffold de `vProducto` o `vMarcaFamilia`, que ya exponen los campos fiscales resueltos. |
| 4 | `ClaveUnidadSat` y `ClaveProdServSat` solo MX | En PE estos campos son null — PE no emite CFDI SAT. |
| 5 | Exento — CFDI 4.0 | Ver nota en sección 2.3 sobre el nodo `Traslado` para conceptos exentos. |
| 6 | Atomicidad con ProquifaDotNet | Los cambios en Finanzas (agregar campos a DTOs) y los cambios en ProquifaDotNet (Back.md Sección 5) deben desplegarse en el mismo sprint. |

---

## Trazabilidad — Modificación por Requisito

| Modificación en Finanzas | Requisito |
|---|---|
| `ProformaPartidaDto` — campos `ClaveProdServSat`, `ClaveUnidadSat`, `TipoFactor`, `TasaOCuota` | RE-016 |
| `ProformaService` — cálculo de IVA desde `TipoFactor`/`TasaOCuota` | RE-016 |
| `ALTER TABLE tpProformaPartidaPedido` — 4 campos fiscales | RE-016 |
| `FacturaConceptoDto` — campos fiscales | RE-019, RE-028 |
| Concepto `factura-anticipo` — perfil fiscal propio del anticipo | RE-019 |
| `AdvanceInvoiceService.GenerateAsync` — mapeo de campos fiscales al armar conceptos | RE-019 |
| `CfdiTrasladoDto` — construcción dinámica, eliminar hard-code `0.16M` | RE-019, RE-028 |
| `AdvanceInvoiceItemDto` — mapeo Finanzas → Timbrado con `ClaveUnidadSat`/`ClaveProdServSat` | RE-019 |
| `ALTER TABLE fccFacturaPartida` — 4 campos fiscales | RE-019 |

---

## Recursos

- `R16A-RE-Cambio-PerfilFiscal-Back.md` — impacto en ProquifaDotNet (.NET 4.8): Core.Pqf update + BOs de IVA + eliminación de hard-codes
- `R16A-RE-Cambio-PerfilFiscal_BD.md` — DDL/DML: `PerfilFiscalConfiguracionFamilia` + catálogos + ALTER VIEW vProducto/vFlete
- `R16A-RE-FU-016-Back-Finanzas.md` — estructura base de la solución Finanzas
- `R16A-RE-FU-019-Back.md` — flujo completo de Generar Factura (paso de armar conceptos CFDI)
- `ER-Finanzas.md` — diagrama ER actualizado
