# Tareas — TPSC-RE-FU-002 Mantenimiento de Catálogo de Clientes (Campo Cobrador)

| Campo | Valor |
|---|---|
| **Requisito** | TPSC-RE-FU-002 |
| **Nombre** | Mantenimiento de Catálogo de Clientes |
| **Total de tareas** | 6 |

---

## Tarea 1

### [ TPSC-RE-FU-002 ] [ IMP-EXIST-SERVICE ] Agregar método ObtenerGestoresDeCobranzaActivos() en UsuariosBO

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el método `ObtenerGestoresDeCobranzaActivos()` en `UsuariosBO.Extensions.cs` para devolver los usuarios activos con rol Gestor de Cobranza (`AnalistaDeCuentasPorCobrar = 1`), que serán la fuente del selector del campo Cobrador en el Catálogo de Clientes. Cumple Regla 2 y Criterio B1.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-002 (GAP-01)
- Confirmar con el equipo que `AnalistaDeCuentasPorCobrar = 1` es el mapeo correcto del rol Gestor de Cobranza (Pendiente P1)
- Implementar el método en `Logic.Pqf.Catalogos\Usuarios\UsuariosBO.Extensions.cs`
- Filtrar `AnalistaDeCuentasPorCobrar = true AND Activo = true`
- Ordenar resultado por `NombreCompleto`
- Ejecutar pruebas unitarias

**Resultado esperado:**
El método retorna únicamente usuarios activos con rol Gestor de Cobranza, ordenados por nombre, listos para poblar el selector del campo Cobrador.

**Entregables:**
- `UsuariosBO.Extensions.cs` con el nuevo método
- Pruebas unitarias del método

**Criterios de aceptación:**
- [ ] El método retorna solo usuarios con `AnalistaDeCuentasPorCobrar = true AND Activo = true`
- [ ] Usuarios inactivos o con otros roles no aparecen en el resultado
- [ ] Confirmado el mapeo de `AnalistaDeCuentasPorCobrar` con el equipo (P1)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Usuarios\UsuariosBO.Extensions.cs`
- GAP documentado: GAP-01 del archivo `TPSC-RE-FU-002-Back.md`
- Pendiente P1: confirmar mapeo exacto del rol antes de implementar

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002.md`

---

## Tarea 2

### [ TPSC-RE-FU-002 ] [ IMP-EXIST-SERVICE ] Agregar método ObtenerIdUsuarioCobradorPorCliente() en ClienteCarteraClienteBO

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el método `ObtenerIdUsuarioCobradorPorCliente(Guid idCliente)` en `ClienteCarteraClienteBO.Extensions.cs` reutilizando el patrón ya existente de `ObtenerIDUsuarioESACPorCliente()`, para que el Catálogo de Clientes pueda mostrar el Cobrador actualmente asignado al cliente. Cumple Criterio A1.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-002 (GAP-02)
- Revisar el método existente `ObtenerIDUsuarioESACPorCliente()` como referencia del patrón
- Implementar `ObtenerIdUsuarioCobradorPorCliente(Guid idCliente)` navegando `ClienteCarteraCliente → ClienteCartera → IdUsuarioCobrador`
- Retornar `null` si el cliente no tiene cartera activa o no tiene cobrador asignado
- Ejecutar pruebas unitarias con cliente con cobrador, sin cobrador y sin cartera

**Resultado esperado:**
El método retorna el `IdUsuarioCobrador` de la cartera activa del cliente, o `null` si no tiene asignación. El Catálogo de Clientes puede mostrar el campo Cobrador correctamente.

**Entregables:**
- `ClienteCarteraClienteBO.Extensions.cs` con el nuevo método
- Pruebas unitarias del método

**Criterios de aceptación:**
- [ ] El método retorna el `IdUsuarioCobrador` de la cartera activa del cliente
- [ ] Retorna `null` si el cliente no tiene cartera activa o no tiene cobrador asignado
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraClienteBO.Extensions.cs`
- GAP documentado: GAP-02 del archivo `TPSC-RE-FU-002-Back.md`
- Patrón de referencia: `ObtenerIDUsuarioESACPorCliente()` en el mismo archivo

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002.md`

---

## Tarea 3

### [ TPSC-RE-FU-002 ] [ SERV-TRANSACT ] Agregar método ReasignarCobrador() en ClienteCarteraBO

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el método `ReasignarCobrador(Guid idCliente, Guid idNuevoCobrador)` en `ClienteCarteraBO.cs` que encapsule la regla de negocio de reasignación de Cobrador: validar que el nuevo cobrador es un Gestor de Cobranza activo, localizar la cartera activa del cliente y actualizar `IdUsuarioCobrador`. Cumple Regla 3, Regla 4 y Criterios B2 / B3 / C2.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-002 (GAP-03)
- Implementar validación: el nuevo cobrador debe tener `AnalistaDeCuentasPorCobrar = true AND Activo = true`
- Obtener la cartera activa del cliente mediante JOIN `ClienteCarteraCliente → ClienteCartera`
- Actualizar `ClienteCartera.IdUsuarioCobrador` y `FechaUltimaActualizacion`
- Lanzar excepción descriptiva si el cobrador no es válido o el cliente no tiene cartera activa
- Ejecutar pruebas unitarias: reasignación válida, cobrador inválido, cliente sin cartera

**Resultado esperado:**
El método actualiza `IdUsuarioCobrador` en la cartera activa del cliente. Tras la operación, la bandeja del nuevo Cobrador incluye al cliente y la del anterior ya no lo incluye en la siguiente consulta.

**Entregables:**
- `ClienteCarteraBO.cs` con el nuevo método `ReasignarCobrador`
- Pruebas unitarias del método

**Criterios de aceptación:**
- [ ] El método actualiza `ClienteCartera.IdUsuarioCobrador` y `FechaUltimaActualizacion`
- [ ] Lanza excepción si el nuevo cobrador no es Gestor de Cobranza activo
- [ ] Lanza excepción si el cliente no tiene cartera activa
- [ ] La bandeja del nuevo Cobrador incluye al cliente tras la reasignación
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraBO.cs`
- GAP documentado: GAP-03 del archivo `TPSC-RE-FU-002-Back.md`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002.md`

---

## Tarea 4

### [ TPSC-RE-FU-002 ] [ LIST-NO-FILTER ] Agregar endpoint GET /Usuario/GestoresDeCobranza

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — WebApi.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el endpoint `GET /Usuario/GestoresDeCobranza` en `UsuarioController.cs` para exponer la lista de Gestores de Cobranza activos al frontend, que lo usará para poblar el selector del campo Cobrador en el Catálogo de Clientes. Cumple Criterio B1.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-002 (GAP-01)
- Agregar el endpoint en `WebApi.Catalogos\Controllers\Sistema\Usuarios\UsuarioController.cs`
- Consumir el método `ObtenerGestoresDeCobranzaActivos()` implementado en la Tarea 1
- Documentar el endpoint en Swagger
- Verificar que el endpoint es accesible desde el Catálogo de Clientes

**Resultado esperado:**
El endpoint retorna la lista de usuarios activos con rol Gestor de Cobranza. El frontend puede poblar el selector del campo Cobrador correctamente.

**Entregables:**
- `UsuarioController.cs` con el nuevo endpoint
- Documentación Swagger del endpoint

**Criterios de aceptación:**
- [ ] `GET /Usuario/GestoresDeCobranza` retorna usuarios con `AnalistaDeCuentasPorCobrar = true AND Activo = true`
- [ ] El endpoint está documentado en Swagger
- [ ] Esta tarea depende de la Tarea 1 (método BO)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `WebApi.Catalogos\Controllers\Sistema\Usuarios\UsuarioController.cs`
- GAP documentado: GAP-01 del archivo `TPSC-RE-FU-002-Back.md`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002.md`

---

## Tarea 5

### [ TPSC-RE-FU-002 ] [ SERV-SIMPLE-PUT ] Agregar endpoint PUT /ClienteCartera/ReasignarCobrador

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — WebApi.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el endpoint `PUT /ClienteCartera/ReasignarCobrador` en `ClienteCarteraController.cs` para exponer la operación de reasignación de Cobrador al frontend. El endpoint debe validar que el usuario solicitante tiene rol Coordinador de Tesorería (`GerenteDeTesoreria = true`) antes de ejecutar la operación. Cumple Criterios A2, B2, B3 y Regla 1.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-002 (GAP-03 y GAP-04)
- Agregar el endpoint en `WebApi.Catalogos\Controllers\Configuracion\Clientes\Cartera\ClienteCarteraController.cs`
- Consumir el método `ReasignarCobrador()` implementado en la Tarea 3
- Confirmar con arquitectura la estrategia de validación de rol (Pendiente P2): ¿en capa BO o filtro de autorización en controller?
- Documentar el endpoint en Swagger indicando que solo Coordinador de Tesorería puede invocarlo

**Resultado esperado:**
El endpoint permite al Coordinador de Tesorería reasignar el Cobrador de un cliente. Roles distintos reciben respuesta de error de autorización.

**Entregables:**
- `ClienteCarteraController.cs` con el nuevo endpoint
- Documentación Swagger del endpoint
- Decisión de arquitectura sobre validación de rol documentada (P2)

**Criterios de aceptación:**
- [ ] `PUT /ClienteCartera/ReasignarCobrador` actualiza `IdUsuarioCobrador` en la cartera activa del cliente
- [ ] El endpoint valida que el solicitante tiene `GerenteDeTesoreria = true`
- [ ] Roles distintos al Coordinador de Tesorería reciben error de autorización
- [ ] Decisión de arquitectura P2 registrada y aprobada
- [ ] Esta tarea depende de la Tarea 3 (método BO)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `WebApi.Catalogos\Controllers\Configuracion\Clientes\Cartera\ClienteCarteraController.cs`
- GAPs documentados: GAP-03 y GAP-04 del archivo `TPSC-RE-FU-002-Back.md`
- Pendiente P2: definir estrategia de validación de rol antes de implementar

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002.md`

---

## Tarea 6

### [ TPSC-RE-FU-002 ] [ BD-OBJ-CH ] Actualizar SP spDesactivarCarterasCliente para incluir IdUsuarioCobrador en el retorno

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Base de Datos ProquifaDotNet

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Actualizar el SP `spDesactivarCarterasCliente` para que incluya `IdUsuarioCobrador` en el `SELECT` de retorno, habilitando la trazabilidad del cobrador al momento de desactivar carteras de un cliente. Cumple Pendiente P3 del requisito.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-002 (Pendiente P3)
- Revisar la definición actual del SP `spDesactivarCarterasCliente` en la BD `ProquifaDotNet`
- Agregar la columna `IdUsuarioCobrador` al `SELECT` de retorno del SP
- Verificar que el SP mantiene su lógica de transacción TRY/CATCH y ROLLBACK
- Ejecutar el SP en ambiente de desarrollo y validar que el retorno incluye `IdUsuarioCobrador`
- Registrar el script de modificación en el formulario de control de scripts del release

**Resultado esperado:**
El SP `spDesactivarCarterasCliente` retorna `IdUsuarioCobrador` en el SELECT de detalle de carteras afectadas, permitiendo auditoría del cobrador en operaciones de baja de carteras.

**Entregables:**
- Script `ALTER PROCEDURE spDesactivarCarterasCliente` con la columna `IdUsuarioCobrador` en el retorno
- Script registrado en el formulario de control de scripts del release

**Criterios de aceptación:**
- [ ] El SP retorna `IdUsuarioCobrador` en el SELECT de carteras desactivadas
- [ ] La lógica TRY/CATCH y ROLLBACK se mantiene intacta
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Objeto a modificar: `dbo.spDesactivarCarterasCliente` en BD `ProquifaDotNet` servidor `RYNL010`
- Pendiente documentado: P3 del archivo `TPSC-RE-FU-002-Back.md`
- El SP fue creado el 06/Febrero/2024 por Carlos Ivan Morales Carreon

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-002/TPSC-RE-FU-002.md`
