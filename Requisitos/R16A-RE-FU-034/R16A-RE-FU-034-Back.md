# Impacto en Backend — R16A-RE-FU-034
**Requisito:** Diseño y generación de Documentos — NDC México (CFDI tipo E)
**Fecha:** 2026-06-09
**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder

---

## Visión general

Este requisito documenta el diseño e implementación de la capa de generación de documentos de la Nota de Crédito México. Es análogo a **R16A-RE-FU-021** (Factura México) y cubre tres componentes principales:

- **Parte A — `NCMexicoXmlBuilder`**: Generación del XML CFDI tipo E.
- **Parte B — `NCMexicoPdfMappingService`**: Mapeo del modelo NC al template HTML (preview + post-timbrado).
- **Parte C — Templates HTML NC México**: Diseño visual e información por empresa emisora (GOL / MUN / PRO / PQF).

---

## Parte A — NCMexicoXmlBuilder

### Descripción
Clase responsable de construir el `XDocument` con el XML CFDI 4.0 tipo E a partir del `NCMexicoDto` generado en el wizard. Sigue el mismo patrón que `FacturaMexicoXmlBuilder`.

**Namespace:** `ProquifaDotNet.Finanzas.Application.Services.NC.Mexico`  
**Dependencias:** `CFDIGeneradaRepository`, `CFDIGeneradaConceptoRepository`, `EmpresaRepository`, `ClienteRepository`, `catMotivoCancelacionSATRepository`

---

### Estructura XML — Mapa de campos CFDI 4.0 tipo E

#### Comprobante (root)

| Atributo XML               | Valor / Origen                                                              | Criterio |
| -------------------------- | --------------------------------------------------------------------------- | -------- |
| `Version`                  | `"4.0"` fijo                                                                | B1       |
| `TipoDeComprobante`        | `"E"` fijo                                                                  | B1       |
| `Exportacion`              | `"01"` fijo                                                                 | B1       |
| `Serie`                    | Serie configurada para NC México por empresa (pendiente P5)                 | B2       |
| `Folio`                    | Consecutivo `EmpresaFolio` por empresa emisora (Serie "P2")                 | B2       |
| `Fecha`                    | Fecha y hora del timbrado en formato `yyyy-MM-ddTHH:mm:ss`                  | B2       |
| `LugarExpedicion`          | CP de la empresa emisora (`Empresa.CodigoPostal`)                           | B2       |
| `Moneda`                   | Moneda heredada de la factura origen (`CFDIGenerada.Moneda`)                | B3       |
| `TipoCambio`               | Tipo de cambio del día del timbrado (sólo si Moneda ≠ MXN)                 | B4       |
| `MetodoPago`               | `"PUE"` fijo                                                                | B5       |
| `FormaPago`                | Heredado de factura origen (`CFDIGenerada.FormaPago`, típicamente `"03"`)   | B6       |
| `SubTotal`                 | Suma de `Importe` de todos los `Concepto` (ver sección E / F)               | G2       |
| `Total`                    | `SubTotal` + `TotalImpuestosTrasladados` (ver P3 Descuento pendiente)       | G2       |
| `Sello`                    | Generado por TurboPac tras la firma del XML                                 | I2       |
| `NoCertificado`            | Número CSD empresa emisora (TurboPac)                                       | —        |
| `Certificado`              | Certificado en base64 empresa emisora (TurboPac)                            | —        |
| `CondicionesDePago`        | Omitir (no aplica en NC)                                                    | —        |
| `Descuento`                | ⚠️ **Pendiente P3** — validar con asesor fiscal si se puebla o se omite     | G2       |

---

#### CfdiRelacionados

| Atributo XML                        | Valor / Origen                                                 | Criterio |
| ----------------------------------- | -------------------------------------------------------------- | -------- |
| `cfdi:CfdiRelacionados[TipoRelacion]` | `"01"` fijo (NC de documentos relacionados)                  | C1       |
| `cfdi:CfdiRelacionado[UUID]`          | UUID de la factura origen (`CFDIGenerada.UUID`)               | C1       |

---

#### Emisor

| Atributo XML      | Valor / Origen                                             | Criterio |
| ----------------- | ---------------------------------------------------------- | -------- |
| `Rfc`             | RFC de la empresa emisora (`Empresa.RFC`)                  | D1       |
| `Nombre`          | Razón Social emisora (`Empresa.RazonSocial`)               | D1       |
| `RegimenFiscal`   | `"601"` (General de Ley Personas Morales) — fijo grupo PQF | D1       |

---

#### Receptor

| Atributo XML                | Valor / Origen                                             | Criterio |
| --------------------------- | ---------------------------------------------------------- | -------- |
| `Rfc`                       | RFC del cliente (`Cliente.RFC`)                            | D2       |
| `Nombre`                    | Razón Social cliente (`Cliente.RazonSocial`)               | D2       |
| `DomicilioFiscalReceptor`   | CP fiscal cliente (`Cliente.CodigoPostalFiscal`)           | D2       |
| `RegimenFiscalReceptor`     | Régimen fiscal cliente (`Cliente.RegimenFiscal`)           | D2       |
| `UsoCFDI`                   | `"G02"` por default (⚠️ ver pendiente UsoCFDI G02 vs G03)  | D3       |

---

#### Conceptos — Modalidad POR PARTIDAS

Por cada `NCMexicoPartidaDto` con `CantNC > 0`:

| Atributo XML         | Valor / Origen                                                               | Criterio |
| -------------------- | ---------------------------------------------------------------------------- | -------- |
| `ClaveProdServ`      | Heredado del concepto original (`CFDIGeneradaConcepto.ClaveProdServ`)        | E2       |
| `ClaveUnidad`        | Heredado (`CFDIGeneradaConcepto.ClaveUnidad`)                                | E2       |
| `NoIdentificacion`   | Heredado (`CFDIGeneradaConcepto.NoIdentificacion`)                           | E2       |
| `Cantidad`           | `NCMexicoPartidaDto.CantNC` (cantidad a notar crédito)                       | E3       |
| `Descripcion`        | Heredado (`CFDIGeneradaConcepto.Descripcion`)                                | E2       |
| `ValorUnitario`      | Heredado (`CFDIGeneradaConcepto.ValorUnitario`)                              | E2       |
| `Importe`            | `CantNC × ValorUnitario` — recalculado                                       | E3       |
| `ObjetoImp`          | Heredado (`CFDIGeneradaConcepto.ObjetoImp`)                                  | E2       |
| `Descuento`          | Omitir                                                                       | —        |

**Impuestos por línea:** recalcular IVA `TasaOCuota` heredada × `Importe` recalculado.

---

#### Conceptos — Modalidad MANUAL

Un único nodo `Concepto`:

| Atributo XML         | Valor / Origen                                                                | Criterio |
| -------------------- | ----------------------------------------------------------------------------- | -------- |
| `ClaveProdServ`      | `"84111506"` fijo (Servicios de facturación)                                  | F2       |
| `ClaveUnidad`        | `"ACT"` fijo (Actividad)                                                      | F2       |
| `NoIdentificacion`   | Omitir                                                                        | —        |
| `Cantidad`           | `"1"` fijo                                                                    | F2       |
| `Descripcion`        | Concepto libre capturado por el usuario en el wizard                          | F2       |
| `ValorUnitario`      | `NCMexicoManualDto.MontoTotalNC`                                               | F2       |
| `Importe`            | `MontoTotalNC` (igual a ValorUnitario × 1)                                    | F2       |
| `ObjetoImp`          | ⚠️ **Pendiente P4** — confirmar valor aplicable en modalidad manual            | F2       |

---

#### Impuestos (nivel comprobante)

| Atributo XML                        | Valor / Origen                                            | Criterio |
| ----------------------------------- | --------------------------------------------------------- | -------- |
| `Traslados/TotalImpuestosTrasladados` | Suma de todos los `Base × TasaOCuota` de los conceptos  | G1       |
| `Impuesto`                          | `"002"` (IVA)                                            | G1       |
| `TipoFactor`                        | `"Tasa"` fijo                                            | G1       |
| `TasaOCuota`                        | `"0.160000"` (IVA 16%) o tasa del concepto heredado      | G1       |

---

#### Complemento TimbreFiscalDigital (asignado por TurboPac / SAT)

| Atributo XML        | Origen                              |
| ------------------- | ----------------------------------- |
| `UUID`              | Asignado por SAT tras timbrado      |
| `FechaTimbrado`     | Asignado por SAT                    |
| `RfcProvCertif`     | `QSO100827UB0` (TurboPac)           |
| `SelloCFD`          | Sello del CFDI firmado por emisor   |
| `NoCertificadoSAT`  | Número CSD SAT                      |
| `SelloSAT`          | Sello SAT tras validación           |

---

### Validación contra ejemplo real B-128

| Campo             | Esperado (B-128)                               | Validación                                |
| ----------------- | ---------------------------------------------- | ----------------------------------------- |
| TipoDeComprobante | E                                              | ✅ Fijo en builder                         |
| Emisor RFC        | PRO970821ML3                                   | ✅ Empresa Proquifa                         |
| Receptor RFC      | NTM230915DQ8                                   | ✅ NATURES TOUCH MEXICO                     |
| Moneda            | USD                                            | ✅ Heredado de factura origen               |
| TipoCambio        | 19.17                                          | ✅ TC del día del timbrado                  |
| SubTotal          | 48.00                                          | ✅ Suma de importes de conceptos            |
| Total             | 55.68 (48.00 + IVA 16% = 7.68)                 | ✅ SubTotal + TotalImpuestosTrasladados      |
| MetodoPago        | PUE                                            | ✅ Fijo                                     |
| FormaPago         | 03 (Transferencia)                             | ✅ Heredado de factura origen               |
| ClaveProdServ     | 41116132                                       | ✅ Heredado del concepto original           |
| ClaveUnidad       | H87                                            | ✅ Heredado del concepto original           |
| TipoRelacion      | 01                                             | ✅ Fijo CfdiRelacionados                    |
| UUID origen       | 66099124-f173-a87d-55d9-336a3b6ad3f6           | ✅ UUID de la factura origen en BD          |
| RfcProvCertif     | QSO100827UB0 (TurboPac)                        | ✅ Constante PAC configurada                |
| UsoCFDI           | ⚠️ B-128 tiene G03; política PQF2 fija G02     | ⚠️ **Pendiente confirmar** (ver Regla 4)    |
| Descuento         | No aparece en B-128                            | ✅ Omitir campo en comprobante root         |

---

## Parte B — NCMexicoPdfMappingService

### Descripción
Servicio que mapea el `NCMexicoVm` (view model de la NC) al `NCMexicoPdfModel` requerido por el template HTML en DocumentBuilder. Cubre dos modos de operación.

**Namespace:** `ProquifaDotNet.Finanzas.Application.Services.NC.Mexico`  
**Patrón base:** `MexicoInvoicePdfMappingService` (R16A-RE-FU-021)

---

### Modo Preview (Paso 2 del wizard — antes de timbrado)

- Datos del emisor, receptor, conceptos y totales disponibles.
- **Sin** UUID, FechaTimbrado, SelloCFD, SelloSAT, NoCertificadoSAT, Cadena Original.
- **Sin** QR de verificación SAT.
- Se marca con leyenda **"PREVISUALIZACIÓN — Sin validez fiscal"** en el PDF.
- Permite al usuario confirmar antes de timbrar.

---

### Modo PostTimbrado (Paso 3 — tras timbrado exitoso)

- Todos los datos del modo Preview, más:
- UUID, FechaTimbrado, RfcProvCertif.
- SelloCFD, SelloSAT, NoCertificadoSAT, CadenaOriginal.
- **Código QR SAT** codificando: `https://verificacfdi.facturaelectronica.sat.gob.mx/default.aspx?id={UUID}&re={RFCEmisor}&rr={RFCReceptor}&tt={Total}&fe={últimos8SelloCFD}`

---

### Secciones del PDF a mapear (criterios J1–J12)

| Sección                  | Contenido a mapear                                                                                    | Criterio |
| ------------------------ | ----------------------------------------------------------------------------------------------------- | -------- |
| Encabezado — Emisor      | Logo empresa, Razón Social, RFC, CP, Fecha y Hora de expedición, Régimen Fiscal                       | J1, J4   |
| Datos del Comprobante    | Serie + Folio, Versión CFDI (4.0), UUID, Fecha/Hora Certificación, Tipo "E-Egreso", Moneda, TC, MetodoPago, FormaPago | J3, J6 |
| Receptor (Cliente)       | Razón Social, RFC, Domicilio Fiscal, Régimen Fiscal, UsoCFDI                                          | J5       |
| Relación CFDI            | Motivo NC (descripción), Tipo Relación SAT (01), Folio interno + UUID factura origen                  | J7       |
| Partidas (por partidas)  | Tabla: ClaveProdServ, Código interno, Descripción, Cant.NC, ClaveUnidad, Valor Unitario, Importe, IVA | J8       |
| Concepto (manual)        | ClaveProdServ (84111506), Cant. 1, ClaveUnidad (ACT), Descripción, Valor Unitario, Importe, IVA       | J9       |
| Totales                  | SubTotal, IVA 16%, Total, Total con letra                                                             | J10      |
| Sellos y Trazabilidad    | No. Serie CSD SAT, No. Serie CSD Emisor, SelloSAT, SelloCFD, CadenaOriginal                          | J11      |
| QR Verificación SAT      | Imagen QR (768×768 px mínimo) con URL verificación SAT                                                | J12      |
| Iconografía Certificaciones | Logos certificaciones (ISO, NEEC, edQM, FELUM, USP, etc.) ⚠️ vigencia pendiente confirmar          | J2, J3   |

---

## Parte C — Templates HTML NC México

### Nomenclatura y registro en DocumentTemplate

Cada empresa emisora tiene su propio set de 3 templates (Header, Body, Footer) registrado en la tabla `DocumentTemplate` (RE-032 T6 como prerrequisito):

| Empresa                           | Template Header          | Template Body          | Template Footer          |
| --------------------------------- | ------------------------ | ---------------------- | ------------------------ |
| Golocaer S.A. de C.V.             | `GOL_MEX_NC_H`           | `GOL_MEX_NC_B`         | `GOL_MEX_NC_F`           |
| Mungen S.A. de C.V.               | `MUN_MEX_NC_H`           | `MUN_MEX_NC_B`         | `MUN_MEX_NC_F`           |
| Proquifa S.A. de C.V.             | `PRO_MEX_NC_H`           | `PRO_MEX_NC_B`         | `PRO_MEX_NC_F`           |
| Proveedora Quimico Farmaceutica   | `PQF_MEX_NC_H`           | `PQF_MEX_NC_B`         | `PQF_MEX_NC_F`           |

---

### Identidad visual por empresa emisora

| Empresa                         | Color principal  | Logo                                      |
| ------------------------------- | ---------------- | ----------------------------------------- |
| Golocaer S.A. de C.V.           | Naranja          | `golocaer_logo.png`                       |
| Mungen S.A. de C.V.             | Verde            | `mungen_logo.png`                         |
| Proquifa S.A. de C.V.           | Cyan             | `proquifa_logo.png`                       |
| Proveedora Quimico Farmaceutica | Cyan             | `pqf_logo.png`                            |

La paleta, tipografía e iconografía de certificaciones deben ser **consistentes** con:
- `GOL/MUN/PRO/PQF_MEX_FAC` (Factura México — R16A-RE-FU-021)
- `GOL/MUN/PRO/PQF_MEX_CDP` (Complemento de Pago México)

---

### Estructura del template HTML

#### Header (H)
- Logo empresa emisora (esquina superior izquierda).
- Título **"NOTA DE CRÉDITO"** en color corporativo.
- Datos de la empresa emisora: Razón Social, RFC, Régimen Fiscal.
- Datos del comprobante: Serie-Folio, Versión, Tipo E-Egreso.
- Fecha y hora de emisión.
- Fecha y hora de certificación (post-timbrado).

#### Body (B)
- **Bloque Receptor:** Razón Social, RFC, Domicilio Fiscal, Régimen Fiscal, UsoCFDI.
- **Bloque Relación CFDI:** Motivo de la NC, TipoRelacion 01, Folio + UUID factura origen.
- **Bloque Partidas:** Tabla con columnas (condicional por modalidad):
  - Modalidad por partidas: ClaveProdServ | Código | Descripción | Cant. | Unidad | Valor Unit. | Importe | IVA
  - Modalidad manual: ClaveProdServ | Cantidad | Unidad | Descripción | Valor Unit. | Importe | IVA
- **Bloque Totales:** SubTotal | IVA 16% | Total | Total en letra (línea debajo de la tabla).
- **UUID** (post-timbrado): Mostrar folio fiscal en destacado.

#### Footer (F)
- **Sellos y trazabilidad SAT:**
  - No. Serie del CSD del SAT
  - No. Serie del CSD del Emisor
  - Sello del CFDI
  - Sello del SAT
  - Cadena Original del Complemento de Certificación Digital del SAT
- **QR de verificación SAT** (esquina inferior derecha).
- **Iconografía certificaciones** del giro químico-farmacéutico: ISO, NEEC, edQM, FELUM, USP, Microbiologics, APACOR, CHATA, Pharmaffiliates, Amex (⚠️ vigencia pendiente confirmar).
- Leyenda de conservación: "Este documento es una representación impresa de un CFDI. Conservar por 5 años (Art. 30 CFF)."

---

## Parte D — PersistirNCMexicoPdfService

### Descripción
Servicio que invoca DocumentBuilder para renderizar el template HTML con los datos del `NCMexicoPdfModel` y guarda el PDF resultante en MinIO.

**Patrón base:** `PersistMexicoInvoicePdfService` (R16A-RE-FU-021)

### Rutas MinIO

| Archivo | Bucket             | Path                                           |
| ------- | ------------------ | ---------------------------------------------- |
| XML     | `notas-credito-mex` | `notas_credito/{anio}/{mes}/{UUID}.xml`        |
| PDF     | `notas-credito-mex` | `notas_credito/{anio}/{mes}/{UUID}.pdf`        |

---

## Parte E — Envío de correo NC México

**Servicio:** `SendMexicoCreditNoteMailService` (extiende patrón de `SendMexicoInvoiceMailService`)

| Campo         | Valor                                                                                        |
| ------------- | -------------------------------------------------------------------------------------------- |
| Para          | Correo del contacto del cliente vinculado a la factura origen                                |
| CC            | ESAC asignado al cliente + analista de Cuentas por Cobrar                                    |
| Adjuntos      | PDF NC + XML NC                                                                              |
| Asunto/Cuerpo | Plantilla PMO #31 ⚠️ **Pendiente confirmar** (transversal Proforma / Factura / NC / CDP)    |

---

## Diferencias clave: NCMexicoXmlBuilder vs FacturaMexicoXmlBuilder

| Aspecto                    | Factura México (RE-021)                    | NC México (RE-034)                                   |
| -------------------------- | ------------------------------------------ | ---------------------------------------------------- |
| TipoDeComprobante          | `I` (Ingreso)                              | `E` (Egreso)                                         |
| CfdiRelacionados           | Ninguno                                    | TipoRelacion=01 + UUID factura origen (obligatorio)  |
| UsoCFDI                    | Según cliente                              | G02 por default                                      |
| MetodoPago                 | PUE o PPD                                  | PUE fijo (inmutable)                                 |
| FormaPago                  | Capturado en la factura                    | Heredado de la factura origen                        |
| Conceptos                  | Desde pedido o manual                      | Heredados de factura origen (por partidas) o manual  |
| Cancelación condicional    | No aplica                                  | Cancela factura origen si totalidad + mismo mes      |
| Serie foliador             | Serie factura por empresa                  | Serie NC México por empresa ("P2" propuesta)         |

---

## Pendientes técnicos

| ID  | Descripción                                                                                   | Impacto                                           |
| --- | --------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| P1  | Brecha B1 RE-032: `IdTPProformaPedido` NOT NULL en `fccNotaCredito`                           | Bloquea T1 RE-032 — resolver antes de iniciar     |
| P2  | UsoCFDI G02 vs G03 (discrepancia ejemplo B-128)                                               | Afecta nodo Receptor del XML                      |
| P3  | Campo `Descuento` comprobante root — validar con asesor fiscal                                | Afecta cálculo de Total y estructura del XML      |
| P4  | `ObjetoImp` modalidad manual — confirmar valor para ClaveProdServ 84111506                    | Afecta nodo Impuestos del Concepto manual         |
| P5  | Serie del foliador NC México — validar nombre definitivo de la serie en `EmpresaFolio`         | Afecta atributo `Serie` del comprobante root      |
| P6  | Vigencia iconografía certificaciones (ISO, NEEC, edQM, FELUM, USP, etc.)                      | Afecta Footer de los 4 templates                  |
| P7  | Plantilla PMO #31 — asunto y cuerpo del correo                                                | Bloquea `SendMexicoCreditNoteMailService`               |
| P8  | `FormaPago` cuando la factura origen no está pagada (escenario raro en R16 prepago)           | Afecta atributo FormaPago del comprobante root    |
