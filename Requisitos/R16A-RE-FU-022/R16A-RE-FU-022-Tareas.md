# Tareas BackEnd - R16A-RE-FU-022
**Requisito:** Diseño y generación de Documentos: Factura Perú
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder

---

> **Orden de ejecución sugerido:** BD bloqueantes → templates DocumentBuilder → servicios backend
> **Dependencias externas:** Los scripts de `CodigoSUNAT`, `ClaveSUNAT` y `catAfectacionIGV` pertenecen a RE-FU-020 y no se duplican aquí; deben ejecutarse antes de las Tareas 4-5.

---

## Resumen de tareas

| # | Clave | Título simple | Tipo | Aplicativo | Bloqueante |
|---|-------|--------------|------|-----------|----------|
| 1 | CREATE-TABL-M | Crear tabla CPEGenerada con datos fiscales del CPE SUNAT | BD | ProquifaDotNet | ✅ |
| 2 | UPDATE-TABL-CH | Agregar IdCPEGenerada (FK) a tpProformaAdelanto | BD | ProquifaDotNet | — |
| 3 | CREATE-PDF | Plantilla PDF Factura CPE UBL 2.1 — GOLPERU_PER_FAC (Golocaer S.A.C.) | DocumentBuilder | DocumentBuilder | — |
| 4 | ALG-COMPLX-LOGIC | Algoritmo de mapeo de datos CPE UBL 2.1 al modelo del PDF de Factura Perú | Back | ProquifaDotNet.Finanzas | ⛔ |
| 5 | SERV-TRANSACT | Persistencia del PDF de Factura Perú en Minio tras timbrado SUNAT exitoso | Back | ProquifaDotNet.Finanzas | ⛔ |
| 6 | IMP-EXIST-SERVICE | Extender FacturaAdelantadoGenerarService branch PER con persistencia real del PDF (RE-FU-020) | Back | ProquifaDotNet.Finanzas | ⛔ |
| 7 | IMP-EXIST-SERVICE | Extender FacturaAdelantadoPreviewService branch PER con template real GOLPERU_PER_FAC (RE-FU-020) | Back | ProquifaDotNet.Finanzas | — |
| 8 | IMP-EXIST-SERVICE | Integrar persistencia del PDF de Factura Perú en el flujo Validar Cobro Perú | Back | ProquifaDotNet.Finanzas | ⛔ |

> ⛔ Tareas 4, 5, 6 y 8 bloqueadas por: datos fiscales SUNAT del producto inexistentes (RE-FU-020/RE-FU-005 Brecha 1) y por integración OSE/PSE SUNAT pendiente (RE-FU-020/RE-FU-005 Brecha 5).

---

## TAREA 1

**[ RE-FU-022 ] [CREATE-TABL-M] Crear tabla CPEGenerada con datos fiscales del CPE SUNAT**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Factura Perú SUNAT

**Consideraciones previas:**
- La tabla `CPEGenerada` no existe en BD. Es necesaria para almacenar los datos fiscales del Comprobante de Pago Electrónico (CPE tipo 01, UBL 2.1) de Golocaer S.A.C. para clientes Perú.
- **No se reutiliza `CFDIGenerada`** porque los campos son estructuralmente distintos: `CFDIGenerada` tiene campos SAT (FormaPago, MetodoPago, UsoCFDI, RegimenFiscal, LugarExpedicion) que no aplican a Perú, y `CPEGenerada` requiere campos SUNAT exclusivos (TipoOperacion cat.51, UbigeoEmisor, DireccionReceptor, IGV, ISC, ICBPER, OtrosTributos) que no existen en `CFDIGenerada`. Reutilizar implicaría NULLs masivos y confusión en ORM/scaffold.
- Es **BLOQUEANTE** para la Tarea 2 que agrega el FK desde `tpProformaAdelanto`.
- Es **BLOQUEANTE** para el flujo de timbrado SUNAT: `INSERT CPEGenerada` ocurre en el paso de persistencia tras el timbrado exitoso del OSE/PSE.
- Las columnas de totales SUNAT (ISC, ICBPER, OtrosTributos) tienen DEFAULT 0 para no romper el caso más común (solo IGV).

**Objetivo general:**
Crear la tabla `CPEGenerada` en ProquifaDotNet con todos los campos fiscales del CPE UBL 2.1 de Golocaer S.A.C. para Perú, habilitando el almacenamiento de los datos del comprobante electrónico SUNAT como artefacto fiscal inmutable.

**Objetivos específicos:**
- Ejecutar el DDL con columnas: `IdCPEGenerada` (PK NEWID), `RUCEmisor` varchar(11), `RazonSocialEmisor` varchar(200), `DireccionEmisor` varchar(300), `UbigeoEmisor` varchar(6), `RUCReceptor` varchar(11), `RazonSocialReceptor` varchar(200), `DireccionReceptor` varchar(300), `TipoComprobante` varchar(2) DEFAULT '01', `TipoOperacion` varchar(4), `Serie` varchar(4), `Correlativo` varchar(8), `CondicionPago` varchar(50), `Moneda` varchar(3), `TipoCambio` decimal(18,6), `ValorVenta` decimal(18,2), `IGV` decimal(18,2), `ISC` decimal(18,2) DEFAULT 0, `ICBPER` decimal(18,2) DEFAULT 0, `OtrosTributos` decimal(18,2) DEFAULT 0, `Total` decimal(18,2), `Observaciones` varchar(500), `FechaEmision` datetime2(7), `Activo` bit DEFAULT 1, `FechaRegistro` datetime2(7) DEFAULT SYSUTCDATETIME(), `FechaUltimaActualizacion` datetime2(7) DEFAULT SYSUTCDATETIME().
- Validar que PK, DEFAULT constraints y tipos de dato queden correctamente definidos.
- Verificar que ningún objeto existente en BD se ve afectado por la creación.

**Resultado esperado:**
Tabla `CPEGenerada` existente en ProquifaDotNet, lista para recibir los datos fiscales del CPE SUNAT al timbrar exitosamente ante el OSE/PSE.

**Entregables:**
- Script DDL: `CREATE TABLE CPEGenerada`
- Script de validación (`SELECT` estructura + constraints)

**Criterios de aceptación:**
- La tabla existe con la estructura definida en `R16A-RE-FU-022_BD.md`.
- Todos los DEFAULT constraints están correctamente configurados (TipoComprobante='01', ISC=0, ICBPER=0, OtrosTributos=0, Activo=1, FechaRegistro=SYSUTCDATETIME()).
- Ningún objeto existente en BD presenta errores tras la creación.

**Más información de la tarea:**
Ver sección *"CREATE TABLE CPEGenerada"* y *"Columnas de CPEGenerada"* en `R16A-RE-FU-022_BD.md`. Ver sección *"Decisión: CPEGenerada vs reutilizar CFDIGenerada"* para justificación del diseño.

**Recursos:**
- `R16A-RE-FU-022_BD.md` — Script propuesto `CREATE TABLE CPEGenerada`
- `R16A-RE-FU-022-Back.md` — Parte C, paso 1

---

## TAREA 2

**[ RE-FU-022 ] [UPDATE-TABL-CH] Agregar IdCPEGenerada (FK) a tabla tpProformaAdelanto**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Factura Perú SUNAT

**Consideraciones previas:**
- La tabla `tpProformaAdelanto` ya tiene el campo `IdCFDIGenerada` (FK hacia `CFDIGenerada`) para México (RE-FU-019). El equivalente para Perú es `IdCPEGenerada` (FK hacia `CPEGenerada`).
- Depende de la Tarea 1 (`CPEGenerada` debe existir antes de agregar el FK).
- La columna se agrega como NULL para no romper registros existentes (los pedidos México no tienen `IdCPEGenerada`).
- Verificar que los ALTER no rompan vistas, stored procedures ni triggers dependientes de `tpProformaAdelanto`.

**Objetivo general:**
Ampliar la tabla `tpProformaAdelanto` con el campo `IdCPEGenerada` (FK hacia `CPEGenerada`) para vincular cada Proforma de Adelanto Perú con su Comprobante de Pago Electrónico SUNAT, de forma análoga a como `IdCFDIGenerada` vincula las Proformas México con `CFDIGenerada`.

**Objetivos específicos:**
- `ALTER TABLE dbo.tpProformaAdelanto ADD IdCPEGenerada uniqueidentifier NULL CONSTRAINT [FK_tpProformaAdelanto_CPEGenerada] FOREIGN KEY REFERENCES dbo.CPEGenerada([IdCPEGenerada])`.
- Verificar que los ALTER no rompan vistas, stored procedures ni triggers dependientes de `tpProformaAdelanto`.
- Validar que el campo queda en NULL para todos los registros existentes.

**Resultado esperado:**
`tpProformaAdelanto` con el campo `IdCPEGenerada` disponible y con FK activo hacia `CPEGenerada`, listo para ser poblado al completar el timbrado SUNAT exitoso.

**Entregables:**
- Script DDL: `ALTER TABLE tpProformaAdelanto ADD IdCPEGenerada`
- Script de validación (`SELECT` con los nuevos campos)
- Checklist de objetos dependientes verificados

**Criterios de aceptación:**
- `tpProformaAdelanto.IdCPEGenerada` existe y acepta valores `uniqueidentifier` NULL.
- FK activo hacia `CPEGenerada.IdCPEGenerada`.
- Ningún SP, vista ni trigger presenta errores de compilación tras el ALTER.
- Todos los registros existentes tienen `IdCPEGenerada` = NULL.

**Más información de la tarea:**
Ver sección *"Vinculación CPEGenerada con tpProformaAdelanto"* en `R16A-RE-FU-022_BD.md`. Ver `R16A-RE-FU-022-Back.md` Parte C, paso 2.

**Recursos:**
- `R16A-RE-FU-022_BD.md` — Script propuesto ALTER tpProformaAdelanto
- `R16A-RE-FU-022-Back.md` — Parte C, paso 2

---

## TAREA 3

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
- Depende de las Tareas 1-2 (campos SUNAT en BD disponibles) y de RE-FU-020 (CodigoSUNAT, ClaveSUNAT, catAfectacionIGV disponibles para armar las partidas del body).
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

## TAREA 4

**[ RE-FU-022 ] [ALG-COMPLX-LOGIC] Algoritmo de mapeo de datos CPE UBL 2.1 al modelo del PDF de Factura Perú**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura Perú — Generación PDF CPE UBL 2.1

**Consideraciones previas:**
- ⛔ **BLOQUEANTE:** depende de RE-FU-020 para los campos `CodigoSUNAT` (Producto), `ClaveSUNAT` (catUnidad) e `IdCatAfectacionIGV` (Producto). Sin estos campos no es posible armar las partidas del PDF conforme al estándar SUNAT (ver RE-FU-005 Brecha 1).
- ⛔ **BLOQUEANTE:** depende de la integración OSE/PSE SUNAT (RE-FU-020) para obtener el XML firmado del cual se extraen QR, Firma Digital y Valor Resumen.
- El PDF de la Factura Perú integra datos de múltiples tablas: `CPEGenerada`, `tpPartidaPedido`, `Producto`, `catUnidad`, `catAfectacionIGV`, `Empresa` (GOLPERU) y los elementos técnicos del XML firmado retornado por el OSE/PSE.
- Los datos del QR, Firma Digital y Valor Resumen se extraen del XML del OSE/PSE, no de BD directamente.
- La comunicación con ProquifaDotNet (.NET Framework 4.8) para obtener los datos del pedido y del CPE se realiza mediante llamadas entre APIs.
- Aplica a Facturas de FAA Perú (RE-FU-020) y de Validar Cobro Perú. Solo empresa GOLPERU.
- Es el contraparte peruano de la Tarea 9 de RE-FU-021 (`FacturaMexicoPdfMappingService`); mismo patrón, tabla y campos SUNAT distintos.

**Objetivo general:**
Implementar el servicio `FacturaPeruPdfMappingService` que consolida todos los datos del CPE UBL 2.1 en un modelo unificado (`FacturaPeruPdfModel`), listo para ser consumido por DocumentBuilder con el template `GOLPERU_PER_FAC`, incluyendo branding de Golocaer S.A.C., datos fiscales SUNAT del emisor y receptor, partidas con códigos SUNAT, totales SUNAT e IGV, elementos técnicos de certificación y QR.

**Objetivos específicos:**
- Crear `FacturaPeruPdfModel` con secciones: Branding (LogoBase64), Emisor (RUC, RazonSocial, Dirección, Ubigeo), Receptor (RUC, RazonSocial, Dirección), Comprobante (Serie, Correlativo, TipoComprobante, TipoOperación, FechaEmision, CondiciónPago, Moneda, TC, FolioPedidoInterno), Partidas `List<FacturaPeruPdfPartidaModel>`, Totales SUNAT (SubTotalVentas, Anticipos, Descuentos, ValorVentaGratuitas, ValorVenta, ISC, IGV, ICBPER, OtrosCargos, OtrosTributos, Redondeo, ImporteTotal, TotalEnLetras), Crédito `FacturaPeruPdfCreditoModel` (MontoNetoPendiente, TotalCuotas, Cuotas), Elementos Técnicos SUNAT (QRBase64, ValorResumen, FirmaDigital — null en preview).
- Mapear `FacturaPeruPdfPartidaModel` por partida: Cantidad, UnidadMedidaSUNAT (catUnidad.ClaveSUNAT), CodigoSUNAT (Producto.CodigoSUNAT — BRECHA RE-FU-020), NumeroOrdenCompra (referencia del pedido del cliente, criterio D2), Descripción (nombre + catálogo + lote), ValorUnitario, Importe, AfectaciónIGV (catAfectacionIGV.Clave — BRECHA RE-FU-020), ICBPERLinea, TipoPrecio (catálogo 16 SUNAT — campo opcional, observado en factura real E001-362; pendiente confirmar con cliente).
- Parsear `xmlFirmadoSunat` → extraer QR, FirmaDigital, ValorResumen del CDR del OSE/PSE.
- Calcular TotalEnLetras según moneda (ej: "SON: DIECIOCHO MIL SETECIENTOS CUATRO Y 18/100 DOLAR AMERICANO").
- Implementar `IFacturaPeruPdfMappingService` con dos métodos: `MapearAsync(idCPEGenerada, xmlFirmadoSunat)` (PDF definitivo) y `MapearPreviewAsync(idCPEGenerada)` (preview sin elementos técnicos SUNAT).

**Resultado esperado:**
Servicio `FacturaPeruPdfMappingService` que recibe el `IdCPEGenerada` y el XML firmado del OSE/PSE, y retorna un `FacturaPeruPdfModel` completamente poblado, listo para ser pasado al generador de DocumentBuilder (Tarea 3) tanto para preview como para PDF definitivo.

**Entregables:**
- Clase `FacturaPeruPdfModel` con todas las secciones del CPE PDF
- Clase `FacturaPeruPdfPartidaModel`, `FacturaPeruPdfCreditoModel`, `FacturaPeruPdfCuotaModel`
- Interfaz `IFacturaPeruPdfMappingService` + servicio `FacturaPeruPdfMappingService`
- Prueba unitaria con datos de la factura real de muestra (E001-362)

**Criterios de aceptación:**
- El modelo generado contiene todos los campos requeridos por las secciones A-J del PDF (criterios del requisito).
- Los campos `CodigoSUNAT`, `ClaveSUNAT` y `AfectaciónIGV` de cada partida se mapean correctamente desde los campos nuevos de RE-FU-020.
- El QR, FirmaDigital y ValorResumen se extraen del XML del OSE/PSE, no se calculan en la aplicación.
- `MapearPreviewAsync` retorna el modelo con la Sección G (Elementos Técnicos) en null.
- El TotalEnLetras se genera correctamente para PEN y USD.

**Más información de la tarea:**
Ver sección *"Parte B — FacturaPeruPdfMappingService"* en `R16A-RE-FU-022-Back.md`. Ver Tarea 4 de RE-FU-020 (datos SUNAT del producto como prerequisito). Es el contraparte peruano de la Tarea 9 de RE-FU-021.

**Recursos:**
- `R16A-RE-FU-022-Back.md` — Parte B, modelo y flujo MapearAsync
- `R16A-RE-FU-022-Pendiente.md` — Criterios A1-J2, Reglas 7 y 8
- R16A-RE-FU-021 Tarea 9 — `FacturaMexicoPdfMappingService` (patrón equivalente)
- Factura real de muestra: Golocaer S.A.C. E001-362

---

## TAREA 5

**[ RE-FU-022 ] [SERV-TRANSACT] Persistencia del PDF de Factura Perú en Minio tras timbrado SUNAT exitoso**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura Perú — Persistencia Artefacto Fiscal

**Consideraciones previas:**
- ⛔ **BLOQUEANTE:** depende de la Tarea 4 (`FacturaPeruPdfMappingService`) y de la integración OSE/PSE SUNAT (RE-FU-020) que retorna el XML firmado necesario para el mapeo.
- Se ejecuta inmediatamente después del timbrado exitoso ante SUNAT/OSE, como parte del flujo de los módulos que generan facturas Perú (FAA RE-FU-020 y Validar Cobro Perú).
- El PDF se genera una única vez al timbrar; posteriormente se sirve desde Minio sin regeneración (criterio J2). La factura timbrada es artefacto fiscal inmutable.
- La persistencia utiliza el patrón existente: `INSERT Archivo` con `FileBucket='facturas'` e `IdRegion='PER'`.
- Si la generación o persistencia del PDF falla tras el timbrado exitoso, debe reintentarse **sin re-timbrar** ante SUNAT/OSE.
- La comunicación con ProquifaDotNet (.NET Framework 4.8) para actualizar referencias se realiza mediante llamadas entre APIs.
- TemplateKey fijo: `GOLPERU_PER_FAC` (única empresa emisora Perú).
- Depende de las Tareas 1-4.
- Es el contraparte peruano de la Tarea 10 de RE-FU-021 (`PersistirFacturaMexicoPdfService`).

**Objetivo general:**
Implementar el servicio `PersistirFacturaPeruPdfService` que, tras el timbrado exitoso ante SUNAT/OSE, genera el PDF definitivo del CPE UBL 2.1 (con QR, Firma Digital y Valor Resumen completos) y lo persiste en Minio vía la tabla `Archivo` con `IdRegion='PER'`, referenciado desde `tpProformaAdelanto.IdCPEGenerada`.

**Objetivos específicos:**
- Invocar `FacturaPeruPdfMappingService.MapearAsync(IdCPEGenerada, xmlFirmadoSunat)` para obtener el `FacturaPeruPdfModel` definitivo.
- Invocar DocumentBuilder con `TemplateKey='GOLPERU_PER_FAC'` para generar el PDF en bytes.
- `INSERT Archivo` (PDF, `FileBucket='facturas'`, `IdRegion='PER'`, `ContentType='application/pdf'`) → obtener `IdArchivo`.
- Llamar API ProquifaDotNet → `UPDATE tpProformaAdelanto SET IdCPEGenerada` con referencia al PDF.
- Garantizar atomicidad: si la persistencia en Minio falla, reintentar sin re-timbrar ante SUNAT/OSE.
- Registrar en Serilog: módulo, `IdCPEGenerada`, fecha, resultado (éxito/error + mensaje).

**Resultado esperado:**
PDF del CPE UBL 2.1 persistido en Minio como artefacto fiscal inmutable, referenciado desde `tpProformaAdelanto`, disponible para consultas y envíos posteriores sin regeneración.

**Entregables:**
- Servicio `PersistirFacturaPeruPdfService` (o equivalente)
- Manejo de reintentos de persistencia sin re-timbrado ante SUNAT/OSE
- Registro en log con Serilog (módulo, `IdCPEGenerada`, fecha, resultado)

**Criterios de aceptación:**
- El PDF se genera con QR, Firma Digital y Valor Resumen del OSE/PSE integrados.
- El `INSERT Archivo` persiste correctamente en Minio con `FileBucket='facturas'` e `IdRegion='PER'`.
- La referencia del PDF queda correctamente almacenada en `tpProformaAdelanto`.
- Si la persistencia falla, el sistema reintenta sin volver a llamar al OSE/PSE SUNAT.
- Al consultar el PDF posteriormente, el sistema sirve el archivo desde Minio sin regenerarlo.
- La operación queda registrada en Serilog con `IdCPEGenerada`, fecha y resultado.

**Más información de la tarea:**
Ver sección *"Parte B — PersistirFacturaPeruPdfService"* en `R16A-RE-FU-022-Back.md`. Ver criterios J1-J2, Reglas 1 y 5 en `R16A-RE-FU-022-Pendiente.md`. Es el contraparte peruano de la Tarea 10 de RE-FU-021.

**Recursos:**
- `R16A-RE-FU-022-Back.md` — Parte B, flujo PersistirFacturaPeruPdfService
- `R16A-RE-FU-022-Pendiente.md` — Criterios J1-J2, Reglas 1 y 5
- R16A-RE-FU-021 Tarea 10 — `PersistirFacturaMexicoPdfService` (patrón equivalente)

---

## TAREA 6

**[ RE-FU-022 ] [IMP-EXIST-SERVICE] Extender FacturaAdelantadoGenerarService branch PER con persistencia real del PDF (RE-FU-020)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura por Adelantado — Generación Perú

**Consideraciones previas:**
- ⛔ **BLOQUEANTE:** depende de la Tarea 5 (`PersistirFacturaPeruPdfService`) y de la integración OSE/PSE SUNAT (RE-FU-020).
- El branch PER de `FacturaAdelantadoGenerarService` (RE-FU-020) almacena el XML firmado y el CDR en Minio tras el timbrado exitoso. El PDF definitivo del CPE queda como placeholder pendiente de este requisito.
- Esta tarea implementa ese placeholder con la llamada real a `PersistirFacturaPeruPdfService` (Tarea 5), de forma análoga a la Tarea 11 de RE-FU-021 para el branch MEX.
- No se modifica la lógica de timbrado SUNAT ni los pasos previos del branch PER; solo se agrega la llamada al servicio de persistencia del PDF.
- Aplica únicamente a facturas de región Perú (GOLPERU). El branch MEX (Tarea 11 de RE-FU-021) tiene su propia integración.

**Objetivo general:**
Integrar la llamada a `PersistirFacturaPeruPdfService` en el branch PER de `FacturaAdelantadoGenerarService` (RE-FU-020), completando el flujo de generación de la Factura por Adelantado Perú con la persistencia del PDF definitivo del CPE UBL 2.1 en Minio inmediatamente tras el timbrado exitoso ante SUNAT/OSE.

**Objetivos específicos:**
- Inyectar `IPersistirFacturaPeruPdfService` en `FacturaAdelantadoGenerarService`.
- Agregar la llamada `await _persistirFacturaPeruPdfService.PersistirAsync(idCPEGenerada, response.XmlFirmado)` en el branch PER tras el timbrado exitoso.
- Validar que si `PersistirAsync` falla, el error se maneja sin re-timbrar ante SUNAT/OSE.
- Registrar en Serilog el resultado de la persistencia dentro del flujo de generación.

**Resultado esperado:**
Branch PER de `FacturaAdelantadoGenerarService` con la persistencia del PDF CPE implementada definitivamente: tras el timbrado exitoso, el PDF se genera y persiste en Minio antes de retornar el response al cliente.

**Entregables:**
- `FacturaAdelantadoGenerarService` branch PER actualizado (llamada real a `PersistirFacturaPeruPdfService`)
- Prueba de integración: flujo completo generar FAA Perú → PDF persistido en Minio

**Criterios de aceptación:**
- Tras el timbrado SUNAT exitoso, el PDF se persiste en Minio y la referencia queda en `tpProformaAdelanto` antes de retornar el response.
- Si la persistencia del PDF falla, el sistema no re-timbra ante SUNAT/OSE y retorna el error correspondiente.
- El flujo de timbrado SUNAT (pasos previos) no presenta regresiones respecto a RE-FU-020.
- La operación queda registrada en Serilog con `IdCPEGenerada`, fecha y resultado.

**Más información de la tarea:**
Ver sección *"Integración con RE-FU-020 — branch PER"* en `R16A-RE-FU-022-Back.md`. Ver Tarea 11 de RE-FU-021 (patrón equivalente para México).

**Recursos:**
- `R16A-RE-FU-022-Back.md` — Parte B, integración branch PER FacturaAdelantadoGenerarService
- R16A-RE-FU-021 Tarea 11 — `FacturaAdelantadoGenerarService` MEX (patrón equivalente)

---

## TAREA 7

**[ RE-FU-022 ] [IMP-EXIST-SERVICE] Extender FacturaAdelantadoPreviewService branch PER con template real GOLPERU_PER_FAC (RE-FU-020)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura por Adelantado — Previsualización PDF Perú

**Consideraciones previas:**
- El branch PER de `FacturaAdelantadoPreviewService` (RE-FU-020) tiene un placeholder para el template del PDF de previsualización, pendiente de definir en este requisito.
- Esta tarea reemplaza ese placeholder con el template real `GOLPERU_PER_FAC` de DocumentBuilder, de forma análoga a la Tarea 12 de RE-FU-021 para el branch MEX.
- Depende de la Tarea 3 (template `GOLPERU_PER_FAC` registrado en DocumentBuilder) y de la Tarea 4 (`FacturaPeruPdfMappingService.MapearPreviewAsync` disponible).
- El preview usa el modelo sin elementos técnicos SUNAT (Sección G en null): QR, Firma Digital y Valor Resumen no están presentes hasta el timbrado real.
- El TemplateKey es fijo (`GOLPERU_PER_FAC`) ya que solo hay una empresa emisora Perú, a diferencia del switch por empresa de México.
- Aplica únicamente a facturas de región Perú (GOLPERU).

**Objetivo general:**
Reemplazar el template placeholder del branch PER de `FacturaAdelantadoPreviewService` (RE-FU-020) con la resolución del template real `GOLPERU_PER_FAC` de DocumentBuilder, completando el flujo de previsualización del PDF de la Factura por Adelantado Perú.

**Objetivos específicos:**
- Inyectar `IFacturaPeruPdfMappingService` en `FacturaAdelantadoPreviewService`.
- Invocar `MapearPreviewAsync(idCPEGenerada)` para obtener el `FacturaPeruPdfModel` sin elementos técnicos SUNAT.
- Invocar DocumentBuilder con `TemplateKey='GOLPERU_PER_FAC'` (fijo — única empresa Perú).
- Retornar el PDF en memoria sin persistir en Minio (el preview no es artefacto fiscal).

**Resultado esperado:**
Branch PER de `FacturaAdelantadoPreviewService` con la generación real del PDF de previsualización usando el template `GOLPERU_PER_FAC` de DocumentBuilder, con todos los campos del CPE disponibles excepto QR, Firma Digital y Valor Resumen.

**Entregables:**
- `FacturaAdelantadoPreviewService` branch PER actualizado (template real, sin placeholder)
- Prueba de integración: flujo completo preview FAA Perú → PDF en memoria con datos correctos

**Criterios de aceptación:**
- El PDF de preview generado usa el template `GOLPERU_PER_FAC` (branding Golocaer S.A.C. Perú).
- El PDF de preview NO contiene QR, Firma Digital ni Valor Resumen (Sección G vacía o ausente).
- El flujo existente del branch PER (pasos previos al template) no presenta regresiones respecto a RE-FU-020.

**Más información de la tarea:**
Ver sección *"Integración con RE-FU-020 — branch PER Preview"* en `R16A-RE-FU-022-Back.md`. Ver Tarea 12 de RE-FU-021 (patrón equivalente para México).

**Recursos:**
- `R16A-RE-FU-022-Back.md` — Parte B, integración branch PER FacturaAdelantadoPreviewService
- R16A-RE-FU-021 Tarea 12 — `FacturaAdelantadoPreviewService` MEX (patrón equivalente)

---

## TAREA 8

**[ RE-FU-022 ] [IMP-EXIST-SERVICE] Integrar persistencia del PDF de Factura Perú en el flujo Validar Cobro Perú**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Generación Factura Perú

**Consideraciones previas:**
- ⛔ **BLOQUEANTE:** depende de la Tarea 5 (`PersistirFacturaPeruPdfService`) y de la integración OSE/PSE SUNAT (RE-FU-020).
- El requisito RE-FU-022 es explícito en su alcance: "Facturas originadas en Validar Cobro (cobro recibido aplicado a Proforma de Prepago sin controlados, Región Perú)."
- En el flujo de Validar Cobro Perú, el punto de timbrado exitoso ante SUNAT/OSE es donde debe invocarse `PersistirFacturaPeruPdfService` (Tarea 5) para generar y persistir el PDF como artefacto fiscal inmutable.
- Aplica únicamente a facturas de región Perú (GOLPERU).
- Si la generación o persistencia del PDF falla tras el timbrado exitoso, debe reintentarse sin re-timbrar ante SUNAT/OSE.
- **No existe Complemento de Pago en Perú** (SUNAT no tiene documento equivalente), por lo que el flujo de Validar Cobro Perú es más simple que el de México.
- Es el contraparte peruano de la Tarea 13 de RE-FU-021 (integración PDF en Validar Cobro México).

**Objetivo general:**
Integrar la llamada a `PersistirFacturaPeruPdfService` en el servicio de Validar Cobro que ejecuta el timbrado de la factura Perú, de modo que el PDF del CPE UBL 2.1 se genere y persista en Minio como artefacto fiscal inmutable inmediatamente después del timbrado exitoso ante SUNAT/OSE.

**Objetivos específicos:**
- Identificar el servicio/comando de Validar Cobro Perú en ProquifaDotNet.Finanzas que ejecuta el timbrado SUNAT.
- Inyectar `IPersistirFacturaPeruPdfService` en dicho servicio.
- Invocar `PersistirAsync(idCPEGenerada, xmlFirmadoSunat)` inmediatamente después del timbrado exitoso.
- Validar que si `PersistirAsync` falla, el error se maneja sin re-timbrar ante SUNAT/OSE.
- Registrar en Serilog el resultado (módulo, `IdCPEGenerada`, fecha, resultado).

**Resultado esperado:**
El flujo de Validar Cobro Perú genera y persiste el PDF del CPE UBL 2.1 en Minio como artefacto fiscal inmutable tras el timbrado exitoso, de forma consistente con el comportamiento implementado para FAA Perú (Tarea 6).

**Entregables:**
- Servicio de Validar Cobro Perú actualizado con llamada real a `PersistirFacturaPeruPdfService`
- Manejo de error de persistencia sin re-timbrado ante SUNAT/OSE
- Registro en Serilog (módulo, `IdCPEGenerada`, fecha, resultado)

**Criterios de aceptación:**
- Tras el timbrado SUNAT exitoso en Validar Cobro Perú, el PDF se persiste en Minio y la referencia queda en `tpProformaAdelanto`.
- Si la persistencia del PDF falla, el sistema no re-timbra ante SUNAT/OSE.
- El flujo de Validar Cobro Perú no presenta regresiones.
- La operación queda registrada en Serilog con `IdCPEGenerada`, fecha y resultado.

**Más información de la tarea:**
Ver sección *"GAP-07 — Validar Cobro Perú"* en `R16A-RE-FU-022-Back.md`. Ver Tarea 13 de RE-FU-021 (patrón equivalente para México). Ver alcance en `R16A-RE-FU-022-Pendiente.md` (facturas originadas en Validar Cobro Perú).

**Recursos:**
- `R16A-RE-FU-022-Back.md` — GAP-07 Validar Cobro Perú
- `R16A-RE-FU-022-Pendiente.md` — Alcance, Criterios J1-J2
- R16A-RE-FU-021 Tarea 13 — Integración Validar Cobro México (patrón equivalente)
