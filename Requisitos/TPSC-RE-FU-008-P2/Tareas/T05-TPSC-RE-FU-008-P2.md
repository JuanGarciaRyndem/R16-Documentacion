# [ TPSC-RE-FU-008 ] [LIST-PAG-MULT-FILTER] Crear BuzonCobrosBO — lista paginada del Buzón de Cobros

---

## Aplicativos

- ProquifaDotNet

## Módulos

- Buzones

## Consideraciones previas

- Tarea T01 ejecutada (clasificación `cobro` existente en catálogo).
- Revisar implementación de Buzones preexistentes (Buzón de Requisiciones, Buzón de Pedidos) para seguir el mismo patrón de consulta, filtros y paginación.
- La visibilidad se filtra por `ClienteCartera.IdUsuarioCobrador` del Gestor autenticado.
- La segregación por región se aplica vía `CorreoRecibido.IdRegion`.

## Objetivo general

Crear la lógica de negocio que retorna la lista paginada del Buzón de Cobros, filtrada por el Gestor de Cobranza autenticado y la región activa.

## Objetivos específicos

1. Crear `Logic.Pqf.Catalogos\Buzones\Cobros\BuzonCobrosBO.cs` con método de consulta paginada.
2. Crear DTO `Logic.Pqf.Catalogos\Buzones\Cobros\Models\BuzonCobrosDetalle.cs` con campos: IdCorreoRecibido, Asunto, CorreoEmisor, FechaRecepcion, Cliente, Region, FolioCobro, TotalMailBot, Leido, Procesado.
3. Implementar el filtro de visibilidad:
   ```
   CorreoRecibidoCliente.IdCliente
     → ClienteCarteraCliente.IdCliente
     → ClienteCartera.IdUsuarioCobrador == Usuario autenticado
   ```
4. Filtrar por `catClasificacionCorreoRecibido.Clave = 'cobro'` (o `'pago'` según decisión T01).
5. Soportar mismos filtros que Buzones existentes (búsqueda por texto, fecha, estado de lectura).

## Resultado esperado

- El método retorna lista paginada de correos clasificados como cobro, visibles únicamente para el Gestor asignado al cliente, segregados por región.

## Entregables

- `BuzonCobrosBO.cs`
- `Models\BuzonCobrosDetalle.cs`

## Criterios de aceptación

- [ ] La consulta retorna solo correos con clasificación `cobro` y `Activo = 1`.
- [ ] La visibilidad está filtrada por `IdUsuarioCobrador` del Gestor autenticado.
- [ ] Soporta paginación (página, tamaño de página).
- [ ] Soporta filtros múltiples (texto, fecha, leído/no leído).
- [ ] La segregación por región funciona correctamente (MEX no ve PER y viceversa).
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.2
- Query de referencia: `TPSC-RE-FU-008_BD.md` — sección "Bandeja del Buzón de Cobros por Gestor"
- Reglas de negocio: Regla 5 (visibilidad por cobrador) y Regla 10 (filtros equivalentes)

## Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Catalogos\Logic.Pqf.Catalogos.csproj`
