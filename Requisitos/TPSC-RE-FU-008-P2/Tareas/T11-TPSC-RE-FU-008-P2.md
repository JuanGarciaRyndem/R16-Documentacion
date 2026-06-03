# [ TPSC-RE-FU-008 ] [SERV-SIMPLE-PUT] Crear endpoint PUT /api/cobros/folio/{id}/cerrar

---

## Aplicativos

- ProquifaDotNet-R14

## Módulos

- Cobros
- Buzones

## Consideraciones previas

- Tarea T08 completada (FolioPagoClienteBO.Cierre con lógica de negocio).
- Este endpoint es invocado internamente desde el módulo Validar Cobro (al vincular cobro a proforma/factura o al marcar inconsistencia).

## Objetivo general

Exponer un endpoint PUT que cierre el pendiente del Buzón de Cobros, desactivando el registro `fccFolioPagoCliente`.

## Objetivos específicos

1. Crear endpoint en `WebApi.Catalogos` con ruta `PUT /api/cobros/folio/{id}/cerrar`.
2. Invocar `FolioPagoClienteBO.CerrarPendiente(id)`.
3. Retornar 200 OK o 404 si no existe.

## Resultado esperado

- Al vincular un cobro a proforma/factura o marcarlo como inconsistencia en Validar Cobro, el pendiente se cierra automáticamente y el correo desaparece del Buzón.

## Entregables

- Endpoint PUT en `WebApi.Catalogos`.

## Criterios de aceptación

- [ ] Responde 200 OK al cerrar exitosamente.
- [ ] Responde 404 si el folio no existe.
- [ ] El registro queda con `Activo = 0` tras la llamada.
- [ ] Requiere autenticación.
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.3
- Criterios de aceptación del requisito: C1 y C2

## Recursos

- Repositorio: ProquifaDotNet-R14, branch `develop-pack04`
- Proyecto: `WebApi.Catalogos\WebApi.Catalogos.csproj`
