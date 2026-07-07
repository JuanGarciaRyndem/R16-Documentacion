# Endpoints — ProquifaDotNet.Timbrado (.NET Core 10)

Endpoints de la solución nueva `ProquifaDotNet.Timbrado`. Consumida exclusivamente por `ProquifaDotNet.Finanzas` vía llamadas inter-API. No expone endpoints al frontend directamente.

> **Base URL:** `https://{servidor-timbrado}/api/`
> **Autenticación:** IdentityServer (token JWT en header `Authorization: Bearer {token}`)
> **Integración:** PAC SAP (México — SAT) / OSE/SUNAT (Perú)
> **Foliador:** `EmpresaFolio` vive físicamente en la BD `ProquifaDotNetTimbrado`, pero es consumido **directamente por Finanzas** (`EmpresaFolioRepository`, EF Core) — no hay endpoint HTTP en Timbrado para folios; el `StampingController` no los toca.

---

## ⚠️ Corrección de arquitectura (06/07/2026)

Timbrado dejó de ser dueño del recurso de negocio "CFDI" y de las rutas por tipo de documento (`/api/timbrado/timbrar-faa`, `/api/timbrado/nota-credito-mexico`, etc.). Desde RE-FU-018, Timbrado es **un servicio técnico genérico**, sin tabla de negocio propia:

- No persiste `CFDIGenerada` — solo audita la llamada técnica en `StampingLog` (`AppSetting` + `StampingLog`, BD `ProquifaDotNetTimbrado`).
- No tiene Worker, cola de reintentos, RabbitMQ ni Brevo — es un servicio síncrono de un solo intento por petición. El reintento es responsabilidad de Finanzas.
- Expone **un único par de endpoints genéricos**, sin discriminar por tipo de documento (FAA, Factura, NC, Complemento de Pago): el discriminador (`TipoComprobante`, `FiscalDocumentTypeId`, etc.) viaja en el body del request armado por Finanzas.
- El recurso de negocio `cfdi` (crear, consultar, cancelar, listar) vive en **ProquifaDotNet.Finanzas** (`CfdiController`), que internamente llama a estos dos endpoints de Timbrado. Ver `Endpoints-Finanzas.md`.

Ver `R16A-RE-FU-018-Back.md` (Parte A — Creación de Solución ProquifaDotNet.Timbrado) para el detalle completo.

---

## Índice

- [RE-018 — Base de Timbrado (StampingController)](#re-018--base-de-timbrado-stampingcontroller)
- [RE-019 — Timbrar FAA México (reutiliza)](#re-019--timbrar-faa-méxico-reutiliza)
- [RE-020 — Timbrar FAA / Factura Perú (reutiliza)](#re-020--timbrar-faa--factura-perú-reutiliza)
- [RE-021 — Persistencia PDF Factura México (sin endpoint en Timbrado)](#re-021--persistencia-pdf-factura-méxico-sin-endpoint-en-timbrado)
- [RE-028/029 — Timbrar Factura y Complemento de Pago (reutiliza)](#re-028029--timbrar-factura-y-complemento-de-pago-reutiliza)
- [RE-032/033 — Timbrar Nota de Crédito México/Perú (reutiliza)](#re-032033--timbrar-nota-de-crédito-méxicoperú-reutiliza)

---

## RE-018 — Base de Timbrado (StampingController)

**Controller:** `StampingController` (uso interno, consumido solo por Finanzas)

> Servicio técnico, no expone un recurso de negocio: las rutas usan una acción (`stamp`) en vez de un sustantivo CRUD, ya que no hay una entidad persistida detrás. Alineado en lo demás a `api/v1/{resource}/{id}/{subresource}` (Reglas al diseñar — regla 9).

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| POST | `/api/v1/stamp` | Recibe los datos fiscales ya armados por Finanzas, invoca al PAC (SAP México / OSE-SUNAT Perú) una sola vez (sin retry automático), registra `StampingLog` y regresa el resultado — **sin persistir CFDI como entidad de negocio** | Body: `StampingRequest { DatosEmisor, DatosReceptor, Conceptos, TipoDocumento, ... }` (genérico, discriminado por el llamador) | `StampingResponse { Uuid, Series, Folio, Xml, FechaEmision }` ó error |
| POST | `/api/v1/stamp/cancel` | Solicita cancelación de un documento ya timbrado ante el PAC/SAT (recibe UUID + datos mínimos, sin leer tabla propia) | Body: `{ Uuid, ClaveMotivo? }` | `StampingCancelResponse { Estado, AcuseAceptacion }` |

**Flujo (síncrono, un solo intento, sin persistencia de negocio):**
```
1. Finanzas arma los datos fiscales del CFDI y llama -> POST /api/v1/stamp
2. StampingService valida el request y registra StampingLog (NewStatus=Pending)
3. StampingService invoca SapStampingClient -> PAC SAP/SUNAT genera el documento (1 sola llamada, sin retry)
4a. Éxito: actualiza StampingLog (NewStatus=Stamped), retorna StampingResponse a Finanzas
4b. Error: actualiza StampingLog (NewStatus=Failed, ErrorMessage), retorna el error a Finanzas de inmediato
     -> Finanzas decide si reintenta, en su propio flujo de generación del documento
5. Finanzas, con un StampingResponse exitoso, INSERT/UPDATE CFDIGenerada + Archivo (XML en Minio) — Timbrado ya no participa
```

Fuente: `R16A-RE-FU-018-Back.md` líneas 101-163.

---

## RE-019 — Timbrar FAA México (reutiliza)

No agrega endpoints nuevos. `AdvanceInvoiceGenerateService` (Finanzas) llama `POST /api/v1/stamp` con el request armado para FAA México (CFDI 4.0, `TipoComprobante='I'`). Ver `Endpoints-Finanzas.md` (RE-019 — `POST /api/v1/advanceInvoice/{id}/generate`).

---

## RE-020 — Timbrar FAA / Factura Perú (reutiliza)

No agrega endpoints nuevos. Mismo `POST /api/v1/stamp`, branch interno por `RegionClave='PER'` arma UBL 2.1 en vez de CFDI 4.0. Ver `R16A-RE-FU-020-Back.md` líneas 131-152.

---

## RE-021 — Persistencia PDF Factura México (sin endpoint en Timbrado)

No aplica a Timbrado. La actualización de `CFDIGenerada.IdArchivoPdf` ocurre con un `UPDATE` directo vía EF Core dentro de **Finanzas** (mismo aplicativo dueño de `CfdiService`/`CFDIGenerada`) — no hay llamada HTTP a Timbrado ni a ningún otro servicio para este paso. Ver `Endpoints-Finanzas.md` (RE-021).

---

## RE-028/029 — Timbrar Factura y Complemento de Pago (reutiliza)

No agrega endpoints nuevos. El endpoint "Timbrar línea" de Finanzas (`POST /api/v1/validate-collection/fiscalDocumentLine/{id}/stamp`) llama internamente `POST /api/v1/stamp`, incluyendo la cascada Factura + Complemento de Pago en secuencia cuando el método de pago es PPD. RE-029 (Perú) reutiliza el mismo endpoint técnico sin diferencias. Ver `Endpoints-Finanzas.md` (RE-028/029).

---

## RE-032/033 — Timbrar Nota de Crédito México/Perú (reutiliza)

No agrega endpoints nuevos. `CreditNoteController.Stamp` (Finanzas, `POST /api/v1/creditNote/{id}/stamp`) llama internamente `POST /api/v1/stamp` con `TipoComprobante='E'` (México) o CPE tipo 07 (Perú, **bloqueado** — pendiente definir proveedor SUNAT/modalidad OSE). La cancelación condicional de la factura origen llama `POST /api/v1/stamp/cancel`. Ver `Endpoints-Finanzas.md` (RE-032/033).

---

## Resumen de Endpoints

| Endpoint | Tipo | Consumido por (vía Finanzas) |
|----------|------|-------------------------------|
| `POST /api/v1/stamp` | Timbrado genérico (SAT MEX / SUNAT PER) | RE-019, RE-020, RE-028, RE-029, RE-032, RE-033 |
| `POST /api/v1/stamp/cancel` | Cancelación genérica ante SAT/SUNAT | RE-032, RE-033 |
| **Total** | | **2** |

> Todo el resto de la lógica de negocio (armado del XML/UBL, discriminación por tipo de documento, folios `EmpresaFolio`, persistencia de `CFDIGenerada`, PDF, MinIO, correo) vive en `ProquifaDotNet.Finanzas` — ver `Endpoints-Finanzas.md`.
