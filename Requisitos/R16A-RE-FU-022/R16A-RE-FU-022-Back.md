# Impacto en Back — R16A-RE-FU-022
**Requisito:** Diseño y generación de Documentos: Factura Perú
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder
**Módulo:** Factura — PDF CPE UBL 2.1 Perú
**Impacto:** Scripts BD ProquifaDotNet (CREATE CPEGenerada + ALTER tpProformaAdelanto) + Plantilla DocumentBuilder x1 (GOLPERU\_PER\_FAC) + FacturaPeruPdfMappingService + PersistirFacturaPeruPdfService

---

## Resumen

Este requisito implementa el **diseño y generación del PDF de la Factura electrónica (CPE tipo 01, UBL 2.1) para Perú**, integrando los elementos técnicos de certificación SUNAT (QR, firma digital, valor resumen) del XML retornado por el OSE/PSE, el branding de Golocaer S.A.C. y la persistencia inmutable del artefacto en Minio. Es el contraparte peruano de R16A-RE-FU-021 (PDF Factura México).

### Diferencias fundamentales vs RE-FU-021 (Factura México)

| Aspecto | México — RE-FU-021 | Perú — RE-FU-022 |
|---------|-------------------|-----------------|
| Estándar fiscal | CFDI 4.0 SAT | **CPE UBL 2.1 SUNAT** |
| Empresa emisora | GOL / MUN / PRO / PQF | **Solo GOLPERU (Golocaer S.A.C.)** |
| Templates DocumentBuilder | 4 (una por empresa) | **1 (GOLPERU_PER_FAC)** |
| ID fiscal | RFC 13 chars | **RUC 11 dígitos** |
| Impuesto | IVA 16% | **IGV 18%** |
| Tabla de datos fiscales | CFDIGenerada | **CPEGenerada (nueva)** |
| Método/Forma de Pago | PPD + 99 (forzados SAT) | **No aplican en Perú** |
| Complemento de Pago | Sí (módulo Validar Cobro) | **No existe en SUNAT** |
| Totales extra | No | **ISC, ICBPER, Anticipos, Descuentos, OtrosTributos, Redondeo** |
| Elementos técnicos | UUID, sellos, cadena original SAT | **QR SUNAT, firma digital, valor resumen** |
| Identificador fiscal del comprobante | Folio Fiscal UUID (SAT) | **No hay UUID — es CPE (Serie + Correlativo)** |
| Serie / Correlativo | varchar(80) / varchar(6) numérico | **4 chars alfanumérica / hasta 8 dígitos** |
| Cuentas bancarias | Sí (dos cuentas MXN+USD, CLABE) | **⚠️ Pendiente — factura real de muestra no las incluye** |

### Infraestructura reutilizada de RE-FU-017/018/019/020

| Componente | Origen | Reutilización |
|------------|--------|---------------|
| `StampingService.TimbrarFacturaSunatAsync` | RE-FU-020 | Sin cambios — ya genera el XML timbrado y retorna CDR SUNAT |
| `ApiCallerStamping.TimbrarSunatAsync` | RE-FU-020 | Sin cambios — ya llama a `POST /api/v1/cfdi` |
| Tabla `Archivo` | RE-FU-017/018 | Sin cambios — patrón `INSERT Archivo` con `FileBucket='facturas'`, `IdRegion='PER'` |
| Minio bucket `facturas` | RE-FU-017/018 | Sin cambios — mismo bucket, `IdRegion='PER'` |
| Tabla `EmpresaFolio` GOLPERU | RE-FU-020 | Sin cambios — serie SUNAT F001/E001 + correlativo 8 dígitos |
| DocumentBuilder | RE-FU-019/021 | Agregar 1 template nuevo `GOLPERU_PER_FAC` |
| `FacturaAdelantadoGenerarService` (branch PER) | RE-FU-020 | Extender con llamada a `PersistirFacturaPeruPdfService` |

### Brechas bloqueantes

> ⛔ **BRECHA BLOQUEANTE — Datos fiscales SUNAT del producto (B1)**
> Los campos `CodigoSUNAT` (Producto), `ClaveSUNAT` (catUnidad) e `IdCatAfectacionIGV` (Producto) son **obligatorios** en el XML UBL 2.1 y en el PDF del CPE. No existen en el catálogo de productos actual. Documentado en R16A-RE-FU-005 Brecha 1. Scripts propuestos en RE-FU-020.

> ⛔ **BRECHA BLOQUEANTE — Integración OSE/PSE SUNAT (B2)**
> El PDF se genera al timbrarse exitosamente ante SUNAT/OSE. Mientras la integración OSE/PSE no esté resuelta (RE-FU-020 brecha mayor), no hay timbrado y por tanto no hay PDF. Documentado en R16A-RE-FU-005 Brecha 5.

> ⚠️ **BRECHA — Datos legales Golocaer S.A.C. Perú (B3)**
> RUC, Ubigeo, dirección fiscal, certificado digital y series SUNAT (F001/E001) de Golocaer S.A.C. **no están confirmados**. Sin estos datos el timbrado no puede ejecutarse.

> ⚠️ **BRECHA — Branding, certificaciones y disclaimer legal SUNAT (B4)**
> Logo, colores, certificaciones aplicables a Perú y texto del disclaimer legal SUNAT de Golocaer S.A.C. no están confirmados. Las certificaciones mexicanas (NEEC, FEUM, ISO 9001 México) **no aplican** a Perú.

> ⚠️ **BRECHA — Cuentas bancarias en PDF (B5)**
> La factura real de muestra de Golocaer S.A.C. (E001-362) **no incluye sección de cuentas bancarias**. Pendiente confirmar con el cliente si aplica a Perú y, de aplicar, su modelo (CCI de 20 dígitos, monedas PEN/USD).

---

## Parte A — DocumentBuilder (Plantilla Factura Perú)

### Descripción

Crear el template HTML (Header, Body, Footer) y el registro en `DocumentTemplate` para la Factura electrónica (CPE UBL 2.1) de Golocaer S.A.C. siguiendo el patrón `{Prefix}_{Region}_{Tipo}` de DocumentBuilder.

### Template

| Empresa | TemplateKey | Header | Body | Footer | CPE de referencia |
|---------|-------------|--------|------|--------|------------------|
| Golocaer S.A.C. | `GOLPERU_PER_FAC` | `GOLPERU_PER_FAC_H.html` | `GOLPERU_PER_FAC_B.html` | `GOLPERU_PER_FAC_F.html` | E001-362 |

### Estructura del template

#### Header (`GOLPERU_PER_FAC_H.html`)
- Logo y branding de Golocaer S.A.C. Perú (**⚠️ pendiente confirmar con cliente**)
- Datos del emisor: RUC, Razón Social ("GOLOCAER S.A.C."), Dirección + Distrito/Provincia/Departamento, Ubigeo, Fecha y Hora de Emisión
- Datos del receptor: RUC, Razón Social, Dirección
- Identificadores del comprobante: Serie-Correlativo (ej: "E001-362"), Tipo de Comprobante ("01 - Factura"), Tipo de Operación (catálogo 51 SUNAT)

#### Body (`GOLPERU_PER_FAC_B.html`)
- **Datos del comprobante:** Condición de Pago (Contado/Crédito), Moneda (nombre completo, ej: "DOLAR AMERICANO"), Tipo de Cambio (si aplica), Folio del Pedido Interno PI
- **Tabla de partidas:** Cantidad, Unidad de Medida SUNAT, Código SUNAT del Producto, Descripción (nombre + orden de compra + catálogo + lote cuando aplique), Valor Unitario, Afectación al IGV por línea, ICBPER por línea (cuando aplique)
- **Totales SUNAT:** Sub Total Ventas, Anticipos, Descuentos, Valor de Venta Operaciones Gratuitas, Valor Venta, ISC, IGV (18%), ICBPER, Otros Cargos, Otros Tributos, Redondeo, Importe Total, Total en Letras
- **Crédito** (cuando CondiciónPago = Crédito): Monto neto pendiente, Total de Cuotas, detalle por cuota (Nº, Fecha vencimiento, Monto) — **⚠️ Pendiente confirmar si aplica a R16 Perú Prepago**

#### Footer (`GOLPERU_PER_FAC_F.html`)
- Elementos técnicos SUNAT: Código QR de validación SUNAT, Valor Resumen (hash), Firma Digital
- Disclaimer: representación impresa de la factura electrónica SUNAT, verificable con clave SOL (**⚠️ texto exacto pendiente asesor legal peruano**)
- Certificaciones aplicables a Golocaer S.A.C. Perú (**⚠️ pendiente — las mexicanas NO aplican**)
- Paginación automática "X de Y"

### Script SQL — INSERT DocumentTemplate

```sql
-- GOLPERU_PER_FAC — Ejecutar en DocumentBuilder
MERGE INTO [dbo].[DocumentTemplate] AS Target
USING (VALUES (
    'GOLPERU_PER_FAC',
    'Factura CPE UBL 2.1 - Golocaer S.A.C. (Peru)',
    1
)) AS Source ([TemplateKey], [Description], [Activo])
ON Target.[TemplateKey] = Source.[TemplateKey]
WHEN NOT MATCHED THEN
    INSERT ([TemplateKey], [Description], [Activo])
    VALUES (Source.[TemplateKey], Source.[Description], Source.[Activo]);
GO
```

---

## Parte B — ProquifaDotNet.Finanzas (Mapeo + Persistencia PDF Perú)

### Descripción

Implementar los dos nuevos servicios en Finanzas para el PDF de la Factura Perú: el servicio de mapeo que consolida los datos del CPE en un modelo unificado, y el servicio de persistencia que almacena el PDF en Minio tras el timbrado exitoso ante SUNAT/OSE.

### Nuevos Componentes

#### Domain — FacturaPeruPdfModel

Modelo unificado que representa todas las secciones del PDF del CPE UBL 2.1:

```csharp
public class FacturaPeruPdfModel
{
    // Sección A — Branding
    public string LogoBase64 { get; set; }   // Golocaer S.A.C. Perú

    // Sección B — Datos del Emisor (Golocaer S.A.C.)
    public string RUCEmisor          { get; set; }
    public string RazonSocialEmisor  { get; set; }
    public string DireccionEmisor    { get; set; }   // Dirección + Distrito/Provincia/Dpto
    public string UbigeoEmisor       { get; set; }   // 6 dígitos SUNAT

    // Sección C — Datos del Receptor
    public string RUCReceptor          { get; set; }
    public string RazonSocialReceptor  { get; set; }
    public string DireccionReceptor    { get; set; }

    // Sección D — Datos del Comprobante
    public string   Serie               { get; set; }   // 4 chars alfanumérica (ej: "E001")
    public string   Correlativo         { get; set; }   // hasta 8 dígitos
    public string   TipoComprobante     { get; set; }   // "01 - Factura"
    public string   TipoOperacion       { get; set; }   // catálogo 51 SUNAT (ej: "0101")
    public DateTime FechaEmision        { get; set; }
    public string   CondicionPago       { get; set; }   // "Contado" / "Crédito"
    public string   Moneda              { get; set; }   // "PEN" / "USD"
    public string   MonedaNombre        { get; set; }   // "DOLAR AMERICANO"
    public decimal? TipoCambio          { get; set; }   // TC SUNAT del día (null si PEN)
    public string   FolioPedidoInterno  { get; set; }   // PI del sistema PQF2 (criterio G3)

    // Sección E — Partidas
    public List<FacturaPeruPdfPartidaModel> Partidas { get; set; }

    // Sección F — Totales SUNAT
    public decimal SubTotalVentas            { get; set; }
    public decimal Anticipos                 { get; set; }
    public decimal Descuentos                { get; set; }
    public decimal ValorVentaGratuitas       { get; set; }
    public decimal ValorVenta                { get; set; }
    public decimal ISC                       { get; set; }
    public decimal IGV                       { get; set; }   // 18%
    public decimal ICBPER                    { get; set; }
    public decimal OtrosCargos               { get; set; }
    public decimal OtrosTributos             { get; set; }
    public decimal Redondeo                  { get; set; }
    public decimal ImporteTotal              { get; set; }
    public string  TotalEnLetras             { get; set; }   // ej: "SON: DIECIOCHO MIL... DOLAR AMERICANO"

    // Sección F.1 — Crédito (null si CondiciónPago = Contado)
    public FacturaPeruPdfCreditoModel Credito { get; set; }

    // Sección G — Elementos Técnicos SUNAT (null en preview; completo en PDF definitivo)
    public string QRBase64      { get; set; }   // QR de validación SUNAT
    public string ValorResumen  { get; set; }   // Hash del comprobante
    public string FirmaDigital  { get; set; }   // Firma digital del OSE/PSE
}

public class FacturaPeruPdfPartidaModel
{
    public decimal Cantidad           { get; set; }
    public string  UnidadMedidaSUNAT  { get; set; }   // catálogo 6 SUNAT (ej: "C62")
    public string  CodigoSUNAT        { get; set; }   // catálogo 25 SUNAT (BRECHA — campo nuevo)
    public string  NumeroOrdenCompra  { get; set; }   // Nº OC del cliente (criterio D2)
    public string  Descripcion        { get; set; }   // Nombre + catálogo + lote
    public decimal ValorUnitario      { get; set; }
    public decimal Importe            { get; set; }
    public string  AfectacionIGV      { get; set; }   // catálogo 7 SUNAT (ej: "10" = Gravado)
    public decimal ICBPERLinea        { get; set; }   // 0 si no aplica
    public string  TipoPrecio         { get; set; }   // catálogo 16 SUNAT (ej: "01" precio unitario incluye IGV) — observado en E001-362; pendiente confirmar
}

public class FacturaPeruPdfCreditoModel
{
    public decimal             MontoNetoPendiente { get; set; }
    public int                 TotalCuotas        { get; set; }
    public List<FacturaPeruPdfCuotaModel> Cuotas  { get; set; }
}

public class FacturaPeruPdfCuotaModel
{
    public int      NumeroCuota      { get; set; }
    public DateTime FechaVencimiento { get; set; }
    public decimal  Monto            { get; set; }
}
```

#### Application — FacturaPeruPdfMappingService

Servicio que consolida los datos del CPE en un `FacturaPeruPdfModel`, listo para ser consumido por DocumentBuilder.

**Fuentes de datos por sección:**

| Sección | Tabla / Fuente |
|---------|---------------|
| Branding / Logo | `Empresa` por prefijo `GOLPERU` |
| Emisor | `CPEGenerada` (RUCEmisor, RazonSocialEmisor, DireccionEmisor, UbigeoEmisor) |
| Receptor | `CPEGenerada` (RUCReceptor, RazonSocialReceptor, DireccionReceptor) |
| Comprobante — metadata | `CPEGenerada` (Serie, Correlativo, TipoComprobante, TipoOperacion, FechaEmision, CondicionPago, Moneda, TipoCambio) |
| Folio Pedido Interno PI | `tpPedido` o equivalente en ProquifaDotNet (criterio G3) |
| Partidas — CodigoSUNAT | `Producto.CodigoSUNAT` (**BRECHA — campo nuevo RE-FU-020**) |
| Partidas — ClaveSUNAT UdM | `catUnidad.ClaveSUNAT` (**BRECHA — campo nuevo RE-FU-020**) |
| Partidas — AfectaciónIGV | `catAfectacionIGV.Clave` (**BRECHA — tabla nueva RE-FU-020**) |
| Partidas — cantidad / PU | `tpPartidaPedido` (NumeroDePiezas, PrecioUnitario) |
| Totales SUNAT | `CPEGenerada` (ValorVenta, IGV, ISC, ICBPER, OtrosTributos, Total) |
| Crédito / Cuotas | `CPEGenerada` (CondicionPago) + tabla de cuotas pendiente definir |
| Elementos técnicos SUNAT | XML del OSE/PSE — parseo directo (no BD) |
| QR SUNAT | Generado / extraído del XML del OSE/PSE |

**Interfaz propuesta:**

```csharp
public interface IFacturaPeruPdfMappingService
{
    // PDF definitivo — incluye QR, firma digital y valor resumen del OSE/PSE
    Task<FacturaPeruPdfModel> MapearAsync(Guid idCPEGenerada, string xmlFirmadoSunat);

    // Preview — sin elementos técnicos SUNAT (Sección G en null)
    Task<FacturaPeruPdfModel> MapearPreviewAsync(Guid idCPEGenerada);
}
```

**Flujo interno — `MapearAsync`:**

```
1. Consultar CPEGenerada → RUCEmisor, RazonSocialEmisor, DireccionEmisor, UbigeoEmisor,
   RUCReceptor, RazonSocialReceptor, Serie, Correlativo, TipoComprobante, TipoOperacion,
   FechaEmision, CondicionPago, Moneda, TipoCambio, ValorVenta, IGV, ISC, ICBPER,
   OtrosTributos, Total
2. Consultar tpPartidaPedido JOIN Producto JOIN catUnidad JOIN catAfectacionIGV
   → CodigoSUNAT, ClaveSUNAT, AfectaciónIGV, NumeroDePiezas, PrecioUnitario, Descripcion
3. Consultar Empresa por prefijo GOLPERU → LogoBase64
4. Consultar FolioPedidoInterno (tpPedido o equivalente)
5. Parsear xmlFirmadoSunat → extraer QR, FirmaDigital, ValorResumen del CDR
6. Calcular totales derivados (SubTotalVentas, TotalEnLetras, etc.)
7. Resolver Crédito/Cuotas si CondicionPago = 'Crédito'
8. Ensamblar y retornar FacturaPeruPdfModel
```

**Consulta BD — Partidas con datos SUNAT** (llamada API a ProquifaDotNet):

```sql
-- Ejecutar en ProquifaDotNet
SELECT
    pp.NumeroDePiezas                              AS Cantidad,
    u.ClaveSUNAT                                   AS UnidadMedidaSUNAT,
    p.CodigoSUNAT                                  AS CodigoSUNAT,
    p.Descripcion                                  AS Descripcion,
    pp.PrecioUnitario                              AS ValorUnitario,
    (pp.NumeroDePiezas * pp.PrecioUnitario)        AS Importe,
    aig.Clave                                      AS AfectacionIGV
FROM tpPartidaPedido pp
INNER JOIN Producto         p   ON pp.IdProducto       = p.IdProducto
INNER JOIN catUnidad        u   ON p.IdUnidad           = u.IdUnidad
LEFT  JOIN catAfectacionIGV aig ON p.IdCatAfectacionIGV = aig.IdCatAfectacionIGV
WHERE pp.IdPedido = @IdPedido
ORDER BY pp.NumeroPartida
```

#### Application — PersistirFacturaPeruPdfService

Servicio transaccional que, tras el timbrado exitoso ante SUNAT/OSE, genera el PDF definitivo del CPE y lo persiste en Minio.

**TemplateKey:** `GOLPERU_PER_FAC` (única empresa emisora Perú)

**Flujo interno:**

```
1. Invocar FacturaPeruPdfMappingService.MapearAsync(IdCPEGenerada, xmlFirmadoSunat)
   → FacturaPeruPdfModel con QR, FirmaDigital, ValorResumen completos
2. TemplateKey = 'GOLPERU_PER_FAC' (fijo — única empresa Perú)
3. Invocar DocumentBuilder → generar PDF en bytes
4. Subir PDF a Minio (bucket 'facturas', IdRegion='PER')
5. INSERT Archivo (NombreArchivo, FileBucket='facturas', IdRegion='PER',
   ContentType='application/pdf') → obtener IdArchivo
6. Llamar API ProquifaDotNet → UPDATE tpProformaAdelanto SET IdCPEGenerada + referencia PDF
7. Si pasos 4-6 fallan: reintentar sin re-timbrar ante SUNAT/OSE
8. Registrar en Serilog: módulo, IdCPEGenerada, fecha, resultado (éxito/error + mensaje)
```

**Integración con RE-FU-020 — `FacturaAdelantadoGenerarService` (branch PER):**

```csharp
// Branch PER del flujo generar FAA — tras el timbrado exitoso SUNAT:
// Paso 10 (reemplaza placeholder): persistir PDF definitivo CPE
await _persistirFacturaPeruPdfService.PersistirAsync(idCPEGenerada, response.XmlFirmado);
// Paso 11 (sin cambios): INSERT Archivo XML CDR — patrón existente
```

**Integración con RE-FU-020 — `FacturaAdelantadoPreviewService` (branch PER):**

```csharp
// Branch PER del preview — template real CPE UBL 2.1:
var model = await _mappingService.MapearPreviewAsync(idCPEGenerada);
// TemplateKey fijo para Perú (una sola empresa emisora)
var pdfBytes = await _documentBuilder.GenerarAsync("GOLPERU_PER_FAC", model);
```

---

## Parte C — Scripts de Base de Datos (ProquifaDotNet)

### Orden de ejecución de scripts

| Paso | Script | Tipo | BD | Dependencia | Estado |
|------|--------|------|----|-------------|--------|
| 1 | `CREATE TABLE CPEGenerada` | DDL | ProquifaDotNet | Ninguna | **BLOQUEANTE para paso 2** |
| 2 | `ALTER TABLE tpProformaAdelanto ADD IdCPEGenerada FK` | DDL | ProquifaDotNet | Paso 1 | Requiere tabla CPEGenerada |
| 3 | `ALTER TABLE Producto ADD CodigoSUNAT` | DDL | ProquifaDotNet | **RE-FU-020** | **BLOQUEANTE partidas PDF** |
| 4 | `ALTER TABLE catUnidad ADD ClaveSUNAT` | DDL | ProquifaDotNet | **RE-FU-020** | **BLOQUEANTE partidas PDF** |
| 5 | `CREATE TABLE catAfectacionIGV` + INSERT catálogo 7 SUNAT | DDL + DML | ProquifaDotNet | **RE-FU-020** | **BLOQUEANTE partidas PDF** |
| 6 | `ALTER TABLE Producto ADD IdCatAfectacionIGV FK` | DDL | ProquifaDotNet | Paso 5 | **BLOQUEANTE partidas PDF** |
| 7 | `UPDATE Empresa GOLPERU` (RUC, DireccionFiscal, Ubigeo) | DML | ProquifaDotNet | **⚠️ BRECHA datos legales** | Pendiente datos |

> **Nota:** Los pasos 3-6 son compartidos con RE-FU-020 — se ejecutan una sola vez. Si ya fueron aplicados en RE-FU-020, no repetir.

### Scripts DDL y DML

#### 1. CREATE TABLE CPEGenerada — BLOQUEANTE

```sql
-- Ejecutar en ProquifaDotNet
CREATE TABLE [dbo].[CPEGenerada](
    [IdCPEGenerada]            uniqueidentifier NOT NULL CONSTRAINT [DF_CPEGenerada_Id]          DEFAULT (NEWID()),
    [RUCEmisor]                varchar(11)      NOT NULL,
    [RazonSocialEmisor]        varchar(200)     NOT NULL,
    [DireccionEmisor]          varchar(300)     NULL,
    [UbigeoEmisor]             varchar(6)       NULL,
    [RUCReceptor]              varchar(11)      NOT NULL,
    [RazonSocialReceptor]      varchar(200)     NOT NULL,
    [DireccionReceptor]        varchar(300)     NULL,
    [TipoComprobante]          varchar(2)       NOT NULL CONSTRAINT [DF_CPEGenerada_TipoCPE]  DEFAULT ('01'),
    [TipoOperacion]            varchar(4)       NULL,     -- catálogo 51 SUNAT (ej: '0101')
    [Serie]                    varchar(4)       NOT NULL, -- ej: 'F001', 'E001'
    [Correlativo]              varchar(8)       NOT NULL, -- hasta 8 dígitos
    [CondicionPago]            varchar(50)      NULL,     -- 'Contado' / 'Crédito'
    [Moneda]                   varchar(3)       NOT NULL, -- PEN, USD
    [TipoCambio]               decimal(18,6)    NULL,
    [ValorVenta]               decimal(18,2)    NOT NULL,
    [IGV]                      decimal(18,2)    NOT NULL,
    [ISC]                      decimal(18,2)    NOT NULL CONSTRAINT [DF_CPEGenerada_ISC]     DEFAULT (0),
    [ICBPER]                   decimal(18,2)    NOT NULL CONSTRAINT [DF_CPEGenerada_ICBPER]  DEFAULT (0),
    [OtrosTributos]            decimal(18,2)    NOT NULL CONSTRAINT [DF_CPEGenerada_Otros]   DEFAULT (0),
    [Total]                    decimal(18,2)    NOT NULL,
    [Observaciones]            varchar(500)     NULL,
    [FechaEmision]             datetime2(7)     NOT NULL,
    [Activo]                   bit              NOT NULL CONSTRAINT [DF_CPEGenerada_Activo]   DEFAULT (1),
    [FechaRegistro]            datetime2(7)     NOT NULL CONSTRAINT [DF_CPEGenerada_FechaReg] DEFAULT (SYSUTCDATETIME()),
    [FechaUltimaActualizacion] datetime2(7)     NOT NULL CONSTRAINT [DF_CPEGenerada_FechaUpd] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_CPEGenerada] PRIMARY KEY CLUSTERED ([IdCPEGenerada])
);
GO
```

#### 2. ALTER TABLE tpProformaAdelanto (IdCPEGenerada FK)

```sql
-- Ejecutar en ProquifaDotNet
-- Vinculación Perú: equivalente a IdCFDIGenerada pero para CPE SUNAT
ALTER TABLE dbo.tpProformaAdelanto
    ADD IdCPEGenerada uniqueidentifier NULL
        CONSTRAINT [FK_tpProformaAdelanto_CPEGenerada]
            FOREIGN KEY REFERENCES dbo.CPEGenerada([IdCPEGenerada]);
GO
```

#### 3–6. Scripts RE-FU-020 (ejecutar si aún no aplicados)

```sql
-- BLOQUEANTE para partidas del PDF Perú — ver scripts en R16A-RE-FU-020-Back.md Parte C
-- ALTER TABLE Producto ADD CodigoSUNAT varchar(10) NULL
-- ALTER TABLE catUnidad ADD ClaveSUNAT varchar(10) NULL
-- CREATE TABLE catAfectacionIGV + INSERT catálogo 7 SUNAT
-- ALTER TABLE Producto ADD IdCatAfectacionIGV FK
```

#### 7. UPDATE Empresa GOLPERU — BRECHA datos legales pendientes

```sql
-- Ejecutar en ProquifaDotNet cuando se tengan los datos legales de Golocaer S.A.C. Perú
UPDATE dbo.Empresa
SET
    RFC             = '20XXXXXXXXXX',   -- RUC Golocaer S.A.C. (11 dígitos) — PENDIENTE
    DireccionFiscal = 'PENDIENTE',       -- Dirección fiscal Perú
    Ubigeo          = 'XXXXXX'           -- Ubigeo SUNAT (6 dígitos) — PENDIENTE
WHERE Prefijo = 'GOLPERU';
GO
```

---

## Gaps de Desarrollo

### En DocumentBuilder

| # | Gap | Acción | Esfuerzo | Estado |
|---|-----|--------|----------|--------|
| GAP-01 | Template `GOLPERU_PER_FAC` — `GOLPERU_PER_FAC_H.html` + `GOLPERU_PER_FAC_B.html` + `GOLPERU_PER_FAC_F.html` + INSERT DocumentTemplate | Crear 3 archivos HTML con branding Golocaer S.A.C. Perú + script SQL MERGE | Alto | **⚠️ Branding pendiente** |

### En ProquifaDotNet.Finanzas

| # | Gap | Acción | Esfuerzo | Estado |
|---|-----|--------|----------|--------|
| GAP-02 | `FacturaPeruPdfModel` + modelos auxiliares (Domain) | Nuevos modelos con secciones A-G del CPE UBL 2.1 | Bajo | Abierto |
| GAP-03 | `IFacturaPeruPdfMappingService` + `FacturaPeruPdfMappingService` (Application) | Consolidar datos CPEGenerada + parsear XML OSE/PSE + QR/Firma | Alto | **⛔ BRECHA datos SUNAT producto + OSE** |
| GAP-04 | `PersistirFacturaPeruPdfService` (Application) | Generar PDF → persistir Minio `IdRegion='PER'` → INSERT Archivo → UPDATE tpProformaAdelanto → reintentos sin re-timbrado | Medio | **⛔ BRECHA OSE/PSE** |
| GAP-05 | Extender `FacturaAdelantadoGenerarService` branch PER (RE-FU-020) | Reemplazar placeholder con llamada real a `PersistirFacturaPeruPdfService` | Bajo | **⛔ BRECHA OSE/PSE** |
| GAP-06 | Extender `FacturaAdelantadoPreviewService` branch PER (RE-FU-020) | Reemplazar template placeholder con template real `GOLPERU_PER_FAC` | Bajo | Abierto |
| GAP-07 | Integrar `PersistirFacturaPeruPdfService` en flujo Validar Cobro Perú | Invocar `PersistirAsync` tras timbrado exitoso SUNAT en Validar Cobro | Bajo | **⛔ BRECHA OSE/PSE** |

### En Base de Datos (ProquifaDotNet)

| # | Gap | Acción | BD | Esfuerzo | Estado |
|---|-----|--------|----|----------|--------|
| GAP-08 | `CREATE TABLE CPEGenerada` | DDL | ProquifaDotNet | Medio | **BLOQUEANTE** |
| GAP-09 | `ALTER TABLE tpProformaAdelanto ADD IdCPEGenerada FK` | DDL | ProquifaDotNet | Bajo | Requiere GAP-08 |
| GAP-10 | Scripts RE-FU-020: `CodigoSUNAT`, `ClaveSUNAT`, `catAfectacionIGV` | DDL + DML | ProquifaDotNet | Medio | **BLOQUEANTE** (compartidos con RE-FU-020) |
| GAP-11 | `UPDATE Empresa GOLPERU` (RUC, DireccionFiscal, Ubigeo) | DML | ProquifaDotNet | Bajo | **⚠️ BRECHA datos legales pendientes** |

---

## Diagrama de Flujo — Generación PDF Factura Perú

```
[FacturaAdelantadoGenerarService]     [ProquifaDotNet .NET Fx 4.8]     [DocumentBuilder]     [Minio]
(branch PER — tras timbrado SUNAT/OSE exitoso)
             |                                     |                          |                  |
1. PersistirFacturaPeruPdfService.PersistirAsync(IdCPEGenerada, xmlFirmadoSunat)
             |                                     |                          |                  |
2. FacturaPeruPdfMappingService.MapearAsync        |                          |                  |
             |---GET CPEGenerada / Partidas SUNAT / LogoGOLPERU ------------->|                  |
             |<---datos fiscales CPE ------------------------------------------                  |
             | 3. Parsear XML OSE/PSE → QR, FirmaDigital, ValorResumen        |                  |
             | 4. Calcular totales SUNAT (TotalEnLetras, SubTotalVentas, etc.) |                  |
             | 5. TemplateKey = 'GOLPERU_PER_FAC' (fijo)                      |                  |
             |---POST /generar-pdf (GOLPERU_PER_FAC + FacturaPeruPdfModel) --> |                  |
             |<---PDF bytes --------------------------------------------------                   |
             |---PUT bucket 'facturas' / IdRegion='PER' / {IdCPEGenerada}.pdf ----------------->|
             |<---referencia Minio ------------------------------------------------------------|
             | 6. INSERT Archivo (FileBucket='facturas', IdRegion='PER')      |                  |
             |---PUT /api/tpProformaAdelanto/{Id}/cpe-generada ------------->  |                  |
             |<---200 OK -------------------------------------------------------                 |
             | 7. Serilog: módulo, IdCPEGenerada, fecha, resultado            |                  |
```

> **Preview (branch PER):** `FacturaAdelantadoPreviewService` llama a `MapearPreviewAsync` (sin QR/Firma/Resumen) → TemplateKey fijo `GOLPERU_PER_FAC` → DocumentBuilder genera PDF en memoria → retorna bytes sin persistir en Minio.
