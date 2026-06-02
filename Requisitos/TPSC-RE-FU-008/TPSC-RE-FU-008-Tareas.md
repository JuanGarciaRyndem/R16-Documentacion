# TPSC-RE-FU-008 — Tareas de Implementación Backend

| Campo | Valor |
|-------|-------|
| **Requisito** | TPSC-RE-FU-008 |
| **Nombre** | Buzón de Cobros |
| **Total de tareas** | 7 |
| **Revisión aplicada** | TPSC-RE-FU-008-Back.md |

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
**Riesgo 1 — Cliente sin cobrador:** Si el cliente del correo no tiene cobrador asignado (`IdUsuarioCobrador` nulo en `ClienteCarteraCliente`), el correo no aparece en ninguna bandeja. Comportamiento por diseño.
**Riesgo 2 — Cliente no identificable:** Si `IdCliente` es nulo (cliente nuevo, referencia errónea, dominio genérico), el correo no tiene entrada asignable a ningún cobrador. ⚠️ Pendiente confirmación con el cliente.

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
**Riesgo 1 — Cliente sin cobrador:** Si el cliente no tiene cobrador asignado, el folio se crea pero no aparece en ninguna bandeja. Registrar advertencia en log.
**Riesgo 2 — Cliente no identificable:** Si `IdCliente` es nulo, no se crea el folio. Registrar en log. ⚠️ Confirmación pendiente con el cliente.

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
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`


