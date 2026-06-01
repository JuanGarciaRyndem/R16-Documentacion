# TPSC-RE-FU-001-Back — Análisis de Impacto Backend
**Requisito:** TPSC-RE-FU-001 — Catálogo de Cuentas Bancarias del Grupo PROQUIFA  
**Proyecto:** ProquifaDotNet-R14  
**Branch:** develop-pack04  
**Fecha:** 2025-07-14  
**Revisión aplicada:** TPSC-RE-FU-001-Revision.md

---

## Cambios respecto a la versión anterior del Back

| # | Cambio aplicado | Origen |
|---|-----------------|--------|
| 1 | Mapeo de reglas actualizado: de 5 reglas técnicas (*cómo*) a 4 reglas declarativas (*qué*) | Revisión — Reglas reescritas como enunciados del *qué* |
| 2 | Terminología "Activo = 1 / cuentas vigentes" → **"existe / no existe vigente en sistema"** | Revisión — Criterios B1, B2 actualizados |
| 3 | GAP "Sin UI en R16" eliminado del mapeo de reglas de negocio: es criterio funcional, no regla de backend | Revisión — "Sin UI / gestión en BD" son criterios, no regla de negocio |
| 4 | Buzón de Pagos marcado como ⚠️ pendiente IA en la sección de pendientes | Revisión — Alcance Buzón de Pagos condicionado a propuesta IA |
| 5 | Criterios de aceptación técnica actualizados con terminología "existe/no existe" | Revisión — Evitar confusión entre estado técnico (`Activo`) y estado de negocio |
| 6 | Cabecera completada con rama, fecha y revisión aplicada | Mejora de trazabilidad |

---

## 1. Estructura actual en el proyecto

### 1.1 Capa de Lógica (`Logic.Pqf.Catalogos`)

| Archivo | Clase | Descripción |
|---------|-------|-------------|
| `Empresas\DatosBancarios\EmpresaDatosBancariosBO.cs` | `EmpresaDatosBancariosBO` | CRUD principal sobre `EmpresaDatosBancarios`. Filtros especiales por `IdCatMoneda` y `NumeroDeCuenta`. |
| `Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs` | `EmpresaDatosBancariosBO` (partial) | `ObtenerTodos()` — devuelve todas las cuentas **sin excluir las no vigentes**. |
| `Empresas\DatosBancarios\EmpresaDatosBancariosDetalleBO.cs` | `EmpresaDatosBancariosDetalleBO` | View-model que enriquece cada cuenta con `DatosBancarios`, `catBanco` y `catMoneda`. |
| `Empresas\DatosBancarios\Models\EmpresaDatosBancariosDetalle.cs` | `EmpresaDatosBancariosDetalle` | Modelo enriquecido: hereda de `EmpresaDatosBancarios` + `DatosBancarios` + `catBanco` + `catMoneda`. |
| `Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.cs` | `DatosBancariosBO` | CRUD de `DatosBancarios` (detalle bancario: número de cuenta, CLABE, banco, moneda). |
| `Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.Empresa.Extensions.cs` | `DatosBancariosBO` (partial) | `ObtenerEmpresaDeDatosBancarios(Guid)` — resuelve la empresa a partir de un `IdDatosBancarios`. Ya filtra cuentas vigentes. |
| `Empresas\Cuentas\CuentaEmpresaBO.cs` / `.Extensions.cs` | `CuentaEmpresaBO` | Vincula cuentas contables (`Cuenta`) con datos bancarios por empresa. |
| `Empresas\EmpresaBO.cs` | `EmpresaBO` | CRUD de `Empresa`. Columna única: `Prefijo` (GOL / MUN / PRO / PQF). |

### 1.2 Capa de API (`WebApi.Catalogos`)

| Archivo | Ruta HTTP | Métodos expuestos |
|---------|-----------|-------------------|
| `Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs` | `/EmpresaDatosBancarios` | GET Obtener, PUT GuardarOActualizar, POST QueryResult, DELETE Desactivar |
| `Controllers\Configuracion\Empresas\EmpresaDatosBancariosDetalleController.cs` | `/EmpresaDatosBancariosDetalle` | POST QueryResult, POST GrupoListaEmpresaDatosBancariosDetalle |

---

## 2. Mapeo de reglas de negocio al código

> Alineado con las **4 reglas declarativas** del requisito TPSC-RE-FU-001 tras la revisión.  
> La Regla anterior "Sin UI en R16" se retiró de este mapeo: es un criterio de aceptación funcional, no una regla de negocio con impacto directo en el backend.

| Regla del Requisito | Descripción de negocio | Implementación esperada en backend | Estado actual |
|---------------------|------------------------|------------------------------------|---------------|
| **Regla 1** — Catálogo único | El sistema mantiene un único catálogo de cuentas bancarias del grupo como fuente de verdad. | `EmpresaDatosBancarios` como tabla centralizada. Sin catálogos paralelos en otros módulos. | ✅ Existe la tabla y el BO |
| **Regla 2** — Estado de existencia vigente | Cada cuenta existe vigente en sistema o no existe vigente. Solo las vigentes se ofrecen a los módulos consumidores. Las no vigentes se conservan para trazabilidad histórica. | Los métodos de consulta devuelven únicamente cuentas vigentes (`Activo = 1`). Las no vigentes (`Activo = 0`) son accesibles para trazabilidad pero no se exponen en listados operativos. | ⚠️ `ObtenerTodos()` NO excluye cuentas no vigentes |
| **Regla 3** — Gestión sin interfaz gráfica | La gestión del catálogo (alta, baja, modificación) no tiene UI en R16. Responsabilidad de Soporte a la Producción vía acceso directo a BD. | Los endpoints de escritura (PUT/DELETE) existen pero no son accesibles desde ninguna pantalla de PQF2. | ✅ No existe UI que los consuma |
| **Regla 4** — Consumo centralizado desde módulos | Cualquier módulo que requiera cuentas bancarias del grupo consulta este catálogo. Sin fuentes paralelas. Solo empresas México en R16. | Filtrar `Empresa.Prefijo IN ('GOL','MUN','PRO','PQF')` en las consultas de módulos. Excluir Perú. | ⚠️ No existe filtro explícito de prefijos en las consultas de módulos |

---

## 3. Gaps identificados y cambios requeridos

### GAP-01 — `ObtenerTodos()` devuelve cuentas que no existen vigentes
**Archivo:** `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs`  
**Impacto:** Criterio B1 — Los módulos consumidores reciben cuentas que no existen vigentes en sistema.  
**Cambio requerido:** Excluir las cuentas no vigentes del resultado.

```csharp
// ANTES — devuelve vigentes y no vigentes
public List<EmpresaDatosBancarios> ObtenerTodos()
{
    using (var db = new ProquifaDotNetEntities())
    {
        return db.EmpresaDatosBancarios.ToList();
    }
}

// DESPUÉS — devuelve únicamente cuentas que existen vigentes en sistema
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
**Impacto:** Criterio A2 — Los módulos que solicitan cuentas de una empresa emisora específica no tienen un método directo que garantice vigencia y alcance R16.  
**Cambio requerido:** Agregar método que devuelva solo cuentas vigentes de una empresa del grupo, excluyendo Perú.

```csharp
/// <summary>
/// Devuelve las cuentas bancarias que existen vigentes en sistema
/// para una empresa del grupo PROQUIFA México.
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

### GAP-03 — No existe endpoint de consulta por empresa emisora
**Archivo:** `WebApi.Catalogos\Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs`  
**Impacto:** Criterio A2 — Los módulos consumidores necesitan un endpoint para obtener las cuentas vigentes filtradas por empresa emisora.  
**Cambio requerido:** Agregar endpoint `GET /EmpresaDatosBancarios/Vigentes/{prefijo}`.

```csharp
/// <summary>
/// Obtiene las cuentas bancarias que existen vigentes en sistema
/// para la empresa del grupo indicada por su prefijo.
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

## 4. Tablas y entidades del modelo de datos

| Tabla BD | Entidad EF | Propiedad clave | Descripción |
|----------|------------|-----------------|-------------|
| `EmpresaDatosBancarios` | `EmpresaDatosBancarios` | `IdEmpresaDatosBancarios` (PK), `Activo` (bit) | Catálogo principal. `Activo = 1` = **existe vigente en sistema**; `Activo = 0` = **no existe vigente** (conservado para trazabilidad histórica). |
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
INNER JOIN dbo.Empresa         e   ON edb.IdEmpresa        = e.IdEmpresa
INNER JOIN dbo.DatosBancarios  db  ON edb.IdDatosBancarios = db.IdDatosBancarios
LEFT  JOIN dbo.catBanco        b   ON db.IdCatBanco        = b.IdCatBanco
LEFT  JOIN dbo.catMoneda       m   ON db.IdCatMoneda       = m.IdCatMoneda
WHERE edb.Activo = 1                              -- existe vigente en sistema
  AND e.Activo   = 1
  AND e.Prefijo IN ('GOL', 'MUN', 'PRO', 'PQF')  -- alcance R16: solo empresas México
ORDER BY e.Prefijo, db.NumeroDeCuenta;
```

---

## 6. Resumen de archivos a modificar

| # | Archivo | Tipo de cambio |
|---|---------|----------------|
| 1 | `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs` | Modificar `ObtenerTodos()` + agregar `ObtenerVigentesPorEmpresa()` |
| 2 | `WebApi.Catalogos\Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs` | Agregar endpoint `GET /EmpresaDatosBancarios/Vigentes/{prefijo}` |

---

## 7. Archivos que NO requieren cambio en R16

| Archivo | Motivo |
|---------|--------|
| `EmpresaDatosBancariosDetalleBO.cs` | View-model suficiente; el filtro de vigencia se aplica en la capa BO. |
| `EmpresaDatosBancariosDetalle.cs` | Modelo correcto, sin cambios. |
| `DatosBancariosBO.Empresa.Extensions.cs` | Ya filtra cuentas vigentes correctamente. |
| `EmpresaDatosBancariosController.cs` (PUT/DELETE) | Existen pero no son accesibles desde ninguna UI. Disponibles solo para Soporte a la Producción. |

---

## 8. Pendientes / Decisiones abiertas

| #   | Pendiente                                                                                                                                                                                             | Responsable                        |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| P1  | Decidir si los endpoints PUT/DELETE de `EmpresaDatosBancarios` se restringen formalmente (autorización) o se documentan como uso interno exclusivo de soporte.                                        | Arquitectura / TechLead            |
| P2  | ⚠️ **Buzón de Pagos** como consumidor del catálogo: pendiente confirmar si la propuesta de identificación automática de pagos con IA aplica en R16. Si no aplica, eliminar del alcance del requisito. | Product Owner                      |
| P3  | Catálogo Perú (Golocaer S.A.C.) fuera de alcance R16 — definir en release posterior con normativa SUNAT.                                                                                              | Product Owner                      |
| P4  | Entrega del detalle completo de cuentas (banco, número, CLABE, moneda) por parte de PROQUIFA para carga inicial en BD.                                                                                | PROQUIFA / Soporte a la Producción |

---

## 9. Criterios de aceptación técnica

- [ ] `ObtenerTodos()` retorna únicamente cuentas que **existen vigentes en sistema**. No retorna cuentas históricas (no vigentes).
- [ ] `ObtenerVigentesPorEmpresa(prefijo)` retorna cuentas vigentes del prefijo solicitado, excluyendo cualquier empresa fuera de `('GOL','MUN','PRO','PQF')`.
- [ ] El endpoint `GET /EmpresaDatosBancarios/Vigentes/{prefijo}` responde correctamente con cuentas filtradas por empresa y por estado de existencia vigente.
- [ ] Ninguna consulta de módulo consumidor devuelve cuentas que **no existen vigentes** en sistema.
- [ ] Las cuentas que dejan de existir vigentes se conservan en BD para trazabilidad histórica (sin DELETE físico).
- [ ] No existe pantalla en PQF2 que permita al usuario final agregar, modificar o dar de baja cuentas bancarias del grupo.
