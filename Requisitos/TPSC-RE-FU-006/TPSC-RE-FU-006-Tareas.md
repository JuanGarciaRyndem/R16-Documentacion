# Tareas — TPSC-RE-FU-006 Referencia Bancaria de Cliente

| Campo | Valor |
|-------|-------|
| **Requisito** | TPSC-RE-FU-006 |
| **Nombre** | Referencia Bancaria de Cliente |
| **Total de tareas** | 8 |
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


---

## Tarea 6

### TPSC-RE-FU-006  [ QUERY-G ]  ETL SSIS Clientes — Análisis de campos R16 a transferir a Legacy

**Aplicativos:**
SSIS / PCconnect

**Módulos:**
ETL — Catálogo de Clientes → Legacy

**Consideraciones previas:**
- El ETL de Clientes a Legacy ya existe en SSIS (PCconnect). Esta tarea analiza **qué campos nuevos de R16 requieren actualización en el paquete existente**.
- R16 agrega o modifica datos del cliente en los siguientes requisitos: RE-002 (Cobrador: `ClienteCartera.IdUsuarioCobrador`), RE-004 (Datos Fiscales: RFC/RUC, Régimen Fiscal, Tipo de Sociedad Mercantil), RE-005 (Cobros: Forma de Pago, Uso CFDI, Método de Pago, Tipo Comprobante), RE-006 (Referencia Bancaria: `ClienteDatosBancarios`). Todos estos campos se almacenan en ProquifaDotNet y deben evaluarse para su transferencia.
- Coordinarse con el equipo Legacy para confirmar qué campos nuevos deben llegar y en qué tablas destino.
- Esta tarea precede a T7 (desarrollo del paquete SSIS) y T8 (pruebas). No puede avanzar implementación sin el mapeo aprobado.
- **Dependencias:** T1–T5 de RE-006 deben estar completadas (tabla `ClienteDatosBancarios` y BO creados) antes de poder mapear la referencia bancaria hacia Legacy.

**Descripción del problema:**
El ETL actual de Clientes fue diseñado antes de R16. Los nuevos campos agregados por R16 (Cobrador, datos fiscales, configuración de cobros, referencia bancaria) no están mapeados en el paquete SSIS. Sin esta actualización, Legacy no recibe información actualizada del cliente al sincronizar.

**Objetivos específicos:**
- Identificar el paquete SSIS existente de transferencia de Clientes en PCconnect y documentar su estructura actual.
- Mapear los campos nuevos por sección:
  - **Datos Generales (RE-002):** `ClienteCartera.IdUsuarioCobrador` → columna/tabla en Legacy.
  - **Datos Legales / Información Fiscal (RE-004):** `DatosFacturacionCliente.RFC`/`RUC`, `IdCatRegimenFiscal`, `IdCatTipoSociedadMercantil` → columna/tabla en Legacy.
  - **Cobros (RE-005):** `DatosFacturacionCliente.IdCatUsoCFDI`, `IdCatMetodoDePagoCFDI`, `ConfiguracionPagos.IdCatMedioDePago`, `IdCatTipoComprobante` → columna/tabla en Legacy.
  - **Referencia Bancaria (RE-006):** `ClienteDatosBancarios` (cuentas bancarias asignadas, `CodigoValidador`) → columna/tabla en Legacy.
- **La transferencia a Legacy aplica únicamente para México. Los datos de Perú (RUC, Régimen SUNAT, cobros Perú) NO se transfieren a Legacy.**
- Confirmar si hay campos de Direcciones o Contactos modificados en R16 que requieran actualización en el ETL.
- Documentar el mapeo completo origen → destino como insumo para T7.

**Resultado esperado:**
Documento de análisis con: paquete SSIS identificado, lista completa de campos R16 a agregar al ETL, mapeo columna-a-columna hacia Legacy por sección, y acuerdo con el equipo Legacy sobre qué campos aplican.

**Entregables:**
- Documento de análisis ETL Clientes con:
  - Referencia al paquete SSIS existente (nombre, ruta en PCconnect)
  - Lista de campos nuevos R16 por sección (Cobrador, Fiscal, Cobros, Referencia Bancaria)
  - Mapeo origen (ProquifaDotNet) → destino (Legacy) columna a columna
  - Alcance regional: **solo México** (Perú no transfiere a Legacy)
  - Acuerdos con equipo Legacy documentados

**Criterios de aceptación:**
- [ ] El paquete SSIS existente de Clientes está identificado y documentado.
- [ ] Los campos de RE-002, RE-004, RE-005 y RE-006 a incluir en el ETL están mapeados columna a columna hacia Legacy (solo datos México).
- [ ] Confirmado que los datos de Perú no se incluyen en la transferencia a Legacy.
- [ ] El documento de análisis está disponible y aprobado como prerequisito para Tarea 7 y Tarea 8.

**Más información de la tarea:**
Los campos a evaluar provienen de: `ClienteCartera` (RE-002), `DatosFacturacionCliente` (RE-004 y RE-005), `ConfiguracionPagos` (RE-005), `ClienteDatosBancarios` (RE-006). Ver archivos Back de cada requisito para detalle de campos.

**Recursos:**
- `TPSC-RE-FU-002-Back.md` — campo `IdUsuarioCobrador` en `ClienteCartera`
- `TPSC-RE-FU-004-Back.md` — campos fiscales en `DatosFacturacionCliente`
- `TPSC-RE-FU-005-Back.md` — campos de cobros en `DatosFacturacionCliente` y `ConfiguracionPagos`
- `TPSC-RE-FU-006-Back.md` — tabla `ClienteDatosBancarios` (Referencia Bancaria)
- Paquete SSIS existente de Clientes en PCconnect

---

## Tarea 7

### TPSC-RE-FU-006  [ BD-OBJ-G ]  ETL SSIS Clientes — Actualización del paquete SSIS con campos R16

**Aplicativos:**
SSIS / PCconnect

**Módulos:**
ETL — Catálogo de Clientes → Legacy (Desarrollo)

**Consideraciones previas:**
- **Predecesora: Tarea 6.** El mapeo de campos debe estar aprobado antes de modificar el paquete SSIS.
- La actualización se hace sobre el paquete SSIS existente — no se crea uno nuevo, se extiende el actual.
- Los campos nuevos se agregan respetando la estructura del paquete y sin romper los flujos de datos existentes.
- Confirmar con el DBA y el equipo Legacy si se requiere ALTER TABLE en tablas Legacy destino antes de ejecutar el nuevo paquete.
- Ejecutar primero en ambiente de desarrollo/QA antes de planificar el paso a producción.

**Descripción del problema:**
El paquete SSIS actual de Clientes no incluye los campos nuevos de R16 (Cobrador, Datos Fiscales, Cobros, Referencia Bancaria). Sin actualizar el paquete, los datos del cliente en Legacy estarán desactualizados respecto a lo capturado en ProquifaDotNet.

**Objetivos específicos:**
- Abrir el paquete SSIS existente de Clientes en Visual Studio (SSDT) o la herramienta correspondiente.
- Agregar los componentes de flujo de datos necesarios para los campos nuevos identificados en Tarea 6: `IdUsuarioCobrador` (Cobrador), datos fiscales (RFC/RUC, RegFiscal, TipoSociedad), configuración de cobros (UsoCFDI, MetodoPago, MedioPago, TipoComprobante) y referencia bancaria (`ClienteDatosBancarios`).
- Actualizar las transformaciones (lookups, derived columns, conditional split) si los nuevos campos requieren lógica adicional (por ejemplo: transformar GUID a clave Legacy, o condicionar por región).
- Si se requiere: ejecutar DDL de ALTER TABLE en las tablas destino Legacy antes de ejecutar el paquete actualizado.
- Implementar manejo de errores por fila: redirigir filas con error a una salida de error sin cancelar todo el paquete.
- Documentar los cambios realizados al paquete (componentes agregados, transformaciones, tablas afectadas).

**Resultado esperado:**
El paquete SSIS de Clientes actualizado incluye los campos R16 y transfiere correctamente los datos nuevos hacia Legacy en el ambiente de desarrollo/QA.

**Entregables:**
- Paquete SSIS actualizado (`.dtsx`) con los nuevos campos R16 integrados
- Script DDL de ALTER TABLE en tablas Legacy destino (si aplica)
- Documento de cambios al paquete: componentes agregados, transformaciones, tablas afectadas
- Instrucciones de despliegue para el paso a QA y producción

**Criterios de aceptación:**
- [ ] El paquete SSIS incluye los campos de Cobrador, Fiscal, Cobros y Referencia Bancaria definidos en Tarea 6.
- [ ] El paquete ejecuta sin errores en ambiente de desarrollo/QA.
- [ ] Las filas con error son redirigidas a la salida de error y logueadas, sin cancelar la ejecución completa.
- [ ] Los cambios al paquete están documentados.
- [ ] El paquete es revisado y aprobado por el líder técnico o DBA.

**Más información de la tarea:**
Ver mapeo de campos en el documento de análisis de Tarea 6. Los campos nuevos provienen de `ClienteCartera`, `DatosFacturacionCliente`, `ConfiguracionPagos` y `ClienteDatosBancarios` en ProquifaDotNet.

**Recursos:**
- Documento de análisis de Tarea 6 (mapeo origen → destino)
- Paquete SSIS existente de Clientes en PCconnect
- `TPSC-RE-FU-002-Back.md`, `TPSC-RE-FU-004-Back.md`, `TPSC-RE-FU-005-Back.md`, `TPSC-RE-FU-006-Back.md`

---

## Tarea 8

### TPSC-RE-FU-006  [ QUERY-M ]  ETL SSIS Clientes — Pruebas de validación de datos transferidos a Legacy

**Aplicativos:**
SSIS / PCconnect

**Módulos:**
ETL — Catálogo de Clientes → Legacy (Validación)

**Consideraciones previas:**
- **Predecesoras: Tarea 6 y Tarea 7.** El paquete SSIS actualizado debe estar disponible en QA antes de ejecutar las pruebas.
- Las pruebas deben ejecutarse con datos reales o representativos en ambiente QA/staging — no con datos sintéticos aislados.
- Validar por comparación directa entre ProquifaDotNet (fuente) y Legacy (destino) para cada campo nuevo.
- Coordinar con el equipo Legacy para tener acceso de lectura a las tablas destino durante las pruebas.

**Descripción del problema:**
Después de actualizar el paquete SSIS, se debe verificar que los datos nuevos de R16 llegan correctamente a Legacy, que no hay pérdida de datos existentes y que el paquete es estable en ambiente QA antes de pasar a producción.

**Objetivos específicos:**
- Ejecutar el paquete SSIS actualizado en ambiente QA con un conjunto representativo de clientes de **México únicamente** (Perú no transfiere a Legacy).
- Validar que los campos nuevos llegan con los valores correctos en Legacy:
  - `IdUsuarioCobrador` → campo Cobrador en Legacy (nombre/clave del cobrador).
  - Campos fiscales (RFC/RUC, RegFiscal, TipoSociedad) → columnas correspondientes en Legacy.
  - Campos de cobros (UsoCFDI, MetodoPago, MedioPago, TipoComprobante) → columnas correspondientes en Legacy.
  - Referencia bancaria (`ClienteDatosBancarios`: cuentas asignadas, `CodigoValidador`) → columnas correspondientes en Legacy.
- Verificar que los campos existentes (Datos Generales, Direcciones, Contactos) no fueron alterados por los cambios al paquete.
- Verificar el comportamiento ante valores nulos: clientes sin Cobrador asignado, clientes sin datos fiscales completos, clientes sin referencia bancaria.
- Documentar los resultados: casos ejecutados, evidencias de comparación, incidencias encontradas y resolución.
- Validar el tiempo de ejecución del paquete y que no genera impacto en rendimiento comparado con el paquete anterior.

**Resultado esperado:**
Los campos R16 (Cobrador, Fiscal, Cobros, Referencia Bancaria) llegan correctamente a Legacy para todos los clientes de prueba. Los campos existentes no fueron afectados. Las pruebas están documentadas con evidencias y el paquete está listo para pasar a producción.

**Entregables:**
- Documento de resultados de pruebas:
  - Casos ejecutados por campo nuevo (Cobrador, Fiscal, Cobros, Referencia Bancaria)
  - Comparación ProquifaDotNet ↔ Legacy por campo
  - Comportamiento ante nulos (sin Cobrador / sin datos fiscales / sin referencia bancaria)
  - Impacto en campos existentes (sin regresión)
  - Tiempo de ejecución vs. paquete anterior
  - Incidencias encontradas y su resolución
- Evidencias de validación (capturas o queries de comparación)
- Confirmación de aprobación para paso a producción

**Criterios de aceptación:**
- [ ] Los campos de Cobrador, Fiscal, Cobros y Referencia Bancaria llegan a Legacy con valores correctos validados por comparación directa.
- [ ] Los clientes sin Cobrador asignado o sin referencia bancaria transfieren correctamente con campo nulo/vacío en Legacy, sin error.
- [ ] Los campos existentes del cliente (Datos Generales, Direcciones, Contactos) no fueron alterados.
- [ ] El paquete ejecuta sin errores en QA con datos representativos de México (Perú no aplica para transferencia a Legacy).
- [ ] Las pruebas están documentadas con evidencias y aprobadas por el líder técnico o DBA.
- [ ] El paquete está aprobado para paso a producción.

**Más información de la tarea:**
Esta tarea cierra el ciclo ETL de Clientes para R16, integrando todos los cambios de RE-002 a RE-006. Los ETL de otras entidades (Marcas, Proveedores, Productos, Buzones, Cotizaciones, Pedidos) son independientes y tienen sus propios paquetes SSIS.

**Recursos:**
- Documento de análisis de Tarea 6 (mapeo de campos)
- Paquete SSIS actualizado de Tarea 7
- Acceso de lectura a tablas Legacy destino en ambiente QA
- `TPSC-RE-FU-002-Back.md`, `TPSC-RE-FU-004-Back.md`, `TPSC-RE-FU-005-Back.md`, `TPSC-RE-FU-006-Back.md`
