# Impacto en Back — R16A-RE-FU-027
**Requisito:** Validar Cobro: Paso 2 Perú — Asociación
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)
**Módulo:** Validar Cobro — Wizard Paso 2 (Perú)
**Impacto:** Scripts BD ProquifaDotNet (ALTER fccNotaCredito ADD PEN/MontoPEN + verificar fccSaldoFavorCliente PEN) + Endpoints Finanzas: listado proformas/facturas pendientes (GOLPERU), panel asociación N:N, aplicación NCs, cálculo saldo multi-divisa (conversión por TC del cobro, moneda base PEN), escenarios pago (exacto/sobrepago/tolerancia-pendiente/insuficiente), modal inconsistencia Paso 2, auto-guardado + llamadas entre APIs (Finanzas → ProquifaDotNet). **Sin Complemento de Pago — asociación solo operativa.**

---

## Resumen

Este requisito implementa la **segunda pantalla del wizard de Validar Cobro (Paso 2 — Asociación) para Región Perú** en ProquifaDotNet.Finanzas. Es la contraparte directa de RE-FU-026 (México) con una diferencia de fondo: **la asociación cobro↔documento NO tiene efecto fiscal en Perú**. No se genera Complemento de Pago ni se reporta a SUNAT. La factura peruana ya nació completa con su IGV; la conciliación es registro operativo interno de cobranza.

La lógica operativa (asociación N:N, escenarios de pago, saldo a favor, multi-divisa, inconsistencias) es idéntica a México. La empresa emisora es siempre Golocaer S.A.C. (GOLPERU), sin mezcla de emisores.

El impacto en BD es **mínimo**: solo 2 `ALTER TABLE` en `fccNotaCredito` para soportar NCs en soles peruanos (PEN) + verificar si `fccSaldoFavorCliente` necesita campo PEN. Toda la infraestructura de RE-FU-026 se reutiliza. El impacto en servicios (Finanzas) es **medio**: los endpoints son los mismos que México con ajuste de catálogos, moneda base PEN y eliminación del trigger de Complemento de Pago en Paso 3.

### Distribución de responsabilidades

| Capa              | Aplicativo                | Responsabilidad                                                                                        |
| ----------------- | ------------------------- | ------------------------------------------------------------------------------------------------------ |
| BD                | ProquifaDotNet            | ALTER `fccNotaCredito` (PEN + MontoPEN) + verificar `fccSaldoFavorCliente` para PEN                    |
| Tablas asociación | ProquifaDotNet            | `fccPagoFacturaPedido` y `fccPagoFacturaAdelanto` ya existen — se reutilizan igual que en México       |
| Lógica Paso 2     | ProquifaDotNet.Finanzas   | Asociación N:N, aplicación NCs, cálculo saldo, escenarios, multi-divisa (PEN base), saldo a favor      |
| Comunicación      | Finanzas → ProquifaDotNet | Llamadas entre APIs para leer documentos pendientes, NCs vigentes y escribir asociaciones              |
| Paso 3            | ProquifaDotNet.Finanzas   | **Sin Complemento de Pago.** La asociación del Paso 2 no genera documento fiscal ni se reporta a SUNAT |

### Infraestructura reutilizada de RE-FU-026

| Componente                    | Origen                   | Reutilización                                                    |
| ----------------------------- | ------------------------ | ---------------------------------------------------------------- |
| `fccPagoFacturaPedido`        | ProquifaDotNet existente | Asociación cobro ↔ proforma Perú                                 |
| `fccPagoFacturaAdelanto`      | ProquifaDotNet existente | Asociación cobro ↔ Factura por Adelantado Perú                   |
| `fccSaldoFavorCliente`        | RE-FU-026                | Misma tabla, registros con `IdRegion = 'PER'` (o PEN=1)          |
| `catTipoInconsistenciaCobro`  | RE-FU-024 + RE-FU-026    | Mismo catálogo; tipos Paso 2 ya incluyen pagoincompletovencido |
| `fccInconsistenciaCobro`      | RE-FU-024                | Registro de inconsistencias del Paso 2 contra el cobro           |
| `fccPagoCliente.TipoDeCambio` | RE-FU-025                | TC vs PEN (uso local/fiscal Perú)                                |
| `fccPagoCliente.TipoDeCambioMonedaFacturacion` | RE-FU-025 (OBS-050) | **TC vs moneda de facturación** — usado para cobertura del cobro contra facturas/proformas (OBS-052) |

---

## Parte A — Base de Datos (ProquifaDotNet)

### A1 — ALTER TABLE fccNotaCredito — Soporte PEN (Notas de Crédito en soles peruanos)

Agrega soporte para NCs emitidas en soles peruanos (PEN), con el mismo patrón de las banderas MXN/USD existentes.

```sql
-- Agregar soporte para Notas de Crédito en soles peruanos (PEN)
ALTER TABLE dbo.fccNotaCredito
    ADD PEN bit NOT NULL
        CONSTRAINT [DF_fccNotaCredito_PEN] DEFAULT (0);

ALTER TABLE dbo.fccNotaCredito
    ADD MontoPEN decimal(18,4) NULL;
```

| Campo      | Tipo          | Descripción                                   | Analogía         |
| ---------- | ------------- | --------------------------------------------- | ---------------- |
| `PEN`      | bit NOT NULL  | 1 = NC emitida en soles peruanos              | Igual que MXN/USD |
| `MontoPEN` | decimal(18,4) | Monto de la NC en soles                       | Igual que MontoMXN/MontoUSD |

### A2 — ALTER TABLE fccSaldoFavorCliente — Verificar soporte PEN

La tabla `fccSaldoFavorCliente` se crea en RE-FU-026. Si se creó sin campo `PEN`, ejecutar:

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
-- Ejecutar SOLO si la tabla fue creada sin campo PEN en RE-FU-026
ALTER TABLE dbo.fccSaldoFavorCliente
    ADD PEN bit NOT NULL
        CONSTRAINT [DF_fccSaldoFavorCliente_PEN] DEFAULT (0);
```

> ⚠️ **Pendiente verificar** al momento de ejecutar RE-FU-026 si ya se incluyó el campo PEN. Si ya existe, este script no aplica. Se recomienda incluir PEN desde la creación inicial en RE-FU-026 para evitar la dependencia de orden.

---

## Parte B — ProquifaDotNet.Finanzas: Servicios y Endpoints

### B1 — Listado de Proformas y Facturas pendientes del cliente (panel central) — Perú

**Descripción:** Endpoint en Finanzas que retorna todas las proformas y facturas pendientes de cobrar del cliente Perú, mezcladas en un único listado sin filtros adicionales.

**Datos obtenidos:** `tpProformaPedido`, `vfccFactura` (RE-FU-015 — antes: `tpProformaAdelanto`), `Empresa` (GOLPERU) y `catMoneda` vía Scaffold Finanzas (`tpProformaPedido` movida a Finanzas 07/07/2026); `tpPedido` vía API ProquifaDotNet

**Filtros:** `IdCliente = @IdCliente` AND `MontoPendiente > 0` AND `Cancelada = 0` AND `Activo = 1` AND `Region = 'PER'`

**Campos por documento:**

| Campo          | Fuente                                                      |
| -------------- | ----------------------------------------------------------- |
| Tipo           | `PROFORMA` o `FACTURA_ADELANTADA`                           |
| Folio          | `tpProformaPedido.Folio` o `vfccFactura.FolioFactura` (antes: `tpProformaAdelanto.Folio`) |
| PedidoInterno  | `tpPedido`                                                  |
| EmpresaEmisora | Siempre `GOLPERU` (Golocaer S.A.C.) — sin mezcla de emisores |
| ImporteTotal, SaldoPendiente | `tpProformaPedido.MontoPendiente`             |
| ClaveMoneda    | `catMoneda` (usualmente PEN o USD)                          |
| CPEGenerada    | `tpProformaPedido.IdCPEGenerada` (análogo a CFDIGenerada en México) |

> En Perú solo hay un emisor (GOLPERU). No hay mezcla de empresas emisoras como en México.

### B2 — Catálogo de Notas de Crédito vigentes del cliente — Perú

**Descripción:** Endpoint que retorna las NCs vigentes del cliente Perú disponibles para aplicar. Una NC vigente tiene `Aplicada=0` y `Activo=1`.

**Datos obtenidos (vía API ProquifaDotNet):** `fccNotaCredito` WHERE `Aplicada=0 AND Activo=1 AND IdCliente=@Id`

**Campos:** `IdFCCNotaCredito`, `Folio`, `Monto`, `PEN`, `MontoPEN`, `ClaveMoneda`

> ⚠️ ~~La mecánica fiscal de referencia de la NC peruana (catálogo 09 SUNAT, cómo se vincula al CPE original) se desarrolla en RE-FU-033/035 y **no se implementa en este Paso 2**. Aquí solo se aplica operativamente al adeudo.~~
> **[DUDA-087 — Descartada, 2026-08-21]** Se cancela la facturación de Perú en R16; no aplica desarrollar la mecánica de NC peruana (catálogo 09 SUNAT) ni su aplicación al adeudo en este Paso.

### B3 — Cálculo dinámico del saldo de la asociación (motor central del Paso 2) — Perú

**Descripción:** Lógica en Finanzas que calcula en tiempo real el saldo de la asociación. Idéntica a México con moneda base PEN.

**Fórmula:**
```
SaldoAsociacion = (SumaCobrosAplicados + SumaNCAplicadas) - SumaAdeudoDocumentosSeleccionados
```

**Multi-divisa (OBS-052 aplicado a Perú):** Cuando los documentos están en moneda distinta a la del cobro, Finanzas convierte usando el **TC del pago** persistido en `fccPagoCliente` — se prioriza `TipoDeCambioMonedaFacturacion` (OBS-050) para la conversión operativa contra facturas/proformas y `TipoDeCambio` (vs PEN) para el registro local. **NO** se usa el TC de emisión del documento; el diferencial contra ese TC se registra como fluctuación cambiaria del emisor y no se exige al cliente. Todos los totales del panel se expresan en moneda del cobro. Moneda base local: PEN.

**Escenarios gobernados:**

| Resultado SaldoAsociacion        | Escenario      | Acción del sistema                                                               |
| -------------------------------- | -------------- | -------------------------------------------------------------------------------- |
| = 0                              | Pago exacto    | Permite avanzar al Paso 3                                                        |
| > 0                              | Sobrepago      | Permite avanzar; registra saldo a favor en `fccSaldoFavorCliente` (PEN=1)        |
| < 0 AND ABS ≤ T (umbral Perú)    | Tolerancia     | Permite avanzar; registra `ToleranciaAplicada` en `fccSaldoFavorCliente` (PEN=1) |
| < 0 AND ABS > T (umbral Perú)    | Insuficiente   | Bloquea avance; requiere inconsistencia o dejar pendiente                        |

> ⚠️ ~~Umbral de tolerancia Perú (T) pendiente de definir con PROQUIFA Tesorería. En México es 100 MXN (Política Interna). Para Perú: monto, moneda y tratamiento cuando la facturación no es PEN están sin confirmar.~~
> **[DUDA-086 — Resuelta, 2026-08-21]** Se aplica la MISMA regla de tolerancia que México (tolerancia equivalente para Perú); los montos límite deben ser configurables a nivel BD.

### B4 — Persistencia de la asociación en ProquifaDotNet — Perú

**Descripción:** Endpoint transaccional en Finanzas que persiste la asociación cobro↔documento(s). Igual que México; sin trigger de Complemento de Pago.

**Operaciones en ProquifaDotNet:**
- `INSERT fccPagoFacturaPedido` por cada cobro↔proforma normal Perú.
- `INSERT fccPagoFacturaAdelanto` por cada cobro↔FAA Perú.
- `UPDATE fccNotaCredito SET Aplicada=1, IdFCCPagoCliente=@IdCobro` por cada NC aplicada.
- `INSERT fccSaldoFavorCliente` si hay sobrepago (`TipoSaldo='SaldoFavor'`, `PEN=1`) o tolerancia (`TipoSaldo='ToleranciaAplicada'`, `PEN=1`).
- `UPDATE fccPagoCliente` para marcar el cobro con identificador "saldo a favor" si aplica.
- `UPDATE tpProformaPedido SET MontoPendiente` al asociar.

Todo en una sola transacción; rollback completo si cualquier operación falla.

**Diferencia clave vs México:** Al avanzar al Paso 3, **no se dispara generación de Complemento de Pago**. La asociación queda registrada operativamente y el Paso 3 Perú (RE-FU-029) continuará el flujo sin emisión de documento fiscal adicional.

### B5 — Modal de inconsistencia del cobro (Paso 2) — Perú

**Descripción:** Endpoint en Finanzas que registra la inconsistencia del Paso 2. Idéntico a México (RE-FU-026/B5); los tipos del catálogo y la lógica de `pagoincompletovencido` son los mismos.

**Tipos del Paso 2 (catálogo `catTipoInconsistenciaCobro` con `AplicaPaso='2'`):**
- `pagoincompletovencido` → habilita opción de marcar pedido como "Pendiente de cancelación" (`AplicaMarkPendienteCancelacion=1`)
- `pagoinsuficiente` → registra inconsistencia, deja asociación pendiente de próximo cobro

**Flujo para `pagoincompletovencido` en Perú:**
1. Usuario selecciona el tipo y confirma.
2. Finanzas detecta `AplicaMarkPendienteCancelacion=1` en el catálogo.
3. Se habilita la opción de marcar el pedido como "Pendiente de cancelación por falta de pago".
4. Si el usuario confirma: `UPDATE tpPedido SET EstadoCancelacion = 'PENDIENTE_CANCELACION_FALTA_PAGO'`.
5. No se ejecuta cancelación fiscal ni devolución.

> ⚠️ En Perú la cancelación fiscal efectiva se realizaría vía Nota de Crédito SUNAT — ver RE-FU-033/035. El mecanismo de transferencia del estado "Pendiente de cancelación" para gestión externa está pendiente de definir.

### B6 — Auto-guardado del Paso 2 — Perú

**Descripción:** Mismo servicio de auto-guardado que México (RE-FU-026/B6). Persiste el estado de asociaciones en progreso de forma transparente. Las asociaciones son editables mientras el usuario esté en el Paso 2; se fijan al avanzar al Paso 3.

---

## Brechas

> ⚠️ ~~BRECHA BLOQUEANTE — Umbral de tolerancia Perú (B1)~~ **[DUDA-086 — Resuelta, 2026-08-21: cerrada]**
> ~~El umbral equivalente a los 100 MXN de México no está definido para Perú: monto, moneda y tratamiento cuando la facturación no es PEN. Sin él, el escenario de pago de menos no puede gobernarse automáticamente en el Paso 2.~~ Se aplica la MISMA regla que México (tolerancia equivalente); los montos límite deben ser configurables a nivel BD.

> ⚠️ **BRECHA BLOQUEANTE — Fuente del TC para Perú (B2)**
> El TC capturado en el Paso 1 Perú (RE-FU-025) no tiene fuente oficial definida. No aplica el DOF mexicano. Posiblemente SBS (Superintendencia de Banca y Seguros del Perú), pendiente confirmar. Es brecha transversal: afecta RE-FU-017, 020, 022, 025 y 027.

> ⚠️ ~~BRECHA BLOQUEANTE — Mecánica fiscal NC peruana catálogo 09 SUNAT (B3)~~ **[DUDA-087 — Descartada, 2026-08-21: cerrada por no aplicabilidad]**
> ~~La aplicación de NCs al adeudo en el Paso 2 es operativa, pero el referenciado fiscal de la NC peruana (catálogo 09, vinculación al CPE original) se define en RE-FU-033/035. Hasta que se resuelva, la aplicación de NCs en Perú no puede implementarse completamente.~~ Se cancela la facturación de Perú en R16; no aplica desarrollar esta mecánica ni la aplicación de NCs en el Paso 2.

> ⚠️ **BRECHA MEDIA — Mecanismo de transferencia "Pago Incompleto Vencido" (B4)**
> El marcado del pedido como "Pendiente de cancelación" notifica para gestión externa. El canal de transferencia del estado a Finanzas/Tesorería no está definido.

> ⚠️ **BRECHA — Catálogo de Tipos de Inconsistencia (transversal)**
> El catálogo completo de tipos está pendiente de definición por PROQUIFA Tesorería. Brecha transversal con México (RE-FU-026).

> ⚠️ **BRECHA — Maquetas Validar Cobro Perú no disponibles**
> La pantalla se especifica como "UI idéntica a México" pero sin maquetas Perú que confirmen el detalle visual. Pendiente validar cuando lleguen.
