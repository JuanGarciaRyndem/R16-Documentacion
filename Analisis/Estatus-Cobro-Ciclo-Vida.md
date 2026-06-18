# Ciclo de Vida del Cobro — Estatus por Etapa

**Alcance:** Desde que llega el comprobante al Buzón de Cobros hasta el cierre del Paso 3 en Validar Cobro.  
**Referencia:** RE-008 (Buzón) → RE-023 (Listado) → RE-024/025 (Paso 1) → RE-026/027 (Paso 2) → RE-028/029 (Paso 3)

---

## Resumen del Flujo

```
[Correo llega al buzón]
       ↓
[MailBot clasifica como COBRO]
       ↓
[fccFolioPagoCliente — PENDIENTE]
       ↓
[Listado Validar Cobro — acción "Realizar Cobros"]
       ↓
[PASO 1 — Captura] → fccPagoCliente.Confirmado = 0 (Borrador) → 1 (Confirmado)
       ↓
[PASO 2 — Asociación] → fccPagoFacturaPedido / fccPagoFacturaAdelanto
       ↓
[PASO 3 — Facturación y Envío] → catDocumentoFiscalCobroEstado: PENDIENTE → GENERADO → ENVIADO
```

---

## Etapa 1 — Buzón de Cobros (RE-008)

**Tabla:** `fccFolioPagoCliente`

| Estado | Descripción | Quién lo establece |
|---|---|---|
| **PENDIENTE (activo)** | El correo fue clasificado como cobro por MailBot. Se genera el folio en `fccFolioPagoCliente`. El cobro queda visible en el listado de Validar Cobro para el cobrador asignado. | `GenerarPendienteUseCase` — automático tras clasificación IA |
| **CERRADO** | El folio es cerrado cuando el cobro es vinculado a un documento (proforma o factura) en el Paso 2 del wizard. El cierre se ejecuta via `PUT /api/cobros/folio/{id}/cerrar` invocado desde Finanzas al confirmar la asociación. | Finanzas → ProquifaDotNet al cerrar el Paso 2 |

> **Nota RE-008:** Los correos de clientes sin Cobrador asignado no aparecen en ninguna bandeja de Gestor de Cobranza hasta que el Coordinador de Tesorería complete la asignación (`ClienteCartera`).

---

## Etapa 2 — Listado Principal de Validar Cobro (RE-023)

**Tabla:** lógica de presentación (no hay un campo de estado en BD para este nivel)

| Estado contextual | Descripción | Condición |
|---|---|---|
| **"Realizar Cobros"** | El cliente tiene cobros recibidos pendientes de aplicar (folios en `fccFolioPagoCliente` activos). El botón abre el wizard en el último paso activo. | `fccFolioPagoCliente` con registros activos para el cliente |
| **"Gestionar Cobranza"** | El cliente no tiene cobros recibidos pendientes, pero sí proformas o facturas por cobrar. El botón abre el modal de gestión de cobranza. | Sin `fccFolioPagoCliente` activos, pero con documentos pendientes |
| **Sin acción / sin cliente en listado** | El cliente no tiene pendientes ni documentos por cobrar. No aparece en el listado. | Sin documentos pendientes de cobrar |

> **Cancelación desde listado:** El Gestor puede cancelar un pedido por falta de pago ("Cancelado por falta de pago") desde el modal de Gestionar Cobranza. Esto cambia el estado del pedido y lo saca del listado. La propagación a Legacy/cancelación fiscal es pendiente de definición (RE-023).

---

## Etapa 3 — Paso 1: Captura del Cobro (RE-024, RE-025)

**Tabla:** `fccPagoCliente`  
**Campo de estado:** `Confirmado` (bit)

| Estado | Valor | Descripción | Visualización en wizard |
|---|---|---|---|
| **Sin capturar** | No existe registro | El correo del buzón aún no tiene cobro capturado en esta sesión. | Identificador temporal "COB-N" (consecutivo de sesión) |
| **Borrador** | `Confirmado = 0` | El formulario de captura fue auto-guardado pero el usuario no ha confirmado. Se puede sobreescribir. | "COB-N" temporal (sin folio definitivo) |
| **Confirmado** | `Confirmado = 1` | El cobro fue capturado y confirmado. Genera folio definitivo `COB-mmddaa-NNNN`. Es inmutable (no se sobreescribe). | Folio `COB-mmddaa-NNNN`, fecha, monto + moneda |
| **Saldo a favor** | `Confirmado = 1` + saldo residual | Cobro confirmado cuyo monto supera el adeudo asociado en Paso 2. La etiqueta del monto cambia a "Saldo a favor". | "Saldo a favor" con monto disponible |
| **Con inconsistencia** | `Confirmado = 0 o 1` + registro en `fccInconsistenciaCobro` | El cobro tiene marcada una inconsistencia (comprobante inválido, datos incompletos, etc.). | Indicador visual de inconsistencia |

**Lógica del estado del wizard (OBS-048):**

| Condición | Paso activo al re-entrar |
|---|---|
| Hay cobros con `Confirmado = 0` (borrador) | Paso 1 |
| Todos los cobros con `Confirmado = 1` y Paso 2 pendiente | Paso 2 |
| Paso 2 completo y Paso 3 pendiente | Paso 3 |

---

## Etapa 4 — Paso 2: Asociación de Cobro con Documento (RE-026 MX, RE-027 Perú)

**Tablas:** `fccPagoFacturaPedido`, `fccPagoFacturaAdelanto`, `fccSaldoFavorCliente`

| Estado de la asociación | Descripción | Tabla / Campo |
|---|---|---|
| **Asociación en progreso** | El usuario está seleccionando cobros y documentos. El motor calcula el saldo en tiempo real. El estado es auto-guardado pero no confirmado. | Auto-guardado en Finanzas |
| **Pago exacto** | Saldo final = 0. Permite avanzar al Paso 3 directamente. | `fccPagoFacturaPedido` / `fccPagoFacturaAdelanto` creados |
| **Sobrepago (Saldo a favor)** | Saldo final > 0. Permite avanzar; registra el excedente en `fccSaldoFavorCliente`. El cobro origen se marca "Saldo a favor". | `fccSaldoFavorCliente` INSERT + `fccPagoCliente` UPDATE |
| **Pago de menos con tolerancia** | Saldo final < 0 y diferencia ≤ 100 MXN. Permite avanzar; la diferencia queda como saldo pendiente en Estado de Cuenta del cliente. | `fccSaldoFavorCliente` (saldo negativo) |
| **Pago insuficiente (inconsistencia)** | Saldo final < 0 y diferencia > 100 MXN. El sistema abre el modal de inconsistencia. Tipos del Paso 2: `PAGO_INCOMPLETO_VENCIDO`, `PAGO_INSUFICIENTE`. | `fccInconsistenciaCobro` + opcionalmente marca `tpPedido` como "Pendiente de cancelación" |
| **Asociación confirmada / Paso 3 habilitado** | El usuario presiona "Continuar". Las asociaciones quedan fijas y ya no son editables. Finanzas invoca el cierre del folio en Buzón. | `fccPagoFacturaPedido` / `fccPagoFacturaAdelanto` persistidos + `PUT /api/cobros/folio/{id}/cerrar` |

> **Perú (RE-027):** La asociación cobro↔factura NO tiene efecto fiscal (no genera Complemento de Pago). Es un registro operativo de conciliación interna. La factura peruana ya fue emitida completa con IGV.

---

## Etapa 5 — Paso 3: Facturación y Envío (RE-028 MX, RE-029 Perú)

**Tabla:** `fccDocumentoFiscalCobro`  
**Catálogo de estados:** `catDocumentoFiscalCobroEstado`

| Estado | Clave | Descripción | Quién lo establece |
|---|---|---|---|
| **Pendiente** | `PENDIENTE` | La línea fue creada al iniciar el Paso 3. El documento fiscal aún no ha sido generado ni timbrado. | Al crear la línea en `fccDocumentoFiscalCobro` (valor inicial) |
| **Generado** | `GENERADO` | El documento fiscal fue timbrado exitosamente. El CFDI fue generado (MX) o el documento fiscal Perú fue emitido. El XML queda en BD y en Minio. | Tras timbrado exitoso — Worker/API de Timbrado actualiza el campo |
| **Enviado** | `ENVIADO` | El documento fue enviado al cliente (correo con PDF + XML). | Tras envío exitoso vía Brevo |

**Vista:** `vfccDocumentoFiscalCobro` expone el campo como `EstadoLinea` (alias del FK resuelto al catálogo).

### Complemento de Pago (RE-030 — solo México)

Aplica cuando el documento fiscal emitido en Paso 3 es una **Factura con pago en parcialidades (PPD)**. Después de registrar el pago efectivo, se genera el Complemento de Pago (CFDI de tipo P).

| Campo | Valor | Descripción |
|---|---|---|
| `fccDocumentoFiscalCobro.EstadoLinea` | `GENERADO` | Se actualiza tras timbrado exitoso del CP (mismo catálogo) |
| `fccDocumentoFiscalCobro.IdCFDIGeneradaComplemento` | UUID del CP | Se pobla con el UUID del CFDI del Complemento de Pago timbrado |

---

## Resumen de Estados por Tabla

| Tabla | Campo de Estado | Valores |
|---|---|---|
| `fccFolioPagoCliente` | `Activo` (bit) | `1` = pendiente activo, `0` = cerrado |
| `fccPagoCliente` | `Confirmado` (bit) | `0` = borrador, `1` = confirmado |
| `fccPagoCliente` | (etiqueta UI) | Sin folio = "COB-N", con folio = "COB-mmddaa-NNNN", con residual = "Saldo a favor" |
| `fccPagoFacturaPedido` / `fccPagoFacturaAdelanto` | (existencia del registro) | No existe = pendiente de asociar, existe = asociado |
| `fccSaldoFavorCliente` | `Saldo` | > 0 = saldo a favor disponible, ≤ 0 = diferencia tolerancia |
| `fccDocumentoFiscalCobro` | `IdCatDocumentoFiscalCobroEstado` | `PENDIENTE` → `GENERADO` → `ENVIADO` |
| `tpPedido` | (campo pendiente de definir) | Normal, "Cancelado por falta de pago", "Pendiente de cancelación" |

---

## Brechas Abiertas Identificadas

| # | Descripción | Requisito |
|---|---|---|
| B1 | Campo exacto en `tpPedido` para "Pendiente de cancelación por falta de pago" no está definido | RE-026 |
| B2 | Mecanismo de transferencia del estado "Pendiente de cancelación" al área de Finanzas no definido | RE-026 |
| B3 | Cancelación desde Gestionar Cobranza: ¿propaga cancelación de proforma/factura y transferencias a Legacy? | RE-023 |
| B4 | Impacto en `tpPedidoVD` / `tpPartidaPedidoVD` al cancelar un pedido tramitado (TaskScheduler podría procesar una OC cancelada) | RE-010/RE-012 |
| B5 | Catálogo completo de tipos de inconsistencia del Paso 1 pendiente de definición por Tesorería | RE-024/025 |
| B6 | Fuente oficial del Tipo de Cambio para Perú (no aplica DOF mexicano) | RE-025/027 |

---

*Generado: 2026-06-18 | Versión 1.0*
