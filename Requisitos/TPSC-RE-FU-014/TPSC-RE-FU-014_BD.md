# Impacto en BD - Tramitacion Pedidos Prepago sin Controlados sin FAA
**Requisito:** TPSC-RE-FU-014
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
Flujo mas comun del bloque Prepago: sin sustancias controladas y sin Factura por Adelantado.
Genera proforma, envia al cliente y crea pendiente en Validar Cobro.
Radio button FAA visible (no seleccionado). Entrega con Remision NO renderizado.
Datos facturacion solo lectura. SIN CAMBIOS ESTRUCTURALES en BD.

---

## Impacto en BD: SIN CAMBIOS ESTRUCTURALES

> Flujo preexistente. Todas las tablas necesarias ya existen.
> tpProformaPedido.Controlados = 0 (sin controlados).
> El pendiente Validar Cobro es tpProformaPedido.MontoPendiente > 0.
> Factura posterior sera NORMAL (no anticipo) porque no hay restricciones regulatorias.

---

## Modelo de Datos

    tpPedido (Tramitar Pedido)
        IdCatCondicionesDePago -> catCondicionesDePago (Prepago: SinCredito=1)
        FacturaPorAdelantado = 0 (disponible pero no activado)
        EntregaConRemision   = 0 (NO RENDERIZADO para Prepago)
        Tramitado = 1
        FolioPedidoInterno (asignado al tramitar)
        IdRegion -> Region (MEX/PER)

    tpProformaPedido (Proforma generada al tramitar)
        MontoTotal, MontoPagado=0, MontoPendiente=MontoTotal
        Controlados = 0 (sin sustancias controladas)
        ReferenciaPago (reconstruida segun RE-FU-006)
        Folio (foliador lineal global)

    tpPedidoProformaPedido (relacion N:N pedido-proforma)
        IdTPPedido FK -> tpPedido
        IdTPProformaPedido FK -> tpProformaPedido

    tpProformaPartidaPedido (partidas de la proforma)
        IdTPProformaPedido, IdTPPartidaPedido, IdProducto, NumeroDePiezas, PrecioUnitario

    tpPedidoCorreoEnviado (correo de proforma al cliente)
        IdTPPedido FK, IdCorreoEnviado FK

    [Validar Cobro]: tpProformaPedido.MontoPendiente > 0 = PENDIENTE

---

## Tablas Involucradas

| Tabla | Rol | Estado |
|-------|-----|--------|
| tpPedido | Cabecera pedido tramitado | Existente - sin cambios |
| tpPartidaPedido | Partidas del pedido | Existente - sin cambios |
| tpProformaPedido | Proforma generada (Controlados=0) | Existente - sin cambios |
| tpPedidoProformaPedido | Relacion pedido-proforma | Existente - sin cambios |
| tpProformaPartidaPedido | Partidas de la proforma | Existente - sin cambios |
| tpPedidoCorreoEnviado | Correo de proforma al cliente | Existente - sin cambios |
| catCondicionesDePago | Prepago: SinCredito=1 | Existente - sin cambios |
| DatosFacturacionCliente | Datos fiscales (solo lectura para Prepago) | Existente - sin cambios |

---

## Diferencia vs TPSC-RE-FU-013 (Prepago CON controlados)

| Aspecto | RE-FU-013 (controlados) | RE-FU-014 (sin controlados) |
|---------|------------------------|----------------------------|
| tpProformaPedido.Controlados | = 1 | = 0 |
| FAA radio button | NO renderizado | SI renderizado (disponible) |
| Remision radio button | NO renderizado | NO renderizado (prepago) |
| Factura posterior (Validar Cobro) | Anticipo | Normal |
| Deteccion de controlados | Requerida | No requerida |

---

## Campos Clave en tpPedido

| Campo | Tipo | Uso en este Flujo |
|-------|------|-------------------|
| FacturaPorAdelantado | bit | = 0 (visible pero no activado) |
| EntregaConRemision | bit | = 0 (NO renderizado para prepago) |
| IdCatCondicionesDePago | uniqueidentifier | Prepago: SinCredito=1 |
| Tramitado | bit | = 1 al completar |
| FolioPedidoInterno | varchar(15) | Folio asignado al tramitar |
| IdRegion | uniqueidentifier | MEX / PER |

---

## Campos Clave en tpProformaPedido

| Campo | Tipo | Uso en este Flujo |
|-------|------|-------------------|
| MontoTotal | decimal | Monto total de la proforma |
| MontoPagado | decimal | = 0 al generar |
| MontoPendiente | decimal | = MontoTotal -> pendiente Validar Cobro |
| Controlados | bit | **= 0** (factura normal posterior) |
| ReferenciaPago | varchar(80) | Referencia bancaria (RE-FU-006) |
| Folio | varchar(80) | Foliador lineal global |
| FechaCompromisoPago | datetime | Fecha limite de pago |
| Activo | bit | 1 |

---

## Flujo en BD al Tramitar

    1. tpPedido.FolioPedidoInterno = siguiente folio (mecanica actual)
    2. tpPedido.Tramitado = 1, FechaTramitacion = GETDATE()
    3. INSERT tpProformaPedido (Controlados=0, MontoPendiente=MontoTotal, Folio=foliador_global)
    4. INSERT tpPedidoProformaPedido (vincular pedido con proforma)
    5. INSERT tpProformaPartidaPedido (por cada partida)
    6. Generar PDF de proforma
    7. Previsualizar PDF al ESAC -> aceptacion explicita
    8. INSERT tpPedidoCorreoEnviado (correo al cliente con PDF adjunto)
    9. Pendiente Validar Cobro = tpProformaPedido.MontoPendiente > 0
   10. Cierre pendiente Tramitar Pedido (tpPedido.Tramitado=1)

---

## Opciones Renderizadas en UI para Prepago sin Controlados

| Opcion | Renderizada | Estado por defecto |
|--------|-------------|--------------------|
| Factura por Adelantado | SI | No seleccionado (si se activa -> flujo RE-FU-015) |
| Entrega con Remision | NO | No aplica para prepago |

---

## Correo de Proforma

| Campo | Fuente | Editable |
|-------|--------|----------|
| Para | ContactoCliente.Correo (default catalogo) | SI |
| CC | Usuario ESAC asignado | SI |
| Asunto | 'Proforma ' + tpPedido.FolioPedidoInterno | NO |
| Adjunto | PDF proforma generado | NO |
| Notas extras | Campo libre | SI (opcional) |

---

## Diferencia MEX vs PER

| Aspecto | Mexico (MEX) | Peru (PER) |
|---------|-------------|------------|
| Tramitacion | Identica | Identica |
| Generacion proforma | SI | SI |
| Envio correo | SI | SI |
| Pendiente Validar Cobro | SI | SI |
| Transferencia a Legacy (post-Validar Cobro) | SI | NO (fuera scope) |

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | Folio proforma lineal global | Verificar mecanismo de foliador en BD |
| 2 | Politica folio si ESAC cancela previsualizacion | Confirmar con cliente |
| 3 | Cancelacion de pedido (Criterio D2) | Verificar campo/flujo cancelacion tpPedido |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| TPSC-RE-FU-006 | ReferenciaPago en proforma |
| TPSC-RE-FU-013 | Variante CON controlados (mismo flujo, diferentes restricciones) |
| TPSC-RE-FU-015 | Si ESAC activa FAA -> cambia a ese flujo |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
