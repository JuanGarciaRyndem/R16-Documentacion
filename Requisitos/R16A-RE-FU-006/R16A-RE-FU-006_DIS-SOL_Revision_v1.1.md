# Revisión — [R16A-RE-FU-006][DIS-SOL] Diseño de la solución (v1.1)

| Campo | Valor |
|---|---|
| **Documento revisado** | `[R16A-RE-FU-006][DIS-SOL] Diseño de la solución.pdf` v1.1 (1 jul 2026) |
| **Documento base de esta ronda** | Revisión v1.0 (`R16A-RE-FU-006_DIS-SOL_Revision.md`, 24-jun-2026) |
| **Autor del diseño** | Jose Armando Santiago Lorenzo |
| **Revisor** | Juan David García Cruz |
| **Fecha de revisión** | 2 jul 2026 |
| **Versión del requisito base** | R16A-RE-FU-006 (post OBS-013/014/015) |

---

## Resumen

La v1.1 corrige **6 de los 10 hallazgos** señalados en la ronda anterior — en particular los tres bloqueantes relacionados con la persistencia de la referencia vigente (H-01, H-02, H-03) y la fuente de datos para el segmento S4 (H-07), que están muy bien resueltos y documentados con evidencia de verificación en vivo. Sin embargo, **queda un hallazgo bloqueante sin atender (H-04)** y tres brechas menores sin cerrar (H-08 parcial, H-09, H-10). Adicionalmente, al comparar contra `R16A-RE-FU-006_BD.md` aparecieron **inconsistencias nuevas** entre ambos documentos que deben alinearse antes de construcción.

No recomiendo pasar el diseño a desarrollo hasta cerrar H-04 — es el mismo tipo de hueco (persistencia de nivel 1 incompleta) que motivó la ronda de revisión anterior, ahora referido a la regeneración de la referencia, no a su creación inicial.

---

## Correcciones confirmadas (v1.0 → v1.1)

| # | Hallazgo | Estado |
|---|---|---|
| H-01 | Falta persistir `ReferenciaVigente` en `ClienteDatosBancarios` | ✅ Cerrado — columnas `ReferenciaVigente varchar(200)` y `FechaReferenciaVigente datetime` agregadas al DDL |
| H-02 | Flujo 1 no arma la referencia al CREATE/UPDATE | ✅ Cerrado — paso 4/5 del Flujo 1 invoca `ReferenciaBancariaBO.Construir` y persiste el resultado |
| H-03 | Flujo 2 reconstruye la referencia en vez de leer la vigente | ✅ Cerrado — RT-04 confirma que el factory solo lee `ReferenciaVigente`, ya no invoca `ReferenciaBancariaBO` |
| H-05 | Selector de cuentas usa `DatosBancarios` en vez de `EmpresaDatosBancarios` | ✅ Cerrado — CA-2/CA-3 y endpoints usan `vEmpresaDatosBancarios` |
| H-06 | Constraint `UNIQUE` plano impide reasignar cuenta tras baja lógica | ✅ Cerrado — índice único filtrado `WHERE Activo = 1` |
| H-07 | Segmento S4 apoyado en `Carga_ClientesR1` (fuente frágil) | ✅ Cerrado — fuente cambiada a `PConnectProquifaDotNet.dbo.Clientes` vía SP (`spObtenerClienteLegacyId`), patrón parámetro, validado en vivo (99.9% cobertura, 1,398/1,400 clientes) |

---

## Hallazgo bloqueante — sigue abierto

### H-04 — Falta lógica de regeneración por cambio en `Cliente.Nombre` o `Cliente.Clave`

- **Estado:** No atendido. No hay ninguna mención de regeneración, hook, trigger ni job batch en todo el documento v1.1.
- **Lo que exige el requisito (Regla 4 nivel 1, Criterio B2):** *"la referencia... solo se regenera si cambia un dato fuente (banco, cuenta, Código Validador o **datos del cliente que la componen** — Nombre y Clave)"*.
- **Por qué sigue siendo bloqueante:** los segmentos S1-S3 dependen de `Cliente.Nombre` y S4 de la clave legacy del cliente. Si el nombre o la clave cambian después de que la cuenta fue asignada, `ReferenciaVigente` queda obsoleta y la próxima proforma casará una referencia incorrecta — exactamente el escenario de inconsistencia que la Regla 4 busca evitar.
- **Nota:** `R16A-RE-FU-006_BD.md` ya documenta esto como Gap #9 (prioridad Alta) y como nota bajo la tabla `ClienteDatosBancarios`, con las mismas 3 opciones de la ronda anterior (hook en `ClienteBO`, trigger BD, regeneración lazy). El diseño técnico (DIS-SOL) es el documento que falta actualizar para reflejarlo.
- **Acción:** Documentar el mecanismo elegido (recomendación: hook en `ClienteBO._GuardarOActualizar`, consistente con el patrón del ecosistema descrito en la sección GAP-E) como un Flujo 3 o sub-flujo, incluyendo diagrama, reglas técnicas y casos de prueba.

---

## Brechas no bloqueantes — sin cerrar

### H-09 — Numeración de Criterios de Aceptación sigue sin mapear con el requisito

- El requisito usa A1-A4, B1-B5, C1-C3. El diseño v1.1 conserva la nomenclatura anterior: CA-1…CA-8, CA-E1, CA-E2, CA-EC1, CA-EC2, CA-12.
- **Acción:** Renumerar 1:1 con los IDs del requisito para trazabilidad directa.

### H-10 — Nomenclatura "PQF2" sigue en uso

- El término aparece de forma extensiva (secciones de Componentes involucrados, Algoritmo Banamex, Migración de datos, Impacto Técnico). La directriz del proyecto pide referirse al sistema siempre como **ProquifaDotNet**.
- **Acción:** Reemplazar "PQF2" por "ProquifaDotNet" en todo el documento.

### H-08 — Diccionario de datos incompleto (parcial)

- Se agregó el índice único filtrado (H-06), pero falta replicar en el DDL del PDF el índice no-único `IX_ClienteDatosBancarios (IdCliente, IdEmpresaDatosBancarios, Activo)` que ya está documentado en `_BD.md`.
- **Acción:** Agregar el índice faltante al script del PDF.

---

## Inconsistencias nuevas encontradas (PDF v1.1 vs. `R16A-RE-FU-006_BD.md`)

Estas no estaban en la revisión anterior porque `_BD.md` no se comparó campo a campo contra el DDL del PDF en esa ronda.

### H-11 — Nombre de la columna FK a `EmpresaDatosBancarios`

- El PDF v1.1 corrigió el diseño para usar `IdEmpresaDatosBancarios` como FK (correcto, resuelve H-05), pero `_BD.md` todavía documenta la columna como `IdDatosBancarios` con FK a `dbo.DatosBancarios`.
- **Acción:** Actualizar `_BD.md` sección 1 para que la columna, la FK y las consultas SQL de ejemplo usen `IdEmpresaDatosBancarios`.

### H-12 — Estrategia de generación del PK difiere entre documentos

- PDF v1.1: `IdClienteDatosBancarios` usa `DEFAULT (NEWSEQUENTIALID())`.
- `_BD.md`: usa `DEFAULT (NEWID())`.
- **Acción:** Confirmar cuál es la estrategia definitiva (NEWSEQUENTIALID reduce fragmentación en PK clustered) y alinear ambos documentos.

### H-13 — Nulabilidad de `CodigoValidador` inconsistente

- PDF v1.1: `CodigoValidador varchar(50) NULL` (la validación de "no vacío" vive solo en `ClienteDatosBancariosBO`, CA-E1).
- `_BD.md`: `CodigoValidador varchar(50) NOT NULL`.
- **Riesgo:** si la validación del BO tiene algún hueco (por ejemplo, un proceso batch o la migración SSIS que inserte directo a BD), un `NULL` a nivel de columna no lo bloquearía.
- **Acción:** Decidir si el constraint debe ser `NOT NULL` a nivel BD como defensa adicional, y alinear ambos documentos con la misma decisión.

### H-14 — Formato: guión normal en vez de em dash en encabezado

- El encabezado repetido en cada página ("Diseño de la solución **-** Requisito R16A-RE-FU-006") usa guión normal `-`, no `—` como pide la directriz de estilo del proyecto para separar secciones en títulos.
- **Acción:** Corregir el guión en la plantilla/encabezado del documento.

---

## Puntos que siguen bien y deben conservarse

1. Persistencia en dos niveles (`ReferenciaVigente` + snapshot en `ReferenciaPago`) ahora completa y coherente con Regla 4/5 y OBS-013.
2. Sección GAP-E (bug `cliente.Clave`): diagnóstico, evidencia en vivo y fix con patrón parámetro — nivel de detalle ejemplar, replicable para H-04.
3. Migración SSIS (Flujo A/B): mapeo de campos, caveats de collation y manejo de errores por fila bien definidos.
4. Diagramas mermaid de Flujo 1 y Flujo 2 actualizados y consistentes con el nuevo diseño.
5. Casos de prueba unitarios para `ReferenciaBancariaBO` con `clienteLegacy` (tabla de casos S4) — buena cobertura de edge cases.

---

## Acciones para José Armando (resumen accionable)

1. **Bloqueante:** Documentar el mecanismo de regeneración de `ReferenciaVigente` ante cambio de `Cliente.Nombre`/clave legacy (H-04) — diagrama, reglas técnicas, manejo de errores y pruebas.
2. Renumerar criterios de aceptación 1:1 con el requisito (H-09).
3. Reemplazar "PQF2" por "ProquifaDotNet" en todo el documento (H-10).
4. Agregar índice `IX_ClienteDatosBancarios (IdCliente, IdEmpresaDatosBancarios, Activo)` al DDL (H-08).
5. Actualizar `_BD.md` para usar `IdEmpresaDatosBancarios` como FK (H-11).
6. Confirmar y alinear estrategia de PK (`NEWSEQUENTIALID` vs `NEWID`) entre PDF y `_BD.md` (H-12).
7. Confirmar y alinear nulabilidad de `CodigoValidador` a nivel BD entre PDF y `_BD.md` (H-13).
8. Corregir guión por em dash en el encabezado del documento (H-14).

---

## Referencias

- Requisito actualizado: `R16A-RE-FU-006.md`
- Diccionario de datos: `R16A-RE-FU-006_BD.md`
- Impacto Backend: `R16A-RE-FU-006-Back.md`
- Revisión anterior (v1.0): `R16A-RE-FU-006_DIS-SOL_Revision.md`
- Observaciones aplicadas: OBS-010 (migración Legacy), OBS-013 (persistencia en 2 niveles), OBS-014 (historial CV de 1 nivel), OBS-015 (PDF en firme = Legacy/Drobo).
