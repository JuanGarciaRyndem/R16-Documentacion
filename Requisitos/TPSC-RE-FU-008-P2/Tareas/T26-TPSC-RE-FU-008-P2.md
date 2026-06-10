# [ TPSC-RE-FU-008 ] [SERV-SIMPLE-PUT] Crear BuzonCotizacionesBO.Reclasificar — reclasificación manual de correo de cotización

---

## Aplicativos

- ProquifaDotNet

## Módulos

- Buzones
- Mailbot

## Consideraciones previas

- Tarea T25 completada (`BuzonCotizacionesBO` lista paginada disponible).
- La reclasificación permite al Gestor Comercial mover un correo clasificado como cotización a otro buzón (por ejemplo, si el Agente IA clasificó incorrectamente).
- Seguir el mismo patrón de `BuzonCobrosBO.Reclasificar` (T06).
- Solo el Gestor Comercial autenticado y propietario del buzón puede reclasificar el correo.
- El destino de reclasificación debe ser una `Clave` válida en `catClasificacionCorreoRecibido`.

## Objetivo general

Implementar la lógica de reclasificación manual que actualiza `CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido` al buzón destino seleccionado por el Gestor Comercial.

## Objetivos específicos

1. Crear `Logic.Pqf.Catalogos\Buzones\Cotizaciones\BuzonCotizacionesBO.Reclasificar.cs`.
2. Crear modelo de entrada `Logic.Pqf.Catalogos\Buzones\Cotizaciones\Models\GMReclasificarCorreoCotizacion.cs` con campos: IdCorreoRecibidoCliente, IdCatClasificacionCorreoRecibidoDestino.
3. Validar que el usuario autenticado es el Gestor asignado al correo.
4. Ejecutar `UPDATE CorreoRecibidoCliente SET IdCatClasificacionCorreoRecibido = @destino WHERE IdCorreoRecibidoCliente = @id`.
5. Registrar la reclasificación en bitácora si aplica el patrón de auditoría del proyecto.

## Resultado esperado

- Un correo del Buzón de Cotizaciones puede ser movido manualmente a otro buzón por el Gestor Comercial, actualizando la clasificación en BD.

## Entregables

- `Logic.Pqf.Catalogos\Buzones\Cotizaciones\BuzonCotizacionesBO.Reclasificar.cs`
- `Logic.Pqf.Catalogos\Buzones\Cotizaciones\Models\GMReclasificarCorreoCotizacion.cs`

## Criterios de aceptación

- [ ] Solo el Gestor autenticado y propietario del correo puede reclasificarlo.
- [ ] El destino de reclasificación es una clave válida en `catClasificacionCorreoRecibido`.
- [ ] `CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido` se actualiza correctamente.
- [ ] El correo desaparece del Buzón de Cotizaciones tras la reclasificación.
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.2
- Patrón de referencia: `Logic.Pqf.Catalogos\Buzones\Cobros\BuzonCobrosBO.Reclasificar.cs` (T06)

## Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Catalogos\Logic.Pqf.Catalogos.csproj`
