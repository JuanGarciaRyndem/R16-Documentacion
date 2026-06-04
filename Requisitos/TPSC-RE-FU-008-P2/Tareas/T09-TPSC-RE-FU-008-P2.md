# [ TPSC-RE-FU-008 ] [LIST-PAG-MULT-FILTER] Crear endpoint GET /api/buzon/cobros — lista paginada + filtros

---

## Aplicativos

- ProquifaDotNet

## Módulos

- Buzones

## Consideraciones previas

- Tarea T05 completada (BuzonCobrosBO con lógica de consulta paginada).
- Seguir el patrón de controladores/endpoints de Buzones existentes en `WebApi.Catalogos`.
- El endpoint requiere autenticación; el `IdUsuario` se obtiene del token JWT.

## Objetivo general

Exponer un endpoint GET que retorne la lista paginada del Buzón de Cobros filtrada por el Gestor de Cobranza autenticado y la región activa.

## Objetivos específicos

1. Crear controlador o endpoint en `WebApi.Catalogos` con ruta `GET /api/buzon/cobros`.
2. Parámetros de entrada: página, tamaño de página, filtros opcionales (texto búsqueda, fecha desde/hasta, leído/no leído).
3. Invocar `BuzonCobrosBO` pasando el `IdUsuario` del token y la región del contexto.
4. Retornar respuesta paginada con formato consistente con otros Buzones.

## Resultado esperado

- El frontend puede consumir `GET /api/buzon/cobros` y obtener la bandeja del Gestor paginada y filtrada.

## Entregables

- Controlador/Endpoint en `WebApi.Catalogos`.

## Criterios de aceptación

- [ ] El endpoint responde 200 con lista paginada en formato consistente con Buzones existentes.
- [ ] Requiere autenticación (401 si no hay token).
- [ ] Filtra por gestor autenticado (no muestra correos de otros gestores).
- [ ] Soporta paginación y filtros múltiples.
- [ ] El proyecto compila y el endpoint es accesible vía Swagger/Postman.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.3
- Criterio de aceptación del requisito: B3 (visibilidad filtrada por cobrador) y D3 (filtros, búsqueda, paginación)

## Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `WebApi.Catalogos\WebApi.Catalogos.csproj`
