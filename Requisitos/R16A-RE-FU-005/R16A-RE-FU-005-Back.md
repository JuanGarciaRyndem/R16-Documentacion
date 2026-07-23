# R16A-RE-FU-005 — Análisis de Impacto Backend

| Campo | Valor |
|-------|-------|
| Requisito | R16A-RE-FU-005 — Catálogos de Cobros y Configuración de Facturación |
| Rama | develop-pack04 |
| Fecha | 2025-07 |
| Revisión | v2.0 — Cancelación timbrado Perú: se elimina regionalización; se consolidan tres campos Cobros en `DatosFacturacionCliente` con validación obligatoria |

---

## Resumen de cambios v2.0

Con la cancelación del timbrado para Perú, se eliminan de este requisito:

- `IdRegion` en catálogos (`catMedioDePago`, `catMetodoDePagoCFDI`, `catUsoCFDI`)
- Registros PE (CONT/CRED en `catMetodoDePagoCFDI`; 01/03/08 en `catUsoCFDI`)
- Campos PE en `DatosFacturacionCliente` (`AgenteRetencionIGV`, `SujetoDetraccion`, `TasaDetraccion`)
- Filtro de región en controllers

Se mantiene y agrega:

- Validación de registros SAT en catálogos — **⏸ En espera** (pendiente Excel de Proquifa)
- `IdCatMedioDePago` (Forma de Pago) se agrega a `DatosFacturacionCliente` para centralizar los tres campos de Cobros en un solo objeto
- Validación obligatoria de los tres campos de Cobros al guardar `DatosFacturacionCliente`

---

## 1. Estructura actual en el proyecto

### 1.1 Capa de Lógica (Logic.Pqf.Catalogos)

| Archivo | Clase | Descripción |
|---------|-------|-------------|
| *(no existe BO específico)* | `TablaGenericaBO<catMetodoDePagoCFDI>` | CRUD genérico |
| *(no existe BO específico)* | `TablaGenericaBO<catUsoCFDI>` | CRUD genérico |
| *(no existe BO específico)* | `TablaGenericaBO<catMedioDePago>` | CRUD genérico |

### 1.2 Capa de API (WebApi.Catalogos)

| Archivo | Ruta HTTP |
|---------|-----------|
| `Controllers\Catalogos\catMetodoDePagoCFDIController.cs` | `/catMetodoDePagoCFDI` |
| `Controllers\Catalogos\catUsoCFDIController.cs` | `/catUsoCFDI` |
| `Controllers\Catalogos\catMedioDePagoController.cs` | `/catMedioDePago` |

> Los tres controllers no requieren filtro de región. Se mantienen con su patrón actual (`ApiController` + `TablaGenericaBO`).

---

## 2. Mapeo de reglas de negocio al código

| Regla | Descripción | Implementación esperada | Estado |
|-------|-------------|-------------------------|--------|
| **R1** — Campos de Cobros en `DatosFacturacionCliente` | Los tres campos (Forma de Pago, Uso de CFDI, Método de Pago) se almacenan en `DatosFacturacionCliente` | FKs desde `DatosFacturacionCliente` a `catMedioDePago`, `catUsoCFDI` y `catMetodoDePagoCFDI` | Parcial — `IdCatMedioDePago` no existe aún en `DatosFacturacionCliente` |
| **R2** — Validación campos obligatorios | Al guardar, los tres campos deben estar capturados; si alguno está vacío → error | Validación en BO antes de `GuardarOActualizar` | PENDIENTE |
| **R3** — Registros SAT validados | Los registros de los tres catálogos deben tener claves SAT correctas (cruzar con catálogo SAT + Excel Proquifa) | UPDATE masivo tras confirmar claves | ⏸ En espera |
| **R4** — Sin restricción de rol | Cualquier usuario con acceso puede consultar catálogos | Sin control de rol en backend | OK — sin cambio |

---

## 3. GAPs identificados

### GAP-01 — `IdCatMedioDePago` ausente en `DatosFacturacionCliente`

**Archivo:** BD — `DatosFacturacionCliente`

**Impacto:** Regla R1 — La Forma de Pago (`catMedioDePago`) se almacena actualmente en `ConfiguracionPagos`. Para centralizar y validar los tres campos de Cobros en un solo objeto, `IdCatMedioDePago` debe agregarse a `DatosFacturacionCliente`.

**Cambio requerido:**

```sql
-- Ejecutar en ProquifaDotNet
ALTER TABLE dbo.DatosFacturacionCliente
    ADD IdCatMedioDePago uniqueidentifier NULL
        CONSTRAINT FK_DatosFacturacionCliente_MedioDePago
            FOREIGN KEY REFERENCES dbo.catMedioDePago(IdCatMedioDePago);
GO
```

---

### GAP-02 — Validación de campos obligatorios de Cobros al guardar cliente

**Archivo:** BO o Controller de `DatosFacturacionCliente`

**Impacto:** Regla R2 — Al invocar `GuardarOActualizar` en `DatosFacturacionCliente`, el sistema debe verificar que los tres campos de Cobros estén capturados. Si alguno está vacío, retornar un error antes de persistir.

**Campos que deben validarse como obligatorios:**

| Campo UI | FK en `DatosFacturacionCliente` | Catálogo |
|----------|----------------------------------|----------|
| Forma de Pago | `IdCatMedioDePago` (NUEVO — GAP-01) | `catMedioDePago` |
| Uso de CFDI | `IdCatUsoCFDI` (existente) | `catUsoCFDI` |
| Método de Pago | `IdCatMetodoDePagoCFDI` (existente) | `catMetodoDePagoCFDI` |

**Lógica de validación:**

```csharp
// En el BO de DatosFacturacionCliente — antes de GuardarOActualizar
private void ValidarCamposCobros(DatosFacturacionCliente dfc)
{
    var errores = new List<string>();

    if (dfc.IdCatMedioDePago == null || dfc.IdCatMedioDePago == Guid.Empty)
        errores.Add("La Forma de Pago es obligatoria.");

    if (dfc.IdCatUsoCFDI == null || dfc.IdCatUsoCFDI == Guid.Empty)
        errores.Add("El Uso de CFDI es obligatorio.");

    if (dfc.IdCatMetodoDePagoCFDI == null || dfc.IdCatMetodoDePagoCFDI == Guid.Empty)
        errores.Add("El Método de Pago es obligatorio.");

    if (errores.Any())
        throw new ValidationException(string.Join(" | ", errores));
}
```

> La excepción debe propagarse al frontend con un mensaje claro por campo vacío.

---

### GAP-03 — Registros SAT sin validar en catálogos ⏸ En espera

**Archivos:** BD — `catMedioDePago`, `catMetodoDePagoCFDI`, `catUsoCFDI`

**Impacto:** Regla R3 — Los registros existentes en los tres catálogos deben tener claves SAT correctas para que el timbrado CFDI no sea rechazado por el PAC.

**Estado:** En espera del Excel de equivalencias que enviará Proquifa. No ejecutar hasta recibirlo y confirmarlo.

Registros con clave SAT incompleta conocida:

| Tabla | Registro | Campo | Estado |
|-------|----------|-------|--------|
| `catMedioDePago` | Aba | `ClaveFormaDePago` vacío | ⏸ Pendiente Excel |
| `catMedioDePago` | NA | `ClaveFormaDePago` vacío | ⏸ Pendiente Excel |
| `catMedioDePago` | —NINGUNO— | `ClaveFormaDePago` vacío | ⏸ Pendiente Excel |
| `catMedioDePago` | Swift | `ClaveFormaDePago` vacío | ⏸ Pendiente Excel |

```sql
-- Ejecutar tras recibir y confirmar el Excel de Proquifa con las claves SAT
-- UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'Aba';
-- UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'NA';
-- UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'NINGUNO';
-- UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'Swift';
```

---

## 4. Tablas y entidades del modelo de datos

| Tabla BD | Propiedades clave R16 | Descripción |
|----------|-----------------------|-------------|
| `catMedioDePago` | `IdCatMedioDePago` (PK), `ClaveFormaDePago` | Forma de Pago — clave SAT c_FormaPago |
| `catMetodoDePagoCFDI` | `IdCatMetodoDePagoCFDI` (PK), `ClaveMetodoDePagoCFDI` | Método de Pago CFDI: PUE/PPD |
| `catUsoCFDI` | `IdCatUsoCFDI` (PK), `ClaveUso` | Uso de CFDI: G01/G03/S01 etc. |
| `DatosFacturacionCliente` | `IdCatMedioDePago` (FK — **NUEVO**), `IdCatUsoCFDI` (FK), `IdCatMetodoDePagoCFDI` (FK) | Tres campos de Cobros del cliente |

---

## 5. Consulta de referencia — clientes con Cobros incompletos

Útil para diagnóstico pre-producción:

```sql
SELECT
    c.Nombre         AS Cliente,
    CASE WHEN dfc.IdCatMedioDePago      IS NULL THEN 'FALTA' ELSE 'OK' END AS FormaDePago,
    CASE WHEN dfc.IdCatUsoCFDI          IS NULL THEN 'FALTA' ELSE 'OK' END AS UsoCFDI,
    CASE WHEN dfc.IdCatMetodoDePagoCFDI IS NULL THEN 'FALTA' ELSE 'OK' END AS MetodoPago
FROM       dbo.Cliente                 c
LEFT JOIN  dbo.DatosFacturacionCliente dfc ON c.IdCliente = dfc.IdCliente AND dfc.Activo = 1
WHERE dfc.IdCatMedioDePago      IS NULL
   OR dfc.IdCatUsoCFDI          IS NULL
   OR dfc.IdCatMetodoDePagoCFDI IS NULL;
```

---

## 6. Resumen de archivos a modificar

| # | Archivo | Tipo de cambio |
|---|---------|----------------|
| 1 | BD — `DatosFacturacionCliente` | `ALTER TABLE`: agregar `IdCatMedioDePago` FK (GAP-01) |
| 2 | BO/Controller — `DatosFacturacionCliente` | Agregar validación de tres campos obligatorios (GAP-02) |
| 3 | BD — `catMedioDePago` | `UPDATE ClaveFormaDePago` — ⏸ En espera Excel Proquifa (GAP-03) |

---

## 7. Archivos que NO requieren cambio en R16

| Archivo | Motivo |
|---------|--------|
| `catMetodoDePagoCFDIController.cs` | Sin filtro de región — sin cambio |
| `catUsoCFDIController.cs` | Ídem |
| `catMedioDePagoController.cs` | Ídem |
| `ConfiguracionPagos` (tabla BD) | Se mantiene para condiciones de crédito y línea de crédito; `IdCatMedioDePago` se agrega a `DatosFacturacionCliente` como campo adicional de Cobros |
| `catCondicionesDePago` (tabla BD) | Plazos de crédito en días — sin cambios |

---

## 8. Pendientes / Decisiones abiertas

| # | Pendiente | Responsable |
|---|-----------|-------------|
| P1 | Excel de equivalencias SAT por catálogo — necesario para completar GAP-03 | Proquifa / Funcional |
| P2 | Confirmar si el `IdCatMedioDePago` que ya existe en `ConfiguracionPagos` se mantiene o se retira tras agregar el FK en `DatosFacturacionCliente` | TechLead |
| P3 | Confirmar clave SAT c_FormaPago para Aba, NA, NINGUNO y Swift | Funcional / Cliente |

---

## 9. Criterios de aceptación técnica

- [ ] `DatosFacturacionCliente` tiene columna `IdCatMedioDePago` (FK → `catMedioDePago`).
- [ ] Al guardar `DatosFacturacionCliente` con **Forma de Pago** vacía → mensaje de error específico.
- [ ] Al guardar `DatosFacturacionCliente` con **Uso de CFDI** vacío → mensaje de error específico.
- [ ] Al guardar `DatosFacturacionCliente` con **Método de Pago** vacío → mensaje de error específico.
- [ ] Los tres catálogos retornan solo registros `Activo = 1`.
- [ ] `catMedioDePago` tiene `ClaveFormaDePago` completa en todos los registros activos — ⏸ condicionado a GAP-03.
