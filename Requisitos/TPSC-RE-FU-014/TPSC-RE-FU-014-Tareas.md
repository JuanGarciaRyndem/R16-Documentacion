# Tareas Back — TPSC-RE-FU-014
**Requisito:** Tramitación de pedidos Prepago sin controlados sin Factura por Adelantado

---

## T1 — [ TPSC-RE-FU-014 ] [ALG-BASIC-LOGIC] Validación Back rechazar Entrega con Remisión en Prepago

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- Regla 2: Entrega con Remisión NO aplica para clientes Prepago en ninguna variante
- La validación debe aplicar para todo pedido con SinCredito=1

### Objetivo general
Implementar validación Back que rechace la tramitación si se envía EntregaConRemision=1 en un pedido Prepago.

### Objetivos específicos
- Validar en `tpPedidoTramitarTransaccionBO.cs`: si `SinCredito=1 && EntregaConRemision=1` -> rechazar con error claro
- Aplicar para todas las variantes Prepago (con/sin controlados, con/sin FAA)

### Resultado esperado
El sistema rechaza tramitaciones Prepago que envíen EntregaConRemision activa.

### Entregables
- Validación en flujo de tramitación

### Criterios de aceptación
- Error claro si EntregaConRemision=1 en pedido Prepago
- Aplica para México y Perú
- No afecta pedidos Crédito

---

## T2 — [ TPSC-RE-FU-014 ] [ALG-BASIC-LOGIC] Validación datos facturación solo lectura en Prepago

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido
- WebApi.Logistica

### Consideraciones previas
- Regla 3: datos de facturación no editables desde Tramitar Pedido para Prepago
- Compartido con RE-FU-013 (misma validación)
- Cualquier ajuste se gestiona en Catálogo de Clientes

### Objetivo general
Implementar validación Back que rechace la edición de datos de facturación cuando el pedido es de cliente Prepago.

### Objetivos específicos
- Identificar endpoint de edición de datos de facturación en WebApi.Logistica
- Agregar validación: si `SinCredito=1` -> rechazar actualización con error
- Los datos se toman del catálogo del cliente vigente (solo lectura)

### Resultado esperado
El sistema impide la edición de datos de facturación para pedidos Prepago desde el módulo Tramitar Pedido.

### Entregables
- Validación en endpoint de edición de datos facturación

### Criterios de aceptación
- Error claro si se intenta editar datos facturación en Prepago
- Aplica para México y Perú
- No afecta pedidos Crédito

---

## T3 — [ TPSC-RE-FU-014 ] [ALG-BASIC-LOGIC] Verificar flujo sin controlados genera proforma correcta

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Facturas\GeneracionProforma

### Consideraciones previas
- El flujo existente en `tpPedidoFacturaToTPProformaPedidoBO.cs` ya genera proforma cuando no hay controlados
- Debe generar una sola proforma con `Controlados=0`
- Verificar que el foliador global (RE-FU-013 T1) se asigna correctamente

### Objetivo general
Verificar que el flujo existente de generación de proforma funciona correctamente para pedidos sin controlados, integrando el foliador global.

### Objetivos específicos
- Confirmar que `TieneControlados()=false` genera una sola proforma normal
- Verificar que `tpProformaPedido.Controlados = false` se asigna correctamente
- Confirmar que el folio global (RE-FU-013 T1) se asigna al campo Folio
- Verificar cálculo de MontoPendiente = MontoTotal
- Confirmar INSERT correcto en tpPedidoProformaPedido y tpProformaPartidaPedido

### Resultado esperado
El flujo de proforma sin controlados funciona correctamente con el foliador global integrado.

### Entregables
- Verificación del flujo existente
- Ajustes (si aplican)

### Criterios de aceptación
- Una sola proforma generada con Controlados=0
- Folio asignado del foliador lineal global
- MontoPendiente = MontoTotal (pendiente Validar Cobro)
- Relaciones pedido-proforma y proforma-partidas correctas

---

## Resumen de tareas

| # | Clave Catálogo | Título | Predecesora |
|---|----------------|--------|-------------|
| T1 | ALG-BASIC-LOGIC | Validación Back rechazar Entrega con Remisión en Prepago | — |
| T2 | ALG-BASIC-LOGIC | Validación datos facturación solo lectura en Prepago | — |
| T3 | ALG-BASIC-LOGIC | Verificar flujo sin controlados genera proforma correcta | RE-FU-013 T1 |

---

## Dependencias con otros requisitos (NO incluidas como tareas)

| Requisito | Tarea relacionada | Relación |
|-----------|-------------------|----------|
| TPSC-RE-FU-006 | ReferenciaPago | Se reconstruye según ese requisito |
| TPSC-RE-FU-010 | Cancelación | Endpoint de cancelación se desarrolla en RE-FU-010 |
| TPSC-RE-FU-013 | T1, T2, T3, T4, T6, T7 | Foliador, previsualización, envío correo, empresa, Perú, vinculación PDF |
| TPSC-RE-FU-016 | Generación PDF | PDF de proforma en DocumentBuilder |
| TPSC-RE-FU-017 | Template PDF | Template de proforma en DocumentBuilder |
