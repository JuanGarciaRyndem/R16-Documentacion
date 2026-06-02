# Diccionario de Datos — Catálogo de Cuentas Bancarias PROQUIFA

**Requisito:** TPSC-RE-FU-001
**Base de Datos:** ProquifaDotNet
**Servidor:** RYNL010
**Versión:** 1.4 — `IdRegion` en `EmpresaDatosBancarios` + Vista `vEmpresaDatosBancarios`

---

## Resumen Ejecutivo

Implementación del Catálogo de Cuentas Bancarias del grupo PROQUIFA como origen único
de información para módulos de cobro y pago.

### Empresas en Alcance R16

| Prefijo | Empresa |
|---------|---------|
| **GOL** | Golocaer |
| **MUN** | Mungen |
| **PRO** | Proquifa |
| **PQF** | Proveedora Químico Farmacéutica |

> ⚠️ Fuera de alcance R16: **GOLPERU** (Golocaer S.A.C. — Perú)

---

## Modelo de Datos

```
EmpresaDatosBancarios  (IdRegion — NUEVO R16)
    FK IdEmpresa         -> Empresa
    FK IdRegion          -> Region  (MEX / PER)
    FK IdDatosBancarios  -> DatosBancarios
            FK IdCatBanco  -> catBanco
            FK IdCatMoneda -> catMoneda

Vista: vEmpresaDatosBancarios  (NUEVA R16)
    Unión de EmpresaDatosBancarios + Empresa + Region + DatosBancarios + catBanco + catMoneda
```

---

## Entidades Afectadas

| Objeto | Tipo | Estado | Descripción |
|--------|------|--------|-------------|
| `EmpresaDatosBancarios` | Tabla | ✏️ Existente — **IdRegion NUEVO R16** | Catálogo principal de cuentas bancarias |
| `vEmpresaDatosBancarios` | Vista | 🆕 **NUEVA R16** | Vista operativa con unión completa |
| `EmpresaRegion` | Tabla | ✅ Existente — sin cambios | Vincula Empresa con Región |
| `DatosBancarios` | Tabla | ✅ Existente — sin cambios | Detalle de cuenta bancaria |
| `Empresa` | Tabla | ✅ Existente — sin cambios | Empresas GOL/MUN/PRO/PQF |
| `Region` | Tabla | ✅ Existente — sin cambios | MEX / PER |
| `catBanco` | Catálogo | ✅ Existente — sin cambios | Instituciones bancarias |
| `catMoneda` | Catálogo | ✅ Existente — sin cambios | MXN / USD |
| `catMedioDePago` | Catálogo | ✅ Existente — sin cambios | Formas de pago |
| `fcppOrdenDePago` | Tabla | ✅ Existente — consumidor | FK `IdEmpresaDatosBancarios` |
| `fppEjecucionOrdenDePago` | Tabla | ✅ Existente — consumidor | FK `IdEmpresaDatosBancarios` |
| `fppSeguimientoPagoFactura` | Tabla | ✅ Existente — consumidor indirecto | Seguimiento por cuenta destino |
| `fppEjecucionOrdenDePagoDestino` | Tabla | ✅ Existente — consumidor indirecto | Destino de ejecución de pago |
| `fcppSeguimientoFactura` | Tabla | ✅ Existente — consumidor indirecto | Seguimiento factura con cuenta |
| `fcppSeguimientoFacturaIndirecto` | Tabla | ✅ Existente — consumidor indirecto | Datos bancarios inline en seguimiento |
| `vClienteDatosBancarios` | Vista | 🚫 Fuera de alcance R16 | Datos bancarios de clientes externos |

---

## Orden de Ejecución de Scripts R16

| Paso | Script | Dependencia |
|------|--------|-------------|
| 1 | `ALTER TABLE EmpresaDatosBancarios` | Ninguna — ejecutar primero |
| 2 | `CREATE VIEW vEmpresaDatosBancarios` | Requiere Paso 1 completado |

---

## 1. EmpresaDatosBancarios (TABLA PRINCIPAL)

**Propósito:** Catálogo principal de cuentas bancarias del grupo PROQUIFA.
**Creada:** 12/10/2020
**Cambio R16:** Agregar `IdRegion` (FK → `Region`) para regionalizar cuentas (MEX / PER).

### Estructura de columnas

| Columna | Tipo | Longitud | Nulo | Default | Descripción |
|---------|------|:--------:|:----:|---------|-------------|
| `IdEmpresaDatosBancarios` | uniqueidentifier | 16 | NO | `NEWID()` | PK |
| `IdDatosBancarios` | uniqueidentifier | 16 | NO | `NEWID()` | FK → DatosBancarios |
| `IdEmpresa` | uniqueidentifier | 16 | SÍ | — | FK → Empresa |
| `FechaRegistro` | datetime | 8 | NO | `GETDATE()` | Fecha de alta |
| `FechaUltimaActualizacion` | datetime | 8 | NO | `GETDATE()` | Fecha de modificación |
| `Activo` | bit | 1 | NO | `1` | 1 = Vigente / 0 = No vigente |
| 🆕 **`IdRegion`** | **uniqueidentifier** | **16** | **SÍ** | **—** | **NUEVO R16 — FK → Region (MEX / PER)** |

### Foreign Keys

| Constraint | Columna | Referencia |
|------------|---------|------------|
| `FK_EmpresaDatosBancarios_DatosBancarios` | `IdDatosBancarios` | `DatosBancarios.IdDatosBancarios` |
| `FK_EmpresaDatosBancarios_Empresa` | `IdEmpresa` | `Empresa.IdEmpresa` |
| 🆕 **`FK_EmpresaDatosBancarios_Region`** | **`IdRegion`** | **`Region.IdRegion`** (NUEVO R16) |

### Tablas que referencian esta tabla

| Tabla | Columna |
|-------|---------|
| `fcppOrdenDePago` | `IdEmpresaDatosBancarios` |
| `fppEjecucionOrdenDePago` | `IdEmpresaDatosBancarios` |

### Script — Paso 1

```sql
-- Paso 1.1: Agregar IdRegion a EmpresaDatosBancarios
ALTER TABLE dbo.EmpresaDatosBancarios
    ADD IdRegion uniqueidentifier NULL
        CONSTRAINT FK_EmpresaDatosBancarios_Region
            FOREIGN KEY REFERENCES dbo.Region(IdRegion);

-- Paso 1.2: Asignar México a todos los registros existentes
DECLARE @IdMexico uniqueidentifier = '60390fda-7773-4ba1-8120-cb874f3a3a53';

UPDATE dbo.EmpresaDatosBancarios
SET    IdRegion                  = @IdMexico,
       FechaUltimaActualizacion  = GETDATE()
WHERE  IdRegion IS NULL;

-- Paso 1.3 (cuando se incorpore Perú — fuera de alcance R16):
-- DECLARE @IdPeru uniqueidentifier = '8278ecd0-c337-4484-b008-5b5e65b0dfaf';
-- INSERT INTO dbo.EmpresaDatosBancarios (..., IdRegion) VALUES (..., @IdPeru);
```

---

## 2. vEmpresaDatosBancarios (VISTA NUEVA R16)

**Propósito:** Vista operativa que expone el catálogo completo de cuentas bancarias
con todos los datos resueltos de Empresa, Región, Banco y Moneda.

> ⚠️ **PRERREQUISITO:** El Paso 1 (ALTER TABLE + IdRegion) debe ejecutarse **antes** de crear esta vista.

### Columnas de la vista

| Columna | Origen | Descripción |
|---------|--------|-------------|
| `IdEmpresaDatosBancarios` | EmpresaDatosBancarios | PK del registro |
| `IdEmpresa` | EmpresaDatosBancarios | FK — Empresa |
| `EmpresaPrefijo` | Empresa.Prefijo | GOL / MUN / PRO / PQF |
| `EmpresaAlias` | Empresa.Alias | Nombre corto de la empresa |
| `IdRegion` | EmpresaDatosBancarios | FK — Región |
| `Region` | Region.Nombre | México / Perú |
| `RegionClave` | Region.ClaveISO | MEX / PER |
| `IdDatosBancarios` | EmpresaDatosBancarios | FK — DatosBancarios |
| `IdCatBanco` | DatosBancarios | FK — catBanco |
| `Banco` | catBanco.Banco | Nombre de la institución bancaria |
| `ClaveBanco` | catBanco.Clave | Código del banco (ej. 002 = Banamex) |
| `NumeroDeCuenta` | DatosBancarios | Número de cuenta |
| `Clabe` | DatosBancarios | CLABE interbancaria |
| `Beneficiario` | DatosBancarios | Empresa beneficiaria de la cuenta |
| `Sucursal` | DatosBancarios | Sucursal bancaria |
| `IdCatMoneda` | DatosBancarios | FK — catMoneda |
| `ClaveMoneda` | catMoneda.ClaveMoneda | MXN / USD |
| `Moneda` | catMoneda.Moneda | Pesos / Dólares |
| `FechaRegistro` | EmpresaDatosBancarios | Fecha de alta |
| `FechaUltimaActualizacion` | EmpresaDatosBancarios | Fecha de modificación |
| `Activo` | EmpresaDatosBancarios | 1 = Vigente / 0 = No vigente |

### Script — Paso 2 (ejecutar DESPUÉS del Paso 1)

```sql
-- PRERREQUISITO: Paso 1 debe estar ejecutado (IdRegion ya debe existir en EmpresaDatosBancarios)
CREATE VIEW dbo.vEmpresaDatosBancarios
AS
SELECT
    edb.IdEmpresaDatosBancarios,
    edb.IdEmpresa,
    e.Prefijo        AS EmpresaPrefijo,
    e.Alias          AS EmpresaAlias,
    edb.IdRegion,
    r.Nombre         AS Region,
    r.ClaveISO       AS RegionClave,
    edb.IdDatosBancarios,
    db.IdCatBanco,
    b.Banco,
    b.Clave          AS ClaveBanco,
    db.NumeroDeCuenta,
    db.Clabe,
    db.Beneficiario,
    db.Sucursal,
    db.IdCatMoneda,
    m.ClaveMoneda,
    m.Moneda,
    edb.FechaRegistro,
    edb.FechaUltimaActualizacion,
    edb.Activo
FROM       dbo.EmpresaDatosBancarios edb
LEFT  JOIN dbo.Empresa               e   ON edb.IdEmpresa        = e.IdEmpresa
LEFT  JOIN dbo.Region                r   ON edb.IdRegion         = r.IdRegion
INNER JOIN dbo.DatosBancarios        db  ON edb.IdDatosBancarios = db.IdDatosBancarios
LEFT  JOIN dbo.catBanco              b   ON db.IdCatBanco        = b.IdCatBanco
LEFT  JOIN dbo.catMoneda             m   ON db.IdCatMoneda       = m.IdCatMoneda;
```

---

## 3. EmpresaRegion (Referencia — sin cambios en R16)

**Propósito:** Vincula cada empresa del grupo con la región en que opera.
**Creada:** 03/10/2025

| Columna | Tipo | Longitud | Nulo | Default | Descripción |
|---------|------|:--------:|:----:|---------|-------------|
| `IdEmpresaRegion` | uniqueidentifier | 16 | NO | `NEWID()` | PK |
| `IdEmpresa` | uniqueidentifier | 16 | NO | — | FK → Empresa |
| `IdRegion` | uniqueidentifier | 16 | NO | — | FK → Region (MEX / PER) |
| `FechaRegistro` | datetime | 8 | NO | `GETDATE()` | Fecha de alta |
| `FechaUltimaActualizacion` | datetime | 8 | NO | `GETDATE()` | Fecha de modificación |
| `Activo` | bit | 1 | NO | `1` | Relación activa |

---

## 4. DatosBancarios

**Propósito:** Detalle de cada cuenta bancaria. Sin cambios en R16.

| Columna | Tipo | Nulo | Descripción |
|---------|------|:----:|-------------|
| `IdDatosBancarios` | uniqueidentifier | NO | PK |
| `IdCatBanco` | uniqueidentifier | SÍ | FK → catBanco |
| `NumeroDeCuenta` | varchar(20) | SÍ | Número de cuenta |
| `Beneficiario` | varchar(200) | SÍ | Empresa beneficiaria |
| `Clabe` | varchar(200) | SÍ | CLABE interbancaria |
| `IdCatMoneda` | uniqueidentifier | SÍ | FK → catMoneda |
| `Sucursal` | varchar(50) | SÍ | Sucursal bancaria |
| `NumeroTarjeta` | varchar(20) | SÍ | Número de tarjeta |

---

## 5. Empresa

**Propósito:** Empresas del grupo PROQUIFA. Sin cambios en R16.

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `IdEmpresa` | uniqueidentifier | PK |
| `Prefijo` | varchar(50) | GOL / MUN / PRO / PQF |
| `Alias` | varchar(50) | Nombre corto |
| `RazonSocial` | varchar(50) | Nombre legal |
| `RFC` | varchar(13) | RFC México |
| `Activo` | bit | Empresa activa |

> ℹ️ `Empresa` no tiene `IdRegion` propio.
> La región se gestiona vía `EmpresaRegion` y se replica en `EmpresaDatosBancarios` por cuenta.

---

## 6. catBanco (Catálogo)

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `IdCatBanco` | uniqueidentifier | PK |
| `Banco` | varchar(180) | Nombre del banco |
| `Clave` | varchar(8) | Código bancario (ej. 002 = Banamex) |
| `Deposito` | bit | Permite depósitos |
| `Transferencia` | bit | Permite transferencias |
| `Activo` | bit | Banco activo |

---

## 7. catMoneda (Catálogo)

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `IdCatMoneda` | uniqueidentifier | PK |
| `ClaveMoneda` | varchar | MXN / USD / EUR |
| `Moneda` | varchar | Pesos / Dólares / Euros |
| `Activo` | bit | Moneda activa |

---

## 8. catMedioDePago (Catálogo)

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `IdCatMedioDePago` | uniqueidentifier | PK |
| `MedioDePago` | nvarchar(200) | Transferencia, Cheque, etc. |
| `ClaveFormaDePago` | varchar(2) | Clave SAT c_FormaPago |
| `RequiereNumeroDeCuenta` | bit | Requiere captura de cuenta |
| `Activo` | bit | Medio activo |

---

## 9. Tablas Consumidoras Directas

| Tabla | Columna FK | Descripción |
|-------|-----------|-------------|
| `fcppOrdenDePago` | `IdEmpresaDatosBancarios` | Cuenta destino de la orden de pago |
| `fppEjecucionOrdenDePago` | `IdEmpresaDatosBancarios` | Cuenta contra la que se ejecuta el pago |

---

## 10. Tablas Consumidoras Indirectas

| Tabla | Columna FK | Descripción |
|-------|-----------|-------------|
| `fcppSeguimientoFactura` | `IdCuentaDestino` | Cuenta destino del seguimiento |
| `fppSeguimientoPagoFactura` | `IdCuentaDestino` | Cuenta destino del pago |
| `fppEjecucionOrdenDePagoDestino` | `IdCuentaDestino` | Destino de la ejecución |
| `fcppSeguimientoFacturaIndirecto` | `IdCatBanco` | Datos bancarios inline |

---

## 11. Consultas SQL Principales

### Cuentas vigentes vía vista por Región (uso principal en módulos)

```sql
-- Uso principal en módulos: siempre consumir la vista, no la tabla directamente
DECLARE @RegionClave varchar(3) = 'MEX';  -- o 'PER'

SELECT
    IdEmpresaDatosBancarios,
    EmpresaPrefijo,
    EmpresaAlias,
    Banco,
    ClaveBanco,
    NumeroDeCuenta,
    Clabe,
    Sucursal,
    ClaveMoneda,
    Moneda
FROM  dbo.vEmpresaDatosBancarios
WHERE RegionClave = @RegionClave
  AND Activo      = 1
ORDER BY EmpresaPrefijo, NumeroDeCuenta;
```

### Órdenes de pago por cuenta del catálogo

```sql
SELECT
    op.IdFCPPOrdenDePago,
    v.EmpresaAlias,
    v.Banco,
    v.NumeroDeCuenta,
    v.Clabe,
    op.FechaPropuesta,
    op.Autorizada
FROM       dbo.fcppOrdenDePago          op
INNER JOIN dbo.vEmpresaDatosBancarios   v  ON op.IdEmpresaDatosBancarios = v.IdEmpresaDatosBancarios
WHERE op.Activo = 1
  AND v.Activo  = 1
ORDER BY op.FechaPropuesta DESC;
```

### Baja lógica de cuenta

```sql
DECLARE @IdCuenta UNIQUEIDENTIFIER;

UPDATE dbo.EmpresaDatosBancarios
SET    Activo                   = 0,
       FechaUltimaActualizacion = GETDATE()
WHERE  IdEmpresaDatosBancarios  = @IdCuenta;
```

---

## Reglas de Negocio

| Regla | Implementación | Objeto |
|-------|----------------|--------|
| Catálogo único | Tabla centralizada | `EmpresaDatosBancarios` |
| Estado vigente | `Activo = 1` | `EmpresaDatosBancarios.Activo` |
| Regionalización de cuentas | `IdRegion` FK | `EmpresaDatosBancarios.IdRegion` |
| Vista operativa única | JOIN completo resuelto | `vEmpresaDatosBancarios` |
| Sin UI en R16 | Gestión manual en BD | Soporte a la Producción |

---

## Mejores Prácticas

- **SIEMPRE** consumir `vEmpresaDatosBancarios` en módulos — nunca la tabla directamente
- **SIEMPRE** filtrar `Activo = 1`
- **SIEMPRE** filtrar `RegionClave` (MEX / PER) según el contexto operativo
- **SIEMPRE** validar `Prefijo IN ('GOL','MUN','PRO','PQF')` para alcance R16
- **NUNCA** hacer DELETE físico de registros
- **NUNCA** crear catálogos paralelos en otros módulos
- **NO** incluir GOLPERU en consultas de alcance R16

---

## Análisis de Gaps

| # | Gap | Descripción | Acción | Prioridad |
|:-:|-----|-------------|--------|:---------:|
| 1 | `IdRegion` ausente en `EmpresaDatosBancarios` | Columna nueva requerida | Ejecutar Script Paso 1 | 🔴 Alta |
| 2 | `vEmpresaDatosBancarios` no existe | Vista nueva requerida | Ejecutar Script Paso 2 tras Paso 1 | 🔴 Alta |
| 3 | `Empresa` sin `IdRegion` propio | Región vía `EmpresaRegion` | Sin cambio estructural — usar JOIN | ℹ️ Info |
| 4 | Cuentas Perú fuera de alcance R16 | GOLPERU no incluida | Deuda técnica — `IdRegion=PER` cuando aplique | 🟢 Baja |
| 5 | Sin UI en R16 | Gestión manual en BD | Validaciones en BD y auditoría | 🟡 Media |

---

## Riesgos

| # | Riesgo | Mitigación | Prioridad |
|:-:|--------|------------|:---------:|
| 1 | Vista creada antes del ALTER | Fallará con `Invalid column name IdRegion` — ejecutar Paso 1 primero | 🔴 Alta |
| 2 | Registros sin `IdRegion` tras ALTER | Ejecutar UPDATE del Paso 1.2 inmediatamente | 🔴 Alta |
| 3 | Modelo Perú no definido | Definir en release posterior con `IdRegion = PER` | 🟢 Baja |
| 4 | Sin UI — errores manuales | Validaciones en BD y auditoría | 🟡 Media |

---

**Generado por:** GitHub Copilot in SSMS
**Versión:** 1.4 — `IdRegion` en `EmpresaDatosBancarios` + Vista `vEmpresaDatosBancarios`
**Base de Datos:** ProquifaDotNet
