# Impacto en BD - Tramitacion Pedidos Credito con Sustancias Controladas
**Requisito:** TPSC-RE-FU-011
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
Tramitacion de pedidos Credito (incluyendo Pago contra entrega) con sustancias controladas.
Oculta Factura por Adelantado y Entrega con Remision. Peru no transfiere a Legacy.
Reutiliza flujo credito existente con restricciones regulatorias.

---

## Impacto en BD: MINIMO - Posible nuevo campo

> Flujo preexistente con restricciones regulatorias.
> La deteccion de controlados usa la cadena: tpPartidaPedido.IdProducto ->
> MarcaFamilia -> Familia -> catControl.Clave IN ('mundiales','nacionales','origen').
> tpPedido NO tiene campo 'Controlados' — la tpProformaPedido SI lo tiene.
> Peru sin transferencia a Legacy se controla por IdRegion en la logica de aplicacion.

---

## Modelo de Datos

    tpPedido (Tramitar)
        IdCatCondicionesDePago -> catCondicionesDePago (Credito: SinCredito=0)
        FacturaPorAdelantado = 0 (OCULTO cuando hay controlados)
        EntregaConRemision   = 0 (OCULTO cuando hay controlados)
        IdRegion -> Region (MEX: transfiere Legacy / PER: NO transfiere)

    tpPartidaPedido (partidas -> detectar controlados)
        IdProducto -> MarcaFamilia -> Familia -> catControl
            Clave IN ('mundiales','nacionales','origen') = controlado

    tpProformaPedido (Confirmacion de Pedido)
        Controlados = 1 (en este flujo)

---

## Tablas Involucradas

| Tabla | Rol | Estado |
|-------|-----|--------|
| tpPedido | Cabecera pedido tramitado | Existente - sin cambios |
| tpPartidaPedido | Partidas - deteccion de controlados | Existente - sin cambios |
| tpProformaPedido | Confirmacion de Pedido (Controlados=1) | Existente - sin cambios |
| tpPedidoCorreoEnviado | Correo de confirmacion | Existente - sin cambios |
| catCondicionesDePago | Determina Credito vs Prepago | Existente - sin cambios |
| catControl | Deteccion de sustancias controladas | Existente - sin cambios |
| Familia | IdCatControl por familia de producto | Existente - sin cambios |
| MarcaFamilia | Vinculo Producto -> Familia | Existente - sin cambios |
| fnEsProductoControlado | Funcion de deteccion | Existente - **requiere 'origen'** |

---

## Campos Clave - Restricciones Regulatorias

| Campo | Tabla | Tipo | Comportamiento con Controlados |
|-------|-------|------|-------------------------------|
| FacturaPorAdelantado | tpPedido | bit | **OCULTO en UI** - siempre 0 si hay controlados |
| EntregaConRemision | tpPedido | bit | **OCULTO en UI** - siempre 0 si hay controlados |
| Controlados | tpProformaPedido | bit | = 1 cuando el pedido tiene sustancias controladas |
| IdRegion | tpPedido | uniqueidentifier | MEX: transfiere a Legacy / PER: no transfiere |

---

## Deteccion de Controlados en Partidas

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    DECLARE @IdTPPedido UNIQUEIDENTIFIER;

    SELECT CASE WHEN EXISTS (
        SELECT 1
        FROM dbo.tpPartidaPedido tp
        INNER JOIN dbo.MarcaFamilia mf ON tp.IdProducto = mf.IdProducto
        INNER JOIN dbo.Familia f        ON mf.IdFamilia = f.IdFamilia
        INNER JOIN dbo.catControl cc    ON f.IdCatControl = cc.IdCatControl
        WHERE tp.IdTPPedido = @IdTPPedido
          AND tp.Activo     = 1
          AND cc.Clave IN ('mundiales','nacionales','origen')
    ) THEN 1 ELSE 0 END AS TieneControlados;

---

## Diferencia MEX vs PER al Tramitar

| Aspecto | Mexico (MEX) | Peru (PER) |
|---------|-------------|------------|
| Tramitacion | Flujo credito normal | Flujo credito normal |
| FAA y Remision | Ocultos por controlados | Ocultos por controlados |
| Confirmacion de Pedido | SI | SI |
| Transferencia a Legacy | SI + marca detencion si Pago c/e | NO |
| Pendiente cerrado | SI | SI |

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | fnEsProductoControlado sin 'origen' | ALTER FUNCTION agregar 'origen' (compartido RE-FU-007/009) |
| 2 | tpPedido sin campo Controlados | Deteccion es dinamica via partidas - confirmar si se requiere campo cache |
| 3 | Peru sin transferencia Legacy | Logica en aplicacion por IdRegion - sin campo adicional en BD |
| 4 | Cancelacion (Criterio C1) | Verificar campo/flujo de cancelacion en tpPedido |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| TPSC-RE-FU-009 | Validacion regulatoria en Pretramitar (prerequisito para llegar aqui) |
| TPSC-RE-FU-010 | Flujo base Credito sin controlados (este agrega restricciones) |
| TPSC-RE-FU-007 | fnEsProductoControlado compartida |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
