# Análisis de Impacto Backend
**Requisito:** TPSC-RE-FU-002 — Asignación de Cobrador a Cliente (Catálogo de Clientes)  
**Proyecto:** ProquifaDotNet

---

## 1. Resumen funcional

El requisito habilita el campo **"Cobrador"** en la sección Datos Generales del Catálogo de Clientes.  
- Solo el rol **Coordinador de Tesorería** puede editarlo.  
- El selector muestra únicamente usuarios con rol **Gestor de Cobranza** (`AnalistaDeCuentasPorCobrar = 1`) activos.  
- La asignación se persiste en `ClienteCartera.IdUsuarioCobrador`.  
- El cambio de Cobrador redistribuye **dinámicamente** todos los pendientes y pagos del cliente a la bandeja del nuevo Cobrador en los módulos: **Validar Cobro**, **Factura por Adelantado** y **Buzón de Pagos**.

---

## 2. Estructura actual en el proyecto

### 2.1 Capa de Lógica (`Logic.Pqf.Catalogos`)

| Archivo | Clase | Descripción relevante |
|---------|-------|-----------------------|
| `Clientes\Carteras\ClienteCarteraBO.cs` | `ClienteCarteraBO` | CRUD sobre `ClienteCartera`. Sin lógica de cobrador propia. |
| `Clientes\Carteras\ClienteCarteraClienteBO.cs` | `ClienteCarteraClienteBO` | Vinculación cliente–cartera. `_GuardarOActualizar` con columnas únicas `IdCliente` + `IdClienteCartera`. |
| `Clientes\Carteras\ClienteCarteraClienteBO.Extensions.cs` | `ClienteCarteraClienteBO` (partial) | `ObtenerIDUsuarioESACPorCliente()` — navega `ClienteCarteraCliente → ClienteCartera`. Patrón reutilizable para cobrador. |
| `Clientes\ClienteBO.cs` | `ClienteBO` | En la creación de cliente ya maneja `IdUsuarioCobrador`: busca la cartera del cobrador y vincula el cliente a esa cartera (`ActualizaCarteraCliente`). |
| `Clientes\ClienteBO.Extensions.cs` | `ClienteBO` (partial) | `ObtenerUsuariosCobrosAsignados(Guid idCliente)` — devuelve `vUsuario` activos asignados como cobradores del cliente via `ClienteCartera.IdUsuarioCobrador`. |
| `Usuarios\GMUsuarioClienteCarteraDetalleBO.cs` | `GMUsuarioClienteCarteraDetalleBO` | `Obtener(Guid IdUsuario)` — construye `CarteraUsuario` incluyendo `ClientesCarteraCobrador` (ya filtra por `IdUsuarioCobrador == Usuario.IdUsuario`). `ProcesarGMCarteraClienteUsuario()` — procesa asignación masiva de clientes a cartera. |
| `Usuarios\Models\GMUsuarioClienteCarteraDetalle.cs` | `CarteraUsuario` | Contiene `ClientesCarteraCobrador` (List\<ClienteCarteraDatos\>). El modelo ya soporta la cartera de cobrador. |
| `Usuarios\Models\GMCarteraClienteUsuario.cs` | `GMCarteraClienteUsuario` | DTO para procesar asignación de clientes a una cartera de usuario. |

### 2.2 Capa de API (`WebApi.Catalogos`)

| Archivo | Ruta HTTP | Métodos expuestos |
|---------|-----------|-------------------|
| `Controllers\Configuracion\Clientes\Cartera\ClienteCarteraController.cs` | `/ClienteCartera` | GET Obtener, PUT GuardarOActualizar, POST QueryResult, DELETE Desactivar |
| `Controllers\Sistema\Usuarios\UsuarioController.cs` | `/Usuario/GMUsuarioClienteCarteraDetalle` | GET — obtiene cartera completa del usuario (incluye `ClientesCarteraCobrador`) |
| `Controllers\Sistema\Usuarios\UsuarioController.cs` | `/Usuario/ClienteCarteraCliente` | POST — `ProcesarGMCarteraClienteUsuario` — asigna lista de clientes a cartera |
| `Controllers\Sistema\Usuarios\UsuarioController.cs` | `/Usuario/ListaUsuariosCartera` | POST QueryInfo — lista usuarios con sus relaciones de cartera |
| `Controllers\Sistema\Usuarios\UsuarioController.cs` | `/Usuario` | POST QueryResult — lista paginada de usuarios (base para filtrar Gestores de Cobranza) |

---

## 3. Mapeo de reglas de negocio al código

| Regla del Requisito | Implementación esperada | Estado actual |
|---------------------|------------------------|---------------|
| **Regla 1** — Solo Coordinador de Tesorería edita Cobrador | Control en capa de aplicación: campo editable solo si `Usuario.GerenteDeTesoreria = true` | ⚠️ No existe endpoint ni validación explícita de rol en la operación de actualización de `IdUsuarioCobrador` |
| **Regla 2** — Cobrador = Gestor de Cobranza activo | Filtrar `Usuario` con `AnalistaDeCuentasPorCobrar = 1 AND Activo = 1` | ⚠️ No existe un método/endpoint de solo Gestores de Cobranza para el selector |
| **Regla 3** — Un solo Cobrador por cliente | `ClienteCartera.IdUsuarioCobrador` es un campo único por cartera | ✅ La estructura de BD lo garantiza (un IdUsuarioCobrador por cartera) |
| **Regla 4** — Filtrado dinámico | El filtro en módulos usa JOIN a `ClienteCartera.IdUsuarioCobrador` en tiempo de consulta | ✅ `ObtenerUsuariosCobrosAsignados()` y `ClientesCarteraCobrador` ya implementan este patrón |
| **Regla 5** — Cliente sin Cobrador invisible en bandejas | Módulos filtran `WHERE IdUsuarioCobrador = @actual` — si es NULL no aparece | ✅ El JOIN natural excluye registros sin cobrador asignado |

---

## 4. Gaps identificados y cambios requeridos

### GAP-01 — No existe endpoint de selector de Gestores de Cobranza activos
**Impacto:** Criterio B1 — El frontend necesita poblar el selector de Cobrador con usuarios `AnalistaDeCuentasPorCobrar = 1 AND Activo = 1`.  
**Archivo a modificar:** `Logic.Pqf.Catalogos\Usuarios\GMUsuarioClienteCarteraDetalleBO.cs` o crear método en `UsuarioBO`.  
**Cambio requerido:** Agregar método para devolver la lista de Gestores de Cobranza activos.

```csharp
// En Logic.Pqf.Catalogos\Usuarios\UsuariosBO.Extensions.cs  (nuevo método)
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
```

**Endpoint a agregar:** `WebApi.Catalogos\Controllers\Sistema\Usuarios\UsuarioController.cs`

```csharp
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
```

---

### GAP-02 — No existe método para obtener el Cobrador actual de un cliente
**Impacto:** Criterio A1 — El Catálogo de Clientes debe mostrar el Cobrador actualmente asignado al cliente en la sección Datos Generales.  
**Patrón existente:** `ObtenerIDUsuarioESACPorCliente()` en `ClienteCarteraClienteBO.Extensions.cs` — el mismo patrón aplica para cobrador.  
**Archivo a modificar:** `Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraClienteBO.Extensions.cs`  
**Cambio requerido:** Agregar método análogo para `IdUsuarioCobrador`.

```csharp
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
```

---

### GAP-03 — No existe endpoint de reasignación explícita de Cobrador para un cliente
**Impacto:** Criterios B2 / B3 — El Coordinador de Tesorería debe poder cambiar el Cobrador de un cliente existente y el cambio debe redistribuir las bandejas dinámicamente.  
**Análisis:** El endpoint `PUT /ClienteCartera` (GuardarOActualizar) actualiza toda la cartera, incluyendo `IdUsuarioCobrador`. Técnicamente sirve, pero no encapsula la regla de negocio ni valida el rol del solicitante.  
**Cambio requerido:** Agregar un método en `ClienteCarteraBO` (o nuevo partial) que encapsule la reasignación de cobrador de forma explícita y validada.

```csharp
// En Logic.Pqf.Catalogos\Clientes\Carteras\ClienteCarteraBO.cs  (nuevo método partial)
/// <summary>
/// Reasigna el Cobrador (Gestor de Cobranza) en la cartera activa de un cliente.
/// El cambio aplica dinámicamente sobre todos los módulos consumidores (Validar Cobro,
/// Factura por Adelantado, Buzón de Pagos) en la siguiente consulta de cada bandeja.
/// </summary>
/// <param name="idCliente">Cliente al que se reasigna el cobrador.</param>
/// <param name="idNuevoCobrador">IdUsuario del nuevo Gestor de Cobranza.</param>
public void ReasignarCobrador(Guid idCliente, Guid idNuevoCobrador)
{
    using (var db = new ProquifaDotNetEntities())
    {
        // Validar que el nuevo cobrador tenga rol Gestor de Cobranza y esté activo
        var cobrador = db.Usuario.FirstOrDefault(u =>
            u.IdUsuario == idNuevoCobrador
            && u.AnalistaDeCuentasPorCobrar == true
            && u.Activo);

        if (cobrador == null)
            throw new ArgumentException("El usuario indicado no es un Gestor de Cobranza activo.");

        // Obtener la cartera activa del cliente
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
```

**Endpoint a agregar:** `WebApi.Catalogos\Controllers\Configuracion\Clientes\Cartera\ClienteCarteraController.cs`

```csharp
/// <summary>
/// Reasigna el Cobrador (Gestor de Cobranza) de un cliente.
/// Solo debe ser invocado por usuarios con rol Coordinador de Tesorería.
/// </summary>
[HttpPut]
[Route("ClienteCartera/ReasignarCobrador")]
public IHttpActionResult ReasignarCobrador(Guid idCliente, Guid idNuevoCobrador)
{
    var bo = new ClienteCarteraBO();
    bo.ReasignarCobrador(idCliente, idNuevoCobrador);
    return Ok();
}
```

---

### GAP-04 — Validación de rol Coordinador de Tesorería no implementada en la capa de API
**Impacto:** Criterios A2 / A3 — La regla indica que solo el Coordinador de Tesorería (`GerenteDeTesoreria = true` en `Usuario`) puede editar el campo Cobrador.  
**Estado actual:** No existe validación de rol en los endpoints de `ClienteCartera` ni en `GuardarOActualizar`.  
**Cambio requerido:** El endpoint `ReasignarCobrador` (GAP-03) debe validar que el usuario que invoca tiene `GerenteDeTesoreria = true`. La validación puede hacerse en la capa de lógica o mediante un filtro de autorización en el controller.

> **Decisión de arquitectura pendiente:** El proyecto actualmente no implementa autorización basada en claims de rol para endpoints específicos. Se debe definir si la validación va en la capa BO (pasando `IdUsuarioSolicitante` como parámetro) o en un atributo de autorización del controller.

---

### GAP-05 — Confirmación del mapeo de roles
**Impacto:** Regla 2 — El documento BD indica que `AnalistaDeCuentasPorCobrar` sería el campo del modelo `Usuario` que mapea al rol **Gestor de Cobranza**.  
**Estado:** ⚠️ Pendiente confirmar con el equipo.  
**Acción:** Verificar en la tabla `Usuario` que `AnalistaDeCuentasPorCobrar = true` sea exactamente el rol Gestor de Cobranza requerido en R16. Si no coincide, puede requerir un campo nuevo.

```sql
-- Verificar qué usuarios tienen AnalistaDeCuentasPorCobrar = 1
SELECT IdUsuario, NombreCompleto, AnalistaDeCuentasPorCobrar, Activo
FROM dbo.Usuario
WHERE AnalistaDeCuentasPorCobrar = 1
ORDER BY NombreCompleto;
```

---

## 5. Tablas y entidades del modelo de datos

| Tabla BD | Entidad EF | Propiedad clave | Descripción |
|----------|------------|-----------------|-------------|
| `ClienteCartera` | `ClienteCartera` | `IdClienteCartera` (PK), `IdUsuarioCobrador` (FK nullable) | Cartera del cliente. `IdUsuarioCobrador` = Gestor de Cobranza asignado. |
| `ClienteCarteraCliente` | `ClienteCarteraCliente` | `IdClienteCarteraCliente` (PK), `IdClienteCartera` (FK), `IdCliente` (FK) | Relación N:N entre cartera y cliente. |
| `Usuario` | `Usuario` | `IdUsuario` (PK), `AnalistaDeCuentasPorCobrar` (bit), `GerenteDeTesoreria` (bit), `Activo` (bit) | Rol Gestor de Cobranza = `AnalistaDeCuentasPorCobrar = 1`. Rol Coordinador de Tesorería = `GerenteDeTesoreria = 1`. |
| `vClienteCarteraCliente` | `vClienteCarteraCliente` (vista) | `IdUsuarioCobrador`, `Cobrador` | Vista operativa principal para bandejas. |
| `vUsuarioCartera` | `vUsuarioCartera` (vista) | `IdCliente`, `IdUsuario`, `Activo` | Vinculación directa usuario–cliente via cartera. Usada por módulos consumidores. |

---

## 6. Consultas SQL de referencia

### Selector — Gestores de Cobranza activos
```sql
SELECT IdUsuario, NombreCompleto
FROM dbo.Usuario
WHERE AnalistaDeCuentasPorCobrar = 1
  AND Activo = 1
ORDER BY NombreCompleto;
```

### Cobrador actual de un cliente
```sql
SELECT cc.IdUsuarioCobrador, u.NombreCompleto AS Cobrador
FROM dbo.ClienteCarteraCliente ccc
INNER JOIN dbo.ClienteCartera cc ON ccc.IdClienteCartera = cc.IdClienteCartera
LEFT  JOIN dbo.Usuario         u  ON cc.IdUsuarioCobrador = u.IdUsuario
WHERE ccc.IdCliente = @IdCliente
  AND ccc.Activo    = 1
  AND cc.Activo     = 1;
```

### Reasignar Cobrador
```sql
UPDATE dbo.ClienteCartera
SET    IdUsuarioCobrador        = @IdNuevoCobrador,
       FechaUltimaActualizacion = GETDATE()
WHERE  IdClienteCartera = @IdClienteCartera;
```

### Bandeja por cobrador (módulos consumidores)
```sql
SELECT v.IdCliente, v.Nombre AS Cliente, v.NombreCartera, v.Cobrador, v.NombreRegion
FROM dbo.vClienteCarteraCliente v
WHERE v.IdUsuarioCobrador = @IdUsuarioCobrador
ORDER BY v.Nombre;
```

### Clientes sin Cobrador (alerta operativa)
```sql
SELECT v.IdCliente, v.Nombre, v.NombreCartera
FROM dbo.vClienteCarteraCliente v
WHERE v.IdUsuarioCobrador IS NULL
ORDER BY v.Nombre;
```

---

## 7. Resumen de archivos a modificar / crear

| # | Archivo | Tipo de cambio |
|---|---------|---------------|
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
| **Validar Cobro** | JOIN `vUsuarioCartera WHERE IdUsuario = @CobActual AND Activo = 1` | Validar que el filtro esté implementado en la consulta de bandeja |
| **Factura por Adelantado** | JOIN `vUsuarioCartera WHERE IdUsuario = @CobActual AND Activo = 1` | Validar que el filtro esté implementado en la consulta de bandeja |
| **Buzón de Pagos** | JOIN `vUsuarioCartera WHERE IdUsuario = @CobActual AND Activo = 1` | Validar que el filtro esté implementado en la consulta de bandeja |

> Los cambios en los módulos consumidores se documentan en sus respectivos requisitos. Este documento solo cubre el catálogo de clientes y el campo Cobrador.

---

## 10. Pendientes / Decisiones abiertas

| # | Pendiente | Responsable |
|---|-----------|-------------|
| P1 | Confirmar que `AnalistaDeCuentasPorCobrar = true` es exactamente el rol **Gestor de Cobranza** requerido en R16. | TechLead / Funcional |
| P2 | Definir estrategia de validación de rol **Coordinador de Tesorería** (`GerenteDeTesoreria`): ¿en capa BO o filtro de autorización en controller? | Arquitectura |
| P3 | El SP `spDesactivarCarterasCliente` no retorna `IdUsuarioCobrador` en el SELECT de retorno. Si se requiere auditoría de reasignaciones, actualizar el SP. | Backend |
| P4 | Historial de asignaciones cobrador-cliente: fuera de alcance R16. Documentar como deuda técnica. | Product Owner |

---

## 11. Criterios de aceptación técnica

- [ ] `GET /Usuario/GestoresDeCobranza` retorna únicamente usuarios con `AnalistaDeCuentasPorCobrar = true AND Activo = true`.
- [ ] `ObtenerIdUsuarioCobradorPorCliente(idCliente)` retorna el `IdUsuarioCobrador` de la cartera activa del cliente, o `null` si no tiene asignación.
- [ ] `PUT /ClienteCartera/ReasignarCobrador` valida que el nuevo cobrador sea un Gestor de Cobranza activo antes de guardar.
- [ ] `PUT /ClienteCartera/ReasignarCobrador` actualiza `ClienteCartera.IdUsuarioCobrador` y `FechaUltimaActualizacion`.
- [ ] Tras la reasignación, la consulta de bandeja del nuevo Cobrador incluye al cliente reasignado; la del Cobrador anterior ya no lo incluye.
- [ ] Clientes sin `IdUsuarioCobrador` no aparecen en la bandeja de ningún Gestor de Cobranza.
- [ ] Roles distintos de Coordinador de Tesorería no pueden invocar el endpoint de reasignación de Cobrador.
