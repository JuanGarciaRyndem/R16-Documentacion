# TPSC-RE-FU-005 — Análisis de Impacto Backend

| Campo | Valor |
|-------|-------|
| Requisito | TPSC-RE-FU-005 — Catálogos de Cobros y Configuración de Facturación |
| Rama | develop-pack04 |
| Fecha | 2025-07 |
| Revisión | v1.1 — Estilo ampliado con código C# de implementación |

---

## 1. Estructura actual en el proyecto

### 1.1 Capa de Lógica (Logic.Pqf.Catalogos)

| Archivo | Clase | Descripción |
|---------|-------|-------------|
| *(no existe BO específico)* | `TablaGenericaBO<catMetodoDePagoCFDI>` | CRUD genérico. Sin filtro de región. |
| *(no existe BO específico)* | `TablaGenericaBO<catUsoCFDI>` | CRUD genérico. Sin filtro de región. |
| *(no existe BO específico)* | `TablaGenericaBO<catMedioDePago>` | CRUD genérico. Sin filtro de región. |

> Los tres catálogos no tienen BO propio. El filtro de región se aplica en el controlador mediante `AsegurarFiltroRegion`.

### 1.2 Capa de API (WebApi.Catalogos)

| Archivo | Ruta HTTP | Herencia actual | Filtro región |
|---------|-----------|-----------------|---------------|
| `Controllers\Catalogos\catMetodoDePagoCFDIController.cs` | `/catMetodoDePagoCFDI` | `ApiController` | No |
| `Controllers\Catalogos\catUsoCFDIController.cs` | `/catUsoCFDI` | `ApiController` | No |
| `Controllers\Catalogos\catMedioDePagoController.cs` | `/catMedioDePago` | `ApiController` | No |

**Patrón de referencia con filtro de región:**

| Archivo | Ruta HTTP | Herencia | Filtro región |
|---------|-----------|----------|---------------|
| `Controllers\Catalogos\catRutaEntregaController.cs` | `/catRutaEntrega` | `BaseApiController` | Sí |

---

## 2. Mapeo de reglas de negocio al código

| Regla | Descripción | Implementación esperada | Estado actual |
|-------|-------------|-------------------------|---------------|
| **R1** — Catálogos como configuración default | `DatosFacturacionCliente` y `ConfiguracionPagos` almacenan opciones default del cliente. | Tablas existentes. FK desde `DatosFacturacionCliente` a `catMetodoDePagoCFDI` y `catUsoCFDI`. | OK — tablas y FK existen |
| **R2** — Catálogos diferenciados por Región | Los tres catálogos filtran por región del usuario logueado. | `IdRegion` en catálogos. Controllers heredan `BaseApiController` y llaman `AsegurarFiltroRegion`. | PENDIENTE — `IdRegion` no existe; controllers sin filtro |
| **R3** — Condición de Pago PE | `catMetodoDePagoCFDI` con `IdRegion = PER` contiene CONT y CRED. | Registros PE en BD con `IdRegion = PER`. | PENDIENTE — sin registros PE |
| **R4** — Método de Pago solo México | PUE/PPD asignados a `IdRegion = MEX`. `catMedioDePago` sin registros PE. | UPDATE masivo tras ALTER TABLE. | PENDIENTE — `IdRegion` aún no existe |
| **R5** — UsoCFDI y TipoComprobante distintos | `catUsoCFDI` con `IdRegion = MEX` para MX y `IdRegion = PER` para PE. | Registros PE: 01, 03 y 08 con `IdRegion = PER`. | PENDIENTE — sin registros PE |
| **R6** — Agente de Retención IGV | `DatosFacturacionCliente.AgenteRetencionIGV` (bit). | Columna bit en `DatosFacturacionCliente`. | NO EXISTE — condicionado a P5 |
| **R7** — Sujeto a Detracción | `DatosFacturacionCliente.SujetoDetraccion` (bit) + `TasaDetraccion` (decimal). | Dos columnas en `DatosFacturacionCliente`. | NO EXISTE — condicionado a P6 |
| **R8** — Sin restricción de rol | Cualquier usuario con acceso puede seleccionar catálogos. | Sin control de rol en backend. | OK — sin restricción, correcto |

---

## 3. GAPs identificados y código de implementación

### GAP-01 — `IdRegion` ausente en los tres catálogos

**Archivos:** BD — `catMetodoDePagoCFDI`, `catUsoCFDI`, `catMedioDePago`
**Impacto:** Regla R2 — Sin `IdRegion` no es posible aplicar filtro por región en ninguno de los tres selectores.
**Cambio requerido:** `ALTER TABLE` + `UPDATE` masivo asignando `IdRegion = MEX` a todos los registros existentes.

```sql
-- catMetodoDePagoCFDI
ALTER TABLE dbo.catMetodoDePagoCFDI
    ADD IdRegion uniqueidentifier NULL
    CONSTRAINT FK_catMetodoDePagoCFDI_Region
        FOREIGN KEY REFERENCES dbo.Region(IdRegion);

UPDATE dbo.catMetodoDePagoCFDI
SET    IdRegion = (SELECT IdRegion FROM dbo.Region WHERE ClaveISO = 'MEX')
WHERE  IdRegion IS NULL;

-- catUsoCFDI
ALTER TABLE dbo.catUsoCFDI
    ADD IdRegion uniqueidentifier NULL
    CONSTRAINT FK_catUsoCFDI_Region
        FOREIGN KEY REFERENCES dbo.Region(IdRegion);

UPDATE dbo.catUsoCFDI
SET    IdRegion = (SELECT IdRegion FROM dbo.Region WHERE ClaveISO = 'MEX')
WHERE  IdRegion IS NULL;

-- catMedioDePago
ALTER TABLE dbo.catMedioDePago
    ADD IdRegion uniqueidentifier NULL
    CONSTRAINT FK_catMedioDePago_Region
        FOREIGN KEY REFERENCES dbo.Region(IdRegion);

UPDATE dbo.catMedioDePago
SET    IdRegion = (SELECT IdRegion FROM dbo.Region WHERE ClaveISO = 'MEX')
WHERE  IdRegion IS NULL;
```
---

### GAP-02 — Registros PE ausentes en `catMetodoDePagoCFDI` y `catUsoCFDI`

**Archivos:** BD — `catMetodoDePagoCFDI`, `catUsoCFDI`
**Impacto:** Reglas R3 y R5 — El usuario de Perú no tiene opciones en los selectores de Condición de Pago y Tipo de Comprobante.
**Cambio requerido:** `INSERT` de registros PE. Ejecutar después de GAP-01.

```sql
DECLARE @IdRegionPER uniqueidentifier =
    (SELECT IdRegion FROM dbo.Region WHERE ClaveISO = 'PER');

-- Condición de Pago PE en catMetodoDePagoCFDI
INSERT INTO dbo.catMetodoDePagoCFDI
    (IdCatMetodoDePagoCFDI, MetodoDePagoCFDI, ClaveMetodoDePagoCFDI, Clave, Activo, IdRegion)
VALUES
    (NEWID(), N'Contado', 'CONT', 'CONT', 1, @IdRegionPER),
    (NEWID(), N'Crédito', 'CRED', 'CRED', 1, @IdRegionPER);

-- Tipo de Comprobante PE en catUsoCFDI
INSERT INTO dbo.catUsoCFDI
    (IdCatUsoCFDI, ClaveUso, Uso, Clave, Activo, IdRegion)
VALUES
    (NEWID(), '01', N'Factura electrónica',               '01', 1, @IdRegionPER),
    (NEWID(), '03', N'Boleta de venta electrónica',       '03', 1, @IdRegionPER),
    (NEWID(), '08', N'Recibo por Honorarios electrónico', '08', 1, @IdRegionPER);
```

---

### GAP-03 — Campos PE ausentes en `DatosFacturacionCliente`

**Archivo:** BD — `DatosFacturacionCliente`
**Impacto:** Reglas R6 y R7 — No es posible registrar si un cliente PE es Agente de Retención IGV o está sujeto a Detracción.
**Cambio requerido:** Condicionado a resolución de P5 y P6.

```sql
-- Ejecutar solo tras confirmar P5 (Agente Retención) y P6 (Detracción) con el cliente
ALTER TABLE dbo.DatosFacturacionCliente
    ADD AgenteRetencionIGV  bit           NOT NULL CONSTRAINT DF_DFC_AgenteRetencionIGV DEFAULT 0,
        SujetoDetraccion    bit           NOT NULL CONSTRAINT DF_DFC_SujetoDetraccion   DEFAULT 0,
        TasaDetraccion      decimal(5,2)  NULL;
```
---

### GAP-04 — Controllers sin filtro de región (`ApiController` en lugar de `BaseApiController`)

**Archivos:**
- `WebApi.Catalogos\Controllers\Catalogos\catMetodoDePagoCFDIController.cs`
- `WebApi.Catalogos\Controllers\Catalogos\catUsoCFDIController.cs`
- `WebApi.Catalogos\Controllers\Catalogos\catMedioDePagoController.cs`

**Impacto:** Regla R2 — `QueryResult` devuelve todos los registros sin filtrar por región. Un usuario de México ve opciones PE y viceversa.

**Estado actual — los tres controllers tienen este mismo patrón:**

```csharp
// catMetodoDePagoCFDIController — estado actual
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

**Cambio requerido — `catMetodoDePagoCFDIController`:**

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

**Cambio requerido — `catUsoCFDIController` (mismo patrón):**

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

**Cambio requerido — `catMedioDePagoController` (mismo patrón):**

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

> **Using a agregar en los tres archivos:** `using Core.Pqf.WebApi.ControllerOperations;`
> **Referencia del patrón:** `catRutaEntregaController.cs` — ya implementado y vigente en el proyecto.
---

### GAP-05 — `vDatosFacturacionCliente` puede requerir ajuste tras ALTER TABLE

**Archivo:** BD — `vDatosFacturacionCliente`
**Impacto:** Si GAP-03 se ejecuta, la vista debe exponer `AgenteRetencionIGV`, `SujetoDetraccion` y `TasaDetraccion`.
**Cambio requerido:** Revisar y actualizar la vista tras confirmar y ejecutar GAP-03. Condicionado a P5/P6.

---

### GAP-06 — `ClaveFormaDePago` incompleta en `catMedioDePago`

**Archivo:** BD — `catMedioDePago`
**Impacto:** Cuatro registros activos (Aba, NA, —NINGUNO—, Swift) no tienen `ClaveFormaDePago` (clave SAT c_FormaPago). Puede afectar el timbrado CFDI.
**Cambio requerido:** Confirmar clave SAT para cada registro (P4). Ejecutar UPDATE tras confirmación.

```sql
-- Ejemplo — ejecutar tras confirmar claves con cliente
UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'Aba';
UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'NA';
UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'NINGUNO';
UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'Swift';
```

---

## 4. Tablas y entidades del modelo de datos

| Tabla BD | Entidad EF | Propiedades clave R16 | Descripción |
|----------|------------|-----------------------|-------------|
| `catMetodoDePagoCFDI` | `catMetodoDePagoCFDI` | `IdCatMetodoDePagoCFDI` (PK), `IdRegion` (FK — NUEVO R16) | MX: PUE/PPD; PE: CONT/CRED |
| `catUsoCFDI` | `catUsoCFDI` | `IdCatUsoCFDI` (PK), `IdRegion` (FK — NUEVO R16) | MX: G01/G03/S01; PE: 01/03/08 |
| `catMedioDePago` | `catMedioDePago` | `IdCatMedioDePago` (PK), `IdRegion` (FK — NUEVO R16) | Forma de Pago — solo MEX |
| `DatosFacturacionCliente` | `DatosFacturacionCliente` | `AgenteRetencionIGV` (NUEVO), `SujetoDetraccion` (NUEVO), `TasaDetraccion` (NUEVO) | Config. fiscal default del cliente |
| `ConfiguracionPagos` | `ConfiguracionPagos` | `IdCatMedioDePago` (FK — sin cambio) | Config. cobros default del cliente |
| `Region` | `Region` | `IdRegion` (PK), `ClaveISO` | MEX / PER |

---

## 5. Consulta SQL de referencia — clientes sin configuración de facturación completa

Útil para diagnóstico pre-producción: identifica clientes a los que les falta algún campo de facturación obligatorio.

```sql
SELECT
    c.Nombre          AS Cliente,
    r.ClaveISO        AS Region,
    cfg.IdCatMedioDePago,
    dfc.IdCatMetodoDePagoCFDI,
    dfc.IdCatUsoCFDI
FROM       dbo.Cliente                  c
INNER JOIN dbo.Region                   r   ON c.IdRegion             = r.IdRegion
LEFT  JOIN dbo.ConfiguracionPagos       cfg ON c.IdConfiguracionPagos = cfg.IdConfiguracionPagos
LEFT  JOIN dbo.DatosFacturacionCliente  dfc ON c.IdCliente = dfc.IdCliente
                                           AND dfc.Activo = 1
WHERE (cfg.IdCatMedioDePago      IS NULL AND r.ClaveISO = 'MEX')
   OR dfc.IdCatMetodoDePagoCFDI IS NULL
   OR dfc.IdCatUsoCFDI          IS NULL;
```

---

## 6. Resumen de archivos a modificar

| # | Archivo | Tipo de cambio |
|---|---------|----------------|
| 1 | BD — `catMetodoDePagoCFDI` | ALTER TABLE IdRegion + UPDATE MEX + INSERT PE |
| 2 | BD — `catUsoCFDI` | ALTER TABLE IdRegion + UPDATE MEX + INSERT PE (01, 03, 08) |
| 3 | BD — `catMedioDePago` | ALTER TABLE IdRegion + UPDATE MEX + UPDATE claves SAT faltantes (tras P4) |
| 4 | BD — `DatosFacturacionCliente` | ALTER TABLE 3 campos PE — condicionado a P5/P6 |
| 5 | BD — `vDatosFacturacionCliente` | Revisar para exponer campos PE — condicionado a #4 |
| 6 | `Controllers\Catalogos\catMetodoDePagoCFDIController.cs` | ApiController a BaseApiController + AsegurarFiltroRegion en QueryResult |
| 7 | `Controllers\Catalogos\catUsoCFDIController.cs` | Ídem |
| 8 | `Controllers\Catalogos\catMedioDePagoController.cs` | Ídem |

---

## 7. Archivos que NO requieren cambio en R16

| Archivo | Motivo |
|---------|--------|
| `ConfiguracionPagos` (tabla BD) | Sin cambios estructurales. `IdCatMedioDePago` ya existe como FK. |
| `catCondicionesDePago` (tabla BD) | Plazos de crédito en días. No es la Condición de Pago SUNAT. Sin cambios. |
| Controllers `Obtener`, `GuardarOActualizar`, `Desactivar` de los tres catálogos | Sin cambio de comportamiento requerido en R16. |

---

## 8. Pendientes / Decisiones abiertas

| # | Pendiente | Responsable |
|---|-----------|-------------|
| P1 | Confirmar denominación final del campo Condición de Pago en pantalla para Perú (Contado/Crédito vs. otra nomenclatura) | Funcional / Cliente |
| P2 | Confirmar si el catálogo legacy debe mapearse para facturas y pedidos existentes al agregar `IdRegion` | Funcional / TechLead |
| P3 | Confirmar porcentaje de detracción: nivel cliente, nivel producto o catálogo SUNAT | Funcional / Cliente |
| P4 | Confirmar clave SAT para Aba, NA, NINGUNO y Swift en `catMedioDePago` para timbrado CFDI | Funcional / Cliente |
| P5 | Confirmar si PROQUIFA Perú tiene clientes designados como Agentes de Retención IGV por SUNAT | Funcional / Cliente |
| P6 | Confirmar si los productos/servicios de PROQUIFA Perú están sujetos a Detracción (SPOT) | Funcional / Cliente |
| P7 | Confirmar si PROQUIFA Perú es Agente de Percepción IGV (condición del emisor) | Funcional / Cliente |

---

## 9. Criterios de aceptación técnica

- [ ] Los catálogos `catMetodoDePagoCFDI`, `catUsoCFDI` y `catMedioDePago` tienen el campo `IdRegion` y todos los registros existentes tienen asignado `IdRegion = MEX`.
- [ ] `catMetodoDePagoCFDI` contiene CONT y CRED con `IdRegion = PER` y `Activo = 1`.
- [ ] `catUsoCFDI` contiene 01, 03 y 08 con `IdRegion = PER` y `Activo = 1`.
- [ ] Los tres controllers heredan `BaseApiController` y aplican `AsegurarFiltroRegion` en `QueryResult`.
- [ ] Un usuario de Región México recibe solo opciones MEX en los tres selectores.
- [ ] Un usuario de Región Perú recibe solo opciones PER en `catMetodoDePagoCFDI` y `catUsoCFDI`. `catMedioDePago` no aplica a Perú.
- [ ] `DatosFacturacionCliente` contiene `AgenteRetencionIGV`, `SujetoDetraccion` y `TasaDetraccion` — condicionado a P5/P6.
- [ ] `vDatosFacturacionCliente` expone los campos nuevos PE correctamente — condicionado al criterio anterior.