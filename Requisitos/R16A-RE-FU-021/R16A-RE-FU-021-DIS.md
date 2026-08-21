#  

# 

# 

# 

# 

# 

# **Diseño de la Solución**

## R16A-RE-FU-021

## PDF de Factura CFDI 4.0 (México)

| FORMATO | Análisis |
| :---- | :---- |
| **PROYECTO** | R16 |
| **REFERENCIA** |  |
| **VERSIÓN** | 1.0 |
| **FECHA** | 17/07/2026 |
| **AUTOR** | [Cristóbal Sebastián García Coss](mailto:cristobal.garcia@ryndem.mx) |
| **REVISOR** | [Juan David García Cruz](mailto:juan.garcia@ryndem.mx) |

## 

## Contenido

[1\. Introducción	3](#heading=)

[1.1 Propósito	3](#heading=)

[1.2 Alcance — qué incluye	3](#heading=)

[1.3 Alcance — qué NO incluye	3](#heading=)

[2\. Visión general	3](#heading=)

[2.1 Objetivo técnico	3](#heading=)

[2.2 Componentes involucrados	3](#heading=)

[3\. Diseño funcional detallado	5](#heading=)

[3.1 Criterios de aceptación (CA) — reconstruidos de las secciones A–J	5](#heading=)

[3.2 Flujo A — Preparación de datos (BD) · tickets R16A-1555 (T1–T4)	5](#heading=)

[3.3 Flujo B — Plantillas PDF · tickets R16A-1555 T5, R16A-1556 (T6–T7), R16A-1557 (T8)	6](#heading=)

[3.4 Flujo C — Mapeo · ticket R16A-1558 (T9)	6](#heading=)

[3.5 Flujo D — Generación y persistencia · tickets R16A-1559 (T10–T11), R16A-1560 (T12–T13)	6](#heading=)

[4\. Diseño de componentes	7](#heading=)

[4.1 Arquitectura de componentes	7](#heading=)

[4.2 Secuencia — Validar Cobro → timbrado → persistencia (flujo definitivo)	8](#heading=)

[4.3 Secuencia — Previsualización (sin timbrado)	9](#heading=)

[5\. Impacto técnico	9](#heading=)

[5.1 Componentes nuevos	9](#heading=)

[5.2 Componentes a modificar	9](#heading=)

[5.3 Dependencias afectadas	9](#heading=)

[6\. Manejo de errores	10](#heading=)

[7\. Estrategia de pruebas	10](#heading=)

[7.1 Unitarias	10](#heading=)

[7.2 Integración	11](#heading=)

[7.3 Funcionales / casos críticos	11](#heading=)

[7.4 Matriz de trazabilidad CA → prueba	11](#heading=)

[Auto-chequeo (Definition of Ready)	11](#heading=)

[Riesgos abiertos (⚠️ — no resolver por asunción)	11](#heading=)

[Control de versiones	13](#heading=)

# 

## **1\. Introducción**

### **1.1 Propósito**

Diseño backend para generar el **PDF de la Factura CFDI 4.0 México** al timbrado exitoso, con branding por empresa emisora (GOL/MUN/PRO/PQF), y persistir junto al XML como artefacto fiscal inmutable.   
1.2 Alcance  
Incluye:

* 4 plantillas HTML de DocumentBuilder {GOL|MUN|PRO|PQF}\_MEX\_FAC (Header/Body/Footer).  
* DTOs de plantilla en inglés (InvoicePdfModel, InvoicePdfItemModel, InvoicePdfItemTaxModel, InvoicePdfBankAccountModel, InvoicePdfTaxRetentionModel).  
* InvoicePdfMappingService para la consolidación de datos del CFDI 4.0.  
* Servicio único PersistInvoicePdfService que consolida la generación (renderizado vía DocumentBuilder) y persistencia (Minio \+ BD Finanzas), reutilizable desde los puntos de timbrado.  
* Integración en los dos puntos de facturación: Factura por Adelantado (AdvanceInvoiceGenerateService / RE-FU-019) y Validar Cobro — Genera Factura Normal PPD (RE-FU-020).  
* Previsualización del PDF en memoria (AdvanceInvoicePreviewService, sin timbrado ni persistencia).  
* Consulta y entrega del PDF almacenado en Minio (artefacto inmutable, sin regeneración).

### **No incluye:**

* Cambios de esquemas de BD en catálogos SAT (c\_ClaveProdServ, catUnidad, exportacion — corresponden a prerrequisitos de catálogos base o RE-FU-018/019).  
* Generación del XML CFDI ni timbrado ante el PAC (resuelto por el aplicativo de Timbrado /api/v1/stamp/invoice).  
* Notas de Crédito / cancelación SAT.  
* Sección K (Factura Anticipo) — fuera de este diseño.  
* Perú (SUNAT / UBL 2.1 \- RE-FU-022): Perú no timbra y utiliza el esquema de Proforma (quedó fuera de alcance de R16). **[2026-08-21 · DUDA-052]** Confirmado: se cancela por completo la facturación de Perú en R16 (no solo se restringe a Prepago); no hay desarrollo de RE-FU-022 en este release.

1.3 Pendientes y aclaraciones que condicionan el diseño

| ID | Pendiente / Aclaración | Impacto |
| :---- | :---- | :---- |
| **PA-1** | Origen de datos de partidas (Clave SAT, ID interno, ClaveUnidad...) | ⚠️ **Bloqueante para el mapping por campo** — la arquitectura no cambia, el diccionario de mapeo sí. |
| **PA-2** | Referencia bancaria del cliente | **Resuelto** [2026-08-21 · DUDA-059]: Se reutiliza directamente la lógica de referencia bancaria (Banamex/no-Banamex) ya resuelta e implementada en el módulo de Proforma (compartida con Perú); no hay reglas propias para la Factura. |
| **PA-5** | Certificaciones/logos vigentes por emisora | Assets de las plantillas. |
| DEP-018/019 | Tablas CFDI/Archivo en BD Finanzas, bucket facturas, Aplicativo de Timbrado (/api/v1/stamp/invoice) | Prerrequisito de infraestructura. |

## **2\. Visión general**

### **2.1 Objetivo técnico**

Al recibir la confirmación de timbrado exitoso desde el aplicativo de timbrado (/api/v1/stamp/invoice con TimbreFiscalDigital), mapear el XML timbrado \+ datos del pedido a InvoicePdfModel, renderizar el PDF vía DocumentBuilder (Puppeteer HTML→PDF) con la plantilla de la emisora, y persistir PDF+XML en Minio con registro Archivo y referencia en CFDI mediante el servicio consolidado PersistInvoicePdfService. La consulta posterior entrega SIEMPRE el binario almacenado.

### **2.2 Componentes involucrados**

| Aplicativo | Clase / Módulo | Responsabilidad | Ubicación | Estado |
| :---- | :---- | :---- | :---- | :---- |
| DocumentBuilder (BD) | DocumentTemplate × 4 (GOL/MUN/PRO/PQF\_MEX\_FAC) | Plantillas HTML H/B/F de Factura por emisora. | BD DocumentBuilder | **NUEVO** (INSERT) |
| DocumentBuilder | RenderDocumentService / ConvertDocumentService (Puppeteer) | Render HTML→PDF con paginación (header/footer repetidos). | Application\\Services\\Pdf\\ | Sin cambios |
| ProquifaDotNet.Finanzas (.NET Core) | InvoicePdfModel (+ InvoicePdfItemModel, InvoicePdfItemTaxModel, etc.) | DTOs unificados en inglés para transporte de datos a la plantilla PDF. | Finanzas.Models.Pdf | **NUEVO** |
| ProquifaDotNet.Finanzas (.NET Core) | InvoicePdfMappingService | XML CFDI timbrado \+ TimbreFiscalDigital \+ datos pedido/emisora → InvoicePdfModel. | Finanzas.Services.Pdf | **NUEVO** |
| ProquifaDotNet.Finanzas (.NET Core) | PersistInvoicePdfService | Servicio único consolidado: invoca DocumentBuilder para generar PDF y lo persiste en Minio (facturas) \+ BD Finanzas (Archivo ×2, referencia en CFDI). | Finanzas.Services.Pdf | **NUEVO** |
| ProquifaDotNet.Finanzas (.NET Core) | AdvanceInvoiceGenerateService / AdvanceInvoicePreviewService | Orquestación post-timbrado y previsualización. | Finanzas.Services.Invoice | **MODIFICADO** |
| ProquifaDotNet.Timbrado | Aplicativo REST de Timbrado (/api/v1/stamp/invoice) | Emisión y timbrado CFDI 4.0 ante el PAC. | ProquifaDotNet.Timbrado | Existente |
| BD ProquifaDotNet.Finanzas | Tablas CFDI / CFDIGenerada / Archivo | Almacenamiento de metadatos fiscales y referencias de archivo. | BD Finanzas | Ubicación en Finanzas |

## **3\. Diseño funcional detallado**

### **3.1 Flujo — Generación del PDF al timbrado exitoso**

| Paso | Acción |
| ----- | :---- |
| **1** | Aplicativo de Timbrado (/api/v1/stamp/invoice) retorna éxito \+ XML con TimbreFiscalDigital. |
| **2** | Orquestador (AdvanceInvoiceGenerateService / Validar Cobro) invoca InvoicePdfMappingService.Map(xmlTimbrado, pedido, emisora) → InvoicePdfModel (secciones A-J: emisor, receptor, CFDI, bancarias \[lógica reutilizada de Proforma\], partidas InvoicePdfItemModel, impuestos por partida InvoicePdfItemTaxModel, totales, técnicos SAT, institucional). |
| **3** | Invocación a PersistInvoicePdfService.PersistAsync(idCfdi, invoicePdfModel, xmlTimbrado, templateKey): • **a)** Invoca DocumentBuilder para renderizar HTML→PDF (Puppeteer): QR de validación SAT generado desde URL de verificación \+ paginación automática "X de Y" (CA-10). • **b)** PUT PDF en Minio bucket 'facturas'. • **c)** PUT XML en Minio (si no lo persistió ya el flujo previo). • **d)** INSERT Archivo ×2 (FileBucket='facturas'). • **e)** UPDATE CFDI en BD Finanzas con referencias (IdArchivoXml / IdArchivoPdf). |
| **4** | Consulta posterior: GET del PDF → SIEMPRE binario de Minio vía Archivo (sin regeneración — CA-11). |

3.2 Estructura de plantillas (por emisora)

| Segmento | Contenido (CAs) |
| :---- | :---- |
| \*\_MEX\_FAC\_H.html | Logo emisora, datos institucionales del emisor, identificadores CFDI (Serie/Folio/UUID/Versión), fechas emisión/certificación (CA-1, CA-3) |
| \*\_MEX\_FAC\_B.html | Receptor CFDI 4.0 (CP \+ Régimen — CA-2), datos fiscales (PPD/99/Exportación — CA-4), tabla de partidas (InvoicePdfItemModel, sin lote — D3), impuestos por partida (InvoicePdfItemTaxModel), totales/traslados/retenciones/total en letra (CA-7), referencias bancarias (InvoicePdfBankAccountModel, reutilizando lógica Banamex/no-Banamex de Proforma — CA-5) |
| \*\_MEX\_FAC\_F.html | Técnicos SAT: QR, sellos, cadena original, series de certificados (CA-8); disclaimer "Representación impresa de un CFDI 4.0", certificaciones/logos (CA-9 ⚠️ PA-5), "X de Y" |

## **4\. Diseño de componentes**

### **4.1 Arquitectura de componentes**

flowchart TD  
    subgraph FIN\["ProquifaDotNet.Finanzas (.NET Core)"\]  
        SVC\["Invoice Services (AdvanceInvoiceGenerate / ValidarCobro)"\]  
        MAP\["InvoicePdfMappingService"\]  
        PER\["PersistInvoicePdfService (Genera & Persiste)"\]  
        CFDI\_DB\[("BD Finanzas: CFDI, Archivo")\]  
    end  
    subgraph STAMP\["ProquifaDotNet.Timbrado"\]  
        API\_STAMP\["API Timbrado (/api/v1/stamp/invoice)"\]  
    end  
    subgraph DOCB\["DocumentBuilder Service"\]  
        DOCB\_RENDER\["Render (Puppeteer HTML \-\> PDF)"\]  
        DOCB\_TMPL\["Templates (\*\_MEX\_FAC)"\]  
    end  
    subgraph INFRA\["Infraestructura"\]  
        MINIO\[("Minio Bucket: facturas")\]  
    end  
    SVC \--\>|"1. Solicitud de Timbrado"| API\_STAMP  
    API\_STAMP \--\>|"2. XML Timbrado \+ TimbreFiscalDigital"| SVC  
    SVC \--\>|"3. Mapeo a InvoicePdfModel"| MAP  
    SVC \--\>|"4. Invoca PersistAsync(model, xml)"| PER  
    PER \--\>|"5. Render HTML"| DOCB\_RENDER  
    DOCB\_TMPL \-.-\> DOCB\_RENDER  
    PER \--\>|"6. Guardar binarios"| MINIO  
    PER \--\>|"7. Registrar Archivo y referencia"| CFDI\_DB  
![][image1]

### **4.2 Diagrama de Secuencia — Flujo de Generación y Persistencia de PDF**

sequenceDiagram  
    participant SVC as Invoice Service (AdvanceInvoiceGenerate)  
    participant STAMP as Timbrado API (/api/v1/stamp/invoice)  
    participant MAP as InvoicePdfMappingService  
    participant PER as PersistInvoicePdfService  
    participant DB as DocumentBuilder  
    participant STORE as Minio \+ BD Finanzas  
    SVC-\>\>STAMP: POST /api/v1/stamp/invoice  
    STAMP--\>\>SVC: 200 OK \+ XML Timbrado (TimbreFiscalDigital)  
    SVC-\>\>MAP: Map(xmlTimbrado, datosPedido, emisora)  
    MAP--\>\>SVC: InvoicePdfModel (DTO en Inglés)  
    SVC-\>\>PER: PersistAsync(idCfdi, invoicePdfModel, xmlTimbrado, templateKey)  
    PER-\>\>DB: RenderHtmlToPdf({Prefijo}\_MEX\_FAC, model)  
    DB--\>\>PER: PDF bytes  
    PER-\>\>STORE: PUT Minio \+ INSERT Archivo \+ UPDATE CFDI  
    STORE--\>\>PER: OK  
    PER--\>\>SVC: Persistencia Exitosa  
![][image2]

### **4.3 Contratos e Interfaces C\#**

```c#
namespace ProquifaDotNet.Finanzas.Services.Pdf
{
    public interface IInvoicePdfMappingService
    {
        /// <summary>
        /// Mapea el XML timbrado y los datos del pedido al modelo unificado del PDF.
        /// </summary>
        InvoicePdfModel Map(CfdiTimbrado xmlTimbrado, DatosPedidoFactura pedido, EmpresaEmisora emisora);
    }

    public interface IPersistInvoicePdfService
    {
        /// <summary>
        /// Genera el PDF mediante DocumentBuilder y persiste PDF+XML en Minio y BD Finanzas.
        /// </summary>
        Task<PersistenciaFacturaResult> PersistAsync(Guid idCfdi, InvoicePdfModel model, string xmlTimbrado, string templateKey);
    }
}
```

### **Modelos DTO (Inglés)**

```c#
namespace ProquifaDotNet.Finanzas.Models.Pdf
{
    public class InvoicePdfModel
    {
        public HeaderModel Header { get; set; }
        public IssuerModel Issuer { get; set; }
        public ReceiverModel Receiver { get; set; }
        public List<InvoicePdfItemModel> Items { get; set; } = new();
        public InvoicePdfBankAccountModel BankAccount { get; set; }
        public TotalsModel Totals { get; set; }
        public SatTechnicalDataModel SatTechnicalData { get; set; }
    }

    public class InvoicePdfItemModel
    {
        public int ItemIndex { get; set; }
        public string SatProductServiceKey { get; set; }
        public string InternalCode { get; set; }
        public string Description { get; set; }
        public decimal Quantity { get; set; }
        public string SatUnitKey { get; set; }
        public decimal UnitPrice { get; set; }
        public decimal Amount { get; set; }
        public List<InvoicePdfItemTaxModel> Taxes { get; set; } = new();
    }

    public class InvoicePdfItemTaxModel
    {
        public string TaxType { get; set; } // IVA, ISR, IEPS
        public string FactorType { get; set; } // Tasa, Cuota, Exento
        public decimal RateOrQuota { get; set; }
        public decimal BaseAmount { get; set; }
        public decimal TaxAmount { get; set; }
        public bool IsRetention { get; set; }
    }

    public class InvoicePdfBankAccountModel
    {
        public string BankName { get; set; }
        public string AccountNumber { get; set; }
        public string Clabe { get; set; }
        public string Reference { get; set; } // Lógica Banamex/no-Banamex reutilizada de Proforma
    }
}
```

## **5\. Impacto técnico**

### **5.1 Componentes nuevos**

* **Modelos DTO (Inglés):** InvoicePdfModel, InvoicePdfItemModel, InvoicePdfItemTaxModel, InvoicePdfBankAccountModel, InvoicePdfTaxRetentionModel.  
* **Servicios backend** (ProquifaDotNet.Finanzas):  
  * InvoicePdfMappingService (+ IInvoicePdfMappingService)  
  * PersistInvoicePdfService (+ IPersistInvoicePdfService — consolida generación y persistencia)  
* **Plantillas HTML DocumentBuilder:** 4 conjuntos de plantillas {GOL|MUN|PRO|PQF}\_MEX\_FAC (Header, Body, Footer).

### **5.2 Componentes a modificar**

* **Orquestación de servicios en Finanzas:**  
  * AdvanceInvoiceGenerateService (pasos de generación de PDF post-timbrado).  
  * AdvanceInvoicePreviewService (previsualización con InvoicePdfModel).  
  * Servicio de Validar Cobro ("Genera Factura Normal PPD").  
* **Integración con Timbrado:** Invocar /api/v1/stamp/invoice de ProquifaDotNet.Timbrado.

## **6\. Manejo de errores**

| Escenario | Comportamiento esperado |
| :---- | :---- |
| API Timbrado inaccesible / error 500 | Sin timbrado exitoso no se inicia la generación del PDF. |
| Timbrado OK, render PDF falla | Estado "timbrada sin PDF". Reintento asíncrono desde el XML timbrado. NUNCA se vuelve a timbrar. |
| Plantilla de emisora inexistente | Error controlado: "Plantilla {prefijo}\_MEX\_FAC no registrada". |
| Fallo en persistencia en Minio / BD | Proceso idempotente con reintento determinístico por UUID. |
| Consulta de PDF en proceso | Retorna estatus "en generación", sin gatillar regeneración en caliente. |
| API Timbrado inaccesible / error 500 | Sin timbrado exitoso no se inicia la generación del PDF. |
| Timbrado OK, render PDF falla | Estado "timbrada sin PDF". Reintento asíncrono desde el XML timbrado. NUNCA se vuelve a timbrar. |
| Plantilla de emisora inexistente | Error controlado: "Plantilla {prefijo}\_MEX\_FAC no registrada". |

## **7\. Estrategia de pruebas**

### **7.1 Unitarias**

* InvoicePdfMappingService: mapeo de cada sección con los 4 CFDIs reales (GOL 7156, MUN 2374, PRO 20913, PQF 143103).  
* Modelos de partidas e impuestos (InvoicePdfItemModel, InvoicePdfItemTaxModel).  
* Referencia bancaria: Verificación de la lógica Banamex/no-Banamex reutilizada de Proforma (aplicable también a Perú).  
* Generación de QR con formato SAT (UUID \+ RFC emisor \+ RFC receptor \+ total).

### **7.2 Integración**

* Flujo completo desde timbrado (/api/v1/stamp/invoice) hasta persistencia del PDF en Minio y actualización de referencias en CFDI en BD Finanzas.  
* Previsualización en memoria sin alterar base de datos ni bucket.

## **Auto-chequeo (Definition of Ready)**

• \[x\] ¿Sé qué flujo implementar? — Flujos A–D \+ integraciones documentados.  
• \[x\] ¿Sé qué pasa si algo falla? — Escenarios E-1…E-8.  
• \[x\] ¿Sé qué reglas no puedo romper? — Reglas técnicas por flujo (inmutabilidad, no re-timbrar, ALTER NULL, TemplateKey por empresa).  
• \[x\] ¿Sé qué pruebas debo pasar? — Sección 7 \+ matriz CA→prueba.  
• \[x\] ¿Sé dónde impacta mi cambio? — Sección 5\.  
• \[x\] ¿Sé qué NO debo hacer / qué quedó fuera? — Sección 1.3.

# **Control de versiones**

| Versión | Fecha | Autor | Tipo de Cambio | Descripción del cambio | Aprobó |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 1.0 | 17/07/2026 | Cristóbal García | Creación | Creación del documento. |  |
|   2.0 | 22/07/2026 | Cristóbal García | Corrección | Atención a todos los comentarios de revisión |  |
|  |  |  |  |  |  |
