# TPSC-RE-FU-001 — Tareas de Implementación Backend

| Campo                 | Valor                                            |
| --------------------- | ------------------------------------------------ |
| **Requisito**         | TPSC-RE-FU-001                                   |
| **Nombre**            | Catálogo de Cuentas Bancarias del Grupo PROQUIFA |
| **Total de tareas**   | 6                                                |

---

## Tarea 1

### [TPSC-RE-FU-001] [BD-OBJ-CH] Script de migración — `IdRegion` en `EmpresaDatosBancarios`

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogo de Cuentas Bancarias del Grupo PROQUIFA

**Consideraciones previas:**
Esta tarea es **prerequisito de Tarea 2**. El `ALTER TABLE` debe ejecutarse antes de crear la vista `vEmpresaDatosBancarios`, ya que la vista depende de la columna `IdRegion`.
Verificar que no existan restricciones de FK que impidan el ALTER antes de ejecutar.
Ejecutar por ambiente en orden: DEV → QA → PROD.
Después de ejecutar el ALTER, el modelo EF (`ProquifaDotNetEntities`) debe actualizarse para incluir la propiedad `IdRegion` en la entidad `EmpresaDatosBancarios` (ver Tarea 3).

**Objetivo general:**
Agregar la columna `IdRegion` a la tabla `EmpresaDatosBancarios` y poblarla con los valores correctos según la empresa, para habilitar la regionalización del catálogo bancario (MEX / PER).

**Objetivos específicos:**
- Ejecutar `ALTER TABLE dbo.EmpresaDatosBancarios ADD IdRegion uniqueidentifier NULL`
- Agregar FK `FK_EmpresaDatosBancarios_Region` → `Region.IdRegion`
- Poblar `IdRegion` en todos los registros existentes asociando `MEX` a empresas GOL/MUN/PRO/PQF
- Verificar que ningún registro quede con `IdRegion = NULL` tras el UPDATE

**Resultado esperado:**
La columna `IdRegion` existe en `EmpresaDatosBancarios`, está correctamente poblada para todos los registros existentes y tiene FK a `Region`.

**Entregables:**
- Script SQL: `Scripts\R16\FU-001_EmpresaDatosBancarios_IdRegion.sql`

**Criterios de aceptación:**
- [ ] La columna `IdRegion uniqueidentifier NULL` existe en `dbo.EmpresaDatosBancarios`
- [ ] Existe `FK_EmpresaDatosBancarios_Region` referenciando `Region.IdRegion`
- [ ] Todos los registros de empresas GOL/MUN/PRO/PQF tienen `IdRegion` correspondiente a la región MEX
- [ ] Ningún registro existente tiene `IdRegion = NULL` después del UPDATE inicial
- [ ] El script incluye bloque de verificación con `SELECT` final
- [ ] Script ejecutado correctamente en DEV sin errores

**Más información de la tarea:**
- GAP-01 del archivo `TPSC-RE-FU-001-Back.md`
- Regla de negocio cubierta: **Regla 4** (consumo regionalizado)
- Ver diccionario de datos: `TPSC-RE-FU-001_BD.md` — sección `EmpresaDatosBancarios`
- ⚠️ Ejecutar **antes** de Tarea 2

**Recursos:**
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 2

### [TPSC-RE-FU-001] [BD-OBJ-CH] Script de creación — Vista `vEmpresaDatosBancarios`

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogo de Cuentas Bancarias del Grupo PROQUIFA

**Consideraciones previas:**
**Depende de Tarea 1.** La columna `IdRegion` debe existir en `EmpresaDatosBancarios` antes de ejecutar este script.
La vista es el **punto único de consulta** para todos los módulos consumidores del catálogo bancario. Ningún módulo debe consultar la tabla directamente.
Ejecutar por ambiente en orden: DEV → QA → PROD.
La vista se creará como `CREATE OR ALTER VIEW` para permitir re-ejecución segura.

**Objetivo general:**
Crear la vista `vEmpresaDatosBancarios` que resuelve el JOIN completo entre `EmpresaDatosBancarios`, `Empresa`, `Region`, `DatosBancarios`, `catBanco` y `catMoneda`, exponiendo solo cuentas vigentes regionalizadas.

**Objetivos específicos:**
- Crear `vEmpresaDatosBancarios` con JOIN completo de todas las entidades relacionadas
- Incluir columnas: `IdEmpresaDatosBancarios`, `EmpresaPrefijo`, `EmpresaAlias`, `IdRegion`, `RegionClave`, `Banco`, `ClaveBanco`, `NumeroDeCuenta`, `Clabe`, `Sucursal`, `ClaveMoneda`, `Moneda`, `Activo`, `FechaRegistro`
- Verificar que el SELECT de prueba retorna registros correctos por región

**Resultado esperado:**
La vista `vEmpresaDatosBancarios` existe en BD y puede consultarse filtrando por `IdRegion` y `Activo = 1`.

**Entregables:**
- Script SQL: `Scripts\R16\FU-001_vEmpresaDatosBancarios.sql`

**Criterios de aceptación:**
- [ ] La vista `dbo.vEmpresaDatosBancarios` existe en BD
- [ ] Retorna correctamente columnas de empresa, banco, moneda y región en un solo resultado
- [ ] El filtro `WHERE IdRegion = @IdRegion AND Activo = 1` retorna solo cuentas vigentes de la región indicada
- [ ] No retorna registros de empresas fuera del alcance R16 (p. ej. GOLPERU para MEX)
- [ ] Script ejecutado correctamente en DEV sin errores

**Más información de la tarea:**
- GAP-02 del archivo `TPSC-RE-FU-001-Back.md`
- Reglas de negocio cubiertas: **Regla 2** (vigencia), **Regla 4** (consumo regionalizado)
- Ver diccionario de datos: `TPSC-RE-FU-001_BD.md` — sección `vEmpresaDatosBancarios`
- ⚠️ Requiere **Tarea 1** completada

**Recursos:**
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 3

### [TPSC-RE-FU-001] [IMP-EXIST-SERVICE] Corregir `ObtenerTodos()` y actualizar modelo EF

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Catalogos

**Módulos:**
Catálogo de Cuentas Bancarias del Grupo PROQUIFA

**Consideraciones previas:**
**Depende de Tarea 1.** El modelo EF (`ProquifaDotNetEntities`) debe actualizarse para reflejar la nueva columna `IdRegion` en `EmpresaDatosBancarios`. Confirmar con TechLead si la actualización es por scaffolding o manual (ver Pendiente P1 del Back.md).
Esta tarea puede ejecutarse en paralelo con Tarea 2.
El cambio en `ObtenerTodos()` es una corrección de bug: actualmente devuelve cuentas no vigentes, lo que viola la Regla 2 del requisito.

**Objetivo general:**
Corregir `ObtenerTodos()` en `EmpresaDatosBancariosBO.Extensions.cs` para que devuelva únicamente cuentas vigentes (`Activo = 1`), y actualizar el modelo EF para incluir la propiedad `IdRegion` en la entidad `EmpresaDatosBancarios`.

**Objetivos específicos:**
- Agregar `.Where(x => x.Activo)` en `ObtenerTodos()`
- Actualizar la entidad `EmpresaDatosBancarios` en el modelo EF para incluir `IdRegion`
- Verificar que los proyectos que consumen `ObtenerTodos()` no se rompan con el cambio

**Resultado esperado:**
`ObtenerTodos()` nunca devuelve cuentas no vigentes. La entidad EF tiene la propiedad `IdRegion` disponible para las Tareas 4 y 5.

**Entregables:**
- `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs` (modificado)
- Entidad EF `EmpresaDatosBancarios` actualizada con `IdRegion`

**Criterios de aceptación:**
- [ ] `ObtenerTodos()` incluye `.Where(x => x.Activo)` — no retorna cuentas con `Activo = false`
- [ ] La entidad `EmpresaDatosBancarios` en EF tiene la propiedad `IdRegion` de tipo `Guid?`
- [ ] Ningún consumidor existente de `ObtenerTodos()` genera error de compilación tras el cambio
- [ ] El proyecto `Logic.Pqf.Catalogos` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-03 del archivo `TPSC-RE-FU-001-Back.md`
- Regla de negocio cubierta: **Regla 2** (estado de existencia vigente)
- Pendiente P1: confirmar estrategia de actualización del modelo EF con TechLead antes de iniciar

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 4

### [TPSC-RE-FU-001] [LIST-PAG-MULT-FILTER] Crear `vEmpresaDatosBancariosBO`

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Catalogos

**Módulos:**
Catálogo de Cuentas Bancarias del Grupo PROQUIFA

**Consideraciones previas:**
**Depende de Tarea 2 y Tarea 3.** La vista `vEmpresaDatosBancarios` debe existir en BD y la entidad EF debe estar disponible antes de crear este BO.
La región se recibe como parámetro `Guid idRegion` — es responsabilidad del controller extraerla del token mediante `ObtenerUsuarioAutenticado()`. El BO no conoce ni extrae el token.
El listado paginado sigue el patrón estándar `QueryInfo` / `QueryResult<T>` del proyecto.

**Objetivo general:**
Crear el BO `vEmpresaDatosBancariosBO` con dos métodos: `ObtenerPorId` para obtener el detalle de una cuenta vigente, y `Lista` para el listado paginado regionalizado filtrado por `IdRegion` del usuario logueado.

**Objetivos específicos:**
- Crear `vEmpresaDatosBancariosBO.cs` en `Logic.Pqf.Catalogos\Empresas\DatosBancarios\`
- Implementar `ObtenerPorId(Guid idEmpresaDatosBancarios)` — retorna `vEmpresaDatosBancarios` vigente o `null`
- Implementar `Lista(QueryInfo info, Guid idRegion)` — retorna `QueryResult<vEmpresaDatosBancarios>` paginado, filtrado por `IdRegion` y `Activo = 1`, ordenado por `EmpresaPrefijo`

**Resultado esperado:**
El BO encapsula toda la lógica de consulta sobre la vista, aplicando correctamente los filtros de vigencia y región.

**Entregables:**
- `Logic.Pqf.Catalogos\Empresas\DatosBancarios\vEmpresaDatosBancariosBO.cs` (nuevo)

**Criterios de aceptación:**
- [ ] `ObtenerPorId` retorna `null` si la cuenta no existe o tiene `Activo = false`
- [ ] `Lista` filtra por `IdRegion == idRegion` y `Activo == true`
- [ ] `Lista` retorna `QueryResult<vEmpresaDatosBancarios>` con `Total` y `Results` correctos
- [ ] `Lista` ordena los resultados por `EmpresaPrefijo` ascendente
- [ ] `Lista` aplica `Skip` y `Take` de `QueryInfo` correctamente
- [ ] El proyecto `Logic.Pqf.Catalogos` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-04 del archivo `TPSC-RE-FU-001-Back.md`
- Reglas de negocio cubiertas: **Regla 2** (vigencia), **Regla 4** (consumo regionalizado)
- Criterios cubiertos: **A1**, **A2**, **B1**

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 5

### [TPSC-RE-FU-001] [LIST-MULT-FILTER] Crear `vEmpresaDatosBancariosController`

**Aplicativos:**
ProquifaNet 2 — WebApi.Catalogos

**Módulos:**
Catálogo de Cuentas Bancarias del Grupo PROQUIFA

**Consideraciones previas:**
**Depende de Tarea 4.** `vEmpresaDatosBancariosBO` debe existir y compilar antes de crear el controller.
La región **se extrae del token** mediante `ObtenerUsuarioAutenticado().IdRegion` — nunca se acepta como parámetro externo en la ruta ni en el body.
El controller hereda de `BaseApiController` para tener acceso a `ObtenerUsuarioAutenticado()` y `TryExecute()`.
Este controller **no expone** endpoints de escritura (PUT/DELETE). Los endpoints de gestión del catálogo permanecen en `EmpresaDatosBancariosController` para uso exclusivo de Soporte a la Producción.

**Objetivo general:**
Crear `vEmpresaDatosBancariosController` que exponga dos endpoints sobre la vista: `GET /vEmpresaDatosBancarios` para obtener una cuenta por Id, y `POST /vEmpresaDatosBancarios` para el listado paginado regionalizado.

**Objetivos específicos:**
- Crear `vEmpresaDatosBancariosController.cs` en `WebApi.Catalogos\Controllers\Configuracion\Empresas\`
- Implementar `GET /vEmpresaDatosBancarios?idEmpresaDatosBancarios={id}` → llama a `vEmpresaDatosBancariosBO.ObtenerPorId`
- Implementar `POST /vEmpresaDatosBancarios` (body: `QueryInfo`) → extrae `IdRegion` del token, llama a `vEmpresaDatosBancariosBO.Lista`
- Responder `200 OK` con resultado o `204 NoContent` si no hay datos

**Resultado esperado:**
El frontend puede consultar el catálogo bancario paginado y regionalizado a través de los dos endpoints, con la región resuelta automáticamente del usuario autenticado.

**Entregables:**
- `WebApi.Catalogos\Controllers\Configuracion\Empresas\vEmpresaDatosBancariosController.cs` (nuevo)

**Criterios de aceptación:**
- [ ] `GET /vEmpresaDatosBancarios?idEmpresaDatosBancarios={id}` retorna `200 OK` con la cuenta o `204 NoContent`
- [ ] `POST /vEmpresaDatosBancarios` retorna `200 OK` con `QueryResult<vEmpresaDatosBancarios>` paginado o `204 NoContent`
- [ ] `IdRegion` se extrae de `ObtenerUsuarioAutenticado().IdRegion` — no se acepta como parámetro externo
- [ ] El controller hereda de `BaseApiController` y requiere autenticación
- [ ] El controller **no expone** endpoints PUT ni DELETE
- [ ] Respuestas de error devuelven `500 InternalServerError` con mensaje descriptivo (vía `TryExecute`)
- [ ] El proyecto `WebApi.Catalogos` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-04 del archivo `TPSC-RE-FU-001-Back.md`
- Reglas de negocio cubiertas: **Regla 2**, **Regla 3** (sin UI de gestión), **Regla 4**
- Criterios cubiertos: **A1**, **A2**, **B1**, **C1**
- Confirmar nombre de ruta con el equipo de front antes de liberar en DEV

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 6

### [TPSC-RE-FU-001] [UPDATE-DOC-PROD] Guía de Resolución para Soporte a la Producción

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogo de Cuentas Bancarias del Grupo PROQUIFA

**Consideraciones previas:**
Esta tarea cubre el **Criterio C2** del requisito: la gestión del catálogo (alta, baja, modificación de cuentas) es responsabilidad exclusiva de Soporte a la Producción mediante acceso directo a BD.
**No existe UI** para que el usuario operativo gestione el catálogo en R16 (Criterio C1).
La guía debe ser lo suficientemente clara para que Soporte a la Producción pueda operar el catálogo sin asistencia del equipo de desarrollo.
Esta tarea puede ejecutarse en paralelo con las demás una vez definido el modelo de datos (Tareas 1 y 2 completadas).

**Objetivo general:**
Crear un documento de Guía de Resolución que instruya a Soporte a la Producción sobre cómo dar de alta, modificar y dar de baja cuentas bancarias del grupo PROQUIFA directamente en BD, incluyendo las consideraciones de regionalización de R16.

**Objetivos específicos:**
- Documentar el procedimiento de **alta** de una cuenta bancaria (INSERT en `DatosBancarios` + `EmpresaDatosBancarios`)
- Documentar el procedimiento de **modificación** de una cuenta bancaria (UPDATE)
- Documentar el procedimiento de **baja lógica** de una cuenta (`Activo = 0` — nunca DELETE físico)
- Incluir scripts SQL parametrizados de referencia para cada operación
- Documentar la validación de integridad post-operación usando `vEmpresaDatosBancarios`
- Especificar el flujo de solicitud: quién solicita, quién autoriza, quién ejecuta

**Resultado esperado:**
Soporte a la Producción cuenta con un documento de referencia que le permite operar el catálogo bancario de forma autónoma, segura y consistente con las reglas de negocio del requisito.

**Entregables:**
- Guía de Resolución

**Criterios de aceptación:**
- [ ] El documento incluye procedimiento de alta con script SQL parametrizado
- [ ] El documento incluye procedimiento de modificación con script SQL parametrizado
- [ ] El documento incluye procedimiento de baja lógica (`Activo = 0`) — sin DELETE físico
- [ ] Cada procedimiento incluye una consulta de verificación post-ejecución sobre `vEmpresaDatosBancarios`
- [ ] El documento especifica que `IdRegion` es obligatorio al dar de alta (MEX para GOL/MUN/PRO/PQF)
- [ ] El documento especifica que nunca se deben eliminar físicamente registros
- [ ] El documento incluye el flujo de solicitud (quién solicita → quién autoriza → Soporte ejecuta)
- [ ] Revisado y aprobado por TechLead o Product Owner antes de entrega a Soporte a la Producción

**Más información de la tarea:**
- Revisión aplicada: `TPSC-RE-FU-001-Revision.md` — "Agregar requisito de mantenimiento post-go-live del catálogo"
- Reglas de negocio cubiertas: **Regla 2** (sin DELETE físico), **Regla 3** (sin UI — gestión manual)
- Criterios cubiertos: **C1**, **C2**
- Pendiente P2: la guía debe actualizarse una vez PROQUIFA entregue el detalle completo de cuentas

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`
- Revisión: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Revision.md`
