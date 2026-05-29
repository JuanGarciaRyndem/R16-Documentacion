# Reporte de Revisión de Requisitos — TPSC

---

## Observaciones Generales

**Estatus:** 🔄 En proceso

- En todos los requisitos, las Reglas y los Criterios de aceptación son muy similares, casi idénticos. Considerar cambiar la redacción, separar cuáles son reglas y cuáles criterios de aceptación o, en su defecto, eliminar la duplicidad.
- Hay reglas o criterios tachados. Como es una Matriz inicial que aún no se aprueba por el cliente, se considera conveniente eliminar lo tachado para que no esté sucia de inicio.

---

## TPSC-RE-FU-001

**Estatus:** ✅ Atendido

### Observaciones

- **(Historia de Usuario)** No es necesario especificar los módulos en los que se utilizará el catálogo de cuentas bancarias de Proquifa. Con mencionar que se necesita el catálogo para el sistema es suficiente.
- **(Requisito)** A nivel requisito no debería definirse la estructura técnica de la solución (se define el *qué*, no el *cómo*). Sugerencia: quitar la redacción sobre tablas, quitarlo también del alcance y de Reglas y Criterios. Así mismo, no debe tomarse como base sistemas legados para definir la estructura del catálogo.
- **(Requisito)** La redacción sobre que no tiene interfaz gráfica y que se gestiona desde BD son criterios de aceptación, no parte del requisito.
- **(Alcance)** ¿Por qué se está considerando en buzón de pagos? ¿Es por la propuesta de IA? — *Buzón de Pagos (identificación de pagos entrantes contra las cuentas del grupo).*
- **(Criterios de aceptación)** ¿Por qué en la Regla 1 se decide que debe ser modelado con la estructura de Legacy?
- **(Criterios de aceptación)** La Regla 2 y 3 deberían redactarse como un *qué*, no como un *cómo*, ya que la solución técnica puede ser diferente por diseño pero cumplir con la regla.
- **(Criterios de aceptación)** La Regla 5 y Criterio 5 no deberían especificarse. Se está entrando en temas de estrategia de borrado del sistema (borrado lógico). No es que se "filtre por cuentas activas" sino que son las cuentas que existen en sistema. Si técnicamente está inactiva, para negocio no existe. Puede confundir los términos *Existir* vs *Activo*. Sugerencia: cambiar a "Existe o no existe en sistema".
- Agregar requisito de mantenimiento post-go-live del catálogo (¿quién actualiza las cuentas y cómo si no hay UI?).

### Notas adicionales

- Guía de resolución para dar de alta, baja o actualizar cuentas bancarias.

### Resumen de cambios aplicados

- **(Alcance)** El Buzón de Pagos se marcará como pendiente con `**` (depende de propuesta IA).
- **(Criterios - Regla 1)** Eliminada la referencia al modelado con estructura de Legacy. La referencia técnica a tabla `CuentaBanco` BD PConnect se conservó solo en Observaciones para el equipo interno.
- **(Mantenimiento post-go-live)** Actualizado y descrito de mejor manera en Criterios C1 y C2.
- HU simplificada (sin enumerar módulos consumidores).
- Requisito reescrito sin estructura técnica, sin referencia a Legacy, sin "no UI/gestión BD" (movido a Criterios).
- Reglas reescritas como enunciados declarativos: de 5 reglas *Cómo* pasaron a 4 reglas declarativas del *Qué*.
- Cambio "filtrar por cuentas activas" → "Existe vs No existe" aplicado en Regla 2 y Criterios B1, B2.
- Criterios organizados en 3 secciones: **A** (Disponibilidad), **B** (Estado de existencia), **C** (Gestión).
- "No tiene UI, se gestiona en BD" movido del Requisito a Criterios C1, C2, C3.
- Riesgos renumerados consecutivamente desde 1.

---

## TPSC-RE-FU-002

**Estatus:** ✅ Atendido

### Observaciones

- **(Criterios de aceptación - Regla 4 / Criterio 7)** ¿Qué sucede si el pendiente ya existe, está asignado a un Cobrador y el Coordinador de Tesorería cambia el cobrador asignado? ¿El pendiente cambia de cobrador o se mantiene con el cobrador que lo nació?
- Especificar que queda fuera del alcance colocar el campo *Cobrador* en el alta de cliente en *Cotizar lo Cotizable*, ya que esa alta está pensada únicamente para poder cotizar, no para gestión del cliente.

### Notas adicionales

- Cartera de cobradores.
- Estimar migrar Mailbot a un agente inteligente.
- Considerar que cuando se asigne un cobrador a un cliente que no tenía cobrador, se deben visualizar sus pendientes (manejo correcto de carteras) — *Criterio 5 → ahora Criterios C2 + C3*.

### Resumen de cambios aplicados

- HU simplificada.
- Reglas reescritas como enunciados declarativos: de 5 reglas *Cómo* pasaron a 5 reglas declarativas del *Qué*.
- **Decisión sobre cambio de Cobrador:** El filtrado de bandeja es dinámico — al reasignar el Cobrador de un cliente, todos sus pendientes y pagos vigentes se mueven inmediatamente a la bandeja del nuevo Cobrador. Documentado en Regla 4 (Filtrado dinámico) y Criterio C2 (Redistribución inmediata al reasignar).
- **Exclusión del campo Cobrador en Cotizar lo Cotizable:** agregada en Alcance "No aplica a".
- Criterios organizados en 3 secciones: **A** (Visibilidad y edición), **B** (Selector), **C** (Filtrado de bandeja).

---

## TPSC-RE-FU-003

**Estatus:** ✅ Atendido

### Observaciones

- **(Requisito)** La visualización del PDF cargado se realiza abriendo el archivo en una pestaña nueva del navegador — esto va en criterios. Se debe revisar la viabilidad técnica, ya que algunos navegadores por defecto (o según configuración del usuario) descargan el archivo en lugar de abrirlo en una nueva pestaña.
- **(Requisito)** "El sistema almacena únicamente la versión vigente de cada documento (sin historial de versiones)" — esto va en criterios. Actualmente no se eliminan archivos de MinIO; el usuario elimina el archivo en registros pero se mantiene en MinIO. Especificar si se mantendrá el mismo mecanismo o si se considerará limpiar los archivos anteriores del almacenamiento MinIO.
- El Criterio 8 debe moverse a otro requisito, ya que la carga de archivos y la habilitación condicionada a esos documentos son funcionalidades separadas.

### Resumen de cambios aplicados

- Frase sobre apertura en pestaña nueva eliminada del Requisito y reformulada agnósticamente en **Criterio C1**: el sistema entrega el archivo al navegador y el comportamiento de apertura depende de la configuración del navegador y del usuario.
- Frase sobre almacenamiento sin historial eliminada del Requisito y reformulada en **Regla 3 + Observaciones**: en pantalla solo se muestra la versión vigente; los archivos físicos en MinIO se mantienen sin purga automática (se conserva el mecanismo actual del backend).
- **Criterio 8** (habilitación de pretramitación) eliminado de esta fila — pertenece al módulo Pretramitar Pedido (**TPSC-RE-FU-009**).
- Reglas reescritas como enunciados declarativos: de 7 reglas *Cómo* pasaron a 5 reglas declarativas del *Qué*.
- Criterios organizados en 5 secciones: **A** (Visibilidad y acceso), **B** (Carga y formato), **C** (Visualización), **D** (Reemplazo y eliminación), **E** (Alcance de validación).

---

## TPSC-RE-FU-004

**Estatus:** ✅ Atendido

### Observaciones

- Validar los catálogos con el cliente, ya que actualmente en ProquifaNet 2 son diferentes.

### Notas adicionales

- ¿Hay validación de formato correcto de SAT actualmente?
- Considerar validación con expresión regular u otro mecanismo donde el formato esté en BD asociado a cada región para no hardcodearlo (e.g., `SELECT formatosValidacion WHERE id_region = cliente.idRegion`).
- Hacer validación de RFC con un *debounce* después de un tiempo sin escribir en el campo, similar a la búsqueda, para no saturar las consultas.
- Agregar catálogo de números de contribuyentes válidos para Perú (RUC, 11 dígitos).

### Resumen de cambios aplicados

- Reglas reescritas como enunciados declarativos: de 7 pasaron a 6. Reglas 5 y 6 originales (catálogos de Tipo de Sociedad y Régimen Fiscal) consolidadas en una sola **Regla 5** con redacción agnóstica al catálogo específico, hasta resolver el pendiente con el cliente.
- Criterios organizados en 5 secciones: **A** (Visualización y acceso), **B** (Validación RFC México), **C** (Validación RUC Perú), **D** (Selectores), **E** (Persistencia y consumo posterior).
- Criterios D1 y D2 reformulados sin listar opciones literales: el catálogo se consulta al sistema según Región. La lista de opciones se gestiona como dato paramétrico.
- Observación sobre catálogos PQF2 vs catálogos del cliente formalizada como pendiente en Observaciones. Se elaboró archivo adjunto `TPSC-RE-FU-004_Equivalencias_MX_PE.xlsx` con cruce de PQF2, archivo del cliente y catálogo SAT vigente.
- Riesgos renumerados consecutivamente (antes 1 y 3; ahora 1 y 2).
- **Pendientes preservados:** confirmación del régimen Perú "Régimen para Personas Naturales" y homoclave del RFC.
- **Nuevos pendientes formalizados:** validación local del RUC en PQF2, denominación oficial S.A.C.S. para Perú, consolidación de catálogos definitivos para ambas regiones.

---

## TPSC-RE-FU-005

**Estatus:** ✅ Atendido

### Observaciones

- **Regla 6:** ¿El porcentaje es a nivel cliente o por producto?
- **Regla 3 / Riesgo 1:** Deben ser campos independientes. Por el tipo de dato, seguramente se creará un catálogo nuevo de *tipo de comprobante* en el cual solo Perú tendrá opciones.
- Si se hará facturación para Perú se debe contemplar la detracción especificada en el Riesgo 7, de lo contrario no funcionará el timbrado.
- Revisar si Criterio 2 y 3 son el mismo catálogo — si son catálogos diferentes y se colocan en el mismo, puede haber complicaciones al momento de la facturación.
- **Criterio 4:** son catálogos separados, por tanto campos separados; no deben estar juntos.
- **Criterio 5.1:** revisar, ya que posiblemente lo que se muestra como *Forma de pago* para Perú en realidad es el *Método de pago*.

### Notas adicionales

- **Criterio 2** (→ Reglas 2 y 6, Criterios B1-B3 y C1-C2): considerar mapeo hacia Legacy de estas opciones en catálogo y en facturas o pedidos si aplica.
- **Criterio 6** (→ Regla 8 / Criterio D2): el valor `%` debe ser configurable a nivel BD.

### Resumen de cambios aplicados

- El campo de dimensión temporal del pago para Perú se renombró de "Forma de Pago" a **"Condición de Pago"** (Contado/Crédito), aclarando que es el equivalente conceptual del Método de Pago mexicano (PUE/PPD) y que para Perú no se captura el medio de pago específico porque la normativa SUNAT no lo exige.
- **Uso de CFDI (México)** y **Tipo de Comprobante (Perú)** se modelan como campos independientes con catálogos separados (conceptos fiscales distintos).
- Los catálogos de México (Forma de Pago y Uso de CFDI) se documentaron con las opciones reales de PQF2. Se identificó que el catálogo de Forma de Pago no usa las claves del catálogo SAT — queda pendiente evaluar el mapeo requerido para el timbrado del CFDI.
- Las banderas **Agente de Retención IGV** y **Sujeto a Detracción** quedaron sujetas a confirmación de aplicabilidad con el cliente para evitar desarrollo innecesario. Se elaboró archivo adjunto con análisis de los tres mecanismos tributarios peruanos (Retención, Detracción, Percepción).
- **Porcentaje de detracción:** según normativa SUNAT, la detracción aplica por bien o servicio (R.S. 183-2004/SUNAT), no por cliente. La bandera a nivel cliente es un indicador; la tasa real se determina por producto. Queda pendiente confirmar con el cliente el nivel de captura (cliente, producto o catálogo). Referencia: Regla 8, Criterio D2, Riesgo 5.
- La Percepción del IGV se documentó como condición del emisor (no del cliente) e incorporada como **Brecha 4**.
- Se eliminaron las reglas de edición por operación en otros módulos (tachadas en el original), ya que pertenecen a los módulos Validar Cobro, Factura por Adelantado y Tramitación.
- Reglas reescritas como enunciados declarativos: de 11 a 9.
- Criterios organizados en 5 secciones: **A** (Visualización), **B** (Campos México), **C** (Campos Perú), **D** (Banderas tributarias), **E** (Obligatoriedad y persistencia).
- Riesgos renumerados consecutivamente (antes saltaba el 2; ahora 1–5).

---

## TPSC-RE-FU-006

**Estatus:** ✅ Atendido

> Sin observaciones.

---

## TPSC-RE-FU-007

**Estatus:** ✅ Atendido

### Observaciones

- Agregar que se envía únicamente en cotizaciones definitivas. En cotizaciones de investigación (donde no van productos y solo se coloca una leyenda de que su cotización se está trabajando) no debería ir, ya que ese documento es temporal.

### Notas adicionales

- La leyenda, al ser diferente por región, debe almacenarse en BD a nivel región para no hardcodearla en la generación del PDF.

### Resumen de cambios aplicados

- Se incorporó que la leyenda regulatoria aplica **únicamente a cotizaciones definitivas**. Las cotizaciones de investigación no la incluyen. Documentado en Regla 1, Criterio A1, Requisito, Alcance y Observaciones.
- Reglas reescritas como enunciados declarativos: de 5 a 6 (se agregó la regla de aplicabilidad a definitivas).
- Criterios organizados en 3 secciones: **A** (Aplicabilidad a cotizaciones definitivas), **B** (Inclusión de la leyenda), **C** (Texto según Región).
- El Criterio 8 (variante dinámica que consulta el Catálogo de Clientes) se retiró como criterio y quedó como propuesta abierta en Observaciones, ya que es una alternativa a evaluar con el cliente.
- Riesgos renumerados consecutivamente (antes 3 y 4; ahora 1 y 2).

---

## TPSC-RE-FU-008

**Estatus:** ✅ Atendido

### Observaciones

- **(Aplica a)** Para la clasificación automática no hay criterios configurables; el mailbot se entrena con una base de conocimiento y si se requiere ajustar se debe realizar un nuevo entrenamiento.
- Eliminar menciones de envío de correo cuando se detecta una inconsistencia — ese es criterio de otro requisito. Aquí solo debe hacerse mención de que se elimina el buzón cuando se marca como inconsistencia.
- **Regla 1:** No existe una clasificación "No cobro". La clasificación ocurre de la misma forma que para los demás pendientes (Cotización, Pedido); solo se agrega la categoría *Cobro*.
- ¿Qué sucede si llega un correo de un cliente no registrado y el mailbot lo clasifica erróneamente como Cobro? ¿Quién lo ve?
- **Criterio 9:** Actualmente los Buzones permiten eliminar pendientes. ¿Estamos seguros de que para Cobros no puede eliminarse? Si no puede eliminar, indicar que tendrá la salida existente para clasificarlo como *Otros* y sea el ESAC quien pueda eliminarlo.

### Notas adicionales

- Realizar estimación de un nuevo mailbot integrando IA para la clasificación.

### Resumen de cambios aplicados

- La clasificación automática se documentó como ejecutada por el **Mailbot**, que agrega la categoría *Cobro* al modelo existente (Cotización, Pedido, Otros). Se eliminó la noción de clasificación "cobro/no-cobro" y la de "criterios configurables".
- Se retiró el detalle del envío de correo de notificación ante inconsistencia. Ese comportamiento pertenece al módulo Validar Cobro; en este requisito solo se contempla que el pendiente del Buzón se elimina automáticamente al marcar como inconsistencia.
- **Manejo de eliminación:** el Buzón de Cobros no ofrece eliminación directa al Gestor de Cobranza. La salida para correos mal clasificados es reclasificarlos al buzón de *Otros*; desde ahí, el rol ESAC puede eliminarlos. Documentado en Regla 9 y Criterio D2.
- Se reforzó el pendiente sobre correos de cobro cuyo cliente no es identificable o no está registrado. Queda pendiente de confirmar con el cliente.
- Se precisó el criterio de aplicabilidad regional (**Criterio E1**): el Buzón opera con la misma mecánica en ambas regiones pero segregando siempre por la región del cliente.
- Reglas reescritas como enunciados declarativos: de 8 a 10.
- Criterios organizados en 5 secciones: **A** (Clasificación y recepción), **B** (Reflejo y pendiente), **C** (Ciclo de vida), **D** (Acciones del Gestor y UX), **E** (Aplicabilidad regional).
- Riesgos renumerados consecutivamente (antes 3 y 4; ahora 1 y 2).

---

## TPSC-RE-FU-009

**Estatus:** 🔲 No revisado

> Sin observaciones.

---

## TPSC-RE-FU-010

**Estatus:** 🔲 No revisado

### Notas adicionales

- En este requisito se debe analizar y generar un **catálogo explícito o implícito de estados de pedidos** (Orden de compra, pedido en curso, pedido confirmado, etc.) para tener claridad de en qué momento es una OC, en qué momento es un pedido con folio interno pero no confirmado, y en qué momento ya es un pedido confirmado. Esto dará claridad a los distintos módulos (facturación, reportes, etc.).
- Considerar la atención del ticket [DTP2-86](https://newryndem.atlassian.net/browse/DTP2-86) como parte de este requisito.

---

## Resumen de estatus

| Requisito | Estatus |
|-----------|:-------:|
| Observaciones generales | 🔄 En proceso |
| TPSC-RE-FU-001 | ✅ Atendido |
| TPSC-RE-FU-002 | ✅ Atendido |
| TPSC-RE-FU-003 | ✅ Atendido |
| TPSC-RE-FU-004 | ✅ Atendido |
| TPSC-RE-FU-005 | ✅ Atendido |
| TPSC-RE-FU-006 | ✅ Atendido |
| TPSC-RE-FU-007 | ✅ Atendido |
| TPSC-RE-FU-008 | ✅ Atendido |
| TPSC-RE-FU-009 | 🔲 No revisado |
| TPSC-RE-FU-010 | 🔲 No revisado |
