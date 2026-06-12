# Impacto en Back — R16A-RE-FU-019
**Requisito:** Factura por Adelantado: Detalle México
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10)
**Módulo:** Factura por Adelantado — Detalle (nuevo)
**Impacto:** Detalle por cliente + Generar factura (timbrado SAT) + Enviar factura + Salida operativa (Legacy/Validar Cobro)

---

## Resumen

Este requisito implementa el flujo completo de **Detalle por cliente** del módulo Factura por Adelantado:

1. **Listado de pedidos** del cliente seleccionado con estado contextual (PendienteGenerar / PendienteEnviar)
2. **Generar Factura** — Modal revisión datos fiscales + Previsualización PDF + Timbrado ante PAC SAT
3. **Enviar Factura** — Modal envío correo con PDF+XML adjuntos
4. **Salida operativa** — Crédito: transferencia a Legacy / Prepago: pendiente en Validar Cobro

La funcionalidad se distribuye en tres soluciones:

| Solución | Rol en RE-FU-019 | Estado |
|----------|-----------------|--------|
| ProquifaDotNet.Finanzas (.NET Core 10) | Orquestador principal: detalle, generación, timbrado, envío, salida operativa | Creada en RE-FU-016, módulo FAA agregado en RE-FU-018 |
| ProquifaDotNet.Timbrado (.NET Core 10) | Ejecutor de timbrado: recibe request, genera CFDI via SAP, asigna folio por empresa | Creada en RE-FU-018 |
| ProquifaDotNet (.NET Framework 4.8) | Consumidor: controlador Detalle FAA que delega a Finanzas, transferencia Legacy | Existente |

---

## Parte A — ProquifaDotNet.Timbrado (Ampliación)

### Descripción

Ampliar la solución Timbrado (creada en RE-FU-018) para soportar el manejo de folios por empresa emisora y el flujo completo de timbrado de Factura por Adelantado con datos fiscales del receptor y emisor.

### Nuevos Componentes

#### Domain — Entidad EmpresaFolio

| Campo | Tipo | Descripción |
|-------|------|-------------|
| IdEmpresaFolio | uniqueidentifier | PK |
| EmpresaClave | varchar(10) | GOL, MUN, PRO, PQF — UNIQUE |
| EmpresaNombre | varchar(200) | Razón social legal |
| Serie | varchar(25) | Serie CFDI (nullable) |
| UltimoFolio | int | Consecutivo actual |
| FormatoFolio | varchar(50) | Template del folio (default: '{folio}') |
| LongitudMaxima | int | Max chars del folio (default: 6) |
| CreatedAt | datetime2(7) | Fecha creación |
| UpdatedAt | datetime2(7) | Última actualización |
| IsActive | bit | Activo |

#### Domain — Interface IEmpresaFolioRepository

```csharp
public interface IEmpresaFolioRepository : IGenericRepository<EmpresaFolio>
{
    Task<EmpresaFolio> GetByClaveAsync(string empresaClave);
    Task<int> ConsumeNextFolioAsync(string empresaClave); // UPDATE atómico con UPDLOCK
}
```

#### Application — EmpresaFolioService

| Método | Descripción |
|--------|-------------|
| GetNextFolioAsync(string empresaClave) | Consume folio atómico y retorna folio formateado (varchar 6) |
| GetByClaveAsync(string empresaClave) | Obtener datos de empresa para el CFDI |

#### Application — TimbradoService (Ampliación)

Ampliar `TimbradoService` para soportar el request de Factura por Adelantado:

| Método nuevo | Descripción |
|--------------|-------------|
| TimbrarFacturaAdelantadoAsync(TimbrarFAARequestDto) | Orquesta: validar datos fiscales, consumir folio, armar XML, llamar SAP, persistir CFDI |

#### Application — DTOs nuevos para FAA

| DTO | Campos principales |
|-----|-------------------|
| TimbrarFAARequestDto | IdProformaAdelanto, DatosReceptor (RFC, RazonSocial, CP, RegimenFiscal, UsoCFDI), DatosEmisor (RFC, RazonSocial, RegimenFiscal, EmpresaClave), Conceptos[], MetodoPago="PPD", FormaPago="99", TipoComprobante="I", Moneda, TipoCambio |
| ConceptoFAADto | Cantidad, Descripcion, PrecioUnitario, Importe, ClaveUnidad, ClaveProdServ — **Descripcion = "catálogo + descripción + marca"; NO se incluye lote ni pedimento (OBS-039)** |
| TimbrarFAAResponseDto | IdCFDI, UUID, Serie, Folio, FechaEmision, Total, XmlBase64, Exitoso, ErrorDescripcion |

#### Infrastructure — EmpresaFolioRepository

```csharp
public class EmpresaFolioRepository : GenericRepository<EmpresaFolio>, IEmpresaFolioRepository
{
    public async Task<int> ConsumeNextFolioAsync(string empresaClave)
    {
        // UPDATE con UPDLOCK, ROWLOCK para atomicidad
        // SET UltimoFolio = UltimoFolio + 1, UpdatedAt = SYSUTCDATETIME()
        // WHERE EmpresaClave = @empresaClave AND IsActive = 1
        // OUTPUT INSERTED.UltimoFolio
    }
}
```

#### API — TimbradoController (Ampliación)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | /api/timbrado/timbrar-faa | Recibe request FAA, retorna TimbrarFAAResponseDto |

---

## Parte B — ProquifaDotNet.Finanzas (Módulo FAA — Detalle)

### Descripción

Ampliar el módulo FAA en Finanzas (creado en RE-FU-018) con los endpoints de Detalle por cliente: listado de pedidos, generación de factura (orquestación completa con timbrado), envío de factura, y salida operativa diferenciada.

### Endpoints Nuevos

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | /api/factura-adelantado/detalle | Listado de pedidos del cliente con estado FAA |
| POST | /api/factura-adelantado/generar | Orquesta: arma datos, llama Timbrado, persiste resultado |
| POST | /api/factura-adelantado/previsualizar-pdf | Genera PDF preview sin timbrar (para modal previsualización) |
| POST | /api/factura-adelantado/enviar | Envía correo con PDF+XML, marca Enviada, ejecuta salida operativa |

---

### Endpoint: Detalle (Pedidos por cliente)

#### Request

```json
{
  "IdCliente": "guid",
  "Filters": [
    { "FieldName": "idUsuarioCobrador", "Value": "{IdUsuarioLogueado}" }
  ],
  "SortField": "fechaPedido",
  "SortDirection": "Desc",
  "PageSize": 50,
  "DesiredPage": 1
}
```

#### Response

```json
{
  "Cliente": {
    "IdCliente": "guid",
    "RazonSocial": "string",
    "RFC": "string",
    "MonedaFacturacion": "string",
    "ClasificacionCrediticia": "string"
  },
  "TotalResults": 5,
  "Results": [
    {
      "IdTPProformaAdelanto": "guid",
      "IdTPPedido": "guid",
      "FolioPedidoInterno": "string",
      "FechaPedido": "datetime",
      "CondicionesDePago": "string",
      "EsPrepago": true,
      "EmpresaAlias": "string",
      "EmpresaClave": "string",
      "Subtotal": 15000.00,
      "IVA": 2400.00,
      "MontoTotal": 17400.00,
      "Moneda": "MXN",
      "EstadoFAA": "PendienteGenerar",
      "FolioFactura": null,
      "SerieFactura": null
    }
  ]
}
```

#### Consulta BD (Vista vtpProformaAdelanto)

```
SELECT FROM vtpProformaAdelanto
WHERE IdCliente = @IdCliente
  AND RegionClave = 'MEX'
  AND EstadoFAA IN ('PendienteGenerar', 'PendienteEnviar')
  AND Activo = 1
ORDER BY FechaTramitacion DESC
```

> La vista vtpProformaAdelanto (CREATE VIEW en BD) resuelve los JOINs complejos de la cadena tpProformaAdelanto → fccPagoFacturaAdelanto → tpProformaPedido → tpPedidoProformaPedido → tpPedido.

---

### Endpoint: Generar Factura

#### Request

```json
{
  "IdTPProformaAdelanto": "guid",
  "UsoCFDI": "G03"
}
```

#### Flujo interno

```
1. Validar que IdTPProformaAdelanto existe y EstadoFAA = 'PendienteGenerar'
2. Obtener datos del pedido (tpPedido, partidas)
3. Obtener datos fiscales del cliente (DatosFacturacionCliente)
4. Obtener datos del emisor (Empresa)
5. Obtener Tipo de Cambio del día (si moneda != MXN)
6. Armar TimbrarFAARequestDto con:
   - Receptor: RFC, RazonSocial, CP, RegimenFiscal, UsoCFDI, Moneda, TipoCambio
   - Emisor: RFC, RazonSocial, RegimenFiscal, EmpresaClave
   - Conceptos: partidas del pedido (cantidad, precioUnitario, importe). La descripción de cada concepto CFDI se construye como "catálogo + descripción + marca"; no se incluye lote ni pedimento (OBS-039).
   - Forzados: MetodoPago="PPD", FormaPago="99", TipoComprobante="I"
7. Llamar ProquifaDotNet.Timbrado POST /api/timbrado/timbrar-faa
8. Si EXITOSO:
   a. Persistir IdCFDIGenerada en tpProformaAdelanto (UPDATE SET IdCFDIGenerada = @id)
   b. Almacenar PDF+XML en Minio (bucket 'facturas')
   c. Insertar registros en Archivo (2 registros: PDF + XML)
   d. Retornar éxito con datos de la factura generada
9. Si ERROR:
   a. Retornar error con descripción del PAC SAT
   b. NO modificar estado del pedido
```

#### Response Exitoso

```json
{
  "Exitoso": true,
  "IdCFDIGenerada": "guid",
  "UUID": "string (UUID del CFDI)",
  "Folio": "000123",
  "Serie": "A",
  "FechaEmision": "datetime",
  "Total": 17400.00,
  "PdfUrl": "string (URL Minio)",
  "XmlUrl": "string (URL Minio)"
}
```

#### Response Error

```json
{
  "Exitoso": false,
  "ErrorDescripcion": "El código postal del receptor no coincide con el registrado en el SAT.",
  "ErrorCodigo": "301"
}
```

---

### Endpoint: Previsualizar PDF

#### Request

```json
{
  "IdTPProformaAdelanto": "guid",
  "UsoCFDI": "G03"
}
```

#### Flujo interno

```
1. Armar datos de la factura (misma lógica que Generar, pasos 1-6)
2. Generar PDF via DocumentBuilder (template factura MEX) SIN timbrar
3. Retornar PDF como byte[] (base64 o stream)
```

> Nota: El contenido y estructura del PDF se documenta en requisito independiente. Este endpoint solo genera el preview.

---

### Endpoint: Enviar Factura

#### Request

```json
{
  "IdTPProformaAdelanto": "guid",
  "Destinatario": "cliente@email.com",
  "CC": "esac@proquifa.com",
  "Notas": "Texto adicional opcional"
}
```

#### Flujo interno

```
1. Validar que IdTPProformaAdelanto tiene EstadoFAA = 'PendienteEnviar'
2. Obtener PDF y XML de Minio (bucket 'facturas')
3. Armar asunto con folio factura + folio pedido interno
4. Enviar correo via Brevo con adjuntos PDF+XML
5. Si envío EXITOSO:
   a. INSERT CorreoEnviado + ArchivoCorreoEnviado (PDF, XML)
   b. UPDATE tpProformaAdelanto SET Enviada = 1
   c. Ejecutar salida operativa según tipo de pedido:
      - Crédito: transferir factura a Legacy (Pendientes, Pedido, Partidas, Cobro, PDF)
      - Prepago: generar pendiente en Validar Cobro
   d. Retornar éxito
6. Si envío FALLA:
   a. Retornar error sin modificar estado
```

#### Response

```json
{
  "Exitoso": true,
  "TipoPedido": "Prepago",
  "AccionPostEnvio": "Pendiente generado en Validar Cobro"
}
```

---

### Componentes Nuevos en Finanzas

#### Application — DTOs

| DTO | Propósito |
|-----|-----------|
| FAADetalleRequestDto | Request del detalle (IdCliente, filtros, paginación) |
| FAADetalleResponseDto | Response con datos cliente + lista pedidos |
| FAAClienteCabeceraDto | RazonSocial, RFC, Moneda, Clasificación |
| FAAPedidoDetalleDto | Pedido con estado, montos, empresa, condiciones pago |
| FAAGenerarRequestDto | IdTPProformaAdelanto + UsoCFDI |
| FAAGenerarResponseDto | Resultado del timbrado (éxito/error) |
| FAAEnviarRequestDto | IdTPProformaAdelanto + Destinatario + CC + Notas |
| FAAEnviarResponseDto | Resultado del envío |
| FAADatosFiscalesClienteDto | RFC, RazonSocial, CP, Régimen, Correo, Moneda, TC |
| FAADatosFiscalesEmisorDto | RFC, RazonSocial, Régimen, EmpresaClave |
| FAAPreviewPdfRequestDto | IdTPProformaAdelanto + UsoCFDI |

#### Application — Servicios

| Servicio | Responsabilidad |
|----------|----------------|
| FacturaAdelantadoDetalleService | Obtener pedidos del cliente con estado FAA |
| FacturaAdelantadoGenerarService | Orquestar generación: armar request, llamar Timbrado, persistir |
| FacturaAdelantadoEnviarService | Orquestar envío: correo Brevo, marcar Enviada, salida operativa |
| FacturaAdelantadoPreviewService | Generar PDF preview sin timbrar (via DocumentBuilder) |

#### Application — Interfaces

| Interface | Métodos |
|-----------|---------|
| IFacturaAdelantadoDetalleService | GetPedidosClienteAsync(FAADetalleRequestDto) |
| IFacturaAdelantadoGenerarService | GenerarFacturaAsync(FAAGenerarRequestDto) |
| IFacturaAdelantadoEnviarService | EnviarFacturaAsync(FAAEnviarRequestDto) |
| IFacturaAdelantadoPreviewService | GenerarPreviewPdfAsync(FAAPreviewPdfRequestDto) |

#### Infrastructure — Repositorios

| Repositorio | Consulta |
|-------------|----------|
| FacturaAdelantadoDetalleRepository | SELECT FROM vtpProformaAdelanto WHERE IdCliente + filtros cartera |
| FacturaAdelantadoDatosFiscalesRepository | JOIN DatosFacturacionCliente + Empresa + catUsoCFDI + catRegimenFiscal |
| FacturaAdelantadoPartidasRepository | tpProformaAdelantoPartida o partidas del pedido vinculado |

#### Infrastructure — Integraciones Externas

| Integración | Componente | Descripción |
|-------------|-----------|-------------|
| ProquifaDotNet.Timbrado | ApiCallerTimbrado | POST /api/timbrado/timbrar-faa (HttpClient con Polly) |
| DocumentBuilder | ApiCallerDocumentBuilder | Generar PDF factura desde template HTML |
| Brevo | BrevoEmailService (existente) | Enviar correo con adjuntos PDF+XML |
| Minio | MinioStorageService (existente) | Almacenar/obtener PDF y XML (bucket 'facturas') |

#### Infrastructure — FinanzasContext (Ampliación)

Agregar al contexto EF Core:

| DbSet nuevo | Tabla/Vista | Uso |
|-------------|------------|-----|
| VtpProformaAdelanto | vtpProformaAdelanto (VIEW) | Consultas de detalle |
| Archivo | Archivo | Persistencia PDF+XML |
| CorreoEnviado | CorreoEnviado | Registro de envío |
| ArchivoCorreoEnviado | ArchivoCorreoEnviado | Vinculo archivo-correo |
| CatUsoCFDI | catUsoCFDI | Catálogo SAT |

#### API — FacturaAdelantadoController (Ampliación)

Ampliar el controlador creado en RE-FU-018:

| Método | Endpoint | Servicio |
|--------|----------|----------|
| POST | /api/factura-adelantado/detalle | FacturaAdelantadoDetalleService |
| POST | /api/factura-adelantado/generar | FacturaAdelantadoGenerarService |
| POST | /api/factura-adelantado/previsualizar-pdf | FacturaAdelantadoPreviewService |
| POST | /api/factura-adelantado/enviar | FacturaAdelantadoEnviarService |

---

## Parte C — ProquifaDotNet (Venta Interna)

### Descripción

Ampliar el controlador FAA existente en Venta Interna (creado en RE-FU-018) con endpoints de detalle, generación y envío que delegan a Finanzas. Incluir la lógica de transferencia a Legacy para pedidos Crédito.

### ApiCallerFinanzas — Métodos Nuevos

| Método | Endpoint Finanzas | Descripción |
|--------|------------------|-------------|
| ObtenerDetalleFAAAsync | POST /api/factura-adelantado/detalle | Pedidos del cliente |
| GenerarFacturaFAAAsync | POST /api/factura-adelantado/generar | Genera y timbra |
| PrevisualizarPdfFAAAsync | POST /api/factura-adelantado/previsualizar-pdf | Preview PDF |
| EnviarFacturaFAAAsync | POST /api/factura-adelantado/enviar | Envía correo |

### Controlador FAA Detalle (Ampliación)

| Método | Ruta | Acción |
|--------|------|--------|
| POST | facturaAdelantado/detalle | Delega a Finanzas, retorna pedidos del cliente |
| POST | facturaAdelantado/generar | Delega a Finanzas, retorna resultado timbrado |
| POST | facturaAdelantado/previsualizar-pdf | Delega a Finanzas, retorna PDF stream |
| POST | facturaAdelantado/enviar | Delega a Finanzas, recibe resultado, ejecuta Legacy si Crédito |

### Transferencia Legacy (Pedido Crédito)

Cuando el tipo de pedido es Crédito y la factura se envió exitosamente, se ejecuta transferencia a Legacy:

| Dato transferido | Tabla Legacy | Descripción |
|-----------------|-------------|-------------|
| Factura | Factura (Legacy) | Datos del CFDI: UUID, Folio, Serie, Total |
| Pedido | Pedidos (Legacy) | FolioPedidoInterno, IdCliente, Total |
| Partidas | Partidas (Legacy) | Conceptos facturados |
| Cobro | Cobro (Legacy) | Registro de cobro pendiente |
| PDF | Archivo (Legacy) | PDF de la factura |

> La transferencia a Legacy reutiliza el patrón existente en `ServicioLegacyBO` / `RestClientLegacy` del proyecto L05.TramitarPedido.

### Pendiente Validar Cobro (Pedido Prepago)

Cuando el tipo de pedido es Prepago y la factura se envió exitosamente:

| Acción | Tabla | Descripción |
|--------|-------|-------------|
| INSERT | fccPendienteValidarCobro (o tabla equivalente) | Genera pendiente para que Cobranza valide pago contra factura |

> La lógica de generación del pendiente puede ejecutarse desde Finanzas directamente (INSERT en BD ProquifaDotNet) o delegarse al controlador de Venta Interna que conoce Legacy.

---

## Parte D — Scripts de Base de Datos

### ProquifaDotNet

| # | Script | Descripción |
|---|--------|-------------|
| 1 | ALTER TABLE tpProformaAdelanto ADD Enviada bit NOT NULL DEFAULT(0) | Flag de envío exitoso |
| 2 | CREATE VIEW dbo.vtpProformaAdelanto | Vista con JOINs resueltos y EstadoFAA calculado |

### ProquifaDotNetTimbrado

| # | Script | Descripción |
|---|--------|-------------|
| 3 | CREATE TABLE dbo.EmpresaFolio | Contador de folios por empresa emisora |
| 4 | INSERT EmpresaFolio (4 empresas MEX) | GOL, MUN, PRO, PQF con UltimoFolio=0 |

### Orden de ejecución

```
1. ALTER TABLE tpProformaAdelanto ADD Enviada (ProquifaDotNet)
2. CREATE VIEW vtpProformaAdelanto (ProquifaDotNet) — requiere paso 1
3. CREATE TABLE EmpresaFolio (ProquifaDotNetTimbrado)
4. INSERT EmpresaFolio datos iniciales (ProquifaDotNetTimbrado) — requiere paso 3
```

---

## Gaps de Desarrollo

### En ProquifaDotNet.Timbrado (Ampliación)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Domain: Entidad EmpresaFolio + Interface IEmpresaFolioRepository | Crear entidad, mapeo, interface con consumo atómico | Medio |
| GAP-02 | Application: EmpresaFolioService + TimbradoService ampliación | Consumo folio + nuevo método TimbrarFacturaAdelantadoAsync | Alto |
| GAP-03 | Infrastructure: EmpresaFolioRepository con UPDATE atómico (UPDLOCK) | Implementar consumo seguro de folio con raw SQL | Medio |
| GAP-04 | Application: DTOs TimbrarFAARequestDto, ConceptoFAADto, TimbrarFAAResponseDto | Modelos de request/response para FAA | Bajo |
| GAP-05 | API: TimbradoController endpoint POST /api/timbrado/timbrar-faa | Nuevo endpoint específico para FAA | Bajo |
| GAP-06 | Scripts DDL: CREATE TABLE EmpresaFolio + DML INSERT 4 empresas | Scripts BD ProquifaDotNetTimbrado | Bajo |

### En ProquifaDotNet.Finanzas (Módulo FAA — Detalle)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-07 | Application: DTOs del Detalle FAA (11 DTOs: Request, Response, Cabecera, Pedido, DatosFiscales, etc.) | Modelos completos para todos los modales | Medio |
| GAP-08 | Infrastructure: FacturaAdelantadoDetalleRepository (consulta vista vtpProformaAdelanto) | Query paginada por cliente con filtros cartera | Medio |
| GAP-09 | Infrastructure: FacturaAdelantadoDatosFiscalesRepository (DatosFacturacionCliente + Empresa + catálogos SAT) | Datos fiscales para modal generación | Medio |
| GAP-10 | Application: FacturaAdelantadoGenerarService (orquestación completa: datos fiscales, armar request, llamar Timbrado, persistir CFDI+archivos) | Servicio de alta complejidad, manejo errores PAC | Alto |
| GAP-11 | Application: FacturaAdelantadoEnviarService (correo Brevo + marcar Enviada + salida operativa) | Servicio con lógica diferenciada Crédito/Prepago | Alto |
| GAP-12 | Application: FacturaAdelantadoPreviewService (generar PDF sin timbrar via DocumentBuilder) | Generar preview para modal previsualización | Medio |
| GAP-13 | Infrastructure: ApiCallerTimbrado (HttpClient + Polly para POST /api/timbrado/timbrar-faa) | Cliente HTTP con retry policy hacia Timbrado | Medio |
| GAP-14 | API: FacturaAdelantadoController ampliación (4 endpoints: detalle, generar, previsualizar-pdf, enviar) | Controlador con validaciones y respuestas | Medio |
| GAP-15 | Infrastructure: FinanzasContext ampliación (DbSets vista, Archivo, CorreoEnviado, catUsoCFDI) | Mapeo EF Core de vista + tablas auxiliares | Medio |

### En ProquifaDotNet (Venta Interna)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-16 | ApiCallerFinanzas: 4 métodos nuevos (detalle, generar, previsualizar, enviar) | Llamadas HTTP a Finanzas | Bajo |
| GAP-17 | Controlador FAA Detalle: 4 endpoints que delegan a Finanzas | WebAPI 2 controller con rutas REST | Medio |
| GAP-18 | Lógica transferencia Legacy para pedido Crédito post-envío | Reutilizar patrón ServicioLegacyBO/RestClientLegacy | Alto |
| GAP-19 | Lógica generación pendiente Validar Cobro para pedido Prepago post-envío | INSERT en tabla de pendientes de cobro | Medio |

### En Base de Datos

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-20 | ALTER TABLE tpProformaAdelanto ADD Enviada bit NOT NULL DEFAULT(0) | ProquifaDotNet | Bajo |
| GAP-21 | CREATE VIEW dbo.vtpProformaAdelanto (cadena JOINs + EstadoFAA calculado) | ProquifaDotNet | Medio |

---

## Diagrama de Flujo — Generar Factura

```
[Venta Interna]                   [Finanzas]                         [Timbrado]              [SAP PAC]
     |                                 |                                  |                      |
     | POST facturaAdelantado/generar  |                                  |                      |
     |-------------------------------->|                                  |                      |
     |                                 | 1. Validar EstadoFAA             |                      |
     |                                 | 2. Obtener datos fiscales        |                      |
     |                                 | 3. Obtener partidas pedido       |                      |
     |                                 | 4. Armar TimbrarFAARequestDto    |                      |
     |                                 |                                  |                      |
     |                                 | POST /api/timbrado/timbrar-faa   |                      |
     |                                 |--------------------------------->|                      |
     |                                 |                                  | 5. Consumir folio    |
     |                                 |                                  | 6. Armar XML CFDI    |
     |                                 |                                  | 7. Llamar SAP        |
     |                                 |                                  |--------------------->|
     |                                 |                                  |    CFDI timbrado      |
     |                                 |                                  |<---------------------|
     |                                 |                                  | 8. Persistir CFDI    |
     |                                 |                                  | 9. Log timbrado      |
     |                                 |    TimbrarFAAResponseDto         |                      |
     |                                 |<---------------------------------|                      |
     |                                 | 10. UPDATE tpProformaAdelanto    |                      |
     |                                 | 11. Guardar PDF+XML Minio        |                      |
     |                                 | 12. INSERT Archivo x2            |                      |
     |   FAAGenerarResponseDto         |                                  |                      |
     |<--------------------------------|                                  |                      |
```

---

## Diagrama de Flujo — Enviar Factura

```
[Venta Interna]                   [Finanzas]                      [Brevo]        [Legacy]
     |                                 |                              |               |
     | POST facturaAdelantado/enviar   |                              |               |
     |-------------------------------->|                              |               |
     |                                 | 1. Validar PendienteEnviar   |               |
     |                                 | 2. Obtener PDF+XML Minio     |               |
     |                                 | 3. Armar correo              |               |
     |                                 |                              |               |
     |                                 | Enviar correo con adjuntos   |               |
     |                                 |----------------------------->|               |
     |                                 |    OK                        |               |
     |                                 |<-----------------------------|               |
     |                                 | 4. INSERT CorreoEnviado      |               |
     |                                 | 5. UPDATE Enviada=1          |               |
     |   FAAEnviarResponseDto          |                              |               |
     |<--------------------------------|                              |               |
     |                                 |                              |               |
     | [Si Crédito]                    |                              |               |
     | Transferir a Legacy             |                              |               |
     |---------------------------------------------------------------->------------->|
     |                                 |                              |               |
     | [Si Prepago]                    |                              |               |
     | INSERT pendiente ValidarCobro   |                              |               |
     |                                 |                              |               |
```

---

## Tablas Consultadas/Modificadas

### ProquifaDotNet (Lectura)

| Tabla | Uso |
|-------|-----|
| vtpProformaAdelanto (VIEW) | Listado detalle por cliente con EstadoFAA |
| tpPedido | Datos del pedido (folio, fecha, condiciones pago) |
| DatosFacturacionCliente | RFC, RazonSocial, CP, RegimenFiscal del cliente |
| Empresa | RFC, RazonSocial, Régimen del emisor |
| catUsoCFDI | Catálogo SAT de usos CFDI |
| catRegimenFiscal | Catálogo SAT de regímenes |
| catCondicionesDePago | Texto condiciones pago |
| ClienteCarteraCliente / ClienteCartera | Validación cartera del usuario |
| tpProformaAdelantoPartida (o partidas pedido) | Conceptos de la factura |
| catMoneda | Moneda y tipo de cambio |
| Contacto | Nombre, correo, teléfono del contacto |

### ProquifaDotNet (Escritura)

| Tabla | Operación |
|-------|-----------|
| tpProformaAdelanto | UPDATE SET IdCFDIGenerada, UPDATE SET Enviada=1 |
| Archivo | INSERT x2 (PDF + XML) |
| CorreoEnviado | INSERT registro de envío |
| ArchivoCorreoEnviado | INSERT vínculo archivo-correo |

### ProquifaDotNetTimbrado (Lectura/Escritura)

| Tabla | Operación |
|-------|-----------|
| EmpresaFolio | UPDATE atómico UltimoFolio+1 (consumo folio) |
| CFDI | INSERT (al crear) + UPDATE (al timbrar exitoso: UUID, XML, Estatus) |
| TimbradoLog | INSERT (registro de cada intento) |

---

## Consideraciones Técnicas

| Tema | Detalle |
|------|---------|
| Atomicidad del folio | UPDATE con UPDLOCK, ROWLOCK para evitar folios duplicados en concurrencia |
| Inmutabilidad post-timbrado | Una vez timbrado exitosamente, el CFDI y la factura son inmutables |
| Idempotencia de envío | Si el correo falla, el estado permanece PendienteEnviar; se puede reintentar |
| Manejo de errores PAC | Retornar descripción del error SAT sin modificar estado del pedido |
| Timeout SAP | Polly con timeout + retry; si excede: error al usuario (no encola en este flujo síncrono) |
| PDF preview vs final | El preview no incluye sello/UUID; el PDF final se regenera post-timbrado con datos fiscales completos |
| Descripción concepto CFDI | Formato: "catálogo + descripción + marca". **No se incluye lote ni pedimento en ningún concepto de la FAA** (OBS-039). La Regla 15 ya cierra este punto. |
| Legacy transfer | Usa RestClientLegacy existente; requiere mapeo de datos factura a formato Legacy |
| Validar Cobro pendiente | INSERT directo en tabla de pendientes; el módulo Validar Cobro lo consume |

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| R16A-RE-FU-018 | Prerequisito: ProquifaDotNet.Timbrado creada + módulo FAA listado inicial |
| R16A-RE-FU-016 | Prerequisito: ProquifaDotNet.Finanzas (solución base) |
| R16A-RE-FU-012 | Genera pendiente FAA desde tramitación Crédito con FAA |
| R16A-RE-FU-015 | Genera pendiente FAA desde tramitación Prepago con FAA |
| R16A-RE-FU-005 | Brecha timbrado SUNAT Perú (bloquea FAA para Perú) |
| DocumentBuilder | Generación de PDF de factura (template pendiente de requisito independiente) |

---

## Resumen de Gaps

| Repositorio | Cantidad | Detalle |
|-------------|----------|---------|
| ProquifaDotNet.Timbrado | 6 (GAP-01 a GAP-06) | EmpresaFolio + endpoint timbrar-faa |
| ProquifaDotNet.Finanzas | 9 (GAP-07 a GAP-15) | Detalle + Generar + Enviar + Preview + ApiCallerTimbrado |
| ProquifaDotNet | 4 (GAP-16 a GAP-19) | Consumidor + Legacy + Validar Cobro |
| Base de Datos | 2 (GAP-20 a GAP-21) | ALTER + VIEW en ProquifaDotNet |
| **Total** | **21 gaps** | |
