# Endpoints — ProquifaDotNet.Finanzas (.NET Core 10)

Endpoints de la solución nueva `ProquifaDotNet.Finanzas`. Arquitectura Clean Architecture con CQRS. Consumida por el frontend de ProquifaDotNet y por llamadas inter-API desde ProquifaDotNet (.NET 4.8).

> **Base URL:** `https://{servidor-finanzas}/api/`
> **Autenticación:** IdentityServer (token JWT en header `Authorization: Bearer {token}`)
> **Nota:** Las tablas de datos (`fcc*`, `CFDIGenerada`, `EmpresaFolio`, etc.) residen físicamente en la BD ProquifaDotNet (EmpresaFolio movida desde ProquifaDotNetTimbrado el 07/07/2026) y son accedidas vía EF Core Scaffold en `Finanzas.Infrastructure`.
> **Nota (07/07/2026):** RE-016 a RE-022 y RE-032/033 fueron resincronizados contra las rutas literales de cada `-Back.md`/`-Tareas.md`. El módulo Validar Cobro (RE-023/024/026/027/028/029/030) usa la convención propia `/api/v1/validate-collection/*` — **confirmada por el equipo como la real** — agrupador de módulo `validate-collection` (traducción de "validar-cobro") seguido del recurso en inglés singular, en lugar del patrón `api/v1/{resource}` sin agrupador usado por el resto de la solución. Solo lo relativo a **Cobros** en ProquifaDotNet queda deprecado; los endpoints de **Catálogos** (`catMoneda`, `catMedioDePago`, `vEmpresaDatosBancarios`, `CorreoRecibido*`) **siguen activos** — Venta Interna los sigue usando — y Finanzas simplemente construye sus propios endpoints bajo `/api/v1/*` que leen de esas mismas fuentes (ver `Endpoints-ProquifaDotNet.md`).

---

## Índice

- [RE-016 — Proforma México](#re-016--proforma-méxico)
- [RE-018 — CfdiController + FAA Listado](#re-018--cfdicontroller--faa-listado)
- [RE-019 — FAA Detalle y Timbrado México](#re-019--faa-detalle-y-timbrado-méxico)
- [RE-020 — FAA / Factura Perú (adaptación regional)](#re-020--faa--factura-perú-adaptación-regional)
- [RE-021 — Persistencia PDF Factura México (sin endpoint)](#re-021--persistencia-pdf-factura-méxico-sin-endpoint)
- [RE-022 — Persistencia CPE Factura Perú (sin endpoint)](#re-022--persistencia-cpe-factura-perú-sin-endpoint)
- [RE-023 — Validar Cobro: Buzón, Cobros y Proformas / Pantalla Principal](#re-023--validar-cobro-buzón-cobros-y-proformas--pantalla-principal)
- [RE-024 — Validar Cobro Paso 1 México (Catálogos + Wizard)](#re-024--validar-cobro-paso-1-méxico-catálogos--wizard)
- [RE-026 — Validar Cobro Paso 2 México (Asociación Cobro-Documento)](#re-026--validar-cobro-paso-2-méxico-asociación-cobro-documento)
- [RE-027 — Validar Cobro Paso 2 Perú](#re-027--validar-cobro-paso-2-perú)
- [RE-028 — Validar Cobro Paso 3 México (Facturación)](#re-028--validar-cobro-paso-3-méxico-facturación)
- [RE-030 — Complemento de Pago (sin endpoint propio)](#re-030--complemento-de-pago-sin-endpoint-propio)
- [RE-032 — Notas de Crédito México](#re-032--notas-de-crédito-méxico)
- [RE-033 — Notas de Crédito Perú](#re-033--notas-de-crédito-perú)

---

## RE-016 — Proforma México

**Controller:** `ProformaController`

| Método | Ruta                        | Descripción                                                                                                                         | Parámetros entrada                                                    | Respuesta                 |
| ------ | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------- |
| GET    | `/api/v1/proforma/{id}`     | Obtener proforma por Id                                                                                                             | `id` en path                                                          | `TpQuoteDto`              |
| POST   | `/api/v1/proforma`          | Crear proforma                                                                                                                      | Body: `TpProformaPedido`                                              | `TpQuoteDto`              |
| PUT    | `/api/v1/proforma/{id}`     | Actualizar proforma                                                                                                                 | `id` en path; Body: `TpProformaPedido`                                | `TpQuoteDto`              |
| DELETE | `/api/v1/proforma/{id}`     | Desactivar proforma                                                                                                                 | `id` en path                                                          | `204 NoContent`           |
| POST   | `/api/v1/proforma/list`     | Listado paginado                                                                                                                    | Body: `QueryInfo`                                                     | `QueryResult<TpQuoteDto>` |
| POST   | `/api/v1/proforma/{id}/pdf` | Genera PDF de proforma sin persistir (previsualización) o con persistencia; invoca DocumentBuilder con template `{EMPRESA}_MEX_PRO` | `id` en path; Body: `GenerateQuotePdfRequest { IdRegion, Persistir }` | `byte[]` PDF              |
| GET    | `/api/v1/proforma/{id}/pdf` | Descarga PDF histórico de proforma desde MinIO sin regenerar                                                                        | `id` en path                                                          | `application/pdf`         |

Fuente: `R16A-RE-FU-016-Back-Finanzas.md` (CRUD base) + `R16A-RE-FU-016-Back.md` líneas 38-42, 183-234 (subrecurso `pdf`).

---

## RE-018 — CfdiController + FAA Listado

**Controllers:** `CfdiController` (recurso de negocio CFDI) + `AdvanceInvoiceController` (listado FAA)

> **Nota de arquitectura:** el CFDI como entidad de negocio (`CFDIGenerada`) vive en Finanzas, no en Timbrado. `CfdiController` arma los datos fiscales, llama al endpoint técnico de Timbrado según el tipo de documento (`POST /api/v1/stamp/invoice`, `/payment-complement` o `/credit-note`, ver `Endpoints-Timbrado.md`) y persiste el resultado. Ruta alineada a `api/v1/{resource}/{id}/{subresource}` (Reglas al diseñar — regla 9): recurso singular `cfdi`.

| Método | Ruta                            | Descripción                                                                               | Parámetros entrada                                                                           | Respuesta                              |
| ------ | ------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | -------------------------------------- |
| POST   | `/api/v1/cfdi`                  | Arma datos fiscales, llama a Timbrado, persiste `CFDIGenerada` + `Archivo` (XML)          | Body: request específico del documento (discriminado por el llamador — FAA, Factura, NC, CP) | `CfdiResponseDto`                      |
| POST   | `/api/v1/cfdi/{id}/cancel`      | Llama a Timbrado para cancelar ante SAT/SUNAT y actualiza Estado en `CFDIGenerada`        | `id` en path; Body: `{ ClaveMotivo }`                                                        | `CfdiCancellationResponse`             |
| GET    | `/api/v1/cfdi/{id}`             | Consulta `CFDIGenerada` por Id                                                            | `id` en path                                                                                 | `CFDIGeneradaDto`                      |
| GET    | `/api/v1/cfdi/{id}/xml`         | Descarga XML desde MinIO vía `Archivo.FileKey/FileBucket`                                 | `id` en path                                                                                 | `application/xml`                      |
| POST   | `/api/v1/cfdi/search`           | Listado paginado con `QueryInfo`                                                          | Body: `QueryInfo`                                                                            | `QueryResult<CFDIDto>`                 |
| POST   | `/api/v1/advanceInvoice/search` | Listado FAA agrupado por cliente, paginado, filtrado por cartera del cobrador autenticado | Body: `QueryInfo { SortField, SortDirection, Filters, PageSize, DesiredPage }`               | `QueryResult<AdvanceInvoiceClientDto>` |

Fuente: `R16A-RE-FU-018-Back.md` líneas 155-217, 244-260.

---

## RE-019 — FAA Detalle y Timbrado México

**Controller:** `AdvanceInvoiceController` (ampliación)

| Método | Ruta                                       | Descripción                                                                                                                          | Parámetros entrada                       | Respuesta                                                      |
| ------ | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------- | -------------------------------------------------------------- |
| POST   | `/api/v1/advanceInvoice/{clientId}/detail` | Listado de pedidos del cliente con estado FAA                                                                                        | `clientId` en path                       | `List<AdvanceOrderDto>`                                        |
| POST   | `/api/v1/advanceInvoice/{id}/generate`     | Orquesta: arma datos → `CfdiService` (llama a Timbrado internamente vía `POST /api/v1/cfdi`) → persiste resultado (PDF/XML en MinIO) | `id` en path (`IdFccFactura`, RE-FU-015) | `GeneratedAdvanceInvoiceDto { IdCFDIGenerada, UUID, RutaPDF }` |
| POST   | `/api/v1/advanceInvoice/{id}/preview`      | Genera PDF preview sin timbrar para modal de previsualización; invoca DocumentBuilder                                                | `id` en path                             | `byte[]` PDF                                                   |
| POST   | `/api/v1/advanceInvoice/{id}/send`         | Envía correo con PDF + XML al cliente, marca `fccFactura.Enviada=1` (RE-FU-015), ejecuta salida operativa                            | `id` en path                             | `SendStatusDto`                                                |

Fuente: `R16A-RE-FU-019-Back.md` líneas 117-124, 394-420.

---

## RE-020 — FAA / Factura Perú (adaptación regional)

Reutiliza los mismos endpoints que RE-019. El branch interno de Perú se activa cuando `RegionClave = 'PER'` en el body del request.

| Ruta                                            | Variante Perú                                                                                         | Estado      |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ----------- |
| `POST /api/v1/advanceInvoice/{clientId}/detail` | `RegionClave='PER'` filtra pedidos Perú                                                               | ✅ Reutiliza |
| `POST /api/v1/advanceInvoice/{id}/generate`     | Si `PER` → arma UBL 2.1 → `CfdiService.GenerateAsync` (internamente `POST /api/v1/stamp/invoice` en Timbrado) | ⚙️ Extiende |
| `POST /api/v1/advanceInvoice/{id}/preview`      | Si `PER` → template PDF `GOLPERU_PER_FAC`                                                             | ⚙️ Extiende |
| `POST /api/v1/advanceInvoice/{id}/send`         | Sin diferencias regionales                                                                            | ✅ Reutiliza |

Fuente: `R16A-RE-FU-020-Back.md` líneas 131-154.

---

## RE-021 — Persistencia PDF Factura México (sin endpoint)

> ⚠️ **Corrección (07/07/2026):** la versión anterior de este archivo listaba `PUT /api/cfdi/{IdCFDI}/archivo-pdf` como endpoint de RE-021. Esa ruta **no existe** — el propio `R16A-RE-FU-021-Back.md` (línea 508) es explícito: la actualización es un `UPDATE CFDIGenerada SET IdArchivoPdf` **directo vía EF Core dentro de Finanzas**, sin endpoint HTTP, "ya que `CfdiService`/`CFDIGenerada` son propiedad de este mismo aplicativo".

No hay tabla de endpoints para este requisito — `PersistMexicoInvoicePdfService` (Application) genera el PDF, sube a MinIO, hace `INSERT Archivo` y `UPDATE CFDIGenerada.IdArchivoPdf` en el mismo proceso que generó/timbró la factura (RE-019 `POST /api/v1/advanceInvoice/{id}/generate` o RE-028 `POST /api/v1/validate-collection/fiscalDocumentLine/{id}/stamp`, según el flujo).

---

## RE-022 — Persistencia CPE Factura Perú (sin endpoint)

> **Migración (06/07/2026):** el endpoint legado quedó obsoleto. `fccFactura` (RE-FU-015, reemplaza `tpProformaAdelanto`) es propiedad de `ProquifaDotNet.Finanzas` (Scaffold EF Core en `Finanzas.Infrastructure`) — Finanzas actualiza `fccFactura.IdCFDIGenerada` directamente en BD vía EF Core, sin necesidad de un controller/endpoint intermedio en ProquifaDotNet .NET Fx 4.8.

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| ~~PUT~~ | ~~`/api/tpProformaAdelanto/{Id}/cpe-generada`~~ | **Retirado** — reemplazado por `UPDATE fccFactura SET IdCFDIGenerada` directo vía EF Core (sin endpoint) | — | — |

---

## RE-023 — Validar Cobro: Buzón, Cobros y Proformas / Pantalla Principal

**Controller:** `PaymentValidationController`

> ✅ **Confirmado (07/07/2026):** el prefijo `/api/v1/validate-collection/*` **es la convención real de Finanzas** para todo el wizard de Validar Cobro — agrupador de módulo `validate-collection` (traducción de "validar-cobro") seguido del recurso/subrecurso en **inglés singular**, conforme a la regla 9 de `Diseño y Desarrollo/Reglas al diseñar.md` (`api/v1/{resource}/{id}/{subresource}`). Solo lo relativo a **Cobros** en ProquifaDotNet queda deprecado conforme Finanzas asume esa pieza; los **Catálogos** que Finanzas consume como fuente de datos (moneda, medio de pago, cuentas bancarias, buzón de correo) **siguen activos** — Venta Interna los sigue usando — ver notas por sección y `Endpoints-ProquifaDotNet.md`.

| Método | Ruta                                                      | Descripción                                                                                                                                                        | Parámetros entrada                                                             | Respuesta                                       |
| ------ | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ | ----------------------------------------------- |
| GET    | `/api/v1/validate-collection/mailbox/{idCliente}/pending` | Lista pendientes activos del Buzón de Cobros para el cliente                                                                                                       | `idCliente` en path                                                            | `List<PendingPaymentFolioDto>`                  |
| PUT    | `/api/v1/validate-collection/mailbox/{id}/close`          | Cierra pendiente del Buzón (`Activo=0`)                                                                                                                            | `id` en path                                                                   | `204 NoContent`                                 |
| GET    | `/api/v1/validate-collection/payment/{id}`                | Obtiene cobro por Id (`fccPagoCliente`)                                                                                                                            | `id` en path                                                                   | `ClientPaymentDto`                              |
| GET    | `/api/v1/validate-collection/payment`                     | Lista cobros activos del cliente                                                                                                                                   | `?idCliente={guid}`                                                            | `List<ClientPaymentDto>`                        |
| PUT    | `/api/v1/validate-collection/payment`                     | INSERT borrador (`Confirmado=0`) / UPDATE borrador del cobro (Paso 1)                                                                                              | Body: `ClientPaymentDto`                                                       | `ClientPaymentDto` con Id asignado              |
| GET    | `/api/v1/validate-collection/quote`                       | Lista pedidos pendientes del cliente para modal Gestionar Cobranza (`MontoPendiente > 0 AND Cancelada = 0`)                                                        | `?idCliente={guid}`                                                            | `List<QuoteDto>` + header `MontoTotalPendiente` |
| PUT    | `/api/v1/validate-collection/quote/promiseDate`           | Actualiza fecha estimada de pago; genera historial en `fccFechaEstimadaPagoHistorial`                                                                              | Body: `List<{ IdTpProformaPedido, Fecha, IdUsuarioCambio, Motivo? }>`          | `204 NoContent`                                 |
| POST   | `/api/v1/validate-collection/client/search`               | Listado paginado de clientes con acción contextual (`REALIZAR_COBROS` o `GESTIONAR_COBRANZA`); incluye saldo pendiente, cobros pendientes y SLA vencido calculados | Body: `QueryInfo { SortField, SortDirection, Filters, PageSize, DesiredPage }` | `QueryResult<PaymentValidationClientDto>`       |

---

## RE-024 — Validar Cobro Paso 1 México (Catálogos + Wizard)

**Controller:** `PaymentValidationController`

> ✅ **Confirmado (07/07/2026):** los **3 catálogos del formulario** (moneda, medio de pago, cuenta destino) y el **detalle de correo/adjuntos del Buzón** **NO tienen endpoint propio en Finanzas** — el frontend los consume **directamente** de los endpoints existentes de ProquifaDotNet: `POST /Catalogos/catMoneda`, `POST /Catalogos/catMedioDePago`, `POST /Catalogos/vEmpresaDatosBancarios`, `GET /Catalogos/CorreoRecibido`, `GET /Catalogos/CorreoRecibidoContenido`, `POST /Catalogos/ArchivoCorreoRecibido`, `GET /Catalogos/Archivo` — todos activos, no deprecados, en uso por Venta Interna y confirmados por captura de tráfico HTTP real — ver `Endpoints-ProquifaDotNet.md` (RE-024).

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| GET | `/api/v1/validate-collection/inconsistencyType` | Catálogo de tipos de inconsistencia filtrado por paso | `?step=1` | `List<InconsistencyTypeDto>` |
| POST | `/api/v1/validate-collection/client/{idCliente}/payment/{idFCCPagoCliente}/inconsistency` | Registra inconsistencia del cobro en el Paso 1; INSERT en `fccInconsistenciaCobro` | `idCliente`, `idFCCPagoCliente` en path; Body: tipo + datos de inconsistencia | `PaymentInconsistencyDto` |
| GET | `/api/v1/validate-collection/client/{idCliente}/wizardStatus` | Retorna estado actual del wizard para el cliente (permite reanudar en el último paso activo) | `idCliente` en path | `WizardStatusDto { ActiveStep, ResumeData }` |
| POST | `/api/v1/validate-collection/payment/{id}/finalize` | Finaliza la captura del cobro (Paso 1): genera folio COB, marca `Confirmado=1`; el cobro queda editable hasta el timbrado | `id` en path | `ClientPaymentDto` con `Folio` asignado |
| PUT | `/api/v1/validate-collection/payment/{id}/edit` | Edita un cobro ya capturado (`Confirmado=1`) mientras `BloqueadoPorTimbrado=0`; recalcula TC si cambia moneda/fecha | `id` en path; Body: campos editables del cobro | `ClientPaymentDto` actualizado |

---

## RE-026 — Validar Cobro Paso 2 México (Asociación Cobro-Documento)

**Controller:** `PaymentValidationStep2Controller`

| Método | Ruta                                                                     | Descripción                                                                                                                                                                           | Parámetros entrada                                         | Respuesta                     |
| ------ | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------- |
| GET    | `/api/v1/validate-collection/client/{idCliente}/pendingDocument`         | Lista proformas y facturas pendientes del cliente para el panel central del Paso 2                                                                                                    | `idCliente` en path                                        | `List<PendingDocumentDto>`    |
| GET    | `/api/v1/validate-collection/client/{idCliente}/activeCreditNote`        | Lista NCs vigentes del cliente disponibles para aplicar (`Aplicada=0, Activo=1`)                                                                                                      | `idCliente` en path                                        | `List<ActiveCreditNoteDto>`   |
| POST   | `/api/v1/validate-collection/client/{idCliente}/balance/calculate`       | Calcula el saldo dinámico de la asociación (multi-divisa) al seleccionar/deseleccionar elementos                                                                                      | `idCliente` en path; Body: `CalculateBalanceRequestDto`    | `CalculateBalanceResponseDto` |
| POST   | `/api/v1/validate-collection/client/{idCliente}/associationConfirmation` | Persiste la asociación cobro ↔ documento(s) transaccionalmente: INSERT `fccPagoFacturaPedido`/`fccPagoFacturaAdelanto` + UPDATE NCs aplicadas + INSERT saldo a favor si hay sobrepago | `idCliente` en path; Body: `ConfirmAssociationRequestDto`  | `204 NoContent`               |
| PUT    | `/api/v1/validate-collection/client/{idCliente}/association/draft`       | Auto-guardado del estado en progreso de la asociación (cobros, documentos, NCs, inconsistencias)                                                                                      | `idCliente` en path; Body: `AutoSaveAssociationRequestDto` | `204 NoContent`               |

> **Nota:** el registro de inconsistencias del Paso 2 reutiliza el mismo endpoint de RE-024 (`POST /api/v1/validate-collection/client/{idCliente}/payment/{idFCCPagoCliente}/inconsistency`), extendiendo el catálogo `GET /api/v1/validate-collection/inconsistencyType?step=2` — no agrega una ruta nueva.

---

## RE-027 — Validar Cobro Paso 2 Perú

Mismos endpoints que RE-026 con variaciones regionales: `region=PER`, moneda base PEN, sin trigger de Complemento de Pago, un único emisor (GOLPERU).

| Endpoint                                                                      | Diferencia vs. México                 |
| ----------------------------------------------------------------------------- | ------------------------------------- |
| `GET /api/v1/validate-collection/client/{idCliente}/pendingDocument`          | Filtra por `IdRegion=PER`             |
| `GET /api/v1/validate-collection/client/{idCliente}/activeCreditNote`         | Filtra por `IdRegion=PER`             |
| `POST /api/v1/validate-collection/client/{idCliente}/balance/calculate`       | Sin diferencias regionales            |
| `POST /api/v1/validate-collection/client/{idCliente}/associationConfirmation` | Sin generación de Complemento de Pago |
| `PUT /api/v1/validate-collection/client/{idCliente}/association/draft`        | Sin diferencias regionales            |

---

## RE-028 — Validar Cobro Paso 3 México (Facturación)

**Controller:** `PaymentValidationStep3Controller`

| Método | Ruta                                                                       | Descripción                                                                                                                                                                    | Parámetros entrada                                    | Respuesta                    |
| ------ | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------- | ---------------------------- |
| POST   | `/api/v1/validate-collection/fiscalDocumentStep/initialize`                | Crea las líneas de `fccDocumentoFiscalCobro` por cada documento de la asociación del Paso 2 al avanzar desde Paso 2, o recupera el estado existente al reingresar sin duplicar | Body: `{ IdFCCPagoCliente }`                          | `Step3InitializedDto`        |
| PUT    | `/api/v1/validate-collection/fiscalDocumentLine/{idLinea}/cfdiConfig`      | Persiste Uso CFDI y/o Método de Pago de forma inmediata (auto-guardado) al modificar campos en pantalla                                                                        | `idLinea` en path; Body: `UpdateLineConfigurationDto` | `UpdateLineConfigurationDto` |
| POST   | `/api/v1/validate-collection/fiscalDocumentLine/{idLinea}/pdfPreview`      | Genera PDF de previsualización en memoria sin persistir vía DocumentBuilder; template según tipo de línea                                                                      | `idLinea` en path                                     | `byte[]` PDF                 |
| POST   | `/api/v1/validate-collection/fiscalDocumentLine/{idLinea}/stamp`           | Llama Timbrado y ejecuta el escenario de timbrado correcto por tipo de línea (incluye cascada Factura + Complemento cuando el método de pago es PPD)                           | `idLinea` en path; Body: `StampLineCommand`           | `StampLineResultDto`         |
| POST   | `/api/v1/validate-collection/fiscalDocumentLine/{idLinea}/send`            | Genera PDF de Confirmación de Pedido, despacha correo con CFDI + Confirmación vía ProquifaDotNet.EnvioCorreo y marca la línea como `ENVIADO`                                   | `idLinea` en path; Body: `SendLineRequestDto`         | `SendLineResultDto`          |
| GET    | `/api/v1/validate-collection/fiscalDocumentStep/{idCliente}/closingStatus` | Verifica si todas las líneas del cliente están en `ENVIADO` para permitir el cierre automático del wizard                                                                      | `idCliente` en path                                   | `Step3ClosureStatusDto`      |

RE-029 (Perú) reutiliza los mismos endpoints del Paso 3 bajo la misma convención `/api/v1/validate-collection/fiscalDocumentStep|fiscalDocumentLine` sin rutas nuevas — branch interno por región (SUNAT en vez de SAT, sin trigger de Complemento de Pago para FAA directa). Ver `Endpoints-Timbrado.md`, que expone un endpoint por tipo de documento (`/api/v1/stamp/invoice`, `/payment-complement`, `/credit-note`) reutilizado tanto para SAT como para SUNAT.

---

## RE-030 — Complemento de Pago (sin endpoint propio)

No agrega endpoints. El Complemento de Pago se genera como parte de la cascada PPD dentro de `POST /api/v1/validate-collection/fiscalDocumentLine/{idLinea}/stamp` (RE-028/029) cuando el método de pago de la factura relacionada es PPD — no hay una ruta separada para "timbrar complemento".

---

## RE-032 — Notas de Crédito México

**Controller:** `CreditNoteController` (recurso `creditNote`) + `CfdiController` (timbrado, compartido con RE-018/021/028)

| Método | Ruta                                    | Descripción                                                                                                                                             | Parámetros entrada                                                           | Respuesta                          |
| ------ | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- | ---------------------------------- |
| GET    | `/api/v1/creditNote`                    | Pantalla principal — NCs agrupadas por cliente (Total NC, Vigentes, Por Aplicar, Monto Total) filtrado por cartera del cobrador autenticado             | `?clientId={guid}&currency={str}&dateFrom={date}&dateTo={date}` (opcionales) | `List<CreditNoteClientSummaryDto>` |
| GET    | `/api/v1/client/{idCliente}/creditNote` | Drill-down — lista NCs del cliente con folio, fecha, monto, estado y comprobante origen                                                                 | `idCliente` en path                                                          | `List<CreditNoteDto>`              |
| GET    | `/api/v1/client/{id}/eligibleInvoice`   | Lista facturas vigentes de prepago del cliente elegibles como origen de NC (≤ 5 años, no canceladas)                                                    | `id` en path                                                                 | `List<EligibleInvoiceDto>`         |
| GET    | `/api/v1/cfdi/{id}/lineItem`            | Lista partidas de la factura origen (para wizard modalidad por partidas)                                                                                | `id` en path (`IdCFDIGenerada`)                                              | `List<InvoiceLineItemDto>`         |
| POST   | `/api/v1/creditNote`                    | Persiste `fccNotaCredito` + `fccNotaCreditoPartida` (si aplica) en estado `PENDIENTE` — borrador previo al timbrado                                     | Body: `CreditNoteDraftRequest`                                               | `CreditNoteDto`                    |
| GET    | `/api/v1/creditNote/{id}/pdf/preview`   | Genera PDF preview de NC en memoria sin timbrar                                                                                                         | `id` en path                                                                 | `byte[]` PDF                       |
| POST   | `/api/v1/creditNote/{id}/stamp`         | Timbra NC México: llama Timbrado (`POST /api/v1/cfdi`), persiste PDF/XML en MinIO, cancela factura origen si `CancelarFacturaOrigen=true`, envía correo | `id` en path; Body: `CreditNoteMexicoRequest`                                | `CreditNoteMexicoResponse`         |
| POST   | `/api/v1/creditNote/{id}/resendEmail`   | Reenvía correo de NC usando PDF/XML ya existentes en MinIO                                                                                              | `id` en path                                                                 | `SendStatusDto`                    |
| GET    | `/api/v1/cancellationReason`            | Catálogo de motivos de cancelación SAT (`catMotivoCancelacionSAT`) para el modal de cancelación condicional                                             | —                                                                            | `List<CancellationReasonDto>`      |

Fuente: `R16A-RE-FU-032-Tareas.md` (rutas), `R16A-RE-FU-032-Back.md` Parte B (comportamiento).

---

## RE-033 — Notas de Crédito Perú

**Controller:** `CreditNoteController` (recurso `creditNote`, compartido con RE-032 — discriminado por región del cliente) + `CfdiController` (timbrado, compartido con RE-018/021/028)

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| GET | `/api/v1/creditNote` | Pantalla principal — NCs Perú agrupadas por cliente filtrado por cartera | `?clientId={guid?}&currency={str?}&dateFrom={date?}&dateTo={date?}` | `List<CreditNoteClientSummaryDto>` |
| GET | `/api/v1/client/{idCliente}/creditNote` | Drill-down — lista NCs Perú del cliente | `idCliente` en path | `List<CreditNoteDto>` |
| GET | `/api/v1/creditNoteReasonSunat` | Catálogo de motivos de NC SUNAT (catálogo 09) | — | `List<SunatCreditReasonDto>` |
| GET | `/api/v1/client/{id}/eligibleInvoice` | Lista facturas Perú elegibles como origen de NC | `id` en path | `List<EligibleInvoiceDto>` |
| GET | `/api/v1/cfdi/{id}/lineItem` | Lista partidas de la factura CPE origen | `id` en path (`IdCFDIGenerada`) | `List<InvoiceLineItemDto>` |
| GET | `/api/v1/creditNote/{id}/pdf/preview` | Genera PDF preview de NC Perú en memoria sin timbrar | `id` en path | `byte[]` PDF |
| POST | `/api/v1/creditNote/{id}/stamp` | Timbra NC Perú: llama Timbrado (`POST /api/v1/cfdi`), persiste PDF/XML en MinIO Perú, envía correo | `id` en path; Body: `CreditNotePeruStampingRequest` | `CreditNotePeruStampingResponse` |
| POST | `/api/v1/creditNote/{id}/resendEmail` | Reenvío de correo NC Perú con PDF/XML existentes | `id` en path | `SendStatusDto` |

Fuente: `R16A-RE-FU-033-Back.md` líneas 34-164.

---

## Resumen de Controllers

| Controller | Requisito(s) | Endpoints | Notas |
|---|---|---|---|
| `ProformaController` | RE-016 | 7 | CRUD + subrecurso `pdf` |
| `CfdiController` | RE-018, RE-021, RE-028, RE-032, RE-033 | 5 | Recurso de negocio compartido; llama a Timbrado internamente |
| `AdvanceInvoiceController` | RE-018, RE-019, RE-020 | 5 (4 reutilizados con branches regionales + 1 listado) | |
| `PaymentValidationController` | RE-023, RE-024 | 13 | Bajo `/api/v1/validate-collection/*`; solo lo relativo a Cobros en ProquifaDotNet queda deprecado; catálogos del formulario y detalle de correo/adjuntos no tienen endpoint propio — consumo directo de `/Catalogos/*` |
| `PaymentValidationStep2Controller` | RE-026, RE-027 | 5 | Bajo `/api/v1/validate-collection/*` |
| `PaymentValidationStep3Controller` | RE-028, RE-029 | 6 | Bajo `/api/v1/validate-collection/*`; RE-029 reutiliza sin rutas nuevas |
| `CreditNoteController` | RE-032, RE-033 | 9 (7 compartidas con RE-033 + 2 propias) / 8 (7 compartidas con RE-032 + 1 propia) | |
| **Total** | | **~50** | |
