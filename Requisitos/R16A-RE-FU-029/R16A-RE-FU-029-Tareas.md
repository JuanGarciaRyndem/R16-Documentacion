# Tareas BackEnd — R16A-RE-FU-029
**Requisito:** Validar Cobro: Paso 3 Perú — Facturación y Envío
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder

---

> **Orden de ejecución sugerido:** BD catálogo SUNAT (T1) → BD ALTER catTipoCFDI (T2) → BD ALTER fccDocumentoFiscalCobro (T3) → BD ALTER vista (T4) → Timbrado endpoint CPE (T5) → Finanzas: inicialización Perú (T6) → autosave SUNAT (T7) → previsualización CPE (T8) → timbrado CPE (T9) → Finanzas: envío Perú (T10) → FEE + CDP sin Legacy (T11) → cierre wizard Perú (T12).
>
> **Dependencias externas:** R16A-RE-FU-028 completo (catálogos, `fccDocumentoFiscalCobro`, `fccConfirmacionPedido`, `CFDIGenerada` con `IdCatTipoCFDI`, `vfccDocumentoFiscalCobro` v1 y `tpPedido.FechaEstimadaEntrega` aplicados). R16A-RE-FU-026 completo (`fccPagoFacturaPedido`, `fccPagoFacturaAdelanto`). R16A-RE-FU-020 completo (`FacturaPdfMappingService` Perú, `GOLPERU_PER_FAC`).
>
> **Brechas bloqueantes activas:** B1 (modalidad emisión SUNAT indefinida) bloquea T5 y T9. B4 (datos fiscales SUNAT del producto) bloquea T9. Las tareas T5 y T9 pueden prepararse en estructura pero no pueden probarse en producción hasta resolver B1 y B4. Ver `R16A-RE-FU-029_BD.md` y `R16A-RE-FU-029-Back.md` — sección Brechas.
>
> **Sin ETL Legacy:** Perú no transfiere nada a Legacy. No existe T17 equivalente para este requisito. La Tarea 12 cubre el post-envío completo (FEE + CDP) sin transferencias ETL.

---

## Resumen de tareas

| #   | Clave                 | Título simple                                                                                    | Tipo            | Aplicativo              |
| --- | --------------------- | ------------------------------------------------------------------------------------------------ | --------------- | ----------------------- |
| 1   | CREATE-TABL-CH        | Crear catTipoOperacionSUNAT (catálogo 51 SUNAT) con DML inicial                                  | BD              | ProquifaDotNet          |
| 2   | UPDATE-TABL-CH        | Extender catTipoCFDI: ADD IdRegion + UPDATE entradas MEX + INSERT FACTURA_CPE                    | BD              | ProquifaDotNet          |
| 3   | UPDATE-TABL-M         | Extender fccDocumentoFiscalCobro con columnas Perú (TipoOperacion, CondicionPago)                | BD              | ProquifaDotNet          |
| 4   | CREATE-SCRIPT-CONTROL | Actualizar vista vfccDocumentoFiscalCobro v2.0 (JOINs Perú + corrección resolución catálogos)   | BD              | ProquifaDotNet          |
| 5   | IMP-EXIST-SERVICE     | Extender endpoint de timbrado en Timbrado para CPE SUNAT Perú (⚠️ bloqueado Brecha B1)           | Back            | ProquifaDotNet.Timbrado |
| 6   | SERV-TRANSACT         | Implementar inicialización del Paso 3 Perú (tipo único FACTURA, sin lógica condicional)          | Back            | ProquifaDotNet.Finanzas |
| 7   | SERV-SIMPLE-PUT       | Implementar auto-guardado Tipo de Operación SUNAT y Condición de Pago por línea                  | Back            | ProquifaDotNet.Finanzas |
| 8   | IMP-EXIST-SERVICE     | Implementar previsualización PDF CPE por línea (reutiliza FacturaPdfMappingService Perú RE-020)  | Back            | ProquifaDotNet.Finanzas |
| 9   | ALG-COMPLX-LOGIC      | Implementar timbrado CPE por línea — escenario único sin cascada (⚠️ bloqueado Brechas B1/B4)    | Back            | ProquifaDotNet.Finanzas |
| 10  | IMP-EXIST-SERVICE     | Implementar modal de Envío Perú: generar CDP + despacho Brevo (PDF CPE + XML CPE + PDF CDP)      | Back            | ProquifaDotNet.Finanzas |
| 11  | SERV-TRANSACT         | Implementar acciones post-envío Perú: FEE + registro CDP (sin transferencia a Legacy)            | Back            | ProquifaDotNet.Finanzas |
| 12  | SERV-SIMPLE-PUT       | Implementar cierre automático del wizard Perú al completar todas las líneas enviadas             | Back            | ProquifaDotNet.Finanzas |

---

## TAREA 1

**[ RE-FU-029 ] [CREATE-TABL-CH] Crear catTipoOperacionSUNAT (catálogo 51 SUNAT) con DML inicial**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3 Perú

**Consideraciones previas:**
- Catálogo nuevo. No existe en BD. Equivalente al `catUsoCFDI` (SAT) pero para SUNAT — se consigna por línea en el Paso 3 Perú.
- Patrón estándar del proyecto: `uniqueidentifier PK` con `NEWID()`, campo `Clave varchar UNIQUE`, `Descripcion nvarchar`, `Activo bit DEFAULT(1)`, `FechaRegistro datetime2(7) DEFAULT SYSUTCDATETIME()`.
- Datos iniciales: los 7 códigos del catálogo 51 SUNAT más frecuentes para distribución B2B. Extender conforme lo requiera Golocaer S.A.C.
- ⚠️ **Pendiente (Brecha B8):** Confirmar con el cliente si el Tipo de Operación lo selecciona el operador o lo fija el sistema (candidato `0101` — Venta interna para el flujo estándar).
- Es **prerrequisito** de la Tarea 3 (`fccDocumentoFiscalCobro` agrega FK a este catálogo).

**Objetivo general:**
Crear el catálogo `catTipoOperacionSUNAT` en ProquifaDotNet con los datos iniciales del catálogo 51 SUNAT, habilitando la FK que `fccDocumentoFiscalCobro` consumirá para el campo de Tipo de Operación de las líneas Perú.

**Objetivos específicos:**
- Ejecutar `CREATE TABLE catTipoOperacionSUNAT` con PK, UNIQUE en `Clave`, `Activo` y `FechaRegistro`.
- Insertar los 7 códigos iniciales del catálogo 51 SUNAT.
- Verificar que PK y UNIQUE quedan correctamente definidos.
- Verificar que todos los registros tienen `Activo=1`.

**Resultado esperado:**
`catTipoOperacionSUNAT` existe en ProquifaDotNet con los datos iniciales del catálogo 51, listo para ser referenciado por `fccDocumentoFiscalCobro`.

**Entregables:**
- Script DDL + DML: `CREATE TABLE catTipoOperacionSUNAT` + `INSERT` datos iniciales
- Script de validación (`SELECT` con conteo y estructura)

**Scripts:**

```sql
-- Ejecutar en ProquifaDotNet
CREATE TABLE [dbo].[catTipoOperacionSUNAT](
    [IdCatTipoOperacionSUNAT]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catTipoOperacionSUNAT_Id]     DEFAULT (NEWID()),
    [Clave]                    varchar(10)      NOT NULL,
        -- Código catálogo 51 SUNAT
    [Descripcion]              nvarchar(200)    NOT NULL,
    [Activo]                   bit              NOT NULL
        CONSTRAINT [DF_catTipoOperacionSUNAT_Activo] DEFAULT (1),
    [FechaRegistro]            datetime2(7)     NOT NULL
        CONSTRAINT [DF_catTipoOperacionSUNAT_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catTipoOperacionSUNAT]
        PRIMARY KEY CLUSTERED ([IdCatTipoOperacionSUNAT]),
    CONSTRAINT [UQ_catTipoOperacionSUNAT_Clave]
        UNIQUE ([Clave])
);
GO

INSERT INTO dbo.catTipoOperacionSUNAT (Clave, Descripcion) VALUES
    ('0101', 'Venta interna'),
    ('0112', 'Venta interna — sustento de traslado de mercancía'),
    ('0200', 'Exportación de bienes'),
    ('0201', 'Exportación de servicios'),
    ('1001', 'Operación sujeta a detracción'),
    ('1002', 'Operación sujeta a detracción con liquidación'),
    ('2001', 'Operación sujeta a percepción');
-- Extender con los códigos adicionales del catálogo 51 que Golocaer S.A.C. requiera.
GO

-- Validación
SELECT COUNT(*) AS Registros, SUM(CASE WHEN Activo=1 THEN 1 ELSE 0 END) AS Activos
FROM dbo.catTipoOperacionSUNAT;
```

**Criterios de aceptación:**
- `catTipoOperacionSUNAT` existe con la estructura definida en `R16A-RE-FU-029_BD.md`.
- Contiene exactamente 7 registros iniciales; todos con `Activo=1`.
- PK y UNIQUE constraint activas.
- ⚠️ La configurabilidad del valor por defecto (B8) se documenta como pendiente.

**Más información de la tarea:**
Ver sección *"Catálogo Nuevo: catTipoOperacionSUNAT"* en `R16A-RE-FU-029_BD.md` y sección *"Parte A / A1"* en `R16A-RE-FU-029-Back.md`.

**Recursos:**
- `R16A-RE-FU-029_BD.md` — DDL + DML catTipoOperacionSUNAT
- `R16A-RE-FU-029-Back.md` — Parte A, sección A1

---

## TAREA 2

**[ RE-FU-029 ] [UPDATE-TABL-CH] Extender catTipoCFDI: ADD IdRegion + UPDATE entradas MEX + INSERT FACTURA_CPE**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — catTipoCFDI

**Consideraciones previas:**
- `catTipoCFDI` fue creado en RE-FU-028 (Tarea 1) exclusivamente para México. RE-029 lo convierte en un catálogo cross-región.
- **Prerequisito:** RE-FU-028 Tarea 1 debe estar ejecutada en producción (`catTipoCFDI` y `catRegion` deben existir).
- Se agregan 3 pasos: (1) ADD columna `IdRegion uniqueidentifier NULL FK → catRegion`; (2) UPDATE entradas México existentes; (3) INSERT entrada Perú `FACTURA_CPE`.
- La columna va como **nullable** para no romper las filas existentes al momento del ALTER.
- Confirmar la estructura de `catRegion` (Clave `'MEX'`/`'PER'`) antes de ejecutar los UPDATEs.
- Es **prerrequisito** de la Tarea 5 (Timbrado usa `catTipoCFDI.IdRegion` para discriminar tipo de comprobante por región).

**Objetivo general:**
Extender `catTipoCFDI` con la columna `IdRegion` para soportar la diferenciación de tipos de documento fiscal por región (MEX/PER), y agregar la clave `FACTURA_CPE` para los CPE SUNAT de Perú.

**Objetivos específicos:**
- `ALTER TABLE catTipoCFDI ADD IdRegion uniqueidentifier NULL` + FK a `catRegion`.
- `UPDATE catTipoCFDI SET IdRegion = (MEX)` para las 4 entradas existentes de RE-028 (`FACTURA_PPD`, `FACTURA_PUE`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO`).
- `INSERT catTipoCFDI (FACTURA_CPE, Perú)` con `IdRegion = (PER)`.
- Verificar que el FK a `catRegion` está activo.
- Verificar que los 4 registros México tienen `IdRegion` poblado y el nuevo registro Perú tiene `IdRegion = PER`.

**Resultado esperado:**
`catTipoCFDI` con columna `IdRegion` activa: 4 entradas México y 1 entrada Perú (`FACTURA_CPE`), todas con `IdRegion` correcto.

**Entregables:**
- Script DDL: `ALTER TABLE catTipoCFDI ADD IdRegion` + FK
- Script DML: `UPDATE` entradas México + `INSERT FACTURA_CPE`
- Script de validación

**Scripts:**

```sql
-- Prerequisito: catTipoCFDI (RE-FU-028 T1) y catRegion deben existir
-- Ejecutar en ProquifaDotNet

-- 1. Agregar columna (nullable para no romper filas existentes)
ALTER TABLE dbo.catTipoCFDI
    ADD [IdRegion] uniqueidentifier NULL;
GO

ALTER TABLE dbo.catTipoCFDI
    ADD CONSTRAINT [FK_catTipoCFDI_Region]
        FOREIGN KEY ([IdRegion]) REFERENCES dbo.catRegion([IdRegion]);
GO

-- 2. Poblar entradas México de RE-FU-028
UPDATE dbo.catTipoCFDI
SET IdRegion = (SELECT IdRegion FROM dbo.catRegion WHERE Clave = 'MEX')
WHERE Clave IN ('FACTURA_PPD', 'FACTURA_PUE', 'FACTURA_ANTICIPO', 'COMPLEMENTO_PAGO');
GO

-- 3. Insertar entrada Perú
INSERT INTO dbo.catTipoCFDI (Clave, Descripcion, IdRegion)
SELECT 'FACTURA_CPE',
       'Factura electrónica SUNAT — CPE tipo 01 UBL 2.1 (Perú)',
       IdRegion
FROM dbo.catRegion
WHERE Clave = 'PER';
GO

-- Validación
SELECT ct.Clave, ct.Descripcion, cr.Clave AS Region
FROM dbo.catTipoCFDI ct
LEFT JOIN dbo.catRegion cr ON ct.IdRegion = cr.IdRegion
ORDER BY cr.Clave, ct.Clave;
```

**Criterios de aceptación:**
- `catTipoCFDI.IdRegion uniqueidentifier NULL` existe con FK activo hacia `catRegion`.
- Las 4 entradas México tienen `IdRegion` poblado con la región MEX.
- `FACTURA_CPE` existe con `IdRegion` poblado con la región PER.
- Total de registros en `catTipoCFDI`: 5 (4 MEX + 1 PER).

**Más información de la tarea:**
Ver sección *"ALTER TABLE catTipoCFDI — Agregar IdRegion + FACTURA_CPE"* en `R16A-RE-FU-029_BD.md` y sección *"Parte A / A2"* en `R16A-RE-FU-029-Back.md`.

**Recursos:**
- `R16A-RE-FU-029_BD.md` — ALTER TABLE catTipoCFDI
- `R16A-RE-FU-029-Back.md` — Parte A, sección A2

---

## TAREA 3

**[ RE-FU-029 ] [UPDATE-TABL-M] Extender fccDocumentoFiscalCobro con columnas Perú (TipoOperacion, CondicionPago)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3

**Consideraciones previas:**
- `fccDocumentoFiscalCobro` fue creado en RE-FU-028 (Tarea 3). Esta tarea la extiende sin modificar columnas existentes.
- **Prerequisito:** Tarea 1 de este requisito (`catTipoOperacionSUNAT` debe existir para el FK). `catCondicionesDePago` ya existe desde RE-FU-018/019.
- Ambas columnas son **nullable**: las líneas México siguen sin poblarlas; las líneas Perú dejan NULL las columnas México (`IdCatUsoCFDI`, `IdCatMetodoDePagoCFDI`).
- La validación de exclusividad regional (MEX rellena columnas México, PER rellena columnas Perú) es responsabilidad de la capa de aplicación (Finanzas), no de un CHECK CONSTRAINT en BD.
- Verificar que ningún SP, vista ni trigger dependiente de `fccDocumentoFiscalCobro` se rompe tras el ALTER.

**Objetivo general:**
Extender `fccDocumentoFiscalCobro` con dos columnas nullable para las líneas Perú: `IdCatTipoOperacionSUNAT` (equivalente al Uso CFDI mexicano) e `IdCatCondicionesDePago` (equivalente al Método de Pago mexicano), habilitando el Paso 3 Perú.

**Objetivos específicos:**
- `ALTER TABLE fccDocumentoFiscalCobro ADD IdCatTipoOperacionSUNAT uniqueidentifier NULL` + FK a `catTipoOperacionSUNAT`.
- `ALTER TABLE fccDocumentoFiscalCobro ADD IdCatCondicionesDePago uniqueidentifier NULL` + FK a `catCondicionesDePago`.
- Verificar que las FKs están activas.
- Verificar que registros existentes de México (si los hay) tienen ambas columnas nuevas en NULL sin errores.
- Verificar que SPs, vistas y triggers dependientes no presentan errores tras el ALTER.

**Resultado esperado:**
`fccDocumentoFiscalCobro` con las 2 columnas Perú disponibles, FKs activas, sin impacto en registros existentes México.

**Entregables:**
- Script DDL: `ALTER TABLE fccDocumentoFiscalCobro ADD IdCatTipoOperacionSUNAT` + FK
- Script DDL: `ALTER TABLE fccDocumentoFiscalCobro ADD IdCatCondicionesDePago` + FK
- Script de validación (estructura + registros existentes sin regresión)

**Scripts:**

```sql
-- Prerequisito: catTipoOperacionSUNAT (T1 de RE-029), catCondicionesDePago (RE-018/019),
--               fccDocumentoFiscalCobro (RE-028 T3) deben existir
-- Ejecutar en ProquifaDotNet

ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [IdCatTipoOperacionSUNAT] uniqueidentifier NULL;
        -- FK a catTipoOperacionSUNAT. NULL para líneas México; código catálogo 51 para líneas Perú.
        -- Equivalente peruano de IdCatUsoCFDI.
GO

ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD CONSTRAINT [FK_fccDocumentoFiscalCobro_TipoOperacionSUNAT]
        FOREIGN KEY ([IdCatTipoOperacionSUNAT])
        REFERENCES dbo.catTipoOperacionSUNAT([IdCatTipoOperacionSUNAT]);
GO

ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [IdCatCondicionesDePago] uniqueidentifier NULL;
        -- FK a catCondicionesDePago (existente RE-018/019). NULL para líneas México.
        -- Equivalente peruano de IdCatMetodoDePagoCFDI.
GO

ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD CONSTRAINT [FK_fccDocumentoFiscalCobro_CondicionesDePago]
        FOREIGN KEY ([IdCatCondicionesDePago])
        REFERENCES dbo.catCondicionesDePago([IdCatCondicionesDePago]);
GO

-- Validación: confirmar columnas nuevas + FK activas
SELECT c.name AS Columna, t.name AS Tipo, c.is_nullable AS EsNullable
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.fccDocumentoFiscalCobro')
  AND c.name IN ('IdCatTipoOperacionSUNAT','IdCatCondicionesDePago');

-- Confirmar registros existentes no afectados (deben tener NULL en columnas nuevas)
SELECT COUNT(*) AS RegistrosExistentes,
       SUM(CASE WHEN IdCatTipoOperacionSUNAT IS NULL THEN 1 ELSE 0 END) AS NullosTipoOp,
       SUM(CASE WHEN IdCatCondicionesDePago  IS NULL THEN 1 ELSE 0 END) AS NulosCondPago
FROM dbo.fccDocumentoFiscalCobro;
```

**Criterios de aceptación:**
- `IdCatTipoOperacionSUNAT uniqueidentifier NULL` existe con FK activo hacia `catTipoOperacionSUNAT`.
- `IdCatCondicionesDePago uniqueidentifier NULL` existe con FK activo hacia `catCondicionesDePago`.
- Registros existentes de México tienen ambas columnas en NULL (sin errores de FK).
- Ningún SP, vista ni trigger presenta errores tras el ALTER.

**Más información de la tarea:**
Ver sección *"ALTER TABLE fccDocumentoFiscalCobro — Columnas Perú"* en `R16A-RE-FU-029_BD.md` y sección *"Parte A / A3"* en `R16A-RE-FU-029-Back.md`.

**Recursos:**
- `R16A-RE-FU-029_BD.md` — ALTER TABLE fccDocumentoFiscalCobro
- `R16A-RE-FU-029-Back.md` — Parte A, sección A3

---

## TAREA 4

**[ RE-FU-029 ] [CREATE-SCRIPT-CONTROL] Actualizar vista vfccDocumentoFiscalCobro v2.0 (JOINs Perú + corrección resolución catálogos)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3

**Consideraciones previas:**
- La vista `vfccDocumentoFiscalCobro` fue creada en RE-FU-028 (Tarea 6). Esta tarea la reemplaza con la versión v2.0.
- Las Tareas 1, 2 y 3 de este requisito deben estar ejecutadas antes de actualizar la vista.
- **Corrección incluida:** En RE-028 la vista accedía a `p3l.TipoDocumentoFiscal` y `p3l.EstadoLinea` como si fueran columnas de texto; son FKs. La v2.0 corrige esto con `INNER JOIN` a `catTipoDocumentoFiscal` y `catDocumentoFiscalCobroEstado`.
- Usar `ALTER VIEW` (no DROP + CREATE) para preservar permisos existentes.
- Agregar columna `c.Region AS ClienteRegion` para que Finanzas pueda discriminar líneas MEX/PER en la misma consulta.
- Los alias `CPE_Serie` y `CPE_Correlativo` reutilizan el JOIN `cg_f` (mismo JOIN que Factura México) — el discriminador `catTipoCFDI.Clave = 'FACTURA_CPE'` en la capa de aplicación distingue si el registro es CPE o CFDI.

**Objetivo general:**
Reemplazar `vfccDocumentoFiscalCobro` con la versión v2.0 que: (1) agrega JOINs a los catálogos Perú (`catTipoOperacionSUNAT`, `catCondicionesDePago`), (2) corrige la resolución de `TipoDocumentoFiscal` y `EstadoLinea` con `INNER JOIN` a sus respectivos catálogos, y (3) expone `ClienteRegion` como discriminador MEX/PER.

**Objetivos específicos:**
- `ALTER VIEW dbo.vfccDocumentoFiscalCobro AS ...` con el script completo de la v2.0.
- Incluir `INNER JOIN catTipoDocumentoFiscal` y `INNER JOIN catDocumentoFiscalCobroEstado` en lugar del acceso directo a columnas FK.
- Agregar `LEFT JOIN catTipoOperacionSUNAT` y `LEFT JOIN catCondicionesDePago`.
- Agregar `c.Region AS ClienteRegion` desde la tabla `Cliente`.
- Agregar alias `CPE_Serie = cg_f.Serie` y `CPE_Correlativo = cg_f.Folio` para líneas Perú.
- Verificar que la vista compila sin errores y retorna datos correctos para líneas MEX y PER.
- Verificar no regresión: las líneas México existentes siguen siendo consultables.

**Resultado esperado:**
Vista `vfccDocumentoFiscalCobro` v2.0 activa, con JOINs correctos a catálogos, discriminador de región y columnas Perú expuestas.

**Entregables:**
- Script DDL: `ALTER VIEW vfccDocumentoFiscalCobro` (v2.0 completa)
- Script de validación (`SELECT TOP 10 *` con verificación de columnas nuevas)

**Scripts:**

```sql
-- Prerequisito: catTipoOperacionSUNAT (T1), catTipoCFDI con IdRegion (T2),
--               fccDocumentoFiscalCobro con columnas Perú (T3) deben existir
-- Ejecutar en ProquifaDotNet

ALTER VIEW [dbo].[vfccDocumentoFiscalCobro]
AS
SELECT
    p3l.IdFCCDocumentoFiscalCobro,
    -- Tipo y estado resueltos desde catálogos (corrige RE-028)
    tdoc.Clave                   AS TipoDocumentoFiscal,
    tdoc.Descripcion             AS TipoDocumentoFiscalDescripcion,
    est.Clave                    AS EstadoLinea,
    est.Descripcion              AS EstadoDescripcion,
    -- Región del cliente (discriminador MEX/PER)
    c.Region                     AS ClienteRegion,
    -- ── CAMPOS MÉXICO ──────────────────────────────────────────────────
    p3l.IdCatUsoCFDI,
    ufo.Clave                    AS UsoCFDIClave,
    ufo.Descripcion              AS UsoCFDIDescripcion,
    p3l.IdCatMetodoDePagoCFDI,
    mpc.Clave                    AS MetodoPagoClave,
    mpc.Descripcion              AS MetodoPagoDescripcion,
    p3l.IdCFDIGeneradaFactura,
    cg_f.UUID                    AS UUID_Factura,
    cg_f.Folio                   AS Folio_Factura,
    cg_f.Serie                   AS Serie_Factura,
    cg_f.FechaEmision            AS FechaEmision_Factura,
    p3l.IdCFDIGeneradaComplemento,
    cg_c.UUID                    AS UUID_Complemento,
    cg_c.Folio                   AS Folio_Complemento,
    -- ── CAMPOS PERÚ ────────────────────────────────────────────────────
    p3l.IdCatTipoOperacionSUNAT,
    tos.Clave                    AS TipoOperacionSUNATClave,
    tos.Descripcion              AS TipoOperacionSUNATDescripcion,
    p3l.IdCatCondicionesDePago,
    cdp.Clave                    AS CondicionPagoClave,
    cdp.Descripcion              AS CondicionPagoDescripcion,
    -- CPE Perú: reutiliza JOIN cg_f; Serie F001 + Correlativo 8 dígitos
    cg_f.Serie                   AS CPE_Serie,
    cg_f.Folio                   AS CPE_Correlativo,
    -- ── CAMPOS COMPARTIDOS ─────────────────────────────────────────────
    p3l.IdFCCPagoFacturaPedido,
    pfp.IdFCCPagoCliente         AS IdFCCPagoCliente_PFP,
    pfp.IdTPProformaPedido,
    pp.Folio                     AS FolioProforma,
    pp.MontoTotal                AS MontoProforma,
    pp.MontoPendiente,
    e_pp.Prefijo                 AS EmpresaEmisoraProforma,
    p3l.IdFCCPagoFacturaAdelanto,
    pfa.IdFCCPagoCliente         AS IdFCCPagoCliente_PFA,
    pfa.IdTPProformaAdelanto,
    pa.Monto                     AS MontoFAA,
    cg_faa.UUID                  AS UUID_FAA,
    COALESCE(pfp.IdFCCPagoCliente, pfa.IdFCCPagoCliente) AS IdFCCPagoCliente,
    fpc.Folio                    AS FolioCobro,
    fpc.Monto                    AS MontoCobro,
    fpc.MXN                      AS CobroMXN,
    fpc.USD                      AS CobroUSD,
    fpc.TipoDeCambio,
    fpc.IdCliente,
    c.Nombre                     AS ClienteNombre,
    dfc.RazonSocial              AS ClienteRazonSocial,
    dfc.RFC                      AS ClienteRFC,
    tp.IdTPPedido,
    tp.FolioPedidoInterno,
    tp.FechaEstimadaEntrega,
    p3l.FechaGeneracion,
    p3l.FechaEnvio,
    p3l.FechaRegistro,
    p3l.FechaUltimaActualizacion
FROM dbo.fccDocumentoFiscalCobro p3l
INNER JOIN dbo.catTipoDocumentoFiscal tdoc
    ON p3l.IdCatTipoDocumentoFiscal = tdoc.IdCatTipoDocumentoFiscal
INNER JOIN dbo.catDocumentoFiscalCobroEstado est
    ON p3l.IdCatDocumentoFiscalCobroEstado = est.IdCatDocumentoFiscalCobroEstado
LEFT JOIN dbo.fccPagoFacturaPedido pfp
    ON p3l.IdFCCPagoFacturaPedido = pfp.IdFCCPagoFacturaPedido
LEFT JOIN dbo.tpProformaPedido pp
    ON pfp.IdTPProformaPedido = pp.IdTPProformaPedido
LEFT JOIN dbo.Empresa e_pp
    ON pp.IdEmpresa = e_pp.IdEmpresa
LEFT JOIN dbo.fccPagoFacturaAdelanto pfa
    ON p3l.IdFCCPagoFacturaAdelanto = pfa.IdFCCPagoFacturaAdelanto
LEFT JOIN dbo.tpProformaAdelanto pa
    ON pfa.IdTPProformaAdelanto = pa.IdTPProformaAdelanto
LEFT JOIN dbo.CFDIGenerada cg_faa
    ON pa.IdCFDIGenerada = cg_faa.IdCFDIGenerada
LEFT JOIN dbo.fccPagoCliente fpc
    ON fpc.IdFCCPagoCliente = COALESCE(pfp.IdFCCPagoCliente, pfa.IdFCCPagoCliente)
LEFT JOIN dbo.Cliente c
    ON fpc.IdCliente = c.IdCliente
LEFT JOIN dbo.DatosFacturacionCliente dfc
    ON fpc.IdCliente = dfc.IdCliente AND dfc.Activo = 1
LEFT JOIN dbo.tpPedidoProformaPedido tpp_pp
    ON pp.IdTPProformaPedido = tpp_pp.IdTPProformaPedido AND tpp_pp.Activo = 1
LEFT JOIN dbo.tpPedido tp
    ON tpp_pp.IdTPPedido = tp.IdTPPedido
-- JOINs México
LEFT JOIN dbo.catUsoCFDI ufo
    ON p3l.IdCatUsoCFDI = ufo.IdCatUsoCFDI
LEFT JOIN dbo.catMetodoDePagoCFDI mpc
    ON p3l.IdCatMetodoDePagoCFDI = mpc.IdCatMetodoDePagoCFDI
LEFT JOIN dbo.CFDIGenerada cg_f
    ON p3l.IdCFDIGeneradaFactura = cg_f.IdCFDIGenerada
LEFT JOIN dbo.CFDIGenerada cg_c
    ON p3l.IdCFDIGeneradaComplemento = cg_c.IdCFDIGenerada
-- JOINs Perú
LEFT JOIN dbo.catTipoOperacionSUNAT tos
    ON p3l.IdCatTipoOperacionSUNAT = tos.IdCatTipoOperacionSUNAT
LEFT JOIN dbo.catCondicionesDePago cdp
    ON p3l.IdCatCondicionesDePago = cdp.IdCatCondicionesDePago;
GO

-- Validación
SELECT TOP 5 TipoDocumentoFiscal, EstadoLinea, ClienteRegion,
             TipoOperacionSUNATClave, CondicionPagoClave,
             CPE_Serie, CPE_Correlativo
FROM dbo.vfccDocumentoFiscalCobro;
```

**Criterios de aceptación:**
- La vista compila sin errores.
- `TipoDocumentoFiscal` y `EstadoLinea` se resuelven correctamente vía `INNER JOIN` (no columnas directas).
- `ClienteRegion` discrimina `'MEX'` / `'PER'` correctamente.
- Las columnas `TipoOperacionSUNATClave`, `CondicionPagoClave`, `CPE_Serie`, `CPE_Correlativo` existen y son NULL para líneas México.
- Líneas México existentes siguen siendo consultables sin regresiones.

**Más información de la tarea:**
Ver sección *"ALTER VIEW vfccDocumentoFiscalCobro — Extensión Perú (v2.0)"* en `R16A-RE-FU-029_BD.md` y sección *"Parte A / A4"* en `R16A-RE-FU-029-Back.md`.

**Recursos:**
- `R16A-RE-FU-029_BD.md` — ALTER VIEW vfccDocumentoFiscalCobro
- `R16A-RE-FU-029-Back.md` — Parte A, sección A4

---

## TAREA 5

**[ RE-FU-029 ] [IMP-EXIST-SERVICE] Extender endpoint de timbrado en Timbrado para CPE SUNAT Perú**

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Timbrado — Módulo Perú CPE SUNAT

**Consideraciones previas:**
- ⚠️ **Brecha B1 BLOQUEANTE:** La modalidad de emisión electrónica SUNAT (SEE-SOL, SEE del Contribuyente, SEE-OSE o Facturador SUNAT) está pendiente de definir. No se reutiliza el PAC TurboPac de México. El endpoint puede prepararse en estructura, pero la llamada al servicio SUNAT queda como stub hasta resolver B1.
- El módulo de timbrado México (PAC TurboPac, RE-FU-019) ya existe. El módulo Perú es nuevo y separado, discriminado por región.
- La Tarea 2 debe estar ejecutada (`catTipoCFDI` tiene `FACTURA_CPE` e `IdRegion`).
- En Perú: siempre **1 CPE por solicitud** — sin cascada (no hay Complemento de Pago).
- Por cada timbrado exitoso: INSERT `CFDIGenerada` (`IdCatTipoCFDI=FACTURA_CPE`, `UUID=NULL`, `Serie=F001`, `Folio=Correlativo`), `UPDATE EmpresaFolio GOLPERU` con UPDLOCK atómico, `INSERT TimbradoLog`.
- Retorna a Finanzas: Serie, Correlativo (Folio), FechaEmision, XML CPE, XML CDR de aceptación SUNAT.
- ⚠️ **Brecha B4:** Los datos fiscales SUNAT del producto (`Producto.CodigoSUNAT`, `catUnidad.ClaveSUNAT`, `catAfectacionIGV`) son obligatorios en UBL 2.1 y están pendientes de migrar (RE-FU-020). Sin ellos no es posible generar el XML válido.

**Objetivo general:**
Implementar en ProquifaDotNet.Timbrado el módulo de timbrado Perú que recibe solicitudes de CPE SUNAT desde Finanzas, genera el XML UBL 2.1, invoca el servicio SUNAT (stub hasta resolver B1) e inserta el resultado en `CFDIGenerada` y `EmpresaFolio GOLPERU`.

**Objetivos específicos:**
- Crear `TimbrarCPESunatCommand` + Handler en ProquifaDotNet.Timbrado (separado del Handler México).
- Recibir request: datos emisor GOLPERU, datos receptor (RUC), partidas UBL 2.1, TipoOperacion catálogo 51, CondicionPago, NCs aplicadas (pendiente mecánica B3 — campo opcional).
- Implementar `ISunatTimbraService` con implementación stub (`SimuladorSunatTimbraService`) hasta resolver B1.
- INSERT `CFDIGenerada`: `IdCatTipoCFDI=FACTURA_CPE`, `UUID=NULL`, `Serie=F001`, `Folio=Correlativo`.
- UPDATE `EmpresaFolio GOLPERU SET UltimoFolio+1` con UPDLOCK atómico (mismo patrón México RE-FU-019).
- INSERT `TimbradoLog` con resultado (éxito o fallo CDR SUNAT).
- Retornar Serie, Correlativo, FechaEmision, XML CPE, XML CDR a Finanzas.
- Manejo de errores SUNAT: si SUNAT responde con código de rechazo, la línea permanece sin modificar `CFDIGenerada`; retornar detalle del error a Finanzas.
- Registrar en Serilog: región, serie, correlativo, resultado y tiempo de respuesta.

**Resultado esperado:**
Endpoint de timbrado Perú en ProquifaDotNet.Timbrado con arquitectura lista para conectar a cualquier modalidad SUNAT. El stub permite probar el flujo completo end-to-end sin integración SUNAT real.

**Entregables:**
- `TimbrarCPESunatCommand` + Handler en ProquifaDotNet.Timbrado
- `ISunatTimbraService` + `SimuladorSunatTimbraService` (stub) + (en el futuro) implementación real
- DTO request: `TimbrarCPESunatRequestDto`
- DTO response: `TimbradoCPEResultadoDto` (Serie, Correlativo, FechaEmision, XmlCPE, XmlCDR)
- Pruebas unitarias con stub (timbrado exitoso + rechazo SUNAT + servicio indisponible)

**Criterios de aceptación:**
- Con stub activo: INSERT `CFDIGenerada` con `IdCatTipoCFDI=FACTURA_CPE`, `UUID=NULL`, `Folio=Correlativo`, `Serie=F001`. Correlativo incrementa atómicamente en `EmpresaFolio GOLPERU` con UPDLOCK.
- Rechazo SUNAT simulado: no INSERT `CFDIGenerada`, retorna código de error al caller.
- El flujo México (PAC TurboPac) no se ve afectado — sin regresiones.
- La interfaz `ISunatTimbraService` permite reemplazar el stub por la implementación real sin cambiar el Handler.
- ⚠️ Brechas B1 y B4 documentadas como pendientes en el código y en los logs (`SUNAT_STUB_ACTIVO`).

**Más información de la tarea:**
Ver sección *"Parte C / C1"* en `R16A-RE-FU-029-Back.md` y sección *"EmpresaFolio GOLPERU"* en `R16A-RE-FU-020_BD.md`.

**Recursos:**
- `R16A-RE-FU-029-Back.md` — Parte C, sección C1
- `R16A-RE-FU-020_BD.md` — EmpresaFolio GOLPERU (patrón Serie/Correlativo)

---

## TAREA 6

**[ RE-FU-029 ] [SERV-TRANSACT] Implementar inicialización del Paso 3 Perú (tipo único FACTURA, sin lógica condicional)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 Perú

**Consideraciones previas:**
- Las Tareas 3 y 4 deben estar ejecutadas (columnas Perú en `fccDocumentoFiscalCobro` y vista v2.0 disponibles).
- En Perú la lógica de tipo por línea es significativamente más simple que en México:
  - `fccPagoFacturaPedido` → siempre `FACTURA` (CPE tipo 01). No hay `HayControlados` check.
  - `fccPagoFacturaAdelanto` → ⚠️ **sin documento fiscal en Perú** (Brecha B7 — pendiente definir acción del sistema para este escenario).
- Al reingresar al Paso 3 con líneas existentes, recuperar desde `vfccDocumentoFiscalCobro WHERE ClienteRegion = 'PER'` sin reinicializar.
- Retornar `PuedeRegresarPasos: bool` = false si alguna línea está en `GENERADO` o `ENVIADO`.
- Incluir por línea: catálogos disponibles (`catTipoOperacionSUNAT`, `catCondicionesDePago`) y NCs aplicadas (para visualización — mecánica SUNAT pendiente de Brecha B3).

**Objetivo general:**
Implementar en Finanzas el endpoint de inicialización del Paso 3 Perú que crea las líneas de `fccDocumentoFiscalCobro` con tipo `FACTURA` para todas las proformas de la asociación, y al reingresar recupera el estado existente sin duplicar.

**Objetivos específicos:**
- Implementar `POST /api/validar-cobro/paso3/peru/inicializar` (o rama Perú dentro del endpoint genérico).
- Crear `InicializarPaso3PeruCommand` + Handler.
- Para cada `fccPagoFacturaPedido`: INSERT `fccDocumentoFiscalCobro` con `IdCatTipoDocumentoFiscal=FACTURA`, estado `PENDIENTE`.
- Para `fccPagoFacturaAdelanto`: ⚠️ documentar comportamiento como pendiente (Brecha B7); no crear fila hasta definir.
- Detectar re-entrada: si ya hay filas en `vfccDocumentoFiscalCobro WHERE ClienteRegion='PER' AND IdCliente=@Id AND EstadoLinea != 'ENVIADO'`, retornar estado existente sin reinicializar.
- Calcular `PuedeRegresarPasos = false` si alguna línea está en `GENERADO` o `ENVIADO`.
- DTO `Paso3PeruInicializadoDto`: lista de líneas (tipo, estado, folio proforma, monto, NCs), catálogos Perú y flag `PuedeRegresarPasos`.

**Resultado esperado:**
Endpoint `POST .../paso3/peru/inicializar` que crea las líneas Perú del Paso 3 o recupera las existentes al reingresar, retornando el estado actual con catálogos SUNAT para el frontend.

**Entregables:**
- Endpoint `POST /api/validar-cobro/paso3/peru/inicializar`
- Command + Handler: `InicializarPaso3PeruCommand`
- DTO: `Paso3PeruInicializadoDto` (lista de líneas, catálogos SUNAT, `PuedeRegresarPasos`)
- Pruebas unitarias (tipo único FACTURA, detección re-entrada, flag PuedeRegresarPasos)

**Criterios de aceptación:**
- Líneas `fccPagoFacturaPedido` crean una fila con `TipoDocumentoFiscal=FACTURA`, estado `PENDIENTE`.
- No existe lógica de `HayControlados` — toda proforma genera tipo `FACTURA`.
- Al reingresar con líneas previas: no se duplican; se retorna estado desde `vfccDocumentoFiscalCobro`.
- `PuedeRegresarPasos = false` si alguna línea está en `GENERADO` o `ENVIADO`; `true` si todas en `PENDIENTE`.
- El DTO incluye los catálogos `catTipoOperacionSUNAT` y `catCondicionesDePago` filtrados por región PER.
- ⚠️ Brecha B7 (escenario FAA sin documento fiscal Perú) documentada como pendiente.

**Más información de la tarea:**
Ver sección *"Parte B / B1"* en `R16A-RE-FU-029-Back.md`.

**Recursos:**
- `R16A-RE-FU-029-Back.md` — Parte B, sección B1
- `R16A-RE-FU-028-Back.md` — Parte B, sección B1 (patrón de referencia México)

---

## TAREA 7

**[ RE-FU-029 ] [SERV-SIMPLE-PUT] Implementar auto-guardado Tipo de Operación SUNAT y Condición de Pago por línea**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 Perú

**Consideraciones previas:**
- La Tarea 3 debe estar ejecutada y la Tarea 6 debe haberse ejecutado al menos una vez para que existan líneas en `fccDocumentoFiscalCobro` Perú.
- El usuario puede modificar el Tipo de Operación SUNAT y la Condición de Pago antes de timbrar; Finanzas persiste el cambio inmediatamente.
- A diferencia de México (donde `COMPLEMENTO_PAGO` no persiste Método de Pago), en Perú **ambos campos siempre se persisten** — toda línea Perú es de tipo `FACTURA`.
- Validar que la línea pertenece a un cliente Perú (`ClienteRegion = 'PER'`) y está en estado `PENDIENTE`.

**Objetivo general:**
Implementar en Finanzas el endpoint de auto-guardado que persiste inmediatamente el Tipo de Operación SUNAT y la Condición de Pago al seleccionarlos en la pantalla del Paso 3 Perú.

**Objetivos específicos:**
- Implementar `PUT /api/validar-cobro/paso3/lineas/{idLinea}/configuracion-peru`.
- Crear `ActualizarConfiguracionLineaPaso3PeruCommand` + Handler.
- `UPDATE fccDocumentoFiscalCobro SET IdCatTipoOperacionSUNAT=@Id, IdCatCondicionesDePago=@Id, FechaUltimaActualizacion WHERE IdFCCDocumentoFiscalCobro=@Id`.
- Validar que la línea existe, pertenece al cliente activo y está en estado `PENDIENTE`.
- Retornar el estado actualizado de la línea.

**Resultado esperado:**
Endpoint `PUT .../lineas/{idLinea}/configuracion-peru` que persiste Tipo de Operación y Condición de Pago de forma inmediata.

**Entregables:**
- Endpoint `PUT /api/validar-cobro/paso3/lineas/{idLinea}/configuracion-peru`
- Command + Handler: `ActualizarConfiguracionLineaPaso3PeruCommand`
- DTO request: `ActualizarConfiguracionLineaPeruDto` (IdCatTipoOperacionSUNAT, IdCatCondicionesDePago)
- Pruebas unitarias (actualización correcta, validación estado PENDIENTE, pertenencia al cliente)

**Criterios de aceptación:**
- `UPDATE fccDocumentoFiscalCobro.IdCatTipoOperacionSUNAT` e `IdCatCondicionesDePago` se ejecutan correctamente.
- Retorna error si la línea está en estado `GENERADO` o `ENVIADO` (inmutable).
- Retorna error si la línea no pertenece al cliente activo.
- No modifica columnas México (`IdCatUsoCFDI`, `IdCatMetodoDePagoCFDI`).

**Más información de la tarea:**
Ver sección *"Parte B / B2"* en `R16A-RE-FU-029-Back.md`.

**Recursos:**
- `R16A-RE-FU-029-Back.md` — Parte B, sección B2
- `R16A-RE-FU-028-Back.md` — Parte B, sección B2 (patrón análogo México)

---

## TAREA 8

**[ RE-FU-029 ] [IMP-EXIST-SERVICE] Implementar previsualización PDF CPE por línea**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 Perú

**Consideraciones previas:**
- `FacturaPdfMappingService` Perú y el template `GOLPERU_PER_FAC` ya existen desde RE-FU-020 (`MapearPreviewAsync` para preview sin CDR/sello).
- La Tarea 6 debe haberse ejecutado para que existan líneas Perú en el Paso 3.
- El PDF de previsualización se genera en memoria sin persistir en BD ni MinIO.
- En Perú **toda línea es de tipo FACTURA** — no hay casos especiales por tipo (a diferencia de México donde COMPLEMENTO_PAGO tenía tratamiento diferente).
- ⚠️ Brecha B4: si `Producto.CodigoSUNAT` o `catUnidad.ClaveSUNAT` no existen, el PDF preview no puede generarse. La previsualización completa está condicionada a la resolución de B4.

**Objetivo general:**
Implementar en Finanzas el endpoint de previsualización PDF del Paso 3 Perú que genera el PDF del CPE en memoria usando `FacturaPdfMappingService` Perú (RE-020) y el template `GOLPERU_PER_FAC`.

**Objetivos específicos:**
- Implementar `GET /api/validar-cobro/paso3/lineas/{idLinea}/preview-pdf-peru`.
- Crear `GetPaso3PeruLineaPreviewPdfQuery` + Handler.
- Leer `vfccDocumentoFiscalCobro` para obtener los datos de la línea Perú.
- Invocar `FacturaPdfMappingService.MapearPreviewAsync()` Perú (sin CDR/sello SUNAT).
- `TemplateKey = 'GOLPERU_PER_FAC'` (fijo — única empresa y región Perú en alcance actual).
- Generar PDF en memoria vía DocumentBuilder y retornar como `byte[]` o stream.
- Sin escrituras en BD.

**Resultado esperado:**
Endpoint `GET .../lineas/{idLinea}/preview-pdf-peru` que retorna el PDF de previsualización del CPE en memoria, usando el template existente de RE-020.

**Entregables:**
- Endpoint `GET /api/validar-cobro/paso3/lineas/{idLinea}/preview-pdf-peru`
- Query + Handler: `GetPaso3PeruLineaPreviewPdfQuery`
- Prueba unitaria (PDF sin CDR/sello, template GOLPERU_PER_FAC, no persistencia BD/MinIO)

**Criterios de aceptación:**
- El PDF de previsualización no contiene CDR ni sello SUNAT (sin timbrado).
- El template utilizado es `GOLPERU_PER_FAC` (RE-020).
- No hay persistencia en BD ni MinIO.
- ⚠️ Brecha B4 documentada: si datos SUNAT del producto no existen, el endpoint retorna error controlado informativo.

**Más información de la tarea:**
Ver sección *"Parte B / B3"* en `R16A-RE-FU-029-Back.md` y `FacturaPdfMappingService` Perú en R16A-RE-FU-020.

**Recursos:**
- `R16A-RE-FU-029-Back.md` — Parte B, sección B3
- `R16A-RE-FU-020-Back.md` / `R16A-RE-FU-020-Tareas.md` — FacturaPdfMappingService Perú, GOLPERU_PER_FAC

---

## TAREA 9

**[ RE-FU-029 ] [ALG-COMPLX-LOGIC] Implementar timbrado CPE por línea — escenario único sin cascada**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 Perú

**Consideraciones previas:**
- ⚠️ **Brecha B1 BLOQUEANTE:** Modalidad SUNAT indefinida. La Tarea 5 provee el stub; esta tarea lo consume. El timbrado puede desarrollarse con stub hasta resolver B1.
- ⚠️ **Brecha B4 BLOQUEANTE:** Datos fiscales SUNAT del producto pendientes (RE-020). Sin ellos el XML UBL 2.1 es inválido.
- Las Tareas 5, 7 y 8 deben estar ejecutadas.
- `ApiCallerTimbrado` (HttpClient + Polly) ya existe de RE-FU-019 — reutilizar sin cambios.
- En Perú **solo existe 1 escenario:** 1 CPE por línea. No hay cascada PPD ni Complemento de Pago.
- Post-timbrado exitoso: invocar `FacturaPdfMappingService.MapearAsync()` Perú para PDF definitivo con CDR/sello → subir a MinIO → UPDATE `CFDIGenerada.IdArchivoPdf`.
- El CPE va en `IdCFDIGeneradaFactura` (mismo campo que la Factura en México — `catTipoCFDI.Clave='FACTURA_CPE'` discrimina).
- Manejo de errores SUNAT: si el servicio rechaza, la línea permanece en `PENDIENTE`; Finanzas retorna el detalle del error (código CDR + descripción).

**Objetivo general:**
Implementar en Finanzas el servicio de timbrado del Paso 3 Perú con el escenario único (1 CPE por línea), invocando ProquifaDotNet.Timbrado vía API, persistiendo el PDF definitivo en MinIO y actualizando el estado de la línea a `GENERADO`.

**Objetivos específicos:**
- Implementar `POST /api/validar-cobro/paso3/lineas/{idLinea}/timbrar-peru`.
- Crear `TimbrarLineaPaso3PeruCommand` + Handler.
- Finanzas → Timbrado: request con `TipoCFDI=FACTURA_CPE`, datos emisor GOLPERU, RUC receptor, partidas UBL 2.1, Tipo de Operación, Condición de Pago.
- Recibir de Timbrado: Serie, Correlativo, FechaEmision, XML CPE, XML CDR.
- Invocar `FacturaPdfMappingService.MapearAsync()` con CDR/sello → subir PDF a MinIO → INSERT `Archivo` → UPDATE `CFDIGenerada.IdArchivoPdf`.
- `UPDATE tpProformaPedido SET IdCFDIGenerada = @IdCPE` vía API ProquifaDotNet.
- `UPDATE fccDocumentoFiscalCobro SET EstadoLinea=GENERADO, IdCFDIGeneradaFactura=@IdCPE, FechaGeneracion`.
- Modal de éxito muestra Serie y Correlativo (no UUID — SUNAT no lo genera).
- Ante error SUNAT/CDR: línea permanece en `PENDIENTE`, retornar código de error con descripción.

**Resultado esperado:**
Endpoint `POST .../lineas/{idLinea}/timbrar-peru` que ejecuta el timbrado CPE, persiste el PDF definitivo y actualiza el estado a `GENERADO`.

**Entregables:**
- Endpoint `POST /api/validar-cobro/paso3/lineas/{idLinea}/timbrar-peru`
- Command + Handler: `TimbrarLineaPaso3PeruCommand`
- DTO response: `TimbradoCPEResultadoDto` (Serie, Correlativo, FechaEmision — sin UUID)
- Pruebas unitarias (timbrado exitoso con stub, error SUNAT → línea permanece PENDIENTE, PDF definitivo en MinIO)

**Criterios de aceptación:**
- Timbrado exitoso: `fccDocumentoFiscalCobro.IdCFDIGeneradaFactura` poblado, estado = `GENERADO`, PDF CPE en MinIO.
- `tpProformaPedido.IdCFDIGenerada` actualizado con `@IdCPE`.
- Error SUNAT/CDR: línea permanece en `PENDIENTE`; detalle del error retornado al frontend.
- El modal de éxito recibe Serie y Correlativo (no UUID).
- No existe lógica de cascada (no hay segundo CFDI por línea).
- ⚠️ Brechas B1 y B4 documentadas como pendientes.

**Más información de la tarea:**
Ver sección *"Parte B / B4"* en `R16A-RE-FU-029-Back.md`.

**Recursos:**
- `R16A-RE-FU-029-Back.md` — Parte B, sección B4; Brechas B1 y B4
- `R16A-RE-FU-028-Back.md` — Parte B, sección B4 Escenario A (patrón de referencia sin cascada)

---

## TAREA 10

**[ RE-FU-029 ] [IMP-EXIST-SERVICE] Implementar modal de Envío Perú: generar CDP + despacho Brevo**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 Perú

**Consideraciones previas:**
- La Tarea 9 debe estar ejecutada (línea en estado `GENERADO` para poder enviar).
- El template `GOLPERU_PER_CDP` ya existe en DocumentBuilder — no requiere creación previa.
- Adjuntos Perú: **siempre 3** — PDF CPE + XML CPE + PDF Confirmación de Pedido. No hay variantes por tipo (a diferencia de México con 4 tipos y adjuntos diferentes).
- ⚠️ **Brecha B2:** Formato del asunto y plantilla del cuerpo del correo para Perú pendientes de confirmar con PMO. Propuesta: `"{FolioPedidoInterno} — Factura {Serie}-{Correlativo}"`.
- **Orden de ejecución al confirmar envío:** (1) Generar PDF Confirmación de Pedido vía DocumentBuilder (`GOLPERU_PER_CDP`) → subir a MinIO → obtener `RutaArchivoPDF`; (2) construir adjuntos (PDF CPE + XML CPE + PDF CDP); (3) llamar a Brevo; (4) al éxito: UPDATE estado `ENVIADO` + INSERT CorreoEnviado + INSERT ArchivoCorreoEnviado. El PDF **debe generarse ANTES de llamar a Brevo**.
- Pasar `RutaArchivoPDF` a la Tarea 11 para el `INSERT fccConfirmacionPedido`.
- Los adjuntos de CPE (PDF + XML) se sirven desde MinIO (ya persistidos en Tarea 9).

**Objetivo general:**
Implementar en Finanzas el endpoint del modal de Envío Perú que: (1) genera el PDF CDP vía DocumentBuilder, (2) despacha el correo Brevo con los 3 adjuntos (PDF CPE + XML CPE + PDF CDP), (3) actualiza el estado a `ENVIADO` y (4) deja `RutaArchivoPDF` disponible para la Tarea 12.

**Objetivos específicos:**
- Implementar `POST /api/validar-cobro/paso3/lineas/{idLinea}/enviar-peru`.
- Crear `EnviarLineaPaso3PeruCommand` + Handler.
- **Paso previo — Generar CDP Perú:**
  - `TemplateKey = 'GOLPERU_PER_CDP'` (fijo).
  - Invocar DocumentBuilder con modelo del pedido (folio, cliente, contacto, partidas, empresa GOLPERU, FEE).
  - Subir PDF a MinIO (bucket `confirmaciones`) → obtener `RutaArchivoPDF`.
  - Si la generación falla: retornar error controlado sin despachar el correo.
- **Asunto:** `"{FolioPedidoInterno} — Factura {Serie}-{Correlativo}"` (⚠️ Brecha B2).
- Construir adjuntos: PDF CPE + XML CPE + PDF CDP.
- Llamar a Brevo con destinatarios y adjuntos.
- Al envío exitoso: `UPDATE fccDocumentoFiscalCobro SET EstadoLinea=ENVIADO, FechaEnvio` + INSERT `CorreoEnviado` + INSERT `ArchivoCorreoEnviado` (×3).
- Pasar `RutaArchivoPDF` a la Tarea 12.

**Resultado esperado:**
Endpoint `POST .../lineas/{idLinea}/enviar-peru` que genera el CDP, adjunta los 3 documentos al correo Brevo, actualiza estado a `ENVIADO` y deja la ruta del CDP para T12.

**Entregables:**
- Endpoint `POST /api/validar-cobro/paso3/lineas/{idLinea}/enviar-peru`
- Command + Handler: `EnviarLineaPaso3PeruCommand`
- DTO request: `EnviarLineaPaso3PeruRequestDto` (destinatarios editables)
- DTO response: incluye `RutaArchivoPDF` para T12
- Pruebas unitarias (3 adjuntos correctos, CDP generado antes de Brevo, fallo CDP → no envío, UPDATE ENVIADO)

**Criterios de aceptación:**
- Adjuntos del correo: exactamente 3 (PDF CPE + XML CPE + PDF CDP). No varía por tipo.
- El PDF CDP se genera con `GOLPERU_PER_CDP` antes de llamar a Brevo.
- Si la generación del PDF CDP falla: correo **no** se despacha, retornar error controlado.
- `fccDocumentoFiscalCobro.EstadoLinea = ENVIADO` tras el envío exitoso.
- `INSERT CorreoEnviado` + `INSERT ArchivoCorreoEnviado` correctamente registrados (3 adjuntos).
- ⚠️ Brechas B2 (asunto/plantilla correo) y B7 (contacto no disponible) documentadas como pendientes.

**Más información de la tarea:**
Ver sección *"Parte B / B5"* en `R16A-RE-FU-029-Back.md`.

**Recursos:**
- `R16A-RE-FU-029-Back.md` — Parte B, sección B5
- `R16A-RE-FU-028-Tareas.md` — Tarea 14 (patrón de referencia modal envío México)

---

## TAREA 11

**[ RE-FU-029 ] [SERV-TRANSACT] Implementar acciones post-envío Perú: FEE + registro CDP (sin Legacy)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 Perú (Post-envío)

**Consideraciones previas:**
- Las Tareas 10 y RE-FU-028 Tarea 4 (`fccConfirmacionPedido`) y Tarea 5 (`tpPedido.FechaEstimadaEntrega`) deben estar ejecutadas.
- Se dispara automáticamente al confirmar el envío exitoso de cada línea Perú, recibiendo `RutaArchivoPDF` de la Tarea 10.
- **El PDF CDP lo genera y sube a MinIO la Tarea 10.** Esta tarea solo persiste la referencia con la ruta ya disponible.
- ⚠️ **Brecha B5 COMPARTIDA:** Las reglas de cálculo de la FEE (días hábiles, fórmula, parámetro por empresa) están pendientes con Operaciones PROQUIFA — igual que México. Implementar con regla parametrizable.
- **DIFERENCIA CLAVE vs México:** Esta tarea **NO ejecuta transferencias ETL a Legacy**. No hay E1/E2/E3/E6 para Perú. El post-envío Perú termina en FEE + CDP.
- Si cualquier acción falla: la línea ya está en `ENVIADO`, manejar error individualmente con log sin revertir.

**Objetivo general:**
Implementar en Finanzas las acciones post-envío del Paso 3 Perú: (1) establecer la FEE en `tpPedido` y (2) registrar la trazabilidad del CDP en `fccConfirmacionPedido`. Sin transferencia a Legacy.

**Objetivos específicos:**

**B6.1 — FEE:**
- `UPDATE tpPedido SET FechaEstimadaEntrega = @FEECalculada` vía API ProquifaDotNet.
- Misma lógica de cálculo parametrizable que México (compartida, pendiente Brecha B5).

**B6.2 — Registro trazabilidad CDP:**
- `INSERT fccConfirmacionPedido (IdFCCDocumentoFiscalCobro, IdTPPedido, FolioConfirmacion, RutaArchivoPDF)` usando la `RutaArchivoPDF` recibida de la Tarea 11.
- No invoca DocumentBuilder ni MinIO — el PDF ya fue generado y subido por T11.

**B6.3 — Sin Legacy:**
- Confirmar explícitamente en el código que no se ejecuta ninguna transferencia ETL.
- Comentar: `// Perú no transfiere a Legacy — ver Regla 11 de R16A-RE-FU-029`.

**Resultado esperado:**
Al confirmar envío exitoso de una línea Perú: FEE actualizada y registro CDP insertado en `fccConfirmacionPedido`. Sin ETL.

**Entregables:**
- Servicio `PostEnvioPeruFeeConfirmacionService` (FEE + INSERT CDP)
- `UPDATE tpPedido SET FechaEstimadaEntrega` (regla parametrizable compartida con México)
- `INSERT fccConfirmacionPedido` (RutaArchivoPDF recibida de T11)
- Pruebas unitarias (FEE e INSERT correctos, fallo en una acción no revierte ENVIADO, sin ETL Legacy)

**Criterios de aceptación:**
- `tpPedido.FechaEstimadaEntrega` se actualiza al enviar una línea Perú exitosamente.
- `fccConfirmacionPedido` contiene un nuevo registro con `RutaArchivoPDF` poblada (ruta de T11).
- Esta tarea **no** invoca DocumentBuilder, MinIO ni Legacy.
- Si cualquier acción post-envío falla: línea permanece en `ENVIADO`, error en Serilog, sin reversión.
- ⚠️ Brecha B5 (regla FEE) documentada como pendiente compartida con México.

**Más información de la tarea:**
Ver secciones *"Parte B / B6.1 y B6.2 y B6.3"* en `R16A-RE-FU-029-Back.md`. Comparar con RE-FU-028 Tarea 15 (FEE + CDP) y Tarea 17 (ETL Legacy — que NO existe para Perú).

**Recursos:**
- `R16A-RE-FU-029-Back.md` — Parte B, secciones B6.1, B6.2, B6.3
- `R16A-RE-FU-028-Tareas.md` — Tarea 15 (patrón FEE + CDP de México como referencia)

---

## TAREA 12

**[ RE-FU-029 ] [SERV-SIMPLE-PUT] Implementar cierre automático del wizard Perú al completar todas las líneas enviadas**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 Perú

**Consideraciones previas:**
- La Tarea 11 debe estar ejecutada. El cierre se verifica después de cada envío exitoso de línea.
- Lógica idéntica a México (RE-FU-028 Tarea 16), pero filtrando por `ClienteRegion = 'PER'` en `vfccDocumentoFiscalCobro`.
- El wizard cierra automáticamente cuando **todas** las líneas Perú del cliente están en estado `ENVIADO`.
- El cliente sale del listado de pendientes de Validar Cobro.

**Objetivo general:**
Implementar en Finanzas la lógica de cierre automático del wizard Paso 3 Perú, detectando cuando todas las líneas del cliente están en `ENVIADO` y retornando al listado de Validar Cobro.

**Objetivos específicos:**
- Integrar `VerificarCierrePaso3PeruQuery` + Handler (o reutilizar la lógica de RE-028 Tarea 16 con filtro de región).
- Consulta: `SELECT COUNT(*) FROM vfccDocumentoFiscalCobro WHERE IdCliente=@Id AND ClienteRegion='PER' AND EstadoLinea != 'ENVIADO'`.
- Si `COUNT=0`: `{ cerrado: true }` — frontend navega al listado.
- Si `COUNT>0`: `{ cerrado: false, lineasPendientes: N }`.

**Resultado esperado:**
Tras cada envío Perú exitoso, el frontend detecta si el wizard puede cerrarse. Al detectar `cerrado=true`, navega al listado y el cliente sale de pendientes.

**Entregables:**
- Query + Handler: `VerificarCierrePaso3PeruQuery` (o extensión del Query México con filtro región)
- DTO de response: `EstadoCierrePaso3Dto` (cerrado: bool, lineasPendientes: int) — reutilizable de RE-028 T16
- Pruebas unitarias (todas ENVIADO → cerrado=true; una PENDIENTE → cerrado=false)

**Criterios de aceptación:**
- Cuando todas las líneas Perú del cliente están en `ENVIADO`: `cerrado=true`.
- Cuando al menos una línea no está en `ENVIADO`: `cerrado=false`, `lineasPendientes=N`.
- La consulta filtra por `ClienteRegion='PER'` para no interferir con líneas México.
- El cliente sale del listado de pendientes de Validar Cobro al detectar `cerrado=true`.

**Más información de la tarea:**
Ver sección *"Parte B / B7"* en `R16A-RE-FU-029-Back.md`. Ver R16A-RE-FU-028 Tarea 16 como referencia.

**Recursos:**
- `R16A-RE-FU-029-Back.md` — Parte B, sección B7
- `R16A-RE-FU-028-Tareas.md` — Tarea 16 (patrón de cierre wizard México)
