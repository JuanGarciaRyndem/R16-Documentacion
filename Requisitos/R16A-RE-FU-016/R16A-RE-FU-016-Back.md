# Impacto en Back — R16A-RE-FU-016
**Requisito:** Diseño y generación de Documentos: Proforma México
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder
**Módulo:** L05.TramitarPedido + Proforma (Finanzas) + DocumentBuilder
**Impacto:** Generación de PDF de Proforma México con armado DTO + servicio en DocumentBuilder + persistencia en Minio + foliador global con SEQUENCE

---

## Resumen

Este requisito implementa la **generación del PDF de Proforma** para pedidos Prepago sin FAA de clientes con Región México. Involucra **3 repositorios/soluciones**:

1. **ProquifaDotNet** (Venta Interna, .NET Framework 4.8) — Dispara la generación al tramitar, consume API de Finanzas
2. **ProquifaDotNet.Finanzas** (.NET Core 10, solución nueva) — Módulo Proforma: arma DTO, llama DocumentBuilder, persiste PDF en Minio
3. **DocumentBuilder** — Servicio de generación: recibe DTO, renderiza template HTML, retorna PDF

### Flujo de integración entre soluciones
```
ProquifaDotNet (Venta Interna)     ProquifaDotNet.Finanzas          DocumentBuilder
┌─────────────────────┐           ┌──────────────────────┐         ┌──────────────────┐
│ ESAC tramita pedido │           │ Módulo Proforma      │         │                  │
│ Llama API Finanzas  │ ────────> │ Arma DTO             │ ──────> │ Renderiza HTML   │
│                     │           │ Consulta datos       │         │ Convierte a PDF  │
│                     │ <──────── │ Recibe PDF           │ <────── │ Retorna byte[]   │
│ Previsualiza PDF    │           │ Persiste en Minio    │         │                  │
└─────────────────────┘           └──────────────────────┘         └──────────────────┘
```

---

## Arquitectura — ProquifaDotNet.Finanzas (Módulo Proforma)

### Capas involucradas en este requisito

| Capa | Responsabilidad para RE-FU-016 |
|------|-------------------------------|
| **Domain** | Entidad Proforma, interfaces de repositorio |
| **Application** | Command GenerarProformaPDF, Query ConsultarProformaPDF, DTO ProformaModel |
| **Infrastructure** | EF Core (consulta BD ProquifaDotNet), Minio (almacenamiento PDF), cliente HTTP DocumentBuilder |
| **API** | Endpoint POST /api/proforma/generar-pdf, GET /api/proforma/{id}/pdf |

### Integraciones de Finanzas para este requisito
| Integración | Uso |
|-------------|-----|
| **Minio** | Almacenamiento del PDF tras envío exitoso (bucket 'pedidos') |
| **IdentityServer** | Autenticación de llamadas desde Venta Interna |
| **Serilog** | Logs con contexto (usuario, módulo, operación) |

---

## Código Existente Relacionado (ProquifaDotNet)

### Cliente HTTP al DocumentBuilder
`Logic.Pqf.Catalogos\ApiCaller\ApiCallerDocumentBuilder.cs`

> - Método `EnvioPost(string url, object parametros)` -> retorna `byte[]`
> - Patrón reutilizable para la llamada desde Finanzas (o directa desde ProquifaDotNet)

### Generación de proformas (datos)
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\GeneracionProforma\tpPedidoFacturaToTPProformaPedidoBO.cs`

### Factory de proforma
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs`

---

## Análisis del DocumentBuilder (Servicio Externo)

### Arquitectura
- .NET (Clean Architecture: API / Application / Domain / Infrastructure)
- Endpoint genérico: `POST api/Report/generic` con `DocumentGenerationAnonymousDto`
- Flujo: DTO JSON -> busca template (BD) -> lee HTML (Header/Body/Footer) -> renderiza -> PDF

### Templates existentes (referencia)
| TemplateKey | Empresa | Tipo |
|-------------|---------|------|
| GOL_MEX_COT / GOL_MEX_PED | Golocaer | Cotización / Pedido México |
| MUN_MEX_COT / MUN_MEX_PED | Mungen | Cotización / Pedido México |
| PQF_MEX_COT / PQF_MEX_PED | Proquifa | Cotización / Pedido México |
| PRO_MEX_COT / PRO_MEX_PED | Proveedora | Cotización / Pedido México |

### Templates a CREAR para Proforma
| TemplateKey | Empresa | Archivos |
|-------------|---------|----------|
| GOL_MEX_PRO | Golocaer | _H.html, _B.html, _F.html |
| MUN_MEX_PRO | Mungen | _H.html, _B.html, _F.html |
| PQF_MEX_PRO | Proquifa | _H.html, _B.html, _F.html |
| PRO_MEX_PRO | Proveedora | _H.html, _B.html, _F.html |

### Variantes visuales confirmadas
| Variante | Empresas | Logo | Color | Banco |
|----------|----------|------|-------|-------|
| 1 | Golocaer | Logo propio (G) | Naranja | BANORTE |
| 2 | Mungen | Logo manuscrito | Verde | BANAMEX |
| 3 | Proquifa + Proveedora | Logo PROQUIFA | Teal | BANAMEX |

---

## Impacto en BD (según R16A-RE-FU-016_BD.md v2.5)

### Cambios DDL

| # | Cambio | Tipo |
|---|--------|------|
| 1 | `ALTER TABLE tpProformaPedido ADD FolioProforma varchar(80) NULL` | ALTER |
| 2 | `ALTER TABLE tpProformaPedido ADD ConsecutivoProforma int DEFAULT(0)` | ALTER |
| 3 | `CREATE SEQUENCE dbo.SeqFolioProforma AS INT START WITH 1 INCREMENT BY 1 NO CYCLE` | DDL Nuevo |
| 4 | `CREATE VIEW dbo.vtpProformaPedido` | Vista nueva |

### Foliador Global PRF

| Aspecto | Valor |
|---------|-------|
| Campo BD | tpProformaPedido.FolioProforma (varchar 80) + ConsecutivoProforma (int) |
| Formato interno BD | MMDDAA-Consecutivo (ej: 031826-691) |
| Formato visual PDF | PRF-MMDDAA-Consecutivo (prefijo PRF solo en render) |
| Segmentación | Ninguna (global) |
| Momento consumo | Al confirmar envío exitoso (sin huecos) |
| Mecanismo | SQL SEQUENCE `dbo.SeqFolioProforma` |

### Persistencia del PDF (Minio)

| Etapa | BD | Minio |
|-------|-----|-------|
| Previsualización | Nada | Nada (PDF en memoria) |
| Envío exitoso | INSERT Archivo + UPDATE tpPedido.IdArchivo + UPDATE tpProformaPedido.FolioProforma | Sube PDF al bucket 'pedidos' |
| Consulta histórica | SELECT Archivo.FileKey, FileBucket | Descarga PDF |

**Flujo de persistencia:**
```
1. NEXT VALUE FOR dbo.SeqFolioProforma -> @Consecutivo
2. @FolioProforma = FORMAT(GETDATE(),'MMddyy') + '-' + CAST(@Consecutivo AS VARCHAR)
3. INSERT dbo.Archivo (FileKey='proformas/PRF-{FolioProforma}.pdf', FileBucket='pedidos', IdRegion=MEX)
4. UPDATE dbo.tpPedido SET IdArchivo = @IdArchivo
5. UPDATE dbo.tpProformaPedido SET FolioProforma = @FolioProforma, ConsecutivoProforma = @Consecutivo
6. PDF binario sube a Minio bucket 'pedidos'
```

---

## DTO de Proforma México (Data para DocumentBuilder)

```json
{
  "header": {
    "title": "Proforma",
    "folio": "PRF-031826-691",
    "vigencia": "17/04/2026",
    "disclaimer": "ESTE ES UN DOCUMENTO INFORMATIVO..."
  },
  "cliente": { "nombre": "LABORATORIO RAAM DE SAHUAYO" },
  "partidas": [
    { "numero": "1", "cantidad": "1", "descripcion": "...", "precioUnitario": "$ 2,972.00", "importe": "$ 2,972.00" }
  ],
  "pago": {
    "subTotal": "$ 2,972.00 M.N.", "ivaPorcentaje": "0 %", "iva": "$ 0.00 M.N.",
    "granTotal": "$ 2,972.00 M.N.", "granTotalEnLetra": "(DOS MIL...)",
    "tipoCambio": "", "condicionesDePago": "PREPAGO 100%", "leyendaExhibicion": "PAGO EN UNA SOLA EXHIBICIÓN"
  },
  "datosBancarios": {
    "cuentaMN": { "moneda": "M.N.", "banca": "BANORTE", "sucursal": "069", "cuenta": "...", "clabe": "...", "refCliente": "..." },
    "cuentaDLS": { "moneda": "DLS", "banca": "BANORTE", "sucursal": "069", "cuenta": "...", "clabe": "...", "refCliente": "..." }
  },
  "facturacion": { "rfc": "...", "razonSocial": "...", "direccionFiscal": "..." },
  "entrega": { "pedido": "147", "parciales": "NO", "contacto": "...", "lugar": "..." }
}
```

---

## Gaps de Desarrollo

### En ProquifaDotNet.Finanzas (Solución nueva — Módulo Proforma)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Crear módulo Proforma en Finanzas | Entidad Proforma en Domain, Command/Query en Application, endpoints en API | Alto |
| GAP-02 | Crear DTO ProformaModel y BO de armado | Consultar BD ProquifaDotNet (EF Core scaffold), armar objeto completo con datos de 15+ tablas | Alto |
| GAP-03 | Integración con DocumentBuilder (cliente HTTP) | Llamar `POST api/Report/proforma` desde Infrastructure, retornar byte[] | Medio |
| GAP-04 | Integración con Minio (persistencia PDF) | Al envío exitoso: subir PDF al bucket 'pedidos', registrar en tabla Archivo | Medio |
| GAP-05 | Endpoint generación bajo demanda (previsualización) | `POST /api/proforma/generar-pdf` — genera sin persistir, retorna byte[] | Medio |
| GAP-06 | Endpoint consulta histórica | `GET /api/proforma/{id}/pdf` — descarga PDF de Minio (sin regenerar) | Bajo |
| GAP-07 | Foliador con SEQUENCE | Consumir `NEXT VALUE FOR dbo.SeqFolioProforma` al confirmar envío, formato MMDDAA-Consecutivo | Medio |
| GAP-08 | Lógica del Código Validador (REF. CLIENTE) | Banamex=7 segmentos; no-Banamex=nombre cliente directo | Medio |
| GAP-09 | Conversión monto a letras | MXN: "PESOS XX/100 M.N." / USD: "DOLARES XX/100" | Bajo |
| GAP-10 | Folio con prefijo PRF (visual) | En DTO para DocumentBuilder: "PRF-" + FolioProforma. No se persiste con prefijo | Bajo |

### En ProquifaDotNet (Venta Interna)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-11 | Llamada a API Finanzas desde tramitación | Al tramitar Prepago sin FAA México, llamar API Finanzas para generar PDF | Medio |
| GAP-12 | Retornar PDF para previsualización | Recibir byte[] de Finanzas y retornarlo al Front | Bajo |

### En DocumentBuilder (Servicio externo)

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-13 | Crear servicio GenerateProformaTemplate | `GenerateDocumentService.ProformaExtension.cs` — patrón similar a QuotationExtension (más simple, sin merge) | Alto |
| GAP-14 | Crear DTO tipado DocumentGenerateProformaDto + ProformaModel | FileName, TemplateKey, ClientName, Base64Images + ProformaModel tipado | Medio |
| GAP-15 | Crear endpoint POST api/Report/proforma | Nuevo endpoint en ReportController.cs | Bajo |
| GAP-16 | Crear Validator DocumentGenerateProformaDtoFluentValidator | Validaciones de campos requeridos | Bajo |
| GAP-17 | Crear 4 templates HTML (12 archivos) | Carpetas GOL/MUN/PQF/PRO_MEX_PRO con _H.html, _B.html, _F.html | Alto |
| GAP-18 | Registrar templates en BD DocumentBuilder | INSERT en tabla DocumentTemplate | Bajo |
| GAP-19 | Diseño HTML/CSS del template | Maquetación: cabecera, partidas, panel inferior 4 columnas, pie con certificaciones. 3 variantes visuales | Alto |
| GAP-20 | Logos e imágenes | Preparar logos por empresa + sellos + catálogos en base64 o repositorio | Medio |

---

## Flujo Back Completo

```
1. ESAC presiona "Tramitar" (Prepago sin FAA, México)
2. ProquifaDotNet llama API ProquifaDotNet.Finanzas: POST /api/proforma/generar-pdf
3. Finanzas consulta datos de BD ProquifaDotNet (EF Core scaffold):
   - tpPedido, tpProformaPedido, tpProformaPartidaPedido, Producto, MarcaFamilia
   - Cliente, DatosFacturacionCliente, Empresa, EmpresaDatosBancarios
   - DatosBancarios, catBanco, catMoneda, catCondicionesDePago
   - DireccionCliente, ContactoCliente
4. Determina TemplateKey: Empresa.Prefijo + "_MEX_PRO"
5. Arma ProformaModel completo (con Código Validador, monto en letras)
6. Llama DocumentBuilder: POST api/Report/proforma
7. DocumentBuilder renderiza HTML con JSON -> retorna byte[] (PDF)
8. Finanzas retorna PDF a ProquifaDotNet (previsualización, sin persistir)
9. ESAC previsualiza -> si cancela, PDF se descarta
10. ESAC acepta -> confirma envío del correo
11. Finanzas:
    - NEXT VALUE FOR SeqFolioProforma -> consecutivo
    - Sube PDF a Minio (bucket 'pedidos', key 'proformas/PRF-MMDDAA-XXX.pdf')
    - INSERT Archivo + UPDATE tpPedido.IdArchivo
    - UPDATE tpProformaPedido.FolioProforma + ConsecutivoProforma
12. PDF consultable desde Validar Cobro via GET /api/proforma/{id}/pdf
```

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| R16A-RE-FU-006 | ReferenciaPago / Código Validador |
| R16A-RE-FU-013 | Flujo base Prepago (foliador, previsualización, envío) |
| R16A-RE-FU-014 | Flujo Prepago sin FAA dispara generación del PDF |
| R16A-RE-FU-017 | Proforma Perú (mismo patrón, templates diferentes) |

---

## Dudas Pendientes

| # | Duda | Impacto |
|---|------|---------|
| 1 | ¿Sección Cliente muestra Alias o Razón Social? | Define campo en DTO |
| 2 | ¿Regla exacta de cálculo de Vigencia? | Define lógica |
| 3 | ¿Endpoint proforma dedicado o genérico en DocumentBuilder? | Recomendación: dedicado (patrón Quotation) |

---

## Conclusión

El requisito R16A-RE-FU-016 tiene **impacto alto** en desarrollo. Involucra **3 repositorios**:

**ProquifaDotNet.Finanzas (10 GAPs — solución nueva):**
- Módulo Proforma con CQRS
- BO de armado complejo (15+ tablas)
- Integración Minio + DocumentBuilder
- Foliador con SEQUENCE

**ProquifaDotNet (2 GAPs):**
- Llamada a API Finanzas desde tramitación
- Retorno de PDF para previsualización

**DocumentBuilder (8 GAPs):**
- Servicio dedicado (Extension + DTO + Validator + Endpoint)
- 4 templates HTML (12 archivos, 3 variantes visuales)
- Diseño/maquetación + logos

El patrón de DocumentBuilder ya está establecido (Cotización/Pedido existen). La novedad principal es que el armado y persistencia se mueven a **ProquifaDotNet.Finanzas** como módulo Proforma, comunicándose con Venta Interna vía API.
