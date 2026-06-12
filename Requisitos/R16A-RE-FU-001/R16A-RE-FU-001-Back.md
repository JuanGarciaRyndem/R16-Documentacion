# R16A-RE-FU-001-Back — Análisis de Impacto Backend
**Requisito:** R16A-RE-FU-001 — Catálogo de Cuentas Bancarias del Grupo PROQUIFA  
**Proyecto:** ProquifaDotNet  

---

## 1. Estructura actual en el proyecto

### 1.1 Capa de Lógica (`Logic.Pqf.Catalogos`)

| Archivo | Clase | Descripción |
|---------|-------|-------------|
| `Empresas\DatosBancarios\EmpresaDatosBancariosBO.cs` | `EmpresaDatosBancariosBO` | CRUD principal sobre `EmpresaDatosBancarios`. Filtros por `IdCatMoneda` y `NumeroDeCuenta`. |
| `Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs` | `EmpresaDatosBancariosBO` (partial) | `ObtenerTodos()` — devuelve todas las cuentas sin excluir las no vigentes. |
| `Empresas\DatosBancarios\EmpresaDatosBancariosDetalleBO.cs` | `EmpresaDatosBancariosDetalleBO` | View-model que enriquece cada cuenta con `DatosBancarios`, `catBanco` y `catMoneda`. |
| `Empresas\DatosBancarios\Models\EmpresaDatosBancariosDetalle.cs` | `EmpresaDatosBancariosDetalle` | Modelo enriquecido: hereda de `EmpresaDatosBancarios` + `DatosBancarios` + `catBanco` + `catMoneda`. |
| `Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.cs` | `DatosBancariosBO` | CRUD de `DatosBancarios` (número de cuenta, CLABE, banco, moneda). |
| `Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.Empresa.Extensions.cs` | `DatosBancariosBO` (partial) | `ObtenerEmpresaDeDatosBancarios(Guid)` — resuelve la empresa a partir de `IdDatosBancarios`. |
| `Empresas\EmpresaBO.cs` / `.Extensions.cs` | `EmpresaBO` | CRUD de `Empresa`. Usa `EmpresaRegion` para filtrar por región. |

### 1.2 Capa de API (`WebApi.Catalogos`)

| Archivo | Ruta HTTP | Métodos expuestos |
|---------|-----------|-------------------|
| `Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs` | `/EmpresaDatosBancarios` | GET Obtener, PUT GuardarOActualizar, POST QueryResult, DELETE Desactivar |
| `Controllers\Configuracion\Empresas\EmpresaDatosBancariosDetalleController.cs` | `/EmpresaDatosBancariosDetalle` | POST QueryResult, POST GrupoListaEmpresaDatosBancariosDetalle |

> **Nuevo controller requerido R16:** `vEmpresaDatosBancariosController` para exponer la vista regional.

---

## 2. Mapeo de reglas de negocio al código

| Regla del Requisito | Descripción de negocio | Implementación esperada en backend | Estado actual |
|---------------------|------------------------|------------------------------------|---------------|
| **Regla 1** — Catálogo único | El sistema mantiene un único catálogo de cuentas bancarias del grupo como fuente de verdad. | `EmpresaDatosBancarios` como tabla centralizada. Sin catálogos paralelos. | ✅ Existe |
| **Regla 2** — Estado de existencia vigente | Solo las cuentas vigentes se ofrecen a los módulos consumidores. Las no vigentes se conservan para trazabilidad. | Consultas filtran `Activo = 1`. Las no vigentes nunca aparecen en listados operativos. | ⚠️ `ObtenerTodos()` NO excluye no vigentes |
| **Regla 3** — Gestión sin interfaz gráfica | La gestión no tiene UI en R16. Responsabilidad de Soporte a la Producción vía BD. | Los endpoints PUT/DELETE existen pero ninguna pantalla los consume. | ✅ Sin UI |
| **Regla 4** — Consumo centralizado desde módulos | Los módulos consultan este catálogo como fuente única. Regionalizado por el usuario logueado. | Módulos usan `vEmpresaDatosBancarios` filtrada por `IdRegion` del usuario logueado. | ⚠️ Vista no existe — `IdRegion` no está en `EmpresaDatosBancarios` |

---

## 3. Gaps identificados y cambios requeridos

### GAP-01 — `EmpresaDatosBancarios` no tiene columna `IdRegion`
**Tabla:** `EmpresaDatosBancarios`  
**Impacto:** Regla 4 — No es posible regionalizar el catálogo sin esta columna.  
**Cambio requerido:** `ALTER TABLE` para agregar `IdRegion` FK → `Region`. Ver `R16A-RE-FU-001_BD.md` para el script completo.

---

### GAP-02 — No existe la vista `vEmpresaDatosBancarios`
**Objeto:** Vista nueva en BD  
**Impacto:** Reglas 2 y 4 — Los módulos no tienen un punto único de consulta que resuelva empresa, banco, moneda y región.  
**Cambio requerido:** Crear `vEmpresaDatosBancarios` con JOIN completo. Ver `R16A-RE-FU-001_BD.md` para el script.

---

### GAP-03 — `ObtenerTodos()` devuelve cuentas no vigentes
**Archivo:** `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs`  
**Impacto:** Regla 2 — Los módulos pueden recibir cuentas no vigentes en sistema.  
**Cambio requerido:**

```csharp
// ANTES
public List<EmpresaDatosBancarios> ObtenerTodos()
{
    using (var db = new ProquifaDotNetEntities())
    {
        return db.EmpresaDatosBancarios.ToList();
    }
}

// DESPUÉS — solo cuentas vigentes
public List<EmpresaDatosBancarios> ObtenerTodos()
{
    using (var db = new ProquifaDotNetEntities())
    {
        return db.EmpresaDatosBancarios
            .Where(x => x.Activo)
            .ToList();
    }
}
```

---

### GAP-04 — No existe BO ni controller para `vEmpresaDatosBancarios`
**Impacto:** Reglas 2 y 4 — El frontend no tiene cómo consultar el catálogo bancario regionalizado.  
**Cambio requerido:** Crear BO y controller para exponer la vista.

**Nuevo archivo:** `Logic.Pqf.Catalogos\Empresas\DatosBancarios\vEmpresaDatosBancariosBO.cs`

```csharp
using Core.CrudTools.Optimization;
using Core.Pqf.ProquifaDotNetContext;
using System;
using System.Linq;

namespace Logic.Pqf.Catalogos.Empresas.DatosBancarios
{
    public class vEmpresaDatosBancariosBO
    {
        /// <summary>
        /// Obtiene el detalle de una cuenta bancaria por su Id.
        /// Solo retorna cuentas vigentes (Activo = 1).
        /// </summary>
        public vEmpresaDatosBancarios ObtenerPorId(Guid idEmpresaDatosBancarios)
        {
            using (var db = new ProquifaDotNetEntities())
            {
                return db.vEmpresaDatosBancarios
                    .FirstOrDefault(v =>
                        v.IdEmpresaDatosBancarios == idEmpresaDatosBancarios &&
                        v.Activo);
            }
        }

        /// <summary>
        /// Listado paginado de cuentas bancarias vigentes por región del usuario logueado.
        /// La región se extrae del usuario autenticado — no se acepta como parámetro externo.
        /// </summary>
        public QueryResult<vEmpresaDatosBancarios> Lista(QueryInfo info, Guid idRegion)
        {
            using (var db = new ProquifaDotNetEntities())
            {
                var query = db.vEmpresaDatosBancarios
                    .Where(v => v.IdRegion == idRegion && v.Activo)
                    .AsQueryable();

                var total = query.Count();
                var items = query
                    .OrderBy(v => v.EmpresaPrefijo)
                    .Skip(info.Skip)
                    .Take(info.Take)
                    .ToList();

                return new QueryResult<vEmpresaDatosBancarios>
                {
                    Total   = total,
                    Results = items
                };
            }
        }
    }
}
```

**Nuevo archivo:** `WebApi.Catalogos\Controllers\Configuracion\Empresas\vEmpresaDatosBancariosController.cs`

```csharp
using System;
using System.Net.Http;
using System.Web.Http;
using System.Web.Http.Description;
using Core.CrudTools.Optimization;
using Core.Pqf.ProquifaDotNetContext;
using Core.Pqf.WebApi.ControllerOperations;
using Logic.Pqf.Catalogos.Empresas.DatosBancarios;

namespace WebApi.Controllers.Configuracion.Empresas
{
    public class vEmpresaDatosBancariosController : BaseApiController
    {
        /// <summary>
        /// Obtiene el detalle de una cuenta bancaria vigente por su Id.
        /// GET /vEmpresaDatosBancarios?idEmpresaDatosBancarios={id}
        /// </summary>
        [HttpGet]
        [Route("vEmpresaDatosBancarios")]
        [ResponseType(typeof(vEmpresaDatosBancarios))]
        public HttpResponseMessage Obtener(Guid idEmpresaDatosBancarios)
        {
            return TryExecute(() =>
            {
                var bo     = new vEmpresaDatosBancariosBO();
                var result = bo.ObtenerPorId(idEmpresaDatosBancarios);
                return result != null
                    ? Request.CreateResponse(System.Net.HttpStatusCode.OK, result)
                    : Request.CreateResponse(System.Net.HttpStatusCode.NoContent);
            });
        }

        /// <summary>
        /// Listado paginado de cuentas bancarias vigentes por región del usuario logueado.
        /// La región se extrae del token — no se acepta como parámetro externo.
        /// POST /vEmpresaDatosBancarios
        /// Body: QueryInfo
        /// </summary>
        [HttpPost]
        [Route("vEmpresaDatosBancarios")]
        [ResponseType(typeof(QueryResult<vEmpresaDatosBancarios>))]
        public HttpResponseMessage QueryResult([FromBody] QueryInfo info)
        {
            return TryExecute(() =>
            {
                var usuario  = ObtenerUsuarioAutenticado();   // token — región del usuario
                var idRegion = usuario.IdRegion;

                var bo     = new vEmpresaDatosBancariosBO();
                var result = bo.Lista(info, idRegion);
                return result != null
                    ? Request.CreateResponse(System.Net.HttpStatusCode.OK, result)
                    : Request.CreateResponse(System.Net.HttpStatusCode.NoContent);
            });
        }
    }
}
```

---

## 4. Tablas y entidades del modelo de datos

| Tabla / Vista | Entidad EF | Propiedad clave | Descripción |
|---------------|------------|-----------------|-------------|
| `EmpresaDatosBancarios` | `EmpresaDatosBancarios` | `IdEmpresaDatosBancarios` (PK), `Activo` (bit) | Catálogo principal. `Activo = 1` = vigente. **`IdRegion` (FK) — NUEVO R16**. |
| `vEmpresaDatosBancarios` | `vEmpresaDatosBancarios` | `IdEmpresaDatosBancarios` | **NUEVA R16** — Vista operativa con JOIN completo: empresa, banco, moneda, región. Módulos deben consumir esta vista. |
| `EmpresaRegion` | `EmpresaRegion` | `IdEmpresa` (FK), `IdRegion` (FK), `Activo` (bit) | Vincula empresa con región. Existente — sin cambios en R16. |
| `DatosBancarios` | `DatosBancarios` | `IdDatosBancarios` (PK) | Número de cuenta, CLABE, banco, moneda. |
| `Empresa` | `Empresa` | `IdEmpresa` (PK), `Prefijo` varchar | GOL / MUN / PRO / PQF. |
| `Region` | `Region` | `IdRegion` (PK), `Clave` varchar | MEX / PER. |
| `catBanco` | `catBanco` | `IdCatBanco` (PK) | Institución bancaria. |
| `catMoneda` | `catMoneda` | `IdCatMoneda` (PK) | MXN / USD. |

---

## 5. Consulta de referencia — uso correcto de la vista

```sql
-- Los módulos SIEMPRE deben consumir la vista, nunca la tabla directamente
SELECT *
FROM  dbo.vEmpresaDatosBancarios
WHERE IdRegion = @IdRegion   -- extraído del usuario logueado
  AND Activo   = 1
ORDER BY EmpresaPrefijo, NumeroDeCuenta;
```

---

## 6. Resumen de archivos a modificar / crear

| # | Archivo | Tipo de cambio |
|---|---------|----------------|
| 1 | `Scripts\R16\FU-001_EmpresaDatosBancarios_IdRegion.sql` | **Nuevo** — ALTER TABLE + UPDATE inicial de `IdRegion` |
| 2 | `Scripts\R16\FU-001_vEmpresaDatosBancarios.sql` | **Nuevo** — CREATE VIEW (requiere Paso 1 completado) |
| 3 | `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs` | **Modificar** — `ObtenerTodos()` agrega filtro `Activo = 1` |
| 4 | `Logic.Pqf.Catalogos\Empresas\DatosBancarios\vEmpresaDatosBancariosBO.cs` | **Nuevo** — BO con `ObtenerPorId` y `Lista` sobre la vista |
| 5 | `WebApi.Catalogos\Controllers\Configuracion\Empresas\vEmpresaDatosBancariosController.cs` | **Nuevo** — GET `/vEmpresaDatosBancarios` y POST `/vEmpresaDatosBancarios` |

---

## 7. Archivos que NO requieren cambio en R16

| Archivo | Motivo |
|---------|--------|
| `EmpresaDatosBancariosDetalleBO.cs` / `EmpresaDatosBancariosDetalle.cs` | Existente para uso interno — la vista reemplaza su uso en módulos consumidores |
| `DatosBancariosBO.Empresa.Extensions.cs` | Ya filtra cuentas vigentes correctamente |
| `EmpresaDatosBancariosController.cs` (PUT/DELETE) | Disponibles solo para Soporte a la Producción — sin cambios |

---

## 8. Pendientes / Decisiones abiertas

| #   | Pendiente                                                                                                           | Responsable                        |
| --- | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| P1  | Confirmar si el modelo EF (`ProquifaDotNetEntities`) se actualiza por scaffolding o manualmente tras el ALTER TABLE | TechLead                           |
| P2  | Confirmar entrega de cuentas reales (banco, número, CLABE, moneda) por PROQUIFA para carga inicial en BD            | PROQUIFA / Soporte a la Producción |
| P3  | Catálogo Perú (GOLPERU) fuera de alcance R16 — definir en release posterior                                         | Product Owner                      |
| P4  | Decisión sobre restricción formal (autorización) de endpoints PUT/DELETE de `EmpresaDatosBancariosController`       | Arquitectura / TechLead            |

---

## 9. Criterios de aceptación técnica

- [ ] `EmpresaDatosBancarios` tiene columna `IdRegion` (FK → `Region`) y todos los registros existentes tienen `IdRegion` poblado.
- [ ] La vista `vEmpresaDatosBancarios` existe en BD y resuelve el JOIN completo: empresa, banco, moneda y región.
- [ ] `ObtenerTodos()` retorna únicamente cuentas con `Activo = 1`. No retorna cuentas no vigentes.
- [ ] `GET /vEmpresaDatosBancarios?idEmpresaDatosBancarios={id}` retorna el detalle de una cuenta vigente o `204 NoContent`.
- [ ] `POST /vEmpresaDatosBancarios` retorna listado paginado de cuentas vigentes filtradas por la región del usuario logueado.
- [ ] La región se extrae de `ObtenerUsuarioAutenticado().IdRegion` — no se acepta como parámetro externo.
- [ ] Un usuario MEX solo obtiene cuentas de empresas GOL/MUN/PRO/PQF. Un usuario PER obtiene cuentas de su región.
- [ ] Las cuentas no vigentes se conservan en BD para trazabilidad histórica (sin DELETE físico).
- [ ] No existe pantalla en PQF2 que permita al usuario final gestionar cuentas bancarias del grupo.
- [ ] Los proyectos `Logic.Pqf.Catalogos` y `WebApi.Catalogos` compilan sin errores.
- [ ] PR aprobado por líder técnico.
