# [ TPSC-RE-FU-008 ] [BD-OBJ-CH] Evaluar y actualizar spActualizarBuzonPagoLegacy

---

## Aplicativos

- ProquifaDotNet (Base de Datos — SQL Server RYNL010)

## Módulos

- Buzones
- Cobros

## Consideraciones previas

- Tarea T01 debe estar definida (decisión sobre renombrar `pago` → `cobro` o nuevo registro).
- Si se optó por Opción A (renombrar), este SP puede quedar inconsistente.
- Si se optó por Opción B (nuevo registro), el SP podría no necesitar cambios.

## Objetivo general

Evaluar el stored procedure `spActualizarBuzonPagoLegacy` para determinar si referencia la clave `pago` directamente y, de ser necesario, actualizarlo para que sea compatible con los cambios de R16.

## Objetivos específicos

1. Revisar el código fuente del SP `spActualizarBuzonPagoLegacy`.
2. Identificar si usa la clave `pago` como filtro literal.
3. Si aplica, actualizar la referencia a la clave vigente (`cobro` si Opción A, o mantener si Opción B).
4. Validar que el SP sigue funcionando correctamente tras el cambio.

## Resultado esperado

- El SP es compatible con la nueva clasificación de cobros sin romper funcionalidad existente.

## Entregables

- Informe de evaluación (si no requiere cambios, documentar la conclusión).
- Script de ALTER PROCEDURE (si requiere cambios).
- Script de rollback.

## Criterios de aceptación

- [ ] Se documenta la evaluación del SP (usa o no usa la clave `pago`).
- [ ] Si se modifica, el SP se ejecuta sin errores y produce resultados correctos.
- [ ] No se rompe funcionalidad legacy existente.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.4

## Recursos

- Servidor: RYNL010
- Base de datos: ProquifaDotNet
