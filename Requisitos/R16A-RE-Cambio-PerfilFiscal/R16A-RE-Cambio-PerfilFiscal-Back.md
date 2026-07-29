# Impacto en Back — R16A-RE-Cambio-PerfilFiscal

**Cambio:** Perfil Fiscal — Configuración fiscal por Familia de Producto (MX + PE)
**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10)
**Módulo:** Infraestructura fiscal — resolución de configuración fiscal de Familia al construir documentos fiscales (CFDI MX / documentos PE)
**Impacto:** Scaffold de 5 entidades nuevas/extendidas + 1 interfaz + 1 servicio de resolución (region-aware) + actualización del `ConceptoBuilder`. **Desbloquea el ⏸ Pendiente de RE-FU-019-Back.md (GAP-7 / GAP-8).**

---

## Resumen

Este cambio resuelve el bloque pendiente de RE-019 sobre la resolución de datos fiscales por producto. La decisión confirmada **simplifica** la lógica prevista y **extiende** el alcance a Perú (IGV):

| | Diseño anterior (pendiente RE-019) | Diseño confirmado (este cambio) |
|---|---|---|
| Nivel de configuración | Producto con fallback a Familia | **Solo Familia** — sin override por Producto |
| `ClaveProdServSat` | Producto → Familia | `FamiliaRegion.ClaveProdServSat` (solo MX) |
| `ClaveUnidadSat` | Producto → Familia | `FamiliaRegion.ClaveUnidadSat` (solo MX) |
| `IdPerfilFiscal` | Producto → Familia (única FK) | `FamiliaRegion.IdPerfilFiscal` — una fila por Familia+Región |
| Regiones | Solo México | México (IVA) + Perú (IGV) — fila separada en `FamiliaRegion` por región |
| `Familia` | Recibe columnas nuevas | **Sin cambios** — configuración fiscal en `FamiliaRegion` |

La resolución se hace en un JOIN `Producto → MarcaFamilia → FamiliaRegion(idRegion) → PerfilFiscal` — sin CASE ni lógica condicional de precedencia. La región se recibe como parámetro del servicio.

---

## Tablas accedidas desde Finanzas

| Tabla | BD | Origen | Acceso |
|---|---|---|---|
| `catImpuesto` | ProquifaDotNet | Este cambio (CREATE) | Scaffold — solo lectura |
| `catTipoFactorSat` | ProquifaDotNet | Este cambio (CREATE) | Scaffold — solo lectura |
| `catObjetoImpuestoSat` | ProquifaDotNet | Este cambio (CREATE) | Scaffold — solo lectura |
| `PerfilFiscal` | ProquifaDotNet | Este cambio (CREATE) | Scaffold — solo lectura |
| `FamiliaRegion` | ProquifaDotNet | Este cambio (CREATE) | Scaffold — entidad nueva; configura perfil fiscal por Familia+Región |
| `Familia` | ProquifaDotNet | Preexistente | Scaffold — ya existe; sin cambios |
| `MarcaFamilia` | ProquifaDotNet | Preexistente | Scaffold — ya existe |

---

## 1. Infrastructure — Scaffold (ProquifaDotNet.Finanzas)

### 1.1 Entidades nuevas

Agregar al scaffold de `Finanzas.Infrastructure` las siguientes entidades (clase generada + DbSet en `FinanzasDbContext`):

**`catImpuesto`**
```csharp
public class CatImpuesto
{
    public Guid     IdCatImpuesto  { get; set; }
    public string   Clave          { get; set; } = null!;  // "002" = IVA (MX) | "IGV" (PE)
    public string   Descripcion    { get; set; } = null!;
    public bool     Activo         { get; set; }
    public DateTime FechaRegistro  { get; set; }
    public DateTime FechaUltimaActualizacion { get; set; }
}
```

**`catTipoFactorSat`**
```csharp
public class CatTipoFactorSat
{
    public Guid     IdCatTipoFactorSat       { get; set; }
    public string   Clave                    { get; set; } = null!;  // "Tasa" | "Cuota" | "Exento"
    public string   Descripcion              { get; set; } = null!;
    public bool     Activo                   { get; set; }
    public DateTime FechaRegistro            { get; set; }
    public DateTime FechaUltimaActualizacion { get; set; }
}
```

**`catObjetoImpuestoSat`**
```csharp
public class CatObjetoImpuestoSat
{
    public Guid     IdCatObjetoImpuestoSat   { get; set; }
    public string   Clave                    { get; set; } = null!;  // "02" = Sí objeto
    public string   Descripcion              { get; set; } = null!;
    public bool     Activo                   { get; set; }
    public DateTime FechaRegistro            { get; set; }
    public DateTime FechaUltimaActualizacion { get; set; }
}
```

**`PerfilFiscal`**
```csharp
public class PerfilFiscal
{
    public Guid     IdPerfilFiscal           { get; set; }
    public Guid     IdRegion                 { get; set; }   // FK → Region (MX | PE)
    public Guid     IdCatImpuesto            { get; set; }   // FK → catImpuesto (NOT NULL — IVA o IGV)
    public Guid     IdCatTipoFactorSat       { get; set; }   // compartido MX+PE
    public decimal? TasaOCuota               { get; set; }   // NULL para Exento
    public Guid?    IdCatObjetoImpuestoSat   { get; set; }   // nullable — NULL para filas PE
    public string?  Fundamento               { get; set; }
    public bool     Activo                   { get; set; }
    public DateTime FechaRegistro            { get; set; }
    public DateTime FechaUltimaActualizacion { get; set; }

    // Navegación
    public CatImpuesto?          CatImpuesto          { get; set; }
    public CatTipoFactorSat?     CatTipoFactorSat     { get; set; }
    public CatObjetoImpuestoSat? CatObjetoImpuestoSat { get; set; }  // null para PE
}
```

### 1.2 Entidad nueva — `FamiliaRegion`

```csharp
public class FamiliaRegion
{
    public Guid     IdFamiliaRegion          { get; set; }
    public Guid     IdFamilia                { get; set; }
    public Guid     IdRegion                 { get; set; }
    public Guid     IdPerfilFiscal           { get; set; }
    public string?  ClaveProdServSat         { get; set; }   // nullable — solo MX; NULL para PE o familia no facturable
    public string?  ClaveUnidadSat           { get; set; }   // nullable — solo MX
    public bool     Activo                   { get; set; }
    public DateTime FechaRegistro            { get; set; }
    public DateTime FechaUltimaActualizacion { get; set; }

    // Navegación
    public PerfilFiscal? PerfilFiscal { get; set; }
}
```

La entidad `Familia` **no recibe cambios** — la configuración fiscal vive en `FamiliaRegion`.

### 1.3 DbSets en FinanzasDbContext

```csharp
public DbSet<CatImpuesto>          CatImpuesto          { get; set; }
public DbSet<CatTipoFactorSat>     CatTipoFactorSat     { get; set; }
public DbSet<CatObjetoImpuestoSat> CatObjetoImpuestoSat { get; set; }
public DbSet<PerfilFiscal>         PerfilFiscal         { get; set; }
public DbSet<FamiliaRegion>        FamiliaRegion        { get; set; }
// Familia y MarcaFamilia ya existen — sin cambios
```

---

## 2. Domain — DTOs

### `FamiliaFiscalDataDto`

DTO de salida del servicio de resolución fiscal, usado por el `ConceptoBuilder` al armar cada partida del documento fiscal. Cubre tanto MX (CFDI SAT) como PE (IGV):

```csharp
/// <summary>
/// Datos fiscales resueltos de la Familia del producto según región.
/// MX: CodigoImpuesto = "IVA", ObjetoImpuestoSat incluido.
/// PE: CodigoImpuesto = "IGV", ObjetoImpuestoSat = null (no aplica SAT).
/// </summary>
public class FamiliaFiscalDataDto
{
    /// <summary>Clave c_ClaveProdServ del SAT (ej. "41116132"). Solo MX.</summary>
    public string? ClaveProdServSat { get; set; }

    /// <summary>Clave c_ClaveUnidad del SAT (ej. "H87", "E48", "ACT"). Solo MX.</summary>
    public string? ClaveUnidadSat { get; set; }

    /// <summary>
    /// Código del impuesto desde catImpuesto.Clave: "IVA" (MX) o "IGV" (PE).
    /// Siempre presente — catImpuesto.IdCatImpuesto es NOT NULL en PerfilFiscal.
    /// </summary>
    public string CodigoImpuesto { get; set; } = null!;

    /// <summary>TipoFactor — "Tasa" o "Exento". Compartido MX+PE.</summary>
    public string TipoFactor { get; set; } = null!;

    /// <summary>
    /// Tasa o cuota del impuesto (ej. 0.160000 MX, 0.180000 PE).
    /// NULL cuando TipoFactor = "Exento".
    /// </summary>
    public decimal? TasaOCuota { get; set; }

    /// <summary>
    /// Clave del catálogo SAT catObjetoImpuestoSat (ej. "02").
    /// Solo MX — null para PE.
    /// </summary>
    public string? ObjetoImpuestoSat { get; set; }
}
```

---

## 3. Application — Interfaz y Servicio

### `IFamiliaFiscalDataService`

```csharp
namespace ProquifaDotNet.Finanzas.Application.Interfaces;

/// <summary>
/// Resuelve la configuración fiscal de la Familia a la que pertenece un producto,
/// según la región del documento (MX → IVA/CFDI; PE → IGV).
/// </summary>
public interface IFamiliaFiscalDataService
{
    /// <summary>
    /// Devuelve la configuración fiscal (ClaveProdServSat, ClaveUnidadSat,
    /// PerfilFiscal) de la Familia del producto indicado, para la región dada.
    /// </summary>
    /// <param name="idProducto">ID del producto a resolver.</param>
    /// <param name="idRegion">ID de la región (MX o PE).</param>
    /// <exception cref="FamiliaNoFacturableException">
    /// Si la familia no tiene configuración fiscal para la región solicitada (Regla 7).
    /// </exception>
    Task<FamiliaFiscalDataDto> GetFiscalDataByProductoAsync(
        Guid idProducto, Guid idRegion, CancellationToken ct = default);

    /// <summary>
    /// Versión batch — resuelve la configuración fiscal para múltiples productos
    /// en una sola consulta. Todos deben corresponder a la misma región.
    /// </summary>
    Task<IReadOnlyDictionary<Guid, FamiliaFiscalDataDto>> GetFiscalDataByProductosBatchAsync(
        IEnumerable<Guid> idProductos, Guid idRegion, CancellationToken ct = default);
}
```

### `FamiliaFiscalDataService`

```csharp
namespace ProquifaDotNet.Finanzas.Application.Services;

public class FamiliaFiscalDataService : IFamiliaFiscalDataService
{
    private readonly FinanzasDbContext _db;

    public FamiliaFiscalDataService(FinanzasDbContext db) => _db = db;

    public async Task<FamiliaFiscalDataDto> GetFiscalDataByProductoAsync(
        Guid idProducto, Guid idRegion, CancellationToken ct = default)
    {
        var row = await _db.MarcaFamilia
            .Where(mf => mf.IdProducto == idProducto && mf.Activo)
            .Join(_db.FamiliaRegion.Where(fr => fr.IdRegion == idRegion && fr.Activo),
                  mf => mf.IdFamilia,
                  fr => fr.IdFamilia,
                  (mf, fr) => fr)
            .Include(fr => fr.PerfilFiscal!.CatImpuesto)
            .Include(fr => fr.PerfilFiscal!.CatTipoFactorSat)
            .Include(fr => fr.PerfilFiscal!.CatObjetoImpuestoSat)
            .FirstOrDefaultAsync(ct)
            ?? throw new FamiliaNoFacturableException(idProducto, idRegion);

        return MapToDto(row);
    }

    public async Task<IReadOnlyDictionary<Guid, FamiliaFiscalDataDto>> GetFiscalDataByProductosBatchAsync(
        IEnumerable<Guid> idProductos, Guid idRegion, CancellationToken ct = default)
    {
        var ids = idProductos.ToList();

        var rows = await _db.MarcaFamilia
            .Where(mf => ids.Contains(mf.IdProducto) && mf.Activo)
            .Join(_db.FamiliaRegion.Where(fr => fr.IdRegion == idRegion && fr.Activo),
                  mf => mf.IdFamilia,
                  fr => fr.IdFamilia,
                  (mf, fr) => new { mf.IdProducto, FamiliaRegion = fr })
            .Include(x => x.FamiliaRegion.PerfilFiscal!.CatImpuesto)
            .Include(x => x.FamiliaRegion.PerfilFiscal!.CatTipoFactorSat)
            .Include(x => x.FamiliaRegion.PerfilFiscal!.CatObjetoImpuestoSat)
            .ToListAsync(ct);

        return rows.ToDictionary(
            r => r.IdProducto,
            r => MapToDto(r.FamiliaRegion));
    }

    // -------------------------------------------------------------------------

    private static FamiliaFiscalDataDto MapToDto(FamiliaRegion fr) => new()
    {
        ClaveProdServSat  = fr.ClaveProdServSat,                     // null para PE
        ClaveUnidadSat    = fr.ClaveUnidadSat,                       // null para PE
        CodigoImpuesto    = fr.PerfilFiscal!.CatImpuesto!.Clave,     // "IVA" | "IGV"
        TipoFactor        = fr.PerfilFiscal.CatTipoFactorSat!.Clave,
        TasaOCuota        = fr.PerfilFiscal.TasaOCuota,
        ObjetoImpuestoSat = fr.PerfilFiscal.CatObjetoImpuestoSat?.Clave  // null para PE
    };
}
```

### `FamiliaNoFacturableException`

```csharp
public class FamiliaNoFacturableException : Exception
{
    public FamiliaNoFacturableException(Guid idProducto, Guid idRegion)
        : base($"El producto {idProducto} pertenece a una familia sin configuración fiscal para la región {idRegion}. No puede procesarse.") { }
}
```

---

## 4. Actualización del ConceptoBuilder

El servicio que construye cada concepto/partida (`InvoiceConceptBuilder` o equivalente en RE-019) debe inyectar `IFamiliaFiscalDataService` y pasar el `idRegion` del documento:

```csharp
// Patrón de uso dentro del builder de partidas (MX — CFDI)
var fiscalData = await _familiaFiscalDataService
    .GetFiscalDataByProductoAsync(partida.IdProducto, idRegionMX, ct);

// Nodo Concepto CFDI (campos SAT)
var concepto = new CfdiConcepto
{
    ClaveProdServ = fiscalData.ClaveProdServSat!,   // ej. "41116132"
    ClaveUnidad   = fiscalData.ClaveUnidadSat!,     // ej. "H87"
    // ...resto de campos del concepto
    Traslados = fiscalData.TasaOCuota.HasValue
        ? new[] { new CfdiTraslado
            {
                Base        = partida.Importe,
                Impuesto    = fiscalData.CodigoImpuesto,  // "IVA" → SAT usa "002"; mapear al serializar
                TipoFactor  = fiscalData.TipoFactor,      // "Tasa"
                TasaOCuota  = fiscalData.TasaOCuota,      // 0.160000
                Importe     = partida.Importe * fiscalData.TasaOCuota.Value
            }}
        : Array.Empty<CfdiTraslado>()   // Exento: nodo Traslados sin importe
};

// Para documentos PE — calcular impuesto con CodigoImpuesto = "IGV"
var fiscalDataPE = await _familiaFiscalDataService
    .GetFiscalDataByProductoAsync(partida.IdProducto, idRegionPE, ct);
decimal igv = fiscalDataPE.TasaOCuota.HasValue
    ? partida.Importe * fiscalDataPE.TasaOCuota.Value   // ej. importe * 0.18
    : 0m;
```

> Para documentos con múltiples partidas, usar `GetFiscalDataByProductosBatchAsync` antes del loop para evitar N+1 queries.

---

## 5. Registro de dependencias (DI)

```csharp
// En Program.cs o extensión de servicios de Application
services.AddScoped<IFamiliaFiscalDataService, FamiliaFiscalDataService>();
```

---

## Flujo de resolución fiscal

```
Documento fiscal (IdProducto + IdRegion)
        │
        ▼
MarcaFamilia (IdProducto → IdFamilia)
        │
        ▼
Familia (ClaveProdServSat, ClaveUnidadSat, IdPerfilFiscalMX, IdPerfilFiscalPE)
        │
        │
        ▼
FamiliaRegion (IdFamilia + IdRegion — una fila por combinación)
        ├── ClaveProdServSat, ClaveUnidadSat  [solo MX, null para PE]
        └── IdPerfilFiscal ──► PerfilFiscal
                ├─── IdRegion = MX ──► catImpuesto "IVA", catTipoFactorSat, catObjetoImpuestoSat
                └─── IdRegion = PE ──► catImpuesto "IGV", catTipoFactorSat (ObjetoImpuestoSat = null)
        │
        ▼
FamiliaFiscalDataDto (CodigoImpuesto, TasaOCuota, TipoFactor, claves SAT si MX)
        │
        ▼
ConceptoBuilder → Nodo XML CFDI (MX) / Cálculo IGV (PE)
```

---

## Impacto en RE-FU-019-Back.md

El bloque `⏸ Pendiente` de la sección **"Catálogos SAT y PerfilFiscal"** en `RE-019-Back.md` queda **desbloqueado**. Actualizar esa sección para referenciar este cambio y eliminar la nota de espera.

Cambios concretos en RE-019-Back.md:
- Eliminar nota `⏸ Pendiente` de la sección de catálogos fiscales SAT
- Sustituir la jerarquía Producto→Familia por "solo Familia" en la lógica de resolución
- Eliminar `Producto.IdPerfilFiscal` del modelo (no se agrega override en Producto)

---

## Consideraciones previas (para implementación)

| # | Consideración | Detalle |
|---|---|---|
| 1 | BD primero | Los scripts DDL/DML del `_BD.md` deben ejecutarse antes de agregar los DbSets al scaffold |
| 2 | Scaffold regenerar | Al regenerar el scaffold, verificar que `FamiliaRegion` incluye todas sus columnas y la FK a `PerfilFiscal` con `Include` de navegación |
| 3 | Sin fila FamiliaRegion → excepción | `FamiliaNoFacturableException` se lanza cuando no existe fila en `FamiliaRegion` para el `(IdFamilia, IdRegion)` del producto |
| 4 | Batch para documentos multipartida | Siempre usar `GetFiscalDataByProductosBatchAsync` para documentos con más de una línea — evita N+1 queries |
| 5 | Catálogos de solo lectura | `catImpuestoSat`, `catTipoFactorSat`, `catObjetoImpuestoSat` no tienen endpoints CRUD — solo se leen |
| 6 | Sin `PerfilFiscal` endpoint en API | La gestión de `PerfilFiscal` y asignación a familias es exclusivamente por BD (sin interfaz gráfica en R16) |
| 7 | `RegionConstants` | Centralizar los GUIDs de región en una clase de constantes para no hardcodear en cada servicio |

---

## Recursos

- `R16A-RE-Cambio-PerfilFiscal_BD.md` — DDL/DML de las 4 tablas nuevas + ALTER TABLE Familia
- `R16A-RE-FU-019_BD.md` — sección "Datos del producto — ClaveProdServ, ClaveUnidad y PerfilFiscal" (GAP-7/GAP-8 resueltos)
- `R16A-RE-FU-019-Back.md` — sección "Catálogos SAT y PerfilFiscal" (desbloquear ⏸ Pendiente)
- `R16A-RE-Cambio-PerfilFiscal/Perfil-Fiscal-REQ-00.md` — requisito funcional
