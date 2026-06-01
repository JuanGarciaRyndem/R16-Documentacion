# Notificación Regulatoria al Cliente en Cotización Definitiva

| Campo | Valor |
|---|---|
| **ID** | TPSC-RE-FU-007 |
| **Nombre** | Notificación regulatoria al cliente |
| **Módulo** | Cotizar lo Cotizable |
| **Categoría** | Funcional |
| **Estatus** | Propuesto |
| **Referencia Legacy** | Sin trazabilidad directa a las matrices originales del cliente; emergente del archivo PMO 5-may fila #68 |

---

## Historia de Usuario

> Yo como ESAC, quiero que el sistema agregue automáticamente una leyenda regulatoria visible en el PDF de la cotización definitiva entregada al cliente cuando la cotización contiene al menos una partida de producto clasificado como Sustancia Controlada (Mundial, Nacional u Origen), para que el cliente conozca desde el primer artefacto comercial qué documentos regulatorios debe tener para que su pedido pueda procesarse, evitando re-trabajos y fricciones tardías en el flujo de pretramitación.

---

## Requisito

El sistema debe agregar automáticamente una leyenda regulatoria al PDF de la cotización definitiva generada por el módulo **Cotizar lo Cotizable** cuando la cotización contenga al menos una partida de producto clasificado como **Sustancia Controlada tipo Mundial, Nacional u Origen**. La leyenda aplica únicamente a las cotizaciones definitivas e informa al cliente que el pedido requiere, para su procesamiento, la entrega de los documentos regulatorios correspondientes (Licencia Sanitaria y Aviso de Responsable Sanitario para Región México; denominación equivalente según normativa DIGEMID para Región Perú). La leyenda no bloquea la generación del PDF: la cotización definitiva se entrega siempre.

---

## Alcance

### Aplica a

- Cotizaciones definitivas generadas en el módulo Cotizar lo Cotizable que contengan al menos una partida de producto clasificado como Sustancia Controlada tipo Mundial, Nacional u Origen.
- Adición de una leyenda regulatoria visible en el PDF de la cotización definitiva entregada al cliente.
- Aplicación a clientes de México y Perú con texto adaptado según la denominación regulatoria de cada región.
- Inclusión de la leyenda como información general del documento, sin desglose por partida.
- Comportamiento no bloqueante: el PDF de la cotización definitiva se genera y entrega al cliente independientemente de si ya tiene o no documentos regulatorios cargados en el Catálogo de Clientes.

### No aplica a

- **Cotizaciones de investigación:** no incluyen la leyenda regulatoria. Las cotizaciones de investigación tienen su propia leyenda de trabajo en curso, que es funcionalidad preexistente del módulo y no es alcance de este requisito.
- Cotizaciones definitivas que **no contienen** ninguna partida de producto clasificado como Sustancia Controlada (esas cotizaciones se generan sin la leyenda regulatoria).
- Validación de presencia de documentos regulatorios en el Catálogo de Clientes (esa validación bloqueante vive en el módulo Pretramitar Pedido, requisito TPSC-RE-FU-009).
- Otros cambios al diseño general del PDF de cotización (mantiene su layout, colores y estructura actuales; solo se agrega la leyenda regulatoria).

---

## Reglas de Negocio

**Regla 1 — Leyenda regulatoria solo en cotizaciones definitivas**
La leyenda regulatoria aplica exclusivamente a las cotizaciones definitivas del módulo Cotizar lo Cotizable. Las cotizaciones de investigación no incluyen la leyenda regulatoria.

**Regla 2 — Detonante de la leyenda regulatoria**
La leyenda regulatoria se agrega a la cotización definitiva cuando al menos una de sus partidas corresponde a un producto clasificado como Sustancia Controlada tipo Mundial, Nacional u Origen. Si ninguna partida es controlada, el PDF se genera sin la leyenda.

**Regla 3 — Leyenda general a nivel documento**
La leyenda regulatoria se incorpora una sola vez como nota general del documento, sin desglose por partida. El número de partidas controladas no afecta cuántas veces aparece: una sola aparición por PDF.

**Regla 4 — Contenido de la leyenda según Región del cliente**
El texto de la leyenda regulatoria corresponde a la Región del cliente:
- **Región México:** referencia Licencia Sanitaria y Aviso de Responsable Sanitario (denominaciones COFEPRIS).
- **Región Perú:** referencia los documentos regulatorios equivalentes según normativa DIGEMID.

> **⚠️ Pendiente** — Denominación exacta para Perú pendiente de confirmar con el cliente.

**Regla 5 — Leyenda no bloqueante**
La leyenda regulatoria es informativa, no validativa. El sistema genera y entrega la cotización definitiva con la leyenda incorporada, sin bloquear la generación del PDF.

**Regla 6 — Sin consulta al Catálogo de Clientes**
La leyenda se agrega siempre que haya al menos una partida controlada en la cotización definitiva, sin importar si el cliente ya tiene documentos regulatorios cargados en el Catálogo. La leyenda funciona como recordatorio universal del requisito regulatorio para la operación con sustancias controladas.

---

## Riesgos

**Riesgo 1 — Cliente confundido al recibir leyenda redundante**
La leyenda aparece siempre que haya productos controlados, incluso si el cliente ya tiene cargados todos sus documentos regulatorios en el Catálogo. Esto puede generar fricción operativa: clientes recurrentes que reciben siempre la misma notificación y podrían interpretarla como un recordatorio innecesario.

> **⚠️ Propuesta abierta** — Para evaluación con el cliente: variante dinámica de la leyenda que consulte el Catálogo de Clientes y solo la agregue cuando el cliente **no** tenga cargados los documentos regulatorios. Esto evitaría ruido para clientes recurrentes que ya cumplieron con su documentación. La propuesta confirmada actual es la leyenda universal.

**Riesgo 2 — Denominación regulatoria Perú no definida**
Para clientes con Región Perú, la denominación exacta de los documentos regulatorios equivalentes (que regula DIGEMID en lugar de COFEPRIS) no está confirmada por el cliente. Si el desarrollo arranca sin esta definición, el texto del PDF para clientes Perú podría usar denominaciones incorrectas.

> **⚠️ Pendiente** — Clarificar con el cliente la nomenclatura exacta para Perú antes del desarrollo.

---

## Criterios de Aceptación

### SECCIÓN A — Aplicabilidad a Cotizaciones Definitivas

**Criterio A1 — Leyenda regulatoria solo en cotizaciones definitivas**
- **Dado** que un usuario genera una cotización en el módulo Cotizar lo Cotizable,
- **Cuando** la cotización es de investigación,
- **Entonces** el sistema **NO** deberá agregar la leyenda regulatoria. La cotización de investigación conserva su comportamiento preexistente.

**Criterio A2 — Detección de cotización definitiva con productos controlados**
- **Dado** que un usuario genera una cotización definitiva en el módulo Cotizar lo Cotizable,
- **Cuando** el sistema procesa el PDF,
- **Entonces** deberá inspeccionar las partidas de la cotización y determinar si al menos una corresponde a un producto clasificado como Sustancia Controlada tipo Mundial, Nacional u Origen.

---

### SECCIÓN B — Inclusión de la Leyenda

**Criterio B1 — Inclusión de la leyenda regulatoria en cotización definitiva con controlados**
- **Dado** que una cotización definitiva contiene al menos una partida controlada,
- **Cuando** el sistema genera el PDF,
- **Entonces** deberá incluir la leyenda regulatoria visible en una posición claramente identificable del documento. La leyenda informa al cliente que el pedido requiere la entrega de los documentos regulatorios para procesarse.

> **⚠️ Pendiente** — La ubicación exacta de la leyenda en el PDF (encabezado, sección dedicada antes de la firma, pie de página, etc.) queda como decisión de diseño UI a definir en sprint de desarrollo.

**Criterio B2 — No inclusión de la leyenda en cotización definitiva sin controlados**
- **Dado** que una cotización definitiva **no** contiene ninguna partida controlada,
- **Cuando** el sistema genera el PDF,
- **Entonces** **NO** deberá incluir la leyenda regulatoria. El PDF se genera con el layout estándar del módulo sin la adición.

**Criterio B3 — Una sola aparición por documento**
- **Dado** que una cotización definitiva contiene múltiples partidas controladas,
- **Cuando** el sistema agrega la leyenda al PDF,
- **Entonces** deberá incluirla una sola vez como nota general del documento, sin repetirla por cada partida ni desglosarla por tipo de control.

**Criterio B4 — Generación del PDF no bloqueada por la leyenda**
- **Dado** que una cotización definitiva contiene partidas controladas,
- **Cuando** el sistema procesa la generación del PDF,
- **Entonces** deberá completar la generación y entregar el documento al cliente, independientemente del estado del catálogo del cliente respecto a sus documentos regulatorios cargados. La leyenda es informativa, no bloquea ni interrumpe el flujo.

---

### SECCIÓN C — Texto de la Leyenda según Región

**Criterio C1 — Texto de la leyenda para clientes México**
- **Dado** que el cliente de la cotización definitiva tiene Región = México,
- **Cuando** el sistema arma el texto de la leyenda,
- **Entonces** deberá usar la denominación México: referenciar **Licencia Sanitaria** y **Aviso de Responsable Sanitario** como documentos requeridos para procesar el pedido.

> **⚠️ Pendiente** — El texto exacto queda como decisión de UX/Marketing del cliente. Propuesta base: *“Producto sujeto a regulación sanitaria. Para procesar el pedido se requiere: Licencia Sanitaria vigente y Aviso de Responsable Sanitario.”*

**Criterio C2 — Texto de la leyenda para clientes Perú**
- **Dado** que el cliente de la cotización definitiva tiene Región = Perú,
- **Cuando** el sistema arma el texto de la leyenda,
- **Entonces** deberá usar la denominación Perú con los documentos regulatorios equivalentes según DIGEMID.

> **⚠️ Pendiente** — Denominación y texto exactos para Perú pendientes de confirmar con el cliente. Mientras tanto se mantiene placeholder.

---

## Notas de Implementación

- Funcionalidad **nueva** en el módulo Cotizar lo Cotizable, que es un módulo preexistente en PQF2. Este es el único cambio funcional que R16 introduce al módulo; el resto del módulo opera conforme al sistema preexistente sin modificaciones.
- La leyenda es general a nivel del PDF: aparece una sola vez como nota informativa del documento, no por partida. El detonante es la presencia de al menos una partida de producto controlado (Mundial, Nacional u Origen) en la cotización definitiva.
- La leyenda **no bloquea** la generación del PDF. La validación bloqueante para que un pedido pueda procesarse cuando el cliente tiene productos controlados vive en el módulo **Pretramitar Pedido** (requisito TPSC-RE-FU-009), que verifica que el cliente tenga registrados los documentos regulatorios en su Catálogo. La leyenda en cotización funciona como recordatorio proactivo previo a esa validación.
- La leyenda funciona como complemento al ciclo regulatorio del proyecto: carga de documentos regulatorios en el Catálogo de Clientes → recordatorio informativo en cotización definitiva → validación bloqueante en Pretramitar Pedido.

> **⚠️ Pendiente** — Ubicación exacta de la leyenda en el PDF (encabezado, sección dedicada, pie de página). Queda como decisión de diseño UI a definir en sprint de desarrollo.

> **⚠️ Pendiente** — Texto exacto de la leyenda para clientes México. El texto definitivo es una decisión de UX/Marketing del cliente.

> **⚠️ Pendiente** — Denominación exacta de los documentos regulatorios equivalentes para clientes con Región Perú según normativa DIGEMID. Mientras no se confirme, el texto para Perú queda como placeholder.
