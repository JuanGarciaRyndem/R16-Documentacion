# Tareas BackEnd — TPSC-RE-FU-011
**Requisito:** Tramitacion de pedidos Credito con sustancias controladas
**Modulo:** L05.TramitarPedido
**Total de tareas:** 3

---

## Dependencias de otros requisitos

| Requisito | Tarea | Descripcion |
|-----------|-------|-------------|
| TPSC-RE-FU-007 | T1 | ALTER FUNCTION fnEsProductoControlado para incluir clave 'origen' |
| TPSC-RE-FU-010 | T3 | Endpoint de Cancelacion de pedido desde Tramitar Pedido |

---

## Tarea 1

**Titulo:** [ TPSC-RE-FU-011 ] [ALG-BASIC-LOGIC] Validacion Back de restriccion regulatoria FAA y Remision con controlados

**Aplicativos:**
ProquifaNet 2

**Modulos:**
Tramitar Pedido — Logic.Pqf.Logistica\L05.TramitarPedido\Liberar

**Consideraciones previas:**
- El Frontend oculta las opciones Factura por Adelantado y Entrega con Remision cuando el pedido tiene sustancias controladas
- Se requiere una validacion de seguridad en Back para prevenir bypass via llamada directa al API
- La deteccion de controlados ya existe en el flujo (`vProducto.Controlado`)
- El punto de insercion es en `tpPedidoTramitarTransaccionBO.GenerarCorreoTramitarPedido()`, despues de las validaciones existentes y antes de la generacion de folio
- Depende de: TPSC-RE-FU-007 T1 (fnEsProductoControlado con 'origen')

**Objetivo general:**
Agregar validacion en el flujo de tramitacion que rechace la operacion si el pedido tiene sustancias controladas y se intenta tramitar con FacturaPorAdelantado=1 o EntregaConRemision=1.

**Objetivos especificos:**
- Identificar el punto exacto en `GenerarCorreoTramitarPedido()` donde insertar la validacion (despues de cargar partidas y detectar controlados)
- Implementar la logica: si `tieneControlados AND (tpPedido.FacturaPorAdelantado OR tpPedido.EntregaConRemision)` -> rechazar con mensaje de validacion
- Agregar mensaje descriptivo: "No se permite Factura por Adelantado ni Entrega con Remision en pedidos con sustancias controladas"
- Ejecutar pruebas: pedido con controlados + FAA=1 (rechazado), pedido con controlados + FAA=0 (aceptado)

**Resultado esperado:**
El sistema rechaza la tramitacion si se detectan sustancias controladas junto con FacturaPorAdelantado=1 o EntregaConRemision=1, retornando Response con status false y mensaje de validacion.

**Entregables:**
- `Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs` (modificado)
- Pruebas unitarias de la validacion

**Criterios de aceptacion:**
- [ ] Pedido con controlados + FacturaPorAdelantado=1 es rechazado con mensaje claro
- [ ] Pedido con controlados + EntregaConRemision=1 es rechazado con mensaje claro
- [ ] Pedido con controlados + FAA=0 + Remision=0 se tramita normalmente
- [ ] Pedido sin controlados + FAA=1 se tramita normalmente (no afecta flujo RE-FU-010)
- [ ] La validacion se ejecuta antes de cualquier escritura en BD (no deja datos inconsistentes)
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- GAP-02 del archivo `TPSC-RE-FU-011-Back.md`
- Criterio B1 del requisito (restriccion regulatoria)
- Regla 2 del requisito

**Recursos:**
- TPSC-RE-FU-011-Back.md (Seccion C)
- TPSC-RE-FU-011.md (Regla 2, Criterio B1)
- `Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

---

## Tarea 2

**Titulo:** [ TPSC-RE-FU-011 ] [IMP-EXIST-SERVICE] Verificar y asegurar asignacion de Controlados=1 en tpProformaPedido

**Aplicativos:**
ProquifaNet 2

**Modulos:**
Tramitar Pedido — Logic.Pqf.Logistica\L05.TramitarPedido

**Consideraciones previas:**
- La tabla `tpProformaPedido` tiene el campo `Controlados` (bit) que debe marcarse en 1 cuando el pedido tiene sustancias controladas
- El flujo de generacion de proforma ya existe — se debe verificar si asigna correctamente este campo
- Si no lo asigna, se debe implementar la logica

**Objetivo general:**
Verificar que al generar la proforma (Confirmacion de Pedido) para un pedido con sustancias controladas, el campo `tpProformaPedido.Controlados` se asigne en 1. Si no existe esta logica, implementarla.

**Objetivos especificos:**
- Localizar el codigo de generacion de proforma en el flujo de tramitacion
- Verificar si `tpProformaPedido.Controlados` se asigna basado en la deteccion de partidas controladas
- Si no existe la asignacion, implementarla reutilizando la deteccion de controlados ya existente
- Ejecutar prueba: tramitar pedido con controlados y verificar que `tpProformaPedido.Controlados = 1`

**Resultado esperado:**
La proforma generada para pedidos con sustancias controladas tiene `Controlados = 1`. Para pedidos sin controlados, `Controlados = 0`.

**Entregables:**
- Evidencia de verificacion (si ya funciona correctamente)
- O modificacion en el BO de generacion de proforma (si requiere ajuste)

**Criterios de aceptacion:**
- [ ] Pedido con controlados genera proforma con `Controlados = 1`
- [ ] Pedido sin controlados genera proforma con `Controlados = 0`
- [ ] La asignacion ocurre dentro de la transaccion existente
- [ ] PR aprobado (si hubo cambio) o evidencia documentada (si ya funcionaba)

**Mas informacion de la tarea:**
- GAP-03 del archivo `TPSC-RE-FU-011-Back.md`
- Tabla: `tpProformaPedido` campo `Controlados`
- Criterio A1 del requisito

**Recursos:**
- TPSC-RE-FU-011-Back.md (Seccion E)
- TPSC-RE-FU-011_BD.md (Campos Clave)
- `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\tpPedidoProformaPedidoBO.cs`

---

## Tarea 3

**Titulo:** [ TPSC-RE-FU-011 ] [IMP-EXIST-SERVICE] Verificacion del flujo completo Credito con controlados (MEX y PER)

**Aplicativos:**
ProquifaNet 2

**Modulos:**
Tramitar Pedido — Logic.Pqf.Logistica\L05.TramitarPedido\Liberar

**Consideraciones previas:**
- El flujo de tramitacion Credito ya existe y ya detecta partidas controladas (envia correo a Regulatory Affairs)
- La condicion Peru sin transferencia Legacy ya esta implementada (`region.Clave == Constants.Regiones.Mexico`)
- Esta tarea es de verificacion integral del flujo con controlados tras las Tareas 1 y 2
- Depende de: TPSC-RE-FU-007 T1 (fnEsProductoControlado), Tarea 1 y 2 de este requisito

**Objetivo general:**
Verificar que el flujo completo de tramitacion Credito con sustancias controladas funciona correctamente para Mexico (con transferencia Legacy) y Peru (sin transferencia Legacy), incluyendo las restricciones implementadas.

**Objetivos especificos:**
1. Tramitar pedido Credito con controlados en region Mexico — verificar folio, PDF, proforma (Controlados=1), correo cliente, correo Regulatory Affairs, transferencia Legacy
2. Tramitar pedido Credito con controlados en region Peru — verificar que NO transfiere a Legacy
3. Tramitar pedido Credito Pago contra entrega con controlados (Mexico) — verificar marca de detencion en Legacy
4. Verificar que FAA y Remision quedan en 0 tras la tramitacion
5. Verificar cierre de pendiente en bandeja de Tramitar Pedido

**Resultado esperado:**
El flujo completo opera correctamente en ambas regiones, con todas las restricciones regulatorias aplicadas y la transferencia a Legacy condicionada por region.

**Entregables:**
- Evidencia de pruebas del flujo en ambas regiones
- Documentacion de casos de prueba ejecutados

**Criterios de aceptacion:**
- [ ] Mexico: pedido tramitado correctamente con transferencia a Legacy
- [ ] Peru: pedido tramitado correctamente SIN transferencia a Legacy
- [ ] Proforma generada con Controlados=1
- [ ] Correo a Regulatory Affairs enviado
- [ ] Pago contra entrega: marca de detencion incluida en transferencia
- [ ] FAA=0 y EntregaConRemision=0 en tpPedido tras tramitacion
- [ ] Pendiente cerrado en bandeja
- [ ] Cancelacion funciona (dependencia RE-FU-010 T3)

**Mas informacion de la tarea:**
- Criterios A1, A2, A3, C2, C3 del requisito
- Reglas 1, 3, 4, 5 del requisito
- Depende de Tareas 1 y 2 de este requisito

**Recursos:**
- TPSC-RE-FU-011-Back.md (todas las secciones)
- TPSC-RE-FU-011.md (Criterios de Aceptacion)

---

## Resumen de Tareas

| # | Clave Catalogo | Descripcion | Tipo |
|---|----------------|-------------|------|
| T1 | ALG-BASIC-LOGIC | Validacion Back restriccion FAA/Remision con controlados | Desarrollo nuevo |
| T2 | IMP-EXIST-SERVICE | Verificar/asegurar Controlados=1 en tpProformaPedido | Verificacion/Ajuste |
| T3 | IMP-EXIST-SERVICE | Verificacion flujo completo MEX y PER | Verificacion |
