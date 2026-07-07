# Creación de Solución ProquifaDotNet.Finanzas — Proceso Técnico
**Requisito origen:** R16A-RE-FU-016
**Referencia base:** Estructura del repositorio `proquifa-punchout-backend` (.NET 10, Clean Architecture)

---

## Estructura de la Solución

```
ProquifaDotNet.Finanzas/
+-- Proquifa.Finanzas.sln
+-- Domain/
|   +-- Proquifa.Finanzas.Domain.csproj (net10.0)
|   +-- Common/
|   |   +-- QueryInfo.cs
|   |   +-- FilterItem.cs
|   |   +-- SortDirection.cs
|   +-- Entities/
|   |   +-- TpProformaPedido.cs
|   +-- Views/
|   |   +-- VtpProformaPedido.cs
|   +-- Interfaces/
|   |   +-- IGenericRepository.cs
|   |   +-- IProformaRepository.cs
|   |   +-- IMinioStorageService.cs
|   |   +-- IUnitOfWork.cs
+-- Application/
|   +-- Proquifa.Finanzas.Application.csproj (net10.0)
|   +-- DTOs/
|   |   +-- QueryResultDto.cs
|   |   +-- ProformaDto.cs
|   |   +-- ProformaPdfModel.cs
|   |   +-- EmailRequestDto.cs
|   |   +-- EmailTemplateRequestDto.cs
|   +-- Interfaces/
|   |   +-- IProformaService.cs
|   |   +-- IDocumentBuilderClient.cs
|   |   +-- IApiCallerMail.cs
|   +-- Services/
|   |   +-- ProformaService.cs
|   +-- Mappers/
|   |   +-- ApplicationMappingProfile.cs
|   +-- Validators/
|       +-- ProformaDtoFluentValidator.cs
+-- Infrastructure/
|   +-- Proquifa.Finanzas.Infrastructure.csproj (net10.0)
|   +-- Persistence/
|   |   +-- Context/
|   |   |   +-- FinanzasContext.cs (scaffold BD ProquifaDotNet)
|   |   +-- Repository/
|   |       +-- GenericRepository.cs
|   |       +-- ProformaRepository.cs
|   +-- Services/
|   |   +-- ApiCallerMail.cs
|   |   +-- MinioStorageService.cs
|   |   +-- DocumentBuilderHttpClient.cs
|   +-- RabbitMQ/
|   |   +-- IRabbitMQClient.cs
|   |   +-- RabbitMQClient.cs
|   |   +-- RabbitMQSettings.cs
|   +-- Configuration/
|   |   +-- MailSettings.cs
|   |   +-- MinioSettings.cs
|   |   +-- DocumentBuilderSettings.cs
|   +-- Extensions/
|       +-- InfrastructureServiceExtensions.cs
+-- API/
|   +-- Proquifa.Finanzas.API.csproj (net10.0)
|   +-- Program.cs
|   +-- Controllers/
|   |   +-- ProformaController.cs
|   +-- appsettings.json
+-- Worker/
|   +-- Proquifa.Finanzas.Worker.csproj (net10.0)
|   +-- Program.cs
|   +-- Worker.cs
+-- Testing/
    +-- Proquifa.Finanzas.Testing.csproj (net10.0)
```

---

## Componentes Base a Crear (Prototipo)

### 1. Domain — Common (basado en PunchOut)

**QueryInfo.cs**
```csharp
namespace Proquifa.Finanzas.Domain.Common;

public class QueryInfo
{
    public string SortField { get; set; } = string.Empty;
    public SortDirection SortDirection { get; set; } = SortDirection.Asc;
    public List<FilterItem> Filters { get; set; } = new();
    public int? PageSize { get; set; }
    public int? DesiredPage { get; set; }
}
```

**FilterItem.cs**
```csharp
namespace Proquifa.Finanzas.Domain.Common;

public class FilterItem
{
    public string FieldName { get; set; } = string.Empty;
    public string Value { get; set; } = string.Empty;
}
```

**SortDirection.cs**
```csharp
namespace Proquifa.Finanzas.Domain.Common;

public enum SortDirection { Asc, Desc }
```

---

### 2. Application — QueryResultDto (basado en PunchOut)

**QueryResultDto.cs**
```csharp
namespace Proquifa.Finanzas.Application.DTOs;

public class QueryResultDto<T>
{
    public int TotalResults { get; set; }
    public IEnumerable<T> Results { get; set; } = [];
}
```

---

### 3. Domain — Entidad TpProformaPedido

```csharp
namespace Proquifa.Finanzas.Domain.Entities;

public class TpProformaPedido
{
    public Guid IdTPProformaPedido { get; set; }
    public Guid IdCliente { get; set; }
    public Guid IdEmpresa { get; set; }
    public string? FolioProforma { get; set; }       // NUEVO - folio PRF global
    public int ConsecutivoProforma { get; set; }     // NUEVO - consecutivo global
    public string? ReferenciaPago { get; set; }
    public string? NumeroFactura { get; set; }
    public decimal MontoTotal { get; set; }
    public decimal MontoPagado { get; set; }
    public decimal MontoPendiente { get; set; }
    public bool Controlados { get; set; }
    public bool MXN { get; set; }
    public bool USD { get; set; }
    public string? Folio { get; set; }               // Folio factura CFDI
    public string? Serie { get; set; }
    public string? Uuid { get; set; }
    public bool Cancelada { get; set; }
    public bool Factura { get; set; }
    public bool Publicaciones { get; set; }
    public bool Contrarecibo { get; set; }
    public bool Revisada { get; set; }
    public bool FacturaFlete { get; set; }
    public string? Comentarios { get; set; }
    public decimal? PrecioFleteKPI { get; set; }
    public DateTime? FechaCompromisoPago { get; set; }
    public DateTime? FechaPromesaPagoMonitoreoCobros { get; set; }
    public DateTime? FechaPagoCompleto { get; set; }
    public Guid? IdCFDI { get; set; }
    public Guid? IdCFDIGenerada { get; set; }
    public Guid? IdTPProformaPedidoReemplazo { get; set; }
    public bool Activo { get; set; }
    public DateTime FechaRegistro { get; set; }
    public DateTime FechaUltimaActualizacion { get; set; }
}
```

---

### 4. Domain — Vista VtpProformaPedido

```csharp
namespace Proquifa.Finanzas.Domain.Views;

public class VtpProformaPedido
{
    public Guid IdTPProformaPedido { get; set; }
    public string? FolioProforma { get; set; }
    public Guid IdCliente { get; set; }
    public string? ClienteNombre { get; set; }
    public Guid IdEmpresa { get; set; }
    public string? EmpresaPrefijo { get; set; }
    public string? EmpresaAlias { get; set; }
    public string? ReferenciaPago { get; set; }
    public string? NumeroFactura { get; set; }
    public decimal MontoTotal { get; set; }
    public decimal MontoPagado { get; set; }
    public decimal MontoPendiente { get; set; }
    public DateTime? FechaCompromisoPago { get; set; }
    public DateTime? FechaPromesaPagoMonitoreoCobros { get; set; }
    public DateTime? FechaPagoCompleto { get; set; }
    public bool FacturaFlete { get; set; }
    public bool Factura { get; set; }
    public bool MXN { get; set; }
    public bool USD { get; set; }
    public string? FolioFactura { get; set; }
    public string? SerieFactura { get; set; }
    public string? UuidCFDI { get; set; }
    public bool Cancelada { get; set; }
    public bool Controlados { get; set; }
    public string? Comentarios { get; set; }
    public bool Publicaciones { get; set; }
    public bool Revisada { get; set; }
    public bool Contrarecibo { get; set; }
    public Guid? IdTPPedido { get; set; }
    public string? FolioPedidoInterno { get; set; }
    public Guid? IdRegion { get; set; }
    public string? Region { get; set; }
    public string? RegionClave { get; set; }
    public Guid? IdCatCondicionesDePago { get; set; }
    public string? CondicionesDePago { get; set; }
    public bool FacturaPorAdelantado { get; set; }
    public bool EntregaConRemision { get; set; }
    public bool Tramitado { get; set; }
    public DateTime? FechaTramitacion { get; set; }
}
```

---

### 5. Infrastructure — FinanzasContext (EF Core scaffold)

```csharp
namespace Proquifa.Finanzas.Infrastructure.Persistence.Context;

public class FinanzasContext : DbContext
{
    public FinanzasContext(DbContextOptions<FinanzasContext> options) : base(options) { }

    public DbSet<TpProformaPedido> TpProformaPedido { get; set; }
    public DbSet<VtpProformaPedido> VtpProformaPedido { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<TpProformaPedido>(entity =>
        {
            entity.ToTable("tpProformaPedido");
            entity.HasKey(e => e.IdTPProformaPedido);
        });

        modelBuilder.Entity<VtpProformaPedido>(entity =>
        {
            entity.ToView("vtpProformaPedido");
            entity.HasNoKey();
        });
    }
}
```

---

### 6. Infrastructure — ProformaRepository (consulta vtpProformaPedido con QueryInfo)

```csharp
namespace Proquifa.Finanzas.Infrastructure.Persistence.Repository;

public class ProformaRepository : IProformaRepository
{
    private readonly FinanzasContext _context;

    public ProformaRepository(FinanzasContext context) => _context = context;

    public async Task<QueryResultDto<VtpProformaPedido>> ListProformas(QueryInfo queryInfo)
    {
        var query = _context.VtpProformaPedido.AsNoTracking().AsQueryable();

        // Apply dynamic filters
        foreach (var filter in queryInfo.Filters)
        {
            query = ApplyFilter(query, filter);
        }

        // Total before pagination
        var totalResults = await query.CountAsync();

        // Sorting
        if (!string.IsNullOrEmpty(queryInfo.SortField))
        {
            query = queryInfo.SortDirection == SortDirection.Asc
                ? query.OrderBy(e => EF.Property<object>(e, queryInfo.SortField))
                : query.OrderByDescending(e => EF.Property<object>(e, queryInfo.SortField));
        }

        // Pagination
        if (queryInfo.PageSize.HasValue && queryInfo.DesiredPage.HasValue)
        {
            var skip = (queryInfo.DesiredPage.Value - 1) * queryInfo.PageSize.Value;
            query = query.Skip(skip).Take(queryInfo.PageSize.Value);
        }

        var results = await query.ToListAsync();

        return new QueryResultDto<VtpProformaPedido>
        {
            TotalResults = totalResults,
            Results = results
        };
    }

    private static IQueryable<VtpProformaPedido> ApplyFilter(
        IQueryable<VtpProformaPedido> query, FilterItem filter)
    {
        return filter.FieldName.ToLower() switch
        {
            "clientenombre" => query.Where(x => x.ClienteNombre!.Contains(filter.Value)),
            "folioproforma" => query.Where(x => x.FolioProforma!.Contains(filter.Value)),
            "foliopedidointerno" => query.Where(x => x.FolioPedidoInterno!.Contains(filter.Value)),
            "empresaprefijo" => query.Where(x => x.EmpresaPrefijo == filter.Value),
            "regionclave" => query.Where(x => x.RegionClave == filter.Value),
            "controlados" => query.Where(x => x.Controlados == bool.Parse(filter.Value)),
            "cancelada" => query.Where(x => x.Cancelada == bool.Parse(filter.Value)),
            "tramitado" => query.Where(x => x.Tramitado == bool.Parse(filter.Value)),
            _ => query
        };
    }
}
```

---

### 7. Application — ProformaService (CRUD basico)

```csharp
namespace Proquifa.Finanzas.Application.Services;

public class ProformaService : IProformaService
{
    private readonly IGenericRepository<TpProformaPedido> _repository;
    private readonly IProformaRepository _proformaRepository;
    private readonly IUnitOfWork _unitOfWork;

    public ProformaService(
        IGenericRepository<TpProformaPedido> repository,
        IProformaRepository proformaRepository,
        IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _proformaRepository = proformaRepository;
        _unitOfWork = unitOfWork;
    }

    public async Task<TpProformaPedido?> GetById(Guid id)
        => await _repository.GetById(id);

    public async Task<Guid> Create(TpProformaPedido entity)
    {
        var id = await _repository.Add(entity);
        await _repository.SaveChanges();
        return id;
    }

    public async Task<Guid> Update(TpProformaPedido entity)
    {
        var id = await _repository.Update(entity);
        await _repository.SaveChanges();
        return id;
    }

    public async Task<bool> Delete(Guid id)
    {
        var result = await _repository.Delete(id);
        await _repository.SaveChanges();
        return result;
    }

    public async Task<QueryResultDto<VtpProformaPedido>> List(QueryInfo queryInfo)
        => await _proformaRepository.ListProformas(queryInfo);
}
```

> **Bitácora (Reglas al diseñar — regla 8):** `Create`/`Update`/`Delete` de `ProformaService` deben registrar el resultado de la operación en **ProquifaDotNet.BitacoraCambios** (Aplicativo Nuevo), igual que Finanzas debe hacerlo al guardar una factura o validar un cobro (ver R16A-RE-FU-018-Back.md). El contrato/endpoint de este aplicativo aún no está documentado en un requisito propio; aquí solo se referencia el punto de integración, no su detalle técnico.

---

### 8. API — ProformaController

```csharp
namespace Proquifa.Finanzas.API.Controllers;

[ApiController]
[Route("api/v1/proforma")] // Reglas al diseñar - regla 9: api/v1/{resource}, recurso explícito en minúsculas (no [controller])
public class ProformaController : ControllerBase
{
    private readonly IProformaService _proformaService;

    public ProformaController(IProformaService proformaService)
        => _proformaService = proformaService;

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(Guid id)
    {
        var result = await _proformaService.GetById(id);
        return result is null ? NotFound() : Ok(result);
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] TpProformaPedido entity)
    {
        var id = await _proformaService.Create(entity);
        return CreatedAtAction(nameof(GetById), new { id }, entity);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(Guid id, [FromBody] TpProformaPedido entity)
    {
        entity.IdTPProformaPedido = id;
        var result = await _proformaService.Update(entity);
        return Ok(result);
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(Guid id)
    {
        var result = await _proformaService.Delete(id);
        return result ? NoContent() : NotFound();
    }

    [HttpPost("list")]
    public async Task<IActionResult> List([FromBody] QueryInfo queryInfo)
    {
        var result = await _proformaService.List(queryInfo);
        return Ok(result);
    }
}
```

---

### 9. Infrastructure — Servicios Base

#### MinioStorageService.cs
```csharp
namespace Proquifa.Finanzas.Infrastructure.Services;

public class MinioStorageService : IMinioStorageService
{
    private readonly IMinioClient _minioClient;
    private readonly MinioSettings _settings;

    public async Task<string> UploadAsync(byte[] fileBytes, string fileName, string bucket)
    {
        using var stream = new MemoryStream(fileBytes);
        await _minioClient.PutObjectAsync(new PutObjectArgs()
            .WithBucket(bucket)
            .WithObject(fileName)
            .WithStreamData(stream)
            .WithObjectSize(fileBytes.Length)
            .WithContentType("application/pdf"));
        return fileName;
    }

    public async Task<byte[]> DownloadAsync(string fileName, string bucket)
    {
        using var stream = new MemoryStream();
        await _minioClient.GetObjectAsync(new GetObjectArgs()
            .WithBucket(bucket)
            .WithObject(fileName)
            .WithCallbackStream(s => s.CopyTo(stream)));
        return stream.ToArray();
    }
}
```

#### ApiCallerMail.cs (Reglas al diseñar — regla 7)

> Finanzas **no** integra con Brevo directamente. El envío de correo se delega al Aplicativo Nuevo **ProquifaDotNet.EnvioCorreo** (solución independiente, distinta de ProquifaDotNet.SendInBlue — que solo migra el envío de correo del sistema legacy ProquifaDotNet/Venta Interna) vía llamadas HTTP entre APIs. `ApiCallerMail` es un cliente HTTP (Polly para timeout, sin reintento interno) que llama al API de ProquifaDotNet.EnvioCorreo con el `EmailRequestDto` / `EmailTemplateRequestDto` correspondiente.
- Soporta envío por plantilla y HTML/texto plano (según lo que exponga el API de EnvioCorreo)
- Autenticación via IdentityServer

#### RabbitMQClient.cs (patron PunchOut)
- Publish/Subscribe con ACK/NACK
- DLQ para mensajes fallidos
- Reintentos configurables

#### DocumentBuilderHttpClient.cs
```csharp
namespace Proquifa.Finanzas.Infrastructure.Services;

public class DocumentBuilderHttpClient : IDocumentBuilderClient
{
    private readonly HttpClient _httpClient;

    public DocumentBuilderHttpClient(HttpClient httpClient)
        => _httpClient = httpClient;

    public async Task<byte[]> GenerateProformaPdf(object proformaDto)
    {
        var response = await _httpClient.PostAsJsonAsync("api/Report/proforma", proformaDto);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsByteArrayAsync();
    }
}
```

---

### 10. Paquetes NuGet Requeridos

| Proyecto | Paquete | Version | Uso |
|----------|---------|---------|-----|
| Infrastructure | Microsoft.EntityFrameworkCore.SqlServer | 8.x | EF Core + SQL Server |
| Infrastructure | AutoMapper | 15.x | Mapeo entidad -> DTO |
| Infrastructure | RabbitMQ.Client | 7.x | Colas |
| Infrastructure | Minio | 6.x | Almacenamiento objetos |
| Infrastructure | Serilog | 4.x | Logs |
| Infrastructure | Polly | 8.x | Timeout HTTP (ApiCallerMail, ApiCallerStamping) |
| Application | FluentValidation | 11.x | Validaciones |
| API | Serilog.AspNetCore | 9.x | Logs en API |
| API | Swashbuckle.AspNetCore | 6.x | Swagger |

---

## Resumen de Tareas de Creación de Solución

| #   | Tarea                                    | Descripción                                                                |
| --- | ---------------------------------------- | -------------------------------------------------------------------------- |
| 1   | Crear solución y proyectos               | sln + 6 csproj (Domain, Application, Infrastructure, API, Worker, Testing) |
| 2   | Domain Common                            | QueryInfo, FilterItem, SortDirection                                       |
| 3   | Application QueryResultDto               | Wrapper genérico paginado                                                  |
| 4   | Domain Entities + Views                  | TpProformaPedido, VtpProformaPedido                                        |
| 5   | Infrastructure Context                   | FinanzasContext con scaffold de tablas necesarias                          |
| 6   | Infrastructure GenericRepository         | CRUD genérico                                                              |
| 7   | Infrastructure ProformaRepository        | Consulta vtpProformaPedido con QueryInfo y paginado                        |
| 8   | Application ProformaService              | CRUD + List con paginado                                                 |
| 9   | API ProformaController                   | Endpoints REST (CRUD + list + PDF)                                       |
| 10  | Infrastructure MinioStorageService       | Upload/Download PDF a Minio                                                |
| 11  | Infrastructure ApiCallerMail      | Cliente HTTP hacia ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo)           |
| 12  | Infrastructure RabbitMQClient            | Cliente RabbitMQ                                                           |
| 13  | Infrastructure DocumentBuilderHttpClient | Cliente HTTP para DocumentBuilder                                          |
| 14  | Configuración DI + appsettings           | Registrar servicios, connection strings, settings                          |
