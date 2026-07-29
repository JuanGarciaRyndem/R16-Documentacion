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
| ProquifaDotNet.Finanzas (.NET Core 10) | Orquestador principal: detalle, generación (incluye `CfdiController`/`CFDIGenerada`), envío, salida operativa | Creada en RE-FU-016, módulo FAA y CfdiController agregados en RE-FU-018 |
| ProquifaDotNet.Timbrado (.NET Core 10) | Servicio técnico: consume folio, arma XML, llama SAP, **regresa el resultado a Finanzas sin persistir el CFDI como entidad de negocio** | Creada en RE-FU-018 |
| ProquifaDotNet (.NET Framework 4.8) | Consumidor: controlador Detalle FAA que delega a Finanzas, transferencia Legacy | Existente |

> **Nota de arquitectura (correccion — el CFDI no va en Timbrado, va en Finanzas):** este documento se corrigio para reflejar que `CfdiController` y la persistencia de `CFDIGenerada` viven en **ProquifaDotNet.Finanzas** (ver R16A-RE-FU-018-Back.md, Parte B), y que este mismo requisito (RE-FU-019) es el que ejecuta el `CREATE TABLE CFDIGenerada` base (ver R16A-RE-FU-019_BD.md). `ProquifaDotNet.Timbrado` expone el endpoint tecnico `POST /api/v1/stamp/invoice` (no `/api/v1/cfdi`, que es el recurso de negocio expuesto por Finanzas).

---

## Parte A — ProquifaDotNet.Timbrado (Ampliación)

### Descripción

Ampliar la solución Timbrado (creada en RE-FU-018) para soportar el manejo de folios por empresa emisora y el flujo completo de timbrado de Factura por Adelantado con datos fiscales del receptor y emisor.

### Nuevos Componentes

#### Domain — Entidad EmpresaFolio

| Campo | Tipo | Descripción |
|-------|------|-------------|
| IdEmpresaFolio | uniqueidentifier | PK |
| IdEmpresa | uniqueidentifier | FK → `Empresa` |
| Serie | varchar(25) | Serie CFDI (nullable): NULL = factura, `'P'` = CDP, `'P2'` = NC |
| UltimoFolio | int | **Consecutivo** — último entero asignado |
| FormatoFolio | varchar(50) | Patrón de formato del folio (default: `'{folio}'`) |
| LongitudMaxima | int | Longitud máxima del folio (default: 6) |
| Activo | bit | Borrado lógico |
| FechaRegistro | datetime | Fecha de alta |
| FechaUltimaActualizacion | datetime | Última actualización del consecutivo |

#### Domain — Interface IEmpresaFolioRepository

```csharp
public interface IEmpresaFolioRepository : IGenericRepository<EmpresaFolio>
{
    Task<EmpresaFolio> GetByEmpresaAsync(Guid idEmpresa, string serie);
    Task<int> ConsumeNextFolioAsync(Guid idEmpresa, string serie); // UPDATE con UPDLOCK — retorna consecutivo incrementado
}
```

#### Application — EmpresaFolioService

| Método | Descripción |
|--------|-------------|
| `GetNextFolioAsync(Guid idEmpresa, string serie)` | `UPDATE EmpresaFolio SET UltimoFolio = UltimoFolio + 1 OUTPUT inserted.UltimoFolio` con UPDLOCK atómico — retorna consecutivo formateado como folio (varchar) |
| `GetByEmpresaAsync(Guid idEmpresa, string serie)` | Obtener registro del foliador para auditoría |

#### Application — StampingService (Ampliación)

Ampliar `StampingService` para soportar el request de Factura por Adelantado:

| Método nuevo | Descripción |
|--------------|-------------|
| StampAdvanceInvoiceAsync(StampAdvanceInvoiceRequestDto) | Orquesta: validar datos fiscales, consumir folio, armar XML, llamar SAP, registrar StampingLog y **regresar el resultado a Finanzas** — no persiste el CFDI como entidad de negocio propia |

#### Application — DTOs nuevos para FAA

| DTO | Campos principales |
|-----|-------------------|
| StampAdvanceInvoiceRequestDto | IdProformaAdelanto, RecipientData (RFC, RazonSocial, CP, RegimenFiscal, UsoCFDI), IssuerData (RFC, RazonSocial, RegimenFiscal, EmpresaClave), Conceptos[], IdCatMetodoDePagoCFDI (FK PPD), IdCatFormaPagoSAT (FK 99), TipoComprobante="I", Moneda, TipoCambio |
| AdvanceInvoiceItemDto | Cantidad, Descripcion, PrecioUnitario, Importe, ClaveUnidad, ClaveProdServ — **Descripcion = "catálogo + descripción + marca"; NO se incluye lote ni pedimento (OBS-039)** |
| StampAdvanceInvoiceResponseDto | Uuid, Serie, Folio, FechaEmision, Total, XmlBase64, Exitoso, ErrorDescripcion — **sin IdCFDI**: el Id de negocio real (`IdCFDIGenerada`) lo asigna Finanzas al persistir, no Timbrado |

#### Infrastructure — EmpresaFolioRepository

```csharp
public class EmpresaFolioRepository : GenericRepository<EmpresaFolio>, IEmpresaFolioRepository
{
    public async Task<int> ConsumeNextFolioAsync(Guid idEmpresa, string serie)
    {
        // UPDATE atómico con UPDLOCK — sin dependencia de tabla legacy
        // UPDATE EmpresaFolio
        // SET    UltimoFolio = UltimoFolio + 1,
        //        FechaUltimaActualizacion = GETDATE()
        // OUTPUT inserted.UltimoFolio
        // WHERE  IdEmpresa = @idEmpresa
        //   AND  (Serie = @serie OR (Serie IS NULL AND @serie IS NULL))
    }
}
```

#### API — StampingController (Ampliación)

| Método | Endpoint       | Descripción                                                                                                                     |
| ------ | -------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| POST   | /api/v1/stamp/invoice  | Recibe el request técnico de timbrado FAA armado por Finanzas, retorna StampAdvanceInvoiceResponseDto — reutiliza el endpoint técnico de facturas (`/api/v1/stamp/invoice`) creado en RE-FU-018 (`StampingController`), sin controller ni ruta separados. El recurso de negocio `cfdi` (`CfdiController`, `POST /api/v1/cfdi`) vive en Finanzas, no aquí. |

---

## Parte B — ProquifaDotNet.Finanzas (Módulo FAA — Detalle)

### Descripción

Ampliar el módulo FAA en Finanzas (creado en RE-FU-018) con los endpoints de Detalle por cliente: listado de pedidos, generación de factura (orquestación completa con timbrado), envío de factura, y salida operativa diferenciada.

### Endpoints Nuevos

> **Nota (Reglas al diseñar — regla 9, corrección):** las rutas se corrigen de `/api/factura-adelantado/*` (español, sin versionar) a `api/v1/{resource}/{id}/{subresource}` — recurso singular en inglés `advanceInvoice` (ya establecido en RE-FU-018 para `POST /api/v1/advanceInvoice/search`), con subrecursos en inglés.

| Método | Endpoint                                  | Descripción                                                       |
| ------ | ----------------------------------------- | ----------------------------------------------------------------- |
| POST   | /api/v1/advanceInvoice/{clientId}/detail  | Listado de pedidos del cliente con estado FAA                     |
| POST   | /api/v1/advanceInvoice/{id}/generate      | Orquesta: arma datos, llama CfdiService (Timbrado), persiste resultado |
| POST   | /api/v1/advanceInvoice/{id}/preview       | Genera PDF preview sin timbrar (para modal previsualización)      |
| POST   | /api/v1/advanceInvoice/{id}/send          | Envía correo con PDF+XML, marca Enviada, ejecuta salida operativa |

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
      "IdFccFactura": "guid",
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

#### Consulta BD (Vista vfccFactura — RE-FU-015)

> **Migración (06/07/2026):** este listado ya no consulta `vtpProformaAdelanto`. Consulta directamente **`vfccFactura`** (vista definida y creada en R16A-RE-FU-015_BD.md), que cubre tanto pedidos Prepago como Crédito sin la cadena de JOINs original.

```
SELECT FROM vfccFactura
WHERE IdCliente = @IdCliente
  AND RegionClave = 'MEX'
  AND EstadoFAA IN ('PendienteGenerar', 'PendienteEnviar')
  AND Activo = 1
ORDER BY FechaTramitacion DESC
```

> `vfccFactura` lee `fccFactura` directamente por FK (`IdTPPedido`), sin necesidad de la cadena `tpProformaAdelanto → fccPagoFacturaAdelanto → tpProformaPedido → tpPedidoProformaPedido → tpPedido` que usaba la vista anterior (`vtpProformaAdelanto`, retirada — ver H-01 en R16A-RE-FU-018_DIS-SOL_Revision.md).

---

### Endpoint: Generar Factura

#### Request

```json
{
  "IdFccFactura": "guid",
  "UsoCFDI": "G03"
}
```

#### Flujo interno

```
1. Validar que IdFccFactura existe y EstadoFAA = 'PendienteGenerar' (vfccFactura)
2. Obtener datos del pedido (tpPedido, partidas)
3. Obtener datos fiscales del cliente (DatosFacturacionCliente)
4. Obtener datos del emisor (Empresa)
5. Obtener Tipo de Cambio del día (si moneda != MXN)
6. Armar StampAdvanceInvoiceRequestDto con:
   - Receptor: RFC, RazonSocial, CP, RegimenFiscal, UsoCFDI, Moneda, TipoCambio
   - Emisor: RFC, RazonSocial, RegimenFiscal, EmpresaClave
   - Conceptos: partidas del pedido (cantidad, precioUnitario, importe). La descripción de cada concepto CFDI se construye como "catálogo + descripción + marca"; no se incluye lote ni pedimento (OBS-039).
   - Forzados: IdCatMetodoDePagoCFDI=<IdPPD>, IdCatFormaPagoSAT=<Id99>, TipoComprobante="I"
7. Llamar ProquifaDotNet.Timbrado POST /api/v1/stamp/invoice (servicio técnico, sin persistir CFDI)
8. Si EXITOSO (Timbrado regresa Uuid, Serie, Folio, FechaEmision, Total, XmlBase64):
   a. INSERT CFDIGenerada (CfdiService, en ProquifaDotNet): UUID, Serie, Folio, FechaEmision, Total,
      IdCatTipoCFDI, IdCatUsoCFDI, IdCatMetodoDePagoCFDI, IdCatMoneda, TipoCambio, Estado='Timbrado'
   b. Persistir IdCFDIGenerada en fccFactura (UPDATE SET IdCFDIGenerada = @IdCFDIGenerada, EsFacturaPorAdelantado = 0 — el Id real del registro insertado en el paso anterior, no un Id de Timbrado)
   c. Almacenar PDF+XML en Minio (bucket 'facturas')
   d. Insertar registros en Archivo (2 registros: PDF + XML) + UPDATE CFDIGenerada SET IdArchivoXml
   e. Registrar el guardado de la factura en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo — regla 8)
   f. Retornar éxito con datos de la factura generada
9. Si ERROR:
   a. Si ya existía un registro Pendiente en CFDIGenerada, UPDATE Estado='Fallido', MensajeError
   b. Retornar error con descripción del PAC SAT
   c. NO modificar estado del pedido
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
  "IdFccFactura": "guid",
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
  "IdFccFactura": "guid",
  "Destinatario": "cliente@email.com",
  "CC": "esac@proquifa.com",
  "Notas": "Texto adicional opcional"
}
```

#### Flujo interno

```
1. Validar que IdFccFactura tiene EstadoFAA = 'PendienteEnviar' (vfccFactura)
2. Obtener PDF y XML de Minio (bucket 'facturas')
3. Armar asunto con folio factura + folio pedido interno
4. Enviar correo via Brevo con adjuntos PDF+XML
5. Si envío EXITOSO:
   a. INSERT CorreoEnviado + ArchivoCorreoEnviado (PDF, XML)
   b. UPDATE fccFactura SET Enviada = 1, FechaEnvio = GETDATE(), IdCatFacturaEstado = ENVIADA
      (catFacturaEstado y FechaEnvio, RE-FU-015 v2.1; antes: UPDATE tpProformaAdelanto SET Enviada = 1)
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

#### Application — Catálogos Fiscales SAT (nuevos — Guía Técnica)

> **⏸ Pendiente** — Los catálogos fiscales SAT (`catImpuestoSat`, `catTipoFactorSat`, `catObjetoImpuestoSat`, `PerfilFiscal`) y toda la lógica de resolución de `ClaveProdServ`, `ClaveUnidad` y `PerfilFiscal` por producto/familia quedan en espera. No implementar interfaces, servicios ni DbSets de esta sección hasta confirmar el nivel de configuración (GAP-7 / GAP-8 en RE-019_BD.md).

Los catálogos `catImpuestoSat`, `catTipoFactorSat`, `catObjetoImpuestoSat` y la tabla `PerfilFiscal` se agregan en este requisito. Finanzas los consume al construir el XML del CFDI (nodo `Conceptos/Impuestos`).

**Interfaces nuevas:**

| Interface | Métodos |
|-----------|---------|
| IPerfilFiscalRepository | `GetByIdAsync(Guid id)`, `GetAllActiveAsync()` |
| ICatImpuestoSatRepository | `GetByClave(string clave)`, `GetAllActiveAsync()` |
| ICatTipoFactorSatRepository | `GetByClave(string clave)`, `GetAllActiveAsync()` |
| ICatObjetoImpuestoSatRepository | `GetByClave(string clave)`, `GetAllActiveAsync()` |

**Servicio nuevo:**

| Servicio | Método | Descripción |
|----------|--------|-------------|
| PerfilFiscalService | `GetPerfilFiscalAsync(Guid idPerfilFiscal)` | Devuelve la configuración fiscal completa (impuesto + factor + tasa + objeto) para construir el nodo `Traslado` del CFDI |
| PerfilFiscalService | `GetAllActiveAsync()` | Listado para configuración de productos |

**DbSets a agregar al FinanzasContext:**

| DbSet | Tabla | Uso |
|-------|-------|-----|
| `CatImpuestoSat` | `catImpuestoSat` | Catálogo c_Impuesto SAT |
| `CatTipoFactorSat` | `catTipoFactorSat` | Catálogo c_TipoFactor SAT |
| `CatObjetoImpuestoSat` | `catObjetoImpuestoSat` | Catálogo c_ObjetoImp SAT |
| `PerfilFiscal` | `PerfilFiscal` | Perfil fiscal por producto/concepto |

**Uso en la generación del CFDI:**

Al armar `StampAdvanceInvoiceRequestDto`, por cada partida del pedido se resuelven los 3 campos fiscales **antes** de llegar a la pantalla de generación:

```
1. Resolver PerfilFiscal del producto:
   a. Si Producto.IdPerfilFiscal IS NOT NULL → usar ese (override específico)
   b. Si NO → usar FamiliaProducto.IdPerfilFiscal (herencia de Familia)
   c. Si ninguno → error de configuración (producto sin perfil fiscal asignado)

2. Resolver ClaveProdServ:
   a. Si Producto.ClaveProdServSAT IS NOT NULL → usar ese
   b. Si NO → usar FamiliaProducto.ClaveProdServSAT

3. Resolver ClaveUnidad: misma precedencia que ClaveProdServ
```

Incluir en `AdvanceInvoiceItemDto`: `ClaveProdServ`, `ClaveUnidad`, y los campos del perfil fiscal (`TasaOCuota`, `ClaveTipoFactor`, `ClaveImpuesto`, `ClaveObjetoImpuesto`) resueltos antes de enviar el request a Timbrado.

> **GAP-8 (pendiente bloqueante — ver RE-019_BD.md):** Confirmar nivel de configuración (Producto / Familia / Producto→Familia) antes de crear la FK en BD. No asumir que los 3 campos comparten el mismo nivel.

---

#### Application — DTOs

| DTO | Propósito |
|-----|-----------|
| AdvanceInvoiceDetailRequestDto | Request del detalle (IdCliente, filtros, paginación) |
| AdvanceInvoiceDetailResponseDto | Response con datos cliente + lista pedidos |
| ClientHeaderDto | RazonSocial, RFC, Moneda, Clasificación |
| OrderDetailDto | Pedido con estado, montos, empresa, condiciones pago |
| AdvanceInvoiceGenerateRequestDto | IdFccFactura + UsoCFDI |
| AdvanceInvoiceGenerateResponseDto | Resultado del timbrado (éxito/error) |
| AdvanceInvoiceSendRequestDto | IdFccFactura + Destinatario + CC + Notas |
| AdvanceInvoiceSendResponseDto | Resultado del envío |
| ClientFiscalDataDto | RFC, RazonSocial, CP, Régimen, Correo, Moneda, TC |
| IssuerFiscalDataDto | RFC, RazonSocial, Régimen, EmpresaClave |
| AdvanceInvoicePreviewPdfRequestDto | IdFccFactura + UsoCFDI |

#### Application — Servicios

| Servicio | Responsabilidad |
|----------|----------------|
| AdvanceInvoiceDetailService | Obtener pedidos del cliente con estado FAA |
| AdvanceInvoiceGenerateService | Orquestar generación: armar request, llamar Timbrado, persistir |
| AdvanceInvoiceSendService | Orquestar envío: correo Brevo, marcar Enviada, salida operativa |
| AdvanceInvoicePreviewService | Generar PDF preview sin timbrar (via DocumentBuilder) |

#### Application — Interfaces

| Interface | Métodos |
|-----------|---------|
| IAdvanceInvoiceDetailService | GetClientOrdersAsync(AdvanceInvoiceDetailRequestDto) |
| IAdvanceInvoiceGenerateService | GenerateInvoiceAsync(AdvanceInvoiceGenerateRequestDto) |
| IAdvanceInvoiceSendService | SendInvoiceAsync(AdvanceInvoiceSendRequestDto) |
| IAdvanceInvoicePreviewService | GeneratePreviewPdfAsync(AdvanceInvoicePreviewPdfRequestDto) |

#### Infrastructure — Repositorios

| Repositorio | Consulta |
|-------------|----------|
| AdvanceInvoiceDetailRepository | SELECT FROM vfccFactura WHERE IdCliente + filtros cartera (antes: vtpProformaAdelanto — ver RE-FU-015) |
| AdvanceInvoiceFiscalDataRepository | JOIN DatosFacturacionCliente + Empresa + catUsoCFDI + catRegimenFiscal |
| AdvanceInvoiceItemsRepository | fccFacturaPartida (antes: tpProformaAdelantoPartida) o partidas del pedido vinculado |

#### Infrastructure — Integraciones Externas

| Integración | Componente | Descripción |
|-------------|-----------|-------------|
| ProquifaDotNet.Timbrado | ApiCallerStamping (existente, RE-FU-018) | POST /api/v1/stamp/invoice — servicio técnico (HttpClient con Polly), Finanzas persiste CFDIGenerada tras la respuesta |
| DocumentBuilder | ApiCallerDocumentBuilder | Generar PDF factura desde template HTML |
| ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo) | ApiCallerMail (existente, RE-FU-016) | Enviar correo con adjuntos PDF+XML — regla 7, sin cliente Brevo propio |
| Minio | MinioStorageService (existente, RE-FU-018) | Almacenar/obtener PDF y XML (bucket 'facturas') |

#### Infrastructure — FinanzasContext (Ampliación)

Agregar al contexto EF Core:

| DbSet nuevo | Tabla/Vista | Uso |
|-------------|------------|-----|
| VfccFactura | vfccFactura (VIEW, `.ToView().HasNoKey()` — RE-FU-015; reemplaza VtpProformaAdelanto/vtpProformaAdelanto) | Consultas de detalle |
| Archivo | Archivo | Persistencia PDF+XML |
| CorreoEnviado | CorreoEnviado | Registro de envío |
| ArchivoCorreoEnviado | ArchivoCorreoEnviado | Vinculo archivo-correo |
| CatUsoCFDI | catUsoCFDI | Catálogo SAT |

#### API — AdvanceInvoiceController (Ampliación)

Ampliar el controlador creado en RE-FU-018:

| Método | Endpoint | Servicio |
|--------|----------|----------|
| POST | /api/v1/advanceInvoice/{clientId}/detail | AdvanceInvoiceDetailService |
| POST | /api/v1/advanceInvoice/{id}/generate | AdvanceInvoiceGenerateService |
| POST | /api/v1/advanceInvoice/{id}/preview | AdvanceInvoicePreviewService |
| POST | /api/v1/advanceInvoice/{id}/send | AdvanceInvoiceSendService |

---

## Parte C — ProquifaDotNet (Venta Interna)

### Descripción

Ampliar el controlador FAA existente en Venta Interna (creado en RE-FU-018) con endpoints de detalle, generación y envío que delegan a Finanzas. Incluir la lógica de transferencia a Legacy para pedidos Crédito.

### ApiCallerFinanzas — Métodos Nuevos

| Método | Endpoint Finanzas | Descripción |
|--------|------------------|-------------|
| GetAdvanceInvoiceDetailAsync | POST /api/v1/advanceInvoice/{clientId}/detail | Pedidos del cliente |
| GenerateAdvanceInvoiceAsync | POST /api/v1/advanceInvoice/{id}/generate | Genera y timbra |
| PreviewAdvanceInvoicePdfAsync | POST /api/v1/advanceInvoice/{id}/preview | Preview PDF |
| SendAdvanceInvoiceAsync | POST /api/v1/advanceInvoice/{id}/send | Envía correo |

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

> **Migración (06/07/2026):** `ALTER TABLE tpProformaAdelanto ADD Enviada` y `CREATE VIEW dbo.vtpProformaAdelanto` se retiraron de este requisito. La columna `Enviada` ahora vive en `fccFactura` y la vista `vfccFactura` se crea en **R16A-RE-FU-015_BD.md** (sección "Migración de `tpProformaAdelanto`"), que unifica el pendiente FAA de Prepago y Crédito. Este requisito ya no ejecuta scripts DDL propios sobre ProquifaDotNet.

### ProquifaDotNetTimbrado

| # | Script | Descripción |
|---|--------|-------------|
| 3 | CREATE TABLE dbo.EmpresaFolio | Contador de folios por empresa emisora |
| 4 | INSERT EmpresaFolio (4 empresas MEX) | GOL, MUN, PRO, PQF con UltimoFolio=0 |

### Orden de ejecución

```
1. CREATE TABLE fccFactura + fccFacturaPartida + fccFacturaReferenciaBancaria + CREATE VIEW vfccFactura (R16A-RE-FU-015 — prerequisito, ya no se ejecuta en este requisito)
2. CREATE TABLE EmpresaFolio (ProquifaDotNet — propiedad Finanzas, movida de ProquifaDotNetTimbrado el 07/07/2026)
3. INSERT EmpresaFolio datos iniciales (ProquifaDotNet) — requiere paso 2
```

---

## Gaps de Desarrollo

### En ProquifaDotNet.Timbrado (Ampliación)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Domain: Entidad EmpresaFolio + Interface IEmpresaFolioRepository | Crear entidad, mapeo, interface con consumo atómico | Medio |
| GAP-02 | Application: EmpresaFolioService + StampingService ampliación | Consumo folio + nuevo método StampAdvanceInvoiceAsync | Alto |
| GAP-03 | Infrastructure: EmpresaFolioRepository con UPDATE atómico (UPDLOCK) | Implementar consumo seguro de folio con raw SQL | Medio |
| GAP-04 | Application: DTOs StampAdvanceInvoiceRequestDto, AdvanceInvoiceItemDto, StampAdvanceInvoiceResponseDto | Modelos de request/response para FAA | Bajo |
| GAP-05 | API: StampingController — sin endpoint nuevo (reutiliza POST /api/v1/stamp/invoice creado en RE-FU-018) | Ajuste de orquestación interna para FAA | Bajo |
| GAP-06 | Scripts DDL: CREATE TABLE EmpresaFolio + DML INSERT 4 empresas | Scripts BD ProquifaDotNet (propiedad Finanzas — movida de ProquifaDotNetTimbrado el 07/07/2026) | Bajo |

### En ProquifaDotNet.Finanzas (Módulo FAA — Detalle)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-07 | Application: DTOs del Detalle FAA (11 DTOs: Request, Response, Cabecera, Pedido, FiscalData, etc.) | Modelos completos para todos los modales | Medio |
| GAP-08 | Infrastructure: AdvanceInvoiceDetailRepository (consulta vista vfccFactura — RE-FU-015) | Query paginada por cliente con filtros cartera | Medio |
| GAP-09 | Infrastructure: AdvanceInvoiceFiscalDataRepository (DatosFacturacionCliente + Empresa + catálogos SAT) | Datos fiscales para modal generación | Medio |
| GAP-10 | Application: AdvanceInvoiceGenerateService (orquestación completa: datos fiscales, armar request, invocar CfdiService.GenerateAsync — que a su vez llama Timbrado y persiste CFDIGenerada+Archivo) | Servicio de alta complejidad, manejo errores PAC | Alto |
| GAP-11 | Application: AdvanceInvoiceSendService (correo via ProquifaDotNet.EnvioCorreo + marcar Enviada + salida operativa) | Servicio con lógica diferenciada Crédito/Prepago | Alto |
| GAP-12 | Application: AdvanceInvoicePreviewService (generar PDF sin timbrar via DocumentBuilder) | Generar preview para modal previsualización | Medio |
| GAP-13 | (Ya cubierto en RE-FU-018, Parte B, GAP-10/GAP-11) Reutilizar CfdiService / ApiCallerStamping existentes — no se duplica aquí | Sin esfuerzo adicional, solo consumo | - |
| GAP-14 | API: AdvanceInvoiceController ampliación (4 endpoints: detalle, generar, previsualizar-pdf, enviar) | Controlador con validaciones y respuestas | Medio |
| GAP-15 | Infrastructure: FinanzasContext ampliación (DbSets vista, Archivo, CorreoEnviado, catUsoCFDI) | Mapeo EF Core de vista + tablas auxiliares | Medio |

### En ProquifaDotNet (Venta Interna)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-16 | ApiCallerFinanzas: 4 métodos nuevos (detalle, generar, previsualizar, enviar) | Llamadas HTTP a Finanzas | Bajo |
| GAP-17 | Controlador FAA Detalle: 4 endpoints que delegan a Finanzas | WebAPI 2 controller con rutas REST | Medio |
| GAP-18 | Lógica transferencia Legacy para pedido Crédito post-envío | Reutilizar patrón ServicioLegacyBO/RestClientLegacy | Alto |
| GAP-19 | Lógica generación pendiente Validar Cobro para pedido Prepago post-envío | INSERT en tabla de pendientes de cobro | Medio |

### En Base de Datos

> **Migración (06/07/2026):** GAP-20 y GAP-21 (ALTER TABLE tpProformaAdelanto / CREATE VIEW vtpProformaAdelanto) se eliminan de este requisito — la columna `Enviada` y la vista `vfccFactura` ya están cubiertas por los gaps de BD de **R16A-RE-FU-015**. Este requisito no aporta gaps propios de Base de Datos.

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
     |                                 | 4. Armar StampAdvanceInvoiceRequestDto    |                      |
     |                                 |                                  |                      |
     |                                 | POST /api/v1/stamp/invoice (tecnico)     |                      |
     |                                 |--------------------------------->|                      |
     |                                 |                                  | 5. Consumir folio    |
     |                                 |                                  | 6. Armar XML CFDI    |
     |                                 |                                  | 7. Llamar SAP        |
     |                                 |                                  |--------------------->|
     |                                 |                                  |    CFDI timbrado      |
     |                                 |                                  |<---------------------|
     |                                 |                                  | 8. Log StampingLog   |
     |                                 |    StampAdvanceInvoiceResponseDto         |                      |
     |                                 |<---------------------------------|                      |
     |                                 | 9. INSERT CFDIGenerada (Finanzas)|                      |
     |                                 | 10. UPDATE fccFactura            |                      |
     |                                 | 11. Guardar PDF+XML Minio        |                      |
     |                                 | 12. INSERT Archivo x2 + UPDATE CFDIGenerada.IdArchivoXml |
     |   AdvanceInvoiceGenerateResponseDto         |                                  |                      |
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
     |   AdvanceInvoiceSendResponseDto          |                              |               |
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
| vfccFactura (VIEW — RE-FU-015, reemplaza vtpProformaAdelanto) | Listado detalle por cliente con EstadoFAA |
| tpPedido | Datos del pedido (folio, fecha, condiciones pago) |
| DatosFacturacionCliente | RFC, RazonSocial, CP, RegimenFiscal del cliente |
| Empresa | RFC, RazonSocial, Régimen del emisor |
| catUsoCFDI | Catálogo SAT de usos CFDI |
| catRegimenFiscal | Catálogo SAT de regímenes |
| catCondicionesDePago | Texto condiciones pago |
| ClienteCarteraCliente / ClienteCartera | Validación cartera del usuario |
| fccFacturaPartida (reemplaza tpProformaAdelantoPartida, o partidas pedido) | Conceptos de la factura |
| catMoneda | Moneda y tipo de cambio |
| Contacto | Nombre, correo, teléfono del contacto |

### ProquifaDotNet (Escritura)

| Tabla | Operación |
|-------|-----------|
| CFDIGenerada | INSERT al timbrar exitosamente (UUID, Serie, Folio, FechaEmision, Total, catalogos, Estado) — propiedad de Finanzas |
| fccFactura (reemplaza tpProformaAdelanto) | UPDATE SET IdCFDIGenerada (Id real de CFDIGenerada), EsFacturaPorAdelantado=0; UPDATE SET Enviada=1 |
| Archivo | INSERT x2 (PDF + XML) + vínculo en CFDIGenerada.IdArchivoXml |
| CorreoEnviado | INSERT registro de envío |
| ArchivoCorreoEnviado | INSERT vínculo archivo-correo |

### ProquifaDotNetTimbrado (Lectura/Escritura — solo técnico, sin tabla de negocio del CFDI)

| Tabla | Operación |
|-------|-----------|
| EmpresaFolio | UPDATE atómico UltimoFolio+1 (consumo folio) |
| StampingLog | INSERT (auditoría técnica de cada intento; `CfdiGeneradaId` es referencia informativa, no FK) |

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
| R16A-RE-FU-018 | Prerequisito: ProquifaDotNet.Timbrado (servicio técnico) + módulo FAA listado inicial + CfdiController/CfdiService en Finanzas (Parte B) |
| R16A-RE-FU-016 | Prerequisito: ProquifaDotNet.Finanzas (solución base) |
| R16A-RE-FU-012 | Genera pendiente FAA (`fccFactura`, origen Crédito) desde tramitación Crédito con FAA |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria`/`vfccFactura` — genera pendiente FAA de Prepago |
| R16A-RE-FU-005 | Brecha timbrado SUNAT Perú (bloquea FAA para Perú) |
| DocumentBuilder | Generación de PDF de factura (template pendiente de requisito independiente) |

---

## Resumen de Gaps

| Repositorio | Cantidad | Detalle |
|-------------|----------|---------|
| ProquifaDotNet.Timbrado | 6 (GAP-01 a GAP-06) | EmpresaFolio + endpoint timbrar-faa |
| ProquifaDotNet.Finanzas | 9 (GAP-07 a GAP-15) | Detalle + Generar + Enviar + Preview + ApiCallerStamping |
| ProquifaDotNet | 4 (GAP-16 a GAP-19) | Consumidor + Legacy + Validar Cobro |
| Base de Datos | 0 (GAP-20/GAP-21 migrados a RE-FU-015) | Sin scripts DDL propios — reutiliza `fccFactura`/`vfccFactura` |
| **Total** | **19 gaps** | |
