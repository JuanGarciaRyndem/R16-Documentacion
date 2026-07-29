# Tareas BackEnd — R16A-RE-FU-034
**Requisito:** Diseño y generación de Documentos — NDC México (CFDI tipo E)
**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder

---

## Resumen de tareas

| #  | Clave             | Título simple                                                               | Tipo  | Aplicativo              |
| -- | ----------------- | --------------------------------------------------------------------------- | ----- | ----------------------- |
| 1  | CREATE-PDF        | Template HTML Golocaer NC México: GOL\_MEX\_NC (H/B/F)                      | Back  | DocumentBuilder         |
| 2  | CREATE-PDF        | Template HTML Mungen NC México: MUN\_MEX\_NC (H/B/F)                        | Back  | DocumentBuilder         |
| 3  | CREATE-PDF        | Template HTML Proquifa NC México: PRO\_MEX\_NC (H/B/F)                      | Back  | DocumentBuilder         |
| 4  | CREATE-PDF        | Template HTML Proveedora Quimico Farmaceutica NC México: PQF\_MEX\_NC (H/B/F) | Back | DocumentBuilder         |
| 5  | ALG-COMPLX-LOGIC  | Implementar NCXmlBuilder — generación XML CFDI tipo E                  | Back  | ProquifaDotNet.Finanzas |
| 6  | IMP-EXIST-SERVICE | Implementar NCPdfMappingService — Preview y PostTimbrado               | Back  | ProquifaDotNet.Finanzas |

---

## TAREA 1

**[ RE-FU-034 ] [CREATE-PDF] Template HTML Golocaer NC México — GOL\_MEX\_NC (H/B/F)**

**Aplicativos:** DocumentBuilder

**Módulos:** Documentos — Nota de Crédito México — Template Golocaer

**Consideraciones previas:**
- Consistente con `GOL_MEX_FAC` (Factura México) y `GOL_MEX_CDP` (Complemento de Pago) en branding, tipografía y estructura.
- Prerrequisito: RE-032 T6 insertó el registro `GOL_MEX_NC` en `DocumentTemplate`.
- 3 archivos: `GOL_MEX_NC_H.html` (Header), `GOL_MEX_NC_B.html` (Body), `GOL_MEX_NC_F.html` (Footer).
- ⚠️ **Vigencia iconografía certificaciones** (P6): confirmar qué logos deben aparecer en el Footer antes de entregarlo al cliente.
- ⚠️ **Plantilla correo PMO #31** (P7): el template no bloquea la tarea, pero el envío de correo del wizard sí requiere esa plantilla.

**Objetivo general:**
Diseñar e implementar los 3 archivos HTML del template de Nota de Crédito México para Golocaer (color corporativo naranja), con identidad visual consistente con Factura y CDP México.

**Objetivos específicos:**
- **Header:** Logo Golocaer (naranja), título "NOTA DE CRÉDITO", datos emisor (Razón Social, RFC, Régimen Fiscal), datos comprobante (Serie-Folio, Versión 4.0, Tipo E, Fecha emisión / certificación, UUID).
- **Body:** Bloque receptor, bloque relación CFDI (Motivo NC + TipoRelacion 01 + UUID/Folio origen), tabla de partidas o concepto manual según modalidad, bloque totales (SubTotal, IVA, Total, Total en letra).
- **Footer:** Sellos y trazabilidad SAT (No. Serie CSD SAT, No. Serie CSD emisor, SelloCFD, SelloSAT, CadenaOriginal), QR verificación SAT, iconografía certificaciones, leyenda conservación 5 años (Art. 30 CFF).
- Validar renderizado correcto con datos de prueba del ejemplo B-128.

**Resultado esperado:**
3 archivos HTML `GOL_MEX_NC_H`, `GOL_MEX_NC_B`, `GOL_MEX_NC_F` registrados en DocumentBuilder, renderizando correctamente el PDF de la NC en modo Preview y PostTimbrado.

**Entregables:**
- `GOL_MEX_NC_H.html`
- `GOL_MEX_NC_B.html`
- `GOL_MEX_NC_F.html`
- PDF de prueba generado con datos del ejemplo B-128

**Criterios de aceptación:**
- El PDF renderizado incluye todas las secciones J1–J12 del requisito RE-034.
- El color corporativo naranja de Golocaer es consistente con `GOL_MEX_FAC`.
- El logo de Golocaer es visible y de alta resolución.
- La tabla de partidas muestra correctamente ClaveProdServ, código, descripción, cantidad, unidad, valor unitario, importe e IVA.
- Los totales (SubTotal, IVA, Total, Total en letra) son correctos.
- El QR de verificación SAT es legible y decodifica la URL correcta.
- Los sellos y cadena original se despliegan completos sin truncamiento.

---

## TAREA 2

**[ RE-FU-034 ] [CREATE-PDF] Template HTML Mungen NC México — MUN\_MEX\_NC (H/B/F)**

**Aplicativos:** DocumentBuilder

**Módulos:** Documentos — Nota de Crédito México — Template Mungen

**Consideraciones previas:**
- Consistente con `MUN_MEX_FAC` y `MUN_MEX_CDP` en branding y estructura.
- Prerrequisito: RE-032 T6 insertó el registro `MUN_MEX_NC` en `DocumentTemplate`.
- 3 archivos: `MUN_MEX_NC_H.html`, `MUN_MEX_NC_B.html`, `MUN_MEX_NC_F.html`.
- El Body y Footer son estructuralmente idénticos al de Golocaer; sólo cambia la paleta (verde Mungen) y el logo.
- ⚠️ Mismos pendientes P6 y P7 que T1.

**Objetivo general:**
Diseñar e implementar los 3 archivos HTML del template NC México para Mungen (color corporativo verde).

**Objetivos específicos:**
- Mismos que T1 aplicando la paleta verde y logo de Mungen.
- Validar que el branding sea consistente con `MUN_MEX_FAC`.

**Resultado esperado:**
3 archivos HTML `MUN_MEX_NC_H`, `MUN_MEX_NC_B`, `MUN_MEX_NC_F` funcionando correctamente en DocumentBuilder.

**Entregables:**
- `MUN_MEX_NC_H.html`
- `MUN_MEX_NC_B.html`
- `MUN_MEX_NC_F.html`
- PDF de prueba generado

**Criterios de aceptación:**
- Mismos que T1 adaptados a la paleta verde Mungen.
- El logo de Mungen es correcto y consistente con Factura México.

---

## TAREA 3

**[ RE-FU-034 ] [CREATE-PDF] Template HTML Proquifa NC México — PRO\_MEX\_NC (H/B/F)**

**Aplicativos:** DocumentBuilder

**Módulos:** Documentos — Nota de Crédito México — Template Proquifa

**Consideraciones previas:**
- Consistente con `PRO_MEX_FAC` y `PRO_MEX_CDP` en branding y estructura.
- Prerrequisito: RE-032 T6 insertó el registro `PRO_MEX_NC` en `DocumentTemplate`.
- 3 archivos: `PRO_MEX_NC_H.html`, `PRO_MEX_NC_B.html`, `PRO_MEX_NC_F.html`.
- Paleta cyan Proquifa — validar que coincide exactamente con los valores hex de `PRO_MEX_FAC`.
- ⚠️ Mismos pendientes P6 y P7 que T1.

**Objetivo general:**
Diseñar e implementar los 3 archivos HTML del template NC México para Proquifa S.A. de C.V. (color corporativo cyan).

**Objetivos específicos:**
- Mismos que T1 aplicando la paleta cyan y logo de Proquifa.
- Validar renderizado con el ejemplo B-128 (RFC PRO970821ML3).

**Resultado esperado:**
3 archivos HTML `PRO_MEX_NC_H`, `PRO_MEX_NC_B`, `PRO_MEX_NC_F` funcionando correctamente en DocumentBuilder.

**Entregables:**
- `PRO_MEX_NC_H.html`
- `PRO_MEX_NC_B.html`
- `PRO_MEX_NC_F.html`
- PDF de prueba generado con datos del ejemplo B-128 (empresa emisora Proquifa, USD, TC 19.17)

**Criterios de aceptación:**
- Mismos que T1 adaptados a la paleta cyan Proquifa.
- El PDF de prueba con datos del ejemplo B-128 coincide visualmente con la representación impresa esperada.

---

## TAREA 4

**[ RE-FU-034 ] [CREATE-PDF] Template HTML PQF NC México — PQF\_MEX\_NC (H/B/F)**

**Aplicativos:** DocumentBuilder

**Módulos:** Documentos — Nota de Crédito México — Template PQF

**Consideraciones previas:**
- Consistente con `PQF_MEX_FAC` y `PQF_MEX_CDP` en branding y estructura.
- Prerrequisito: RE-032 T6 insertó el registro `PQF_MEX_NC` en `DocumentTemplate`.
- 3 archivos: `PQF_MEX_NC_H.html`, `PQF_MEX_NC_B.html`, `PQF_MEX_NC_F.html`.
- Paleta cyan PQF — puede compartir los mismos valores hex que Proquifa (confirmar).
- ⚠️ Mismos pendientes P6 y P7 que T1.

**Objetivo general:**
Diseñar e implementar los 3 archivos HTML del template NC México para Proveedora Quimico Farmaceutica (color corporativo cyan).

**Objetivos específicos:**
- Mismos que T1 aplicando la paleta cyan y logo de PQF.
- Validar consistencia con `PQF_MEX_FAC`.

**Resultado esperado:**
3 archivos HTML `PQF_MEX_NC_H`, `PQF_MEX_NC_B`, `PQF_MEX_NC_F` funcionando correctamente en DocumentBuilder.

**Entregables:**
- `PQF_MEX_NC_H.html`
- `PQF_MEX_NC_B.html`
- `PQF_MEX_NC_F.html`
- PDF de prueba generado

**Criterios de aceptación:**
- Mismos que T1 adaptados a la paleta cyan PQF.
- Logo PQF correcto y de alta resolución.

---

## TAREA 5

**[ RE-FU-034 ] [ALG-COMPLX-LOGIC] Implementar NCXmlBuilder — generación XML CFDI tipo E**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application — Services — NC — Mexico — NCXmlBuilder

**Consideraciones previas:**
- Patrón base: `FacturaXmlBuilder` (RE-021). Reutilizar la misma estructura pero adaptada a TipoDeComprobante=E.
- **Diferencias clave vs Factura:** TipoDeComprobante=E, CfdiRelacionados obligatorio (TipoRelacion=01), MetodoPago=PUE fijo, FormaPago heredado, UsoCFDI=G02 default, Conceptos heredados de la factura origen o manual.
- ⚠️ **Pendiente P2 (UsoCFDI):** avanzar con G02; el valor puede ajustarse cuando se confirme.
- ⚠️ **Pendiente P3 (Descuento):** implementar sin campo Descuento en el comprobante root (basado en B-128); ajustar si el asesor fiscal confirma que debe poblarse.
- ⚠️ **Pendiente P4 (ObjetoImp modalidad manual):** usar el mismo ObjetoImp heredado de la factura origen hasta confirmar.
- ⚠️ **Pendiente P5 (Serie foliador):** usar la serie configurada en `EmpresaFolio`; confirmar nombre definitivo.
- Prerrequisito funcional: RE-032 T7 (endpoint timbrado NC en Timbrado) debe estar disponible para la integración end-to-end.

**Objetivo general:**
Implementar `NCXmlBuilder` que recibe el `NCDto` y genera el `XDocument` CFDI 4.0 tipo E conforme al mapa de campos documentado en R16A-RE-FU-034-Back.md, validado contra el ejemplo real B-128.

**Objetivos específicos:**
- Implementar el comprobante root con todos los atributos CFDI 4.0 (Version, TipoDeComprobante=E, Exportacion, Serie, Folio, Fecha, LugarExpedicion, Moneda, TipoCambio, MetodoPago=PUE, FormaPago, SubTotal, Total).
- Implementar el nodo `CfdiRelacionados` con TipoRelacion=01 y UUID de la factura origen.
- Implementar nodo `Emisor` (RFC, Nombre, RegimenFiscal=601).
- Implementar nodo `Receptor` (RFC, Nombre, DomicilioFiscalReceptor, RegimenFiscalReceptor, UsoCFDI=G02).
- Implementar nodo `Conceptos` en modalidad por partidas: un `Concepto` por partida con CantNC > 0, heredando datos fiscales y recalculando importes.
- Implementar nodo `Conceptos` en modalidad manual: un único `Concepto` con ClaveProdServ=84111506, ClaveUnidad=ACT, Cantidad=1.
- Implementar cálculo automático de `Impuestos` (IVA 16%) a nivel concepto y a nivel comprobante.
- Calcular SubTotal y Total correctamente.
- Serializar el XML con el namespace SAT correcto (`cfdi:Comprobante`).
- Validar el XML generado contra el XSD oficial del SAT para CFDI 4.0 tipo E.

**Resultado esperado:**
`NCXmlBuilder` genera un XML CFDI 4.0 tipo E válido que TurboPac puede timbrar exitosamente. El XML generado con los datos del ejemplo B-128 produce SubTotal=48.00, Total=55.68, IVA=7.68, Moneda=USD, TipoCambio=19.17.

**Entregables:**
- Clase `NCXmlBuilder` en `ProquifaDotNet.Finanzas.Application.Services.NC.Mexico`
- Pruebas unitarias con el dataset del ejemplo B-128 (ambas modalidades: por partidas y manual)
- Validación contra XSD SAT CFDI 4.0

**Criterios de aceptación:**
- El XML generado con datos del ejemplo B-128 coincide atributo a atributo con el XML real (excepto Sello, NoCertificado, Certificado — generados por TurboPac).
- `TipoDeComprobante=E` fijo en todos los casos.
- `CfdiRelacionados` siempre presente con TipoRelacion=01 y UUID de la factura origen.
- `MetodoPago=PUE` fijo en todos los casos.
- `FormaPago` hereda correctamente de la factura origen.
- En modalidad por partidas: sólo se incluyen conceptos con CantNC > 0.
- En modalidad manual: un único concepto con ClaveProdServ=84111506.
- IVA calculado correctamente a nivel concepto y totales comprobante.
- El XML es válido según el XSD oficial SAT para CFDI 4.0.

**Más información:**
Ver sección **Parte A** de `R16A-RE-FU-034-Back.md` — mapa completo de campos CFDI 4.0 tipo E y validación contra ejemplo B-128.

**Recursos:**
- `R16A-RE-FU-034-Back.md` — Parte A: Mapa de campos CFDI 4.0 tipo E
- `FacturaXmlBuilder` — patrón base
- XSD SAT CFDI 4.0 (Apéndice 5 Anexo 20)
- Ejemplo B-128 entregado por el cliente

---

## TAREA 6

**[ RE-FU-034 ] [IMP-EXIST-SERVICE] Implementar NCPdfMappingService — Preview y PostTimbrado**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Application — Services — NC — Mexico — NCPdfMappingService

**Consideraciones previas:**
- Patrón base: `InvoicePdfMappingService` (RE-021). Misma separación Preview / PostTimbrado.
- Prerrequisito: T1–T4 (templates HTML) deben estar registrados en `DocumentTemplate` para validar el renderizado end-to-end.
- Prerrequisito: RE-032 T6 (DML `DocumentTemplate`) debe estar ejecutado.
- El modo Preview no incluye UUID, sellos ni QR; el modo PostTimbrado sí los incluye.
- El QR SAT codifica: `https://verificacfdi.facturaelectronica.sat.gob.mx/default.aspx?id={UUID}&re={RFCEmisor}&rr={RFCReceptor}&tt={Total}&fe={últimos8SelloCFD}`.
- `PersistirNCPdfService` usa el mismo patrón que `PersistInvoicePdfService` — rutas MinIO: `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID}.pdf`.

**Objetivo general:**
Implementar `NCPdfMappingService` que mapea el `NCVm` al `NCPdfModel` y lo pasa a DocumentBuilder para generar el PDF, en modo Preview (wizard Paso 2) y PostTimbrado (confirmación exitosa).

**Objetivos específicos:**
- Implementar mapeo de todos los campos requeridos por las secciones J1–J12 del requisito RE-034.
- Implementar modo Preview: sin UUID, sellos, QR; con leyenda "PREVISUALIZACIÓN — Sin validez fiscal".
- Implementar modo PostTimbrado: con UUID, FechaTimbrado, SelloCFD, SelloSAT, NoCertificadoSAT, CadenaOriginal y QR.
- Implementar generación del QR de verificación SAT con los 5 parámetros requeridos.
- Implementar mapeo de la sección Relación CFDI: Motivo NC, TipoRelacion 01, Folio + UUID factura origen.
- Implementar mapeo condicional de conceptos: tabla de partidas (modalidad por partidas) o concepto único (modalidad manual).
- Implementar mapeo de totales: SubTotal, IVA, Total, Total en letra.
- Implementar selección del template correcto según empresa emisora (GOL/MUN/PRO/PQF).
- Implementar `PersistirNCPdfService` para guardar el PDF en MinIO bajo `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID}.pdf`.

**Resultado esperado:**
- El wizard muestra un PDF Preview correcto en el Paso 2 antes de timbrar.
- Tras timbrado exitoso, el PDF PostTimbrado con UUID y QR se genera y persiste en MinIO correctamente.
- El PDF generado con datos del ejemplo B-128 (Proquifa, USD, TC 19.17) es visualmente correcto.

**Entregables:**
- Clase `NCPdfMappingService`
- Clase `PersistirNCPdfService`
- Pruebas unitarias: modo Preview y PostTimbrado con dataset del ejemplo B-128
- PDF de prueba generado en ambos modos

**Criterios de aceptación:**
- El PDF Preview muestra leyenda "PREVISUALIZACIÓN — Sin validez fiscal" y no contiene UUID ni QR.
- El PDF PostTimbrado contiene UUID, QR legible, sellos completos y cadena original sin truncamiento.
- El QR decodifica la URL SAT correcta con los 5 parámetros.
- El PDF aplica el template correcto según la empresa emisora.
- El PDF persiste en MinIO en la ruta `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID}.pdf`.
- Todas las secciones J1–J12 del criterio RE-034 están presentes y correctas.

**Más información:**
Ver sección **Parte B** y **Parte D** de `R16A-RE-FU-034-Back.md`.

**Recursos:**
- `R16A-RE-FU-034-Back.md` — Partes B y D
- `InvoicePdfMappingService` — patrón base
- `PersistInvoicePdfService` — patrón base para MinIO
- Templates `GOL/MUN/PRO/PQF_MEX_NC` (entregables de T1–T4)
