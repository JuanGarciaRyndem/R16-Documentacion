# Impacto en Back — R16A-RE-FU-030
**Requisito:** Diseño y generación de Documentos: CDP México — Complemento de Pago (CFDI tipo P / Pagos20 v2.0)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder
**Módulo:** Validar Cobro — Wizard Paso 3 (México) — Complemento de Pago
**Impacto:** Scripts BD ProquifaDotNet (1 catálogo nuevo + 1 DML catálogo + 1 ALTER tabla + 1 ALTER vista) + Scripts BD ProquifaDotNetTimbrado (1 DML EmpresaFolio) + Scripts DocumentBuilder (4 DML DocumentTemplate) + Servicios Finanzas: construcción XML Pagos20 v2.0 con cálculo de parcialidades y saldos, previsualización PDF CP, persistencia post-timbrado en MinIO bucket `cobranza` (MEX) + Timbrado: generación CFDI P vía PAC TurboPac + DocumentBuilder: 4 templates PDF Complemento de Pago México. **Complementa la cascada PPD de RE-028 (Escenarios B y D). Sin ETL a Legacy — los mismos pasos post-envío de RE-028 aplican.**

---

## Resumen

RE-030 implementa la generación del **Complemento de Pago (CFDI tipo P, Pagos20 v2.0)** para México, completando los pasos que RE-028 dejó explícitamente pendientes en su sección B4:

- **Escenario B (cascada PPD)** — pasos 4–6: timbrado del CP inmediatamente después de la Factura PPD, generación y persistencia del PDF CP.
- **Escenario D (COMPLEMENTO_PAGO desde FAA existente)** — pasos 2–3: generación y persistencia del PDF CP para el complemento de una Factura por Adelantado.

No hay pantalla ni módulo independiente. El único disparador es la confirmación del timbrado de la Factura PPD (o la FAA) en el Paso 3 de Validar Cobro. El Complemento de Pago se genera automáticamente en cascada, sin interacción adicional del usuario entre la Factura y el CP.

La política operativa de R16 es **1 CP por factura**: si un cobro cubre N facturas PPD, el sistema genera N Complementos de Pago independientes, cada uno con un único Pago y un único DoctoRelacionado.

### Distribución de responsabilidades

| Capa                 | Aplicativo                | Responsabilidad                                                                                                                         |
| -------------------- | ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| BD — Catálogo nuevo  | ProquifaDotNet            | `catFormaPagoSAT` (c_FormaPago SAT)                                                                                                     |
| BD — DML catálogo    | ProquifaDotNet            | `catUsoCFDI`: INSERT clave CP01                                                                                                         |
| BD — ALTER tabla     | ProquifaDotNet            | `fccDocumentoFiscalCobro`: ADD 8 columnas snapshot DR del CP                                                                            |
| BD — ALTER vista     | ProquifaDotNet            | `vfccDocumentoFiscalCobro` v3.0: exponer columnas DR + JOIN catFormaPagoSAT                                                             |
| BD — DML foliador    | ProquifaDotNetTimbrado    | `EmpresaFolio`: INSERT 4 filas Serie "P" (GOL, MUN, PRO, PQF)                                                                           |
| BD — DML templates   | DocumentBuilder           | `DocumentTemplate`: INSERT 4 templates PDF CP México                                                                                    |
| XML Pagos20          | ProquifaDotNet.Finanzas   | Construcción del `PaymentComplementRequest`: cálculo NumParcialidad, ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto, EquivalenciaDR, Totales    |
| Previsualización PDF | ProquifaDotNet.Finanzas   | `PaymentComplementPdfMappingService.MapearPreviewAsync()` sin sello digital                                                               |
| Timbrado CP          | ProquifaDotNet.Timbrado   | Generación XML CFDI P + PAC TurboPac + `INSERT CFDIGenerada` + `UPDATE EmpresaFolio` Serie P                                            |
| Persistencia PDF CP  | ProquifaDotNet.Finanzas   | `PersistPaymentComplementPdfService.PersistirAsync()` → DocumentBuilder → MinIO → `INSERT Archivo` → `UPDATE CFDIGenerada.IdArchivoPdf` |
| Snapshot DR          | ProquifaDotNet.Finanzas   | `UPDATE fccDocumentoFiscalCobro` con 8 columnas DR tras timbrado exitoso                                                                |
| Envío                | ProquifaDotNet.Finanzas   | Adjuntos CP (PDF + XML) en modal de envío del Paso 3 (RE-028 B6)                                                                        |
| Comunicación         | Finanzas → Timbrado       | Llamada API por CP a timbrar (igual que Factura)                                                                                        |
| Comunicación         | Finanzas → ProquifaDotNet | Lecturas: catFormaPagoSAT, CFDIGenerada, DatosFacturacionCliente, Empresa                                                               |

### Infraestructura reutilizada

| Componente | Origen | Reutilización en RE-030 |
|---|---|---|
| `fccDocumentoFiscalCobro` (estructura base) | RE-028 | Se extiende con 8 columnas DR; ciclo PENDIENTE→GENERADO→ENVIADO sin cambios |
| `CFDIGenerada` | RE-019/028 | Se inserta una fila por CP con `IdCatTipoCFDI='COMPLEMENTO_PAGO'`, `IdCFDIRelacionado` = UUID de la Factura PPD o FAA relacionada |
| `catTipoCFDI.COMPLEMENTO_PAGO` | RE-028 T1 | Discriminador del tipo de CFDI; sin cambios |
| `fccDocumentoFiscalCobro.IdCFDIGeneradaComplemento` | RE-028 | Se puebla con el `IdCFDIGenerada` del CP timbrado |
| `CFDIGenerada.IdCFDIRelacionado` | RE-028 | FK blanda al UUID de la Factura PPD/FAA que este CP complementa |
| `EmpresaFolio` (estructura) | RE-019 | Foliador atómico con UPDLOCK; RE-030 agrega filas Serie "P" |
| PAC TurboPac | RE-019 | Mismo cliente/servicio para timbrar el CP |
| `catUsoCFDI` | RE-028 o anterior | CP01 se inserta en el cambio BD #2; se lee al armar Receptor |
| `catMetodoDePagoCFDI.PPD` | RE-028 T1 | Solo facturas PPD generan CP; discriminador en la cascada |
| `CorreoEnviado` / `ArchivoCorreoEnviado` | RE-028 | Registro del correo enviado con PDF + XML del CP adjuntos |
| `ApiCallerStamping` (HttpClient + Polly) | RE-019 | Cliente HTTP con retry policy hacia Timbrado — se usa `StampPaymentComplementAsync` (`POST /api/v1/stamp/payment-complement`), método creado en RE-019/018 |
| `MexicoInvoicePdfMappingService` | RE-021 | Patrón de referencia para `PaymentComplementPdfMappingService` |
| `PersistMexicoInvoicePdfService` | RE-021 | Patrón de referencia para `PersistPaymentComplementPdfService` |
| Templates `GOL/MUN/PRO/PQF_MEX_FAC` | RE-021 | Referencia de branding e identidad visual para diseño de templates CP |
| `DatosFacturacionCliente` | RE-004 | RFC, Razón Social, RegimenFiscalReceptor del CFDI 4.0 |
| `Empresa` | Existente | RFC Emisor, RegimenFiscal, Prefijo por empresa PROQUIFA México |
| `RegionConfiguracionMinioBucket` bucket `cobranza` (MEX) | Existente | Almacenamiento PDF y XML del CP en MinIO; bucket ya registrado para México |

---

## Parte A — Base de Datos (ProquifaDotNet + ProquifaDotNetTimbrado + DocumentBuilder)

### A1 — CREATE TABLE catFormaPagoSAT

Catálogo c_FormaPago SAT. Almacena las formas reales de cobro para el campo `FormaDePagoP` del nodo Pago del Complemento de Pago. Confirmado inexistente en ProquifaDotNet.

| Clave representativa | Descripción |
|---|---|
| `01` | Efectivo |
| `02` | Cheque nominativo |
| `03` | Transferencia electrónica de fondos |
| `04` | Tarjeta de crédito |
| `28` | Tarjeta de débito |
| *(otros)* | 22 claves iniciales; completar con catálogo SAT vigente |

> Ver script completo en `R16A-RE-FU-030_BD.md` — Catálogo: catFormaPagoSAT.

### A2 — DML catUsoCFDI — INSERT clave CP01

La clave `CP01` (Pagos) es obligatoria en el nodo Receptor del CP (Regla 6 / Criterio C3). Confirmado que no existe en la tabla actual.

> Ver script completo en `R16A-RE-FU-030_BD.md` — DML catUsoCFDI.

### A3 — ALTER TABLE fccDocumentoFiscalCobro — Columnas DR del CP

Se agregan 8 columnas nullable que almacenan el **snapshot fiscal inmutable** del DoctoRelacionado y del nodo Pago en el momento del timbrado:

| Columna | Tipo | Campo XML |
|---|---|---|
| `FechaPagoCP` | datetime2(7) | `FechaPago` del nodo Pago |
| `IdCatFormaPagoSAT` | uniqueidentifier FK | `FormaDePagoP` (FK → catFormaPagoSAT) |
| `TipoCambioP_CP` | decimal(18,6) | `TipoCambioP` cuando MonedaP ≠ MXN |
| `NumParcialidad` | int | `NumParcialidad` del DoctoRelacionado |
| `ImpSaldoAnt` | decimal(18,6) | `ImpSaldoAnt` en MonedaDR |
| `ImpPagado` | decimal(18,6) | `ImpPagado` en MonedaDR |
| `ImpSaldoInsoluto` | decimal(18,6) | `ImpSaldoInsoluto` = ImpSaldoAnt − ImpPagado |
| `EquivalenciaDR` | decimal(18,6) | `EquivalenciaDR` (1 si MonedaDR = MonedaP) |

Son NULL para líneas no-CP (FACTURA, FACTURA_ANTICIPO, líneas Perú).

> Ver script completo en `R16A-RE-FU-030_BD.md` — ALTER TABLE fccDocumentoFiscalCobro.

### A4 — DML EmpresaFolio — Serie "P" para las 4 empresas México

Se insertan 4 filas en `ProquifaDotNetTimbrado.EmpresaFolio` (estructura creada en RE-019) con Serie "P" para las empresas GOL, MUN, PRO, PQF. Mismo patrón UPDLOCK que el foliador de Facturas.

> ⚠️ **Pendiente P2:** Formato definitivo de la Serie "P" y `FormatoFolio` pendiente de validar con PMO (Regla 12 del requisito). Valores del INSERT son propuesta inicial.

> Ver script completo en `R16A-RE-FU-030_BD.md` — DML EmpresaFolio.

### A5 — DML DocumentTemplate — 4 templates CP México

Se registran en `DocumentBuilder.DocumentTemplate` los 4 templates PDF del Complemento de Pago, uno por empresa emisora. Convención confirmada: `{TemplateKey}_{H/B/F}.html` con Header, Body y Footer.

| TemplateKey | Empresa emisora |
|---|---|
| `GOL_MEX_CP` | Golocaer |
| `MUN_MEX_CP` | Mungen |
| `PRO_MEX_CP` | Proquifa |
| `PQF_MEX_CP` | Proveedora Quimico Farmaceutica |

> Ver script completo en `R16A-RE-FU-030_BD.md` — DML DocumentTemplate.

### A6 — ALTER VIEW vfccDocumentoFiscalCobro v3.0

La vista se extiende incrementalmente sobre v2.0 (RE-029) para exponer las 8 columnas DR del CP y agregar `LEFT JOIN catFormaPagoSAT` para resolver `FormaPagoClave` y `FormaPagoDescripcion`.

> Ver script completo en `R16A-RE-FU-030_BD.md` — ALTER VIEW vfccDocumentoFiscalCobro v3.0.

---

## Parte B — ProquifaDotNet.Finanzas: Servicios y Endpoints

### B1 — Previsualización PDF del Complemento de Pago

**Descripción:** Al presionar "Previsualizar" en una línea con `IdCatTipoDocumentoFiscal = COMPLEMENTO_PAGO`, Finanzas genera el PDF representativo en memoria sin timbrar ni persistir. Completa el stub que RE-028 B3 (paso 3) dejó pendiente.

**Flujo:**
1. Finanzas lee `vfccDocumentoFiscalCobro` para la línea.
2. Calcula el `NumParcialidad` estimado (sin UPDLOCK — solo informativo para preview):
   ```sql
   SELECT COUNT(*) + 1
   FROM CFDIGenerada
   WHERE IdCFDIRelacionado = @IdCFDIFacturaPPD
     AND IdCatTipoCFDI = 'COMPLEMENTO_PAGO'
   ```
3. Calcula `ImpSaldoAnt` estimado (ver sección B2 para la lógica completa).
4. Invoca `PaymentComplementPdfMappingService.MapearPreviewAsync()` — consolida datos en `PaymentComplementPdfModel` sin `TimbreFiscalDigital`, resuelve `TemplateKey` dinámicamente (`GOL/MUN/PRO/PQF_MEX_CP`).
5. Genera PDF en memoria vía DocumentBuilder.
6. Retorna el PDF al frontend para el modal de previsualización.
7. Sin escrituras en BD.

**Nota:** La previsualización no incluye UUID, sello digital ni QR, que solo existen tras el timbrado.

### B2 — Cálculo de valores fiscales del DoctoRelacionado

**Descripción:** Antes de solicitar el timbrado del CP, Finanzas calcula y arma el `PaymentComplementRequest`. Este cálculo es la lógica central de RE-030.

#### Cálculo de NumParcialidad

```sql
-- Ejecutar con UPDLOCK en la misma transacción que el timbrado
SELECT COUNT(*) + 1 AS NumParcialidad
FROM dbo.CFDIGenerada WITH (UPDLOCK)
WHERE IdCFDIRelacionado = @IdCFDIFacturaPPD
  AND IdCatTipoCFDI = (SELECT IdCatTipoCFDI FROM catTipoCFDI WHERE Clave = 'COMPLEMENTO_PAGO')
```

El UPDLOCK evita que dos cobros concurrentes sobre la misma factura PPD asignen el mismo número de parcialidad.

#### Cálculo de ImpSaldoAnt

| Escenario | Fuente de ImpSaldoAnt |
|---|---|
| Primer CP para esta factura PPD (`NumParcialidad = 1`) | Total de la Factura PPD (`CFDIGenerada.Total` o equivalente) |
| CPs subsecuentes (`NumParcialidad > 1`) | `ImpSaldoInsoluto` del CP anterior: `MAX(fccDocumentoFiscalCobro.ImpSaldoInsoluto) WHERE IdCFDIGeneradaComplemento.IdCFDIRelacionado = @IdCFDIFacturaPPD` |

#### Cálculo de ImpSaldoInsoluto

```
ImpSaldoInsoluto = ImpSaldoAnt - ImpPagado
```

`ImpPagado` es la porción del cobro aplicada a esta factura específica (Regla 8 del requisito: `Monto del nodo Pago = ImpPagado del DoctoRelacionado`).

#### EquivalenciaDR

```
Si MonedaDR = MonedaP → EquivalenciaDR = 1  (obligatorio en XML aunque monedas sean iguales)
Si MonedaDR ≠ MonedaP → EquivalenciaDR = factor de conversión del día del cobro
```

#### TipoCambioP

```
Si MonedaP = MXN  → omitir TipoCambioP del nodo Pago
Si MonedaP ≠ MXN  → TipoCambioP = TC del día del cobro vs MXN
```

#### FechaPago

La fecha del cobro confirmado en el Paso 2. ⚠️ **Pendiente P1:** convención de hora (12:00:00 fija vs hora real del cobro) pendiente de validar con asesor fiscal.

#### FormaDePagoP

Resolución desde `catFormaPagoSAT` según la forma de cobro registrada en el Paso 2. Debe ser la forma real (Regla 7: no usar 99).

### B3 — Cascada PPD: timbrado del Complemento de Pago (Escenario B de RE-028)

**Descripción:** Implementa los pasos 4–6 del Escenario B de RE-028 B4. Inmediatamente después de que la Factura PPD es timbrada exitosamente, Finanzas solicita el timbrado del CP.

Este flujo completa la cascada para la línea `FACTURA` con `MetodoPago = PPD`:

**Paso 4 — Solicitar timbrado del CP:**

Finanzas calcula los valores del DR (sección B2) y envía a Timbrado:

```
PaymentComplementRequest {
  IdCFDIFacturaPPD,         // UUID de la Factura PPD recién timbrada
  EmisorPrefijo,            // GOL / MUN / PRO / PQF
  ReceptorRFC,              // DatosFacturacionCliente.RFC
  ReceptorNombre,
  ReceptorDomicilioFiscal,
  ReceptorRegimenFiscal,
  FechaPago,                // fecha del cobro ⚠️ hora pendiente
  IdCatFormaPagoSAT,        // forma real del cobro
  MonedaP,
  TipoCambioP,              // NULL si MXN
  ImpPagado,                // monto aplicado a esta factura
  // DoctoRelacionado
  UUIDFacturaPPD,
  SerieFacturaPPD,
  FolioFacturaPPD,
  MonedaDR,
  EquivalenciaDR,
  NumParcialidad,
  ImpSaldoAnt,
  ImpSaldoInsoluto,
  ObjetoImpDR,
  ImpuestosDR               // cuando ObjetoImpDR = 02
}
```

**Paso 5 — Persistencia PDF CP (post-timbrado exitoso):**
1. Finanzas invoca `PersistPaymentComplementPdfService.PersistirAsync(IdCFDIGeneradaCP, xmlTimbradoCP)`.
2. Servicio invoca `PaymentComplementPdfMappingService.MapearAsync()` — consolida datos con `TimbreFiscalDigital`, resuelve `TemplateKey` = `{Prefijo}_MEX_CP`.
3. Genera PDF definitivo vía DocumentBuilder.
4. Resuelve bucket vía `RegionConfiguracionMinioBucket` (BucketClave=`cobranza`, Región=MEX).
5. Sube PDF a MinIO bucket `cobranza`, ruta `cobranza/complementos/{anio}/{mes}/{UUID_CP}.pdf` → `INSERT Archivo` → `UPDATE CFDIGenerada SET IdArchivoPdf = @IdArchivo`.
6. Sube XML a MinIO bucket `cobranza`, ruta `cobranza/complementos/{anio}/{mes}/{UUID_CP}.xml` → `INSERT Archivo` (ver Parte E para flujo detallado).

**Paso 6 — Actualización de fccDocumentoFiscalCobro:**

```sql
UPDATE dbo.fccDocumentoFiscalCobro
SET EstadoLinea                = 'GENERADO',
    IdCFDIGeneradaComplemento  = @IdCFDIGeneradaCP,
    FechaGeneracion            = SYSUTCDATETIME(),
    -- Snapshot DR inmutable
    FechaPagoCP                = @FechaPago,
    IdCatFormaPagoSAT          = @IdCatFormaPagoSAT,
    TipoCambioP_CP             = @TipoCambioP,
    NumParcialidad             = @NumParcialidad,
    ImpSaldoAnt                = @ImpSaldoAnt,
    ImpPagado                  = @ImpPagado,
    ImpSaldoInsoluto           = @ImpSaldoInsoluto,
    EquivalenciaDR             = @EquivalenciaDR
WHERE IdFCCDocumentoFiscalCobro = @Id
```

**Política de fallo del CP tras Factura PPD timbrada:**
Si el timbrado del CP falla, la Factura PPD permanece vigente con UUID. La línea queda en estado `PENDIENTE` para reintento del CP en el siguiente acceso al Paso 3. ⚠️ **Pendiente:** Política formal de reintento pendiente (Requisito 030 No aplica — brecha transversal con Factura Anticipo y NC).

### B4 — COMPLEMENTO_PAGO desde FAA existente (Escenario D de RE-028)

**Descripción:** Implementa los pasos 2–3 del Escenario D de RE-028 B4. Para líneas donde el origen es una Factura por Adelantado ya existente, el CP ya se timbra en RE-028 (1 CFDI). RE-030 agrega la generación y persistencia del PDF.

**Diferencias respecto al Escenario B:**
- No hay Factura PPD nueva: el `IdCFDIRelacionado` del CP apunta al UUID de la FAA (`fccFactura.IdCFDIGenerada`, RE-FU-015 — antes `tpProformaAdelanto.IdCFDIGenerada`).
- `NumParcialidad = 1` (política R16: 1 pago por FAA; pago único).
- `ImpSaldoAnt` = Total de la FAA.
- `ImpPagado` = monto del cobro aplicado.
- `ImpSaldoInsoluto = ImpSaldoAnt - ImpPagado`.

**Post-timbrado:** mismo flujo de persistencia PDF e `UPDATE fccDocumentoFiscalCobro` que el Escenario B (pasos 5–6 de B3).

### B5 — Adjuntos del CP en el Modal de Envío

El modal de envío del Paso 3 (RE-028 B6) ya contempla los adjuntos del CP. RE-030 provee los PDFs y XMLs del CP para que el modal pueda incluirlos:

| Adjunto | Para líneas FACTURA PPD + cascada | Para líneas COMPLEMENTO_PAGO desde FAA |
|---|---|---|
| PDF Factura PPD | ✓ (RE-028) | — |
| XML Factura PPD | ✓ (RE-028) | — |
| PDF Complemento de Pago | ✓ **(RE-030)** | ✓ **(RE-030)** |
| XML Complemento de Pago | ✓ (RE-028 Timbrado) | ✓ (RE-028 Timbrado) |
| PDF Confirmación de Pedido | ✓ (RE-028 B7.2) | ✓ (RE-028 B7.2) |

El PDF del CP se obtiene de `MinIO` a través de `CFDIGenerada.IdArchivoPdf` → `Archivo.RutaArchivo`.

### B6 — PaymentComplementPdfMappingService

**Descripción:** Servicio de Finanzas responsable de mapear los datos del Complemento de Pago timbrado a un modelo `PaymentComplementPdfModel` para la generación del PDF en DocumentBuilder. Sigue el patrón de `MexicoInvoicePdfMappingService` (RE-021).

**Datos de entrada:**
- `CFDIGenerada` (UUID, Serie, Folio, FechaEmision, TimbreFiscalDigital)
- `fccDocumentoFiscalCobro` (snapshot DR: NumParcialidad, ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto, EquivalenciaDR, FechaPagoCP, IdCatFormaPagoSAT, TipoCambioP_CP)
- `catFormaPagoSAT` (Clave, Descripcion para FormaDePagoP)
- `DatosFacturacionCliente` (datos del receptor)
- `Empresa` (datos del emisor + Prefijo para resolver TemplateKey)
- `CFDIGenerada` de la Factura relacionada (UUID, Serie, Folio de la factura PPD/FAA para DoctoRelacionado)

**Resolución de TemplateKey:** `{Prefijo}_MEX_CP` → `GOL_MEX_CP` / `MUN_MEX_CP` / `PRO_MEX_CP` / `PQF_MEX_CP`

**Modos:**
- `MapearPreviewAsync()`: sin `TimbreFiscalDigital`; `NumParcialidad` estimado; sin QR.
- `MapearAsync()`: con `TimbreFiscalDigital`, QR de verificación SAT codificado.

**Secciones del `PaymentComplementPdfModel`** (criterios I4–I14 del requisito):

| Sección PDF | Campos |
|---|---|
| Encabezado (Emisor) | Razón Social, RFC, LugarExpedición, FechaHoraExpedición |
| Receptor | Razón Social, RFC, DomicilioFiscal, UsoCFDI (CP01) |
| Comprobante | Serie, Folio, Versión CFDI (4.0), UUID, FechaCertificación, TipoComprobante (P-Pago), RegimenFiscal emisor |
| Concepto fijo | ClaveProdServ 84111506, Cantidad 1, ClaveUnidad ACT, Descripción "Pago", ValorUnitario 0.00, Importe 0.00, Subtotal 0.00, Total 0.00, Moneda XXX, Total en letra "CERO XXX 00/100" |
| Totales Complemento | MontoTotalPagos, TotalTrasladosBaseIVA16 (si aplica), TotalTrasladosImpuestoIVA16 (si aplica) |
| Nodo Pago | Versión 2.0, FechaPago, FormaDePagoP (Clave + Descripción), MonedaP, Monto, TipoCambioP (si aplica) |
| DoctoRelacionado | UUID, Serie, Folio factura relacionada, MonedaDR, EquivalenciaDR, NumParcialidad, ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto, MetodoPago (PPD), TipoCambioDR (si aplica) |
| Impuestos DR | Base DR, Impuesto DR (002), TipoFactor (Tasa), TasaOCuota (0.160000), Importe DR — solo si ObjetoImpDR=02 |
| Traslados resumen pago | Base P, Impuesto P, TipoFactor P, Tasa P, Importe P — solo si aplica IVA |
| Sello y trazabilidad | NoCertificadoSAT, NoCertificadoCSD, SelloDigitalSAT, SelloDigitalCFDI, CadenaOriginal |
| QR verificación | URL SAT: UUID + RFC emisor + RFC receptor + Total + últimos 8 chars sello |

---

## Parte C — ProquifaDotNet.Timbrado

### C1 — Endpoint timbrado Complemento de Pago (`POST /api/v1/stamp/payment-complement`)

Timbrado recibe la solicitud del CP desde Finanzas (sección B3) en el endpoint `POST /api/v1/stamp/payment-complement` (RE-018) y ejecuta el mismo pipeline interno que para Facturas, pero construyendo el XML según la especificación CFDI 4.0 Pagos20 v2.0.

**Por cada timbrado exitoso:**
1. Arma el XML CFDI P con estructura Pagos20:
   - Cabecera: `Version=4.0`, `TipoDeComprobante=P`, `Exportacion=01`, `SubTotal=0`, `Total=0`, `Moneda=XXX`, `Serie=P`, `Folio={ConsecutivoEmpresa}`, `LugarExpedicion={CPEmisor}`
   - Emisor: RFC, Nombre, RegimenFiscal (601) de la empresa
   - Receptor: RFC, Nombre, DomicilioFiscalReceptor, RegimenFiscalReceptor, UsoCFDI=CP01
   - Conceptos: único nodo fijo (ClaveProdServ=84111506, Cantidad=1, ClaveUnidad=ACT, Descripcion=Pago, ValorUnitario=0, Importe=0, ObjetoImp=01)
   - Complemento/Pagos20: Totales + 1 Pago con 1 DoctoRelacionado (+ ImpuestosDR si ObjetoImpDR=02)
2. Consume folio con UPDLOCK:
   ```sql
   UPDATE dbo.EmpresaFolio WITH (UPDLOCK)
   SET UltimoFolio = UltimoFolio + 1
   OUTPUT inserted.UltimoFolio
   WHERE IdEmpresa = @IdEmpresa AND Serie = 'P'
   ```
3. Llama al PAC TurboPac con el XML firmado con el CSD de la empresa emisora.
4. Recibe respuesta del PAC con UUID, FechaTimbre, SelloSAT, NoCertificadoSAT, CadenaOriginal.
5. `INSERT CFDIGenerada`:
   - `IdCatTipoCFDI` → `COMPLEMENTO_PAGO`
   - `IdCFDIRelacionado` = `IdCFDIGenerada` de la Factura PPD/FAA relacionada
   - `UUID`, `Serie = 'P'`, `Folio`, `FechaEmision`
6. `INSERT StampingLog` (trazabilidad de la petición al PAC).
7. Retorna a Finanzas: `UUID`, `Serie`, `Folio`, `FechaTimbre`, `xmlTimbrado` (con `TimbreFiscalDigital` sellado).

**Manejo de errores PAC:** Si el PAC rechaza el XML (código de error PAC, esquema inválido, RFC no registrado), Timbrado retorna el error a Finanzas sin insertar en `CFDIGenerada`. Finanzas muestra el detalle al usuario para corrección y reintento.

> **Diferencia con Factura:** El CP no tiene partidas (`Concepto` fijo), no tiene `IVA` en la cabecera (`Impuestos` solo aparece en el nodo Pagos20 vía `ImpuestosDR` cuando aplica). La validación del SAT rechaza si `SubTotal ≠ 0` o `Total ≠ 0` en un CFDI tipo P.

---

## Parte D — DocumentBuilder

Los 4 templates del Complemento de Pago México son nuevos en este requisito. El diseño HTML (Header, Body, Footer) debe ser consistente con los templates de Factura México (`*_MEX_FAC`) en branding, tipografía e identidad visual por empresa emisora.

| TemplateKey | Archivos | Empresa emisora | Estado |
|---|---|---|---|
| `GOL_MEX_CP` | `GOL_MEX_CP_H.html`, `GOL_MEX_CP_B.html`, `GOL_MEX_CP_F.html` | Golocaer | **Nueva — definir en RE-030** |
| `MUN_MEX_CP` | `MUN_MEX_CP_H.html`, `MUN_MEX_CP_B.html`, `MUN_MEX_CP_F.html` | Mungen | **Nueva — definir en RE-030** |
| `PRO_MEX_CP` | `PRO_MEX_CP_H.html`, `PRO_MEX_CP_B.html`, `PRO_MEX_CP_F.html` | Proquifa | **Nueva — definir en RE-030** |
| `PQF_MEX_CP` | `PQF_MEX_CP_H.html`, `PQF_MEX_CP_B.html`, `PQF_MEX_CP_F.html` | Proveedora Quimico Farmaceutica | **Nueva — definir en RE-030** |

Los registros `DocumentTemplate` en `DocumentBuilder` se insertan como parte del impacto BD (sección A5).

**Secciones HTML del PDF** (basado en criterios I4–I14 del requisito):
- Header: logo empresa, datos fiscales del emisor, número de folio y UUID.
- Body: datos del receptor, sección Concepto fijo, Totales del Complemento, datos del Pago, sección DoctoRelacionado con saldos e impuestos desglosados (cuando ObjetoImpDR=02).
- Footer: sellos digitales, cadena original, QR de verificación SAT, iconografía de certificaciones del giro.

> El patrón de referencia para el diseño son los templates `GOL/MUN/PRO/PQF_MEX_FAC` (RE-021), que ya tienen resueltos el layout de sello, QR, paleta de colores e iconografía por empresa.

---

## Parte E — MinIO: Almacenamiento de archivos del CP

### E1 — Resolución del bucket por región

Finanzas resuelve el bucket de destino consultando `RegionConfiguracionMinioBucket` antes de subir cada archivo. Para el Complemento de Pago México:

| BucketClave | Región | Archivo |
|---|---|---|
| `cobranza` | MEX | PDF Complemento de Pago + XML CFDI P timbrado |

El bucket `cobranza` para México **ya existe** en `RegionConfiguracionMinioBucket` — no se requiere INSERT.

```csharp
// Finanzas — resolución del bucket al subir archivo del CP
var bucket = await _regionConfiguracionMinioBucketRepository
    .ObtenerBucketAsync(bucketClave: "cobranza", regionClave: "MEX");
// bucket.BucketNombre → "cobranza"
```

Equivale a:
```sql
SELECT BucketNombre
FROM dbo.RegionConfiguracionMinioBucket rcmb
INNER JOIN dbo.Region r ON rcmb.IdRegion = r.IdRegion
WHERE rcmb.BucketClave = 'cobranza'
  AND r.Clave           = 'MEX'
  AND rcmb.Activo       = 1
```

### E2 — Rutas de archivos en MinIO

| Archivo | Ruta | Descripción |
|---|---|---|
| PDF Complemento de Pago | `cobranza/complementos/{anio}/{mes}/{UUID_CP}.pdf` | PDF representativo generado por DocumentBuilder post-timbrado |
| XML CFDI P timbrado | `cobranza/complementos/{anio}/{mes}/{UUID_CP}.xml` | XML timbrado con TimbreFiscalDigital retornado por PAC TurboPac |

> ⚠️ Confirmar convención de ruta con el equipo para mantener consistencia con la estructura usada en el mismo bucket por RE-028 (Facturas PPD y sus XMLs).

### E3 — Flujo de persistencia en MinIO (PersistPaymentComplementPdfService)

```
1. Recibe: IdCFDIGeneradaCP + xmlTimbradoCP
2. Invoca PaymentComplementPdfMappingService.MapearAsync()
     → PaymentComplementPdfModel con TimbreFiscalDigital + QR
3. Invoca DocumentBuilder con TemplateKey = {Prefijo}_MEX_CP
     → PDF binario en memoria
4. Sube PDF a MinIO bucket "cobranza":
     ruta = cobranza/complementos/{anio}/{mes}/{UUID_CP}.pdf
5. INSERT dbo.Archivo (Nombre, Extension='.pdf', RutaArchivo=ruta, FechaRegistro)
6. UPDATE dbo.CFDIGenerada SET IdArchivoPdf = @IdArchivo
     WHERE IdCFDIGenerada = @IdCFDIGeneradaCP
7. Sube XML a MinIO bucket "cobranza":
     ruta = cobranza/complementos/{anio}/{mes}/{UUID_CP}.xml
8. INSERT dbo.Archivo (Nombre, Extension='.xml', RutaArchivo=ruta, FechaRegistro)
9. UPDATE dbo.CFDIGenerada SET IdArchivoXml = @IdArchivoXml  (si columna existe)
10. Registrar el guardado del Complemento de Pago en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo — Reglas al diseñar, regla 8)
```

### E4 — Adjuntos del correo desde MinIO

Al despachar el correo en el modal de envío, Finanzas recupera los archivos del CP desde MinIO por `CFDIGenerada.IdArchivoPdf` y el XML retornado por Timbrado:

```csharp
// Obtener PDF del CP desde MinIO para adjuntarlo al correo
var pdf = await _minioService.DescargarArchivoAsync(
    bucket: "cobranza",
    ruta: archivo.RutaArchivo
);
```

---

## Diferencias clave vs RE-028 / RE-029

| Aspecto | RE-028 (Paso 3 México base) | RE-029 (Paso 3 Perú) | RE-030 (CP México) |
|---|---|---|---|
| Alcance | Infraestructura Paso 3 + Factura/FA México | Paso 3 Perú completo | Extiende RE-028 con CP en cascada |
| Disparador | Avance al Paso 3 | Avance al Paso 3 | Timbrado exitoso de Factura PPD (cascada) o cobro vs FAA |
| CFDIs generados | 1 (PUE/PPD/FA) | 1 (CPE SUNAT) | 1 CP adicional en cascada PPD |
| Norma fiscal | CFDI 4.0 Ingreso | CPE UBL 2.1 SUNAT | CFDI 4.0 Pagos20 v2.0 |
| Catálogo 1 por línea | `catUsoCFDI` | `catTipoOperacionSUNAT` | `catFormaPagoSAT` (nuevo) |
| DoctoRelacionado | — | — | UUID factura PPD/FAA + saldos |
| Folio serie | Facturas (letras var.) | F001 SUNAT | Serie "P" (nuevo, pendiente PMO) |
| ETL Legacy | Sí (B7.3 RE-028) | No | No aplica — misma post-envío de RE-028 |
| Transferencia a Legacy | RE-028 B7.3 | — | El CP se incluye en la misma transferencia de la Factura PPD |

---

## Pendientes que impactan Back

| # | Pendiente | Impacto en implementación |
|---|---|---|
| P1 | Convención hora en `FechaPago` (12:00:00 fija vs hora real del cobro) | Determina cómo Finanzas construye `FechaPagoCP` al armar el `PaymentComplementRequest` |
| P2 | Formato Serie "P" en `EmpresaFolio` | `FormatoFolio` y `LongitudMaxima` en los INSERT de EmpresaFolio (sección A4) |
| P3 | Soporte de tasas de IVA distintas a 16% (8%, 0%) | Lógica de cálculo de `TrasladoDR` en `PaymentComplementRequest`; si solo se soporta 16%, se documenta restricción en el servicio |
| P4 | Plantilla cuerpo del correo de envío del CP (PMO #31 transversal) | Configuración de la plantilla en ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo) para el modal de envío del Paso 3 (RE-028 B6) |
| P5 | Confirmar con negocio si el reintento del CP bloquea el envío de la Factura o si se permite enviarla sin CP | Ownership y mecanismo ya definidos (Finanzas, local en este flujo, ver B3); falta solo la decisión de negocio sobre el bloqueo del envío |

---

## Brechas

> ⚠️ **BRECHA MEDIA — FechaPago: hora fija 12:00:00 vs hora real del cobro (B1)**
> Los ejemplos del sistema legacy usan hora 12:00:00 fija en FechaPago. Si el SAT requiere la hora real, hay riesgo fiscal. Pendiente validar con asesor fiscal PROQUIFA antes de implementar `PaymentComplementRequest.FechaPago`. Brecha compartida con los criterios D1 e I9 del requisito.

> ⚠️ **BRECHA MEDIA — Plantilla del correo de envío del Complemento de Pago (B2)**
> El asunto y cuerpo del correo de envío para el CP están pendientes de confirmar con PMO (PMO #31, transversal con Proforma, Factura, NC, CP e Inconsistencia). Sin esto, la plantilla en ProquifaDotNet.EnvioCorreo del modal de envío del Paso 3 no puede finalizarse.

> ✅ **RESUELTA — Mecanismo de reintento del CP tras Factura PPD timbrada (B3)**
> Si el timbrado de la Factura PPD es exitoso pero el CP inmediatamente posterior falla, la Factura PPD queda vigente sin CP. Timbrado no reintenta (servicio síncrono de un solo intento, ver R16A-RE-FU-018); el reintento se implementa en este mismo flujo de generación en Finanzas: la línea permanece `PENDIENTE`, se incrementa un contador de reintentos y se notifica a soporte por correo si se supera el límite (mismo patrón de `Diagramas/Diagrama Secuencia Encolamiento Finanzas y Timbrado Factura.md`). Queda pendiente solo confirmar con negocio si el reintento bloquea el envío de la Factura o si se permite enviarla sin CP mientras se reintenta. Brecha transversal con Factura Anticipo (RE-028 Brecha B6) y NC.

> ⚠️ **BRECHA BAJA — Soporte de tasas de IVA distintas a 16% (B4)**
> El criterio F3 del requisito documenta `TasaOCuotaDR = 0.160000` y agrega "Confirmar soporte para tasas distintas a 16%". Si hay clientes en zona fronteriza (8%) o con operaciones exentas (0%), la lógica de construcción del `TrasladoDR` debe soportar múltiples tasas. Confirmar con PMO el alcance real de escenarios de tasa variable para R16.

> ⚠️ **BRECHA BAJA — Esquema del foliador Serie "P" (B5)**
> El formato del folio del Complemento de Pago (Serie "P") está pendiente de validar con PMO (Regla 12 del requisito). Mientras no esté definido, el `FormatoFolio` y la `LongitudMaxima` en `EmpresaFolio` son valores propuestos. La implementación del consumo de folio en Timbrado es correcta independientemente del formato final.
