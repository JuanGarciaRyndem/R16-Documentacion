# Impacto en BD - Validar Cobro: Paso 2 Peru (Asociacion)
**Requisito:** TPSC-RE-FU-027
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
Paso 2 wizard VC Peru: asociacion N:N cobros <-> proformas/facturas.
UI identica a MEX (RE-FU-026). Diferencia clave: SIN Complemento de Pago,
asociacion es solo operativa (conciliacion interna). Empresa emisora unica: GOLPERU.
Tolerancia pendiente de definir (equivalente a 100 MXN de MEX).

---

## Impacto en BD: MINIMO (reutiliza RE-FU-026)

| #   | Cambio                                                         | Tipo                    | Prioridad |
| --- | -------------------------------------------------------------- | ----------------------- | --------- |
| 1   | ALTER TABLE fccNotaCredito ADD PEN bit NOT NULL DEFAULT(0)     | DDL                     | Media     |
| 2   | ALTER TABLE fccNotaCredito ADD MontoPEN decimal NULL           | DDL                     | Media     |
| 3   | Reutiliza: fccPagoFacturaPedido (cobro <-> Proforma PER)       | Existente               | -         |
| 4   | Reutiliza: fccPagoFacturaAdelanto (cobro <-> FAA PER)          | Existente               | -         |
| 5   | Reutiliza: fccSaldoFavorCliente (RE-FU-026)                    | Existente (post RE-026) | -         |
| 6   | Reutiliza: catTipoInconsistenciaCobro + fccInconsistenciaCobro | Existente               | -         |
| 7   | Reutiliza: fccNotaCredito.Aplicada                             | Existente               | -         |

> **Sin nuevas tablas.** La infraestructura de RE-FU-026 cubre Peru.
> Solo se agrega soporte PEN en fccNotaCredito para NCs de clientes Peru.

---

## Diferencias MEX (RE-FU-026) vs PER (RE-FU-027) en BD

| Aspecto                    | Mexico                            | Peru                                             |
| -------------------------- | --------------------------------- | ------------------------------------------------ |
| Efecto fiscal Paso 3       | Genera Complemento de Pago (CFDI) | **SIN efecto fiscal (solo operativo)**           |
| Empresa emisora documentos | GOL/MUN/PRO/PQF (4 opciones)      | GOLPERU (siempre)                                |
| Moneda base conversiones   | MXN                               | **PEN**                                          |
| Moneda NC                  | MXN/USD                           | MXN/USD/**PEN**                                  |
| Tolerancia pago de menos   | 100 MXN (confirmada)              | **Pendiente de definir**                         |
| Cancelacion fiscal         | Via CFDI cancelacion SAT          | **Via Nota de Credito SUNAT**                    |
| Estado de Cuenta           | fccSaldoFavorCliente (MEX)        | fccSaldoFavorCliente (misma tabla, IdRegion=PER) |

---

## ALTER TABLE fccNotaCredito (soporte PEN)

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Agregar soporte para Notas de Credito en soles peruanos (PEN)
    ALTER TABLE dbo.fccNotaCredito
        ADD PEN bit NOT NULL
            CONSTRAINT [DF_fccNotaCredito_PEN] DEFAULT (0);

    ALTER TABLE dbo.fccNotaCredito
        ADD MontoPEN decimal(18,4) NULL;

| Campo | Descripcion | Analogia |
|-------|-------------|----------|
| PEN | bit: 1=NC en soles peruanos | Igual que MXN/USD |
| MontoPEN | Monto en soles | Igual que MontoMXN/MontoUSD |

---

## fccSaldoFavorCliente para Peru

La tabla fccSaldoFavorCliente (creada en RE-FU-026) ya soporta PER porque:

    fccSaldoFavorCliente
        IdCliente -> Cliente (tiene region via ClienteCarteraCliente)
        MXN/USD -> moneda del saldo (agregar PEN si falta)

**Verificar si necesita campo PEN:**

    -- Si la tabla fue creada sin campo PEN, ejecutar:
    ALTER TABLE dbo.fccSaldoFavorCliente
        ADD PEN bit NOT NULL CONSTRAINT [DF_fccSaldoFavorCliente_PEN] DEFAULT (0);

> Pendiente confirmar al ejecutar RE-FU-026 si ya incluye campo PEN.

---

## Tablas Leidas (Lectura) - Mismo patron que MEX

| Tabla | Datos leidos | Diferencia vs MEX |
|-------|-------------|-------------------|
| fccPagoCliente | Folio, Monto, FechaPago, PEN, TipoDeCambio, Confirmado | PEN vs MXN |
| fccSaldoFavorCliente | Monto, TipoSaldo, PEN, Aplicado (cliente PER) | Misma tabla, region PER |
| tpProformaPedido | MontoPendiente, Folio, IdCPEGenerada | CPEGenerada vs CFDIGenerada |
| fccPagoFacturaPedido | Monto aplicado previo | Igual |
| fccNotaCredito | Monto, PEN, Aplicada=0 | PEN nuevo |
| catTipoInconsistenciaCobro | AplicaPaso='2' | Igual |
| Empresa (GOLPERU) | Prefijo emisor (siempre GOLPERU) | Solo 1 empresa |

---

## Tablas Escritas (runtime) - Mismo patron que MEX

| Tabla | Momento | Operacion |
|-------|---------|-----------|
| fccPagoFacturaPedido | Al confirmar asociacion (Proforma) | INSERT |
| fccPagoFacturaAdelanto | Al confirmar asociacion (FAA) | INSERT |
| fccNotaCredito | Al aplicar NC | UPDATE Aplicada=1, IdFCCPagoCliente=@id |
| fccSaldoFavorCliente | Sobrepago | INSERT TipoSaldo='SaldoFavor', PEN=1 |
| fccSaldoFavorCliente | Tolerancia PER | INSERT TipoSaldo='ToleranciaAplicada', PEN=1 |
| tpProformaPedido | Al asociar | UPDATE MontoPendiente |
| fccInconsistenciaCobro | Al marcar inconsistencia | INSERT |
| tpPedido | Pago Incompleto Vencido | UPDATE FechaCancelacionPorFaltaPago |

---

## Calculo del Saldo (mismo que MEX, moneda base PEN)

    Adeudo = SUM(tpProformaPedido.MontoPendiente) de docs seleccionados
    Cobros = SUM(fccPagoCliente.Monto) de cobros seleccionados (a moneda cobro via TC)
    NCs    = SUM(fccNotaCredito.Monto) de NCs aplicadas (a moneda cobro via TC)
    Saldo  = (Cobros + NCs) - Adeudo

    SI Saldo = 0:   Exacto -> avanza Paso 3
    SI Saldo > 0:   Sobrepago -> fccSaldoFavorCliente + avanza Paso 3
    SI -T <= Saldo < 0: Tolerancia -> fccSaldoFavorCliente + avanza Paso 3
    SI Saldo < -T:  Bloquea Paso 3, requiere inconsistencia
    (T = umbral tolerancia Peru - PENDIENTE DE DEFINIR)

---

## Brechas Criticas

| # | Brecha | Bloqueante | Accion |
|---|--------|-----------|--------|
| B1 | Tolerancia pago de menos Peru (monto + moneda) | SI | Confirmar con PROQUIFA Tesoreria |
| B2 | Fuente TC Peru (no DOF) | SI | Confirmar fuente (SBS?) |
| B3 | Mecanica fiscal NC peruana cat.09 SUNAT | SI | RE-FU-033/035 |
| B4 | Mecanismo transferencia 'Pago Incompleto Vencido' | Media | Definir canal |

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | Tolerancia Peru: monto + moneda + tratamiento | Negocio | Confirmar Tesoreria |
| 2 | fccSaldoFavorCliente necesita campo PEN | Tecnico | Verificar al ejecutar RE-FU-026 |
| 3 | NC peruana: referencia SUNAT cat.09 | Fiscal | RE-FU-033/035 |
| 4 | Mecanismo cancelacion pedido Peru (NC SUNAT) | Tecnico | RE-FU-033/035 |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| TPSC-RE-FU-025 | Paso 1 PER (fccPagoCliente confirmado) |
| TPSC-RE-FU-026 | Paso 2 MEX (fccSaldoFavorCliente, catTipoInconsistenciaCobro) |
| TPSC-RE-FU-033/035 | NCs Peru (mecanica SUNAT cat.09) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
