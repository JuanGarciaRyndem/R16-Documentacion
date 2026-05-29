# 04 — R16 Tramitar Pedido sin Crédito: 4ta Sesión — Revisión de Pantallas (Continuación)

| Campo | Detalle |
|---|---|
| **Fecha** | 8 de abril de 2026 |
| **Hora** | 15:52 CST |
| **Sesión** | 4ta Sesión de Entendimiento |
| **Tema principal** | Módulo Factura por Adelantado — moneda, tipo de cambio y campos editables |
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
| Cornelio M. Ramírez B. | Finanzas / Proquifa |
| Mayra Franco | Finanzas / Proquifa |
| Biridiana Arias | Finanzas / Proquifa |

---

## Resumen Ejecutivo

La sesión se centró en la revisión del módulo **Factura por Adelantado**, abordando la moneda de la Proforma, las reglas del tipo de cambio y la definición final de qué campos serán editables. Se concluyó que la mayoría de los campos serán de **solo lectura**, con excepción del **Uso de CFDI**.

---

## Temas Tratados

### 1. Moneda de la Proforma y configuración del catálogo de clientes

- La moneda de la Proforma debe reflejar la **moneda de facturación** configurada en el catálogo del cliente.
- El catálogo de clientes puede tener dos monedas: **moneda de la oferta (cotización)** y **moneda de facturación**. La Proforma usa la de facturación.
- Para **prepagos de productos controlados**, la factura generada es una **factura de anticipo**, pero sí se genera una Proforma.

### 2. Tipo de cambio — Proforma y validación de pago

- Si el pedido tiene una moneda diferente a la de facturación, la Proforma debe obedecer a la misma moneda del pedido para evitar incongruencias con la factura final.
- Si la moneda de facturación es en pesos y los precios están dolarizados, se utiliza el **tipo de cambio del día** de generación de la Proforma.
- El sistema debe registrar la **trazabilidad del tipo de cambio**, ya que el mismo se utilizará para la emisión final de la factura tras la validación del pago.
- La regla sobre qué tipo de cambio usar al **validar el pago** (el de la Proforma vs. el del día del pago) quedó pendiente de revisión por el equipo.

### 3. Vista resumen y vista detalle del módulo Factura por Adelantado

- **Vista resumen:** Listado agrupado por cliente con RFC, facturas pendientes por generar y monto total. Incluye buscador por cliente o RFC. El monto total se muestra en **dólares** para mejorar usabilidad.
- **Vista detalle:** Al seleccionar un cliente muestra sus datos fiscales. La información del **contacto del comprador** debe aparecer por línea de pedido (no en la cabecera), ya que puede haber múltiples compradores.

### 4. Campos editables en el modal de generación de factura

- Se discutió si la **moneda de facturación** podría ser editable. Conclusión: ya viene definida del catálogo, no es editable.
- La **forma de pago PPD** se establece automáticamente como “por definir” y debe visualizarse en todo momento, pero **bloqueada y de solo lectura**.
- El **importe** debe ser más llamativo y visible, posiblemente cambiando su ubicación en la interfaz cerca de la opción de generar factura.
- El **tipo de cambio** no se modificará por defecto. Se usa el del sistema. Se descartó hacerlo editable para cubrir el 98% de los casos.
- **Decisión final:** El único campo editable será el **Uso de CFDI**. Todos los demás campos serán de solo lectura.
- Se agregará el campo **Razón Social de la empresa que factura** en el modal para evitar refacturaciones.

---

## Pendientes y Compromisos

| # | Responsable | Compromiso | Fecha límite |
|---|---|---|---|
| 1 | Larissa Calvo | Revisar regla de tipo de cambio al validar pago: ¿se usa el de la Proforma o el del día del pago? | Próxima sesión |
| 2 | Valdemar Farina Sanchez | Incluir dato de contacto del comprador y correo electrónico en cada línea de pedido pendiente (vista detalle del cliente) | Próxima sesión |
| 3 | Valdemar Farina Sanchez | Configurar funcionalidad de búsqueda por coincidencia (no exacta) para pedidos | Próxima sesión |
| 4 | Valdemar Farina Sanchez | Incluir totalizadores en pantalla inicial: total de clientes, total de facturas pendientes y monto total | Próxima sesión |
| 5 | Valdemar Farina Sanchez | Agregar campo Razón Social de la empresa que factura en el modal de generación de factura | Próxima sesión |
| 6 | Roberto Baez Muñoz | Mostrar forma de pago en el modal, bloqueado y de solo lectura | Próxima sesión |
| 7 | Roberto Baez Muñoz | Mejorar visualización del importe: cambiar posición o destacarlo visualmente | Próxima sesión |

---

## Próxima Sesión

- **Temas a revisar:** Validación del pago, reglas del tipo de cambio. Duración sugerida: **2 horas**.
- **Nota:** Se enfatizó la importancia de la asistencia del equipo de finanzas, especialmente para la validación del pago.
