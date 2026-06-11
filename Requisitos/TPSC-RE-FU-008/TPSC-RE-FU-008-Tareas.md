# TPSC-RE-FU-008 — Tareas de Implementación Backend

| Campo | Valor |
|-------|-------|
| **Requisito** | TPSC-RE-FU-008 |
| **Nombre** | Buzón de Cobros |
| **Total de tareas** | 12 |
| **Revisión aplicada** | TPSC-RE-FU-008-Back.md |
| **Última actualización** | 2026-06-11 — Tareas 10, 11 y 12 agregadas: ETL Buzón de Cobros → Legacy |

---

## Tarea 1

### TPSC-RE-FU-008  [ BD-OBJ-CH ]  Script de migración de catálogos para Buzón de Cobros

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
MailBot L11 — Clasificación de correo recibido

**Consideraciones previas:**
Esta tarea es **prerequisito de Tarea 3**. La clave `"cobro"` debe existir en `catClasificacionCorreoRecibido` antes de que el listado paginado pueda filtrar correctamente.
Al ejecutar este script en cada ambiente, en el mismo PR de código se debe renombrar la constante `Pago = "pago"` a `Cobro = "cobro"` en `ClasificacionCorreoRecibidoConstants.cs` (ver Tarea 2).
Verificar que ningún otro módulo dependa de la clave `"pago"` antes de ejecutar el UPDATE.
Nota de implementación: toda la terminología del módulo usa el término **"cobro"** — no "pago". El nombre refleja la perspectiva operativa de PROQUIFA: son cobros hacia el cliente, aunque el correo original sea el comprobante de pago que el cliente envía.

**Descripción del problema:**
La tabla `catClasificacionCorreoRecibido` tiene el registro de cobros con clave `"pago"` y nombre `"Pago"`. El requisito FU-008 establece que la clasificación debe llamarse `"cobro"` para representar correctamente el Buzón de Cobros (Regla 1 — R16 agrega la categoría Cobro al modelo existente). Además, no existe proceso `"cobros"` en `catProceso`, necesario para vincular la clasificación.

**Archivo a crear:**
```
Scripts\R16\FU-008_CatalogosCobros.sql
```

**Cambios requeridos:**

```sql
-- ============================================================
-- FU-008 Tarea 1 — Migración catálogos Buzón de Cobros
-- Ejecutar por ambiente: DEV → QA → PROD
-- Después de ejecutar: actualizar ClasificacionCorreoRecibidoConstants.Pago
--   a  Cobro = "cobro"  en el mismo PR de código.
-- ============================================================

BEGIN TRANSACTION;

-- 1. Insertar proceso Cobros en catProceso (si no existe)
IF NOT EXISTS (SELECT 1 FROM dbo.catProceso WHERE Clave = 'cobros')
BEGIN
    INSERT INTO dbo.catProceso
        (IdCatProceso, Proceso, Clave, FechaRegistro, FechaUltimaActualizacion, Activo)
    VALUES
        (NEWID(), 'Cobros', 'cobros', GETDATE(), GETDATE(), 1);
END

-- 2. Actualizar clasificación: pago → cobro
UPDATE dbo.catClasificacionCorreoRecibido
SET
    ClasificacionCorreoRecibido = 'Cobro',
    Clave                       = 'cobro',
    IdCatProceso                = (SELECT TOP 1 IdCatProceso FROM dbo.catProceso WHERE Clave = 'cobros'),
    FechaUltimaActualizacion    = GETDATE()
WHERE Clave = 'pago';

-- Verificación
SELECT ClasificacionCorreoRecibido, Clave, AnalistaDeCuentasPorCobrar
FROM   dbo.catClasificacionCorreoRecibido
WHERE  Clave IN ('cobro', 'pago');

COMMIT;
```

**Criterios de aceptación:**
- [ ] **[A1 — R1]** Existe un registro en `catClasificacionCorreoRecibido` con `Clave = 'cobro'` y `AnalistaDeCuentasPorCobrar = 1` — la categoría Cobro está disponible para el Mailbot
- [ ] **[A1 — R1]** El registro antiguo con `Clave = 'pago'` ya no existe en la tabla
- [ ] Existe un registro en `catProceso` con `Clave = 'cobros'` y `Activo = 1`
- [ ] El campo `IdCatProceso` del registro `cobro` referencia al proceso `cobros`
- [ ] La constante C# `Pago = "pago"` se renombra a `Cobro = "cobro"` en el mismo PR (Tarea 2)

**Más información de la tarea:**
- GAP-01 del archivo `TPSC-RE-FU-008-Back.md`
- Regla de negocio cubierta: **R1** (R16 agrega categoría Cobro al modelo del Mailbot)
- La clave actual `"pago"` tiene `AnalistaDeCuentasPorCobrar = 1` — ese campo no cambia, solo se actualiza `Clave` y `ClasificacionCorreoRecibido`
- **R2** (sin criterios configurables) no requiere implementación backend — el Mailbot usa entrenamiento de base de conocimiento. El entrenamiento inicial con correos reales de PROQUIFA es prerequisito operativo (ver Tarea 7).

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 2

### TPSC-RE-FU-008  [ IMP-EXIST-SERVICE ]  Crear constantes de clasificación y refactorizar switch del MailBot

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Logistica

**Módulos:**
MailBot L11 — GeneradorProcesoMailBotBO

**Consideraciones previas:**
Esta tarea puede ejecutarse **en paralelo con Tarea 1**. No requiere que el script SQL esté ejecutado para compilar, ya que la constante transitoria `Pago = "pago"` es válida mientras BD tenga esa clave.
`ClasificacionCorreoRecibidoConstants.cs` ya fue creado en el repositorio con la constante transitoria. Solo se requiere modificar `GeneradorProcesoMailBotBO.cs`.
Cuando se ejecute el script SQL de Tarea 1 en el ambiente correspondiente, en el **mismo PR** se debe renombrar `Pago = "pago"` a `Cobro = "cobro"` y actualizar todos los usos.
Patrón de referencia: `Logic.Pqf.Catalogos\Constantes\ConstantesMonitoreoOferta.cs`.
Nota de implementación: el **Buzón de Cobros** y el módulo **Validar Cobro** NO son el mismo módulo. El switch del MailBot solo desencadena la clasificación (entrada al Buzón) y la creación del folio en Validar Cobro (Tarea 5). Las acciones de captura, vinculación e inconsistencia pertenecen a Validar Cobro.
**R8 — Reclasificación manual:** `CorreoRecibidoClienteExtensionsController` ya soporta UPDATE de `IdCatClasificacionCorreoRecibido`. Verificar que el refactor del switch no afecta esta funcionalidad existente.
**R9 — Sin eliminación directa:** El Buzón no expone acción de eliminación. Verificar que ningún endpoint del refactor la habilite accidentalmente.

**Descripción del problema:**
El switch en `GeneradorProcesoMailBotBO.Procesar()` evalúa `ClasificacionCorreoRecibido` (nombre display con tilde), lo que lo hace frágil ante cambios de nombre en catálogo. El `case "Pago"` está vacío — el correo de cobros no desencadena ningún proceso. Se necesita migrar el switch al campo técnico `Clave` usando constantes tipadas, y conectar el case de cobros a `CorreoRecibidoClienteToPagoBO` para cumplir R1 y A1.

**Archivos a modificar / crear:**
```
Logic.Pqf.Logistica\L11.MailBot\Constantes\ClasificacionCorreoRecibidoConstants.cs  (ya creado)
Logic.Pqf.Logistica\L11.MailBot\Procesos\GeneradorProcesoMailBotBO.cs               (modificar switch)
```

**Cambios requeridos:**

`ClasificacionCorreoRecibidoConstants.cs` — estado actual en repositorio (ya correcto):

```csharp
namespace Logic.Pqf.Logistica.L11.MailBot.Constantes
{
    public static class ClasificacionCorreoRecibidoConstants
    {
        public const string Requisicion = "requisicion";
        public const string Pedido      = "pedido";
        /// Clave BD actual: "pago". Renombrar a Cobro="cobro" en el PR que ejecute Tarea 1.
        public const string Pago        = "pago";
        public const string Otros       = "otros";
    }
}
```

`GeneradorProcesoMailBotBO.cs` — refactorizar el switch:

Antes:
```csharp
switch (catClasificacionCorreoRecibido.ClasificacionCorreoRecibido)
{
    case "Requisición":
        var generadorA = new CorreoRecibidoClienteToCotizacionBO(usuario.IdUsuario);
        generadorA.Process(correoRecibidoCliente);
        break;
    case "Pedido":
        var generadorB = new CorreoRecibidoClienteToPretramitacionPedidoBO(usuario.IdUsuario);
        generadorB.Process(correoRecibidoCliente);
        break;
    case "Pago":
        // vacío — sin lógica
        break;
}
```

Después:
```csharp
// Agregar using: Logic.Pqf.Logistica.L11.MailBot.Constantes
switch (catClasificacionCorreoRecibido.Clave?.Trim().ToLowerInvariant())
{
    case ClasificacionCorreoRecibidoConstants.Requisicion:   // A1 — R1
        Logger.Debug("Generando requisición de cotización");
        var generadorA = new CorreoRecibidoClienteToCotizacionBO(usuario.IdUsuario);
        generadorA.Process(correoRecibidoCliente);
        break;

    case ClasificacionCorreoRecibidoConstants.Pedido:        // A1 — R1
        Logger.Debug("Generando pretramitación de pedido");
        var generadorB = new CorreoRecibidoClienteToPretramitacionPedidoBO(usuario.IdUsuario);
        generadorB.Process(correoRecibidoCliente);
        break;

    case ClasificacionCorreoRecibidoConstants.Pago:          // A1 — R1 — FU-008: "pago" → "cobro" tras Tarea 1
        Logger.Debug("FU-008: Generando pendiente de cobro del cliente en Validar Cobro");
        var generadorC = new CorreoRecibidoClienteToPagoBO();
        generadorC.Process(input, context);
        break;

    default:                                                  // clasificación sin proceso asignado
        Logger.Debug("Clasificación sin proceso asignado — pendiente en buzón correspondiente");
        break;
}
```

**Criterios de aceptación:**
- [ ] **[A1 — R1]** El switch evalúa `catClasificacion.Clave?.Trim().ToLowerInvariant()` — no el display `ClasificacionCorreoRecibido`
- [ ] **[A1 — R1]** El `case ClasificacionCorreoRecibidoConstants.Pago` invoca `CorreoRecibidoClienteToPagoBO.Process()` — el correo de cobro desencadena el proceso
- [ ] **[A2 — R2]** No existen criterios configurables en el switch — la clasificación depende únicamente del modelo entrenado del Mailbot (ver Tarea 7)
- [ ] **[R8]** Prueba de regresión: la reclasificación manual desde `CorreoRecibidoClienteExtensionsController` sigue funcionando correctamente después del refactor
- [ ] **[R9]** No existe ningún endpoint ni acción que permita eliminación directa del correo desde el Buzón de Cobros
- [ ] No quedan literales `"Requisición"`, `"Pedido"`, `"Pago"` hardcodeados en el switch
- [ ] Prueba de regresión: correos de requisición y pedido siguen enrutando correctamente
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-02 del archivo `TPSC-RE-FU-008-Back.md`
- Reglas de negocio cubiertas: **R1**, **R2** (verificar), **R8** (reclasificación — regresión), **R9** (sin eliminación — verificar)
- `CorreoRecibidoClienteToPagoBO` ya existe y crea `fccFolioPagoCliente` — solo se conecta al switch

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 3

### TPSC-RE-FU-008  [ LIST-PAG-MULT-FILTER ]  CorreoRecibidoClienteBuzonCobrosBO — listado paginado del Buzón de Cobros

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Logistica

**Módulos:**
MailBot L11 — Buzón de Cobros

**Consideraciones previas:**
**Depende de Tarea 1 y Tarea 2.** La clave `"cobro"` debe existir en BD y la constante debe estar actualizada antes de iniciar esta tarea.
Confirmar que el campo `Clave` está mapeado en el modelo Entity Framework de `catClasificacionCorreoRecibido` antes de implementar la query.
**D3 — Patrón de filtros/búsqueda/paginación:** El Buzón de Cobros debe aplicar el mismo patrón de filtros, búsqueda y paginación que los Buzones preexistentes (Buzón de Requisiciones, Buzón de Pedidos). El BO debe soportar los mismos parámetros de filtrado que usa `QueryInfo` en esos buzones.
**OBS-021 — Bandeja del Coordinador de Tesorería:** Los correos de cobro no enrutables (cliente sin Cobrador — Caso 1; remitente no dado de alta — Caso 2) se concentran en la bandeja del Coordinador de Tesorería (ver Tarea 8). Esta tarea solo cubre la bandeja del Gestor. La retroactividad al asignar Cobrador es automática por el diseño de la query.

**Descripción del problema:**
No existe BO que liste los correos del Buzón de Cobros filtrados por cobrador y región. El Gestor de Cobranza solo debe ver los correos cuyo cliente pertenece a su cartera (R5) y a su región (E1 — segregación MEX / PER). Sin filtro de región, un cobrador MEX podría ver correos de PER y viceversa.

**Archivos a crear:**
```
Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\vCorreoRecibidoBuzonCobros.cs
Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\CorreoRecibidoClienteBuzonCobrosBO.cs
```

**Cambios requeridos:**

`vCorreoRecibidoBuzonCobros.cs` — modelo POCO del resultado:

```csharp
namespace Logic.Pqf.Logistica.L11.MailBot.Procesos.Cobros
{
    public class vCorreoRecibidoBuzonCobros
    {
        public Guid   IdCorreoRecibidoCliente  { get; set; }
        public Guid   IdCorreoRecibido         { get; set; }
        public Guid   IdCliente                { get; set; }
        public string NombreCliente            { get; set; }
        public string Asunto                   { get; set; }
        public string De                       { get; set; }
        public bool   Leido                    { get; set; }
        public string IdRegion                 { get; set; }   // E1 — segregación MEX / PER
        public System.DateTime FechaRegistro   { get; set; }
    }
}
```

`CorreoRecibidoClienteBuzonCobrosBO.cs` — BO de listado paginado con filtros:

```csharp
using System.Linq;
using Logic.Pqf.Logistica.L11.MailBot.Constantes;

namespace Logic.Pqf.Logistica.L11.MailBot.Procesos.Cobros
{
    public class CorreoRecibidoClienteBuzonCobrosBO
    {
        /// <summary>
        /// Lista paginada del Buzón de Cobros del Gestor autenticado.
        /// Filtros: cobrador (R5 — B3), región (R5 — E1), activo.
        /// Patrón equivalente al Buzón de Requisiciones y Buzón de Pedidos (D3).
        /// </summary>
        public QueryResult<vCorreoRecibidoBuzonCobros> Lista(
            QueryInfo queryInfo,
            Guid idUsuarioCobrador,
            string idRegion)          // E1 — segregación MEX / PER
        {
            using (var context = new ProquifaDotNetEntities())
            {
                var query = from crc in context.CorreoRecibidoCliente
                            join cr  in context.CorreoRecibido
                                on crc.IdCorreoRecibido equals cr.IdCorreoRecibido
                            join cat in context.catClasificacionCorreoRecibido
                                on crc.IdCatClasificacionCorreoRecibido equals cat.IdCatClasificacionCorreoRecibido
                            join ccc in context.ClienteCarteraCliente
                                on crc.IdCliente equals ccc.IdCliente
                            join cc  in context.ClienteCartera
                                on ccc.IdClienteCartera equals cc.IdClienteCartera
                            where cat.Clave            == ClasificacionCorreoRecibidoConstants.Pago  // → .Cobro tras T1
                               && cc.IdUsuarioCobrador == idUsuarioCobrador   // R5 — B3: bandeja personal
                               && cr.IdRegion          == idRegion            // R5 — E1: segregación regional
                               && crc.Activo           == true
                            orderby cr.FechaRegistro descending
                            select new vCorreoRecibidoBuzonCobros
                            {
                                IdCorreoRecibidoCliente = crc.IdCorreoRecibidoCliente,
                                IdCorreoRecibido        = cr.IdCorreoRecibido,
                                IdCliente               = crc.IdCliente,
                                Asunto                  = cr.Asunto,
                                De                      = cr.De,
                                Leido                   = crc.Leido,
                                IdRegion                = cr.IdRegion,
                                FechaRegistro           = cr.FechaRegistro
                            };

                var total = query.Count();
                var items = query
                    .Skip(queryInfo.Skip)
                    .Take(queryInfo.Take)
                    .ToList();

                return new QueryResult<vCorreoRecibidoBuzonCobros>
                {
                    Total   = total,
                    Results = items
                };
            }
        }
    }
}
```

**Criterios de aceptación:**
- [ ] **[B3 — R5]** La query filtra por `cc.IdUsuarioCobrador == idUsuarioCobrador` — solo correos de la cartera del Gestor autenticado
- [ ] **[E1 — R5]** La query filtra por `cr.IdRegion == idRegion` — correos MEX no aparecen en buzón PER y viceversa
- [ ] **[B1 — R3]** La query filtra `cat.Clave == ClasificacionCorreoRecibidoConstants.Pago` — solo correos clasificados como cobro
- [ ] **[D3]** El BO recibe `QueryInfo` y devuelve `QueryResult<vCorreoRecibidoBuzonCobros>` con `Total` y `Results` — patrón equivalente al Buzón de Requisiciones y Buzón de Pedidos
- [ ] **[D3]** Los filtros y parámetros de búsqueda soportados son equivalentes a los Buzones preexistentes
- [ ] Filtra por `crc.Activo == true`
- [ ] Ordena por `FechaRegistro descending`
- [ ] **[Riesgo 1]** Si el cliente no tiene cobrador asignado, el correo no aparece en ninguna bandeja — sin error
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-03 del archivo `TPSC-RE-FU-008-Back.md`
- Reglas de negocio cubiertas: **R3**, **R5**
- Criterios cubiertos: **B1**, **B3**, **D3**, **E1**
- Pendiente P1 del Back: confirmar si `idUsuarioCobrador` se extrae del token o llega como parámetro (impacta firma del BO y del controller)

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 4

### TPSC-RE-FU-008  [ QUERY-CH ]  BuzonCobrosController — endpoint POST /BuzonCobros con QueryInfo

**Aplicativos:**
ProquifaNet 2 — WebApi.Logistica

**Módulos:**
MailBot L11 — Buzón de Cobros

**Consideraciones previas:**
**Depende de Tarea 3.** `CorreoRecibidoClienteBuzonCobrosBO` debe existir y compilar antes de crear el controller.
El endpoint recibe `QueryInfo` en el body (patrón estándar equivalente al Buzón de Requisiciones y Buzón de Pedidos — D3) y retorna `QueryResult<vCorreoRecibidoBuzonCobros>`.
**`idUsuarioCobrador` e `idRegion` se extraen del usuario logueado** (token), no se reciben como parámetros externos. Usar el mecanismo de autenticación del `BaseApiController` para obtenerlos (equivalente al patrón usado en otros buzones). Si `QueryInfo` los transporta por algún motivo técnico, deben validarse contra el token para evitar escalada de privilegios.
**D2 — Sin eliminación directa:** Este controller NO debe exponer ningún endpoint de eliminación individual o múltiple.

**Descripción del problema:**
No existe endpoint que exponga el listado paginado del Buzón de Cobros al frontend. El front requiere `POST /BuzonCobros` con `QueryInfo` con soporte de filtros, búsqueda y paginación equivalente al Buzón de Requisiciones y Buzón de Pedidos (Criterio D3). La región y el cobrador deben resolverse del usuario autenticado, nunca del cuerpo de la petición.

**Archivo a crear:**
```
WebApi.Logistica\Controllers\Procesos\Mailbot\BuzonCobrosController.cs
```

**Cambios requeridos:**

```csharp
using System.Net.Http;
using System.Web.Http;
using System.Web.Http.Description;
using Logic.Pqf.Logistica.L11.MailBot.Procesos.Cobros;

namespace WebApi.Logistica.Controllers.Procesos.Mailbot
{
    public class BuzonCobrosController : BaseApiController
    {
        // POST api/BuzonCobros
        // Body: QueryInfo
        // idUsuarioCobrador e idRegion se obtienen del usuario logueado (token)
        // D3: patrón equivalente al Buzón de Requisiciones y Buzón de Pedidos
        // D2: este controller NO expone endpoints de eliminación
        [HttpPost]
        [Route("BuzonCobros")]
        [ResponseType(typeof(QueryResult<vCorreoRecibidoBuzonCobros>))]
        public HttpResponseMessage Lista([FromBody] QueryInfo queryInfo)
        {
            return TryExecute(() =>
            {
                var idUsuarioCobrador = UsuarioLogueado.IdUsuario;   // del token — R5 / B3
                var idRegion          = UsuarioLogueado.IdRegion;    // del token — E1: segregación MEX / PER

                var bo     = new CorreoRecibidoClienteBuzonCobrosBO();
                var result = bo.Lista(queryInfo, idUsuarioCobrador, idRegion);
                return result != null
                    ? Request.CreateResponse(System.Net.HttpStatusCode.OK, result)
                    : Request.CreateResponse(System.Net.HttpStatusCode.NoContent);
            });
        }
    }
}
```

**Criterios de aceptación:**
- [ ] **[D3 — B3 — E1]** `POST /BuzonCobros` recibe `QueryInfo` en el body y devuelve `QueryResult<vCorreoRecibidoBuzonCobros>` paginado con `Total` y `Results`
- [ ] **[D3]** La paginación, filtros y búsqueda siguen el mismo patrón que el Buzón de Requisiciones y Buzón de Pedidos
- [ ] **[E1 — R5]** `idRegion` e `idUsuarioCobrador` se obtienen del usuario logueado — no se aceptan como parámetros externos ni en el body
- [ ] **[D2 — R9]** El controller NO expone ningún endpoint de eliminación individual o múltiple de correos
- [ ] **[D1 — R8]** La reclasificación manual sigue disponible en `CorreoRecibidoClienteExtensionsController` — verificar que no hay conflicto de rutas
- [ ] El endpoint requiere autenticación (hereda `[Authorize]` de `BaseApiController`)
- [ ] Respuesta `200 OK` con resultados cuando hay correos en el buzón
- [ ] Respuesta `204 NoContent` cuando el buzón está vacío
- [ ] El proyecto `WebApi.Logistica` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-04 del archivo `TPSC-RE-FU-008-Back.md`
- Reglas de negocio cubiertas: **R5**, **R8** (verificar regresión), **R9** (sin eliminación)
- Criterios cubiertos: **B3**, **D1**, **D2**, **D3**, **E1**
- Confirmar nombre de ruta con el equipo de front antes de liberar en DEV


## Tarea 5

### TPSC-RE-FU-008  [ IMP-EXIST-SERVICE ]  Proceso de Clasificación de Cobros — completar CorreoRecibidoClienteToPagoBO

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Logistica

**Módulos:**
MailBot L11 — Proceso de Clasificación / Validar Cobro

**Consideraciones previas:**
**Depende de Tarea 2** (el switch debe estar conectado antes de que este proceso se active).
`CorreoRecibidoClienteToPagoBO` ya existe y ya crea `fccFolioPagoCliente`. Esta tarea revisa, valida y completa su lógica interna para cumplir R3 y R4.
**Nota de implementación:** El pendiente en Validar Cobro se genera **sin capturar previamente datos del cobro** (monto, banco, cuenta, fecha, referencia). La captura ocurre la primera vez que el Gestor trabaja el pendiente en Validar Cobro.
**Nota de implementación:** El **Buzón de Cobros** y **Validar Cobro** NO son el mismo módulo. Este BO genera la entrada en ambos simultáneamente al clasificar el correo.
**OBS-021 — Correos no enrutables:** Si el cliente no tiene Cobrador (Caso 1) o el remitente no está dado de alta (Caso 2), el folio se crea igualmente y el correo aparece en la bandeja del Coordinador de Tesorería (Tarea 8). Registrar en log con motivo. Ver GAP-06 en `TPSC-RE-FU-008-Back.md`.

**Descripción del problema:**
Cuando el MailBot clasifica un correo como cobro, debe generarse automáticamente un pendiente en Validar Cobro (`fccFolioPagoCliente`) y el correo debe ser visible en el Buzón del Gestor asignado. El BO `CorreoRecibidoClienteToPagoBO` existe pero su lógica no ha sido validada para cumplir todos los criterios de FU-008: verificación de cobrador asignado, creación del folio sin datos previos, y marcado de `Procesado = true`.

**Archivos a modificar:**
```
Logic.Pqf.Logistica\L11.MailBot\Procesos\Pagos\CorreoRecibidoClienteToPagoBO.cs
```

**Cambios requeridos:**

```csharp
// Flujo completo requerido por R3 y R4:

// 1. Validar que IdCliente no es nulo (Riesgo 2)
//    Si IdCliente es nulo: Logger.Warn("FU-008: Correo sin cliente identificable — sin folio"); return;

// 2. Verificar cobrador asignado en ClienteCarteraCliente (Riesgo 1)
//    Si no tiene cobrador: Logger.Warn("FU-008: Cliente sin cobrador — folio sin Gestor visible");
//    Continuar de todas formas: el folio se crea pero no aparece en ninguna bandeja

// 3. Crear registro en fccFolioPagoCliente sin datos previos del cobro (R4 — B2):
//    - IdCorreoRecibidoCliente   = crc.IdCorreoRecibidoCliente
//    - IdCliente                 = crc.IdCliente
//    - FechaRegistro             = DateTime.Now
//    - Activo                    = true
//    - FechaUltimaActualizacion  = DateTime.Now
//    NO pre-capturar: Monto, Banco, Cuenta, Referencia, FechaDeposito

// 4. Marcar CorreoRecibidoCliente.Procesado = true

// 5. context.SaveChanges() con manejo de excepción:
//    catch (Exception ex) { Logger.Error("FU-008: Error al crear folio de cobro", ex); }
```

**Criterios de aceptación:**
- [ ] **[B2 — R4]** Al clasificar un correo como cobro, se crea un registro en `fccFolioPagoCliente` con `Activo = true` y sin datos previos del cobro
- [ ] **[B1 — R3]** El correo queda reflejado en el Buzón del Gestor asignado (visible por `CorreoRecibidoClienteBuzonCobrosBO.Lista()`)
- [ ] **[Riesgo 2]** Si `IdCliente` es nulo, no se crea el folio y se registra advertencia en log
- [ ] **[Riesgo 1]** Si el cliente no tiene cobrador asignado, se registra advertencia en log pero el folio se crea igualmente
- [ ] `CorreoRecibidoCliente.Procesado` queda en `true` después de ejecutar el proceso
- [ ] Si ocurre una excepción, se registra en log y no interrumpe el flujo del MailBot
- [ ] Prueba manual: clasificar un correo de prueba → verificar INSERT en `fccFolioPagoCliente` y `Procesado = true`
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-02 del archivo `TPSC-RE-FU-008-Back.md`
- Reglas de negocio cubiertas: **R3**, **R4**
- Criterios cubiertos: **B1**, **B2**
- El cierre del folio (R6 — C1) y la eliminación por inconsistencia (R7 — C2) son responsabilidad de Tarea 6

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 6

### TPSC-RE-FU-008  [ ALG-COMPLX-LOGIC ]  Proceso de Integración y Validación de Cobros — ValidacionCobroBO

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Logistica

**Módulos:**
MailBot L11 — Validar Cobro / Buzón de Cobros

**Consideraciones previas:**
**Depende de Tarea 5.** El folio `fccFolioPagoCliente` debe crearse correctamente antes de implementar la validación.
**Ciclo de vida automático:** El correo aparece en el Buzón al clasificarse (Tarea 5). Se cierra automáticamente al vincularse (R6 — C1). Se elimina automáticamente al marcarse inconsistencia (R7 — C2). El Gestor nunca elimina manualmente desde el Buzón.
**D1 — Reclasificación:** Ya está en `CorreoRecibidoClienteExtensionsController`. Este BO NO reimplementa reclasificación.
**D2 — Sin eliminación directa:** El Buzón no ofrece eliminación. La eliminación del correo (si aplica) es responsabilidad del rol ESAC desde el buzón de Otros.
FU-009 puede extender este BO — diseñar con extensibilidad en mente.

**Descripción del problema:**
El ciclo de vida del folio en el Buzón de Cobros es automático y está acoplado a acciones en Validar Cobro. Se necesita un BO que centralice las validaciones de negocio antes de cada transición de estado (cierre por vinculación — R6; eliminación por inconsistencia — R7), evitando que los controllers apliquen reglas de negocio directamente.

**Criterios de aceptación:**
- [ ] **[C1 — R6]** `VincularAProforma()` cierra el folio (`Activo = false`) solo si existe y está activo — el correo sale del Buzón sin acción manual del Gestor
- [ ] **[C2 — R7]** `MarcarInconsistencia()` cierra el folio (`Activo = false`) solo si existe y está activo — la notificación al cliente pertenece a Validar Cobro
- [ ] Ambos métodos retornan `ValidacionCobroResult.Error()` si el folio no existe o ya está cerrado — sin cierres dobles
- [ ] **[D2 — R9]** El BO NO expone método de eliminación directa por parte del Gestor
- [ ] **[D1 — R8]** La reclasificación en `CorreoRecibidoClienteExtensionsController` sigue funcionando — el correo sale del Buzón de Cobros y aparece en el buzón de destino
- [ ] **[C3]** Al reclasificar a otro buzón, el folio en Validar Cobro refleja el cambio (confirmar con equipo si aplica lógica adicional)
- [ ] El controller existente delega las transiciones a este BO — no aplica reglas de negocio directamente
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Reglas de negocio cubiertas: **R6**, **R7**, **R8** (verificar regresión), **R9** (sin eliminación)
- Criterios cubiertos: **C1**, **C2**, **C3**, **D1**, **D2**
- R6 y R7 ya existen — esta tarea centraliza la lógica dispersa en controllers

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 7

### TPSC-RE-FU-008  [ IMP-EXIST-SERVICE ]  Entrenamiento del MailBot — agregar categoría Cobro al modelo de clasificación

**Aplicativos:**
ConsumidorMailBot — _Ejecutables\_Catalogo\ConsumidorMailBot

**Módulos:**
MailBot L11 — Modelo de clasificación ML.NET (`DescargarCorreosBO`, `ModelBuilder`, `MLModel.zip`)

**Consideraciones previas:**
Esta tarea puede ejecutarse **en paralelo con Tareas 1 y 2**, pero el modelo resultante no estará activo hasta que Tarea 1 y Tarea 2 estén completas y desplegadas.
**Mecanismo de entrenamiento:** El MailBot usa Microsoft ML.NET. El modelo aprende a clasificar correos a partir de un archivo `.vsv` con formato `TipoDocumento~texto` (separador `~`). El modelo actual fue entrenado con el archivo `result-correos_2020_automaticos_15865_sin_proquifa.vsv` y reconoce los tipos: `pago`, `pedido`, `requisicion`, `otros`.
**A2 — R2:** El entrenamiento NO expone criterios configurables al usuario. El ajuste de la clasificación se realiza agregando ejemplos al archivo `.vsv` y regenerando `MLModel.zip`.
**Ciclo de entrenamiento:**
1. Agregar filas de ejemplos de cobro al archivo `.vsv` de entrenamiento
2. Ejecutar `ModelBuilder.CreateModel()` → genera nuevo `MLModel.zip`
3. Desplegar `MLModel.zip` en el servidor donde corre el ConsumidorMailBot
4. Actualizar `_ObtieneClasificacionDeCorreo()` y `clavesValidas` para reconocer el nuevo tipo
**Prerequisito operativo:** Se requiere un conjunto representativo de correos reales de clientes de PROQUIFA clasificados como cobro (comprobantes de pago, transferencias, referencias bancarias) para que el modelo alcance una tasa de acierto aceptable.

**Descripción del problema:**
El modelo ML actual no reconoce la categoría `"cobro"` — solo conoce `pago`, `pedido`, `requisicion` y `otros`. Los correos de cobro solo se clasifican correctamente porque `"pago"` fue el tipo de entrenamiento previo. Al renombrar la categoría a `"cobro"` en BD (Tarea 1), el modelo debe actualizarse para predecir `"cobro"` en lugar de `"pago"`, y el código que mapea la predicción al catálogo debe reconocer este nuevo tipo. Además, `clavesValidas` en el flujo de descarga de correos debe incluir `"cobro"` para que los correos clasificados reciban la etiqueta correcta en Gmail.

**Archivos a modificar:**
```
_Ejecutables\_Catalogo\ConsumidorMailBot\result-correos_2020_automaticos_15865_sin_proquifa.vsv   (agregar filas cobro)
_Ejecutables\_Catalogo\ConsumidorMailBot\ArchivosML\MLModel.zip                                   (regenerar)
_Ejecutables\_Catalogo\ConsumidorMailBot\Library\DescargarCorreos\DescargarCorreosBO.cs           (actualizar switch y clavesValidas)
```

**Cambios requeridos:**

**Paso 1 — Agregar ejemplos de cobro al archivo de entrenamiento `.vsv`:**

El archivo usa separador `~` y formato `TipoDocumento~texto`. Agregar filas con tipo `cobro` y ejemplos de correos reales de comprobantes de pago de clientes de PROQUIFA:

```
TipoDocumento~texto
cobro~Buen día. Adjunto comprobante de transferencia por la factura pendiente. Quedo en espera de confirmación.
cobro~Envío comprobante de pago de la orden 12345. Favor de confirmar recepción.
cobro~Anexo pago de la OC 6198. Saludos.
cobro~Aquí te envío el comprobante de depósito. Quedo en espera de factura.
cobro~Buenas tardes, reenvío el comprobante bancario del pago realizado ayer.
cobro~Adjunto ficha de depósito correspondiente a la proforma 2024-0045.
... (agregar mínimo 200 ejemplos representativos de correos reales de clientes de PROQUIFA)
```

> ⚠️ **Prerequisito:** Recopilar correos reales de clientes de PROQUIFA clasificados como cobro en producción.
> La cantidad mínima recomendada es **200 ejemplos** para que el modelo alcance precisión aceptable.
> Los ejemplos deben representar la diversidad real: distintas redacciones, idiomas (ES), formatos de asunto y cuerpo.

**Paso 2 — Regenerar `MLModel.zip`:**

Ejecutar `ModelBuilder.CreateModel()` desde el proyecto `ConsumidorMailBot` apuntando al `.vsv` actualizado.
El nuevo `MLModel.zip` se genera en `ArchivosML\MLModel.zip`.
Desplegar el nuevo `MLModel.zip` en el servidor de producción del ConsumidorMailBot.

**Paso 3 — Actualizar `DescargarCorreosBO.cs`:**

Actualizar `_ObtieneClasificacionDeCorreo()` para agregar el `case "COBRO"` y buscar por `Clave` en BD (no por display name):

Antes:
```csharp
// Busca por ClasificacionCorreoRecibido (display name) — frágil
case "PAGO":
    catClasificacionCorreoRecibido = catClasificacionCorreoRecibidoBO.Buscar(
        new FilterTuple(Constantes.ClasificacionCorreoRecibido, "Pago"),
        new FilterTuple(Constantes.filtroActivo, true.ToString()));
    break;
```

Después:
```csharp
// FU-008: agregar case "COBRO" — busca por Clave (campo técnico)
case "COBRO":
    catClasificacionCorreoRecibido = catClasificacionCorreoRecibidoBO.Buscar(
        new FilterTuple("Clave", "cobro"),      // busca por Clave, no por display name
        new FilterTuple(Constantes.filtroActivo, true.ToString()));
    break;
// Mantener "PAGO" como fallback durante la transición (eliminar después de Tarea 1 en PROD)
case "PAGO":
    catClasificacionCorreoRecibido = catClasificacionCorreoRecibidoBO.Buscar(
        new FilterTuple("Clave", "cobro"),      // tras Tarea 1, "pago" ya es "cobro" en BD
        new FilterTuple(Constantes.filtroActivo, true.ToString()));
    break;
```

Actualizar `clavesValidas` para incluir `"cobro"`:

Antes:
```csharp
var clavesValidas = new[] { "requisicion", "pedido", "pago", "otros" };
```

Después:
```csharp
// FU-008: agregar "cobro". Mantener "pago" durante transición hasta que Tarea 1 esté en PROD.
var clavesValidas = new[] { "requisicion", "pedido", "cobro", "pago", "otros" };
```

**Criterios de aceptación:**
- [ ] **[A1 — R1]** El modelo ML predice `"cobro"` para correos de comprobantes de pago de clientes
- [ ] **[A1 — R1]** `_ObtieneClasificacionDeCorreo()` tiene `case "COBRO"` que busca por `Clave = "cobro"` en `catClasificacionCorreoRecibido`
- [ ] **[A1 — R1]** `clavesValidas` incluye `"cobro"` — el correo clasificado como cobro recibe la etiqueta correcta en Gmail
- [ ] **[A2 — R2]** La clasificación depende del modelo entrenado — no existen criterios configurables por el usuario en la interfaz
- [ ] El modelo regenerado no rompe la clasificación de `requisicion`, `pedido` y `otros` — prueba de regresión con correos de cada tipo
- [ ] El nuevo `MLModel.zip` está desplegado en el servidor del ConsumidorMailBot
- [ ] La tasa de acierto del modelo para correos de cobro es aceptable (definir umbral con el equipo — sugerido ≥ 70%)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Reglas de negocio cubiertas: **R1** (clasificación automática — categoría Cobro), **R2** (sin criterios configurables)
- Criterios cubiertos: **A1**, **A2**
- `Constantes.PrecisionTolerada` define el umbral mínimo de score para aceptar una predicción — revisar si aplica ajuste para cobro
- El archivo de entrenamiento actual tiene ~15,865 correos de 2020. Los ejemplos de cobro deben ser correos reales de PROQUIFA en producción para reflejar el vocabulario real de los clientes.
- **Propuesta futura (fuera de alcance FU-008):** Incorporar IA para leer el documento adjunto (PDF/imagen del comprobante) y precargar datos en Validar Cobro automáticamente.

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008-Back.md`
- Archivo de entrenamiento actual: `_Ejecutables\_Catalogo\ConsumidorMailBot\result-correos_2020_automaticos_15865_sin_proquifa.vsv`
- Modelo actual: `_Ejecutables\_Catalogo\ConsumidorMailBot\ArchivosML\MLModel.zip`
- Clasificador: `_Ejecutables\_Catalogo\ConsumidorMailBot\Library\DescargarCorreos\DescargarCorreosBO.cs`
- Constructor del modelo: `_Ejecutables\_Catalogo\ConsumidorMailBot\ML\ModelBuilder.cs`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/
---

## Tarea 8

### TPSC-RE-FU-008  [ LIST-PAG-MULT-FILTER ]  BandejaCoordenadorTesoreriaBO — correos de cobro no enrutables

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Logistica

**Módulos:**
MailBot L11 — Buzón de Cobros — Bandeja del Coordinador de Tesorería

**Consideraciones previas:**
**Depende de Tarea 1 y Tarea 2.** La clave `"cobro"` debe existir en BD antes de que esta query filtre correctamente.
**Depende de Tarea 3.** El modelo POCO `vCorreoRecibidoBuzonCobros` debe existir antes de crear este BO.
Esta tarea cubre **OBS-021 (Reglas 11 y 12)**: correos clasificados como cobro que no pueden enrutarse a ningún Gestor van a esta bandeja.
**Retroactividad automática:** Al asignar un Cobrador al cliente (FU-002), los correos del Caso 1 dejan de aparecer en esta bandeja automáticamente y aparecen en la bandeja del nuevo Gestor. No se requiere lógica adicional — es consecuencia estructural de la query por `IdUsuarioCobrador`.
**Solo accesible para Coordinador de Tesorería:** confirmar mecanismo de identificación del rol (P5 del Back).

**Descripción del problema:**
Los correos clasificados como cobro cuyo cliente no tiene Cobrador asignado (Caso 1) o cuyo remitente no está dado de alta (Caso 2) no aparecen en ninguna bandeja de Gestor. OBS-021 establece que deben concentrarse en la bandeja del Coordinador de Tesorería para que tome acción. Se requiere un BO que devuelva estos correos no enrutables con el motivo de no enrutamiento.

**Archivo a crear:**
```
Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\BandejaCoordenadorTesoreriaBO.cs
```

**Cambios requeridos:**
Ver fragmento de código en GAP-05 de `TPSC-RE-FU-008-Back.md`.

**Criterios de aceptación:**
- [ ] **[R11 — F1 OBS-021]** La query retorna correos de cobro activos donde el cliente no tiene Cobrador asignado (Caso 1)
- [ ] **[R12 — F3 OBS-021]** La query retorna correos de cobro activos donde `IdCliente` es nulo (Caso 2)
- [ ] Cada registro incluye el motivo de no enrutamiento (Caso 1 vs Caso 2) para que el Coordinador sepa qué acción tomar
- [ ] Los correos que ya tienen Cobrador asignado NO aparecen en esta bandeja
- [ ] El BO recibe `QueryInfo` y devuelve `QueryResult<vCorreoRecibidoBuzonCobros>` paginado
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-05 del archivo `TPSC-RE-FU-008-Back.md`
- Reglas cubiertas: **R11**, **R12** (OBS-021)
- Criterios cubiertos: **F1**, **F2**, **F3**, **F4** (SECCIÓN F del requisito)

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 9

### TPSC-RE-FU-008  [ QUERY-CH ]  BandejaCoordenadorTesoreriaController — endpoint POST /BandejaCoordenadorTesoreria

**Aplicativos:**
ProquifaNet 2 — WebApi.Logistica

**Módulos:**
MailBot L11 — Buzón de Cobros — Bandeja del Coordinador de Tesorería

**Consideraciones previas:**
**Depende de Tarea 8.** `BandejaCoordenadorTesoreriaBO` debe existir y compilar antes de crear el controller.
**Solo accesible para Coordinador de Tesorería.** El endpoint debe validar el rol del usuario autenticado. Confirmar el mecanismo de identificación del rol con el arquitecto (P5 del Back).
El endpoint retorna el mismo modelo `vCorreoRecibidoBuzonCobros` que usa `BuzonCobrosController` para consistencia de frontend.

**Descripción del problema:**
No existe endpoint que exponga la bandeja del Coordinador de Tesorería. El Coordinador necesita acceder a los correos no enrutables para asignar un Cobrador al cliente (Caso 1) o dar de alta el contacto (Caso 2). OBS-021 establece este módulo como punto de entrada de atención para correos sin ruta.

**Archivo a crear:**
```
WebApi.Logistica\Controllers\Procesos\Mailbot\BandejaCoordenadorTesoreriaController.cs
```

**Criterios de aceptación:**
- [ ] **[F1 OBS-021]** `POST /BandejaCoordenadorTesoreria` retorna correos de cobro no enrutables paginados con `Total` y `Results`
- [ ] El endpoint solo es accesible por el rol Coordinador de Tesorería — retorna `403 Forbidden` para cualquier otro rol
- [ ] El endpoint retorna `200 OK` con resultados cuando hay correos no enrutables, `204 NoContent` si no hay
- [ ] El modelo de respuesta es `QueryResult<vCorreoRecibidoBuzonCobros>` — consistente con `BuzonCobrosController`
- [ ] El proyecto `WebApi.Logistica` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-05 del archivo `TPSC-RE-FU-008-Back.md`
- Reglas cubiertas: **R11**, **R12** (OBS-021)

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`


---

## Tarea 10

### TPSC-RE-FU-008  [ QUERY-G ]  Análisis ETL Legacy — Mapeo de datos Buzón de Cobros y definición de trigger

**Aplicativos:**
ProquifaDotNet.NuevoAplicativoLegacy / ProquifaNet 2 (consulta) / PConnect

**Módulos:**
Buzón de Cobros — ETL Legacy (Análisis)

**Consideraciones previas:**
- ⚠️ **Brecha B3 BLOQUEANTE (compartida con TPSC-RE-FU-028):** El canal de transferencia a Legacy (tabla ETL, cola RabbitMQ, API directa) no está definido. Esta tarea contribuye a su resolución desde la perspectiva del Buzón de Cobros.
- La transferencia E1 (Datos Buzón de Cobros) está documentada en `TPSC-RE-FU-028-Back.md` Parte E como dependencia de RE-008. Esta tarea define el **qué y cuándo** desde el origen.
- El análisis debe coordinarse con el equipo de arquitectura y con el equipo Legacy para confirmar las tablas/directorio destino y el campo de referencia cruzada.
- Precede a Tarea 11 (implementación builders) y Tarea 12 (canal definitivo). No puede avanzar implementación sin los resultados de este análisis.
- El disparador de E1 no está resuelto: puede ocurrir al clasificar el correo (RE-008), al confirmar el cobro en Paso 1 (RE-024) o al completar el Paso 3 (RE-028). Esta tarea debe definirlo formalmente.

**Decisiones ya tomadas (resultado de análisis previo):**
- **Archivos a transferir:** Se transfieren **todos** los archivos adjuntos del correo, obtenidos desde `ArchivoCorreoRecibido` (N:N entre `CorreoRecibido` y `Archivo`). No solo el comprobante seleccionado en Paso 1.
- **Destino de archivos:** Los archivos se guardan en un **directorio específico en Legacy**. El equipo Legacy debe confirmar la ruta exacta y si varía por empresa o región.
- **Política de reintentos:** La transferencia reintenta **N veces configurables** (valor en AppSettings o tabla de configuración). Si se agotan los reintentos, se registra en una **bitácora de fallos** con el motivo de error. No queda pendiente manual sin trazabilidad.
- **Ante inconsistencia R7 (anulación del cobro):** El archivo ya transferido **permanece en Legacy como historial**. No se elimina ni se notifica a Legacy sobre la anulación.

**Descripción del problema:**
El Buzón de Cobros genera registros en `fccPagoCliente` y `fccFolioPagoCliente` que Legacy necesita conocer para mantener la continuidad del ciclo de surtido. Adicionalmente, los correos clasificados como cobro tienen adjuntos (comprobantes de pago) almacenados en MinIO que deben copiarse a un directorio en Legacy. Quedan por definir: el trigger exacto de E1, el campo de referencia cruzada, el mapeo columna-a-columna y la ruta destino del directorio en Legacy.

**Objetivos específicos:**
- Definir el **trigger** de E1: ¿se dispara al clasificar el correo (RE-008), al confirmar el cobro Paso 1 (RE-024) o al avanzar al Paso 3 (RE-028)? Documentar la decisión con el equipo.
- Confirmar las **tablas destino en Legacy** que reciben los datos del cobro (registro cobro recibido, vinculación cliente/pedido).
- Identificar el **campo de referencia cruzada** que vincula el cobro de ProquifaDotNet con el registro en Legacy.
- Mapear columna a columna: `fccPagoCliente` (Folio, Monto, FechaPago, FormaPago, Moneda, TipoDeCambio, CuentaDestino, IdCliente) → columnas exactas en Legacy.
- Confirmar con Legacy la **ruta del directorio destino** para los archivos adjuntos (¿varía por empresa emisora o región?).
- Confirmar si los archivos se copian como nombre original o con un esquema de renombrado basado en folio/cliente.
- Definir el **valor inicial de N reintentos** y la **tabla/estructura de la bitácora de fallos** (columnas: IdFolio, FechaIntento, MotivoFallo, NúmeroIntento, Resuelto).
- Documentar el contrato completo E1 como insumo para Tarea 11 y Tarea 12.

**Resultado esperado:**
Documento de análisis E1 completamente resuelto: trigger definido, campo de referencia cruzada identificado, mapeo columna-a-columna hacia Legacy, ruta directorio destino confirmada, estructura de bitácora diseñada, acuerdos documentados con arquitectura y Legacy.

**Entregables:**
- Documento de análisis ETL E1 con:
  - Trigger definitivo (RE-008 / RE-024 / RE-028)
  - Mapeo de campos: `fccPagoCliente` → tablas/columnas Legacy
  - Campo de referencia cruzada ProquifaDotNet ↔ Legacy
  - Ruta del directorio destino en Legacy para archivos adjuntos
  - Esquema de nombre de archivo en Legacy (nombre original o renombrado)
  - Valor inicial de N reintentos y estructura de bitácora de fallos
  - Acuerdos firmados con arquitectura y equipo Legacy
- Script DDL o diseño de la **tabla bitácora de fallos ETL** (puede ser en BD ProquifaDotNet o en Legacy según decisión de arquitectura)
- Actualización de la sección E1 en `TPSC-RE-FU-028-Back.md` con los acuerdos alcanzados

**Criterios de aceptación:**
- [ ] El trigger de E1 está definido y documentado.
- [ ] El campo de referencia cruzada ProquifaDotNet ↔ Legacy está identificado y confirmado con el equipo Legacy.
- [ ] El mapeo columna-a-columna para `fccPagoCliente` está documentado con columnas exactas en Legacy.
- [ ] La ruta del directorio destino en Legacy para los archivos adjuntos está confirmada.
- [ ] El esquema de nombre de archivo en directorio Legacy está definido.
- [ ] El valor de N reintentos y la estructura de la bitácora de fallos están definidos y aprobados.
- [ ] El documento de análisis está disponible y aprobado como prerequisito para Tarea 11 y Tarea 12.

**Más información de la tarea:**
Ver E1 en `TPSC-RE-FU-028-Back.md` — Parte E y Brecha B3. Los archivos adjuntos del correo se acceden desde `ArchivoCorreoRecibido` (N:N) — NO solo desde `fccFolioPagoCliente.IdArchivo`. Los archivos están en MinIO bucket `mailbot` y se recuperan vía `ArchivoBO.Obtener(IdArchivo)`.

**Recursos:**
- `TPSC-RE-FU-008-Back.md` — GAP-02, GAP-03, GAP-06 (modelo de datos del Buzón)
- `TPSC-RE-FU-008_BD.md` — tablas: `fccFolioPagoCliente`, `ArchivoCorreoRecibido`, `Archivo`
- `Logic.Pqf.Logistica/L11.MailBot/Archivos/ArchivoCorreoRecibidoBO.cs` — lógica existente de archivos en MinIO
- `TPSC-RE-FU-028-Back.md` — Parte E, E1, Brecha B3
- Requisito funcional: `TPSC-RE-FU-008.md`

---

## Tarea 11

### TPSC-RE-FU-008  [ SERV-COMPLEX-TRANSACT ]  Implementar payload builder ETL E1 (Buzón de Cobros) e interfaz IEtlBuzonCobrosLegacyService

**Aplicativos:**
ProquifaDotNet.NuevoAplicativoLegacy / ProquifaNet 2 (consulta) / PConnect

**Módulos:**
Buzón de Cobros — ETL Legacy (Implementación)

**Consideraciones previas:**
- **Predecesora: Tarea 10** — Análisis ETL E1 (trigger, mapeo de campos, campo de referencia cruzada). No iniciar sin el documento de análisis aprobado.
- El trigger de E1 determina dónde se invoca este servicio: si es en RE-024 (Paso 1), se invoca desde `ProquifaDotNet.Finanzas` al confirmar el cobro; si es en RE-028 (Paso 3), se invoca como parte del flujo post-envío junto con E2/E3/E6.
- Mientras la Brecha B3 esté pendiente: implementar con stub que registra el payload como `ETL_PENDIENTE` en Serilog sin bloquear el flujo.
- El canal definitivo se implementa en Tarea 12 sin cambiar la interfaz ni los callers.
- Coordinación con `TPSC-RE-FU-028 T18`: si el canal general `IEtlLegacyTransferenciaService` ya fue implementado allá, evaluar si E1 se integra como tipo adicional al mismo servicio o como servicio independiente.

**Descripción del problema:**
No existe la capa de construcción del payload ETL para los datos del Buzón de Cobros hacia Legacy. Una vez definido el mapeo en Tarea 10, se debe implementar el builder que toma los datos de `fccPagoCliente` y `fccBuzonCobro` y construye el payload estructurado listo para ser despachado al canal Legacy.

**Objetivos específicos:**
- Crear `IEtlBuzonCobrosLegacyService` con métodos `EnviarDatosAsync(EtlBuzonCobrosPayload)` y `EnviarArchivosAsync(IEnumerable<ArchivoAdjuntoEtl>)` (o integrar E1 al `IEtlLegacyTransferenciaService` existente de RE-028 T18 según decisión de arquitectura).
- Crear `EtlBuzonCobrosLegacyServiceStub`: implementación que loguea en Serilog como `ETL_PENDIENTE` y retorna éxito simulado.
- Crear `EtlBuzonCobrosPayload` con los campos del análisis de Tarea 10: Folio, Monto, FechaPago, FormaPago, Moneda, TipoDeCambio, CuentaDestino, IdCliente, referencia cruzada Legacy, lista de `ArchivoAdjuntoEtl`.
- Crear `ArchivoAdjuntoEtl`: DTO con `NombreArchivo`, `Bytes` (o URL según formato acordado en T10), `IdArchivo`.
- Implementar `EtlBuzonCobrosPayloadBuilder`: construye el payload de datos desde `fccPagoCliente` usando la referencia cruzada de T10; obtiene **todos** los archivos adjuntos del correo desde `ArchivoCorreoRecibido` filtrado por `IdCorreoRecibido`, descarga bytes desde MinIO via `ArchivoBO.Obtener(IdArchivo)` → `ArchivoDetalle.Url`.
- Implementar política de reintentos: N intentos configurables (AppSettings); al agotar intentos, registrar en la **bitácora de fallos ETL** con: `IdFolioPagoCliente`, `FechaIntento`, `MotivoFallo`, `NumeroIntento`. No lanzar excepción que bloquee el flujo principal.
- Conectar la invocación del builder en el trigger definido en T10 (RE-024 Paso 1 o RE-028 Paso 3).

**Resultado esperado:**
El builder construye correctamente el payload E1 usando el mapeo de Tarea 10. La interfaz permite inyectar stub o implementación real sin cambiar callers. El flujo no se bloquea mientras B3 esté pendiente.

**Entregables:**
- `IEtlBuzonCobrosLegacyService` (interfaz) o extensión de `IEtlLegacyTransferenciaService` con tipo E1
- `EtlBuzonCobrosLegacyServiceStub` (implementación stub con log `ETL_PENDIENTE`)
- `EtlBuzonCobrosPayload` (DTO con campos del análisis T10)
- `EtlBuzonCobrosPayloadBuilder` (construye desde `fccPagoCliente` + `fccBuzonCobro`)
- Pruebas unitarias:
  - Payload construido correctamente con datos reales de `fccPagoCliente`
  - Campo de referencia cruzada Legacy incluido
  - Referencia al adjunto de `fccBuzonCobro` incluida
  - Stub loguea `ETL_PENDIENTE` y no interrumpe el flujo
  - Cobro con inconsistencia (R7): comportamiento definido en T10

**Criterios de aceptación:**
- [ ] El builder produce el payload E1 correcto: datos de `fccPagoCliente` con referencia cruzada Legacy.
- [ ] El payload incluye TODOS los adjuntos del correo obtenidos desde `ArchivoCorreoRecibido`, no solo el de `fccFolioPagoCliente.IdArchivo`.
- [ ] La política de reintentos lee N desde AppSettings y registra en bitácora al agotar intentos sin lanzar excepción al flujo principal.
- [ ] El fallo de descarga de un archivo individual no cancela la transferencia de los demás archivos ni los datos del cobro.
- [ ] La implementación stub loguea `ETL_PENDIENTE` en Serilog y retorna éxito; el flujo no revierte el registro del cobro.
- [ ] La inyección de dependencias permite sustituir stub por implementación real sin cambiar callers.
- [ ] Todas las pruebas unitarias pasan.
- [ ] El código es revisado y aprobado por el equipo.

**Más información de la tarea:**
El mapeo columna-a-columna para construir el payload proviene del análisis de Tarea 10. Coordinar con TPSC-RE-FU-028 T18 para evitar duplicación de la interfaz general de canal ETL.

**Recursos:**
- `TPSC-RE-FU-008-Back.md` — GAP-03, GAP-06, modelo `fccPagoCliente`, `fccBuzonCobro`
- `TPSC-RE-FU-008_BD.md` — tablas fuente ETL
- `TPSC-RE-FU-028-Back.md` — Parte E, E1, Brecha B3
- Análisis de Tarea 10 (mapeo de campos y campo de referencia cruzada)

---

## Tarea 12

### TPSC-RE-FU-008  [ SERV-TRANSACT ]  Canal ETL Legacy definitivo Buzón de Cobros (E1) — integración, pruebas y documentación

**Aplicativos:**
ProquifaDotNet.NuevoAplicativoLegacy / ProquifaNet 2 (consulta)

**Módulos:**
Buzón de Cobros — ETL Legacy (Integración y Validación)

**Consideraciones previas:**
- **Predecesoras: Tarea 10 y Tarea 11.** Esta tarea NO inicia sin ambas completadas y sin la Brecha B3 resuelta.
- Requiere que el canal de transferencia esté definido (resultado de TPSC-RE-FU-028 T17 o de Tarea 10 si B3 se resuelve desde aquí).
- Implementa la clase real que reemplaza `EtlBuzonCobrosLegacyServiceStub` sin modificar la interfaz ni los callers.
- Si el canal es RabbitMQ: configurar cola, exchange, binding y política de reintento para E1.
- Si el canal es tabla ETL: confirmar que el proceso lector en Legacy (SSIS u otro) consume el tipo E1.
- Si el canal es API Legacy directa: documentar endpoint, contrato y autenticación para el cobro.
- Las pruebas de integración deben ejecutarse en ambiente QA/staging con Legacy real.
- El fallo en el canal no revierte el registro del cobro en `fccPagoCliente`; manejar con log, reencola o bandera de reintento según política definida en Tarea 10.
- Validar el comportamiento ante inconsistencia (R7): si el cobro se marca inconsistencia en ProquifaDotNet después de la transferencia, confirmar si Legacy recibe notificación de anulación.

**Descripción del problema:**
Una vez construido el payload E1 en Tarea 11, se debe conectar el canal de transferencia definitivo hacia Legacy. Sin esta tarea, el cobro del Buzón solo queda logueado como `ETL_PENDIENTE` en Serilog y Legacy no recibe la información necesaria para continuar el ciclo de surtido.

**Objetivos específicos:**
- Implementar `EtlBuzonCobrosLegacyService` real según el mecanismo definido en Tarea 10 y TPSC-RE-FU-028 T17.
- Registrar la implementación real en el contenedor DI en sustitución del stub.
- Validar en Legacy que el registro del cobro se almacena correctamente en las tablas destino con los datos esperados.
- Ejecutar pruebas de integración end-to-end: clasificación correo → generación cobro → payload E1 → registro en Legacy.
- Validar el comportamiento ante inconsistencia (R7): cobro anulado en ProquifaDotNet → Legacy notificado o bandera en payload.
- Verificar que el fallo de canal no cambia el estado del cobro en `fccPagoCliente`.
- Documentar resultados de pruebas: casos ejecutados, evidencias, incidencias y resolución.
- Actualizar la sección E1 en `TPSC-RE-FU-028-Back.md` confirmando la implementación como completada.

**Resultado esperado:**
El payload E1 se transfiere exitosamente a Legacy a través del canal definitivo. Las pruebas de integración están ejecutadas y documentadas. La transferencia del Buzón de Cobros a Legacy queda formalmente cerrada.

**Entregables:**
- `EtlBuzonCobrosLegacyService` real (implementación definitiva del canal)
- Configuración del canal según tecnología: DDL tabla ETL / configuración RabbitMQ / contrato API Legacy para E1
- Pruebas de integración documentadas:
  - E1 completo: cobro visible en Legacy vinculado al cliente y al pedido
  - Campos mapeados correctamente (Folio, Monto, FormaPago, TC, referencia cruzada)
  - Adjunto/referencia de `fccBuzonCobro` incluido en Legacy
  - Fallo de canal: estado del cobro en `fccPagoCliente` no cambia; payload logueado o reencolado
  - Inconsistencia R7: comportamiento ante anulación posterior validado con equipo Legacy
- Documento de resultados de pruebas (casos, evidencias, incidencias y resolución)
- Actualización de E1 en `TPSC-RE-FU-028-Back.md` como implementado

**Criterios de aceptación:**
- [ ] El payload E1 llega a Legacy con todos los datos correctos validados en tablas destino.
- [ ] El campo de referencia cruzada ProquifaDotNet ↔ Legacy está presente y correcto en Legacy.
- [ ] El fallo de canal no altera el estado de `fccPagoCliente` ni `fccFolioPagoCliente`.
- [ ] El comportamiento ante inconsistencia (R7) está validado y documentado.
- [ ] Las pruebas de integración están ejecutadas con evidencias en ambiente QA/staging.
- [ ] La implementación real pasa revisión de código del equipo.
- [ ] E1 actualizado como implementado en `TPSC-RE-FU-028-Back.md`.

**Más información de la tarea:**
Esta tarea cierra el ciclo ETL de RE-008 para Legacy. El resto de los ETL del flujo Validar Cobro (E2 Proforma, E3 Factura, E6 PDF) se implementan en TPSC-RE-FU-028 T18/T19.

**Recursos:**
- `TPSC-RE-FU-008-Back.md` — GAP-03, GAP-06, modelo `fccPagoCliente`, `fccBuzonCobro`
- `TPSC-RE-FU-008_BD.md` — tablas fuente ETL
- `TPSC-RE-FU-028-Back.md` — Parte E, E1, Brecha B3, Consideraciones transversales ETL
- Análisis de Tarea 10 (documento de mapeo y definición de canal)
- Entregables de Tarea 11 (interfaz + builder + stub)
