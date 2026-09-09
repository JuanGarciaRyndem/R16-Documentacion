# Impacto en BD - Tramitacion de Pedidos Credito
**Requisito:** R16A-RE-FU-010
**Base de Datos:** ProquifaDotNet
**Version:** 1.1

---

## Resumen
Flujo preexistente de tramitacion de pedidos Credito sin sustancias controladas y sin
Factura por Adelantado. Incluye variante Pago contra entrega (misma mecanica que credito
normal + marca de detencion en Legacy) y el endpoint de Cancelacion (Criterio B1).
**SIN CAMBIOS ESTRUCTURALES en BD propios de este requisito** — la unica estructura nueva
que este requisito consume (`catMotivoCancelacion`) la crea R16A-RE-FU-015 (T7).

---

## Impacto en BD: SIN CAMBIOS ESTRUCTURALES PROPIOS

> Este requisito documenta funcionalidad PREEXISTENTE del modulo Tramitar Pedido, mas la
> Cancelacion (Criterio B1), que ya esta construida (ver Gap #3, resuelto).
> No requiere ALTER TABLE, CREATE TABLE ni INSERT de catalogos nuevos **propios**.
> La Cancelacion consume `dbo.catMotivoCancelacion`, tabla creada por R16A-RE-FU-015 (T7),
> no por este requisito.

---

## Modelo de Datos Involucrado

    ppPedido (Pretramitar)
        -> ppPartidaPedido (IdProducto - sin controlados en este flujo)
        -> ppPedidoConfiguracion (IdEmpresa, IdCatCondicionesDePago)
        Cancelada (bit) -> usado por el endpoint de Cancelacion (T3)

    ppPartidaPedido (partidas del pedido pretramitado)
        Cancelada (bit) -> marcada por el endpoint de Cancelacion (T3), una BitacoraCRUD por partida

    tpPedido (Tramitar)
        IdCatCondicionesDePago -> catCondicionesDePago (Credito: SinCredito=0, Dias>0;
                                   Pago contra entrega: SinCredito=1 pero tratado como
                                   Credito via CondicionPagoFlujoHelper, sin cambio de BD)
        FacturaPorAdelantado = 0 (no requiere FAA en este flujo)
        Tramitado = 1 al completar la accion
        FolioPedidoInterno -> folio asignado al tramitar
        IdRegion -> Region (MEX/PER)
        Activo (bit) -> se pone en false al Cancelar (T3); es lo que lo retira de
                         vTramitarPedido (bandeja del ESAC)

    tpPartidaPedido (partidas tramitadas)
        IdTPPedido FK -> tpPedido
        IdProducto FK -> Producto

    tpProformaPedido (Confirmacion de Pedido)
        IdCliente, IdEmpresa, MontoTotal, FechaCompromisoPago
        Controlados = 0 (en este flujo)

    tpPedidoCorreoEnviado (correo de confirmacion al cliente)
        IdTPPedido FK, IdCorreoEnviado FK

    dbo.catMotivoCancelacion  -- NO es de este requisito, la crea R16A-RE-FU-015 (T7)
        IdCatMotivoCancelacion (PK), Clave, Descripcion, Activo
        5 motivos sembrados: intramitable, ocnoajustada, faltapago, operativo, solicitudcliente
        Consumida por el endpoint de Cancelacion de este requisito (T3), por clave (no por Id,
        porque los GUID difieren entre ambientes)

    BitacoraCRUD (bitacora generica ya existente)
        Usada por T3: 1 registro por ppPedido cancelado + 1 por cada ppPartidaPedido cancelada
        Detalle incluye la descripcion del motivo cuando se informa

---

## Tablas Involucradas

| Tabla | Rol | Estado |
|-------|-----|--------|
| tpPedido | Cabecera del pedido tramitado; `Activo=false` al cancelar (T3) | Existente - reusada, sin cambio de esquema |
| tpPartidaPedido | Partidas del pedido tramitado | Existente - sin cambios |
| tpProformaPedido | Confirmacion de Pedido / Proforma | Existente - sin cambios |
| tpPedidoCorreoEnviado | Correo de confirmacion enviado | Existente - sin cambios |
| ppPedido | Pedido origen en pretramitacion; `Cancelada=true` al cancelar (T3) | Existente - reusada, sin cambio de esquema |
| ppPartidaPedido | Partidas del pedido pretramitado; `Cancelada=true` por partida (T3) | Existente - reusada, sin cambio de esquema |
| catCondicionesDePago | Determina si es Credito (via helper, sin cambio de bandera) | Existente - sin cambios |
| BitacoraCRUD | Bitacora de la cancelacion (1 por pedido + 1 por partida) | Existente - reusada |
| catBitacoraAccion | Accion "Cancelacion" usada por T3 | Existente - reusada |
| **catMotivoCancelacion** | Motivo de cancelacion consumido por T3 | **Nueva, creada por R16A-RE-FU-015 (T7)** — no propiedad de este requisito |
| **pedidoEstadoActual** | Estatus consolidado del pedido (FU-015, T7/T8) | **Nueva, creada por R16A-RE-FU-015** — este requisito NO la actualiza (ver Gaps) |

---

## Campos Clave en tpPedido

| Campo | Tipo | Uso en este Flujo |
|-------|------|-------------------|
| IdTPPedido | uniqueidentifier | PK del pedido tramitado |
| IdPPPedido | uniqueidentifier | FK al pedido pretramitado origen |
| IdCliente | uniqueidentifier | Cliente del pedido |
| IdCatCondicionesDePago | uniqueidentifier | FK - catCondicionesDePago (Credito, evaluado via `CondicionPagoFlujoHelper`) |
| FacturaPorAdelantado | bit | = 0 en este flujo (sin FAA) |
| Tramitado | bit | Se pone en 1 al completar la tramitacion |
| FolioPedidoInterno | varchar(15) | Folio asignado al tramitar |
| FechaTramitacion | datetime | Timestamp de la tramitacion |
| IdRegion | uniqueidentifier | Region del pedido (MEX/PER) |
| IdEmpresa | uniqueidentifier | Empresa que factura |
| IdCatMetodoDePagoCFDI | uniqueidentifier | Metodo de pago para CFDI |
| IdCatUsoCFDI | uniqueidentifier | Uso CFDI del cliente |
| Finalizado | bit | Pedido finalizado tras transferencia |
| **Activo** | bit | Se pone en **false** al cancelar (T3); retira el pedido de `vTramitarPedido` |

---

## Campos Clave en tpProformaPedido (Confirmacion de Pedido)

| Campo | Tipo | Uso en este Flujo |
|-------|------|-------------------|
| IdTPProformaPedido | uniqueidentifier | PK |
| IdCliente | uniqueidentifier | Cliente del pedido |
| IdEmpresa | uniqueidentifier | Empresa que factura |
| MontoTotal | decimal | Monto total del pedido |
| FechaCompromisoPago | datetime | FEE calculada |
| Controlados | bit | = 0 en este flujo |
| ReferenciaPago | varchar(80) | Referencia bancaria reconstruida (RE-FU-006) |
| Activo | bit | 1 = confirmacion vigente |

---

## Determinacion Credito vs Prepago

    -- Credito (incluye Pago contra entrega): CondicionPagoFlujoHelper.EsCondicionCredito(catCDP) = true
    -- Prepago: CondicionPagoFlujoHelper.EsCondicionCredito(catCDP) = false
    -- catCondicionesDePago.SinCredito NO se modifica en BD; la traduccion es solo a nivel de codigo (T1/T2)

**Registros relevantes en catCondicionesDePago:**

| Condicion | SinCredito | Dias | Flujo (via helper) |
|-----------|-----------|------|-------|
| 15 DIAS | 0 | 15 | Credito |
| 30 DIAS | 0 | 30 | Credito |
| 45 DIAS | 0 | 45 | Credito |
| 60 DIAS | 0 | 60 | Credito |
| 90 DIAS | 0 | 90 | Credito |
| PAGO CONTRA ENTREGA | 1 | 0 | Credito (variante con detencion en Legacy; `Clave='pagocontraentrega'`) |
| PREPAGO 100% | 1 | 0 | Prepago (otro flujo) |

> NOTA: Pago contra entrega tiene SinCredito=1 en BD y se sigue tratando como Credito en
> Tramitar Pedido. Desde T1/T2 (2026-09-04), esa traduccion vive en codigo
> (`CondicionPagoFlujoHelper.EsCondicionCredito`), aislada del BO principal, no en una
> consulta ad-hoc por clave dentro del BO.

---

## Transferencia a Legacy

    tpPedido.Tramitado = 1
        -> si region.ProcesarEnETL = true: spActualizarBuzonPedidoLegacy(Encolar)
        -> si region.ProcesarEnETL = false (Peru): se omite, solo queda rastro en log
        -> Legacy recibe datos del pedido + marca de detencion si Pago contra entrega

**SP relacionado encontrado en BD:**
- spActualizarBuzonPedidoLegacy
- spActualizarBuzonPedidoLegacyEncolar

**Pendiente:** los SP en si no se modificaron en T1/T2 (solo se agrego el condicional
regional alrededor de la llamada). Sigue sin evidencia levantada de que el SP incluya la
marca de detencion para Pago contra entrega (Gap #2 abajo).

---

## Consulta - Pedidos Credito tramitados

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    SELECT
        tp.FolioPedidoInterno,
        c.Nombre            AS Cliente,
        cp.CondicionesDePago,
        cp.Dias             AS DiasCredito,
        tp.Monto,
        tp.FechaTramitacion,
        tp.FacturaPorAdelantado,
        tp.Activo,
        r.ClaveISO          AS Region
    FROM dbo.tpPedido tp
    INNER JOIN dbo.Cliente c ON tp.IdCliente = c.IdCliente
    INNER JOIN dbo.catCondicionesDePago cp ON tp.IdCatCondicionesDePago = cp.IdCatCondicionesDePago
    INNER JOIN dbo.Region r ON tp.IdRegion = r.IdRegion
    WHERE tp.Tramitado            = 1
      AND tp.FacturaPorAdelantado = 0
      AND cp.SinCredito           = 0  -- Credito (no incluye Pago contra entrega por clave)
    ORDER BY tp.FechaTramitacion DESC;

    -- Pedidos cancelados via el endpoint de Cancelacion (T3)
    SELECT
        pp.OrdenDeCompra,
        pp.Cancelada          AS ppPedidoCancelada,
        tp.FolioPedidoInterno,
        tp.Activo             AS tpPedidoActivo,
        b.Detalle,
        b.FechaRegistro
    FROM dbo.ppPedido pp
    LEFT JOIN dbo.tpPedido tp ON tp.IdPPPedido = pp.IdPPPedido
    LEFT JOIN dbo.BitacoraCRUD b ON b.IdRegistroAfectado = pp.IdPPPedido
    WHERE pp.Cancelada = 1
    ORDER BY b.FechaRegistro DESC;

---

## Reglas de Negocio en BD

| Regla                      | Implementacion                                      | Campo                                |
| -------------------------- | --------------------------------------------------- | ------------------------------------ |
| Flujo Credito preexistente | `CondicionPagoFlujoHelper.EsCondicionCredito`       | tpPedido.IdCatCondicionesDePago      |
| Sin controlados            | No se valida en este flujo (prerequisito RE-FU-009) | -                                    |
| Sin FAA                    | FacturaPorAdelantado = 0                            | tpPedido.FacturaPorAdelantado        |
| Folio asignado             | Se genera al tramitar                               | tpPedido.FolioPedidoInterno          |
| Confirmacion de Pedido     | INSERT tpProformaPedido                             | tpProformaPedido                     |
| FEE calculada              | Fecha compromiso pago                               | tpProformaPedido.FechaCompromisoPago |
| Correo al cliente          | INSERT tpPedidoCorreoEnviado                        | tpPedidoCorreoEnviado                |
| Cierre del pendiente       | Tramitado = 1                                       | tpPedido.Tramitado                   |
| Marca detencion Pago c/e   | Info en transferencia a Legacy (SP sin cambios)     | Clave 'pagocontraentrega'            |
| ETL solo regiones que transfieren | Condicional `region.ProcesarEnETL` (T1/T2)    | Region.ProcesarEnETL                 |
| Cancelacion de pedido (B1) | `ppPedido.Cancelada` + `ppPartidaPedido.Cancelada` + `tpPedido.Activo=false` + `BitacoraCRUD` + `catMotivoCancelacion` | ppPedido, ppPartidaPedido, tpPedido, BitacoraCRUD |

---

## Dependencias

| Requisito      | Relacion                                                              |
| -------------- | --------------------------------------------------------------------- |
| R16A-RE-FU-009 | Si tiene controlados -> no entra a este flujo (validacion bloqueante) |
| R16A-RE-FU-006 | ReferenciaPago se reconstruye al generar proforma                     |
| R16A-RE-FU-005 | IdCatMetodoDePagoCFDI y IdCatUsoCFDI del cliente Region               |
| R16A-RE-FU-015 | Crea `catMotivoCancelacion` (consumida por T3) y `pedidoEstadoActual` (relacion sin definir, ver Gap #4) |

---

## Gaps

| # | Gap | Estado / Accion |
|---|-----|--------|
| 1 | Pago contra entrega: SinCredito=1 pero se trata como Credito | **Resuelto en codigo** (T1/T2, `CondicionPagoFlujoHelper`), pendiente de PR y evidencia en ambiente |
| 2 | SP de transferencia Legacy: verificar que incluya marca de detencion para Pago contra entrega | **Sigue abierto** — T1/T2 solo agrego el condicional regional; el SP en si no se modifico ni se probo la marca |
| 3 | Cancelacion (Criterio B1): verificar si tpPedido tiene campo Cancelada o similar | **Resuelto** — no se agrego campo nuevo; se reutilizan `ppPedido.Cancelada`, `ppPartidaPedido.Cancelada` y `tpPedido.Activo`, mas el catalogo `catMotivoCancelacion` (de FU-015). Falta prueba end-to-end con token valido |
| 4 | Relacion entre la Cancelacion de este requisito y `pedidoEstadoActual` (FU-015) | **Abierto** — el endpoint de Cancelacion (T3) no actualiza `pedidoEstadoActual`; el estado `cancelado` en esa tabla lo maneja, segun lo construido, el endpoint generico `PUT /api/orders/status` de FU-015. Falta definir si deben sincronizarse |
| 5 | Defecto de Rollback enmascarado en `ppPedidoCancelacionBO` (codigo compartido con Gestionar Intramitables) | **Abierto**, deliberadamente no corregido en este cambio — ver R16A-RE-FU-010-Back.md, Seccion B |

---

**Generado por:** GitHub Copilot in SSMS (base) + actualizado manualmente el 2026-09-08 tras revision de R16A-1477/R16A-1478/R16A-1503 y codigo real de ProquifaDotNet
**Base de Datos:** ProquifaDotNet
