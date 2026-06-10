# TPSC-NO-FU-002 — Tareas

## T1 — [ TPSC-NO-FU-002 ] [CREATE-TABL-M] Crear tabla BitacoraTransaccion en ProquifaDotNet

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Sistema — Auditoría

**Consideraciones previas:**
- Nueva tabla `BitacoraTransaccion` en la base de datos `ProquifaDotNet`.
- Actúa como encabezado de transacción: agrupa múltiples filas de `BitacoraCRUD` bajo un `IdBitacoraTransaccion` único.
- La columna `Detalle` almacena el JSON agregado de todos los cambios de la transacción.
- `NEWSEQUENTIALID()` como default en PK para rendimiento de índice clustered.
- FK a `catBitacoraAccion` — catálogo ya existente en ProquifaDotNet.
- Deben crearse 2 índices adicionales para las consultas frecuentes: por `TablaOrigen + IdRegistroOrigen` y por `IdUsuario + FechaRegistro`.

**Objetivo general:**
Crear la tabla `BitacoraTransaccion` con su estructura completa, FK, índices y script de validación.

**Objetivos específicos:**
- DDL: `CREATE TABLE BitacoraTransaccion` con 14 columnas según diccionario de datos de TPSC-NO-FU-002.
- FK: `FK_BitacoraTransaccion_catBitacoraAccion` → `catBitacoraAccion(IdCatBitacoraAccion)`.
- Índice `IX_BitacoraTransaccion_TablaOrigen_IdRegistroOrigen` (non-clustered).
- Índice `IX_BitacoraTransaccion_IdUsuario_FechaRegistro` (non-clustered, DESC en FechaRegistro).
- Script DML de validación con INSERT + SELECT.

**Resultado esperado:**
Tabla `BitacoraTransaccion` creada en ProquifaDotNet, accesible desde el scaffold EF del proyecto y lista para recibir encabezados de transacción.

**Entregables:**
- Script DDL: `CREATE TABLE BitacoraTransaccion` + índices
- Script de validación

**Propuesta de scripts:**

```sql
-- ============================================================
-- TPSC-NO-FU-002 T1 — BitacoraTransaccion en ProquifaDotNet
-- ============================================================

USE [ProquifaDotNet];
GO

CREATE TABLE [dbo].[BitacoraTransaccion] (
    [IdBitacoraTransaccion] UNIQUEIDENTIFIER NOT NULL CONSTRAINT [DF_BitacoraTransaccion_Id]               DEFAULT NEWSEQUENTIALID(),
    [TablaOrigen]           NVARCHAR(100)    NOT NULL,
    [IdRegistroOrigen]      UNIQUEIDENTIFIER NOT NULL,
    [Operacion]             NVARCHAR(200)    NOT NULL,
    [IdCatBitacoraAccion]   UNIQUEIDENTIFIER NOT NULL,
    [IdUsuario]             UNIQUEIDENTIFIER NULL,
    [DireccionIP]           NVARCHAR(50)     NULL,
    [InfoDispositivo]       NVARCHAR(500)    NULL,
    [Origen]                NVARCHAR(200)    NULL,
    [Detalle]               NVARCHAR(MAX)    NULL,
    [TotalTablasAfectadas]  INT              NOT NULL CONSTRAINT [DF_BitacoraTransaccion_TotalTablasAfectadas] DEFAULT 0,
    [FechaRegistro]         DATETIME2(7)     NOT NULL CONSTRAINT [DF_BitacoraTransaccion_FechaRegistro]       DEFAULT SYSUTCDATETIME(),
    [FechaFinalizacion]     DATETIME2(7)     NULL,
    [Activo]                BIT              NOT NULL CONSTRAINT [DF_BitacoraTransaccion_Activo]              DEFAULT 1,
    CONSTRAINT [PK_BitacoraTransaccion] PRIMARY KEY CLUSTERED ([IdBitacoraTransaccion] ASC),
    CONSTRAINT [FK_BitacoraTransaccion_catBitacoraAccion] FOREIGN KEY ([IdCatBitacoraAccion])
        REFERENCES [dbo].[catBitacoraAccion] ([IdCatBitacoraAccion])
);
GO

-- Índice para consultas por registro afectado (historial de un objeto)
CREATE NONCLUSTERED INDEX [IX_BitacoraTransaccion_TablaOrigen_IdRegistroOrigen]
ON [dbo].[BitacoraTransaccion] ([TablaOrigen] ASC, [IdRegistroOrigen] ASC);
GO

-- Índice para auditoría por usuario y fecha (DESC para consultas de lo más reciente)
CREATE NONCLUSTERED INDEX [IX_BitacoraTransaccion_IdUsuario_FechaRegistro]
ON [dbo].[BitacoraTransaccion] ([IdUsuario] ASC, [FechaRegistro] DESC);
GO

-- -------------------------------------------------------
-- DML: INSERT de validación + limpieza
-- -------------------------------------------------------
DECLARE @IdAccion UNIQUEIDENTIFIER =
    (SELECT TOP 1 [IdCatBitacoraAccion] FROM [dbo].[catBitacoraAccion] WHERE [Activo] = 1 ORDER BY [FechaRegistro]);

INSERT INTO [dbo].[BitacoraTransaccion]
    ([TablaOrigen], [IdRegistroOrigen], [Operacion], [IdCatBitacoraAccion],
     [DireccionIP], [Detalle], [TotalTablasAfectadas])
VALUES (
    'ppPedido',
    NEWID(),
    'CancelarPedido_TEST',
    @IdAccion,
    '127.0.0.1',
    '{"operacion":"CancelarPedido_TEST","tablaOrigen":"ppPedido","tablas":[]}',
    0
);
GO

-- Validar que el registro existe
SELECT [IdBitacoraTransaccion], [TablaOrigen], [Operacion], [FechaRegistro], [Activo]
FROM [dbo].[BitacoraTransaccion]
WHERE [Operacion] = 'CancelarPedido_TEST';
GO

-- Limpieza del registro de prueba
DELETE FROM [dbo].[BitacoraTransaccion] WHERE [Operacion] = 'CancelarPedido_TEST';
GO
```

**Criterios de aceptación:**
- Tabla creada con las 14 columnas del diccionario de datos
- PK con `NEWSEQUENTIALID()` como default
- FK a `catBitacoraAccion` funcional
- 2 índices adicionales creados
- INSERT de prueba + SELECT exitoso

**Más información de la tarea:**
Ver sección 5.1 y 5.3 de TPSC-NO-FU-002.md para DDL completo y diccionario.

**Recursos:**
- TPSC-NO-FU-002.md — Secciones 5.1 y 5.3

---

## T2 — [ TPSC-NO-FU-002 ] [UPDATE-TABL-CH] ALTER TABLE BitacoraCRUD — agregar IdBitacoraTransaccion

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Sistema — Auditoría

**Consideraciones previas:**
- Agregar columna `IdBitacoraTransaccion uniqueidentifier NULL` a la tabla existente `BitacoraCRUD`.
- La columna es **nullable** para mantener compatibilidad con los registros históricos y con el Patrón A (CRUD simples) donde no aplica la agrupación transaccional.
- FK hacia `BitacoraTransaccion(IdBitacoraTransaccion)`.
- Índice filtrado `WHERE IdBitacoraTransaccion IS NOT NULL` para no degradar consultas sobre registros sin transacción.
- Ejecutar **después de T1** — la FK requiere que `BitacoraTransaccion` exista.
- La tabla `BitacoraCRUD` puede tener millones de filas; el `ALTER TABLE` es un cambio de metadatos (columna nullable) que no requiere reconstruir la tabla en SQL Server.

**Objetivo general:**
Modificar `BitacoraCRUD` para agregar la columna de correlación `IdBitacoraTransaccion` con FK e índice.

**Objetivos específicos:**
- `ALTER TABLE BitacoraCRUD ADD IdBitacoraTransaccion uniqueidentifier NULL`.
- `ALTER TABLE BitacoraCRUD ADD CONSTRAINT FK_BitacoraCRUD_BitacoraTransaccion FOREIGN KEY ...`.
- `CREATE INDEX IX_BitacoraCRUD_IdBitacoraTransaccion ... WHERE IdBitacoraTransaccion IS NOT NULL`.
- Verificar que registros existentes tienen `NULL` y no generan errores.

**Resultado esperado:**
`BitacoraCRUD` tiene la columna `IdBitacoraTransaccion` nullable con FK funcional. Los registros históricos conservan `NULL` sin impacto.

**Entregables:**
- Script DDL: `ALTER TABLE BitacoraCRUD` + `CREATE INDEX`
- Script de validación

**Propuesta de scripts:**

```sql
-- ============================================================
-- TPSC-NO-FU-002 T2 — ALTER TABLE BitacoraCRUD
-- Prerequisito: T1 (BitacoraTransaccion debe existir)
-- La tabla BitacoraCRUD puede tener millones de filas;
-- agregar columna nullable es cambio de metadatos en SQL Server
-- (no requiere reconstrucción de tabla ni bloqueo prolongado).
-- ============================================================

USE [ProquifaDotNet];
GO

-- 1. Agregar columna nullable (compatible con registros históricos)
ALTER TABLE [dbo].[BitacoraCRUD]
ADD [IdBitacoraTransaccion] UNIQUEIDENTIFIER NULL;
GO

-- 2. FK hacia BitacoraTransaccion (requiere T1 ejecutado)
ALTER TABLE [dbo].[BitacoraCRUD]
ADD CONSTRAINT [FK_BitacoraCRUD_BitacoraTransaccion]
    FOREIGN KEY ([IdBitacoraTransaccion])
    REFERENCES [dbo].[BitacoraTransaccion] ([IdBitacoraTransaccion]);
GO

-- 3. Índice filtrado — solo cubre las filas transaccionales,
--    no impacta las consultas históricas donde el valor es NULL
CREATE NONCLUSTERED INDEX [IX_BitacoraCRUD_IdBitacoraTransaccion]
ON [dbo].[BitacoraCRUD] ([IdBitacoraTransaccion] ASC)
WHERE [IdBitacoraTransaccion] IS NOT NULL;
GO

-- -------------------------------------------------------
-- Validación: confirmar que registros existentes tienen NULL
-- -------------------------------------------------------
SELECT
    COUNT(*)                                                             AS TotalRegistros,
    SUM(CASE WHEN [IdBitacoraTransaccion] IS NULL     THEN 1 ELSE 0 END) AS RegistrosSinTransaccion,
    SUM(CASE WHEN [IdBitacoraTransaccion] IS NOT NULL THEN 1 ELSE 0 END) AS RegistrosConTransaccion
FROM [dbo].[BitacoraCRUD];
GO
```

**Criterios de aceptación:**
- Columna `IdBitacoraTransaccion` presente en `BitacoraCRUD` como `NULL`able
- FK a `BitacoraTransaccion` funcional
- Índice filtrado creado correctamente
- SELECT sobre registros existentes retorna `NULL` en la nueva columna sin errores

**Más información de la tarea:**
Ver sección 5.2 de TPSC-NO-FU-002.md.

**Recursos:**
- TPSC-NO-FU-002.md — Sección 5.2
- Dependencia: T1 (CREATE TABLE BitacoraTransaccion)

---

## T3 — [ TPSC-NO-FU-002 ] [ALG-BASIC-LOGIC] Crear BitacoraTransaccionManager en Logic.Pqf.Catalogos

**Aplicativos:** ProquifaDotNet

**Módulos:** Logic.Pqf.Catalogos — ServiciosSistema

**Consideraciones previas:**
- Nueva clase `BitacoraTransaccionManager` en `Logic.Pqf.Catalogos/ServiciosSistema/`.
- Reutiliza los métodos privados de `BitacoraMovimientos_<T>` para obtener usuario, IP, dispositivo y endpoint (`ObtenerIdUsuarioLogueado`, `ObtenerDireccionIP`, `ObtenerDetalleNavegador`, `ObtenerEndpointSolicitado`). Considerar refactorizar estos métodos a una clase utilitaria compartida `HttpContextBitacoraHelper`.
- `Iniciar(...)` crea el encabezado `BitacoraTransaccion` en memoria (sin persistir aún).
- `AgregarMovimiento(BitacoraCRUD)` setea `IdBitacoraTransaccion` en la entidad y la agrega a la lista interna.
- `Cerrar(ProquifaDotNetEntities db)` construye el JSON de `Detalle`, actualiza `TotalTablasAfectadas` y `FechaFinalizacion`, persiste el encabezado y todos los movimientos dentro del mismo `DbContext` (que está bajo la transacción EF del BO llamador).
- El scaffold EF debe incluir `BitacoraTransaccion` y la nueva columna de `BitacoraCRUD` antes de implementar esta clase — coordinar con T1 y T2.

**Objetivo general:**
Implementar `BitacoraTransaccionManager` con los tres métodos: `Iniciar`, `AgregarMovimiento` y `Cerrar`.

**Objetivos específicos:**
- Clase `BitacoraTransaccionManager` en `Logic.Pqf.Catalogos.ServiciosSistema`.
- Método `Iniciar(tablaOrigen, idRegistroOrigen, operacion, idCatBitacoraAccion)` → `Guid`.
- Método `AgregarMovimiento(BitacoraCRUD)` → vincula FK y agrega a lista interna.
- Método `Cerrar(db)` → serializa JSON de `Detalle`, persiste encabezado y movimientos.
- Método privado `ConstruirDetalleJson()` → serializa lista de movimientos como JSON estructurado con `tablas`, `cambios`, `accion` por cada `BitacoraCRUD`.
- Reutilizar o extraer helper `HttpContextBitacoraHelper` con métodos de IP, usuario, dispositivo y endpoint.

**Resultado esperado:**
`BitacoraTransaccionManager` funcional. Un BO transaccional puede llamar `Iniciar` → N×`AgregarMovimiento` → `Cerrar` y el resultado en BD es una fila en `BitacoraTransaccion` + N filas en `BitacoraCRUD` con `IdBitacoraTransaccion` poblado.

**Entregables:**
- `BitacoraTransaccionManager.cs`
- `HttpContextBitacoraHelper.cs` (helper de contexto HTTP) — si se refactoriza

**Criterios de aceptación:**
- `Iniciar` crea el encabezado con todos los campos de contexto HTTP (usuario, IP, dispositivo, endpoint)
- `AgregarMovimiento` setea correctamente `IdBitacoraTransaccion` en cada `BitacoraCRUD`
- `Cerrar` persiste `BitacoraTransaccion` + `BitacoraCRUD` en el mismo `DbContext`
- `Detalle` JSON tiene la estructura definida en sección 4.1 con `tablas[]`
- `TotalTablasAfectadas` == cantidad de `BitacoraCRUD` registrados
- `FechaFinalizacion` se llena al cerrar

**Más información de la tarea:**
Ver secciones 4.3 y 6.2 de TPSC-NO-FU-002.md para firma de métodos y pseudocódigo.

**Recursos:**
- TPSC-NO-FU-002.md — Secciones 4.3 y 6.2
- `Logic.Pqf.Catalogos/ServiciosSistema/BitacoraMovimientos_.cs` — referencia de métodos HTTP
- Dependencias: T1, T2

---

## T4 — [ TPSC-NO-FU-002 ] [IMP-EXIST-SERVICE] Integrar BitacoraTransaccionManager en ProductoTransaccionBO

**Aplicativos:** ProquifaDotNet

**Módulos:** Logic.Pqf.Catalogos — Productos

**Consideraciones previas:**
- `ProductoTransaccionBO` en `Logic.Pqf.Catalogos/Productos/ProductoTransaccionBO.cs`.
- Ya usa `BitacoraMovimientos<Producto>` con constructor `(Logger, gMProductoTransaccion.IdMovimiento)`.
- `TablaOrigen = "Producto"`, `Operacion = "ActualizarProducto"`.
- El patrón de integración es el mismo que T4.
- Verificar si `ProductoTransaccionBO` afecta otras tablas además de `Producto` dentro de la misma transacción; si es así, registrar también esos movimientos.

**Objetivo general:**
Integrar `BitacoraTransaccionManager` en `ProductoTransaccionBO` para correlacionar la auditoría de actualización de producto.

**Objetivos específicos:**
- Instanciar `BitacoraTransaccionManager` al inicio del método transaccional.
- `Iniciar("Producto", idProducto, "ActualizarProducto", idCatBitacoraAccion)`.
- `AgregarMovimiento(regBitacoraProducto)`.
- `Cerrar(db)` antes del `Commit`.
- Revisar si hay otras tablas en la transacción y agregarlas.

**Resultado esperado:**
Al actualizar un producto se genera `BitacoraTransaccion` con `TablaOrigen = "Producto"` y los `BitacoraCRUD` correspondientes tienen `IdBitacoraTransaccion` poblado.

**Entregables:**
- `ProductoTransaccionBO.cs` modificado

**Criterios de aceptación:**
- Actualizar producto genera 1 `BitacoraTransaccion` + N `BitacoraCRUD` vinculados
- `TablaOrigen = "Producto"` y `Operacion = "ActualizarProducto"` en el encabezado
- Rollback no deja registros huérfanos

**Recursos:**
- TPSC-NO-FU-002.md — Sección 6.3
- `Logic.Pqf.Catalogos/Productos/ProductoTransaccionBO.cs`
- Dependencia: T3

---

## T5 — [ TPSC-NO-FU-002 ] [ALG-BASIC-LOGIC] Crear endpoint BitacoraTransaccionController — consulta por transacción y por registro

**Aplicativos:** ProquifaDotNet

**Módulos:** WebApi.Catalogos — Sistema — Bitácora

**Consideraciones previas:**
- Nuevo controlador `BitacoraTransaccionController` en `WebApi.Catalogos/Controllers/Sistema/Bitacora/`.
- Dos endpoints de consulta (read-only):
  - `GET /BitacoraTransaccion/{idBitacoraTransaccion}` — devuelve el encabezado + lista de `BitacoraCRUD` vinculados.
  - `GET /BitacoraTransaccion?tablaOrigen={tabla}&idRegistroOrigen={id}` — historial de transacciones del registro.
- Crear BO `BitacoraTransaccionBO` y GM `GMBitacoraTransaccion` con la misma estructura que `BitacoraCRUDBO` / `GMBitacoraCRUD`.
- El `Detalle` de `BitacoraTransaccion` se devuelve deserializado como `JObject` (mismo patrón que `BitacoraCRUDBO._Obtener`).
- Requiere autorización (Bearer token, mismo patrón que el resto de WebApi.Catalogos).

**Objetivo general:**
Exponer dos endpoints REST para consultar transacciones por ID y por registro principal afectado.

**Objetivos específicos:**
- `BitacoraTransaccionBO`: métodos `_Obtener(Guid)` y `_ObtenerPorRegistro(tablaOrigen, idRegistroOrigen)`.
- `GMBitacoraTransaccion`: modelo de presentación con `Detalle` como `JObject` y `Movimientos` como `List<GMBitacoraCRUD>`.
- `BitacoraTransaccionController` con los 2 endpoints GET.
- Paginación en `_ObtenerPorRegistro` usando `QueryInfo`.

**Resultado esperado:**
Un consumidor puede llamar `GET /BitacoraTransaccion/{id}` y obtener el resumen completo de una transacción con todos sus movimientos en un solo response.

**Entregables:**
- `BitacoraTransaccionBO.cs`
- `GMBitacoraTransaccion.cs`
- `BitacoraTransaccionController.cs`

**Criterios de aceptación:**
- `GET /BitacoraTransaccion/{id}` retorna encabezado + lista de movimientos correctamente
- `GET /BitacoraTransaccion?tablaOrigen=ppPedido&idRegistroOrigen={id}` retorna lista paginada
- Sin Bearer token retorna 401
- `Detalle` retorna deserializado como objeto JSON (no string)
- Response 404 si no existe el `IdBitacoraTransaccion`

**Más información de la tarea:**
Ver sección 6.4 de TPSC-NO-FU-002.md.

**Recursos:**
- TPSC-NO-FU-002.md — Sección 6.4
- `WebApi.Catalogos/Controllers/Sistema/Bitacora/BitacoraCRUDController.cs` — referencia de patrón
- Dependencias: T3
