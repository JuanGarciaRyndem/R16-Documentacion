# Tareas BackEnd — R16A-RE-FU-035
**Requisito:** Diseño y generación de Documentos — NDC Perú (CPE tipo 07 — UBL 2.1)
**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder

---

## Resumen de tareas

| #  | Clave             | Título simple                                                                | Tipo | Aplicativo              |
| -- | ----------------- | ---------------------------------------------------------------------------- | ---- | ----------------------- |
| 1  | CREATE-PDF        | Template HTML Golocaer NC Perú: GOL\_PER\_NC (H/B/F)                        | Back | DocumentBuilder         |
| 2  | ALG-COMPLX-LOGIC  | Implementar NCPeruXmlBuilder — generación XML CPE tipo 07 (UBL 2.1)          | Back | ProquifaDotNet.Finanzas |
| 3  | IMP-EXIST-SERVICE | Implementar NCPeruPdfMappingService — Preview y PostTimbrado + Persistencia   | Back | ProquifaDotNet.Finanzas |

---

## TAREA 1

**[ RE-FU-035 ] [CREATE-PDF] Template HTML Golocaer NC Perú — GOL\_PER\_NC (H/B/F)**

**Aplicativos:** DocumentBuilder

**Módulos:** Documentos — Nota de Crédito Perú — Template Golocaer S.A.C.

**Consideraciones previas:**
- Consistente con `GOL_PER_FAC` (Factura Perú, R16A-RE-FU-022) en branding, tipografía y estructura.
- 3 archivos: `GOL_PER_NC_H.html` (Header), `GOL_PER_NC_B.html` (Body), `GOL_PER_NC_F.html` (Footer).
- Prerrequisito: RE-033 T5 insertó el registro `GOL_PER_NC` en `DocumentTemplate`.
- Solo una empresa emisora en Perú (vs 4 en México) — no hay variantes de branding.
- ⚠️ **Maquetas PDF NC Perú no disponibles (P5):** diseño se basa en la representación impresa SUNAT UBL 2.1; ajustar cuando lleguen las maquetas.
- ⚠️ **Formato QR SUNAT pendiente (P8):** implementar QR con URL placeholder y ajustar cuando se confirme el formato exacto.
- Las secciones del PDF deben cumplir la representación impresa SUNAT: sin sello SAT, sin cadena complementaria, sin UsoCFDI, sin MetodoPago/FormaPago.

**Objetivo general:**
Diseñar e implementar los 3 archivos HTML del template de Nota de Crédito Perú para Golocaer S.A.C., conforme a la representación impresa SUNAT para documentos electrónicos UBL 2.1.

**Objetivos específicos:**
- **Header:** Logo Golocaer (paleta `GOL_PER_FAC`), título "NOTA DE CRÉDITO ELECTRÓNICA", datos emisor (Razón Social, RUC, Domicilio Fiscal, Régimen Tributario), datos comprobante (tipo 07, Serie-Correlativo, Fecha emisión).
- **Body:** Bloque receptor (Razón Social, RUC/documento), bloque referencia (código catálogo 09 + descripción + CPE origen serie-correlativo), bloque moneda + TC si aplica, tabla de líneas o concepto manual según modalidad, totales (Valor Venta, IGV 18%, Importe Total, monto en letras).
- **Footer:** QR verificación SUNAT, leyenda obligatoria representación impresa, valor resumen hash (PostTimbrado), leyenda conservación 5 años (R.S. 117-2017/SUNAT).
- Validar renderizado correcto en modo Preview (sin QR/hash) y PostTimbrado (con QR/hash).

**Resultado esperado:**
3 archivos HTML `GOL_PER_NC_H`, `GOL_PER_NC_B`, `GOL_PER_NC_F` registrados en DocumentBuilder, generando un PDF conforme a la representación impresa SUNAT en ambos modos.

**Entregables:**
- `GOL_PER_NC_H.html`
- `GOL_PER_NC_B.html`
- `GOL_PER_NC_F.html`
- PDF de prueba en modo Preview y PostTimbrado

**Criterios de aceptación:**
- El PDF Preview muestra leyenda "PREVISUALIZACIÓN — Sin validez fiscal" y no contiene QR ni hash.
- El PDF PostTimbrado incluye el QR de verificación SUNAT y el valor resumen del XML firmado.
- Los totales muestran Valor Venta, IGV 18% e Importe Total con nomenclatura SUNAT.
- El bloque de referencia muestra el motivo (código + descripción catálogo 09) y el CPE origen.
- **No aparecen** campos SAT: sin UUID, sin sello SAT, sin cadena complementaria, sin MetodoPago, sin FormaPago, sin UsoCFDI.
- Branding de Golocaer S.A.C. es consistente con `GOL_PER_FAC`.
- La leyenda obligatoria de representación impresa está presente en el Footer.

**Más información:**
Ver sección **Parte C** de `R16A-RE-FU-035-Back.md` — estructura detallada del template y diferencias vs `GOL_MEX_NC`.

**Recursos:**
- `R16A-RE-FU-035-Back.md` — Parte C
- `GOL_PER_FAC` (RE-022) — referencia de branding Golocaer Perú
- Representación impresa SUNAT para Notas de Crédito Electrónicas (UBL 2.1)

---

## TAREA 2

**[ RE-FU-035 ] [ALG-COMPLX-LOGIC] Implementar NCPeruXmlBuilder — generación XML CPE tipo 07 (UBL 2.1)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application — Services — NC — Peru — NCPeruXmlBuilder

**Consideraciones previas:**
- Patrón base: `FacturaPeruXmlBuilder` (R16A-RE-FU-022). Adaptar a CPE tipo 07 (NC).
- **Diferencias clave vs NCMexicoXmlBuilder:** UBL 2.1 (no CFDI 4.0), `InvoiceTypeCode=07`, sin UUID, sin sello SAT, referencia por serie-correlativo (`BillingReference`), motivo en `DiscrepancyResponse` (catálogo 09), IGV 18%, TC heredado de la fecha de emisión del CPE origen.
- ⚠️ **BLOQUEADO B1 para integración:** El XML puede construirse y probarse unitariamente, pero el envío a SUNAT y la recepción del CDR dependen de la modalidad de emisión (ver R16A-RE-FU-029/033).
- ⚠️ **Mecánica fiscal SUNAT pendiente de validar (P2):** avanzar con la estructura normativa investigada; puede requerir ajustes tras validación con asesor fiscal.
- ⚠️ **TipoCambioOrigen (P3):** usar `fccNotaCredito.TipoCambioOrigen` — TC de la fecha de emisión del CPE origen, no el del día del timbrado.
- Prerrequisito funcional: RE-033 T9 (endpoint timbrado NC Perú) debe estar disponible para integración end-to-end.

**Objetivo general:**
Implementar `NCPeruXmlBuilder` que recibe el `NCPeruDto` y genera el `XDocument` UBL 2.1 CPE tipo 07 conforme al mapa de campos documentado en R16A-RE-FU-035-Back.md, con firma digital y estructura válida para SUNAT.

**Objetivos específicos:**
- Implementar cabecera UBL 2.1: `UBLVersionID`, `CustomizationID`, `ProfileID`, `ID` (serie-correlativo), `IssueDate`, `InvoiceTypeCode=07`, `DocumentCurrencyCode`.
- Implementar `DiscrepancyResponse`: ReferenceID (serie-correlativo CPE origen), ResponseCode (catálogo 09), Description (sustento).
- Implementar `BillingReference`: ID del CPE origen + DocumentTypeCode.
- Implementar `AccountingSupplierParty` (Golocaer S.A.C. — RUC `20612772941`).
- Implementar `AccountingCustomerParty` (cliente — RUC/DNI/CE según tipo de documento).
- Implementar `TaxExchangeRate` cuando moneda ≠ PEN, usando `TipoCambioOrigen` (fecha emisión CPE origen).
- Implementar `CreditNoteLine` × N (por partidas, heredando datos fiscales del concepto original y recalculando por `CantNC`) o único concepto manual.
- Implementar cálculo automático de IGV 18% a nivel línea y comprobante.
- Implementar `LegalMonetaryTotal` (Valor Venta, Importe Total, monto a pagar).
- Aplicar firma digital con certificado de Golocaer S.A.C. (misma implementación que `FacturaPeruXmlBuilder`).

**Resultado esperado:**
`NCPeruXmlBuilder` genera un XML UBL 2.1 CPE tipo 07 válido que puede ser enviado a SUNAT para su timbrado. El XML es verificable con el schema UBL 2.1 para NC peruanas.

**Entregables:**
- Clase `NCPeruXmlBuilder` en `ProquifaDotNet.Finanzas.Application.Services.NC.Peru`
- Pruebas unitarias con dataset de prueba en ambas modalidades (por partidas y manual)
- Validación contra XSD UBL 2.1 para NC electrónica SUNAT

**Criterios de aceptación:**
- El XML generado tiene `InvoiceTypeCode=07`, `UBLVersionID=2.1`, `CustomizationID=2.0`.
- `DiscrepancyResponse` contiene ReferenceID (serie-correlativo CPE origen), ResponseCode (catálogo 09) y Description.
- `BillingReference` contiene el ID del CPE origen.
- `AccountingSupplierParty` tiene RUC `20612772941` (Golocaer S.A.C.).
- En modalidad por partidas: solo se incluyen líneas con CantNC > 0; datos fiscales heredados correctamente.
- En modalidad manual: un único concepto con el monto capturado.
- IGV 18% calculado correctamente a nivel línea y total.
- Cuando moneda ≠ PEN: `TaxExchangeRate` usa el TC de la fecha de emisión del CPE origen.
- El XML es válido según el XSD SUNAT para NC electrónica UBL 2.1.
- **No aparecen** campos SAT: sin `TipoDeComprobante`, sin `MetodoPago`, sin `FormaPago`, sin `UsoCFDI`.

**Más información:**
Ver sección **Parte A** de `R16A-RE-FU-035-Back.md` — mapa completo de campos UBL 2.1 CPE tipo 07 y diferencias vs `NCMexicoXmlBuilder`.

**Recursos:**
- `R16A-RE-FU-035-Back.md` — Parte A
- `FacturaPeruXmlBuilder` (RE-022) — patrón base UBL 2.1
- Schema UBL 2.1 para Nota de Crédito electrónica SUNAT (Catálogo de documentos electrónicos)

---

## TAREA 3

**[ RE-FU-035 ] [IMP-EXIST-SERVICE] Implementar NCPeruPdfMappingService — Preview y PostTimbrado + Persistencia**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application — Services — NC — Peru — NCPeruPdfMappingService + PersistirNCPeruPdfService

**Consideraciones previas:**
- Patrón base: `FacturaPeruPdfMappingService` (R16A-RE-FU-022). Misma separación Preview / PostTimbrado.
- Prerrequisito: T1 (template `GOL_PER_NC`) debe estar registrado en DocumentBuilder para validar renderizado end-to-end.
- Prerrequisito: RE-033 T5 (DML DocumentTemplate) ejecutado.
- Modo Preview: sin QR SUNAT, sin hash/resumen, con leyenda "PREVISUALIZACIÓN".
- Modo PostTimbrado: con QR SUNAT + valor resumen del XML firmado.
- Ruta MinIO: `notas-credito-per/notas_credito/{anio}/{mes}/{serie}-{correlativo}.pdf` (usa serie-correlativo, no UUID).
- ⚠️ **BLOQUEADO B1 para integración:** El servicio de mapeo y el template pueden desarrollarse y probarse, pero `PersistirNCPeruPdfService` solo puede validarse end-to-end cuando B1 esté resuelto y haya un CPE timbrado real.
- ⚠️ **Formato QR SUNAT (P8):** implementar con URL placeholder; ajustar cuando se confirme el formato exacto.

**Objetivo general:**
Implementar `NCPeruPdfMappingService` que mapea el `NCPeruVm` al `NCPeruPdfModel`, genera el PDF con DocumentBuilder usando el template `GOL_PER_NC`, y lo persiste en MinIO vía `PersistirNCPeruPdfService`.

**Objetivos específicos:**
- Implementar mapeo de todos los campos requeridos por las secciones I1–I8 del requisito RE-035.
- Implementar modo Preview: sin QR, sin hash; leyenda "PREVISUALIZACIÓN — Sin validez fiscal".
- Implementar modo PostTimbrado: con QR SUNAT (URL verificación) y valor resumen del XML firmado.
- Implementar mapeo de la sección Referencia: código catálogo 09 + descripción + CPE origen serie-correlativo.
- Implementar mapeo condicional: tabla de líneas (por partidas) o concepto único (manual).
- Implementar mapeo de totales: Valor Venta, IGV 18%, Importe Total, monto en letras.
- Implementar selección de template: siempre `GOL_PER_NC` (empresa única Perú).
- Implementar `PersistirNCPeruPdfService`: PDF → MinIO `notas-credito-per/notas_credito/{anio}/{mes}/{serie}-{correlativo}.pdf`.
- Implementar `PersistirNCPeruXmlService`: XML → MinIO `notas-credito-per/notas_credito/{anio}/{mes}/{serie}-{correlativo}.xml`.

**Resultado esperado:**
- El wizard muestra PDF Preview correcto en el Paso 2 (sin QR/hash).
- Tras timbrado exitoso, PDF PostTimbrado con QR SUNAT y hash se genera y persiste en MinIO.
- El PDF usa nomenclatura peruana (Valor Venta, IGV, Importe Total) sin campos SAT.

**Entregables:**
- Clase `NCPeruPdfMappingService`
- Clase `PersistirNCPeruPdfService`
- Clase `PersistirNCPeruXmlService`
- Pruebas unitarias en modos Preview y PostTimbrado

**Criterios de aceptación:**
- El PDF Preview muestra leyenda "PREVISUALIZACIÓN — Sin validez fiscal" y no contiene QR ni hash.
- El PDF PostTimbrado contiene QR SUNAT legible y valor resumen del XML.
- El PDF **no contiene** campos SAT: sin UUID, sin sello SAT, sin cadena complementaria, sin MetodoPago, sin FormaPago, sin UsoCFDI.
- Los totales muestran Valor Venta, IGV 18% e Importe Total con nomenclatura SUNAT.
- PDF y XML persisten en MinIO en la ruta `notas-credito-per/notas_credito/{anio}/{mes}/{serie}-{correlativo}.pdf/.xml`.
- El template `GOL_PER_NC` se selecciona siempre (empresa única).

**Más información:**
Ver secciones **Parte B** y **Parte D** de `R16A-RE-FU-035-Back.md`.

**Recursos:**
- `R16A-RE-FU-035-Back.md` — Partes B y D
- `FacturaPeruPdfMappingService` (RE-022) — patrón base
- `PersistirFacturaPeruPdfService` (RE-022) — patrón base MinIO
- Template `GOL_PER_NC` (entregable de T1)
