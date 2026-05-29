# 03 — R16 Tramitar Pedido sin Crédito: 3ra Sesión de Entendimiento / Presentación de Pantallas

| Campo | Detalle |
|---|---|
| **Fecha** | 6 de abril de 2026 |
| **Hora** | 14:55 CST |
| **Sesión** | 3ra Sesión de Entendimiento |
| **Tema principal** | Presentación de pantallas — Entrega con Remisión y Factura por Adelantado |
| **Proyecto** | R16 — Tramitar Pedido Sin Crédito |

---

## Participantes

| Nombre | Área / Rol |
|---|---|
| Rose Ríos Gómez | Líder de proyecto / Ryndem |
| Roberto Baez Muñoz | Desarrollo / Ryndem |
| Juan David García Cruz | Desarrollo / Ryndem |
| Alan Fernandez Garcia | Desarrollo / Ryndem |
| Irma Andrade Aguado | Desarrollo / Ryndem |
| Francisco Uriel Guerrero Rivera | Desarrollo / Ryndem |
| Jose Antonio Chavez Amador | Coordinador QA / Ryndem |
| Valdemar Farina Sanchez | Coordinador de Desarrollo / Ryndem |
| Larissa Calvo | Negocio / Proquifa |
| Sara Sánchez | Negocio / Proquifa |
| Biridiana Arias | Finanzas / Proquifa |

---

## Resumen Ejecutivo

El equipo revisó los flujos de las opciones **Entrega con Remisión** y **Factura por Adelantado** dentro del módulo Tramitar Pedido. Se acordó que ambas opciones son **mutuamente excluyentes** y se implementarán como **radio buttons** (seleccionar una, la otra, o ninguna). También se discutieron las reglas de edición de datos fiscales, la vigencia del código de verificación y el foliador de proformas.

---

## Temas Tratados

### 1. Entrega con Remisión — Reglas y funcionamiento

- La opción **Entrega con Remisión** anula las restricciones de entrega de los últimos días del mes, permitiendo que la fecha de entrega caiga en días bloqueados.
- Cuando está marcada, el sistema de entrega genera una **nota de remisión** en lugar de una factura.
- La marca **debe ser establecida manualmente** por el usuario en cada pedido; no se hereda automáticamente del catálogo del cliente.
- Si el campo está marcado, la fecha estimada de entrega puede caer en los últimos días del mes. Si no está marcado, esos días se bloquean y la fecha se mueve al mes siguiente.
- Marcar el campo es **opcional**; no obliga al usuario a continuar.

### 2. Exclusión mutua: Factura por Adelantado vs. Entrega con Remisión

- Ambas opciones son **excluyentes**: se elige una, la otra, o ninguna.
- Se cambiará el diseño de casillas de verificación a **radio buttons** para garantizar la selección excluyente.
- Para pedidos con **productos controlados**, ambas opciones se **bloquearán completamente**, manteniendo el flujo actual.
- Ambas opciones requerirán **autorización del mismo rol** (Finanzas).

### 3. Código de verificación — Vigencia de 24 horas

- Se discutió la funcionalidad de que el código de verificación de la proforma tenga una **vigencia de 24 horas**.
- Se solicitó estimar el desarrollo para esta funcionalidad, contemplando dos versiones:
  - Funcionamiento normal (expiración sin opción de retroceso).
  - Versión que permite regresar a la pantalla anterior al expirar.

### 4. Edición de datos fiscales en el pedido

- Existe una opción **“Editar datos”** en el sistema actual (para pedidos crédito) que permite al ESAC, con autorización de Finanzas, modificar el **uso de CFDI** y el **método de pago**.
- **Opinión de Finanzas (Biridiana):** Solo el área financiera debería estar facultada para editar estos datos, ya que errores en método de pago o uso de CFDI tienen implicaciones legales ante el SAT.
- **Propuesta:** Al activar Factura por Adelantado, el sistema cambiaría automáticamente el método de pago de **PUE a PPD**.
- **Edición de Razón Social y RFC:** Aplica a nivel de **catálogo de cliente**, no a nivel de pedido. La edición en pedido se limita a uso de CFDI y método de pago.
- **Conclusión:** El equipo de Proquifa revisará si se mantienen, eliminan o reubican los campos de edición.
- Se recuerda considerar el **alcance para Perú** en estos flujos.

### 5. Foliador de proformas

- Biridiana sugirió que el foliador de proformas es un **contador lineal**, no por empresa.
- Pendiente confirmar a nivel código que el contador de folios es lineal.
- Pendiente que el equipo proporcione las **reglas del foliador** de proformas.

---

## Pendientes y Compromisos

| # | Responsable | Compromiso | Fecha límite |
|---|---|---|---|
| 1 | Irma Andrade Aguado | Agregar a lista de tareas: equipo proporcione reglas del foliador de proformas | Próxima sesión |
| 2 | Sara Sánchez | Confirmar a nivel código que el contador de folios es lineal y no se maneja por empresa | Próxima sesión |
| 3 | Biridiana Arias | Revisar y proporcionar la plantilla de correo existente que se envía al cliente | Próxima sesión |
| 4 | Rose Ríos Gómez | Estimar desarrollo para vigencia del código de verificación de 24 horas (ambas versiones) | Próxima sesión |
| 5 | Biridiana / Sara Sánchez / Larissa Calvo | Revisar impactos de órdenes de compra en flujos Factura por Adelantado y Entrega con Remisión | Próxima sesión |
| 6 | Larissa Calvo | Gestionar programación de próxima sesión asegurando asistencia de Área de Cuentas por Cobrar | A la brevedad |
| 7 | Biridiana / Sara Sánchez | Definir qué campos (Razón Social, RFC) pueden editarse en el trámite de pedido y cuáles deben gestionarse a nivel catálogo | Próxima sesión |

---

## Próxima Sesión

- **Temas a revisar:** Conclusión sobre edición de datos fiscales, módulo de Factura por Adelantado con participación del Área de Cuentas por Cobrar.
- **Coordinación:** Larissa Calvo.
