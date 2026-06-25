# Impacto en Back — R16A-RE-FU-015
**Requisito:** Tramitación de pedidos Prepago con Factura por Adelantado
**Aplicativo:** ProquifaDotNet
**Módulo:** L05.TramitarPedido
**Impacto:** Flujo preexistente — variante Prepago con FAA activada, genera pendiente en módulo Factura por Adelantado (NO en Validar Cobro)

---

## ⛔ BLOQUEANTE — OBS-027: Transición de estados del pedido

> **BLOQUEANTE — En espera de propuesta del cliente.**
>
> Se requiere el catálogo `catEstatusPedido` y el campo `IdEstatusPedido` en `tpPedido` para implementar la lógica de transición de estados durante el flujo de tramitación. Sin esta definición no es posible implementar la lógica de transición en Back (T7 de este requisito).
>
> **Las tareas T6 (BD) y T7 (Back) están BLOQUEADAS hasta recibir la propuesta del cliente sobre `catEstatusPedido`.**

---

## Resumen

Este requisito es la variante Prepago sin controlados donde el ESAC **activa Factura por Adelantado**. Comparte el flujo base de RE-FU-013/014 (proforma, previsualización, envío) con una diferencia clave:

- **Al tramitar NO genera pendiente en Validar Cobro**
- **Genera pendiente en módulo Factura por Adelantado** (INSERT en `tpProformaAdelanto`)
- El pendiente Validar Cobro se generará después, cuando FAA emita la factura PPD (RE-FU-018/019/020)

**Otras diferencias:**
- `tpPedido.FacturaPorAdelantado = 1`
- Activación directa sin código de autorización (Regla 3 — eliminar validación anterior si existe)
- Datos de facturación bloqueados y fijados al activar FAA

---

## Código Existente Relacionado

### Tramitación principal
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

### Proforma Adelanto (pendiente FAA)
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Anticipos\tpProformaAdelantoBO.cs`

> Entidad `tpProformaAdelanto` — representa el pendiente en módulo FAA. Ya existe como CrudBO.

### Factory de proforma
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs`

> Mismo comportamiento que RE-FU-013/014: genera proforma con `Folio = null` (foliador global lo asigna)

---

## Análisis — Flujo compartido con RE-FU-013/014

| Componente                      | Se desarrolla en | Aplica a RE-FU-015      |
| ------------------------------- | ---------------- | ----------------------- |
| Foliador lineal global          | RE-FU-013 T1     | Sí — mismo foliador     |
| Verificar por empresa           | RE-FU-013 T2     | Sí — misma verificación |
| Previsualización PDF            | RE-FU-013 T3     | Sí — mismo endpoint     |
| Envío correo proforma           | RE-FU-013 T4     | Sí — mismo endpoint     |
| Verificación Perú               | RE-FU-013 T6     | Sí — misma verificación |
| Vinculación PDF (RE-FU-016/017) | RE-FU-013 T7     | Sí — misma vinculación  |
| Validación Remisión Prepago     | RE-FU-014 T1     | Sí — misma validación   |
| Datos facturación solo lectura  | RE-FU-014 T2     | Sí — misma validación   |

---

## Gaps de Desarrollo Específicos de RE-FU-015

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Generación pendiente FAA al tramitar con FAA=1 | Al confirmar envío del correo, INSERT en `tpProformaAdelanto` con datos del pedido/cliente/empresa/monto. Atómico con la transacción | Medio |
| GAP-02 | NO generar pendiente Validar Cobro cuando FAA=1 | Verificar/ajustar que cuando `FacturaPorAdelantado=1`, NO se marca `MontoPendiente > 0` para Validar Cobro (o se omite la generación del pendiente VC). El pendiente VC lo genera el módulo FAA al emitir factura | Medio |
| GAP-03 | Eliminar código de autorización para FAA | Buscar y eliminar validación de código de autorización para activar Factura por Adelantado (Regla 3: activación directa) | Bajo |
| GAP-04 | Bloquear datos facturación al activar FAA | Fijar datos de facturación del catálogo del cliente vigente al momento de activar FAA. Rechazar edición posterior | Bajo |
| GAP-05 | Vinculación con módulo FAA (RE-FU-018/019/020) | Tarea para asegurar que el pendiente generado en `tpProformaAdelanto` sea consumido correctamente por el módulo FAA | Bajo |
| GAP-06 | Cancelación del pedido | Dependencia de R16A-RE-FU-010 (endpoint de cancelación) | Referencia |

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
5. Fija datos de facturación del catálogo del cliente
6. Genera proforma por empresa:
   - tpProformaPedido con Controlados=0
   - Folio = foliador lineal global (RE-FU-013 T1)
   - MontoPendiente = MontoTotal
   - INSERT tpPedidoProformaPedido + tpProformaPartidaPedido
7. Genera PDF vía DocumentBuilder (RE-FU-016/017)
8. Retorna PDF para previsualización (RE-FU-013 T3)
9. ESAC acepta -> Front llama endpoint de envío
10. Endpoint envío (RE-FU-013 T4):
    - Recibe: Para, CC, NotasExtras
    - Asunto = "Proforma " + FolioPedidoInterno
    - Adjunto = PDF
    - Envía correo
11. Al confirmar envío exitoso:
    - INSERT tpProformaAdelanto (pendiente FAA)
    - **NO genera pendiente Validar Cobro**
    - Cierra pendiente de Tramitar Pedido
12. tpPedido.Tramitado = 1
```

---

## Datos del pendiente FAA (tpProformaAdelanto)

| Dato | Origen |
|------|--------|
| IdCliente | tpPedido.IdCliente |
| IdEmpresa | tpPedido -> empresa de la proforma |
| Monto | MontoTotal de la proforma |
| IdCFDIGenerada | NULL (pendiente emisión) |
| Datos facturación | DatosFacturacionCliente fijados |
| Folio pedido | tpPedido.FolioPedidoInterno |
| Estado | Pendiente de emisión |

---

## Transferencia a Legacy

- En Tramitar Pedido NO hay transferencia para Prepago
- La transferencia ocurre después de Validar Cobro (fuera de scope)

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| R16A-RE-FU-006 | ReferenciaPago de la proforma |
| R16A-RE-FU-010 | Cancelación de pedido (endpoint compartido) |
| R16A-RE-FU-013 | Flujo base compartido: foliador, previsualización, envío, Perú |
| R16A-RE-FU-014 | Validaciones compartidas: Remisión Prepago, datos facturación |
| R16A-RE-FU-016 | Generación de PDF de proforma en DocumentBuilder |
| R16A-RE-FU-017 | Template/endpoint de proforma en DocumentBuilder |
| R16A-RE-FU-018/019/020 | Módulo FAA consume el pendiente generado aquí |

---

## Conclusión

El requisito R16A-RE-FU-015 tiene **impacto medio** en desarrollo Back. Reutiliza el flujo base de RE-FU-013/014 (proforma, foliador, previsualización, envío) pero agrega lógica específica:

1. **Pendiente FAA** (GAP-01) — INSERT en `tpProformaAdelanto` al tramitar
2. **No generar pendiente VC** (GAP-02) — comportamiento diferente al flujo sin FAA
3. **Sin código de autorización** (GAP-03) — eliminar validación anterior
4. **Bloqueo datos facturación** (GAP-04) — fijar al activar FAA
5. **Vinculación con FAA** (GAP-05) — asegurar consumo del pendiente por RE-FU-018/019/020

El desarrollador debe revisar `tpProformaAdelantoBO.cs` para entender la estructura del pendiente FAA y cómo se integra con el flujo de tramitación.
