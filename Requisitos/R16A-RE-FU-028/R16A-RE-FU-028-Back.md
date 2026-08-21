# Impacto en Back — R16A-RE-FU-028
**Requisito:** Validar Cobro: Paso 3 México — Facturación y Envío
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder
**Módulo:** Validar Cobro — Wizard Paso 3 (México)
**Impacto:** Scripts BD ProquifaDotNet (3 catálogos nuevos + 2 tablas nuevas + 2 ALTER + 1 vista) + Endpoints Finanzas: inicialización Paso 3, lógica condicional tipo CFDI por línea, previsualización PDF, timbrado (con cascada PPD Factura + Complemento), envío vía ProquifaDotNet.EnvioCorreo + acciones post-envío automáticas (FEE, transferencia Legacy, Confirmación de Pedido). Comunicación Finanzas → Timbrado vía API. **Solo México — operaciones unitarias por línea, sin acciones masivas.**

---

## Resumen

Este requisito implementa la **tercera y última pantalla del wizard de Validar Cobro (Paso 3 — Facturación y Envío) para Región México** en ProquifaDotNet.Finanzas. Es el paso donde se materializan fiscalmente las asociaciones cerradas en el Paso 2: por cada línea derivada de la asociación, el sistema determina el tipo de CFDI a emitir, previsualiza, timbra y envía.

La lógica condicional del tipo de CFDI por línea es el núcleo del requisito:
- Proforma sin controlados → **Factura** (CFDI Ingreso PUE o PPD)
- Proforma con controlados → **Factura Anticipo** (CFDI Ingreso). ~~rel. 07 SAT~~ **INCORRECTO — DUDA-088:** la Factura Anticipo NO usa relación 07. La relación 07 es de la Factura Final (fuera de alcance, se genera en Legacy). Ver `Guia_Tecnica_Facturas_Ingreso_MX.md` sección 6.
- Factura por Adelantado existente + cobro → **Complemento de Pago** (CFDI Pagos 2.0)

Al confirmar el envío de cada línea, el sistema dispara automáticamente tres acciones: establece la Fecha Estimada de Entrega (FEE), transfiere el pedido y documentos a Legacy, y genera la Confirmación de Pedido adjunta al correo. La operación es individual por línea — sin acciones masivas (confirmado con cliente, DUDA-050).

### Distribución de responsabilidades

| Capa              | Aplicativo                | Responsabilidad                                                                                                     |
| ----------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| BD — Catálogos    | ProquifaDotNet            | `catTipoDocumentoFiscal`, `catDocumentoFiscalCobroEstado`, `catTipoCFDI`                                            |
| BD — Tablas       | ProquifaDotNet            | `fccDocumentoFiscalCobro` (estado Paso 3), `fccConfirmacionPedido`                                                  |
| BD — Alteraciones | ProquifaDotNet            | `CFDIGenerada` (IdCatTipoCFDI + IdCFDIRelacionado), `tpPedido` (FechaEstimadaEntrega)                               |
| BD — Vista        | ProquifaDotNet            | `vfccDocumentoFiscalCobro` (consolidación estado Paso 3 por cliente)                                                |
| Lógica Paso 3     | ProquifaDotNet.Finanzas   | Inicialización líneas, lógica condicional tipo CFDI, auto-guardado, estados                                         |
| Timbrado          | ProquifaDotNet.Timbrado   | Timbrado Factura PUE/PPD, Factura Anticipo, Complemento cascada; inserción CFDIGenerada                             |
| Generación PDF    | DocumentBuilder           | PDF Factura México (RE-FU-021), PDF Complemento de Pago, PDF Confirmación de Pedido                                 |
| Envío             | ProquifaDotNet.Finanzas   | Modal envío, integración con ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo — correo con PDF + XML adjuntos, regla 7) |
| Post-envío        | ProquifaDotNet.Finanzas   | FEE en `tpPedido`, transferencia Legacy, generación `fccConfirmacionPedido` en MinIO                                |
| Comunicación      | Finanzas → Timbrado       | Llamadas entre APIs para timbrado de cada CFDI de la línea                                                          |
| Comunicación      | Finanzas → ProquifaDotNet | Llamadas entre APIs para leer datos y escribir resultados del Paso 3                                                |

### Infraestructura reutilizada

| Componente                                          | Origen        | Reutilización                                                                                                                                          |
| --------------------------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `CFDIGenerada`                                      | RE-FU-019     | Registro central de CFDIs timbrados; se extiende con `IdCatTipoCFDI` e `IdCFDIRelacionado`                                                             |
| `EmpresaFolio` (ProquifaDotNet — Finanzas)          | RE-FU-019     | Foliador por empresa; consumo atómico con UPDLOCK al timbrar                                                                                           |
| `fccPagoFacturaPedido`                              | RE-FU-026     | FK desde `fccDocumentoFiscalCobro` (origen proforma)                                                                                                   |
| `fccPagoFacturaAdelanto`                            | RE-FU-026     | FK desde `fccDocumentoFiscalCobro` (origen FAA)                                                                                                        |
| `fccNotaCredito.IdCFDI`                             | RE-FU-026     | UUID de la NC para incluir en nodo `CFDIRelacionados` al timbrar                                                                                       |
| `tpProformaPedido.HayControlados`                   | RE-FU-013/014 | Flag que determina `FACTURA` vs `FACTURA_ANTICIPO` en la lógica condicional                                                                            |
| `fccFactura.IdCFDIGenerada` (RE-FU-015, antes `tpProformaAdelanto.IdCFDIGenerada`) | RE-FU-015     | UUID de la FAA para `CFDIRelacionados` del Complemento de Pago                                                                                         |
| `catFacturaEstado` + `fccFactura.IdCatFacturaEstado` | RE-FU-015 v2.1 | Estado del ciclo de vida de la factura FAA — al llegar al Paso 3 la FAA ya está en PAGADA/PAGADA_PARCIAL (transición ejecutada en el Paso 2, RE-FU-026); el Paso 3 no la modifica. Aplica solo a facturas con registro en `fccFactura` (origen FAA); las facturas emitidas desde proforma no tienen registro en `fccFactura` y su ciclo se rastrea por `EstadoLinea` + `CFDIGenerada` |
| `DatosFacturacionCliente`                           | Preexistente (RE-FU-004 cancelado) | RFC, Razón Social, Régimen Fiscal Receptor del CFDI 4.0                                                                                                |
| `Empresa`                                           | Existente     | RFC Emisor, Régimen Fiscal Emisor, Prefijo por empresa PROQUIFA México                                                                                 |
| Patrón timbrado PAC TurboPac                        | RE-FU-019     | Mismo flujo de timbrado y manejo de errores del PAC                                                                                                    |
| `ApiCallerStamping` (HttpClient + Polly)            | RE-FU-019     | Cliente HTTP con retry policy hacia Timbrado — ya implementado; el Paso 3 usa `StampInvoiceAsync` (`POST /api/v1/stamp/invoice`) para Factura/Factura Anticipo y `StampPaymentComplementAsync` (`POST /api/v1/stamp/payment-complement`) para el CP en cascada                                                               |
| `InvoicePdfMappingService`                    | RE-FU-021     | Consolida datos CFDI 4.0 en `InvoicePdfModel`; `MapearPreviewAsync` para preview, `MapearAsync` para PDF definitivo                                    |
| `PersistInvoicePdfService`                  | RE-FU-021     | Genera PDF definitivo post-timbrado → lo sube a MinIO → INSERT `Archivo` → UPDATE `CFDI`. GAP-10 de RE-021 anticipó esta integración con Validar Cobro |
| Templates DocumentBuilder `GOL/MUN/PRO/PQF_MEX_FAC` | RE-FU-021     | Plantillas de Factura CFDI 4.0 por empresa emisora — ya implementadas, se usan en Paso 3 sin cambios                                                   |

---

## Parte A — Base de Datos (ProquifaDotNet)

### A1 — CREATE TABLE catTipoDocumentoFiscal

Catálogo de tipos de documento fiscal generables en el Paso 3. Determina el tipo de CFDI a emitir por línea según el origen de la asociación del Paso 2.

| Clave              | Descripción                                                          |
| ------------------ | -------------------------------------------------------------------- |
| `FACTURA`          | CFDI Ingreso PUE o PPD (proforma sin productos controlados)          |
| `FACTURA_ANTICIPO` | CFDI Ingreso rel. 07 SAT (proforma con productos controlados)        |
| `COMPLEMENTO_PAGO` | CFDI Pagos 2.0 (Factura por Adelantado existente con cobro asociado) |

> Ver script completo en `R16A-RE-FU-028_BD.md` — Catálogo Nuevo: `catTipoDocumentoFiscal`.

### A2 — CREATE TABLE catDocumentoFiscalCobroEstado

Catálogo de estados del ciclo de vida de cada línea del Paso 3.

| Clave       | Descripción                                                  |
| ----------- | ------------------------------------------------------------ |
| `PENDIENTE` | Estado inicial; línea creada, aún no timbrada ni enviada     |
| `GENERADO`  | CFDIs timbrados exitosamente; pendiente de envío al cliente  |
| `ENVIADO`   | Documentos enviados al cliente; línea cerrada operativamente |

> Ver script completo en `R16A-RE-FU-028_BD.md` — Catálogo Nuevo: `catDocumentoFiscalCobroEstado`.

### A3 — CREATE TABLE catTipoCFDI

Catálogo de tipos de CFDI timbrados. Discrimina el comprobante fiscal almacenado en `CFDIGenerada`.

| Clave              | Descripción                               |
| ------------------ | ----------------------------------------- |
| `FACTURA_PPD`      | Factura CFDI Ingreso con método PPD       |
| `FACTURA_PUE`      | Factura CFDI Ingreso con método PUE       |
| `FACTURA_ANTICIPO` | Factura Anticipo CFDI Ingreso rel. 07 SAT |
| `COMPLEMENTO_PAGO` | CFDI Pagos 2.0                            |

> Ver script completo en `R16A-RE-FU-028_BD.md` — Catálogo Nuevo: `catTipoCFDI`.

### A4 — CREATE TABLE fccDocumentoFiscalCobro

Tabla central del Paso 3. Una fila por cada documento fiscal a generar, derivada de la asociación cerrada en el Paso 2. Apunta directamente a `fccPagoFacturaPedido` o `fccPagoFacturaAdelanto` mediante FK exclusiva (CHECK CONSTRAINT).

**Ciclo de vida de una línea:**

```
INSERT (EstadoLinea = PENDIENTE)
  → UPDATE (UsoCFDI, MetodoPago — auto-guardado)
    → UPDATE (EstadoLinea = GENERADO, IdCFDIGeneradaFactura, [IdCFDIGeneradaComplemento])
      → UPDATE (EstadoLinea = ENVIADO, FechaEnvio)
```

> Ver script completo en `R16A-RE-FU-028_BD.md` — Tabla Nueva: `fccDocumentoFiscalCobro`.

### A5 — CREATE TABLE fccConfirmacionPedido

Almacena las Confirmaciones de Pedido generadas al enviar cada línea para pedidos Prepago México. El PDF se genera vía DocumentBuilder y se almacena en MinIO; la ruta se guarda en esta tabla.

> Ver script completo en `R16A-RE-FU-028_BD.md` — Tabla Nueva: `fccConfirmacionPedido`.

### A6 — ALTER TABLE CFDIGenerada

Se extiende la tabla central de CFDIs (creada en RE-FU-019) con dos columnas:

```sql
ALTER TABLE dbo.CFDIGenerada ADD IdCatTipoCFDI uniqueidentifier NULL;
    -- FK a catTipoCFDI; NULL en registros previos (normalizar con UPDATE posterior)

ALTER TABLE dbo.CFDIGenerada ADD IdCFDIRelacionado uniqueidentifier NULL;
    -- Auto-referencia blanda: UUID de la Factura PPD a la que este Complemento complementa
    -- NULL para todos los demás tipos
```

> Se incluye script de normalización para los registros de FAA existentes (RE-FU-019).
> Ver script completo en `R16A-RE-FU-028_BD.md` — ALTER TABLE CFDIGenerada.

### A7 — ALTER TABLE tpPedido (condicional)

```sql
-- Ejecutar solo si la columna no existe en RYNL010
ALTER TABLE dbo.tpPedido ADD FechaEstimadaEntrega datetime NULL;
```

Se establece automáticamente al confirmar el envío exitoso de cada línea del Paso 3 (solo México).

> ⚠️ Verificar existencia del campo en `sys.columns` antes de ejecutar — puede existir en la tabla pre-R16.

### A8 — CREATE VIEW vfccDocumentoFiscalCobro

Vista operativa que consolida el estado del Paso 3 por cliente. Navega desde `fccDocumentoFiscalCobro` hacia los registros de asociación del Paso 2, cobro, proforma, pedido, catálogos y CFDIs generados en una sola consulta. Usada por Finanzas para renderizar el estado actual al reingresar al Paso 3.

> Ver script completo en `R16A-RE-FU-028_BD.md` — CREATE VIEW vfccDocumentoFiscalCobro.

---

## Parte B — ProquifaDotNet.Finanzas: Servicios y Endpoints

### B1 — Inicialización del Paso 3 (creación de líneas)

**Descripción:** Al avanzar desde el Paso 2 con la asociación cerrada, Finanzas inicializa el Paso 3 creando una línea en `fccDocumentoFiscalCobro` por cada documento de la asociación.

**Lógica de determinación del tipo por línea:**

| Origen (Paso 2) | Condición adicional | `IdCatTipoDocumentoFiscal` resultante |
|-----------------|--------------------|------------------------------------|
| `fccPagoFacturaPedido` | `tpProformaPedido.HayControlados = 0` | `FACTURA` |
| `fccPagoFacturaPedido` | `tpProformaPedido.HayControlados = 1` | `FACTURA_ANTICIPO` |
| `fccPagoFacturaAdelanto` | — | `COMPLEMENTO_PAGO` |

**Datos leídos (Scaffold Finanzas):** `fccPagoFacturaPedido`, `fccPagoFacturaAdelanto`, `tpProformaPedido.HayControlados` (movida a Finanzas 07/07/2026), `DatosFacturacionCliente`, `Empresa`, `catTipoDocumentoFiscal`, `catDocumentoFiscalCobroEstado`

**Escritura:** `INSERT fccDocumentoFiscalCobro` (una fila por documento, estado inicial `PENDIENTE`, `IdCatUsoCFDI` = default del cliente)

Si al reingresar al Paso 3 ya existen filas en `fccDocumentoFiscalCobro` para el cliente, Finanzas las recupera directamente desde `vfccDocumentoFiscalCobro` sin reinicializar.

> **Cadena de origen para líneas `COMPLEMENTO_PAGO`:** provienen de FAAs enviadas como Prepago en RE-FU-019. El flujo `FacturaAdelantadoEnviarService` (RE-FU-019) genera un pendiente en Validar Cobro al enviar una FAA Prepago. Ese pendiente es el que el Gestor de Cobranza atiende en los Pasos 1 y 2 del wizard. Al llegar al Paso 3, la línea referencia `fccPagoFacturaAdelanto.IdCFDIGenerada` = UUID de la FAA original para armar el `CFDIRelacionados` del Complemento de Pago.

### B2 — Auto-guardado de Uso CFDI y Método de Pago

**Descripción:** Cada vez que el usuario modifica el Uso CFDI o el Método de Pago de una línea, Finanzas persiste el cambio de forma inmediata.

**Escritura:** `UPDATE fccDocumentoFiscalCobro SET IdCatUsoCFDI = @Id, IdCatMetodoDePagoCFDI = @Id WHERE IdFCCDocumentoFiscalCobro = @Id`

**Regla de negocio:** Para líneas con `IdCatTipoDocumentoFiscal` → `COMPLEMENTO_PAGO`, el campo `IdCatMetodoDePagoCFDI` no se persiste (PPD es fijo e implícito por tipo).

### B3 — Previsualización PDF por línea

**Descripción:** Al presionar "Previsualizar", Finanzas genera el PDF representativo del documento en memoria sin persistirlo en BD. El usuario valida visualmente antes de timbrar.

**Flujo:**
1. Finanzas lee `vfccDocumentoFiscalCobro` + `fccNotaCredito` (NCs a incluir en `CFDIRelacionados`).
2. Para líneas `FACTURA` y `FACTURA_ANTICIPO`: invoca `InvoicePdfMappingService.MapearPreviewAsync(idCFDIGenerada)` (RE-FU-021) — consolida datos fiscales en `InvoicePdfModel` sin `TimbreFiscalDigital`, resuelve `TemplateKey` dinámicamente (`GOL/MUN/PRO/PQF_MEX_FAC`) y genera PDF en memoria vía DocumentBuilder.
3. Para líneas `COMPLEMENTO_PAGO`: la previsualización del PDF del Complemento se implementa en **R16A-RE-FU-030** (Diseño y generación: Complemento de Pago México).
4. Retorna el PDF en memoria al frontend para mostrar en el modal de previsualización.
5. Sin escrituras en BD.

**DocumentBuilder — documentos por tipo:**

| Tipo | Plantilla DocumentBuilder | Estado |
|------|--------------------------|--------|
| `FACTURA` / `FACTURA_ANTICIPO` | `GOL/MUN/PRO/PQF_MEX_FAC` | Ya implementada en RE-FU-021 |
| `COMPLEMENTO_PAGO` | `GOL/MUN/PRO/PQF_MEX_COP` | **Nueva — definir en RE-FU-030** |

### B4 — Timbrado por línea (con cascada PPD)

**Descripción:** Al confirmar el timbrado, Finanzas invoca ProquifaDotNet.Timbrado vía API. El comportamiento varía según el tipo y el Método de Pago:

**Escenario A — FACTURA PUE (1 CFDI):**
1. Finanzas → Timbrado: solicita timbrado de Factura PUE con datos del documento y NCs en `CFDIRelacionados`.
2. Timbrado invoca PAC TurboPac, inserta en `CFDIGenerada` (`IdCatTipoCFDI` → `FACTURA_PUE`), actualiza `EmpresaFolio`.
3. Retorna UUID + Folio + XML timbrado a Finanzas.
4. Finanzas: invoca `PersistInvoicePdfService.PersistirAsync(IdCFDI, xmlTimbrado)` (RE-FU-021 GAP-10) — genera PDF definitivo con `TimbreFiscalDigital`, lo sube a MinIO, INSERT `Archivo`, UPDATE `CFDI.IdArchivoPdf`.
5. Finanzas: `UPDATE fccDocumentoFiscalCobro SET EstadoLinea = GENERADO, IdCFDIGeneradaFactura = @Id, FechaGeneracion`.
6. Finanzas: `UPDATE tpProformaPedido SET IdCFDIGenerada = @IdCFDIFactura`.

**Escenario B — FACTURA PPD + Complemento en cascada (2 CFDIs):**
1. Finanzas → Timbrado: solicita timbrado de Factura PPD.
2. Timbrado: INSERT `CFDIGenerada` (`IdCatTipoCFDI` → `FACTURA_PPD`), llama PAC, actualiza `EmpresaFolio`. Retorna UUID + XML timbrado Factura.
3. Finanzas: invoca `PersistInvoicePdfService.PersistirAsync(IdCFDIFactura, xmlFactura)` (RE-FU-021 GAP-10).
4. Finanzas → Timbrado: solicita inmediatamente timbrado del Complemento de Pago, referenciando el UUID de la Factura PPD.
5. Timbrado: INSERT segundo `CFDIGenerada` (`IdCatTipoCFDI` → `COMPLEMENTO_PAGO`, `IdCFDIRelacionado` = `IdCFDIGenerada` de la Factura PPD). Retorna UUID + XML Complemento.
6. Finanzas: la persistencia del PDF del Complemento de Pago (plantilla `*_MEX_COP`, MinIO) se implementa en **R16A-RE-FU-030** — en este requisito se depende de ese servicio.
7. Finanzas: `UPDATE fccDocumentoFiscalCobro SET EstadoLinea = GENERADO, IdCFDIGeneradaFactura = @IdFactura, IdCFDIGeneradaComplemento = @IdComplemento, FechaGeneracion`.
8. Finanzas: `UPDATE tpProformaPedido SET IdCFDIGenerada = @IdCFDIFactura`.

> ⚠️ **Brecha B6 (resuelta — ownership):** Si el Complemento falla tras la Factura PPD timbrada exitosamente, la Factura PPD permanece vigente. **Timbrado no reintenta** (es un servicio síncrono de un solo intento, ver R16A-RE-FU-018); el reintento del Complemento se implementa en Finanzas, en este mismo flujo de generación (R16A-RE-FU-030): la línea permanece `PENDIENTE`, se incrementa un contador de reintentos y se notifica a soporte si se supera el límite — ver Brechas.

**Escenario C — FACTURA_ANTICIPO (1 CFDI):**
1. Igual que Escenario A con `IdCatTipoCFDI` → `FACTURA_ANTICIPO`. ~~y tipo de relación 07 SAT en el XML~~ **INCORRECTO — DUDA-088:** la Factura Anticipo NO lleva `CfdiRelacionados`/relación 07. Se genera conforme `Guia_Tecnica_Facturas_Ingreso_MX.md` (sección 6): `ClaveProdServ=84111506`, `ClaveUnidad=ACT`, sin nodo de relación.
2. Las NCs aplicadas se incluyen en el nodo `CFDIRelacionados` con tipo de relación SAT correspondiente.

> ⚠️ ~~**Brecha B1:** El uso del tipo de relación 07 SAT para la Factura Anticipo de controlados está pendiente de confirmar con asesor fiscal PROQUIFA.~~ **RESUELTO — DUDA-088 (2026-08-21):** confirmado que es incorrecto usar la relación 07 en la Factura Anticipo; la relación 07 se usa en la Factura Final (fuera de alcance). Brecha B1 cerrada.

**Escenario D — COMPLEMENTO_PAGO desde FAA existente (1 CFDI):**
1. Finanzas → Timbrado: solicita timbrado de Complemento de Pago referenciando el UUID de la FAA existente (`fccFactura.IdCFDIGenerada`, RE-FU-015 — antes `tpProformaAdelanto.IdCFDIGenerada`).
2. Timbrado: INSERT `CFDIGenerada` (`IdCatTipoCFDI` → `COMPLEMENTO_PAGO`, `IdCFDIRelacionado` = UUID FAA). Retorna UUID.
3. Finanzas: `UPDATE fccDocumentoFiscalCobro SET EstadoLinea = GENERADO, IdCFDIGeneradaFactura = @IdComplemento, FechaGeneracion`.

**Manejo de errores PAC:** Mismo comportamiento que RE-FU-019. Si el PAC responde con error, la línea permanece en `PENDIENTE`; Finanzas muestra el detalle del error al usuario para corrección y reintento.

### B5 — Inclusión de Notas de Crédito en CFDIRelacionados

**Descripción:** Cuando la línea tiene NCs aplicadas en el Paso 2, Finanzas las incluye en el nodo `CFDIRelacionados` del XML al timbrar.

**Datos leídos (vía API ProquifaDotNet):** `fccNotaCredito` WHERE `IdFCCPagoCliente` en los cobros de la línea AND `Aplicada = 1`

**Campos incluidos por NC en el XML:**
- `UUID` de la NC timbrada (`fccNotaCredito.IdCFDI`)
- Monto aplicado al documento
- Tipo de relación SAT: `01` (Nota de crédito de los documentos relacionados) o `07` (Aplicación de anticipo), según el caso fiscal

### B6 — Modal de Envío y despacho vía ProquifaDotNet.EnvioCorreo

**Descripción:** Al presionar "Enviar" en una línea en estado `GENERADO`, Finanzas abre el modal de envío y, al confirmar, despacha el correo a través del Aplicativo Nuevo **ProquifaDotNet.EnvioCorreo** (Reglas al diseñar, regla 7 — no se integra Brevo directamente desde Finanzas) con los adjuntos correspondientes.

**Destinatarios:**
- **Para:** Contacto del cliente del pedido (`tpPedido.IdContacto` → datos de contacto) — editable en el modal.
- **CC:** ESAC asignado al cliente/pedido — editable en el modal.

**Adjuntos por tipo de línea:**

| Tipo de línea | Adjuntos |
|---------------|----------|
| `FACTURA` PUE | PDF Factura + XML Factura + PDF Confirmación de Pedido |
| `FACTURA` PPD + Complemento cascada | PDF Factura + XML Factura + PDF Complemento + XML Complemento + PDF Confirmación de Pedido |
| `FACTURA_ANTICIPO` | PDF Factura Anticipo + XML Factura Anticipo + PDF Confirmación de Pedido |
| `COMPLEMENTO_PAGO` desde FAA | PDF Complemento + XML Complemento + PDF Confirmación de Pedido |

**Asunto del correo:** Generado automáticamente por plantilla según tipo de línea. Propuesta: `<Folio Pedido Interno> - <Folio Factura>`.

> ⚠️ **Brecha B2:** Plantilla de asunto y cuerpo del correo para Complementos de Pago pendiente de confirmar con PMO (PMO #31).

**Escritura al confirmar envío exitoso:**
- `UPDATE fccDocumentoFiscalCobro SET EstadoLinea = ENVIADO, FechaEnvio`
- `INSERT CorreoEnviado` + `INSERT ArchivoCorreoEnviado` (x N adjuntos)
- Registrar la validación de cobro en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo — Reglas al diseñar, regla 8)

### B7 — Acciones post-envío automáticas (solo México)

Al confirmar el envío exitoso de cada línea, Finanzas dispara automáticamente las siguientes acciones. Todas aplican solo a México:

**B7.1 — Fecha Estimada de Entrega (FEE)**

```
UPDATE tpPedido SET FechaEstimadaEntrega = @FEECalculada
WHERE IdTPPedido = @IdTPPedido
```

> ⚠️ **Brecha B4:** Las reglas de cálculo de la FEE (¿días hábiles?, ¿fecha fija?, ¿parámetro configurable?) están pendientes de confirmar con Operaciones PROQUIFA México.

**B7.2 — Generación de Confirmación de Pedido**

1. Finanzas solicita a DocumentBuilder la generación del PDF de Confirmación de Pedido.
2. DocumentBuilder genera el PDF y lo almacena en MinIO (bucket `confirmaciones`).
3. Finanzas: `INSERT fccConfirmacionPedido (FolioConfirmacion, RutaArchivoPDF)`.
4. El PDF se adjunta al correo del modal de envío.

> ⚠️ **Brecha B5:** Formato y foliador de la Confirmación de Pedido para Prepago pendiente de confirmar con PMO.

**B7.3 — Transferencia a Legacy**

Finanzas transfiere al sistema Legacy el pedido y los documentos generados: factura (y/o complemento), NCs aplicadas e información del cobro, para continuidad operativa (logística, surtido, entrega).

> ⚠️ **Brecha B3 — BLOQUEANTE:** El mecanismo de transferencia a Legacy desde el Paso 3 (canal: tabla ETL, cola RabbitMQ o API Legacy) no está definido. Pendiente definir con arquitectura antes de implementar.

### B8 — Cierre del wizard

Al detectar que todas las líneas del cliente están en estado `ENVIADO`, Finanzas cierra el wizard y retorna al listado principal de Validar Cobro. El cliente sale del listado de pendientes.

**Lectura:** `SELECT COUNT(*) FROM fccDocumentoFiscalCobro WHERE IdFCCPagoCliente = @Id AND EstadoLinea != 'ENVIADO'` (vía `vfccDocumentoFiscalCobro`)

---

## Parte C — ProquifaDotNet.Timbrado

### C1 — Endpoints de timbrado por tipo de CFDI

Timbrado recibe una solicitud por cada CFDI a generar desde Finanzas, en el endpoint del tipo de documento correspondiente (RE-018): `POST /api/v1/stamp/invoice` para Factura y Factura Anticipo, `POST /api/v1/stamp/payment-complement` para el Complemento de Pago en cascada. La solicitud incluye los datos del emisor/receptor, partidas, NCs en `CFDIRelacionados` (cuando aplica) y, para el Complemento de Pago en cascada, el UUID de la Factura PPD relacionada.

**Por cada timbrado exitoso:**
1. INSERT en `CFDIGenerada` con `IdCatTipoCFDI` resuelto desde `catTipoCFDI`.
2. Consumo atómico del folio en `EmpresaFolio` (UPDLOCK).
3. Llamada al PAC TurboPac.
4. UPDATE `CFDIGenerada` con UUID, Folio, FechaEmision retornados por el PAC.
5. INSERT en `StampingLog` (trazabilidad).
6. Retorna a Finanzas: UUID, Folio, Serie, FechaEmision.

**Para el Complemento en cascada PPD:** el `IdCFDIRelacionado` en `CFDIGenerada` se popula con el `IdCFDIGenerada` de la Factura PPD ya insertada en el mismo flujo.

---

## Parte E — ETL y Transferencias a Legacy

Esta sección documenta todas las transferencias de datos y documentos que ProquifaDotNet debe enviar al sistema Legacy como parte del flujo de Validar Cobro Paso 3 México. Cada ítem tiene una dependencia de requisito que define el modelo de datos y la lógica de generación del artefacto a transferir.

> ⚠️ El mecanismo de transferencia (canal: tabla ETL, cola RabbitMQ o API Legacy directa) está pendiente de definir con arquitectura — ver Brecha B3. Esta sección documenta **qué** se transfiere y sus dependencias; el **cómo** se define transversalmente.

### Resumen de transferencias

| #   | Transferencia                         | Tipo      | Dependencia    | Implementado en | Disparador                                          |
| --- | ------------------------------------- | --------- | -------------- | --------------- | --------------------------------------------------- |
| E1  | ETL Datos Buzón de Cobros             | Datos     | R16A-RE-FU-008 | **RE-FU-028**   | Al confirmar cobro en Paso 1                        |
| E2  | ETL Datos Proforma                    | Datos     | R16A-RE-FU-016 | **RE-FU-028**   | Al tramitar pedido Prepago                          |
| E3  | ETL Datos Factura                     | Datos     | R16A-RE-FU-019 | **RE-FU-028**   | Al enviar línea Paso 3 (Factura o Factura Anticipo) |
| E4  | ETL Datos Complemento de Pago         | Datos     | R16A-RE-FU-030 | **RE-FU-030**   | Al enviar línea Paso 3 (Complemento)                |
| E5  | ETL Datos Nota de Crédito             | Datos     | R16A-RE-FU-032 | **RE-FU-032**   | Al enviar línea Paso 3 (si hay NCs aplicadas)       |
| E6  | Transferencia PDF Factura             | Documento | R16A-RE-FU-021 | **RE-FU-028**   | Al enviar línea Paso 3 (Factura o Factura Anticipo) |
| E7  | Transferencia PDF Complemento de Pago | Documento | R16A-RE-FU-030 | **RE-FU-030**   | Al enviar línea Paso 3 (Complemento)                |
| E8  | Transferencia PDF Nota de Crédito     | Documento | R16A-RE-FU-034 | **RE-FU-034**   | Al enviar línea Paso 3 (si hay NCs aplicadas)       |

> **Alcance de RE-FU-028:** Solo se implementan E1, E2, E3 y E6. Los ítems E4 y E7 (Complemento de Pago) se implementan en R16A-RE-FU-030. Los ítems E5 y E8 (Nota de Crédito) se implementan en R16A-RE-FU-032 y R16A-RE-FU-034 respectivamente.

---

### E1 — ETL Datos Buzón de Cobros a Legacy

**Descripción:** Transferencia de los datos del cobro capturado en el Buzón de Cobros (Paso 1 del wizard) hacia Legacy. Incluye folio del cobro, monto, fecha, forma de pago, moneda, tipo de cambio y cuenta destino.

**Dependencia:** R16A-RE-FU-008 — Buzón de Cobros (define el modelo del cobro y sus datos).

**Datos origen (ProquifaDotNet):**

| Tabla | Campos |
|-------|--------|
| `fccPagoCliente` | Folio, Monto, FechaPago, FormaPago, Moneda, TipoDeCambio, CuentaDestino, IdCliente |
| `fccBuzonCobro` | Referencia al correo clasificado como cobro (adjunto de comprobante) |

**Datos destino (Legacy):** Registro de cobro recibido vinculado al cliente y al pedido para continuidad en logística y surtido.

**Disparador:** Al confirmar el cobro en Paso 1 del wizard o al avanzar exitosamente al Paso 3 (coordinación pendiente definir con la brecha B3).

---

### E2 — ETL Datos Proforma a Legacy

**Descripción:** Transferencia de los datos de la Proforma generada en el flujo de tramitación Prepago hacia Legacy. Permite que Legacy conozca el documento de cargo previo a la factura.

**Dependencia:** R16A-RE-FU-016 — Proforma México (define estructura y datos de la proforma).

**Datos origen (ProquifaDotNet):**

| Tabla | Campos |
|-------|--------|
| `tpProformaPedido` | Folio (PRF-MMDDAA-Consecutivo), MontoTotal, MontoPendiente, IdEmpresa, FechaTramitacion, HayControlados |
| `tpPedido` | FolioPedidoInterno, IdCliente, IdContacto |
| `Empresa` | Prefijo, RazonSocial, RFC |

**Datos destino (Legacy):** Registro de proforma/cargo previo asociado al pedido del cliente.

**Disparador:** Al tramitar el pedido Prepago (RE-FU-013/014/015) o al avanzar al Paso 3 de Validar Cobro.

---

### E3 — ETL Datos Factura a Legacy

**Descripción:** Transferencia de los datos fiscales de la Factura CFDI 4.0 timbrada (Factura normal o Factura Anticipo) hacia Legacy. Es la transferencia central del flujo de cobro para pedidos Prepago México.

**Dependencia:** R16A-RE-FU-019 — Factura por Adelantado Detalle México (define el modelo de `CFDIGenerada` y el flujo de timbrado).

**Datos origen (ProquifaDotNet):**

| Tabla | Campos |
|-------|--------|
| `CFDIGenerada` | UUID, Serie, Folio, FechaEmision, Total, Subtotal, Exportacion, IdCatTipoCFDI |
| `tpPedido` | FolioPedidoInterno, IdCliente |
| `tpProformaPedido` | Folio proforma origen |
| `DatosFacturacionCliente` | RFC, RazonSocial receptor |
| `Empresa` | RFC, RazonSocial emisor |

**Datos destino (Legacy):** Factura en tablas de documentos fiscales Legacy (Factura, Pedidos, Partidas, Cobro).

**Disparador:** Al enviar exitosamente la línea de tipo `FACTURA` o `FACTURA_ANTICIPO` en el Paso 3.

---

### E4 — ETL Datos Complemento de Pago a Legacy *(fuera de alcance RE-FU-028)*

> Este ítem **no se implementa en RE-FU-028**. El diseño, estructura fiscal y ETL del Complemento de Pago se definen y se implementan en **R16A-RE-FU-030 — Diseño y generación de Documentos: Complemento de Pago México**.

---

### E5 — ETL Datos Nota de Crédito a Legacy *(fuera de alcance RE-FU-028)*

> Este ítem **no se implementa en RE-FU-028**. El modelo de la NC, su estructura fiscal y su ETL a Legacy se definen en **R16A-RE-FU-032 — Notas de Crédito: México**.

---

### E6 — Transferencia PDF Factura a Legacy

**Descripción:** Envío del archivo PDF de la Factura CFDI 4.0 generada hacia Legacy para almacenamiento y consulta desde el sistema legado.

**Dependencia:** R16A-RE-FU-021 — Diseño y generación PDF Factura México (define el PDF generado por `PersistInvoicePdfService` almacenado en MinIO).

**Datos origen:**

| Componente | Detalle |
|------------|---------|
| MinIO bucket `facturas` | PDF Factura referenciado por `Archivo.IdArchivo` desde `CFDI.IdArchivoPdf` |
| `CFDIGenerada` | UUID, Folio, Serie — para identificar el PDF en Legacy |

**Datos destino (Legacy):** Archivo PDF en repositorio de documentos Legacy vinculado a la factura.

**Disparador:** Al enviar exitosamente la línea de tipo `FACTURA` o `FACTURA_ANTICIPO` en el Paso 3 (mismo evento que E3).

---

### E7 — Transferencia PDF Complemento de Pago a Legacy *(fuera de alcance RE-FU-028)*

> Este ítem **no se implementa en RE-FU-028**. El PDF del Complemento de Pago (plantilla `*_MEX_COP`) y su transferencia a Legacy se definen en **R16A-RE-FU-030 — Diseño y generación de Documentos: Complemento de Pago México**.

---

### E8 — Transferencia PDF Nota de Crédito a Legacy *(fuera de alcance RE-FU-028)*

> Este ítem **no se implementa en RE-FU-028**. El PDF de la Nota de Crédito y su transferencia a Legacy se definen en **R16A-RE-FU-034 — Diseño y generación de Documentos: Nota de Crédito México**.

---

### Consideraciones transversales ETL

| Aspecto | Detalle |
|---------|---------|
| **Mecanismo de transferencia** | Pendiente definir con arquitectura (tabla ETL, cola RabbitMQ o API Legacy directa) — Brecha B3 |
| **Atomicidad** | Las transferencias E3+E6 (Factura + PDF) deben ejecutarse en conjunto. La atomicidad de E4+E7 (Complemento) y E5+E8 (NCs) la define RE-FU-030 y RE-FU-032/034 respectivamente |
| **Orden de ejecución (alcance RE-028)** | E2 (Proforma) → E1 (Cobro) → E3+E6 (Factura). E4+E7 y E5+E8 son coordinados en sus requisitos propios |
| **Re-intentos** | Si la transferencia falla, la línea queda en estado `ENVIADO` en ProquifaDotNet pero pendiente en Legacy — requiere mecanismo de cola o log de transferencia |
| **Perú** | Ninguna de estas transferencias aplica a Perú — Golocaer S.A.C. no transfiere a Legacy |

---

## Parte D — DocumentBuilder

| Documento                          | TemplateKey               | Estado                                                        | Disparado por                                                         |
| ---------------------------------- | ------------------------- | ------------------------------------------------------------- | --------------------------------------------------------------------- |
| PDF Factura México                 | `GOL/MUN/PRO/PQF_MEX_FAC` | **Existente** (RE-FU-021)                                     | `InvoicePdfMappingService` + `PersistInvoicePdfService` |
| PDF Factura Anticipo México        | `GOL/MUN/PRO/PQF_MEX_FAC` | **Existente** (RE-FU-021, misma plantilla con datos anticipo) | `InvoicePdfMappingService` + `PersistInvoicePdfService` |
| PDF Complemento de Pago México     | `GOL/MUN/PRO/PQF_MEX_COP` | **Nueva — definir en RE-FU-030**                              | Ver R16A-RE-FU-030 — Diseño y generación: Complemento de Pago México  |
| PDF Confirmación de Pedido Prepago | `GOL/MUN/PRO/PQF_MEX_CDP` | **Nueva — definir en RE-FU-028**                              | Finanzas al enviar (post-envío, solo México)                          |

> Las plantillas `*_MEX_FAC` ya existen con toda su estructura (Header, Body, Footer) y registros `DocumentTemplate`. Solo se reutilizan.
> La plantilla `*_MEX_CDP` (Confirmación de Pedido) es nueva y se define en este requisito — una variante por empresa emisora (GOL, MUN, PRO, PQF).
> La plantilla `*_MEX_COP` (Complemento de Pago) se define en R16A-RE-FU-030.

---

## Brechas

> ⚠️ ~~**BRECHA BLOQUEANTE — Tipo de relación SAT para Factura Anticipo de controlados (B1)**
> Pendiente confirmar con asesor fiscal PROQUIFA si el tipo de relación correcto es `07` (Aplicación de Anticipo) u otro. Sin este dato el XML de la Factura Anticipo no puede estructurarse correctamente para el timbrado ante el PAC TurboPac.~~
> **RESUELTO — DUDA-088 (2026-08-21), Brecha B1 cerrada:** la Factura Anticipo NO usa relación 07 ni ningún `CfdiRelacionados`. La relación 07 corresponde a la Factura Final (documento fuera de alcance, se construye en Legacy). Ver `Guia_Tecnica_Facturas_Ingreso_MX.md` sección 6 para la estructura correcta de la Factura Anticipo.

> ⚠️ **BRECHA MEDIA — Plantilla de correo Complemento de Pago (B2)**
> El asunto y cuerpo del correo de envío para líneas de Complemento de Pago están pendientes de confirmar con PMO (PMO #31). Propuesta inicial de asunto: `<Folio Pedido Interno> - <Folio Factura>`. Sin esto el modal de envío no puede finalizarse para este tipo de línea.

> ⚠️ **BRECHA BLOQUEANTE — Mecanismo de transferencia a Legacy desde Paso 3 (B3)**
> El canal de transferencia (tabla ETL, cola RabbitMQ, llamada API Legacy directa) no está definido para las transferencias implementadas en RE-028: E1 (Buzón de Cobros), E2 (Proforma), E3 (Factura) y E6 (PDF Factura). Es prerequisito para implementar las acciones post-envío. Pendiente definir con arquitectura antes de implementar. Ver Parte E — Consideraciones transversales ETL. Las transferencias de Complemento de Pago (E4, E7) y Notas de Crédito (E5, E8) están sujetas a la misma brecha pero se resuelven en R16A-RE-FU-030 y R16A-RE-FU-032/034 respectivamente.

> ⚠️ **BRECHA BLOQUEANTE — Reglas de cálculo de la FEE y granularidad (B4)**
> `tpPartidaPedido.FechaEstimadaEntrega` ya existe por partida (calculada al tramitar con base en stock). La FEE del Paso 3 es conceptualmente distinta: es la fecha confirmada post-pago que aparece en la Confirmación de Pedido y se transfiere a Legacy. Pendiente confirmar con Operaciones PROQUIFA México: (a) si RE-028 debe actualizar `tpPartidaPedido` (por partida) o solo `tpPedido` (cabecera), o ambas; (b) si el pedido tiene partidas con FEEs distintas, qué valor se establece en la cabecera; (c) la regla de cálculo exacta (días hábiles, días calendario, fecha fija, parámetro por empresa). Sin resolver esto, el `ALTER TABLE tpPedido` y el UPDATE post-envío no pueden implementarse.

> ⚠️ **BRECHA MEDIA — Formato y foliador de la Confirmación de Pedido Prepago (B5)**
> El formato del folio de la Confirmación de Pedido para pedidos Prepago (extensión a R16 del concepto existente en ProquifaNet para Crédito) está pendiente de confirmar con PMO/Operaciones.

> ✅ **RESUELTA — Mecanismo de reintento ante fallo del Complemento en cascada PPD (B6)**
> Si la Factura PPD se timbra exitosamente pero el Complemento falla inmediatamente después, la línea queda en estado inconsistente (Factura vigente sin Complemento). El reintento es responsabilidad de Finanzas, no de Timbrado (Timbrado es síncrono, un solo intento por petición, ver R16A-RE-FU-018), y se implementa en este mismo flujo de generación (Validar Cobro Paso 3 / R16A-RE-FU-030): la línea permanece `PENDIENTE`, se incrementa un contador de reintentos y se notifica a soporte por correo si se supera el límite (mismo patrón documentado en `Diagramas/Diagrama Secuencia Encolamiento Finanzas y Timbrado Factura.md`).

> ⚠️ ~~**BRECHA MEDIA — Comportamiento si el contacto del pedido no está disponible al armar el modal de Envío (B7)**
> Si el pedido no tiene contacto asignado o hay múltiples contactos, el comportamiento del modal (¿bloquea el envío?, ¿permite captura manual?) está pendiente de confirmar con negocio.~~
> **RESUELTO — DUDA-089 (2026-08-21), Brecha B7 cerrada:** se usa el mismo mecanismo de envíos que el sistema actual ya tiene; no requiere desarrollo adicional (se descarta como funcionalidad nueva).

> ⚠️ **BRECHA — Denominación canónica del rol operativo (transversal)**
> Pendiente resolver formalmente entre "Gestor de Cobranza" y "Analista de Cuentas por Cobrar". Brecha transversal con RE-FU-023 a RE-FU-027.
