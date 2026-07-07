# Impacto en Back — R16A-RE-FU-032
**Requisito:** Notas de Crédito México (CFDI tipo E — Egreso)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder
**Módulo:** Notas de Crédito México — Módulo independiente (Tesorería)
**Impacto:** Scripts BD ProquifaDotNet (2 ALTER TABLE + 2 DML catálogos) + Scripts BD ProquifaDotNetTimbrado (1 DML EmpresaFolio) + Scripts DocumentBuilder (4 DML DocumentTemplate) + Módulo NC Finanzas: wizard 4 pasos, construcción XML CFDI E CFDI 4.0, previsualización PDF, persistencia en MinIO, envío correo + Timbrado: generación CFDI E + DocumentBuilder: 4 templates PDF NC México + ETL SSIS: transferencia NC a PCconnect (Legacy).

## Revisiones aplicadas

| # | Cambio | Origen |
|---|--------|--------|
| 1 | **Criterio A5 agregado en B7:** Pantalla principal filtrada por cartera del Cobrador autenticado (JOIN `vUsuarioCartera`) | OBS-004 |

---

## Resumen

RE-032 implementa el **módulo independiente de Notas de Crédito para Región México**, operado por el área de Tesorería. El módulo es completamente independiente de Validar Cobro; el acoplamiento es uni-direccional: las NCs vigentes quedan disponibles en el Paso 2 de Validar Cobro, pero Validar Cobro no genera ni cancela NCs.

El wizard de 4 pasos (Buscar Factura → Capturar Datos → Confirmar + Previsualización → NC Emitida) permite seleccionar una factura vigente de prepago como origen, capturar la NC en modalidad **por partidas** (devolución de mercancía, con cálculo automático) o **manual** (descuento/bonificación, monto libre), previsualizarla antes de timbrar y emitirla vía PAC TurboPac como CFDI tipo E.

Una vez timbrada, la NC se transfiere al sistema legado **PCconnect** mediante **SSIS**.

La NC aplica exclusivamente a **clientes prepago** y a **facturas vigentes con antigüedad máxima de 5 años**. La cancelación de la factura origen es condicional (NC por totalidad + mismo mes calendario).

### Distribución de responsabilidades

| Capa                     | Aplicativo                   | Responsabilidad                                                                                     |
| ------------------------ | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| BD — ALTER tabla         | ProquifaDotNet               | `fccNotaCredito`: ADD 13 columnas R16                                                               |
| BD — ALTER tabla         | ProquifaDotNet               | `fccNotaCreditoPartida`: ADD 6 columnas R16                                                         |
| BD — DML catálogo        | ProquifaDotNet               | `catUsoCFDI`: INSERT G02 si no existe                                                               |
| BD — DML catálogo        | ProquifaDotNet               | `catTipoCFDI`: INSERT NOTA_CREDITO (prereq RE-028)                                                  |
| BD — DML foliador        | ProquifaDotNetTimbrado       | `EmpresaFolio`: INSERT 4 filas Serie "P2" (GOL, MUN, PRO, PQF)                                     |
| BD — DML templates       | DocumentBuilder              | `DocumentTemplate`: INSERT 4 templates PDF NC México                                                |
| BD — DML bucket          | ProquifaDotNet               | `RegionConfiguracionMinioBucket`: INSERT bucket NC MEX si no existe                                 |
| Wizard Paso 1            | ProquifaDotNet.Finanzas      | Búsqueda y listado de facturas vigentes prepago del cliente (máx. 5 años)                           |
| Wizard Paso 2            | ProquifaDotNet.Finanzas      | Captura de modalidad, motivo, partidas/monto, cancelación condicional de factura                    |
| Previsualización PDF     | ProquifaDotNet.Finanzas      | `CreditNoteMexicoPdfMappingService.MapearPreviewAsync()` — sin sello, sin UUID                              |
| Wizard Paso 3            | ProquifaDotNet.Finanzas      | Confirmación, resumen, previsualización PDF, acción Timbrar                                         |
| Construcción XML CFDI E  | ProquifaDotNet.Finanzas      | Armado del `CreditNoteMexicoRequest`: CFDI 4.0 TipoDocumento=E, conceptos, impuestos               |
| Timbrado NC              | ProquifaDotNet.Timbrado      | Generación CFDI E + PAC TurboPac + `INSERT CFDIGenerada` + `UPDATE EmpresaFolio` Serie P2           |
| Cancelación factura      | ProquifaDotNet.Timbrado      | Llamada condicional al PAC para cancelar la factura origen ante el SAT                              |
| Persistencia post-timbre | ProquifaDotNet.Finanzas      | `PersistMexicoCreditNotePdfService`: DocumentBuilder → MinIO → `INSERT Archivo` → `UPDATE fccNotaCredito` |
| Correo automático        | ProquifaDotNet.Finanzas      | Envío con PDF + XML adjuntos al timbrar (Para = contacto; CC = ESAC + CxC)                         |
| Acoplamiento Validar Cobro | ProquifaDotNet.Finanzas    | NCs VIGENTE quedan disponibles en Paso 2 de Validar Cobro vía query sobre `fccNotaCredito`          |
| ETL Legacy               | SSIS                         | Transferencia de NC timbrada a PCconnect                                                            |
| Comunicación             | Finanzas → Timbrado          | Llamada API por NC a timbrar (mismo patrón que Factura)                                             |
| Comunicación             | Finanzas → ProquifaDotNet    | Lecturas: `CFDIGenerada`, `DatosFacturacionCliente`, `Empresa`, `fccNotaCredito`                    |

### Infraestructura reutilizada

| Componente                                     | Origen    | Reutilización en RE-032                                                                 |
| ---------------------------------------------- | --------- | --------------------------------------------------------------------------------------- |
| `fccNotaCredito` (estructura)                  | Pre-R16   | Base extendida; ciclo de estados PENDIENTE → GENERADA → ENVIADA                        |
| `fccNotaCreditoPartida` (estructura)           | Pre-R16   | Base extendida con columnas R16 de importes fiscales                                   |
| `fccNotaCreditoPedido`                         | Pre-R16   | Registro de aplicación de NC a pedido en Validar Cobro (sin cambios)                   |
| `CFDIGenerada`                                 | RE-019    | INSERT NC como TipoDocumento='E', MetodoDePago='PUE'                                   |
| `CFDIGeneradaConcepto`                         | RE-019    | INSERT conceptos de la NC (por partidas o concepto manual)                              |
| `CFDIGeneradaRelacionado`                      | Pre-R16   | INSERT UUID factura origen con ClaveTipoRelacion='01'                                  |
| `CFDICancelacion`                              | Pre-R16   | INSERT condicional si usuario cancela factura origen                                    |
| `CFDI`                                         | Pre-R16   | Poblado por Timbrado con UUID SAT, sello y certificado                                 |
| `EmpresaFolio` (estructura)                    | RE-019    | Foliador UPDLOCK atómico; RE-032 agrega filas Serie "P2"                               |
| PAC TurboPac                                   | RE-019    | Mismo cliente/servicio para timbrar la NC vía ProquifaDotNet.Timbrado                  |
| `catTipoCFDI` (tabla)                          | RE-028 T1 | RE-032 inserta clave NOTA_CREDITO                                                      |
| `Archivo`                                      | Pre-R16   | PDF + XML de la NC almacenados en MinIO                                                |
| `CorreoEnviado` + `ArchivoCorreoEnviado`       | Pre-R16   | Trazabilidad del correo automático al timbrar y reenvíos                               |
| `ApiCallerStamping` (HttpClient + Polly)       | RE-019    | Cliente HTTP con retry hacia Timbrado — reutilizado sin cambios                        |
| `MexicoInvoicePdfMappingService`               | RE-021    | Patrón de referencia para `CreditNoteMexicoPdfMappingService`                                  |
| `PersistMexicoInvoicePdfService`             | RE-021    | Patrón de referencia para `PersistMexicoCreditNotePdfService`                                |
| Templates `GOL/MUN/PRO/PQF_MEX_FAC`           | RE-021    | Referencia de branding para diseño de templates NC                                     |
| `DatosFacturacionCliente`                      | RE-004    | RFC, Razón Social, RegimenFiscalReceptor del CFDI 4.0                                  |
| `Empresa`                                      | Existente | RFC Emisor, RegimenFiscal, Prefijo por empresa PROQUIFA México                         |
| `RegionConfiguracionMinioBucket`               | Existente | Resolución del bucket 'notas_credito' (MEX) al subir PDF/XML                          |

---

## Parte A — Base de Datos

### A1 — ALTER TABLE fccNotaCredito — Columnas R16

Se agregan 13 columnas a la tabla existente para soportar el módulo R16:

| Columna                       | Tipo             | Nulable | Propósito                                                                |
| ----------------------------- | ---------------- | ------- | ------------------------------------------------------------------------ |
| `IdEmpresa`                   | uniqueidentifier | No      | Empresa emisora (GOL, MUN, PRO, PQF)                                    |
| `IdCliente`                   | uniqueidentifier | No      | Cliente receptor                                                         |
| `Serie`                       | varchar(10)      | Sí      | Serie "P2" ⚠️ pendiente validar con PMO                                |
| `Modalidad`                   | varchar(20)      | No      | 'POR_PARTIDAS' \| 'MANUAL'                                              |
| `Motivo`                      | varchar(50)      | Sí      | Clave del motivo SAT principal                                           |
| `Estado`                      | varchar(20)      | No      | 'VIGENTE' \| 'CANCELADA' — default 'VIGENTE'                            |
| `CancelarFacturaOrigen`       | bit              | No      | 1 si se solicitó cancelar la factura origen ante el SAT — default 0     |
| `ClaveMotivosCancelacion`     | varchar(4)       | Sí      | Clave SAT c_MotivoCancelacion ('01','02','03','04'). Null si no cancela  |
| `IdCFDIGeneradaFacturaOrigen` | uniqueidentifier | No      | FK `CFDIGenerada` — factura PPD que originó la NC                       |
| `ConceptoManual`              | nvarchar(500)    | Sí      | Materialidad fiscal en modalidad MANUAL                                  |
| `ObservacionesManual`         | nvarchar(500)    | Sí      | Observaciones opcionales en modalidad MANUAL                             |
| `IdArchivoXml`                | uniqueidentifier | Sí      | FK `Archivo` — XML timbrado. Null hasta timbrado exitoso                 |
| `IdArchivoPdf`                | uniqueidentifier | Sí      | FK `Archivo` — PDF representativo. Null hasta generación                 |

> ⚠️ **Brecha B1:** `IdTPProformaPedido` es NOT NULL en la tabla existente. Las NCs R16 no
> provienen de un pedido directo — confirmar si la columna puede ser NULL o requiere un
> valor placeholder.

> Ver script completo en `R16A-RE-FU-032_BD.md` — ALTER TABLE fccNotaCredito.

### A2 — ALTER TABLE fccNotaCreditoPartida — Columnas R16

Se agregan 6 columnas para soportar la modalidad por partidas con trazabilidad al concepto
original de la factura:

| Columna                        | Tipo             | Nulable | Propósito                                             |
| ------------------------------ | ---------------- | ------- | ----------------------------------------------------- |
| `IdCFDIGeneradaConceptoOrigen` | uniqueidentifier | Sí      | FK `CFDIGeneradaConcepto` — concepto de la factura   |
| `CantidadNC`                   | decimal(18,6)    | Sí      | Cantidad a devolver (0, parcial o total)              |
| `Importe`                      | decimal(18,6)    | Sí      | CantidadNC × ValorUnitario                            |
| `Subtotal`                     | decimal(18,6)    | Sí      | Subtotal sin IVA de la línea                          |
| `IVA`                          | decimal(18,6)    | Sí      | IVA calculado sobre el Subtotal                       |
| `Total`                        | decimal(18,6)    | Sí      | Subtotal + IVA de la línea                            |

> ⚠️ **Brecha B2:** `NumeroDePiezas` existente es `int` — si hay productos con cantidad
> fraccionaria se requiere cambio de tipo a `decimal(18,6)`.

> Ver script completo en `R16A-RE-FU-032_BD.md` — ALTER TABLE fccNotaCreditoPartida.

### A3 — DML catUsoCFDI — G02

La NC usa `UsoCFDI='G02'` (Devoluciones, descuentos o bonificaciones) por default.
Verificar existencia antes de insertar.

> Ver script en `R16A-RE-FU-032_BD.md` — DML catUsoCFDI.

### A4 — DML catTipoCFDI — NOTA_CREDITO

Discriminador de tipo CFDI para las NCs. Prereq: RE-028 crea la tabla `catTipoCFDI`.

> Ver script en `R16A-RE-FU-032_BD.md` — DML catTipoCFDI.

### A5 — DML EmpresaFolio — Serie "P2"

4 filas en `ProquifaDotNetTimbrado.EmpresaFolio` para GOL, MUN, PRO, PQF.

> Ver script en `R16A-RE-FU-032_BD.md` — DML EmpresaFolio.

### A6 — DML DocumentTemplate — 4 templates NC México

| TemplateKey   | Empresa emisora                  |
| ------------- | -------------------------------- |
| `GOL_MEX_NC`  | Golocaer                         |
| `MUN_MEX_NC`  | Mungen                           |
| `PRO_MEX_NC`  | Proquifa                         |
| `PQF_MEX_NC`  | Proveedora Quimico Farmaceutica  |

> Ver script en `R16A-RE-FU-032_BD.md` — DML DocumentTemplate.

---

## Parte B — ProquifaDotNet.Finanzas: Módulo NC México

### B1 — Wizard Paso 1: Buscar Factura

**Descripción:** Endpoint que retorna las facturas vigentes de prepago de un cliente, elegibles como origen de NC.

**Flujo:**
1. Finanzas recibe `IdCliente`.
2. Consulta `CFDIGenerada` filtrando:
   - `TipoDocumento = 'I'` (Ingreso — facturas)
   - Factura en estado Vigente (no cancelada en `CFDICancelacion`)
   - Cliente prepago del `IdCliente`
   - `FechaEmision >= DATEADD(YEAR, -5, GETDATE())` — máximo 5 años de antigüedad (Regla 10)
3. Retorna: Folio, UUID (`CFDI.UUID`), FechaEmision, Subtotal, Total, Moneda.
4. Sin escrituras en BD.

**Reglas de negocio:**
- Solo muestra facturas Vigente (no canceladas SAT). Regla 3.
- Solo facturas de hasta 5 años de antigüedad. Regla 10.
- Un cliente puede tener NCs sobre varias facturas; el wizard solo permite seleccionar UNA por NC. Criterio C5.

### B2 — Wizard Paso 2: Capturar Datos — Modalidad por Partidas

**Descripción:** Cuando el usuario selecciona motivo "Devolución de mercancía", Finanzas carga las partidas de la factura origen y calcula el monto en tiempo real según las cantidades capturadas.

**Flujo:**
1. Finanzas recibe `IdCFDIGenerada` de la factura origen.
2. Consulta `CFDIGeneradaConcepto WHERE IdCFDIGenerada = @IdFacturaOrigen`.
3. Retorna tabla de partidas: ClaveProductoServicio, NoIdentificacion, Descripcion, ClaveUnidad, Cantidad (original), ValorUnitario.
4. Finanzas renderiza la tabla con columna `CantidadNC` editable (0, parcial, total).
5. En cada cambio de `CantidadNC`, el front calcula en tiempo real:
   ```
   Importe   = CantidadNC × ValorUnitario
   Subtotal  = Σ Importe por partida
   IVA       = Subtotal × TasaIVA (16% o tasa del producto)
   Total NC  = Subtotal + IVA
   ```

**Cancelación condicional (Regla 8):**
- Si todas las `CantidadNC = CantidadFacturada` (totalidad) **Y** `MONTH(FechaFactura) = MONTH(GETDATE())`:
  - Mostrar opción "Cancelar Factura Origen" con combo `c_MotivoCancelacion` SAT.
- En cualquier otro caso, no mostrar la opción.

**Resultado esperado:** `fccNotaCredito` + `fccNotaCreditoPartida` en estado PENDIENTE (pre-timbrado).

### B3 — Wizard Paso 2: Capturar Datos — Modalidad Manual

**Descripción:** Cuando el usuario selecciona motivo "Descuento o bonificación", Finanzas habilita captura libre de monto y concepto.

**Flujo:**
1. No se carga tabla de partidas (Criterio F1).
2. Usuario captura:
   - **Monto Total NC** — editable, en moneda de la factura origen. Máximo = Total de la factura origen (Criterio F2).
   - **Concepto** — obligatorio, texto libre con materialidad fiscal (Criterio F3).
   - **Observaciones** — opcional (Criterio F4).
3. Finanzas calcula IVA sobre el monto:
   ```
   Subtotal = Monto / (1 + TasaIVA)
   IVA      = Monto - Subtotal
   ```
4. Al armar el XML, Finanzas usa ClaveProdServ=84111506 y ClaveUnidad=ACT como default. ⚠️ Pendiente confirmar con PROQUIFA (Criterio F5).

**Resultado esperado:** `fccNotaCredito` en estado PENDIENTE con `ConceptoManual` y `ObservacionesManual` poblados.

### B4 — Armado del XML CFDI E (CFDI 4.0)

**Descripción:** Finanzas construye el `CreditNoteMexicoRequest` que enviará a Timbrado. Todos los campos del XML se calculan antes del timbrado.

**Campos fijos del XML (Regla 6):**

| Campo XML                        | Valor                                             |
| -------------------------------- | ------------------------------------------------- |
| `TipoDeComprobante`              | `E` — Egreso                                     |
| `MetodoPago`                     | `PUE` — fijo e inmutable (Regla 6)               |
| `UsoCFDI` (receptor)             | `G02` — default (Criterio D4 / H2)               |
| `CfdiRelacionados.TipoRelacion`  | `01` — Nota de crédito (Criterio D5)             |
| `CfdiRelacionados.UUID`          | UUID SAT de la factura origen (`CFDI.UUID`)      |

**Campos heredados de la factura origen:**

| Campo XML    | Fuente                                               |
| ------------ | ---------------------------------------------------- |
| `Moneda`     | `CFDIGenerada.Moneda` de la factura origen           |
| `TipoCambio` | TC del día del timbrado (null si MXN)                |
| `FormaPago`  | `CFDIGenerada.FormaPago` de la factura origen pagada (típicamente '03') — ⚠️ modalidad manual pendiente confirmar (Regla 7) |
| `RegimenFiscalEmisor` | `Empresa.RegimenFiscal` de la empresa emisora |
| `RegimenFiscalReceptor` | `DatosFacturacionCliente.RegimenFiscal` |
| `CodigoPostalReceptor` | `DatosFacturacionCliente.CodigoPostal` |

**Conceptos del XML:**

*Modalidad por partidas (Criterio E4/E5):*
Por cada partida con `CantidadNC > 0`, un nodo `Concepto` con:
- `ClaveProdServ`, `ClaveUnidad`, `NoIdentificacion`, `Descripcion` — heredados del `CFDIGeneradaConcepto` de la factura origen.
- `Cantidad` = `CantidadNC` capturada.
- `ValorUnitario` = heredado de la factura origen.
- `Importe` = `CantidadNC × ValorUnitario`.
- Impuestos trasladados recalculados sobre el nuevo importe.

*Modalidad manual (Criterio F5):*
Un único nodo `Concepto` con:
- `ClaveProdServ` = `84111506` (default ⚠️ pendiente confirmar)
- `ClaveUnidad` = `ACT` (default ⚠️ pendiente confirmar)
- `Cantidad` = `1`
- `ValorUnitario` = Subtotal capturado
- `Descripcion` = `ConceptoManual` capturado por el usuario

### B5 — Previsualización PDF (Paso 3)

**Descripción:** Antes de timbrar, Finanzas genera el PDF representativo de la NC en memoria para validación visual del usuario (Criterio I3).

**Flujo:**
1. Finanzas consolida los datos de la NC en `CreditNoteMexicoPdfModel` (sin TimbreFiscalDigital, sin UUID, sin QR).
2. Invoca `CreditNoteMexicoPdfMappingService.MapearPreviewAsync()`.
3. Resuelve `TemplateKey` dinámicamente según empresa emisora: `{Prefijo}_MEX_NC` (GOL_MEX_NC, MUN_MEX_NC, PRO_MEX_NC, PQF_MEX_NC).
4. Llama a DocumentBuilder con el `TemplateKey` y el modelo.
5. Retorna PDF en memoria al frontend.
6. Sin escrituras en BD.

**Modelo `CreditNoteMexicoPdfModel` — secciones:**

| Sección                    | Campos clave                                                                |
| -------------------------- | --------------------------------------------------------------------------- |
| Cabecera emisor            | RFC, Razón Social, Régimen Fiscal, CP emisión, Serie, Folio (tentativo)    |
| Cabecera receptor          | RFC, Razón Social, Régimen Fiscal, CP receptor, UsoCFDI G02                |
| CFDI relacionado           | Folio + UUID de la factura origen, TipoRelacion 01                         |
| Datos de la NC             | Modalidad, Motivo, FormaPago, Moneda, TipoCambio (si aplica)               |
| Partidas (por partidas)    | Código, Descripción, Cant. NC, Precio Unitario, Importe, IVA, Total línea  |
| Concepto (manual)          | ClaveProdServ, ClaveUnidad, Descripcion (materialidad fiscal), Monto        |
| Totales                    | Subtotal, IVA, Total NC con moneda                                          |
| Cancelación (si aplica)    | Indicador de cancelación de factura origen + Motivo SAT                    |
| Nota legal                 | "Este documento es una representación impresa de un CFDI" + Art. 30 CFF   |

### B6 — Timbrado de la NC (Paso 3 — acción Timbrar)

**Descripción:** Al confirmar el timbrado, Finanzas orquesta la secuencia: timbrar NC + (condicional) cancelar factura origen + persistir + enviar correo.

**Flujo:**

**Paso 1 — Enviar a Timbrado:**
Finanzas construye el `CreditNoteMexicoRequest` (sección B4) y llama al API de Timbrado:
```
POST /api/v1/cfdi
Body: CreditNoteMexicoRequest
```
Muestra feedback visual de progreso al usuario (Criterio J1).

**Paso 2 — Timbrado exitoso:**
Timbrado retorna `CreditNoteMexicoResponse` con:
- `IdCFDIGenerada` recién insertado
- `UUID` SAT
- `XML` timbrado completo con `TimbreFiscalDigital`

**Paso 3 — Cancelación condicional de factura origen:**
Si `fccNotaCredito.CancelarFacturaOrigen = 1`:
- Finanzas solicita a Timbrado la cancelación de la factura origen ante el SAT:
  ```
  POST /api/v1/cfdi/{id}/cancel
  Body: { ClaveMotivo: <ClaveMotivosCancelacion> }
  ```
- Timbrado invoca PAC TurboPac con motivo de cancelación.
- Timbrado actualiza `CFDICancelacion` con `ClaveMotivo`, `Estatus='CANCELADA'`.

**Paso 4 — Persistencia post-timbrado:**
Finanzas ejecuta `PersistMexicoCreditNotePdfService.PersistirAsync()`:
1. Genera PDF final con sello digital, UUID y QR (llamada a DocumentBuilder con modelo completo).
2. Resuelve bucket MinIO:
   ```sql
   SELECT BucketNombre FROM RegionConfiguracionMinioBucket rcmb
   INNER JOIN Region r ON rcmb.IdRegion = r.IdRegion
   WHERE rcmb.BucketClave = 'notas_credito' AND r.Clave = 'MEX' AND rcmb.Activo = 1
   ```
3. Sube PDF a MinIO → path: `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID_NC}.pdf`
4. Sube XML a MinIO → path: `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID_NC}.xml`
5. INSERT `Archivo` (PDF) → obtiene `IdArchivoPdf`
6. INSERT `Archivo` (XML) → obtiene `IdArchivoXml`
7. INSERT `CFDIGeneradaRelacionado` con UUID factura origen + ClaveTipoRelacion='01'
8. UPDATE `fccNotaCredito`:
   - `IdCFDIGenerada` = IdCFDIGenerada del timbre
   - `IdCFDI` = IdCFDI del CFDI timbrado
   - `IdArchivoXml`, `IdArchivoPdf`
   - `Estado` = 'VIGENTE'
   - `Folio`, `Serie` = valores asignados por el foliador
9. Si modalidad POR_PARTIDAS: INSERT `fccNotaCreditoPartida` para cada partida con `CantidadNC > 0`.
10. Si modalidad MANUAL: UPDATE `fccNotaCredito.ConceptoManual`.

**Paso 5 — Correo automático al cliente (Criterio J3):**
Finanzas envía correo al timbrar exitosamente, a través del Aplicativo Nuevo **ProquifaDotNet.EnvioCorreo** (Reglas al diseñar, regla 7 — no se integra Brevo directamente desde Finanzas):
- **Para:** contacto del cliente vinculado a la factura origen (editable).
- **CC:** ESAC asignado al cliente + analista de Cuentas por Cobrar (editable).
- **Adjuntos:** PDF + XML de la NC.
- **Asunto:** "Nota de Crédito {Folio NC} — Factura {Folio Factura Origen}" ⚠️ plantilla final PMO #31.
- INSERT `CorreoEnviado` + INSERT `ArchivoCorreoEnviado` (PDF + XML).

**Paso 6 — Bitácora:**
Finanzas registra el guardado de la Nota de Crédito en **ProquifaDotNet.BitacoraCambios** (Aplicativo Nuevo — Reglas al diseñar, regla 8).

**Paso 7 — Navegación al Paso 4:**
Finanzas retorna al frontend la respuesta con todos los datos de la NC timbrada para renderizar el Paso 4 NC Emitida (Criterio J4).

### B7 — Consulta de NCs (Pantalla Principal + Drill-down)

**Descripción:** Endpoints de consulta del módulo NC para la pantalla principal (agrupada por cliente) y el drill-down por cliente.

**Criterio A5 — Visibilidad filtrada por cartera del Cobrador (OBS-004):**
La pantalla principal solo muestra NCs de los clientes asignados al Cobrador autenticado. Se aplica el mismo patrón JOIN sobre `vUsuarioCartera` que usan los módulos Validar Cobro, Factura por Adelantado y Buzón de Pagos (ver R16A-RE-FU-002 Módulos Consumidores).

**Pantalla principal — datos requeridos (Criterio A1 + Criterio A5):**
```sql
-- Vista agrupada por cliente, filtrada por cartera del Cobrador
SELECT
    c.IdCliente,
    c.NombreCliente,
    COUNT(nc.IdFCCNotaCredito)                                    AS TotalNC,
    SUM(CASE WHEN nc.Estado = 'VIGENTE' THEN 1 ELSE 0 END)        AS Vigentes,
    SUM(CASE WHEN nc.Aplicada = 0 AND nc.Estado = 'VIGENTE' THEN 1 ELSE 0 END) AS PorAplicar,
    SUM(CASE WHEN nc.Estado = 'VIGENTE' THEN nc.Monto ELSE 0 END) AS MontoTotal,
    cg.Moneda
FROM dbo.fccNotaCredito nc
INNER JOIN dbo.Cliente c ON nc.IdCliente = c.IdCliente
INNER JOIN dbo.CFDIGenerada cg ON nc.IdCFDIGenerada = cg.IdCFDIGenerada
-- Criterio A5 (OBS-004): filtrar por clientes en cartera del Cobrador autenticado
INNER JOIN dbo.vUsuarioCartera vc ON vc.IdCliente = nc.IdCliente
                                  AND vc.IdUsuario = @IdUsuarioCobrador
                                  AND vc.Activo = 1
-- Filtros adicionales: Moneda, Fecha
GROUP BY c.IdCliente, c.NombreCliente, cg.Moneda
```

> **Nota (Criterio A5):** El parámetro `@IdUsuarioCobrador` se obtiene del usuario autenticado en la sesión. Si el usuario no es Gestor de Cobranza, este filtro debe omitirse o definirse la política de visibilidad correspondiente (pendiente confirmar con funcional).

**Drill-down por cliente — columnas (Criterio B2):**
Fecha, Cobrador, Folio NC (acción → PDF), XML (descarga), Emisor, Monto+Moneda, Factura Asociada, Pedido Interno Asociado, Estado, Factura destino, Pedido destino.

### B8 — Reenvío de correo

**Descripción:** Permite reenviar el correo de la NC desde la vista de detalle (Paso 4 / Criterios L1–L4).

**Flujo:**
1. Finanzas pre-popula destinatarios desde `DatosFacturacionCliente` (Para) + ESAC/CxC (CC).
2. Usuario puede editar Para, CC y mensaje libre.
3. Finanzas arma el correo con PDF + XML como adjuntos (recuperados de `Archivo` vía MinIO).
4. INSERT `CorreoEnviado` + INSERT `ArchivoCorreoEnviado`.

---

## Parte C — ProquifaDotNet.Timbrado

### C1 — Endpoint: Timbrar NC México

**Descripción:** Timbrado recibe el `CreditNoteMexicoRequest` de Finanzas, construye el XML CFDI E, lo envía al PAC TurboPac, guarda el XML y retorna el CFDI timbrado.

**Flujo:**
1. Timbrado recibe `CreditNoteMexicoRequest`.
2. Construye XML CFDI 4.0 TipoDocumento='E' con todos los nodos (Emisor, Receptor, CfdiRelacionados TipoRelacion='01', Conceptos, Impuestos, MetodoPago='PUE').
3. Obtiene folio con UPDLOCK atómico: `SELECT UltimoFolio+1 FROM EmpresaFolio WITH (UPDLOCK) WHERE Serie='P2' AND IdEmpresa=@IdEmpresa`.
4. Envía XML al PAC TurboPac.
5. Recibe XML timbrado con `TimbreFiscalDigital` (UUID SAT, sello SAT, certificado).
6. INSERT `CFDIGenerada` (TipoDocumento='E', MetodoDePago='PUE', UsoCFDI='G02', IdCatTipoCFDI=NOTA_CREDITO).
7. INSERT `CFDI` con UUID SAT.
8. Guarda XML timbrado en BD (o MinIO — mismo patrón que Factura).
9. UPDATE `EmpresaFolio.UltimoFolio`.
10. Retorna `CreditNoteMexicoResponse` a Finanzas.

**Manejo de errores (Regla 16 / Criterio J5):**
- Si PAC retorna error: no se persiste la NC, se retorna el error a Finanzas con detalle del PAC.
- Finanzas muestra mensaje al usuario con posibilidad de reintentar.

### C2 — Endpoint: Cancelar CFDI (cancelación condicional de factura origen)

**Descripción:** Timbrado cancela la factura origen ante el SAT cuando Finanzas lo solicita (Regla 8).

**Flujo:**
1. Timbrado recibe `{ IdCFDI, ClaveMotivo }`.
2. Invoca PAC TurboPac con UUID de la factura origen y motivo de cancelación.
3. INSERT/UPDATE `CFDICancelacion` con `ClaveMotivo`, `Estatus='CANCELADA'`.
4. Retorna resultado a Finanzas.

> Este endpoint ya puede existir de requisitos anteriores (Factura México). Si existe, RE-032 lo reutiliza. Si no, se crea como nuevo endpoint.

---

## Parte D — DocumentBuilder: Templates PDF NC México

Se crean 4 archivos HTML de template (H/B/F × 4 empresas) para el PDF representativo de la NC México.

| TemplateKey   | Archivos                                         | Empresa               |
| ------------- | ------------------------------------------------ | --------------------- |
| `GOL_MEX_NC`  | `GOL_MEX_NC_H.html` / `_B.html` / `_F.html`    | Golocaer              |
| `MUN_MEX_NC`  | `MUN_MEX_NC_H.html` / `_B.html` / `_F.html`    | Mungen                |
| `PRO_MEX_NC`  | `PRO_MEX_NC_H.html` / `_B.html` / `_F.html`    | Proquifa              |
| `PQF_MEX_NC`  | `PQF_MEX_NC_H.html` / `_B.html` / `_F.html`    | Proveedora Q.F.       |

El diseño del PDF de la NC se documenta en un requisito independiente (análogo a R16A-RE-FU-021 para Factura México). Esta sección cubre únicamente el registro de los `TemplateKey` en BD y la integración funcional.

**Preview vs Timbrado:**
- **Preview (Paso 3):** PDF sin sello, sin UUID, sin QR — generado por `CreditNoteMexicoPdfMappingService.MapearPreviewAsync()`.
- **Post-timbrado:** PDF completo con `TimbreFiscalDigital`, UUID, QR — generado por `PersistMexicoCreditNotePdfService.PersistirAsync()`.

---

## Parte E — MinIO: Almacenamiento PDF y XML

### E1 — Resolución del bucket

Finanzas resuelve el bucket NC para México consultando `RegionConfiguracionMinioBucket`:

```sql
SELECT rcmb.BucketNombre
FROM dbo.RegionConfiguracionMinioBucket rcmb
INNER JOIN dbo.Region r ON rcmb.IdRegion = r.IdRegion
WHERE rcmb.BucketClave = 'notas_credito'
  AND r.Clave           = 'MEX'
  AND rcmb.Activo       = 1
```

### E2 — Rutas de almacenamiento

| Documento | Ruta MinIO                                                       |
| --------- | ---------------------------------------------------------------- |
| PDF NC    | `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID_NC}.pdf`    |
| XML NC    | `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID_NC}.xml`    |

### E3 — Flujo completo de persistencia

```
PersistMexicoCreditNotePdfService.PersistirAsync()
  ├── DocumentBuilder.GenerarPdf(TemplateKey, CreditNoteMexicoPdfModel)   → PDF bytes
  ├── MinIO.PutObject(bucket, path_pdf, PDF bytes)                → OK
  ├── MinIO.PutObject(bucket, path_xml, XML bytes)                → OK
  ├── INSERT Archivo (PDF) → IdArchivoPdf
  ├── INSERT Archivo (XML) → IdArchivoXml
  └── UPDATE fccNotaCredito SET IdArchivoPdf, IdArchivoXml, Estado='VIGENTE'
```

---

## Parte F — ETL: Transferencia NC a PCconnect (Legacy) vía SSIS

### F1 — Descripción general

Las NCs timbradas en ProquifaDotNet deben transferirse al sistema legado **PCconnect**
mediante un **paquete SSIS nuevo** (no reutiliza los paquetes de Facturas o Pedidos ya que el
tipo de documento y las tablas destino difieren).

La transferencia se dispara cuando la NC queda en estado `VIGENTE` en `fccNotaCredito`
(es decir, después del timbrado exitoso y la persistencia del XML/PDF).

### F2 — Alcance del paquete SSIS

| Evento                                       | Datos transferidos a PCconnect                              |
| -------------------------------------------- | ----------------------------------------------------------- |
| NC timbrada (Estado = VIGENTE)               | Cabecera NC: folio, UUID, RFC emisor/receptor, fecha, montos, moneda, TC, estado |
| NC modalidad por partidas                    | Partidas: código producto, descripción, cantidad, precio unitario, importe, IVA |
| Factura origen cancelada simultáneamente     | Actualización de estado de factura en PCconnect             |
| NC aplicada a cobro (Validar Cobro)          | Aplicación de NC a pedido (`fccNotaCreditoPedido`)          |

### F3 — Datos fuente en ProquifaDotNet

```sql
-- Consulta base para el paquete SSIS
SELECT
    nc.IdFCCNotaCredito,
    nc.Serie,
    nc.Folio,
    cfdi.UUID                        AS UUID_NC,
    cg.RFCEmisor,
    cg.RazonSocialEmisor,
    cg.RFCReceptor,
    cg.RazonSocialReceptor,
    cg.FechaEmision,
    cg.Subtotal,
    cg.Total,
    cg.Moneda,
    cg.TipoDeCambio,
    nc.Estado,
    nc.Modalidad,
    nc.Motivo,
    rel.UUID                         AS UUID_FacturaOrigen,
    nc.CancelarFacturaOrigen,
    nc.ClaveMotivosCancelacion,
    a_pdf.FileKey                    AS RutaPdf,
    a_xml.FileKey                    AS RutaXml
FROM dbo.fccNotaCredito nc
INNER JOIN dbo.CFDIGenerada cg       ON nc.IdCFDIGenerada = cg.IdCFDIGenerada
INNER JOIN dbo.CFDI cfdi             ON cg.IdCFDI = cfdi.IdCFDI
LEFT  JOIN dbo.CFDIGeneradaRelacionado rel ON cg.IdCFDIGenerada = rel.IdCFDIGenerada
LEFT  JOIN dbo.Archivo a_pdf         ON nc.IdArchivoPdf = a_pdf.IdArchivo
LEFT  JOIN dbo.Archivo a_xml         ON nc.IdArchivoXml = a_xml.IdArchivo
WHERE nc.Estado = 'VIGENTE'
  -- Filtro adicional: solo registros no transferidos aún (requiere columna de control)
```

### F4 — Consideraciones del paquete SSIS

- **Idempotencia:** Usar UUID SAT de la NC como clave natural en PCconnect para evitar duplicados en reenvíos.
- **Control de transferencia:** Agregar columna `TransferidoPCConnect bit DEFAULT 0` en `fccNotaCredito` o usar tabla de control SSIS, para marcar los registros ya transferidos.
- **Cancelación de factura:** Si `CancelarFacturaOrigen = 1`, el paquete debe actualizar el estado de la factura origen en PCconnect además de insertar la NC.
- **Mapeo completo:** Pendiente hasta recibir estructura de tablas PCconnect (estructura de PCconnect no disponible al redactar este documento).
- **Trigger:** El paquete puede ejecutarse en polling periódico o disparado por evento post-timbrado vía RabbitMQ (a definir en diseño técnico).

> ⚠️ **Pendiente:** Una vez disponible la estructura de PCconnect, documentar el mapeo
> columna a columna para cabecera y partidas de la NC.

---

## Diferencias clave vs RE-028/RE-029/RE-030

| Aspecto                         | RE-028 / RE-029 / RE-030                  | RE-032                                         |
| ------------------------------- | ----------------------------------------- | ---------------------------------------------- |
| Disparador                      | Cobro del cliente (Validar Cobro)         | Acción deliberada de Tesorería                 |
| Módulo                          | Validar Cobro (subproceso)                | Módulo independiente NC                        |
| CFDI tipo                       | I (Ingreso) / P (Pago)                   | E (Egreso)                                     |
| Tabla principal                 | `fccDocumentoFiscalCobro` (nueva RE-028)  | `fccNotaCredito` (existente, extendida)        |
| Foliador serie                  | P (Complemento de Pago)                  | P2 (Nota de Crédito)                           |
| ETL Legacy                      | No aplica (RE-028/029/030)               | Sí — nuevo paquete SSIS a PCconnect            |
| Cancelación factura origen      | No aplica                                | Condicional (totalidad + mismo mes calendario) |
| Modalidades de captura          | N/A                                      | Por partidas / Manual                          |
| Clientes aplica                 | Prepago y crédito                        | Solo prepago en R16                            |

---

## Pendientes

| ID | Pendiente                                                                                                         | Sección afectada              |
| -- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| P1 | Verificar `catUsoCFDI` G02 en BD real (query SSMS)                                                               | A3 / B4                       |
| P2 | Verificar bucket MinIO 'notas_credito' en `RegionConfiguracionMinioBucket` (query SSMS)                          | E1                            |
| P3 | Validar Serie "P2" y formato de folio con PMO (Regla 9)                                                          | A5 / C1                       |
| P4 | Confirmar FormaPago en modalidad manual — '99' fiscalmente incorrecto para NC PUE (Regla 7)                      | B3 / B4                       |
| P5 | Confirmar ClaveProdServ y ClaveUnidad default para modalidad manual (candidato: 84111506 / ACT — Criterio F5)    | B3 / B4                       |
| P6 | Recibir estructura de tablas PCconnect para completar mapeo ETL SSIS                                             | F3 / F4                       |
| P7 | Confirmar si endpoint Cancelar CFDI ya existe en Timbrado (de RE-021/028) o debe crearse nuevo                  | C2                            |
| P8 | Confirmar plantilla final del asunto/cuerpo del correo NC (PMO #31)                                              | B6 paso 5                     |
| P9 | Definir si control de transferencia SSIS usa columna en `fccNotaCredito` o tabla SSIS dedicada                   | F4                            |

---

## Brechas

| ID | Brecha                                                                                                                              | Impacto                              |
| -- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| B1 | `fccNotaCredito.IdTPProformaPedido` NOT NULL — NCs R16 no provienen de pedido; confirmar si acepta NULL o requiere placeholder       | ALTER TABLE, integridad referencial  |
| B2 | `fccNotaCreditoPartida.NumeroDePiezas` es `int` — productos con cantidad fraccionaria requieren cambio a `decimal(18,6)`           | ALTER TABLE fccNotaCreditoPartida    |
| B3 | Estructura PCconnect desconocida — paquete SSIS no puede diseñarse completamente hasta tener el esquema destino                    | ETL SSIS                             |
| B4 | `catMotivoCancelacionSAT` no existe como tabla catálogo — `CFDICancelacion.ClaveMotivo` es varchar libre; el front debe traer el catálogo SAT hardcodeado o desde config | Validación UI                       |
| B5 | Políticas de autorización por monto (PMO #54) fuera de scope en R16 — si se requieren en el futuro, impactan el wizard Paso 2     | Wizard Paso 2                        |
