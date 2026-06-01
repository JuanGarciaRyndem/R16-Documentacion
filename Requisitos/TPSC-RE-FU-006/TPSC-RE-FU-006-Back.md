# TPSC-RE-FU-006 — Análisis de Impacto Backend

| Campo | Valor |
|-------|-------|
| Requisito | TPSC-RE-FU-006 — Referencia Bancaria de Cliente |
| Rama | develop-pack04 |
| Fecha | 2025-07 |
| Revisión | v1.0 |

---

## 1. Estructura actual en el proyecto

### 1.1 Capa de Lógica

| Archivo | Clase | Descripción |
|---------|-------|-------------|
| `Logic.Pqf.Catalogos\Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.cs` | `DatosBancariosBO` | CRUD de cuentas bancarias del grupo PROQUIFA. |
| `Logic.Pqf.Catalogos\Cuentas\ConfiguracionDatosBancarios\DatosBancariosBO.Extensions.cs` | `DatosBancariosBO` Extensions | Filtros especiales por `IdEmpresa`, `IdConfiguracionPagos`, `IdCatMedioDePago`. |
| `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.cs` | `EmpresaDatosBancariosBO` | Catálogo de cuentas del grupo PROQUIFA — fuente del selector de pantalla. |
| `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs` | `tpProformaPedidoFactory` | Fábrica que construye `tpProformaPedido`. Asigna `ReferenciaPago = null` en estado actual. |
| *(no existe)* | `ClienteDatosBancariosBO` | **NUEVO R16** — Gestión de relación cliente-cuenta-CódigoValidador. |
| *(no existe)* | `ReferenciaBancariaBO` | **NUEVO R16** — Algoritmo de construcción de referencia bancaria (Banamex/no-Banamex). |

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
| **R5** — Referencia se reconstruye dinámicamente | No se almacena; se calcula al generar cada proforma. | Lógica en `tpProformaPedidoFactory` o extensión. `ReferenciaPago` ya existe en la entidad. | Existe en entidad pero se asigna `null` — pendiente implementar |
| **R6** — Referencia no-Banamex = nombre del cliente | `Cliente.Nombre` como cadena directa sin transformación. | `ReferenciaBancariaBO.Construir()` retorna `cliente.Nombre`. | BO no existe |
| **R7** — Referencia Banamex = 7 segmentos | Concatenación determinista: 3 letras + 4 chars clave + código banco + P/D + CódValidador. | `ReferenciaBancariaBO.ConstruirBanamex()`. | BO no existe |
| **R8** — Identificación Banamex por `Clave = "002"` | Cruce con `catBanco.Clave = "002"` (simplificación propuesta). | Consulta `catBanco` en el BO de referencia. | Pendiente confirmar con desarrollo |
| **R9** — Sin restricción de rol | Sin control de rol. Acceso por cartera del cliente. | Sin middleware de rol en controller. | Sin restricción en controllers de cliente — correcto, pendiente confirmar con cliente |

---

## 3. GAPs identificados y código de implementación

### GAP-01 — Tabla `ClienteDatosBancarios` no existe

**Archivo:** BD — nueva tabla
**Impacto:** Reglas R2, R3 y R4 — sin la tabla no es posible persistir la relación cliente-cuenta ni el Código Validador.
**Cambio requerido:** Crear la tabla (condicionado a confirmar longitud de `CodigoValidador` con cliente — Pendiente P2).

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
    CodigoValidador          varchar(50)      NULL,   -- longitud provisional; confirmar P2
    FechaRegistro            datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaRegistro DEFAULT (GETDATE()),
    FechaUltimaActualizacion datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaActualizacion DEFAULT (GETDATE()),
    Activo                   bit              NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_Activo DEFAULT (1)
);

CREATE NONCLUSTERED INDEX IX_ClienteDatosBancarios
    ON dbo.ClienteDatosBancarios (IdCliente, IdDatosBancarios, Activo);
```

---

### GAP-02 — `ClienteDatosBancariosBO` no existe (CRUD de la relación cliente-cuenta)

**Archivo a crear:** `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ClienteDatosBancariosBO.cs`
**Impacto:** Regla R4 — no es posible insertar ni actualizar la relación cliente-cuenta-CódValidador.
**Cambio requerido:** Crear BO usando el patrón `TablaGenericaBO<T>` del proyecto. Depende de GAP-01.

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

            // Segmentos 1-3: primeras tres letras del nombre, fallback "X"
            var seg1 = nombre.Length > 0 ? nombre[0].ToString() : "X";
            var seg2 = nombre.Length > 1 ? nombre[1].ToString() : "X";
            var seg3 = nombre.Length > 2 ? nombre[2].ToString() : "X";

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

### GAP-04 — `tpProformaPedidoFactory` no inyecta `ReferenciaPago`

**Archivo:** `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs`
**Impacto:** Regla R5 — el campo `ReferenciaPago` existe en `tpProformaPedido` pero se asigna `null`. La referencia nunca llega al PDF de la proforma.
**Cambio requerido:** Invocar `ReferenciaBancariaBO.Construir()` antes de persistir la proforma.

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
// DESPUÉS — tpProformaPedidoFactory.cs
// 1. Obtener cuenta y código validador del cliente
var clienteDatosBancariosBO = new ClienteDatosBancariosBO();
var clienteCuenta = clienteDatosBancariosBO.ObtenerCuentaActivaDelCliente(tpPedido.IdCliente);

// 2. Construir la referencia
string referenciaPago = null;
if (clienteCuenta != null)
{
    var cuenta  = new TablaGenericaBO<DatosBancarios>().Obtener(clienteCuenta.IdDatosBancarios);
    var refBO   = new ReferenciaBancariaBO();
    referenciaPago = refBO.Construir(cliente, cuenta, clienteCuenta.CodigoValidador);
}

var tpProformaPedido = new tpProformaPedido
{
    // ...
    ReferenciaPago = referenciaPago,
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

## 4. Tablas y entidades del modelo de datos

| Tabla BD | Entidad EF | Propiedades clave R16 | Descripción |
|----------|------------|-----------------------|-------------|
| `ClienteDatosBancarios` | `ClienteDatosBancarios` | `IdClienteDatosBancarios` (PK), `IdCliente` (FK), `IdDatosBancarios` (FK), `CodigoValidador` | **NUEVA R16** — Relación N:N cliente-cuenta con CódValidador |
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
| 1 | BD — nueva tabla `ClienteDatosBancarios` | CREATE TABLE + INDEX |
| 2 | `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ClienteDatosBancariosBO.cs` | Clase nueva — CRUD relación cliente-cuenta |
| 3 | `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ReferenciaBancariaBO.cs` | Clase nueva — algoritmo de referencia (Banamex / no-Banamex) |
| 4 | `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs` | Modificar — inyectar `ReferenciaPago` llamando a `ReferenciaBancariaBO` |
| 5 | `WebApi.Catalogos\Controllers\Configuracion\Clientes\ClienteDatosBancariosController.cs` | Controller nuevo — CRUD `/ClienteDatosBancarios` |

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
| P2 | Confirmar longitud máxima y formato del `CodigoValidador`. Longitud provisional: varchar(50). | Cliente / PMO |
| P3 | Confirmar si el campo `Clave` existe en la tabla `Cliente` y su tipo de dato (para segmento 4 de la referencia Banamex). | Desarrollo |
| P4 | Validar con el cliente si la asignación de cuentas y captura del CódValidador debe restringirse al rol Coordinador de Tesorería. | Funcional / Cliente |
| P5 | Confirmar si puede haber más de una cuenta bancaria activa por cliente y si se requiere tope máximo. | Funcional / Cliente |
| P6 | Confirmar si la funcionalidad aplica para clientes de Perú. Modelo bancario PE no definido. | Funcional / Cliente |
| P7 | Verificar longitud máxima de `Cliente.Nombre` en BD para asegurar que varchar(80) de `ReferenciaPago` en `tpProformaPedido` es suficiente. | Desarrollo |

---

## 9. Criterios de aceptación técnica

- [ ] La tabla `ClienteDatosBancarios` existe en BD con PK, FK a `Cliente` y FK a `DatosBancarios`.
- [ ] El índice `IX_ClienteDatosBancarios (IdCliente, IdDatosBancarios, Activo)` existe.
- [ ] `ClienteDatosBancariosBO` permite insertar, actualizar y consultar la relación cliente-cuenta.
- [ ] El endpoint `PUT /ClienteDatosBancarios` guarda correctamente la combinación con su `CodigoValidador`.
- [ ] `ReferenciaBancariaBO.Construir()` retorna `Cliente.Nombre` para cuentas de bancos distintos de Banamex.
- [ ] `ReferenciaBancariaBO.Construir()` retorna la concatenación de 7 segmentos para cuentas de Banamex (`catBanco.Clave = "002"`).
- [ ] Los 7 segmentos del algoritmo Banamex aplican correctamente el fallback "X" y padding de ceros.
- [ ] `tpProformaPedidoFactory` asigna `ReferenciaPago` con la referencia construida (no `null`) cuando el cliente tiene cuenta asignada.
- [ ] Una proforma generada para un cliente MEX con cuenta Banamex muestra la referencia de 7 segmentos en el PDF.
- [ ] Una proforma generada para un cliente MEX con cuenta no-Banamex muestra el nombre del cliente como referencia.

