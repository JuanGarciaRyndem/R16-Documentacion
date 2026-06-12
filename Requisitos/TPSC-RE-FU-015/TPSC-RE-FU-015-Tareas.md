# Tareas Back — TPSC-RE-FU-015
**Requisito:** Tramitación de pedidos Prepago con Factura por Adelantado

---

## T1 — [ TPSC-RE-FU-015 ] [SERV-TRANSACT] Generación de pendiente FAA al tramitar con Factura por Adelantado

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Liberar
- L05.TramitarPedido\Facturas\Anticipos

### Consideraciones previas
- El pendiente se genera al confirmar el envío exitoso del correo de proforma
- INSERT en `tpProformaAdelanto` con datos del pedido/cliente/empresa/monto
- Debe ser atómico con la transacción de tramitación
- La entidad `tpProformaAdelanto` ya existe como CrudBO

### Objetivo general
Implementar la generación del pendiente en el módulo Factura por Adelantado al tramitar un pedido Prepago con FAA activada.

### Objetivos específicos
- Modificar flujo en `tpPedidoTramitarTransaccionBO.cs` para detectar `FacturaPorAdelantado=1`
- INSERT en `tpProformaAdelanto` con: IdCliente, IdEmpresa, Monto, FolioPedido, DatosFacturación, Estado=Pendiente
- Asegurar atomicidad con la transacción de tramitación
- `IdCFDIGenerada = NULL` (pendiente de emisión por módulo FAA)

### Resultado esperado
Al tramitar un pedido Prepago con FAA=1, se genera automáticamente un pendiente en el módulo Factura por Adelantado con todos los datos necesarios para la emisión posterior de la factura.

### Entregables
- Modificación de `tpPedidoTramitarTransaccionBO.cs`
- Lógica de INSERT en `tpProformaAdelanto`

### Criterios de aceptación
- Pendiente FAA se genera al confirmar envío del correo
- Contiene todos los datos requeridos (cliente, empresa, monto, datos facturación)
- Es atómico con la transacción
- No se genera si FAA=0
- IdCFDIGenerada queda NULL (pendiente emisión)

---

## T2 — [ TPSC-RE-FU-015 ] [ALG-COMPLX-LOGIC] No generar pendiente Validar Cobro cuando FAA está activa

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- En flujo normal (RE-FU-014), al tramitar se genera pendiente Validar Cobro (MontoPendiente > 0)
- Con FAA=1, el pendiente VC NO debe generarse al tramitar
- El pendiente VC lo generará el módulo FAA cuando emita la factura PPD (RE-FU-018/019/020)

### Objetivo general
Ajustar la lógica de tramitación para que cuando FAA=1, NO se genere el pendiente en Validar Cobro.

### Objetivos específicos
- Identificar dónde se genera el pendiente Validar Cobro en el flujo actual
- Agregar condición: si `FacturaPorAdelantado=1` -> omitir generación de pendiente VC
- Verificar que `tpProformaPedido.MontoPendiente` se maneja correctamente (puede quedar > 0 pero sin disparar pendiente VC)
- Documentar que el pendiente VC será responsabilidad del módulo FAA

### Resultado esperado
Al tramitar con FAA=1, no se genera pendiente en Validar Cobro. El pendiente VC se generará posteriormente al emitir la factura.

### Entregables
- Ajuste en lógica de tramitación
- Documentación del cambio de comportamiento

### Criterios de aceptación
- Con FAA=1: NO hay pendiente en Validar Cobro al tramitar
- Con FAA=0: SÍ se genera pendiente en Validar Cobro (comportamiento normal)
- El cambio no afecta otros flujos (Crédito, Prepago sin FAA)

---

## T3 — [ TPSC-RE-FU-015 ] [ALG-BASIC-LOGIC] Eliminar código de autorización para Factura por Adelantado

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- Regla 3: la activación de FAA es directa, sin código de autorización
- Anteriormente se requería código para activar FAA
- Buscar y eliminar la validación si existe en el código actual

### Objetivo general
Eliminar la validación de código de autorización para la activación de Factura por Adelantado.

### Objetivos específicos
- Buscar en `tpPedidoTramitarTransaccionBO.cs` y archivos relacionados la validación de código de autorización para FAA
- Eliminar o deshabilitar la validación
- Confirmar que la activación es directa (Front envía `FacturaPorAdelantado=1` sin campo de código)

### Resultado esperado
La activación de FAA no requiere código de autorización.

### Entregables
- Eliminación de validación de código de autorización

### Criterios de aceptación
- No se solicita código al activar FAA
- La tramitación procede con FAA=1 sin validación adicional
- No se afectan otros flujos que pudieran usar códigos de autorización

---

## T4 — [ TPSC-RE-FU-015 ] [ALG-BASIC-LOGIC] Bloquear datos de facturación al activar FAA

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido
- WebApi.Logistica

### Consideraciones previas
- Regla 4: al activar FAA se fijan los datos de facturación del catálogo vigente del cliente
- Los datos fijados son: RFC, Razón Social, Uso CFDI, Método de Pago, Forma de Pago, Correo, Régimen Fiscal
- Rechazar edición posterior si FAA=1

### Objetivo general
Implementar el bloqueo y fijación de datos de facturación al activar Factura por Adelantado.

### Objetivos específicos
- Al tramitar con FAA=1: tomar datos de `DatosFacturacionCliente` vigente y fijarlos en el pedido/proforma
- Agregar validación en endpoint de edición: si `FacturaPorAdelantado=1` -> rechazar con error
- Los datos fijados quedan inmutables desde Tramitar Pedido

### Resultado esperado
Los datos de facturación se fijan al activar FAA y no pueden editarse posteriormente desde Tramitar Pedido.

### Entregables
- Lógica de fijación de datos al tramitar
- Validación en endpoint de edición

### Criterios de aceptación
- Datos se fijan del catálogo vigente al tramitar con FAA=1
- Error claro si se intenta editar datos con FAA activa
- Los datos fijados son los correctos (RFC, Razón Social, Uso CFDI, etc.)

---

## T5 — [ TPSC-RE-FU-015 ] [ALG-BASIC-LOGIC] Vinculación del pendiente FAA con módulo de facturación (RE-FU-018/019/020)

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Facturas\Anticipos

### Consideraciones previas
- La emisión de factura, CFDI y timbrado se desarrollan en RE-FU-018, RE-FU-019 y RE-FU-020
- Esta tarea asegura que el pendiente generado en T1 sea consumible por esos módulos
- Mismo patrón que RE-FU-012 T4

### Objetivo general
Vincular el proceso de generación del pendiente FAA con el flujo de facturación desarrollado en RE-FU-018/019/020.

### Objetivos específicos
- Verificar que la estructura de `tpProformaAdelanto` es compatible con lo esperado por RE-FU-018
- Ajustar campos/relaciones si es necesario para la integración
- Validar que el estado del pendiente se actualice correctamente cuando facturación lo consume
- Documentar contrato de datos entre pendiente y módulo de facturación

### Resultado esperado
El pendiente FAA generado en la tramitación es consumido correctamente por el módulo de facturación sin inconsistencias.

### Entregables
- Ajustes de integración (si aplican)
- Documentación del contrato de datos

### Criterios de aceptación
- El módulo FAA (RE-FU-018) puede leer y procesar el pendiente
- El estado del pendiente se actualiza al ser procesado
- No hay datos faltantes ni inconsistencias

---

---

## T6 — [ TPSC-RE-FU-015 ] [UPDATE-TABL-CH] ⛔ BLOQUEANTE — Crear catálogo catEstatusPedido e IdEstatusPedido en tpPedido (OBS-027)

> **⛔ BLOQUEANTE — OBS-027**
>
> Esta tarea está **BLOQUEADA** hasta recibir la propuesta del cliente sobre el catálogo `catEstatusPedido`.
>
> **Pendiente del cliente:**
> - Definición de estados requeridos para `catEstatusPedido` (clave, descripción, estados terminales, transiciones permitidas).
> - Confirmación de si `IdEstatusPedido` aplica en `tpPedido`, `ppPedido`, o ambas.
>
> **No iniciar desarrollo hasta que se resuelva el BLOQUEANTE.**
>
> Una vez desbloqueada, esta tarea incluirá:
> - `CREATE TABLE dbo.catEstatusPedido` con los campos definidos por el cliente.
> - `ALTER TABLE dbo.tpPedido ADD IdEstatusPedido uniqueidentifier NULL FK → catEstatusPedido`.
> - Scripts DDL + DML (INSERT de catálogo) con los estados acordados.

---

## T7 — [ TPSC-RE-FU-015 ] [ALG-COMPLX-LOGIC] ⛔ BLOQUEANTE — Lógica de transición de estados del pedido (OBS-027)

> **⛔ BLOQUEANTE — OBS-027**
>
> Esta tarea está **BLOQUEADA** hasta que T6 (catEstatusPedido + IdEstatusPedido) esté resuelta y desbloqueada.
>
> **Depende de:** T6 (catEstatusPedido definido por el cliente).
>
> **No iniciar desarrollo hasta que se resuelva el BLOQUEANTE.**
>
> Una vez desbloqueada, esta tarea incluirá:
> - Lógica de transición de estados en `tpPedidoTramitarTransaccionBO.cs` (o la clase que aplique) usando `IdEstatusPedido` → `catEstatusPedido`.
> - Validación de transiciones permitidas según el catálogo acordado.
> - Actualización de `tpPedido.IdEstatusPedido` en los puntos del flujo definidos.

---

## Resumen de tareas

| # | Clave Catálogo | Título | Predecesora |
|---|----------------|--------|-------------|
| T1 | SERV-TRANSACT | Generación de pendiente FAA al tramitar con Factura por Adelantado | — |
| T2 | ALG-COMPLX-LOGIC | No generar pendiente Validar Cobro cuando FAA está activa | T1 |
| T3 | ALG-BASIC-LOGIC | Eliminar código de autorización para Factura por Adelantado | — |
| T4 | ALG-BASIC-LOGIC | Bloquear datos de facturación al activar FAA | — |
| T5 | ALG-BASIC-LOGIC | Vinculación del pendiente FAA con módulo de facturación (RE-FU-018/019/020) | T1 |
| T6 | UPDATE-TABL-CH | ⛔ BLOQUEANTE — Crear catEstatusPedido + IdEstatusPedido en tpPedido (OBS-027) | — |
| T7 | ALG-COMPLX-LOGIC | ⛔ BLOQUEANTE — Lógica de transición de estados del pedido (OBS-027) | T6 |

---

## Dependencias con otros requisitos (NO incluidas como tareas)

| Requisito | Tarea relacionada | Relación |
|-----------|-------------------|----------|
| TPSC-RE-FU-010 | Cancelación | Endpoint de cancelación se desarrolla en RE-FU-010 |
| TPSC-RE-FU-013 | T1-T4, T6, T7 | Foliador, previsualización, envío correo, empresa, Perú, vinculación PDF |
| TPSC-RE-FU-014 | T1, T2 | Validación Remisión Prepago, datos facturación solo lectura |
| TPSC-RE-FU-016 | Generación PDF | PDF de proforma en DocumentBuilder |
| TPSC-RE-FU-017 | Template PDF | Template de proforma en DocumentBuilder |
| TPSC-RE-FU-018/019/020 | Módulo FAA | Consume el pendiente generado en T1 |
