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

Asignación de cuentas bancarias del grupo PROQUIFA a clientes y captura del Código Validador por combinación cliente-cuenta. La referencia bancaria se **arma al configurar/actualizar la cuenta del cliente** y se persiste como **referencia vigente del cliente** en `ClienteDatosBancarios.ReferenciaVigente`; al generar una proforma, esa referencia se **casa al PDF** (snapshot inmutable en `tpProformaPedido.ReferenciaPago`) y las proformas históricas conservan su referencia. Solo se regenera la referencia vigente si cambia un dato fuente (banco, cuenta, Código Validador, `Cliente.Nombre` o `Cliente.Clave`). Adicionalmente, cada cambio del Código Validador se registra en el Aplicativo Nuevo ProquifaDotNet.BitacoraCambios (historial completo: valor anterior/nuevo, autor y fecha — actualización 2026-07-07, sustituye la rotación de un nivel de OBS-014). Funcionalidad **NUEVA en ProquifaDotNet R16**. Solo aplica a clientes México.

> **⚠️ Hallazgo crítico** — La tabla de relación cliente-cuenta con Código Validador (**`ClienteDatosBancarios`**) **NO existe en la BD actual** y debe crearse como nuevo objeto en R16. Debe incluir el campo **`ReferenciaVigente`** para almacenar la referencia armada del cliente. El campo `tpProformaPedido.ReferenciaPago` (varchar 80) ya existe y recibe el snapshot inmutable de la referencia vigente al generar el PDF de la proforma en firme.

---

## Modelo de Datos

```
Cliente  (Nombre, Clave)
└── [NUEVO R16] ClienteDatosBancarios  (IdCliente + IdDatosBancarios + CodigoValidador + ReferenciaVigente; historial en BitacoraCambios)
        └── FK IdDatosBancarios
                DatosBancarios  (NumeroDeCuenta, Beneficiario, Sucursal, IdCatBanco, IdCatMoneda)
                ├── FK IdCatBanco  → catBanco   (Banco, Clave = código para referencia Banamex)
                └── FK IdCatMoneda → catMoneda  (ClaveMoneda = 'MXN'/'USD' para carácter P/D)

EmpresaDatosBancarios  (catálogo de cuentas del grupo PROQUIFA)
└── FK IdDatosBancarios → DatosBancarios

Consumidor:
tpProformaPedido.ReferenciaPago  ← snapshot inmutable copiado de ClienteDatosBancarios.ReferenciaVigente al generar el PDF
```

---

## Entidades Afectadas

| Objeto | Tipo | Estado | Descripción |
|---|---|---|---|
| `ClienteDatosBancarios` | Tabla | ✨ NUEVO R16 | Relación N:N Cliente-DatosBancarios con Código Validador y referencia vigente; historial del código en ProquifaDotNet.BitacoraCambios |
| `DatosBancarios` | Tabla | ✅ Existente — sin cambios | Datos de cuenta bancaria (banco, cuenta, sucursal, moneda) |
| `catBanco` | Catálogo | ✅ Existente — sin cambios | Bancos con campo `Clave` usado en referencia Banamex |
| `EmpresaDatosBancarios` | Tabla | ✅ Existente — sin cambios | Catálogo de cuentas del grupo PROQUIFA (fuente del selector) |
| `catMoneda` | Catálogo | ✅ Existente — sin cambios | Moneda de la cuenta (MXN=P, otro=D en referencia Banamex) |
| `Cliente` | Tabla | ⚠️ Verificar campo `Clave` | Nombre y Clave usados en construcción de referencia. Pendiente confirmar si `Clave` existe; si no, hay que agregarlo o resolver alternativa. |
| `tpProformaPedido` | Tabla | ✅ Existente — campo `ReferenciaPago` ya existe | Campo donde se persiste el **snapshot** de la referencia al generar el PDF |

---

## 1. ClienteDatosBancarios (TABLA NUEVA — R16)

**Propósito:** Relación N:N entre `Cliente` y `DatosBancarios` del grupo PROQUIFA. Persiste el Código Validador por combinación cliente-cuenta y **la referencia bancaria vigente armada** (Regla 4, nivel 1). El historial del Código Validador NO lleva columnas propias: cada cambio se registra en ProquifaDotNet.BitacoraCambios (tabla afectada, campo, valor anterior/nuevo, usuario, fecha — actualización 2026-07-07, sustituye a OBS-014).

| Columna Propuesta               | Tipo             | Nulo | Descripción                                                                                              |
| ------------------------------- | ---------------- | ---- | -------------------------------------------------------------------------------------------------------- |
| `IdClienteDatosBancarios`       | uniqueidentifier | NO   | PK. Default: NEWID()                                                                                     |
| `IdCliente`                     | uniqueidentifier | NO   | FK — `Cliente`                                                                                           |
| `IdDatosBancarios`              | uniqueidentifier | NO   | FK — `DatosBancarios` (cuenta del grupo PROQUIFA)                                                        |
| `CodigoValidador`               | varchar(50)      | NO   | Código validador capturado manualmente por el usuario                                                    |
| `ReferenciaVigente`             | varchar(80)      | SÍ   | **Referencia bancaria armada vigente del cliente** (Regla 4 nivel 1, OBS-013). Se regenera solo ante cambio de dato fuente. |
| `FechaReferenciaVigente`        | datetime         | SÍ   | Fecha y hora en que se generó/actualizó la referencia vigente. Útil para auditoría y troubleshooting.    |
| `FechaRegistro`                 | datetime         | NO   | Default: GETDATE()                                                                                       |
| `FechaUltimaActualizacion`      | datetime         | NO   | Default: GETDATE()                                                                                       |
| `Activo`                        | bit              | NO   | 1 = Asignación activa, 0 = Eliminada. Default: 1                                                         |

> **⚠️ Pendiente** — Confirmar longitud máxima de `CodigoValidador` con el cliente. Actualmente propuesto como `varchar(50)` — ajustar antes de ejecutar en producción.

> **Nota — Disparadores de regeneración de `ReferenciaVigente`:** además del CREATE/UPDATE en `ClienteDatosBancarios` (cambio de banco, cuenta o Código Validador), debe regenerarse cuando cambia `Cliente.Nombre` o `Cliente.Clave` (segmentos S1-S4 de la fórmula Banamex). La estrategia de actualización en cascada (hook en `ClienteBO.Actualizar` o job batch) queda como decisión de diseño técnico — ver `R16A-RE-FU-006-Back.md`.

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
    -- OBS-013: referencia vigente del cliente (Regla 4 nivel 1)
    ReferenciaVigente            varchar(80)      NULL,
    FechaReferenciaVigente       datetime         NULL,
    FechaRegistro                datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaRegistro    DEFAULT (GETDATE()),
    FechaUltimaActualizacion datetime         NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_FechaActualizacion DEFAULT (GETDATE()),
    Activo                   bit              NOT NULL
        CONSTRAINT DF_ClienteDatosBancarios_Activo            DEFAULT (1)
);

-- Indice filtrado: permite reasignar una cuenta tras baja logica (Activo = 0)
CREATE UNIQUE NONCLUSTERED INDEX UX_ClienteDatosBancarios_ClienteCuentaActiva
    ON dbo.ClienteDatosBancarios (IdCliente, IdDatosBancarios)
    WHERE Activo = 1;

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

**Propósito:** Almacena el **snapshot inmutable** de la referencia bancaria casada al PDF de la proforma. Al generar el PDF, el valor de `ClienteDatosBancarios.ReferenciaVigente` se copia a este campo y queda fijo aunque cambien los datos del cliente posteriormente (OBS-013, OBS-015).

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `ReferenciaPago` | varchar | 80 | SÍ | Snapshot inmutable de la referencia vigente al momento de generar el PDF |

> La referencia armada cabe en `varchar(80)` para ambos casos:
> - **No-Banamex:** nombre del cliente (longitud variable).
> - **Banamex:** 3 chars + 4 chars + código banco + 1 char + CódValidador.
>
> **⚠️ Pendiente** — Verificar que los nombres de clientes largos no excedan los 80 caracteres.

---

## Lógica de Armado de la Referencia Bancaria

> Esta lógica se ejecuta **una sola vez al configurar/actualizar la cuenta del cliente** (o cuando cambia un dato fuente: `Cliente.Nombre`, `Cliente.Clave`, banco, cuenta o `CodigoValidador`). El resultado se persiste en `ClienteDatosBancarios.ReferenciaVigente`. Al generar el PDF de una proforma, se copia ese valor a `tpProformaPedido.ReferenciaPago` (snapshot inmutable).


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
| 1 | `ClienteDatosBancarios` no existe | Tabla de relación cliente-cuenta con `ReferenciaVigente` ausente en BD | Crear tabla con todos los campos (script Sección 1); el historial del Código Validador se registra en ProquifaDotNet.BitacoraCambios, no en columnas propias | Alta |
| 2 | Longitud `CodigoValidador` indefinida | Cliente no especificó longitud máxima ni formato | Confirmar con cliente antes de crear tabla | Alta |
| 3 | Lógica identificación Banamex truncada | Condición de moneda en documentación del cliente incompleta | Usar `Clave = '002'` en `catBanco` como simplificación | Media |
| 4 | Campo `Clave` en `Cliente` no verificado | Segmento S4 depende de `Cliente.Clave` que podría no existir en BD | Verificar existencia/tipo en `Cliente`; si no existe, agregar columna permanente o definir fuente alternativa (no apoyarse en tabla ETL de migración a largo plazo) | Alta |
| 5 | `ReferenciaPago` varchar(80) | Nombres de clientes largos podrían exceder 80 caracteres | Verificar longitud máx de nombres de clientes | Media |
| 6 | Sin restricción de rol | Asignación de cuentas sin validación de rol | Confirmar con cliente si debe restringirse a Coordinador Tesorería | Media |
| 7 | Modelo Perú no definido | Lógica bancaria Perú fuera de alcance R16 | Levantar como duda formal del proyecto | Baja |
| 8 | Sin validación `CodigoValidador` | Input manual sin reglas de formato | Confirmar con cliente si se requiere validación | Media |
| 9 | Trigger de regeneración de `ReferenciaVigente` por cambio en datos del cliente | Si cambia `Cliente.Nombre` o `Cliente.Clave`, los segmentos S1-S4 quedan obsoletos en `ReferenciaVigente` de todas las asignaciones activas del cliente | Definir mecanismo (hook en `ClienteBO.Actualizar`, job batch o trigger BD) y documentarlo en el diseño técnico | Alta |
| 10 | Selector de cuentas en pantalla | El selector debe traer las cuentas del **grupo PROQUIFA** (vía `EmpresaDatosBancarios` cruzado con `DatosBancarios`), no todas las cuentas activas | Endpoint debe consultar `EmpresaDatosBancarios` con `Activo = 1` filtrando por banco | Alta |

---

## Reglas de Negocio

| Regla | Descripción | Implementación en BD |
|---|---|---|
| Regla 1 | Pantalla Referencia de Pago en Cobros | Sub-sección nueva en UI dentro de Cobros |
| Regla 2 | Una o más cuentas por cliente | `ClienteDatosBancarios`: N registros por `IdCliente` |
| Regla 3 | Código Validador por combinación cliente-cuenta | `ClienteDatosBancarios.CodigoValidador` + historial de un nivel (OBS-014) |
| Regla 4 — nivel 1 | Referencia vigente del cliente | `ClienteDatosBancarios.ReferenciaVigente` — se arma al CREATE/UPDATE de la asignación o al cambiar dato fuente; persiste hasta nueva regeneración |
| Regla 4 — nivel 2 | Referencia casada a la proforma | `tpProformaPedido.ReferenciaPago` — copia inmutable de `ReferenciaVigente` al generar PDF |
| Regla 5 | Generación al configurar cuenta + casado al PDF | Cálculo en BO (`ReferenciaBancariaBO.Construir`) → persistido en `ReferenciaVigente`; al generar PDF, copia a `ReferenciaPago` |
| Regla 6 | Referencia no-Banamex = nombre del cliente | `Cliente.Nombre` como cadena directa |
| Regla 7 | Referencia Banamex = 7 segmentos | Concatenación determinista al armar la `ReferenciaVigente` |
| Regla 8 | Identificación Banamex | Por `catBanco.Clave = '002'` (propuesta simplificada) |
| Regla 9 | Edición sin restricción de rol | Sin control de rol en BD — pendiente confirmar |

---

## Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| 1 | `CodigoValidador` sin validación de formato | Confirmar reglas con cliente antes de desarrollo |
| 2 | Sin restricción de rol sobre asignación de cuentas | Confirmar con cliente si aplica restricción |
| 3 | Modelo Perú no definido | Levantar como duda formal — clientes PE sin referencia |
| ~~Antiguo R1~~ | ~~Inconsistencia entre proformas re-emitidas~~ | **Retirado** — OBS-013 introduce persistencia en dos niveles y snapshot inmutable en PDF |
| ~~Antiguo R5~~ | ~~Pérdida de trazabilidad al sobrescribir `CodValidador`~~ | **Retirado** — OBS-014 introduce historial de un nivel (valor actual + anterior con autor y fecha) |

---

## Módulos Consumidores

| Módulo                       | Tabla                    | Campo                | Descripción                                                                                                                  |
| ---------------------------- | ------------------------ | -------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Catálogo de Clientes (Cobros)| `ClienteDatosBancarios`  | `ReferenciaVigente`  | Se arma al CREATE/UPDATE de la asignación y se regenera ante cambio en `Cliente.Nombre`, `Cliente.Clave`, banco, cuenta o CV |
| Generación de Proforma       | `tpProformaPedido`       | `ReferenciaPago`     | Snapshot inmutable copiado de `ClienteDatosBancarios.ReferenciaVigente` al generar el PDF en firme                           |
| Buzón de Cobros              | `fccPagoCliente`         | `ReferenciaBancaria` | Identificación de pagos contra la referencia del cliente                                                                     |
