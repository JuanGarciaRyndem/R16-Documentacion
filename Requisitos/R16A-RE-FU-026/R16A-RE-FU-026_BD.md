# Impacto en BD - Validar Cobro: Paso 2 Mexico (Asociacion)
**Requisito:** R16A-RE-FU-026
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
Paso 2 wizard VC Mexico: asociacion N:N cobros <-> proformas/facturas.
Aplicacion OPCIONAL de NCs vigentes. Calculo saldo (exacto/sobrepago/tolerancia).
Tablas de asociacion cobro<->proforma ya existen. Se requiere tabla nueva de
Estado de Cuenta para registrar saldo a favor y diferencias de tolerancia 100 MXN.

---

## Impacto en BD

| #   | Cambio                                                                        | Tipo      | Prioridad |
| --- | ----------------------------------------------------------------------------- | --------- | --------- |
| 1   | CREATE TABLE fccSaldoFavorCliente (Estado de Cuenta/Auxiliar)                 | DDL       | Alta      |
| 2   | ALTER TABLE fccNotaCredito ADD IdFCCPagoCliente (vinculo NC aplicada a cobro) | DDL       | Alta      |
| 3   | ALTER TABLE catTipoInconsistenciaCobro ADD AplicaMarkPendienteCancelacion bit | DDL       | Media     |
| 4   | Reutiliza: fccPagoFacturaPedido (cobro <-> Proforma)                          | Existente | -         |
| 5   | Reutiliza: fccPagoFacturaAdelanto (cobro <-> FAA)                             | Existente | -         |
| 6   | Reutiliza: fccNotaCredito.Aplicada (bit)                                      | Existente | -         |
| 7   | Reutiliza: catTipoInconsistenciaCobro + fccInconsistenciaCobro (RE-FU-024)    | Existente | -         |

---

## Tablas de Asociacion Cobro <-> Documento (existentes)

### fccPagoFacturaPedido - cobro aplicado a Proforma normal

| Columna                | Tipo             | Uso Paso 2                             |
| ---------------------- | ---------------- | -------------------------------------- |
| IdFCCPagoCliente       | uniqueidentifier | FK cobro capturado en Paso 1           |
| IdTPProformaPedido     | uniqueidentifier | FK proforma a la que se aplica         |
| Monto                  | decimal          | Monto aplicado del cobro a la proforma |
| MontoPendienteAnterior | decimal          | Saldo antes de la aplicacion           |
| NumeroDeParcialidad    | int              | Numero de pago parcial                 |
| FechaAplicacion        | datetime         | Fecha de la asociacion                 |

### fccPagoFacturaAdelanto - cobro aplicado a Factura por Adelantado (FAA)

> **Migración (06/07/2026):** la columna `IdTPProformaAdelanto` (FK a la extinta `tpProformaAdelanto`) se retarget a **`IdFccFactura`** (FK a `fccFactura`, creada en **R16A-RE-FU-015**), que unifica el pendiente FAA de origen Prepago y Crédito. Ver "Migración de `tpProformaAdelanto`" en `R16A-RE-FU-015_BD.md`.

| Columna | Tipo | Uso Paso 2 |
|---------|------|------------|
| IdFCCPagoCliente | uniqueidentifier | FK cobro |
| IdTPProformaPedido | uniqueidentifier | FK proforma vinculada a FAA |
| IdFccFactura (antes: IdTPProformaAdelanto) | uniqueidentifier | FK a `fccFactura` (RE-FU-015) |
| IdCFDIGenerada | uniqueidentifier | FK CFDI generado |
| Monto | decimal | Monto aplicado |
| NumeroParcialidad | int | Numero de parcialidad |

---

## Tabla Existente: fccNotaCredito

| Columna            | Tipo             | Uso Paso 2                                 |
| ------------------ | ---------------- | ------------------------------------------ |
| IdFCCNotaCredito   | uniqueidentifier | PK                                         |
| IdTPProformaPedido | uniqueidentifier | FK proforma de origen                      |
| IdCFDI             | uniqueidentifier | UUID timbrado (para nodo CFDIRelacionados) |
| Monto              | decimal          | Monto de la NC                             |
| MXN / USD          | bit              | Moneda                                     |
| Aplicada           | bit              | 0=vigente / 1=ya aplicada                  |
| Folio              | varchar(80)      | Folio de la NC                             |

> NC se filtra: WHERE Aplicada=0 AND Activo=1 = vigentes disponibles para aplicar.
> Al aplicarla en Paso 2: UPDATE fccNotaCredito SET Aplicada=1.

---

## Tabla Nueva: fccSaldoFavorCliente (Estado de Cuenta)

**Proposito:** Registrar saldo a favor del cliente (sobrepago) y diferencias de
tolerancia 100 MXN, para uso en futuras sesiones de Validar Cobro.


    CREATE TABLE [dbo].[fccSaldoFavorCliente](
        [IdFCCSaldoFavorCliente] uniqueidentifier NOT NULL
            CONSTRAINT [DF_fccSaldoFavorCliente_Id] DEFAULT (NEWID()),
        [IdCliente] uniqueidentifier NOT NULL,
        [IdFCCPagoCliente] uniqueidentifier NOT NULL,    -- cobro origen del saldo
        [TipoSaldo] varchar(30) NOT NULL,               -- 'SaldoFavor' o 'ToleranciaAplicada'
        [Monto] decimal(18,4) NOT NULL,
        [MXN] bit NOT NULL CONSTRAINT [DF_fccSaldoFavorCliente_MXN] DEFAULT (0),
        [USD] bit NOT NULL CONSTRAINT [DF_fccSaldoFavorCliente_USD] DEFAULT (0),
        [TipoCambio] decimal(18,6) NULL,
        [Aplicado] bit NOT NULL CONSTRAINT [DF_fccSaldoFavorCliente_Aplicado] DEFAULT (0),
        [IdFCCPagoFacturaPedido] uniqueidentifier NULL,  -- cuando se aplica a futura proforma
        [Observaciones] varchar(500) NULL,
        [Activo] bit NOT NULL CONSTRAINT [DF_fccSaldoFavorCliente_Activo] DEFAULT (1),
        [FechaRegistro] datetime NOT NULL
            CONSTRAINT [DF_fccSaldoFavorCliente_FechaReg] DEFAULT (GETDATE()),
        [FechaUltimaActualizacion] datetime NOT NULL
            CONSTRAINT [DF_fccSaldoFavorCliente_FechaUpd] DEFAULT (GETDATE()),
        CONSTRAINT [PK_fccSaldoFavorCliente] PRIMARY KEY CLUSTERED ([IdFCCSaldoFavorCliente]),
        CONSTRAINT [FK_fccSaldoFavorCliente_Cliente]
            FOREIGN KEY ([IdCliente]) REFERENCES dbo.Cliente([IdCliente]),
        CONSTRAINT [FK_fccSaldoFavorCliente_PagoCliente]
            FOREIGN KEY ([IdFCCPagoCliente]) REFERENCES dbo.fccPagoCliente([IdFCCPagoCliente])
    );

| TipoSaldo            | Cuando se genera                              |
| -------------------- | --------------------------------------------- |
| 'SaldoFavor'         | Cobros+NCs > Adeudo (sobrepago)               |
| 'ToleranciaAplicada' | Cobros+NCs < Adeudo AND diferencia <= 100 MXN |

---

## Vinculo NC con Cobro (ALTER fccNotaCredito)

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Permite identificar en que sesion de cobro se aplico la NC
    ALTER TABLE dbo.fccNotaCredito
        ADD IdFCCPagoCliente uniqueidentifier NULL
            CONSTRAINT [FK_fccNotaCredito_PagoCliente]
                FOREIGN KEY REFERENCES dbo.fccPagoCliente([IdFCCPagoCliente]);

---

## Inconsistencias Paso 2 (catTipoInconsistenciaCobro)

    -- Datos adicionales para tipos del Paso 2
    -- AplicaMarkPendienteCancelacion: indica si este tipo activa el marcado del pedido
    ALTER TABLE dbo.catTipoInconsistenciaCobro
        ADD AplicaMarkPendienteCancelacion bit NOT NULL
            CONSTRAINT [DF_catTipoInconsistenciaCobro_Mark] DEFAULT (0);

    -- Actualizar tipo Pago Incompleto Vencido para habilitar el marcado
    UPDATE dbo.catTipoInconsistenciaCobro
    SET AplicaMarkPendienteCancelacion = 1
    WHERE Clave = 'PAGO_INCOMPLETO_VENCIDO';

| Tipo (Clave)            | AplicaPaso | AplicaMarkPendienteCancelacion |
| ----------------------- | ---------- | ------------------------------ |
| DATOS_INCOMPLETOS       | '1'        | 0                              |
| COMPROBANTE_INVALIDO    | '1'        | 0                              |
| FORMATO_INCORRECTO      | '1'        | 0                              |
| MONTO_ILEGIBLE          | '1'        | 0                              |
| PAGO_INCOMPLETO_VENCIDO | '2'        | **1**                          |
| PAGO_INSUFICIENTE       | '2'        | 0                              |

---

## Tablas Leidas (Lectura)

| Tabla | Datos leidos | Uso Paso 2 |
|-------|-------------|------------|
| fccPagoCliente | Folio, Monto, FechaPago, MXN, USD, TipoDeCambio, Confirmado | Listado cobros disponibles |
| fccSaldoFavorCliente | Monto, TipoSaldo, Aplicado | Saldo a favor previo del cliente |
| tpProformaPedido | MontoPendiente, MontoTotal, Folio, IdCFDIGenerada | Docs pendientes de cobrar |
| fccPagoFacturaPedido | Monto aplicado previo | Saldo pendiente actualizado |
| fccPagoFacturaAdelanto | Monto aplicado previo (FAA) | Saldo pendiente actualizado |
| fccNotaCredito | Monto, MXN, USD, Aplicada=0, IdCFDI (UUID) | NCs vigentes del cliente |
| CFDI | UUID, Serie, Folio | UUID NC para CFDIRelacionados |
| catTipoInconsistenciaCobro | AplicaPaso='2', AplicaMarkPendienteCancelacion | Modal inconsistencia |
| EmpresaDatosBancarios + Empresa | Prefijo emisor del documento | Identificar empresa en listado |

---

## Tablas Escritas (runtime)

| Tabla | Momento | Operacion |
|-------|---------|-----------|
| fccPagoFacturaPedido | Al confirmar asociacion (Proforma) | INSERT (cobro -> proforma) |
| fccPagoFacturaAdelanto | Al confirmar asociacion (FAA) | INSERT (cobro -> FAA) |
| fccFactura | Al confirmar asociacion (FAA) | UPDATE IdCatFacturaEstado = PAGADA (adeudo cubierto total, incluida tolerancia <=100 MXN) o PAGADA_PARCIAL (cobro aplicado con saldo pendiente) — catFacturaEstado, RE-FU-015 v2.1 |
| fccNotaCredito | Al aplicar NC | UPDATE Aplicada=1, IdFCCPagoCliente=@id |
| fccSaldoFavorCliente | Sobrepago | INSERT TipoSaldo='SaldoFavor' |
| fccSaldoFavorCliente | Tolerancia 100 MXN | INSERT TipoSaldo='ToleranciaAplicada' |
| tpProformaPedido | Al asociar | UPDATE MontoPendiente (reducir por cobro aplicado) |
| fccInconsistenciaCobro | Al marcar inconsistencia | INSERT |
| tpPedido | Pago Incompleto Vencido | UPDATE FechaCancelacionPorFaltaPago, IdUsuarioCancelacion |

---

## Calculo del Saldo de Asociacion

    Adeudo = SUM(tpProformaPedido.MontoPendiente) de documentos seleccionados
    Cobros = SUM(fccPagoCliente.Monto) de cobros seleccionados (convertido a moneda cobro)
    NCs    = SUM(fccNotaCredito.Monto) de NCs aplicadas (convertido a moneda cobro)
    Saldo  = (Cobros + NCs) - Adeudo

    SI Saldo = 0:   Pago exacto -> avanza Paso 3
    SI Saldo > 0:   Sobrepago -> INSERT fccSaldoFavorCliente + avanza Paso 3
    SI -100 <= Saldo < 0: Tolerancia -> INSERT fccSaldoFavorCliente (Tolerancia) + avanza Paso 3
    SI Saldo < -100: Pago insuficiente -> bloquea Paso 3, requiere inconsistencia

---

## Multi-Divisa

| Escenario | Conversion |
|-----------|-----------|
| Cobro MXN + Doc MXN | Sin conversion |
| Cobro USD + Doc MXN | Importe Doc * TC (del Paso 1 cobro) -> USD |
| Cobro MXN + Doc USD | Importe Doc * TC -> MXN |
| NC en moneda distinta al cobro | Monto NC * TC -> moneda cobro |

> TC usado = fccPagoCliente.TipoDeCambio (capturado en Paso 1, solo lectura).
> Todos los totales del panel se muestran en moneda del cobro.

---

## Orden de Ejecucion de Scripts

| Paso | Script | Prerequisito |
|------|--------|--------------|
| 1 | CREATE TABLE fccSaldoFavorCliente | Ninguno |
| 2 | ALTER TABLE fccNotaCredito ADD IdFCCPagoCliente | Ninguno |
| 3 | ALTER TABLE catTipoInconsistenciaCobro ADD AplicaMarkPendienteCancelacion | RE-FU-024 |
| 4 | UPDATE catTipoInconsistenciaCobro SET AplicaMarkPendienteCancelacion=1 WHERE PAGO_INCOMPLETO_VENCIDO | Paso 3 |

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | Duda fiscal tolerancia 100 MXN: factura por total o por monto recibido | Fiscal | Confirmar con asesor PROQUIFA |
| 2 | Saldo remanente NCs aplicadas parcialmente | Negocio | Auxiliar interno fuera de R16 |
| 3 | Mecanismo transferencia 'Pago Incompleto Vencido' a Finanzas | Tecnico | Definir canal (Legacy flag, correo, etc.) |
| 4 | Tolerancia 100 MXN en USD u otra moneda | Negocio | Confirmar con Tesoreria |
| 5 | Catalogo completo tipos inconsistencia Paso 2 | Negocio | Solicitar a Tesoreria |
| 6 | Vigencia/expiracion de NCs en BD | Tecnico | Definir si fccNotaCredito tiene fecha caducidad |
| 7 | Notificacion al cliente por Pago Insuficiente | Negocio | Confirmar plantilla y mecanismo |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-024 | Paso 1 MEX (fccPagoCliente confirmado, catTipoInconsistenciaCobro) |
| R16A-RE-FU-023 | fccPagoCliente -> fccPagoFacturaPedido (Validar Cobro) |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura` y del catálogo `catFacturaEstado` — destino de `fccPagoFacturaAdelanto.IdFccFactura`; el Paso 2 ejecuta las transiciones PAGADA / PAGADA_PARCIAL |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
