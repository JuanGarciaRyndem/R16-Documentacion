# Tareas BackEnd — R16A-RE-FU-019
**Requisito:** Factura por Adelantado: Detalle México
**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10)

---

## Parte A — ProquifaDotNet.Timbrado (Ampliación)

---

### Tarea 1

**Título:** [ R16A-RE-FU-019 ] [CREATE-SCRIPT-CONTROL] Crear scripts DDL/DML para tabla EmpresaFolio en ProquifaDotNetTimbrado

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
- R16A-RE-FU-019_BD.md (CREATE TABLE EmpresaFolio + INSERT)
- R16A-RE-FU-019-Back.md (Parte D — Scripts, Orden de ejecución)

---

### Tarea 2

**Título:** [ R16A-RE-FU-019 ] [SIMPLE-CRUD] Crear entidad EmpresaFolio e interface `IEmpresaFolioRepository`

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
- La entidad refleja exactamente la estructura de la tabla EmpresaFolio (R16A-RE-FU-019_BD.md)
- La interface define `ConsumeNextFolioAsync` que retorna `Task<int>` (próximo folio)
- El proyecto Domain compila sin errores y sin dependencias externas

**Más información de la tarea:**
Corresponde a GAP-01. El folio se consume atómicamente solo al timbrar exitosamente (sin huecos por errores PAC).

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte A — Domain)
- R16A-RE-FU-019_BD.md (CREATE TABLE EmpresaFolio)

---

### Tarea 3

**Título:** [ R16A-RE-FU-019 ] [IMP-EXIST-SERVICE] Implementar `EmpresaFolioRepository` con UPDATE atómico (UPDLOCK)

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Infrastructure/Persistence/Repository

**Consideraciones previas:**
- Depende de Tarea 1 (tabla EmpresaFolio en BD) y Tarea 2 (interface `IEmpresaFolioRepository`)
- El consumo de folio debe ser thread-safe ante concurrencia (múltiples timbrados simultáneos)
- Usar raw SQL con hints UPDLOCK, ROWLOCK y OUTPUT INSERTED para atomicidad
- Agregar `DbSet<EmpresaFolio>` al StampingContext existente

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
- Agregar `DbSet<EmpresaFolio>` y mapeo Fluent API en StampingContext

**Resultado esperado:**
Repositorio funcional que consume folios sin posibilidad de duplicados bajo concurrencia.

**Entregables:**
- `Repository/EmpresaFolioRepository.cs`
- Ampliación de `StampingContext` (DbSet + mapeo)

**Criterios de aceptación:**
- `ConsumeNextFolioAsync` usa UPDATE con UPDLOCK, ROWLOCK
- Bajo concurrencia no se producen folios duplicados
- El mapeo Fluent API respeta tipos de columna y constraint UNIQUE en EmpresaClave

**Más información de la tarea:**
Corresponde a GAP-03. Patrón documentado en R16A-RE-FU-019_BD.md sección "Consumo atómico del folio".

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte A — Infrastructure)
- R16A-RE-FU-019_BD.md (Consumo atómico del folio)

---

### Tarea 4

**Título:** [ R16A-RE-FU-019 ] [SIMPLE-CRUD] Crear DTOs de request/response para timbrado FAA

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Application/DTOs

**Consideraciones previas:**
- Los DTOs modelan el contrato del endpoint técnico `POST /api/v1/stamp/invoice` (ProquifaDotNet.Timbrado) — **no** `/api/v1/cfdi` (ese es el recurso de negocio expuesto por `CfdiController` en Finanzas, ver R16A-RE-FU-018-Back.md Parte B)
- Valores forzados por normativa SAT (PPD, 99, I) se incluyen como propiedades con defaults
- El response indica éxito/error con el resultado del timbrado (UUID, XML, Serie, Folio) — **sin Id de negocio**: Timbrado no tiene tabla `Cfdi` propia, el Id real (`IdCFDIGenerada`) lo asigna Finanzas al persistir

**Objetivo general:**
Crear los modelos DTO específicos para el timbrado técnico de Factura por Adelantado.

**Objetivos específicos:**
- Crear `DTOs/StampAdvanceInvoiceRequestDto.cs` con: IdProformaAdelanto, RecipientData, IssuerData, `Conceptos[]`, MetodoPago (default "PPD"), FormaPago (default "99"), TipoComprobante (default "I"), Moneda, TipoCambio
- Crear `DTOs/AdvanceInvoiceItemDto.cs` con: Cantidad, Descripcion, PrecioUnitario, Importe, ClaveUnidad, ClaveProdServ — el campo Descripcion debe construirse como "catálogo + descripción + marca"; no incluir lote ni pedimento (OBS-039)
- Crear `DTOs/StampAdvanceInvoiceResponseDto.cs` con: Uuid, Serie, Folio, FechaEmision, Total, XmlBase64, Exitoso, ErrorDescripcion, ErrorCodigo — sin IdCFDI
- Crear `DTOs/RecipientDataDto.cs` con: RFC, RazonSocial, CP, RegimenFiscal, UsoCFDI
- Crear `DTOs/IssuerDataDto.cs` con: RFC, RazonSocial, RegimenFiscal, EmpresaClave

**Resultado esperado:**
DTOs completos que modelan el contrato de comunicación del endpoint técnico `/api/v1/stamp/invoice`.

**Entregables:**
- `DTOs/StampAdvanceInvoiceRequestDto.cs`
- `DTOs/AdvanceInvoiceItemDto.cs`
- `DTOs/StampAdvanceInvoiceResponseDto.cs`
- `DTOs/RecipientDataDto.cs`
- `DTOs/IssuerDataDto.cs`

**Criterios de aceptación:**
- MetodoPago, FormaPago y TipoComprobante tienen valores default forzados por normativa SAT
- `StampAdvanceInvoiceResponseDto` incluye campo Exitoso (bool) y ErrorDescripcion para manejo de errores PAC, y NO incluye ningún Id de negocio
- Los DTOs compilan sin errores

**Más información de la tarea:**
Corresponde a GAP-04. Valores forzados SAT para factura PPD documentados en R16A-RE-FU-019.md (Regla 5).

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte A — DTOs nuevos para FAA)
- R16A-RE-FU-019.md (Regla 5 — Forma de Pago, Método de Pago forzados)

---

### Tarea 5

**Título:** [ R16A-RE-FU-019 ] [ALG-COMPLX-LOGIC] Ampliar StampingService con `EmpresaFolioService` y método `StampAdvanceInvoiceAsync`

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Application/Services

**Consideraciones previas:**
- Depende de Tarea 3 (`EmpresaFolioRepository`) y Tarea 4 (DTOs)
- `StampingService` ya existe (RE-FU-018); se amplía con método específico para FAA
- El nuevo método orquesta: validar datos fiscales → consumir folio → armar XML → llamar SAP → registrar StampingLog → regresar resultado a Finanzas (Timbrado **no persiste** el CFDI como entidad de negocio)
- Si SAP retorna error, retornar `StampAdvanceInvoiceResponseDto` con Exitoso=false sin modificar BD

**Objetivo general:**
Ampliar la capa Application con el servicio de consumo de folio y el método de timbrado de Factura por Adelantado.

**Objetivos específicos:**
- Crear `Interfaces/IEmpresaFolioService.cs` con métodos `GetNextFolioAsync` y `GetByClaveAsync`
- Crear `Services/EmpresaFolioService.cs` que consume folio atómico y retorna folio formateado (padding a 6 chars)
- Ampliar `IStampingService.cs` agregando `StampAdvanceInvoiceAsync(StampAdvanceInvoiceRequestDto)`
- Ampliar `StampingService.cs` implementando el flujo:

```
1. Validar request (FluentValidation)
2. Consumir folio via EmpresaFolioService.GetNextFolioAsync(EmpresaClave)
3. Armar estructura XML CFDI con datos fiscales + conceptos
4. INSERT StampingLog (NewStatus=Pending)
5. Llamar SapStampingClient -> PAC SAP
6. Si ERROR: UPDATE StampingLog (NewStatus=Failed, ErrorMessage) y retornar StampAdvanceInvoiceResponseDto con Exitoso=false + ErrorDescripcion (sin persistir ningun CFDI de negocio)
7. Si ÉXITO: UPDATE StampingLog (NewStatus=Stamped)
8. Retornar StampAdvanceInvoiceResponseDto con Exitoso=true, Uuid, Serie, Folio, FechaEmision, XmlBase64 (el XML se regresa en el response; Finanzas es quien lo sube a Minio y lo persiste, ver R16A-RE-FU-018-Back.md Parte B)
```

> Nota: los pasos "INSERT/UPDATE CFDI" y "Subir XML a Minio (bucket 'timbrado')" de versiones previas de esta tarea se eliminaron — Timbrado no tiene tabla `Cfdi` propia ni sube el XML a Minio; ambas responsabilidades son de Finanzas (`CfdiService`, ver R16A-RE-FU-018-Back.md Parte B).

**Resultado esperado:**
Servicio de timbrado FAA funcional que orquesta el flujo técnico completo con manejo de errores PAC, sin persistir el CFDI como entidad de negocio.

**Entregables:**
- `Interfaces/IEmpresaFolioService.cs`
- `Services/EmpresaFolioService.cs`
- Ampliación de `IStampingService.cs` y `StampingService.cs`

**Criterios de aceptación:**
- El folio se consume SOLO al timbrar exitosamente (no se incrementa si SAP falla)
- Si SAP retorna error, el response incluye ErrorDescripcion y Exitoso=false sin modificar folio
- `StampingLog` registra request, response, duración y error (si aplica)

**Más información de la tarea:**
Corresponde a GAP-02. Ver diagrama de flujo "Generar Factura" en R16A-RE-FU-019-Back.md.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte A — Application, Flujo Generar Factura)
- R16A-RE-FU-018-Back.md (Flujo Funcional de Timbrado — referencia base)

---

### Tarea 6

**Título:** [ R16A-RE-FU-019 ] [IMP-EXIST-SERVICE] Enrutar timbrado FAA a través de `POST /api/v1/stamp/invoice` en StampingController

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** API/Controllers

**Consideraciones previas:**
- Depende de Tarea 5 (`StampingService.StampAdvanceInvoiceAsync`) y Tarea 4 (DTOs)
- `StampingController` y el endpoint técnico `POST /api/v1/stamp/invoice` ya existen (RE-FU-018); no se crea endpoint ni ruta nueva
- Timbrado no discrimina por catálogo propio (no tiene `FiscalDocumentType`): el tipo de documento lo resuelve Finanzas (via `catTipoCFDI`) antes de llamar a Timbrado; Timbrado solo recibe los datos fiscales ya armados
- El recurso de negocio `cfdi` (`CfdiController`, `POST /api/v1/cfdi`) vive en Finanzas, no aquí (ver R16A-RE-FU-018-Back.md Parte B)
- Autenticación via IdentityServer (token desde Finanzas)

**Objetivo general:**
Conectar el timbrado técnico de Factura por Adelantado al endpoint de facturas (`POST /api/v1/stamp/invoice`) del controlador existente.

**Objetivos específicos:**
- Ampliar la acción `POST` de `StampingController` para que delegue a `IStampingService.StampAdvanceInvoiceAsync([FromBody] StampAdvanceInvoiceRequestDto request)`
- Ruta: `/api/v1/stamp/invoice` (sin ruta ni recurso separados)
- Retornar 200 OK con `StampAdvanceInvoiceResponseDto` (tanto en éxito como en error de PAC)
- Retornar 400 BadRequest si validación de request falla
- Retornar 500 si error interno no controlado

**Resultado esperado:**
Endpoint técnico de facturas funcional que recibe la solicitud de timbrado FAA desde Finanzas y retorna el resultado (sin persistir el CFDI).

**Entregables:**
- Ampliación de `Controllers/StampingController.cs`

**Criterios de aceptación:**
- El endpoint es accesible en `/api/v1/stamp/invoice`
- Valida autenticación IdentityServer
- Retorna `StampAdvanceInvoiceResponseDto` con Exitoso=true y datos del timbrado cuando es exitoso
- Retorna `StampAdvanceInvoiceResponseDto` con Exitoso=false y ErrorDescripcion cuando PAC falla
- Errores de validación retornan 400

**Más información de la tarea:**
Corresponde a GAP-05. Consumido exclusivamente por ProquifaDotNet.Finanzas (via `IApiCallerStamping`, creado en RE-FU-018 Parte B).

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte A — API StampingController)

---

## Parte B — ProquifaDotNet.Finanzas (Módulo FAA — Detalle)

---

### Tarea 7

> **Tarea eliminada de este requisito — migrada a R16A-RE-FU-015 (06/07/2026):** el `ALTER TABLE tpProformaAdelanto ADD Enviada` y el `CREATE VIEW dbo.vtpProformaAdelanto` (cadena de 8 tablas) originalmente planeados aquí se retiran por completo. La columna `Enviada` y la vista equivalente (`vfccFactura`, que unifica origen Prepago y Crédito sobre la tabla `fccFactura`) se crean en **R16A-RE-FU-015-Tareas.md** como parte de la tarea de BD de ese requisito. Razón: `RE-FU-015` no genera registros en `tpProformaAdelanto` (usa el nuevo esquema `fccFactura` desde su origen), por lo que esta vista nunca hubiera cubierto los pendientes FAA de Prepago (ver hallazgo H-01 en `R16A-RE-FU-018_DIS-SOL_Revision.md`, resuelto el 06/07/2026). Las Tareas 8-18 de este documento ahora dependen de `fccFactura`/`vfccFactura` (R16A-RE-FU-015) en vez de `tpProformaAdelanto`/`vtpProformaAdelanto`. No se aporta tarea de BD propia en este requisito.

---

### Tarea 8

**Título:** [ R16A-RE-FU-019 ] [SIMPLE-CRUD] Ampliar FinanzasContext con `DbSet` de vista vfccFactura y tablas auxiliares

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Infrastructure/Persistence/Context

**Consideraciones previas:**
- Depende de que `fccFactura`/`vfccFactura` ya existan en BD (script DDL en **R16A-RE-FU-015-Tareas.md**, no en este requisito — ver nota en Tarea 7)
- El FinanzasContext fue creado en RE-FU-016 y ampliado en RE-FU-018
- La vista vfccFactura se mapea con `HasNoKey()` y `ToView()` (solo lectura)
- Las tablas Archivo, CorreoEnviado, ArchivoCorreoEnviado se usan para escritura post-timbrado/envío

**Objetivo general:**
Ampliar el contexto EF Core con los mapeos necesarios para el flujo Detalle FAA.

**Objetivos específicos:**
- Agregar `DbSet<VfccFactura>` mapeado con `.ToView("vfccFactura").HasNoKey()` (reemplaza el `DbSet<VtpProformaAdelanto>` planeado originalmente sobre `vtpProformaAdelanto`)
- Agregar `DbSet<Archivo>` con mapeo a tabla Archivo
- Agregar `DbSet<CorreoEnviado>` con mapeo a tabla CorreoEnviado
- Agregar `DbSet<ArchivoCorreoEnviado>` con mapeo a tabla ArchivoCorreoEnviado
- Agregar `DbSet<CatUsoCFDI>` con mapeo a tabla catUsoCFDI
- Configurar Fluent API con tipos de columna correctos (varchar, uniqueidentifier, datetime2, bit)

**Resultado esperado:**
FinanzasContext con todos los DbSets necesarios para consultar la vista y persistir registros del flujo FAA.

**Entregables:**
- Ampliación de `Context/FinanzasContext.cs`
- Entidades de dominio: `VfccFactura.cs`, `Archivo.cs`, `CorreoEnviado.cs`, `ArchivoCorreoEnviado.cs`, `CatUsoCFDI.cs`

**Criterios de aceptación:**
- `VfccFactura` se mapea como vista (HasNoKey, sin tracking de cambios)
- Las entidades de escritura (Archivo, CorreoEnviado) permiten INSERT sin errores
- El contexto compila sin conflictos con DbSets existentes
- Los tipos de columna coinciden con la BD

**Más información de la tarea:**
Corresponde a GAP-15. La vista vfccFactura (R16A-RE-FU-015) resuelve el acceso directo a `fccFactura` sin la cadena de JOINs original.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte B — FinanzasContext Ampliación)
- R16A-RE-FU-015_BD.md (columnas de vfccFactura)

---

### Tarea 9

**Título:** [ R16A-RE-FU-019 ] [SIMPLE-CRUD] Crear DTOs del módulo FAA Detalle (11 modelos)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/DTOs

**Consideraciones previas:**
- Los DTOs cubren: request/response del detalle, cabecera cliente, pedido con estado, datos fiscales cliente/emisor, generación, envío y preview
- Valores forzados SAT: MetodoPago=PPD, FormaPago=99, TipoComprobante=I

**Objetivo general:**
Crear los modelos DTO completos para todos los endpoints del Detalle FAA.

**Objetivos específicos:**
- Crear `AdvanceInvoiceDetailRequestDto` (IdCliente, `Filters[]`, SortField, SortDirection, PageSize, DesiredPage)
- Crear `AdvanceInvoiceDetailResponseDto` (Cliente: cabecera, TotalResults, `Results[]`)
- Crear `ClientHeaderDto` (IdCliente, RazonSocial, RFC, MonedaFacturacion, ClasificacionCrediticia)
- Crear `OrderDetailDto` (IdFccFactura, IdTPPedido, FolioPedidoInterno, FechaPedido, CondicionesDePago, EsPrepago, EmpresaAlias, EmpresaClave, Subtotal, IVA, MontoTotal, Moneda, EstadoFAA, FolioFactura, SerieFactura, EsacCorreo)
- Crear `AdvanceInvoiceGenerateRequestDto` (IdFccFactura, UsoCFDI)
- Crear `AdvanceInvoiceGenerateResponseDto` (Exitoso, IdCFDIGenerada, UUID, Folio, Serie, FechaEmision, Total, PdfUrl, XmlUrl, ErrorDescripcion, ErrorCodigo)
- Crear `AdvanceInvoiceSendRequestDto` (IdFccFactura, Destinatario, CC, Notas)
- Crear `AdvanceInvoiceSendResponseDto` (Exitoso, TipoPedido, AccionPostEnvio)
- Crear `ClientFiscalDataDto` (RFC, RazonSocial, CP, RegimenFiscal, Correo, Moneda, TipoCambio)
- Crear `IssuerFiscalDataDto` (RFC, RazonSocial, RegimenFiscal, EmpresaClave)
- Crear `AdvanceInvoicePreviewPdfRequestDto` (IdFccFactura, UsoCFDI)

**Resultado esperado:**
11 DTOs que cubren el contrato completo de los 4 endpoints del Detalle FAA.

**Entregables:**
- 11 archivos `.cs` en `Application/DTOs/AdvanceInvoice/`

**Criterios de aceptación:**
- `OrderDetailDto` incluye campo EstadoFAA (string: PendienteGenerar/PendienteEnviar) y campo EsacCorreo (correo del ESAC asignado al cliente, default del campo CC en modal de Envío)
- `AdvanceInvoiceGenerateResponseDto` incluye Exitoso y ErrorDescripcion para manejo de errores SAT
- Los DTOs compilan sin errores

**Más información de la tarea:**
Corresponde a GAP-07. Los DTOs definen el contrato público de los 4 endpoints nuevos.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte B — Endpoints, Request/Response)
- R16A-RE-FU-019.md (Criterios de Aceptación secciones A-G)

---

### Tarea 10

**Título:** [ R16A-RE-FU-019 ] [LIST-PAG-MULT-FILTER] Implementar `AdvanceInvoiceDetailRepository` (consulta vista vfccFactura)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Infrastructure/Persistence/Repository

**Consideraciones previas:**
- Depende de que `vfccFactura` ya exista en BD (R16A-RE-FU-015-Tareas.md) y de Tarea 8 (`DbSet<VfccFactura>` en contexto)
- La vista resuelve el acceso directo a `fccFactura` y calcula EstadoFAA sin la cadena de JOINs original de `vtpProformaAdelanto`
- Filtra por IdCliente + RegionClave='MEX' + EstadoFAA IN ('PendienteGenerar','PendienteEnviar')
- Requiere validación de cartera del usuario (ClienteCarteraCliente)
- Debe resolver el correo del ESAC asignado al cliente via `ClienteCartera.IDUsuarioESAC` para pre-rellenar el campo CC del modal de Envío (Criterio F3)

**Objetivo general:**
Implementar el repositorio que consulta los pedidos pendientes de un cliente usando la vista vfccFactura.

**Objetivos específicos:**
- Crear `AdvanceInvoiceDetailRepository` con método `GetPedidosClienteAsync(Guid idCliente, QueryInfo info)`
- Consultar `vfccFactura` filtrado por IdCliente, RegionClave='MEX', EstadoFAA != 'Completada', Activo=1
- Implementar paginación con PageSize y DesiredPage
- Implementar ordenamiento dinámico (FechaTramitacion, MontoTotal, FolioPedidoInterno)
- Validar acceso por cartera del usuario (filtro idUsuarioCobrador via ClienteCarteraCliente)
- Obtener correo del ESAC asignado al cliente: JOIN ClienteCarteraCliente → ClienteCartera (IDUsuarioESAC) → vUsuario/CorreoElectronico, incluir como campo EsacCorreo en el resultado

**Resultado esperado:**
Repositorio funcional que retorna los pedidos pendientes del cliente paginados y validados por cartera.

**Entregables:**
- `Infrastructure/Persistence/Repository/AdvanceInvoiceDetailRepository.cs`
- Interface `IAdvanceInvoiceDetailRepository` en Application

**Criterios de aceptación:**
- La consulta usa la vista vfccFactura (no JOINs directos a tablas base)
- Filtra correctamente por IdCliente + Region MEX + estados pendientes
- Soporta paginación con TotalResults correcto
- Valida que el cliente pertenece a la cartera del usuario logueado
- El campo EsacCorreo se resuelve vía: ClienteCarteraCliente → ClienteCartera.IDUsuarioESAC → correo del usuario ESAC (permite pre-rellenar el campo CC del modal de Envío, Criterio F3)

**Más información de la tarea:**
Corresponde a GAP-08.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte B — Endpoint Detalle, Consulta BD)
- R16A-RE-FU-015_BD.md (CREATE VIEW vfccFactura)

---

### Tarea 11

**Título:** [ R16A-RE-FU-019 ] [LIST-MULT-FILTER] Implementar `AdvanceInvoiceFiscalDataRepository` (datos fiscales cliente + emisor + catálogos SAT + Comentarios de Facturación)

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
- Crear `AdvanceInvoiceFiscalDataRepository` con métodos:
  - `GetClientFiscalDataAsync(Guid idCliente)` — RFC, RazonSocial, CP, RegimenFiscal, Correo, Moneda
  - `GetIssuerFiscalDataAsync(Guid idEmpresa)` — RFC, RazonSocial, RegimenFiscal, EmpresaClave
  - `GetClientContactAsync(Guid idCliente)` — Nombre, Correo, Teléfono
  - `GetCfdiUsageCatalogAsync()` — Lista de usos CFDI SAT activos
  - `GetComentariosFacturacionAsync(Guid idTPPedido)` — Texto de Comentarios de Facturación del pedido (puede ser null)
- JOIN entre DatosFacturacionCliente + catRegimenFiscal para datos del cliente
- JOIN entre Empresa + datos fiscales emisor para datos de la empresa
- Consulta a tpPedido para obtener el campo ComentariosFacturacion vinculado al pedido

**Resultado esperado:**
Repositorio que provee todos los datos necesarios para armar el modal de Generación (datos fiscales cliente, emisor, contacto, catálogo Uso CFDI y Comentarios de Facturación) y el request de timbrado.

**Entregables:**
- `Infrastructure/Persistence/Repository/AdvanceInvoiceFiscalDataRepository.cs`
- Interface `IAdvanceInvoiceFiscalDataRepository` en Application

**Criterios de aceptación:**
- `GetClientFiscalDataAsync` retorna datos del registro activo (Activo=1) de DatosFacturacionCliente
- `GetIssuerFiscalDataAsync` retorna datos fiscales de la empresa emisora del pedido
- `GetCfdiUsageCatalogAsync` retorna solo registros activos del catálogo SAT
- `GetComentariosFacturacionAsync` retorna el texto de Comentarios de Facturación (null/vacío si no tiene)
- Todos los métodos manejan null safety (cliente sin datos fiscales = error controlado)

**Más información de la tarea:**
Corresponde a GAP-09. Datos fiscales y Comentarios de Facturación alimentan el modal de revisión (Criterio B6) y el request de timbrado.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte B — Datos fiscales)
- R16A-RE-FU-019.md (Criterios B4, B5, B6 — Datos Fiscales Cliente, Emisor y Comentarios de Facturación)

---

### Tarea 12

> **Tarea ya cubierta en RE-FU-018 (Parte B, Tarea 10 — no se duplica aquí):** `IApiCallerStamping`/`ApiCallerStamping` (cliente HTTP hacia `POST /api/v1/stamp/invoice` en ProquifaDotNet.Timbrado, sin reintentos, con Polly solo para timeout) se crea en **R16A-RE-FU-018-Tareas.md, Parte B, Tarea 10**, junto con `CfdiService` (Tarea 9) y `CfdiController` (Tarea 12 de ese requisito). Este requisito (RE-FU-019) **consume** esos componentes existentes desde `AdvanceInvoiceGenerateService` (ver Tarea 13) — no crea un `ApiCallerStamping` propio ni llama a Timbrado directamente.

**Título:** [ R16A-RE-FU-019 ] [IMP-EXIST-SERVICE] Verificar disponibilidad de `IApiCallerStamping`/`ICfdiService` (dependencia de RE-FU-018)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/Interfaces

**Consideraciones previas:**
- `IApiCallerStamping`, `ApiCallerStamping`, `ICfdiService` y `CfdiService` ya fueron creados en R16A-RE-FU-018 (Parte B, Tareas 9 y 10) — este requisito solo verifica que estén disponibles e inyectables antes de implementar `AdvanceInvoiceGenerateService` (Tarea 13)
- No se crea ningún cliente HTTP nuevo hacia Timbrado en este requisito

**Objetivo general:**
Confirmar que `ICfdiService` (y transitivamente `IApiCallerStamping`) de RE-FU-018 están disponibles para ser consumidos por el flujo de Generar Factura de este requisito.

**Objetivos específicos:**
- Verificar registro en DI de `ICfdiService` en el proyecto Application de Finanzas
- Verificar que `CfdiService.GenerateAsync` acepta los datos fiscales de Factura por Adelantado (Recipient/Issuer/Items) sin requerir cambios

**Resultado esperado:**
Confirmación de que `AdvanceInvoiceGenerateService` (Tarea 13) puede inyectar `ICfdiService` directamente, sin necesidad de un cliente HTTP adicional.

**Entregables:**
- Nota de verificación (sin código nuevo si RE-FU-018 ya cubre el contrato)

**Criterios de aceptación:**
- `ICfdiService` se resuelve correctamente via DI en Finanzas
- No existe una segunda implementación de `IApiCallerStamping` duplicada

**Más información de la tarea:**
Corresponde a GAP-13 (ya satisfecho por RE-FU-018 GAP-10/GAP-11; esta tarea es de verificación, no de construcción).

**Recursos:**
- R16A-RE-FU-018-Tareas.md (Parte B, Tareas 9 y 10)
- R16A-RE-FU-019-Back.md (Parte B — Infrastructure Integraciones, Consideraciones Técnicas)

---

### Tarea 13

**Título:** [ R16A-RE-FU-019 ] [ALG-COMPLX-LOGIC] Implementar `AdvanceInvoiceGenerateService` (orquestación completa: datos fiscales → timbrado → persistencia)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/Services

**Consideraciones previas:**
- Depende de Tareas 9 (DTOs), 11 (AdvanceInvoiceFiscalDataRepository) y 12 (verificación de `ICfdiService`/`IApiCallerStamping`, ya creados en RE-FU-018)
- Servicio de ALTA complejidad: orquesta el flujo Generar Factura delegando el timbrado y la persistencia del CFDI a `ICfdiService` (RE-FU-018, Parte B) — **no llama a Timbrado directamente ni persiste CFDIGenerada/Archivo por su cuenta**, para no duplicar esa lógica
- Si PAC falla, retorna error sin modificar estado del pedido
- La factura es inmutable una vez timbrada exitosamente (Regla 9)

**Objetivo general:**
Implementar el servicio que orquesta el flujo completo desde la obtención de datos fiscales hasta la actualización de `fccFactura` post-timbrado, delegando el timbrado y la persistencia del CFDI a `ICfdiService`.

**Objetivos específicos:**
- Crear `Services/AdvanceInvoiceGenerateService.cs` implementando `IAdvanceInvoiceGenerateService`
- Implementar flujo:

```
1. Validar que IdFccFactura existe y EstadoFAA='PendienteGenerar' (vfccFactura)
2. Obtener datos del pedido (partidas, montos, ComentariosFacturacion)
3. Obtener datos fiscales del cliente (AdvanceInvoiceFiscalDataRepository)
4. Obtener datos del emisor (empresa del pedido)
5. Obtener Tipo de Cambio del día (si moneda != MXN)
6. Armar StampAdvanceInvoiceRequestDto con valores forzados SAT (PPD, 99, I)
7. Llamar ICfdiService.GenerateAsync(request) — internamente: llama IApiCallerStamping -> Timbrado
   POST /api/v1/stamp/invoice, y si es exitoso INSERT CFDIGenerada + Archivo (XML) en ProquifaDotNet
8. Si ERROR: UPDATE fccFactura SET IdCatFacturaEstado = ERROR_TIMBRADO (sin tocar IdCFDIGenerada) y
   retornar AdvanceInvoiceGenerateResponseDto con Exitoso=false + ErrorDescripcion
9. Si ÉXITO: UPDATE fccFactura SET IdCFDIGenerada = @IdCFDIGenerada, EsFacturaPorAdelantado = 0,
   IdCatFacturaEstado = GENERADA (catFacturaEstado, RE-FU-015 v2.1)
   (Id real retornado por ICfdiService.GenerateAsync, correspondiente al registro insertado en CFDIGenerada)
10. Retornar AdvanceInvoiceGenerateResponseDto con Exitoso=true + datos factura
```

> Nota: los pasos "Almacenar PDF+XML en Minio" e "INSERT Archivo" de versiones previas de esta tarea ya no los ejecuta `AdvanceInvoiceGenerateService` — `ICfdiService.GenerateAsync` (RE-FU-018) se encarga de la subida del XML a Minio y de vincularlo en `CFDIGenerada.IdArchivoXml`. Este servicio solo persiste el PDF (que no es responsabilidad de Timbrado ni de CfdiService) si aplica, o delega tambien esa persistencia segun se defina en el detalle de DocumentBuilder.

**Resultado esperado:**
Servicio funcional que ejecuta el flujo completo de generación de factura con manejo de errores robusto, sin duplicar la logica de timbrado/persistencia del CFDI ya resuelta en RE-FU-018.

**Entregables:**
- `Services/AdvanceInvoiceGenerateService.cs`
- Interface `IAdvanceInvoiceGenerateService`

**Criterios de aceptación:**
- Si el PAC falla, retorna error sin modificar estado del pedido
- Si el timbrado es exitoso, `ICfdiService.GenerateAsync` persiste CFDIGenerada + Archivo y este servicio actualiza `fccFactura.IdCFDIGenerada` (y `EsFacturaPorAdelantado=0`) con el Id real retornado
- La factura es inmutable post-timbrado (rechaza reintentos de generación para misma proforma)
- Validación inicial rechaza pedidos que no estén en estado PendienteGenerar
- La descripción de cada concepto CFDI se construye como "catálogo + descripción + marca"; no se incluye lote ni pedimento (OBS-039)

**Más información de la tarea:**
Corresponde a GAP-10. Servicio de alta complejidad. Ver diagrama "Generar Factura" en Back.md.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte B — Endpoint Generar, Flujo interno, Diagrama)
- R16A-RE-FU-019.md (Reglas 7, 8, 9 — Generación en dos pasos, errores, persistencia)

---

### Tarea 14

**Título:** [ R16A-RE-FU-019 ] [ALG-COMPLX-LOGIC] Implementar `AdvanceInvoiceSendService` (correo vía ProquifaDotNet.EnvioCorreo + Enviada=1 + salida operativa)

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application/Services

**Consideraciones previas:**
- Depende de Tarea 9 (DTOs)
- El envío usa ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo — regla 7, sin cliente Brevo propio de Finanzas) con adjuntos PDF+XML obtenidos desde Minio
- Post-envío: Crédito → Finanzas ejecuta directamente la transferencia a Legacy (Tarea 17) / Prepago → Finanzas genera el pendiente en Validar Cobro (Tarea 18)
- Finanzas es responsable de ejecutar la salida operativa completa (no requiere intermediario en Venta Interna)
- Si el correo falla, el estado permanece PendienteEnviar (idempotente)

**Objetivo general:**
Implementar el servicio de envío de factura que envía el correo, marca como enviada y prepara la salida operativa.

**Objetivos específicos:**
- Crear `Services/AdvanceInvoiceSendService.cs` implementando `IAdvanceInvoiceSendService`
- Implementar flujo:

```
1. Validar que IdFccFactura tiene EstadoFAA='PendienteEnviar' (vfccFactura)
2. Obtener PDF y XML desde Minio (bucket 'facturas')
3. Armar asunto: formato canónico con folio factura + folio pedido interno
4. Enviar correo vía ProquifaDotNet.EnvioCorreo con destinatario, CC, adjuntos PDF+XML, notas
5. Si envío FALLA: retornar error sin modificar estado
6. INSERT CorreoEnviado + ArchivoCorreoEnviado (2 registros: PDF, XML)
7. UPDATE fccFactura SET Enviada = 1, FechaEnvio = SYSUTCDATETIME(), IdCatFacturaEstado = ENVIADA
   (antes: UPDATE tpProformaAdelanto SET Enviada = 1)
8. Registrar el guardado/envío de la factura en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo — regla 8)
9. Si Crédito (fccFactura.IdTPProformaPedido NOT NULL): ejecutar AdvanceInvoiceLegacyService.TransferInvoiceToLegacy
10. Si Prepago (fccFactura.IdTPProformaPedido NULL): ejecutar AdvanceInvoiceValidateCollectionService.GenerarPendienteValidarCobro
11. Retornar AdvanceInvoiceSendResponseDto con Exitoso=true y AccionPostEnvio
```

**Resultado esperado:**
Servicio funcional que ejecuta el envío de factura y la salida operativa diferenciada.

**Entregables:**
- `Services/AdvanceInvoiceSendService.cs`
- Interface `IAdvanceInvoiceSendService`

**Criterios de aceptación:**
- El correo se envía con PDF y XML como adjuntos no removibles
- Si el envío falla, el estado permanece PendienteEnviar
- POST-envío Crédito: invoca `AdvanceInvoiceLegacyService` directamente desde Finanzas
- POST-envío Prepago: invoca `AdvanceInvoiceValidateCollectionService` directamente desde Finanzas
- INSERT CorreoEnviado registra fecha, destinatario, asunto y archivos adjuntos

**Más información de la tarea:**
Corresponde a GAP-11. Ver diagrama "Enviar Factura" en Back.md. La salida operativa (Legacy + Validar Cobro) se ejecuta directamente desde Finanzas en Tareas 17 y 18.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte B — Endpoint Enviar, Diagrama)
- R16A-RE-FU-019.md (Reglas 11, 12, 13 — Envío y salida operativa)

---

### Tarea 15

**Título:** [ R16A-RE-FU-019 ] [IMP-EXIST-SERVICE] Implementar `AdvanceInvoicePreviewService` (PDF sin timbrar via DocumentBuilder)

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
- Crear `Services/AdvanceInvoicePreviewService.cs` implementando `IAdvanceInvoicePreviewService`
- Implementar flujo:

```
1. Obtener datos del pedido y partidas
2. Obtener datos fiscales cliente y emisor (reutilizar AdvanceInvoiceFiscalDataRepository)
3. Armar modelo para DocumentBuilder (sin UUID, sin sello, sin cadena original)
4. Llamar DocumentBuilder para generar PDF desde template
5. Retornar PDF como byte[]
```

**Resultado esperado:**
Servicio que genera un PDF preview de la factura para el modal de previsualización.

**Entregables:**
- `Services/AdvanceInvoicePreviewService.cs`
- Interface `IAdvanceInvoicePreviewService`

**Criterios de aceptación:**
- El PDF se genera sin datos de timbrado (sin UUID, sin sello)
- El PDF incluye datos fiscales del cliente, emisor y conceptos del pedido
- El servicio retorna `byte[]` consumible por el frontend para renderizar en modal
- Si DocumentBuilder falla, retorna error controlado sin afectar estado del pedido

**Más información de la tarea:**
Corresponde a GAP-12. Contenido visual del PDF definido en requisito independiente.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte B — Endpoint Previsualizar PDF)
- R16A-RE-FU-019.md (Criterio C1 — Previsualización)

---

### Tarea 16

**Título:** [ R16A-RE-FU-019 ] [IMP-EXIST-SERVICE] Ampliar `AdvanceInvoiceController` con 4 endpoints de Detalle

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** API/Controllers

**Consideraciones previas:**
- El controlador `AdvanceInvoiceController` fue creado en RE-FU-018 (endpoint `/search`)
- Se amplía con 4 endpoints nuevos que delegan a sus servicios correspondientes
- Depende de Tareas 10-15 (repositorios y servicios)
- **Rutas (Reglas al diseñar — regla 9, corrección):** en inglés bajo `api/v1/advanceInvoice/{id}/{subresource}`, no `/api/factura-adelantado/*`

**Objetivo general:**
Ampliar el controlador existente con los 4 endpoints del flujo Detalle FAA.

**Objetivos específicos:**
- Agregar `POST /api/v1/advanceInvoice/{clientId}/detail` → `AdvanceInvoiceDetailService`
- Agregar `POST /api/v1/advanceInvoice/{id}/generate` → `AdvanceInvoiceGenerateService`
- Agregar `POST /api/v1/advanceInvoice/{id}/preview` → `AdvanceInvoicePreviewService`
- Agregar `POST /api/v1/advanceInvoice/{id}/send` → `AdvanceInvoiceSendService`
- Inyectar las 4 interfaces de servicio via constructor
- Manejar respuestas: 200 OK, 400 (validación), 401 (no autenticado), 500 (error interno)

**Resultado esperado:**
Controlador con 5 endpoints totales (search + 4 nuevos) que cubren el flujo completo del módulo FAA.

**Entregables:**
- Ampliación de `Controllers/AdvanceInvoiceController.cs`

**Criterios de aceptación:**
- El endpoint `/generate` retorna `AdvanceInvoiceGenerateResponseDto` (con error SAT si aplica)
- El endpoint `/preview` retorna `FileContentResult` con PDF `byte[]`
- El endpoint `/send` retorna `AdvanceInvoiceSendResponseDto` con TipoPedido para salida operativa
- Todos los endpoints validan autenticación IdentityServer

**Más información de la tarea:**
Corresponde a GAP-14. Los endpoints son consumidos por ProquifaDotNet (Venta Interna).

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte B — API AdvanceInvoiceController)

---

---

## Parte C — ProquifaDotNet.Finanzas (Salida Operativa Post-Envío)

---

### Tarea 17

**Título:** [ R16A-RE-FU-019 ] [SERV-COMPLEX-TRANSACT] Implementar transferencia a Legacy para pedido Crédito post-envío FAA

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
- Crear `Services/AdvanceInvoice/AdvanceInvoiceLegacyService.cs` implementando `IAdvanceInvoiceLegacyService`
- Implementar método `TransferInvoiceToLegacy(Guid idFccFactura)`:

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
- `Services/AdvanceInvoice/AdvanceInvoiceLegacyService.cs`

**Criterios de aceptación:**
- La transferencia envía los 5 elementos a Legacy (Factura, Pedido, Partidas, Cobro, PDF)
- Reutiliza `RestClientLegacy` existente (no crea nuevo canal)
- Si Legacy falla, registra error pero NO revierte el envío de factura
- Solo se ejecuta para pedidos Crédito (SinCredito=0 en catCondicionesDePago)

**Más información de la tarea:**
Corresponde a GAP-18. Patrón de referencia: `ServicioLegacyBO` en `L05.TramitarPedido/Legacy/`.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte C — Transferencia Legacy)
- R16A-RE-FU-019.md (Regla 13 — Salida operativa Crédito)

---

### Tarea 18

**Título:** [ R16A-RE-FU-019 ] [SERV-TRANSACT] Implementar generación de pendiente Validar Cobro para pedido Prepago post-envío FAA

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
- Crear `Services/AdvanceInvoice/AdvanceInvoiceValidateCollectionService.cs` implementando `IAdvanceInvoiceValidateCollectionService`
- Implementar método `GeneratePendingCollectionValidation(Guid idFccFactura)`:

```
1. Obtener datos del pedido y la factura enviada
2. Verificar que no exista pendiente previo para esta proforma (idempotencia)
3. INSERT en tabla de pendientes Validar Cobro:
   vincula IdFccFactura + IdTPPedido + IdCFDIGenerada + MontoTotal
4. Establecer estado inicial del pendiente como activo/pendiente
5. Registrar resultado
```

- Si la generación falla: registrar error pero NO revertir el envío de factura

**Resultado esperado:**
Lógica funcional que genera el pendiente en Validar Cobro para que Cobranza procese el pago contra la factura emitida.

**Entregables:**
- `Services/AdvanceInvoice/AdvanceInvoiceValidateCollectionService.cs`

**Criterios de aceptación:**
- El pendiente se genera solo para pedidos Prepago (SinCredito=1)
- El INSERT vincula correctamente factura + pedido + monto
- Si ya existe un pendiente para la misma proforma adelanto, no se duplica
- El pendiente queda visible en el módulo Validar Cobro para el equipo de Cobranza
- Si la generación falla, registra error pero NO revierte el envío de factura

**Más información de la tarea:**
Corresponde a GAP-19. El módulo Validar Cobro valida el pago del cliente contra esta factura.

**Recursos:**
- R16A-RE-FU-019-Back.md (Parte C — Pendiente Validar Cobro)
- R16A-RE-FU-019.md (Regla 13 — Salida operativa Prepago)
