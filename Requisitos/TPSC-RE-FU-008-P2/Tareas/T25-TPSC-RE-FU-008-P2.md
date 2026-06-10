# [ TPSC-RE-FU-008 ] [LIST-PAG-MULT-FILTER] Crear BuzonCotizacionesBO — lista paginada del Buzón de Cotizaciones

---

## Aplicativos

- ProquifaDotNet

## Módulos

- Buzones
- Mailbot

## Consideraciones previas

- Tarea T01 ejecutada (clasificación `requisicion` existente en catálogo).
- Revisar implementación de `BuzonCobrosBO` (T05) y de los Buzones preexistentes (Buzón de Requisiciones, Buzón de Pedidos) para seguir el mismo patrón de consulta, filtros y paginación.
- La clasificación de los correos en este buzón corresponde a `catClasificacionCorreoRecibido.Clave = 'requisicion'` (correos procesados por el Agente IA como cotización/requisición).
- La visibilidad se filtra por el Gestor Comercial autenticado y por `CorreoRecibido.IdRegion`.
- Los correos MEX no deben aparecer en resultados para usuarios PER y viceversa.

## Objetivo general

Crear la lógica de negocio que retorna la lista paginada del Buzón de Cotizaciones, filtrada por el Gestor Comercial autenticado y la región activa.

## Objetivos específicos

1. Crear `Logic.Pqf.Catalogos\Buzones\Cotizaciones\BuzonCotizacionesBO.cs` con método de consulta paginada.
2. Crear DTO `Logic.Pqf.Catalogos\Buzones\Cotizaciones\Models\BuzonCotizacionesDetalle.cs` con campos: IdCorreoRecibido, Asunto, CorreoEmisor, FechaRecepcion, Cliente, Region, EstadoCotizacion, Leido, Procesado.
3. Implementar filtro por `catClasificacionCorreoRecibido.Clave = 'requisicion'`.
4. Filtrar por Gestor Comercial autenticado (`IdUsuario`) y por `CorreoRecibido.IdRegion`.
5. Soportar los mismos filtros que Buzones existentes (búsqueda por texto, fecha, estado de lectura).

## Resultado esperado

- El método retorna lista paginada de correos clasificados como cotización, visibles únicamente para el Gestor Comercial autenticado y segregados por región.

## Entregables

- `Logic.Pqf.Catalogos\Buzones\Cotizaciones\BuzonCotizacionesBO.cs`
- `Logic.Pqf.Catalogos\Buzones\Cotizaciones\Models\BuzonCotizacionesDetalle.cs`

## Criterios de aceptación

- [ ] La consulta retorna solo correos con clasificación `Clave = 'requisicion'` y `Activo = 1`.
- [ ] La visibilidad está filtrada por el Gestor Comercial autenticado.
- [ ] Soporta paginación (página, tamaño de página).
- [ ] Soporta filtros múltiples (texto, fecha, leído/no leído).
- [ ] La segregación por región funciona correctamente (MEX no ve PER y viceversa).
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.2
- Patrón de referencia: `Logic.Pqf.Catalogos\Buzones\Cobros\BuzonCobrosBO.cs` (T05)

## Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Catalogos\Logic.Pqf.Catalogos.csproj`
