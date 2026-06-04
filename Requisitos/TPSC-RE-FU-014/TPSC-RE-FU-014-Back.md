# Impacto en Back — TPSC-RE-FU-014
**Requisito:** Tramitación de pedidos Prepago sin controlados sin Factura por Adelantado
**Aplicativo:** ProquifaDotNet
**Módulo:** L05.TramitarPedido
**Impacto:** Flujo preexistente — mismo flujo base de tramitación Prepago que RE-FU-013 pero sin restricciones de controlados

---

## Resumen

Este es el **flujo más común** del bloque Prepago: pedidos sin sustancias controladas y sin Factura por Adelantado activada, para México y Perú. El flujo es **idéntico** al de RE-FU-013 en cuanto a:

- Generación de proforma con foliador lineal global
- Previsualización obligatoria del PDF
- Pantalla de datos de envío de correo
- Generación de pendiente Validar Cobro
- Cierre de pendiente Tramitar Pedido

**Diferencias respecto a RE-FU-013:**
- `tpProformaPedido.Controlados = 0`
- Radio button FAA **se renderiza** (disponible pero no seleccionado)
- No se requiere validación de controlados
- No aplica restricción regulatoria sobre FAA/Remisión
- La factura posterior será **normal** (no anticipo)

---

## Código Existente Relacionado

### Generación de proformas (mismo flujo)
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\GeneracionProforma\tpPedidoFacturaToTPProformaPedidoBO.cs`

> **Lógica existente para no-controlados:**
> - `TieneControlados()` retorna `false`
> - Se genera una sola proforma (no separada por partida)
> - `tpProformaPedido.Controlados = false`
> - Factory genera proforma normal vía `tpProformaPedidoFactory.Process(tpPedido)`

### Factory de proforma
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs`

> Mismo comportamiento que RE-FU-013: genera `tpProformaPedido` por empresa con `Folio = null` (pendiente asignar vía foliador global)

### Tramitación principal
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

---

## Análisis — Flujo compartido con RE-FU-013

La mayor parte de la lógica Back de este requisito es **la misma** que RE-FU-013. Los componentes compartidos son:

| Componente | Tarea en RE-FU-013 | Aplica a RE-FU-014 |
|------------|--------------------|--------------------|
| Foliador lineal global | T1 (ALG-COMPLX-LOGIC) | Sí — mismo foliador |
| Verificar por empresa | T2 (ALG-BASIC-LOGIC) | Sí — misma verificación |
| Previsualización PDF | T3 (IMP-EXIST-SERVICE) | Sí — mismo endpoint |
| Envío correo proforma | T4 (IMP-EXIST-SERVICE) | Sí — mismo endpoint |
| Verificación Perú | T6 (ALG-BASIC-LOGIC) | Sí — misma verificación |
| Vinculación PDF (RE-FU-016/017) | T7 (ALG-BASIC-LOGIC) | Sí — misma vinculación |

> **Nota:** Los componentes compartidos se desarrollan en RE-FU-013 y son reutilizados por RE-FU-014 sin modificación adicional.

---

## Gaps de Desarrollo Específicos de RE-FU-014

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Validación Back: rechazar EntregaConRemisión en Prepago | Si `SinCredito=1 && EntregaConRemision=1` -> rechazar tramitación (Regla 2: no renderizado para Prepago) | Bajo |
| GAP-02 | Validación Back: datos facturación solo lectura en Prepago | Rechazar edición de datos facturación cuando SinCredito=1 (Regla 3). Compartido con RE-FU-013 | Bajo |
| GAP-03 | Confirmar flujo sin controlados genera proforma correcta | Verificar que cuando `TieneControlados()=false`, se genera una sola proforma con `Controlados=0` y todos los campos correctos | Bajo |
| GAP-04 | Cancelación del pedido | Dependencia de TPSC-RE-FU-010 (endpoint de cancelación) | Referencia |

---

## Flujo Back Completo

```
1. ESAC ejecuta Tramitar (FAA=0, sin controlados)
2. Back valida:
   - Condición de pago = Prepago (SinCredito=1)
   - TieneControlados() = false
   - FacturaPorAdelantado = 0
   - EntregaConRemision debe ser 0 (rechazar si 1)
3. Asigna FolioPedidoInterno (mecánica actual)
4. Genera proforma por empresa:
   - tpProformaPedido con Controlados=0
   - Folio = foliador lineal global (desarrollado en RE-FU-013 T1)
   - MontoPendiente = MontoTotal
   - INSERT tpPedidoProformaPedido + tpProformaPartidaPedido
5. Genera PDF vía DocumentBuilder (RE-FU-016/017)
6. Retorna PDF para previsualización (RE-FU-013 T3)
7. ESAC acepta -> Front llama endpoint de envío
8. Endpoint envío (RE-FU-013 T4):
   - Recibe: Para, CC, NotasExtras
   - Asunto = "Proforma " + FolioPedidoInterno
   - Adjunto = PDF
   - Envía correo
9. Al confirmar envío exitoso:
   - Pendiente Validar Cobro activo (MontoPendiente > 0)
   - Cierra pendiente de Tramitar Pedido
10. tpPedido.Tramitado = 1
```

---

## Transferencia a Legacy

- **México:** Posterior al flujo de Validar Cobro (fuera de scope)
- **Perú:** Sin transferencia a Legacy
- En Tramitar Pedido NO hay transferencia para Prepago

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| TPSC-RE-FU-006 | ReferenciaPago de la proforma |
| TPSC-RE-FU-010 | Cancelación de pedido (endpoint compartido) |
| TPSC-RE-FU-013 | Flujo base compartido: foliador, previsualización, envío, Perú |
| TPSC-RE-FU-016 | Generación de PDF de proforma en DocumentBuilder |
| TPSC-RE-FU-017 | Template/endpoint de proforma en DocumentBuilder |

---

## Conclusión

El requisito TPSC-RE-FU-014 tiene **impacto bajo** en desarrollo Back. Es el flujo Prepago más común y **reutiliza íntegramente** los componentes desarrollados en RE-FU-013 (foliador, previsualización, envío correo, verificación Perú, vinculación PDF). Los gaps específicos son mínimos:

1. **Validación Remisión en Prepago** (GAP-01) — rechazar si se envía EntregaConRemision=1
2. **Datos facturación solo lectura** (GAP-02) — compartido con RE-FU-013
3. **Verificar proforma sin controlados** (GAP-03) — confirmar flujo existente

El desarrollador debe confirmar que el flujo existente en `tpPedidoFacturaToTPProformaPedidoBO.cs` genera correctamente la proforma cuando no hay controlados (una sola proforma, Controlados=0).
