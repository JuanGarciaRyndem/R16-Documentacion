# 05 — R16 Tramitar Pedido sin Crédito: 5ta Sesión — Revisión de Pantallas (Continuación)

| Campo | Detalle |
|---|---|
| **Fecha** | 9 de abril de 2026 |
| **Hora** | 14:59 CST |
| **Sesión** | 5ta Sesión de Entendimiento |
| **Tema principal** | Módulo Factura por Adelantado (modal) y Módulo Validación de Pago (prepago) |
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
| Jose Antonio Chavez Amador | Coordinador QA / Ryndem |
| Valdemar Farina Sanchez | Coordinador de Desarrollo / Ryndem |
| Larissa Calvo | Negocio / Proquifa |
| Sara Sánchez | Negocio / Proquifa |
| Biridiana Arias | Finanzas / Proquifa |
| Mayra Franco | Finanzas / Proquifa |
| Cornelio M. Ramírez B. | Finanzas / Proquifa |

---

## Resumen Ejecutivo

La sesión cubrió el **modal de generación de Factura por Adelantado** (folio interno, flujo de timbrado, asunto del correo) y el **módulo de Validación de Pago para clientes prepago** (clasificación automática, captura del cobro, escenarios de subpago, notas de crédito y facturas de anticipo). Se identificaron políticas pendientes de definir para condonación de montos y estados de cuenta.

---

## Temas Tratados

### 1. Modal “Generar Factura por Adelantado” — Mejoras visuales

- Se realizaron cambios en el modal: el **total de la factura** se muestra más grande y visible.
- Se agregaron **datos de contacto** y **datos fiscales del cliente** en modo de solo lectura, excepto el **Uso de CFDI** (editable).
- El flujo incluye la opción de **generar y previsualizar la factura** antes del timbrado.
- Si el timbrado falla, se muestra un error (catálogo de errores pendiente de definir).
- Si la generación es exitosa, se procede al modal de **envío de la factura por correo**.

### 2. Asunto del correo y folio interno de factura

- El **asunto del correo** de envío debe incluir: **folio del pedido interno + folio de la factura**.
- El **folio interno** de la factura es un autoincrementable por empresa (campo con límite máximo en BD, sin reinicio).
- Se propuso evaluar un **folio independiente** para facturas de prepago en ProquifaNet 2, para evitar dependencia con sistemas legados.

### 3. Alcance del lanzamiento — Clientes crédito vs. prepago

- El alcance de este lanzamiento cubre hasta la **factura por adelantado en ProquifaNet 2**.
- El flujo posterior (cargar la factura a revisión) continúa en **sistemas legados**.
- Las facturas timbradas seguirán apareciendo en el aplicativo **“Contador” de Legacy**, con posible retraso.

### 4. Módulo de Validación de Pago — Solo clientes prepago

- El módulo aplica únicamente para **clientes prepago** y **sustituye la validación de cobro en SAP**.
- La única modificación en el buzón de cobros existente es la **clasificación automática** (por palabras clave), con posibilidad de reclasificar.
- **Vista resumen:** Listado por cliente con RFC, cobros recibidos y facturas/proformas pendientes de cobrar.
- Se requiere una vista **coordinadora para Tesorería** donde el coordinador vea todos los pagos y el estado de procesamiento de su cartera de analistas.

### 5. Captura del cobro — Paso 1

- El lado izquierdo muestra los cobros del buzón. Se define un **folio interno de cobro** (propuesta: prefijo COB + secuencial).
- Rose Ríos Gómez enfatizó que el folio debe ser **consecutivo e independiente** por transacción.
- Si existe inconsistencia entre el cuerpo del correo y el comprobante adjunto, se **marca la inconsistencia** y se detiene el flujo.
- En el nuevo flujo de prepago, el área de **Tesorería** lleva todo el proceso.

### 6. Escenarios de pago — Subpago, saldo a favor y cancelación

- **Saldo a favor:** Fiscalmente correcto es generar **notas de crédito**. Si el saldo no se sabe a qué asignar, se emite una **factura de anticipo** que luego se cancela con nota de crédito.
- **Subpago:** Si el cliente deposita de menos, se le contacta para el faltante. Si el faltante es significativo (ej. +$5,000 MXN), se asocian **dos cobros a la misma proforma**.
- **Cancelación por subpago total:** Se propone un botón **“Cancelar pedido”** en el flujo de validación de cobro.
- **Condonación de montos menores:** No existe política formal. Se requeriría autorización del especialista de CxC y coordinador.

### 7. Notas de crédito — Integración y generación en PQF2

- Las notas de crédito se generan actualmente **fuera del sistema Legacy**, lo que dificulta su conexión.
- Se propone una pantalla para **generar notas de crédito directamente en ProquifaNet 2**.
- Se solicitó a Sara Sánchez compartir un ejemplo de nota de crédito (PDF y XML).

### 8. Paso 3 — Facturación y envío

- Tanto **PPD** como **PUE** están disponibles como métodos de pago en las proformas. PPD requiere complemento de pago.
- Estatus: **pendiente → factura/complemento generado → enviado**. Una vez enviado no se puede reenviar desde este flujo.
- Se propone un **módulo de facturas** para consulta, descarga y reenvío posterior.

### 9. Tramitación del pedido post-validación

- Una vez validado el cobro, el **pedido prepago se desbloquea** en el módulo de tramitación.
- Se envía la **factura de forma independiente** y luego la **confirmación del pedido** al tramitarlo.

---

## Pendientes y Compromisos

| # | Responsable | Compromiso | Fecha límite |
|---|---|---|---|
| 1 | Sara Sánchez | Compartir reglas técnicas y campos de BD utilizados para folios por empresa | Próxima sesión |
| 2 | Sara Sánchez | Evaluar folio de factura independiente para prepagos con Biridiana y Mayra | Próxima sesión |
| 3 | El grupo | Especificar punto de inserción en sistema legado para la factura por adelantado | Próxima sesión |
| 4 | Sara Sánchez / Biridiana Arias | Proporcionar listado de correos representativos de cobros para entrenamiento del clasificador automático | Próxima sesión |
| 5 | El grupo | Evaluar incorporación de estado de cuenta auxiliar con emisión automática de cartas por saldos a favor | Próxima sesión |
| 6 | Mayra Franco | Definir escenarios concretos para subpagos y base para políticas de condonación | Próxima sesión |
| 7 | El grupo | Generar mini diagrama de procesos fiscales complejos (notas de crédito, facturas anticipo) | Próxima sesión |
| 8 | Roberto Baez Muñoz | Proveer escenarios de validación al equipo | Próxima sesión |
| 9 | El grupo | Mapear todos los escenarios posibles de pago y compartir políticas de condonación | Próxima sesión |
| 10 | Larissa Calvo | Verificar flujo de cancelación e informar a Valdemar si regresa a tramitación o notifica directamente | Próxima sesión |
| 11 | El grupo | Acotar catálogo de motivos de cancelación y definir comportamiento esperado del sistema | Próxima sesión |
| 12 | Sara Sánchez | Compartir nota de crédito de muestra en PDF y XML | Próxima sesión |
| 13 | El grupo | Ratificar si los métodos de pago PPD y PUE deben estar disponibles para Proformas | Próxima sesión |
| 14 | Sara Sánchez | Recopilar y compartir capturas de pantalla de reportes legados (Pedidos y Contador) | Próxima sesión |
| 15 | Roberto Baez Muñoz | Compartir flujo de pantallas presentado durante la sesión por correo | Inmediato |
| 16 | Larissa Calvo | Coordinar y agendar próxima sesión de seguimiento (lunes preferentemente) | Inmediato |

---

## Próxima Sesión

- **Temas a revisar:** Reportes (pedidos, facturas y cobros), pendientes de políticas de pago y cancelación.
- **Estimado:** Reunión preferentemente el día lunes.
