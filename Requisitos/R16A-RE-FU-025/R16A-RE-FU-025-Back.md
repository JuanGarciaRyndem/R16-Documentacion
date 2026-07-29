# Impacto en Back — R16A-RE-FU-025
**Requisito:** Validar Cobro: Paso 1 Perú — Captura del Cobro
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)
**Módulo:** Validar Cobro — Wizard Paso 1 (Perú)
**Impacto:** Scripts BD ProquifaDotNet (DML: INSERT catMedioDePago Perú + INSERT cuentas bancarias GOLPERU — ambos BRECHA) + Endpoints Finanzas: reutilización completa de RE-FU-024 con variantes de catálogos Perú (medio de pago interno, cuentas GOLPERU, TC vs PEN) + llamadas entre APIs (Finanzas → ProquifaDotNet)

---

## Resumen

Este requisito implementa el **Paso 1 del wizard de Validar Cobro para Región Perú** en ProquifaDotNet.Finanzas. La **estructura UI y los endpoints son idénticos a México (RE-FU-024)**; las diferencias son exclusivamente los catálogos (medio de pago interno sin ClaveFormaDePago SAT, cuentas bancarias de Golocaer S.A.C. Perú, TC vs PEN con fuente peruana) y el identificador fiscal (RUC en lugar de RFC). La vinculación posterior del cobro con una factura en el Paso 3 **NO genera Complemento de Pago** (SUNAT no tiene este mecanismo); la relación cobro↔factura es solo operativa interna.

El impacto en DDL de BD es **CERO** (todo el esquema fue creado en RE-FU-024). El impacto en DML es **BRECHA**: los catálogos de medio de pago Perú y las cuentas bancarias de Golocaer S.A.C. están pendientes de definición por PROQUIFA. El impacto en Finanzas es la **variante Perú** de los servicios ya implementados en RE-FU-024.

### Distribución de responsabilidades

| Capa               | Aplicativo                | Responsabilidad                                                             |
| ------------------ | ------------------------- | --------------------------------------------------------------------------- |
| BD                 | ProquifaDotNet            | Sin DDL nuevo. Solo DML: INSERT catálogos Perú (BRECHA)                     |
| API Datos          | ProquifaDotNet            | Mismos endpoints de RE-FU-024 — retornan catálogos según región del cliente |
| Lógica Paso 1 Perú | ProquifaDotNet.Finanzas   | Variante Perú: medio de pago interno, cuentas GOLPERU, TC vs PEN            |
| TC del día         | ProquifaDotNet.Finanzas   | Nuevo servicio `TipoCambioPeruService` (fuente peruana — BRECHA)            |
| Comunicación       | Finanzas → ProquifaDotNet | Llamadas entre APIs — mismos patrones RE-FU-024                             |

### Reutilización de RE-FU-024

| Componente | Creado en | Uso en Perú |
|------------|-----------|-------------|
| `fccPagoCliente` + campos nuevos (Confirmado, Notas, etc.) | RE-FU-024 | Misma tabla — mismo flujo |
| `catTipoInconsistenciaCobro` | RE-FU-024 | Mismo catálogo (AplicaPaso='1') |
| `fccInconsistenciaCobro` | RE-FU-024 | Misma tabla — cobros PER |
| `SeqFolioCobro` | RE-FU-024 | Pendiente confirmar: global o por región |
| Endpoints Finanzas (listado cobros, detalle correo, auto-guardado, confirmar, modal inconsistencia) | RE-FU-024 | Reutilizados con parámetro de región o catálogos distintos |

### Diferencias México vs Perú en catálogos

| Aspecto              | México — RE-FU-024                                | Perú — RE-FU-025                                              |
| -------------------- | ------------------------------------------------- | ------------------------------------------------------------- |
| Forma/Medio de pago  | `catMedioDePago.ClaveFormaDePago` SAT c_FormaPago | Catálogo interno PROQUIFA sin `ClaveFormaDePago` (**BRECHA**) |
| Cuenta destino       | `EmpresaDatosBancarios` GOL/MUN/PRO/PQF           | `EmpresaDatosBancarios` GOLPERU (**BRECHA — 0 registros**)    |
| Moneda base TC       | MXN (DOF/Banxico)                                 | PEN (fuente pendiente — **BRECHA**)                           |
| ID fiscal cliente    | RFC (13 chars)                                    | RUC (11 dígitos)                                              |
| Efecto fiscal Paso 3 | Genera Complemento de Pago CFDI                   | **SIN Complemento de Pago (solo operativo)**                  |

---

## Parte A — Base de Datos (ProquifaDotNet): Solo DML — BRECHAS

> **Sin cambios DDL.** Todo el esquema fue creado en RE-FU-024.
> Los cambios son de datos (DML) que dependen de información pendiente de PROQUIFA Perú.

### A1 — INSERT catMedioDePago Perú (BRECHA)

SUNAT no exige declarar el medio de pago en el comprobante; el catálogo es interno de PROQUIFA para control de Tesorería.

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
-- Catálogo interno medio de pago Perú (ClaveFormaDePago = NULL — no es catálogo SAT)
-- Opciones pendientes de confirmar con PROQUIFA Tesorería
INSERT INTO [dbo].[catMedioDePago]
    ([MedioDePago], [RequiereNumeroDeCuenta], [ObligatorioEnProveedor], [ClaveFormaDePago], [Clave], [ObligatorioEnCliente])
VALUES
    (N'Transferencia bancaria', 1, 0, NULL, 'PER-TRF', 0),
    (N'Depósito bancario',      1, 0, NULL, 'PER-DEP', 0),
    (N'Cheque',                 0, 0, NULL, 'PER-CHQ', 0),
    (N'Efectivo',               0, 0, NULL, 'PER-EFE', 0);
-- Ajustar opciones al catálogo real que defina PROQUIFA Tesorería
```

> ⚠️ `catMedioDePago.ClaveFormaDePago` es NULLABLE — soporta registros sin clave SAT. No requiere ALTER TABLE.

### A2 — INSERT Cuentas Bancarias GOLPERU (BRECHA)

```sql
-- Estructura a poblar (datos pendientes de PROQUIFA Perú):
-- 1. INSERT catBanco (si los bancos peruanos de Golocaer S.A.C. no existen)
-- 2. INSERT DatosBancarios (NumeroDeCuenta, Clabe→CCI 20 dígitos, Sucursal, IdCatBanco, IdCatMoneda PEN/USD)
-- 3. INSERT EmpresaDatosBancarios (IdEmpresa→GOLPERU, IdRegion→PER, IdDatosBancarios)
-- Verificar que catMoneda PEN ya existe; si no, INSERT catMoneda PEN.
```

> ⚠️ Validación en BD: `EmpresaDatosBancarios WHERE Empresa.Prefijo='GOLPERU' AND Activo=1 = 0 registros`. Bloqueante para el campo "Cuenta destino" del formulario del Paso 1 Perú.

---

## Parte B — ProquifaDotNet.Finanzas: Variante Perú del Paso 1

### B1 — Servicio TC del día para Perú (TipoCambioPeruService)

**Descripción:** Servicio paralelo a `TipoCambioService` (RE-FU-024) pero para Región Perú. Calcula el TC del día de la moneda no-PEN involucrada respecto a PEN.

**Lógica:**
| Moneda cobro | Moneda facturación cliente | TC a capturar |
|---|---|---|
| PEN | PEN | N/A |
| PEN | Distinta a PEN | TC del día de la moneda de facturación vs PEN |
| Distinta a PEN | Cualquiera | TC del día de la moneda del cobro vs PEN |

**Fuente:** Pendiente de definir para Perú (no aplica TC FIX Banxico/DOF mexicano).
> ⚠️ **BRECHA BLOQUEANTE:** sin fuente del TC definida, el servicio no puede implementarse completamente.

### B2 — Adaptación de catálogos en formulario del Paso 1 para Perú

Los endpoints del Paso 1 ya implementados en RE-FU-024 se adaptan para Perú mediante el parámetro de región del cliente:

| Campo del formulario | México | Perú |
|---|---|---|
| Medio de pago | `catMedioDePago` con `ClaveFormaDePago IS NOT NULL` (SAT) | `catMedioDePago` con `ClaveFormaDePago IS NULL` (interno) |
| Cuenta destino | `EmpresaDatosBancarios` donde empresa es GOL/MUN/PRO/PQF | `EmpresaDatosBancarios` donde empresa es GOLPERU |
| TC del día | `TipoCambioService` (vs MXN) | `TipoCambioPeruService` (vs PEN) |
| ID fiscal en cabecera | RFC | RUC |

### B3 — Vinculación cobro↔factura sin efecto fiscal en Perú

El cobro capturado en el Paso 1 se vinculará en el Paso 2 a un documento (factura). En Perú esa vinculación **NO genera Complemento de Pago** ni se reporta a SUNAT (la factura peruana ya se emitió completa con su IGV). La vinculación es solo un registro operativo interno de conciliación de cobranza. El Paso 3 para Perú no genera documento fiscal adicional.

> Esta naturaleza impacta el diseño del Paso 2 Perú (RE-FU-027) y del Paso 3 Perú (RE-FU-029), no del Paso 1.

---

## Brechas

> ⛔ **BRECHA BLOQUEANTE — Catálogo interno de medio de pago Perú**
> Los valores del catálogo interno de medio de pago para Perú están pendientes de definición con PROQUIFA Tesorería. Sin este catálogo, el combo del formulario Paso 1 Perú queda vacío.

> ⛔ **BRECHA BLOQUEANTE — Cuentas bancarias GOLPERU (0 registros)**
> Las cuentas bancarias de Golocaer S.A.C. Perú no están en BD. Sin estos datos, el combo "Cuenta destino" del formulario Paso 1 Perú queda vacío. Depende de R16A-RE-FU-006.

> ⛔ **BRECHA BLOQUEANTE — Fuente del TC para Perú**
> La fuente oficial del TC del día para Perú no está definida. Sin ella, `TipoCambioPeruService` no puede implementarse completamente.

> ⚠️ **BRECHA — Catálogo de Tipos de Inconsistencia del Paso 1 (transversal con México)**
> Mismo catálogo que RE-FU-024. Pendiente catálogo completo de PROQUIFA Tesorería.

> ⚠️ **BRECHA — Foliador global vs por región**
> Pendiente confirmar si `SeqFolioCobro` es compartido MEX+PER o independiente por región.
