# Impacto en Back — TPSC-RE-FU-016
**Requisito:** Diseño y generación de Documentos: Proforma México
**Aplicativo:** ProquifaDotNet + DocumentBuilder
**Módulo:** L05.TramitarPedido + DocumentBuilder (servicio externo)
**Impacto:** Nuevo endpoint de generación de PDF de Proforma para México + armado del DTO + template en DocumentBuilder + persistencia post-envío

---

## Resumen

Este requisito implementa la **generación del PDF de Proforma** para pedidos Prepago sin FAA de clientes con Región México. El PDF se genera vía el servicio externo **DocumentBuilder** (ya integrado). El Back es responsable de:

1. **Armar el DTO** con todos los datos del documento (de múltiples tablas/catálogos)
2. **Enviar al DocumentBuilder** para generar el PDF
3. **Retornar el PDF** para previsualización (sin persistir)
4. **Persistir el PDF** en BD tras confirmación de envío exitoso
5. **Consulta histórica** del PDF desde Validar Cobro

El branding varía por empresa emisora (4 empresas, 3 variantes visuales).

---

## Análisis del DocumentBuilder (Servicio Externo)

### Arquitectura del DocumentBuilder
- **Tecnología:** .NET (Clean Architecture: API / Application / Domain / Infrastructure)
- **Endpoint genérico:** `POST api/Report/generic`
- **DTO de entrada:** `DocumentGenerationAnonymousDto`
  - `FileName`: nombre del archivo PDF
  - `TemplateKey`: clave del template (busca en BD + carpeta de templates)
  - `ClientName`: nombre del cliente consumidor
  - `Base64Images`: diccionario de imágenes en base64 (logos)
  - `Data`: **object** (cualquier estructura JSON que el template HTML consume)
- **Respuesta:** `byte[]` (PDF generado)

### Flujo interno del DocumentBuilder
```
1. Recibe DTO con TemplateKey + Data (JSON)
2. Busca template en BD (DocumentTemplate) por TemplateKey
3. Lee archivos HTML: Header (_H.html), Body (_B.html), Footer (_F.html)
4. Serializa Data a JSON
5. Renderiza HTML con datos JSON (motor de templates)
6. Convierte HTML renderizado a PDF
7. Retorna byte[] del PDF
```

### Templates existentes (por TemplateKey)
| TemplateKey | Empresa | Tipo | Archivos |
|-------------|---------|------|----------|
| GOL_MEX_COT | Golocaer | Cotización México | H, B, F |
| GOL_MEX_PED | Golocaer | Pedido México | H, B, F |
| MUN_MEX_COT | Mungen | Cotización México | H, B, F |
| MUN_MEX_PED | Mungen | Pedido México | H, B, F |
| PQF_MEX_COT | Proquifa | Cotización México | H, B, F |
| PQF_MEX_PED | Proquifa | Pedido México | H, B, F |
| PRO_MEX_COT | Proveedora | Cotización México | H, B, F |
| PRO_MEX_PED | Proveedora | Pedido México | H, B, F |
| PHS_MEX_COT | PHS? | Cotización México | H, B, F |
| PHS_MEX_PED | PHS? | Pedido México | H, B, F |
| GOLPERU_PER_COT | Golocaer | Cotización Perú | H, B, F |
| GOLPERU_PER_PED | Golocaer | Pedido Perú | H, B, F |

### Templates que se deben CREAR para Proforma
| TemplateKey (propuesto) | Empresa | Tipo |
|-------------------------|---------|------|
| GOL_MEX_PRO | Golocaer | Proforma México |
| MUN_MEX_PRO | Mungen | Proforma México |
| PQF_MEX_PRO | Proquifa | Proforma México |
| PRO_MEX_PRO | Proveedora | Proforma México |

### Variantes visuales confirmadas (de imágenes reales)
| Variante | Empresas | Logo | Color | Banco |
|----------|----------|------|-------|-------|
| 1 | Golocaer | Logo propio (G) | Naranja | BANORTE |
| 2 | Mungen | Logo manuscrito "Mungen" | Verde | BANAMEX |
| 3 | Proquifa + Proveedora | Logo PROQUIFA | Teal | BANAMEX |

> **Nota:** Proquifa y Proveedora comparten logo y color pero difieren en razón social, dirección y números de cuenta.

---

## Código Existente Relacionado (ProquifaDotNet)

### Cliente HTTP al DocumentBuilder
`Logic.Pqf.Catalogos\ApiCaller\ApiCallerDocumentBuilder.cs`

> - Método `EnvioPost(string url, object parametros)` -> retorna `byte[]` (PDF)
> - Base URL: `AppSettings["DocumentBuilder:Url"]`
> - Envía POST con JSON, espera `application/pdf`
> - Patrón de consumo: serializa objeto a JSON, envía, recibe bytes

### Endpoints existentes en DocumentBuilder
| Endpoint | DTO | Uso actual |
|----------|-----|------------|
| `POST api/Report/generic` | `DocumentGenerationAnonymousDto` | Documentos genéricos |
| `POST api/Report/quotation` | `DocumentGenerateQuoationDto` | Cotizaciones |
| `POST api/Report/orderConfirmation` | `DocumentGenerationOrderConfirmationDto` | Confirmación de pedido |

> **Para Proforma:** Se puede usar el endpoint `generic` con un `Data` tipado para proforma, o crear un endpoint dedicado `POST api/Report/proforma`.

---

## DTO de Proforma México (Data a enviar)

### Estructura propuesta del objeto Data

```json
{
  "header": {
    "title": "Proforma",
    "folio": "PRF-031826-691",
    "vigencia": "17/04/2026",
    "disclaimer": "ESTE ES UN DOCUMENTO INFORMATIVO..."
  },
  "cliente": {
    "nombre": "LABORATORIO RAAM DE SAHUAYO"
  },
  "partidas": [
    {
      "numero": "1",
      "cantidad": "1",
      "descripcion": "978-607-460-641-6 Versión Impresa...",
      "precioUnitario": "$ 2,972.00",
      "importe": "$ 2,972.00"
    }
  ],
  "pago": {
    "subTotal": "$ 2,972.00 M.N.",
    "ivaPorcentaje": "0 %",
    "iva": "$ 0.00 M.N.",
    "granTotal": "$ 2,972.00 M.N.",
    "granTotalEnLetra": "(DOS MIL NOVECIENTOS SETENTA Y DOS PESOS 00/100 M.N.)",
    "tipoCambio": "",
    "condicionesDePago": "PREPAGO 100%",
    "leyendaExhibicion": "PAGO EN UNA SOLA EXHIBICIÓN"
  },
  "datosBancarios": {
    "cuentaMN": {
      "moneda": "M.N.",
      "banca": "BANORTE",
      "sucursal": "069",
      "cuenta": "0017616395",
      "clabe": "072180001761639522",
      "refCliente": "LABORATORIO RAAM DE..."
    },
    "cuentaDLS": {
      "moneda": "DLS",
      "banca": "BANORTE",
      "sucursal": "069",
      "cuenta": "0090060296",
      "clabe": "072180009006029644",
      "refCliente": "LABORATORIO RAAM DE..."
    }
  },
  "facturacion": {
    "rfc": "LRS030905Q16",
    "razonSocial": "LABORATORIO RAAM DE SAHUAYO",
    "direccionFiscal": "BOULEVARD LAZARO CARDENAS SUR # 794..."
  },
  "entrega": {
    "pedido": "147",
    "parciales": "NO",
    "contacto": "NANCY HERNANDEZ",
    "lugar": "BOULEVARD LAZARO CARDENAS No. 794..."
  }
}
```

### Fuentes de datos por sección
| Sección | Tablas consultadas |
|---------|-------------------|
| Header | Empresa, tpProformaPedido |
| Cliente | Cliente (Alias o RazonSocial) |
| Partidas | tpProformaPartidaPedido, tpPartidaPedido, Producto, MarcaFamilia |
| Pago | tpProformaPedido, catMoneda, catCondicionesDePago, tpPedido |
| Datos Bancarios | EmpresaDatosBancarios, DatosBancarios, catBanco + lógica Código Validador |
| Facturación | DatosFacturacionCliente |
| Entrega | tpPedido, DireccionCliente, ContactoCliente |

---

## Gaps de Desarrollo

### En ProquifaDotNet (Back consumidor)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Crear DTO/Modelo de Proforma México | Crear clase con estructura completa (header, cliente, partidas, pago, datosBancarios, facturación, entrega) | Medio |
| GAP-02 | Crear BO de armado del DTO | Clase que consulta 15+ tablas y arma el objeto completo para enviar al DocumentBuilder | Alto |
| GAP-03 | Endpoint de generación bajo demanda | Endpoint que genera PDF dinámicamente y lo retorna para previsualización (sin persistir) | Medio |
| GAP-04 | Lógica del Código Validador (REF. CLIENTE) | Implementar/verificar construcción dinámica de referencia bancaria: Banamex = 7 segmentos codificados; no-Banamex = nombre cliente directo | Medio |
| GAP-05 | Conversión de monto a letras | Implementar/verificar conversión del Gran Total a palabras según moneda (MXN/USD) | Bajo |
| GAP-06 | Folio con prefijo PRF en representación visual | Formato "PRF-MMDDAA-Consecutivo" en el DTO. Pendiente confirmar si se persiste con prefijo | Bajo |
| GAP-07 | Persistencia del PDF tras envío exitoso | Almacenar PDF en BD como artefacto inmutable tras confirmación de envío | Medio |
| GAP-08 | Consulta histórica del PDF desde Validar Cobro | Endpoint que retorna PDF almacenado (sin regenerar) | Bajo |
| GAP-09 | Regeneración al reintentar | Si ESAC abandona y reintenta, regenerar con datos vigentes actualizados | Bajo |
| GAP-10 | Selección de TemplateKey por empresa | Mapear empresa emisora a TemplateKey: GOL->GOL_MEX_PRO, MUN->MUN_MEX_PRO, PQF->PQF_MEX_PRO, PRO->PRO_MEX_PRO | Bajo |

### En DocumentBuilder (Servicio externo)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-11 | Crear servicio de generación de Proforma (similar a Quotation) | Crear `GenerateDocumentService.ProformaExtension.cs` con método `GenerateProformaTemplate()`. Patrón idéntico a `GenerateDocumentService.QuotationExtension.cs`: obtiene template, renderiza Header/Body/Footer con JSON, convierte a PDF. Sin lógica de merge ni Usage Letter (más simple que Cotización) | Alto |
| GAP-12 | Crear DTO tipado `DocumentGenerateProformaDto` | Similar a `DocumentGenerateQuoationDto`: FileName, TemplateKey, ClientName, Base64Images + `ProformaModel` tipado con las secciones (header, cliente, partidas, pago, datosBancarios, facturación, entrega) | Medio |
| GAP-13 | Crear endpoint `POST api/Report/proforma` en ReportController | Nuevo endpoint en `ReportController.cs` que recibe `DocumentGenerateProformaDto` y llama al servicio de generación. Patrón idéntico a `GenerateReportOrderConfirmation` | Bajo |
| GAP-14 | Crear Validator para ProformaDto | Crear `DocumentGenerateProformaDtoFluentValidator.cs` con validaciones del DTO (campos requeridos). Seguir patrón de `DocumentGenerateQuoationDtoFluentValidator.cs` | Bajo |
| GAP-15 | Crear 4 templates HTML de Proforma México | Crear carpetas GOL_MEX_PRO, MUN_MEX_PRO, PQF_MEX_PRO, PRO_MEX_PRO con archivos _H.html, _B.html, _F.html | Alto |
| GAP-16 | Registrar templates en BD DocumentBuilder | INSERT en tabla DocumentTemplate para cada TemplateKey nuevo | Bajo |
| GAP-17 | Diseño HTML/CSS del template | Maquetación del documento con secciones: cabecera con branding, tabla partidas, panel inferior (pago, bancarios, facturación, entrega), pie con certificaciones. 3 variantes visuales (naranja/verde/teal) | Alto |
| GAP-18 | Logos e imágenes en base64 | Preparar logos por empresa (Golocaer, Mungen, PROQUIFA) + sellos (ISO, NEEC, AmEx) + logos catálogos (EDQM, FEUM, USP, etc.) en base64 | Medio |

---

## Flujo Back Completo

```
1. ESAC presiona "Tramitar" (Prepago sin FAA, México)
2. ProquifaDotNet consulta datos de 15+ tablas
3. Determina TemplateKey según empresa (GOL/MUN/PQF/PRO + _MEX_PRO)
4. Arma ProformaMexicoDTO completo
5. Llama ApiCallerDocumentBuilder.EnvioPost("api/Report/generic", dto)
   - dto = { FileName, TemplateKey, ClientName, Base64Images, Data }
6. DocumentBuilder:
   - Busca template por TemplateKey
   - Lee HTMLs (Header, Body, Footer)
   - Renderiza con datos JSON
   - Convierte a PDF
   - Retorna byte[]
7. ProquifaDotNet recibe byte[] (PDF)
8. Retorna PDF al Front para previsualización (NO persiste)
9. Si ESAC cancela -> PDF se descarta, al reintentar se regenera
10. Si ESAC acepta -> Front llama endpoint de envío
11. Al envío exitoso -> persiste PDF en BD (inmutable)
12. PDF consultable desde Validar Cobro
```

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| TPSC-RE-FU-006 | ReferenciaPago / Código Validador |
| TPSC-RE-FU-013 | Foliador global (asigna folio a tpProformaPedido.Folio) |
| TPSC-RE-FU-014 | Flujo Prepago sin FAA (dispara generación del PDF) |
| TPSC-RE-FU-017 | Proforma Perú (requisito hermano, mismo patrón) |

---

## Dudas Pendientes

| # | Duda | Impacto |
|---|------|---------|
| 1 | ¿Folio se consume al previsualizar o al confirmar envío? | Define cuándo se asigna en el DTO |
| 2 | ¿Prefijo PRF se persiste en BD o solo es visual? | Define formato almacenado |
| 3 | ¿Sección Cliente muestra Alias o Razón Social? | Define campo en DTO |
| 4 | ¿Regla exacta de cálculo de Vigencia? | Define lógica |
| 5 | ¿Usar endpoint generic o crear endpoint dedicado para proforma? | Define approach en DocumentBuilder |

---

## Conclusión

El requisito TPSC-RE-FU-016 tiene **impacto alto** en desarrollo. Involucra **dos repositorios**:

**ProquifaDotNet (10 GAPs):**
- DTO complejo consultando 15+ tablas
- BO de orquestación
- Lógica de Código Validador
- Persistencia y consulta histórica

**DocumentBuilder (8 GAPs):**
- Servicio de generación de Proforma (similar a Quotation)
- DTO tipado + Validator + Endpoint dedicado
- 4 templates HTML nuevos (Header + Body + Footer por empresa)
- Diseño/maquetación del documento (3 variantes visuales)
- Logos e imágenes

El patrón de integración ya está establecido (Cotización y Confirmación de Pedido ya existen). La Proforma sigue el mismo patrón con un servicio dedicado `GenerateProformaTemplate()`, un endpoint `POST api/Report/proforma` y un DTO tipado `DocumentGenerateProformaDto`.
