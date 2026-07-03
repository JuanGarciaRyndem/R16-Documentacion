# Revisión — [R16A-RE-FU-014][DIS-SOL] Diseño de la solución

| Campo | Valor |
|---|---|
| **Documento revisado** | `[R16A-RE-FU-014][DIS-SOL] Diseño de la solución.pdf` v1.0 (23 jun 2026) |
| **Autor del diseño** | Osmar Calderón Vázquez |
| **Revisor** | Juan David García Cruz |
| **Fecha de revisión** | 02 jul 2026 |
| **Versión del requisito base** | R16A-RE-FU-014 (Estatus: Propuesto) |
| **Documentos cruzados** | `R16A-RE-FU-014.md`, `R16A-RE-FU-014-Back.md`, `R16A-RE-FU-014_BD.md`, `R16A-RE-FU-014-Tareas.md`, `[R16A-RE-FU-013][DIS-SOL] Diseño de la solución.pdf`, `R16A-RE-FU-013_DIS-SOL_Revision.md` |

---

## Resumen

014 es un documento diferencial que remite casi por completo al DIS-SOL de 013 (ya aprobado). Con 013 disponible se pudo verificar el diseño de 014 contra el contrato real en vez de contra supuestos. El enfoque diferencial es correcto y buena práctica; los problemas encontrados son de **precisión** (nombres de endpoint/clase que no coinciden con el contrato aprobado de 013) y de **documentación complementaria desactualizada** (`-Back.md`/`-Tareas.md`/`_BD.md` describen una arquitectura anterior a la del DIS-SOL). Dos puntos que se habían marcado como hallazgos en la primera pasada quedaron resueltos tras aclaración directa del equipo y se documentan aquí solo como notas de contexto.

---

## Hallazgos críticos (bloqueantes)

### H-01 — Endpoint y nombre de clase citados en 014 no coinciden con el contrato aprobado de 013

- **Sección del PDF:** "Tareas propias de RE-014" — T1 y T2 citan *"Módulo: `ProquifaDotNetFinanzas` — endpoint T4 (`POST /tpProformaPedido/EnviarCorreo`)"*; la tabla "Referencia a RE-013" cita `GeneradorFolioProforma` como el foliador.
- **Lo que dice el diseño aprobado de 013:** El endpoint interno real es `POST /v1/api/proforma/send-email/process` (Finanzas), y el front-facing es `POST /v1/api/proforma/send-email/{orderId}` (ProquifaDotNet). La clase del foliador se llama `ProformaFolioGenerator`, no `GeneradorFolioProforma`.
- **Acción:** Corregir ambas referencias en el PDF de 014 para que citen el contrato real de 013. Un desarrollador que implemente 014 a partir de este documento buscaría un endpoint y una clase que no existen.

---

## Brechas (no bloqueantes pero deben corregirse)

### H-02 — `-Back.md`, `-Tareas.md` y `_BD.md` de 014 describen la arquitectura previa al DIS-SOL de 013

- **Sección del PDF:** No aplica directamente al PDF; aplica a los documentos complementarios de 014.
- **Problema:** `R16A-RE-FU-014-Tareas.md` ubica T1/T2 en `tpPedidoTramitarTransaccionBO.cs` (`L05.TramitarPedido\Liberar`, `WebApi.Logistica`). El DIS-SOL aprobado de 013 confirma que esa clase **ya no se usa** en este flujo (*"Sin modificaciones en `tpPedidoTramitarTransaccionBO` — no se reutiliza en este flujo"*) y que las validaciones ahora ocurren en Fase 1 de T4, dentro de `ProquifaDotNetFinanzas`. `_BD.md` de 014 tampoco refleja que `tpPedidoProformaPedido` se elimina (RT-06 de 013) ni que aparecen `tpProformaConsecutivo` y `CatEstadoTpPedido`.
- **Acción:** Actualizar `-Back.md`, `-Tareas.md` y `_BD.md` de 014 para alinearlos con la arquitectura del DIS-SOL de 013, igual que se señaló para los documentos equivalentes de 013 en `R16A-RE-FU-013_DIS-SOL_Revision.md` (H-01).

### H-03 — Falta cobertura explícita del Criterio D2 (cancelación del pedido)

- **Sección del PDF:** No mencionado en ningún lado — ni cubierto, ni excluido, ni remitido a RE-013/RE-010.
- **Acción:** Agregar una línea remitiendo a RE-FU-010 (o al criterio equivalente de 013, marcado "Pendiente" en su propio DIS-SOL) para dejar explícito que no es responsabilidad de 014.

### H-04 — Encabezados de página sin actualizar

- Todas las páginas del PDF muestran "R16A-RE-FU-013." en el encabezado superior — indicio de copia de la plantilla de 013 sin actualizar. Cosmético, pero genera confusión de trazabilidad.

---

## Notas de contexto (ya aclaradas, sin acción pendiente)

- **Aplicativo `ProquifaDotNetFinanzas`:** confirmado como arquitectura real y aprobada (ver 013). No es un error de 014.
- **014 no revalida "sin controlados" en el backend (RT-03):** confirmado como decisión de diseño intencional — el flujo confía en el flag `TieneProductosControlados` calculado por trigger sobre `fnEsProductoControlado`. Documentado aquí solo para que quede explícito que es una decisión aceptada, no un descuido. Hereda la misma dependencia del ALTER FUNCTION `fnEsProductoControlado` (clave `'origen'`) que ya se rastrea para 007/009/011/013.
- **Conflicto 013 vs 009 sobre ubicación de la validación regulatoria:** en corrección del lado de R16A-RE-FU-009. No aplica a 014 de forma directa.

---

## Puntos que están bien y deben conservarse

1. Enfoque diferencial (documentar solo lo que cambia respecto a 013) — evita duplicar contenido y reduce el riesgo de que ambos documentos diverjan con el tiempo, siempre que se mantenga sincronizado con el DIS-SOL base.
2. Tabla de diferencias técnicas (Controlados, `TieneControlados()`, FAA, factura posterior) clara y fácil de auditar.
3. Reglas técnicas RT-01 a RT-04 capturan bien las variantes puntuales frente al flujo con controlados de 013.

---

## Referencias

- Requisito: `R16A-RE-FU-014.md`
- Impacto Backend (pre-DIS-SOL): `R16A-RE-FU-014-Back.md`
- Diccionario de datos (pre-DIS-SOL): `R16A-RE-FU-014_BD.md`
- Tareas (pre-DIS-SOL): `R16A-RE-FU-014-Tareas.md`
- Diseño base: `[R16A-RE-FU-013][DIS-SOL] Diseño de la solución.pdf`
- Revisión relacionada: `R16A-RE-FU-013_DIS-SOL_Revision.md`
