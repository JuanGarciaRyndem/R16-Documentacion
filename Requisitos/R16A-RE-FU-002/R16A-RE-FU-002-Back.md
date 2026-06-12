# R16A-RE-FU-002-Back — Análisis de Impacto Backend
**Requisito:** R16A-RE-FU-002 — Asignación de Cobrador a Cliente (Catálogo de Clientes)
**Proyecto:** ProquifaDotNet-R14
**Branch:** develop-pack04
**Fecha:** 2025-07-14
**Revisión aplicada:** R16A-RE-FU-002-Revision.md

---

## Cambios respecto a la versión anterior del Back

| # | Cambio aplicado | Origen |
|---|-----------------|--------|
| 1 | Cabecera completada con branch, fecha y revisión aplicada | Mejora de trazabilidad |
| 2 | Mapeo de reglas: las 5 reglas ya declarativas se mantienen; se refuerza **Regla 4** con la decisión tomada de filtrado dinámico | Revisión — decisión sobre redistribución inmediata documentada |
| 3 | Referencias de criterios actualizadas a las 3 secciones definitivas: A (Visibilidad/edición), B (Selector), C (Filtrado bandeja) | Revisión — criterios reorganizados |
| 4 | GAP-05 nuevo: exclusión del campo Cobrador en el alta de cliente desde **Cotizar lo Cotizable** | Revisión — agregado en Alcance `No aplica a` del requisito |
| 5 | Criterios de aceptación técnica actualizados con referencias a Criterios C2 y C3 (redistribución dinámica y cliente sin cobrador) | Revisión — Criterio 7 anterior dividido en C2 + C3 |
| 6 | **Regla 1 y GAP-04 ampliados:** Gerente de Tesorería también puede asignar Cobradores (además del Coordinador de Tesorería) | OBS-003 |
| 7 | **Módulos consumidores:** se agrega Notas de Crédito al alcance de visibilidad por cartera | OBS-004 |
| 8 | **Regla 4 matizada:** redistribución por pantalla/módulo — solo pendientes abiertos al nuevo cobrador; trabajo previo se conserva. Nuevo **Criterio C4** | OBS-005 |

---

## 1. Resumen funcional

El requisito habilita el campo **"Cobrador"** en la sección Datos Generales del Catálogo de Clientes.

- El rol **Coordinador de Tesorería** O **Gerente de Tesorería** puede editar el campo Cobrador (OBS-003 — pendiente confirmar campo exacto en `Usuario` para Gerente de Tesorería).
- El selector muestra únicamente usuarios con rol **Gestor de Cobranza** (`AnalistaDeCuentasPorCobrar = 1`) activos.
- La asignación se persiste en `ClienteCartera.IdUsuarioCobrador`.
- El filtrado de bandeja es **dinámico por pantalla/módulo** (OBS-005): al reasignar el Cobrador, solo los pendientes **abiertos** (aún aparecen en su pantalla sin finalizar) pasan al nuevo Cobrador. El trabajo ya completado por el cobrador anterior permanece registrado donde fue ejecutado — no se reasigna ni se pierde (Regla 4 / Criterio C4). Un pendiente con avances parciales guardados (seguimiento registrado, un paso del wizard guardado) **no** se considera "trabajado" — sigue siendo reasignable.
- El campo **no aplica** en el alta de cliente desde **Cotizar lo Cotizable** (ese flujo es solo para habilitar cotización, no para gestión del cliente).
- Módulos consumidores: **Validar Cobro**, **Factura por Adelantado**, **Buzón de Pagos** y **Notas de Crédito** (OBS-004).

---

## 2. Estructura actual en el proyecto

### 2.1 Capa de Lógica (`Logic.Pqf.Catalogos`)

| Archivo | Clase | Descripción relevante |
|---------|-------|-----------------------|
| `Clientes\Carteras\ClienteCarteraBO.cs` | `ClienteCarteraBO` | CRUD sobre `ClienteCartera`. Sin lógica de cobrador propia. |
| `Clientes\Carteras\ClienteCarteraClienteBO.cs` | `ClienteCarteraClienteBO` | Vinculación cliente–cartera. `_GuardarOActualizar` con columnas únicas `IdCliente` + `IdClienteCartera`. |
| `Clientes\Carteras\ClienteCarteraClienteBO.Extensions.cs` | `ClienteCarteraClienteBO` (partial) | `ObtenerIDUsuarioESACPorCliente()` — navega `ClienteCarteraCliente → ClienteCartera`. Patrón reutilizable para cobrador. |
| `Clientes\ClienteBO.cs` | `ClienteBO` | En la creación de cliente ya maneja `IdUsuarioCobrador`: busca la cartera del cobrador y vincula el cliente a esa cartera. |
| `Clientes\ClienteBO.Extensions.cs` | `ClienteBO` (partial) | `ObtenerUsuariosCobrosAsignados(Guid idCliente)` — devuelve `vUsuario` activos asignados como cobradores del cliente via `ClienteCartera.IdUsuarioCobrador`. |
| `Usuarios\GMUsuarioClienteCarteraDetalleBO.cs` | `GMUsuarioClienteCarteraDetalleBO` | `Obtener(Guid IdUsuario)` — construye `CarteraUsuario` incluyendo `ClientesCarteraCobrador`. `ProcesarGMCarteraClienteUsuario()` — procesa asignación masiva. |
| `Usuarios\Models\GMUsuarioClienteCarteraDetalle.cs` | `CarteraUsuario` | Contiene `ClientesCarteraCobrador` (List<ClienteCarteraDatos>). El modelo ya soporta la cartera de cobrador. |
| `Usuarios\Models\GMCarteraClienteUsuario.cs` | `GMCarteraClienteUsuario` | DTO para procesar asignación de clientes a una cartera de usuario. |

### 2.2 Capa de API (`WebApi.Catalogos`)

| Archivo | Ruta HTTP | Métodos expuestos |
|---------|-----------|-------------------|
| `Controllers\Configuracion\Clientes\Cartera\ClienteCarteraController.cs` | `/ClienteCartera` | GET Obtener, PUT GuardarOActualizar, POST QueryResult, DELETE Desactivar |
| `Controllers\Sistema\Usuarios\UsuarioController.cs` | `/Usuario/GMUsuarioClienteCarteraDetalle` | GET — cartera completa del usuario (incluye `ClientesCarteraCobrador`) |
| `Controllers\Sistema\Usuarios\UsuarioController.cs` | `/Usuario/ClienteCarteraCliente` | POST — asigna lista de clientes a cartera |
| `Controllers\Sistema\Usuarios\UsuarioController.cs` | `/Usuario/ListaUsuariosCartera` | POST QueryInfo — lista usuarios con relaciones de cartera |
| `Controllers\Sistema\Usuarios\UsuarioController.cs` | `/Usuario` | POST QueryResult — lista paginada de usuarios |

---

## 3. Mapeo de reglas de negocio al código

> Las 5 reglas del requisito son declarativas (el *qué*). Alineadas con R16A-RE-FU-002 tras la revisión.

| Regla | Descripción de negocio | Implementación esperada | Estado actual |
|-------|------------------------|------------------------|---------------|
| **Regla 1** — Edición exclusiva del Coordinador de Tesorería **O Gerente de Tesorería** (OBS-003) | `GerenteDeTesoreria = true` (Coordinador) O el campo que mapee al Gerente de Tesorería (pendiente confirmar) puede editar el campo Cobrador. | Validar ambos roles en el endpoint de reasignación. Campo de solo lectura para otros roles. | ⚠️ No existe validación de rol en la operación de actualización de `IdUsuarioCobrador` |
| **Regla 2** — Cobrador debe ser Gestor de Cobranza activo | Solo usuarios con `AnalistaDeCuentasPorCobrar = 1 AND Activo = 1` son asignables. | Método/endpoint que filtre el selector a Gestores de Cobranza activos. | ⚠️ No existe endpoint específico de solo Gestores de Cobranza |
| **Regla 3** — Un solo Cobrador por cliente | Un único `IdUsuarioCobrador` por cartera en un momento dado. La asignación nueva reemplaza la anterior. | `ClienteCartera.IdUsuarioCobrador` es campo único por cartera. `ReasignarCobrador` sobreescribe el valor. | ✅ La estructura de BD lo garantiza |
| **Regla 4** — Filtrado dinámico por cobrador actual (OBS-005) | **Al reasignar el Cobrador, solo los pendientes aún abiertos (que aún aparecen en la pantalla del cobrador sin haber sido finalizados) pasan al nuevo Cobrador.** El trabajo completado por el cobrador anterior permanece registrado donde fue ejecutado — no se reasigna ni se elimina (Criterio C4). | El filtro en módulos usa JOIN a `ClienteCartera.IdUsuarioCobrador` en tiempo de consulta — el cambio de `IdUsuarioCobrador` surte efecto en la siguiente consulta de bandeja de cada pantalla/módulo. No se requiere migración de registros. | ✅ El JOIN dinámico ya implementa este comportamiento por pantalla/módulo. Solo requiere que `ReasignarCobrador` actualice `IdUsuarioCobrador` correctamente. |
| **Regla 5** — Cliente sin Cobrador invisible en bandejas | Los pendientes y pagos de clientes sin cobrador se registran pero no aparecen en ninguna bandeja. | El JOIN natural de módulos excluye registros donde `IdUsuarioCobrador IS NULL`. | ✅ El JOIN natural ya excluye estos casos |

---

## 4. Gaps identificados y cambios requeridos

### GAP-01 — No existe endpoint de selector de Gestores de Cobranza activos
**Impacto:** Criterio B1 — El frontend necesita poblar el selector del campo Cobrador con usuarios `AnalistaDeCuentasPorCobrar = 1 AND Activo = 1`.
**Archivo a modificar:** `Logic.Pqf.Catalogos\Usuarios\UsuariosBO.Extensions.cs`

`csharp
/// <summary>
/// Devuelve los usuarios activos con rol Gestor de Cobranza,
/// para poblar el selector del campo Cobrador en el Catálogo de Clientes.
/// </summary>
public List<Usuario> ObtenerGestoresDeCobranzaActivos()
{
    using (var db = new ProquifaDotNetEntities())
    {
        return db.Usuario
            .Where(u => u.AnalistaDeCuentasPorCobrar == true && u.Activo)
            .OrderBy(u => u.NombreCompleto)
            .ToList();
    }
}
`

**Endpoint a agregar:** `WebApi.Catalogos\Controllers\Sistema\Usuarios\UsuarioController.cs`

`csharp
/// <summary>
/// Devuelve usuarios activos con rol Gestor de Cobranza.
/// Usado para poblar el selector del campo Cobrador en el Catálogo de Clientes.
/// </summary>
[HttpGet]
[Route("Usuario/GestoresDeCobranza")]
[ResponseType(typeof(List<Usuario>))]
public HttpResponseMessage ObtenerGestoresDeCobranzaActivos()
{
    return TryExecute(() =>
    {
        var usuarioBO = new UsuarioBO();
        var results = usuarioBO.ObtenerGestoresDeCobranzaActivos();
        return Request.CreateResponse(HttpStatusCode.OK, results);
    });
}
`

---

### GAP-02 — No existe método para obtener el Cobrador actual de un cliente
**Impacto:** Criterio A1 — El Catálogo de Clientes debe mostrar el Cobrador actualmente asignado al cliente en Datos Generales.
**Archivo a modificar:** `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraClienteBO.Extensions.cs`
**Patrón de referencia:** `ObtenerIDUsuarioESACPorCliente()` ya existente en el mismo archivo.

`csharp
/// <summary>
/// Obtiene el IdUsuarioCobrador (Gestor de Cobranza) asignado a un cliente
/// mediante su ClienteCarteraCliente activa.
/// Navega: ClienteCarteraCliente → ClienteCartera → IdUsuarioCobrador.
/// </summary>
public static Guid? ObtenerIdUsuarioCobradorPorCliente(Guid idCliente)
{
    try
    {
        using (var db = new ProquifaDotNetEntities())
        {
            Guid? idUsuarioCobrador = db.ClienteCartera
                .Join(
                    db.ClienteCarteraCliente.Where(ccc => ccc.IdCliente == idCliente && ccc.Activo),
                    cc => cc.IdClienteCartera,
                    ccc => ccc.IdClienteCartera,
                    (cc, ccc) => cc)
                .Where(cc => cc.Activo)
                .Select(cc => cc.IdUsuarioCobrador)
                .FirstOrDefault();

            Logger.DebugFormat("IdUsuarioCobrador {0} obtenido para el cliente {1}", idUsuarioCobrador, idCliente);
            return idUsuarioCobrador;
        }
    }
    catch (Exception e)
    {
        Logger.FatalFormat("{0}", e);
        throw new InvalidOperationException("Error al obtener el IdUsuarioCobrador del cliente.", e);
    }
}
`

---

### GAP-03 — No existe método de reasignación explícita de Cobrador
**Impacto:** Criterios B2 / B3 / C2 — El Coordinador de Tesorería debe poder cambiar el Cobrador. El cambio redistribuye dinámicamente las bandejas (Regla 4).
**Archivo a modificar:** `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraBO.cs`

> **Decisión de diseño (Regla 4):** Al actualizar `IdUsuarioCobrador` en `ClienteCartera`, el filtrado de bandeja en los módulos consumidores automáticamente refleja el cambio en la siguiente consulta — no se requiere migración de registros de pendientes o pagos. El filtro es JOIN dinámico sobre el valor actual de `IdUsuarioCobrador`.

`csharp
/// <summary>
/// Reasigna el Cobrador (Gestor de Cobranza) en la cartera activa de un cliente.
/// Al actualizar IdUsuarioCobrador, los pendientes aún abiertos en cada pantalla/módulo
/// aparecerán en la bandeja del nuevo Cobrador en la siguiente consulta (Regla 4 / Criterio C2).
/// El trabajo ya completado por el cobrador anterior no se reasigna (Criterio C4).
/// </summary>
public void ReasignarCobrador(Guid idCliente, Guid idNuevoCobrador)
{
    using (var db = new ProquifaDotNetEntities())
    {
        var cobrador = db.Usuario.FirstOrDefault(u =>
            u.IdUsuario == idNuevoCobrador
            && u.AnalistaDeCuentasPorCobrar == true
            && u.Activo);

        if (cobrador == null)
            throw new ArgumentException("El usuario indicado no es un Gestor de Cobranza activo.");

        var cartera = db.ClienteCartera
            .Join(db.ClienteCarteraCliente.Where(ccc => ccc.IdCliente == idCliente && ccc.Activo),
                cc => cc.IdClienteCartera,
                ccc => ccc.IdClienteCartera,
                (cc, ccc) => cc)
            .FirstOrDefault(cc => cc.Activo);

        if (cartera == null)
            throw new InvalidOperationException("El cliente no tiene una cartera activa.");

        cartera.IdUsuarioCobrador = idNuevoCobrador;
        cartera.FechaUltimaActualizacion = DateTime.Now;
        db.SaveChanges();

        Logger.DebugFormat("Cobrador reasignado: cliente {0} → nuevo cobrador {1}", idCliente, idNuevoCobrador);
    }
}
`

**Endpoint a agregar:** `WebApi.Catalogos\Controllers\Configuracion\Clientes\Cartera\ClienteCarteraController.cs`

`csharp
/// <summary>
/// Reasigna el Cobrador (Gestor de Cobranza) de un cliente.
/// Solo debe ser invocado por usuarios con rol Coordinador de Tesorería (Regla 1).
/// Al reasignar, los pendientes abiertos del cliente pasan al nuevo Cobrador
/// en la siguiente consulta de cada pantalla/módulo (Regla 4 / Criterio C2).
/// El trabajo completado por el cobrador anterior no se reasigna (Criterio C4).
/// </summary>
[HttpPut]
[Route("ClienteCartera/ReasignarCobrador")]
public IHttpActionResult ReasignarCobrador(Guid idCliente, Guid idNuevoCobrador)
{
    var bo = new ClienteCarteraBO();
    bo.ReasignarCobrador(idCliente, idNuevoCobrador);
    return Ok();
}
`

---

### GAP-04 — Validación de rol Coordinador de Tesorería / Gerente de Tesorería no implementada (OBS-003)
**Impacto:** Criterios A2 / A3 — Solo usuarios con rol **Coordinador de Tesorería** (`GerenteDeTesoreria = true`) **O Gerente de Tesorería** (campo a confirmar en `Usuario`) pueden editar el campo Cobrador y llamar el endpoint de reasignación.
**Estado actual:** No existe validación de rol en los endpoints de `ClienteCartera`.
**Cambio requerido:** El endpoint `ReasignarCobrador` (GAP-03) debe validar que el usuario solicitante tiene `GerenteDeTesoreria = true` o el campo correspondiente al Gerente de Tesorería.

> **Pendiente P2:** Definir si la validación va en la capa BO (pasando `IdUsuarioSolicitante` como parámetro) o en un filtro de autorización del controller.
>
> **Pendiente OBS-003:** Confirmar qué campo de `Usuario` mapea al rol **Gerente de Tesorería** antes de implementar esta validación.

---

### GAP-05 — Alta de cliente desde Cotizar lo Cotizable no debe incluir campo Cobrador
**Impacto:** Alcance "No aplica a" del requisito (nuevo tras la revisión) — El flujo de alta de cliente en Cotizar lo Cotizable está orientado exclusivamente a habilitar la cotización. El campo Cobrador **no debe aparecer** en ese flujo.
**Estado actual:** Verificar que el formulario de alta de cliente en Cotizar lo Cotizable no expone el campo Cobrador.
**Cambio requerido:** Confirmar que el endpoint/BO utilizado por Cotizar lo Cotizable para crear clientes no incluye ni valida `IdUsuarioCobrador`. Si lo incluye, eliminarlo del flujo.

---

## 5. Tablas y entidades del modelo de datos

| Tabla BD | Entidad EF | Propiedad clave | Descripción |
|----------|------------|-----------------|-------------|
| `ClienteCartera` | `ClienteCartera` | `IdClienteCartera` (PK), `IdUsuarioCobrador` (FK nullable) | Cartera del cliente. `IdUsuarioCobrador` = Gestor de Cobranza asignado. |
| `ClienteCarteraCliente` | `ClienteCarteraCliente` | `IdClienteCarteraCliente` (PK), `IdClienteCartera` (FK), `IdCliente` (FK) | Relación N:N entre cartera y cliente. |
| `Usuario` | `Usuario` | `IdUsuario` (PK), `AnalistaDeCuentasPorCobrar` (bit), `GerenteDeTesoreria` (bit), `Activo` (bit) | Gestor de Cobranza = `AnalistaDeCuentasPorCobrar = 1`. Coordinador de Tesorería = `GerenteDeTesoreria = 1`. |
| `vClienteCarteraCliente` | Vista | `IdUsuarioCobrador`, `Cobrador` | Vista operativa principal para bandejas. |
| `vUsuarioCartera` | Vista | `IdCliente`, `IdUsuario`, `Activo` | Vinculación directa usuario–cliente via cartera. Usada por módulos consumidores. |

---

## 6. Consultas SQL de referencia

### Selector — Gestores de Cobranza activos
`sql
SELECT IdUsuario, NombreCompleto
FROM dbo.Usuario
WHERE AnalistaDeCuentasPorCobrar = 1
  AND Activo = 1
ORDER BY NombreCompleto;
`

### Cobrador actual de un cliente
`sql
SELECT cc.IdUsuarioCobrador, u.NombreCompleto AS Cobrador
FROM dbo.ClienteCarteraCliente ccc
INNER JOIN dbo.ClienteCartera cc ON ccc.IdClienteCartera = cc.IdClienteCartera
LEFT  JOIN dbo.Usuario         u  ON cc.IdUsuarioCobrador = u.IdUsuario
WHERE ccc.IdCliente = @IdCliente
  AND ccc.Activo    = 1
  AND cc.Activo     = 1;
`

### Reasignar Cobrador (redistribuye bandeja dinámicamente — Regla 4 / Criterio C2)
`sql
UPDATE dbo.ClienteCartera
SET    IdUsuarioCobrador        = @IdNuevoCobrador,
       FechaUltimaActualizacion = GETDATE()
WHERE  IdClienteCartera = @IdClienteCartera;
`

### Bandeja por cobrador (módulos consumidores — filtrado dinámico)
`sql
SELECT v.IdCliente, v.Nombre AS Cliente, v.NombreCartera, v.Cobrador, v.NombreRegion
FROM dbo.vClienteCarteraCliente v
WHERE v.IdUsuarioCobrador = @IdUsuarioCobrador
ORDER BY v.Nombre;
`

### Clientes sin Cobrador — Criterio C3 (alerta operativa)
`sql
SELECT v.IdCliente, v.Nombre, v.NombreCartera
FROM dbo.vClienteCarteraCliente v
WHERE v.IdUsuarioCobrador IS NULL
ORDER BY v.Nombre;
`

---

## 7. Resumen de archivos a modificar / crear

| # | Archivo | Tipo de cambio |
|---|---------|----------------|
| 1 | `Logic.Pqf.Catalogos\Usuarios\UsuariosBO.Extensions.cs` | Agregar `ObtenerGestoresDeCobranzaActivos()` |
| 2 | `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraClienteBO.Extensions.cs` | Agregar `ObtenerIdUsuarioCobradorPorCliente(Guid idCliente)` |
| 3 | `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraBO.cs` | Agregar `ReasignarCobrador(Guid idCliente, Guid idNuevoCobrador)` |
| 4 | `WebApi.Catalogos\Controllers\Sistema\Usuarios\UsuarioController.cs` | Agregar endpoint `GET /Usuario/GestoresDeCobranza` |
| 5 | `WebApi.Catalogos\Controllers\Configuracion\Clientes\Cartera\ClienteCarteraController.cs` | Agregar endpoint `PUT /ClienteCartera/ReasignarCobrador` |

---

## 8. Archivos que NO requieren cambio en R16

| Archivo | Motivo |
|---------|--------|
| `GMUsuarioClienteCarteraDetalleBO.cs` | Ya soporta `ClientesCarteraCobrador` en la cartera del usuario. Sin cambio. |
| `GMUsuarioClienteCarteraDetalle.cs` | Modelo ya contiene `ClientesCarteraCobrador`. Sin cambio. |
| `ClienteCarteraClienteBO.cs` | CRUD base suficiente. El filtro de cobrador va en extensions. |
| `ClienteBO.cs` | La lógica de alta de cliente con `IdUsuarioCobrador` ya existe y es correcta. |
| `ClienteBO.Extensions.cs` | `ObtenerUsuariosCobrosAsignados()` ya implementa el patrón correcto. |

---

## 9. Módulos consumidores del campo Cobrador

| Módulo | Cómo consume el campo | Cambio requerido en R16 |
|--------|-----------------------|-------------------------|
| **Validar Cobro** | JOIN `vUsuarioCartera WHERE IdUsuario = @CobActual AND Activo = 1` | Verificar que el filtro esté implementado en la consulta de bandeja |
| **Factura por Adelantado** | JOIN `vUsuarioCartera WHERE IdUsuario = @CobActual AND Activo = 1` | Verificar que el filtro esté implementado en la consulta de bandeja |
| **Buzón de Pagos** | JOIN `vUsuarioCartera WHERE IdUsuario = @CobActual AND Activo = 1` | Verificar que el filtro esté implementado en la consulta de bandeja |
| **Notas de Crédito** | JOIN `vUsuarioCartera WHERE IdUsuario = @CobActual AND Activo = 1` (OBS-004) | Ver R16A-RE-FU-032 Criterio A5 (México) y R16A-RE-FU-033 Criterio A3 (Perú — pendiente de alcance) |

> Los cambios en los módulos consumidores se documentan en sus respectivos requisitos. Este documento cubre únicamente el Catálogo de Clientes y el campo Cobrador.

---

## 10. Pendientes / Decisiones abiertas

| # | Pendiente | Responsable |
|---|-----------|-------------|
| P1 | Confirmar que `AnalistaDeCuentasPorCobrar = true` es exactamente el rol **Gestor de Cobranza** requerido en R16. | TechLead / Funcional |
| P2 | Definir estrategia de validación de rol **Coordinador de Tesorería** (`GerenteDeTesoreria`): ¿en capa BO o filtro de autorización en controller? | Arquitectura |
| P3 | El SP `spDesactivarCarterasCliente` no retorna `IdUsuarioCobrador` en el SELECT de retorno. Si se requiere auditoría de reasignaciones, actualizar el SP. | Backend |
| P4 | Historial de asignaciones cobrador-cliente: fuera de alcance R16. Documentar como deuda técnica. | Product Owner |
| P5 | Verificar que el flujo de alta de cliente en **Cotizar lo Cotizable** no incluye ni valida `IdUsuarioCobrador` (GAP-05). | Backend |

---

## 11. Criterios de aceptación técnica

- [ ] `GET /Usuario/GestoresDeCobranza` retorna únicamente usuarios con `AnalistaDeCuentasPorCobrar = true AND Activo = true`.
- [ ] `ObtenerIdUsuarioCobradorPorCliente(idCliente)` retorna el `IdUsuarioCobrador` de la cartera activa del cliente, o `null` si no tiene asignación (Criterio A1).
- [ ] `PUT /ClienteCartera/ReasignarCobrador` valida que el nuevo cobrador es un Gestor de Cobranza activo antes de guardar (Criterio B1 / Regla 2).
- [ ] `PUT /ClienteCartera/ReasignarCobrador` actualiza `ClienteCartera.IdUsuarioCobrador` y `FechaUltimaActualizacion` (Criterio B2 / B3).
- [ ] Tras la reasignación, la consulta de bandeja del nuevo Cobrador incluye al cliente en los pendientes abiertos; la del Cobrador anterior ya no lo incluye — redistribución dinámica por pantalla/módulo sin migración de registros (Regla 4 / Criterio C2 — OBS-005).
- [ ] **Criterio C4 (OBS-005):** El trabajo completado por el cobrador anterior (registros finalizados en Validar Cobro, FxA, Buzón de Pagos, Notas de Crédito) permanece registrado bajo ese cobrador. No se reasigna ni se elimina. Solo los pendientes aún abiertos al momento de la reasignación pasan al nuevo cobrador.
- [ ] Clientes sin `IdUsuarioCobrador` no aparecen en la bandeja de ningún Gestor de Cobranza (Regla 5 / Criterio C3).
- [ ] Roles distintos de Coordinador de Tesorería y Gerente de Tesorería no pueden invocar el endpoint de reasignación (Regla 1 / Criterio A2 — OBS-003).
- [ ] El flujo de alta de cliente en Cotizar lo Cotizable no expone ni procesa el campo Cobrador (GAP-05).