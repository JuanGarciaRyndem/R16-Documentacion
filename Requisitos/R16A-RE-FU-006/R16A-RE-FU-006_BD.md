# Diccionario de Datos — Referencia de Pago y Código Validador

| Campo | Valor |
|---|---|
| **Requisito** | R16A-RE-FU-006 |
| **Base de Datos** | ProquifaDotNet |
| **Servidor** | RYNL010 |
| **Versión** | 1.0 |
| **Generado por** | GitHub Copilot in SSMS |
| **Alcance** | Solo clientes México (MEX). Perú: pendiente definición. |

---

## Resumen Ejecutivo

Asignación de cuentas bancarias del grupo PROQUIFA a clientes y captura del Código Validador por combinación cliente-cuenta. La referencia bancaria se reconstruye dinámicamente al generar cada proforma según el banco asociado a la cuenta seleccionada (Banamex: 7 segmentos / Otros: nombre del cliente). Funcionalidad **NUEVA en PQF2 R16**. Solo aplica a clientes México.

> **⚠️ Hallazgo crítico** — La tabla de relación cliente-cuenta con Código Validador (**`ClienteDatosBancarios`**) **NO existe en la BD actual** y debe crearse como nuevo objeto en R16. El campo `tpProformaPedido.ReferenciaPago` (varchar 80) ya existe y es donde se escribe la referencia reconstruida dinámicamente al generar la proforma.

---

## Modelo de Datos

```
Cliente  (Nombre, Clave)
└── [NUEVO R16] ClienteDatosBancarios  (IdCliente + IdDatosBancarios + CodigoValidador)
        └── FK IdDatosBancarios
                DatosBancarios  (NumeroDeCuenta, Beneficiario, Sucursal, IdCatBanco, IdCatMoneda)
                ├── FK IdCatBanco  → catBanco   (Banco, Clave = código para referencia Banamex)
                └── FK IdCatMoneda → catMoneda  (ClaveMoneda = 'MXN'/'USD' para carácter P/D)

EmpresaDatosBancarios  (catálogo de cuentas del grupo PROQUIFA)
└── FK IdDatosBancarios → DatosBancarios

Consumidor:
tpProformaPedido.ReferenciaPago  ← referencia reconstruida dinámicamente (no persiste entre generaciones)
```

---

## Entidades Afectadas

| Objeto | Tipo | Estado | Descripción |
|---|---|---|---|
| `ClienteDatosBancarios` | Tabla | ✨ NUEVO R16 | Relación N:N Cliente-DatosBancarios con CódigoValidador |
| `DatosBancarios` | Tabla | ✅ Existente — sin cambios | Datos de cuenta bancaria (banco, cuenta, sucursal, moneda) |
| `catBanco` | Catálogo | ✅ Existente — sin cambios | Bancos con campo `Clave` usado en referencia Banamex |
| `EmpresaDatosBancarios` | Tabla | ✅ Existente — sin cambios | Catálogo de cuentas del grupo PROQUIFA |
| `catMoneda` | Catálogo | ✅ Existente — sin cambios | Moneda de la cuenta (MXN=P, otro=D en referencia Banamex) |
| `Cliente` | Tabla | ✅ Existente — sin cambios | Nombre y Clave usados en construcción de referencia |
| `tpProformaPedido` | Tabla | ✅ Existente — campo `ReferenciaPago` ya existe | Campo donde se escribe la referencia al generar proforma |

---

## 1. ClienteDatosBancarios (TABLA NUEVA — R16)

**Propósito:** Relación N:N entre `Cliente` y `DatosBancarios` del grupo PROQUIFA. Persiste el Código Validador por combinación cliente-cuenta. **La referencia bancaria armada NO se almacena aquí — se reconstruye dinámicamente.**

| Columna Propuesta          | Tipo             | Nulo | Descripción                                           |
| -------------------------- | ---------------- | ---- | ----------------------------------------------------- |
| `IdClienteDatosBancarios`  | uniqueidentifier | NO   | PK. Default: NEWID()                                  |
| `IdCliente`                | uniqueidentifier | NO   | FK — `Cliente`                                        |
| `IdDatosBancarios`         | uniqueidentifier | NO   | FK — `DatosBancarios` (cuenta del grupo PROQUIFA)     |
| `CodigoValidador`               | varchar(50)      | NO   | Código validador capturado manualmente por el usuario                 |
| `CodigoValidadorAnterior`       | varchar(50)      | SÍ   | Código validador previo antes de la última actualización (OBS-014)    |
| `FechaModificacionAnterior`     | datetime         | SÍ   | Fecha en que se realizó la modificación anterior del código (OBS-014) |
| `IdUsuarioModificacionAnterior` | uniqueidentifier | SÍ   | Usuario que realizó la modificación anterior del código (OBS-014)     |
| `FechaRegistro`                 | datetime         | NO   | Default: GETDATE()                                                    |
| `FechaUltimaActualizacion`      | datetime         | NO   | Default: GETDATE()                                                    |
| `Activo`                        | bit              | NO   | 1 = Asignación activa, 0 = Eliminada. Default: 1                      |

> **⚠️ Pendiente** — Confirmar longitud máxima de `CodigoValidador` con el cliente. Actualmente propuesto como `varchar(50)` — ajustar antes de ejecutar en producción.

**Script de creación:**

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
CREATE TABLE dbo.ClienteDatosBancarios (
    IdClienteDatosBancarios  uniqueidentifier NOT NULL
        CONSTRAINT PK_ClienteDatosBancarios  PRIMARY KEY CLUSTERED
        CONSTRAINT DF_ClienteDatosBancarios_Id DEFAULT (NEWID()),
    IdCliente                uniqueidentifier NOT NULL
        CONSTRAINT FK_ClienteDatosBancarios_Cliente
        FOREIGN KEY REFERENCES dbo.Cliente(IdCliente),
    IdDatosBancarios         uniqueidentifier NOT NULL
        CONSTRAINT FK_ClienteDatosBancarios_DatosBancarios
        FOREIGN KEY REFERENCES dbo.DatosBancarios(IdDatosBancarios),
    CodigoValidador              varchar(50)      NOT NULL,   -- Pendiente definir longitud maxima con cliente
    -- OBS-014: historial del código anterior para trazabilidad de rotación
    CodigoValidadorAnterior      varchar(50)      NULL,
    FechaModificacionAnterior    datetime         NULL,
    IdUsuarioModificacionAnterior uniqueidentifier NULL,
    FechaRegistro                datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaRegistro    DEFAULT (GETDATE()),
    FechaUltimaActualizacion datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaActualizacion DEFAULT (GETDATE()),
    Activo                   bit              NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_Activo            DEFAULT (1)
);

CREATE NONCLUSTERED INDEX IX_ClienteDatosBancarios
    ON dbo.ClienteDatosBancarios (IdCliente, IdDatosBancarios, Activo);
```

---

## 2. DatosBancarios (Existente — sin cambios)

**Propósito:** Detalle de cada cuenta bancaria del grupo PROQUIFA.
**Fuente de datos para la referencia:** Sucursal (autopoblado), `IdCatBanco`, `IdCatMoneda`.

| Columna            | Tipo             | Longitud | Nulo | Uso en Referencia                                       |
| ------------------ | ---------------- | -------- | ---- | ------------------------------------------------------- |
| `IdDatosBancarios` | uniqueidentifier | 16       | NO   | PK                                                      |
| `IdCatBanco`       | uniqueidentifier | 16       | SÍ   | FK — determina si es Banamex o no                       |
| `NumeroDeCuenta`   | varchar          | 20       | SÍ   | Número de cuenta bancaria                               |
| `Beneficiario`     | varchar          | 200      | SÍ   | Empresa beneficiaria (usada en lógica Banamex original) |
| `Sucursal`         | varchar          | 50       | SÍ   | Autopoblado en pantalla Referencia de Pago              |
| `Clabe`            | varchar          | 200      | SÍ   | CLABE interbancaria                                     |
| `IdCatMoneda`      | uniqueidentifier | 16       | SÍ   | Moneda — determina ‘P’ (MXN) o ‘D’ (otro) en Banamex    |
| `NumeroTarjeta`    | varchar          | 20       | SÍ   | Número de tarjeta                                       |

---

## 3. catBanco (Existente — sin cambios)

**Propósito:** Catálogo de instituciones bancarias.
**Campo crítico para la referencia:** `Clave` (código del banco, segmento 5 de la referencia Banamex).

| Columna         | Tipo             | Longitud | Nulo | Uso en Referencia                                   |
| --------------- | ---------------- | -------- | ---- | --------------------------------------------------- |
| `IdCatBanco`    | uniqueidentifier | 16       | NO   | PK                                                  |
| `Banco`         | varchar          | 180      | NO   | Nombre del banco                                    |
| `Clave`         | varchar          | 8        | SÍ   | Código del banco — segmento 5 en referencia Banamex |
| `ClaveBroker`   | varchar          | 8        | SÍ   | Clave broker                                        |
| `Deposito`      | bit              | 1        | NO   | Permite depósitos                                   |
| `Transferencia` | bit              | 1        | NO   | Permite transferencias                              |
| `Chequera`      | bit              | 1        | NO   | Permite chequera                                    |
| `Activo`        | bit              | 1        | NO   | Banco activo                                        |

**Banamex en BD:**

| Banco | Clave | ClaveBroker | Activo |
|---|---|---|---|
| Banamex | 002 | 40002 | ✅ Sí |

> La identificación de Banamex en la lógica de referencia se recomienda hacer por `IdCatBanco` o `Clave = '002'` en lugar del cruce con Beneficiario/Empresa, cuya condición de moneda aparece truncada en la documentación del cliente.

---

## 4. EmpresaDatosBancarios (Existente — sin cambios)

**Propósito:** Catálogo de cuentas bancarias del grupo PROQUIFA. **Fuente del selector de cuentas** en la pantalla Referencia de Pago.

| Columna                    | Tipo             | Nulo | Descripción                                 |
| -------------------------- | ---------------- | ---- | ------------------------------------------- |
| `IdEmpresaDatosBancarios`  | uniqueidentifier | NO   | PK                                          |
| `IdDatosBancarios`         | uniqueidentifier | NO   | FK — `DatosBancarios`                       |
| `IdEmpresa`                | uniqueidentifier | SÍ   | FK — Empresa del grupo PROQUIFA             |
| `FechaRegistro`            | datetime         | NO   | Default: GETDATE()                          |
| `FechaUltimaActualizacion` | datetime         | NO   | Default: GETDATE()                          |
| `Activo`                   | bit              | NO   | 1 = Cuenta vigente del catálogo. Default: 1 |

---

## 5. tpProformaPedido — Campo ReferenciaPago (Existente)

**Propósito:** Almacena la referencia bancaria reconstruida al generar la proforma.
**⚠️ Importante:** Este campo **NO persiste entre generaciones** — se sobreescribe en cada generación.

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `ReferenciaPago` | varchar | 80 | SÍ | Referencia bancaria reconstruida dinámicamente |

> La referencia reconstruida cabe en `varchar(80)` para ambos casos:
> - **No-Banamex:** nombre del cliente (longitud variable).
> - **Banamex:** 3 chars + 4 chars + código banco + 1 char + CódValidador.
>
> **⚠️ Pendiente** — Verificar que los nombres de clientes largos no excedan los 80 caracteres.

---

## Lógica de Reconstrucción de Referencia Bancaria

### Caso A — Banco distinto de Banamex (Regla 6)

```
Referencia = Cliente.Nombre
```

### Caso B — Banamex, 7 segmentos (Regla 7)

| # | Segmento | Fuente | Fallback |
|---|---|---|---|
| 1 | 1ª letra del nombre del cliente | `Cliente.Nombre[0]` | ‘X’ |
| 2 | 2ª letra del nombre del cliente | `Cliente.Nombre[1]` | ‘X’ |
| 3 | 3ª letra del nombre del cliente | `Cliente.Nombre[2]` | ‘X’ |
| 4 | Últimos 4 chars de la clave del cliente | `Cliente.Clave` (RIGHT 4, pad ‘000’) | ‘0000’ |
| 5 | Código del banco | `catBanco.Clave` | — |
| 6 | Carácter de moneda | ‘P’ si Moneda = MXN / ‘D’ en otro caso | — |
| 7 | Código Validador | `ClienteDatosBancarios.CodigoValidador` | — |

**Ejemplo Banamex:**

```
Cliente: 'QUIMICOS SA DE CV'  Clave: '12345'  Banco: '002'  Moneda: MXN  CodVal: 'ABC'
Referencia = 'Q' + 'U' + 'I' + '2345' + '002' + 'P' + 'ABC' = 'QUI2345002PABC'
```

### Identificación de Banamex (propuesta simplificada)

```sql
-- Identificar si la cuenta es Banamex por Clave del banco
SELECT CASE WHEN b.Clave = '002' THEN 'Banamex' ELSE 'OtroBanco' END AS TipoBanco
FROM dbo.DatosBancarios db
INNER JOIN dbo.catBanco b ON db.IdCatBanco = b.IdCatBanco
WHERE db.IdDatosBancarios = @IdDatosBancarios;
```

---

## Consultas SQL Principales

### Cuentas bancarias asignadas a un cliente con Código Validador

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
DECLARE @IdCliente UNIQUEIDENTIFIER;

SELECT
    cdb.IdClienteDatosBancarios,
    b.Banco,
    db.NumeroDeCuenta,
    db.Sucursal,
    m.ClaveMoneda,
    cdb.CodigoValidador
FROM dbo.ClienteDatosBancarios cdb
INNER JOIN dbo.DatosBancarios db ON cdb.IdDatosBancarios = db.IdDatosBancarios
INNER JOIN dbo.catBanco b         ON db.IdCatBanco = b.IdCatBanco
LEFT  JOIN dbo.catMoneda m        ON db.IdCatMoneda = m.IdCatMoneda
WHERE cdb.IdCliente = @IdCliente
  AND cdb.Activo    = 1
ORDER BY b.Banco;
```

### Selector de cuentas del grupo PROQUIFA (filtradas por banco)

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
DECLARE @IdCatBanco UNIQUEIDENTIFIER;

SELECT
    edb.IdEmpresaDatosBancarios,
    db.IdDatosBancarios,
    db.NumeroDeCuenta,
    db.Sucursal,
    db.Clabe,
    m.ClaveMoneda,
    e.Prefijo AS Empresa
FROM dbo.EmpresaDatosBancarios edb
INNER JOIN dbo.DatosBancarios db ON edb.IdDatosBancarios = db.IdDatosBancarios
INNER JOIN dbo.Empresa e          ON edb.IdEmpresa = e.IdEmpresa
LEFT  JOIN dbo.catMoneda m        ON db.IdCatMoneda = m.IdCatMoneda
WHERE db.IdCatBanco = @IdCatBanco
  AND edb.Activo    = 1
ORDER BY e.Prefijo, db.NumeroDeCuenta;
```

### Reconstrucción de referencia Banamex (T-SQL)

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
-- Solo para consulta/validación - la lógica vive en la capa de aplicación
DECLARE @IdCliente       UNIQUEIDENTIFIER;
DECLARE @IdDatosBanc     UNIQUEIDENTIFIER;

SELECT
    -- Segmentos 1-3: primeras 3 letras del nombre con fallback X
    ISNULL(NULLIF(SUBSTRING(c.Nombre, 1, 1), ''), 'X') +
    ISNULL(NULLIF(SUBSTRING(c.Nombre, 2, 1), ''), 'X') +
    ISNULL(NULLIF(SUBSTRING(c.Nombre, 3, 1), ''), 'X') +
    -- Segmento 4: últimos 4 chars de clave con padding de ceros
    RIGHT('0000' + c.Clave, 4) +
    -- Segmento 5: código del banco
    ISNULL(b.Clave, '') +
    -- Segmento 6: P si MXN, D en otro caso
    CASE WHEN m.ClaveMoneda = 'MXN' THEN 'P' ELSE 'D' END +
    -- Segmento 7: Código Validador
    ISNULL(cdb.CodigoValidador, '') AS ReferenciaBaseBanamex
FROM dbo.Cliente c
INNER JOIN dbo.ClienteDatosBancarios cdb ON c.IdCliente = cdb.IdCliente
INNER JOIN dbo.DatosBancarios db          ON cdb.IdDatosBancarios = db.IdDatosBancarios
INNER JOIN dbo.catBanco b                 ON db.IdCatBanco = b.IdCatBanco
LEFT  JOIN dbo.catMoneda m                ON db.IdCatMoneda = m.IdCatMoneda
WHERE c.IdCliente          = @IdCliente
  AND cdb.IdDatosBancarios = @IdDatosBanc
  AND cdb.Activo           = 1
  AND b.Clave              = '002';  -- Banamex
```

### Clientes México sin cuentas bancarias asignadas

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
SELECT c.IdCliente, c.Nombre, c.Clave
FROM dbo.Cliente c
INNER JOIN dbo.Region r ON c.IdRegion = r.IdRegion
LEFT  JOIN dbo.ClienteDatosBancarios cdb
       ON c.IdCliente = cdb.IdCliente AND cdb.Activo = 1
WHERE r.ClaveISO = 'MEX'
  AND c.Activo   = 1
  AND cdb.IdClienteDatosBancarios IS NULL
ORDER BY c.Nombre;
```

---

## Gaps y Acciones Pendientes

| # | Gap | Descripción | Acción | Prioridad |
|---|---|---|---|---|
| 1 | `ClienteDatosBancarios` no existe | Tabla de relación cliente-cuenta-CódigoValidador ausente en BD | Crear tabla (script Sección 1) | Alta |
| 2 | Longitud `CodigoValidador` indefinida | Cliente no especificó longitud máxima ni formato | Confirmar con cliente antes de crear tabla | Alta |
| 3 | Lógica identificación Banamex truncada | Condición de moneda en documentación del cliente incompleta | Usar `Clave = '002'` en `catBanco` como simplificación | Media |
| 4 | Campo `Clave` en `Cliente` no verificado | Se asume que `Cliente` tiene campo `Clave` para segmento 4 | Verificar existencia y tipo del campo en tabla `Cliente` | Alta |
| 5 | `ReferenciaPago` varchar(80) | Nombres de clientes largos podrían exceder 80 caracteres | Verificar longitud máx de nombres de clientes | Media |
| 6 | Sin restricción de rol | Asignación de cuentas sin validación de rol | Confirmar con cliente si debe restringirse a Coordinador Tesorería | Media |
| 7 | Modelo Perú no definido | Lógica bancaria Perú fuera de alcance R16 | Levantar como duda formal del proyecto | Baja |
| 8 | Sin validación `CodigoValidador` | Input manual sin reglas de formato | Confirmar con cliente si se requiere validación | Media |

---

## Reglas de Negocio

| Regla | Descripción | Implementación en BD |
|---|---|---|
| Regla 1 | Pantalla Referencia de Pago en Cobros | Sub-sección nueva en UI dentro de Cobros |
| Regla 2 | Una o más cuentas por cliente | `ClienteDatosBancarios`: N registros por `IdCliente` |
| Regla 3 | Código Validador por combinación cliente-cuenta | `ClienteDatosBancarios.CodigoValidador` |
| Regla 4 | Persistencia combinación cliente-cuenta-CodVal | INSERT/UPDATE en `ClienteDatosBancarios` |
| Regla 5 | Referencia se reconstruye dinámicamente | NO se almacena en BD — se calcula al generar proforma |
| Regla 6 | Referencia no-Banamex = nombre del cliente | `Cliente.Nombre` como cadena directa |
| Regla 7 | Referencia Banamex = 7 segmentos | Concatenación determinista al generar proforma |
| Regla 8 | Identificación Banamex | Por `catBanco.Clave = '002'` (propuesta simplificada) |
| Regla 9 | Edición sin restricción de rol | Sin control de rol en BD — pendiente confirmar |

---

## Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| 1 | Inconsistencia entre proformas re-emitidas | Documentar comportamiento — replica diseño Legacy |
| 2 | `CodigoValidador` sin validación de formato | Confirmar reglas con cliente antes de desarrollo |
| 3 | Sin restricción de rol sobre asignación de cuentas | Confirmar con cliente si aplica restricción |
| 4 | Modelo Perú no definido | Levantar como duda formal — clientes PE sin referencia |
| 5 | Pérdida de trazabilidad al sobrescribir `CodValidador` | Sin historial de versiones — comportamiento intencional R16 |

---

## Módulos Consumidores

| Módulo                 | Tabla              | Campo                | Descripción                                              |
| ---------------------- | ------------------ | -------------------- | -------------------------------------------------------- |
| Generación de Proforma | `tpProformaPedido` | `ReferenciaPago`     | Se escribe la referencia reconstruida al generar el PDF  |
| Buzón de Pagos         | `fccPagoCliente`   | `ReferenciaBancaria` | Identificación de pagos contra la referencia del cliente |
