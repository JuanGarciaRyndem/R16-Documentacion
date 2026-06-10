# Impacto en BD — Notas de Crédito Perú (CPE tipo 07 — UBL 2.1)
**Requisito:** TPSC-RE-FU-033
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (lectura/escritura) + DocumentBuilder (escritura)
**Versión:** 1.0

---

## Resumen

RE-033 habilita el módulo de Notas de Crédito para Región Perú, equivalente funcional de TPSC-RE-FU-032 (México) adaptado a la normativa SUNAT. Genera CPEs de tipo 07 (Nota de Crédito Electrónica) en formato UBL 2.1, con motivo del catálogo 09 de SUNAT, referencia al comprobante origen por serie-correlativo (no por UUID), cálculo de IGV (18 %) y dos modalidades de captura (por partidas / manual). La empresa emisora es exclusivamente **Golocaer S.A.C.**

RE-032 ya extendió `fccNotaCredito` y `fccNotaCreditoPartida` con columnas genéricas reutilizables. RE-033 **solo agrega las 3 columnas específicas de Perú** que no existen en el esquema R16 actual.

> ⚠️ **Nota transversal:** Toda la mecánica fiscal SUNAT (catálogo 09, campos CPE tipo 07, plazos, tipo de cambio, conservación) está sujeta a validación con el asesor fiscal peruano de PROQUIFA antes de implementarse.

> ⚠️ **Pendientes que afectan BD:**
> — Motivos habilitados del catálogo 09: pendiente de confirmar con el cliente (afecta DML catMotivoCreditoSUNAT09).
> — Serie y formato de folio NC Perú: pendiente de validar con PMO (Regla 9 — ej. "FC01" + correlativo 8 dígitos).
> — Modalidad de emisión SUNAT (OSE/directa): pendiente de definir — compartida con RE-029.
> — Bucket MinIO NC Perú: pendiente de verificar en `RegionConfiguracionMinioBucket` (Region='PER').
> — Comunicación de Baja: pendiente de confirmar si se implementa en R16 (tabla `fccComunicacionBaja` en scope futuro).
> — Aplicación de NC en moneda extranjera en Validar Cobro: pendiente de confirmar TC de conversión.

---

## Impacto en BD

| #   | Cambio                                                                                                                                                     | Base de Datos          | Tipo      | Prioridad |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- | --------- | --------- |
| 1   | ALTER TABLE fccNotaCredito — ADD 3 columnas Perú (ResponseCode, ResponseDescription, TipoCambioOrigen)                                                     | ProquifaDotNet         | DDL       | Alta      |
| 2   | CREATE TABLE catMotivoCreditoSUNAT09 + DML 11 motivos catálogo 09 SUNAT                                                                                    | ProquifaDotNet         | DDL + DML | Alta      |
| 3   | DML catTipoCFDI — INSERT clave NOTA_CREDITO_PERU (TipoDocumento='07')                                                                                      | ProquifaDotNet         | DML       | Alta      |
| 4   | DML EmpresaFolio — INSERT Serie NC Perú para Golocaer S.A.C.                                                                                               | ProquifaDotNetTimbrado | DML       | Alta      |
| 5   | DML DocumentTemplate — INSERT template GOL_PER_NC                                                                                                          | DocumentBuilder        | DML       | Media     |
| 6   | DML RegionConfiguracionMinioBucket — INSERT bucket NC Perú si no existe                                                                                    | ProquifaDotNet         | DML       | Media     |
| —   | Reutiliza: `fccNotaCredito` — estructura base RE-032; RE-033 agrega columnas Perú (cambio #1)                                                              | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `fccNotaCreditoPartida` — columnas RE-032 suficientes para partidas CPE Perú                                                                    | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `CFDIGenerada` — INSERT NC como TipoDocumento='07' (CPE tipo 07)                                                                                | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `CFDIGeneradaConcepto` — INSERT partidas/líneas de la NC CPE                                                                                    | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `CFDI` — respuesta timbrado SUNAT (estructura pendiente de validar con RE-029)                                                                  | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `Archivo` — INSERT PDF representativo y XML CPE de la NC                                                                                        | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `CorreoEnviado` + `ArchivoCorreoEnviado` — trazabilidad correo al cliente                                                                       | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `EmpresaFolio` — estructura existente (RE-019); Serie NC en cambio #4                                                                           | ProquifaDotNetTimbrado | Existente | —         |
| —   | Reutiliza: `DocumentTemplate` — tabla existente (DocumentBuilder)                                                                                          | DocumentBuilder        | Existente | —         |
| —   | **NO usa** `CFDIGeneradaRelacionado` — la referencia al CPE origen es por serie-correlativo en el XML (cac:BillingReference), no por UUID con TipoRelacion | ProquifaDotNet         | —         | —         |
| —   | **NO usa** `CFDICancelacion` — en Perú la anulación es vía NC motivo 01, no vía cancelación SAT                                                            | ProquifaDotNet         | —         | —         |
| —   | **NO usa** `catMotivoCancelacionSAT` (RE-032) — Perú usa catálogo 09 SUNAT (cambio #2)                                                                     | ProquifaDotNet         | —         | —         |

---

## Diccionario de Datos

### ALTER TABLE fccNotaCredito — Columnas Perú (RE-033)

**Base de datos:** ProquifaDotNet
**Descripción:** Extensión de la tabla `fccNotaCredito` para las columnas específicas de la NC Perú. Las columnas genéricas (empresa, cliente, modalidad, estado, etc.) fueron agregadas en RE-032.

| Nombre | Tipo | Nulable | Default | Descripción |
|---|---|---|---|---|
| `ResponseCode` | `varchar(2)` | SÍ | NULL | Código del motivo catálogo 09 SUNAT (cbc:ResponseCode). FK lógica a `catMotivoCreditoSUNAT09.Clave`. |
| `ResponseDescription` | `nvarchar(500)` | SÍ | NULL | Sustento del motivo (cbc:Description). Obligatorio en XML SUNAT. Capturado o auto-generado según el origen del texto (pendiente P3). |
| `TipoCambioOrigen` | `decimal(18,6)` | SÍ | NULL | Tipo de cambio de la FECHA DE EMISIÓN de la factura origen. En Perú la NC hereda el TC de la operación original, no el del día del timbrado. NULL si la moneda es PEN. |

**Script ALTER TABLE:**

```sql
-- Ejecutar en ProquifaDotNet
-- Prerrequisito: RE-032 T1 (columnas genéricas ya existentes)

ALTER TABLE dbo.fccNotaCredito ADD [ResponseCode] varchar(2) NULL;
    -- Código motivo catálogo 09 SUNAT (cbc:ResponseCode). Ej: '01','06','04'.
GO
ALTER TABLE dbo.fccNotaCredito ADD [ResponseDescription] nvarchar(500) NULL;
    -- Sustento del motivo (cbc:Description). Obligatorio en UBL 2.1.
GO
ALTER TABLE dbo.fccNotaCredito ADD [TipoCambioOrigen] decimal(18,6) NULL;
    -- TC de la fecha de emisión de la factura origen. NULL si moneda=PEN.
    -- Diferencia clave vs México: Perú hereda el TC de la operación original.
GO

-- Validación
SELECT c.name AS Columna, t.name AS Tipo, c.is_nullable AS EsNulable
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.fccNotaCredito')
  AND c.name IN ('ResponseCode','ResponseDescription','TipoCambioOrigen')
ORDER BY c.column_id;
-- Debe retornar 3 filas
```

---

### CREATE TABLE catMotivoCreditoSUNAT09 (tabla nueva)

**Base de datos:** ProquifaDotNet
**Descripción:** Catálogo 09 de SUNAT — Motivos de emisión de Notas de Crédito Electrónicas (CPE tipo 07). Determina la modalidad de captura (por partidas o manual) y el código que viaja en `cbc:ResponseCode` del XML.

#### Columnas

| Nombre | Tipo | Nulable | Default | Descripción |
|---|---|---|---|---|
| `IdCatMotivoCreditoSUNAT09` | `uniqueidentifier` | NO | `NEWID()` | PK — Identificador único |
| `Clave` | `varchar(2)` | NO | — | Clave catálogo 09 SUNAT UNIQUE ('01'–'13') |
| `Descripcion` | `nvarchar(150)` | NO | — | Descripción oficial SUNAT del motivo |
| `Modalidad` | `varchar(20)` | NO | — | Modalidad de captura: 'POR_PARTIDAS' o 'MANUAL' |
| `Activo` | `bit` | NO | `1` | Control de vigencia |
| `FechaRegistro` | `datetime2(7)` | NO | `SYSUTCDATETIME()` | Fecha de inserción |

#### Índices

| Nombre | Columnas | Tipo |
|---|---|---|
| `PK_catMotivoCreditoSUNAT09` | `IdCatMotivoCreditoSUNAT09` | PK Clustered |
| `UQ_catMotivoCreditoSUNAT09_Clave` | `Clave` | Unique NonClustered |

#### Relaciones

Sin FK saliente. La `Clave` es referenciada lógicamente por `fccNotaCredito.ResponseCode varchar(2)`.

#### Datos iniciales — catálogo 09 SUNAT (11 motivos vigentes)

| Clave | Descripcion                            | Modalidad      |
| ----- | -------------------------------------- | -------------- |
| `01`  | Anulación de la operación              | `POR_PARTIDAS` |
| `02`  | Anulación por error en el RUC          | `POR_PARTIDAS` |
| `03`  | Corrección por error en la descripción | `POR_PARTIDAS` |
| `04`  | Descuento global                       | `MANUAL`       |
| `05`  | Descuento por ítem                     | `POR_PARTIDAS` |
| `06`  | Devolución total                       | `POR_PARTIDAS` |
| `07`  | Devolución por ítem                    | `POR_PARTIDAS` |
| `08`  | Bonificación                           | `MANUAL`       |
| `09`  | Disminución en el valor                | `MANUAL`       |
| `10`  | Otros conceptos                        | `MANUAL`       |
| `13`  | Ajustes de operaciones de exportación  | `MANUAL`       |

> ⚠️ **Pendiente:** El cliente debe confirmar cuáles de los 11 motivos se habilitan en R16. El campo `Activo` permite desactivar los no habilitados sin eliminar registros.

#### Script DDL + DML

```sql
-- Ejecutar en ProquifaDotNet

CREATE TABLE dbo.catMotivoCreditoSUNAT09 (
    [IdCatMotivoCreditoSUNAT09] uniqueidentifier NOT NULL
        CONSTRAINT [PK_catMotivoCreditoSUNAT09] PRIMARY KEY DEFAULT NEWID(),
    [Clave]       varchar(2)     NOT NULL,
    [Descripcion] nvarchar(150)  NOT NULL,
    [Modalidad]   varchar(20)    NOT NULL,
    [Activo]      bit            NOT NULL CONSTRAINT [DF_catMotivoCreditoSUNAT09_Activo] DEFAULT (1),
    [FechaRegistro] datetime2(7) NOT NULL CONSTRAINT [DF_catMotivoCreditoSUNAT09_FechaReg] DEFAULT SYSUTCDATETIME(),
    CONSTRAINT [UQ_catMotivoCreditoSUNAT09_Clave] UNIQUE ([Clave]),
    CONSTRAINT [CHK_catMotivoCreditoSUNAT09_Modalidad]
        CHECK ([Modalidad] IN ('POR_PARTIDAS','MANUAL'))
);
GO

INSERT INTO dbo.catMotivoCreditoSUNAT09 (Clave, Descripcion, Modalidad) VALUES
('01', 'Anulación de la operación',              'POR_PARTIDAS'),
('02', 'Anulación por error en el RUC',          'POR_PARTIDAS'),
('03', 'Corrección por error en la descripción', 'POR_PARTIDAS'),
('04', 'Descuento global',                        'MANUAL'),
('05', 'Descuento por ítem',                      'POR_PARTIDAS'),
('06', 'Devolución total',                        'POR_PARTIDAS'),
('07', 'Devolución por ítem',                     'POR_PARTIDAS'),
('08', 'Bonificación',                            'MANUAL'),
('09', 'Disminución en el valor',                 'MANUAL'),
('10', 'Otros conceptos',                         'MANUAL'),
('13', 'Ajustes de operaciones de exportación',   'MANUAL');
GO

-- Validación
SELECT Clave, Descripcion, Modalidad, Activo
FROM dbo.catMotivoCreditoSUNAT09
ORDER BY Clave;
-- Debe retornar 11 filas con Activo=1
```

#### Consideraciones especiales

- El motivo `01` hereda **todas** las partidas del comprobante origen (deja la operación en cero). En el frontend la tabla de partidas se pre-carga con `CantidadNC = CantidadFacturada` y no es editable.
- Los motivos `02` y `03` también son `POR_PARTIDAS` aunque corresponden a errores de datos; el XML debe referenciar el comprobante completo.
- `Activo=0` desactiva el motivo sin perder el registro histórico.

---

### DML catTipoCFDI — INSERT NOTA_CREDITO_PERU

```sql
-- Prerrequisito: catTipoCFDI existe (RE-028 T1)
-- Ejecutar en ProquifaDotNet

SELECT * FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO_PERU';
-- Debe retornar 0 filas antes de continuar

INSERT INTO dbo.catTipoCFDI (IdCatTipoCFDI, Clave, Descripcion, TipoDocumentoSAT, Activo)
SELECT NEWID(), 'NOTA_CREDITO_PERU', 'Nota de Crédito Electrónica Perú (CPE tipo 07)', '07', 1
WHERE NOT EXISTS (SELECT 1 FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO_PERU');
GO

-- Validación
SELECT IdCatTipoCFDI, Clave, TipoDocumentoSAT, Activo
FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO_PERU';
-- Debe retornar 1 fila con TipoDocumentoSAT='07'
```

---

### DML EmpresaFolio — INSERT Serie NC Perú

```sql
-- Prerrequisito: EmpresaFolio y Golocaer S.A.C. existen (RE-019)
-- Ejecutar en ProquifaDotNetTimbrado
-- ⚠️ Serie y FormatoFolio pendientes de validar con PMO (Regla 9 — candidato: Serie='FC01', correlativo 8 dígitos)

-- Verificar ausencia
SELECT e.Prefijo, ef.Serie FROM dbo.EmpresaFolio ef
INNER JOIN dbo.Empresa e ON ef.IdEmpresa = e.IdEmpresa
WHERE e.Prefijo = 'GOL' AND ef.Serie LIKE 'FC%';
-- 0 filas antes de continuar

-- Golocaer S.A.C. — NC Perú Serie FC01
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa, 'FC01', 0, 'FC01{folio:00000000}', 12, 1
FROM dbo.Empresa e WHERE e.Prefijo = 'GOL' AND e.IdRegion = (
    SELECT IdRegion FROM dbo.Region WHERE Clave = 'PER'
);
GO

-- Validación
SELECT e.Prefijo, ef.Serie, ef.UltimoFolio, ef.FormatoFolio
FROM dbo.EmpresaFolio ef
INNER JOIN dbo.Empresa e ON ef.IdEmpresa = e.IdEmpresa
WHERE e.Prefijo = 'GOL' AND ef.Serie LIKE 'FC%';
-- Debe retornar 1 fila con UltimoFolio=0
```

---

### DML DocumentTemplate — INSERT GOL_PER_NC

```sql
-- Ejecutar en DocumentBuilder

SELECT TemplateKey FROM dbo.DocumentTemplate WHERE TemplateKey = 'GOL_PER_NC';
-- 0 filas antes de continuar

INSERT INTO dbo.DocumentTemplate
    (TemplateKey, HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName,
     HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, RegistrationDate, LastUpdateDate, IsActive)
VALUES
    ('GOL_PER_NC', 'GOL_PER_NC_H.html', 'GOL_PER_NC_B.html', 'GOL_PER_NC_F.html',
     1, 1, 1, GETDATE(), GETDATE(), 1);
GO

-- Validación
SELECT TemplateKey, HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, IsActive
FROM dbo.DocumentTemplate WHERE TemplateKey = 'GOL_PER_NC';
-- Debe retornar 1 fila con Has*=1 e IsActive=1
```

---

## Tablas reutilizadas — referencias cruzadas

| Tabla | Origen | Uso en RE-033 |
|---|---|---|
| `fccNotaCredito` | Pre-R16 + RE-032 | Cabecera de la NC Perú; RE-033 agrega 3 columnas Perú |
| `fccNotaCreditoPartida` | Pre-R16 + RE-032 | Partidas de la NC CPE; columnas RE-032 son suficientes |
| `CFDIGenerada` | RE-019 + RE-028 | INSERT NC como `TipoDocumento='07'`, `IdCatTipoCFDI`=NOTA_CREDITO_PERU |
| `CFDIGeneradaConcepto` | RE-019 | INSERT líneas CPE (cac:CreditNoteLine) |
| `CFDI` | RE-019 | Respuesta timbrado SUNAT (estructura pendiente RE-029) |
| `Archivo` | Pre-R16 | PDF representativo y XML CPE timbrado |
| `CorreoEnviado` + `ArchivoCorreoEnviado` | Pre-R16 | Trazabilidad correo automático al cliente |
| `EmpresaFolio` | RE-019 | Foliador UPDLOCK atómico por Serie NC Perú |
| `DocumentTemplate` | DocumentBuilder | Template HTML GOL_PER_NC (H/B/F) |
| `RegionConfiguracionMinioBucket` | Pre-R16 | Resolución bucket Perú NC (⚠️ verificar) |

---

## Diferencias clave vs RE-032 (México)

| Aspecto | RE-032 México | RE-033 Perú |
|---|---|---|
| Tipo documento | TipoDocumento='E' (CFDI 4.0) | TipoDocumento='07' (CPE UBL 2.1) |
| Motivo | `catMotivoCancelacionSAT` (4 claves SAT) | `catMotivoCreditoSUNAT09` (11 motivos SUNAT) |
| Referencia origen | `CFDIGeneradaRelacionado` (UUID + TipoRelacion='01') | cac:BillingReference en XML por serie-correlativo; sin CFDIGeneradaRelacionado |
| Cancelación SAT | `CFDICancelacion` (condicional totalidad+mismo mes) | NO APLICA — anulación es NC motivo 01 |
| Tipo de cambio | Del día del timbrado | Heredado de la fecha de emisión de la factura origen |
| Empresas | GOL, MUN, PRO, PQF | Solo GOL (Golocaer S.A.C.) |
| ETL a Legacy | SSIS PCconnect | NO — Perú no transfiere a PCconnect |
| Tabla catálogo | `catMotivoCancelacionSAT` (5 cols) | `catMotivoCreditoSUNAT09` (6 cols + Modalidad) |
| catTipoCFDI | NOTA_CREDITO (TipoDocumento='E') | NOTA_CREDITO_PERU (TipoDocumento='07') |
| Templates PDF | GOL/MUN/PRO/PQF_MEX_NC | Solo GOL_PER_NC |

---

## Pendientes

| ID | Pendiente | Sección afectada |
|---|---|---|
| P1 | Confirmar motivos habilitados del catálogo 09 para R16 (de 11 disponibles) | DML catMotivoCreditoSUNAT09 |
| P2 | Confirmar Serie y formato de folio NC Perú con PMO (candidato: FC01 + 8 dígitos) | DML EmpresaFolio |
| P3 | Confirmar origen de cbc:Description: auto-generado desde motivo/partidas o capturado por el usuario | fccNotaCredito.ResponseDescription |
| P4 | Confirmar bucket MinIO NC Perú en `RegionConfiguracionMinioBucket` (Region='PER') | DML MinIO |
| P5 | Confirmar estructura de la respuesta de timbrado SUNAT para mapeo en tabla `CFDI` (RE-029 pendiente) | CFDI |
| P6 | Confirmar si se implementa Comunicación de Baja en R16 (requeriría tabla `fccComunicacionBaja`) | Scope futuro |
| P7 | Confirmar tipo de cambio en aplicación de NC en moneda extranjera en Validar Cobro | fccNotaCreditoPartida / Validar Cobro |
| P8 | Validar toda la mecánica fiscal SUNAT con el asesor fiscal peruano de PROQUIFA | Global |

---

## Brechas

| ID | Brecha | Impacto |
|---|---|---|
| B1 | **Modalidad de emisión SUNAT no definida** — no se ha confirmado si se usa OSE, emisión directa ni se reutiliza TurboPac. Brecha compartida con RE-029. | Bloquea T7 (endpoint timbrado Timbrado) y T10 (flujo timbrado Finanzas). Las tareas de BD, catálogos y wizard (T1–T8) pueden avanzar. |
| B2 | **Estructura de respuesta timbrado SUNAT desconocida** — la tabla `CFDI` puede necesitar columnas adicionales Perú (constancia, estado SUNAT). Depende de RE-029. | Depende de resolución B1. |
| B3 | **`fccNotaCredito.IdTPProformaPedido` NOT NULL** — brecha heredada de RE-032 B1. Si no se resolvió en RE-032, sigue bloqueando ALTER TABLE en RE-033. | Bloquea T1 si no fue resuelta en RE-032. |
| B4 | **Comunicación de Baja no definida** — si se implementa, requiere tabla `fccComunicacionBaja` y endpoint adicional. | Sin definir: no se crea tabla. Se documenta como pendiente. |
