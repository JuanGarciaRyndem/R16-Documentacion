# Tareas BackEnd — TPSC-RE-FU-019
**Requisito:** Factura por Adelantado: Detalle México
**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10)

---

## Parte A — ProquifaDotNet.Timbrado (Ampliación)

---

### Tarea 1

**Título:** [ TPSC-RE-FU-019 ] [CREATE-SCRIPT-CONTROL] Crear scripts DDL/DML para tabla EmpresaFolio en ProquifaDotNetTimbrado

**Aplicativos:** ProquifaDotNet.Timbrado (BD)

**Módulos:** Base de Datos — ProquifaDotNetTimbrado

**Consideraciones previas:**
- La tabla EmpresaFolio es NUEVA en ProquifaDotNetTimbrado
- La BD ProquifaDotNetTimbrado fue creada en RE-FU-018; esta tabla se agrega en este requisito
- Contiene 4 registros iniciales: GOL, MUN, PRO, PQF
- UltimoFolio debe ajustarse al MAX existente en producción antes del go-live
- Debe ejecutarse ANTES de implementar el repositorio EmpresaFolioRepository (Tarea 3)

**Objetivo general:**
Crear los scripts DDL y DML para la tabla EmpresaFolio y sus datos iniciales de las 4 empresas emisoras de México.

**Objetivos específicos:**
- Crear script DDL:

```sql
CREATE TABLE [dbo].[EmpresaFolio] (
    [IdEmpresaFolio]  uniqueidentifier NOT NULL CONSTRAINT [DF_EmpresaFolio_Id]         DEFAULT (NEWID()),
    [EmpresaClave]    varchar(10)      NOT NULL,
    [EmpresaNombre]   varchar(200)     NOT NULL,
    [Serie]           varchar(25)          NULL,
    [UltimoFolio]     int              NOT NULL CONSTRAINT [DF_EmpresaFolio_UltimoFolio] DEFAULT (0),
    [FormatoFolio]    varchar(50)      NOT NULL CONSTRAINT [DF_EmpresaFolio_Formato]     DEFAULT ('{folio}'),
    [LongitudMaxima]  int              NOT NULL CONSTRAINT [DF_EmpresaFolio_Longitud]    DEFAULT (6),
    [CreatedAt]       datetime2(7)     NOT NULL CONSTRAINT [DF_EmpresaFolio_CreatedAt]   DEFAULT (SYSUTCDATETIME()),
    [UpdatedAt]       datetime2(7)     NOT NULL CONSTRAINT [DF_EmpresaFolio_UpdatedAt]   DEFAULT (SYSUTCDATETIME()),
    [IsActive]        bit              NOT NULL CONSTRAINT [DF_EmpresaFolio_IsActive]    DEFAULT (1),
    CONSTRAINT [PK_EmpresaFolio]   PRIMARY KEY CLUSTERED ([IdEmpresaFolio]),
    CONSTRAINT [UQ_EmpresaFolio_Clave] UNIQUE ([EmpresaClave])
);
```

- Crear script DML:

```sql
INSERT INTO [dbo].[EmpresaFolio] ([EmpresaClave], [EmpresaNombre], [UltimoFolio])
VALUES
    ('GOL', 'Golocaer S.A. de C.V.',                          0),
    ('MUN', 'Mungen S.A. de C.V.',                            0),
    ('PRO', 'Proquifa S.A. de C.V.',                          0),
    ('PQF', 'Proveedora Quimico Farmaceutica S.A. de C.V.',   0);
-- Ajustar UltimoFolio al MAX(folio) existente en producción antes del go-live
```

- Incluir script de validación post-ejecución
- Hacer el script idempotente (verificar existencia antes de CREATE)

**Resultado esperado:**
Tabla EmpresaFolio creada en ProquifaDotNetTimbrado con los 4 registros iniciales listos para consumo de folios.

**Entregables:**
- Script DDL: CREATE TABLE EmpresaFolio
- Script DML: INSERT datos iniciales
- Script de validación

**Criterios de aceptación:**
- La tabla se crea correctamente en ProquifaDotNetTimbrado
- Constraint UNIQUE en EmpresaClave previene duplicados
- Los 4 registros se insertan con UltimoFolio=0 e IsActive=1
- El script es idempotente

**Más información de la tarea:**
Corresponde a GAP-06. Ejecutar ANTES de implementar EmpresaFolioRepository (Tarea 3).

**Recursos:**
- TPSC-RE-FU-019_BD.md (CREATE TABLE EmpresaFolio + INSERT)
- TPSC-RE-FU-019-Back.md (Parte D — Scripts, Orden de ejecución)

---

### Tarea 2

**Título:** [ TPSC-RE-FU-019 ] [SIMPLE-CRUD] Crear entidad EmpresaFolio e interface `IEmpresaFolioRepository`

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Domain

**Consideraciones previas:**
- La solución ProquifaDotNet.Timbrado fue creada en RE-FU-018
- La tabla EmpresaFolio fue creada en Tarea 1 (script DDL)
- La interface debe extender `IGenericRepository<EmpresaFolio>`
- El folio es un consecutivo numérico independiente por empresa emisora (GOL, MUN, PRO, PQF), de tipo varchar(6)

**Objetivo general:**
Crear la entidad de dominio EmpresaFolio y la interface del repositorio con método de consumo atómico de folio.

**Objetivos específicos:**
- Crear `Entities/EmpresaFolio.cs` con campos: IdEmpresaFolio, EmpresaClave, EmpresaNombre, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, CreatedAt, UpdatedAt, IsActive
- Crear `Interfaces/IEmpresaFolioRepository.cs`:

```csharp
public interface IEmpresaFolioRepository : IGenericRepository<EmpresaFolio>
{
    Task<EmpresaFolio> GetByClaveAsync(string empresaClave);
    Task<int> ConsumeNextFolioAsync(string empresaClave); // UPDATE atómico con UPDLOCK
}
```

**Resultado esperado:**
Entidad EmpresaFolio y su interface de repositorio disponibles en la capa Domain.

**Entregables:**
- `Entities/EmpresaFolio.cs`
- `Interfaces/IEmpresaFolioRepository.cs`

**Criterios de aceptación:**
- La entidad refleja exactamente la estructura de la tabla EmpresaFolio (TPSC-RE-FU-019_BD.md)
- La interface define `ConsumeNextFolioAsync` que retorna `Task<int>` (próximo folio)
- El proyecto Domain compila sin errores y sin dependencias externas

**Más información de la tarea:**
Corresponde a GAP-01. El folio se consume atómicamente solo al timbrar exitosamente (sin huecos por errores PAC).

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte A — Domain)
- TPSC-RE-FU-019_BD.md (CREATE TABLE EmpresaFolio)

---

### Tarea 3

**Título:** [ TPSC-RE-FU-019 ] [IMP-EXIST-SERVICE] Implementar `EmpresaFolioRepository` con UPDATE atómico (UPDLOCK)

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Infrastructure/Persistence/Repository

**Consideraciones previas:**
- Depende de Tarea 1 (tabla EmpresaFolio en BD) y Tarea 2 (interface `IEmpresaFolioRepository`)
- El consumo de folio debe ser thread-safe ante concurrencia (múltiples timbrados simultáneos)
- Usar raw SQL con hints UPDLOCK, ROWLOCK y OUTPUT INSERTED para atomicidad
- Agregar `DbSet<EmpresaFolio>` al TimbradoContext existente

**Objetivo general:**
Implementar el repositorio de EmpresaFolio con consumo atómico de folio concurrency-safe.

**Objetivos específicos:**
- Crear `Repository/EmpresaFolioRepository.cs` extendiendo `GenericRepository<EmpresaFolio>`
- Implementar `ConsumeNextFolioAsync` con raw SQL:

```sql
UPDATE [dbo].[EmpresaFolio] WITH (UPDLOCK, ROWLOCK)
SET    UltimoFolio = UltimoFolio + 1,
       UpdatedAt   = SYSUTCDATETIME()
OUTPUT INSERTED.UltimoFolio
WHERE  EmpresaClave = @clave
  AND  IsActive = 1;
```

- Implementar `GetByClaveAsync` con consulta EF Core
- Agregar `DbSet<EmpresaFolio>` y mapeo Fluent API en TimbradoContext

**Resultado esperado:**
Repositorio funcional que consume folios sin posibilidad de duplicados bajo concurrencia.

**Entregables:**
- `Repository/EmpresaFolioRepository.cs`
- Ampliación de `TimbradoContext` (DbSet + mapeo)

**Criterios de aceptación:**
- `ConsumeNextFolioAsync` usa UPDATE con UPDLOCK, ROWLOCK
- Bajo concurrencia no se producen folios duplicados
- El mapeo Fluent API respeta tipos de columna y constraint UNIQUE en EmpresaClave

**Más información de la tarea:**
Corresponde a GAP-03. Patrón documentado en TPSC-RE-FU-019_BD.md sección "Consumo atómico del folio".

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte A — Infrastructure)
- TPSC-RE-FU-019_BD.md (Consumo atómico del folio)

---

### Tarea 4

**Título:** [ TPSC-RE-FU-019 ] [SIMPLE-CRUD] Crear DTOs de request/response para timbrado FAA

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Application/DTOs

**Consideraciones previas:**
- Los DTOs modelan el contrato del endpoint `POST /api/timbrado/timbrar-faa`
- Valores forzados por normativa SAT (PPD, 99, I) se incluyen como propiedades con defaults
- El response indica éxito/error con datos del CFDI generado

**Objetivo general:**
Crear los modelos DTO específicos para el timbrado de Factura por Adelantado.

**Objetivos específicos:**
- Crear `DTOs/TimbrarFAARequestDto.cs` con: IdProformaAdelanto, DatosReceptor, DatosEmisor, `Conceptos[]`, MetodoPago (default "PPD"), FormaPago (default "99"), TipoComprobante (default "I"), Moneda, TipoCambio
- Crear `DTOs/ConceptoFAADto.cs` con: Cantidad, Descripcion, PrecioUnitario, Importe, ClaveUnidad, ClaveProdServ — el campo Descripcion debe construirse como "catálogo + descripción + marca"; no incluir lote ni pedimento (OBS-039)
- Crear `DTOs/TimbrarFAAResponseDto.cs` con: IdCFDI, UUID, Serie, Folio, FechaEmision, Total, XmlBase64, Exitoso, ErrorDescripcion, ErrorCodigo
- Crear `DTOs/DatosReceptorDto.cs` con: RFC, RazonSocial, CP, RegimenFiscal, UsoCFDI
- Crear `DTOs/DatosEmisorDto.cs` con: RFC, RazonSocial, RegimenFiscal, EmpresaClave

**Resultado esperado:**
DTOs completos que modelan el contrato de comunicación del endpoint `/api/timbrado/timbrar-faa`.

**Entregables:**
- `DTOs/TimbrarFAARequestDto.cs`
- `DTOs/ConceptoFAADto.cs`
- `DTOs/TimbrarFAAResponseDto.cs`
- `DTOs/DatosReceptorDto.cs`
- `DTOs/DatosEmisorDto.cs`

**Criterios de aceptación:**
- MetodoPago, FormaPago y TipoComprobante tienen valores default forzados por normativa SAT
- `TimbrarFAAResponseDto` incluye campo Exitoso (bool) y ErrorDescripcion para manejo de errores PAC
- Los DTOs compilan sin errores

**Más información de la tarea:**
Corresponde a GAP-04. Valores forzados SAT para factura PPD documentados en TPSC-RE-FU-019.md (Regla 5).

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte A — DTOs nuevos para FAA)
- TPSC-RE-FU-019.md (Regla 5 — Forma de Pago, Método de Pago forzados)

---

### Tarea 5

**Título:** [ TPSC-RE-FU-019 ] [ALG-COMPLX-LOGIC] Ampliar TimbradoService con `EmpresaFolioService` y método `TimbrarFacturaAdelantadoAsync`

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Application/Services

**Consideraciones previas:**
- Depende de Tarea 3 (`EmpresaFolioRepository`) y Tarea 4 (DTOs)
- `TimbradoService` ya existe (RE-FU-018); se amplía con método específico para FAA
- El nuevo método orquesta: validar datos fiscales → consumir folio → armar XML → llamar SAP → persistir CFDI → log
- Si SAP retorna error, retornar `TimbrarFAAResponseDto` con Exitoso=false sin modificar BD

**Objetivo general:**
Ampliar la capa Application con el servicio de consumo de folio y el método de timbrado de Factura por Adelantado.

**Objetivos específicos:**
- Crear `Interfaces/IEmpresaFolioService.cs` con métodos `GetNextFolioAsync` y `GetByClaveAsync`
- Crear `Services/EmpresaFolioService.cs` que consume folio atómico y retorna folio formateado (padding a 6 chars)
- Ampliar `ITimbradoService.cs` agregando `TimbrarFacturaAdelantadoAsync(TimbrarFAARequestDto)`
- Ampliar `TimbradoService.cs` implementando el flujo:

```
1. Validar request (FluentValidation)
2. Consumir folio via EmpresaFolioService.GetNextFolioAsync(EmpresaClave)
3. Armar estructura XML CFDI con datos fiscales + conceptos
4. INSERT CFDI en BD (EstatusTimbrado = Pendiente)
5. Llamar SapTimbradoClient -> PAC SAP
6. Si ERROR: retornar TimbrarFAAResponseDto con Exitoso=false + ErrorDescripcion
7. Si ÉXITO: UPDATE CFDI (UUID, XML firmado, EstatusTimbrado = Timbrado)
8. Subir XML a Minio (bucket 'timbrado')
9. INSERT TimbradoLog (request, response, duración)
10. Retornar TimbrarFAAResponseDto con Exitoso=true
```

**Resultado esperado:**
Servicio de timbrado FAA funcional que orquesta el flujo completo con manejo de errores PAC.

**Entregables:**
- `Interfaces/IEmpresaFolioService.cs`
- `Services/EmpresaFolioService.cs`
- Ampliación de `ITimbradoService.cs` y `TimbradoService.cs`

**Criterios de aceptación:**
- El folio se consume SOLO al timbrar exitosamente (no se incrementa si SAP falla)
- Si SAP retorna error, el response incluye ErrorDescripcion y Exitoso=false sin modificar folio
- `TimbradoLog` registra request, response, duración y error (si aplica)

**Más información de la tarea:**
Corresponde a GAP-02. Ver diagrama de flujo "Generar Factura" en TPSC-RE-FU-019-Back.md.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte A — Application, Flujo Generar Factura)
- TPSC-RE-FU-018-Back.md (Flujo Funcional de Timbrado — referencia base)

---

### Tarea 6

**Título:** [ TPSC-RE-FU-019 ] [IMP-EXIST-SERVICE] Agregar endpoint `POST /api/timbrado/timbrar-faa` en TimbradoController

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** API/Controllers

**Consideraciones previas:**
- Depende de Tarea 5 (`TimbradoService.TimbrarFacturaAdelantadoAsync`) y Tarea 4 (DTOs)
- `TimbradoController` ya existe (RE-FU-018); se amplía con un nuevo endpoint
- Autenticación via IdentityServer (token desde Finanzas)

**Objetivo general:**
Agregar el endpoint de timbrado de Factura por Adelantado al controlador existente.

**Objetivos específicos:**
- Agregar método `POST TimbrarFacturaAdelantado([FromBody] TimbrarFAARequestDto request)` en `TimbradoController`
- Ruta: `/api/timbrado/timbrar-faa`
- Delegar a `ITimbradoService.TimbrarFacturaAdelantadoAsync`
- Retornar 200 OK con `TimbrarFAAResponseDto` (tanto en éxito como en error de PAC)
- Retornar 400 BadRequest si validación de request falla
- Retornar 500 si error interno no controlado

**Resultado esperado:**
Endpoint funcional que recibe solicitud de timbrado FAA desde Finanzas y retorna el resultado.

**Entregables:**
- Ampliación de `Controllers/TimbradoController.cs`

**Criterios de aceptación:**
- El endpoint es accesible en `/api/timbrado/timbrar-faa`
- Valida autenticación IdentityServer
- Retorna `TimbrarFAAResponseDto` con Exitoso=true y datos CFDI cuando el timbrado es exitoso
- Retorna `TimbrarFAAResponseDto` con Exitoso=false y ErrorDescripcion cuando PAC falla
- Errores de validación retornan 400

**Más información de la tarea:**
Corresponde a GAP-05. Consumido exclusivamente por ProquifaDotNet.Finanzas.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte A — API TimbradoController)

---

## Parte B — ProquifaDotNet.Finanzas (Módulo FAA — Detalle)

---

### Tarea 7

**Título:** [ TPSC-RE-FU-019 ] [CREATE-SCRIPT-CONTROL] Crear scripts DDL: ALTER TABLE tpProformaAdelanto y CREATE VIEW vtpProformaAdelanto

**Aplicativos:** ProquifaDotNet (BD)

**Módulos:** Base de Datos — ProquifaDotNet

**Consideraciones previas:**
- El ALTER agrega la columna Enviada (bit NOT NULL DEFAULT 0) para rastrear el envío exitoso de la factura
- La vista resuelve la cadena compleja de JOINs (8 tablas) y calcula el campo EstadoFAA
- La vista debe crearse DESPUÉS del ALTER (requiere que la columna Enviada exista)
- Ejecutar ANTES de implementar `FacturaAdelantadoDetalleRepository` y `FinanzasContext` (Tareas 8 y 9)

**Objetivo general:**
Preparar la estructura de BD en ProquifaDotNet con la columna Enviada y la vista vtpProformaAdelanto necesarias para el flujo Detalle FAA.

**Objetivos específicos:**
- Crear script ALTER TABLE:

```sql
ALTER TABLE dbo.tpProformaAdelanto
    ADD Enviada bit NOT NULL
        CONSTRAINT [DF_tpProformaAdelanto_Enviada] DEFAULT (0);
```

- Crear script CREATE VIEW con cadena de JOINs y EstadoFAA calculado:

```sql
CREATE VIEW dbo.vtpProformaAdelanto AS
SELECT
    pa.IdTPProformaAdelanto,
    pa.Monto, pa.MXN, pa.USD, pa.TipoDeCambio,
    pa.IdCliente,
    c.Nombre           AS ClienteNombre,
    dfc.RazonSocial    AS ClienteRazonSocial,
    dfc.RFC            AS ClienteRFC,
    pa.IdEmpresa,
    e.Prefijo          AS EmpresaPrefijo,
    e.Alias            AS EmpresaAlias,
    pa.IdCFDIGenerada,
    cg.Folio           AS FolioFactura,
    cg.Serie           AS SerieFactura,
    pa.Enviada,
    pa.FechaRegistro,
    pa.Activo,
    CASE
        WHEN pa.IdCFDIGenerada IS NULL                              THEN 'PendienteGenerar'
        WHEN pa.IdCFDIGenerada IS NOT NULL AND pa.Enviada = 0       THEN 'PendienteEnviar'
        ELSE 'Completada'
    END AS EstadoFAA,
    tp.IdTPPedido,
    tp.FolioPedidoInterno,
    tp.FechaTramitacion,
    tp.IdCatCondicionesDePago,
    cdp.CondicionesDePago,
    cdp.SinCredito AS EsPrepago,
    r.ClaveISO     AS RegionClave
FROM dbo.tpProformaAdelanto pa
LEFT JOIN dbo.Cliente c                     ON pa.IdCliente  = c.IdCliente
LEFT JOIN dbo.DatosFacturacionCliente dfc   ON pa.IdCliente  = dfc.IdCliente AND dfc.Activo = 1
LEFT JOIN dbo.Empresa e                     ON pa.IdEmpresa  = e.IdEmpresa
LEFT JOIN dbo.CFDIGenerada cg               ON pa.IdCFDIGenerada = cg.IdCFDIGenerada
LEFT JOIN dbo.fccPagoFacturaAdelanto fpfa   ON fpfa.IdTPProformaAdelanto = pa.IdTPProformaAdelanto AND fpfa.Activo = 1
LEFT JOIN dbo.tpProformaPedido pp           ON fpfa.IdTPProformaPedido   = pp.IdTPProformaPedido
LEFT JOIN dbo.tpPedidoProformaPedido tpp    ON pp.IdTPProformaPedido     = tpp.IdTPProformaPedido AND tpp.Activo = 1
LEFT JOIN dbo.tpPedido tp                   ON tpp.IdTPPedido            = tp.IdTPPedido
LEFT JOIN dbo.Region r                      ON tp.IdRegion               = r.IdRegion
LEFT JOIN dbo.catCondicionesDePago cdp      ON tp.IdCatCondicionesDePago = cdp.IdCatCondicionesDePago;
```

- Incluir script de validación: `SELECT TOP 5 * FROM vtpProformaAdelanto`
- Scripts idempotentes (verificar existencia antes de ejecutar)

**Resultado esperado:**
Columna Enviada disponible en tpProformaAdelanto y vista vtpProformaAdelanto creada con EstadoFAA calculado.

**Entregables:**
- Script DDL: ALTER TABLE tpProformaAdelanto
- Script DDL: CREATE VIEW vtpProformaAdelanto
- Script de validación

**Criterios de aceptación:**
- El ALTER se ejecuta sin errores y el campo Enviada tiene default 0
- La vista calcula correctamente los 3 estados: PendienteGenerar, PendienteEnviar, Completada
- La cadena de JOINs retorna datos correctos para pedidos existentes
- Scripts idempotentes

**Más información de la tarea:**
Corresponde a GAP-20 y GAP-21. Ejecutar en orden: primero ALTER, luego VIEW. Ejecutar ANTES de Tareas 8 y 9.

**Recursos:**
- TPSC-RE-FU-019_BD.md (ALTER TABLE + CREATE VIEW completa)
- TPSC-RE-FU-019-Back.md (Parte D — Scripts)

---

### Tarea 8

**Título:** [ TPSC-RE-FU-019 ] [SIMPLE-CRUD] Ampliar FinanzasContext con `DbSet` de vista vtpProformaAdelanto y tablas auxiliares

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Infrastructure/Persistence/Context

**Consideraciones previas:**
- Depende de Tarea 7 (ALTER + VIEW ejecutados en BD)
- El FinanzasContext fue creado en RE-FU-016 y ampliado en RE-FU-018
- La vista vtpProformaAdelanto se mapea con `HasNoKey()` y `ToView()` (solo lectura)
- Las tablas Archivo, CorreoEnviado, ArchivoCorreoEnviado se usan para escritura post-timbrado/envío

**Objetivo general:**
Ampliar el contexto EF Core con los mapeos necesarios para el flujo Detalle FAA.

**Objetivos específicos:**
- Agregar `DbSet<VtpProformaAdelanto>` mapeado con `.ToView("vtpProformaAdelanto").HasNoKey()`
- Agregar `DbSet<Archivo>` con mapeo a tabla Archivo
- Agregar `DbSet<CorreoEnviado>` con mapeo a tabla CorreoEnviado
- Agregar `DbSet<ArchivoCorreoEnviado>` con mapeo a tabla ArchivoCorreoEnviado
- Agregar `DbSet<CatUsoCFDI>` con mapeo a tabla catUsoCFDI
- Configurar Fluent API con tipos de columna correctos (varchar, uniqueidentifier, datetime2, bit)

**Resultado esperado:**
FinanzasContext con todos los DbSets necesarios para consultar la vista y persistir registros del flujo FAA.

**Entregables:**
- Ampliación de `Context/FinanzasContext.cs`
- Entidades de dominio: `VtpProformaAdelanto.cs`, `Archivo.cs`, `CorreoEnviado.cs`, `ArchivoCorreoEnviado.cs`, `CatUsoCFDI.cs`

**Criterios de aceptación:**
- `VtpProformaAdelanto` se mapea como vista (HasNoKey, sin tracking de cambios)
- Las entidades de escritura (Archivo, CorreoEnviado) permiten INSERT sin errores
- El contexto compila sin conflictos con DbSets existentes
- Los tipos de columna coinciden con la BD

**Más información de la tarea:**
Corresponde a GAP-15. La vista vtpProformaAdelanto resuelve la cadena compleja de JOINs.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte B — FinanzasContext Ampliación)
- TPSC-RE-FU-019_BD.md (columnas de vtpProformaAdelanto)

---

### Tarea 9

**Título:** [ TPSC-RE-FU-019 ] [SIMPLE-CRUD] Crear DTOs del módulo FAA Detalle (11 modelos)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/DTOs

**Consideraciones previas:**
- Los DTOs cubren: request/response del detalle, cabecera cliente, pedido con estado, datos fiscales cliente/emisor, generación, envío y preview
- Valores forzados SAT: MetodoPago=PPD, FormaPago=99, TipoComprobante=I

**Objetivo general:**
Crear los modelos DTO completos para todos los endpoints del Detalle FAA.

**Objetivos específicos:**
- Crear `FAADetalleRequestDto` (IdCliente, `Filters[]`, SortField, SortDirection, PageSize, DesiredPage)
- Crear `FAADetalleResponseDto` (Cliente: cabecera, TotalResults, `Results[]`)
- Crear `FAAClienteCabeceraDto` (IdCliente, RazonSocial, RFC, MonedaFacturacion, ClasificacionCrediticia)
- Crear `FAAPedidoDetalleDto` (IdTPProformaAdelanto, IdTPPedido, FolioPedidoInterno, FechaPedido, CondicionesDePago, EsPrepago, EmpresaAlias, EmpresaClave, Subtotal, IVA, MontoTotal, Moneda, EstadoFAA, FolioFactura, SerieFactura, EsacCorreo)
- Crear `FAAGenerarRequestDto` (IdTPProformaAdelanto, UsoCFDI)
- Crear `FAAGenerarResponseDto` (Exitoso, IdCFDIGenerada, UUID, Folio, Serie, FechaEmision, Total, PdfUrl, XmlUrl, ErrorDescripcion, ErrorCodigo)
- Crear `FAAEnviarRequestDto` (IdTPProformaAdelanto, Destinatario, CC, Notas)
- Crear `FAAEnviarResponseDto` (Exitoso, TipoPedido, AccionPostEnvio)
- Crear `FAADatosFiscalesClienteDto` (RFC, RazonSocial, CP, RegimenFiscal, Correo, Moneda, TipoCambio)
- Crear `FAADatosFiscalesEmisorDto` (RFC, RazonSocial, RegimenFiscal, EmpresaClave)
- Crear `FAAPreviewPdfRequestDto` (IdTPProformaAdelanto, UsoCFDI)

**Resultado esperado:**
11 DTOs que cubren el contrato completo de los 4 endpoints del Detalle FAA.

**Entregables:**
- 11 archivos `.cs` en `Application/DTOs/FacturaAdelantado/`

**Criterios de aceptación:**
- `FAAPedidoDetalleDto` incluye campo EstadoFAA (string: PendienteGenerar/PendienteEnviar) y campo EsacCorreo (correo del ESAC asignado al cliente, default del campo CC en modal de Envío)
- `FAAGenerarResponseDto` incluye Exitoso y ErrorDescripcion para manejo de errores SAT
- Los DTOs compilan sin errores

**Más información de la tarea:**
Corresponde a GAP-07. Los DTOs definen el contrato público de los 4 endpoints nuevos.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte B — Endpoints, Request/Response)
- TPSC-RE-FU-019.md (Criterios de Aceptación secciones A-G)

---

### Tarea 10

**Título:** [ TPSC-RE-FU-019 ] [LIST-PAG-MULT-FILTER] Implementar `FacturaAdelantadoDetalleRepository` (consulta vista vtpProformaAdelanto)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Infrastructure/Persistence/Repository

**Consideraciones previas:**
- Depende de Tarea 7 (vista vtpProformaAdelanto en BD) y Tarea 8 (`DbSet<VtpProformaAdelanto>` en contexto)
- La vista resuelve los JOINs complejos y calcula EstadoFAA; el repositorio consulta la vista directamente
- Filtra por IdCliente + RegionClave='MEX' + EstadoFAA IN ('PendienteGenerar','PendienteEnviar')
- Requiere validación de cartera del usuario (ClienteCarteraCliente)
- Debe resolver el correo del ESAC asignado al cliente via `ClienteCartera.IDUsuarioESAC` para pre-rellenar el campo CC del modal de Envío (Criterio F3)

**Objetivo general:**
Implementar el repositorio que consulta los pedidos pendientes de un cliente usando la vista vtpProformaAdelanto.

**Objetivos específicos:**
- Crear `FacturaAdelantadoDetalleRepository` con método `GetPedidosClienteAsync(Guid idCliente, QueryInfo info)`
- Consultar `vtpProformaAdelanto` filtrado por IdCliente, RegionClave='MEX', EstadoFAA != 'Completada', Activo=1
- Implementar paginación con PageSize y DesiredPage
- Implementar ordenamiento dinámico (FechaTramitacion, MontoTotal, FolioPedidoInterno)
- Validar acceso por cartera del usuario (filtro idUsuarioCobrador via ClienteCarteraCliente)
- Obtener correo del ESAC asignado al cliente: JOIN ClienteCarteraCliente → ClienteCartera (IDUsuarioESAC) → vUsuario/CorreoElectronico, incluir como campo EsacCorreo en el resultado

**Resultado esperado:**
Repositorio funcional que retorna los pedidos pendientes del cliente paginados y validados por cartera.

**Entregables:**
- `Infrastructure/Persistence/Repository/FacturaAdelantadoDetalleRepository.cs`
- Interface `IFacturaAdelantadoDetalleRepository` en Application

**Criterios de aceptación:**
- La consulta usa la vista vtpProformaAdelanto (no JOINs directos a tablas base)
- Filtra correctamente por IdCliente + Region MEX + estados pendientes
- Soporta paginación con TotalResults correcto
- Valida que el cliente pertenece a la cartera del usuario logueado
- El campo EsacCorreo se resuelve vía: ClienteCarteraCliente → ClienteCartera.IDUsuarioESAC → correo del usuario ESAC (permite pre-rellenar el campo CC del modal de Envío, Criterio F3)

**Más información de la tarea:**
Corresponde a GAP-08.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte B — Endpoint Detalle, Consulta BD)
- TPSC-RE-FU-019_BD.md (CREATE VIEW vtpProformaAdelanto)

---

### Tarea 11

**Título:** [ TPSC-RE-FU-019 ] [LIST-MULT-FILTER] Implementar `FacturaAdelantadoDatosFiscalesRepository` (datos fiscales cliente + emisor + catálogos SAT + Comentarios de Facturación)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Infrastructure/Persistence/Repository

**Consideraciones previas:**
- Depende de Tarea 8 (FinanzasContext con DbSets)
- Los datos fiscales se obtienen de: DatosFacturacionCliente, Empresa, catRegimenFiscal, catUsoCFDI
- Los datos del contacto se obtienen de: Contacto vinculado al cliente
- Los datos del emisor dependen de la empresa del pedido (Golocaer, Mungen, Proquifa, Proveedora)
- Los Comentarios de Facturación se obtienen de tpPedido y se muestran en solo lectura en el modal de Generación (Criterio B6)
- El Tipo de Cambio del día se obtiene de tabla de tipo de cambio

**Objetivo general:**
Implementar el repositorio que obtiene los datos fiscales y datos complementarios necesarios para el modal de Generación de Factura.

**Objetivos específicos:**
- Crear `FacturaAdelantadoDatosFiscalesRepository` con métodos:
  - `GetDatosFiscalesClienteAsync(Guid idCliente)` — RFC, RazonSocial, CP, RegimenFiscal, Correo, Moneda
  - `GetDatosFiscalesEmisorAsync(Guid idEmpresa)` — RFC, RazonSocial, RegimenFiscal, EmpresaClave
  - `GetContactoClienteAsync(Guid idCliente)` — Nombre, Correo, Teléfono
  - `GetUsoCFDICatalogoAsync()` — Lista de usos CFDI SAT activos
  - `GetComentariosFacturacionAsync(Guid idTPPedido)` — Texto de Comentarios de Facturación del pedido (puede ser null)
- JOIN entre DatosFacturacionCliente + catRegimenFiscal para datos del cliente
- JOIN entre Empresa + datos fiscales emisor para datos de la empresa
- Consulta a tpPedido para obtener el campo ComentariosFacturacion vinculado al pedido

**Resultado esperado:**
Repositorio que provee todos los datos necesarios para armar el modal de Generación (datos fiscales cliente, emisor, contacto, catálogo Uso CFDI y Comentarios de Facturación) y el request de timbrado.

**Entregables:**
- `Infrastructure/Persistence/Repository/FacturaAdelantadoDatosFiscalesRepository.cs`
- Interface `IFacturaAdelantadoDatosFiscalesRepository` en Application

**Criterios de aceptación:**
- `GetDatosFiscalesClienteAsync` retorna datos del registro activo (Activo=1) de DatosFacturacionCliente
- `GetDatosFiscalesEmisorAsync` retorna datos fiscales de la empresa emisora del pedido
- `GetUsoCFDICatalogoAsync` retorna solo registros activos del catálogo SAT
- `GetComentariosFacturacionAsync` retorna el texto de Comentarios de Facturación (null/vacío si no tiene)
- Todos los métodos manejan null safety (cliente sin datos fiscales = error controlado)

**Más información de la tarea:**
Corresponde a GAP-09. Datos fiscales y Comentarios de Facturación alimentan el modal de revisión (Criterio B6) y el request de timbrado.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte B — Datos fiscales)
- TPSC-RE-FU-019.md (Criterios B4, B5, B6 — Datos Fiscales Cliente, Emisor y Comentarios de Facturación)

---

### Tarea 12

**Título:** [ TPSC-RE-FU-019 ] [IMPL-THIRD-SERV] Implementar `ApiCallerTimbrado` (HttpClient + Polly hacia ProquifaDotNet.Timbrado)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Infrastructure/Services

**Consideraciones previas:**
- Finanzas llama a Timbrado via HTTP; usar `IHttpClientFactory` con named client "Timbrado"
- Polly para retry policy ante indisponibilidad (timeout + reintentos limitados)
- Autenticación via token IdentityServer en header Authorization
- Flujo SÍNCRONO: el usuario espera respuesta directa (no se encola en RabbitMQ)

**Objetivo general:**
Implementar el cliente HTTP que permite a Finanzas solicitar timbrado de FAA con resiliencia.

**Objetivos específicos:**
- Crear `Services/ApiCallerTimbrado.cs` con interface `IApiCallerTimbrado`
- Implementar método `TimbrarFAAAsync(TimbrarFAARequestDto)` que llama `POST /api/timbrado/timbrar-faa`
- Configurar Polly: retry 2 intentos, timeout 30s, exponential backoff
- Incluir token IdentityServer en headers de la petición
- Deserializar `TimbrarFAAResponseDto` del response
- Manejar timeout: retornar error controlado al servicio llamante
- Crear `TimbradoSettings.cs` (BaseUrl, Timeout, MaxRetries)
- Registrar en `InfrastructureServiceExtensions` con DI + Polly policies

**Resultado esperado:**
Cliente HTTP funcional con resiliencia para que Finanzas solicite timbrado a la solución Timbrado.

**Entregables:**
- `Services/ApiCallerTimbrado.cs`
- Interface `IApiCallerTimbrado`
- `Configuration/TimbradoSettings.cs`
- Registro en `InfrastructureServiceExtensions`

**Criterios de aceptación:**
- La llamada HTTP incluye autenticación IdentityServer
- Polly ejecuta máximo 2 reintentos con backoff exponencial
- Si timeout excede 30s retorna error controlado (no deja request colgado)
- El HttpClient se resuelve via `IHttpClientFactory` (no `new HttpClient()`)

**Más información de la tarea:**
Corresponde a GAP-13. Flujo síncrono; RabbitMQ es solo para reintentos async del Worker de Timbrado.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte B — Infrastructure Integraciones, Consideraciones Técnicas)

---

### Tarea 13

**Título:** [ TPSC-RE-FU-019 ] [ALG-COMPLX-LOGIC] Implementar `FacturaAdelantadoGenerarService` (orquestación completa: datos fiscales → timbrado → persistencia)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/Services

**Consideraciones previas:**
- Depende de Tareas 9 (DTOs), 11 (DatosFiscalesRepository) y 12 (ApiCallerTimbrado)
- Servicio de ALTA complejidad: orquesta 12 pasos del flujo Generar Factura
- Si PAC falla, retorna error sin modificar estado del pedido
- La factura es inmutable una vez timbrada exitosamente (Regla 9)

**Objetivo general:**
Implementar el servicio que orquesta el flujo completo desde la obtención de datos fiscales hasta la persistencia post-timbrado.

**Objetivos específicos:**
- Crear `Services/FacturaAdelantadoGenerarService.cs` implementando `IFacturaAdelantadoGenerarService`
- Implementar flujo de 12 pasos:

```
1.  Validar que IdTPProformaAdelanto existe y EstadoFAA='PendienteGenerar'
2.  Obtener datos del pedido (partidas, montos, ComentariosFacturacion)
3.  Obtener datos fiscales del cliente (DatosFiscalesRepository)
4.  Obtener datos del emisor (empresa del pedido)
5.  Obtener Tipo de Cambio del día (si moneda != MXN)
6.  Armar TimbrarFAARequestDto con valores forzados SAT (PPD, 99, I)
7.  Llamar ApiCallerTimbrado POST /api/timbrado/timbrar-faa
8.  Si ERROR: retornar FAAGenerarResponseDto con Exitoso=false + ErrorDescripcion
9.  Si ÉXITO: UPDATE tpProformaAdelanto SET IdCFDIGenerada = @idCFDI
10. Almacenar PDF+XML en Minio (bucket 'facturas')
11. INSERT Archivo x2 (PDF + XML, FileBucket='facturas')
12. Retornar FAAGenerarResponseDto con Exitoso=true + datos factura
```

**Resultado esperado:**
Servicio funcional que ejecuta el flujo completo de generación de factura con manejo de errores robusto.

**Entregables:**
- `Services/FacturaAdelantadoGenerarService.cs`
- Interface `IFacturaAdelantadoGenerarService`

**Criterios de aceptación:**
- Si el PAC falla, retorna error sin modificar estado del pedido
- Si el timbrado es exitoso, persiste CFDI + archivos y actualiza tpProformaAdelanto
- La factura es inmutable post-timbrado (rechaza reintentos de generación para misma proforma)
- Validación inicial rechaza pedidos que no estén en estado PendienteGenerar
- La descripción de cada concepto CFDI se construye como "catálogo + descripción + marca"; no se incluye lote ni pedimento (OBS-039)

**Más información de la tarea:**
Corresponde a GAP-10. Servicio de alta complejidad. Ver diagrama "Generar Factura" en Back.md.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte B — Endpoint Generar, Flujo interno, Diagrama)
- TPSC-RE-FU-019.md (Reglas 7, 8, 9 — Generación en dos pasos, errores, persistencia)

---

### Tarea 14

**Título:** [ TPSC-RE-FU-019 ] [ALG-COMPLX-LOGIC] Implementar `FacturaAdelantadoEnviarService` (correo Brevo + Enviada=1 + salida operativa)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/Services

**Consideraciones previas:**
- Depende de Tarea 9 (DTOs)
- El envío usa Brevo con adjuntos PDF+XML obtenidos desde Minio
- Post-envío: Crédito → Finanzas ejecuta directamente la transferencia a Legacy (Tarea 17) / Prepago → Finanzas genera el pendiente en Validar Cobro (Tarea 18)
- Finanzas es responsable de ejecutar la salida operativa completa (no requiere intermediario en Venta Interna)
- Si el correo falla, el estado permanece PendienteEnviar (idempotente)

**Objetivo general:**
Implementar el servicio de envío de factura que envía el correo, marca como enviada y prepara la salida operativa.

**Objetivos específicos:**
- Crear `Services/FacturaAdelantadoEnviarService.cs` implementando `IFacturaAdelantadoEnviarService`
- Implementar flujo:

```
1. Validar que IdTPProformaAdelanto tiene EstadoFAA='PendienteEnviar'
2. Obtener PDF y XML desde Minio (bucket 'facturas')
3. Armar asunto: formato canónico con folio factura + folio pedido interno
4. Enviar correo via Brevo con destinatario, CC, adjuntos PDF+XML, notas
5. Si envío FALLA: retornar error sin modificar estado
6. INSERT CorreoEnviado + ArchivoCorreoEnviado (2 registros: PDF, XML)
7. UPDATE tpProformaAdelanto SET Enviada = 1
8. Si Crédito: ejecutar FacturaAdelantadoLegacyService.TransferirFacturaALegacy
9. Si Prepago: ejecutar FacturaAdelantadoValidarCobroService.GenerarPendienteValidarCobro
10. Retornar FAAEnviarResponseDto con Exitoso=true y AccionPostEnvio
```

**Resultado esperado:**
Servicio funcional que ejecuta el envío de factura y la salida operativa diferenciada.

**Entregables:**
- `Services/FacturaAdelantadoEnviarService.cs`
- Interface `IFacturaAdelantadoEnviarService`

**Criterios de aceptación:**
- El correo se envía con PDF y XML como adjuntos no removibles
- Si el envío falla, el estado permanece PendienteEnviar
- POST-envío Crédito: invoca `FacturaAdelantadoLegacyService` directamente desde Finanzas
- POST-envío Prepago: invoca `FacturaAdelantadoValidarCobroService` directamente desde Finanzas
- INSERT CorreoEnviado registra fecha, destinatario, asunto y archivos adjuntos

**Más información de la tarea:**
Corresponde a GAP-11. Ver diagrama "Enviar Factura" en Back.md. La salida operativa (Legacy + Validar Cobro) se ejecuta directamente desde Finanzas en Tareas 17 y 18.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte B — Endpoint Enviar, Diagrama)
- TPSC-RE-FU-019.md (Reglas 11, 12, 13 — Envío y salida operativa)

---

### Tarea 15

**Título:** [ TPSC-RE-FU-019 ] [IMP-EXIST-SERVICE] Implementar `FacturaAdelantadoPreviewService` (PDF sin timbrar via DocumentBuilder)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/Services

**Consideraciones previas:**
- El PDF preview usa los mismos datos de la factura pero SIN sello fiscal ni UUID (aún no timbrado)
- Usa DocumentBuilder para generar PDF desde template HTML de factura México
- El contenido y estructura del PDF se documentan en requisito independiente; usar template placeholder
- El preview permite al usuario verificar datos antes de confirmar el timbrado (Criterio C1)

**Objetivo general:**
Implementar el servicio que genera el PDF de previsualización de la factura sin timbrar.

**Objetivos específicos:**
- Crear `Services/FacturaAdelantadoPreviewService.cs` implementando `IFacturaAdelantadoPreviewService`
- Implementar flujo:

```
1. Obtener datos del pedido y partidas
2. Obtener datos fiscales cliente y emisor (reutilizar DatosFiscalesRepository)
3. Armar modelo para DocumentBuilder (sin UUID, sin sello, sin cadena original)
4. Llamar DocumentBuilder para generar PDF desde template
5. Retornar PDF como byte[]
```

**Resultado esperado:**
Servicio que genera un PDF preview de la factura para el modal de previsualización.

**Entregables:**
- `Services/FacturaAdelantadoPreviewService.cs`
- Interface `IFacturaAdelantadoPreviewService`

**Criterios de aceptación:**
- El PDF se genera sin datos de timbrado (sin UUID, sin sello)
- El PDF incluye datos fiscales del cliente, emisor y conceptos del pedido
- El servicio retorna `byte[]` consumible por el frontend para renderizar en modal
- Si DocumentBuilder falla, retorna error controlado sin afectar estado del pedido

**Más información de la tarea:**
Corresponde a GAP-12. Contenido visual del PDF definido en requisito independiente.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte B — Endpoint Previsualizar PDF)
- TPSC-RE-FU-019.md (Criterio C1 — Previsualización)

---

### Tarea 16

**Título:** [ TPSC-RE-FU-019 ] [IMP-EXIST-SERVICE] Ampliar `FacturaAdelantadoController` con 4 endpoints de Detalle

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** API/Controllers

**Consideraciones previas:**
- El controlador `FacturaAdelantadoController` fue creado en RE-FU-018 (endpoint `/listar`)
- Se amplía con 4 endpoints nuevos que delegan a sus servicios correspondientes
- Depende de Tareas 10-15 (repositorios y servicios)

**Objetivo general:**
Ampliar el controlador existente con los 4 endpoints del flujo Detalle FAA.

**Objetivos específicos:**
- Agregar `POST /api/factura-adelantado/detalle` → `FacturaAdelantadoDetalleService`
- Agregar `POST /api/factura-adelantado/generar` → `FacturaAdelantadoGenerarService`
- Agregar `POST /api/factura-adelantado/previsualizar-pdf` → `FacturaAdelantadoPreviewService`
- Agregar `POST /api/factura-adelantado/enviar` → `FacturaAdelantadoEnviarService`
- Inyectar las 4 interfaces de servicio via constructor
- Manejar respuestas: 200 OK, 400 (validación), 401 (no autenticado), 500 (error interno)

**Resultado esperado:**
Controlador con 5 endpoints totales (listar + 4 nuevos) que cubren el flujo completo del módulo FAA.

**Entregables:**
- Ampliación de `Controllers/FacturaAdelantadoController.cs`

**Criterios de aceptación:**
- El endpoint `/generar` retorna `FAAGenerarResponseDto` (con error SAT si aplica)
- El endpoint `/previsualizar-pdf` retorna `FileContentResult` con PDF `byte[]`
- El endpoint `/enviar` retorna `FAAEnviarResponseDto` con TipoPedido para salida operativa
- Todos los endpoints validan autenticación IdentityServer

**Más información de la tarea:**
Corresponde a GAP-14. Los endpoints son consumidos por ProquifaDotNet (Venta Interna).

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte B — API FacturaAdelantadoController)

---

---

## Parte C — ProquifaDotNet.Finanzas (Salida Operativa Post-Envío)

---

### Tarea 17

**Título:** [ TPSC-RE-FU-019 ] [SERV-COMPLEX-TRANSACT] Implementar transferencia a Legacy para pedido Crédito post-envío FAA

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/Services

**Consideraciones previas:**
- Se ejecuta SOLO cuando `TipoPedido='Credito'` y la factura se envió exitosamente
- Reutiliza el patrón existente de `ServicioLegacyBO` / `RestClientLegacy` en `L05.TramitarPedido/Legacy/`
- Datos a transferir a Legacy: Factura (UUID, Folio, Serie, Total), Pedido, Partidas, Cobro, PDF
- Si la transferencia a Legacy falla, la factura sigue como enviada (no se revierte el envío)

**Objetivo general:**
Implementar la lógica de transferencia de datos de factura a Legacy para pedidos Crédito tras envío exitoso.

**Objetivos específicos:**
- Crear `Services/FacturaAdelantado/FacturaAdelantadoLegacyService.cs` implementando `IFacturaAdelantadoLegacyService`
- Implementar método `TransferirFacturaALegacy(Guid idTPProformaAdelanto)`:

```
1. Obtener datos de la factura timbrada (CFDI: UUID, Folio, Serie, Total)
2. Obtener datos del pedido (FolioPedidoInterno, IdCliente, partidas)
3. Armar payload para RestClientLegacy
4. Transferir: Factura, Pedido, Partidas, Cobro, PDF
5. Registrar resultado de transferencia
```

- Llamar `RestClientLegacy` de ProquifaDotNet a través de un `ApiCallerVentaInterna` (o equivalente) para ejecutar la transferencia
- Si Legacy falla: registrar error pero NO revertir el envío de la factura

**Resultado esperado:**
Lógica de transferencia funcional que envía datos de factura a Legacy para continuar el flujo Crédito.

**Entregables:**
- `Services/FacturaAdelantado/FacturaAdelantadoLegacyService.cs`

**Criterios de aceptación:**
- La transferencia envía los 5 elementos a Legacy (Factura, Pedido, Partidas, Cobro, PDF)
- Reutiliza `RestClientLegacy` existente (no crea nuevo canal)
- Si Legacy falla, registra error pero NO revierte el envío de factura
- Solo se ejecuta para pedidos Crédito (SinCredito=0 en catCondicionesDePago)

**Más información de la tarea:**
Corresponde a GAP-18. Patrón de referencia: `ServicioLegacyBO` en `L05.TramitarPedido/Legacy/`.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte C — Transferencia Legacy)
- TPSC-RE-FU-019.md (Regla 13 — Salida operativa Crédito)

---

### Tarea 18

**Título:** [ TPSC-RE-FU-019 ] [SERV-TRANSACT] Implementar generación de pendiente Validar Cobro para pedido Prepago post-envío FAA

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/Services

**Consideraciones previas:**
- Se ejecuta SOLO cuando `TipoPedido='Prepago'` y la factura se envió exitosamente
- Genera un registro de pendiente en la tabla de Validar Cobro para que Cobranza valide el pago
- El pendiente vincula la Factura timbrada con el pedido para seguimiento
- Debe ser idempotente: si ya existe pendiente para la misma proforma, no crear otro

**Objetivo general:**
Implementar la lógica de generación del pendiente en Validar Cobro para pedidos Prepago tras envío exitoso.

**Objetivos específicos:**
- Crear `Services/FacturaAdelantado/FacturaAdelantadoValidarCobroService.cs` implementando `IFacturaAdelantadoValidarCobroService`
- Implementar método `GenerarPendienteValidarCobro(Guid idTPProformaAdelanto)`:

```
1. Obtener datos del pedido y la factura enviada
2. Verificar que no exista pendiente previo para esta proforma (idempotencia)
3. INSERT en tabla de pendientes Validar Cobro:
   vincula IdTPProformaAdelanto + IdTPPedido + IdCFDIGenerada + MontoTotal
4. Establecer estado inicial del pendiente como activo/pendiente
5. Registrar resultado
```

- Si la generación falla: registrar error pero NO revertir el envío de factura

**Resultado esperado:**
Lógica funcional que genera el pendiente en Validar Cobro para que Cobranza procese el pago contra la factura emitida.

**Entregables:**
- `Services/FacturaAdelantado/FacturaAdelantadoValidarCobroService.cs`

**Criterios de aceptación:**
- El pendiente se genera solo para pedidos Prepago (SinCredito=1)
- El INSERT vincula correctamente factura + pedido + monto
- Si ya existe un pendiente para la misma proforma adelanto, no se duplica
- El pendiente queda visible en el módulo Validar Cobro para el equipo de Cobranza
- Si la generación falla, registra error pero NO revierte el envío de factura

**Más información de la tarea:**
Corresponde a GAP-19. El módulo Validar Cobro valida el pago del cliente contra esta factura.

**Recursos:**
- TPSC-RE-FU-019-Back.md (Parte C — Pendiente Validar Cobro)
- TPSC-RE-FU-019.md (Regla 13 — Salida operativa Prepago)
