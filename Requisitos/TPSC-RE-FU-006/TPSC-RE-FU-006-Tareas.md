# Tareas — TPSC-RE-FU-006 Referencia Bancaria de Cliente

| Campo | Valor |
|-------|-------|
| **Requisito** | TPSC-RE-FU-006 |
| **Nombre** | Referencia Bancaria de Cliente |
| **Total de tareas** | 5 |
| **Revisión aplicada** | TPSC-RE-FU-006_Revision.md |

---

## Tarea 1

### TPSC-RE-FU-006  GAP-01  [ CREATE-TABL-CH ] Crear tabla `ClienteDatosBancarios` en base de datos

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogo de Clientes — Referencia de Pago (sección Cobros)

**Consideraciones previas:**
Esta tarea es **prerequisito de las Tareas 2, 3 y 5**. Sin la tabla no se puede crear el BO ni el controller.
Condicionada al Pendiente **P2**: confirmar con el cliente la longitud máxima y formato del `CodigoValidador` antes de ejecutar el CREATE TABLE. La longitud provisional es `varchar(50)`; ajustar si se confirma otra longitud.
Confirmar también el Pendiente **P3**: verificar que el campo `Clave` existe en la tabla `Cliente` (se usa en el algoritmo de referencia Banamex) antes de crear el índice.

**Descripción del problema:**
No existe en el modelo de datos ninguna tabla que relacione un `Cliente` con una o más cuentas bancarias del grupo PROQUIFA (`DatosBancarios`) ni que almacene el `CodigoValidador` por combinación cliente-cuenta.
Sin esta tabla es imposible implementar la pantalla Referencia de Pago (Regla R1), persistir la relación N:N (Regla R2) ni guardar el Código Validador por combinación (Regla R3).

**Cambios requeridos:**

```sql
CREATE TABLE dbo.ClienteDatosBancarios
(
    IdClienteDatosBancarios  uniqueidentifier NOT NULL
        CONSTRAINT PK_ClienteDatosBancarios  PRIMARY KEY
        CONSTRAINT DF_ClienteDatosBancarios_Id DEFAULT (NEWSEQUENTIALID()),
    IdCliente                uniqueidentifier NOT NULL
        CONSTRAINT FK_ClienteDatosBancarios_Cliente
            FOREIGN KEY REFERENCES dbo.Cliente(IdCliente),
    IdDatosBancarios         uniqueidentifier NOT NULL
        CONSTRAINT FK_ClienteDatosBancarios_DatosBancarios
            FOREIGN KEY REFERENCES dbo.DatosBancarios(IdDatosBancarios),
    CodigoValidador          varchar(50)      NULL,
    FechaRegistro            datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaRegistro    DEFAULT (GETDATE()),
    FechaUltimaActualizacion datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaActualizacion DEFAULT (GETDATE()),
    Activo                   bit              NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_Activo           DEFAULT (1)
);

CREATE NONCLUSTERED INDEX IX_ClienteDatosBancarios
    ON dbo.ClienteDatosBancarios (IdCliente, IdDatosBancarios, Activo);

SELECT OBJECT_ID('dbo.ClienteDatosBancarios') AS TablaCreada;
```

**Criterios de aceptación:**
- [ ] Pendientes P2 (longitud `CodigoValidador`) y P3 (campo `Clave` en `Cliente`) confirmados antes de ejecutar
- [ ] La tabla `ClienteDatosBancarios` existe en BD con PK `IdClienteDatosBancarios`
- [ ] FK `FK_ClienteDatosBancarios_Cliente` referencia `dbo.Cliente(IdCliente)`
- [ ] FK `FK_ClienteDatosBancarios_DatosBancarios` referencia `dbo.DatosBancarios(IdDatosBancarios)`
- [ ] Índice `IX_ClienteDatosBancarios` existe sobre `(IdCliente, IdDatosBancarios, Activo)`
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-01 y Pendientes P2, P3 del archivo `TPSC-RE-FU-006-Back.md`
- Reglas R2 y R3 del requisito: relación N:N cliente-cuenta y Código Validador por combinación

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006.md`

---

## Tarea 2

### TPSC-RE-FU-006  GAP-02  [ SERV-SIMPLE-PUT ] Crear `ClienteDatosBancariosBO` — CRUD de relación cliente-cuenta-CódigoValidador

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Catalogos

**Módulos:**
Catálogo de Clientes — Referencia de Pago (sección Cobros)

**Consideraciones previas:**
**Depende de Tarea 1.** La tabla `ClienteDatosBancarios` debe existir en BD antes de que Entity Framework pueda mapear la entidad.
Verificar que el modelo EF (`ProquifaDotNetEntities`) haya sido actualizado con la nueva entidad `ClienteDatosBancarios` antes de compilar.

**Descripción del problema:**
No existe clase de negocio para gestionar la relación cliente-cuenta-CódigoValidador. Sin el BO no es posible insertar, actualizar ni consultar desde la capa de API los registros de `ClienteDatosBancarios` (Regla R4).

**Archivos a crear:**
- `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ClienteDatosBancariosBO.cs`

**Cambios requeridos:**

```csharp
using System;
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

        /// <summary>
        /// Obtiene la cuenta bancaria activa de un cliente.
        /// Si el cliente tiene varias cuentas activas, retorna la más reciente.
        /// </summary>
        public ClienteDatosBancarios ObtenerCuentaActivaDelCliente(Guid idCliente)
        {
            using (var db = new ProquifaDotNetEntities())
            {
                return db.ClienteDatosBancarios
                    .Where(x => x.IdCliente == idCliente && x.Activo)
                    .OrderByDescending(x => x.FechaRegistro)
                    .FirstOrDefault();
            }
        }
    }
}
```

**Criterios de aceptación:**
- [ ] La clase `ClienteDatosBancariosBO` compila sin errores
- [ ] `GuardarOActualizar` persiste correctamente un registro con `IdCliente`, `IdDatosBancarios` y `CodigoValidador`
- [ ] `ObtenerCuentaActivaDelCliente` retorna el registro más reciente activo del cliente
- [ ] `Desactivar` marca el registro con `Activo = false`
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-02 del archivo `TPSC-RE-FU-006-Back.md`
- Regla R4 del requisito: persistencia de la combinación cliente-cuenta-CódValidador
- Patrón de referencia: `DatosBancariosBO.cs` en el mismo proyecto

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006.md`

---

## Tarea 3

### TPSC-RE-FU-006  GAP-03  [ ALG-BASIC-LOGIC ] Crear `ReferenciaBancariaBO` — algoritmo de construcción de referencia bancaria (Banamex / no-Banamex)

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Catalogos

**Módulos:**
Catálogo de Clientes — Referencia de Pago / Generación de Proforma

**Consideraciones previas:**
**Depende de Tarea 2.** El BO utiliza `ClienteDatosBancariosBO` internamente.
Condicionada al Pendiente **P1**: confirmar con el cliente y desarrollo la lógica completa de identificación de Banamex (la condición de moneda en el documento original aparece truncada). La propuesta aprobada es identificar Banamex por `catBanco.Clave = "002"`.
Condicionada al Pendiente **P3**: confirmar que el campo `Clave` existe en la tabla `Cliente` y su tipo de dato, ya que es el insumo del segmento 4 de la referencia Banamex.

**Descripción del problema:**
No existe lógica de construcción de la referencia bancaria en el proyecto. Las Reglas R5, R6 y R7 definen dos algoritmos:
- **No-Banamex (R6):** la referencia es el `Nombre` del cliente como cadena directa.
- **Banamex (R7):** concatenación determinista de 7 segmentos: tres primeras letras del nombre del cliente (fallback "X"), últimos 4 caracteres de la clave del cliente (padding con ceros), código del banco, carácter de moneda ("P" para MXN, "D" para otro) y Código Validador.

**Archivos a crear:**
- `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ReferenciaBancariaBO.cs`

**Cambios requeridos:**

```csharp
using System;
using Core.Pqf.BusinessBasicTools._Misc.Crud;
using Core.Pqf.ProquifaDotNetContext;

namespace Logic.Pqf.Catalogos.Clientes.DatosBancarios
{
    public class ReferenciaBancariaBO
    {
        private const string ClaveBanamex = "002";

        /// <summary>
        /// Construye la referencia bancaria para una proforma.
        /// R6: no-Banamex = nombre del cliente.
        /// R7: Banamex    = concatenación de 7 segmentos.
        /// </summary>
        public string Construir(Cliente cliente, DatosBancarios cuenta, string codigoValidador)
        {
            var banco = new TablaGenericaBO<catBanco>().Obtener(cuenta.IdCatBanco.Value);

            if (banco?.Clave == ClaveBanamex)
                return ConstruirBanamex(cliente, cuenta, banco, codigoValidador);

            return cliente.Nombre ?? string.Empty;
        }

        private string ConstruirBanamex(Cliente cliente, DatosBancarios cuenta,
                                         catBanco banco, string codigoValidador)
        {
            var nombre = cliente.Nombre ?? string.Empty;
            var clave  = cliente.Clave  ?? string.Empty;

            // Segmentos 1-3: primeras tres letras del nombre, fallback "X"
            var seg1 = nombre.Length > 0 ? nombre[0].ToString() : "X";
            var seg2 = nombre.Length > 1 ? nombre[1].ToString() : "X";
            var seg3 = nombre.Length > 2 ? nombre[2].ToString() : "X";

            // Segmento 4: últimos 4 chars de la clave, padding con ceros a la izquierda
            var seg4 = clave.Length >= 4
                ? clave.Substring(clave.Length - 4)
                : clave.PadLeft(4, '0');

            // Segmento 5: código del banco
            var seg5 = banco.Clave ?? string.Empty;

            // Segmento 6: P si moneda MXN (ClaveMoneda empieza en "M"), D en otro caso
            var moneda = new TablaGenericaBO<catMoneda>().Obtener(cuenta.IdCatMoneda.Value);
            var seg6 = (moneda?.ClaveMoneda?.StartsWith("M") == true) ? "P" : "D";

            // Segmento 7: Código Validador
            var seg7 = codigoValidador ?? string.Empty;

            return $"{seg1}{seg2}{seg3}{seg4}{seg5}{seg6}{seg7}";
        }
    }
}
```

**Tabla de verificación de segmentos Banamex:**

| Segmento | Origen | Regla |
|----------|--------|-------|
| 1 | `Cliente.Nombre[0]` | "X" si nombre vacío |
| 2 | `Cliente.Nombre[1]` | "X" si nombre < 2 chars |
| 3 | `Cliente.Nombre[2]` | "X" si nombre < 3 chars |
| 4 | Últimos 4 chars de `Cliente.Clave` | Padding con "0" si clave < 4 chars |
| 5 | `catBanco.Clave` | Código SAT del banco |
| 6 | Primera letra de `catMoneda.ClaveMoneda` | "P" si empieza en "M" (MXN), "D" en otro caso |
| 7 | `ClienteDatosBancarios.CodigoValidador` | Capturado manualmente por el usuario |

**Criterios de aceptación:**
- [ ] Pendiente P1 (lógica identificación Banamex) y P3 (campo `Clave` en `Cliente`) confirmados antes de implementar
- [ ] `Construir()` retorna `Cliente.Nombre` cuando el banco NO es Banamex
- [ ] `Construir()` retorna la cadena de 7 segmentos cuando el banco ES Banamex (`catBanco.Clave = "002"`)
- [ ] Segmentos 1-3 aplican fallback "X" cuando el nombre tiene menos de 3 caracteres
- [ ] Segmento 4 aplica padding de ceros cuando la clave tiene menos de 4 caracteres
- [ ] Segmento 6 devuelve "P" para MXN y "D" para USD u otra moneda
- [ ] La clase compila sin errores y no rompe la compilación de `Logic.Pqf.Catalogos`
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-03 y Pendientes P1, P3 del archivo `TPSC-RE-FU-006-Back.md`
- Reglas R5, R6 y R7 del requisito
- Documento de referencia del cliente: "Especificación: Proceso para generar Referencia de Cliente (Código Validador)" recibido el 2026-04-28

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006.md`

---

## Tarea 4

### TPSC-RE-FU-006  GAP-04  [ IMP-EXIST-SERVICE ] Inyectar `ReferenciaPago` en `tpProformaPedidoFactory`

**Aplicativos:**
ProquifaNet 2 — Logic.Pqf.Logistica

**Módulos:**
Tramitar Pedido — Generación de Proforma

**Consideraciones previas:**
**Depende de Tarea 3.** `ReferenciaBancariaBO` debe existir y compilar antes de modificar la fábrica.
Verificar el Pendiente **P7**: consultar en BD la longitud máxima de `Cliente.Nombre` para asegurarse de que no supera los 80 caracteres que define `tpProformaPedido.ReferenciaPago varchar(80)`. Si supera, coordinar ampliar el campo con DBA.

**Descripción del problema:**
`tpProformaPedidoFactory.Process()` asigna `ReferenciaPago = null` al construir cada `tpProformaPedido`. El campo ya existe en la entidad pero nunca se calcula. Esto significa que ninguna proforma generada en PQF2 incluye la referencia bancaria del cliente en el PDF (Regla R5).

**Archivo a modificar:**
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs`

**Estado actual:**

```csharp
// tpProformaPedidoFactory.cs — estado actual
var tpProformaPedido = new tpProformaPedido
{
    // ...
    ReferenciaPago = null,   // nunca se calcula
    // ...
};
```

**Cambio requerido:**

```csharp
// tpProformaPedidoFactory.cs — después del cambio
// cliente ya se obtiene en el método Process() existente

// 1. Obtener cuenta bancaria activa del cliente
var clienteDatosBancariosBO = new ClienteDatosBancariosBO();
var clienteCuenta = clienteDatosBancariosBO.ObtenerCuentaActivaDelCliente(tpPedido.IdCliente);

// 2. Construir referencia bancaria (null si el cliente no tiene cuenta asignada)
string referenciaPago = null;
if (clienteCuenta != null)
{
    var cuenta = new TablaGenericaBO<DatosBancarios>().Obtener(clienteCuenta.IdDatosBancarios);
    var refBO  = new ReferenciaBancariaBO();
    referenciaPago = refBO.Construir(cliente, cuenta, clienteCuenta.CodigoValidador);
}

var tpProformaPedido = new tpProformaPedido
{
    // ...
    ReferenciaPago = referenciaPago,  // calculada o null si sin cuenta asignada
    // ...
};
```

**Criterios de aceptación:**
- [ ] `tpProformaPedidoFactory.Process()` invoca `ClienteDatosBancariosBO.ObtenerCuentaActivaDelCliente()` y `ReferenciaBancariaBO.Construir()`
- [ ] Si el cliente tiene cuenta bancaria asignada, `tpProformaPedido.ReferenciaPago` contiene la referencia construida (no `null`)
- [ ] Si el cliente NO tiene cuenta asignada, `ReferenciaPago` queda en `null` sin lanzar excepción
- [ ] Una proforma para cliente MEX con cuenta Banamex incluye la referencia de 7 segmentos
- [ ] Una proforma para cliente MEX con cuenta no-Banamex incluye el nombre del cliente como referencia
- [ ] El resto del comportamiento de `tpProformaPedidoFactory` no es alterado
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-04 y Pendiente P7 del archivo `TPSC-RE-FU-006-Back.md`
- Regla R5 del requisito: referencia se reconstruye dinámicamente en cada generación de proforma
- Criterios C1, C2 y C3 del requisito

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006.md`

---

## Tarea 5

### TPSC-RE-FU-006  GAP-05  [ SIMPLE-CRUD ] Crear `ClienteDatosBancariosController` — endpoint CRUD `/ClienteDatosBancarios`

**Aplicativos:**
ProquifaNet 2 — WebApi.Catalogos

**Módulos:**
Catálogo de Clientes — Referencia de Pago (sección Cobros)

**Consideraciones previas:**
**Depende de Tarea 2.** `ClienteDatosBancariosBO` debe existir y compilar antes de crear el controller.
Confirmar Pendiente **P4**: validar con el cliente si la asignación de cuentas y captura del CódValidador debe restringirse al rol Coordinador de Tesorería. Si se confirma restricción de rol, añadir middleware de autorización antes del merge.
Confirmar Pendiente **P5**: si puede haber más de una cuenta activa por cliente, el `QueryResult` debe soportar filtro por `IdCliente`.

**Descripción del problema:**
No existe endpoint en la API que permita a la pantalla Referencia de Pago leer, crear, actualizar ni desactivar la combinación cliente-cuenta-CódigoValidador (Regla R1). Sin el controller, la pantalla no tiene backend disponible.

**Archivo a crear:**
`WebApi.Catalogos\Controllers\Configuracion\Clientes\ClienteDatosBancariosController.cs`

**Cambios requeridos:**

```csharp
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

**Criterios de aceptación:**
- [ ] Pendientes P4 (restricción de rol) y P5 (múltiples cuentas por cliente) confirmados antes de hacer merge
- [ ] `POST /ClienteDatosBancarios` retorna la lista paginada filtrando correctamente por `IdCliente`
- [ ] `GET /ClienteDatosBancarios?idClienteDatosBancarios={id}` retorna el registro correcto
- [ ] `PUT /ClienteDatosBancarios` guarda un nuevo registro con `CodigoValidador` y retorna el `Guid`
- [ ] `PUT /ClienteDatosBancarios` actualiza el `CodigoValidador` de un registro existente
- [ ] `DELETE /ClienteDatosBancarios?idClienteDatosBancarios={id}` desactiva el registro (`Activo = false`)
- [ ] El proyecto `WebApi.Catalogos` compila sin errores
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-05 y Pendientes P4, P5 del archivo `TPSC-RE-FU-006-Back.md`
- Reglas R1 y R9 del requisito: pantalla Referencia de Pago y sin restricción de rol (pendiente confirmar)
- Patrón de referencia: `DatosBancariosController.cs` en el mismo proyecto

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-006/TPSC-RE-FU-006.md`
