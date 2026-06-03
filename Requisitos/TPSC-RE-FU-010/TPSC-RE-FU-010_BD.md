# Impacto en BD - Tramitacion de Pedidos Credito
**Requisito:** TPSC-RE-FU-010
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
Flujo preexistente de tramitacion de pedidos Credito sin sustancias controladas y sin
Factura por Adelantado. Incluye variante Pago contra entrega (misma mecanica que credito
normal + marca de detencion en Legacy). SIN CAMBIOS ESTRUCTURALES en BD.

---

## Impacto en BD: SIN CAMBIOS ESTRUCTURALES

> Este requisito documenta funcionalidad PREEXISTENTE del modulo Tramitar Pedido.
> No requiere ALTER TABLE, CREATE TABLE ni INSERT de catalogos nuevos.
> Las tablas y campos necesarios ya existen en la BD.

---

## Modelo de Datos Involucrado

    ppPedido (Pretramitar)
        -> ppPartidaPedido (IdProducto - sin controlados en este flujo)
        -> ppPedidoConfiguracion (IdEmpresa, IdCatCondicionesDePago)

    tpPedido (Tramitar)
        IdCatCondicionesDePago -> catCondicionesDePago (Credito: SinCredito=0, Dias>0)
        FacturaPorAdelantado = 0 (no requiere FAA en este flujo)
        Tramitado = 1 al completar la accion
        FolioPedidoInterno -> folio asignado al tramitar
        IdRegion -> Region (MEX/PER)

    tpPartidaPedido (partidas tramitadas)
        IdTPPedido FK -> tpPedido
        IdProducto FK -> Producto

    tpProformaPedido (Confirmacion de Pedido)
        IdCliente, IdEmpresa, MontoTotal, FechaCompromisoPago
        Controlados = 0 (en este flujo)

    tpPedidoCorreoEnviado (correo de confirmacion al cliente)
        IdTPPedido FK, IdCorreoEnviado FK

---

## Tablas Involucradas

| Tabla | Rol | Estado |
|-------|-----|--------|
| tpPedido | Cabecera del pedido tramitado | Existente - sin cambios |
| tpPartidaPedido | Partidas del pedido tramitado | Existente - sin cambios |
| tpProformaPedido | Confirmacion de Pedido / Proforma | Existente - sin cambios |
| tpPedidoCorreoEnviado | Correo de confirmacion enviado | Existente - sin cambios |
| ppPedido | Pedido origen en pretramitacion | Existente - sin cambios |
| catCondicionesDePago | Determina si es Credito (SinCredito=0) | Existente - sin cambios |

---

## Campos Clave en tpPedido

| Campo | Tipo | Uso en este Flujo |
|-------|------|-------------------|
| IdTPPedido | uniqueidentifier | PK del pedido tramitado |
| IdPPPedido | uniqueidentifier | FK al pedido pretramitado origen |
| IdCliente | uniqueidentifier | Cliente del pedido |
| IdCatCondicionesDePago | uniqueidentifier | FK - catCondicionesDePago (Credito: SinCredito=0) |
| FacturaPorAdelantado | bit | = 0 en este flujo (sin FAA) |
| Tramitado | bit | Se pone en 1 al completar la tramitacion |
| FolioPedidoInterno | varchar(15) | Folio asignado al tramitar |
| FechaTramitacion | datetime | Timestamp de la tramitacion |
| IdRegion | uniqueidentifier | Region del pedido (MEX/PER) |
| IdEmpresa | uniqueidentifier | Empresa que factura |
| IdCatMetodoDePagoCFDI | uniqueidentifier | Metodo de pago para CFDI |
| IdCatUsoCFDI | uniqueidentifier | Uso CFDI del cliente |
| Finalizado | bit | Pedido finalizado tras transferencia |

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

    -- Credito: catCondicionesDePago.SinCredito = 0 (Dias > 0 o Pago contra entrega)
    -- Prepago: catCondicionesDePago.SinCredito = 1 (PREPAGO 100%, Pago contra entrega)

**Registros relevantes en catCondicionesDePago:**

| Condicion | SinCredito | Dias | Flujo |
|-----------|-----------|------|-------|
| 15 DIAS | 0 | 15 | Credito |
| 30 DIAS | 0 | 30 | Credito |
| 45 DIAS | 0 | 45 | Credito |
| 60 DIAS | 0 | 60 | Credito |
| 90 DIAS | 0 | 90 | Credito |
| PAGO CONTRA ENTREGA | 1 | 0 | Credito (variante con detencion en Legacy) |
| PREPAGO 100% | 1 | 0 | Prepago (otro flujo) |

> NOTA: Pago contra entrega tiene SinCredito=1 en BD pero segun el requisito se
> comporta como Credito en Tramitar Pedido. La distincion se hace por la clave
> 'pagocontraentrega' y genera la marca de detencion al transferir a Legacy.

---

## Transferencia a Legacy

    tpPedido.Tramitado = 1
        -> Interface Legacy (job o SP de sincronizacion)
        -> Legacy recibe datos del pedido + marca de detencion si Pago contra entrega

**SP relacionado encontrado en BD:**
- spActualizarBuzonPedidoLegacy
- spActualizarBuzonPedidoLegacyEncolar

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
        r.ClaveISO          AS Region
    FROM dbo.tpPedido tp
    INNER JOIN dbo.Cliente c ON tp.IdCliente = c.IdCliente
    INNER JOIN dbo.catCondicionesDePago cp ON tp.IdCatCondicionesDePago = cp.IdCatCondicionesDePago
    INNER JOIN dbo.Region r ON tp.IdRegion = r.IdRegion
    WHERE tp.Tramitado            = 1
      AND tp.FacturaPorAdelantado = 0
      AND tp.Activo               = 1
      AND cp.SinCredito           = 0  -- Credito
    ORDER BY tp.FechaTramitacion DESC;

---

## Reglas de Negocio en BD

| Regla                      | Implementacion                                      | Campo                                |
| -------------------------- | --------------------------------------------------- | ------------------------------------ |
| Flujo Credito preexistente | catCondicionesDePago.SinCredito = 0                 | tpPedido.IdCatCondicionesDePago      |
| Sin controlados            | No se valida en este flujo (prerequisito RE-FU-009) | -                                    |
| Sin FAA                    | FacturaPorAdelantado = 0                            | tpPedido.FacturaPorAdelantado        |
| Folio asignado             | Se genera al tramitar                               | tpPedido.FolioPedidoInterno          |
| Confirmacion de Pedido     | INSERT tpProformaPedido                             | tpProformaPedido                     |
| FEE calculada              | Fecha compromiso pago                               | tpProformaPedido.FechaCompromisoPago |
| Correo al cliente          | INSERT tpPedidoCorreoEnviado                        | tpPedidoCorreoEnviado                |
| Cierre del pendiente       | Tramitado = 1                                       | tpPedido.Tramitado                   |
| Marca detencion Pago c/e   | Info en transferencia a Legacy                      | Clave 'pagocontraentrega'            |

---

## Dependencias

| Requisito      | Relacion                                                              |
| -------------- | --------------------------------------------------------------------- |
| TPSC-RE-FU-009 | Si tiene controlados -> no entra a este flujo (validacion bloqueante) |
| TPSC-RE-FU-006 | ReferenciaPago se reconstruye al generar proforma                     |
| TPSC-RE-FU-005 | IdCatMetodoDePagoCFDI y IdCatUsoCFDI del cliente Region               |

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | Pago contra entrega: SinCredito=1 pero se trata como Credito | Verificar logica de aplicacion - puede usar Clave='pagocontraentrega' |
| 2 | SP de transferencia Legacy | Verificar que incluya marca de detencion para Pago contra entrega |
| 3 | Cancelacion (Criterio B1) | Verificar si tpPedido tiene campo Cancelada o similar |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
