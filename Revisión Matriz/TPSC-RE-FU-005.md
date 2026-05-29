# TPSC-RE-FU-005

**Estatus:** ✅ Atendido

---

## Observaciones

- **Regla 6:** ¿El porcentaje es a nivel cliente o por producto?
- **Regla 3 / Riesgo 1:** Deben ser campos independientes. Por el tipo de dato, seguramente se creará un catálogo nuevo de *tipo de comprobante* en el cual solo Perú tendrá opciones.
- Si se hará facturación para Perú se debe contemplar la detracción especificada en el Riesgo 7, de lo contrario no funcionará el timbrado.
- Revisar si Criterio 2 y 3 son el mismo catálogo — si son catálogos diferentes y se colocan en el mismo, puede haber complicaciones al momento de la facturación.
- **Criterio 4:** son catálogos separados, por tanto campos separados; no deben estar juntos.
- **Criterio 5.1:** revisar, ya que posiblemente lo que se muestra como *Forma de pago* para Perú en realidad es el *Método de pago*.

## Notas adicionales

- **Criterio 2** (→ Reglas 2 y 6, Criterios B1-B3 y C1-C2): considerar mapeo hacia Legacy de estas opciones en catálogo y en facturas o pedidos si aplica.
- **Criterio 6** (→ Regla 8 / Criterio D2): el valor `%` debe ser configurable a nivel BD.

## Resumen de cambios aplicados

- El campo de dimensión temporal del pago para Perú se renombró de "Forma de Pago" a **"Condición de Pago"** (Contado/Crédito), aclarando que es el equivalente conceptual del Método de Pago mexicano (PUE/PPD) y que para Perú no se captura el medio de pago específico porque la normativa SUNAT no lo exige.
- **Uso de CFDI (México)** y **Tipo de Comprobante (Perú)** se modelan como campos independientes con catálogos separados (conceptos fiscales distintos).
- Los catálogos de México (Forma de Pago y Uso de CFDI) se documentaron con las opciones reales de PQF2. Se identificó que el catálogo de Forma de Pago no usa las claves del catálogo SAT — queda pendiente evaluar el mapeo requerido para el timbrado del CFDI.
- Las banderas **Agente de Retención IGV** y **Sujeto a Detracción** quedaron sujetas a confirmación de aplicabilidad con el cliente para evitar desarrollo innecesario.
- **Porcentaje de detracción:** según normativa SUNAT, la detracción aplica por bien o servicio (R.S. 183-2004/SUNAT), no por cliente. La bandera a nivel cliente es un indicador; la tasa real se determina por producto. Queda pendiente confirmar con el cliente el nivel de captura (cliente, producto o catálogo). Referencia: Regla 8, Criterio D2, Riesgo 5.
- La Percepción del IGV se documentó como condición del emisor (no del cliente) e incorporada como **Brecha 4**.
- Se eliminaron las reglas de edición por operación en otros módulos (tachadas en el original), ya que pertenecen a los módulos Validar Cobro, Factura por Adelantado y Tramitación.
- Reglas reescritas como enunciados declarativos: de 11 a 9.
- Criterios organizados en 5 secciones: **A** (Visualización), **B** (Campos México), **C** (Campos Perú), **D** (Banderas tributarias), **E** (Obligatoriedad y persistencia).
- Riesgos renumerados consecutivamente (antes saltaba el 2; ahora 1–5).
