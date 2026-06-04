# [ TPSC-RE-FU-008 ] [SERV-SIMPLE-PUT] Crear BuzonCobrosBO.Reclasificar — reclasificación manual de correo

---

## Aplicativos

- ProquifaDotNet

## Módulos

- Buzones

## Consideraciones previas

- Tarea T05 completada (BuzonCobrosBO existe).
- La reclasificación es un UPDATE de `CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido` al buzón destino.
- No existe opción "marcar como no-cobro"; el Gestor elige el buzón destino (incluido Otros).
- Si se reclasifica, el pendiente `fccFolioPagoCliente` asociado debe desactivarse (`Activo = 0`).

## Objetivo general

Implementar la lógica de reclasificación manual que permite al Gestor de Cobranza mover un correo del Buzón de Cobros a otro buzón del sistema.

## Objetivos específicos

1. Crear `Logic.Pqf.Catalogos\Buzones\Cobros\BuzonCobrosBO.Reclasificar.cs`.
2. Crear modelo de entrada `Logic.Pqf.Catalogos\Buzones\Cobros\Models\GMReclasificarCorreo.cs` con: `IdCorreoRecibidoCliente`, `IdCatClasificacionDestino`.
3. Implementar lógica:
   - UPDATE `CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido` = clasificación destino.
   - UPDATE `CorreoRecibidoCliente.FechaUltimaActualizacion` = GETDATE().
   - Si existía `fccFolioPagoCliente` asociado, UPDATE `Activo = 0`.
4. Validar que el `IdCatClasificacionDestino` existe y está activo en `catClasificacionCorreoRecibido`.

## Resultado esperado

- El correo se retira del Buzón de Cobros y aparece en el buzón destino seleccionado.
- El pendiente en Validar Cobro se desactiva si existía.

## Entregables

- `BuzonCobrosBO.Reclasificar.cs`
- `Models\GMReclasificarCorreo.cs`

## Criterios de aceptación

- [ ] El correo desaparece del Buzón de Cobros tras la reclasificación.
- [ ] El correo aparece en el buzón destino correspondiente.
- [ ] Si había un `fccFolioPagoCliente` asociado, queda con `Activo = 0`.
- [ ] No se permite reclasificar a una clasificación inexistente o inactiva.
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.2
- Query: `TPSC-RE-FU-008_BD.md` — sección "Reclasificar correo a otro buzón"
- Regla de negocio: Regla 8 (Reclasificación manual hacia otro buzón)

## Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Catalogos\Logic.Pqf.Catalogos.csproj`
