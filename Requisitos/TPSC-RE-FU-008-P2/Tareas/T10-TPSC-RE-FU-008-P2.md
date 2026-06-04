# [ TPSC-RE-FU-008 ] [SERV-SIMPLE-PUT] Crear endpoint PUT /api/buzon/cobros/{id}/reclasificar

---

## Aplicativos

- ProquifaDotNet

## Módulos

- Buzones

## Consideraciones previas

- Tarea T06 completada (BuzonCobrosBO.Reclasificar con lógica de negocio).
- El endpoint requiere autenticación y validar que el correo pertenece al Gestor autenticado.

## Objetivo general

Exponer un endpoint PUT que permita al Gestor de Cobranza reclasificar un correo del Buzón de Cobros moviéndolo a otro buzón del sistema.

## Objetivos específicos

1. Crear endpoint en `WebApi.Catalogos` con ruta `PUT /api/buzon/cobros/{id}/reclasificar`.
2. Body: `{ "IdCatClasificacionDestino": "guid" }`.
3. Validar que el `id` corresponde a un correo visible para el Gestor autenticado.
4. Invocar `BuzonCobrosBO.Reclasificar`.
5. Retornar 200 OK o 400/404 según corresponda.

## Resultado esperado

- El Gestor puede reclasificar un correo desde el frontend del Buzón de Cobros.

## Entregables

- Endpoint PUT en `WebApi.Catalogos`.

## Criterios de aceptación

- [ ] Responde 200 OK al reclasificar exitosamente.
- [ ] Responde 404 si el correo no existe o no pertenece al Gestor.
- [ ] Responde 400 si la clasificación destino no es válida.
- [ ] Requiere autenticación.
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.3
- Criterio de aceptación del requisito: D1 (Acción de reclasificación manual)

## Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `WebApi.Catalogos\WebApi.Catalogos.csproj`
