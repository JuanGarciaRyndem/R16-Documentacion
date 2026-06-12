# R16A-NO-FU-002 — Auditoría Transaccional: IdTransaccion y Agrupación de BitacoraCRUD

## 1. Objetivo

Agregar un mecanismo de correlación transaccional a ProquifaDotNet que permita agrupar todos los registros de `BitacoraCRUD` generados dentro de una misma petición HTTP atómica bajo un `IdBitacoraTransaccion` único, registrando además la tabla principal afectada, el usuario, la IP, el endpoint y un resumen JSON de todos los cambios de la transacción.

**El sistema complementa `BitacoraCRUD`; no lo reemplaza.**

---

## 2. Estado Actual

### 2.1 Tabla `BitacoraCRUD`

Cada operación CRUD registra una fila con los siguientes campos:

| Campo                      | Tipo                  | Descripción                                                              |
| -------------------------- | --------------------- | ------------------------------------------------------------------------ |
| `IdBitacoraCRUD`           | `uniqueidentifier` PK | Identificador del registro                                               |
| `IdUsuario`                | `uniqueidentifier`    | Usuario que ejecutó la acción                                            |
| `IdCatBitacoraAccion`      | `uniqueidentifier` FK | Acción: Agregar / Actualizar / Borrado lógico / Cancelacion / Solicitud  |
| `TablaAfectada`            | `nvarchar`            | Nombre de la entidad afectada (e.g. `ppPedido`, `ppPartidaPedido`)       |
| `IdRegistroAfectado`       | `uniqueidentifier`    | PK del registro modificado                                               |
| `Detalle`                  | `nvarchar(MAX)`       | JSON con diferencias campo a campo `{"Campo":[valorAntes,valorDespues]}` |
| `DireccionIP`              | `nvarchar`            | IP de origen de la petición                                              |
| `InfoDispositivo`          | `nvarchar`            | User-Agent del navegador                                                 |
| `Origen`                   | `nvarchar`            | Endpoint HTTP (método + ruta)                                            |
| `FechaRegistro`            | `datetime`            | Fecha/hora UTC del registro                                              |
| `FechaUltimaActualizacion` | `datetime`            | Timestamp de última modificación                                         |
| `Activo`                   | `bit`                 | Borrado lógico                                                           |

### 2.2 Catálogo `catBitacoraAccion`

Valores conocidos: `Agregar`, `Actualizar`, `Borrado lógico`, `Cancelacion`, `Solicitud`.

### 2.3 Patrones de registro de bitácora

#### Patrón A — `BitacoraMovimientos_<T>` (operaciones CRUD simples)

Clase genérica en `Logic.Pqf.Catalogos.ServiciosSistema`. Invocada desde `CrudBO<T>` base para operaciones de alta, actualización y borrado lógico simples. Detecta automáticamente el tipo de acción y calcula diferencias campo a campo por reflexión.

```csharp
public Guid ProcesarBitacora(T actualizado)
{
    diferencias = ObtenerDiferencias(); // reflexión: antes vs después
    bitacora.TablaAfectada = typeof(T).Name;
    bitacora.Detalle = diferencias;     // JSON de cambios
    db.BitacoraCRUD.Add(bitacora);
    db.SaveChanges();
}
```

Cada llamada genera **una fila independiente** en `BitacoraCRUD`.

#### Patrón B — `BitacoraMovimientos<T>` (servicios transaccionales)

Clase genérica en ensamblado `Core` (DLL). Usada en los BOs transaccionales. Expone `GenerarBitacoraCRUD(idAccion, idRegistro)` que construye la entidad `BitacoraCRUD` en memoria sin guardarla; el BO la agrega al DbContext y aplica `SaveChanges` dentro de su `DbContextTransaction`.

```csharp
// ppPedidoCancelacionBO.cs
BitacoraMovimientos<ppPedido> _bitacorappPedido = new BitacoraMovimientos<ppPedido>(Logger);
var regBitacoraPedido = _bitacorappPedido.GenerarBitacoraCRUD(idAccion, ppPedido.IdPPPedido);

BitacoraMovimientos<ppPartidaPedido> _bitacorappPartida = new BitacoraMovimientos<ppPartidaPedido>(Logger);
foreach (var partida in partidas)
{
    var regBitacoraPartida = _bitacorappPartida.GenerarBitacoraCRUD(idAccion, partida.IdPPPartidaPedido);
    listaResgitroBitacoraPpPartida.Add(regBitacoraPartida);
}
// Guarda todos al final dentro de la transacción
db.BitacoraCRUD._AddOrUpdateUniqueElement(regBitacoraPedido, columnaUnica, out _);
listaResgitroBitacoraPpPartida.ForEach(r => db.BitacoraCRUD._AddOrUpdateUniqueElement(r, columnaUnica, out _));
db.SaveChanges();
dbContextTransaccion.Commit();
```

### 2.4 BOs transaccionales que ya usan BitacoraCRUD

De los **43 BOs** que usan `BeginTransaction` en el repositorio, solo **3** ya integran `BitacoraMovimientos<T>`:

| BO                                   | Tablas auditadas              |
| ------------------------------------ | ----------------------------- |
| `ppPedidoCancelacionBO`              | `ppPedido`, `ppPartidaPedido` |
| `ppPedidosSolicitarFEATransaccionBO` | `ppPedido`, `ppPartidaPedido` |
| `ProductoTransaccionBO`              | `Producto`                    |

Los 40 BOs restantes ejecutan transacciones atómicas sin dejar ningún rastro en `BitacoraCRUD`.

---

## 3. Brecha Identificada

### Problema principal

Cuando un BO transaccional ejecuta una operación que afecta múltiples tablas (e.g. cancelar un pedido afecta `ppPedido` + N filas de `ppPartidaPedido`), cada tabla genera su propio registro independiente en `BitacoraCRUD`. **No existe ningún vínculo entre ellos.**

Consecuencias:
- Imposible responder "¿qué cambió en una sola petición del usuario?" sin cruzar por timestamp y usuario, lo cual es inexacto.
- Imposible saber cuál fue la **tabla/operación principal** que desencadenó los cambios.
- El JSON de `Detalle` en cada fila solo describe los cambios de *esa tabla*, nunca el contexto de la operación completa.
- Los 40 BOs sin BitacoraCRUD son completamente opacos en auditoría.

### Ejemplo concreto

Un usuario cancela un pedido con 5 partidas. Resultado actual en `BitacoraCRUD`:

| IdBitacoraCRUD | TablaAfectada | IdRegistroAfectado | Acción |
|---|---|---|---|
| `aaa...` | `ppPedido` | `pedido-001` | Cancelacion |
| `bbb...` | `ppPartidaPedido` | `partida-001` | Cancelacion |
| `ccc...` | `ppPartidaPedido` | `partida-002` | Cancelacion |
| `ddd...` | `ppPartidaPedido` | `partida-003` | Cancelacion |
| ... | ... | ... | ... |

**No hay forma de saber que estas 6 filas son de la misma petición.**

---

## 4. Diseño del Nuevo Sistema

### 4.1 Nuevo concepto: `BitacoraTransaccion`

Un registro "encabezado" por petición HTTP. Agrupa todos los `BitacoraCRUD` generados dentro de esa petición bajo un `IdBitacoraTransaccion` único.

**Estructura:**

| Campo                   | Tipo                  | Descripción                                                                                    |
| ----------------------- | --------------------- | ---------------------------------------------------------------------------------------------- |
| `IdBitacoraTransaccion` | `uniqueidentifier` PK | Identificador único de la transacción                                                          |
| `TablaOrigen`           | `nvarchar(150)`       | Tabla principal de la transacción (e.g. `ppPedido`)                                            |
| `IdRegistroOrigen`      | `uniqueidentifier`    | PK del registro principal                                                                      |
| `Operacion`             | `nvarchar(200)`       | Nombre semántico de la operación (e.g. `CancelarPedido`, `SolicitarFEA`, `ActualizarProducto`) |
| `IdCatBitacoraAccion`   | `uniqueidentifier` FK | Acción principal de la transacción                                                             |
| `IdUsuario`             | `uniqueidentifier`    | Usuario que ejecutó la operación                                                               |
| `DireccionIP`           | `nvarchar(45)`        | IP de origen                                                                                   |
| `InfoDispositivo`       | `nvarchar(500)`       | User-Agent                                                                                     |
| `Origen`                | `nvarchar(500)`       | Endpoint HTTP (método + ruta)                                                                  |
| `Detalle`               | `nvarchar(MAX)`       | JSON con resumen agregado de todos los cambios de la transacción                               |
| `TotalTablasAfectadas`  | `int`                 | Cantidad de entidades distintas modificadas                                                    |
| `FechaRegistro`         | `datetime2`           | Fecha/hora UTC de inicio de la transacción                                                     |
| `FechaFinalizacion`     | `datetime2` NULL      | Fecha/hora UTC de commit                                                                       |
| `Activo`                | `bit`                 | Borrado lógico                                                                                 |

**Estructura del JSON en `Detalle`:**

```json
{
  "operacion": "CancelarPedido",
  "tablaOrigen": "ppPedido",
  "idRegistroOrigen": "pedido-001",
  "tablas": [
    {
      "tabla": "ppPedido",
      "idRegistro": "pedido-001",
      "accion": "Cancelacion",
      "cambios": { "Activo": [true, false], "OrdenDeCompra": ["OC-123", "OC-123"] }
    },
    {
      "tabla": "ppPartidaPedido",
      "idRegistro": "partida-001",
      "accion": "Cancelacion",
      "cambios": { "Cancelada": [false, true] }
    }
  ]
}
```

### 4.2 Modificación a `BitacoraCRUD`

Agregar columna `IdBitacoraTransaccion` (nullable) para la correlación:

```sql
ALTER TABLE BitacoraCRUD
    ADD IdBitacoraTransaccion uniqueidentifier NULL
        CONSTRAINT FK_BitacoraCRUD_BitacoraTransaccion
        FOREIGN KEY REFERENCES BitacoraTransaccion(IdBitacoraTransaccion);
```

La columna es **nullable** para mantener compatibilidad con el Patrón A (operaciones CRUD simples) donde no aplica el concepto de transacción grupal.

### 4.3 Nueva clase `BitacoraTransaccionManager`

Clase de coordinación que vive en `Logic.Pqf.Catalogos.ServiciosSistema`. Los BOs transaccionales la instancian al inicio de la operación.

**API propuesta:**

```csharp
public class BitacoraTransaccionManager
{
    // Crea el encabezado BitacoraTransaccion e inicia la agrupación
    public Guid Iniciar(
        string tablaOrigen,
        Guid idRegistroOrigen,
        string operacion,
        Guid idCatBitacoraAccion);

    // Registra un BitacoraCRUD en memoria y lo vincula al IdBitacoraTransaccion activo
    public void AgregarMovimiento(BitacoraCRUD bitacoraCRUD);

    // Persiste BitacoraTransaccion + BitacoraCRUD vinculados dentro del DbContext activo
    // Debe llamarse antes del Commit de la transacción EF
    public void Cerrar(ProquifaDotNetEntities db);
}
```

**Ejemplo de integración en `ppPedidoCancelacionBO`:**

```csharp
// Antes:
var regBitacoraPedido = _bitacorappPedido.GenerarBitacoraCRUD(idAccion, pedido.IdPPPedido);

// Después:
var txManager = new BitacoraTransaccionManager();
txManager.Iniciar("ppPedido", pedido.IdPPPedido, "CancelarPedido", idAccion.IdCatBitacoraAccion);

var regBitacoraPedido = _bitacorappPedido.GenerarBitacoraCRUD(idAccion, pedido.IdPPPedido);
txManager.AgregarMovimiento(regBitacoraPedido);

partidas.ForEach(p => {
    var regPartida = _bitacorappPartida.GenerarBitacoraCRUD(idAccion, p.IdPPPartidaPedido);
    txManager.AgregarMovimiento(regPartida);
});

txManager.Cerrar(db); // persiste BitacoraTransaccion + actualiza FK en cada BitacoraCRUD
dbContextTransaccion.Commit();
```

### 4.4 Alcance de integración — Fase 1

En esta fase se integra `BitacoraTransaccionManager` en los **3 BOs transaccionales que ya usan BitacoraCRUD**:

| BO                                   | Operación            | TablaOrigen |
| ------------------------------------ | -------------------- | ----------- |
| `ppPedidoCancelacionBO`              | `CancelarPedido`     | `ppPedido`  |
| `ppPedidosSolicitarFEATransaccionBO` | `SolicitarFEA`       | `ppPedido`  |
| `ProductoTransaccionBO`              | `ActualizarProducto` | `Producto`  |

Los 40 BOs restantes (sin BitacoraCRUD actual) son candidatos para **Fase 2**, donde además se les agregará la auditoría de tablas afectadas.

---

## 5. Impacto en Base de Datos

### 5.1 Nueva tabla `BitacoraTransaccion`

```sql
CREATE TABLE BitacoraTransaccion (
    IdBitacoraTransaccion   uniqueidentifier    NOT NULL DEFAULT NEWSEQUENTIALID(),
    TablaOrigen             nvarchar(150)       NOT NULL,
    IdRegistroOrigen        uniqueidentifier    NOT NULL,
    Operacion               nvarchar(200)       NOT NULL,
    IdCatBitacoraAccion     uniqueidentifier    NOT NULL,
    IdUsuario               uniqueidentifier    NOT NULL,
    DireccionIP             nvarchar(45)        NOT NULL DEFAULT '',
    InfoDispositivo         nvarchar(500)       NOT NULL DEFAULT '',
    Origen                  nvarchar(500)       NOT NULL DEFAULT '',
    Detalle                 nvarchar(MAX)       NOT NULL DEFAULT '{}',
    TotalTablasAfectadas    int                 NOT NULL DEFAULT 0,
    FechaRegistro           datetime2           NOT NULL DEFAULT SYSUTCDATETIME(),
    FechaFinalizacion       datetime2           NULL,
    Activo                  bit                 NOT NULL DEFAULT 1,
    CONSTRAINT PK_BitacoraTransaccion PRIMARY KEY (IdBitacoraTransaccion),
    CONSTRAINT FK_BitacoraTransaccion_catBitacoraAccion
        FOREIGN KEY (IdCatBitacoraAccion)
        REFERENCES catBitacoraAccion(IdCatBitacoraAccion)
);
```

**Índices:**

```sql
-- Consulta por registro principal (¿qué transacciones tocaron este pedido?)
CREATE INDEX IX_BitacoraTransaccion_TablaOrigen_IdRegistroOrigen
    ON BitacoraTransaccion (TablaOrigen, IdRegistroOrigen);

-- Consulta por usuario y fecha (¿qué hizo este usuario hoy?)
CREATE INDEX IX_BitacoraTransaccion_IdUsuario_FechaRegistro
    ON BitacoraTransaccion (IdUsuario, FechaRegistro DESC);
```

### 5.2 Modificación a `BitacoraCRUD`

```sql
-- Agregar FK a BitacoraTransaccion (nullable para compatibilidad)
ALTER TABLE BitacoraCRUD
    ADD IdBitacoraTransaccion uniqueidentifier NULL;

ALTER TABLE BitacoraCRUD
    ADD CONSTRAINT FK_BitacoraCRUD_BitacoraTransaccion
        FOREIGN KEY (IdBitacoraTransaccion)
        REFERENCES BitacoraTransaccion(IdBitacoraTransaccion);

-- Índice para agrupar todos los registros de una transacción
CREATE INDEX IX_BitacoraCRUD_IdBitacoraTransaccion
    ON BitacoraCRUD (IdBitacoraTransaccion)
    WHERE IdBitacoraTransaccion IS NOT NULL;
```

### 5.3 Diccionario de datos — `BitacoraTransaccion`

| Nombre | Tipo | Descripción |
|---|---|---|
| `IdBitacoraTransaccion` | `uniqueidentifier` PK | Identificador único de la transacción; secuencial para rendimiento de índice |
| `TablaOrigen` | `nvarchar(150)` NOT NULL | Nombre de la entidad principal (tipo de la clase EF) |
| `IdRegistroOrigen` | `uniqueidentifier` NOT NULL | PK del registro principal de la operación |
| `Operacion` | `nvarchar(200)` NOT NULL | Nombre semántico de la operación de negocio |
| `IdCatBitacoraAccion` | `uniqueidentifier` NOT NULL FK | Acción principal (FK a `catBitacoraAccion`) |
| `IdUsuario` | `uniqueidentifier` NOT NULL | Usuario autenticado que ejecutó la operación |
| `DireccionIP` | `nvarchar(45)` NOT NULL | IPv4 o IPv6 de origen; `'Localhost'` si es local |
| `InfoDispositivo` | `nvarchar(500)` NOT NULL | User-Agent del cliente |
| `Origen` | `nvarchar(500)` NOT NULL | Método HTTP + ruta del endpoint (e.g. `PATCH /CancelarPedido`) |
| `Detalle` | `nvarchar(MAX)` NOT NULL | JSON agregado con todas las tablas y cambios de la transacción |
| `TotalTablasAfectadas` | `int` NOT NULL | Cantidad de filas `BitacoraCRUD` asociadas |
| `FechaRegistro` | `datetime2` NOT NULL | UTC de creación del encabezado (antes del commit) |
| `FechaFinalizacion` | `datetime2` NULL | UTC de commit exitoso; NULL si la transacción falló |
| `Activo` | `bit` NOT NULL | Borrado lógico |

**Relaciones:**

| Tabla relacionada | Tipo | Campo |
|---|---|---|
| `catBitacoraAccion` | N:1 | `IdCatBitacoraAccion` → `catBitacoraAccion.IdCatBitacoraAccion` |
| `BitacoraCRUD` | 1:N | `BitacoraCRUD.IdBitacoraTransaccion` → `BitacoraTransaccion.IdBitacoraTransaccion` |

**Índices:**

| Nombre | Columnas | Tipo |
|---|---|---|
| `PK_BitacoraTransaccion` | `IdBitacoraTransaccion` | Clustered PK |
| `IX_BitacoraTransaccion_TablaOrigen_IdRegistroOrigen` | `TablaOrigen, IdRegistroOrigen` | Non-clustered |
| `IX_BitacoraTransaccion_IdUsuario_FechaRegistro` | `IdUsuario, FechaRegistro DESC` | Non-clustered |

---

## 6. Impacto en Backend

### 6.1 Proyectos afectados

| Proyecto                        | Tipo de cambio                                                                          |
| ------------------------------- | --------------------------------------------------------------------------------------- |
| `Logic.Pqf.Catalogos`           | Nueva clase `BitacoraTransaccionManager`                                                |
| `Logic.Pqf.Logistica`           | Integración en `ppPedidoCancelacionBO` y `ppPedidosSolicitarFEATransaccionBO`           |
| `Logic.Pqf.Catalogos.Productos` | Integración en `ProductoTransaccionBO`                                                  |
| `WebApi.Catalogos`              | Nuevo endpoint de consulta `GET /BitacoraTransaccion`                                   |
| EF Model (Core — scaffold)      | Regenerar scaffold para incluir `BitacoraTransaccion` y columna nueva en `BitacoraCRUD` |

### 6.2 Nueva clase `BitacoraTransaccionManager`

**Ubicación:** `Logic.Pqf.Catalogos/ServiciosSistema/BitacoraTransaccionManager.cs`

```csharp
public class BitacoraTransaccionManager
{
    private BitacoraTransaccion _encabezado;
    private readonly List<BitacoraCRUD> _movimientos = new List<BitacoraCRUD>();

    public Guid Iniciar(string tablaOrigen, Guid idRegistroOrigen,
                        string operacion, Guid idCatBitacoraAccion)
    {
        _encabezado = new BitacoraTransaccion
        {
            IdBitacoraTransaccion = Guid.NewGuid(),
            TablaOrigen           = tablaOrigen,
            IdRegistroOrigen      = idRegistroOrigen,
            Operacion             = operacion,
            IdCatBitacoraAccion   = idCatBitacoraAccion,
            IdUsuario             = ObtenerIdUsuarioLogueado(),
            DireccionIP           = ObtenerDireccionIP(),
            InfoDispositivo       = ObtenerDetalleNavegador(),
            Origen                = ObtenerEndpointSolicitado(),
            FechaRegistro         = DateTime.UtcNow,
            Activo                = true
        };
        return _encabezado.IdBitacoraTransaccion;
    }

    public void AgregarMovimiento(BitacoraCRUD bitacoraCRUD)
    {
        bitacoraCRUD.IdBitacoraTransaccion = _encabezado.IdBitacoraTransaccion;
        _movimientos.Add(bitacoraCRUD);
    }

    public void Cerrar(ProquifaDotNetEntities db)
    {
        _encabezado.TotalTablasAfectadas = _movimientos.Count;
        _encabezado.Detalle              = ConstruirDetalleJson();
        _encabezado.FechaFinalizacion    = DateTime.UtcNow;

        db.BitacoraTransaccion.Add(_encabezado);
        _movimientos.ForEach(m => db.BitacoraCRUD._AddOrUpdateUniqueElement(m, columnaUnica, out _));
        db.SaveChanges();
    }

    private string ConstruirDetalleJson()
    {
        // Serializa _movimientos como array de tablas afectadas con su Detalle
    }
}
```

### 6.3 Modificaciones en BOs transaccionales — Fase 1

#### `ppPedidoCancelacionBO`
- Instanciar `BitacoraTransaccionManager` al inicio del método.
- `Iniciar("ppPedido", idppPedido, "CancelarPedido", catBitacoraAccionCancelacion.IdCatBitacoraAccion)`
- Registrar cada `BitacoraCRUD` generado con `AgregarMovimiento(...)`.
- Llamar `Cerrar(db)` antes del `Commit`.

#### `ppPedidosSolicitarFEATransaccionBO`
- Mismo patrón.
- `Iniciar("ppPedido", GMCorreoPedidoSolicitarFEA.IdPPPedido, "SolicitarFEA", catBitacoraAccionSolicitud.IdCatBitacoraAccion)`

#### `ProductoTransaccionBO`
- Mismo patrón.
- `Iniciar("Producto", gMProductoTransaccion.IdProducto, "ActualizarProducto", idAccion)`

### 6.4 Endpoint de consulta

**Nuevo:** `GET /BitacoraTransaccion/{idBitacoraTransaccion}` — devuelve encabezado + lista de `BitacoraCRUD` vinculados.

**Nuevo:** `GET /BitacoraTransaccion?tablaOrigen=ppPedido&idRegistroOrigen={id}` — historial de transacciones por registro principal.

Estos endpoints se agregan al controlador existente `BitacoraCRUDController` o en un nuevo `BitacoraTransaccionController`.

### 6.5 Scaffold EF

Al agregar `BitacoraTransaccion` y la columna `IdBitacoraTransaccion` en `BitacoraCRUD`, se debe regenerar el modelo EF en el ensamblado `Core.Pqf.ProquifaDotNetContext` (o actualizar manualmente la clase parcial si el scaffold es administrado).

---

## 7. Consideraciones

- **Rollback**: si la transacción EF hace rollback, `BitacoraTransaccion` y los `BitacoraCRUD` vinculados se revierten junto con el resto — consistencia garantizada por la transacción.
- **Compatibilidad**: los `BitacoraCRUD` del Patrón A (CRUD simples) siguen funcionando sin cambios; su columna `IdBitacoraTransaccion` queda `NULL`.
- **Fase 2**: los 40 BOs transaccionales sin auditoría pueden integrarse posteriormente usando el mismo `BitacoraTransaccionManager`, agregando `BitacoraMovimientos<T>` donde haga falta.
- **`catBitacoraAccion`**: si se requieren nuevas acciones semánticas (e.g. `TramitarPedido`, `PretramitarPedido`) deben insertarse en el catálogo antes de integrar esos BOs.
