# R16 – Tramitar Pedido Sin Crédito: Análisis Consolidado de Sesiones

> **Sesiones analizadas:** 01 (2026-03-20) · 03 (2026-04-06) · 04 (2026-04-08) · 05 (2026-04-09) · 06 (2026-04-14) · 07 (2026-04-15)
> **Participantes clave:** Rose Ríos Gómez, Valdemar Farina Sanchez, Roberto Baez Muñoz, Sara Sánchez, Larissa Calvo, Biridiana Arias, Cornelio M. Ramirez B., Mayra Franco, Irma Andrade Aguado

---

## 1. Alcance General del Release

- El release fue renombrado de **"R16 Adquisiciones"** a **"Tramitar Pedido Sin Crédito"**, nombre que ya se usaba internamente.
- **Perú está dentro del alcance**, lo que implica regionalización de catálogos fiscales (tipo de sociedad mercantil, régimen fiscal).
- Se debe investigar si el proveedor de timbrado actual (**TurboPac**) puede operar en Perú; de lo contrario, se evalúa con Finanzas.
- El alcance inicial de notas de crédito es **solo para clientes de prepago** y solo para documentos generados en ProquifaNet 2 (excluye legacy).

---

## 2. Módulos / Flujos Identificados

### 2.1 Factura por Adelantado (Clientes de Crédito)

**¿Qué es?**
Permite a un cliente de crédito solicitar una factura antes de recibir su mercancía, para justificar fiscalmente el gasto de presupuesto. Los días de crédito comienzan a correr al generar la factura.

**Reglas de negocio:**
- El botón/opción se ubica en el módulo de **Tramitar Pedido** (no en Pretramitar).
- **No aplica** para pedidos con **productos controlados** ? el campo se bloquea y muestra ícono informativo.
- Es **excluyente** con "Entrega con Remisión" ? se implementa con **radio button** (elige una, la otra, o ninguna).
- Requiere **código de verificación** al tramitar (duración: 24 horas).
  - Se deben presentar **dos estimaciones**: con y sin persistencia del código (poder salir y regresar a la pantalla dentro de las 24 hrs).
- Al activar "Factura por Adelantado", el método de pago cambia automáticamente de **PUE ? PPD**.
- Requiere **autorización** (mismo rol que "Entrega con Remisión").
- Solo el campo **"Uso de CFDI"** es editable en la pantalla; el resto es de solo lectura.
- La **forma de pago** (PPD ? "por definir") se muestra bloqueada, siempre visible.
- El **tipo de cambio** queda bloqueado; se usa el valor que arroja el sistema por defecto.
- Clientes de crédito sin factura adelantada ? flujo normal sin cambios.

**Vista del módulo "Factura por Adelantado" (pantalla nueva):**
- **Vista resumen**: listado agrupado por cliente con RFC, número de facturas pendientes y monto total en **dólares** (para comparabilidad).
  - Buscador por cliente, RFC o pedido interno (búsqueda por coincidencia, no exacta).
  - Listado ordenado por antigüedad de la solicitud.
  - Totalizadores: total de clientes, total de facturas pendientes, monto total (considerando operatividad vs. analítica).
- **Vista detalle por cliente**: datos fiscales del cliente + lista de pedidos pendientes.
  - Columnas: pedido interno, fecha de generación del pedido, condiciones de pago, quién factura, subtotal, IVA, monto total, contacto del comprador (quién generó la OC) + correo electrónico por línea.
  - La razón social de la empresa que factura debe aparecer en el modal de generación de factura (para evitar refacturaciones).
- **Modal de generación de factura**: pedido interno, monto, condiciones de pago, RFC, razón social, correo, detalles de facturación.

---

### 2.2 Entrega con Remisión

**Reglas de negocio:**
- Selección **manual** por el usuario para cada pedido (no se hereda automáticamente del catálogo del cliente).
- Afecta la **fecha estimada de entrega**: si está marcada, permite arrojar fechas en los últimos días del mes; sin ella, esos días se bloquean.
- Es **excluyente** con "Factura por Adelantado".
- **No aplica** para pedidos con productos controlados ? se bloquea.
- Requiere **autorización** (mismo rol que Factura por Adelantado).
- Campo opcional; no marcarla no impide continuar el flujo.

---

### 2.3 Flujo de Prepago (Cobranza y Validación de Pago)

**Cambio de responsabilidad:**
- En el nuevo flujo, el área de **Tesorería / Cobros** es responsable de todo el proceso (recepción, validación y relación del pago con proforma/factura). Antes lo hacían los ESAC.

**Flujo general:**
1. ESAC tramita pedido de cliente prepago ? se genera la **proforma** ? aparece **pendiente de pago** para Tesorería.
2. Tesorería recibe cobro desde el **buzón de pagos** (similar al buzón de cotizaciones del MailBot).
3. **Paso 1 – Captura del cobro:**
   - Se valida el comprobante de pago (monto en correo vs. adjunto).
   - Se asigna un **folio interno de cobro** (formato propuesto: `COB` + secuencial).
   - Si hay inconsistencia ? se marca y se detiene el flujo.
   - Sin política de condonación de montos faltantes: cualquier diferencia rechaza el pago.
4. **Paso 2 – Asociación de facturas/proformas:**
   - Se asocian facturas o proformas al cobro capturado.
   - Se muestra diferencia entre monto capturado y total a cobrar.
   - Escenarios: saldo a favor (remanente), pago de menos.
5. El pedido prepago se **desbloquea en Tramitar Pedido** una vez validado el pago.
6. La factura se envía de forma independiente; luego se envía la confirmación del pedido al tramitar.

**Escenarios de saldo a favor:**
- El cliente puede aplicar el saldo a futuras compras (factura de anticipo).
- O solicitar devolución.
- Se propone notificar al cliente mediante carta/correo automático sobre el saldo disponible.
- Se evaluó (pendiente) generar un **estado de cuenta auxiliar** en ProquifaNet 2.

**Seguimiento proactivo de cobranza en prepago:**
- Para el ~50% de clientes que no pagan de inmediato: se propone registrar una **Fecha Estimada de Pago (FEP)** en la pantalla de validación actual (sin pantalla adicional).
- La columna "Días Restantes de Crédito" se renombra a **"Días Restantes para Pago"** en prepago, calculado desde la FEP capturada.
- Si el cliente paga de inmediato y no se requiere seguimiento, la columna permanece vacía.
- Tesorería tiene **72 horas** para validar el pago una vez recibido.

**Estados de factura:**
- **Abierto**: no se ha cobrado.
- **Cerrado**: pago ejecutado y complemento realizado.

---

### 2.4 Notas de Crédito

**Alcance:**
- Solo para clientes de **prepago** en este lanzamiento.
- Solo documentos generados en ProquifaNet 2 (no legacy).
- Obligación de conservar el XML mínimo **5 años**.

**Flujo de generación (4 pasos):**
1. **Selección de factura** a relacionar con la nota de crédito.
   - Tabla: folio, UUID SAT, RFC, cliente, razón social, fecha, total (con moneda MXN/USD), estado SAT.
   - Filtro por: cliente, RFC, fecha, moneda.
   - Se muestran **todas las facturas de clientes prepago** de los últimos 5 años.
   - Facturas canceladas ante SAT **no** pueden generar nota de crédito.
   - Se retira columna "Saldo Disponible" (genera confusión).
2. **Captura de datos** de la nota de crédito.
3. **Generación y timbrado** (vía TurboPac).
4. **Resumen final**: folio, UUID SAT, pack (TurboPac), certificación SAT, RFC, datos específicos, saldos y totales actualizados.
   - Acciones disponibles: descargar XML, descargar PDF, reenviar por correo, cancelar nota de crédito (opcional según alcance).

**Reglas de cancelación de nota de crédito:**
- Mismas reglas SAT que un CFDI: **24 horas sin autorización**; después requiere autorización del cliente.
- Si la factura original **no** está cancelada ante SAT ? cancelar la nota inhabilita la cancelación previa; se puede emitir nueva nota.
- Si la factura original **ya** está cancelada ante SAT ? no se puede generar nueva nota; se emite carta al cliente para usar saldo.
- En ambos casos, la acción en sistema es solo cancelar la nota, sin acción adicional sobre la factura original.

---

### 2.5 Trazabilidad del Pedido (Vista para el cliente / área interna)

**Pedidos que NO aceptan parciales:**
- Trazabilidad global del pedido.
- Si una línea entra a back order y el cliente solicita apertura de parciales ? se detona embalaje y entrega de lo disponible aunque la factura ya esté emitida.

**Pedidos que SÍ aceptan parciales:**
- Trazabilidad por **partida individual** con resumen de entrega por cada una.

**Información por estatus:**
| Estatus | Datos a mostrar |
|---|---|
| **Recepción** | Fecha de recepción |
| **Tramitación** | Usuario del sistema (ej. nombre del ESAC) |
| **Compra** | Proveedor, comprador, guía de embarque, factura del proveedor, PDF de OC |
| **Importación** | Fecha estimada de arribo, número de pedimento |
| **Inspección** | Fecha inicio/fin, inspector, resultado, observaciones, certificado, hoja de seguridad |
| **Facturación** | Número de factura, UUID, método de pago, factura y complemento de pago |
| **Envío** | Fecha de entrega/envío, número de guía, dirección, acuse, documento de conformidad |

**Sección Facturación y Cobranza:**
- Se muestran: Proforma, Factura (con estado SAT) y para pedidos a parciales, el desglose de partidas por factura.
- Si se usó remisión para entrega, se incluye en la visualización.

---

## 3. Decisiones de Diseño Relevantes

| Decisión | Resolución |
|---|---|
| Nombre del módulo de cobros | Pendiente definición interna (opciones: "Validar Cobro", "Gestionar Cobranza") |
| Ubicación de "Factura por Adelantado" | **Tramitar Pedido** (no Pretramitar) |
| Control de selección FA / Remisión | **Radio button** (excluyentes entre sí, ambos opcionales) |
| Campos editables en modal FA | Solo **Uso de CFDI**; resto de solo lectura |
| Moneda en vista resumen FA | **Dólares** para comparabilidad; detalle muestra moneda de la transacción |
| Tipo de cambio al validar pago | Pendiente confirmar: ¿el de la Proforma o el del día del pago? |
| Estado de cuenta auxiliar | Evaluación pendiente para este release |
| Folio de cobro interno | Formato `COB` + secuencial |
| Edición de datos del cliente en tramitar | Solo a nivel catálogo; en pedido solo Uso de CFDI |
| Serie de folios para FA en PQF2 | Se evalúa usar **nuevo número de serie** para evitar colisión con legacy |

---

## 4. Pendientes / Puntos Abiertos

- [ ] Definir nombre oficial del módulo de cobranza.
- [ ] Confirmar si TurboPac opera en Perú.
- [ ] Confirmar regla de tipo de cambio al validar pago (Proforma vs. día del pago).
- [ ] Definir catálogos fiscales para Perú (tipo de sociedad mercantil, régimen fiscal, usos de CFDI).
- [ ] Definir restricciones de incompatibilidad entre métodos de pago y forma de pago (SAT).
- [ ] Revisar políticas de Tesorería sobre condonación de montos faltantes.
- [ ] Definir catálogo de motivos de cancelación de pedido.
- [ ] Definir punto de inyección en sistema legacy para factura por adelantado.
- [ ] Entregar muestras de correos de cobro para entrenamiento del clasificador automático (MailBot).
- [ ] Evaluar generación de estado de cuenta auxiliar en ProquifaNet 2.
- [ ] Revisar reglas del foliador de proformas (si es lineal o por empresa).
- [ ] Confirmar si PPD y PUE deben estar disponibles para Proformas.
- [ ] Mapear todos los escenarios posibles de pago (faltante, sobrante, saldo a favor).
- [ ] Analizar impacto de órdenes de compra internas / sin orden en flujos de FA y Entrega con Remisión.
- [ ] Revisar si se mantiene opción "Editar datos" (Uso de CFDI / Método de pago) en tramitar pedido para crédito.
- [ ] Presentar dos estimaciones de código de verificación (con y sin persistencia de 24 hrs).

---

## 5. Impactos en Módulos Existentes de ProquifaNet 2

| Módulo afectado | Impacto |
|---|---|
| **L05 – Tramitar Pedido** | Nuevas opciones: Factura por Adelantado y Entrega con Remisión (radio buttons). Código de verificación. Cambio automático PUE ? PPD. |
| **L04 – Pretramitar Pedido** | Sin cambios (se descartó ubicar FA aquí). |
| **L06 – Orden de Compra** | Impacto en flujos de OC interna / sin OC con FA y Remisión (pendiente análisis). |
| **MailBot (L11)** | Extensión para buzón de pagos: clasificación automática de correos de cobro. |
| **CFDIs** | Generación de factura por adelantado, proformas, notas de crédito y complementos de pago. |
| **Catálogos** | Nuevos catálogos para Perú (fiscal). Catálogo de motivos de cancelación. |
| **Trazabilidad / Reportes** | Nueva vista de seguimiento de pedido por etapas y por partida. |
