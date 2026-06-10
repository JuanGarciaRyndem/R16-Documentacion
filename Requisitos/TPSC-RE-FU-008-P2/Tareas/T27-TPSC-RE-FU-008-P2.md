# [ TPSC-RE-FU-008 ] [LIST-PAG-MULT-FILTER] Crear endpoint GET /api/buzon/cotizaciones — lista paginada con filtros

---

## Aplicativos

- ProquifaDotNet

## Módulos

- WebApi.Catalogos
- Buzones

## Consideraciones previas

- Tarea T25 completada (`BuzonCotizacionesBO` disponible).
- Seguir el mismo patrón del endpoint `GET /api/buzon/cobros` (T09): misma estructura de respuesta, mismos filtros y paginación.
- El Gestor Comercial autenticado y la región se extraen del token — **no se aceptan como parámetros externos**.
- Aplicar autorización por rol de Gestor Comercial.

## Objetivo general

Exponer el endpoint `GET /api/buzon/cotizaciones` en `WebApi.Catalogos` que retorna la lista paginada del Buzón de Cotizaciones filtrada por el Gestor Comercial y región del token.

## Objetivos específicos

1. Crear controller o agregar acción en `WebApi.Catalogos\Controllers\Buzones\BuzonCotizacionesController.cs`.
2. Implementar `GET /api/buzon/cotizaciones` que acepta parámetros de paginación y filtros (texto, fecha, leído/no leído) vía query string.
3. Obtener `IdUsuario` e `IdRegion` del token mediante el patrón de autenticación existente.
4. Invocar `BuzonCotizacionesBO` y retornar el resultado paginado.
5. Aplicar atributo de autorización para rol de Gestor Comercial.

## Resultado esperado

- `GET /api/buzon/cotizaciones` retorna la lista paginada de correos de cotización visibles para el Gestor autenticado, segregados por región.

## Entregables

- `WebApi.Catalogos\Controllers\Buzones\BuzonCotizacionesController.cs`

## Criterios de aceptación

- [ ] `GET /api/buzon/cotizaciones` retorna `200 OK` con estructura paginada.
- [ ] El `IdUsuario` e `IdRegion` se extraen del token — no de parámetros externos.
- [ ] Soporta filtros múltiples (texto, fecha, leído/no leído) vía query string.
- [ ] Sin token válido retorna `401 Unauthorized`.
- [ ] El proyecto `WebApi.Catalogos` compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.3
- Patrón de referencia: endpoint `GET /api/buzon/cobros` (T09)

## Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `WebApi.Catalogos`
