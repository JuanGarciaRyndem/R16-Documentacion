# R16A-RE-FU-006 — Análisis de Impacto Backend

| Campo | Valor |
|-------|-------|
| Requisito | R16A-RE-FU-006 — Referencia Bancaria de Cliente |
| Rama | develop-pack04 |
| Fecha | 2026-06 (rev. tras OBS-013/014/015) |
| Revisión | v2.0 |

> **Cambio de modelo (v2.0):** la versión inicial asumía reconstrucción dinámica de la referencia. Tras OBS-013/014/015, la referencia se persiste en **dos niveles**: (1) referencia vigente del cliente en `ClienteDatosBancarios.ReferenciaVigente`, y (2) snapshot inmutable en `tpProformaPedido.ReferenciaPago` al generar el PDF. Esto cambia los GAP-03 y GAP-04 y agrega el GAP-07 (regeneración por cambio en `Cliente.Nombre`/`Clave`).

---

## 1. Estructura actual en el proyecto

### 1.1 Capa de Lógica

| Archivo                                                                                  | Clase                         | Descripción                                                                                |
| ---------------------------------------------------------------------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------ |
| `Logic.Pqf.Catalogos\Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.cs`            | `DatosBancariosBO`            | CRUD de cuentas bancarias del grupo PROQUIFA.                                              |
| `Logic.Pqf.Catalogos\Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.Extensions.cs` | `DatosBancariosBO` Extensions | Filtros especiales por `IdEmpresa`, `IdConfiguracionPagos`, `IdCatMedioDePago`.            |
| `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.cs`                 | `EmpresaDatosBancariosBO`     | Catálogo de cuentas del grupo PROQUIFA — fuente del selector de pantalla.                  |
| `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs`     | `tpProformaPedidoFactory`     | Fábrica que construye `tpProformaPedido`. Asigna `ReferenciaPago = null` en estado actual. |
| *(no existe)*                                                                            | `ClienteDatosBancariosBO`     | **NUEVO R16** — Gestión de relación cliente-cuenta-CódigoValidador.                        |
| *(no existe)*                                                                            | `ReferenciaBancariaBO`        | **NUEVO R16** — Algoritmo de construcción de referencia bancaria (Banamex/no-Banamex).     |

### 1.2 Capa de API (`WebApi.Catalogos`)

| Archivo | Ruta HTTP | Estado |
|---------|-----------|--------|
| `Controllers\Configuracion\Clientes\Relaciones\vClienteDatosBancariosController.cs` | `/vClienteDatosBancarios` | Existente — vista de consulta |
| `Controllers\Configuracion\Cuentas\DatosBancariosController.cs` | `/DatosBancarios` | Existente — CRUD de cuentas bancarias del grupo |
| `Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs` | `/EmpresaDatosBancarios` | Existente — catálogo de cuentas empresa |
| *(no existe)* | `/ClienteDatosBancarios` | **NUEVO R16** — CRUD de relación cliente-cuenta-CódigoValidador |

---

## 2. Mapeo de reglas de negocio al código

| Regla | Descripción | Implementación esperada | Estado actual |
|-------|-------------|-------------------------|---------------|
| **R1** — Pantalla Referencia de Pago en Cobros | Nueva sub-sección en Cobros del cliente. | Controller `/ClienteDatosBancarios` + BO nuevo. | No existe |
| **R2** — Una o más cuentas por cliente | Relación N:N entre `Cliente` y `DatosBancarios`. | Tabla `ClienteDatosBancarios` con FK a ambos. | Tabla no existe |
| **R3** — Código Validador por combinación cliente-cuenta | `CodigoValidador` único por `IdCliente + IdDatosBancarios`. | Campo `CodigoValidador` en `ClienteDatosBancarios`. | Tabla no existe |
| **R4** — Persistencia INSERT/UPDATE | Guardar o actualizar combinación cliente-cuenta-CódValidador. | `_GuardarOActualizar` en `ClienteDatosBancariosBO`. | BO no existe |
| **R4-N1** — Referencia vigente del cliente | Persistir en `ClienteDatosBancarios.ReferenciaVigente`; armada al CREATE/UPDATE de la asignación. | Llamar `ReferenciaBancariaBO.Construir()` en `ClienteDatosBancariosBO._GuardarOActualizar` y persistir resultado. | Campo no existe; BO no existe |
| **R4-N2** — Snapshot casado a la proforma | Copiar `ReferenciaVigente` a `tpProformaPedido.ReferenciaPago` al generar PDF. | En `tpProformaPedidoFactory`, **leer** `ReferenciaVigente` (no recalcular) y asignarla. | `ReferenciaPago` existe pero se asigna `null` |
| **R5** — Generación al configurar cuenta + casado al PDF | Armar referencia al CREATE/UPDATE de cuenta, casar al PDF en firme. | Hook en escritura de `ClienteDatosBancarios` + lectura en `tpProformaPedidoFactory`. | No implementado |
| **R6** — Referencia no-Banamex = nombre del cliente | `Cliente.Nombre` como cadena directa sin transformación. | `ReferenciaBancariaBO.Construir()` retorna `cliente.Nombre`. | BO no existe |
| **R7** — Referencia Banamex = 7 segmentos | Concatenación determinista: 3 letras + 4 chars clave + código banco + P/D + CódValidador. | `ReferenciaBancariaBO.ConstruirBanamex()`. | BO no existe |
| **R8** — Identificación Banamex por `Clave = "002"` | Cruce con `catBanco.Clave = "002"` (simplificación propuesta). | Consulta `catBanco` en el BO de referencia. | Pendiente confirmar con desarrollo |
| **R9** — Sin restricción de rol | Sin control de rol. Acceso por cartera del cliente. | Sin middleware de rol en controller. | Sin restricción en controllers de cliente — correcto, pendiente confirmar con cliente |

---

## 3. GAPs identificados y código de implementación

### GAP-01 — Tabla `ClienteDatosBancarios` no existe

**Archivo:** BD — nueva tabla
**Impacto:** Reglas R2, R3, R4 (ambos niveles) — sin la tabla no es posible persistir la relación cliente-cuenta, el Código Validador ni la referencia vigente.
**Cambio requerido:** Crear la tabla con todos los campos (referencia vigente). El historial del Código Validador ya no lleva columnas propias — se registra en ProquifaDotNet.BitacoraCambios (ver GAP-06). Ver DDL completo en `R16A-RE-FU-006_BD.md` sección 1.

```sql
CREATE TABLE dbo.ClienteDatosBancarios
(
    IdClienteDatosBancarios  uniqueidentifier NOT NULL
        CONSTRAINT PK_ClienteDatosBancarios PRIMARY KEY
        CONSTRAINT DF_ClienteDatosBancarios_Id DEFAULT (NEWSEQUENTIALID()),
    IdCliente                uniqueidentifier NOT NULL
        CONSTRAINT FK_ClienteDatosBancarios_Cliente
            FOREIGN KEY REFERENCES dbo.Cliente(IdCliente),
    IdDatosBancarios         uniqueidentifier NOT NULL
        CONSTRAINT FK_ClienteDatosBancarios_DatosBancarios
            FOREIGN KEY REFERENCES dbo.DatosBancarios(IdDatosBancarios),
    CodigoValidador          varchar(50)      NOT NULL,   -- longitud provisional; confirmar P2
    -- OBS-013: referencia vigente del cliente (Regla 4 nivel 1)
    ReferenciaVigente        varchar(80)      NULL,
    FechaReferenciaVigente   datetime         NULL,
    FechaRegistro            datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaRegistro DEFAULT (GETDATE()),
    FechaUltimaActualizacion datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaActualizacion DEFAULT (GETDATE()),
    Activo                   bit              NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_Activo DEFAULT (1)
);

CREATE UNIQUE NONCLUSTERED INDEX UX_ClienteDatosBancarios_ClienteCuentaActiva
    ON dbo.ClienteDatosBancarios (IdCliente, IdDatosBancarios)
    WHERE Activo = 1;

CREATE NONCLUSTERED INDEX IX_ClienteDatosBancarios
    ON dbo.ClienteDatosBancarios (IdCliente, IdDatosBancarios, Activo);
```

---

### GAP-02 — `ClienteDatosBancariosBO` no existe (CRUD + armado de referencia vigente)

**Archivo a crear:** `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ClienteDatosBancariosBO.cs`
**Impacto:** Reglas R4 (nivel 1) y R5 — no es posible insertar/actualizar la relación cliente-cuenta-CódValidador ni armar la `ReferenciaVigente` al guardar.
**Cambio requerido:** Crear BO usando el patrón `TablaGenericaBO<T>`. Al guardar, invocar `ReferenciaBancariaBO.Construir()` y persistir el resultado en `ReferenciaVigente`. Combinar con el registro del cambio del Código Validador en BitacoraCambios (GAP-06). Depende de GAP-01 y GAP-03.

```csharp
// NUEVO — ClienteDatosBancariosBO.cs
using Core.Pqf.BusinessBasicTools._Misc.Crud;
using Core.Pqf.ProquifaDotNetContext;

namespace Logic.Pqf.Catalogos.Clientes.DatosBancarios
{
    public class ClienteDatosBancariosBO : TablaGenericaBO<ClienteDatosBancarios>
    {
        protected override Guid _GuardarOActualizar(ClienteDatosBancarios entity)
        {
            entity.FechaUltimaActualizacion = DateTime.Now;

            // Regla 4 nivel 1 — Armar y persistir ReferenciaVigente
            using (var db = new ProquifaDotNetEntities())
            {
                var cliente = db.Cliente.FirstOrDefault(x => x.IdCliente == entity.IdCliente);
                var cuenta  = db.DatosBancarios.FirstOrDefault(x => x.IdDatosBancarios == entity.IdDatosBancarios);

                if (cliente != null && cuenta != null)
                {
                    var refBO = new ReferenciaBancariaBO();
                    entity.ReferenciaVigente      = refBO.Construir(cliente, cuenta, entity.CodigoValidador);
                    entity.FechaReferenciaVigente = DateTime.Now;
                }
            }

            return base._GuardarOActualizar(entity);
        }
    }
}
```

---

### GAP-03 — `ReferenciaBancariaBO` no existe (algoritmo de construcción de referencia)

**Archivo a crear:** `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ReferenciaBancariaBO.cs`
**Impacto:** Reglas R5, R6 y R7 — el campo `ReferenciaPago` de `tpProformaPedido` se asigna `null` en la fábrica actual. No existe lógica de construcción.
**Cambio requerido:** Crear la clase con dos métodos: `Construir` (no-Banamex) y `ConstruirBanamex` (7 segmentos). Identificar Banamex por `catBanco.Clave = "002"`.

```csharp
// NUEVO — ReferenciaBancariaBO.cs
using Core.Pqf.BusinessBasicTools._Misc.Crud;
using Core.Pqf.ProquifaDotNetContext;
using System.Linq;

namespace Logic.Pqf.Catalogos.Clientes.DatosBancarios
{
    public class ReferenciaBancariaBO
    {
        private const string ClaveBanamex = "002";

        /// <summary>
        /// Construye la referencia bancaria para una proforma dado el cliente y la cuenta asignada.
        /// </summary>
        public string Construir(Cliente cliente, DatosBancarios cuenta, string codigoValidador)
        {
            var banco = new TablaGenericaBO<catBanco>().Obtener(cuenta.IdCatBanco.Value);

            if (banco?.Clave == ClaveBanamex)
                return ConstruirBanamex(cliente, cuenta, banco, codigoValidador);

            return cliente.Nombre;  // R6 — no-Banamex: nombre del cliente
        }

        /// <summary>
        /// Referencia Banamex: concatenación de 7 segmentos (Regla 7).
        /// </summary>
        private string ConstruirBanamex(Cliente cliente, DatosBancarios cuenta, catBanco banco, string codigoValidador)
        {
            var nombre = cliente.Nombre ?? string.Empty;
            var clave  = cliente.Clave  ?? string.Empty;

            // Segmentos 1-3: primeras tres letras del nombre ignorando espacios, fallback "X"
            // BUG-001: los espacios se ignoran — "BP Farmaceutica" → "BPF", no "BP "
            // Si el nombre sin espacios tiene menos de 3 chars, los faltantes se rellenan con "X"
            // Ejemplos: "BP Farmaceutica" → "BPF" | "GP" → "GPX" | "A" → "AXX"
            var letras = nombre.Replace(" ", string.Empty);
            var seg1 = letras.Length > 0 ? letras[0].ToString() : "X";
            var seg2 = letras.Length > 1 ? letras[1].ToString() : "X";
            var seg3 = letras.Length > 2 ? letras[2].ToString() : "X";

            // Segmento 4: últimos 4 chars de la clave del cliente, padding con ceros
            var seg4 = clave.Length >= 4
                ? clave.Substring(clave.Length - 4)
                : clave.PadLeft(4, "0"[0]);

            // Segmento 5: código del banco
            var seg5 = banco.Clave ?? string.Empty;

            // Segmento 6: P si moneda MXN (primera letra "M"), D en otro caso
            var moneda = new TablaGenericaBO<catMoneda>().Obtener(cuenta.IdCatMoneda.Value);
            var seg6 = (moneda?.ClaveMoneda?.StartsWith("M") == true) ? "P" : "D";

            // Segmento 7: Código Validador
            var seg7 = codigoValidador ?? string.Empty;

            return $"{seg1}{seg2}{seg3}{seg4}{seg5}{seg6}{seg7}";
        }
    }
}
```

---

### GAP-04 — `tpProformaPedidoFactory` no casa `ReferenciaPago` desde `ReferenciaVigente`

**Archivo:** `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs`
**Impacto:** Regla R4 nivel 2 / R5 / Criterio C1 — el campo `ReferenciaPago` existe en `tpProformaPedido` pero se asigna `null`. La referencia vigente persistida nunca se casa al PDF.
**Cambio requerido:** **Leer** `ClienteDatosBancarios.ReferenciaVigente` de la asignación activa del cliente y copiarla a `ReferenciaPago` (snapshot inmutable). **NO recalcular** la referencia aquí — el cálculo vive en `ClienteDatosBancariosBO` al CREATE/UPDATE.

```csharp
// ANTES — tpProformaPedidoFactory.cs (estado actual)
var tpProformaPedido = new tpProformaPedido
{
    // ...
    ReferenciaPago = null,
    // ...
};
```

```csharp
// DESPUÉS — tpProformaPedidoFactory.cs (R4 nivel 2)
// 1. Obtener la asignacion activa del cliente para la cuenta seleccionada del pedido
var clienteDatosBancariosBO = new ClienteDatosBancariosBO();
var clienteCuenta = clienteDatosBancariosBO.ObtenerAsignacionActiva(
    tpPedido.IdCliente,
    tpPedido.IdDatosBancariosSeleccionada);   // ver DUDA-118 sobre seleccion de cuenta destino

// 2. Casar la referencia vigente al PDF (snapshot inmutable)
var tpProformaPedido = new tpProformaPedido
{
    // ...
    ReferenciaPago = clienteCuenta?.ReferenciaVigente,   // OBS-013 — snapshot
    // ...
};
```

---

### GAP-05 — Controller `/ClienteDatosBancarios` no existe

**Archivo a crear:** `WebApi.Catalogos\Controllers\Configuracion\Clientes\ClienteDatosBancariosController.cs`
**Impacto:** Regla R1 — la pantalla Referencia de Pago no tiene endpoint para leer ni guardar la relación cliente-cuenta.
**Cambio requerido:** Controller con `QueryResult`, `Obtener`, `GuardarOActualizar` y `Desactivar`. Sin restricción de rol (pendiente R9 / P4).

```csharp
// NUEVO — ClienteDatosBancariosController.cs
using System;
using System.Web.Http;
using Core.CrudTools.Optimization;
using Core.Pqf.ProquifaDotNetContext;
using Logic.Pqf.Catalogos.Clientes.DatosBancarios;

namespace WebApi.Controllers.Configuracion.Clientes
{
    public class ClienteDatosBancariosController : ApiController
    {
        [HttpPost]
        [Route("ClienteDatosBancarios")]
        public QueryResult<ClienteDatosBancarios> QueryResult([FromBody] QueryInfo info)
        {
            var bo = new ClienteDatosBancariosBO();
            return bo.QueryResult(info);
        }

        [HttpGet]
        [Route("ClienteDatosBancarios")]
        public ClienteDatosBancarios Obtener(Guid idClienteDatosBancarios)
        {
            var bo = new ClienteDatosBancariosBO();
            return bo.Obtener(idClienteDatosBancarios);
        }

        [HttpPut]
        [Route("ClienteDatosBancarios")]
        public Guid GuardarOActualizar([FromBody] ClienteDatosBancarios entity)
        {
            var bo = new ClienteDatosBancariosBO();
            return bo.GuardarOActualizar(entity);
        }

        [HttpDelete]
        [Route("ClienteDatosBancarios")]
        public Guid Desactivar(Guid idClienteDatosBancarios)
        {
            var bo = new ClienteDatosBancariosBO();
            return bo.Desactivar(idClienteDatosBancarios);
        }
    }
}
```

---

### GAP-06 — Historial de `CodigoValidador`: registro en ProquifaDotNet.BitacoraCambios (actualización 2026-07-07, sustituye a OBS-014)

**Archivo:** `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ClienteDatosBancariosBO.cs`
**Impacto:** Al actualizar `CodigoValidador`, el código anterior se pierde. Se requiere trazabilidad completa de los cambios.
**Cambio requerido:** Al guardar o actualizar el `CodigoValidador` en `_GuardarOActualizar`, registrar el cambio en el Aplicativo Nuevo **ProquifaDotNet.BitacoraCambios** (Reglas al diseñar — regla 8) vía `ApiCallerBitacoraCambios`, con: tabla afectada (`ClienteDatosBancarios`), id del registro, campo (`CodigoValidador`), valor anterior, valor nuevo, usuario y fecha. Cada cambio genera un registro nuevo — historial completo, sin límite de niveles. Se **eliminan** las columnas de rotación de un nivel (`CodigoValidadorAnterior`, `FechaModificacionAnterior`, `IdUsuarioModificacionAnterior`) del diseño de `ClienteDatosBancarios`.

```csharp
// DESPUÉS — ClienteDatosBancariosBO.cs con registro en BitacoraCambios (sustituye la rotación OBS-014)
protected override Guid _GuardarOActualizar(ClienteDatosBancarios entity)
{
    entity.FechaUltimaActualizacion = DateTime.Now;

    string codigoAnterior = null;
    if (entity.IdClienteDatosBancarios != Guid.Empty)
    {
        using (var db = new ProquifaDotNetEntities())
        {
            var existente = db.ClienteDatosBancarios
                .FirstOrDefault(x => x.IdClienteDatosBancarios == entity.IdClienteDatosBancarios);
            if (existente != null && existente.CodigoValidador != entity.CodigoValidador)
                codigoAnterior = existente.CodigoValidador;
        }
    }

    var id = base._GuardarOActualizar(entity);

    // Registro del cambio en ProquifaDotNet.BitacoraCambios (regla 8) — alta y cada modificación
    if (entity.IdClienteDatosBancarios == Guid.Empty || codigoAnterior != null)
    {
        _apiCallerBitacoraCambios.RegistrarCambio(
            tabla: "ClienteDatosBancarios",
            idRegistro: id,
            campo: "CodigoValidador",
            valorAnterior: codigoAnterior,          // null en el alta
            valorNuevo: entity.CodigoValidador,
            idUsuario: entity.IdUsuarioModificacion, // pasa desde la capa de API
            fecha: DateTime.Now);
    }

    return id;
}
```

> **Notas:**
> - El usuario del cambio se pasa desde el controller (contexto del usuario autenticado) antes de llamar a `GuardarOActualizar`.
> - El registro en BitacoraCambios NO debe bloquear el guardado: si la llamada falla, se loguea el error (Serilog) y el guardado procede — definir con arquitectura si se requiere reintento.
> - El contrato/endpoint de ProquifaDotNet.BitacoraCambios aún no está documentado en un requisito propio (mismo estado que en RE-016/018) — aquí se referencia el punto de integración, no su detalle técnico.
> - La consulta del historial se hace desde BitacoraCambios; sin UI en R16.

---

### GAP-07 — Regeneración de `ReferenciaVigente` por cambio en `Cliente.Nombre` o `Cliente.Clave`



**Archivos involucrados:** `Logic.Pqf.Catalogos\Clientes\ClienteBO.cs` (o equivalente) + `ClienteDatosBancariosBO.cs`
**Impacto:** Regla 4 nivel 1 — *"solo se regenera si cambia un dato fuente (banco, cuenta, Código Validador o **datos del cliente que la componen**)"*. Los segmentos S1-S3 dependen de `Cliente.Nombre` y S4 depende de `Cliente.Clave`. Si cambian, todas las asignaciones activas del cliente quedan con `ReferenciaVigente` obsoleta.
**Cambio requerido:** Definir el mecanismo de regeneración en cascada. Tres opciones a evaluar con arquitectura:

1. **Hook en `ClienteBO._GuardarOActualizar`** — detectar cambios en `Nombre`/`Clave`, recorrer asignaciones activas y regenerar `ReferenciaVigente` en cada una. Ventaja: transaccional; desventaja: acopla `ClienteBO` con `ClienteDatosBancariosBO`.
2. **Trigger de BD `AFTER UPDATE` en `Cliente`** — calcular y actualizar en T-SQL. Ventaja: garantizado; desventaja: la lógica de armado queda duplicada en BD y en .NET.
3. **Lazy regeneration en `tpProformaPedidoFactory`** — al casar al PDF, comparar timestamps; si `Cliente.FechaUltimaActualizacion > ReferenciaVigente.Fecha`, regenerar antes de copiar. Ventaja: simple, sin cascada; desventaja: nunca se actualiza el campo `ReferenciaVigente` en BD para clientes que no generan proforma.

**Recomendación:** opción 1 (hook en `ClienteBO`) por consistencia transaccional. Documentar como decisión de diseño técnico.

---

## 4. Tablas y entidades del modelo de datos

| Tabla BD | Entidad EF | Propiedades clave R16 | Descripción |
|----------|------------|-----------------------|-------------|
| `ClienteDatosBancarios` | `ClienteDatosBancarios` | `IdClienteDatosBancarios` (PK), `IdCliente` (FK), `IdDatosBancarios` (FK), `CodigoValidador`, `ReferenciaVigente` + audit | **NUEVA R16** — Relación N:N cliente-cuenta con referencia vigente persistida; historial del Código Validador en ProquifaDotNet.BitacoraCambios (GAP-06) |
| `DatosBancarios` | `DatosBancarios` | `IdDatosBancarios` (PK), `IdCatBanco` (FK), `IdCatMoneda` (FK) | Existente — cuenta bancaria del grupo PROQUIFA |
| `catBanco` | `catBanco` | `IdCatBanco` (PK), `Clave` (002 = Banamex) | Existente — catálogo de bancos |
| `catMoneda` | `catMoneda` | `IdCatMoneda` (PK), `ClaveMoneda` (MXN / USD) | Existente — determina P/D en segmento 6 |
| `Cliente` | `Cliente` | `IdCliente` (PK), `Nombre`, `Clave` | Existente — segmentos 1-4 de la referencia Banamex |
| `tpProformaPedido` | `tpProformaPedido` | `ReferenciaPago` (varchar — ya existe, asignado `null`) | Existente — campo donde se escribe la referencia construida |

---

## 5. Consulta SQL de referencia — verificar campo `Clave` del cliente y cuenta Banamex

```sql
SELECT TOP 10
    c.IdCliente, c.Nombre, c.Clave,
    db.IdDatosBancarios, db.NumeroDeCuenta, db.Beneficiario,
    b.Banco, b.Clave AS ClaveBanco,
    m.ClaveMoneda
FROM       dbo.Cliente          c
INNER JOIN dbo.Region           r   ON c.IdRegion       = r.IdRegion
INNER JOIN dbo.EmpresaDatosBancarios edb ON 1=1
INNER JOIN dbo.DatosBancarios   db  ON edb.IdDatosBancarios = db.IdDatosBancarios
INNER JOIN dbo.catBanco         b   ON db.IdCatBanco    = b.IdCatBanco
LEFT  JOIN dbo.catMoneda        m   ON db.IdCatMoneda   = m.IdCatMoneda
WHERE r.ClaveISO = 'MEX'
  AND b.Clave    = '002'
ORDER BY c.Nombre;
```

```sql
SELECT c.IdCliente, c.Nombre, c.Clave, LEN(c.Clave) AS LongitudClave
FROM dbo.Cliente c
INNER JOIN dbo.Region r ON c.IdRegion = r.IdRegion
WHERE r.ClaveISO = 'MEX' AND c.Activo = 1
ORDER BY LEN(c.Nombre) DESC;
```

---

## 6. Resumen de archivos a modificar / crear

| # | Archivo | Tipo de cambio |
|---|---------|----------------|
| 1 | BD — nueva tabla `ClienteDatosBancarios` | CREATE TABLE con `ReferenciaVigente` + índice filtrado por `Activo = 1` (sin columnas de historial) |
| 2 | `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ClienteDatosBancariosBO.cs` | Clase nueva — CRUD + armado/persistencia de `ReferenciaVigente` + registro del cambio de Código Validador en BitacoraCambios |
| 3 | `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ReferenciaBancariaBO.cs` | Clase nueva — algoritmo de referencia (Banamex / no-Banamex), invocada desde el BO al CREATE/UPDATE |
| 4 | `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs` | Modificar — **leer** `ClienteDatosBancarios.ReferenciaVigente` y copiarla a `ReferenciaPago` (snapshot) |
| 5 | `WebApi.Catalogos\Controllers\Configuracion\Clientes\ClienteDatosBancariosController.cs` | Controller nuevo — CRUD `/ClienteDatosBancarios` |
| 6 | `Logic.Pqf.Catalogos\Clientes\ClienteBO.cs` (o equivalente) | Modificar — hook tras actualizar `Cliente`: si cambió `Nombre` o `Clave`, regenerar `ReferenciaVigente` de todas las asignaciones activas (GAP-07) |

---

## 7. Archivos que NO requieren cambio en R16

| Archivo | Motivo |
|---------|--------|
| `DatosBancariosBO.cs` / `DatosBancariosController.cs` | Gestión de cuentas del grupo PROQUIFA — sin cambios estructurales. |
| `EmpresaDatosBancariosBO.cs` / `EmpresaDatosBancariosController.cs` | Fuente del selector de cuentas en pantalla — sin cambios. |
| `vClienteDatosBancariosController.cs` | Vista de consulta existente — sin cambios en R16. |
| `tpProformaPedidoBO.cs` | Lógica de persistencia de proforma — sin cambios directos. |

---

## 8. Pendientes / Decisiones abiertas

| # | Pendiente | Responsable |
|---|-----------|-------------|
| P1 | Confirmar lógica completa de identificación de Banamex (condición de moneda truncada en documento cliente). Propuesta: usar `catBanco.Clave = "002"`. | Desarrollo / Cliente |
| ~~P2~~ | ~~Confirmar longitud máxima y formato del `CodigoValidador`.~~ **[Resuelto]** El Código Validador es **numérico, siempre 2 dígitos con cero a la izquierda** (rango `01`–`99`). El Front siempre envía exactamente 2 caracteres; la columna en BD puede ser mayor (varchar(50) provisional) pero el valor nunca excederá 2 caracteres. | Cerrado |
| P3 | Confirmar si el campo `Clave` existe en la tabla `Cliente` y su tipo de dato (para segmento 4 de la referencia Banamex). Si no existe, decidir entre agregarlo permanentemente o definir fuente alternativa (no depender de tabla ETL `Carga_ClientesR1` a largo plazo). | Desarrollo |
| P4 | Validar con el cliente si la asignación de cuentas y captura del CódValidador debe restringirse al rol Coordinador de Tesorería. | Funcional / Cliente |
| P5 | Confirmar si puede haber más de una cuenta bancaria activa por cliente y si se requiere tope máximo. | Funcional / Cliente |
| ~~P6~~ | ~~Confirmar si la funcionalidad aplica para clientes de Perú. Modelo bancario PE no definido.~~ **[Resuelto — Duda FU-006/FU-017]** Perú no tiene mecanismo de Código Validador; la referencia se genera por default con la **Razón Social** del cliente (mismo camino que bancos no-Banamex). La pantalla de captura no aplica para Perú. Ver Regla 6-PER en el requisito. | Cerrado |
| P7 | Verificar longitud máxima de `Cliente.Nombre` en BD para asegurar que varchar(80) de `ReferenciaPago` en `tpProformaPedido` es suficiente. | Desarrollo |
| P8 | Decidir mecanismo de regeneración de `ReferenciaVigente` ante cambio en `Cliente.Nombre` o `Cliente.Clave` (GAP-07: hook en `ClienteBO` vs trigger BD vs lazy). | Arquitectura |
| P9 | Decidir cómo se selecciona la cuenta destino del pedido cuando el cliente tiene varias asignaciones activas (DUDA-118 — Mayra/Daniel). | Funcional / Cliente |

---

## 9. Criterios de aceptación técnica

- [ ] La tabla `ClienteDatosBancarios` existe en BD con PK, FK a `Cliente` y FK a `DatosBancarios`, e incluye `ReferenciaVigente` y `FechaReferenciaVigente` (sin columnas de historial).
- [ ] Existen el índice filtrado `UX_ClienteDatosBancarios_ClienteCuentaActiva (IdCliente, IdDatosBancarios) WHERE Activo = 1` y el índice `IX_ClienteDatosBancarios (IdCliente, IdDatosBancarios, Activo)`.
- [ ] `ClienteDatosBancariosBO` permite insertar, actualizar y consultar la relación cliente-cuenta y **arma + persiste** `ReferenciaVigente` al CREATE/UPDATE.
- [ ] Al guardar o modificar el `CodigoValidador`, el BO registra el cambio en ProquifaDotNet.BitacoraCambios (valor anterior, nuevo, usuario, fecha) sin bloquear el guardado (GAP-06).
- [ ] El endpoint `PUT /ClienteDatosBancarios` guarda correctamente la combinación con su `CodigoValidador` y devuelve `ReferenciaVigente` calculada.
- [ ] `ReferenciaBancariaBO.Construir()` retorna `Cliente.Nombre` para cuentas de bancos distintos de Banamex.
- [ ] `ReferenciaBancariaBO.Construir()` retorna la concatenación de 7 segmentos para cuentas de Banamex (`catBanco.Clave = "002"`).
- [ ] Los 7 segmentos del algoritmo Banamex aplican correctamente el fallback "X" y padding de ceros. Los segmentos 1-3 ignoran espacios en el nombre (ej. "BP Farmaceutica" → "BPF", no "BP ").
- [ ] `tpProformaPedidoFactory` **lee** `ClienteDatosBancarios.ReferenciaVigente` y la copia a `ReferenciaPago` (no recalcula).
- [ ] Una proforma generada para un cliente MEX con cuenta Banamex muestra la referencia de 7 segmentos en el PDF.
- [ ] Una proforma generada para un cliente MEX con cuenta no-Banamex muestra el nombre del cliente como referencia.
- [ ] Al cambiar `Cliente.Nombre` o `Cliente.Clave`, `ReferenciaVigente` se regenera en todas las asignaciones activas del cliente (GAP-07).
- [ ] Las proformas ya emitidas conservan su `ReferenciaPago` original aunque después se regenere la `ReferenciaVigente` del cliente (snapshot inmutable).

