# Impacto en BD - Tramitacion Pedidos Credito con Factura por Adelantado
**Requisito:** TPSC-RE-FU-012
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
Activacion opcional de Factura por Adelantado para pedidos Credito (MEX) sin controlados.
Al tramitar genera pendiente en modulo FAA (tpProformaAdelanto). Confirmacion de Pedido
se genera de inmediato sin esperar la factura. Datos de facturacion se bloquean.
Peru NO tiene esta opcion para Credito en R16.

---

## Impacto en BD: SIN CAMBIOS ESTRUCTURALES

> Las tablas necesarias para Factura por Adelantado ya existen en la BD:
> tpProformaAdelanto, tpProformaAdelantoProformaPedido y fccPagoFacturaAdelanto.
> El campo tpPedido.FacturaPorAdelantado (bit) ya existe.
> No se requiere ALTER TABLE ni CREATE TABLE.

---

## Modelo de Datos

    tpPedido (Tramitar)
        FacturaPorAdelantado = 1 (activado)
        IdCliente -> Cliente -> DatosFacturacionCliente (datos fijados)
        IdCatCondicionesDePago -> catCondicionesDePago (Credito: SinCredito=0)
        Tramitado = 1 al completar

    tpProformaPedido (Confirmacion de Pedido - se genera inmediatamente)
        Controlados = 0 (sin controlados en este flujo)

    tpProformaAdelanto (Pendiente FAA - se genera al tramitar)
        IdCliente, IdEmpresa, Monto, NumeroOrdenDeCompra
        -> Posteriormente: IdCFDIGenerada, IdCFDI (al emitir factura PPD)

    tpProformaAdelantoProformaPedido (Relacion proforma-adelanto con proforma-pedido)
        IdTPProformaPedido FK -> tpProformaPedido
        IdFCCPagoFacturaAdelanto FK -> fccPagoFacturaAdelanto
        MontoAplicado

    fccPagoFacturaAdelanto (Pagos contra la factura adelanto)
        IdFCCPagoCliente FK -> fccPagoCliente
        IdTPProformaAdelanto FK -> tpProformaAdelanto

---

## Tablas Involucradas

| Tabla | Rol | Estado |
|-------|-----|--------|
| tpPedido | Cabecera - FacturaPorAdelantado = 1 | Existente - sin cambios |
| tpProformaPedido | Confirmacion de Pedido (se genera inmediatamente) | Existente - sin cambios |
| tpProformaAdelanto | Pendiente generado en modulo FAA | Existente - sin cambios |
| tpProformaAdelantoProformaPedido | Relacion proforma-adelanto con proforma-pedido | Existente - sin cambios |
| fccPagoFacturaAdelanto | Pagos contra la factura adelanto | Existente - sin cambios |
| DatosFacturacionCliente | Datos de facturacion del cliente (se fijan al activar) | Existente - sin cambios |
| catCondicionesDePago | Determina Credito (SinCredito=0) | Existente - sin cambios |
| tpPedidoCorreoEnviado | Correo de confirmacion | Existente - sin cambios |

---

## Campos Clave en tpPedido

| Campo | Tipo | Uso en este Flujo |
|-------|------|-------------------|
| FacturaPorAdelantado | bit | **= 1 cuando se activa FAA** |
| IdCatCondicionesDePago | uniqueidentifier | Credito: SinCredito=0 |
| EntregaConRemision | bit | No aplica en este flujo (sin controlados) |
| Tramitado | bit | = 1 al completar |
| FolioPedidoInterno | varchar(15) | Folio asignado al tramitar |
| IdRegion | uniqueidentifier | Solo MEX puede activar FAA para Credito |

---

## tpProformaAdelanto (Pendiente FAA)

| Campo | Tipo | Descripcion |
|-------|------|-------------|
| IdTPProformaAdelanto | uniqueidentifier | PK |
| Monto | decimal | Monto total del pedido a facturar |
| MXN / USD | bit | Moneda de la factura |
| IdCFDIGenerada | uniqueidentifier | NULL hasta que Finanzas emita la factura PPD |
| IdCFDI | uniqueidentifier | NULL hasta el timbrado |
| TipoDeCambio | decimal | Tipo de cambio al momento de emision |
| IdCliente | uniqueidentifier | Cliente del pedido |
| IdEmpresa | uniqueidentifier | Empresa que factura |
| NumeroOrdenDeCompra | varchar(80) | Referencia del pedido |
| Activo | bit | 1 = pendiente vigente |

---

## DatosFacturacionCliente (datos que se fijan al activar FAA)

| Campo | Tipo | Se fija al activar |
|-------|------|--------------------|
| RFC | varchar(50) | SI - no editable en Tramitar |
| RazonSocial | varchar(120) | SI |
| IdCatUsoCFDI | uniqueidentifier | SI |
| IdCatMetodoDePagoCFDI | uniqueidentifier | SI |
| Correo | varchar(200) | SI |
| IdCatRegimenFiscal | uniqueidentifier | SI |

> Al activar FAA los datos se toman del catalogo vigente y quedan de solo lectura.
> Cualquier ajuste se gestiona en el modulo FAA o actualizando el catalogo del cliente.

---

## Flujo en BD al Tramitar con FAA

    1. tpPedido.FacturaPorAdelantado = 1
    2. tpPedido.Tramitado = 1, FechaTramitacion = GETDATE()
    3. INSERT tpProformaPedido (Confirmacion de Pedido - inmediata)
    4. INSERT tpProformaAdelanto (Pendiente FAA - datos del pedido)
    5. INSERT tpPedidoCorreoEnviado (Correo de confirmacion al cliente)
    6. Transferencia a Legacy (solo MEX, con marca detencion si Pago c/e)

---

## Diferencia MEX vs PER

| Aspecto | Mexico (MEX) | Peru (PER) |
|---------|-------------|------------|
| FAA disponible para Credito | SI | NO (timbrado peruano R16 = solo Prepago) |
| Tramitacion Credito | Flujo normal | Flujo normal |
| Transferencia a Legacy | SI | NO |

---

## Consulta - Pedidos con FAA activa

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    SELECT
        tp.FolioPedidoInterno,
        c.Nombre          AS Cliente,
        tp.Monto,
        tp.FechaTramitacion,
        pa.IdTPProformaAdelanto,
        CASE WHEN pa.IdCFDIGenerada IS NOT NULL THEN 'Emitida' ELSE 'Pendiente' END AS EstadoFAA
    FROM dbo.tpPedido tp
    INNER JOIN dbo.Cliente c ON tp.IdCliente = c.IdCliente
    LEFT  JOIN dbo.tpProformaAdelanto pa ON pa.IdCliente = tp.IdCliente
                                       AND pa.NumeroOrdenDeCompra = tp.NumeroOrdenDeCompra
                                       AND pa.Activo = 1
    WHERE tp.FacturaPorAdelantado = 1
      AND tp.Tramitado           = 1
      AND tp.Activo              = 1
    ORDER BY tp.FechaTramitacion DESC;

---

## Cambios de Comportamiento R16 (sin cambio en BD)

| Cambio | Antes | Ahora (R16) |
|--------|-------|-------------|
| Codigo de autorizacion para FAA | Requeria codigo | Activacion directa sin codigo |
| Boton Editar Datos con FAA activa | Disponible | **Oculto** cuando FAA=1 |
| FAA para Peru Credito | No existia Peru | NO disponible (timbrado Peru = solo Prepago) |

> Estos cambios son de LOGICA DE APLICACION, no de estructura de BD.

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | Relacion tpProformaAdelanto con tpPedido no es FK directa | Se vincula via IdCliente + NumeroOrdenDeCompra - confirmar logica |
| 2 | Rol que gestiona FAA no confirmado | Finanzas o Coordinador de Planeacion - confirmar con cliente |
| 3 | Pendiente 'Relacionar facturas' en Legacy | Mecanismo PQF2->Legacy para este pendiente fuera de scope |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| TPSC-RE-FU-010 | Flujo base Credito sin FAA |
| TPSC-RE-FU-011 | Con controlados: FAA no disponible (mutuamente excluyentes) |
| TPSC-RE-FU-005 | Datos de facturacion (IdCatUsoCFDI, IdCatMetodoDePagoCFDI) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
