# 07 — R16 Tramitar Pedido sin Crédito: 7ma Sesión de Entendimiento

| Campo | Detalle |
|---|---|
| **Fecha** | 15 de abril de 2026 |
| **Hora** | 14:55 CST |
| **Sesión** | 7ma Sesión de Entendimiento |
| **Tema principal** | Reportes de Facturas, Cobros y Pedidos — Cancelación prepago — Trazabilidad por partida |
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
| Alejandro Cervantes Flores | Desarrollo / Ryndem |
| Jose Antonio Chavez Amador | Coordinador QA / Ryndem |
| Valdemar Farina Sanchez | Coordinador de Desarrollo / Ryndem |
| Larissa Calvo | Negocio / Proquifa |
| Sara Sánchez | Negocio / Proquifa |
| Biridiana Arias | Finanzas / Proquifa |
| Mayra Franco | Finanzas / Proquifa |
| Cornelio M. Ramírez B. (Cornel) | Finanzas / Proquifa |

---

## Resumen Ejecutivo

La sesión cubrió los tres **reportes operativos** del sistema (Facturas, Cobros y Pedidos), los **protocolos de cancelación por falta de pago** para pedidos prepago, y la **trazabilidad detallada por partida** incluyendo estados de Recepción, Tramitación, Compra, Importación, Inspección, Facturación y Envío. Se acordó una última sesión de validación final.

---

## Temas Tratados

### 1. Reporte de Facturas — Diseño y columnas

- **Vista principal:** folio, cliente, estado (por cobrar / cobrada), fecha de facturación, tipo de factura, folio pedido interno, condiciones de pago, emisor, monto.
- Se eliminan columnas innecesarias para mantener la vista limpia. El UUID se mantiene por utilidad en validación de duplicados.
- Se elimina el campo **“Monto Estimado de Cobro” (MEC)**; se mantiene el **Monto de la Factura**.
- **Estados:** “abierto” (no cobrada) y “cerrado” (pago ejecutado y complemento generado).
- **Fecha de pago real** (fecha en que el dinero ingresó al banco) se incluye en el detalle, además de la fecha de timbrado del complemento.
- El detalle de la factura incluirá un **link al pedido interno** y al cobro asociado.

### 2. Seguimiento de cobranza prepago — Semáforo y Fecha Estimada de Pago

- Para clientes prepago que no pagan de inmediato, se registra una **Fecha Estimada de Pago (FEP)** en lugar de crear una pantalla adicional de cobranza.
- La columna **“Días Restantes de Crédito (DRC)”** se reemplaza por **“Días Restantes para Pago”** en prepago, calculada desde la FEP. Si el cliente paga de inmediato, la columna va vacía.
- La FEP la captura el **analista de CxC** cuando contacta al cliente o cuando se aprueba la factura.
- **Semáforo visual** (verde / amarillo / rojo / morado) según días restantes. Larissa Calvo proporcionará las reglas exactas por escrito.
- Se conserva el diseño visual porque también aplica al flujo de crédito.

### 3. Reporte de Cobros — Listado y detalle

- **Filtros:** cliente, cuenta de destino, factura.
- **Columnas vista principal:** folio de cobro, cliente, fecha de cobro, tipo de cambio, medio de pago, cuentas origen/destino, monto de cobro.
- El **medio de pago** debe alinearse con el **catálogo del SAT**. Cornel compartirá el listado.
- **Vista detalle:** resumen de la transacción + listado de facturas a las que se aplicó (folio, pedido interno, cliente, fechas, montos, proforma y complemento de pago).

### 4. Reporte de Pedidos — Estados y alcance

- Un “pedido” en este reporte es aquel con **folio asignado** (crédito o prepago).
- **Estados:** “abierto” (pendiente de entrega), “cerrado” (entregado), “cancelado” (folio que nunca continuó).
- Pedidos **intramitables** no aparecen en el listado.
- **Limitación MVP:** El cierre de pedidos sigue en Legacy. Se propone **conexión bidireccional** con Legacy para actualizar reportes.
- La función de **ordenar por columna** está contemplada (no agrupar, sino ordenar).

### 5. Cancelación de pedidos prepago por falta de pago

- La cancelación será ejecutada **manualmente por SAC (Servicio al Cliente)**.
- Al cancelar, el sistema **anula automáticamente las proformas y facturas asociadas** dentro del mes vigente.

### 6. Trazabilidad detallada por partida

- Estructura de trazabilidad **por partida individual** para pedidos con y sin parciales.
- Ajustes por estado:
  - **Recepción:** solo “fecha de recepción”.
  - **Tramitación:** usuario del sistema.
  - **Compra:** proveedor, comprador, guía de embarque, factura del proveedor, PDF de la OC.
  - **Importación:** fecha estimada de arribo, número de pedimento.
  - **Inspección Matriz:** fecha inicio/fin, inspector, resultado, observaciones, certificado y hoja de seguridad.
  - **Facturación:** número de factura, UID, método de pago, documentos de facturación. Carga al portal y comprobante van en el reporte de facturas, solo un enlace en el detalle.
  - **Envío:** fecha de entrega/envío, número de guía, dirección, acuse y documento de conformidad.
- Para pedidos que **aceptan parciales**: trazabilidad por partida individual con resumen de entrega por cada una.

### 7. Facturación y Cobranza — Visualización de documentos

- Se mostrarán: **Proforma**, **Factura** (con estado SAT) y, para pedidos a parciales, el desglose de partidas por factura.
- Para prepagos se maneja similar a la Proforma, incluyendo la remisión si se utilizó.

---

## Pendientes y Compromisos

| # | Responsable | Compromiso | Fecha límite |
|---|---|---|---|
| 1 | Valdemar Farina Sanchez | Analizar eliminación de columnas innecesarias como WIP y MEC en el reporte de facturas | Próxima sesión |
| 2 | Valdemar Farina Sanchez | Proponer mejor método para registrar la Fecha Estimada de Pago (FEP) para múltiples proformas/facturas | Próxima sesión |
| 3 | Larissa Calvo | Proporcionar por escrito las reglas del semáforo (días exactos para verde, amarillo, rojo, morado) | Próxima sesión |
| 4 | El grupo | Elaborar propuesta para que SAC cancele pedido por falta de pago con cancelación automática de proforma/factura | Próxima sesión |
| 5 | Sara Sánchez | Compartir ejemplo del reporte actual que se descarga para referencia del equipo de desarrollo | Próxima sesión |
| 6 | Cornelio M. Ramírez B. | Enviar listado de medios de pago del catálogo SAT al equipo | Próxima sesión |
| 7 | El grupo | Incluir fecha real de pago recibido en el reporte detallado de facturas | Próxima sesión |
| 8 | Larissa Calvo | Enviar correo a Valdemar para estimar la solicitud de cambio sobre la regla de aceptación de parciales | Inmediato |
| 9 | Biridiana Arias | Compartir video para detallar información visible por línea de pedido (datos por partida individual) | Próxima sesión |
| 10 | Biridiana Arias | Confirmar nombre exacto del campo “fecha de inicio” en importación con el equipo de compras | Próxima sesión |
| 11 | Sara Sánchez | Compartir reglas necesarias (datos y documentos) desde sistemas legados con ubicación para conexión de extracción | Próxima sesión |
| 12 | El grupo | Revisar internamente los ajustes y definir fecha para la última sesión de validación | Inmediato |
| 13 | Larissa Calvo / Sara Sánchez | Resolver dudas y pendientes listados en el archivo compartido antes de la última sesión | Inmediato |

---

## Próxima Sesión

- **Objetivo:** Última sesión de **validación final** de ajustes presentados.
- El equipo de Larissa Calvo y Sara Sánchez debe resolver todos los pendientes en paralelo antes de esa sesión.
