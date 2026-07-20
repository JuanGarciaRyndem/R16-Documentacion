# Impacto en Backend — R16A-RE-FU-035
**Requisito:** Diseño y generación de Documentos — NDC Perú (CPE tipo 07 — UBL 2.1)
**Fecha:** 2026-06-09
**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder

> ⚠️ **Nota transversal:** Toda la mecánica fiscal SUNAT de este requisito está pendiente de validación con el asesor fiscal peruano de PROQUIFA antes de implementarse. Los campos, estructuras y cálculos descritos aquí se basan en investigación normativa UBL 2.1 SUNAT y pueden requerir ajustes.

---

## Visión general

Este requisito documenta el diseño e implementación de la capa de generación de documentos de la Nota de Crédito Perú. Es análogo a **R16A-RE-FU-035** para Perú vs **R16A-RE-FU-034** para México. Cubre tres componentes:

- **Parte A — `NCPeruXmlBuilder`**: Generación del XML CPE tipo 07 (UBL 2.1).
- **Parte B — `NCPeruPdfMappingService`**: Mapeo del modelo NC al template HTML (preview + post-timbrado).
- **Parte C — Template HTML GOL\_PER\_NC**: Diseño visual e información conforme a representación impresa SUNAT.

---

## Parte A — NCPeruXmlBuilder

### Descripción
Clase responsable de construir el documento XML UBL 2.1 CPE tipo 07 a partir del `NCPeruDto`. Sigue el mismo patrón que `FacturaPeruXmlBuilder` (R16A-RE-FU-022) pero para Nota de Crédito.

**Namespace:** `ProquifaDotNet.Finanzas.Application.Services.NC.Peru`  
**Dependencias:** `CFDIGeneradaRepository`, `CFDIGeneradaConceptoRepository`, `EmpresaRepository`, `ClienteRepository`, `catMotivoCreditoSUNAT09Repository`

---

### Namespaces XML UBL 2.1 requeridos

```xml
xmlns="urn:oasis:names:specification:ubl:schema:xsd:CreditNote-2"
xmlns:cac="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2"
xmlns:cbc="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2"
xmlns:ds="http://www.w3.org/2000/09/xmldsig#"
xmlns:ext="urn:oasis:names:specification:ubl:schema:xsd:CommonExtensionComponents-2"
```

---

### Estructura XML — Mapa de campos CPE tipo 07 UBL 2.1

#### Cabecera del documento

| Elemento XML                          | Valor / Origen                                                                    | Criterio |
| ------------------------------------- | --------------------------------------------------------------------------------- | -------- |
| `cbc:UBLVersionID`                    | `"2.1"` fijo                                                                      | B1       |
| `cbc:CustomizationID`                 | `"2.0"` fijo (customización SUNAT)                                                | B1       |
| `cbc:ProfileID`                       | `"0601"` fijo (Facturación electrónica SUNAT)                                     | B1       |
| `cbc:ID`                              | `{Serie}-{Correlativo}` (ej. `FC01-00000001`) — folio consecutivo Golocaer S.A.C. | B2       |
| `cbc:IssueDate`                       | Fecha de emisión de la NC en formato `yyyy-MM-dd`                                 | B2       |
| `cbc:InvoiceTypeCode listID="07"`     | `"07"` fijo (Nota de Crédito Electrónica)                                         | B1       |
| `cbc:DocumentCurrencyCode`            | Moneda heredada del CPE origen (`CFDIGenerada.Moneda`)                            | B3       |
| `cbc:Note languageLocaleID="1000"`    | Monto en letras del Importe Total                                                 | I7       |

---

#### DiscrepancyResponse (motivo y referencia)

| Elemento XML                          | Valor / Origen                                                                | Criterio |
| ------------------------------------- | ----------------------------------------------------------------------------- | -------- |
| `cac:DiscrepancyResponse/cbc:ReferenceID` | Serie-Correlativo del CPE origen (ej. `F001-00000123`)                    | C1       |
| `cac:DiscrepancyResponse/cbc:ResponseCode listAgencyName="PE:SUNAT"` | Código catálogo 09 (`catMotivoCreditoSUNAT09.Clave`) | C2 |
| `cac:DiscrepancyResponse/cbc:Description` | Sustento del motivo (`catMotivoCreditoSUNAT09.Descripcion`) — ⚠️ ver P6  | C2       |

---

#### BillingReference (comprobante afectado)

| Elemento XML                                          | Valor / Origen                                          | Criterio |
| ----------------------------------------------------- | ------------------------------------------------------- | -------- |
| `cac:BillingReference/cac:InvoiceDocumentReference/cbc:ID` | Serie-Correlativo del CPE origen                   | C1       |
| `cac:BillingReference/cac:InvoiceDocumentReference/cbc:DocumentTypeCode` | `"01"` (Factura) o según el tipo del CPE origen | C1 |

---

#### AccountingSupplierParty (Emisor — Golocaer S.A.C.)

| Elemento XML                                        | Valor / Origen                                                  | Criterio |
| --------------------------------------------------- | --------------------------------------------------------------- | -------- |
| `...Party/cac:PartyIdentification/cbc:ID schemeID="6"` | RUC Golocaer S.A.C. `20612772941`                            | D1       |
| `...Party/cac:PartyName/cbc:Name`                   | Razón social Golocaer S.A.C.                                    | D1       |
| `...Party/cac:PostalAddress/cbc:StreetName`         | Domicilio fiscal de Golocaer (`Empresa.Direccion`)              | D1       |
| `...Party/cac:PostalAddress/cbc:CountrySubentityCode` | Ubigeo SUNAT (`Empresa.Ubigeo`)                               | D1       |
| `...Party/cac:PartyTaxScheme/cbc:RegistrationName`  | Razón social Golocaer                                           | D1       |
| `...Party/cac:PartyTaxScheme/cbc:CompanyID schemeID="6"` | RUC `20612772941`                                          | D1       |

---

#### AccountingCustomerParty (Receptor — Cliente)

| Elemento XML                                        | Valor / Origen                                                  | Criterio |
| --------------------------------------------------- | --------------------------------------------------------------- | -------- |
| `...Party/cac:PartyIdentification/cbc:ID schemeID` | RUC (6), DNI (1) o CE (4) según tipo de documento del cliente   | D2       |
| `...Party/cac:PartyName/cbc:Name`                   | Razón social del cliente                                        | D2       |
| `...Party/cac:PartyTaxScheme/cbc:RegistrationName`  | Razón social del cliente                                        | D2       |
| `...Party/cac:PartyTaxScheme/cbc:CompanyID`         | Número de documento del cliente                                 | D2       |

> **Diferencia con México:** NO se consigna UsoCFDI, DomicilioFiscalReceptor, ni RegimenFiscalReceptor (conceptos SAT, no SUNAT).

---

#### TaxExchangeRate (tipo de cambio — solo si moneda ≠ PEN)

| Elemento XML                            | Valor / Origen                                                                    | Criterio |
| --------------------------------------- | --------------------------------------------------------------------------------- | -------- |
| `cac:TaxExchangeRate/cbc:SourceCurrencyCode` | Moneda del CPE origen                                                       | B4       |
| `cac:TaxExchangeRate/cbc:TargetCurrencyCode` | `"PEN"` fijo                                                                | B4       |
| `cac:TaxExchangeRate/cbc:CalculationRate`    | TC de la FECHA DE EMISIÓN del CPE origen (`fccNotaCredito.TipoCambioOrigen`) | B4      |

> ⚠️ **Diferencia fiscal clave vs México:** El TC es el de la fecha de emisión del CPE origen (no el del día del timbrado de la NC), conforme a SUNAT. Ver Regla 6.

---

#### CreditNoteLine — Modalidad POR PARTIDAS

Por cada `NCPeruPartidaDto` con `CantNC > 0`:

| Elemento XML                                              | Valor / Origen                                                            | Criterio |
| --------------------------------------------------------- | ------------------------------------------------------------------------- | -------- |
| `cbc:CreditedQuantity unitCode`                           | Código de unidad UBL (`CFDIGeneradaConcepto.ClaveUnidad`)                 | E2, E3   |
| `cbc:CreditedQuantity`                                    | `CantNC` capturada                                                        | E3       |
| `cbc:LineExtensionAmount currencyID`                      | Valor de venta de la línea = `CantNC × ValorUnitario` (sin IGV)           | E3       |
| `cac:Item/cbc:Description`                                | Descripción del concepto original                                         | E2       |
| `cac:Item/cac:SellersItemIdentification/cbc:ID`           | Código interno del producto (`CFDIGeneradaConcepto.NoIdentificacion`)     | E2       |
| `cac:Price/cbc:PriceAmount`                               | Valor unitario heredado                                                   | E2       |
| `cac:TaxTotal/cac:TaxSubtotal` (IGV)                      | Base × 18% — o tasa heredada del concepto original                        | E3       |
| `cac:TaxTotal/cac:TaxSubtotal/cac:TaxCategory/cbc:TaxExemptionReasonCode` | Código de afectación IGV heredado (`CFDIGeneradaConcepto.ObjetoImp`) | E2 |

---

#### CreditNoteLine — Modalidad MANUAL

Un único `cac:CreditNoteLine`:

| Elemento XML                          | Valor / Origen                                              | Criterio |
| ------------------------------------- | ----------------------------------------------------------- | -------- |
| `cbc:CreditedQuantity unitCode="NIU"` | `"1"` fijo                                                  | F1       |
| `cbc:LineExtensionAmount`             | Monto total NC capturado (sin IGV)                          | F1       |
| `cac:Item/cbc:Description`            | Concepto libre capturado por el usuario                     | F1       |
| `cac:Price/cbc:PriceAmount`           | Monto total NC                                              | F1       |
| `cac:TaxTotal/cac:TaxSubtotal`        | Base × 18% (IGV)                                            | G1       |

---

#### TaxTotal (nivel documento)

| Elemento XML                                | Valor / Origen                                     | Criterio |
| ------------------------------------------- | -------------------------------------------------- | -------- |
| `cbc:TaxAmount currencyID`                  | Suma de IGV de todas las líneas                    | G1       |
| `cac:TaxSubtotal/cbc:TaxableAmount`         | Suma de los `LineExtensionAmount` (Valor Venta)    | G1       |
| `cac:TaxSubtotal/cac:TaxCategory/cbc:ID`    | `"VAT"` (IGV) o `"EXC"` (exonerado) según líneas  | G1       |
| `cac:TaxSubtotal/cac:TaxScheme/cbc:ID`      | `"1000"` (IGV), `"1010"` (ISC) según aplique       | G1       |
| `cac:TaxSubtotal/cac:TaxScheme/cbc:Name`    | `"IGV"` fijo para la tasa estándar                 | G1       |
| `cac:TaxSubtotal/cac:TaxScheme/cbc:TaxTypeCode` | `"VAT"` fijo                                   | G1       |

---

#### LegalMonetaryTotal (totales del documento)

| Elemento XML                                 | Valor / Origen                                              | Criterio |
| -------------------------------------------- | ----------------------------------------------------------- | -------- |
| `cbc:LineExtensionAmount`                    | Valor Venta total (suma líneas, sin IGV)                    | G2       |
| `cbc:TaxInclusiveAmount`                     | Importe Total (Valor Venta + IGV)                           | G2       |
| `cbc:PayableAmount`                          | Importe a pagar (= Importe Total)                           | G2       |

---

### Firma digital y respaldo SUNAT

| Elemento                        | Descripción                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------- |
| `cac:Signature/cbc:ID`          | Identificador de firma — `"IDSign${RUC_Emisor}"` fijo                           |
| Firma XML (`ds:Signature`)      | Generada por el servicio de firma con el certificado digital de Golocaer S.A.C. |
| Valor resumen/hash              | Calculado sobre el XML firmado — es el respaldo ante SUNAT (no hay sello SAT)   |

> ⚠️ **BLOQUEADO B1:** La implementación end-to-end del timbrado SUNAT (envío y recepción del CDR) depende de la modalidad de emisión electrónica que defina Golocaer ante SUNAT. Ver R16A-RE-FU-029/033.

---

### Diferencias clave: NCPeruXmlBuilder vs NCMexicoXmlBuilder

| Aspecto                        | México (RE-034)                                    | Perú (RE-035)                                        |
| ------------------------------ | -------------------------------------------------- | ---------------------------------------------------- |
| Formato XML                    | CFDI 4.0 (Annexo 20 SAT)                           | UBL 2.1 (SUNAT)                                      |
| Tipo de comprobante            | `TipoDeComprobante=E`                              | `InvoiceTypeCode=07`                                 |
| Referencia al origen           | UUID SAT en `CfdiRelacionados TipoRelacion=01`     | Serie-Correlativo en `BillingReference`              |
| Motivo                         | `c_MotivoCancelacion` (4 claves)                   | Catálogo 09 SUNAT (11 códigos) — `DiscrepancyResponse` |
| Impuesto                       | IVA 16% (`ImpIVA`)                                 | IGV 18% (`TaxScheme ID="1000"`)                      |
| Tipo de cambio                 | Del día del timbrado                               | De la fecha de emisión del CPE origen                |
| Empresas emisoras              | GOL, MUN, PRO, PQF (4)                             | Solo Golocaer S.A.C. (1)                             |
| Método / Forma de pago         | Obligatorios (PUE/03)                              | No aplica (conceptos SAT)                            |
| Respaldo del timbrado          | UUID SAT + sello SAT + cadena complementaria       | Firma digital + valor resumen/hash SUNAT             |
| Cancelación SAT condicional    | Sí (totalidad + mismo mes)                         | No aplica — anulación vía NC motivo 01 o Baja         |

---

## Parte B — NCPeruPdfMappingService

### Descripción
Servicio que mapea el `NCPeruVm` al `NCPeruPdfModel` para el template HTML `GOL_PER_NC` en DocumentBuilder. Cubre dos modos.

**Patrón base:** `PeruInvoicePdfMappingService` (R16A-RE-FU-022)

---

### Modo Preview (antes del timbrado)
- Datos del emisor, receptor, conceptos y totales disponibles.
- **Sin** firma digital, sin hash/resumen, sin QR SUNAT.
- Se marca con leyenda **"PREVISUALIZACIÓN — Sin validez fiscal"**.

### Modo PostTimbrado (tras conformidad SUNAT)
- Todos los datos del Preview, más:
- Serie-Correlativo definitivo.
- Hash/resumen del XML firmado.
- **QR SUNAT**: URL de verificación (formato SUNAT). ⚠️ formato exacto pendiente de validar (B1).
- Leyenda obligatoria de representación impresa.

---

### Secciones del PDF a mapear (criterios I1–I8)

| Sección                      | Contenido a mapear                                                                                    | Criterio |
| ---------------------------- | ----------------------------------------------------------------------------------------------------- | -------- |
| Encabezado — Emisor          | Logo Golocaer S.A.C., Razón Social, RUC, Domicilio Fiscal                                             | I1, I2   |
| Datos del Comprobante        | Tipo "07 – Nota de Crédito Electrónica", Serie-Correlativo, Fecha de emisión                          | I4       |
| Receptor (Cliente)           | Razón Social, RUC/documento                                                                           | I3       |
| Motivo y Referencia          | Código catálogo 09 + Descripción, Serie-Correlativo del CPE origen                                    | I5       |
| Líneas (por partidas)        | Tabla: código, descripción, cant., unidad, valor unitario, valor venta, IGV                           | I6       |
| Concepto (manual)            | Descripción, cantidad 1, valor, IGV                                                                   | I6       |
| Totales                      | Valor Venta (sin IGV), IGV 18%, Importe Total; monto en letras                                        | I7       |
| QR y Leyenda                 | QR de verificación SUNAT + leyenda "Representación impresa de la Nota de Crédito Electrónica..."      | I8       |
| Hash/Resumen                 | Valor resumen del XML firmado (PostTimbrado únicamente) — no lleva sello SAT ni cadena complementaria | I8       |

---

## Parte C — Template HTML GOL\_PER\_NC

### Descripción
Un único set de 3 archivos HTML (Header, Body, Footer) para Golocaer S.A.C. — empresa emisora única en Perú.

| Template Header    | Template Body      | Template Footer    |
| ------------------ | ------------------ | ------------------ |
| `GOL_PER_NC_H`     | `GOL_PER_NC_B`     | `GOL_PER_NC_F`     |

---

### Identidad visual
- **Color corporativo:** Igual que `GOL_PER_FAC` (Factura Perú, R16A-RE-FU-022).
- **Logo:** `golocaer_logo.png` — mismo que Factura Perú.
- Consistente con la representación impresa exigida por SUNAT para documentos electrónicos.

---

### Estructura del template HTML

#### Header (H)
- Logo Golocaer S.A.C. (esquina superior izquierda).
- Título **"NOTA DE CRÉDITO ELECTRÓNICA"** en color corporativo.
- Datos del emisor: Razón Social, RUC `20612772941`, Domicilio Fiscal, Régimen Tributario.
- Datos del comprobante: Tipo 07, Serie-Correlativo, Fecha de emisión.

#### Body (B)
- **Bloque Receptor:** Razón Social, RUC/documento.
- **Bloque Referencia:** Motivo (código catálogo 09 + descripción), CPE origen (Serie-Correlativo).
- **Bloque Moneda:** Moneda del comprobante + Tipo de cambio (si aplica).
- **Bloque Líneas:** Tabla condicional por modalidad:
  - Por partidas: código | descripción | cant. | unidad | valor unit. | valor venta | IGV
  - Manual: descripción | cantidad 1 | unidad NIU | valor venta | IGV
- **Bloque Totales:** Valor Venta (sin IGV) | IGV 18% | **Importe Total** | Monto en letras (debajo de la tabla).

> ⚠️ **Diferencias vs GOL\_MEX\_NC\_B:** Sin UUID, sin MetodoPago/FormaPago, sin UsoCFDI, nomenclatura peruana (Valor Venta/IGV/Importe Total en vez de SubTotal/IVA/Total).

#### Footer (F)
- **QR de verificación SUNAT** (esquina inferior derecha).
- **Leyenda obligatoria:** "Representación impresa de la Nota de Crédito Electrónica – consúltela en [URL SUNAT]".
- **Valor resumen:** Hash del XML firmado (PostTimbrado).
- **Leyenda de conservación:** "Conservar por 5 años (R.S. 117-2017/SUNAT)."

> ⚠️ **Sin:** sello digital SAT, cadena complementaria, ni iconografía de certificaciones SAT (conceptos del SAT no aplican en Perú).

---

## Parte D — PersistirNCPeruPdfService

**Patrón base:** `PersistPeruInvoicePdfService` (R16A-RE-FU-022)

### Rutas MinIO

| Archivo | Bucket               | Path                                                        |
| ------- | -------------------- | ----------------------------------------------------------- |
| XML     | `notas-credito-per`  | `notas_credito/{anio}/{mes}/{serie}-{correlativo}.xml`      |
| PDF     | `notas-credito-per`  | `notas_credito/{anio}/{mes}/{serie}-{correlativo}.pdf`      |

> **Diferencia vs México:** La ruta usa `{serie}-{correlativo}` (no UUID) como identificador del archivo, ya que Perú no tiene UUID SAT.

---

## Parte E — Envío de correo NC Perú

| Campo         | Valor                                                                              |
| ------------- | ---------------------------------------------------------------------------------- |
| Para          | Contacto del cliente vinculado al CPE origen                                       |
| CC            | ESAC asignado al cliente                                                           |
| Adjuntos      | PDF NC + XML NC                                                                    |
| Asunto        | Pre-rellenado con folio de la NC (serie-correlativo). ⚠️ Plantilla cuerpo pendiente |

---

## Pendientes técnicos

| ID  | Descripción                                                                                      | Impacto                                                        |
| --- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| P1  | **Brecha B1 RE-033:** Modalidad emisión SUNAT no definida — OSE vs facturador directo            | Bloquea timbrado end-to-end; XML builder + PDF pueden avanzar  |
| P2  | **Mecánica fiscal SUNAT completa** — sujeta a validación con asesor fiscal peruano              | Puede requerir reestructurar campos y cálculos del XML         |
| P3  | **TipoCambioOrigen:** confirmar que se usa el TC de la fecha de emisión del CPE origen           | Clave diferencial Perú vs México — afecta `TaxExchangeRate`    |
| P4  | **Código de afectación IGV** en modalidad manual — confirmar valor aplicable                    | Afecta `TaxExemptionReasonCode` del único concepto manual      |
| P5  | **Maquetas PDF NC Perú** no disponibles — diseño se validará contra ellas cuando lleguen         | Puede requerir ajustes al template GOL\_PER\_NC                |
| P6  | **Sustento (cbc:Description):** confirmar si es editable o fijo por catálogo 09                  | Afecta `DiscrepancyResponse/cbc:Description`                   |
| P7  | **Plantilla correo Perú:** asunto y cuerpo del correo pendientes de confirmar                    | Bloquea `SendPeruCreditNoteMailService`                              |
| P8  | **Formato QR SUNAT** para NC Perú — confirmar parámetros exactos de la URL de verificación      | Afecta Footer del template GOL\_PER\_NC                        |
