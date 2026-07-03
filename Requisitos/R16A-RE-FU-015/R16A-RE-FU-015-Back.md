# Impacto en Back — R16A-RE-FU-015
**Requisito:** Tramitación de pedidos Prepago (variante con Factura por Adelantado)
**Aplicativo:** ProquifaDotNet
**Módulo:** L05.TramitarPedido
**Impacto:** Flujo preexistente — variante Prepago con FAA activada, genera pendiente directamente en el módulo Factura por Adelantado (NO en Validar Cobro, y sin generar proforma)

---

## ⛔ BLOQUEANTE — OBS-027: Transición de estados del pedido

> **BLOQUEANTE — En espera de propuesta del cliente.**
>
> Se requiere el catálogo `catEstatusPedido` y el campo `IdEstatusPedido` en `tpPedido` para implementar la lógica de transición de estados durante el flujo de tramitación. Sin esta definición no es posible implementar la lógica de transición en Back (T7 de este requisito).
>
> **Las tareas T6 (BD) y T7 (Back) están BLOQUEADAS hasta recibir la propuesta del cliente sobre `catEstatusPedido`.**

---

## Resumen

> **Actualización de requisito (matriz cliente):** el requisito ya no contempla generación de proforma, PDF ni envío de correo en este flujo. Este documento se actualiza para reflejar esa reducción de alcance. La definición técnica final (tablas, endpoints) se documenta en el DIS-SOL — este documento describe el impacto conforme al requisito, sin adoptar aún decisiones de diseño.

Este requisito es la variante Prepago sin controlados donde el ESAC **activa Factura por Adelantado**. A diferencia de RE-FU-013/014, este flujo **no genera proforma, PDF ni correo**: al tramitar, el sistema genera directamente el pendiente en el módulo Factura por Adelantado y cierra el pendiente operativo de Tramitar Pedido.

- **Al tramitar NO genera pendiente en Validar Cobro** (no hay proforma que lo dispare)
- **Genera pendiente en módulo Factura por Adelantado** directamente al tramitar (INSERT en `tpProformaAdelanto`)
- El pendiente Validar Cobro se generará después, cuando FAA emita la factura PPD (RE-FU-018/019/020)

**Otras diferencias:**
- `tpPedido.FacturaPorAdelantado = 1`
- Activación directa sin código de autorización (Regla 3 — eliminar validación anterior si existe)
- Datos de facturación bloqueados y fijados al activar FAA
- El cierre del pendiente operativo en Tramitar Pedido no implica que el pedido quede "tramitado en su totalidad" (Regla 13 / Criterio D3) — el estatus real del pedido a lo largo del flujo queda pendiente de definir (Criterio D5, ligado a OBS-027)

---

## Código Existente Relacionado

### Tramitación principal
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

### Pendiente FAA (sin proforma previa)
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Anticipos\tpProformaAdelantoBO.cs`

> Entidad `tpProformaAdelanto` — representa el pendiente en módulo FAA. Ya existe como CrudBO. En este flujo se puebla directamente con datos del pedido/partidas/cliente, sin pasar por la generación de una proforma.

---

## Análisis — Flujo compartido con RE-FU-014

| Componente | Se desarrolla en | Aplica a RE-FU-015 |
|------------|-------------------|----------------------|
| Verificación Perú | RE-FU-013 T6 | Sí — misma verificación |
| Validación Remisión Prepago | RE-FU-014 T1 | Sí — misma validación |
| Datos facturación solo lectura | RE-FU-014 T2 | Sí — misma validación |

> **Nota:** El foliador lineal global de proforma (RE-013 T1), la previsualización del PDF (RE-013 T3), el envío de correo (RE-013 T4) y la vinculación con DocumentBuilder (RE-013 T7) **ya no aplican a RE-015** — el requisito actualizado no genera proforma, PDF ni correo en este flujo.

---

## Gaps de Desarrollo Específicos de RE-FU-015

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Generación pendiente FAA al tramitar con FAA=1 | Al confirmar la acción de tramitar (sin generación previa de proforma), INSERT en `tpProformaAdelanto` con datos del pedido/cliente/empresa/monto. Atómico con la transacción | Medio |
| GAP-02 | ~~NO generar pendiente Validar Cobro cuando FAA=1~~ | **Ya no aplica.** Como este flujo no genera `tpProformaPedido`, no existe `MontoPendiente` que pudiera disparar un pendiente en Validar Cobro — no hay nada que suprimir | — |
| GAP-03 | Eliminar código de autorización para FAA | Buscar y eliminar validación de código de autorización para activar Factura por Adelantado (Regla 3: activación directa) | Bajo |
| GAP-04 | Bloquear datos facturación al activar FAA | Fijar datos de facturación del catálogo del cliente vigente al momento de activar FAA. Confirmar si además se requiere rechazar edición posterior desde un endpoint dedicado, o si la fijación por snapshot es suficiente | Bajo |
| GAP-05 | Vinculación con módulo FAA (RE-FU-018/019/020) | Tarea para asegurar que el pendiente generado en `tpProformaAdelanto` sea consumido correctamente por el módulo FAA. Como no hay proforma previa, los datos deben poblarse directamente desde pedido/partidas/cliente | Bajo |
| GAP-06 | Cancelación del pedido | Dependencia de R16A-RE-FU-010 (endpoint de cancelación) | Referencia |
| GAP-07 | Ausencia de documento/PDF disponible para TaskScheduler de Venta Digital | Confirmar si el job de TaskScheduler que transfiere PDFs a Legacy puede operar cuando no existe ningún PDF generado en Tramitar Pedido para este flujo (ver `R16A-RE-FU-015-Tareas.md` T8) | Medio |

---

## Flujo Back Completo

```
1. ESAC ejecuta Tramitar (FAA=1, sin controlados)
2. Back valida:
   - Condición de pago = Prepago (SinCredito=1)
   - TieneControlados() = false (FAA + controlados NO permitido)
   - FacturaPorAdelantado = 1
   - EntregaConRemision debe ser 0 (rechazar si 1)
   - No requiere código de autorización
3. Asigna FolioPedidoInterno (mecánica actual)
4. tpPedido.FacturaPorAdelantado = 1
5. Fija datos de facturación del catálogo del cliente vigente
6. Genera el pendiente FAA directamente (sin proforma previa):
   - INSERT tpProformaAdelanto con datos del pedido/cliente/empresa/monto
   - IdCFDIGenerada = NULL (pendiente de emisión por módulo FAA)
7. NO genera proforma, PDF ni correo — el requisito actualizado excluye
   explícitamente estos pasos de este flujo
8. NO genera pendiente Validar Cobro (no hay proforma que lo dispare)
9. Cierra el pendiente operativo de Tramitar Pedido
   (⛔ bloqueado hasta resolver OBS-027 — ver catEstatusPedido)
10. Actualiza el estado/indicador del pedido conforme a lo que finalmente
    se defina para "pendiente operativo cerrado" (Criterio D3/D5) — el
    catálogo de estatus del pedido está pendiente de definición del cliente
```

---

## Datos del pendiente FAA (tpProformaAdelanto)

| Dato              | Origen                             |
| ----------------- | ---------------------------------- |
| IdCliente         | tpPedido.IdCliente                 |
| IdEmpresa         | tpPedido -> empresa de la proforma |
| Monto             | MontoTotal de la proforma          |
| IdCFDIGenerada    | NULL (pendiente emisión)           |
| Datos facturación | DatosFacturacionCliente fijados    |
| Folio pedido      | tpPedido.FolioPedidoInterno        |
| Estado            | Pendiente de emisión               |

---

## Transferencia a Legacy

- En Tramitar Pedido NO hay transferencia para Prepago
- La transferencia ocurre después de Validar Cobro (fuera de scope)
- **Punto abierto:** al no generarse ningún PDF en este flujo (ni proforma ni confirmación), queda pendiente confirmar qué documento, si alguno, puede transferir a Legacy el job de TaskScheduler de Venta Digital para pedidos Prepago con FAA (ver Tareas T8/T9)

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| R16A-RE-FU-006 | Referencia bancaria del documento fiscal (Código Validador) |
| R16A-RE-FU-010 | Cancelación de pedido (endpoint compartido) |
| R16A-RE-FU-013 | Verificación Perú (única parte del flujo base aún compartida) |
| R16A-RE-FU-014 | Validaciones compartidas: Remisión Prepago, datos facturación |
| R16A-RE-FU-018/019/020 | Módulo FAA consume el pendiente generado aquí |

> RE-FU-016/017 (generación de PDF de proforma en DocumentBuilder) **ya no aplica** a este requisito — no se genera proforma en este flujo.

---

## Conclusión

El requisito R16A-RE-FU-015 tiene **impacto medio-bajo** en desarrollo Back tras la actualización del requisito que elimina la proforma de este flujo. Ya no reutiliza el flujo base de proforma de RE-FU-013 (foliador, previsualización, envío) — solo comparte con RE-FU-014 las validaciones de Remisión y datos de facturación. Lógica específica de RE-015:

1. **Pendiente FAA directo** (GAP-01) — INSERT en `tpProformaAdelanto` al tramitar, sin proforma previa
2. **Sin código de autorización** (GAP-03) — eliminar validación anterior
3. **Bloqueo datos facturación** (GAP-04) — fijar al activar FAA
4. **Vinculación con FAA** (GAP-05) — asegurar consumo del pendiente por RE-FU-018/019/020
5. **Punto abierto de Venta Digital/Legacy** (GAP-07) — confirmar comportamiento de TaskScheduler sin PDF disponible

El desarrollador debe revisar `tpProformaAdelantoBO.cs` para entender la estructura del pendiente FAA y cómo se integra con el flujo de tramitación. La estructura técnica definitiva (si se mantiene `tpProformaAdelanto` o se introduce un modelo nuevo) se confirma en el DIS-SOL.
