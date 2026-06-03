# Impacto en BD - Tramitacion Pedidos Prepago sin Controlados con FAA
**Requisito:** TPSC-RE-FU-015
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
Prepago sin controlados con Factura por Adelantado activada.
Genera proforma + pendiente en Factura por Adelantado (NO en Validar Cobro).
El pendiente Validar Cobro se genera despues, al emitirse la factura PPD.
SIN CAMBIOS ESTRUCTURALES en BD.

---

## Impacto en BD: SIN CAMBIOS ESTRUCTURALES

> Todas las tablas necesarias ya existen (tpProformaAdelanto, tpProformaPedido).
> tpPedido.FacturaPorAdelantado = 1.
> Al tramitar: pendiente en FAA (tpProformaAdelanto) — NO en Validar Cobro.
> El pendiente en Validar Cobro lo genera el modulo FAA cuando emite la factura PPD.

---

## Modelo de Datos

    tpPedido (Tramitar Pedido)
        IdCatCondicionesDePago -> catCondicionesDePago (Prepago: SinCredito=1)
        FacturaPorAdelantado = 1 (ACTIVADO por ESAC)
        EntregaConRemision   = 0 (NO RENDERIZADO para Prepago)
        Tramitado = 1
        FolioPedidoInterno (asignado al tramitar)
        IdRegion -> Region (MEX/PER)

    tpProformaPedido (Proforma - se genera al tramitar)
        MontoTotal, MontoPagado=0, MontoPendiente=MontoTotal
        Controlados = 0
        Folio (foliador lineal global)

    tpProformaAdelanto (Pendiente FAA - se genera al tramitar)
        IdCliente, IdEmpresa, Monto
        IdCFDIGenerada = NULL (pendiente emision)

    ** NO SE GENERA PENDIENTE EN VALIDAR COBRO AL TRAMITAR **
    ** Validar Cobro se genera DESPUES cuando FAA emite la factura PPD **

---

## Tablas Involucradas

| Tabla | Rol | Estado |
|-------|-----|--------|
| tpPedido | Cabecera - FacturaPorAdelantado=1, Prepago | Existente - sin cambios |
| tpProformaPedido | Proforma generada (Controlados=0) | Existente - sin cambios |
| tpPedidoProformaPedido | Relacion pedido-proforma | Existente - sin cambios |
| tpProformaPartidaPedido | Partidas de la proforma | Existente - sin cambios |
| tpProformaAdelanto | Pendiente FAA generado al tramitar | Existente - sin cambios |
| tpPedidoCorreoEnviado | Correo de proforma al cliente | Existente - sin cambios |
| DatosFacturacionCliente | Datos fiscales (solo lectura) | Existente - sin cambios |

---

## Diferencia clave vs RE-FU-014 (sin FAA)

| Aspecto | RE-FU-014 (sin FAA) | RE-FU-015 (con FAA) |
|---------|--------------------|--------------------|
| tpPedido.FacturaPorAdelantado | = 0 | = 1 |
| Pendiente generado al tramitar | Validar Cobro | **Factura por Adelantado** |
| Pendiente Validar Cobro | SI (inmediato) | NO (posterior, al emitir factura PPD) |
| Documento a cobrar | Proforma | Factura PPD |

---

## Flujo en BD al Tramitar Prepago con FAA

    1. tpPedido.FacturaPorAdelantado = 1
    2. tpPedido.FolioPedidoInterno = siguiente folio
    3. tpPedido.Tramitado = 1, FechaTramitacion = GETDATE()
    4. INSERT tpProformaPedido (Controlados=0, MontoPendiente=MontoTotal)
    5. INSERT tpPedidoProformaPedido
    6. INSERT tpProformaPartidaPedido (por cada partida)
    7. Generar PDF proforma -> previsualizar -> enviar
    8. INSERT tpPedidoCorreoEnviado
    9. INSERT tpProformaAdelanto (Pendiente FAA - IdCFDIGenerada=NULL)
   10. ** NO INSERT pendiente Validar Cobro **
   11. Cierre pendiente Tramitar Pedido

   === POSTERIORMENTE (fuera scope este requisito) ===
   12. Modulo FAA emite factura PPD -> tpProformaAdelanto.IdCFDIGenerada = valor
   13. FAA genera pendiente Validar Cobro (tpProformaPedido.MontoPendiente > 0)

---

## Cambios de Comportamiento R16 (sin cambio en BD)

| Cambio | Antes | Ahora (R16) |
|--------|-------|-------------|
| Codigo autorizacion para FAA | Requeria codigo | Activacion directa |
| Boton Editar Datos | Visible | Oculto para Prepago siempre |

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | Folio proforma lineal global | Verificar mecanismo de foliador |
| 2 | Politica folio si ESAC cancela previsualizacion | Confirmar con cliente |
| 3 | Vinculacion FAA -> Validar Cobro posterior | Confirmar logica del modulo FAA |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| TPSC-RE-FU-014 | Flujo base Prepago sin FAA (este agrega FAA) |
| TPSC-RE-FU-012 | Misma mecanica FAA pero para Credito (ambos usan tpProformaAdelanto) |
| TPSC-RE-FU-006 | ReferenciaPago en proforma |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
