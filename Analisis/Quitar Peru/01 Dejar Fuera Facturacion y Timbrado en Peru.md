# R16 · Perú sin timbrado — Análisis de impacto

## 1. ¿Vale la pena personalizar los formatos de impresión de Perú?

**No, para Factura y Nota de Crédito. Sí se conserva la Proforma.**

Sin timbrado en el sistema no se puede generar la representación impresa de un comprobante peruano. Los elementos que la normativa exige en ese documento, código QR de validación, hash y firma digital, etc, **no los calcula el sistema: los devuelve el proceso de timbrado**. Si el timbrado ocurre por fuera, el sistema no los tiene.

Consecuencias:

- Un PDF generado por el sistema sería **informativo, sin validez fiscal**.
- **Duplicaría** la representación impresa oficial que emitirá el sistema externo donde se timbre.
- **Riesgo de cumplimiento:** un documento con apariencia de factura peruana pero sin validez se puede enviar al cliente por error.
- Exigiría cerrar antes todas las definiciones fiscales peruanas abiertas (texto legal, series, régimen tributario, catálogos, certificaciones, etc), que requieren asesor fiscal peruano y/o respuesta de cumplimiento.
- El volumen (un pedido, ~900 USD) no sostiene el esfuerzo de diseñarlos y mantenerlos.

**Excepción — la Proforma se conserva:** no es comprobante fiscal, no requiere timbrado y es el documento con el que se inicia el cobro. Su diseño ya está trabajado.

---

## 2. ¿Hay esfuerzo adicional para que Perú tenga todo el proceso de Tramitación Prepago?

**Prácticamente nulo: ya está cubierto por la estimación de R16.**

Los requisitos de tramitación prepago **ya son bi-región por diseño**: aplican a México y Perú, y sus reglas de folio, proforma y envío no distinguen región.

|Concepto|Estado|Esfuerzo adicional|
|---|---|---|
|Tramitación prepago sin Factura por Adelantado|Ya bi-región|Ninguno|
|Tramitación prepago con Factura por Adelantado|Se excluye de Perú|Ninguno (sale)|
|Prepago con controlados|No aplica a Perú|Ninguno|
|PDF de Proforma Perú|Requisito propio ya especificado|Ya estimado|
|Datos fiscales del cliente Perú|Requisitos propios ya especificados|Ya estimado|
|Localización de catálogos y etiquetas (Cat de Clientes, Validar Cobro)|Ya cubierto|Ya estimado|

  

**Esfuerzo nuevo que sí introduce el rediseño (no estimado):**

1. Bloquear/no mostrar la opción **Factura por Adelantado** en Tramitar Pedido para Región Perú (cambio menor, condicionado por región).
2. **Redefinir el Paso 3 de Validar Cobro para Perú**: de "Facturación y Envío" a solo envío de Confirmación de Pedido. Es una **simplificación** (retirar lógica), no una adición. (cambio menor).
3. **[Propuesta]** Ajustar la Asociación del Paso 2 para que en Perú opere siempre contra Proformas. (Sólo se documenta, cambio menor).
4. **Retirar la aplicación de Notas de Crédito del Paso 2 de Validar Cobro para Perú**, con el ajuste del cálculo del saldo que ello implica. También es una **simplificación**.(cambio menor).

---

## 3. Impacto en esfuerzo

**Sale del alcance:**

- Factura por Adelantado de Perú (módulo completo y su listado)
- PDF de Factura de Perú
- Módulo de Notas de Crédito de Perú y su PDF
- Aplicación de Notas de Crédito en el Paso 2 de Validar Cobro de Perú
- Tramitación prepago con Factura por Adelantado en Perú
- Timbrado, generación y envío de documentos en el Paso 3 de Validar Cobro
- Complemento de Pago de Perú (ya estaba fuera)

**Entra:** solo el bloqueo de Factura por Adelantado por región y la simplificación del Paso 3. **Lo que entra es menor que lo que sale.** 

**Adicional:** Cambios a matriz de requisitos de validarse los cambios en el flujo de validar cobro y tramitar pedido

---

## 4. Cambios al flujo de Perú

Perú queda con **una sola ruta**:

```
Tramitar Pedido (sin opción de Factura por Adelantado)
   → genera Proforma (PDF, sin timbrado) y la envía al cliente
   → genera pendiente en Validar Cobro
Validar Cobro P1 — Captura del Cobro
Validar Cobro P2 — Asociación (cobro ↔ Proforma, sin Notas de Crédito) [Propuesta]
Validar Cobro P3 — Envía solo la Confirmación de Pedido (sin generar ni timbrar documentos)
   → termina el pendiente
```

**Cambios concretos:**

- **Factura por Adelantado se excluye de Perú:** la opción se bloquea/no se muestra en Tramitar Pedido para Región Perú. Todo pedido prepago peruano **llega siempre a Validar Cobro**; nunca pasa por Factura por Adelantado.
- **Los escenarios de tramitación de Perú se reducen a uno:** prepago sin controlados y sin Factura por Adelantado.
- **Paso 3 simplificado:** ya no genera ni timbra documentos. Solo envía la **Confirmación de Pedido sola** y cierra el pendiente.
- **[Propuesta]** La Asociación del Paso 2 opera siempre contra **Proformas** (la Proforma no se convierte en ningún otro documento).
- **Se elimina la aplicación de Notas de Crédito del Paso 2 en Perú:** al salir las NC del alcance, no hay NCs que aplicar. El cálculo del saldo se simplifica a _adeudo de las Proformas − cobros aplicados_ (sin el componente de NCs), y desaparecen las reglas asociadas (NC por documento, conversión de NC en moneda distinta, aplicar/remover NC).
- **[Propuesta]** La **configuración fiscal de Perú se conserva capturable** en el sistema (datos fiscales del cliente, tipo de operación, régimen) aunque no se timbre, para no perder el trabajo de análisis y quedar listos si el timbrado se habilita.

---

## 5. Riesgos

**1. Doble captura y descuadre.** La operación se captura dos veces (en el sistema y en el de timbrado). Riesgo de inconsistencia en datos, folios y montos.

**2. Estado de Cuenta y reportes incompletos.** Los reportes de Perú no reflejan los documentos fiscales reales, de ser necesarios.

---

## 6. Acciones y confirmaciones requeridas

1. **Descartar del alcance:** Factura por Adelantado de Perú, PDF de Factura de Perú, módulo y PDF de Notas de Crédito de Perú, aplicación de Notas de Crédito en el Paso 2, tramitación con Factura por Adelantado en Perú, y las secciones de timbrado del Paso 3.
2. **Confirmar propuestas:** generar proforma, asociación de cobros siempre contra Proformas, y conservación de la configuración fiscal capturable en catálogo de clientes (Cobros, Entrega y Facturación) y Validar Cobro