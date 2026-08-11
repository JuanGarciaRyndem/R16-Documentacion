# Endpoints — ProquifaDotNet.Timbrado (.NET Core 10)

Endpoints de la solución nueva `ProquifaDotNet.Timbrado`. Consumida exclusivamente por `ProquifaDotNet.Finanzas` vía llamadas inter-API. No expone endpoints al frontend directamente.

> **Base URL:** `https://{servidor-timbrado}/api/`
> **Autenticación:** IdentityServer (token JWT en header `Authorization: Bearer {token}`)
> **Integración:** PAC SAP (México — SAT) / OSE/SUNAT (Perú)
> **Foliador:** `EmpresaFolio` vive en la BD `ProquifaDotNet`, **propiedad de Finanzas** (`EmpresaFolioRepository`, EF Core — movida desde `ProquifaDotNetTimbrado` el 07/07/2026) — no hay endpoint HTTP en Timbrado para folios; el `StampingController` no los toca.

---

## ⚠️ Corrección de arquitectura (06/07/2026, actualizada 07/07/2026)

Timbrado dejó de ser dueño del recurso de negocio "CFDI" y de las rutas por región (`/api/timbrado/timbrar-faa`, `/api/timbrado/nota-credito-mexico`, etc.). Desde RE-FU-018, Timbrado es **un servicio técnico**, sin tabla de negocio propia:

- No persiste `CFDIGenerada` — solo audita la llamada técnica en `StampingLog` (`AppSetting` + `StampingLog`, BD `ProquifaDotNetTimbrado`).
- No tiene Worker, cola de reintentos, RabbitMQ ni Brevo — es un servicio síncrono de un solo intento por petición. El reintento es responsabilidad de Finanzas.
- Expone **un endpoint por tipo de documento fiscal** (Factura, Complemento de Pago, Nota de Crédito) más la cancelación — en lugar de un endpoint genérico con discriminador en el body. Cada endpoint tiene DTO y validador propios; los tres comparten el mismo pipeline interno (`StampingService` → `SapStampingClient` → `StampingLog`). SAT (México) y SUNAT (Perú) comparten endpoint por tipo de documento; la región se resuelve por los datos del request, sin rutas separadas por región.
- El recurso de negocio `cfdi` (crear, consultar, cancelar, listar) vive en **ProquifaDotNet.Finanzas** (`CfdiController`), que internamente llama a estos endpoints de Timbrado. Ver `Endpoints-Finanzas.md`.

Ver `R16A-RE-FU-018-Back.md` (Parte A — Creación de Solución ProquifaDotNet.Timbrado) para el detalle completo.

---

## Índice

- [RE-018 — Base de Timbrado (StampingController)](#re-018--base-de-timbrado-stampingcontroller)
- [RE-019 — Timbrar FAA México (reutiliza invoice)](#re-019--timbrar-faa-méxico-reutiliza-invoice)
- [RE-020 — Timbrar FAA / Factura Perú (reutiliza invoice)](#re-020--timbrar-faa--factura-perú-reutiliza-invoice)
- [RE-021 — Persistencia PDF Factura México (sin endpoint en Timbrado)](#re-021--persistencia-pdf-factura-méxico-sin-endpoint-en-timbrado)
- [RE-023 — Cancelar Factura (CFDI) ante el SAT (interno — invocado por orquestador Finanzas)](#re-023--cancelar-factura-cfdi-ante-el-sat-interno--invocado-por-orquestador-finanzas)
- [RE-028/029 — Timbrar Factura y Complemento de Pago (reutiliza invoice + payment-complement)](#re-028029--timbrar-factura-y-complemento-de-pago-reutiliza-invoice--payment-complement)
- [RE-032/033 — Timbrar Nota de Crédito México/Perú (reutiliza credit-note)](#re-032033--timbrar-nota-de-crédito-méxicoperú-reutiliza-credit-note)

---

## RE-018 — Base de Timbrado (StampingController)

**Controller:** `StampingController` (uso interno, consumido solo por Finanzas)

> Servicio técnico, no expone un recurso de negocio: las rutas usan una acción (`stamp`) más el tipo de documento en vez de un sustantivo CRUD, ya que no hay una entidad persistida detrás. Alineado en lo demás a `api/v1/{resource}/{id}/{subresource}` (Reglas al diseñar — regla 9). Se define un endpoint por tipo de documento para que cada uno tenga contrato y validación propios: la Factura lleva conceptos e impuestos; el Complemento de Pago lleva el nodo `Pagos` con documentos relacionados (`SubTotal`/`Total` = 0); la Nota de Crédito exige `CFDIRelacionados` con el UUID de la factura origen.

| Método | Ruta                               | Descripción                                                                                                                                                                                                                                                                                    | Parámetros entrada                                                                                                  | Respuesta                                                             |
| ------ | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| POST   | `/api/v1/stamp/invoice`            | Timbra una **Factura** (CFDI tipo I México / CPE 01 Perú, incluye FAA): recibe los datos fiscales ya armados por Finanzas, invoca al PAC (SAP / OSE-SUNAT) una sola vez (sin retry automático), registra `StampingLog` y regresa el resultado — **sin persistir CFDI como entidad de negocio** | Body: `StampInvoiceRequest { DatosEmisor, DatosReceptor, Conceptos, Impuestos, UsoCfdi, MetodoPago, ... }`          | `StampingResponse { Uuid, Series, Folio, Xml, FechaEmision }` ó error |
| POST   | `/api/v1/stamp/payment-complement` | Timbra un **Complemento de Pago** (CFDI tipo P, Pagos20 v2.0): recibe el nodo Pagos con documentos relacionados (UUID factura PPD/FAA, saldos, parcialidad); `SubTotal`/`Total` = 0                                                                                                            | Body: `StampPaymentComplementRequest { DatosEmisor, DatosReceptor, Pagos { DoctoRelacionado, ImpuestosDR? }, ... }` | `StampingResponse { Uuid, Series, Folio, Xml, FechaEmision }` ó error |
| POST   | `/api/v1/stamp/credit-note`        | Timbra una **Nota de Crédito** (CFDI tipo E México / CPE 07 Perú): recibe conceptos + `CFDIRelacionados` obligatorio (UUID factura origen, TipoRelacion 01/03)                                                                                                                                 | Body: `StampCreditNoteRequest { DatosEmisor, DatosReceptor, Conceptos, CfdiRelacionados, Motivo, ... }`             | `StampingResponse { Uuid, Series, Folio, Xml, FechaEmision }` ó error |
| POST   | `/api/v1/stamp/cancel`             | Solicita cancelación de un documento ya timbrado ante el PAC/SAT (recibe UUID + datos mínimos, sin leer tabla propia)                                                                                                                                                                          | Body: `{ Uuid, ClaveMotivo? }`                                                                                      | `StampingCancelResponse { Estado, AcuseAceptacion }`                  |

**Flujo (síncrono, un solo intento, sin persistencia de negocio — común a los 3 endpoints de timbrado):**
```
1. Finanzas arma los datos fiscales del documento y llama al endpoint de su tipo:
   Factura -> POST /api/v1/stamp/invoice
   Complemento de Pago -> POST /api/v1/stamp/payment-complement
   Nota de Crédito -> POST /api/v1/stamp/credit-note
2. StampingService valida el request con el validador del tipo de documento y registra StampingLog (NewStatus=Pending)
3. StampingService invoca SapStampingClient -> PAC SAP/SUNAT genera el documento (1 sola llamada, sin retry)
4a. Éxito: actualiza StampingLog (NewStatus=Stamped), retorna StampingResponse a Finanzas
4b. Error: actualiza StampingLog (NewStatus=Failed, ErrorMessage), retorna el error a Finanzas de inmediato
     -> Finanzas decide si reintenta, en su propio flujo de generación del documento
5. Finanzas, con un StampingResponse exitoso, INSERT/UPDATE CFDIGenerada + Archivo (XML en Minio) — Timbrado ya no participa
```

Fuente: `R16A-RE-FU-018-Back.md` (Parte A — API Endpoints).

---

## RE-019 — Timbrar FAA México (reutiliza invoice)

No agrega endpoints nuevos. `AdvanceInvoiceGenerateService` (Finanzas) llama `POST /api/v1/stamp/invoice` con el request armado para FAA México (CFDI 4.0, `TipoComprobante='I'`). Ver `Endpoints-Finanzas.md` (RE-019 — `POST /api/v1/advanceInvoice/{id}/generate`).

---

## RE-020 — Timbrar FAA / Factura Perú (reutiliza invoice)

No agrega endpoints nuevos. Mismo `POST /api/v1/stamp/invoice`, branch interno por `RegionClave='PER'` arma UBL 2.1 en vez de CFDI 4.0. Ver `R16A-RE-FU-020-Back.md` líneas 131-152.

---

## RE-021 — Persistencia PDF Factura México (sin endpoint en Timbrado)

No aplica a Timbrado. La actualización de `CFDIGenerada.IdArchivoPdf` ocurre con un `UPDATE` directo vía EF Core dentro de **Finanzas** (mismo aplicativo dueño de `CfdiService`/`CFDIGenerada`) — no hay llamada HTTP a Timbrado ni a ningún otro servicio para este paso. Ver `Endpoints-Finanzas.md` (RE-021).

---

## RE-023 — Cancelar Factura (CFDI) ante el SAT (interno — invocado por orquestador Finanzas)

**Controller:** `TimbradoController` (extensión)

> **Rol:** endpoint **interno** que **solo cancela el CFDI de la factura** ante el SAT vía PAC (TurboPac). Invocado exclusivamente por el orquestador de Finanzas (`PUT /api/v1/validate-collection/orders/{orderId}/cancel-non-payment`) dentro del flujo Caso B (pedido con CFDI timbrado, Conducta 2). NO cancela pedido ni proforma — esos los coordina Finanzas. **Idempotente** (consulta `CFDICancelacion` antes de solicitar al PAC). Puede devolver estado asíncrono (`EnProceso`) que Finanzas monitorea con Hangfire.

| Método | Ruta                                             | Descripción                                                                                                                                                                                                       | Parámetros entrada                                       | Respuesta                                                                                                       | Estado    |
| ------ | ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | --------- |
| POST   | `/api/v1/invoices/{invoiceId}/cancel`            | Solicita al PAC la cancelación del CFDI de la factura (`CFDIGenerada`). Arma `requestCancelacion` firmado, invoca `TurboPac.CancelaCfdi`, INSERT/UPDATE en `CFDICancelacion`. Idempotente: si ya hay cancelación registrada, retorna el estado actual sin re-solicitar al PAC. NO toca `tpPedido` ni `tpProformaPedido`. | `invoiceId` (=`IdCFDIGenerada`) en path; Body: `{ claveMotivo }` (fija `03` — No se llevó a cabo la operación, D3 diseño RE-023; a confirmar con área fiscal) | `200 OK { estatus: 'Cancelado' \| 'EnProceso' \| 'Rechazado', folio, fechaCancelacion }` / `409 Conflict` (SAT rechaza síncrono) | **Nuevo** |
| GET    | `/api/v1/invoices/{invoiceId}/cancel/status`     | Consulta el estatus de una cancelación en curso ante el PAC. Invocado por el job de Hangfire de Finanzas para polling cuando la cancelación quedó `EnProceso`. Actualiza `CFDICancelacion.Estatus` según respuesta del PAC. | `invoiceId` en path                                      | `200 OK { estatus: 'Cancelado' \| 'EnProceso' \| 'Rechazado', motivoRechazo? }`                                | **Nuevo** |

> **Diferencia con `POST /api/v1/stamp/cancel` (RE-032/033):** aquel es el endpoint genérico usado por el flujo de Nota de Crédito para cancelar la factura origen; este par (`/invoices/{id}/cancel` + `/cancel/status`) es específico del flujo de **cancelación de pedido por falta de pago** de RE-023 y soporta el ciclo asíncrono con polling. Ambos consumen internamente el mismo `TimbradoService.CancelInvoiceAsync` + `TurboPac.CancelaCfdi`; la diferencia está en el consumidor (RE-023 orquestador vs RE-032/033 Nota de Crédito) y en el soporte de consulta de estatus.

---

## RE-028/029 — Timbrar Factura y Complemento de Pago (reutiliza invoice + payment-complement)

No agrega endpoints nuevos. El endpoint "Timbrar línea" de Finanzas (`POST /api/v1/validate-collection/fiscalDocumentLine/{id}/stamp`) llama internamente `POST /api/v1/stamp/invoice` para Factura/Factura Anticipo y `POST /api/v1/stamp/payment-complement` para el Complemento de Pago — incluida la cascada Factura + Complemento en secuencia cuando el método de pago es PPD. RE-029 (Perú) reutiliza el mismo endpoint de facturas sin diferencias. Ver `Endpoints-Finanzas.md` (RE-028/029).

---

## RE-032/033 — Timbrar Nota de Crédito México/Perú (reutiliza credit-note)

No agrega endpoints nuevos. `CreditNoteController.Stamp` (Finanzas, `POST /api/v1/creditNote/{id}/stamp`) llama internamente `POST /api/v1/stamp/credit-note` con `TipoComprobante='E'` (México) o CPE tipo 07 (Perú, **bloqueado** — pendiente definir proveedor SUNAT/modalidad OSE). La cancelación condicional de la factura origen llama `POST /api/v1/stamp/cancel`. Ver `Endpoints-Finanzas.md` (RE-032/033).

---

## Resumen de Endpoints

| Endpoint                              | Tipo                                                        | Consumido por (vía Finanzas)           |
| ------------------------------------- | ------------------------------------------------------------ | --------------------------------------- |
| `POST /api/v1/stamp/invoice`           | Factura — CFDI I (SAT MEX) / CPE 01 (SUNAT PER), incluye FAA | RE-019, RE-020, RE-028, RE-029          |
| `POST /api/v1/stamp/payment-complement`| Complemento de Pago — CFDI P (Pagos20 v2.0, solo México)     | RE-028, RE-030                          |
| `POST /api/v1/stamp/credit-note`       | Nota de Crédito — CFDI E (SAT MEX) / CPE 07 (SUNAT PER)      | RE-032, RE-033                          |
| `POST /api/v1/stamp/cancel`            | Cancelación genérica ante SAT/SUNAT (Nota de Crédito → factura origen) | RE-032, RE-033                          |
| `POST /api/v1/invoices/{invoiceId}/cancel` | Cancelación de CFDI de factura por falta de pago (orquestador RE-023, soporta asíncrono) | RE-023                                  |
| `GET /api/v1/invoices/{invoiceId}/cancel/status` | Polling del estatus de cancelación en curso (invocado por Hangfire de Finanzas) | RE-023                                  |
| **Total**                              |                                                              | **6**                                   |

> Todo el resto de la lógica de negocio (armado del XML/UBL, folios `EmpresaFolio`, persistencia de `CFDIGenerada`, PDF, MinIO, correo) vive en `ProquifaDotNet.Finanzas` — ver `Endpoints-Finanzas.md`.
