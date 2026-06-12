# Tareas — R16A-RE-FU-005 Catálogos de Cobros y Configuración de Facturación

| Campo | Valor |
|---|---|
| **Requisito** | R16A-RE-FU-005 |
| **Nombre** | Catálogos de Cobros y Configuración de Facturación |
| **Total de tareas** | 6 |
| **Revisión aplicada** | R16A-RE-FU-005 Revision.md |

---
## Tarea 1

### R16A-RE-FU-005  GAP-01  [ UPDATE-TABL-CH ] Agregar `IdRegion` a `catMetodoDePagoCFDI`, `catUsoCFDI` y `catMedioDePago`

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogos de Cobros — Configuración de Facturación de Cliente

**Consideraciones previas:**
Esta tarea es **prerequisito de todas las demás**. Ejecutar primero y verificar que el UPDATE masivo asignó `IdRegion = MEX` a todos los registros existentes antes de continuar.
Confirmar con TechLead el Pendiente P2: si el catálogo legacy debe mapearse para facturas y pedidos históricos al agregar `IdRegion`.

**Descripción del problema:**
Los tres catálogos de cobros no tienen el campo `IdRegion`. Sin él no es posible filtrar por región del usuario logueado en los selectores de Configuración de Facturación del Cliente. Un usuario de México vería opciones de Perú y viceversa.
**Cambios requeridos:**

```sql
ALTER TABLE dbo.catMetodoDePagoCFDI
    ADD IdRegion uniqueidentifier NULL
    CONSTRAINT FK_catMetodoDePagoCFDI_Region FOREIGN KEY REFERENCES dbo.Region(IdRegion);
UPDATE dbo.catMetodoDePagoCFDI SET IdRegion = (SELECT IdRegion FROM dbo.Region WHERE ClaveISO = 'MEX') WHERE IdRegion IS NULL;
SELECT COUNT(*) AS SinRegion_MetodoPago FROM dbo.catMetodoDePagoCFDI WHERE IdRegion IS NULL;

ALTER TABLE dbo.catUsoCFDI
    ADD IdRegion uniqueidentifier NULL
    CONSTRAINT FK_catUsoCFDI_Region FOREIGN KEY REFERENCES dbo.Region(IdRegion);
UPDATE dbo.catUsoCFDI SET IdRegion = (SELECT IdRegion FROM dbo.Region WHERE ClaveISO = 'MEX') WHERE IdRegion IS NULL;
SELECT COUNT(*) AS SinRegion_UsoCFDI FROM dbo.catUsoCFDI WHERE IdRegion IS NULL;

ALTER TABLE dbo.catMedioDePago
    ADD IdRegion uniqueidentifier NULL
    CONSTRAINT FK_catMedioDePago_Region FOREIGN KEY REFERENCES dbo.Region(IdRegion);
UPDATE dbo.catMedioDePago SET IdRegion = (SELECT IdRegion FROM dbo.Region WHERE ClaveISO = 'MEX') WHERE IdRegion IS NULL;
SELECT COUNT(*) AS SinRegion_MedioPago FROM dbo.catMedioDePago WHERE IdRegion IS NULL;
```

**Criterios de aceptación:**
- [ ] Los tres catálogos tienen columna `IdRegion` con FK a `Region`
- [ ] Todos los registros existentes tienen `IdRegion` asignado a la región México
- [ ] Los tres SELECTs de verificación devuelven 0
- [ ] Scripts incluidos en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-01 y Pendiente P2 del archivo `R16A-RE-FU-005-Back.md`
- Regla R2 del requisito: catálogos diferenciados por Región

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005.md`

---
## Tarea 2

### R16A-RE-FU-005  GAP-02  [ QUERY-CH ] Insertar registros PE en `catMetodoDePagoCFDI` (CONT/CRED) y `catUsoCFDI` (01/03/08)

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogos de Cobros — Configuración de Facturación de Cliente

**Consideraciones previas:**
**Depende de Tarea 1.** Ejecutar solo después de que `IdRegion` exista en ambas tablas.
Condicionada al Pendiente P1: confirmar denominación final del campo Condición de Pago en pantalla para Perú (¿Contado/Crédito o nomenclatura distinta?). Las descripciones pueden ajustarse tras confirmación.

**Descripción del problema:**
`catMetodoDePagoCFDI` solo contiene registros MX (PUE/PPD). No existen registros para Región Perú con Condición de Pago SUNAT (Contado/Crédito); el selector aparece vacío para usuarios PE.
`catUsoCFDI` solo contiene registros MX (G01/G03/S01 entre otros). No existen registros para Tipo de Comprobante PE (Factura/Boleta/Recibo de Honorarios); el selector también aparece vacío para usuarios PE.

**Cambios requeridos:**

```sql
DECLARE @PER uniqueidentifier = (SELECT IdRegion FROM dbo.Region WHERE ClaveISO = 'PER');

INSERT INTO dbo.catMetodoDePagoCFDI
    (IdCatMetodoDePagoCFDI, MetodoDePagoCFDI, ClaveMetodoDePagoCFDI, Clave, Activo, IdRegion)
VALUES
    (NEWID(), N'Contado', 'CONT', 'CONT', 1, @PER),
    (NEWID(), N'Crédito', 'CRED', 'CRED', 1, @PER);

INSERT INTO dbo.catUsoCFDI
    (IdCatUsoCFDI, ClaveUso, Uso, Clave, Activo, IdRegion)
VALUES
    (NEWID(), '01', N'Factura electrónica',               '01', 1, @PER),
    (NEWID(), '03', N'Boleta de venta electrónica',       '03', 1, @PER),
    (NEWID(), '08', N'Recibo por Honorarios electrónico', '08', 1, @PER);

SELECT ClaveMetodoDePagoCFDI, MetodoDePagoCFDI, Activo FROM dbo.catMetodoDePagoCFDI WHERE IdRegion = @PER;
SELECT ClaveUso, Uso, Activo FROM dbo.catUsoCFDI WHERE IdRegion = @PER;
```

**Criterios de aceptación:**
- [ ] `catMetodoDePagoCFDI` contiene CONT y CRED con `IdRegion = PER` y `Activo = 1`
- [ ] `catUsoCFDI` contiene 01, 03 y 08 con `IdRegion = PER` y `Activo = 1`
- [ ] Los registros MX existentes no son modificados
- [ ] Scripts incluidos en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-02 y Pendiente P1 del archivo `R16A-RE-FU-005-Back.md`
- Reglas R3 y R5 del requisito: Condición de Pago PE diferenciada de UsoCFDI MX

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005.md`

---
## Tarea 3

### R16A-RE-FU-005  GAP-03  [ UPDATE-TABL-CH ] Agregar campos PE a `DatosFacturacionCliente` (`AgenteRetencionIGV`, `SujetoDetraccion`, `TasaDetraccion`)

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogos de Cobros — Configuración de Facturación de Cliente

**Consideraciones previas:**
Esta tarea está **bloqueada por los Pendientes P5 y P6**: confirmar con el cliente si PROQUIFA Perú tiene clientes designados como Agentes de Retención IGV (P5) y si sus productos/servicios están sujetos a Detracción SPOT (P6).
No ejecutar el ALTER TABLE sin confirmación. Si tanto P5 como P6 se resuelven como "No aplica", esta tarea se cancela.
El Pendiente P3 también afecta esta tarea: confirmar si la tasa de detracción se captura a nivel cliente, nivel producto o catálogo SUNAT.

**Descripción del problema:**
`DatosFacturacionCliente` no tiene los campos de banderas tributarias PE requeridos por las Reglas R6 y R7 del requisito:
- `AgenteRetencionIGV` (bit): identifica si el cliente PE ha sido designado por SUNAT como Agente de Retención del IGV.
- `SujetoDetraccion` (bit): indica si las operaciones con el cliente están sujetas al Sistema de Pago de Obligaciones Tributarias (SPOT).
- `TasaDetraccion` (decimal 5,2): tasa aplicable cuando `SujetoDetraccion = 1` (condicionado a resolución de P3).

**Cambios requeridos (ejecutar solo tras confirmar P5 y P6):**

```sql
ALTER TABLE dbo.DatosFacturacionCliente
    ADD AgenteRetencionIGV  bit           NOT NULL CONSTRAINT DF_DFC_AgenteRetencionIGV DEFAULT 0,
        SujetoDetraccion    bit           NOT NULL CONSTRAINT DF_DFC_SujetoDetraccion   DEFAULT 0,
        TasaDetraccion      decimal(5,2)  NULL;

SELECT TOP 5 IdCliente, AgenteRetencionIGV, SujetoDetraccion, TasaDetraccion
FROM dbo.DatosFacturacionCliente;
```

**Criterios de aceptación:**
- [ ] Pendientes P5 y P6 confirmados con el cliente antes de ejecutar
- [ ] `DatosFacturacionCliente` contiene las columnas `AgenteRetencionIGV` (bit NOT NULL DEFAULT 0), `SujetoDetraccion` (bit NOT NULL DEFAULT 0) y `TasaDetraccion` (decimal NULL)
- [ ] Todos los registros existentes tienen `AgenteRetencionIGV = 0` y `SujetoDetraccion = 0` por el DEFAULT
- [ ] SELECT de verificación devuelve los tres campos correctamente
- [ ] Scripts incluidos en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-03 y Pendientes P3, P5, P6 del archivo `R16A-RE-FU-005-Back.md`
- Reglas R6 y R7 del requisito: banderas tributarias PE

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005.md`

---
## Tarea 4

### R16A-RE-FU-005  GAP-04  [ IMP-EXIST-SERVICE ] Aplicar filtro de región en `catMetodoDePagoCFDIController`, `catUsoCFDIController` y `catMedioDePagoController`

**Aplicativos:**
ProquifaNet 2 — WebApi.Catalogos

**Módulos:**
Catálogos de Cobros — Configuración de Facturación de Cliente

**Consideraciones previas:**
**Depende de Tarea 1.** Los tres catálogos deben tener `IdRegion` en BD antes de que el filtro en el controller tenga efecto.
Revisar que el using `Core.Pqf.WebApi.ControllerOperations` se agregue a los tres archivos al reemplazar `ApiController`.
Referencia del patrón: `WebApi.Catalogos\Controllers\Catalogos\catRutaEntregaController.cs`.

**Descripción del problema:**
Los tres controllers de catálogos de cobros heredan `ApiController` directamente. El método `QueryResult` no aplica ningún filtro por región: devuelve todos los registros de BD sin importar la región del usuario logueado.
Esto significa que un usuario de México ve opciones PE en los selectores y un usuario de Perú ve opciones MX — violando la Regla R2 del requisito.

**Estado actual de los tres controllers:**

```csharp
public class catMetodoDePagoCFDIController : ApiController, IControllerCatalogoBO<catMetodoDePagoCFDI>
{
    [HttpPost]
    [Route("catMetodoDePagoCFDI")]
    public QueryResult<catMetodoDePagoCFDI> QueryResult([FromBody] QueryInfo info)
    {
        var modelBO = new TablaGenericaBO<catMetodoDePagoCFDI>();
        return modelBO.QueryResult(info);  // SIN filtro de región
    }
}
```

**Cambios requeridos — `catMetodoDePagoCFDIController.cs`:**

```csharp
// ANTES — declaración de clase
public class catMetodoDePagoCFDIController : ApiController, IControllerCatalogoBO<catMetodoDePagoCFDI>

// DESPUÉS — declaración de clase
public class catMetodoDePagoCFDIController : BaseApiController
```

```csharp
// ANTES — QueryResult
public QueryResult<catMetodoDePagoCFDI> QueryResult([FromBody] QueryInfo info)
{
    var modelBO = new TablaGenericaBO<catMetodoDePagoCFDI>();
    return modelBO.QueryResult(info);
}

// DESPUÉS — QueryResult
public QueryResult<catMetodoDePagoCFDI> QueryResult([FromBody] QueryInfo info)
{
    var user = ObtenerUsuarioAutenticado();
    AsegurarFiltroRegion(info, user.IdRegion);
    var modelBO = new TablaGenericaBO<catMetodoDePagoCFDI>();
    return modelBO.QueryResult(info);
}
```

**Cambios requeridos — `catUsoCFDIController.cs` (mismo patrón):**

```csharp
// ANTES — declaración de clase
public class catUsoCFDIController : ApiController, IControllerCatalogoBO<catUsoCFDI>

// DESPUÉS — declaración de clase
public class catUsoCFDIController : BaseApiController
```

```csharp
// ANTES — QueryResult
public QueryResult<catUsoCFDI> QueryResult([FromBody] QueryInfo info)
{
    var modelBO = new TablaGenericaBO<catUsoCFDI>();
    return modelBO.QueryResult(info);
}

// DESPUÉS — QueryResult
public QueryResult<catUsoCFDI> QueryResult([FromBody] QueryInfo info)
{
    var user = ObtenerUsuarioAutenticado();
    AsegurarFiltroRegion(info, user.IdRegion);
    var modelBO = new TablaGenericaBO<catUsoCFDI>();
    return modelBO.QueryResult(info);
}
```

**Cambios requeridos — `catMedioDePagoController.cs` (mismo patrón):**

```csharp
// ANTES — declaración de clase
public class catMedioDePagoController : ApiController, IControllerCatalogoBO<catMedioDePago>

// DESPUÉS — declaración de clase
public class catMedioDePagoController : BaseApiController
```

```csharp
// ANTES — QueryResult
public QueryResult<catMedioDePago> QueryResult([FromBody] QueryInfo info)
{
    var modelBO = new TablaGenericaBO<catMedioDePago>();
    return modelBO.QueryResult(info);
}

// DESPUÉS — QueryResult
public QueryResult<catMedioDePago> QueryResult([FromBody] QueryInfo info)
{
    var user = ObtenerUsuarioAutenticado();
    AsegurarFiltroRegion(info, user.IdRegion);
    var modelBO = new TablaGenericaBO<catMedioDePago>();
    return modelBO.QueryResult(info);
}
```

**Archivos a modificar:**
- `WebApi.Catalogos\Controllers\Catalogos\catMetodoDePagoCFDIController.cs`
- `WebApi.Catalogos\Controllers\Catalogos\catUsoCFDIController.cs`
- `WebApi.Catalogos\Controllers\Catalogos\catMedioDePagoController.cs`

**Criterios de aceptación:**
- [ ] Los tres controllers heredan `BaseApiController` (ya no `ApiController`)
- [ ] El método `QueryResult` de cada controller invoca `ObtenerUsuarioAutenticado()` y `AsegurarFiltroRegion(info, user.IdRegion)` antes de consultar
- [ ] Un usuario de Región México recibe solo opciones MEX en los tres selectores
- [ ] Un usuario de Región Perú recibe solo opciones PER en `catMetodoDePagoCFDI` y `catUsoCFDI` (y ningún resultado en `catMedioDePago` porque no hay registros PE)
- [ ] Los métodos `Obtener`, `GuardarOActualizar` y `Desactivar` no son modificados
- [ ] El using `Core.Pqf.WebApi.ControllerOperations` está presente en los tres archivos
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-04 del archivo `R16A-RE-FU-005-Back.md`
- Patrón de referencia: `WebApi.Catalogos\Controllers\Catalogos\catRutaEntregaController.cs`
- Criterios C4, C5 y C6 del análisis de impacto

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005.md`

---
## Tarea 5

### R16A-RE-FU-005  GAP-05  [ BD-OBJ-CH ] Revisar y actualizar `vDatosFacturacionCliente` para exponer campos PE

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogos de Cobros — Configuración de Facturación de Cliente

**Consideraciones previas:**
**Depende de Tarea 3.** Esta tarea solo aplica si se ejecuta el ALTER TABLE de `DatosFacturacionCliente` (GAP-03).
Si la Tarea 3 se cancela por resolución de P5 y P6, esta tarea también se cancela.
Confirmar con TechLead si hay otros módulos que consumen `vDatosFacturacionCliente` antes de modificar la vista.

**Descripción del problema:**
`vDatosFacturacionCliente` es una vista que expone los datos de facturación del cliente a otros módulos del sistema (Factura por Adelantado, Validar Cobro). Si se agregan `AgenteRetencionIGV`, `SujetoDetraccion` y `TasaDetraccion` a la tabla `DatosFacturacionCliente`, la vista debe actualizarse para exponer esos campos. De lo contrario, los módulos consumidores no podrán leer las banderas tributarias PE.

**Cambios requeridos:**
1. Revisar la definición actual de `vDatosFacturacionCliente` con:

```sql
SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.vDatosFacturacionCliente'));
```

2. Agregar las tres nuevas columnas al SELECT de la vista:

```sql
ALTER VIEW dbo.vDatosFacturacionCliente AS
SELECT
    -- columnas existentes sin cambio --
    dfc.AgenteRetencionIGV,
    dfc.SujetoDetraccion,
    dfc.TasaDetraccion
FROM dbo.DatosFacturacionCliente dfc
-- resto del JOIN existente sin cambio --;
```

> El script completo se genera tras revisar la definición actual de la vista con el SELECT anterior.

**Criterios de aceptación:**
- [ ] Tarea 3 ejecutada y confirmada antes de iniciar esta tarea
- [ ] `vDatosFacturacionCliente` expone `AgenteRetencionIGV`, `SujetoDetraccion` y `TasaDetraccion`
- [ ] Los campos ya existentes de la vista no son modificados ni eliminados
- [ ] Los módulos consumidores (Factura por Adelantado, Validar Cobro) pueden leer los tres campos nuevos
- [ ] Script de ALTER VIEW incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-05 del archivo `R16A-RE-FU-005-Back.md`
- Criterios C7 y C8 del análisis de impacto
- Módulos consumidores: Factura por Adelantado y Validar Cobro

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005.md`

---
## Tarea 6

### R16A-RE-FU-005  GAP-06  [ QUERY-CH ] Completar `ClaveFormaDePago` en `catMedioDePago` para registros sin clave SAT

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogos de Cobros — Forma de Pago México

**Consideraciones previas:**
Esta tarea está **bloqueada por el Pendiente P4**: confirmar con el cliente qué clave SAT c_FormaPago corresponde a los registros Aba, NA, —NINGUNO— y Swift.
No ejecutar los UPDATE sin confirmación. Si el cliente determina que esos registros no afectan el timbrado CFDI (porque siempre se selecciona otro medio de pago para timbrado), la tarea puede reducirse a solo documentar la decisión.

**Descripción del problema:**
Cuatro registros activos en `catMedioDePago` no tienen valor en `ClaveFormaDePago` (clave SAT c_FormaPago):

| Descripción | Clave SAT actual | Activo |
|-------------|-----------------|--------|
| Aba | *(vacío)* | Sí |
| NA | *(vacío)* | Sí |
| —NINGUNO— | *(vacío)* | Sí |
| Swift | *(vacío)* | Sí |

Si alguno de estos registros está asociado a una factura que se timbra ante el SAT, el XML CFDI puede ser rechazado por clave de FormaPago vacía o inválida.

**Diagnóstico previo recomendado:**

```sql
SELECT mp.MedioDePago, mp.ClaveFormaDePago, COUNT(cp.IdConfiguracionPagos) AS ClientesAsociados
FROM       dbo.catMedioDePago      mp
LEFT  JOIN dbo.ConfiguracionPagos  cp ON mp.IdCatMedioDePago = cp.IdCatMedioDePago
WHERE mp.ClaveFormaDePago IS NULL OR mp.ClaveFormaDePago = ''
GROUP BY mp.MedioDePago, mp.ClaveFormaDePago
ORDER BY ClientesAsociados DESC;
```

**Cambios requeridos (ejecutar tras confirmar P4):**

```sql
UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'Aba';
UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'NA';
UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'NINGUNO';
UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'Swift';
```

> Reemplazar `'XX'` con la clave SAT confirmada por el cliente en cada caso.

**Criterios de aceptación:**
- [ ] Pendiente P4 confirmado con el cliente antes de ejecutar
- [ ] Diagnóstico previo ejecutado e interpretado: se sabe cuántos clientes activos usan cada registro sin clave SAT
- [ ] Los cuatro registros tienen `ClaveFormaDePago` asignada con el valor SAT confirmado (o se documenta la decisión de no asignar si aplica 99-Por definir)
- [ ] SELECT de verificación confirma que no quedan registros activos con `ClaveFormaDePago` vacía
- [ ] Scripts incluidos en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-06 y Pendiente P4 del archivo `R16A-RE-FU-005-Back.md`
- Regla R4 del requisito: Forma de Pago exclusiva de México
- Ver catálogo SAT c_FormaPago vigente para CFDI 4.0 al asignar claves

**Recursos:**
- Análisis de impacto backend: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005-Back.md`
- Diccionario de datos: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005_BD.md`
- Requisito funcional: `Requisitos/R16A-RE-FU-005/R16A-RE-FU-005.md`