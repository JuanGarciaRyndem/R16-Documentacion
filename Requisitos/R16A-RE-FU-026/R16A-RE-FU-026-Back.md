# Impacto en Back — R16A-RE-FU-026
**Requisito:** Validar Cobro: Paso 2 México — Asociación
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)
**Módulo:** Validar Cobro — Wizard Paso 2 (México)
**Impacto:** Scripts BD ProquifaDotNet (CREATE fccSaldoFavorCliente + ALTER fccNotaCredito + ALTER catTipoInconsistenciaCobro) + Endpoints Finanzas: listado proformas/facturas pendientes, panel asociación N:N, aplicación NCs, cálculo saldo multi-divisa (conversión por TC del cobro), escenarios pago (exacto/sobrepago/tolerancia/insuficiente), modal inconsistencia Paso 2, auto-guardado + llamadas entre APIs (Finanzas → ProquifaDotNet)

---

## Resumen

Este requisito implementa la **segunda pantalla del wizard de Validar Cobro (Paso 2 - Asociación) para Región México** en ProquifaDotNet.Finanzas. El usuario asocia cobros capturados en el Paso 1 con proformas/facturas pendientes del cliente en relación N:N, puede aplicar opcionalmente Notas de Crédito vigentes, y el sistema calcula el saldo resultante gobernando cuatro escenarios: pago exacto, sobrepago (saldo a favor), pago de menos con tolerancia ≤100 MXN, y pago insuficiente >100 MXN (inconsistencia). Una vez cerrada la asociación, el flujo avanza al Paso 3 para emitir documentos fiscales.

El impacto en BD (ProquifaDotNet) es **moderado**: 1 tabla nueva (`fccSaldoFavorCliente`) + 1 ALTER en `fccNotaCredito` + 1 ALTER en `catTipoInconsistenciaCobro`. Las tablas de asociación cobro↔documento (`fccPagoFacturaPedido`, `fccPagoFacturaAdelanto`) ya existen. El impacto en servicios (Finanzas) es **alto**: orquestación completa del Paso 2 con lógica multi-divisa.

### Distribución de responsabilidades

| Capa              | Aplicativo                | Responsabilidad                                                                            |
| ----------------- | ------------------------- | ------------------------------------------------------------------------------------------ |
| BD                | ProquifaDotNet            | `fccSaldoFavorCliente` (nueva), ALTER `fccNotaCredito`, ALTER `catTipoInconsistenciaCobro` |
| Tablas asociación | ProquifaDotNet            | `fccPagoFacturaPedido` y `fccPagoFacturaAdelanto` ya existen — se usan en Paso 2           |
| Lógica Paso 2     | ProquifaDotNet.Finanzas   | Asociación N:N, aplicación NCs, cálculo saldo, escenarios, multi-divisa, saldo a favor     |
| Comunicación      | Finanzas → ProquifaDotNet | Llamadas entre APIs para leer documentos pendientes, NCs vigentes y escribir asociaciones  |

### Infraestructura reutilizada

| Componente                                 | Origen                   | Reutilización                                               |
| ------------------------------------------ | ------------------------ | ----------------------------------------------------------- |
| `fccPagoFacturaPedido`                     | ProquifaDotNet existente | Asociación cobro ↔ proforma normal                          |
| `fccPagoFacturaAdelanto`                   | ProquifaDotNet existente | Asociación cobro ↔ Factura por Adelantado                   |
| `fccNotaCredito` (Aplicada, IdCFDI, Monto) | ProquifaDotNet existente | Catálogo de NCs vigentes del cliente                        |
| `catTipoInconsistenciaCobro`               | RE-FU-024                | Se extiende con campo `AplicaMarkPendienteCancelacion`      |
| `fccInconsistenciaCobro`                   | RE-FU-024                | Registro de inconsistencias del Paso 2 contra el cobro      |
| `fccPagoCliente.TipoDeCambio`              | RE-FU-024                | TC vs MXN (uso fiscal — CFDI/CP)                            |
| `fccPagoCliente.TipoDeCambioMonedaFacturacion` | RE-FU-024 (OBS-050)  | **TC vs moneda de facturación** — TC usado para el cálculo de cobertura del cobro contra proformas/facturas (OBS-052) |

---

## Parte A — Base de Datos (ProquifaDotNet)

### A1 — CREATE TABLE fccSaldoFavorCliente (Estado de Cuenta / Auxiliar)

Registra saldo a favor del cliente (sobrepago) y diferencias por tolerancia 100 MXN, para uso en futuras sesiones de Validar Cobro.

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
CREATE TABLE [dbo].[fccSaldoFavorCliente](
    [IdFCCSaldoFavorCliente]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_fccSaldoFavorCliente_Id] DEFAULT (NEWID()),
    [IdCliente]               uniqueidentifier NOT NULL,
    [IdFCCPagoCliente]        uniqueidentifier NOT NULL,   -- cobro origen del saldo
    [TipoSaldo]               varchar(30)      NOT NULL,   -- 'SaldoFavor' | 'ToleranciaAplicada'
    [Monto]                   decimal(18,4)    NOT NULL,
    [MXN]                     bit              NOT NULL CONSTRAINT [DF_fccSaldoFavor_MXN] DEFAULT (0),
    [USD]                     bit              NOT NULL CONSTRAINT [DF_fccSaldoFavor_USD] DEFAULT (0),
    [TipoCambio]              decimal(18,6)    NULL,
    [Aplicado]                bit              NOT NULL CONSTRAINT [DF_fccSaldoFavor_Aplicado] DEFAULT (0),
    [IdFCCPagoFacturaPedido]  uniqueidentifier NULL,       -- cuando se aplica a futura proforma
    [Observaciones]           varchar(500)     NULL,
    [Activo]                  bit              NOT NULL CONSTRAINT [DF_fccSaldoFavor_Activo] DEFAULT (1),
    [FechaRegistro]           datetime     NOT NULL
        CONSTRAINT [DF_fccSaldoFavor_FechaReg] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion] datetime    NOT NULL
        CONSTRAINT [DF_fccSaldoFavor_FechaUpd] DEFAULT (GETDATE()),
    CONSTRAINT [PK_fccSaldoFavorCliente]
        PRIMARY KEY CLUSTERED ([IdFCCSaldoFavorCliente]),
    CONSTRAINT [FK_fccSaldoFavor_Cliente]
        FOREIGN KEY ([IdCliente]) REFERENCES dbo.Cliente([IdCliente]),
    CONSTRAINT [FK_fccSaldoFavor_PagoCliente]
        FOREIGN KEY ([IdFCCPagoCliente]) REFERENCES dbo.fccPagoCliente([IdFCCPagoCliente])
);
```

| TipoSaldo | Cuándo se genera |
|-----------|-----------------|
| `SaldoFavor` | Cobros + NCs > Adeudo (sobrepago) |
| `ToleranciaAplicada` | Cobros + NCs < Adeudo AND diferencia ≤ 100 MXN |

### A2 — ALTER TABLE fccNotaCredito — Vínculo con cobro

Permite identificar en qué sesión de cobro se aplicó la NC para trazabilidad.

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
ALTER TABLE dbo.fccNotaCredito
    ADD IdFCCPagoCliente uniqueidentifier NULL
        CONSTRAINT [FK_fccNotaCredito_PagoCliente]
            FOREIGN KEY REFERENCES dbo.fccPagoCliente([IdFCCPagoCliente]);
```

### A3 — ALTER TABLE catTipoInconsistenciaCobro — Campo para marcar pedido cancelación

Extiende el catálogo (creado en RE-FU-024) con un flag que indica si el tipo de inconsistencia activa la opción de marcar el pedido como "Pendiente de cancelación por falta de pago".

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
ALTER TABLE dbo.catTipoInconsistenciaCobro
    ADD AplicaMarkPendienteCancelacion bit NOT NULL
        CONSTRAINT [DF_catTipoInconsistenciaCobro_Mark] DEFAULT (0);

-- Activar el flag solo para el tipo Pago Incompleto Vencido
UPDATE dbo.catTipoInconsistenciaCobro
SET AplicaMarkPendienteCancelacion = 1
WHERE Clave = 'PAGO_INCOMPLETO_VENCIDO';
```

---

## Parte B — ProquifaDotNet.Finanzas: Servicios y Endpoints

### B1 — Listado de Proformas y Facturas pendientes del cliente (panel central)

**Descripción:** Endpoint en Finanzas que retorna todas las proformas y facturas pendientes de cobrar del cliente, mezcladas en un único listado sin filtros adicionales.

**Datos obtenidos:** `tpProformaPedido`, `vfccFactura` (RE-FU-015 — antes: `tpProformaAdelanto`), `Empresa` y `catMoneda` vía Scaffold Finanzas (`tpProformaPedido` movida a Finanzas 07/07/2026); `tpPedido` vía API ProquifaDotNet

**Filtros:** `IdCliente = @IdCliente` AND `MontoPendiente > 0` AND `Cancelada = 0` AND `Activo = 1`

**Campos por documento:**

| Campo | Fuente |
|-------|--------|
| Tipo | `PROFORMA` o `FACTURA_ADELANTADA` |
| Folio | `tpProformaPedido.Folio` o `vfccFactura.FolioFactura` (antes: `tpProformaAdelanto.Folio`) |
| PedidoInterno | `tpPedido` |
| EmpresaEmisora | `Empresa.Alias` (puede ser GOL, MUN, PRO, PQF) |
| ImporteTotal, SaldoPendiente | `tpProformaPedido.MontoPendiente` |
| ClaveMoneda | `catMoneda` |

> Pueden mezclarse documentos de diferentes empresas emisoras del grupo en la misma sesión. Cada documento se procesa independientemente en el Paso 3.

### B2 — Catálogo de Notas de Crédito vigentes del cliente

**Descripción:** Endpoint que retorna las NCs vigentes del cliente disponibles para aplicar en el Paso 2. Una NC vigente tiene `Aplicada=0` y `Activo=1`.

**Datos obtenidos (vía API ProquifaDotNet):** `fccNotaCredito` WHERE `Aplicada=0 AND Activo=1 AND IdCliente=@Id`

**Campos:** `IdFCCNotaCredito`, `Folio`, `IdCFDI` (UUID timbrado), `Monto`, `MXN/USD`, `ClaveMoneda`

> La aplicación de NCs es OPCIONAL. El usuario selecciona cuáles aplica; las no seleccionadas permanecen vigentes para sesiones futuras.

### B3 — Cálculo dinámico del saldo de la asociación (motor central del Paso 2)

**Descripción:** Lógica en Finanzas que calcula en tiempo real el saldo de la asociación a medida que el usuario selecciona cobros, documentos y NCs. **No es un endpoint independiente**; es el motor de cálculo invocado por los endpoints de selección/deselección.

**Fórmula:**
```
SaldoAsociacion = (SumaCobrosAplicados + SumaNCAplicadas) - SumaAdeudoDocumentosSeleccionados
```

**Multi-divisa (OBS-052):** Cuando los documentos están en moneda distinta a la moneda del cobro, Finanzas convierte usando el **TC del pago** persistido en el cobro — **NO** el TC de emisión del documento. Regla concreta:

- Si el cobro es MXN y el documento está en moneda de facturación ≠ MXN → convertir el monto MXN del cobro a la moneda del documento con `fccPagoCliente.TipoDeCambioMonedaFacturacion` (OBS-050).
- Si el cobro es en moneda distinta a MXN → convertir el importe del documento a moneda del cobro con `fccPagoCliente.TipoDeCambio` (vs MXN) y el TC del documento, o usar `TipoDeCambioMonedaFacturacion` directamente cuando la moneda de facturación coincide con la del documento.

**Fundamento legal/fiscal:** Art. 8 Ley Monetaria EUM + Guía CFDI 4.0 / CRP 2.0 (`EquivalenciaDR` se determina con el TC del pago). La diferencia contra el TC de emisión del documento **NO** se cobra al cliente ni genera saldo pendiente — se registra como **fluctuación cambiaria** (pérdida/ganancia) del emisor conforme a LISR / NIF B-15. El motor NO debe exigir el diferencial cambiario como saldo insoluto.

Todos los totales del panel se expresan en moneda del cobro. El TC aplicado se expone vía tooltip en el Front, indicando fuente (`TipoDeCambioMonedaFacturacion` u `TipoDeCambio`) y fecha (`FechaPago` del cobro).

**Escenarios gobernados:**
| Resultado SaldoAsociacion | Escenario | Acción del sistema |
|---|---|---|
| = 0 | Pago exacto | Permite avanzar al Paso 3 |
| > 0 | Sobrepago | Permite avanzar; registra saldo a favor en `fccSaldoFavorCliente` |
| < 0 AND ABS ≤ 100 MXN | Tolerancia | Permite avanzar; registra `ToleranciaAplicada` en `fccSaldoFavorCliente` |
| < 0 AND ABS > 100 MXN | Insuficiente | Bloquea avance; requiere inconsistencia o dejar pendiente |

> ⚠️ Pendiente confirmar tratamiento de tolerancia 100 MXN cuando la moneda de facturación es USD u otra (¿se convierte al equivalente del día?).
> ⚠️ Pendiente confirmar fiscalmente si la factura se timbra por el total de la proforma o por el monto recibido cuando se aplica la tolerancia (Riesgo 1 del requisito).

### B4 — Persistencia de la asociación en ProquifaDotNet

**Descripción:** Endpoint transaccional en Finanzas que persiste la asociación cobro↔documento(s) una vez que el usuario confirma o avanza al Paso 3.

**Operaciones en ProquifaDotNet:**
- `INSERT fccPagoFacturaPedido` por cada cobro↔proforma normal.
- `INSERT fccPagoFacturaAdelanto` por cada cobro↔FAA.
- `UPDATE fccNotaCredito SET Aplicada=1, IdFCCPagoCliente=@IdCobro` por cada NC aplicada.
- `INSERT fccSaldoFavorCliente` si hay sobrepago (`TipoSaldo='SaldoFavor'`) o tolerancia (`TipoSaldo='ToleranciaAplicada'`).
- `UPDATE fccPagoCliente` para marcar el cobro con identificador "saldo a favor" si aplica.

Todo en una sola transacción; rollback completo si cualquier operación falla.

### B5 — Modal de inconsistencia del cobro (Paso 2)

**Descripción:** Endpoint en Finanzas que registra la inconsistencia del Paso 2. Similar al del Paso 1 (RE-FU-024) pero filtra tipos con `AplicaPaso='2'` y soporta adicionalmente el marcado del pedido como "Pendiente de cancelación por falta de pago" cuando el tipo es `PAGO_INCOMPLETO_VENCIDO`.

**Tipos del Paso 2 (catálogo `catTipoInconsistenciaCobro` con `AplicaPaso='2'`):**
- `PAGO_INCOMPLETO_VENCIDO` → habilita opción de marcar pedido como "Pendiente de cancelación" (`AplicaMarkPendienteCancelacion=1`)
- `PAGO_INSUFICIENTE` → registra inconsistencia, deja asociación pendiente de próximo cobro

**Flujo para `PAGO_INCOMPLETO_VENCIDO`:**
1. Usuario selecciona el tipo y confirma.
2. Finanzas detecta `AplicaMarkPendienteCancelacion=1` en el catálogo.
3. Se habilita la opción de marcar el pedido asociado como "Pendiente de cancelación por falta de pago".
4. Si el usuario confirma el marcado: `UPDATE tpPedido SET EstadoCancelacion = 'PENDIENTE_CANCELACION_FALTA_PAGO'` (campo pendiente de confirmar en BD).
5. No se ejecuta cancelación fiscal ni devolución (fuera de R16).

> ⚠️ Pendiente definir el campo exacto en `tpPedido` para el estado "Pendiente de cancelación por falta de pago".
> ⚠️ Pendiente definir el mecanismo de transferencia de este estado al área de Finanzas.

### B6 — Auto-guardado del Paso 2

**Descripción:** Servicio en Finanzas que persiste el estado de las asociaciones en progreso (cobros seleccionados, documentos seleccionados, NCs aplicadas, inconsistencias) de forma transparente. Las asociaciones son editables mientras el usuario esté en el Paso 2; se fijan al avanzar al Paso 3.

---

## Brechas

> ⚠️ **BRECHA FISCAL — Monto de la factura en tolerancia 100 MXN (Riesgo 1)**
> Pendiente confirmar con asesor fiscal si la factura se timbra por el total de la proforma original o por el monto efectivamente recibido cuando se aplica la tolerancia. Decisión fiscal con implicaciones SAT. Impacta el diseño del Paso 3.

> ⚠️ **BRECHA — Saldo remanente de NCs aplicadas parcialmente (Riesgo 2)**
> Si una NC excede el adeudo del documento, el saldo remanente queda fuera de scope R16. Pendiente confirmar con PROQUIFA el tratamiento operativo (auxiliar interno, descarte, otro).

> ⚠️ **BRECHA — Cancelación sin cancelación fiscal ni devolución (Riesgo 3)**
> El tipo `PAGO_INCOMPLETO_VENCIDO` marca el pedido pero R16 no ejecuta cancelación fiscal ni devolución. Pendiente definir el mecanismo de transferencia al área de Finanzas.

> ⚠️ **BRECHA — Tolerancia 100 MXN en moneda distinta a MXN**
> Pendiente confirmar si la tolerancia aplica solo a operaciones MXN o se convierte al equivalente del día para otras monedas.

> ⚠️ **BRECHA — Campo en tpPedido para estado "Pendiente de cancelación por falta de pago"**
> Pendiente definir si se reutiliza un campo existente o se agrega uno nuevo en `tpPedido` para registrar este estado con trazabilidad desde el Paso 2.
