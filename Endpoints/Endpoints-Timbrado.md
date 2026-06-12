# Endpoints — ProquifaDotNet.Timbrado (.NET Core 10)

Endpoints de la solución nueva `ProquifaDotNet.Timbrado`. Consumida exclusivamente por `ProquifaDotNet.Finanzas` vía llamadas inter-API. No expone endpoints al frontend directamente.

> **Base URL:** `https://{servidor-timbrado}/api/`
> **Autenticación:** IdentityServer (token JWT en header `Authorization: Bearer {token}`)
> **Integración:** PAC TurboPac (México — SAT) / OSE/SUNAT (Perú)
> **Foliador:** Base de datos `ProquifaDotNetTimbrado.EmpresaFolio` (UPDLOCK en cada timbrado)

---

## Índice

- [RE-018 — Base de Timbrado](#re-018--base-de-timbrado)
- [RE-019 — Timbrar FAA México](#re-019--timbrar-faa-méxico)
- [RE-020 — Timbrar FAA Perú (SUNAT)](#re-020--timbrar-faa-perú-sunat)
- [RE-030 — Timbrar Complemento de Pago](#re-030--timbrar-complemento-de-pago)
- [RE-032 — Timbrar Nota de Crédito México](#re-032--timbrar-nota-de-crédito-méxico)
- [RE-033 — Timbrar Nota de Crédito Perú](#re-033--timbrar-nota-de-crédito-perú)

---

## RE-018 — Base de Timbrado

**Controller:** `TimbradoController` / `CfdiController`

| Método | Ruta                     | Descripción                                                                                                                                | Parámetros entrada                                                                                                                                                                                                    | Respuesta                                                               |
| ------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| POST   | `/api/timbrado/timbrar`  | Timbra CFDI genérico; invoca PAC TurboPac, INSERT `CFDIGenerada`, actualiza `EmpresaFolio.UltimoFolio` (con UPDLOCK), INSERT `TimbradoLog` | Body: `TimbrarCFDIRequest { Emisor, Receptor, Conceptos, MetodoPago, TipoComprobante }`                                                                                                                               | `TimbrarCFDIResponse { UUID, Folio, Serie, FechaEmision, XMLTimbrado }` |
| POST   | `/api/timbrado/cancelar` | Solicita cancelación de CFDI ante el PAC/SAT; INSERT `CFDICancelacion`                                                                     | Body: `{ IdCFDI, ClaveMotivo }` — Claves: `01` Comprobante emitido con errores con relación / `02` sin relación / `03` No se llevó a cabo la operación / `04` Operación nominativa relacionada con una factura global | `CancelacionCFDIResponse { AcuseAceptacion, Estado }`                   |
| GET    | `/api/cfdi/{id}`         | Consulta CFDI por Id                                                                                                                       | `id` en path (guid)                                                                                                                                                                                                   | `CFDIGeneradaDto`                                                       |
| GET    | `/api/cfdi/{id}/xml`     | Descarga XML del CFDI timbrado desde MinIO                                                                                                 | `id` en path                                                                                                                                                                                                          | `application/xml`                                                       |
| POST   | `/api/cfdi/listar`       | Listado paginado de CFDIs con filtros                                                                                                      | Body: `QueryInfo`                                                                                                                                                                                                     | `QueryResult<CFDIDto>`                                                  |

---

## RE-019 — Timbrar FAA México

**Controller:** `TimbradoController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| POST | `/api/timbrado/timbrar-faa` | Timbra Factura por Adelantado México (CFDI 4.0, `TipoComprobante='I'`); consume folio `Serie='FA'` con UPDLOCK; INSERT `CFDIGenerada (IdCatTipoCFDI=FACTURA_ANTICIPO)` | Body: `TimbrarFAARequestDto { DatosEmisor, DatosReceptor, DatosConcepto, UsoCFDI, MetodoPago }` | `TimbrarFAAResponseDto { UUID, Folio, Serie, XMLTimbrado, FechaTimbre }` |

---

## RE-020 — Timbrar FAA Perú (SUNAT)

**Controller:** `TimbradoController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| POST | `/api/timbrado/timbrar-sunat` | Timbra Factura electrónica Perú (UBL 2.1 ante SUNAT/OSE); INSERT `CFDIGenerada (IdCatTipoCFDI=FACTURA_CPE)` con número correlativo | Body: `TimbrarFacturaSunatRequestDto { DatosEmisor, DatosReceptor, Lineas[], TipoOperacionSUNAT, CondicionesDePago }` | `TimbrarFacturaSunatResponseDto { Serie, Correlativo, CDR, XMLFirmado }` |

---

## RE-030 — Timbrar Complemento de Pago

**Controller:** `TimbradoController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| POST | `/api/timbrado/complemento-pago` | Timbra Complemento de Pago CFDI 4.0 (Pagos 2.0 `TipoComprobante='P'`); consume folio `Serie='P'` con UPDLOCK; INSERT `CFDIGenerada (IdCatTipoCFDI=COMPLEMENTO_PAGO)` | Body: `TimbrarComplementoPagoRequest { UUIDFacturaRelacionada, FechaPago, FormaPago, Monto, TipoCambio, NumParcialidad, ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto, EquivalenciaDR }` | `TimbrarComplementoPagoResponse { UUID, Serie, Folio, FechaTimbre, XMLTimbrado }` |

---

## RE-032 — Timbrar Nota de Crédito México

**Controller:** `TimbradoController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| POST | `/api/timbrado/nota-credito-mexico` | Timbra NC México CFDI 4.0 (`TipoComprobante='E'`, `ClaveTipoRelacion='01'`); consume folio `Serie='P2'` con UPDLOCK; INSERT `CFDIGenerada (TipoDocumento='E')` + INSERT `CFDI` con UUID SAT | Body: `NotaCreditoMexicoRequest { DatosEmisor, DatosReceptor, UUIDFacturaOrigen, Conceptos[], Motivo, ClaveMotivosCancelacion? }` | `NotaCreditoMexicoResponse { IdCFDIGenerada, UUID, XMLTimbrado }` |
| POST | `/api/timbrado/cancelar-cfdi` | Cancela CFDI ante el SAT; reutilizable para cancelación condicional de factura origen al generar NC | Body: `{ IdCFDI, ClaveMotivo }` | `CancelacionCFDIResponse { AcuseAceptacion, Estado }` |

---

## RE-033 — Timbrar Nota de Crédito Perú

**Controller:** `TimbradoController`

| Método | Ruta | Descripción | Estado | Req. |
|--------|------|-------------|--------|------|
| POST | `/api/timbrado/nota-credito-peru` | Timbra NC Perú CPE tipo 07 (UBL 2.1) ante SUNAT/OSE; INSERT `CFDIGenerada (TipoDocumento='07')` | **Bloqueado** — pendiente definir proveedor SUNAT/modalidad OSE | RE-033 |

---

## Flujo general de timbrado

```
ProquifaDotNet.Finanzas
    │
    ├─ POST /api/timbrado/timbrar-faa          → FAA México (CFDI 4.0 Ingreso)
    ├─ POST /api/timbrado/timbrar-sunat         → FAA / Facturas Perú (UBL 2.1)
    ├─ POST /api/timbrado/complemento-pago      → CP México (CFDI 4.0 Pagos 2.0)
    ├─ POST /api/timbrado/nota-credito-mexico   → NC México (CFDI 4.0 Egreso)
    ├─ POST /api/timbrado/nota-credito-peru     → NC Perú (UBL 2.1 tipo 07) [Bloqueado]
    └─ POST /api/timbrado/cancelar-cfdi         → Cancelación ante SAT
         │
         ↓
   ProquifaDotNetTimbrado.EmpresaFolio
   (UPDLOCK para folio secuencial)
         │
         ↓
   PAC TurboPac (MX) / OSE-SUNAT (PE)
         │
         ↓
   CFDIGenerada (ProquifaDotNet BD)
   XML → MinIO
```

---

## Resumen de Endpoints

| Endpoint                                 | Tipo documento         | País              | Req.   |
| ---------------------------------------- | ---------------------- | ----------------- | ------ |
| `POST /api/timbrado/timbrar`             | CFDI genérico          | México            | RE-018 |
| `POST /api/timbrado/cancelar`            | Cancelación CFDI       | México            | RE-018 |
| `GET /api/cfdi/{id}`                     | Consulta               | México/Perú       | RE-018 |
| `GET /api/cfdi/{id}/xml`                 | Descarga XML           | México/Perú       | RE-018 |
| `POST /api/cfdi/listar`                  | Listado                | México/Perú       | RE-018 |
| `POST /api/timbrado/timbrar-faa`         | FAA — Factura Anticipo | México            | RE-019 |
| `POST /api/timbrado/timbrar-sunat`       | Factura electrónica    | Perú              | RE-020 |
| `POST /api/timbrado/complemento-pago`    | Complemento de Pago    | México            | RE-030 |
| `POST /api/timbrado/nota-credito-mexico` | Nota de Crédito        | México            | RE-032 |
| `POST /api/timbrado/cancelar-cfdi`       | Cancelación CFDI       | México            | RE-032 |
| `POST /api/timbrado/nota-credito-peru`   | Nota de Crédito        | Perú 🚫 Bloqueado | RE-033 |
| **Total**                                |                        |                   | **11** |
