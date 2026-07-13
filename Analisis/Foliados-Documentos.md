# Foliados de Documentos — R16

Referencia centralizada de los esquemas de foliado para cada tipo de documento emitido por ProquifaDotNet.Finanzas. Incluye tipo de foliador, serie, formato, segmentación y tablas de BD involucradas.

**Tabla maestra de foliadores `EmpresaFolio`** (gestionada por `EmpresaFolioRepository` con `UPDLOCK` atómico al asignar folio):

| Campo            | Descripción                                                 |
| ---------------- | ----------------------------------------------------------- |
| `IdEmpresa`      | Empresa emisora del documento                               |
| `Serie`          | Código que identifica el tipo de documento (A, P, P2, PRF…) |
| `UltimoFolio`    | Último consecutivo asignado                                 |
| `FormatoFolio`   | Patrón de formato del folio                                 |
| `LongitudMaxima` | Longitud máxima del campo folio                             |

---

## 1. Proforma

| Campo            | Valor                                                                                                    |
| ---------------- | -------------------------------------------------------------------------------------------------------- |
| **Tipo CFDI**    | N/A (documento interno no fiscal)                                                                        |
| **Módulo**       | Tramitar Pedido                                                                                          |
| **Serie**        | PQF2 / Prefijo visual `PRF-`                                                                             |
| **Formato**      | `PRF-MMDDAA-[consecutivo]` (ej. `PRF-031826-691`)                                                        |
| **Segmentación** | **Global** — un solo contador para todo el sistema (no por empresa)                                      |
| **Mecanismo**    | `EmpresaFolio` con `UPDLOCK` atómico                                                                     |
| **Tabla BD**     | `EmpresaFolio` (Series PQF2) + `tpProformaPedido.FolioProforma` + `tpProformaPedido.ConsecutivoProforma` |
| **Región**       | México únicamente (pedidos Prepago sin Factura por Adelantado)                                           |
| **Requisito**    | R16A-RE-FU-016                                                                                           |

> **Pendiente: confirmar si el prefijo `PRF-` se persiste en el campo `FolioProforma` de BD o solo en la representación visual del PDF.**

---

## 2. Factura

| Campo            | Valor                                                                                                                         |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Tipo CFDI**    | `I` — Ingreso (CFDI 4.0)                                                                                                      |
| **Módulo**       | Factura por Adelantado / Validar Cobro                                                                                        |
| **Serie**        | `A` (operación actual)                                                                                                        |
| **Formato**      | `A[NNNNNN]` — consecutivo numérico, varchar(6) (ej. `A002374`)                                                                |
| **Segmentación** | **Por empresa emisora** — cada empresa tiene su propio contador                                                               |
| **Empresas**     | Golocaer S.A. de C.V. · Mungen S.A. de C.V. · Proquifa S.A. de C.V. · Proveedora Quimico Farmaceutica S.A. de C.V.            |
| **Folio Fiscal** | UUID de 36 chars asignado por el SAT al timbrar vía TurboPac                                                                  |
| **Mecanismo**    | Integración con tabla legacy `consecutivo` (lectura + incremento atómico)                                                     |
| **Tabla BD**     | `consecutivo` (legacy, fuente del folio) + `EmpresaFolio` (PQF2) + `CFDIGenerada.Serie` + `CFDIGenerada.Folio` + `fccFactura` |
| **Región**       | México (Perú: serie SUNAT `F###` + correlativo — pendiente confirmar con asesor peruano)                                      |
| **Requisito**    | R16A-RE-FU-019 (Factura por Adelantado) · R16A-RE-FU-021 (Diseño PDF Factura México)                                          |

**Proceso de asignación del folio (México — integración legacy):**

1. **Paso 1 — Leer folio actual:** Se consulta la tabla legacy `consecutivo` para obtener el valor vigente por empresa emisora → se asigna como folio de la factura.
2. **Paso 2 — Incrementar contador:** Se actualiza la tabla `consecutivo` incrementando el valor en 1 para que la siguiente factura tome el consecutivo correcto.

> Folios reales de referencia en operación actual: Mungen 2374 · Golocaer 7156 · Proquifa 20913 · Proveedora QF 143103.
>
> **Nota: se evalúa usar una nueva serie en PQF2 para evitar colisión con la numeración del sistema legacy mientras ambos coexistan. Pendiente definir estrategia de sincronización.**

---

## 3. Cobro

| Campo                       | Valor                                                                                                                                                  |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Tipo CFDI**               | N/A (identificador interno operativo, no fiscal)                                                                                                       |
| **Módulo**                  | Validar Cobro — Paso 1                                                                                                                                 |
| **Serie**                   | `COB-` (prefijo fijo)                                                                                                                                  |
| **Formato**                 | `COB-mmddaa-NNNNNN` donde `mmddaa` = fecha efectiva del cobro y `NNNNNN` = siguiente valor del SEQUENCE con relleno de ceros (ej. `COB-071326-000042`) |
| **Segmentación**            | **Pendiente confirmar**: global (MEX + PER compartido) o por región (SeqFolioCobroMEX / SeqFolioCobroPER)                                              |
| **Mecanismo**               | SQL Server SEQUENCE `dbo.SeqFolioCobro` (`NEXT VALUE FOR`)                                                                                             |
| **Tabla BD**                | `fccPagoCliente.Folio` varchar(80)                                                                                                                     |
| **Ciclo de vida del campo** | `NULL` (borrador) → `COB-mmddaa-NNNNNN` al confirmar captura → inmutable al timbrar doc. en Paso 3                                                     |
| **Región**                  | México y Perú (alcance del SEQUENCE pendiente de confirmar)                                                                                            |
| **Requisito**               | R16A-RE-FU-024 (Paso 1 México) · R16A-RE-FU-025 (Paso 1 Perú)                                                                                          |

> El `fccFolioPagoCliente.Folio` + `Consecutivo` es el identificador del correo recibido en el Buzón de Cobros (generado por MailBot), distinto del folio COB del cobro capturado en Validar Cobro.

---

## 4. Complemento de Pago (CDP)

| Campo | Valor |
|---|---|
| **Tipo CFDI** | `P` — Pago (CFDI 4.0) |
| **Módulo** | Validar Cobro — Paso 3 |
| **Serie** | `P` (propuesta — ** pendiente validar **) |
| **Formato** | `P[NNNNNN]` — consecutivo numérico por empresa (ej. `P000123`) |
| **Segmentación** | **Por empresa emisora** |
| **Folio Fiscal** | UUID de 36 chars asignado por el SAT al timbrar vía TurboPac |
| **Mecanismo** | `EmpresaFolio` con `UPDLOCK` atómico al timbrar |
| **Tabla BD** | `EmpresaFolio` + `CFDIGenerada.Serie` + `CFDIGenerada.Folio` + `fccDocumentoFiscalCobro.IdCFDIGeneradaComplemento` |
| **Región** | México (aplica a facturas con Método de Pago PPD) |
| **Requisito** | R16A-RE-FU-030 (Diseño y generación CDP México) |

> **Pendiente: validar serie definitiva "P" con PROQUIFA antes del desarrollo.**

---

## 5. Nota de Crédito (NC)

| Campo | Valor |
|---|---|
| **Tipo CFDI** | `E` — Egreso (CFDI 4.0) |
| **Módulo** | Notas de Crédito |
| **Serie** | `P2` (propuesta — ** pendiente validar **) |
| **Formato** | `P2[NNNNNN]` — consecutivo numérico por empresa (ej. `P2000045`) |
| **Segmentación** | **Por empresa emisora** |
| **Folio Fiscal** | UUID de 36 chars asignado por el SAT al timbrar vía TurboPac |
| **Mecanismo** | `EmpresaFolio` con `UPDLOCK` atómico al timbrar |
| **Tabla BD** | `EmpresaFolio` + `CFDIGenerada.Serie` + `CFDIGenerada.Folio` |
| **CFDI relacionado** | Tipo relación SAT `01` — Nota de Crédito de documentos relacionados |
| **Región** | México (Perú: serie SUNAT pendiente — documentada en R16A-RE-FU-035) |
| **Requisito** | R16A-RE-FU-032 (Módulo NC México) · R16A-RE-FU-034 (Diseño PDF NC México) |

> **Pendiente: validar serie definitiva "P2" con PROQUIFA antes del desarrollo.**

---

## Resumen comparativo

| Documento | Tipo CFDI | Serie (actual/propuesta) | Segmentación | Foliador / Mecanismo | Región |
|---|---|---|---|---|---|
| Proforma | N/A | `PRF-` (prefijo visual) | **Global PQF2** | `EmpresaFolio` UPDLOCK | México |
| Factura | I — Ingreso | `A` | Por empresa | `EmpresaFolio` UPDLOCK | México / Perú⚠️ |
| Cobro | N/A | `COB-` | Global o por región ⚠️ | `SEQUENCE SeqFolioCobro` | México / Perú |
| Complemento de Pago | P — Pago | `P` ⚠️ | Por empresa | `EmpresaFolio` UPDLOCK | México |
| Nota de Crédito | E — Egreso | `P2` ⚠️ | Por empresa | `EmpresaFolio` UPDLOCK | México / Perú⚠️ |

⚠️ = pendiente de validar o confirmar

---

## Pendientes abiertos

| # | Documento | Pendiente |
|---|---|---|
| 1 | Proforma | Confirmar si el prefijo `PRF-` se persiste en `tpProformaPedido.FolioProforma` o solo en el PDF |
| 2 | Factura | Confirmar formato de folio y serie para Región Perú (serie SUNAT F### + correlativo) |
| 3 | Cobro | Confirmar si `SeqFolioCobro` es global (MEX+PER) o un SEQUENCE por región |
| 4 | CDP | Validar serie definitiva "P" con PROQUIFA |
| 5 | NC | Validar serie definitiva "P2" con PROQUIFA |
| 6 | NC | Confirmar esquema de foliado para Región Perú (R16A-RE-FU-035) |
