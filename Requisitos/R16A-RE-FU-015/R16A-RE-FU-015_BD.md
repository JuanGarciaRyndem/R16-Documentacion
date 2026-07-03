# Impacto en BD - Tramitacion Pedidos Prepago sin Controlados con FAA
**Requisito:** R16A-RE-FU-015
**Base de Datos:** ProquifaDotNet
**Version:** 1.1 (actualizada conforme a requisito — sin adoptar aún el DIS-SOL)

---

## Resumen
Prepago sin controlados con Factura por Adelantado activada.
Genera directamente el pendiente en Factura por Adelantado al tramitar
(sin generar proforma, PDF ni correo — actualización de requisito).
NO genera pendiente en Validar Cobro al tramitar.
El pendiente Validar Cobro se genera despues, al emitirse la factura PPD.
SIN CAMBIOS ESTRUCTURALES conocidos en BD (estructura tecnica definitiva
del pendiente FAA pendiente de confirmar en diseno).

---

## ⛔ BLOQUEANTE — OBS-027: catEstatusPedido + IdEstatusPedido en tpPedido

> **BLOQUEANTE — En espera de propuesta del cliente.**
> 
> Se requiere definir el catálogo `catEstatusPedido` y el campo `IdEstatusPedido` en `tpPedido` para gestionar el estatus del pedido durante el flujo de tramitación y vida del pedido. Sin esta definición no es posible implementar las tareas de BD relacionadas con la transición de estados del pedido (T6) ni la lógica de transición en Backend (T7).
>
> **Pendiente:**
> - Propuesta del cliente con los estados necesarios para `catEstatusPedido` (clave, descripción, estados terminales, transiciones permitidas).
> - Confirmación de si `IdEstatusPedido` aplica en `tpPedido`, `ppPedido`, o ambas.
>
> **Las tareas T6 y T7 de este requisito están BLOQUEADAS hasta recibir esta propuesta.**

---

## Impacto en BD: SIN CAMBIOS ESTRUCTURALES CONOCIDOS

> Todas las tablas necesarias ya existen (`tpProformaAdelanto`).
> `tpPedido.FacturaPorAdelantado = 1`.
> Al tramitar: pendiente en FAA (`tpProformaAdelanto`), generado directamente
> sin proforma previa — NO en Validar Cobro.
> El pendiente en Validar Cobro lo genera el modulo FAA cuando emite la factura PPD.
> `tpProformaPedido` YA NO aplica a este requisito: el requisito actualizado
> excluye explicitamente la generacion de proforma en este flujo.

---

## Modelo de Datos

    tpPedido (Tramitar Pedido)
        IdCatCondicionesDePago -> catCondicionesDePago (Prepago: SinCredito=1)
        FacturaPorAdelantado = 1 (ACTIVADO por ESAC)
        EntregaConRemision   = 0 (NO RENDERIZADO para Prepago)
        FolioPedidoInterno (asignado al tramitar)
        IdRegion -> Region (MEX/PER)
        ** Estatus/indicador de "pendiente operativo cerrado" pendiente de
           definir (ligado a catEstatusPedido, OBS-027, Criterio D5) **

    tpProformaAdelanto (Pendiente FAA - se genera DIRECTAMENTE al tramitar,
                         sin proforma previa)
        IdCliente, IdEmpresa, Monto
        Datos de facturacion fijados del catalogo vigente del cliente
        IdCFDIGenerada = NULL (pendiente emision)

    ** NO SE GENERA tpProformaPedido EN ESTE FLUJO (actualizacion de requisito) **
    ** NO SE GENERA PENDIENTE EN VALIDAR COBRO AL TRAMITAR **
    ** Validar Cobro se genera DESPUES cuando FAA emite la factura PPD **

---

## Tablas Involucradas

| Tabla | Rol | Estado |
|-------|-----|--------|
| tpPedido | Cabecera - FacturaPorAdelantado=1, Prepago | Existente - sin cambios |
| tpProformaAdelanto | Pendiente FAA generado directamente al tramitar (sin proforma previa) | Existente - sin cambios |
| DatosFacturacionCliente | Datos fiscales (solo lectura, se fijan al activar FAA) | Existente - sin cambios |

> `tpProformaPedido`, `tpPedidoProformaPedido`, `tpProformaPartidaPedido` y `tpPedidoCorreoEnviado` **ya no aplican** a este requisito — no se genera proforma ni correo en este flujo.

---

## Diferencia clave vs RE-FU-014 (sin FAA)

| Aspecto | RE-FU-014 (sin FAA) | RE-FU-015 (con FAA) |
|---------|--------------------|--------------------|
| tpPedido.FacturaPorAdelantado | = 0 | = 1 |
| Proforma/PDF/correo generado en Tramitar Pedido | SI | **NO** (actualización de requisito) |
| Pendiente generado al tramitar | Validar Cobro | **Factura por Adelantado** |
| Pendiente Validar Cobro | SI (inmediato) | NO (posterior, al emitir factura PPD) |
| Documento a cobrar | Proforma | Factura PPD |

---

## Flujo en BD al Tramitar Prepago con FAA

    1. tpPedido.FacturaPorAdelantado = 1
    2. tpPedido.FolioPedidoInterno = siguiente folio
    3. Fija datos de facturacion del catalogo vigente del cliente
    4. INSERT tpProformaAdelanto (Pendiente FAA - IdCFDIGenerada=NULL),
       poblado directamente con datos del pedido/cliente/empresa/partidas
    5. ** NO se genera tpProformaPedido, PDF ni correo en este flujo **
    6. ** NO INSERT pendiente Validar Cobro **
    7. Cierre del pendiente operativo Tramitar Pedido
       (bloqueado hasta resolver OBS-027 - catEstatusPedido)

   === POSTERIORMENTE (fuera scope este requisito) ===
   8. Modulo FAA emite factura PPD -> tpProformaAdelanto.IdCFDIGenerada = valor
   9. FAA genera pendiente Validar Cobro

---

## Cambios de Comportamiento R16 (sin cambio en BD)

| Cambio | Antes | Ahora (R16) |
|--------|-------|-------------|
| Codigo autorizacion para FAA | Requeria codigo | Activacion directa |
| Boton Editar Datos | Visible | Oculto para Prepago siempre |
| Generacion de proforma/PDF/correo en Tramitar Pedido (con FAA) | N/A | Ya no aplica — el pendiente FAA se genera directo |

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | Vinculacion FAA -> Validar Cobro posterior | Confirmar logica del modulo FAA |
| 2 | Estructura tecnica definitiva del pendiente FAA | Confirmar en diseno si se mantiene `tpProformaAdelanto` o se define un modelo nuevo |
| 3 | Documento disponible para TaskScheduler/Legacy | Confirmar si el job de Venta Digital puede operar sin ningun PDF generado en este flujo |
| 4 | Catalogo de estatus del pedido (OBS-027 / Criterio D5) | Pendiente propuesta del cliente |

> Ya no aplican: "Folio proforma lineal global" y "Politica de folio si ESAC cancela previsualizacion" — este flujo no genera proforma.

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-014 | Flujo base Prepago sin FAA (este agrega FAA, ya sin proforma compartida) |
| R16A-RE-FU-012 | Misma mecanica FAA pero para Credito (ambos usan tpProformaAdelanto) |
| R16A-RE-FU-006 | Referencia bancaria del documento fiscal (Codigo Validador) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
