# R16A-RE-FU-008 — Análisis de Impacto Backend
## Buzón de Cobros — Clasificación y Reflejo de Correos de Pago del Cliente

> **Módulo:** Buzón de Cobros (nuevo)  
> **Tipo de cambio:** Funcionalidad nueva sobre infraestructura preexistente de MailBot  
> **Alcance:** Correos clasificados como Cobro por el Mailbot → Buzón del Gestor de Cobranza + pendiente en Validar Cobro

---

## 1. Estructura Actual del Proyecto

### Proyectos y carpetas involucrados

```
Logic.Pqf.Logistica\
  L11.MailBot\
    Constantes\                              <-- NUEVO — archivo de constantes de clasificación
      ClasificacionCorreoRecibidoConstants.cs
    GeneradorProcesoMailBotBO.cs             <-- Orquestador: switch por clasificación
    CorreoRecibidoClienteRequerimientoBO.cs  <-- BO del Buzón de Requisiciones (patrón a seguir)
    Procesos\Pagos\
      CorreoRecibidoClienteToPagoBO.cs       <-- Ya crea fccFolioPagoCliente al clasificar
    Procesos\Cobros\                         <-- NUEVO
      vCorreoRecibidoBuzonCobros.cs
      CorreoRecibidoClienteBuzonCobrosBO.cs

WebApi.Logistica\
  Controllers\Procesos\Mailbot\
    BuzonCobrosController.cs                 <-- NUEVO
    ConstruirBuzonCotizacionController.cs    <-- Patrón de referencia
    CorreoRecibidoClienteRequerimientoController.cs

WebApi.Catalogos\
  Controllers\Sistema\Correos\MailBots\Clientes\
    CorreoRecibidoClienteExtensionsController.cs  <-- Reclasificación (R8) ya existe

BD (SQL Server)
  catClasificacionCorreoRecibido  <-- CAMBIO: actualizar clave 'pago' → 'cobro' (según opción A/B)
  catProceso                      <-- CAMBIO: insertar proceso Cobros (solo Opción B)
  CorreoRecibido                  <-- Sin cambio — tiene IdRegion para MEX/PER
  CorreoRecibidoCliente           <-- Sin cambio — clasificación + cliente + cobrador
  fccFolioPagoCliente             <-- Sin cambio — pendiente en Validar Cobro
  ClienteCartera                  <-- Sin cambio — IdUsuarioCobrador para filtrar bandeja
  ClienteCarteraCliente           <-- Sin cambio — relación cliente-cartera
```

### Claves actuales en `catClasificacionCorreoRecibido`

> Estas son las claves **reales en BD**. Las constantes C# deben coincidir exactamente con ellas.

| Clave (BD)               | Nombre display           | AnalistaCxC | ESAC | Default | Activo |
| ------------------------ | ------------------------ | ----------- | ---- | ------- | ------ |
| `pago`                   | Pago                     | ✅ SÍ        | No   | No      | ✅ Sí   |
| `pedido`                 | Pedido                   | No          | ✅ Sí | No      | ✅ Sí   |
| `requisicion`            | Requisición              | No          | ✅ Sí | No      | ✅ Sí   |
| `solicituddeinformacion` | Solicitud de información | No          | ✅ Sí | No      | ✅ Sí   |
| `queja`                  | Queja                    | No          | ✅ Sí | No      | ✅ Sí   |
| `sugerencia`             | Sugerencia               | No          | ✅ Sí | No      | ✅ Sí   |
| `otros`                  | Otros                    | No          | ✅ Sí | ✅ Sí    | ✅ Sí   |

> ⚠️ La clave `cobro` **no existe aún** en BD. El FU-008 la crea en Tarea 1.
> La constante `ClasificacionCorreoRecibidoConstants.Pago = "pago"` refleja la clave actual.
> Cuando Tarea 1 ejecute el script de migración, la constante se actualizará a `"cobro"`
> y el nombre de la constante cambiará a `Cobro`.

### Flujo actual de procesamiento de correo por el Mailbot

```
Mailbot clasifica correo entrante
  └── GeneradorProcesoMailBotBO.Procesar()
        ├── Obtiene catClasificacionCorreoRecibido
        ├── switch (ClasificacionCorreoRecibido)   ← campo display, con tilde, hardcodeado
        │     case "Requisición" → CorreoRecibidoClienteToCotizacionBO.Process()
        │     case "Pedido"      → CorreoRecibidoClienteToPretramitacionPedidoBO.Process()
        │     case "Pago"        → [vacío] Logger.Debug — ❌ sin lógica real
        │     default            → Logger.Debug
        └── Marca CorreoRecibidoCliente.Procesado = true
```

### Hallazgos clave

> 1. El `case "Pago"` existe pero está vacío — nunca llama a `CorreoRecibidoClienteToPagoBO`.
> 2. `CorreoRecibidoClienteToPagoBO` ya existe y ya crea `fccFolioPagoCliente` — falta conectarlo.
> 3. El switch evalúa `ClasificacionCorreoRecibido` (nombre de display con tilde) en lugar del campo `Clave`.
>    FU-008 aprovecha para migrar el switch a `Clave` y centralizar los valores en constantes.

---

## 2. Mapeo Reglas de Negocio → Código

| Regla | Descripción | Estado actual | Ubicación |
|-------|-------------|---------------|-----------|
| R1 | Clasificación automática por Mailbot — categoría Cobro | ⚠️ Parcial | Existe `pago` con `AnalistaDeCuentasPorCobrar=1`; falta renombrar/insertar clave `cobro` |
| R2 | Sin criterios configurables — entrenamiento del modelo | ✅ Por diseño | Sin tabla de criterios — el Mailbot usa modelo de ML |
| R3 | Reflejo en Buzón del Gestor de Cobranza | ⚠️ Parcial | Campo `AnalistaDeCuentasPorCobrar=1` ya existe; falta BO de consulta por cobrador |
| R4 | Pendiente automático en Validar Cobro al clasificar | ⚠️ Parcial | `CorreoRecibidoClienteToPagoBO` ya crea `fccFolioPagoCliente` — falta conectarlo al switch |
| R5 | Visibilidad filtrada por cobrador asignado al cliente y región | ❌ Falta | Sin BO ni endpoint de buzón de cobros con filtro `IdUsuarioCobrador` + `IdRegion` |
| R6 | Cierre automático al vincular a proforma/factura | ✅ Existe | `fccFolioPagoCliente.Activo = 0` en módulo Validar Cobro — fuera de alcance FU-008 |
| R7 | Eliminación automática al marcar inconsistencia | ✅ Existe | `fccPagoCliente.Activo = 0` — fuera de alcance FU-008 |
| R8 | Reclasificación manual por Gestor | ✅ Existe | `CorreoRecibidoClienteExtensionsController` ya soporta UPDATE de `IdCatClasificacionCorreoRecibido` |
| R9 | Sin eliminación directa del correo por Gestor | ✅ Por diseño | No hay DELETE en `CorreoRecibidoClienteController` |
| R10 | Filtros, búsqueda y paginación como buzones existentes | ❌ Falta | Sin endpoint de listado de buzón de cobros con `QueryInfo` paginado |
| R11 | Bandeja del Coordinador de Tesorería — Caso 1: cliente sin Cobrador | ❌ Falta | Correo de cliente existente sin `IdUsuarioCobrador` → va a bandeja del Coordinador. Ver GAP-05. OBS-021 |
| R12 | Bandeja del Coordinador de Tesorería — Caso 2: remitente no dado de alta | ❌ Falta | `IdCliente` nulo en `CorreoRecibidoCliente` → va a bandeja del Coordinador. Ver GAP-05. OBS-021 |

---

## 3. GAPs de Implementación

### GAP-01 — BD: Actualizar `catClasificacionCorreoRecibido` — clave `pago` → `cobro`

**Problema:** La clave actual en BD es `pago`. El FU-008 la renombra a `cobro` para alinear
la nomenclatura con el módulo Buzón de Cobros. La constante `ClasificacionCorreoRecibidoConstants.Pago`
usa `"pago"` mientras exista esa clave; se actualiza a `"cobro"` tras ejecutar el script.

**Decisión previa requerida (Opción A vs B):**

```sql
-- Verificar si 'pago' está en uso en correos activos
SELECT COUNT(*) AS TotalCorreosClasificadosPago
FROM   dbo.CorreoRecibidoCliente   crc
JOIN   dbo.catClasificacionCorreoRecibido cat
         ON crc.IdCatClasificacionCorreoRecibido = cat.IdCatClasificacionCorreoRecibido
WHERE  cat.Clave = 'pago' AND crc.Activo = 1;

-- OPCIÓN A — Renombrar (recomendada si 'pago' no está en uso)
UPDATE dbo.catClasificacionCorreoRecibido
SET    ClasificacionCorreoRecibido = 'Cobro',
       Clave                       = 'cobro',
       FechaUltimaActualizacion    = GETDATE()
WHERE  Clave = 'pago';

-- OPCIÓN B — Nuevo registro (si 'pago' se usa en otros contextos)
DECLARE @IdProcesoCobros UNIQUEIDENTIFIER = NEWID();
INSERT INTO dbo.catProceso
    (IdCatProceso, Proceso, Clave, FechaRegistro, FechaUltimaActualizacion, Activo)
VALUES (@IdProcesoCobros, 'Cobros', 'cobros', GETDATE(), GETDATE(), 1);

INSERT INTO dbo.catClasificacionCorreoRecibido
    (IdCatClasificacionCorreoRecibido, IdCatProceso, ClasificacionCorreoRecibido,
     Posicion, ClasificacionDefault, EVI, EVE, ESAC, AnalistaDeCuentasPorCobrar,
     CoordinadorDeServicioAlCliente, Clave, Activo)
VALUES (NEWID(), @IdProcesoCobros, 'Cobro', 8, 0, 0, 0, 0, 1, 0, 'cobro', 1);

UPDATE dbo.catClasificacionCorreoRecibido
SET    Activo = 0, FechaUltimaActualizacion = GETDATE()
WHERE  Clave  = 'pago';
```

---

### GAP-02 — Logic: Crear `ClasificacionCorreoRecibidoConstants` y refactorizar el switch del orquestador

**Problema:**
1. Los valores del switch están hardcodeados como literales con el nombre de display (`"Requisición"`,
   `"Pedido"`, `"Pago"`) en lugar del campo técnico `Clave` de BD.
2. El switch evalúa `ClasificacionCorreoRecibido` (con tilde) — si el nombre se edita en BD, el
   enrutamiento falla silenciosamente sin error de compilación.
3. El `case "Pago"` existe pero está vacío — nunca genera el pendiente `fccFolioPagoCliente`.

**Constantes — valores alineados con claves reales de BD:**

```csharp
// Logic.Pqf.Logistica\L11.MailBot\Constantes\ClasificacionCorreoRecibidoConstants.cs
// Los valores de cada constante son las claves EXACTAS de catClasificacionCorreoRecibido.Clave en BD.
namespace Logic.Pqf.Logistica.L11.MailBot.Constantes
{
    public static class ClasificacionCorreoRecibidoConstants
    {
        public const string Requisicion = "requisicion";  // BD: clave existente
        public const string Pedido      = "pedido";       // BD: clave existente

        // BD actual: clave = 'pago'. Se actualizará a "cobro" tras ejecutar Tarea 1.
        // Renombrar también esta constante a Cobro = "cobro" cuando BD esté migrada.
        public const string Pago        = "pago";

        public const string Otros       = "otros";        // BD: clave existente
    }
}
```

**Switch refactorizado — evalúa `Clave` usando constantes:**

```csharp
// Logic.Pqf.Logistica\L11.MailBot\Procesos\GeneradorProcesoMailBotBO.cs
using Logic.Pqf.Logistica.L11.MailBot.Constantes;
using Logic.Pqf.Logistica.L11.MailBot.Procesos.Pagos;

// ANTES — switch sobre nombre de display, literales hardcodeados:
switch (catClasificacionCorreoRecibido.ClasificacionCorreoRecibido)
{
    case "Requisición": ...
    case "Pedido":      ...
    case "Pago":        // vacío — sin lógica
}

// DESPUÉS — switch sobre Clave (campo técnico) usando constantes:
switch (catClasificacionCorreoRecibido.Clave?.Trim().ToLowerInvariant())
{
    case ClasificacionCorreoRecibidoConstants.Requisicion:
        Logger.Debug("Generando requisición de cotización");
        var generadorA = new CorreoRecibidoClienteToCotizacionBO(usuario.IdUsuario);
        generadorA.Process(correoRecibidoCliente);
        break;

    case ClasificacionCorreoRecibidoConstants.Pedido:
        Logger.Debug("Generando pretramitación de pedido");
        var generadorB = new CorreoRecibidoClienteToPretramitacionPedidoBO(usuario.IdUsuario);
        generadorB.Process(correoRecibidoCliente);
        break;

    case ClasificacionCorreoRecibidoConstants.Pago:    // FU-008 — clave actual "pago"
        Logger.Debug("Generando pendiente de cobro del cliente en Validar Cobro");
        var generadorC = new CorreoRecibidoClienteToPagoBO();
        generadorC.Process(input, context);
        break;

    default:
        Logger.Debug("Clasificación sin proceso asignado — pendiente en buzón correspondiente");
        break;
}
```

> **Ciclo de vida de la constante `Pago`:**
> - **Antes de Tarea 1:** `Pago = "pago"` — refleja la clave real actual en BD.
> - **Después de Tarea 1:** cambiar a `Cobro = "cobro"` en el mismo PR que ejecuta el script SQL.
> - El cambio de nombre de la constante es un refactor de 1 línea en el archivo de constantes;
>   el compilador señala todos los usos que deban actualizarse.

---

### GAP-03 — Logic: Crear `CorreoRecibidoClienteBuzonCobrosBO` — consulta paginada del Buzón de Cobros

**Problema:** No existe BO que liste los correos del Buzón de Cobros filtrados por cobrador y región.
La query filtra por `cat.Clave == ClasificacionCorreoRecibidoConstants.Pago` (clave real actual).

```csharp
// NUEVO: Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\CorreoRecibidoClienteBuzonCobrosBO.cs
using Logic.Pqf.Logistica.L11.MailBot.Constantes;

// Fragmento de la query:
where cat.Clave  == ClasificacionCorreoRecibidoConstants.Pago  // "pago" actual → "cobro" tras Tarea 1
   && crc.Activo == true
```

---

### GAP-04 — WebApi: Crear `BuzonCobrosController` — endpoint `POST /BuzonCobros`

**Problema:** No existe endpoint que exponga el listado paginado del Buzón de Cobros al frontend.

```csharp
// NUEVO: WebApi.Logistica\Controllers\Procesos\Mailbot\BuzonCobrosController.cs
// ⚠️ PENDIENTE P1: evaluar extraer idUsuarioCobrador del token de autenticación
[HttpPost]
[Route("BuzonCobros")]
[ResponseType(typeof(QueryResult<vCorreoRecibidoBuzonCobros>))]
public HttpResponseMessage Lista([FromBody] QueryInfo queryInfo, Guid idUsuarioCobrador)
{
    return TryExecute(() =>
    {
        var bo = new CorreoRecibidoClienteBuzonCobrosBO();
        var result = bo.Lista(queryInfo, idUsuarioCobrador);
        return result != null
            ? Request.CreateResponse(HttpStatusCode.OK, result)
            : Request.CreateResponse(HttpStatusCode.BadRequest);
    });
}
```

---

## 4. Entidades de Base de Datos Involucradas

| Entidad | Rol en FU-008 | Cambio requerido |
|---------|---------------|-----------------|
| `catClasificacionCorreoRecibido` | Define las categorías que el Mailbot asigna | **UPDATE/INSERT** — actualizar clave `pago`→`cobro` (Opción A o B) |
| `catProceso` | Agrupa clasificaciones por proceso de negocio | **INSERT** proceso `Cobros` solo si Opción B |
| `CorreoRecibido` | Correo entrante con `IdRegion` para segregación MEX/PER | Sin cambio |
| `CorreoRecibidoCliente` | Clasificación + `IdCliente` + `Leido/Procesado` | Sin cambio |
| `CorreoRecibidoEstatus` | Estado de lectura del correo | Sin cambio |
| `fccFolioPagoCliente` | Pendiente en Validar Cobro — se crea al clasificar | Sin cambio |
| `ClienteCartera` | `IdUsuarioCobrador` — filtro de visibilidad de la bandeja | Sin cambio |
| `ClienteCarteraCliente` | Relación cliente-cartera para filtro de cobrador | Sin cambio |

---

### GAP-05 — Logic + WebApi: Bandeja del Coordinador de Tesorería (OBS-021)

**Problema:** Los correos clasificados como cobro que no pueden enrutarse a ningún Gestor (cliente sin Cobrador asignado — Caso 1; remitente no dado de alta — Caso 2) no aparecen en ninguna bandeja. OBS-021 establece que deben concentrarse en la bandeja del Coordinador de Tesorería para que tome acción: asignar un Cobrador al cliente (Caso 1) o dar de alta el contacto (Caso 2).

**Mecanismo de retroactividad (automático por diseño de query):**
La bandeja del Gestor de Cobranza ya filtra por `ClienteCartera.IdUsuarioCobrador`. Cuando el Coordinador asigna un Cobrador al cliente (mediante FU-002), los correos de ese cliente que antes aparecían en la bandeja del Coordinador **automáticamente dejan de aparecer allí** (porque el cliente ya tiene cobrador) y **automáticamente aparecen en la bandeja del nuevo Cobrador** (porque el JOIN por `IdUsuarioCobrador` ya los incluye). No se requiere lógica adicional de migración — la retroactividad es estructural.

**Cómo identificar al Coordinador de Tesorería:**
El sistema debe tener un mecanismo para identificar al usuario con rol de Coordinador de Tesorería (análogo a `AnalistaDeCuentasPorCobrar` en `catClasificacionCorreoRecibido`, o mediante el catálogo de roles de usuario existente). El endpoint de la bandeja del Coordinador solo debe ser accesible por ese rol.

```csharp
// NUEVO: Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\BandejaCoordenadorTesoreriaBO.cs

public class BandejaCoordenadorTesoreriaBO
{
    /// <summary>
    /// Bandeja del Coordinador de Tesorería: correos de cobro no enrutables.
    /// Caso 1 (R11 — OBS-021): cliente existe pero sin Cobrador asignado (IdUsuarioCobrador IS NULL).
    /// Caso 2 (R12 — OBS-021): remitente no dado de alta (IdCliente IS NULL).
    /// La retroactividad al asignar Cobrador es automática por el diseño de la query del Gestor.
    /// </summary>
    public QueryResult<vCorreoRecibidoBuzonCobros> Lista(QueryInfo queryInfo)
    {
        using (var context = new ProquifaDotNetEntities())
        {
            var claveCobro = ClasificacionCorreoRecibidoConstants.Pago; // → .Cobro tras Tarea 1

            var query = (from crc in context.CorreoRecibidoCliente
                         join cr  in context.CorreoRecibido
                             on crc.IdCorreoRecibido equals cr.IdCorreoRecibido
                         join cat in context.catClasificacionCorreoRecibido
                             on crc.IdCatClasificacionCorreoRecibido equals cat.IdCatClasificacionCorreoRecibido
                         join c   in context.Cliente
                             on crc.IdCliente equals c.IdCliente into clienteGroup
                         from c in clienteGroup.DefaultIfEmpty()   // LEFT JOIN para Caso 2
                         join ccc in context.ClienteCarteraCliente
                             on (c == null ? (Guid?)null : c.IdCliente) equals ccc.IdCliente into carteraGroup
                         from ccc in carteraGroup.DefaultIfEmpty() // LEFT JOIN para Caso 1
                         join cc  in context.ClienteCartera
                             on (ccc == null ? (Guid?)null : ccc.IdClienteCartera) equals cc.IdClienteCartera into ccGroup
                         from cc in ccGroup.DefaultIfEmpty()       // LEFT JOIN para Caso 1
                         where cat.Clave == claveCobro
                            && crc.Activo == true
                            && (crc.IdCliente == null              // Caso 2: remitente no dado de alta
                                || cc == null                      // Caso 1: cliente sin cartera
                                || cc.IdUsuarioCobrador == null)   // Caso 1: cliente sin Cobrador
                         orderby cr.FechaRecepcion ascending       // más antiguo primero
                         select new vCorreoRecibidoBuzonCobros
                         {
                             IdCorreoRecibidoCliente = crc.IdCorreoRecibidoCliente,
                             IdCorreoRecibido        = cr.IdCorreoRecibido,
                             IdCliente               = crc.IdCliente,
                             NombreCliente           = c == null ? null : c.Nombre,
                             Asunto                  = cr.Asunto,
                             De                      = cr.CorreoEmisor,
                             Leido                   = crc.Leido,
                             IdRegion                = cr.IdRegion,
                             FechaRegistro           = cr.FechaRecepcion
                         });

            return new QueryResult<vCorreoRecibidoBuzonCobros>
            {
                Total   = query.Count(),
                Results = query.Skip(queryInfo.Skip).Take(queryInfo.Take).ToList()
            };
        }
    }
}

// NUEVO: WebApi.Logistica\Controllers\Procesos\Mailbot\BandejaCoordenadorTesoreriaController.cs
// Endpoint: POST /BandejaCoordenadorTesoreria
// Solo accesible por rol Coordinador de Tesorería (heredar [Authorize] + validación de rol)
```

---

### GAP-06 — Tarea 5 actualizada: enrutamiento condicional al clasificar cobro (OBS-021)

**Problema:** `CorreoRecibidoClienteToPagoBO.Process()` siempre genera `fccFolioPagoCliente`. Con OBS-021, el comportamiento cambia: si el correo es no enrutable (Caso 1 o Caso 2), el folio se crea igualmente pero se registra en log para que el Coordinador sepa que debe intervenir. El correo aparece en la bandeja del Coordinador (GAP-05) y no en ninguna bandeja de Gestor hasta que se resuelva.

```csharp
// CorreoRecibidoClienteToPagoBO.cs — flujo actualizado con OBS-021:

// 1. Verificar IdCliente
if (correoRecibidoCliente.IdCliente == null)
{
    Logger.Info("FU-008 OBS-021 Caso 2: remitente no dado de alta → bandeja Coordinador Tesorería");
    // Crear folio igualmente para que el Coordinador lo vea en su bandeja
    // NO asignar a ningún Gestor (sin IdUsuarioCobrador)
}
else
{
    // Verificar cobrador
    var cobrador = ObtenerCobradorDeCliente(correoRecibidoCliente.IdCliente);
    if (cobrador == null)
        Logger.Info("FU-008 OBS-021 Caso 1: cliente sin Cobrador → bandeja Coordinador Tesorería");
    // Crear folio en ambos casos; la bandeja del Coordinador lo mostrará por query (GAP-05)
}
// Crear fccFolioPagoCliente con Activo = true (sin datos de cobro pre-capturados)
// Marcar CorreoRecibidoCliente.Procesado = true
```

---

## 5. Archivos a Modificar

| # | Archivo | Tipo de cambio | GAP |
|---|---------|----------------|-----|
| 1 | `BD: catClasificacionCorreoRecibido` | UPDATE Opción A o INSERT Opción B | GAP-01 |
| 2 | `BD: catProceso` | INSERT proceso `Cobros` — solo Opción B | GAP-01 |
| 3 | `Logic.Pqf.Logistica\L11.MailBot\Procesos\GeneradorProcesoMailBotBO.cs` | Refactorizar switch a `Clave` + constantes + conectar case `Pago` | GAP-02 |

## 6. Archivos a Crear

| # | Archivo | Descripción | GAP |
|---|---------|-------------|-----|
| 4 | `Logic.Pqf.Logistica\L11.MailBot\Constantes\ClasificacionCorreoRecibidoConstants.cs` | Constantes alineadas con claves reales de BD: `Requisicion="requisicion"`, `Pedido="pedido"`, `Pago="pago"`, `Otros="otros"` | GAP-02 |
| 5 | `Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\vCorreoRecibidoBuzonCobros.cs` | Modelo POCO del resultado del buzón | GAP-03 |
| 6 | `Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\CorreoRecibidoClienteBuzonCobrosBO.cs` | BO de listado paginado filtrado por cobrador y región | GAP-03 |
| 7 | `WebApi.Logistica\Controllers\Procesos\Mailbot\BuzonCobrosController.cs` | Controller con endpoint `POST /BuzonCobros` | GAP-04 |
| 8 | `Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\BandejaCoordenadorTesoreriaBO.cs` | BO de bandeja del Coordinador: correos no enrutables (sin cobrador o sin cliente) | GAP-05 — OBS-021 |
| 9 | `WebApi.Logistica\Controllers\Procesos\Mailbot\BandejaCoordenadorTesoreriaController.cs` | Controller con endpoint `POST /BandejaCoordenadorTesoreria` | GAP-05 — OBS-021 |

---

## 7. Archivos Sin Cambio

| Archivo | Motivo |
|---------|--------|
| `CorreoRecibidoClienteToPagoBO.cs` | Ya implementa la creación de `fccFolioPagoCliente` — solo se conecta al switch |
| `CorreoRecibidoClienteExtensionsController.cs` | La reclasificación manual (R8) ya está implementada |
| `CorreoRecibidoClienteController.cs` | Sin eliminación directa — R9 ya se cumple por diseño actual |
| `fccFolioPagoCliente` (tabla) | Estructura existente sin cambios de schema |
| `fccPagoCliente` (tabla) | Cierre por inconsistencia (R7) pertenece a FU-009 |
| `CorreoRecibido`, `CorreoRecibidoEstatus`, `ClienteCartera`, `ClienteCarteraCliente` | Sin cambios de schema |

---

## 8. Pendientes Externos al Backend

| ID | Pendiente | Bloqueante | Responsable |
|----|-----------|------------|-------------|
| P1 | Definir si `idUsuarioCobrador` se extrae del token o como parámetro de URL | Sí — impacta firma del endpoint y seguridad | Arquitecto / Líder técnico |
| P2 | Decidir Opción A (renombrar `pago`→`cobro`) vs Opción B (nuevo registro) | **Sí** — bloquea GAP-01 y GAP-02 | Líder técnico + DBA |
| P3 | Entrenamiento del Mailbot con categoría `Cobro` sobre correos reales de PROQUIFA | Sí — sin entrenamiento el Mailbot no clasifica correos como Cobro | Equipo de IA / Mailbot |
| P4 | Confirmar flujo para correos cuyo `IdCliente` es null (cliente no identificable) | **Resuelto OBS-021** — van a bandeja del Coordinador de Tesorería (Caso 2, Regla 12). Sin pendiente. | — |
| P5 | Confirmar mecanismo de identificación del rol Coordinador de Tesorería en el sistema de usuarios | Sí — impacta la seguridad del endpoint de la bandeja del Coordinador | Arquitecto / Líder técnico |

---

## 9. Criterios de Aceptación Backend

| Criterio | Cómo verificar |
|----------|----------------|
| Al clasificar un correo como Cobro, se crea registro en `fccFolioPagoCliente` | Clasificar correo de prueba → verificar INSERT en `fccFolioPagoCliente` |
| El switch evalúa `Clave` usando `ClasificacionCorreoRecibidoConstants` — sin literales hardcodeados | Inspeccionar `GeneradorProcesoMailBotBO` — switch sobre `catClasificacionCorreoRecibido.Clave` |
| Las constantes coinciden exactamente con las claves de `catClasificacionCorreoRecibido` en BD | `SELECT Clave FROM catClasificacionCorreoRecibido` → comparar con los valores de las constantes |
| Prueba de regresión: `requisicion` y `pedido` siguen enrutando correctamente | Clasificar correos de requisición y pedido → procesos generados sin cambio de comportamiento |
| El Buzón de Cobros muestra solo los correos del cobrador activo | `POST /BuzonCobros` → solo correos de clientes de la cartera del cobrador |
| Correos MEX no aparecen en el buzón de cobrador PER y viceversa | Verificar segregación por `IdRegion` en los resultados |
| El Buzón retorna resultados paginados | `POST /BuzonCobros` con `QueryInfo` → resultado paginado correcto |
| `catClasificacionCorreoRecibido` tiene registro con `Clave = 'cobro'` y `AnalistaDeCuentasPorCobrar = 1` | `SELECT * FROM catClasificacionCorreoRecibido WHERE Clave = 'cobro'` → 1 fila activa |
| Los proyectos `Logic.Pqf.Logistica` y `WebApi.Logistica` compilan sin errores | Build en Visual Studio sin errores |

---

## 10. Resumen de Cambios

```
ARCHIVOS NUEVOS:        4
  - ClasificacionCorreoRecibidoConstants.cs  (Logic — claves reales de BD: pago/pedido/requisicion/otros)
  - vCorreoRecibidoBuzonCobros.cs             (Logic — modelo POCO)
  - CorreoRecibidoClienteBuzonCobrosBO.cs     (Logic — listado paginado Buzón Cobros)
  - BuzonCobrosController.cs                 (WebApi)

ARCHIVOS MODIFICADOS:   3
  - catClasificacionCorreoRecibido  (BD — UPDATE clave pago→cobro o INSERT según Opción A/B)
  - catProceso                      (BD — INSERT solo si Opción B)
  - GeneradorProcesoMailBotBO.cs    (Logic — refactorizar switch a Clave + constantes + conectar Pago)

LITERALES HARDCODEADOS ELIMINADOS: 3  ("Requisición","Pedido","Pago" → constantes sobre campo Clave)
CONSULTAS BD EXTRA:     0  (el BO usa JOIN sobre tablas ya en el contexto EF)
SCRIPTS SQL NUEVOS:     1  (catClasificacionCorreoRecibido + catProceso)
PENDIENTE CRÍTICO:      Entrenamiento del Mailbot (P3) — sin él el flujo no se detona
```
