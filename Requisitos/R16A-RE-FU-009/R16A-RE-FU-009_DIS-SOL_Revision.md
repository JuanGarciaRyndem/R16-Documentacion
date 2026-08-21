# Revisión — [R16A-RE-FU-009][DIS-SOL] Diseño de la solución

| Campo | Valor |
|---|---|
| **Documento revisado** | `R16A-RE-FU-009-DIS-SOL-Back.pdf` v1.0 (18 jun 2026) |
| **Autor del diseño** | Cristóbal Sebastián García Coss |
| **Revisor** | Juan David García Cruz |
| **Fecha de revisión** | 02 jul 2026 |
| **Versión del requisito base** | R16A-RE-FU-009 (Estatus: Propuesto) |
| **Documentos cruzados** | `R16A-RE-FU-009.md`, `R16A-RE-FU-009-Back.md`, `R16A-RE-FU-009_BD.md` |

---

## Resumen

El diseño está bien estructurado (flujo principal, subflujos, diagramas de secuencia y de decisión, manejo de errores, estrategia de pruebas y Definition of Ready) y mantiene consistencia con `-Back.md`/`_BD.md` en la decisión de arquitectura OBS-023 (validación movida a `TramitarPedidoBO` en L05). **Hay 4 hallazgos críticos que bloquean el inicio de desarrollo** y 4 brechas menores. La causa raíz de los hallazgos críticos es que el diseño se apoya en una versión de la Historia de Usuario (Jira R16A-79, con "Regla 6" y alcance limitado a México) que **no coincide** con el requisito `.md` vigente en el repositorio, el cual cubre México y Perú y solo llega hasta la Regla 5 sin contemplar tramitación parcial.

Antes de pasar a construcción hay que reconciliar la fuente de verdad del requisito y cerrar los gaps de scripts de BD.

---

## Hallazgos críticos (bloqueantes)

### H-01 — Perú queda fuera de alcance sin que el requisito lo respalde

- **Sección del PDF:** §1.2 Alcance / No incluye, OBS-007 (§1.3), §3.2 paso 4, Regla R1 (§3.4), AC-D (§7.1).
- **Lo que dice tu diseño:** *"Alcance México únicamente... el sistema no restringe por código [para Perú]; riesgo operativo asumido y documentado."*
- **Lo que dice el requisito vigente:** El Alcance dice *"Operaciones México y Perú: la validación aplica con la misma mecánica en ambas regiones"*; la Regla 2 exige documentos "equivalentes según normativa DIGEMID" para Perú; las Notas repiten *"Aplicable a las operaciones de México y Perú"*. Lo único pendiente en el requisito es la **denominación** exacta de los documentos DIGEMID (Riesgo 4) — no una exclusión de la validación.
- **Acción:** Si la exclusión de Perú fue una decisión real de negocio, actualizar `R16A-RE-FU-009.md` (Alcance, Regla 2, Notas) para reflejarla formalmente antes de construir. Si no fue decidida, corregir el diseño para que Perú sí ejecute la validación con los documentos DIGEMID (aun si su denominación queda como TODO de datos, no de alcance).

> **Resuelto (2026-08-21) — DUDA-027 / DUDA-029 / DUDA-051:** el cliente confirmó que Perú no soporta sustancias controladas en R16; el riesgo de tramitar/facturar un controlado de Perú se asume como riesgo operativo comunicado al cliente, sin bloqueo técnico. `R16A-RE-FU-009.md` ya quedó alineado (Alcance, Regla 2 y Notas dicen explícitamente "Aplicable a Región México... Para Región Perú no se construye validación regulatoria en esta release"). Los documentos `-Back.md` y `_BD.md` fueron actualizados en consecuencia para retirar los pendientes de denominación DIGEMID. El diseño de Cristóbal debe simplificarse para no construir la rama Perú.

### H-02 — Bifurcación por "entregas parciales" se apoya en una regla que no existe en el requisito local

- **Sección del PDF:** §3.1 pasos 6b/6c, §3.3 completo, Reglas R9/R10 (§3.4), CA-B3/CA-B6 (Validación de criterios de aceptación).
- **Lo que dice tu diseño:** Cita repetidamente *"Regla 6 de la Historia"* como fuente de las reglas de bifurcación (pedido hijo, `AceptaEntregasParciales`, partidas controladas nunca facturadas por adelantado).
- **Lo que dice el requisito vigente:** `R16A-RE-FU-009.md` solo llega hasta la **Regla 5**. La Regla 4 es explícita: *"el sistema bloquea el avance del pedido... El mensaje no especifica cuál documento falta"* — sin mención de tramitación parcial ni pedido hijo. Tampoco hay Criterios C1-C2/D/E en el `.md` (llega hasta B5); el PDF usa una numeración distinta (AC-A, AC-B, AC-C1/C2, AC-D, AC-E) que solo aparece citando "Historia R16A-79 (fuente de verdad)".
- **Acción:** Confirmar con negocio/PM cuál es la fuente de verdad vigente. Si Jira R16A-79 tiene una versión más reciente de la historia con la Regla 6 y los AC adicionales, **actualizar `R16A-RE-FU-009.md`** para que el repositorio quede alineado antes de construir — hoy un desarrollador que solo lea el `.md` no sabría que existe la bifurcación.

> **Resuelto (2026-08-14):** se cerró en sentido contrario a la Regla 6 de Jira. El cliente confirmó que el escenario de pedidos que mezclan partidas controladas y no controladas **no ocurre** en la operación real; en consecuencia se **retiró por completo** la mecánica de separación/bifurcación (pedido hijo, `AceptaEntregasParciales`, `IdPedidoOrigenControlado`). `R16A-RE-FU-009.md` quedó reescrito para que, ante documentación regulatoria faltante, el sistema retenga siempre **el pedido completo** en su folio original (Regla 4, Criterio B3), sin excepción por composición del pedido. El diseño de Cristóbal (§3.1 pasos 6b/6c, §3.3, Reglas R9/R10) debe simplificarse retirando toda la lógica de entregas parciales / pedido hijo; los objetos de BD asociados (`ppPedido.AceptaEntregasParciales`, `tpPedido.IdPedidoOrigenControlado` en `_BD.md` H-03/H-04) quedan sin sustento y deben retirarse del alcance de construcción salvo que el cliente reabra el escenario.

### H-03 — Falta el script `ALTER TABLE ppPedido ADD AceptaEntregasParciales`

- **Sección del PDF:** §5.3 Scripts de base de datos (solo lista 3 scripts: `tpPedido`, `fnEsProductoControlado`, `catUsoArchivoSistema`).
- **Problema:** Toda la lógica de bifurcación depende de `ppPedido.AceptaEntregasParciales` (§3.1, §3.3, Regla R4), pero el diseño no incluye el script para agregar esa columna. `R16A-RE-FU-009_BD.md` sí lo tenía documentado: `ALTER TABLE dbo.ppPedido ADD AceptaEntregasParciales bit NOT NULL CONSTRAINT DF_ppPedido_AceptaEntregasParciales DEFAULT (0);`
- **Acción:** Agregar el script como Script 0 (o renumerar) en §5.3, tomando la definición ya validada en `_BD.md`.

### H-04 — `tpPedido.IdPedidoOrigenControlado` tiene definiciones contradictorias entre documentos

- **Sección del PDF:** §5.3 Script 1, tabla de componentes involucrados §2.2.
- **Problema:** `R16A-RE-FU-009_BD.md` define la columna como FK self-referencing: `FOREIGN KEY REFERENCES dbo.tpPedido(IdTpPedido)` (apunta al `tpPedido` padre ya tramitado). El PDF, en cambio, describe que la columna *"almacena el `IdPPPedido` del pedido original"* (apunta a `ppPedido`, el pedido en pretramitar) y su Script 1 no define ningún `CONSTRAINT FOREIGN KEY`. Son dos diseños de la misma columna que no pueden coexistir.
- **Acción:** Decidir si la trazabilidad es `tpPedido → tpPedido` o `tpPedido → ppPedido`, agregar el constraint FK explícito en el script, y alinear `_BD.md` con el diseño final.

---

## Brechas (no bloqueantes pero deben corregirse)

### H-05 — Esquema de `catUsoArchivoSistema` inconsistente

- **Sección del PDF:** §5.3 Script 3.
- **Problema:** `_BD.md` documenta las columnas `IdCatUsoArchivoSistema`, `UsoArchivoSistema varchar(50)`, `Activo`. El Script 3 del PDF inserta usando columnas `IdCatUsoArchivoSistema`, `Clave`, `Descripcion`, `Activo` — un esquema distinto.
- **Acción:** Confirmar el esquema real de la tabla contra la BD antes de ejecutar cualquiera de los dos scripts; unificar en ambos documentos.

### H-06 — Script 2 reescribe la cadena de detección de "controlado" con una tabla no confirmada

- **Sección del PDF:** §5.3 Script 2 (`ALTER FUNCTION dbo.fnEsProductoControlado`).
- **Problema:** `_BD.md` ya documenta la cadena real y verificada: `ppPartidaPedido → MarcaFamilia (IdFamilia) → Familia (IdCatControl) → catControl (Clave)`. El PDF reescribe la función usando una tabla `catTipoControlado` unida directo a `Producto`, y el propio documento anota *"⚠️ Pendiente confirmar nombre exacto de tabla `catTipoControlado` y relación con `Producto`"*.
- **Acción:** Sustituir el Script 2 por el ALTER sobre la cadena ya confirmada en `_BD.md`, evitando introducir una tabla hipotética.

### H-07 — Riesgo 5 del requisito (puntos de entrada alternos a Tramitar) no se retoma

- **Sección del PDF:** No cubierto explícitamente en observaciones/riesgos (§1.3) ni en el flujo.
- **Problema:** El Riesgo 5 del requisito y la PARTE 3.2 de `R16A-RE-FU-009-Back.md` ya habían mapeado los flujos que podrían no pasar por la validación (Gestionar Intramitable → OC corregida, "tramitar con errores", aceptación de OC Interna). El PDF no retoma ese análisis ni confirma si esos caminos también invocan `TramitarPedidoBO.Process()`.
- **Acción:** Incorporar la tabla de `-Back.md` §3.2 al diseño (o referenciarla explícitamente) y confirmar cobertura de los tres caminos alternos.

> **Resuelto (2026-08-21) — DUDA-024:** se confirmó que todos los caminos hacia Tramitar Pedido (Pretramitar, validar ajustes, aceptar OC, pedido intramitable/"tramitar con errores") convergen en Tramitar Pedido, donde se ejecuta la validación regulatoria como último paso. Los tres caminos alternos quedan cubiertos por construcción, sin necesidad de re-validar en cada uno. `-Back.md` §3.2 y `_BD.md` (Gap 5) fueron actualizados para reflejar el cierre.

### H-08 — Numeración de criterios de aceptación no mapea 1:1 con el requisito

- **Sección del PDF:** Todo el documento usa AC-A, AC-B, AC-C1/C2, AC-D, AC-E.
- **Problema:** El requisito `.md` numera Criterio A1/A2, B1-B5. La nomenclatura del PDF viene de la Historia de Jira citada como fuente de verdad, pero mientras el `.md` no se actualice (ver H-02), no hay forma de trazar 1:1 entre ambos documentos.
- **Acción:** Una vez resuelto H-02, renumerar para que ambos documentos usen el mismo esquema de IDs.

---

## Puntos que están bien y deben conservarse

1. **Decisión OBS-023 (validación en `TramitarPedidoBO`, L05)**: consistente entre el PDF, `-Back.md` y `_BD.md` — buena señal de que la arquitectura ya está acordada entre los tres documentos.
2. **Nomenclatura del proyecto**: usa correctamente "ProquifaDotNet" sin el sufijo "-R14".
3. **Diagramas de secuencia (§4.1) y de flujo de decisión (§4.2)**: claros y completos, cubren todas las ramas (sin controlados, Perú, válido, inválido × parcial/total).
4. **§6 Manejo de errores y excepciones**: cubre 8 escenarios incluyendo casos borde (endpoint caído, 404, `NULL` de `fnEsProductoControlado`, fallo de persistencia con rollback).
5. **§4.4 Interfaz de consumo `IValidacionClienteService`**: buen desacople para pruebas unitarias en vez de HTTP directo.
6. **§7 Estrategia de pruebas**: buena cobertura funcional/unitaria/integración con casos críticos identificados explícitamente (incluye el caso borde de pedido 100% controlado que queda correctamente marcado como pendiente de confirmar).

---

## Acciones para Cristóbal (resumen accionable)

1. Confirmar con negocio/PM si Perú queda fuera de alcance de esta release; actualizar `R16A-RE-FU-009.md` en consecuencia o corregir el diseño (H-01).
2. Reconciliar la Historia de Jira R16A-79 con `R16A-RE-FU-009.md`: agregar la Regla 6 (bifurcación) y los criterios C1/C2/D/E si son válidos, o retirarlos del diseño si no lo son (H-02).
3. Agregar el script `ALTER TABLE ppPedido ADD AceptaEntregasParciales` a §5.3 (H-03).
4. Definir si `tpPedido.IdPedidoOrigenControlado` referencia a `tpPedido` o a `ppPedido`, y agregar el constraint FK explícito (H-04).
5. Unificar el esquema de `catUsoArchivoSistema` (`Clave`/`Descripcion` vs. `UsoArchivoSistema`) con la BD real (H-05).
6. Reemplazar el Script 2 por el ALTER sobre la cadena `MarcaFamilia → Familia → catControl` ya confirmada en `_BD.md` (H-06).
7. Incorporar el análisis de puntos de entrada alternos a Tramitar de `-Back.md` §3.2 (H-07).
8. Renumerar criterios de aceptación 1:1 con el requisito una vez resuelto H-02 (H-08).

---

## Referencias

- Requisito: `R16A-RE-FU-009.md`
- Impacto Backend: `R16A-RE-FU-009-Back.md`
- Diccionario de datos: `R16A-RE-FU-009_BD.md`
- Diseño revisado: `R16A-RE-FU-009-DIS-SOL-Back.pdf`
- Observaciones citadas en el diseño: OBS-007 (Perú fuera de scope), OBS-023 (validación movida a L05), GAP-RE-FU-003, GAP-fnControlado, GAP-catUsoArchivoSistema
