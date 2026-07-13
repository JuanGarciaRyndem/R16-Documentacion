# Impacto en BD — Complemento de Pago México (CFDI tipo P / Pagos 2.0)
**Requisito:** R16A-RE-FU-030
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (lectura/escritura) + DocumentBuilder (escritura)
**Versión:** 1.0

---

## Resumen

RE-030 habilita la generación del Complemento de Pago (CFDI tipo P, Pagos20 v2.0) para
México como complemento de la infraestructura del Paso 3 de Validar Cobro (RE-028).

La estructura de control de estado del Paso 3 (`fccDocumentoFiscalCobro`,
`catTipoCFDI.COMPLEMENTO_PAGO`, `fccDocumentoFiscalCobro.IdCFDIGeneradaComplemento`,
`CFDIGenerada.IdCFDIRelacionado`) fue creada en RE-028 y se **reutiliza**.

RE-030 agrega únicamente lo necesario para materializar los datos fiscales específicos del
Complemento de Pago: el catálogo `catFormaPagoSAT` (c_FormaPago SAT, **creado en RE-FU-019**
como prerrequisito de `CFDIGenerada.IdCatFormaPagoSAT`), la clave `CP01` en `catUsoCFDI` (confirmada inexistente), las columnas del
DoctoRelacionado (NumParcialidad, saldos, equivalencias) en `fccDocumentoFiscalCobro`,
las entradas del foliador con Serie "P" en `EmpresaFolio` y los templates PDF en
`DocumentTemplate` (tabla existente desde requisito anterior).

> **Política de 1 CP por factura:**
> En R16 cada Complemento de Pago tiene exactamente un Pago y un DoctoRelacionado.
> Los campos de saldo (`ImpSaldoAnt`, `ImpPagado`, `ImpSaldoInsoluto`) se almacenan
> en `fccDocumentoFiscalCobro` como snapshot fiscal en el momento del timbrado.
> No se recomputarán post-timbrado; son inmutables una vez generado el CFDI.

> ⚠️ **Pendientes que afectan BD:**
> — Serie "P" en `EmpresaFolio`: formato del folio pendiente de validar con PMO (Regla 12).
> — `FechaPago`: convención de hora 12:00:00 fija vs. hora real pendiente de confirmar
>   con asesor fiscal (Riesgo 2 del requisito).
> — Soporte de tasas de IVA distintas a 16% (frontera 8%, 0%) pendiente de confirmar.

---

## Impacto en BD

| #   | Cambio                                                                                                            | Base de Datos             | Tipo      | Prioridad |
| --- | ----------------------------------------------------------------------------------------------------------------- | ------------------------- | --------- | --------- |
| 1   | ~~CREATE TABLE catFormaPagoSAT~~ — **creada en RE-FU-019** (prerrequisito disponible)             | ProquifaDotNet            | Existente | —         |
| 2   | DML catUsoCFDI — INSERT clave CP01 (Pagos) — confirmado no existe                                                 | ProquifaDotNet            | DML       | Alta      |
| 3   | ALTER TABLE fccDocumentoFiscalCobro — ADD columnas DR del Complemento de Pago (8 cols)                            | ProquifaDotNet            | DDL       | Alta      |
| 4   | DML EmpresaFolio — INSERT filas Serie "P" para GOL, MUN, PRO, PQF                                                 | ProquifaDotNet (Finanzas) | DML       | Alta      |
| 5   | DML DocumentTemplate — INSERT 4 templates PDF Complemento de Pago México                                          | DocumentBuilder           | DML       | Media     |
| 6   | ALTER VIEW vfccDocumentoFiscalCobro v3.0 — exponer columnas DR nuevas                                             | ProquifaDotNet            | DDL       | Media     |
| —   | Reutiliza: CFDIGenerada con `IdCatTipoCFDI='COMPLEMENTO_PAGO'` (RE-028)                                           | ProquifaDotNet            | Existente | —         |
| —   | Reutiliza: `catTipoCFDI.COMPLEMENTO_PAGO` (clave creada en RE-028 T1)                                             | ProquifaDotNet            | Existente | —         |
| —   | Reutiliza: `fccDocumentoFiscalCobro.IdCFDIGeneradaComplemento` (RE-028 T3)                                        | ProquifaDotNet            | Existente | —         |
| —   | Reutiliza: `CFDIGenerada.IdCFDIRelacionado` — link CP → Factura PPD (RE-028 T5)                                   | ProquifaDotNet            | Existente | —         |
| —   | Reutiliza: `EmpresaFolio` — estructura existente (RE-019); solo se insertan filas Serie P                         | ProquifaDotNet (Finanzas) | Existente | —         |
| —   | Reutiliza: PAC TurboPac — misma integración que Factura México (RE-019)                                           | ProquifaDotNet            | Existente | —         |
| —   | Reutiliza: `catUsoCFDI` — tabla existente; CP01 se inserta en cambio #2                                           | ProquifaDotNet            | Existente | —         |
| —   | Reutiliza: `catMetodoDePagoCFDI.PPD` — solo facturas PPD generan CP (RE-028 T1)                                   | ProquifaDotNet            | Existente | —         |
| —   | Reutiliza: `CorreoEnviado`, `ArchivoCorreoEnviado` — envío de correo al cliente (RE-028)                          | ProquifaDotNet            | Existente | —         |
| —   | Reutiliza: `fccDocumentoFiscalCobro` base completa (RE-028 T3, RE-029 T3)                                         | ProquifaDotNet            | Existente | —         |
| —   | Reutiliza: `DocumentTemplate` — tabla existente desde requisito anterior                                          | DocumentBuilder           | Existente | —         |
| —   | Reutiliza: bucket `cobranza` en `RegionConfiguracionMinioBucket` (IdRegion=MEX) — almacenamiento PDF y XML del CP | ProquifaDotNet            | Existente | —         |

---

## Catálogo: catFormaPagoSAT

**Confirmado:** no existe en ProquifaDotNet ningún catálogo equivalente. Ejecutar el CREATE TABLE sin condición.

El Complemento de Pago usa `FormaDePagoP` = forma real del cobro (Regla 7 del requisito).
A diferencia de `catMetodoDePagoCFDI` (PUE/PPD), este catálogo describe cómo entró el
dinero físicamente.

```sql
-- Ejecutar en ProquifaDotNet
CREATE TABLE [dbo].[catFormaPagoSAT](
    [IdCatFormaPagoSAT]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catFormaPagoSAT_Id]      DEFAULT (NEWID()),
    [Clave]              varchar(10)      NOT NULL,
        -- Código SAT c_FormaPago: '01', '02', '03', '04', '05', '06', '08', '12', etc.
    [Descripcion]        nvarchar(150)    NOT NULL,
    [Activo]             bit              NOT NULL
        CONSTRAINT [DF_catFormaPagoSAT_Activo]  DEFAULT (1),
    [FechaRegistro]      datetime2(7)     NOT NULL
        CONSTRAINT [DF_catFormaPagoSAT_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catFormaPagoSAT]
        PRIMARY KEY CLUSTERED ([IdCatFormaPagoSAT]),
    CONSTRAINT [UQ_catFormaPagoSAT_Clave]
        UNIQUE ([Clave])
);
GO

-- Datos iniciales (subconjunto frecuente; completar con catálogo SAT vigente)
INSERT INTO dbo.catFormaPagoSAT (Clave, Descripcion) VALUES
    ('01', 'Efectivo'),
    ('02', 'Cheque nominativo'),
    ('03', 'Transferencia electrónica de fondos'),
    ('04', 'Tarjeta de crédito'),
    ('05', 'Monedero electrónico'),
    ('06', 'Dinero electrónico'),
    ('08', 'Vales de despensa'),
    ('12', 'Dación en pago'),
    ('13', 'Pago por subrogación'),
    ('14', 'Pago por consignación'),
    ('15', 'Condonación'),
    ('17', 'Compensación'),
    ('23', 'Novación'),
    ('24', 'Confusión'),
    ('25', 'Remisión de deuda'),
    ('26', 'Prescripción o caducidad'),
    ('27', 'A satisfacción del acreedor'),
    ('28', 'Tarjeta de débito'),
    ('29', 'Tarjeta de servicios'),
    ('30', 'Aplicación de anticipos'),
    ('31', 'Intermediario pagos'),
    ('99', 'Por definir');
GO

-- Validación
SELECT COUNT(*) AS Registros FROM dbo.catFormaPagoSAT;
```

### Diccionario de datos — catFormaPagoSAT

| Nombre de tabla | Descripción |
|---|---|
| catFormaPagoSAT | Catálogo SAT c_FormaPago. Formas reales de pago usadas en `FormaDePagoP` del nodo Pago del Complemento de Pago (CFDI tipo P) y en `FormaPago` de facturas PUE. |

**Columnas:**

| Nombre | Tipo | Descripción |
|---|---|---|
| IdCatFormaPagoSAT | uniqueidentifier PK | Identificador único |
| Clave | varchar(10) NOT NULL UNIQUE | Código SAT c_FormaPago: `'01'`, `'03'`, `'04'`, etc. |
| Descripcion | nvarchar(150) NOT NULL | Descripción legible |
| Activo | bit NOT NULL | 1 = vigente en catálogo SAT |
| FechaRegistro | datetime2(7) NOT NULL | Fecha de inserción |

**Índices:**

| Nombre | Columnas | Tipo |
|---|---|---|
| PK_catFormaPagoSAT | IdCatFormaPagoSAT | PRIMARY KEY CLUSTERED |
| UQ_catFormaPagoSAT_Clave | Clave | UNIQUE non-clustered |

---

## DML catUsoCFDI — INSERT clave CP01 (Pagos)

**Confirmado:** la tabla `catUsoCFDI` existe pero **no contiene la clave CP01**. Claves actuales: `P01`, `G03`, `S01`, `G02`, `G01`, `N/A`. El Complemento de Pago requiere `UsoCFDI=CP01` fijo en el nodo Receptor (Regla 6 / Criterio C3 del requisito).

Los nombres de columna se infieren del resultado de la consulta (`Uso`, `ClaveUso`, `Clave`, `Activo`). Verificar que existe también la columna PK `IdCatUsoCFDI` antes de ejecutar.

```sql
-- Ejecutar en ProquifaDotNet
-- ⚠️ Verificar que no existe CP01 antes de ejecutar:
-- SELECT * FROM dbo.catUsoCFDI WHERE ClaveUso = 'CP01'

INSERT INTO dbo.catUsoCFDI (ClaveUso, Uso, Clave, Activo)
VALUES ('CP01', 'CP01 Pagos', 'CP01', 1);
GO

-- Validación
SELECT ClaveUso, Uso, Clave, Activo FROM dbo.catUsoCFDI WHERE ClaveUso = 'CP01';
```

---

## ALTER TABLE fccDocumentoFiscalCobro — Columnas DR del Complemento de Pago

La tabla `fccDocumentoFiscalCobro` (creada en RE-028, extendida en RE-029) registra una
línea por documento fiscal del Paso 3. Para el Complemento de Pago (CFDI tipo P) se
necesitan persistir los valores fiscales del nodo `DoctoRelacionado` y del nodo `Pago`
como **snapshot inmutable** al momento del timbrado.

Dado que la política de R16 es 1 Complemento = 1 DoctoRelacionado, estos campos se
agregan directamente a `fccDocumentoFiscalCobro` (relación 1:1). Son NULL para líneas
México tipo `FACTURA` / `FACTURA_ANTICIPO` y para todas las líneas Perú (que ya tienen
sus propias columnas de RE-029).

> **Columnas del snapshot fiscal del CP:**

| Campo XML          | Columna BD          | Descripción                                                                                  |
| ------------------ | ------------------- | -------------------------------------------------------------------------------------------- |
| `FechaPago`        | `FechaPagoCP`       | Fecha/hora del cobro para el nodo Pago. ⚠️ Pendiente confirmar hora (12:00:00 fija vs real). |
| `FormaDePagoP`     | `IdCatFormaPagoSAT` | FK catFormaPagoSAT. Forma real del cobro (típicamente 03).                                   |
| `TipoCambioP`      | `TipoCambioP_CP`    | TC del pago vs MXN. NULL cuando MonedaP = MXN.                                               |
| `NumParcialidad`   | `NumParcialidad`    | Número consecutivo de pagos aplicados a la factura PPD (1, 2, 3...).                         |
| `ImpSaldoAnt`      | `ImpSaldoAnt`       | Saldo de la factura PPD antes de este pago. En MonedaDR.                                     |
| `ImpPagado`        | `ImpPagado`         | Porción del cobro aplicada a esta factura. En MonedaDR.                                      |
| `ImpSaldoInsoluto` | `ImpSaldoInsoluto`  | Saldo restante después de este pago (ImpSaldoAnt − ImpPagado). En MonedaDR.                  |
| `EquivalenciaDR`   | `EquivalenciaDR`    | Factor de conversión cuando MonedaDR ≠ MonedaP. 1 si las monedas coinciden.                  |

```sql
-- Prerequisito: catFormaPagoSAT debe existir (o el catálogo equivalente ya existente)
-- Prerequisito: fccDocumentoFiscalCobro (RE-028 T3) y columnas Perú (RE-029 T3) deben existir
-- Ejecutar en ProquifaDotNet

-- 1. Fecha y hora del pago (snapshot para nodo Pago del XML)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [FechaPagoCP] datetime2(7) NULL;
        -- FechaPago del nodo Pago en el XML del CP.
        -- NULL para líneas no-CP (FACTURA, FACTURA_ANTICIPO, FACTURA_CPE).
        -- ⚠️ Pendiente: confirmar si hora es 12:00:00 fija o la hora real del cobro.
GO

-- 2. Forma de pago real (FormaDePagoP del nodo Pago)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [IdCatFormaPagoSAT] uniqueidentifier NULL;
        -- FK catFormaPagoSAT. Típicamente '03' Transferencia.
        -- NULL para líneas no-CP.
GO

ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD CONSTRAINT [FK_fccDocumentoFiscalCobro_FormaPagoSAT]
        FOREIGN KEY ([IdCatFormaPagoSAT])
        REFERENCES dbo.catFormaPagoSAT([IdCatFormaPagoSAT]);
GO

-- 3. Tipo de cambio del pago vs MXN (TipoCambioP del nodo Pago)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [TipoCambioP_CP] decimal(18,6) NULL;
        -- NULL cuando MonedaP = MXN. Presente cuando el cobro es en divisa extranjera.
GO

-- 4. Número de parcialidad (NumParcialidad del DoctoRelacionado)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [NumParcialidad] int NULL;
        -- Consecutivo de pagos aplicados a la factura PPD referenciada.
        -- 1 en el primer CP, 2 en el segundo, etc.
        -- NULL para líneas no-CP.
GO

-- 5. Saldo anterior (ImpSaldoAnt del DoctoRelacionado — en MonedaDR)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [ImpSaldoAnt] decimal(18,6) NULL;
        -- Saldo de la factura PPD antes de este cobro. En MonedaDR.
        -- Para el primer CP: igual al Total de la factura PPD.
        -- Para CPs subsecuentes: ImpSaldoInsoluto del CP anterior.
        -- NULL para líneas no-CP.
GO

-- 6. Importe pagado (ImpPagado del DoctoRelacionado — en MonedaDR)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [ImpPagado] decimal(18,6) NULL;
        -- Porción del cobro aplicada a esta factura. En MonedaDR.
        -- = Monto del nodo Pago en este Complemento de Pago.
        -- NULL para líneas no-CP.
GO

-- 7. Saldo insoluto (ImpSaldoInsoluto del DoctoRelacionado — en MonedaDR)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [ImpSaldoInsoluto] decimal(18,6) NULL;
        -- Saldo restante = ImpSaldoAnt - ImpPagado. En MonedaDR.
        -- El sistema lo calcula y persiste al timbrar; no se recomputa post-timbrado.
        -- NULL para líneas no-CP.
GO

-- 8. Equivalencia DR (EquivalenciaDR del DoctoRelacionado)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [EquivalenciaDR] decimal(18,6) NULL;
        -- Factor de conversión cuando MonedaDR ≠ MonedaP.
        -- Valor 1 cuando las monedas coinciden (obligatorio en el XML aunque sean iguales).
        -- NULL para líneas no-CP.
GO

-- Validación: confirmar columnas nuevas
SELECT c.name AS Columna, t.name AS Tipo, c.is_nullable AS EsNullable
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.fccDocumentoFiscalCobro')
  AND c.name IN (
      'FechaPagoCP','IdCatFormaPagoSAT','TipoCambioP_CP',
      'NumParcialidad','ImpSaldoAnt','ImpPagado','ImpSaldoInsoluto','EquivalenciaDR'
  )
ORDER BY c.column_id;
```

### Diccionario de datos — Columnas nuevas en fccDocumentoFiscalCobro

| Nombre | Tipo | Descripción |
|---|---|---|
| FechaPagoCP | datetime2(7) NULL | Fecha/hora del cobro para el nodo Pago del XML del CP. Snapshot inmutable al timbrar. ⚠️ Hora fija 12:00:00 vs real pendiente confirmar. NULL para líneas no-CP. |
| IdCatFormaPagoSAT | uniqueidentifier FK NULL | FK a `catFormaPagoSAT`. Forma real del cobro (`FormaDePagoP`). Típicamente clave '03' Transferencia. NULL para líneas no-CP. |
| TipoCambioP_CP | decimal(18,6) NULL | Tipo de cambio del pago vs MXN cuando MonedaP ≠ MXN. NULL si el cobro es en MXN. NULL para líneas no-CP. |
| NumParcialidad | int NULL | Número de parcialidad del DoctoRelacionado (1 = primer CP para esta factura, 2 = segundo, etc.). NULL para líneas no-CP. |
| ImpSaldoAnt | decimal(18,6) NULL | Importe Saldo Anterior de la factura PPD en MonedaDR antes de este cobro. NULL para líneas no-CP. |
| ImpPagado | decimal(18,6) NULL | Importe pagado en este CP en MonedaDR (= Monto del nodo Pago). NULL para líneas no-CP. |
| ImpSaldoInsoluto | decimal(18,6) NULL | Importe Saldo Insoluto = ImpSaldoAnt − ImpPagado en MonedaDR. Inmutable tras timbrado. NULL para líneas no-CP. |
| EquivalenciaDR | decimal(18,6) NULL | EquivalenciaDR del DoctoRelacionado. 1 cuando MonedaDR = MonedaP. Factor de conversión cuando difieren. NULL para líneas no-CP. |

**Relaciones nuevas:**

| Tabla relacionada | Tipo | FK |
|---|---|---|
| catFormaPagoSAT | N:1 (nullable) | IdCatFormaPagoSAT |

**Consideraciones especiales:**
- Las 8 columnas son NULL para toda línea no-CP (`FACTURA`, `FACTURA_ANTICIPO`, líneas Perú). La validación de presencia obligatoria para líneas CP se realiza en la capa de aplicación (Finanzas), no con CHECK CONSTRAINT en BD.
- `ImpSaldoInsoluto` se calcula en la capa de aplicación antes del INSERT y se persiste como snapshot. No hay trigger ni computed column para evitar recomputación posterior que podría diferir si cambian datos relacionados.
- `NumParcialidad` se calcula contando los `CFDIGenerada` existentes con `IdCatTipoCFDI = 'COMPLEMENTO_PAGO'` y `IdCFDIRelacionado = @IdFacturaPPD` + 1, en la misma transacción del timbrado con UPDLOCK para evitar concurrencia.
- `EquivalenciaDR = 1` cuando MonedaDR = MonedaP — el valor 1 debe persistirse explícitamente ya que el SAT lo requiere en el XML aunque las monedas sean iguales.

---

## DML EmpresaFolio — Entradas Serie "P" para Complemento de Pago México

El Complemento de Pago utiliza Serie "P" por empresa emisora del grupo PROQUIFA México
(Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica). Las 4 filas se insertan en
`EmpresaFolio` (estructura creada en RE-019) en `ProquifaDotNet` (propiedad Finanzas).

> ⚠️ **Brecha:** El esquema definitivo del foliador con serie "P" (formato, longitud máxima,
> prefijo) está pendiente de validar con PMO (Regla 12 del requisito). Los valores del
> INSERT son propuesta inicial sujeta a cambio. El `FormatoFolio` debe confirmarse con el
> mismo patrón que las facturas de cada empresa (`{serie}{folio:00000000}` o variante).

```sql
-- Prerequisito: EmpresaFolio y las 4 empresas México deben existir
-- Ejecutar en ProquifaDotNetTimbrado
-- ⚠️ Confirmar IdEmpresa de cada empresa antes de ejecutar;
--    ajustar FormatoFolio y LongitudMaxima según validación con PMO.

-- Golocaer México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa,
       'P',
       0,
       'P{folio:00000000}',  -- ⚠️ Formato pendiente confirmar con PMO
       8,
       1
FROM dbo.Empresa e
WHERE e.Prefijo = 'GOL'; -- Prefijo identifica unívocamente la empresa México (sin columna Region)

-- Mungen México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa,
       'P',
       0,
       'P{folio:00000000}',  -- ⚠️ Formato pendiente confirmar con PMO
       8,
       1
FROM dbo.Empresa e
WHERE e.Prefijo = 'MUN'; -- Prefijo identifica unívocamente la empresa México (sin columna Region)

-- Proquifa México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa,
       'P',
       0,
       'P{folio:00000000}',  -- ⚠️ Formato pendiente confirmar con PMO
       8,
       1
FROM dbo.Empresa e
WHERE e.Prefijo = 'PRO'; -- Prefijo identifica unívocamente la empresa México (sin columna Region)

-- Proveedora Quimico Farmaceutica México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa,
       'P',
       0,
       'P{folio:00000000}',  -- ⚠️ Formato pendiente confirmar con PMO
       8,
       1
FROM dbo.Empresa e
WHERE e.Prefijo = 'PQF'; -- Prefijo identifica unívocamente la empresa México (sin columna Region)

GO

-- Validación: 4 filas Serie P, UltimoFolio=0
SELECT e.Prefijo, ef.Serie, ef.UltimoFolio, ef.FormatoFolio, ef.LongitudMaxima
FROM dbo.EmpresaFolio ef
INNER JOIN dbo.Empresa e ON ef.IdEmpresa = e.IdEmpresa
WHERE ef.Serie = 'P'
ORDER BY e.Prefijo;
```

---

## DML DocumentTemplate — Templates PDF Complemento de Pago México

Se registran 4 templates en la base de datos `DocumentBuilder` (tabla `dbo.DocumentTemplate`),
uno por empresa emisora México, identificados por `TemplateKey` con la nomenclatura
`{PREFIJO}_MEX_CP`. El archivo de plantilla HTML (Body) se crea en la tarea de DocumentBuilder
del requisito.

> **Convención de claves:** `GOL_MEX_CP`, `MUN_MEX_CP`, `PRO_MEX_CP`, `PQF_MEX_CP`.
> Sufijo `_CP` = Complemento de Pago. No confundir con `_CDP` (Confirmación de Pedido).

> **Base de datos:** `DocumentBuilder` — NO `ProquifaDotNet`. La tabla `DocumentTemplate`
> vive en la BD del servicio de generación de PDFs.

La tabla no tiene `IdEmpresa` ni `TipoDocumento`; la identificación es únicamente por
`TemplateKey`. La convención de nombres de archivo es `{TemplateKey}_{H/B/F}.html`
(confirmada con datos existentes: `GOL_MEX_PED_H.html`, `GOL_MEX_PED_B.html`,
`GOL_MEX_PED_F.html`). Los tres archivos siempre se usan (`HasHeader/Body/Footer = 1`).

```sql
-- Ejecutar en DocumentBuilder

-- Golocaer México
INSERT INTO dbo.DocumentTemplate (
    TemplateKey,
    HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName,
    HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate,
    RegistrationDate, LastUpdateDate, IsActive
)
VALUES (
    'GOL_MEX_CP',
    'GOL_MEX_CP_H.html', 'GOL_MEX_CP_B.html', 'GOL_MEX_CP_F.html',
    1, 1, 1,
    GETDATE(), GETDATE(), 1
);

-- Mungen México
INSERT INTO dbo.DocumentTemplate (
    TemplateKey,
    HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName,
    HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate,
    RegistrationDate, LastUpdateDate, IsActive
)
VALUES (
    'MUN_MEX_CP',
    'MUN_MEX_CP_H.html', 'MUN_MEX_CP_B.html', 'MUN_MEX_CP_F.html',
    1, 1, 1,
    GETDATE(), GETDATE(), 1
);

-- Proquifa México
INSERT INTO dbo.DocumentTemplate (
    TemplateKey,
    HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName,
    HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate,
    RegistrationDate, LastUpdateDate, IsActive
)
VALUES (
    'PRO_MEX_CP',
    'PRO_MEX_CP_H.html', 'PRO_MEX_CP_B.html', 'PRO_MEX_CP_F.html',
    1, 1, 1,
    GETDATE(), GETDATE(), 1
);

-- Proveedora Quimico Farmaceutica México
INSERT INTO dbo.DocumentTemplate (
    TemplateKey,
    HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName,
    HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate,
    RegistrationDate, LastUpdateDate, IsActive
)
VALUES (
    'PQF_MEX_CP',
    'PQF_MEX_CP_H.html', 'PQF_MEX_CP_B.html', 'PQF_MEX_CP_F.html',
    1, 1, 1,
    GETDATE(), GETDATE(), 1
);
GO

-- Validación: 4 registros con TemplateKey _CP
SELECT TemplateKey, BodyTemplateFileName, HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, IsActive
FROM dbo.DocumentTemplate
WHERE TemplateKey IN ('GOL_MEX_CP','MUN_MEX_CP','PRO_MEX_CP','PQF_MEX_CP');
```

### Diccionario de datos — DocumentTemplate (columnas relevantes)

| Nombre | Tipo | Descripción |
|---|---|---|
| IdDocumentTemplate | uniqueidentifier PK | Identificador único (default `newid()`) |
| TemplateKey | varchar(100) NOT NULL | Clave única del template. Para CP: `GOL_MEX_CP`, `MUN_MEX_CP`, `PRO_MEX_CP`, `PQF_MEX_CP` |
| BodyTemplateFileName | varchar(255) NOT NULL | Nombre del archivo HTML de cuerpo del documento en DocumentBuilder |
| HeaderTemplateFileName | varchar(255) NULL | Archivo de encabezado. Convención: `{TemplateKey}_H.html` |
| FooterTemplateFileName | varchar(255) NULL | Archivo de pie de página. Convención: `{TemplateKey}_F.html` |
| HasHeaderTemplate | bit NOT NULL | 1 = con header. Los templates CP usan los tres archivos |
| HasBodyTemplate | bit NOT NULL | 1 = con body (default) |
| HasFooterTemplate | bit NOT NULL | 1 = con footer. Los templates CP usan los tres archivos |
| RegistrationDate | datetime NOT NULL | Fecha de alta (default `getdate()`) |
| LastUpdateDate | datetime NOT NULL | Fecha de última modificación (default `getdate()`) |
| IsActive | bit NOT NULL | 1 = activo (default) |

**Índices existentes:** `IX_DocumentTemplate_TemplateKey` (non-clustered), `IX_DocumentTemplate_BodyTemplateFileName`, `IX_DocumentTemplate_HeaderTemplateFileName`, `IX_DocumentTemplate_FooterTemplateFileName`.

---

## ALTER VIEW vfccDocumentoFiscalCobro v3.0

La vista `vfccDocumentoFiscalCobro` fue creada en RE-028 y actualizada a v2.0 en RE-029.
RE-030 la extiende a v3.0 para exponer las 8 columnas nuevas del DoctoRelacionado del
Complemento de Pago y el JOIN al catálogo de forma de pago.

Solo se documentan los cambios incrementales respecto a v2.0. El script completo debe
construirse sobre la base de v2.0 (`R16A-RE-FU-029_BD.md`) con los siguientes agregados:

```sql
-- Prerequisito: todas las columnas del ALTER TABLE anterior deben existir
-- Ejecutar en ProquifaDotNet

ALTER VIEW [dbo].[vfccDocumentoFiscalCobro]
AS
-- [ INCLUIR DEFINICIÓN COMPLETA DE v2.0 (RE-029_BD.md) ]
-- [ AGREGAR LOS SIGUIENTES ELEMENTOS: ]

-- En SELECT, agregar estas columnas después de los campos Perú (v2.0):
--
--     -- ── CAMPOS COMPLEMENTO DE PAGO (RE-030) ───────────────────────────
--     p3l.FechaPagoCP,
--     p3l.IdCatFormaPagoSAT,
--     fpago.Clave                  AS FormaPagoClave,
--     fpago.Descripcion            AS FormaPagoDescripcion,
--     p3l.TipoCambioP_CP,
--     p3l.NumParcialidad,
--     p3l.ImpSaldoAnt,
--     p3l.ImpPagado,
--     p3l.ImpSaldoInsoluto,
--     p3l.EquivalenciaDR,

-- En la sección FROM / JOINs, agregar después de los JOINs Perú (v2.0):
--
-- -- JOIN Complemento de Pago
-- LEFT JOIN dbo.catFormaPagoSAT fpago
--     ON p3l.IdCatFormaPagoSAT = fpago.IdCatFormaPagoSAT
```

> **Nota de implementación:** El script completo de v3.0 debe incluir TODA la definición
> de v2.0 más los agregados anteriores. Se recomienda mantener un script versionado
> en el repositorio de migraciones para tener el historial completo de cada versión
> de la vista.

```sql
-- Validación post-ALTER
SELECT TOP 5
    TipoDocumentoFiscal, EstadoLinea, ClienteRegion,
    FormaPagoClave, NumParcialidad,
    ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto, EquivalenciaDR
FROM dbo.vfccDocumentoFiscalCobro
WHERE TipoDocumentoFiscal = 'COMPLEMENTO_PAGO';
```

---

## Estructuras reutilizadas — Resumen de uso en RE-030

| Estructura | Origen | Uso en RE-030 |
|---|---|---|
| `CFDIGenerada` | RE-019 | INSERT por cada CP timbrado con `IdCatTipoCFDI='COMPLEMENTO_PAGO'`, `UUID` del SAT, `IdCFDIRelacionado` = `IdCFDIGenerada` de la Factura PPD relacionada |
| `catTipoCFDI.COMPLEMENTO_PAGO` | RE-028 T1 | Discriminador del tipo de CFDI en `CFDIGenerada` |
| `catTipoCFDI.IdRegion` (MEX) | RE-029 T2 | `COMPLEMENTO_PAGO` tiene `IdRegion = MEX` — solo aplica a México |
| `fccDocumentoFiscalCobro.IdCFDIGeneradaComplemento` | RE-028 T3 | Se puebla con el `IdCFDIGenerada` del CP timbrado en la cascada PPD |
| `CFDIGenerada.IdCFDIRelacionado` | RE-028 T5 | FK blanda al `IdCFDIGenerada` de la Factura PPD que este CP complementa |
| `EmpresaFolio` (estructura) | RE-019 | Foliador con UPDLOCK atómico; RE-030 agrega filas Serie P |
| PAC TurboPac | RE-019 | Mismo cliente/servicio para timbrar el CP vía ProquifaDotNet.Timbrado |
| `catUsoCFDI` (CP01) | RE-028 o anterior | UsoCFDI=CP01 fijo en el receptor del Complemento de Pago — verificar que `Clave='CP01'` exista |
| `catMetodoDePagoCFDI.PPD` | RE-028 T1 | Solo facturas PPD generan CP; se verifica al inicializar líneas del Paso 3 |
| `CorreoEnviado` / `ArchivoCorreoEnviado` | RE-028 | Trazabilidad del correo con PDF + XML del CP adjuntos |

---

## MinIO — Configuración de Buckets

Los archivos del Complemento de Pago (PDF representativo y XML timbrado) se almacenan en MinIO. La referencia de bucket por región se resuelve leyendo `RegionConfiguracionMinioBucket` al momento de subir el archivo.

### Bucket utilizado

| BucketClave | Región | Uso en RE-030 |
|---|---|---|
| `cobranza` | México | PDF del Complemento de Pago + XML CFDI P timbrado |

El bucket `cobranza` para México ya existe en `RegionConfiguracionMinioBucket` — **no se requiere INSERT nuevo**.

### Consulta de resolución de bucket (Finanzas)

```sql
-- Ejecutar en ProquifaDotNet
-- Finanzas resuelve el bucket de cobranza MEX al subir el PDF/XML del CP
SELECT rcmb.BucketNombre
FROM dbo.RegionConfiguracionMinioBucket rcmb
INNER JOIN dbo.Region r ON rcmb.IdRegion = r.IdRegion
WHERE rcmb.BucketClave = 'cobranza'
  AND r.Clave           = 'MEX'
  AND rcmb.Activo       = 1
```

### Estructura de rutas en MinIO

| Archivo | Ruta propuesta | Descripción |
|---|---|---|
| PDF Complemento de Pago | `cobranza/complementos/{anio}/{mes}/{UUID_CP}.pdf` | PDF representativo generado por DocumentBuilder |
| XML CFDI P timbrado | `cobranza/complementos/{anio}/{mes}/{UUID_CP}.xml` | XML timbrado retornado por PAC TurboPac |

> La convención de ruta (`{anio}/{mes}/`) debe validarse con el equipo para mantener consistencia con la estructura usada por RE-028 en el mismo bucket.

### Registro en tabla Archivo

Tras subir a MinIO, Finanzas inserta en `Archivo` y actualiza `CFDIGenerada`:

```sql
-- INSERT Archivo (ruta MinIO obtenida tras upload)
INSERT INTO dbo.Archivo (IdArchivo, Nombre, Extension, RutaArchivo, FechaRegistro)
VALUES (NEWID(), @NombreArchivo, @Extension, @RutaMinIO, SYSUTCDATETIME());

-- UPDATE CFDIGenerada para ligar el PDF
UPDATE dbo.CFDIGenerada
SET IdArchivoPdf = @IdArchivo
WHERE IdCFDIGenerada = @IdCFDIGeneradaCP;
```

---

## Pendientes que impactan BD

| # | Pendiente | Impacto en BD |
|---|---|---|
| P1 | Convención FechaPago (hora 12:00:00 fija vs hora real del cobro) | Determina si `FechaPagoCP` se copia del campo Fecha del cobro o si se fuerza a `CAST(@Fecha AS date) + '12:00:00'` en la capa de aplicación |
| P2 | Formato y longitud de Serie "P" en EmpresaFolio | `FormatoFolio` y `LongitudMaxima` en los INSERT de la sección DML EmpresaFolio |
| P3 | Soporte de tasas de IVA distintas a 16% | No impacta estructura BD; solo lógica en capa de aplicación (Finanzas) para construir `TrasladoDR` con tasa variable |
| P4 | Existencia previa de `catFormaPagoSAT` con otro nombre | Si ya existe, usar la FK al catálogo existente en lugar de crear uno nuevo; renombrar columna `IdCatFormaPagoSAT` al nombre real |
| P5 | Clave `CP01` en `catUsoCFDI` | Verificar existencia antes de la implementación; si no existe, insertar el DML correspondiente |
