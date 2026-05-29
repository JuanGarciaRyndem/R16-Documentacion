# Tareas — TPSC-RE-FU-001 Catálogo de Cuentas Bancarias del Grupo PROQUIFA

| Campo | Valor |
|---|---|
| **Requisito** | TPSC-RE-FU-001 |
| **Nombre** | Mantenimiento de catálogos del sistema |
| **Total de tareas** | 6 |

---

## Tarea 1

### [ TPSC-RE-FU-001 ] [ IMP-EXIST-SERVICE ] Corrección de ObtenerTodos() para filtrar cuentas vigentes

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Corregir el método `ObtenerTodos()` en `EmpresaDatosBancariosBO.Extensions.cs` para que retorne únicamente cuentas con `Activo = true`, cumpliendo con la Regla 2 y el Criterio B1 del requisito.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-001 (GAP-01)
- Localizar el método `ObtenerTodos()` en `EmpresaDatosBancariosBO.Extensions.cs`
- Agregar el filtro `Where(x => x.Activo)` al query
- Verificar que ningún módulo consumidor dependa del comportamiento anterior (sin filtro)
- Ejecutar pruebas unitarias sobre el método corregido

**Resultado esperado:**
El método `ObtenerTodos()` retorna únicamente las cuentas con `Activo = true`. Las cuentas históricas (`Activo = false`) no son devueltas a los módulos consumidores.

**Entregables:**
- `EmpresaDatosBancariosBO.Extensions.cs` modificado con filtro `Activo = true`
- Pruebas unitarias del método

**Criterios de aceptación:**
- [ ] `ObtenerTodos()` retorna únicamente cuentas con `Activo = true`
- [ ] Las cuentas con `Activo = false` no aparecen en el resultado
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs`
- GAP documentado: GAP-01 del archivo `TPSC-RE-FU-001-Back.md`
- Las cuentas inactivas se conservan en BD para trazabilidad histórica, solo se excluyen de las consultas operativas.

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 2

### [ TPSC-RE-FU-001 ] [ IMP-EXIST-SERVICE ] Agregar método ObtenerVigentesPorEmpresa() en EmpresaDatosBancariosBO

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el método `ObtenerVigentesPorEmpresa(string prefijo)` en `EmpresaDatosBancariosBO.Extensions.cs` para permitir que los módulos consulten cuentas vigentes filtradas por empresa emisora del grupo, cumpliendo el Criterio A2 del requisito (GAP-02).

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-001 (GAP-02)
- Implementar el método `ObtenerVigentesPorEmpresa(string prefijo)` en `EmpresaDatosBancariosBO.Extensions.cs`
- Validar que el prefijo recibido esté dentro del set permitido: `GOL`, `MUN`, `PRO`, `PQF`
- Aplicar filtro `Activo = true` tanto en `Empresa` como en `EmpresaDatosBancarios`
- Retornar lista vacía si el prefijo no es válido o la empresa no existe
- Ejecutar pruebas unitarias con cada prefijo válido e inválido

**Resultado esperado:**
El método retorna las cuentas vigentes de la empresa cuyo prefijo coincide con el solicitado. Prefijos fuera del alcance R16 (ej. `GOLPERU`) retornan lista vacía.

**Entregables:**
- `EmpresaDatosBancariosBO.Extensions.cs` con el nuevo método
- Pruebas unitarias del método para los 4 prefijos válidos y casos inválidos

**Criterios de aceptación:**
- [ ] `ObtenerVigentesPorEmpresa("GOL")` retorna cuentas vigentes de Golocaer
- [ ] `ObtenerVigentesPorEmpresa("GOLPERU")` retorna lista vacía
- [ ] Solo se retornan cuentas con `Activo = true`
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs`
- GAP documentado: GAP-02 del archivo `TPSC-RE-FU-001-Back.md`
- Prefijos válidos R16: `GOL`, `MUN`, `PRO`, `PQF`. Excluir `GOLPERU`.

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 3

### [ TPSC-RE-FU-001 ] [ LIST-NO-FILTER ] Agregar endpoint GET /EmpresaDatosBancarios/Vigentes/{prefijo}

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — WebApi.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el endpoint `GET /EmpresaDatosBancarios/Vigentes/{prefijo}` en `EmpresaDatosBancariosController.cs` para exponer las cuentas vigentes filtradas por empresa emisora a los módulos consumidores, cumpliendo el Criterio A2 del requisito (GAP-03).

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-001 (GAP-03)
- Agregar el método `ObtenerVigentesPorEmpresa(string prefijo)` en el controller
- Consumir el método del mismo nombre implementado en la Tarea 2 (`EmpresaDatosBancariosBO`)
- Documentar el endpoint en Swagger con descripción de prefijos válidos R16
- Verificar que el endpoint no requiera UI (solo consumo interno entre módulos)

**Resultado esperado:**
El endpoint `GET /EmpresaDatosBancarios/Vigentes/{prefijo}` responde con la lista de cuentas vigentes de la empresa solicitada. La respuesta excluye cuentas inactivas y empresas fuera del alcance R16.

**Entregables:**
- `EmpresaDatosBancariosController.cs` con el nuevo endpoint
- Documentación Swagger del endpoint

**Criterios de aceptación:**
- [ ] `GET /EmpresaDatosBancarios/Vigentes/GOL` retorna cuentas vigentes de Golocaer
- [ ] `GET /EmpresaDatosBancarios/Vigentes/GOLPERU` retorna lista vacía o 404
- [ ] El endpoint está documentado en Swagger con descripción de prefijos válidos
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `WebApi.Catalogos\Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs`
- GAP documentado: GAP-03 del archivo `TPSC-RE-FU-001-Back.md`
- Esta tarea depende de la Tarea 2 (método BO)

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 4

### [ TPSC-RE-FU-001 ] [ UPDATE-TABL-CH ] Documentar restricción de endpoints PUT/DELETE en EmpresaDatosBancariosController

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — WebApi.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, aprobación del líder técnico mediante PR y liberación en dev.

**Objetivo general:**
Documentar y restringir los endpoints `PUT GuardarOActualizar` y `DELETE Desactivar` de `EmpresaDatosBancariosController.cs` para dejar explícito que son de uso exclusivo de Soporte a la Producción y no deben exponerse en ninguna pantalla de PQF2, cumpliendo la Regla 3 y el Criterio C1 del requisito (GAP-04).

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-001 (GAP-04)
- Agregar documentación Swagger en los endpoints PUT y DELETE indicando uso exclusivo de Soporte a la Producción
- Confirmar con arquitectura/TechLead si se aplica `[ApiExplorerSettings(IgnoreApi = true)]` o política de autorización restrictiva
- Verificar que ninguna pantalla de PQF2 consuma estos endpoints en R16

**Resultado esperado:**
Los endpoints de escritura quedan documentados como internos. Ninguna pantalla de PQF2 los consume. La decisión de ocultarlos del Swagger queda registrada.

**Entregables:**
- `EmpresaDatosBancariosController.cs` con documentación actualizada en PUT y DELETE
- Decisión de arquitectura documentada (ocultar o restringir)

**Criterios de aceptación:**
- [ ] Los endpoints PUT y DELETE tienen documentación Swagger indicando uso exclusivo de Soporte a la Producción
- [ ] Ninguna pantalla de PQF2 en R16 consume estos endpoints
- [ ] Decisión de arquitectura (P1) registrada y aprobada
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a revisar: `WebApi.Catalogos\Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs`
- GAP documentado: GAP-04 del archivo `TPSC-RE-FU-001-Back.md`
- Pendiente P1: decidir entre `[ApiExplorerSettings(IgnoreApi = true)]` o política de autorización restrictiva.

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 5

### [ TPSC-RE-FU-001 ] [ BD-OBJ-CH ] Construcción de vista vEmpresaDatosBancariosVigentes

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — Base de Datos ProquifaDotNet

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Construir la vista `vEmpresaDatosBancariosVigentes` en la BD `ProquifaDotNet` que centralice el filtro de cuentas vigentes del grupo PROQUIFA México, evitando que cada módulo consumidor duplique la lógica de filtrado por `Activo = 1` y `Prefijo IN ('GOL','MUN','PRO','PQF')`. Cumple con la Regla 4 y el GAP-05 del requisito.

**Objetivos específicos:**
- Leer el diccionario de datos `TPSC-RE-FU-001_BD.md` y el análisis backend `TPSC-RE-FU-001-Back.md` (GAP-05)
- Diseñar y construir la vista con JOIN entre `EmpresaDatosBancarios`, `Empresa`, `DatosBancarios`, `catBanco` y `catMoneda`
- Aplicar filtros: `EmpresaDatosBancarios.Activo = 1`, `Empresa.Activo = 1`, `Empresa.Prefijo IN ('GOL','MUN','PRO','PQF')`
- Verificar que la vista excluye registros de `GOLPERU`
- Documentar la vista en el diccionario de datos del requisito

**Resultado esperado:**
La vista `vEmpresaDatosBancariosVigentes` retorna únicamente las cuentas vigentes de las empresas mexicanas del grupo. Los módulos consumidores pueden hacer un `SELECT` simple contra la vista sin necesidad de replicar los filtros.

**Entregables:**
- Script de creación de la vista `vEmpresaDatosBancariosVigentes`
- Vista registrada en el script de control de BD del release

**Criterios de aceptación:**
- [ ] La vista retorna cuentas con `Activo = 1` de `EmpresaDatosBancarios`
- [ ] La vista excluye empresas fuera del alcance R16 (`GOLPERU` y cualquier prefijo no listado)
- [ ] La vista expone al menos: `Prefijo`, `Alias`, `Banco`, `NumeroDeCuenta`, `Clabe`, `ClaveMoneda`, `FechaRegistro`
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP documentado: GAP-05 del archivo `TPSC-RE-FU-001-Back.md`
- Tablas involucradas: `EmpresaDatosBancarios`, `Empresa`, `DatosBancarios`, `catBanco`, `catMoneda`
- La vista es de solo lectura; la gestión del catálogo sigue siendo responsabilidad de Soporte a la Producción directamente en BD.

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 6

### [ TPSC-RE-FU-001 ] [ MIG-DATOS ] Carga inicial de cuentas bancarias del grupo PROQUIFA

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — Base de Datos ProquifaDotNet

**Consideraciones previas:**
Para esta actividad están contempladas la recepción de datos por parte de PROQUIFA, construcción del script de carga, validación, aprobación del líder técnico mediante PR y liberación en dev.

> ⚠️ **Bloqueante:** Esta tarea está bloqueada hasta recibir el detalle completo de cuentas por empresa del grupo (banco, número de cuenta, CLABE, moneda, códigos). Pendiente entrega por PROQUIFA — ver Pendiente P4 en `TPSC-RE-FU-001-Back.md`.

**Objetivo general:**
Cargar los datos iniciales de las cuentas bancarias vigentes de las cuatro empresas del grupo PROQUIFA México (`GOL`, `MUN`, `PRO`, `PQF`) en las tablas `EmpresaDatosBancarios` y `DatosBancarios` de la BD `ProquifaDotNet`, como requisito previo al funcionamiento del catálogo en cualquier módulo consumidor.

**Objetivos específicos:**
- Recibir y validar el archivo de datos de cuentas bancarias entregado por PROQUIFA
- Verificar que cada cuenta tenga: empresa emisora, banco, número de cuenta, CLABE, moneda
- Construir script `INSERT` para `DatosBancarios` y `EmpresaDatosBancarios` con `Activo = 1`
- Validar que los `IdEmpresa` referenciados corresponden a `GOL`, `MUN`, `PRO`, `PQF` en la tabla `Empresa`
- Validar que los `IdCatBanco` e `IdCatMoneda` existan en los catálogos correspondientes
- Ejecutar script en ambiente de desarrollo y verificar resultados con la vista `vEmpresaDatosBancariosVigentes`
- Registrar el script en el formulario de control de scripts del release

**Resultado esperado:**
Las cuentas bancarias del grupo están disponibles en BD con `Activo = 1`. Los módulos consumidores pueden consultar el catálogo correctamente a través de la vista `vEmpresaDatosBancariosVigentes`.

**Entregables:**
- Script de carga inicial `INSERT` para `DatosBancarios` y `EmpresaDatosBancarios`
- Script registrado en el formulario de control de scripts del release
- Evidencia de validación en ambiente de desarrollo

**Criterios de aceptación:**
- [ ] Las cuatro empresas del grupo (`GOL`, `MUN`, `PRO`, `PQF`) tienen al menos una cuenta cargada con `Activo = 1`
- [ ] Todos los registros tienen `IdCatBanco` e `IdCatMoneda` válidos
- [ ] La vista `vEmpresaDatosBancariosVigentes` retorna los registros cargados correctamente
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Pendiente P4 del archivo `TPSC-RE-FU-001-Back.md`: entrega del detalle completo de cuentas por PROQUIFA
- Tablas destino: `DatosBancarios` (detalle bancario) y `EmpresaDatosBancarios` (vínculo con empresa)
- Esta tarea depende de la Tarea 5 (vista `vEmpresaDatosBancariosVigentes`) para la validación final

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`
