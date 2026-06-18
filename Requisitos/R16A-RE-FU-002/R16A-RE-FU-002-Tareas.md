# Tareas — R16A-RE-FU-002 Catálogo de Clientes (Campo Cobrador)

| Campo | Valor |
|---|---|
| **Requisito** | R16A-RE-FU-002 |
| **Nombre** | Mantenimiento de Catálogo de Clientes |
| **Total de tareas** | 7 |
| **Revisión aplicada** | R16A-RE-FU-002-Revision.md |

### Cambios respecto a la versión anterior de Tareas

| #   | Cambio                                                                                                                                                                                                | Origen                                                   |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| 1   | Total de tareas: de 6 → **7** (nueva Tarea 7 por GAP-05)                                                                                                                                              | Revisión — exclusión de Cobrador en Cotizar lo Cotizable |
| 2   | Referencias de criterios actualizadas a secciones A/B/C definitivas del requisito                                                                                                                     | Revisión — criterios reorganizados                       |
| 3   | Tarea 3 (ReasignarCobrador): se agrega nota de **Regla 4 dinámica** — la redistribución de bandeja es automática por JOIN, sin migración de registros                                                 | Revisión — decisión sobre Criterio C2 documentada        |
| 4   | Tarea 5 (endpoint ReasignarCobrador): se agrega referencia a Criterio C2 en descripción y criterios de aceptación                                                                                     | Revisión — Criterio C2 redistribución inmediata          |
| 5   | **Tarea 7 nueva** [ R16A-RE-FU-002 ] [ SCOPE-VERIFY ] Verificar exclusión del campo Cobrador en alta de cliente desde Cotizar lo Cotizable (GAP-05)                                                   | Revisión — nuevo en Alcance `No aplica a`                |
| 6   | Cabecera completada con revisión aplicada y tabla de cambios                                                                                                                                          | Mejora de trazabilidad                                   |
| 7   | **Tarea 3 actualizada:** Regla 4 matizada — redistribución solo aplica a pendientes abiertos por pantalla/módulo; trabajo completado no se reasigna. Nuevo **Criterio C4** en criterios de aceptación | OBS-005                                                  |
| 8   | **Tarea 5 actualizada:** GAP-04 ampliado — validación de rol incluye Gerente de Tesorería además del Coordinador de Tesorería. Criterio C4 agregado. Referencias a OBS-003/005                        | OBS-003 / OBS-005                                        |

---

## Tarea 1

### [ R16A-RE-FU-002 ] [ IMP-EXIST-SERVICE ] Agregar método ObtenerGestoresDeCobranzaActivos() en UsuariosBO

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el método `ObtenerGestoresDeCobranzaActivos()` en `UsuariosBO.Extensions.cs` para devolver los usuarios activos con rol Gestor de Cobranza (`AnalistaDeCuentasPorCobrar = 1`), que serán la fuente del selector del campo Cobrador en el Catálogo de Clientes. Cumple **Regla 2** y **Criterio B1**.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito R16A-RE-FU-002 (GAP-01)
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
- [ ] Confirmado el mapeo de `AnalistaDeCuentasPorCobrar` con el equipo (Pendiente P1)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Usuarios\UsuariosBO.Extensions.cs`
- GAP documentado: GAP-01 del archivo `R16A-RE-FU-002-Back.md`
- Pendiente P1: confirmar mapeo exacto del rol antes de implementar

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002.md`

---

## Tarea 2

### [ R16A-RE-FU-002 ] [ IMP-EXIST-SERVICE ] Agregar método ObtenerIdUsuarioCobradorPorCliente() en ClienteCarteraClienteBO

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el método `ObtenerIdUsuarioCobradorPorCliente(Guid idCliente)` en `ClienteCarteraClienteBO.Extensions.cs` reutilizando el patrón de `ObtenerIDUsuarioESACPorCliente()`, para que el Catálogo de Clientes pueda mostrar el Cobrador actualmente asignado. Cumple **Criterio A1**.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito R16A-RE-FU-002 (GAP-02)
- Revisar el método existente `ObtenerIDUsuarioESACPorCliente()` como referencia del patrón
- Implementar `ObtenerIdUsuarioCobradorPorCliente(Guid idCliente)` navegando `ClienteCarteraCliente → ClienteCartera → IdUsuarioCobrador`
- Retornar `null` si el cliente no tiene cartera activa o no tiene cobrador asignado (Criterio C3)
- Ejecutar pruebas unitarias: cliente con cobrador, sin cobrador y sin cartera

**Resultado esperado:**
El método retorna el `IdUsuarioCobrador` de la cartera activa del cliente, o `null` si no tiene asignación. El Catálogo de Clientes puede mostrar el campo Cobrador correctamente.

**Entregables:**
- `ClienteCarteraClienteBO.Extensions.cs` con el nuevo método
- Pruebas unitarias del método

**Criterios de aceptación:**
- [ ] El método retorna el `IdUsuarioCobrador` de la cartera activa del cliente (Criterio A1)
- [ ] Retorna `null` si el cliente no tiene cartera activa o no tiene cobrador asignado (Criterio C3)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraClienteBO.Extensions.cs`
- GAP documentado: GAP-02 del archivo `R16A-RE-FU-002-Back.md`
- Patrón de referencia: `ObtenerIDUsuarioESACPorCliente()` en el mismo archivo

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002.md`

---

## Tarea 3

### [ R16A-RE-FU-002 ] [ SERV-TRANSACT ] Agregar método ReasignarCobrador() en ClienteCarteraBO

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el método `ReasignarCobrador(Guid idCliente, Guid idNuevoCobrador)` en `ClienteCarteraBO.cs` para encapsular la reasignación del Cobrador de un cliente con validación de negocio. Cumple **Criterios B2 / B3 / C2**.

> **Regla 4 — Filtrado dinámico por pantalla/módulo (OBS-005):** Al actualizar `IdUsuarioCobrador` en `ClienteCartera`, cada pantalla/módulo consumidor refleja el cambio en su siguiente consulta de bandeja — **no se requiere migración de registros**. Solo los pendientes **aún abiertos** (no finalizados) aparecen en la bandeja del nuevo Cobrador. El trabajo ya completado por el cobrador anterior permanece registrado donde fue ejecutado — no se reasigna ni se pierde (**Criterio C4**).

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito R16A-RE-FU-002 (GAP-03)
- Implementar `ReasignarCobrador(Guid idCliente, Guid idNuevoCobrador)` en `ClienteCarteraBO.cs`
- Validar que el nuevo cobrador tiene `AnalistaDeCuentasPorCobrar = true AND Activo = true`
- Obtener la cartera activa del cliente via JOIN `ClienteCartera → ClienteCarteraCliente`
- Actualizar `IdUsuarioCobrador` y `FechaUltimaActualizacion`
- Ejecutar pruebas unitarias: reasignación válida, cobrador inválido, cliente sin cartera

**Resultado esperado:**
El método reasigna el cobrador en la cartera activa del cliente. En la siguiente consulta de bandeja de cada módulo consumidor, los pendientes aún abiertos del cliente aparecen en la bandeja del nuevo Cobrador. El trabajo completado por el cobrador anterior no se reasigna.

**Entregables:**
- `ClienteCarteraBO.cs` con el nuevo método `ReasignarCobrador`
- Pruebas unitarias del método

**Criterios de aceptación:**
- [ ] El método actualiza `ClienteCartera.IdUsuarioCobrador` al nuevo valor (Criterio B2 / B3)
- [ ] El método actualiza `FechaUltimaActualizacion`
- [ ] Lanza excepción si el nuevo cobrador no es un Gestor de Cobranza activo (Regla 2)
- [ ] Lanza excepción si el cliente no tiene cartera activa
- [ ] Tras la actualización, el JOIN dinámico de módulos redistribuye los pendientes abiertos por pantalla/módulo sin migración de registros (Regla 4 / Criterio C2 — OBS-005)
- [ ] **Criterio C4 (OBS-005):** El trabajo completado por el cobrador anterior (registros finalizados) permanece registrado bajo ese cobrador — no se reasigna ni elimina
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraBO.cs`
- GAP documentado: GAP-03 del archivo `R16A-RE-FU-002-Back.md`

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002.md`

---

## Tarea 4

### [ R16A-RE-FU-002 ] [ LIST-NO-FILTER ] Agregar endpoint GET /ClienteCartera/GestoresDeCobranza (ListaUsuariosCartera )

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — WebApi.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el endpoint `GET /ClienteCartera/GestoresDeCobranza` en `UsuarioController.cs` para exponer la lista de Gestores de Cobranza activos al frontend, que la usará para poblar el selector del campo Cobrador. Cumple **Regla 2** y **Criterio B1**.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito R16A-RE-FU-002 (GAP-01)
- Agregar el método `ObtenerGestoresDeCobranzaActivos()` en el controller consumiendo el método implementado en la Tarea 1
- Documentar el endpoint en Swagger
- Verificar que el endpoint solo requiere autenticación básica (no rol específico para consulta)

**Resultado esperado:**
`GET /Usuario/GestoresDeCobranza` retorna la lista de usuarios activos con rol Gestor de Cobranza, ordenados por nombre, lista para poblar el selector.

**Entregables:**
- `UsuarioController.cs` con el nuevo endpoint documentado en Swagger
- Prueba manual del endpoint en ambiente de desarrollo

**Criterios de aceptación:**
- [ ] El endpoint retorna solo usuarios con `AnalistaDeCuentasPorCobrar = true AND Activo = true` (Criterio B1)
- [ ] El endpoint está documentado en Swagger
- [ ] Depende de Tarea 1 (método BO)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `WebApi.Catalogos\Controllers\Sistema\Usuarios\UsuarioController.cs`
- GAP documentado: GAP-01 del archivo `R16A-RE-FU-002-Back.md`

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002-Back.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002.md`

---

## Tarea 5

### [ R16A-RE-FU-002 ] [ SERV-TRANSACT ] Agregar endpoint PUT /ClienteCartera/ReasignarCobrador

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — WebApi.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, aprobación del líder técnico mediante PR y liberación en dev.

**Objetivo general:**
Agregar el endpoint `PUT /ClienteCartera/ReasignarCobrador` en `ClienteCarteraController.cs` para que el **Coordinador de Tesorería o Gerente de Tesorería** puedan reasignar el Cobrador de un cliente desde PQF2 (OBS-003). Al reasignar, los pendientes abiertos por pantalla/módulo pasan al nuevo Cobrador dinámicamente (Regla 4 / Criterio C2 — OBS-005). Cumple **Criterios B2 / B3 / C2 / C4**.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito R16A-RE-FU-002 (GAP-03 y GAP-04)
- Agregar el endpoint `PUT /ClienteCartera/ReasignarCobrador` consumiendo el método implementado en la Tarea 3
- Confirmar con Arquitectura la estrategia de validación de rol Coordinador/Gerente de Tesorería (Pendiente P2): ¿en capa BO o filtro de autorización en controller?
- Confirmar qué campo de `Usuario` mapea al rol **Gerente de Tesorería** (Pendiente OBS-003) antes de implementar la validación
- Documentar el endpoint en Swagger indicando los roles permitidos

**Resultado esperado:**
El endpoint permite al Coordinador o Gerente de Tesorería reasignar el Cobrador de un cliente. En la siguiente consulta de bandeja de cada módulo, los pendientes abiertos del cliente aparecen en la bandeja del nuevo Cobrador. El trabajo completado por el cobrador anterior no se reasigna (Criterio C4).

**Entregables:**
- `ClienteCarteraController.cs` con el nuevo endpoint
- Documentación Swagger del endpoint
- Decisión de arquitectura sobre validación de rol documentada (Pendiente P2)

**Criterios de aceptación:**
- [ ] `PUT /ClienteCartera/ReasignarCobrador` actualiza `IdUsuarioCobrador` en la cartera activa del cliente (Criterio B2 / B3)
- [ ] El endpoint valida que el solicitante tiene rol **Coordinador de Tesorería** (`GerenteDeTesoreria = true`) O **Gerente de Tesorería** (campo a confirmar — OBS-003) (Criterio A2 / Regla 1)
- [ ] Roles distintos a los autorizados reciben error de autorización (Criterio A3)
- [ ] Tras la reasignación, el filtrado dinámico redistribuye pendientes abiertos por pantalla/módulo sin migración de registros (Regla 4 / Criterio C2 — OBS-005)
- [ ] **Criterio C4 (OBS-005):** El trabajo completado por el cobrador anterior no se reasigna ni elimina
- [ ] Decisión de arquitectura Pendiente P2 registrada y aprobada
- [ ] Depende de Tarea 3 (método BO)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `WebApi.Catalogos\Controllers\Configuracion\Clientes\Cartera\ClienteCarteraController.cs`
- GAPs documentados: GAP-03 y GAP-04 del archivo `R16A-RE-FU-002-Back.md`
- Pendiente P2: definir estrategia de validación de rol antes de implementar
- Pendiente OBS-003: confirmar campo en `Usuario` para Gerente de Tesorería antes de implementar la validación de rol

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002-Back.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002.md`

---

## Tarea 6

### [ R16A-RE-FU-002 ] [ BD-OBJ-CH ] Actualizar SP spDesactivarCarterasCliente para incluir IdUsuarioCobrador en el retorno

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Base de Datos ProquifaDotNet

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Actualizar el SP `spDesactivarCarterasCliente` para que incluya `IdUsuarioCobrador` en el `SELECT` de retorno, habilitando la trazabilidad del cobrador al momento de desactivar carteras de un cliente. Resuelve **Pendiente P3** del Back.

**Objetivos específicos:**
- Leer el Pendiente P3 del archivo `R16A-RE-FU-002-Back.md`
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
- Objeto a modificar: `dbo.spDesactivarCarterasCliente` en BD `ProquifaDotNet`
- Pendiente documentado: P3 del archivo `R16A-RE-FU-002-Back.md`

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002.md`

---

## Tarea 7

### [ R16A-RE-FU-002 ] [ SCOPE-VERIFY ] Verificar exclusión del campo Cobrador en alta de cliente desde Cotizar lo Cotizable

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Cotizar lo Cotizable — Logic.Pqf.Logistica / WebApi.Logistica

**Consideraciones previas:**
Para esta actividad están contempladas revisión de código, verificación funcional, y PR de confirmación o corrección según resultado.

**Objetivo general:**
Verificar que el flujo de alta de cliente desde **Cotizar lo Cotizable** no incluye, no expone ni procesa el campo Cobrador (`IdUsuarioCobrador`). Según el requisito R16A-RE-FU-002 (Alcance — No aplica a), ese alta está orientada exclusivamente a habilitar la cotización, no a la gestión del cliente. La asignación de Cobrador se realiza posteriormente en el Catálogo de Clientes. Resuelve **GAP-05** del Back.

**Objetivos específicos:**
- Leer el GAP-05 del archivo `R16A-RE-FU-002-Back.md` y el Alcance `No aplica a` del requisito funcional
- Identificar el endpoint/BO utilizado por Cotizar lo Cotizable para el alta de cliente
- Verificar que `IdUsuarioCobrador` no es enviado ni procesado en ese flujo
- Si se detecta que el flujo lo incluye, eliminarlo y ajustar el BO correspondiente
- Documentar el resultado de la verificación

**Resultado esperado:**
El flujo de alta de cliente en Cotizar lo Cotizable no expone ni procesa el campo Cobrador. La asignación de Cobrador solo es posible desde el Catálogo de Clientes por el Coordinador de Tesorería.

**Entregables:**
- Informe de verificación (sin cambios si el flujo ya cumple, o PR de corrección si no cumple)
- Pendiente P5 del `R16A-RE-FU-002-Back.md` cerrado con el resultado

**Criterios de aceptación:**
- [ ] Confirmado que el alta de cliente en Cotizar lo Cotizable no expone el campo Cobrador al usuario
- [ ] Confirmado que el BO/endpoint del alta no procesa `IdUsuarioCobrador`
- [ ] Pendiente P5 del Back registrado como resuelto
- [ ] PR de verificación (o corrección) aprobado por líder técnico

**Más información de la tarea:**
- GAP documentado: GAP-05 del archivo `R16A-RE-FU-002-Back.md`
- Pendiente P5 del archivo `R16A-RE-FU-002-Back.md`
- Referencia de alcance: sección `No aplica a` del requisito `R16A-RE-FU-002.md`

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002-Back.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-002/R16A-RE-FU-002.md`
