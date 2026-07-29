# Tareas BackEnd - R16A-RE-FU-022
**Requisito:** Diseño y generación de Documentos: Factura Perú
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder

---

> **Nota de arquitectura:** la versión previa de estas tareas creaba una tabla `CPEGenerada`
> separada, duplicando el registro de negocio del CPE que ya vive en `CFDIGenerada`
> (propiedad de ProquifaDotNet.Finanzas, reutilizada por Perú desde RE-FU-020). Se corrige:
> **no se crea tabla nueva** — se extiende `CFDIGenerada` con las columnas SUNAT que faltan.
> `fccFactura` (RE-FU-015, antes `tpProformaAdelanto` — migración del 06/07/2026) reutiliza
> `IdCFDIGenerada`; no se agrega un FK nuevo. Ver `R16A-RE-FU-022_BD.md` para el detalle
> completo. Como consecuencia, la antigua Tarea 2 (ALTER tpProformaAdelanto ADD IdCPEGenerada)
> se elimina y las tareas se renumeran.

> **Orden de ejecución sugerido:** BD bloqueantes → templates DocumentBuilder → servicios backend
> **Dependencias externas:** Los scripts de `CodigoSUNAT`, `ClaveSUNAT` y `catAfectacionIGV` pertenecen a RE-FU-020 y no se duplican aquí; deben ejecutarse antes de las Tareas 3-4.

---

## Resumen de tareas

| # | Clave | Título simple | Tipo | Aplicativo | Bloqueante |
|---|-------|--------------|------|-----------|----------|
| 1 | UPDATE-TABL-CH | Agregar columnas SUNAT (UbigeoEmisor, DireccionEmisor, DireccionReceptor, TipoOperacion, ISC, ICBPER, OtrosTributos) a CFDIGenerada | BD | ProquifaDotNet | ✅ |
| 2 | CREATE-PDF | Plantilla PDF Factura CPE UBL 2.1 — GOLPERU_PER_FAC (Golocaer S.A.C.) | DocumentBuilder | DocumentBuilder | — |
| 3 | ALG-COMPLX-LOGIC | Algoritmo de mapeo de datos CPE UBL 2.1 al modelo del PDF de Factura Perú | Back | ProquifaDotNet.Finanzas | ⛔ |
| 4 | SERV-TRANSACT | Persistencia del PDF de Factura Perú en Minio tras timbrado SUNAT exitoso | Back | ProquifaDotNet.Finanzas | ⛔ |
| 5 | IMP-EXIST-SERVICE | Extender AdvanceInvoiceGenerateService branch PER con persistencia real del PDF (RE-FU-020) | Back | ProquifaDotNet.Finanzas | ⛔ |
| 6 | IMP-EXIST-SERVICE | Extender AdvanceInvoicePreviewService branch PER con template real GOLPERU_PER_FAC (RE-FU-020) | Back | ProquifaDotNet.Finanzas | — |
| 7 | IMP-EXIST-SERVICE | Integrar persistencia del PDF de Factura Perú en el flujo Validar Cobro Perú | Back | ProquifaDotNet.Finanzas | ⛔ |

> ⛔ Tareas 3, 4, 5 y 7 bloqueadas por: datos fiscales SUNAT del producto inexistentes (RE-FU-020/RE-FU-005 Brecha 1) y por integración OSE/PSE SUNAT pendiente (RE-FU-020/RE-FU-005 Brecha 5).

---

## TAREA 1

**[ RE-FU-022 ] [UPDATE-TABL-CH] Agregar columnas SUNAT a tabla CFDIGenerada**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Factura Perú SUNAT

**Consideraciones previas:**
- `CFDIGenerada` (creada por RE-FU-019, extendida por RE-FU-018/028/021) es la tabla única de registro de negocio del CFDI/CPE, propiedad de ProquifaDotNet.Finanzas — reutilizada por Perú desde RE-FU-020 (mismo patrón que México).
- Le faltan columnas específicas de SUNAT que no existen todavía: `UbigeoEmisor`, `DireccionEmisor`, `DireccionReceptor`, `TipoOperacion` (catálogo 51 SUNAT), `ISC`, `ICBPER`, `OtrosTributos`.
- Columnas ya existentes que se **reutilizan** para Perú (sin cambios): `RFCEmisor`/`RFCReceptor` (el RUC de 11 dígitos cabe en los `varchar` existentes), `RazonSocialEmisor`/`RazonSocialReceptor` (RE-FU-021), `Serie`/`Folio` (Folio almacena el Correlativo SUNAT), `FechaEmision`, `Total`, `Subtotal` (RE-FU-021, equivalente a ValorVenta), `IdCatMoneda`, `TipoCambio`, `CondicionesPago` (RE-FU-021, equivalente a CondicionPago), `Estado`, `IdArchivoXml`, `IdArchivoPdf`, `FechaCertificacionSat` (RE-FU-021/018).
- El IGV (18%) **no se persiste como columna**: se calcula desde las partidas, igual patrón que el IVA en México.
- `fccFactura.IdCFDIGenerada` (RE-FU-015, antes `tpProformaAdelanto.IdCFDIGenerada` de RE-FU-019) ya vincula el pendiente FAA con `CFDIGenerada` para ambas regiones — **no se requiere ALTER adicional** sobre `fccFactura`.
- Las columnas de totales SUNAT (`ISC`, `ICBPER`, `OtrosTributos`) tienen `DEFAULT 0` para no romper los registros de México, donde no aplican.

**Objetivo general:**
Extender la tabla `CFDIGenerada` con las columnas fiscales específicas de SUNAT necesarias para el CPE UBL 2.1 de Golocaer S.A.C. (Perú), reutilizando la misma tabla e infraestructura ya implementada para México.

**Objetivos específicos:**
- Ejecutar `ALTER TABLE dbo.CFDIGenerada ADD UbigeoEmisor varchar(6) NULL, DireccionEmisor varchar(300) NULL, DireccionReceptor varchar(300) NULL, TipoOperacion varchar(4) NULL, ISC decimal(18,2) NOT NULL DEFAULT(0), ICBPER decimal(18,2) NOT NULL DEFAULT(0), OtrosTributos decimal(18,2) NOT NULL DEFAULT(0)`.
- Validar que los DEFAULT constraints (`ISC=0`, `ICBPER=0`, `OtrosTributos=0`) queden correctamente configurados.
- Verificar que ningún objeto existente en BD (vistas, SPs, triggers de `CFDIGenerada`) se ve afectado por el ALTER.

**Resultado esperado:**
`CFDIGenerada` extendida con las columnas SUNAT, lista para recibir los datos fiscales del CPE al timbrar exitosamente ante el OSE/PSE, sin necesidad de una tabla nueva.

**Entregables:**
- Script DDL: `ALTER TABLE CFDIGenerada ADD` columnas SUNAT
- Script de validación (`SELECT` estructura + constraints)

**Criterios de aceptación:**
- Las 7 columnas nuevas existen en `CFDIGenerada` con los tipos y defaults definidos en `R16A-RE-FU-022_BD.md`.
- Los registros existentes de México quedan con `ISC=0`, `ICBPER=0`, `OtrosTributos=0` y el resto de columnas nuevas en `NULL`.
- Ningún objeto existente en BD presenta errores tras el ALTER.

**Más información de la tarea:**
Ver sección *"ALTER TABLE CFDIGenerada — columnas específicas SUNAT"* y *"Columnas de CFDIGenerada usadas para el CPE Perú"* en `R16A-RE-FU-022_BD.md`.

**Recursos:**
- `R16A-RE-FU-022_BD.md` — Script propuesto ALTER TABLE CFDIGenerada
- `R16A-RE-FU-022-Back.md` — Parte C, paso 1

---

## TAREA 2

**[ RE-FU-022 ] [CREATE-PDF] Plantilla PDF Factura CPE UBL 2.1 — GOLPERU_PER_FAC (Golocaer S.A.C.)**

**Aplicativos:** DocumentBuilder

**Módulos:** DocumentBuilder — Plantillas Factura Perú

**Consideraciones previas:**
- DocumentBuilder usa el patrón `{Prefix}_{Region}_{Tipo}` para la `TemplateKey` (`GOLPERU_PER_FAC`).
- Cada template requiere 3 archivos HTML: Header (`GOLPERU_PER_FAC_H.html`), Body (`GOLPERU_PER_FAC_B.html`) y Footer (`GOLPERU_PER_FAC_F.html`).
- El registro en `DocumentTemplate` se inserta via script SQL con el patrón MERGE existente en `Scripts/`.
- **⚠️ BRECHA de branding:** logo, colores, certificaciones aplicables a Perú y texto del disclaimer legal SUNAT de Golocaer S.A.C. **no están confirmados**. Las certificaciones y el disclaimer mexicanos (NEEC, FEUM, SAT) **no aplican** a Perú.
- **⚠️ BRECHA cuentas bancarias:** la factura real de muestra de Golocaer S.A.C. (E001-362) **no incluye sección bancaria**. Pendiente confirmar con el cliente si aplica a Perú.
- La estructura del PDF es distinta a México: sin UUID, sin sellos SAT, sin cadena original; en su lugar QR SUNAT, firma digital y valor resumen del OSE/PSE.
- Depende de la Tarea 1 (columnas SUNAT en `CFDIGenerada` disponibles) y de RE-FU-020 (CodigoSUNAT, ClaveSUNAT, catAfectacionIGV disponibles para armar las partidas del body).
- Validar el diseño contra la factura real de muestra de Golocaer S.A.C. (E001-362).

**Objetivo general:**
Crear la plantilla HTML (Header, Body, Footer) y el registro en `DocumentTemplate` para la Factura electrónica (CPE UBL 2.1) de la empresa **Golocaer S.A.C.** (`GOLPERU_PER_FAC`), con el branding y disclaimer SUNAT correspondientes y todas las secciones requeridas por el estándar SUNAT.

**Objetivos específicos:**
- Crear `GOLPERU_PER_FAC_H.html` — cabecera con logo Golocaer S.A.C. Perú, datos del emisor (RUC, Razón Social, Dirección, Ubigeo, Fecha y Hora de Emisión) y datos del receptor (RUC, Razón Social, Dirección), identificadores del comprobante (Serie-Correlativo, Tipo Comprobante, Tipo Operación).
- Crear `GOLPERU_PER_FAC_B.html` — cuerpo con: datos del comprobante (Condición de Pago, Moneda, Tipo de Cambio, Folio Pedido Interno PI), tabla de partidas (Cantidad, Unidad de Medida SUNAT, Código SUNAT Producto, Descripción, Valor Unitario, Afectación al IGV por línea, ICBPER por línea cuando aplique), totales SUNAT (Sub Total Ventas, Anticipos, Descuentos, Valor Venta Gratuitas, Valor Venta, ISC, IGV 18%, ICBPER, Otros Cargos, Otros Tributos, Redondeo, Importe Total, Total en Letras), sección Crédito/Cuotas (cuando CondiciónPago = Crédito, pendiente confirmar si aplica a R16 Perú Prepago).
- Crear `GOLPERU_PER_FAC_F.html` — pie con elementos técnicos SUNAT (QR de validación, Valor Resumen hash, Firma Digital), disclaimer de representación impresa SUNAT verificable con clave SOL (texto pendiente asesor legal peruano), certificaciones aplicables a Golocaer S.A.C. Perú (pendiente confirmar), paginación "X de Y".
- Insertar el registro en `DocumentTemplate` via script SQL con patrón MERGE.

**Resultado esperado:**
Template `GOLPERU_PER_FAC` registrado en DocumentBuilder con los 3 archivos HTML correctos, listo para generar el PDF de la Factura CPE UBL 2.1 de Golocaer S.A.C.

**Entregables:**
- `GOLPERU_PER_FAC_H.html`, `GOLPERU_PER_FAC_B.html`, `GOLPERU_PER_FAC_F.html`
- Script SQL: `INSERT DocumentTemplate GOLPERU_PER_FAC` (patrón MERGE)
- Prueba de generación con datos de la factura real de muestra (E001-362)

**Criterios de aceptación:**
- El PDF generado con el template `GOLPERU_PER_FAC` contiene todas las secciones A-J del requisito.
- El branding (logo, colores) corresponde a Golocaer S.A.C. Perú, validado con el cliente (pendiente confirmar).
- El disclaimer identifica el documento como CPE SUNAT verificable con clave SOL (no como CFDI SAT).
- Las certificaciones corresponden a Golocaer S.A.C. Perú (no las mexicanas).
- La paginación automática "X de Y" funciona correctamente.
- **No aparecen** UUID, sellos digitales SAT ni cadena original (son campos México).

**Más información de la tarea:**
Ver criterios A1-J2 en `R16A-RE-FU-022-Pendiente.md`. Ver sección *"Parte A — DocumentBuilder"* en `R16A-RE-FU-022-Back.md`. Patrón de templates en `Scripts/20251001_1439/01_Insert_templates.sql` de DocumentBuilder.

**Recursos:**
- `R16A-RE-FU-022-Pendiente.md` — Criterios A1-J2
- `R16A-RE-FU-022-Back.md` — Parte A, estructura del template
- DocumentBuilder — `C:\Users\juan.garcia\Documents\DocumentBuilder-R14`
- Factura real de muestra: Golocaer S.A.C. E001-362 (XML + PDF)

---

## TAREA 3

**[ RE-FU-022 ] [ALG-COMPLX-LOGIC] Algoritmo de mapeo de datos CPE UBL 2.1 al modelo del PDF de Factura Perú**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura Perú — Generación PDF CPE UBL 2.1

**Consideraciones previas:**
- ⛔ **BLOQUEANTE:** depende de RE-FU-020 para los campos `CodigoSUNAT` (Producto), `ClaveSUNAT` (catUnidad) e `IdCatAfectacionIGV` (Producto). Sin estos campos no es posible armar las partidas del PDF conforme al estándar SUNAT (ver RE-FU-005 Brecha 1).
- ⛔ **BLOQUEANTE:** depende de la integración OSE/PSE SUNAT (RE-FU-020) para obtener el XML firmado del cual se extraen QR, Firma Digital y Valor Resumen.
- El PDF de la Factura Perú integra datos de múltiples tablas: `CFDIGenerada` (misma tabla que México, con las columnas SUNAT de la Tarea 1), `tpPartidaPedido`, `Producto`, `catUnidad`, `catAfectacionIGV`, `Empresa` (GOLPERU) y los elementos técnicos del XML firmado retornado por el OSE/PSE.
- Los datos del QR, Firma Digital y Valor Resumen se extraen del XML del OSE/PSE, no de BD directamente.
- La comunicación con ProquifaDotNet (.NET Framework 4.8) para obtener los datos del pedido se realiza mediante llamadas entre APIs. Los datos del CPE (`CFDIGenerada`) se consultan directamente, ya que la tabla es propiedad de ProquifaDotNet.Finanzas.
- Aplica a Facturas de FAA Perú (RE-FU-020) y de Validar Cobro Perú. Solo empresa GOLPERU.
- Es el contraparte peruano de la Tarea 9 de RE-FU-021 (`InvoicePdfMappingService`); mismo patrón, misma tabla, columnas SUNAT adicionales.

**Objetivo general:**
Implementar el servicio `PeruInvoicePdfMappingService` que consolida todos los datos del CPE UBL 2.1 en un modelo unificado (`PeruInvoicePdfModel`), listo para ser consumido por DocumentBuilder con el template `GOLPERU_PER_FAC`, incluyendo branding de Golocaer S.A.C., datos fiscales SUNAT del emisor y receptor, partidas con códigos SUNAT, totales SUNAT e IGV, elementos técnicos de certificación y QR.

**Objetivos específicos:**
- Crear `PeruInvoicePdfModel` con secciones: Branding (LogoBase64), Emisor (RUC, RazonSocial, Dirección, Ubigeo), Receptor (RUC, RazonSocial, Dirección), Comprobante (Serie, Correlativo, TipoComprobante, TipoOperación, FechaEmision, CondiciónPago, Moneda, TC, FolioPedidoInterno), Partidas `List<PeruInvoicePdfLineItemModel>`, Totales SUNAT (SubTotalVentas, Anticipos, Descuentos, ValorVentaGratuitas, ValorVenta, ISC, IGV, ICBPER, OtrosCargos, OtrosTributos, Redondeo, ImporteTotal, TotalEnLetras), Crédito `PeruInvoicePdfCreditModel` (MontoNetoPendiente, TotalCuotas, Cuotas), Elementos Técnicos SUNAT (QRBase64, ValorResumen, FirmaDigital — null en preview).
- Mapear `PeruInvoicePdfLineItemModel` por partida: Cantidad, UnidadMedidaSUNAT (catUnidad.ClaveSUNAT), CodigoSUNAT (Producto.CodigoSUNAT — BRECHA RE-FU-020), NumeroOrdenCompra (referencia del pedido del cliente, criterio D2), Descripción (nombre + catálogo + lote), ValorUnitario, Importe, AfectaciónIGV (catAfectacionIGV.Clave — BRECHA RE-FU-020), ICBPERLinea, TipoPrecio (catálogo 16 SUNAT — campo opcional, observado en factura real E001-362; pendiente confirmar con cliente).
- Mapear Emisor/Receptor/Comprobante/Totales desde `CFDIGenerada`: `RFCEmisor` (RUC), `RazonSocialEmisor`, `DireccionEmisor`, `UbigeoEmisor`, `RFCReceptor` (RUC), `RazonSocialReceptor`, `DireccionReceptor`, `Serie`, `Folio` (Correlativo), `TipoOperacion`, `FechaEmision`, `CondicionesPago`, `IdCatMoneda`, `TipoCambio`, `Subtotal` (ValorVenta), `ISC`, `ICBPER`, `OtrosTributos`, `Total`; el IGV se calcula desde las partidas (18%).
- Parsear `xmlFirmadoSunat` → extraer QR, FirmaDigital, ValorResumen del CDR del OSE/PSE.
- Calcular TotalEnLetras según moneda (ej: "SON: DIECIOCHO MIL SETECIENTOS CUATRO Y 18/100 DOLAR AMERICANO").
- Implementar `IPeruInvoicePdfMappingService` con dos métodos: `MapearAsync(idCFDIGenerada, xmlFirmadoSunat)` (PDF definitivo) y `MapearPreviewAsync(idCFDIGenerada)` (preview sin elementos técnicos SUNAT).

**Resultado esperado:**
Servicio `PeruInvoicePdfMappingService` que recibe el `IdCFDIGenerada` y el XML firmado del OSE/PSE, y retorna un `PeruInvoicePdfModel` completamente poblado, listo para ser pasado al generador de DocumentBuilder (Tarea 2) tanto para preview como para PDF definitivo.

**Entregables:**
- Clase `PeruInvoicePdfModel` con todas las secciones del CPE PDF
- Clase `PeruInvoicePdfLineItemModel`, `PeruInvoicePdfCreditModel`, `PeruInvoicePdfInstallmentModel`
- Interfaz `IPeruInvoicePdfMappingService` + servicio `PeruInvoicePdfMappingService`
- Prueba unitaria con datos de la factura real de muestra (E001-362)

**Criterios de aceptación:**
- El modelo generado contiene todos los campos requeridos por las secciones A-J del PDF (criterios del requisito).
- Los campos `CodigoSUNAT`, `ClaveSUNAT` y `AfectaciónIGV` de cada partida se mapean correctamente desde los campos nuevos de RE-FU-020.
- El QR, FirmaDigital y ValorResumen se extraen del XML del OSE/PSE, no se calculan en la aplicación.
- `MapearPreviewAsync` retorna el modelo con la Sección G (Elementos Técnicos) en null.
- El TotalEnLetras se genera correctamente para PEN y USD.

**Más información de la tarea:**
Ver sección *"Parte B — PeruInvoicePdfMappingService"* en `R16A-RE-FU-022-Back.md`. Ver Tarea 4 de RE-FU-020 (datos SUNAT del producto como prerequisito). Es el contraparte peruano de la Tarea 9 de RE-FU-021.

**Recursos:**
- `R16A-RE-FU-022-Back.md` — Parte B, modelo y flujo MapearAsync
- `R16A-RE-FU-022-Pendiente.md` — Criterios A1-J2, Reglas 7 y 8
- R16A-RE-FU-021 Tarea 9 — `InvoicePdfMappingService` (patrón equivalente)
- Factura real de muestra: Golocaer S.A.C. E001-362

---

## TAREA 4

**[ RE-FU-022 ] [SERV-TRANSACT] Persistencia del PDF de Factura Perú en Minio tras timbrado SUNAT exitoso**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura Perú — Persistencia Artefacto Fiscal

**Consideraciones previas:**
- ⛔ **BLOQUEANTE:** depende de la Tarea 3 (`PeruInvoicePdfMappingService`) y de la integración OSE/PSE SUNAT (RE-FU-020) que retorna el XML firmado necesario para el mapeo.
- Se ejecuta inmediatamente después del timbrado exitoso ante SUNAT/OSE, como parte del flujo de los módulos que generan facturas Perú (FAA RE-FU-020 y Validar Cobro Perú).
- El PDF se genera una única vez al timbrar; posteriormente se sirve desde Minio sin regeneración (criterio J2). La factura timbrada es artefacto fiscal inmutable.
- La persistencia utiliza el patrón existente: `INSERT Archivo` con `FileBucket='facturas'` e `IdRegion='PER'`, referenciado desde `CFDIGenerada.IdArchivoPdf`.
- Si la generación o persistencia del PDF falla tras el timbrado exitoso, debe reintentarse **sin re-timbrar** ante SUNAT/OSE.
- La actualización de `CFDIGenerada.IdArchivoPdf` y de `fccFactura.IdCFDIGenerada` (RE-FU-015, antes `tpProformaAdelanto.IdCFDIGenerada`) se realizan ambas directamente en BD vía EF Core, sin llamada API: **corrección arquitectónica 06/07/2026** — a diferencia de la extinta `tpProformaAdelanto` (propiedad del sistema legado, sí requería API), `fccFactura` es propiedad de `ProquifaDotNet.Finanzas` (Scaffold EF Core en `Finanzas.Infrastructure`).
- TemplateKey fijo: `GOLPERU_PER_FAC` (única empresa emisora Perú).
- Depende de las Tareas 1-3.
- Es el contraparte peruano de la Tarea 10 de RE-FU-021 (`PersistInvoicePdfService`).

**Objetivo general:**
Implementar el servicio `PersistPeruInvoicePdfService` que, tras el timbrado exitoso ante SUNAT/OSE, genera el PDF definitivo del CPE UBL 2.1 (con QR, Firma Digital y Valor Resumen completos) y lo persiste en Minio vía la tabla `Archivo` con `IdRegion='PER'`, referenciado desde `CFDIGenerada.IdArchivoPdf`.

**Objetivos específicos:**
- Invocar `PeruInvoicePdfMappingService.MapearAsync(IdCFDIGenerada, xmlFirmadoSunat)` para obtener el `PeruInvoicePdfModel` definitivo.
- Invocar DocumentBuilder con `TemplateKey='GOLPERU_PER_FAC'` para generar el PDF en bytes.
- `INSERT Archivo` (PDF, `FileBucket='facturas'`, `IdRegion='PER'`, `ContentType='application/pdf'`) → obtener `IdArchivo`.
- Actualizar `CFDIGenerada.IdArchivoPdf` directamente en BD (vía EF Core, sin llamada API).
- Actualizar `fccFactura.IdCFDIGenerada` directamente en BD (vía EF Core, sin llamada API — RE-FU-015; antes `UPDATE tpProformaAdelanto SET IdCFDIGenerada` sí requería llamada API a ProquifaDotNet .NET Fx 4.8; si aún no estaba vinculado).
- Garantizar atomicidad: si la persistencia en Minio falla, reintentar sin re-timbrar ante SUNAT/OSE.
- Registrar en Serilog: módulo, `IdCFDIGenerada`, fecha, resultado (éxito/error + mensaje).

**Resultado esperado:**
PDF del CPE UBL 2.1 persistido en Minio como artefacto fiscal inmutable, referenciado desde `CFDIGenerada.IdArchivoPdf`, disponible para consultas y envíos posteriores sin regeneración.

**Entregables:**
- Servicio `PersistPeruInvoicePdfService` (o equivalente)
- Manejo de reintentos de persistencia sin re-timbrado ante SUNAT/OSE
- Registro en log con Serilog (módulo, `IdCFDIGenerada`, fecha, resultado)

**Criterios de aceptación:**
- El PDF se genera con QR, Firma Digital y Valor Resumen del OSE/PSE integrados.
- El `INSERT Archivo` persiste correctamente en Minio con `FileBucket='facturas'` e `IdRegion='PER'`.
- La referencia del PDF queda correctamente almacenada en `CFDIGenerada.IdArchivoPdf`.
- Si la persistencia falla, el sistema reintenta sin volver a llamar al OSE/PSE SUNAT.
- Al consultar el PDF posteriormente, el sistema sirve el archivo desde Minio sin regenerarlo.
- La operación queda registrada en Serilog con `IdCFDIGenerada`, fecha y resultado.

**Más información de la tarea:**
Ver sección *"Parte B — PersistPeruInvoicePdfService"* en `R16A-RE-FU-022-Back.md`. Ver criterios J1-J2, Reglas 1 y 5 en `R16A-RE-FU-022-Pendiente.md`. Es el contraparte peruano de la Tarea 10 de RE-FU-021.

**Recursos:**
- `R16A-RE-FU-022-Back.md` — Parte B, flujo PersistPeruInvoicePdfService
- `R16A-RE-FU-022-Pendiente.md` — Criterios J1-J2, Reglas 1 y 5
- R16A-RE-FU-021 Tarea 10 — `PersistInvoicePdfService` (patrón equivalente)

---

## TAREA 5

**[ RE-FU-022 ] [IMP-EXIST-SERVICE] Extender AdvanceInvoiceGenerateService branch PER con persistencia real del PDF (RE-FU-020)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura por Adelantado — Generación Perú

**Consideraciones previas:**
- ⛔ **BLOQUEANTE:** depende de la Tarea 4 (`PersistPeruInvoicePdfService`) y de la integración OSE/PSE SUNAT (RE-FU-020).
- El branch PER de `AdvanceInvoiceGenerateService` (RE-FU-020) almacena el XML firmado y el CDR en Minio tras el timbrado exitoso. El PDF definitivo del CPE queda como placeholder pendiente de este requisito.
- Esta tarea implementa ese placeholder con la llamada real a `PersistPeruInvoicePdfService` (Tarea 4), de forma análoga a la Tarea 11 de RE-FU-021 para el branch MEX.
- No se modifica la lógica de timbrado SUNAT ni los pasos previos del branch PER; solo se agrega la llamada al servicio de persistencia del PDF.
- Aplica únicamente a facturas de región Perú (GOLPERU). El branch MEX (Tarea 11 de RE-FU-021) tiene su propia integración.

**Objetivo general:**
Integrar la llamada a `PersistPeruInvoicePdfService` en el branch PER de `AdvanceInvoiceGenerateService` (RE-FU-020), completando el flujo de generación de la Factura por Adelantado Perú con la persistencia del PDF definitivo del CPE UBL 2.1 en Minio inmediatamente tras el timbrado exitoso ante SUNAT/OSE.

**Objetivos específicos:**
- Inyectar `IPersistPeruInvoicePdfService` en `AdvanceInvoiceGenerateService`.
- Agregar la llamada `await _persistirFacturaPeruPdfService.PersistirAsync(idCFDIGenerada, response.XmlFirmado)` en el branch PER tras el timbrado exitoso.
- Validar que si `PersistirAsync` falla, el error se maneja sin re-timbrar ante SUNAT/OSE.
- Registrar en Serilog el resultado de la persistencia dentro del flujo de generación.

**Resultado esperado:**
Branch PER de `AdvanceInvoiceGenerateService` con la persistencia del PDF CPE implementada definitivamente: tras el timbrado exitoso, el PDF se genera y persiste en Minio antes de retornar el response al cliente.

**Entregables:**
- `AdvanceInvoiceGenerateService` branch PER actualizado (llamada real a `PersistPeruInvoicePdfService`)
- Prueba de integración: flujo completo generar FAA Perú → PDF persistido en Minio

**Criterios de aceptación:**
- Tras el timbrado SUNAT exitoso, el PDF se persiste en Minio y la referencia queda en `CFDIGenerada.IdArchivoPdf` antes de retornar el response.
- Si la persistencia del PDF falla, el sistema no re-timbra ante SUNAT/OSE y retorna el error correspondiente.
- El flujo de timbrado SUNAT (pasos previos) no presenta regresiones respecto a RE-FU-020.
- La operación queda registrada en Serilog con `IdCFDIGenerada`, fecha y resultado.

**Más información de la tarea:**
Ver sección *"Integración con RE-FU-020 — branch PER"* en `R16A-RE-FU-022-Back.md`. Ver Tarea 11 de RE-FU-021 (patrón equivalente para México).

**Recursos:**
- `R16A-RE-FU-022-Back.md` — Parte B, integración branch PER AdvanceInvoiceGenerateService
- R16A-RE-FU-021 Tarea 11 — `AdvanceInvoiceGenerateService` MEX (patrón equivalente)

---

## TAREA 6

**[ RE-FU-022 ] [IMP-EXIST-SERVICE] Extender AdvanceInvoicePreviewService branch PER con template real GOLPERU_PER_FAC (RE-FU-020)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura por Adelantado — Previsualización PDF Perú

**Consideraciones previas:**
- El branch PER de `AdvanceInvoicePreviewService` (RE-FU-020) tiene un placeholder para el template del PDF de previsualización, pendiente de definir en este requisito.
- Esta tarea reemplaza ese placeholder con el template real `GOLPERU_PER_FAC` de DocumentBuilder, de forma análoga a la Tarea 12 de RE-FU-021 para el branch MEX.
- Depende de la Tarea 2 (template `GOLPERU_PER_FAC` registrado en DocumentBuilder) y de la Tarea 3 (`PeruInvoicePdfMappingService.MapearPreviewAsync` disponible).
- El preview usa el modelo sin elementos técnicos SUNAT (Sección G en null): QR, Firma Digital y Valor Resumen no están presentes hasta el timbrado real.
- El TemplateKey es fijo (`GOLPERU_PER_FAC`) ya que solo hay una empresa emisora Perú, a diferencia del switch por empresa de México.
- Aplica únicamente a facturas de región Perú (GOLPERU).

**Objetivo general:**
Reemplazar el template placeholder del branch PER de `AdvanceInvoicePreviewService` (RE-FU-020) con la resolución del template real `GOLPERU_PER_FAC` de DocumentBuilder, completando el flujo de previsualización del PDF de la Factura por Adelantado Perú.

**Objetivos específicos:**
- Inyectar `IPeruInvoicePdfMappingService` en `AdvanceInvoicePreviewService`.
- Invocar `MapearPreviewAsync(idCFDIGenerada)` para obtener el `PeruInvoicePdfModel` sin elementos técnicos SUNAT.
- Invocar DocumentBuilder con `TemplateKey='GOLPERU_PER_FAC'` (fijo — única empresa Perú).
- Retornar el PDF en memoria sin persistir en Minio (el preview no es artefacto fiscal).

**Resultado esperado:**
Branch PER de `AdvanceInvoicePreviewService` con la generación real del PDF de previsualización usando el template `GOLPERU_PER_FAC` de DocumentBuilder, con todos los campos del CPE disponibles excepto QR, Firma Digital y Valor Resumen.

**Entregables:**
- `AdvanceInvoicePreviewService` branch PER actualizado (template real, sin placeholder)
- Prueba de integración: flujo completo preview FAA Perú → PDF en memoria con datos correctos

**Criterios de aceptación:**
- El PDF de preview generado usa el template `GOLPERU_PER_FAC` (branding Golocaer S.A.C. Perú).
- El PDF de preview NO contiene QR, Firma Digital ni Valor Resumen (Sección G vacía o ausente).
- El flujo existente del branch PER (pasos previos al template) no presenta regresiones respecto a RE-FU-020.

**Más información de la tarea:**
Ver sección *"Integración con RE-FU-020 — branch PER Preview"* en `R16A-RE-FU-022-Back.md`. Ver Tarea 12 de RE-FU-021 (patrón equivalente para México).

**Recursos:**
- `R16A-RE-FU-022-Back.md` — Parte B, integración branch PER AdvanceInvoicePreviewService
- R16A-RE-FU-021 Tarea 12 — `AdvanceInvoicePreviewService` MEX (patrón equivalente)

---

## TAREA 7

**[ RE-FU-022 ] [IMP-EXIST-SERVICE] Integrar persistencia del PDF de Factura Perú en el flujo Validar Cobro Perú**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Generación Factura Perú

**Consideraciones previas:**
- ⛔ **BLOQUEANTE:** depende de la Tarea 4 (`PersistPeruInvoicePdfService`) y de la integración OSE/PSE SUNAT (RE-FU-020).
- El requisito RE-FU-022 es explícito en su alcance: "Facturas originadas en Validar Cobro (cobro recibido aplicado a Proforma de Prepago sin controlados, Región Perú)."
- En el flujo de Validar Cobro Perú, el punto de timbrado exitoso ante SUNAT/OSE es donde debe invocarse `PersistPeruInvoicePdfService` (Tarea 4) para generar y persistir el PDF como artefacto fiscal inmutable.
- Aplica únicamente a facturas de región Perú (GOLPERU).
- Si la generación o persistencia del PDF falla tras el timbrado exitoso, debe reintentarse sin re-timbrar ante SUNAT/OSE.
- **No existe Complemento de Pago en Perú** (SUNAT no tiene documento equivalente), por lo que el flujo de Validar Cobro Perú es más simple que el de México.
- Es el contraparte peruano de la Tarea 13 de RE-FU-021 (integración PDF en Validar Cobro México).

**Objetivo general:**
Integrar la llamada a `PersistPeruInvoicePdfService` en el servicio de Validar Cobro que ejecuta el timbrado de la factura Perú, de modo que el PDF del CPE UBL 2.1 se genere y persista en Minio como artefacto fiscal inmutable inmediatamente después del timbrado exitoso ante SUNAT/OSE.

**Objetivos específicos:**
- Identificar el servicio/comando de Validar Cobro Perú en ProquifaDotNet.Finanzas que ejecuta el timbrado SUNAT.
- Inyectar `IPersistPeruInvoicePdfService` en dicho servicio.
- Invocar `PersistirAsync(idCFDIGenerada, xmlFirmadoSunat)` inmediatamente después del timbrado exitoso.
- Validar que si `PersistirAsync` falla, el error se maneja sin re-timbrar ante SUNAT/OSE.
- Registrar en Serilog el resultado (módulo, `IdCFDIGenerada`, fecha, resultado).

**Resultado esperado:**
El flujo de Validar Cobro Perú genera y persiste el PDF del CPE UBL 2.1 en Minio como artefacto fiscal inmutable tras el timbrado exitoso, de forma consistente con el comportamiento implementado para FAA Perú (Tarea 5).

**Entregables:**
- Servicio de Validar Cobro Perú actualizado con llamada real a `PersistPeruInvoicePdfService`
- Manejo de error de persistencia sin re-timbrado ante SUNAT/OSE
- Registro en Serilog (módulo, `IdCFDIGenerada`, fecha, resultado)

**Criterios de aceptación:**
- Tras el timbrado SUNAT exitoso en Validar Cobro Perú, el PDF se persiste en Minio y la referencia queda en `CFDIGenerada.IdArchivoPdf`.
- Si la persistencia del PDF falla, el sistema no re-timbra ante SUNAT/OSE.
- El flujo de Validar Cobro Perú no presenta regresiones.
- La operación queda registrada en Serilog con `IdCFDIGenerada`, fecha y resultado.

**Más información de la tarea:**
Ver sección *"GAP-07 — Validar Cobro Perú"* en `R16A-RE-FU-022-Back.md`. Ver Tarea 13 de RE-FU-021 (patrón equivalente para México). Ver alcance en `R16A-RE-FU-022-Pendiente.md` (facturas originadas en Validar Cobro Perú).

**Recursos:**
- `R16A-RE-FU-022-Back.md` — GAP-07 Validar Cobro Perú
- `R16A-RE-FU-022-Pendiente.md` — Alcance, Criterios J1-J2
- R16A-RE-FU-021 Tarea 13 — Integración Validar Cobro México (patrón equivalente)
