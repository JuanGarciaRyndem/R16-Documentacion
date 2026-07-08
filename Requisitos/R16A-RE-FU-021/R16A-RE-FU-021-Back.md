# Impacto en Back — R16A-RE-FU-021
**Requisito:** Diseño y generación de Documentos: Factura México
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder
**Módulo:** Factura — PDF CFDI 4.0 México
**Impacto:** Scripts BD ProquifaDotNet (ALTER x3 + CREATE x1) + Plantillas DocumentBuilder x4 (GOL/MUN/PRO/PQF\_MEX\_FAC) + MexicoInvoicePdfMappingService + PersistMexicoInvoicePdfService

---

## Resumen

Este requisito implementa el **diseño y generación del PDF de la Factura CFDI 4.0 para México**, integrando todos los datos fiscales del `TimbreFiscalDigital` del PAC (UUID, sellos, cadena original, QR), el branding por empresa emisora (GOL/MUN/PRO/PQF) y la persistencia inmutable del artefacto en Minio.

> **Nota de arquitectura (corrección — el CFDI no va en Timbrado, va en Finanzas):** versiones previas de este documento asumían dos tablas separadas: `CFDI` (en `ProquifaDotNetTimbrado`, propiedad de Timbrado) y `CFDIGenerada` (en `ProquifaDotNet`, propiedad de Finanzas). Esa tabla `CFDI` **ya no existe** — Timbrado se redujo a un servicio técnico sin tabla de negocio propia (ver `R16A-RE-FU-018-Back.md` Parte B). Todo lo que antes vivía en `CFDI` (UUID, FechaEmision, FechaCertificacionSat, IdArchivoPdf/IdArchivoXml, Estado) ahora vive en **`CFDIGenerada`** (extendida en `R16A-RE-FU-018_BD.md` Parte 3). Este documento se corrige para referenciar únicamente `CFDIGenerada`, y agrega dos columnas propias de este requisito (`IdArchivoPdf`, `FechaCertificacionSat`) vía ALTER TABLE (ver Parte C). También se corrigen los nombres de servicio ya traducidos en RE-FU-019 (`FacturaAdelantadoGenerarService` → `AdvanceInvoiceGenerateService`, `FacturaAdelantadoPreviewService` → `AdvanceInvoicePreviewService`, `TimbrarFAAAsync` → `StampAdvanceInvoiceAsync`) y la ruta técnica de Timbrado (`POST /api/v1/cfdi` → `POST /api/v1/stamp/invoice`).

Es el complemento definitivo de los placeholders dejados en RE-FU-019:

| Placeholder RE-FU-019 | Implementación definitiva en RE-FU-021 |
|-----------------------|----------------------------------------|
| T13 — pasos 10-11: almacenar PDF+XML en Minio, INSERT Archivo x2 — "PDF real en requisito independiente" | T10 — `PersistMexicoInvoicePdfService` |
| T15 — preview PDF con "template placeholder — definir en requisito independiente" | T9 — `MexicoInvoicePdfMappingService` + T5-T8 templates DocumentBuilder |

### Infraestructura reutilizada de RE-FU-018/019

| Componente                                 | Origen                                   | Reutilización                                                                                            |
| ------------------------------------------ | ---------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `StampingService.StampAdvanceInvoiceAsync` | RE-FU-018/019                            | Sin cambios — ya genera el XML timbrado (vía Timbrado) y Finanzas obtiene `TimbreFiscalDigital`          |
| `AdvanceInvoiceGenerateService`            | RE-FU-019                                | Extender con llamada a `PersistMexicoInvoicePdfService` (pasos 10-11 placeholder)                      |
| `AdvanceInvoicePreviewService`             | RE-FU-019                                | Extender con template real `GOL/MUN/PRO/PQF_MEX_FAC` (reemplaza placeholder)                             |
| `ApiCallerStamping`                        | RE-FU-018/019                            | Sin cambios — ya llama a `POST /api/v1/stamp/invoice` (servicio técnico de Timbrado)                             |
| Tabla `CFDIGenerada`                       | RE-FU-019 (base) + RE-FU-018 (extensión) | Se agregan aquí `IdArchivoPdf` y `FechaCertificacionSat` (Parte C); `IdArchivoXml` ya existe para el XML |
| Tabla `Archivo`                            | RE-FU-018/019                            | Sin cambios — patrón `INSERT Archivo` con `FileBucket='facturas'`                                        |
| Minio bucket `facturas`                    | RE-FU-018/019                            | Sin cambios — mismo bucket para PDF México                                                               |
| DocumentBuilder                            | RE-FU-019                                | Agregar 4 templates nuevos `GOL/MUN/PRO/PQF_MEX_FAC`                                                     |

### Brechas conocidas

> ⚠️ **Datos SAT por producto no poblados**
> Los campos `ClaveProdServSAT` (Producto) y `ClaveSAT` (catUnidad) se agregan mediante ALTER TABLE. Los registros existentes quedarán en NULL hasta que el equipo de operaciones capture los valores por producto/unidad. El PDF puede generarse con campo vacío en fase inicial de desarrollo, pero el CFDI 4.0 rechazará el XML ante el PAC si la clave no está presente al momento del timbrado.

> ⚠️ **Campo `Exportacion` en CFDIGenerada**
> El campo se agrega como NULL y se actualiza a `'01'` para registros existentes. Nuevas facturas deben forzar `'01'` en el servicio hasta que exista interfaz para capturarlo.

---

## Parte A — DocumentBuilder (4 Plantillas Factura México)

### Descripción

Crear los 4 templates HTML (Header, Body, Footer) y los registros en `DocumentTemplate` para la Factura CFDI 4.0 de cada empresa emisora México, siguiendo el patrón `{Prefix}_{Region}_{Tipo}` de DocumentBuilder.

### Templates por empresa emisora

| Empresa | TemplateKey | Header | Body | Footer | CFDI de referencia |
|---------|-------------|--------|------|--------|--------------------|
| Golocaer S.A. de C.V. | `GOL_MEX_FAC` | `GOL_MEX_FAC_H.html` | `GOL_MEX_FAC_B.html` | `GOL_MEX_FAC_F.html` | Folio 7156 |
| Mungen S.A. de C.V. | `MUN_MEX_FAC` | `MUN_MEX_FAC_H.html` | `MUN_MEX_FAC_B.html` | `MUN_MEX_FAC_F.html` | Folio 2374 |
| Proquifa S.A. de C.V. | `PRO_MEX_FAC` | `PRO_MEX_FAC_H.html` | `PRO_MEX_FAC_B.html` | `PRO_MEX_FAC_F.html` | Folio 20913 |
| Proveedora Quimico Farmaceutica S.A. de C.V. | `PQF_MEX_FAC` | `PQF_MEX_FAC_H.html` | `PQF_MEX_FAC_B.html` | `PQF_MEX_FAC_F.html` | Folio 143103 |

### Estructura de cada template

#### Header (`*_MEX_FAC_H.html`)
- Logo empresa emisora (branding específico por empresa: logo, colores, certificaciones)
- Datos del emisor: RFC, Razón Social, Dirección, Lugar de Expedición (CP)
- Datos del receptor: RFC, Razón Social, CP fiscal, Régimen Fiscal, Uso CFDI

#### Body (`*_MEX_FAC_B.html`)
- **Datos del CFDI:** Serie, Folio, Versión 4.0, UUID, Fecha de Emisión, Fecha de Certificación, Método de Pago (PPD), Forma de Pago (99), Tipo de Comprobante (I - Ingreso), Moneda, Tipo de Cambio, Exportación (01 - No aplica), Condiciones de Pago
- **Tabla de partidas:** ClaveProdServSAT, Descripción (catálogo + marca + lote + caducidad), Pedimento (si aplica), Nº ID interno, Cantidad, Unidad de Medida, ClaveSAT unidad (c_ClaveUnidad), Valor Unitario, Importe, IVA 16% por línea
- **Totales:** Subtotal, IVA trasladado, Total, Total en letras
- **Datos bancarios:** Banco, Cuenta, CLABE, Moneda, Referencia del cliente

#### Footer (`*_MEX_FAC_F.html`)
- Elementos técnicos SAT: QR, Nº serie certificados emisor y SAT, sellos digitales emisor/SAT, cadena original
- Disclaimer: "Representación impresa de un CFDI 4.0"
- Certificaciones: ISO 9001:2015, NEEC (varían por empresa)
- Logos institucionales: EDQM, FEUM, USP, etc. (varían por empresa)
- Paginación automática "X de Y"

### Scripts SQL — INSERT DocumentTemplate

Patrón MERGE para cada template (4 scripts independientes en `Scripts/`):

```sql
-- GOL_MEX_FAC — Ejecutar en DocumentBuilder
MERGE INTO [dbo].[DocumentTemplate] AS Target
USING (VALUES (
    'GOL_MEX_FAC',
    'Factura CFDI 4.0 - Golocaer S.A. de C.V.',
    1
)) AS Source ([TemplateKey], [Description], [Activo])
ON Target.[TemplateKey] = Source.[TemplateKey]
WHEN NOT MATCHED THEN
    INSERT ([TemplateKey], [Description], [Activo])
    VALUES (Source.[TemplateKey], Source.[Description], Source.[Activo]);
GO
```

> Mismo patrón para `MUN_MEX_FAC` (Mungen S.A. de C.V.), `PRO_MEX_FAC` (Proquifa S.A. de C.V.) y `PQF_MEX_FAC` (Proveedora Quimico Farmaceutica S.A. de C.V.).

---

## Parte B — ProquifaDotNet.Finanzas (Mapeo + Persistencia PDF)

### Descripción

Implementar los dos nuevos servicios en Finanzas que completan el flujo de generación del PDF de la Factura CFDI 4.0 México: el servicio de mapeo que consolida todos los datos en un modelo unificado, y el servicio de persistencia que almacena el PDF en Minio tras el timbrado exitoso del PAC.

### Nuevos Componentes

#### Domain — InvoicePdfModel

Modelo unificado que representa todas las secciones del PDF de la Factura CFDI 4.0:

```csharp
public class InvoicePdfModel
{
    // Sección A — Branding
    public string EmpresaClave   { get; set; }   // GOL / MUN / PRO / PQF
    public string LogoBase64     { get; set; }

    // Sección B — Datos del Emisor
    public string RFCEmisor            { get; set; }
    public string RazonSocialEmisor    { get; set; }
    public string RegimenFiscal        { get; set; }   // ej: "601 - Ley PM"
    public string LugarExpedicion      { get; set; }   // CP
    public string DireccionEmisor      { get; set; }

    // Sección C — Datos del Receptor
    public string RFCReceptor              { get; set; }
    public string RazonSocialReceptor      { get; set; }
    public string CodigoPostalReceptor     { get; set; }
    public string RegimenFiscalReceptor    { get; set; }
    public string UsoCFDI                  { get; set; }

    // Sección D — Datos del CFDI
    public string   Serie               { get; set; }
    public string   Folio               { get; set; }
    public string   Version             { get; set; }   // "4.0"
    public string   UUID                { get; set; }
    public DateTime FechaEmision        { get; set; }
    public DateTime FechaCertificacion  { get; set; }
    public string   MetodoPago          { get; set; }   // "PPD"
    public string   FormaPago           { get; set; }   // "99"
    public string   TipoComprobante     { get; set; }   // "I - Ingreso"
    public string   Moneda              { get; set; }
    public decimal? TipoCambio          { get; set; }
    public string   Exportacion         { get; set; }   // "01 - No aplica"
    public string   CondicionesPago     { get; set; }
    public string   FolioPedidoInterno  { get; set; }   // PI del sistema PQF2 (criterio G3)

    // Sección E — Partidas
    public List<InvoicePdfLineItemModel> Partidas { get; set; }

    // Sección F — Totales
    public decimal Subtotal       { get; set; }
    public decimal IVA            { get; set; }
    public decimal Total          { get; set; }
    public string  TotalEnLetras  { get; set; }

    // Sección G — Datos Bancarios (dos cuentas: MXN + USD — criterio D1)
    public List<FacturaPdfCuentaBancariaModel> CuentasBancarias { get; set; }
    public string ReferenciaCliente { get; set; }   // Referencia del pedido del cliente (criterio D3)

    // Sección F.1 — Retenciones (vacío si no hay retenciones — criterio F1)
    public List<FacturaPdfRetencionModel> Retenciones { get; set; }

    // Sección H — Timbre Fiscal Digital (null en preview; completo en PDF definitivo)
    public string SelloDigitalEmisor    { get; set; }
    public string SelloDigitalSAT       { get; set; }
    public string CadenaOriginal        { get; set; }
    public string NoCertificadoEmisor   { get; set; }
    public string NoCertificadoSAT      { get; set; }
    public string QRBase64              { get; set; }   // Generado dinámicamente
}

public class InvoicePdfLineItemModel
{
    public string  ClaveProdServSAT { get; set; }   // c_ClaveProdServ SAT (campo nuevo)
    public string  Descripcion      { get; set; }   // Catálogo + Marca + Lote + Caducidad
    public string  Pedimento        { get; set; }   // Si aplica
    public string  NumeroId         { get; set; }   // Nº ID interno
    public decimal Cantidad         { get; set; }
    public string  UnidadMedida     { get; set; }
    public string  ClaveSATUnidad   { get; set; }   // c_ClaveUnidad SAT (campo nuevo)
    public decimal ValorUnitario    { get; set; }
    public decimal Importe          { get; set; }
    public List<FacturaPdfImpuestoPartidaModel> Impuestos { get; set; }  // Base, Impuesto, TipoFactor, TasaCuota, Importe (criterio E1)
}

public class FacturaPdfCuentaBancariaModel
{
    public string Banco             { get; set; }
    public string NumeroCuenta      { get; set; }
    public string CLABE             { get; set; }
    public string Moneda            { get; set; }   // MXN / USD
    public string Sucursal          { get; set; }
    public string ReferenciaCliente { get; set; }
}

public class FacturaPdfImpuestoPartidaModel
{
    public string  Base        { get; set; }   // Importe base del impuesto
    public string  Impuesto    { get; set; }   // Clave SAT (ej: "002" = IVA)
    public string  TipoFactor  { get; set; }   // "Tasa"
    public string  TasaCuota   { get; set; }   // "0.160000"
    public decimal Importe     { get; set; }
}

public class FacturaPdfRetencionModel
{
    public string  Impuesto   { get; set; }   // Clave SAT
    public decimal Importe    { get; set; }
}
```

#### Application — MexicoInvoicePdfMappingService

Servicio que consolida todos los datos del CFDI 4.0 en un `InvoicePdfModel`, listo para ser consumido por DocumentBuilder.

**Fuentes de datos por sección:**

| Sección | Tabla / Fuente |
|---------|---------------|
| Branding / Logo | `Empresa` por `EmpresaClave` |
| Emisor | `CFDIGenerada` (RFC, RazonSocial, RegimenFiscal, LugarExpedicion) |
| Receptor | `CFDIGenerada` (RFC, RazonSocial, CP, RegimenFiscalReceptor, UsoCFDI) |
| CFDI — metadata | `CFDIGenerada` (UUID, FechaEmision, FechaCertificacionSat, Serie, Folio, MetodoPago, FormaPago, Moneda, TipoCambio, Exportacion, CondicionesPago) — tabla única, propiedad de Finanzas |
| Partidas — ClaveProdServSAT | `Producto.ClaveProdServSAT` (campo nuevo Tarea 2) |
| Partidas — ClaveSAT unidad | `catUnidad.ClaveSAT` (campo nuevo Tarea 3) |
| Partidas — cantidad / PU | `tpPartidaPedido` (NumeroDePiezas, PrecioUnitario) |
| Totales | `CFDIGenerada` (Subtotal, Total) + cálculo IVA por partidas |
| Datos Bancarios (dos cuentas MXN+USD, con Sucursal) | `EmpresaDatosBancarios` por `EmpresaClave` |
| Folio Pedido Interno PI | `tpPedido` o equivalente en ProquifaDotNet |
| Retenciones | `CFDIGenerada` / XML timbrado — sección si aplica |
| Timbre Fiscal Digital | XML del PAC — parseo directo (no BD) |
| QR | Generado dinámicamente: UUID + RFC emisor + RFC receptor + Total (formato SAT) |

**Interfaz propuesta:**

```csharp
public interface IMexicoInvoicePdfMappingService
{
    // PDF definitivo — incluye TimbreFiscalDigital del PAC
    Task<InvoicePdfModel> MapearAsync(Guid idCFDI, string xmlTimbradoPac);

    // Preview — sin TimbreFiscalDigital (propiedades Sección H en null)
    Task<InvoicePdfModel> MapearPreviewAsync(Guid idCFDIGenerada);
}
```

**Flujo interno — `MapearAsync`:**

```
1. Consultar CFDIGenerada → UUID, FechaEmision, FechaCertificacionSat, Serie, Folio,
   RFC/RS emisor y receptor, RegimenFiscal, LugarExpedicion,
   UsoCFDI, FormaPago, MetodoPago, TipoDocumento, Moneda, TipoCambio,
   Subtotal, Total, Exportacion, CondicionesPago
2. Consultar tpPartidaPedido JOIN Producto JOIN catUnidad
   → ClaveProdServSAT, ClaveSAT, NumeroDePiezas, PrecioUnitario, Descripcion, Pedimento
3. Consultar EmpresaDatosBancarios por EmpresaClave
4. Consultar Empresa por EmpresaClave → LogoBase64
5. Parsear xmlTimbradoPac → extraer TimbreFiscalDigital
   (UUID, SelloEmisor, SelloSAT, CadenaOriginal, NoCertEmisor, NoCertSAT)
6. Calcular IVA por partida (16% sobre Importe)
7. Calcular Total en Letras
8. Generar QR en base64 con cadena SAT:
   https://verificacfdi.facturaelectronica.sat.gob.mx/default.aspx?id={UUID}&re={RFCEmisor}&rr={RFCReceptor}&tt={Total}&fe={UltimosSelloEmisor8}
9. Ensamblar y retornar InvoicePdfModel
```

**Consulta BD — Partidas con datos SAT** (llamada API a ProquifaDotNet):

```sql
-- Ejecutar en ProquifaDotNet
SELECT
    p.ClaveProdServSAT,
    p.Descripcion,
    pp.Pedimento,
    pp.NumeroId,
    pp.NumeroDePiezas                             AS Cantidad,
    pp.PrecioUnitario                             AS ValorUnitario,
    (pp.NumeroDePiezas * pp.PrecioUnitario)       AS Importe,
    u.Clave                                       AS UnidadMedida,
    u.ClaveSAT                                    AS ClaveSATUnidad,
    -- Desglose impuesto por partida (criterio E1):
    (pp.NumeroDePiezas * pp.PrecioUnitario)       AS BaseImpuesto,
    '002'                                          AS ClaveImpuesto,  -- IVA SAT
    'Tasa'                                         AS TipoFactor,
    '0.160000'                                     AS TasaCuota,
    (pp.NumeroDePiezas * pp.PrecioUnitario * 0.16) AS ImporteIVA
FROM tpPartidaPedido pp
INNER JOIN Producto   p ON pp.IdProducto = p.IdProducto
INNER JOIN catUnidad  u ON p.IdUnidad    = u.IdUnidad
WHERE pp.IdPedido = @IdPedido
ORDER BY pp.NumeroPartida
```

#### Application — PersistMexicoInvoicePdfService

Servicio transaccional que, tras el timbrado exitoso del PAC, genera el PDF definitivo y lo persiste en Minio.

**TemplateKey por empresa emisora:**

| EmpresaClave | TemplateKey DocumentBuilder |
|-------------|------------------------------|
| `GOL` | `GOL_MEX_FAC` |
| `MUN` | `MUN_MEX_FAC` |
| `PRO` | `PRO_MEX_FAC` |
| `PQF` | `PQF_MEX_FAC` |

**Flujo interno:**

```
1. Invocar MexicoInvoicePdfMappingService.MapearAsync(IdCFDI, xmlTimbrado)
   → InvoicePdfModel con TimbreFiscalDigital completo
2. Resolver TemplateKey según EmpresaClave del modelo
3. Invocar DocumentBuilder → generar PDF en bytes
4. Subir PDF a Minio (bucket 'facturas')
5. INSERT Archivo (NombreArchivo, FileBucket='facturas', IdRegion='MEX',
   ContentType='application/pdf') → obtener IdArchivo
6. UPDATE CFDIGenerada SET IdArchivoPdf = @IdArchivo (misma tabla que el XML, columna distinta)
7. Si pasos 4-6 fallan: reintentar sin re-timbrar (sin llamar al PAC nuevamente)
8. Registrar en Serilog: módulo, IdCFDIGenerada, fecha, resultado (éxito/error + mensaje)
9. Registrar el guardado de la factura en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo — Reglas al diseñar, regla 8)
```

**Integración con RE-FU-019 — T13 (`AdvanceInvoiceGenerateService`) pasos 10-11:**

```csharp
// Extender el flujo de AdvanceInvoiceGenerateService con la implementación real:
// Paso 10 (reemplaza placeholder): persistir PDF definitivo
await _persistirFacturaMexicoPdfService.PersistirAsync(idCFDIGenerada, response.XmlTimbrado);
// Paso 11 (sin cambios): INSERT Archivo XML — patrón existente
```

**Integración con RE-FU-019 — T15 (`AdvanceInvoicePreviewService`) template:**

```csharp
// Reemplazar template placeholder por template real de DocumentBuilder:
var model      = await _mappingService.MapearPreviewAsync(idCFDIGenerada);
var templateKey = model.EmpresaClave switch
{
    "GOL" => "GOL_MEX_FAC",
    "MUN" => "MUN_MEX_FAC",
    "PRO" => "PRO_MEX_FAC",
    "PQF" => "PQF_MEX_FAC",
    _     => throw new InvalidOperationException($"EmpresaClave no reconocida: {model.EmpresaClave}")
};
var pdfBytes = await _documentBuilder.GenerarAsync(templateKey, model);
```

---

## Parte C — Scripts de Base de Datos (ProquifaDotNet)

### Orden de ejecución de scripts

| Paso | Script                                                             | Tipo | BD             | Dependencia | Estado                                                                               |
| ---- | ------------------------------------------------------------------ | ---- | -------------- | ----------- | ------------------------------------------------------------------------------------ |
| 1    | `CREATE TABLE catClaveProdServSAT`                                 | DDL  | ProquifaDotNet | Ninguna     | **BLOQUEANTE para paso 2**                                                           |
| 2    | `INSERT catClaveProdServSAT` catálogo c_ClaveProdServ SAT          | DML  | ProquifaDotNet | Paso 1      | Requiere catálogo SAT                                                                |
| 3    | `ALTER TABLE Producto ADD ClaveProdServSAT varchar(10)`            | DDL  | ProquifaDotNet | Paso 1      | **BLOQUEANTE para partidas PDF**                                                     |
| 4    | `ALTER TABLE Producto ADD IdCatClaveProdServSAT FK`                | DDL  | ProquifaDotNet | Paso 1      | **BLOQUEANTE para partidas PDF**                                                     |
| 5    | `ALTER TABLE catUnidad ADD ClaveSAT varchar(10)`                   | DDL  | ProquifaDotNet | Ninguna     | Independiente                                                                        |
| 6    | `UPDATE catUnidad SET ClaveSAT` mapeo c_ClaveUnidad SAT            | DML  | ProquifaDotNet | Paso 5      | Requiere mapeo manual                                                                |
| 7    | `ALTER TABLE CFDIGenerada ADD Exportacion varchar(2)`              | DDL  | ProquifaDotNet | Ninguna     | Independiente                                                                        |
| 8    | `UPDATE CFDIGenerada SET Exportacion = '01'`                       | DML  | ProquifaDotNet | Paso 7      | Registros existentes                                                                 |
| 9    | `ALTER TABLE CFDIGenerada ADD IdArchivoPdf, FechaCertificacionSat` | DDL  | ProquifaDotNet | Ninguna     | Complementa la extensión de RE-FU-018 (Parte 3) — columnas propias de este requisito |

### Scripts DDL y DML

#### 1. CREATE TABLE catClaveProdServSAT — BLOQUEANTE

```sql
-- Ejecutar en ProquifaDotNet — BLOQUEANTE para Producto
CREATE TABLE [dbo].[catClaveProdServSAT](
    [IdCatClaveProdServSAT] uniqueidentifier NOT NULL
        CONSTRAINT [DF_catClaveProdServSAT_Id] DEFAULT (NEWID()),
    [Clave]       varchar(10)  NOT NULL,
    [Descripcion] varchar(300) NOT NULL,
    [Activo]      bit          NOT NULL
        CONSTRAINT [DF_catClaveProdServSAT_Activo] DEFAULT (1),
    CONSTRAINT [PK_catClaveProdServSAT]       PRIMARY KEY CLUSTERED ([IdCatClaveProdServSAT]),
    CONSTRAINT [UQ_catClaveProdServSAT_Clave] UNIQUE ([Clave])
);
GO
```

#### 2. ALTER TABLE Producto (ClaveProdServSAT + FK) — BLOQUEANTE

```sql
-- Ejecutar en ProquifaDotNet — campo varchar directo
ALTER TABLE dbo.Producto
    ADD ClaveProdServSAT varchar(10) NULL;
GO

-- FK hacia catClaveProdServSAT
ALTER TABLE dbo.Producto
    ADD IdCatClaveProdServSAT uniqueidentifier NULL
        CONSTRAINT [FK_Producto_ClaveProdServSAT]
            FOREIGN KEY REFERENCES dbo.catClaveProdServSAT([IdCatClaveProdServSAT]);
GO
```

#### 3. ALTER TABLE catUnidad (ClaveSAT)

```sql
-- Ejecutar en ProquifaDotNet
-- catUnidad.Clave (varchar 150) NO es la clave SAT del catálogo c_ClaveUnidad
-- Se agrega ClaveSAT independiente para no alterar el comportamiento actual de Clave
ALTER TABLE dbo.catUnidad
    ADD ClaveSAT varchar(10) NULL;
GO
-- Ejemplos de mapeo a poblar: KGM - Kilogramo, H87 - Pieza, LTR - Litro, E48 - Unidad de servicio
```

#### 4. ALTER TABLE CFDIGenerada (Exportacion)

```sql
-- Ejecutar en ProquifaDotNet
-- Campo obligatorio CFDI 4.0 — '01' = No aplica para operaciones nacionales México
ALTER TABLE dbo.CFDIGenerada
    ADD Exportacion varchar(2) NULL;
GO

-- Poblar registros existentes con valor por defecto
UPDATE dbo.CFDIGenerada
    SET Exportacion = '01'
WHERE Exportacion IS NULL;
GO
```

#### 5. ALTER TABLE CFDIGenerada (IdArchivoPdf + FechaCertificacionSat)

```sql
-- Ejecutar en ProquifaDotNet
-- IdArchivoPdf: FK -> Archivo, mismo patron que IdArchivoXml (RE-FU-018 Parte 3)
-- FechaCertificacionSat: fecha en que el PAC certifico el CFDI (puede diferir de FechaEmision)
ALTER TABLE dbo.CFDIGenerada
    ADD IdArchivoPdf uniqueidentifier NULL,
        FechaCertificacionSat datetime2(7) NULL;
GO

ALTER TABLE dbo.CFDIGenerada
    ADD CONSTRAINT [FK_CFDIGenerada_ArchivoPdf]
        FOREIGN KEY (IdArchivoPdf) REFERENCES dbo.Archivo([IdArchivo]);
GO
```

---

## Gaps de Desarrollo

### En DocumentBuilder

| # | Gap | Acción | Esfuerzo | Estado |
|---|-----|--------|----------|--------|
| GAP-01 | Template `GOL_MEX_FAC` — `GOL_MEX_FAC_H.html` + `GOL_MEX_FAC_B.html` + `GOL_MEX_FAC_F.html` + INSERT DocumentTemplate | Crear 3 archivos HTML con branding Golocaer + script SQL MERGE | Alto | Abierto |
| GAP-02 | Template `MUN_MEX_FAC` — `MUN_MEX_FAC_H.html` + `MUN_MEX_FAC_B.html` + `MUN_MEX_FAC_F.html` + INSERT DocumentTemplate | Crear 3 archivos HTML con branding Mungen + script SQL MERGE | Alto | Abierto |
| GAP-03 | Template `PRO_MEX_FAC` — `PRO_MEX_FAC_H.html` + `PRO_MEX_FAC_B.html` + `PRO_MEX_FAC_F.html` + INSERT DocumentTemplate | Crear 3 archivos HTML con branding Proquifa + script SQL MERGE | Alto | Abierto |
| GAP-04 | Template `PQF_MEX_FAC` — `PQF_MEX_FAC_H.html` + `PQF_MEX_FAC_B.html` + `PQF_MEX_FAC_F.html` + INSERT DocumentTemplate | Crear 3 archivos HTML con branding Proveedora Quimico Farmaceutica + script SQL MERGE | Alto | Abierto |

### En ProquifaDotNet.Finanzas

| # | Gap | Acción | Esfuerzo | Estado |
|---|-----|--------|----------|--------|
| GAP-05 | `InvoicePdfModel` + `InvoicePdfLineItemModel` (Domain) | Nuevos modelos con todas las secciones A-H del PDF | Bajo | Abierto |
| GAP-06 | `IMexicoInvoicePdfMappingService` + `MexicoInvoicePdfMappingService` (Application) | Consolidar datos de múltiples tablas + parsear TimbreFiscalDigital + generar QR | Alto | Abierto |
| GAP-07 | `PersistMexicoInvoicePdfService` (Application) | Generar PDF → persistir Minio → INSERT Archivo → UPDATE CFDIGenerada.IdArchivoPdf → reintentos sin re-timbrado | Medio | Abierto |
| GAP-08 | Extender `AdvanceInvoiceGenerateService` (RE-FU-019 T13) pasos 10-11 | Reemplazar placeholder con llamada real a `PersistMexicoInvoicePdfService` | Bajo | Abierto |
| GAP-09 | Extender `AdvanceInvoicePreviewService` (RE-FU-019 T15) | Reemplazar template placeholder con resolución dinámica `GOL/MUN/PRO/PQF_MEX_FAC` | Bajo | Abierto |
| GAP-10 | Integrar `PersistMexicoInvoicePdfService` en flujo Validar Cobro — paso "Genera Factura Normal PPD" | Invocar `PersistirAsync` tras timbrado exitoso antes de transicionar a "Pendiente en Validar Pago" | Bajo | Abierto |

### En Base de Datos (ProquifaDotNet)

| # | Gap | Acción | BD | Esfuerzo | Estado |
|---|-----|--------|----|----------|--------|
| GAP-11 | `CREATE TABLE catClaveProdServSAT` + INSERT catálogo c_ClaveProdServ SAT | DDL + DML | ProquifaDotNet | Medio | **BLOQUEANTE** |
| GAP-12 | `ALTER TABLE Producto ADD ClaveProdServSAT` + `ALTER TABLE Producto ADD IdCatClaveProdServSAT FK` | DDL | ProquifaDotNet | Bajo | **BLOQUEANTE** |
| GAP-13 | `ALTER TABLE catUnidad ADD ClaveSAT` + UPDATE mapeo c_ClaveUnidad SAT | DDL + DML | ProquifaDotNet | Bajo | Abierto |
| GAP-14 | `ALTER TABLE CFDIGenerada ADD Exportacion` + UPDATE '01' registros existentes | DDL + DML | ProquifaDotNet | Bajo | Abierto |
| GAP-15 | `ALTER TABLE CFDIGenerada ADD IdArchivoPdf, FechaCertificacionSat` | DDL | ProquifaDotNet | Bajo | Abierto |

---

## Diagrama de Flujo — Generación PDF Factura México

```
[AdvanceInvoiceGenerateService]       [ProquifaDotNet (BD)]            [DocumentBuilder]     [Minio]
             |                                     |                          |                  |
  (timbrado exitoso PAC — pasos 10-11 RE-FU-019 T13)                        |                  |
             |                                     |                          |                  |
  1. PersistMexicoInvoicePdfService.PersistirAsync(IdCFDIGenerada, xmlPAC) |                  |
             |                                     |                          |                  |
  2. MexicoInvoicePdfMappingService.MapearAsync    |                          |                  |
             |---GET CFDIGenerada / Partidas / Bancarios --------------------->|                  |
             |<---datos fiscales ----------------------------------------------                  |
             | 3. Parsear TimbreFiscalDigital del XML PAC                    |                  |
             | 4. Calcular IVA por partida (16%)                             |                  |
             | 5. Generar QR base64 (cadena SAT)                             |                  |
             | 6. Resolver TemplateKey (GOL/MUN/PRO/PQF_MEX_FAC)            |                  |
             |---POST /generar-pdf (templateKey + InvoicePdfModel) -------> |                  |
             |<---PDF bytes -------------------------------------------------                  |
             |---PUT bucket 'facturas' / {IdCFDIGenerada}.pdf -------------------------------->|
             |<---referencia Minio -------------------------------------------------------------|
             | 7. INSERT Archivo (FileBucket='facturas', IdRegion='MEX')    |                  |
             | 8. UPDATE CFDIGenerada SET IdArchivoPdf = @IdArchivo (directo en BD, vía EF Core — no via PUT /api/v1/cfdi/{id}/pdf, ya que CfdiService/CFDIGenerada son propiedad de este mismo aplicativo, Finanzas) |
             | 9. Serilog: módulo, IdCFDIGenerada, fecha, resultado          |                  |
```

> **Preview (RE-FU-019 T15):** `AdvanceInvoicePreviewService` llama a `MapearPreviewAsync` (sin TimbreFiscalDigital) → resuelve TemplateKey → DocumentBuilder genera PDF en memoria → retorna bytes sin persistir en Minio.
