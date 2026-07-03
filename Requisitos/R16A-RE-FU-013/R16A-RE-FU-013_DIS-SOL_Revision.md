# Revisión — [R16A-RE-FU-013][DIS-SOL] Diseño de la solución

| Campo | Valor |
|---|---|
| **Documento revisado** | `[R16A-RE-FU-013][DIS-SOL] Diseño de la solución.pdf` v1.0 (19 jun 2026) |
| **Autor del diseño** | Osmar Calderón Vázquez |
| **Revisor** | Juan David García Cruz |
| **Fecha de revisión** | 02 jul 2026 |
| **Estatus** | ✅ Aprobado — hallazgos de esta revisión son seguimiento de documentación, no bloquean construcción |
| **Documentos cruzados** | `R16A-RE-FU-013.md`, `R16A-RE-FU-013-Back.md`, `R16A-RE-FU-013_BD.md`, `R16A-RE-FU-013-Tareas.md`, `R16A-RE-FU-009_DIS-SOL_Revision.md` |

---

## Resumen

El diseño ya está aprobado y define correctamente la arquitectura de referencia para todo el bloque Prepago (013/014/015): `ProquifaDotNet` genera `FolioPedidoInterno` y delega a `ProquifaDotNetFinanzas`, que concentra validaciones, foliador de proforma, PDF, MinIO, correo y cierre de pendiente. Esta revisión no cuestiona esa decisión — se limita a dos brechas de documentación que conviene cerrar para que `_BD.md`/`-Back.md`/`-Tareas.md` (y los diseños derivados como 014) no queden desalineados con lo aprobado.

---

## Brechas (seguimiento post-aprobación, no bloqueantes)

### H-01 — RT-10 describe mal el origen de `TieneProductosControlados`

- **Sección del PDF:** §2 Reglas técnicas aplicadas, RT-10.
- **Lo que dice el diseño:** *"`TieneProductosControlados` se calcula en `vTramitarPedidoDetalle` mediante un nuevo método que consulta las partidas del pedido contra el catálogo de productos... No se reutiliza `fnEsProductoControlado` para este cálculo."*
- **Lo que se confirmó en la revisión:** El campo se mantiene mediante un **trigger** que sí deriva de `fnEsProductoControlado` — no es un método independiente que recalcula desde cero.
- **Acción:** Corregir la redacción de RT-10 para reflejar el mecanismo real (trigger sobre `fnEsProductoControlado`), y documentar en `_BD.md` en qué tabla vive el campo materializado y qué evento dispara el trigger (alta/edición de partidas). Ninguno de los dos documentos lo menciona actualmente.

### H-02 — El campo hereda el gap ya conocido de `fnEsProductoControlado` (clave `'origen'`)

- **Sección del PDF:** RT-10, relacionado con Flujo 4 Fase 1 (`TieneControlados(IdTPPedido, IdRegion)`).
- **Problema:** Las revisiones de R16A-RE-FU-007/009/011 ya señalan que `fnEsProductoControlado` no detecta la clave `'origen'` (ALTER FUNCTION pendiente y compartido entre esos tres requisitos). Como `TieneProductosControlados` depende de esa misma función vía trigger, hereda el mismo hueco: un producto controlado por Origen podría no marcarse correctamente hasta que se aplique el ALTER.
- **Acción:** Añadir R16A-RE-FU-013 (y por extensión 014/015) a la lista de requisitos que dependen del ALTER FUNCTION `fnEsProductoControlado` con `'origen'`. No requiere cambio de diseño — es una dependencia a rastrear.

### Nota de seguimiento — Conflicto con R16A-RE-FU-009 sobre ubicación de la validación regulatoria

El PDF (§2 Alcance, "No se consideran") asume que la validación de documentos regulatorios del cliente es responsabilidad de Pretramitar Pedido vía RE-FU-009. La revisión de `R16A-RE-FU-009_DIS-SOL_Revision.md` (H-02) había señalado que el diseño de 009 movía esa validación a `TramitarPedidoBO` en L05, no a Pretramitar. Confirmado con el equipo: la corrección ya está en curso del lado de 009 para que la validación quede donde corresponde. No se requiere acción en 013 — se deja esta nota únicamente para trazabilidad entre ambos documentos.

---

## Puntos que están bien y deben conservarse

1. Separación clara de fases en T4 (pre-computación fuera de transacción, bloqueo e inserción atómica, post-procesamiento I/O) — buen manejo de la concurrencia del foliador sin extender el alcance del `UPDLOCK`.
2. Idempotencia explícita de T4 (RT-12) y de `FolioPedidoInterno` (RT-15) — cubre bien el caso de reintentos tras fallos parciales.
3. Tabla de manejo de errores por fase (Fase 1/2/3) muy clara sobre qué queda persistido y qué es reintentable en cada escenario de falla.
4. Diagramas de secuencia (Flujo 1, 2, 4) legibles y consistentes con el texto.

---

## Referencias

- Requisito: `R16A-RE-FU-013.md`
- Impacto Backend (pre-DIS-SOL): `R16A-RE-FU-013-Back.md`
- Diccionario de datos (pre-DIS-SOL): `R16A-RE-FU-013_BD.md`
- Diseño aprobado: `[R16A-RE-FU-013][DIS-SOL] Diseño de la solución.pdf`
- Revisión relacionada: `R16A-RE-FU-009_DIS-SOL_Revision.md` (H-02)
