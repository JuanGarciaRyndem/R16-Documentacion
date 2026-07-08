# Impacto en Back — R16A-RE-FU-020
**Requisito:** Factura por Adelantado: Detalle Perú
**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10)
**Módulo:** Factura por Adelantado — Detalle Perú (adaptación regional)
**Impacto:** Timbrado SUNAT/OSE (UBL 2.1) + Detalle Perú + Envío + Validar Cobro

---

## Resumen

Este requisito implementa el flujo de **Detalle por cliente para Región Perú** del módulo Factura por Adelantado, como adaptación regional de R16A-RE-FU-019 (México). Las diferencias fundamentales respecto a México son:

| Aspecto | México (RE-FU-019) | Perú (RE-FU-020) |
|---------|--------------------|------------------|
| Tipo de pedido | Crédito + Prepago | **Solo Prepago** |
| Empresa emisora | GOL / MUN / PRO / PQF | **Solo GOLPERU (Golocaer S.A.C.)** |
| Proveedor timbrado | SAP PAC (TurboPac) | **OSE/PSE SUNAT** (por definir) |
| Formato XML | CFDI 4.0 | **UBL 2.1** |
| Impuesto | IVA (variable) | **IGV 18% (fijo)** |
| Identificador fiscal | RFC (13 chars) | **RUC (11 dígitos)** |
| Método/Forma de Pago | PPD + 99 (forzados SAT) | **No existen en SUNAT** |
| Complemento de Pago | Genera en Validar Cobro | **No aplica en Perú** |
| Dato configurable | Uso CFDI (catálogo SAT) | **Tipo de Operación (catálogo 51 SUNAT)** |
| Folio/Serie | varchar(6) numérico | **Serie 4 chars + correlativo 8 dígitos** |
| Salida operativa | Legacy (Crédito) o Validar Cobro (Prepago) | **Solo Validar Cobro (siempre Prepago)** |
| Transferencia Legacy | Sí (Crédito) | **NO — nada va a Legacy** |

### Infraestructura reutilizada de RE-FU-015/018/019

> **Migración (06/07/2026):** `tpProformaAdelanto.Enviada` y `vtpProformaAdelanto` (originalmente creados en RE-FU-019) se retiraron de ese requisito. Ambos objetos ahora viven en **R16A-RE-FU-015** como `fccFactura.Enviada` y `vfccFactura` — Perú sigue reutilizando el mismo objeto compartido, solo que ahora es `fccFactura`/`vfccFactura`.

| Componente | Origen | Reutilización |
|------------|--------|---------------|
| `fccFactura.Enviada` | RE-FU-015 (movido desde RE-FU-019) | Mismo campo para PER |
| `vfccFactura` | RE-FU-015 (movido desde RE-FU-019) | Ya filtra por RegionClave |
| Tabla `EmpresaFolio` | RE-FU-018/019 | Solo agregar fila GOLPERU |
| Tabla `CFDIGenerada` (ProquifaDotNet, propiedad de Finanzas) | RE-FU-019 (base) + RE-FU-018 (extension) | Misma tabla que MEX, distinto XML/CDR SUNAT |
| Tabla `StampingLog` (ProquifaDotNetTimbrado) | RE-FU-018 | Misma tabla (auditoria tecnica, sin FK real) |
| `AdvanceInvoiceDetailRepository` | RE-FU-019 | Reutilizar con filtro RegionClave='PER' |
| `AdvanceInvoiceSendService` | RE-FU-019 | Solo rama Prepago (sin Crédito) |
| `FinanzasContext` | RE-FU-016/019 | Ampliar si se requieren tablas nuevas SUNAT |

### Brechas bloqueantes

> ⛔ **BRECHA BLOQUEANTE — Datos fiscales SUNAT del producto**
> Los campos `CodigoSUNAT`, `ClaveSUNAT` (unidad de medida) y `AfectacionIGV` son **obligatorios** en el XML UBL 2.1 y **no existen** en el catálogo de productos actual. Sin ellos no es posible generar el CPE. Documentado en R16A-RE-FU-005 Brecha 1.

> ⛔ **BRECHA MAYOR — Integración OSE/PSE SUNAT**
> El proveedor de timbrado para Perú está **pendiente de definir**. No se reutiliza el PAC de México. Documentado en R16A-RE-FU-005 Brecha 5.

> ⚠️ **BRECHA — Datos legales Golocaer S.A.C.**
> RUC, domicilio fiscal, ubigeo, certificado digital y series de facturación **no están registrados** en el sistema. Sin estos datos el timbrado no puede ejecutarse.

---

## Parte A — ProquifaDotNet.Timbrado (Ampliación para SUNAT)

### Descripción

Ampliar la solución Timbrado (RE-FU-018) con soporte para el estándar SUNAT/OSE: cliente HTTP hacia el OSE/PSE peruano, armado de XML UBL 2.1, recepción del CDR de aceptación y serie SUNAT para GOLPERU.

### Diferencias vs Flujo México

| Paso | México (SapStampingClient) | Perú (OseStampingClient) |
|------|---------------------------|--------------------------|
| Formato XML | CFDI 4.0 | UBL 2.1 |
| Validación previa | PAC valida contra SAT | OSE valida contra SUNAT |
| Respuesta exitosa | UUID + XML timbrado | CDR (Constancia de Recepción) + XML timbrado |
| Folio | Numérico varchar(6) | Serie F001 + correlativo 8 dígitos |
| Cancelación | POST /cancelar CFDI | Proceso de anulación SUNAT (distinto) |

### Nuevos Componentes

#### Domain — Interface IOseStampingClient

```csharp
public interface IOseStampingClient
{
    Task<OseStampingResponse> StampAsync(OseStampingRequest request);
}
```

#### Application — DTOs SUNAT

| DTO | Campos principales |
|-----|--------------------|
| `StampSunatInvoiceRequestDto` | IdProformaAdelanto, IssuerData (RUC, RazonSocial, DireccionFiscal, Ubigeo, Serie), RecipientData (RUC, RazonSocial, DireccionFiscal, RegimenTributario), `Conceptos[]`, TipoOperacion (cat. 51), Moneda, TipoCambio (si aplica) |
| `SunatItemDto` | Cantidad, Descripcion, ValorUnitario, Importe, CodigoSUNAT, ClaveSUNAT (UdM), IdAfectacionIGV, ValorIGV |
| `SunatIssuerDataDto` | RUC, RazonSocial, DireccionFiscal, Ubigeo, Serie, EmpresaClave |
| `SunatRecipientDataDto` | RUC, RazonSocial, DireccionFiscal, RegimenTributario, Correo, Moneda |
| `StampSunatInvoiceResponseDto` | NumeroCPE (Serie-Correlativo), FechaEmision, Total, XmlBase64, CdrBase64, Exitoso, ErrorDescripcion, CodigoErrorSunat — sin IdCFDI: el Id de negocio real (`IdCFDIGenerada`) lo asigna Finanzas al persistir |

#### Application — StampingService (Ampliación)

Agregar método `StampSunatInvoiceAsync(StampSunatInvoiceRequestDto)` que orquesta:

```
1. Validar request (FluentValidation)
2. Consumir folio via EmpresaFolioService.GetNextFolioAsync('GOLPERU')
   → retorna Serie-Correlativo formateado (ej: F001-00000001)
3. Armar XML UBL 2.1 con datos fiscales + conceptos SUNAT + IGV 18%
4. INSERT StampingLog (NewStatus=Pending)
5. Llamar OseStampingClient → OSE/PSE SUNAT
6. Si ERROR: UPDATE StampingLog (NewStatus=Failed) y retornar StampSunatInvoiceResponseDto con
   Exitoso=false + CodigoErrorSunat (sin persistir ningun CPE de negocio)
7. Si ÉXITO (CDR aceptado): UPDATE StampingLog (NewStatus=Stamped)
8. Retornar StampSunatInvoiceResponseDto con Exitoso=true + NumeroCPE, XmlBase64, CdrBase64
   (el XML/CDR se regresan en el response; Finanzas los sube a Minio y los persiste, ver
   R16A-RE-FU-018-Back.md Parte B)
```

> Nota: los pasos "INSERT/UPDATE CFDI" y "Subir XML+CDR a Minio (bucket 'facturas-peru')" de
> versiones previas de esta tarea se eliminaron — Timbrado no tiene tabla `Cfdi` propia ni sube
> archivos a Minio; ambas responsabilidades son de Finanzas (`CfdiService`, igual que en MEX).

#### Infrastructure — OseStampingClient

```csharp
public class OseStampingClient : IOseStampingClient
{
    // POST al endpoint OSE/PSE configurado en AppSetting 'SUNAT_OSE_Endpoint'
    // Autenticación: certificado digital PFX de Golocaer S.A.C.
    // Payload: XML UBL 2.1 firmado digitalmente
    // Response: CDR (Constancia de Recepción) con código de respuesta SUNAT
    Task<OseStampingResponse> StampAsync(OseStampingRequest request);
}
```

> El proveedor OSE/PSE está **pendiente de definir** (Brecha 5 de R16A-RE-FU-005). El cliente HTTP debe ser intercambiable via interface para facilitar el cambio de proveedor.

#### API — StampingController (Ampliación)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/v1/stamp/invoice` | Recibe el request técnico SUNAT armado por Finanzas, retorna `StampSunatInvoiceResponseDto` — reutiliza el endpoint técnico de facturas (`/api/v1/stamp/invoice`) creado en RE-FU-018, sin controller ni ruta separados. El recurso de negocio `cfdi` (`CfdiController`, `POST /api/v1/cfdi`) vive en Finanzas. |

---

## Parte B — ProquifaDotNet.Finanzas (Adaptación Regional Perú)

### Descripción

Adaptar el módulo FAA en Finanzas (RE-FU-019) para soportar el flujo de Perú. La arquitectura es la misma; los servicios branch internamente según `RegionClave` del cliente ('MEX' o 'PER').

### Endpoints — Reutilización y Extensión

| Endpoint | Reutilización | Adaptación Perú |
|----------|---------------|-----------------|
| `POST /api/v1/advanceInvoice/{clientId}/detail` | ✅ Reutiliza | Filtro `RegionClave='PER'` en request |
| `POST /api/v1/advanceInvoice/{id}/generate` | ⚙️ Extiende | Branch interno: si PER → arma UBL 2.1 → llama `ICfdiService.GenerateAsync` (que a su vez llama `POST /api/v1/stamp/invoice` en Timbrado y persiste CFDIGenerada) |
| `POST /api/v1/advanceInvoice/{id}/preview` | ⚙️ Extiende | Branch interno: si PER → template PDF GOLPERU |
| `POST /api/v1/advanceInvoice/{id}/send` | ✅ Reutiliza | Sin rama Crédito; solo Validar Cobro |

> **Nota (Reglas al diseñar — regla 9, corrección):** rutas corregidas de `/api/factura-adelantado/*` (español) al recurso ya establecido `advanceInvoice` (RE-FU-018/019) bajo `api/v1/`.

### Endpoint: Detalle Perú (Pedidos por cliente)

Reutiliza `AdvanceInvoiceDetailRepository` con el filtro `RegionClave='PER'` en el request.

La vista `vfccFactura` (RE-FU-015) ya incluye `RegionClave` por lo que no requiere cambios en BD.

Diferencias en el response para Perú:

| Campo | México | Perú |
|-------|--------|------|
| Cabecera cliente | RFC | **RUC** |
| Impuesto en listado | IVA | **IGV** |
| Empresa Emisora | GOL/MUN/PRO/PQF | **GOLPERU** |

### Endpoint: Generar Factura Perú

#### Adaptaciones en `AdvanceInvoiceGenerateService`

El servicio detecta la región del cliente y ejecuta el branch correspondiente:

```
Si RegionClave = 'PER':
  1. Validar EstadoFAA = 'PendienteGenerar'
  2. Obtener datos del pedido y partidas (con CodigoSUNAT, ClaveSUNAT, AfectacionIGV) ← BRECHA
  3. Obtener datos fiscales SUNAT del cliente (RUC, RazonSocial, DireccionFiscal, RegimenTributario)
  4. Obtener datos de Golocaer S.A.C. (RUC emisor, DireccionFiscal, Ubigeo, Serie) ← BRECHA datos legales
  5. Obtener Tipo de Cambio PEN (si moneda != PEN) ← Fuente pendiente definir
  6. Armar StampSunatInvoiceRequestDto con:
     - Conceptos con CodigoSUNAT + ClaveSUNAT + AfectacionIGV + IGV 18% por línea
     - TipoOperacion cat. 51 SUNAT (fijo "0101" o seleccionable — pendiente definir)
     - SIN MetodoPago / FormaPago / TipoComprobante SAT
  7. Llamar ICfdiService.GenerateAsync(request) — internamente: llama ApiCallerStamping.StampSunatInvoiceAsync
     -> Timbrado POST /api/v1/stamp/invoice, y si el CDR es aceptado, INSERT CFDIGenerada + Archivo (PDF+XML+CDR,
     bucket 'facturas-peru') en ProquifaDotNet
  8. Si ERROR: UPDATE fccFactura SET IdCatFacturaEstado = ERROR_TIMBRADO (sin tocar IdCFDIGenerada) y
     retornar AdvanceInvoiceGenerateResponseDto con Exitoso=false + ErrorDescripcion (SUNAT)
  9. Si ÉXITO: UPDATE fccFactura SET IdCFDIGenerada = @IdCFDIGenerada, EsFacturaPorAdelantado = 0,
     IdCatFacturaEstado = GENERADA (catFacturaEstado, RE-FU-015 v2.1)
     (Id real retornado por ICfdiService.GenerateAsync, correspondiente al registro insertado en CFDIGenerada)
  10. Registrar el guardado de la factura en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo — regla 8)
  11. Retornar AdvanceInvoiceGenerateResponseDto con Exitoso=true + NumeroCPE (Serie-Correlativo)
```

> Nota: los pasos "Almacenar PDF+XML+CDR en Minio" e "INSERT Archivo" de versiones previas de esta
> tarea ya no los ejecuta `AdvanceInvoiceGenerateService` directamente — `ICfdiService.GenerateAsync`
> (RE-FU-018) se encarga de la subida a Minio y del vínculo en `CFDIGenerada.IdArchivoXml`, igual que
> en el flujo México (RE-FU-019).

#### Adaptaciones en `AdvanceInvoiceFiscalDataRepository`

Métodos adaptados para Perú:

| Método | Adaptación |
|--------|-----------|
| `GetClientFiscalDataAsync` | Retorna RUC (en vez de RFC), DireccionFiscal, RegimenTributario (Perú) |
| `GetIssuerFiscalDataAsync` | Retorna datos de GOLPERU: RUC, RazonSocial, DireccionFiscal, Ubigeo, Serie SUNAT |
| `GetOperationTypeCatalogAsync()` | **NUEVO** — retorna catálogo 51 SUNAT (equivalente a GetCfdiUsageCatalogAsync para México) |
| `GetSunatProductDataAsync` | **NUEVO** — obtiene CodigoSUNAT, ClaveSUNAT, IdAfectacionIGV por partida ← **BRECHA** |

#### Adaptaciones en `AdvanceInvoicePreviewService`

El preview detecta la región y usa el template correspondiente:
- Perú → template DocumentBuilder `GOLPERU_PER_FAA` (por definir en requisito independiente)
- México → template DocumentBuilder `GOLMEX_MEX_FAA` (existente)

### Nuevos DTOs Perú-Específicos en Finanzas

| DTO | Propósito |
|-----|-----------|
| `ClientFiscalDataPeruDto` | RUC, RazonSocial, DireccionFiscal, RegimenTributario, Correo, Moneda, TipoCambio |
| `IssuerFiscalDataPeruDto` | RUC, RazonSocial, DireccionFiscal, Ubigeo, Serie SUNAT, EmpresaClave='GOLPERU' |
| `OperationTypeDto` | Clave (ej: '0101'), Descripcion ('Venta interna') |

### ApiCallerStamping — Método Nuevo

> `IApiCallerStamping`/`ApiCallerStamping` ya existen (RE-FU-018 Parte B, Tarea 10); aquí solo se agrega el método SUNAT.

| Método | Endpoint Timbrado | Descripción |
|--------|------------------|-------------|
| `StampSunatInvoiceAsync(StampSunatInvoiceRequestDto)` | `POST /api/v1/stamp/invoice` | Timbrado UBL 2.1 SUNAT (invocado internamente por `CfdiService.GenerateAsync`, no directamente por `AdvanceInvoiceGenerateService`) |

---

## Parte C — Scripts de Base de Datos

### ProquifaDotNetTimbrado — Nuevos registros

**INSERT EmpresaFolio GOLPERU:**

```sql
-- Ejecutar en ProquifaDotNetTimbrado
-- Serie SUNAT de Golocaer S.A.C. PENDIENTE DE CONFIRMAR
INSERT INTO [dbo].[EmpresaFolio]
    ([EmpresaClave], [EmpresaNombre], [Serie], [UltimoFolio], [FormatoFolio], [LongitudMaxima])
VALUES
    ('GOLPERU', 'Golocaer S.A.C.', 'F001', 0, 'F001-{folio:00000000}', 8);
-- Ajustar Serie y UltimoFolio al último usado en producción
```

**INSERT AppSetting config OSE/PSE Perú:**

```sql
-- Ejecutar en ProquifaDotNetTimbrado
-- Valores PENDIENTES hasta definir proveedor OSE/PSE
INSERT INTO [dbo].[AppSetting] ([Name], [Value], [Description])
VALUES
    ('SUNAT_OSE_Endpoint',  'https://ose.PENDIENTE.com/api', 'Endpoint OSE/PSE Perú — PENDIENTE proveedor'),
    ('SUNAT_RUC_Emisor',    '20XXXXXXXXXX',                  'RUC Golocaer S.A.C. — PENDIENTE datos legales'),
    ('SUNAT_CertPath',      '/secrets/golperu_cert.pfx',     'Certificado digital Golocaer S.A.C. — PENDIENTE');
```

### ProquifaDotNet — Scripts BLOQUEANTES

> ⛔ Los siguientes scripts son **prerrequisito bloqueante** para el desarrollo del flujo de facturación Perú. Sin ellos el XML UBL 2.1 no puede generarse.

**ALTER TABLE Producto (CodigoSUNAT):**

```sql
-- BLOQUEANTE: catálogo 25 SUNAT — pendiente mapeo con equipo técnico y asesor peruano
ALTER TABLE dbo.Producto
    ADD CodigoSUNAT varchar(10) NULL; -- catálogo 25 SUNAT (ej: '10211502')
```

**ALTER TABLE catUnidad (ClaveSUNAT):**

```sql
-- BLOQUEANTE: catálogo 6 SUNAT — pendiente mapeo
ALTER TABLE dbo.catUnidad
    ADD ClaveSUNAT varchar(10) NULL; -- catálogo 6 SUNAT (ej: KGM, NIU, ZZ)
```

**CREATE TABLE catAfectacionIGV:**

```sql
-- BLOQUEANTE: catálogo 7 SUNAT — afectación IGV por línea de factura
CREATE TABLE [dbo].[catAfectacionIGV] (
    [IdCatAfectacionIGV] uniqueidentifier NOT NULL
        CONSTRAINT [DF_catAfectacionIGV_Id]     DEFAULT (NEWID()),
    [Clave]              varchar(4)       NOT NULL,  -- cat. 7 SUNAT: 10, 11, 20, 30, 40...
    [Descripcion]        varchar(200)     NOT NULL,
    [Activo]             bit              NOT NULL
        CONSTRAINT [DF_catAfectacionIGV_Activo] DEFAULT (1),
    CONSTRAINT [PK_catAfectacionIGV]       PRIMARY KEY CLUSTERED ([IdCatAfectacionIGV]),
    CONSTRAINT [UQ_catAfectacionIGV_Clave] UNIQUE ([Clave])
);
GO

-- INSERT datos iniciales catálogo 7 SUNAT (principales)
INSERT INTO [dbo].[catAfectacionIGV] ([Clave], [Descripcion])
VALUES
    ('10', 'Gravado - Operación Onerosa'),
    ('20', 'Exonerado - Operación Onerosa'),
    ('30', 'Inafecto - Operación Onerosa');
```

**ALTER TABLE Producto (FK AfectacionIGV):**

```sql
-- BLOQUEANTE: vincula producto con su afectación IGV para el CPE
ALTER TABLE dbo.Producto
    ADD IdCatAfectacionIGV uniqueidentifier NULL
        CONSTRAINT [FK_Producto_AfectacionIGV]
            FOREIGN KEY REFERENCES dbo.catAfectacionIGV ([IdCatAfectacionIGV]);
```

**UPDATE Empresa GOLPERU (datos legales — BRECHA):**

```sql
-- BRECHA: datos legales de Golocaer S.A.C. pendientes de recopilación
-- Ejecutar cuando se tengan los datos reales
UPDATE dbo.Empresa
SET
    RFC             = '20XXXXXXXXXX',        -- RUC Golocaer S.A.C. (11 dígitos)
    DireccionFiscal = 'PENDIENTE',           -- Domicilio fiscal Perú
    Ubigeo          = 'PENDIENTE'            -- Ubigeo SUNAT (6 dígitos)
WHERE Prefijo = 'GOLPERU';
```

### Orden de ejecución de scripts

| Paso | Script | BD | Dependencia |
|------|--------|----|-------------|
| 1 | INSERT EmpresaFolio GOLPERU | ProquifaDotNetTimbrado | Tabla existe (RE-FU-018) |
| 2 | INSERT AppSetting OSE Perú | ProquifaDotNetTimbrado | Tabla existe (RE-FU-018) |
| 3 | ALTER TABLE Producto ADD CodigoSUNAT | ProquifaDotNet | **BLOQUEANTE** |
| 4 | ALTER TABLE catUnidad ADD ClaveSUNAT | ProquifaDotNet | **BLOQUEANTE** |
| 5 | CREATE TABLE catAfectacionIGV | ProquifaDotNet | **BLOQUEANTE** |
| 6 | INSERT catAfectacionIGV datos iniciales | ProquifaDotNet | Requiere paso 5 |
| 7 | ALTER TABLE Producto ADD IdCatAfectacionIGV | ProquifaDotNet | Requiere paso 5 |
| 8 | UPDATE Empresa GOLPERU datos legales | ProquifaDotNet | **BRECHA — datos pendientes** |

---

## Gaps de Desarrollo

### En ProquifaDotNet.Timbrado

| # | Gap | Acción | Esfuerzo | Estado |
|---|-----|--------|----------|--------|
| GAP-01 | DTOs SUNAT: `StampSunatInvoiceRequestDto`, `SunatItemDto`, `SunatIssuerDataDto`, `SunatRecipientDataDto`, `StampSunatInvoiceResponseDto` | Modelos específicos UBL 2.1 con campos SUNAT | Medio | Abierto |
| GAP-02 | Infrastructure: `OseStampingClient` (interface + implementación HTTP hacia OSE/PSE SUNAT) | Cliente HTTP intercambiable; proveedor pendiente | Alto | **BLOQUEANTE (brecha OSE)** |
| GAP-03 | Application: ampliar `StampingService` con `StampSunatInvoiceAsync` (UBL 2.1, serie SUNAT, CDR) | Flujo completo: validar → folio → UBL 2.1 → OSE → persistir | Alto | **BLOQUEANTE (brecha OSE + datos SUNAT producto)** |
| GAP-04 | API: `StampingController` — sin endpoint nuevo (reutiliza `POST /api/v1/stamp/invoice` creado en RE-FU-018) | Ajuste de orquestación interna para SUNAT | Bajo | Abierto |

### En ProquifaDotNet.Finanzas

| # | Gap | Acción | Esfuerzo | Estado |
|---|-----|--------|----------|--------|
| GAP-05 | DTOs Peru: `ClientFiscalDataPeruDto`, `IssuerFiscalDataPeruDto`, `OperationTypeDto` | Modelos con campos RUC, DireccionFiscal, RegimenTributario, Ubigeo | Bajo | Abierto |
| GAP-06 | Infrastructure: adaptar `AdvanceInvoiceFiscalDataRepository` para Perú (RUC, dirección fiscal, Tipo Operación cat. 51, datos SUNAT producto) | Branch regional en métodos existentes o métodos nuevos _Peru | Medio | **BLOQUEANTE (datos SUNAT producto)** |
| GAP-07 | Application: extender `AdvanceInvoiceGenerateService` con branch Perú (UBL 2.1, IGV, sin PPD/99, Tipo Operación, llama `ICfdiService.GenerateAsync`) | Branch interno RegionClave='PER', alta complejidad | Alto | **BLOQUEANTE (datos SUNAT producto + OSE)** |
| GAP-08 | Application: extender `AdvanceInvoicePreviewService` con template PDF GOLPERU (DocumentBuilder) | Branch regional para template PDF Perú | Medio | Abierto |
| GAP-09 | Infrastructure: `ApiCallerStamping` (existente, RE-FU-018) agregar método `StampSunatInvoiceAsync` | Nuevo método HTTP hacia `POST /api/v1/stamp/invoice` | Bajo | Abierto |

### En Base de Datos

| # | Gap | Acción | BD | Esfuerzo | Estado |
|---|-----|--------|----|----------|--------|
| GAP-10 | INSERT EmpresaFolio GOLPERU con serie SUNAT | DML — serie pendiente confirmar | ProquifaDotNetTimbrado | Bajo | Brecha (serie pendiente) |
| GAP-11 | INSERT AppSetting config OSE/PSE Perú | DML — endpoint/credenciales pendientes | ProquifaDotNetTimbrado | Bajo | Brecha (proveedor pendiente) |
| GAP-12 | ALTER TABLE Producto + catUnidad + CREATE TABLE catAfectacionIGV | DDL bloqueante | ProquifaDotNet | Medio | **BLOQUEANTE** |
| GAP-13 | INSERT catAfectacionIGV datos iniciales (catálogo 7 SUNAT) | DML | ProquifaDotNet | Bajo | Requiere GAP-12 |
| GAP-14 | UPDATE Empresa GOLPERU (RUC, DireccionFiscal, Ubigeo) | DML — datos pendientes | ProquifaDotNet | Bajo | Brecha (datos legales pendientes) |

---

## Diagrama de Flujo — Timbrado SUNAT Perú

```
[Finanzas]                                  [Timbrado]                      [OSE/PSE SUNAT]
     |                                           |                                 |
     | POST /api/v1/stamp/invoice (tecnico)              |                                 |
     |------------------------------------------>|                                 |
     |                                           | 1. Validar request              |
     |                                           | 2. Consumir folio GOLPERU       |
     |                                           |    (F001-00000001)               |
     |                                           | 3. Armar XML UBL 2.1            |
     |                                           |    + firma digital               |
     |                                           | 4. INSERT StampingLog (Pending) |
     |                                           |                                 |
     |                                           | Enviar XML firmado              |
     |                                           |-------------------------------->|
     |                                           |                                 |
     |                                           |    CDR (aceptado/rechazado)     |
     |                                           |<--------------------------------|
     |                                           |                                 |
     |                                           | 5. UPDATE StampingLog (Stamped) |
     |    StampSunatInvoiceResponseDto          |                                 |
     |<------------------------------------------|                                 |
     |                                           |                                 |
     | 6. CfdiService (Finanzas): INSERT CFDIGenerada +                           |
     |    Archivo (XML+CDR, bucket 'facturas-peru')                               |
```

> Nota: la persistencia del CPE (`CFDIGenerada` + `Archivo`) y la subida a Minio ya no ocurren del lado de Timbrado — Timbrado solo audita la llamada (`StampingLog`). Igual que en México (RE-FU-019), es `CfdiService` (Finanzas) quien persiste el resultado.

---

## Tablas Consultadas / Modificadas

### ProquifaDotNet (Lectura)

| Tabla | Uso | Diferencia vs MEX |
|-------|-----|-------------------|
| `vfccFactura` (RE-FU-015, reemplaza vtpProformaAdelanto) | Listado detalle PER (filtro RegionClave='PER') | Igual, ya incluye Region |
| `DatosFacturacionCliente` | RUC (11 dígitos), RazonSocial, DireccionFiscal | RUC vs RFC |
| `Empresa` (GOLPERU) | RUC emisor, RazonSocial, DireccionFiscal, Ubigeo | Solo GOLPERU; campos nuevos |
| `catCondicionesDePago` | Contado/Crédito | Sin PPD/99 |
| `catMoneda` | PEN/USD | PEN vs MXN |
| `Producto.CodigoSUNAT` | Código SUNAT del producto | **BRECHA — no existe** |
| `catUnidad.ClaveSUNAT` | Unidad de medida SUNAT | **BRECHA — no existe** |
| `catAfectacionIGV` | Afectación IGV por línea | **BRECHA — tabla no existe** |
| `ClienteCarteraCliente + ClienteCartera` | Filtro cartera + ESAC (CC email) | Igual |
| `Contacto` | Nombre, correo, teléfono | Igual |

### ProquifaDotNet (Escritura)

| Tabla | Operación | Diferencia vs MEX |
|-------|-----------|-------------------|
| `CFDIGenerada` | INSERT al timbrar exitosamente (NumeroCPE/UUID, CDR, FechaEmision, Total, catalogos, Estado) — propiedad de Finanzas | Mismos campos base que MEX; sin IdCatMetodoDePagoCFDI/IdCatUsoCFDI (no aplican SUNAT) |
| `fccFactura` (reemplaza tpProformaAdelanto) | UPDATE SET IdCFDIGenerada (Id real de CFDIGenerada), EsFacturaPorAdelantado=0; UPDATE SET Enviada=1 | Igual |
| `Archivo` | INSERT x2 (PDF + XML/CDR) + vínculo en CFDIGenerada.IdArchivoXml | FileBucket='facturas-peru' |
| `CorreoEnviado` | INSERT registro envío | Igual |
| `ArchivoCorreoEnviado` | INSERT vínculo archivo-correo | Igual |

### ProquifaDotNetTimbrado (solo técnico, sin tabla de negocio del CFDI)

| Tabla | Operación | Diferencia vs MEX |
|-------|-----------|-------------------|
| `EmpresaFolio` | UPDATE atómico GOLPERU (F001 serie) | FormatoFolio distinto |
| `StampingLog` | INSERT auditoría técnica (`CfdiGeneradaId` es referencia informativa, no FK) | Igual |

---

## Consideraciones Técnicas

| Tema | Detalle |
|------|---------|
| Firma digital XML | El XML UBL 2.1 debe firmarse con certificado digital de Golocaer S.A.C. (PFX/PEM) antes de enviarse al OSE |
| CDR de aceptación | La respuesta exitosa del OSE incluye el CDR (ZIP con XML de respuesta SUNAT); debe almacenarse junto al XML timbrado |
| IGV por línea | Cada concepto del CPE debe incluir BaseImponible, MontoIGV, PorcentajeIGV (18%) y código de afectación del catálogo 7 SUNAT |
| Tipo de Operación | Catálogo 51 SUNAT; pendiente definir si es fijo "0101" o seleccionable por el operador (Regla 7 del requisito) |
| Sin Método/Forma de Pago | El modal de Generación Perú NO muestra estos campos; el DTO no los incluye |
| Sin Complemento de Pago | El CPE peruano se emite completo con IGV; no existe el esquema PPD + Complemento del SAT |
| Branch regional en Finanzas | `AdvanceInvoiceGenerateService` detecta RegionClave y arma el request correspondiente; ambos casos llaman a `ICfdiService.GenerateAsync`, que internamente invoca el mismo endpoint técnico de facturas `POST /api/v1/stamp/invoice` en Timbrado y persiste `CFDIGenerada` (resuelve el tipo con `catTipoCFDI`, sin discriminador propio de Timbrado) |
| Template PDF Perú | DocumentBuilder template `GOLPERU_PER_FAA` independiente del template México; definir en requisito independiente |
| Fuente Tipo de Cambio Perú | La fuente oficial para TC en Perú (SBS, SUNAT, BCR) está pendiente de definir (no aplica el DOF mexicano) |
| Interoperabilidad OSE | El `OseStampingClient` debe implementar via interface para facilitar cambio de proveedor OSE/PSE sin modificar la capa Application |

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| R16A-RE-FU-018 | Prerequisito: ProquifaDotNet.Timbrado creada (EmpresaFolio, CFDI, StampingLog) |
| R16A-RE-FU-019 | Prerequisito: flujo FAA Detalle México implementado (reutilizar servicios con branch regional) |
| R16A-RE-FU-005 | Brechas Perú: Brecha 1 (datos SUNAT producto — BLOQUEANTE) + Brecha 5 (OSE/PSE SUNAT — BLOQUEANTE) |
| R16A-RE-FU-012 | Define que Crédito Perú no usa Factura por Adelantado |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura`/`vfccFactura` (columna `Enviada` y vista reutilizadas por este requisito) — genera pendiente FAA desde tramitación Prepago Perú |
| R16A-RE-FU-022 | PDF de la factura Perú (template DocumentBuilder GOLPERU_PER_FAA) |

---

## Resumen de Gaps

| Repositorio | Cantidad | Detalle |
|-------------|----------|---------|
| ProquifaDotNet.Timbrado | 4 (GAP-01 a GAP-04) | DTOs SUNAT + OseStampingClient + StampingService + endpoint |
| ProquifaDotNet.Finanzas | 5 (GAP-05 a GAP-09) | DTOs Perú + AdvanceInvoiceFiscalDataRepository + AdvanceInvoiceGenerateService + AdvanceInvoicePreviewService + ApiCallerStamping |
| Base de Datos | 5 (GAP-10 a GAP-14) | EmpresaFolio GOLPERU + AppSetting OSE + datos SUNAT producto (BLOQUEANTE) + catAfectacionIGV + datos legales GOLPERU |
| **Total** | **14 gaps** | 3 bloqueantes por brechas Perú |
