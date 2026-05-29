# Borrador - Analisis de Flujo y Reglas de Tramitacion de Pedidos
## Cobranza de Prepagos y Facturacion por Adelantado

> **Fuente:** Borrador - Analisis de flujo y reglas de Tramitacion de pedidos con cobranza de prepagos y facturacion por adelantado
> **Requisitos referenciados:** R16M-RE-FU-002, R16M-RE-FU-006, R16M-RE-FU-011, R16M-RE-FU-015

---

## 1. Resumen del Documento

Este borrador define el comportamiento del sistema segun el **tipo de cliente** (Credito / Prepago),
la presencia de **productos controlados** y la seleccion de **Factura por Adelantado (FA)**,
en tres puntos de decision clave del flujo de tramitacion de pedidos.

---

## 2. Acciones al Dar Click en "Tramitar Pedido" (Primera vez)

| # | Escenario | Acciones del sistema |
|---|---|---|
| 1 | Cliente **Credito** + productos **controlados** | Genera Folio PI → Tramita el pedido → Genera confirmacion de pedido → Transfiere a Legacy |
| 2 | Cliente **Credito** sin controlados + **sin FA** | Genera Folio PI → Tramita el pedido → Genera confirmacion de pedido → Transfiere a Legacy |
| 3 | Cliente **Credito** sin controlados + **con FA** | Genera pendiente en **Facturar por Adelantado** |
| 4 | Cliente **Prepago** + productos **controlados** | Genera Folio PI *(R16M-RE-FU-015)* → Genera Proforma *(R16M-RE-FU-006)* → Genera pendiente en **Validar Pago** *(R16M-RE-FU-002)* |
| 5 | Cliente **Prepago** sin controlados + **sin FA** | Genera Folio PI → Genera Proforma → Genera pendiente en **Validar Pago** |
| 6 | Cliente **Prepago** sin controlados + **con FA** | Genera Folio PI → Genera pendiente en **Facturar por Adelantado** |

### Observaciones
- Los escenarios 1 y 2 son identicos en comportamiento: el cliente de credito va directo a tramitar sin documento previo.
- Los escenarios 4 y 5 son identicos: todo prepago sin FA genera proforma y bloquea en Validar Pago.
- La Factura por Adelantado en credito **no genera proforma**; va directo al modulo de FA.
- La Factura por Adelantado en prepago **tampoco genera proforma**; va directo al modulo de FA.
- El Folio PI *(R16M-RE-FU-015)* se genera en Pretramitar para todos los escenarios de prepago
  y en Tramitar para credito directo.

---

## 3. Acciones al Dar Click en "Generar Factura por Adelantado"

| # | Escenario | Acciones del sistema |
|---|---|---|
| 1 | Cliente **Credito** | Genera factura normal (PPD) → Establece FEE *(R16M-RE-FU-011)* → Desbloquea pendiente en Tramitar Pedido |
| 2 | Cliente **Prepago** | Genera factura normal (PPD) → Genera pendiente en **Validar Pago** |

### Observaciones
- El comportamiento es **distinto** segun el tipo de cliente:
  - **Credito:** la FA desbloquea directamente el pedido para tramitarse. La FEE se establece aqui.
  - **Prepago:** la FA **no desbloquea** el pedido; aun requiere pasar por Validar Pago para confirmar el pago recibido.
- En ambos casos la factura se genera con **metodo de pago PPD**.
- La FEE *(R16M-RE-FU-011)* solo se establece en este paso para el cliente de **Credito**.

---

## 4. Acciones al Dar Click en "Validar Pago"

| # | Escenario | Acciones del sistema |
|---|---|---|
| 1 | Cliente **Prepago** + controlados | Genera **factura anticipo** → Establece FEE *(R16M-RE-FU-011)* → Desbloquea pendiente en Tramitar Pedido |
| 2 | Cliente **Prepago** sin controlados + **sin FA** | Genera **factura normal** → Establece FEE *(R16M-RE-FU-011)* → Desbloquea pendiente en Tramitar Pedido |
| 3 | Cliente **Prepago** sin controlados + **con FA** | Genera **complemento de pago** → Actualiza estatus de factura adelantada → Establece FEE *(R16M-RE-FU-011)* → Desbloquea pendiente en Tramitar Pedido |

### Observaciones
- Los tres escenarios terminan en el mismo resultado: **FEE establecida + pedido desbloqueado**.
- La diferencia esta en el **tipo de documento generado**:
  - Controlados → **Factura Anticipo**
  - Sin controlados sin FA → **Factura Normal**
  - Sin controlados con FA → **Complemento de Pago** (cierra el ciclo de la FA previa)
- El complemento de pago ademas **actualiza el estatus** de la factura generada por adelantado.
- La FEE *(R16M-RE-FU-011)* se establece en este paso para **todos los escenarios de prepago**.

---

## 5. Acciones al Dar Click en "Tramitar Pedido" (Pendiente Desbloqueado)

| # | Escenario | Acciones del sistema |
|---|---|---|
| 1 | Cliente **Credito** + controlados o sin FA | *(Este escenario no sucede - el pedido ya fue tramitado en el primer click)* |
| 2 | Cliente **Credito** con FA | Ya tiene FEE → Genera Folio PI → Tramita el pedido → Genera confirmacion → Transfiere a Legacy |
| 3 | **Todos los escenarios Prepago** | Ya tiene FEE → Ya tiene Folio PI → Tramita el pedido → Genera confirmacion → Transfiere a Legacy |

### Observaciones
- Para **credito con FA**: el Folio PI se genera en este segundo momento (no en Pretramitar).
- Para **prepago**: el Folio PI ya existe desde Pretramitar *(R16M-RE-FU-015)*, no se regenera.
- La FEE ya esta establecida en pasos anteriores en todos los casos; aqui solo se usa.
- La confirmacion de pedido y la transferencia a Legacy son identicas para todos los escenarios desbloqueados.

---

## 6. Matriz Resumen de Documentos Generados por Escenario

| Escenario             | Proforma | Factura Normal | Factura Anticipo | Complemento Pago | FEE en                  |
| --------------------- | :------: | :------------: | :--------------: | :--------------: | ----------------------- |
| Credito + Controlados |    -     |       -        |        -         |        -         | Tramitar (directo)      |
| Credito sin FA        |    -     |       -        |        -         |        -         | Tramitar (directo)      |
| Credito con FA        |    -     |     ✅ PPD      |        -         |        -         | Facturar por Adelantado |
| Prepago + Controlados |    ✅     |       -        |        ✅         |        -         | Validar Pago            |
| Prepago sin FA        |    ✅     |       ✅        |        -         |        -         | Validar Pago            |
| Prepago con FA        |    -     |     ✅ PPD      |        -         |        ✅         | Validar Pago            |

---

## 7. Requisitos Referenciados

| Requisito          | Descripcion                                                                                      |
| ------------------ | ------------------------------------------------------------------------------------------------ |
| **R16M-RE-FU-002** | Validar Pago: el pago puede aplicarse a varias facturas; monto no puede exceder saldo disponible |
| **R16M-RE-FU-006** | Generar Proforma para clientes de tipo Prepago                                                   |
| **R16M-RE-FU-011** | Establecer Fecha Estimada de Entrega (FEE) al liberar el candado del pedido                      |
| **R16M-RE-FU-015** | Generar Folio PI automaticamente al finalizar Pretramitar Pedido para clientes Prepago           |

---

## 8. Puntos de Atencion Identificados

| # | Punto | Detalle |
|---|---|---|
| 1 | **Folio PI en Credito con FA** | Se genera al tramitar el pedido desbloqueado, no en Pretramitar. Diferente al comportamiento de Prepago. |
| 2 | **FEE en Credito directo** | Se establece al momento de tramitar directamente, sin paso previo. No aplica R16M-RE-FU-011 explicitamente en este escenario segun el borrador. |
| 3 | **Complemento de Pago** | Solo aplica para Prepago sin controlados con FA. Es el unico escenario que genera complemento en Validar Pago. |
| 4 | **Factura Anticipo vs Factura Normal** | Prepago con controlados genera **Anticipo**; Prepago sin controlados sin FA genera **Normal**. La diferencia esta en la naturaleza del producto. |
| 5 | **Credito con FA no pasa por Validar Pago** | A diferencia de Prepago con FA, el credito con FA desbloquea directamente en el modulo de FA, sin necesidad de validar un cobro. |
