# 06 — R16 Tramitar Pedido sin Crédito: 6ta Sesión de Entendimiento

| Campo | Detalle |
|---|---|
| **Fecha** | 14 de abril de 2026 |
| **Hora** | 15:52 CST |
| **Sesión** | 6ta Sesión de Entendimiento |
| **Tema principal** | Módulo de Notas de Crédito — diseño, flujos, cancelación y foliado |
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
| Cornelio M. Ramírez B. | Finanzas / Proquifa |

---

## Resumen Ejecutivo

La sesión se enfocó en la **revisión exhaustiva del módulo de Notas de Crédito**: diseño visual, listado con filtros, flujo de generación, reglas de negocio para cancelación de facturas, estados, motivos y foliado. Se diferenció el flujo de nota de crédito del flujo de devolución de dinero, y se definieron las tres entradas base del módulo.

---

## Temas Tratados

### 1. Propuesta visual del Módulo de Notas de Crédito

- El módulo mostrará **todas las notas de crédito generadas** con opción de crear nuevas.
- **Filtros disponibles:** rango de fechas, cliente, emisor, estado (aplicada / disponible), búsqueda por factura o pedido interno.
- **Columnas del listado:** fecha, cliente, cobrador, folio, XML, emisor, monto, factura asociada, pedido interno y estado.
- **Folio:** al hacer clic muestra el **PDF**. Columna XML permite **descargar el archivo XML**.
- **Trazabilidad:** La trazabilidad completa parte del pedido. Se sugirió discutir el enriquecimiento de la trazabilidad con Daniel en una ocasión posterior.

### 2. Estados de las Notas de Crédito

- Los estados en pantalla (ej. “por aplicar”, “vigente”) requieren definición formal.
- Pendiente: determinar si son **estados del SAT** o **estados internos** de Proquifa.
- Valdemar Farina Sanchez coordinará con Cornelio M. Ramírez B. y Mayra Franco para definirlos.

### 3. Motivos y entradas para generación de Notas de Crédito

- **Tres entradas base** definidas para el flujo:
  1. **Devolución de producto** (total o parcial).
  2. **Bonificación / descuento**.
  3. **Error en facturación**.
- El flujo de **devolución de dinero** es un proceso **alterno y separado** del módulo de notas de crédito.
- El concepto de la nota de crédito debe incluir **las partidas con materialidad** (no solo “devolución” genérico).

### 4. Cancelación de facturas al generar una Nota de Crédito

- La cancelación de una factura y la generación de una nota de crédito son posibles bajo dos condiciones:
  1. La nota de crédito es por la **totalidad de la factura**.
  2. La operación ocurre dentro del **mes corriente** de emisión de la factura.
- El área de **Tesorería** decide si se cancela o no la factura bajo estas condiciones.
- Al cancelar la factura, se puede incluir un **comentario o nota adicional** como aclaración.

### 5. Autorización y decimales en Notas de Crédito

- Mayra Franco propuso firma de autorización para notas de crédito por **montos mayores a $50,000 MXN**.
- Se discutió el número de decimales: Mayra Franco sugiere **4 decimales**. Pendiente revisar reglas del SAT para timbrado.

### 6. Resumen final de la Nota de Crédito

- El resumen se muestra después de enviar el correo. Incluye: folio, ID del SAT (UUID), pack (Turbo Pack), certificación SAT, RFC y datos específicos de la nota.
- Se muestran **saldos y totales actualizados**.
- La información del XML se conserva por un **mínimo de 5 años** para cancelación ante el SAT.
- **Acciones disponibles:** descargar XML, descargar PDF, reenviar nota por correo, cancelar nota de crédito (opcional según alcance).
- El resumen incluye el **folio y UUID (folio fiscal)** de la factura original.

### 7. Reglas de cancelación de la Nota de Crédito

- Aplican las mismas reglas del SAT que un CFDI:
  - **Dentro de 24 horas:** se puede cancelar sin autorización del cliente.
  - **Después de 24 horas:** se requiere autorización del cliente.
- **Escenario A — Factura original NO cancelada ante el SAT:** cancelar la nota de crédito inhabilita la cancelación hecha por medio de esta, y se puede expedir una nueva nota de crédito.
- **Escenario B — Factura original YA cancelada ante el SAT:** si la nota de crédito estaba mal, ya no se puede generar una nueva nota. Se debe emitir una **carta al cliente** para que use su saldo.
- En ambos escenarios, la acción del sistema es simplemente **cancelar la nota de crédito** sin acción adicional sobre la factura original.

### 8. Foliado de Notas de Crédito y facturas de prepago

- Se discutió la compatibilidad entre el foliado de ProquifaNet 2 y el sistema Legacy (uno numérico, otro con letras).
- Rose Ríos Gómez propuso usar un **nuevo número de serie distinto** para las facturas por adelantado en PQF2.
- Si todo se cierra del lado de PQF2, se puede evitar la transferencia al Legacy, ahorrando una integración.

### 9. Reportes pendientes y cierre del módulo

- Pendiente de revisar los **tres reportes** (pedidos, facturas, cobros) para concluir el módulo.
- Se programó sesión específica para el día siguiente, **4:00 PM – 6:00 PM** (2 horas estimadas).

---

## Pendientes y Compromisos

| # | Responsable | Compromiso | Fecha límite |
|---|---|---|---|
| 1 | Valdemar Farina Sanchez | Coordinar con Cornelio y Mayra para definir estados de notas de crédito (SAT vs. internos) | Próxima sesión |
| 2 | Sara Sánchez / Larissa Calvo | Revisar y estandarizar estructura de foliado de documentos (notas y facturas) | Próxima sesión |
| 3 | Sara Sánchez | Definir reglas del consecutivo de folios (reinicio, número máximo) e incluir en diccionario de datos | Próxima sesión |
| 4 | Sara Sánchez | Evaluar reglas de generación de folios nuevos para prepagos y verificar compatibilidad con caracteres de Legacy | Próxima sesión |
| 5 | Larissa Calvo / Irma Andrade Aguado | Agendar sesión específica para revisión de reportes pendientes (4 PM a 6 PM, 2 horas) | Inmediato |

---

## Próxima Sesión

- **Fecha propuesta:** 15 de abril de 2026, 4:00 PM – 6:00 PM.
- **Temas a revisar:** Reportes de pedidos, facturas y cobros.
- **Objetivo:** Concluir la revisión del módulo completo.
