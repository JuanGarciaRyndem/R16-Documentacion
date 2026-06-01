# Diccionario de Datos — Configuración de Cobros y Facturación del Cliente

| Campo | Valor |
|---|---|
| **Requisito** | TPSC-RE-FU-005 |
| **Base de Datos** | ProquifaDotNet |
| **Servidor** | RYNL010 |
| **Versión** | 1.2 — Estrategia IdRegion en catálogos existentes (confirmada con BD) |
| **Generado por** | GitHub Copilot in SSMS |
| **Última actualización** | Incluye IdRegion en catálogos, registros PE, banderas tributarias |

---

## Resumen Ejecutivo

Captura y mantenimiento de la configuración de cobros y facturación del cliente en la sección **Cobros** del Catálogo de Clientes. Los campos y catálogos varían por Región (México/Perú). Los valores son configuración default consumida por los módulos **Factura por Adelantado** y **Validar Cobro**.

**Estrategia de BD aprobada:** Agregar `IdRegion` (FK→Region) a los catálogos existentes en lugar de crear tablas nuevas. Los registros Perú se insertan en las mismas tablas.

---

## Estado por Región

| Aspecto | México (MEX) | Perú (PER) |
|---|---|---|
| Estado en R16 | Preexistente — se mantiene | **NUEVO en R16** |
| Cómo paga (medio) | `catMedioDePago` (IdRegion=MEX) | No aplica — SUNAT no lo exige |
| Cuándo paga (temporal) | `catMetodoDePagoCFDI` PUE/PPD (IdRegion=MEX) | `catMetodoDePagoCFDI` Contado/Crédito (IdRegion=PER) |
| Tipo de documento fiscal | `catUsoCFDI` G01/G03/S01 (IdRegion=MEX) | `catUsoCFDI` Factura/Boleta/Recibo (IdRegion=PER) |
| Bandera Agente de Retención IGV | No aplica | Campo nuevo en `DatosFacturacionCliente` — ⚠️ pendiente confirmar |
| Bandera Sujeto a Detracción | No aplica | Campo nuevo en `DatosFacturacionCliente` — ⚠️ pendiente confirmar |

---

## Modelo de Datos

```
Cliente  (IdRegion → Region)
└── FK IdConfiguracionPagos
        ConfiguracionPagos
        ├── FK IdCatMedioDePago       → catMedioDePago      (filtrar WHERE IdRegion = MEX)
        └── FK IdCatCondicionesDePago → catCondicionesDePago (plazos en días)
└── DatosFacturacionCliente
        ├── FK IdCatMetodoDePagoCFDI  → catMetodoDePagoCFDI (MX: PUE/PPD | PE: Contado/Crédito)
        ├── FK IdCatUsoCFDI           → catUsoCFDI          (MX: G01/G03   | PE: Factura/Boleta)
        ├── [NUEVO R16] AgenteRetencionIGV  bit          (PE — pendiente confirmar)
        ├── [NUEVO R16] SujetoDetraccion    bit          (PE — pendiente confirmar)
        └── [NUEVO R16] TasaDetraccion      decimal(5,2) (PE — solo cuando SujetoDetraccion = 1)

Catálogos con IdRegion agregado (ALTER TABLE pendiente ejecutar):
├── catMetodoDePagoCFDI.IdRegion → Region  [MEX: PUE/PPD existentes | PER: CONT/CRED nuevos]
├── catUsoCFDI.IdRegion          → Region  [MEX: G01/G03/S01 exist. | PER: 01/03/08 nuevos]
└── catMedioDePago.IdRegion      → Region  [solo MEX — sin registros PER]
```

---

## Entidades Afectadas

| Entidad | Tipo | Región | Cambio R16 | Observación |
|---|---|---|---|---|
| `catMedioDePago` | Catálogo | Solo MX | Existente + IdRegion | SUNAT no exige medio de pago |
| `catMetodoDePagoCFDI` | Catálogo | MX + PE | Existente + IdRegion + INSERTs PE | MX: PUE/PPD / PE: Contado/Crédito |
| `catUsoCFDI` | Catálogo | MX + PE | Existente + IdRegion + INSERTs PE | MX: G01/G03/S01 / PE: Factura/Boleta/Recibo |
| `DatosFacturacionCliente` | Tabla | MX + PE | Existente + 3 campos nuevos PE | AgenteRetencionIGV, SujetoDetraccion, TasaDetraccion |
| `vDatosFacturacionCliente` | Vista | MX | Existente — requiere revisión | Revisar si expone campos nuevos PE tras ALTER |

---

## 1. catMedioDePago (Forma de Pago — Solo México)

**Propósito:** Medio de pago. SUNAT no lo exige en el comprobante, por lo que **no aplica a Perú**.
**Cambio R16:** Agregar `IdRegion`. Todos los registros existentes → MEX. Sin registros PE.

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdCatMedioDePago` | uniqueidentifier | 16 | NO | PK |
| `MedioDePago` | nvarchar | 200 | NO | Descripción del medio |
| `ClaveFormaDePago` | varchar | 2 | SÍ | Clave SAT c_FormaPago (nullable) |
| `Clave` | varchar | 150 | NO | Clave interna del sistema |
| `RequiereNumeroDeCuenta` | bit | 1 | NO | Requiere captura de número de cuenta |
| `ObligatorioEnCliente` | bit | 1 | SÍ | Obligatorio en catálogo cliente |
| `Activo` | bit | 1 | NO | Default: 1 |
| `IdRegion` ✨ | uniqueidentifier | 16 | SÍ | **NUEVO R16** — FK Región (solo MEX) |

**Catálogo actual en BD (12 registros — todos serán asignados a MEX):**

| Descripción | Clave SAT | Activo | Observación |
|---|---|---|---|
| Aba | *(vacío)* | ✅ Sí | ⚠️ Sin clave SAT — pendiente mapeo |
| Cheque | 02 | ✅ Sí | ✅ OK |
| Depósito bancario | 31 | ✅ Sí | ✅ OK |
| Efectivo | 01 | ✅ Sí | ✅ OK |
| NA | *(vacío)* | ✅ Sí | ⚠️ Sin clave SAT — pendiente mapeo |
| —NINGUNO— | *(vacío)* | ✅ Sí | ⚠️ Sin clave SAT — pendiente mapeo |
| Otros | 99 | ✅ Sí | ✅ OK |
| Swift | *(vacío)* | ✅ Sí | ⚠️ Sin clave SAT — pendiente mapeo |
| Tarjeta | 04 | ✅ Sí | ✅ OK |
| Transferencia | 03 | ✅ Sí | ✅ OK |
| Transferencia Clabe | 03 | ❌ No (inactivo) | Inactivo |
| Transferencia Cuenta | 03 | ❌ No (inactivo) | Inactivo |

> **⚠️ Pendiente** — Aba, NA, NINGUNO y Swift no tienen `ClaveFormaDePago`. Verificar si el XML CFDI exige la clave SAT y si se requiere mapeo para timbrado. Ver `TPSC-RE-FU-005_Equivalencias_Cobros_MX_PE.xlsx`.

---

## 2. catMetodoDePagoCFDI (Método de Pago MX / Condición de Pago PE)

**Propósito:** Dimensión temporal del pago. Reutilizado para MX y PE con `IdRegion`.
**Cambio R16:** Agregar `IdRegion` + insertar 2 registros PE (Contado/Crédito SUNAT).

| Columna                 | Tipo             | Longitud | Nulo | Descripción                                     |
| ----------------------- | ---------------- | -------- | ---- | ----------------------------------------------- |
| `IdCatMetodoDePagoCFDI` | uniqueidentifier | 16       | NO   | PK                                              |
| `MetodoDePagoCFDI`      | nvarchar         | 100      | NO   | Descripción                                     |
| `ClaveMetodoDePagoCFDI` | nvarchar         | 6        | NO   | Clave SAT (MX: PUE/PPD) o SUNAT (PE: CONT/CRED) |
| `Clave`                 | varchar          | 150      | NO   | Clave interna                                   |
| `Activo`                | bit              | 1        | NO   | Default: 1                                      |
| `IdRegion` ✨            | uniqueidentifier | 16       | SÍ   | **NUEVO R16** — FK Región                       |

**Registros actuales y nuevos:**

| Clave | Descripción                      | Región | Estado      |
| ----- | -------------------------------- | ------ | ----------- |
| PPD   | Pago en parcialidades o diferido | MEX    | ✅ Existente |
| PUE   | Pago en una sola exhibición      | MEX    | ✅ Existente |
| CONT  | Contado (R.S. N° 193-2020/SUNAT) | PER    | ✨ NUEVO R16 |
| CRED  | Crédito (R.S. N° 193-2020/SUNAT) | PER    | ✨ NUEVO R16 |

**Equivalencia conceptual MX↔PE:**

| México (SAT) | Perú (SUNAT) | Dimensión |
|---|---|---|
| PUE — Pago en una exhibición | Contado | Pago inmediato |
| PPD — Pago diferido/parcialidades | Crédito | Pago diferido |

> **⚠️ Pendiente** — Confirmar denominación final del campo en pantalla para Perú.
> No renderizar campos MEX para clientes PER y viceversa.

---

## 3. catUsoCFDI (Uso de CFDI MX / Tipo de Comprobante PE)

**Propósito:** Tipo de documento fiscal. Reutilizado para MX y PE con `IdRegion`.
**Cambio R16:** Agregar `IdRegion` + insertar 3 registros PE (Factura/Boleta/Recibo SUNAT).
**⚠️ Importante:** Uso CFDI (MX) y Tipo Comprobante (PE) son conceptos **distintos** — no mezclar.

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdCatUsoCFDI` | uniqueidentifier | 16 | NO | PK |
| `ClaveUso` | nvarchar | 6 | NO | Clave SAT c_UsoCFDI (MX) o código SUNAT (PE) |
| `Uso` | nvarchar | 300 | NO | Descripción |
| `Clave` | varchar | 150 | NO | Clave interna |
| `Activo` | bit | 1 | NO | Default: 1 |
| `IdRegion` ✨ | uniqueidentifier | 16 | SÍ | **NUEVO R16** — FK Región |

**Registros actuales y nuevos:**

| Clave | Descripción                               | Región | Estado      |
| ----- | ----------------------------------------- | ------ | ----------- |
| G01   | Adquisición de mercancías                 | MEX    | ✅ Existente |
| G02   | Devoluciones, descuentos o bonificaciones | MEX    | ✅ Existente |
| G03   | Gastos en general                         | MEX    | ✅ Existente |
| N/A   | N/A (valor interno)                       | MEX    | ✅ Existente |
| P01   | Por definir                               | MEX    | ✅ Existente |
| S01   | Sin efectos fiscales                      | MEX    | ✅ Existente |
| 01    | Factura electrónica                       | PER    | ✨ NUEVO R16 |
| 03    | Boleta de venta electrónica               | PER    | ✨ NUEVO R16 |
| 08    | Recibo por Honorarios electrónico         | PER    | ✨ NUEVO R16 |

---

## 4. ConfiguracionPagos (sin cambios en R16)

**Propósito:** Configuración de cobros default del cliente.
**Vínculo:** `Cliente.IdConfiguracionPagos` → `ConfiguracionPagos`

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `IdConfiguracionPagos` | uniqueidentifier | NO | PK |
| `IdCatCondicionesDePago` | uniqueidentifier | SÍ | FK — plazos de crédito en días |
| `IdCatMedioDePago` | uniqueidentifier | SÍ | FK — `catMedioDePago` (Forma de Pago MX) |
| `LineaCredito` | decimal | SÍ | Monto de línea de crédito |
| `LimiteLineaCredito` | decimal | SÍ | Límite de línea de crédito |
| `PorcentajeSobregiroLineaCredito` | decimal | SÍ | Porcentaje de sobregiro |
| `NumeroDeCuenta` | varchar(20) | SÍ | Número de cuenta asociada |
| `MontoDeCredito` | decimal | SÍ | Monto de crédito autorizado |
| `FechaRegistro` | datetime | NO | Default: GETDATE() |
| `FechaUltimaActualizacion` | datetime | NO | Default: GETDATE() |
| `Activo` | bit | NO | Default: 1 |

---

## 5. catCondicionesDePago (sin cambios en R16)

**Propósito:** Plazos de crédito en días.

> **⚠️ Nota crítica:** Este catálogo define plazos de crédito en días. **NO es la Condición de Pago SUNAT** (Contado/Crédito) — ese concepto va en `catMetodoDePagoCFDI` con `IdRegion = PER`.

| Descripción | Sin Crédito | Días | Activo |
|---|---|---|---|
| 8 DÍAS | No | 8 | ✅ Sí |
| 15 DÍAS | No | 15 | ✅ Sí |
| 21 DÍAS | No | 21 | ✅ Sí |
| 30 DÍAS | No | 30 | ✅ Sí |
| 45 DÍAS | No | 45 | ✅ Sí |
| 60 DÍAS | No | 60 | ✅ Sí |
| 90 DÍAS | No | 90 | ✅ Sí |
| PAGO CONTRA ENTREGA | Sí | 0 | ✅ Sí |
| PREPAGO 100% | Sí | 0 | ✅ Sí |
| ANTICIPO 50% | No | 0 | ❌ No (inactivo) |

---

## 6. DatosFacturacionCliente (campos de Cobros — con cambios R16)

**Propósito:** Almacena Uso CFDI, Método de Pago (MX) y nuevas banderas tributarias (PE).
**Cambio R16:** Agregar 3 columnas para Perú (tras confirmar aplicabilidad con cliente).

| Columna | Tipo | Nulo | Estado | Descripción |
|---|---|---|---|---|
| `IdCatUsoCFDI` | uniqueidentifier | SÍ | ✅ Existente | FK — `catUsoCFDI` (MX: G01/G03 / PE: Factura/Boleta) |
| `IdCatMetodoDePagoCFDI` | uniqueidentifier | SÍ | ✅ Existente | FK — `catMetodoDePagoCFDI` (MX: PUE/PPD / PE: Cont/Cred) |
| `AgenteRetencionIGV` ✨ | bit | SÍ | ✨ NUEVO R16 | Bandera PE: Agente de Retención IGV SUNAT (default 0 = No) |
| `SujetoDetraccion` ✨ | bit | SÍ | ✨ NUEVO R16 | Bandera PE: Sujeto a Detracción SPOT SUNAT (default 0 = No) |
| `TasaDetraccion` ✨ | decimal(5,2) | SÍ | ✨ NUEVO R16 | Tasa % de detracción. Solo cuando `SujetoDetraccion = 1` |

> `IdCatUsoCFDI` e `IdCatMetodoDePagoCFDI` son NULLABLE en BD pero **obligatorios en UI** por Región.
> **⚠️ Pendiente** — `AgenteRetencionIGV` y `SujetoDetraccion`: NO agregar hasta confirmar aplicabilidad con el cliente.

---

## Scripts de Cambios Estructurales R16

### Paso 1 — Agregar IdRegion a catálogos (ejecutar primero)

```sql
DECLARE @IdMexico uniqueidentifier = '60390fda-7773-4ba1-8120-cb874f3a3a53'; -- MEX

ALTER TABLE dbo.catMetodoDePagoCFDI
    ADD IdRegion uniqueidentifier NULL
        CONSTRAINT FK_catMetodoDePagoCFDI_Region
        FOREIGN KEY REFERENCES dbo.Region(IdRegion);

ALTER TABLE dbo.catUsoCFDI
    ADD IdRegion uniqueidentifier NULL
        CONSTRAINT FK_catUsoCFDI_Region
        FOREIGN KEY REFERENCES dbo.Region(IdRegion);

ALTER TABLE dbo.catMedioDePago
    ADD IdRegion uniqueidentifier NULL
        CONSTRAINT FK_catMedioDePago_Region
        FOREIGN KEY REFERENCES dbo.Region(IdRegion);

-- Asignar México a todos los registros existentes
UPDATE dbo.catMetodoDePagoCFDI SET IdRegion = @IdMexico WHERE IdRegion IS NULL;
UPDATE dbo.catUsoCFDI          SET IdRegion = @IdMexico WHERE IdRegion IS NULL;
UPDATE dbo.catMedioDePago      SET IdRegion = @IdMexico WHERE IdRegion IS NULL;
```

### Paso 2 — Insertar registros Perú

```sql
DECLARE @IdPeru uniqueidentifier = '8278ecd0-c337-4484-b008-5b5e65b0dfaf'; -- PER

-- Condición de Pago SUNAT (R.S. N° 193-2020/SUNAT)
INSERT INTO dbo.catMetodoDePagoCFDI (MetodoDePagoCFDI, ClaveMetodoDePagoCFDI, Activo, Clave, IdRegion)
VALUES
    ('Contado', 'CONT', 1, 'contado', @IdPeru),
    ('Credito', 'CRED', 1, 'credito', @IdPeru);

-- Tipo de Comprobante SUNAT
INSERT INTO dbo.catUsoCFDI (Uso, ClaveUso, Activo, Clave, IdRegion)
VALUES
    ('Factura electrónica',               '01', 1, 'facturasunat', @IdPeru),
    ('Boleta de venta electrónica',        '03', 1, 'boletasunat',  @IdPeru),
    ('Recibo por Honorarios electrónico',  '08', 1, 'recibosunat',  @IdPeru);
```

### Paso 3 — Agregar banderas tributarias Perú a DatosFacturacionCliente

> **⚠️ EJECUTAR SOLO DESPUÉS DE CONFIRMAR APLICABILIDAD CON EL CLIENTE**

```sql
ALTER TABLE dbo.DatosFacturacionCliente
    ADD AgenteRetencionIGV bit          NULL,
        SujetoDetraccion   bit          NULL,
        TasaDetraccion     decimal(5,2) NULL;
```

---

## Mapeo de Campos por Región

| Concepto | Tabla | Campo MX | Campo PE | Estrategia |
|---|---|---|---|---|
| Cómo paga (medio) | `ConfiguracionPagos` | `IdCatMedioDePago` (MEX) | No aplica | Sin registros PE en `catMedioDePago` |
| Cuándo paga (temporal) | `DatosFacturacionCliente` | `IdCatMetodoDePagoCFDI` PUE/PPD | `IdCatMetodoDePagoCFDI` CONT/CRED | Misma FK filtrada por `IdRegion` |
| Tipo de documento | `DatosFacturacionCliente` | `IdCatUsoCFDI` G01/G03/S01 | `IdCatUsoCFDI` 01/03/08 | Misma FK filtrada por `IdRegion` |
| Agente Retención IGV | `DatosFacturacionCliente` | No aplica | `AgenteRetencionIGV` (bit) | Campo nuevo — ⚠️ pendiente confirmar |
| Sujeto a Detracción | `DatosFacturacionCliente` | No aplica | `SujetoDetraccion` (bit) + `TasaDetraccion` | Campo nuevo — ⚠️ pendiente confirmar |

---

## Consultas SQL Principales

### Configuración de cobros — cliente México

```sql
DECLARE @IdCliente UNIQUEIDENTIFIER;

SELECT
    md.MedioDePago           AS FormaDePago,
    md.ClaveFormaDePago      AS ClaveFormaPagoSAT,
    mp.ClaveMetodoDePagoCFDI AS MetodoPago,
    mp.MetodoDePagoCFDI,
    uc.ClaveUso,
    uc.Uso                   AS UsoCFDI,
    cp.CondicionesDePago,
    cp.Dias                  AS DiasCredito
FROM dbo.Cliente c
INNER JOIN dbo.Region r                  ON c.IdRegion = r.IdRegion
LEFT  JOIN dbo.ConfiguracionPagos cfg    ON c.IdConfiguracionPagos = cfg.IdConfiguracionPagos
LEFT  JOIN dbo.catMedioDePago md         ON cfg.IdCatMedioDePago = md.IdCatMedioDePago
LEFT  JOIN dbo.catCondicionesDePago cp   ON cfg.IdCatCondicionesDePago = cp.IdCatCondicionesDePago
LEFT  JOIN dbo.DatosFacturacionCliente dfc ON c.IdCliente = dfc.IdCliente AND dfc.Activo = 1
LEFT  JOIN dbo.catMetodoDePagoCFDI mp   ON dfc.IdCatMetodoDePagoCFDI = mp.IdCatMetodoDePagoCFDI
LEFT  JOIN dbo.catUsoCFDI uc             ON dfc.IdCatUsoCFDI = uc.IdCatUsoCFDI
WHERE c.IdCliente = @IdCliente
  AND r.ClaveISO  = 'MEX';
```

### Configuración de cobros — cliente Perú

```sql
DECLARE @IdCliente UNIQUEIDENTIFIER;

SELECT
    mp.ClaveMetodoDePagoCFDI AS CondicionPago,
    mp.MetodoDePagoCFDI      AS DescripcionCondicion,
    uc.ClaveUso              AS CodigoTipoComprobante,
    uc.Uso                   AS TipoComprobante,
    dfc.AgenteRetencionIGV,
    dfc.SujetoDetraccion,
    dfc.TasaDetraccion
FROM dbo.Cliente c
INNER JOIN dbo.Region r                    ON c.IdRegion = r.IdRegion
LEFT  JOIN dbo.DatosFacturacionCliente dfc ON c.IdCliente = dfc.IdCliente AND dfc.Activo = 1
LEFT  JOIN dbo.catMetodoDePagoCFDI mp      ON dfc.IdCatMetodoDePagoCFDI = mp.IdCatMetodoDePagoCFDI
LEFT  JOIN dbo.catUsoCFDI uc               ON dfc.IdCatUsoCFDI = uc.IdCatUsoCFDI
WHERE c.IdCliente = @IdCliente
  AND r.ClaveISO  = 'PER';
```

### Selector de catálogo por Región del cliente

```sql
-- Método de Pago (MX) o Condición de Pago (PE) según la región del cliente
DECLARE @IdCliente UNIQUEIDENTIFIER;

SELECT mp.IdCatMetodoDePagoCFDI, mp.ClaveMetodoDePagoCFDI, mp.MetodoDePagoCFDI
FROM dbo.catMetodoDePagoCFDI mp
INNER JOIN dbo.Cliente c ON mp.IdRegion = c.IdRegion
WHERE c.IdCliente = @IdCliente
  AND mp.Activo   = 1
ORDER BY mp.MetodoDePagoCFDI;

-- Uso de CFDI (MX) o Tipo de Comprobante (PE) según la región del cliente
SELECT uc.IdCatUsoCFDI, uc.ClaveUso, uc.Uso
FROM dbo.catUsoCFDI uc
INNER JOIN dbo.Cliente c ON uc.IdRegion = c.IdRegion
WHERE c.IdCliente = @IdCliente
  AND uc.Activo   = 1
ORDER BY uc.ClaveUso;
```

### Clientes con cobros incompletos por Región

```sql
SELECT
    c.IdCliente,
    c.Nombre,
    r.ClaveISO AS Region,
    CASE WHEN r.ClaveISO = 'MEX' AND cfg.IdCatMedioDePago IS NULL
         THEN 'Sin Forma de Pago' ELSE 'OK' END AS FormaDePago,
    CASE WHEN dfc.IdCatMetodoDePagoCFDI IS NULL
         THEN 'Sin Método/Condición Pago' ELSE 'OK' END AS MetodoPago,
    CASE WHEN dfc.IdCatUsoCFDI IS NULL
         THEN 'Sin UsoCFDI/TipoComprobante' ELSE 'OK' END AS UsoCFDI
FROM dbo.Cliente c
INNER JOIN dbo.Region r                    ON c.IdRegion = r.IdRegion
LEFT  JOIN dbo.ConfiguracionPagos cfg      ON c.IdConfiguracionPagos = cfg.IdConfiguracionPagos
LEFT  JOIN dbo.DatosFacturacionCliente dfc ON c.IdCliente = dfc.IdCliente AND dfc.Activo = 1
WHERE (cfg.IdCatMedioDePago IS NULL AND r.ClaveISO = 'MEX')
   OR dfc.IdCatMetodoDePagoCFDI IS NULL
   OR dfc.IdCatUsoCFDI IS NULL;
```

---

## Módulos Consumidores

| Módulo | Campos Consumidos MX | Campos Consumidos PE |
|---|---|---|
| Factura por Adelantado | `IdCatMedioDePago`, `IdCatMetodoDePagoCFDI` (PUE/PPD), `IdCatUsoCFDI` (G01/G03) | `IdCatMetodoDePagoCFDI` (CONT/CRED), `IdCatUsoCFDI` (01/03/08), `AgenteRetencionIGV`, `SujetoDetraccion` |
| Validar Cobro | `IdCatMedioDePago`, `IdCatMetodoDePagoCFDI`, `IdCatUsoCFDI` | `IdCatMetodoDePagoCFDI`, `IdCatUsoCFDI`, `AgenteRetencionIGV`, `SujetoDetraccion` |

---

## Gaps y Acciones Pendientes

| # | Gap | Descripción | Acción | Prioridad |
|---|---|---|---|---|
| 1 | IdRegion ausente en catálogos | `catMetodoDePagoCFDI`, `catUsoCFDI`, `catMedioDePago` sin `IdRegion` | Ejecutar Paso 1 | Alta |
| 2 | Registros PE ausentes | `catMetodoDePagoCFDI` sin CONT/CRED; `catUsoCFDI` sin 01/03/08 | Ejecutar Paso 2 | Alta |
| 3 | Campos PE en DatosFacturacionCliente | `AgenteRetencionIGV`, `SujetoDetraccion`, `TasaDetraccion` no existen | Ejecutar Paso 3 tras confirmar | Media |
| 4 | Clave SAT incompleta en catMedioDePago | Aba, NA, NINGUNO, Swift sin `ClaveFormaDePago` | Confirmar si afecta timbrado | Media |
| 5 | Aplicabilidad banderas PE sin confirmar | `AgenteRetencionIGV` y `SujetoDetraccion` pendientes | Confirmar con cliente | Media |
| 6 | `vDatosFacturacionCliente` | Vista puede requerir ajuste para campos PE nuevos | Revisar tras ALTER TABLE Paso 3 | Baja |

---

## Brechas Reconocidas — Facturación Electrónica Perú (Fuera de Alcance R16)

| # | Brecha | Descripción |
|---|---|---|
| 1 | Datos SUNAT en catálogo de productos | Código SUNAT, unidad de medida SUNAT, tipo de afectación IGV por línea — no existen |
| 2 | Guía de Remisión Electrónica (GRE) | Requerida por SUNAT para despacho físico — PROQUIFA despacha mercancía |
| 3 | Tipo de Operación SUNAT (Catálogo 51) | Campo obligatorio en XML UBL 2.1 por factura |
| 4 | Agente de Percepción IGV del emisor | Condición de PROQUIFA Perú — pendiente confirmar si aplica |
| 5 | Infraestructura emisión electrónica | Certificado digital, OSE/SEE-SOL, CDR, Resúmenes Diarios, Comunicaciones de Baja |

---

## Reglas de Negocio

| Regla | Descripción | Implementación en BD |
|---|---|---|
| Regla 1 | Campos como configuración default | `DatosFacturacionCliente` y `ConfiguracionPagos` |
| Regla 2 | Catálogos diferenciados por Región | `IdRegion` en `catMetodoDePagoCFDI` y `catUsoCFDI` |
| Regla 3 | Condición de Pago PE = dimensión temporal | `catMetodoDePagoCFDI` IdRegion=PER (CONT/CRED) |
| Regla 4 | Método de Pago solo México | `catMetodoDePagoCFDI` IdRegion=MEX (PUE/PPD) |
| Regla 5 | Forma de Pago solo México | `catMedioDePago` sin registros PE |
| Regla 6 | UsoCFDI y TipoComprobante son distintos | Misma tabla `catUsoCFDI` diferenciada por `IdRegion` |
| Regla 7 | Agente de Retención IGV | `DatosFacturacionCliente.AgenteRetencionIGV` (bit) — pendiente |
| Regla 8 | Sujeto a Detracción | `DatosFacturacionCliente.SujetoDetraccion` + `TasaDetraccion` — pendiente |
| Regla 9 | Edición sin restricción de rol | Sin control de rol en BD — acceso por cartera |

---

## Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| 1 | Confusión nomenclatura MX/PE | Documentar equivalencias — ver archivo Equivalencias |
| 2 | Catálogos desactualizados SAT/SUNAT | Mantenimiento periódico de registros en BD |
| 3 | Brechas facturación electrónica Perú | Gestionar como pendientes formales del proyecto |
| 4 | Retenciones/Detracciones mal calculadas | Confirmar aplicabilidad con cliente antes de implementar |
| 5 | Productos sujetos a Detracción no identificados | Requiere campo Detracción en catálogo de productos |
