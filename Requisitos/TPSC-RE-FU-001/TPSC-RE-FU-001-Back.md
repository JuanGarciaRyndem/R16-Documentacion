# Análisis de Impacto Backend
**Requisito:** TPSC-RE-FU-001 — Catálogo de Cuentas Bancarias del Grupo PROQUIFA  
**Proyecto:** ProquifaDotNet

---

## 1. Estructura actual en el proyecto

### 1.1 Capa de Lógica (`Logic.Pqf.Catalogos`)

| Archivo | Clase | Descripción |
|---------|-------|-------------|
| `Empresas\DatosBancarios\EmpresaDatosBancariosBO.cs` | `EmpresaDatosBancariosBO` | CRUD principal sobre `EmpresaDatosBancarios`. Filtros especiales por `IdCatMoneda` y `NumeroDeCuenta`. |
| `Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs` | `EmpresaDatosBancariosBO` (partial) | `ObtenerTodos()` — devuelve todas las cuentas **sin filtro de `Activo`**. |
| `Empresas\DatosBancarios\EmpresaDatosBancariosDetalleBO.cs` | `EmpresaDatosBancariosDetalleBO` | View-model que enriquece cada cuenta con `DatosBancarios`, `catBanco` y `catMoneda`. |
| `Empresas\DatosBancarios\Models\EmpresaDatosBancariosDetalle.cs` | `EmpresaDatosBancariosDetalle` | Modelo enriquecido: hereda de `EmpresaDatosBancarios` + `DatosBancarios` + `catBanco` + `catMoneda`. |
| `Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.cs` | `DatosBancariosBO` | CRUD de `DatosBancarios` (detalle bancario: número de cuenta, CLABE, banco, moneda). |
| `Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.Empresa.Extensions.cs` | `DatosBancariosBO` (partial) | `ObtenerEmpresaDeDatosBancarios(Guid)` — resuelve la empresa a partir de un `IdDatosBancarios`. Ya filtra `Activo`. |
| `Empresas\Cuentas\CuentaEmpresaBO.cs` / `.Extensions.cs` | `CuentaEmpresaBO` | Vincula cuentas contables (`Cuenta`) con datos bancarios por empresa. |
| `Empresas\EmpresaBO.cs` | `EmpresaBO` | CRUD de `Empresa`. Columna única: `Prefijo` (GOL / MUN / PRO / PQF). |

### 1.2 Capa de API (`WebApi.Catalogos`)

| Archivo | Ruta HTTP | Métodos expuestos |
|---------|-----------|-------------------|
| `Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs` | `/EmpresaDatosBancarios` | GET Obtener, PUT GuardarOActualizar, POST QueryResult, DELETE Desactivar |
| `Controllers\Configuracion\Empresas\EmpresaDatosBancariosDetalleController.cs` | `/EmpresaDatosBancariosDetalle` | POST QueryResult, POST GrupoListaEmpresaDatosBancariosDetalle |

---

## 2. Mapeo de reglas de negocio al código

| Regla del Requisito | Implementación esperada | Estado actual |
|---------------------|------------------------|---------------|
| **Regla 1** — Catálogo único (`EmpresaDatosBancarios`) | Tabla `EmpresaDatosBancarios` como fuente única. Sin catálogos paralelos. | ✅ Existe la tabla y el BO |
| **Regla 2** — `Activo = 1` para cuentas vigentes | Filtrar `WHERE Activo = 1` en todos los métodos de consulta consumidos por módulos | ⚠️ `ObtenerTodos()` NO filtra por `Activo` |
| **Regla 3** — Sin UI en R16 | Endpoints de escritura (PUT / DELETE) no deben ser accesibles desde UI | ⚠️ Existen `GuardarOActualizar` y `Desactivar` en el controller |
| **Regla 4** — Consumo desde módulos | Filtrar `Prefijo IN ('GOL','MUN','PRO','PQF')` y excluir Perú | ⚠️ No existe filtro de prefijos en las consultas de módulos |

---

## 3. Gaps identificados y cambios requeridos

### GAP-01 — `ObtenerTodos()` no filtra por `Activo = 1`
**Archivo:** `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs`  
**Impacto:** Criterio B1 — Los módulos consumidores reciben cuentas inactivas (históricas).  
**Cambio requerido:** Agregar `WHERE Activo = 1` al método `ObtenerTodos()`.

```csharp
// ANTES (incorrecto para módulos consumidores)
public List<EmpresaDatosBancarios> ObtenerTodos()
{
    using (var db = new ProquifaDotNetEntities())
    {
        return db.EmpresaDatosBancarios.ToList();
    }
}

// DESPUÉS
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

### GAP-02 — No existe método `ObtenerVigentesPorEmpresa`
**Archivo:** `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs`  
**Impacto:** Criterio A2 — Los módulos que necesitan filtrar por empresa emisora no tienen un método directo.  
**Cambio requerido:** Agregar método que filtre por `Prefijo` de empresa y por `Activo = 1`.

```csharp
/// <summary>
/// Devuelve las cuentas bancarias vigentes de una empresa del grupo PROQUIFA México.
/// Prefijos válidos en R16: GOL, MUN, PRO, PQF. No incluye Perú (GOLPERU).
/// </summary>
public List<EmpresaDatosBancarios> ObtenerVigentesPorEmpresa(string prefijo)
{
    using (var db = new ProquifaDotNetEntities())
    {
        var prefijosValidos = new[] { "GOL", "MUN", "PRO", "PQF" };
        var empresa = db.Empresa.FirstOrDefault(e =>
            e.Prefijo == prefijo && prefijosValidos.Contains(e.Prefijo) && e.Activo);

        if (empresa == null)
            return new List<EmpresaDatosBancarios>();

        return db.EmpresaDatosBancarios
            .Where(x => x.IdEmpresa == empresa.IdEmpresa && x.Activo)
            .ToList();
    }
}
```

---

### GAP-03 — No existe endpoint de consulta por empresa en el controller
**Archivo:** `WebApi.Catalogos\Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs`  
**Impacto:** Criterio A2 — Los módulos consumidores necesitan un endpoint para obtener las cuentas vigentes filtradas por empresa emisora.  
**Cambio requerido:** Agregar endpoint `GET /EmpresaDatosBancarios/Vigentes/{prefijo}`.

```csharp
/// <summary>
/// Obtener cuentas bancarias vigentes del grupo filtradas por prefijo de empresa.
/// Prefijos válidos R16: GOL, MUN, PRO, PQF.
/// </summary>
[HttpGet]
[Route("EmpresaDatosBancarios/Vigentes/{prefijo}")]
public List<EmpresaDatosBancarios> ObtenerVigentesPorEmpresa(string prefijo)
{
    var bo = new EmpresaDatosBancariosBO();
    return bo.ObtenerVigentesPorEmpresa(prefijo);
}
```

---

### GAP-04 — Endpoints de escritura expuestos sin restricción de alcance R16
**Archivo:** `WebApi.Catalogos\Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs`  
**Impacto:** Criterio C1 / Regla 3 — El requisito indica que no habrá UI para gestión en R16. Los endpoints `PUT GuardarOActualizar` y `DELETE Desactivar` existen pero deben quedar restringidos a Soporte a la Producción (sin UI que los consuma).  
**Cambio requerido:** Documentar en Swagger que estos endpoints son de uso exclusivo de Soporte a la Producción. No eliminar (pueden ser usados vía herramientas de administración internas), pero no exponer en ninguna pantalla de PQF2.

> **Nota:** Si el equipo decide deshabilitar estos endpoints en R16, se puede agregar `[ApiExplorerSettings(IgnoreApi = true)]` o una política de autorización restrictiva. Pendiente decisión de arquitectura.

---

### GAP-05 — Consultas SQL de módulos deben filtrar prefijos México
**Impacto:** Regla 4 / Criterio A1 — Al consultar cuentas bancarias, ningún módulo debe incluir registros de empresas fuera del alcance R16 (Perú).  
**Cambio requerido:** En cualquier join entre `EmpresaDatosBancarios` y `Empresa`, agregar:

```csharp
// En cualquier consulta LINQ que cruce con Empresa
.Where(e => new[] { "GOL", "MUN", "PRO", "PQF" }.Contains(e.Prefijo))
```

```sql
-- En cualquier consulta SQL directa
AND e.Prefijo IN ('GOL', 'MUN', 'PRO', 'PQF')
```

---

## 4. Tablas y entidades del modelo de datos

| Tabla BD | Entidad EF | Propiedad clave | Descripción |
|----------|------------|-----------------|-------------|
| `EmpresaDatosBancarios` | `EmpresaDatosBancarios` | `IdEmpresaDatosBancarios` (PK), `Activo` (bit) | Catálogo principal. `Activo=1` vigente. |
| `DatosBancarios` | `DatosBancarios` | `IdDatosBancarios` (PK) | Número de cuenta, CLABE, banco, moneda. |
| `Empresa` | `Empresa` | `IdEmpresa` (PK), `Prefijo` varchar | GOL / MUN / PRO / PQF. |
| `catBanco` | `catBanco` | `IdCatBanco` (PK) | Institución bancaria. |
| `catMoneda` | `catMoneda` | `IdCatMoneda` (PK) | MXN / USD. |
| `CuentaEmpresa` | `CuentaEmpresa` | `IdCuentaEmpresa` (PK) | Vincula cuenta contable con datos bancarios. |

---

## 5. Consulta SQL de referencia — cuentas vigentes del grupo

```sql
SELECT
    e.Prefijo,
    e.Alias        AS Empresa,
    b.Banco,
    db.NumeroDeCuenta,
    db.Clabe,
    m.ClaveMoneda  AS Moneda,
    edb.FechaRegistro
FROM dbo.EmpresaDatosBancarios edb
INNER JOIN dbo.Empresa         e   ON edb.IdEmpresa       = e.IdEmpresa
INNER JOIN dbo.DatosBancarios  db  ON edb.IdDatosBancarios = db.IdDatosBancarios
LEFT  JOIN dbo.catBanco        b   ON db.IdCatBanco        = b.IdCatBanco
LEFT  JOIN dbo.catMoneda       m   ON db.IdCatMoneda       = m.IdCatMoneda
WHERE edb.Activo = 1
  AND e.Activo   = 1
  AND e.Prefijo IN ('GOL', 'MUN', 'PRO', 'PQF')
ORDER BY e.Prefijo, db.NumeroDeCuenta;
```

---

## 6. Resumen de archivos a modificar

| # | Archivo | Tipo de cambio |
|---|---------|---------------|
| 1 | `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs` | Modificar `ObtenerTodos()` + agregar `ObtenerVigentesPorEmpresa()` |
| 2 | `WebApi.Catalogos\Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs` | Agregar endpoint `GET /EmpresaDatosBancarios/Vigentes/{prefijo}` |

---

## 7. Archivos que NO requieren cambio en R16

| Archivo | Motivo |
|---------|--------|
| `EmpresaDatosBancariosDetalleBO.cs` | El view-model es suficiente; el filtro se aplica en la capa BO. |
| `EmpresaDatosBancariosDetalle.cs` | Modelo correcto, sin cambios. |
| `DatosBancariosBO.Empresa.Extensions.cs` | Ya filtra `Activo` correctamente. |
| `EmpresaDatosBancariosController.cs` (PUT/DELETE) | Existen pero no se consumen desde UI. Quedan restringidos a Soporte a la Producción. |

---

## 8. Pendientes / Decisiones abiertas

| #   | Pendiente                                                                                                                                 | Responsable                        |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| P1  | Decidir si los endpoints PUT/DELETE de `EmpresaDatosBancarios` se deshabilitan o solo se documentan como internos en R16.                 | Arquitectura / TechLead            |
| P2  | Confirmar si el Buzón de Pagos consumirá el catálogo en R16 (propuesta IA pendiente). Si aplica, agregar endpoint de consulta específico. | Product Owner                      |
| P3  | Catálogo Perú (Golocaer S.A.C.) fuera de alcance R16 — definir en release posterior con normativa SUNAT.                                  | Product Owner                      |
| P4  | Entrega del detalle completo de cuentas (banco, número, CLABE, moneda) por parte de PROQUIFA para carga inicial en BD.                    | PROQUIFA / Soporte a la Producción |

---

## 9. Criterios de aceptación técnica

- [ ] `ObtenerTodos()` retorna únicamente cuentas con `Activo = true`.
- [ ] `ObtenerVigentesPorEmpresa(prefijo)` retorna cuentas vigentes del prefijo solicitado, excluyendo cualquier empresa fuera de `('GOL','MUN','PRO','PQF')`.
- [ ] El endpoint `GET /EmpresaDatosBancarios/Vigentes/{prefijo}` responde correctamente con cuentas filtradas.
- [ ] Ninguna consulta de módulo consumidor devuelve cuentas con `Activo = false`.
- [ ] No existe pantalla en PQF2 que exponga alta/baja/modificación de cuentas bancarias del grupo.
- [ ] Los tests de integración validan el filtro `Activo` y el filtro de prefijos México.
