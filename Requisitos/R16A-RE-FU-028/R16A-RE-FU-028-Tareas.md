# Tareas BackEnd — R16A-RE-FU-028
**Requisito:** Validar Cobro: Paso 3 México — Facturación y Envío
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder

---

> **Orden de ejecución sugerido:** BD catálogos (T1) → BD ALTER CFDIGenerada (T2) → BD tablas principales (T3, T4) → BD ALTER tpPedido (T5) → BD vista (T6) → Timbrado endpoint (T7) → Finanzas: inicialización (T8) → autosave (T9) → NCs en CFDI (T10) → previsualización (T11) → timbrado (T12) → DocumentBuilder plantilla CDP (T13) → Finanzas: envío (T14) → FEE + Confirmación (T15) → ETL Legacy (T17) → cierre wizard (T16).
>
> **Dependencias externas:** RE-FU-026 completo (asociación Paso 2 cerrada con `fccPagoFacturaPedido` y `fccPagoFacturaAdelanto`). RE-FU-019 completo (`CFDIGenerada`, `EmpresaFolio`, `ApiCallerStamping`). RE-FU-021 completo (`InvoicePdfMappingService`, `PersistInvoicePdfService`, templates `*_MEX_FAC`).
>
> **Brechas bloqueantes activas:** B1 (relación SAT tipo 07), B3 (mecanismo ETL Legacy), B4 (FEE: granularidad y reglas), B6 (política fallo cascada PPD). Las tareas T5, T12, T15, T17 dependen de resolverlas antes de implementar. Ver `R16A-RE-FU-028_BD.md` y `R16A-RE-FU-028-Back.md` — sección Brechas.
>
> **Nota — `*_MEX_COP` y ETL Complemento/NC:** El diseño de la plantilla Complemento de Pago y sus transferencias a Legacy corresponden a R16A-RE-FU-030 y R16A-RE-FU-032/034 respectivamente. Este requisito solo implementa las plantillas `*_MEX_CDP` (Confirmación de Pedido) y los ETL E1/E2/E3/E6 (Buzón, Proforma, Factura, PDF Factura).

---

## Resumen de tareas

| #   | Clave                 | Título simple                                                                                       | Tipo            | Aplicativo              |
| --- | --------------------- | --------------------------------------------------------------------------------------------------- | --------------- | ----------------------- |
| 1   | CREATE-TABL-CH        | Crear catálogos catTipoDocumentoFiscal, catDocumentoFiscalCobroEstado y catTipoCFDI con DML inicial | BD              | ProquifaDotNet          |
| 2   | UPDATE-TABL-M         | Agregar IdCatTipoCFDI (FK) e IdCFDIRelacionado a CFDIGenerada + script normalización FAA            | BD              | ProquifaDotNet          |
| 3   | CREATE-TABL-M         | Crear tabla fccDocumentoFiscalCobro (control de estado Paso 3, CHECK origen exclusivo)              | BD              | ProquifaDotNet          |
| 4   | CREATE-TABL-M         | Crear tabla fccConfirmacionPedido (Confirmación de Pedido Prepago México)                           | BD              | ProquifaDotNet          |
| 5   | UPDATE-TABL-CH        | Agregar FechaEstimadaEntrega a tpPedido (condicional — pendiente resolución Brecha B4)              | BD              | ProquifaDotNet          |
| 6   | CREATE-SCRIPT-CONTROL | Crear vista vfccDocumentoFiscalCobro (consolidación estado Paso 3 por cliente)                      | BD              | ProquifaDotNet          |
| 7   | IMP-EXIST-SERVICE     | Extender endpoint de timbrado en Timbrado para soporte de tipo CFDI y cascada PPD                   | Back            | ProquifaDotNet.Timbrado |
| 8   | SERV-TRANSACT         | Implementar inicialización del Paso 3 — creación de líneas fccDocumentoFiscalCobro                  | Back            | ProquifaDotNet.Finanzas |
| 9   | SERV-SIMPLE-PUT       | Implementar auto-guardado de Uso CFDI y Método de Pago por línea                                    | Back            | ProquifaDotNet.Finanzas |
| 10  | IMP-EXIST-SERVICE     | Implementar resolución de Notas de Crédito aplicadas en nodo CFDIRelacionados al timbrar            | Back            | ProquifaDotNet.Finanzas |
| 11  | IMP-EXIST-SERVICE     | Implementar previsualización PDF por línea: FACTURA y FACTURA_ANTICIPO                              | Back            | ProquifaDotNet.Finanzas |
| 12  | ALG-COMPLX-LOGIC      | Implementar timbrado por línea: 4 escenarios (PUE, PPD+cascada, Anticipo, Complemento FAA)          | Back            | ProquifaDotNet.Finanzas |
| 13  | CREATE-PDF            | Plantilla PDF Confirmación de Pedido Prepago México — 4 variantes (GOL/MUN/PRO/PQF_MEX_CDP)         | DocumentBuilder | DocumentBuilder         |
| 14  | IMP-EXIST-SERVICE     | Implementar modal de Envío: generar Confirmación de Pedido + despacho vía ProquifaDotNet.EnvioCorreo (adjuntos CFDI + CDP) | Back            | ProquifaDotNet.Finanzas |
| 15  | SERV-TRANSACT         | Implementar acciones post-envío: FEE + registro trazabilidad Confirmación de Pedido                 | Back            | ProquifaDotNet.Finanzas |
| 16  | SERV-SIMPLE-PUT       | Implementar cierre automático del wizard al completar todas las líneas enviadas                     | Back            | ProquifaDotNet.Finanzas |
| 17  | QUERY-G               | Análisis ETL Legacy — Mapeo de datos E1/E2/E3/E6 y resolución Brecha B3                             | Análisis        | ProquifaDotNet.Finanzas |
| 18  | SERV-COMPLEX-TRANSACT | Implementar builders de payloads ETL E1/E2/E3/E6 e interfaz IEtlLegacyTransferService          | Back            | ProquifaDotNet.Finanzas |
| 19  | SERV-TRANSACT         | Canal ETL Legacy definitivo E1/E2/E3/E6 — integración, pruebas y documentación                      | Back            | ProquifaDotNet.Finanzas |

---

## TAREA 1

**[ RE-FU-028 ] [CREATE-TABL-CH] Crear catálogos catTipoDocumentoFiscal, catDocumentoFiscalCobroEstado y catTipoCFDI con DML inicial**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3

**Consideraciones previas:**
- Los tres catálogos son nuevos. No existen en BD.
- Patrón estándar del proyecto: `uniqueidentifier PK` con `NEWID()`, campo `Clave varchar UNIQUE`, `Descripcion nvarchar`, `Activo bit DEFAULT(1)`, `FechaRegistro datetime2(7) DEFAULT SYSUTCDATETIME()`.
- `catTipoDocumentoFiscal` define el tipo de documento a emitir por línea del Paso 3: `FACTURA`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO`.
- `catDocumentoFiscalCobroEstado` define el ciclo de vida de cada línea: `PENDIENTE`, `GENERADO`, `ENVIADO`.
- `catTipoCFDI` discrimina el comprobante almacenado en `CFDIGenerada`: `FACTURA_PPD`, `FACTURA_PUE`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO`.
- Son **prerrequisito** de la Tarea 3 (`fccDocumentoFiscalCobro` referencia las tres vía FK) y de la Tarea 2 (ALTER `CFDIGenerada` referencia `catTipoCFDI`).
- Se pueden ejecutar los tres DDL en el mismo script o en scripts separados — no tienen dependencia entre sí.

**Objetivo general:**
Crear los tres catálogos del Paso 3 en ProquifaDotNet con sus datos iniciales, habilitando las referencias de tipo y estado que consumirán `fccDocumentoFiscalCobro` y `CFDIGenerada`.

**Objetivos específicos:**
- Ejecutar `CREATE TABLE catTipoDocumentoFiscal` + `INSERT` con claves: `FACTURA`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO`.
- Ejecutar `CREATE TABLE catDocumentoFiscalCobroEstado` + `INSERT` con claves: `PENDIENTE`, `GENERADO`, `ENVIADO`.
- Ejecutar `CREATE TABLE catTipoCFDI` + `INSERT` con claves: `FACTURA_PPD`, `FACTURA_PUE`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO`.
- Verificar que PK, UNIQUE y DEFAULT quedan correctamente definidos en los tres catálogos.
- Verificar que los registros iniciales son consultables y que `Activo=1` en todos.

**Resultado esperado:**
Los tres catálogos existen en ProquifaDotNet con sus datos iniciales completos, listos para ser referenciados por `fccDocumentoFiscalCobro` y `CFDIGenerada`.

**Entregables:**
- Script DDL + DML: `CREATE TABLE catTipoDocumentoFiscal` + `INSERT` datos iniciales
- Script DDL + DML: `CREATE TABLE catDocumentoFiscalCobroEstado` + `INSERT` datos iniciales
- Script DDL + DML: `CREATE TABLE catTipoCFDI` + `INSERT` datos iniciales
- Script de validación (`SELECT` con conteo y estructura de los tres catálogos)

**Scripts:**

```sql
-- =========================================================
-- 1. catTipoDocumentoFiscal
-- =========================================================
CREATE TABLE [dbo].[catTipoDocumentoFiscal](
    [IdCatTipoDocumentoFiscal]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catTipoDocumentoFiscal_Id]    DEFAULT (NEWID()),
    [Clave]                     varchar(30)      NOT NULL,
        -- 'FACTURA'          -> CFDI Ingreso PUE o PPD (proforma sin controlados)
        -- 'FACTURA_ANTICIPO' -> CFDI Ingreso rel. 07 SAT (proforma con controlados)
        -- 'COMPLEMENTO_PAGO' -> CFDI Pagos 2.0 (FAA existente con cobro asociado)
    [Descripcion]               nvarchar(150)    NOT NULL,
    [Activo]                    bit              NOT NULL
        CONSTRAINT [DF_catTipoDocumentoFiscal_Activo] DEFAULT (1),
    [FechaRegistro]             datetime2(7)     NOT NULL
        CONSTRAINT [DF_catTipoDocumentoFiscal_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catTipoDocumentoFiscal]
        PRIMARY KEY CLUSTERED ([IdCatTipoDocumentoFiscal]),
    CONSTRAINT [UQ_catTipoDocumentoFiscal_Clave]
        UNIQUE ([Clave])
);
GO
INSERT INTO dbo.catTipoDocumentoFiscal (Clave, Descripcion) VALUES
    ('FACTURA',          'Factura — CFDI Ingreso (proforma sin productos controlados)'),
    ('FACTURA_ANTICIPO', 'Factura Anticipo — CFDI Ingreso rel. 07 SAT (proforma con productos controlados)'),
    ('COMPLEMENTO_PAGO', 'Complemento de Pago — CFDI Pagos 2.0 (Factura por Adelanto existente)');
GO

-- =========================================================
-- 2. catDocumentoFiscalCobroEstado
-- =========================================================
CREATE TABLE [dbo].[catDocumentoFiscalCobroEstado](
    [IdCatDocumentoFiscalCobroEstado]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catDocumentoFiscalCobroEstado_Id]    DEFAULT (NEWID()),
    [Clave]                            varchar(20)      NOT NULL,
        -- 'PENDIENTE' -> Estado inicial; aún no se ha timbrado ni enviado
        -- 'GENERADO'  -> CFDIs timbrados exitosamente; pendiente de envío al cliente
        -- 'ENVIADO'   -> Enviado al cliente; línea cerrada operativamente
    [Descripcion]                      nvarchar(150)    NOT NULL,
    [Activo]                           bit              NOT NULL
        CONSTRAINT [DF_catDocumentoFiscalCobroEstado_Activo] DEFAULT (1),
    [FechaRegistro]                    datetime2(7)     NOT NULL
        CONSTRAINT [DF_catDocumentoFiscalCobroEstado_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catDocumentoFiscalCobroEstado]
        PRIMARY KEY CLUSTERED ([IdCatDocumentoFiscalCobroEstado]),
    CONSTRAINT [UQ_catDocumentoFiscalCobroEstado_Clave]
        UNIQUE ([Clave])
);
GO
INSERT INTO dbo.catDocumentoFiscalCobroEstado (Clave, Descripcion) VALUES
    ('PENDIENTE', 'Pendiente — línea creada, aún no timbrada ni enviada'),
    ('GENERADO',  'Generado — CFDIs timbrados exitosamente, pendiente de envío al cliente'),
    ('ENVIADO',   'Enviado — documentos enviados al cliente, línea cerrada operativamente');
GO

-- =========================================================
-- 3. catTipoCFDI
-- =========================================================
CREATE TABLE [dbo].[catTipoCFDI](
    [IdCatTipoCFDI]     uniqueidentifier NOT NULL
        CONSTRAINT [DF_catTipoCFDI_Id]       DEFAULT (NEWID()),
    [Clave]             varchar(20)      NOT NULL,
        -- 'FACTURA_PPD'       -> Factura generada con método PPD
        -- 'FACTURA_PUE'       -> Factura generada con método PUE
        -- 'FACTURA_ANTICIPO'  -> Factura Anticipo tipo relación 07 SAT (controlados)
        -- 'COMPLEMENTO_PAGO'  -> CFDI Pagos 2.0
    [Descripcion]       nvarchar(150)    NOT NULL,
    [Activo]            bit              NOT NULL
        CONSTRAINT [DF_catTipoCFDI_Activo]   DEFAULT (1),
    [FechaRegistro]     datetime2(7)     NOT NULL
        CONSTRAINT [DF_catTipoCFDI_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catTipoCFDI]
        PRIMARY KEY CLUSTERED ([IdCatTipoCFDI]),
    CONSTRAINT [UQ_catTipoCFDI_Clave]
        UNIQUE ([Clave])
);
GO
INSERT INTO dbo.catTipoCFDI (Clave, Descripcion) VALUES
    ('FACTURA_PPD',      'Factura — CFDI Ingreso con método de pago PPD (Pago en parcialidades o diferido)'),
    ('FACTURA_PUE',      'Factura — CFDI Ingreso con método de pago PUE (Pago en una sola exhibición)'),
    ('FACTURA_ANTICIPO', 'Factura Anticipo — CFDI Ingreso con tipo de relación 07 SAT (productos controlados)'),
    ('COMPLEMENTO_PAGO', 'Complemento de Pago — CFDI Pagos 2.0');
GO

-- =========================================================
-- Validación
-- =========================================================
SELECT 'catTipoDocumentoFiscal'        AS Catalogo, COUNT(*) AS Registros FROM dbo.catTipoDocumentoFiscal
UNION ALL
SELECT 'catDocumentoFiscalCobroEstado' AS Catalogo, COUNT(*) AS Registros FROM dbo.catDocumentoFiscalCobroEstado
UNION ALL
SELECT 'catTipoCFDI'                   AS Catalogo, COUNT(*) AS Registros FROM dbo.catTipoCFDI;
```

**Criterios de aceptación:**
- Los tres catálogos existen con la estructura definida en `R16A-RE-FU-028_BD.md`.
- `catTipoDocumentoFiscal` contiene exactamente 3 registros: `FACTURA`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO`.
- `catDocumentoFiscalCobroEstado` contiene exactamente 3 registros: `PENDIENTE`, `GENERADO`, `ENVIADO`.
- `catTipoCFDI` contiene exactamente 4 registros: `FACTURA_PPD`, `FACTURA_PUE`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO`.
- Todos los registros tienen `Activo=1`.
- Las constraints de PK y UNIQUE están activas en los tres catálogos.

**Más información de la tarea:**
Ver secciones *"Catálogo Nuevo: catTipoDocumentoFiscal"*, *"Catálogo Nuevo: catDocumentoFiscalCobroEstado"* y *"Catálogo Nuevo: catTipoCFDI"* en `R16A-RE-FU-028_BD.md`.

**Recursos:**
- `R16A-RE-FU-028_BD.md` — DDL + DML de los tres catálogos
- `R16A-RE-FU-028-Back.md` — Parte A, secciones A1, A2, A3

---

## TAREA 2

**[ RE-FU-028 ] [UPDATE-TABL-M] Agregar IdCatTipoCFDI (FK) e IdCFDIRelacionado a CFDIGenerada + script de normalización FAA**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — CFDIGenerada

**Consideraciones previas:**
- `CFDIGenerada` fue creada en RE-FU-019. Esta tarea la extiende sin modificar columnas existentes.
- `IdCatTipoCFDI uniqueidentifier NULL` + FK a `catTipoCFDI`: discrimina el tipo de CFDI almacenado. NULL en registros históricos (FAAs pre-RE-028).
- `IdCFDIRelacionado uniqueidentifier NULL`: auto-referencia blanda — UUID de la Factura PPD a la que un Complemento complementa. **No se declara FK hard** para evitar conflictos de orden de inserción en cascada PPD.
- Incluir script de normalización: `UPDATE CFDIGenerada SET IdCatTipoCFDI = (SELECT Id FROM catTipoCFDI WHERE Clave='FACTURA_ANTICIPO')` para los registros de FAA generados en RE-FU-019.
- La Tarea 1 debe estar ejecutada (`catTipoCFDI` debe existir antes del ALTER con FK).
- Verificar que ningún SP, vista ni trigger dependiente de `CFDIGenerada` se rompe tras el ALTER.
- Es **prerrequisito** de la Tarea 7 (el endpoint de Timbrado inserta `IdCatTipoCFDI` al timbrar).

**Objetivo general:**
Extender `CFDIGenerada` con el discriminador de tipo de CFDI (`IdCatTipoCFDI`) y la auto-referencia blanda (`IdCFDIRelacionado`) para soportar el timbrado de múltiples tipos de CFDI en el Paso 3 y la trazabilidad del Complemento respecto a su Factura PPD origen.

**Objetivos específicos:**
- `ALTER TABLE dbo.CFDIGenerada ADD IdCatTipoCFDI uniqueidentifier NULL CONSTRAINT [FK_CFDIGenerada_TipoCFDI] FOREIGN KEY REFERENCES dbo.catTipoCFDI([IdCatTipoCFDI])`.
- `ALTER TABLE dbo.CFDIGenerada ADD IdCFDIRelacionado uniqueidentifier NULL` — sin FK hard.
- Ejecutar script de normalización: poblar `IdCatTipoCFDI = FACTURA_ANTICIPO` en todos los registros existentes de FAA (identificables por el origen del registro en RE-FU-019).
- Verificar que SPs, vistas y triggers dependientes de `CFDIGenerada` no presentan errores.
- Verificar que registros históricos tienen `IdCatTipoCFDI = FACTURA_ANTICIPO` y `IdCFDIRelacionado = NULL`.

**Resultado esperado:**
`CFDIGenerada` con `IdCatTipoCFDI` (FK activo) e `IdCFDIRelacionado` disponibles, con registros históricos de FAA normalizados con el tipo correcto.

**Entregables:**
- Script DDL: `ALTER TABLE CFDIGenerada ADD IdCatTipoCFDI` (con FK)
- Script DDL: `ALTER TABLE CFDIGenerada ADD IdCFDIRelacionado` (sin FK)
- Script DML: normalización `UPDATE CFDIGenerada SET IdCatTipoCFDI = FACTURA_ANTICIPO` (FAAs existentes)
- Script de validación (estructura + registros históricos normalizados)

**Scripts:**

```sql
-- Prerequisito: catTipoCFDI debe existir (Tarea 1)
-- Ejecutar en ProquifaDotNet

ALTER TABLE dbo.CFDIGenerada
    ADD IdCatTipoCFDI uniqueidentifier NULL;
        -- FK a catTipoCFDI: FACTURA_PPD | FACTURA_PUE | FACTURA_ANTICIPO | COMPLEMENTO_PAGO
        -- NULL en registros previos (FAA generadas antes de RE-FU-028); normalizar con UPDATE posterior
GO

ALTER TABLE dbo.CFDIGenerada
    ADD CONSTRAINT [FK_CFDIGenerada_TipoCFDI]
        FOREIGN KEY ([IdCatTipoCFDI])
        REFERENCES dbo.catTipoCFDI([IdCatTipoCFDI]);
GO

ALTER TABLE dbo.CFDIGenerada
    ADD IdCFDIRelacionado uniqueidentifier NULL;
        -- Para IdCatTipoCFDI -> 'COMPLEMENTO_PAGO': referencia al IdCFDIGenerada de la Factura PPD relacionada
        -- Para IdCatTipoCFDI -> 'FACTURA_ANTICIPO': NULL (la relación tipo 07 se arma en XML al timbrar)
        -- NULL para FACTURA_PPD y FACTURA_PUE
        -- FK blanda (sin FOREIGN KEY de BD): la integridad se garantiza a nivel de servicio
GO

-- Script de normalización de registros existentes (FAA generadas en RE-FU-019)
-- NOTA: validar si las FAA previas son PPD o PUE antes de ejecutar; ajustar el UPDATE según corresponda.
UPDATE dbo.CFDIGenerada
    SET IdCatTipoCFDI = (SELECT IdCatTipoCFDI FROM dbo.catTipoCFDI WHERE Clave = 'FACTURA_PPD')
WHERE IdCatTipoCFDI IS NULL;
GO

-- Validación
SELECT
    c.name              AS Columna,
    t.name              AS Tipo,
    c.is_nullable       AS EsNullable
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.CFDIGenerada')
  AND c.name IN ('IdCatTipoCFDI','IdCFDIRelacionado');

SELECT COUNT(*) AS RegistrosNormalizados
FROM dbo.CFDIGenerada
WHERE IdCatTipoCFDI IS NOT NULL;
```

**Criterios de aceptación:**
- `CFDIGenerada.IdCatTipoCFDI uniqueidentifier NULL` existe con FK activo hacia `catTipoCFDI`.
- `CFDIGenerada.IdCFDIRelacionado uniqueidentifier NULL` existe sin FK declarado.
- Todos los registros existentes de FAA tienen `IdCatTipoCFDI` poblado con el UUID de `FACTURA_ANTICIPO`.
- Registros anteriores que no sean FAA tienen `IdCatTipoCFDI = NULL` (si los hay).
- Ningún SP, vista ni trigger presenta errores tras el ALTER.

**Más información de la tarea:**
Ver sección *"ALTER TABLE CFDIGenerada"* en `R16A-RE-FU-028_BD.md` y sección *"Parte A / A6"* en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028_BD.md` — ALTER TABLE CFDIGenerada (IdCatTipoCFDI + IdCFDIRelacionado + normalización)
- `R16A-RE-FU-028-Back.md` — Parte A, sección A6

---

## TAREA 3

**[ RE-FU-028 ] [CREATE-TABL-M] Crear tabla fccDocumentoFiscalCobro (control de estado Paso 3, CHECK origen exclusivo)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3

**Consideraciones previas:**
- Tabla nueva. Es la entidad central del Paso 3: una fila por cada documento fiscal a generar, derivada de la asociación del Paso 2.
- La Tarea 1 debe estar ejecutada (FK a `catTipoDocumentoFiscal` y `catDocumentoFiscalCobroEstado`).
- **CHECK CONSTRAINT exclusivo:** `IdFCCPagoFacturaPedido` y `IdFCCPagoFacturaAdelanto` son mutuamente excluyentes — exactamente uno debe ser NOT NULL y el otro NULL. Implementar con `CK_fccDocumentoFiscalCobro_OrigenExclusivo`.
- FK hacia `fccPagoFacturaPedido` (proformas normales Paso 2) y `fccPagoFacturaAdelanto` (FAA Paso 2) — ambas existen desde RE-FU-026.
- `IdCFDIGeneradaFactura` y `IdCFDIGeneradaComplemento` son nullable: se poblan al timbrar exitosamente.
- El ciclo de vida de la línea sigue el orden: `INSERT (PENDIENTE)` → `UPDATE (GENERADO)` → `UPDATE (ENVIADO)`.
- Es **prerrequisito** de la Tarea 6 (vista), Tarea 8 (inicialización), Tarea 9 (autosave), Tarea 12 (timbrado), Tarea 14 (envío), Tarea 16 (cierre).

**Objetivo general:**
Crear la tabla `fccDocumentoFiscalCobro` para controlar el estado de cada documento fiscal a generar en el Paso 3, con FK exclusiva al origen de la asociación del Paso 2 y trazabilidad completa de los CFDIs generados.

**Objetivos específicos:**
- Ejecutar `CREATE TABLE fccDocumentoFiscalCobro` con columnas: `IdFCCDocumentoFiscalCobro` (PK NEWID), `IdFCCPagoFacturaPedido uniqueidentifier NULL` (FK), `IdFCCPagoFacturaAdelanto uniqueidentifier NULL` (FK), `IdCatTipoDocumentoFiscal uniqueidentifier NOT NULL` (FK), `IdCatDocumentoFiscalCobroEstado uniqueidentifier NOT NULL` (FK), `IdCatUsoCFDI uniqueidentifier NULL`, `IdCatMetodoDePagoCFDI uniqueidentifier NULL`, `IdCFDIGeneradaFactura uniqueidentifier NULL`, `IdCFDIGeneradaComplemento uniqueidentifier NULL`, `FechaGeneracion datetime2(7) NULL`, `FechaEnvio datetime2(7) NULL`, `Activo bit DEFAULT(1)`, `FechaRegistro datetime2(7) DEFAULT SYSUTCDATETIME()`, `FechaUltimaActualizacion datetime2(7) DEFAULT SYSUTCDATETIME()`.
- Implementar `CONSTRAINT [CK_fccDocumentoFiscalCobro_OrigenExclusivo] CHECK (([IdFCCPagoFacturaPedido] IS NOT NULL AND [IdFCCPagoFacturaAdelanto] IS NULL) OR ([IdFCCPagoFacturaPedido] IS NULL AND [IdFCCPagoFacturaAdelanto] IS NOT NULL))`.
- Crear índices filtrados sobre las dos FKs nullable (patrón de RE-FU-028_BD).
- Verificar que las FKs hacia `fccPagoFacturaPedido`, `fccPagoFacturaAdelanto`, `catTipoDocumentoFiscal` y `catDocumentoFiscalCobroEstado` están activas.
- Verificar que el CHECK CONSTRAINT rechaza correctamente filas con ambas FKs NULL o ambas NOT NULL.

**Resultado esperado:**
Tabla `fccDocumentoFiscalCobro` creada en ProquifaDotNet con el CHECK CONSTRAINT exclusivo activo, lista para recibir las líneas del Paso 3.

**Entregables:**
- Script DDL: `CREATE TABLE fccDocumentoFiscalCobro` con todas las constraints
- Script de validación (estructura + prueba del CHECK CONSTRAINT con casos válidos e inválidos)

**Scripts:**

```sql
-- Prerequisito: catTipoDocumentoFiscal, catDocumentoFiscalCobroEstado (Tarea 1)
-- Prerequisito: fccPagoFacturaPedido, fccPagoFacturaAdelanto (RE-FU-026)
CREATE TABLE [dbo].[fccDocumentoFiscalCobro](
    [IdFCCDocumentoFiscalCobro]              uniqueidentifier NOT NULL
        CONSTRAINT [DF_fccDocumentoFiscalCobro_Id]             DEFAULT (NEWID()),
    -- Origen: exactamente UNO de los dos siguientes debe ser NOT NULL (CK garantiza exclusividad)
    [IdFCCPagoFacturaPedido]                 uniqueidentifier NULL,
        -- NOT NULL cuando el origen es proforma normal o con controlados (RE-FU-026)
    [IdFCCPagoFacturaAdelanto]               uniqueidentifier NULL,
        -- NOT NULL cuando el origen es FAA existente con cobro asociado (RE-FU-026)
    -- Tipo de documento fiscal a generar (determinado al crear la línea)
    [IdCatTipoDocumentoFiscal]               uniqueidentifier NOT NULL,
        -- FK a catTipoDocumentoFiscal: FACTURA | FACTURA_ANTICIPO | COMPLEMENTO_PAGO
    [IdCatDocumentoFiscalCobroEstado]        uniqueidentifier NOT NULL,
        -- FK a catDocumentoFiscalCobroEstado: PENDIENTE | GENERADO | ENVIADO
    [IdCatUsoCFDI]                           uniqueidentifier NULL,   -- FK catUsoCFDI (catálogo SAT c_UsoCFDI)
    [IdCatMetodoDePagoCFDI]                  uniqueidentifier NULL,   -- FK catMetodoDePagoCFDI; NULL para COMPLEMENTO_PAGO
    [IdCFDIGeneradaFactura]                  uniqueidentifier NULL,   -- FK CFDIGenerada: Factura/Anticipo/Complemento FAA timbrado
    [IdCFDIGeneradaComplemento]              uniqueidentifier NULL,   -- FK CFDIGenerada: Complemento cascada PPD (solo MetodoPago=PPD)
    [FechaGeneracion]                        datetime2(7)     NULL,
    [FechaEnvio]                             datetime2(7)     NULL,
    [Activo]                                 bit              NOT NULL
        CONSTRAINT [DF_fccDocumentoFiscalCobro_Activo]         DEFAULT (1),
    [FechaRegistro]                          datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccDocumentoFiscalCobro_FechaReg]       DEFAULT (SYSUTCDATETIME()),
    [FechaUltimaActualizacion]               datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccDocumentoFiscalCobro_FechaUpd]       DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_fccDocumentoFiscalCobro]
        PRIMARY KEY CLUSTERED ([IdFCCDocumentoFiscalCobro]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_PagoFacturaPedido]
        FOREIGN KEY ([IdFCCPagoFacturaPedido])
        REFERENCES dbo.fccPagoFacturaPedido([IdFCCPagoFacturaPedido]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_PagoFacturaAdelanto]
        FOREIGN KEY ([IdFCCPagoFacturaAdelanto])
        REFERENCES dbo.fccPagoFacturaAdelanto([IdFCCPagoFacturaAdelanto]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_TipoDocumentoFiscal]
        FOREIGN KEY ([IdCatTipoDocumentoFiscal])
        REFERENCES dbo.catTipoDocumentoFiscal([IdCatTipoDocumentoFiscal]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_Estado]
        FOREIGN KEY ([IdCatDocumentoFiscalCobroEstado])
        REFERENCES dbo.catDocumentoFiscalCobroEstado([IdCatDocumentoFiscalCobroEstado]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_UsoCFDI]
        FOREIGN KEY ([IdCatUsoCFDI])
        REFERENCES dbo.catUsoCFDI([IdCatUsoCFDI]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_MetodoPago]
        FOREIGN KEY ([IdCatMetodoDePagoCFDI])
        REFERENCES dbo.catMetodoDePagoCFDI([IdCatMetodoDePagoCFDI]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_CFDIFactura]
        FOREIGN KEY ([IdCFDIGeneradaFactura])
        REFERENCES dbo.CFDIGenerada([IdCFDIGenerada]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_CFDIComplemento]
        FOREIGN KEY ([IdCFDIGeneradaComplemento])
        REFERENCES dbo.CFDIGenerada([IdCFDIGenerada]),
    CONSTRAINT [CK_fccDocumentoFiscalCobro_OrigenExclusivo]
        CHECK (
            (IdFCCPagoFacturaPedido IS NOT NULL AND IdFCCPagoFacturaAdelanto IS NULL)
            OR
            (IdFCCPagoFacturaPedido IS NULL  AND IdFCCPagoFacturaAdelanto IS NOT NULL)
        )
);
GO
CREATE INDEX [IX_fccDocumentoFiscalCobro_PagoFacturaPedido]
    ON dbo.fccDocumentoFiscalCobro ([IdFCCPagoFacturaPedido])
    WHERE [IdFCCPagoFacturaPedido] IS NOT NULL;

CREATE INDEX [IX_fccDocumentoFiscalCobro_PagoFacturaAdelanto]
    ON dbo.fccDocumentoFiscalCobro ([IdFCCPagoFacturaAdelanto])
    WHERE [IdFCCPagoFacturaAdelanto] IS NOT NULL;
GO

-- Validación CHECK CONSTRAINT (casos inválidos deben lanzar error)
-- Caso inválido 1: ambas FKs NULL
-- INSERT INTO dbo.fccDocumentoFiscalCobro (IdCatTipoDocumentoFiscal, IdCatDocumentoFiscalCobroEstado)
--   VALUES (...) -- debe fallar

-- Caso inválido 2: ambas FKs NOT NULL
-- INSERT INTO dbo.fccDocumentoFiscalCobro (IdFCCPagoFacturaPedido, IdFCCPagoFacturaAdelanto, ...)
--   VALUES (..., ..., ...) -- debe fallar
```

**Criterios de aceptación:**
- La tabla existe con la estructura definida en `R16A-RE-FU-028_BD.md`.
- `CK_fccDocumentoFiscalCobro_OrigenExclusivo` rechaza filas con ambas FKs NULL.
- `CK_fccDocumentoFiscalCobro_OrigenExclusivo` rechaza filas con ambas FKs NOT NULL.
- FKs activas hacia `fccPagoFacturaPedido`, `fccPagoFacturaAdelanto`, `catTipoDocumentoFiscal` y `catDocumentoFiscalCobroEstado`.
- Tabla vacía al crear.

**Más información de la tarea:**
Ver sección *"Tabla Nueva: fccDocumentoFiscalCobro"* en `R16A-RE-FU-028_BD.md` y sección *"Parte A / A4"* en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028_BD.md` — DDL fccDocumentoFiscalCobro
- `R16A-RE-FU-028-Back.md` — Parte A, sección A4

---

## TAREA 4

**[ RE-FU-028 ] [CREATE-TABL-M] Crear tabla fccConfirmacionPedido (Confirmación de Pedido Prepago México)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3

**Consideraciones previas:**
- Tabla nueva. Almacena las Confirmaciones de Pedido generadas al enviar cada línea del Paso 3 para pedidos Prepago México.
- El PDF de la Confirmación se genera vía DocumentBuilder (plantilla `*_MEX_CDP`, Tarea 13) y se almacena en MinIO; la ruta o referencia se guarda en esta tabla.
- FK hacia `tpPedido` (pedido al que corresponde la confirmación).
- El campo `FolioConfirmacion` almacena el folio asignado (formato pendiente de confirmar con PMO — Brecha B5).
- Es **prerrequisito** de la Tarea 15 (post-envío inserta en esta tabla al generar la Confirmación de Pedido).

**Objetivo general:**
Crear la tabla `fccConfirmacionPedido` para registrar las Confirmaciones de Pedido Prepago generadas en el Paso 3, con referencia al PDF almacenado en MinIO y al pedido origen.

**Objetivos específicos:**
- Ejecutar `CREATE TABLE fccConfirmacionPedido` con columnas: `IdFCCConfirmacionPedido` (PK NEWID), `IdTPPedido uniqueidentifier NOT NULL` (FK a `tpPedido`), `FolioConfirmacion varchar(50) NULL`, `RutaArchivoPDF varchar(500) NULL`, `FechaGeneracion datetime2(7) NULL`, `Activo bit DEFAULT(1)`, `FechaRegistro datetime2(7) DEFAULT SYSUTCDATETIME()`.
- Verificar que la FK hacia `tpPedido.IdTPPedido` está activa.
- Verificar que ningún objeto dependiente se ve afectado.

**Resultado esperado:**
Tabla `fccConfirmacionPedido` creada en ProquifaDotNet lista para recibir los registros de Confirmaciones de Pedido generadas en el Paso 3.

**Entregables:**
- Script DDL: `CREATE TABLE fccConfirmacionPedido`
- Script de validación (estructura + FK activa)

**Scripts:**

```sql
-- Prerequisito: fccDocumentoFiscalCobro debe existir (Tarea 3)
CREATE TABLE [dbo].[fccConfirmacionPedido](
    [IdFCCConfirmacionPedido]       uniqueidentifier NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_Id]      DEFAULT (NEWID()),
    [IdFCCDocumentoFiscalCobro]     uniqueidentifier NOT NULL,
    [IdTPPedido]                    uniqueidentifier NOT NULL,
    [FolioConfirmacion]             varchar(80)      NOT NULL,
    [RutaArchivoPDF]                nvarchar(500)    NULL,
    [FechaGeneracion]               datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_FechaGen] DEFAULT (SYSUTCDATETIME()),
    [Activo]                        bit              NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_Activo]  DEFAULT (1),
    [FechaRegistro]                 datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_FechaReg] DEFAULT (SYSUTCDATETIME()),
    [FechaUltimaActualizacion]      datetime2(7)     NOT NULL
        CONSTRAINT [DF_fccConfirmacionPedido_FechaUpd] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_fccConfirmacionPedido]
        PRIMARY KEY CLUSTERED ([IdFCCConfirmacionPedido]),
    CONSTRAINT [FK_fccConfirmacionPedido_DocumentoFiscalCobro]
        FOREIGN KEY ([IdFCCDocumentoFiscalCobro])
        REFERENCES dbo.fccDocumentoFiscalCobro([IdFCCDocumentoFiscalCobro]),
    CONSTRAINT [FK_fccConfirmacionPedido_Pedido]
        FOREIGN KEY ([IdTPPedido])
        REFERENCES dbo.tpPedido([IdTPPedido])
);
GO
CREATE INDEX [IX_fccConfirmacionPedido_Pedido]
    ON dbo.fccConfirmacionPedido ([IdTPPedido]);
GO

-- Validación
SELECT
    c.name        AS Columna,
    t.name        AS Tipo,
    c.is_nullable AS EsNullable
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.fccConfirmacionPedido')
ORDER BY c.column_id;
```

**Criterios de aceptación:**
- La tabla existe con la estructura definida en `R16A-RE-FU-028_BD.md`.
- FK activa hacia `tpPedido.IdTPPedido`.
- Tabla vacía al crear.

**Más información de la tarea:**
Ver sección *"Tabla Nueva: fccConfirmacionPedido"* en `R16A-RE-FU-028_BD.md` y sección *"Parte A / A5"* en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028_BD.md` — DDL fccConfirmacionPedido
- `R16A-RE-FU-028-Back.md` — Parte A, sección A5

---

## TAREA 5

**[ RE-FU-028 ] [UPDATE-TABL-CH] Agregar FechaEstimadaEntrega a tpPedido (condicional — pendiente resolución Brecha B4)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — tpPedido

**Consideraciones previas:**
- ⚠️ **Brecha B4 BLOQUEANTE:** Las reglas de cálculo de la FEE y su granularidad (cabecera vs. partida) están pendientes de confirmar con Operaciones PROQUIFA México. Esta tarea no debe ejecutarse hasta resolver B4.
- `tpPartidaPedido.FechaEstimadaEntrega` ya existe a nivel de partida (calculada al tramitar con base en stock, RE-FU-010/011/013/014). El campo propuesto en `tpPedido` es la **FEE confirmada post-pago** que se comunica en la Confirmación de Pedido y se transfiere a Legacy como cabecera.
- El script es **condicional**: ejecutar solo si el campo no existe previamente en `tpPedido` (tabla pre-R16 — verificar con `sys.columns` en RYNL010).
- Verificar también que el ALTER no rompe SPs, vistas ni triggers dependientes de `tpPedido`.
- Es **prerrequisito** de la Tarea 15 (post-envío ejecuta `UPDATE tpPedido SET FechaEstimadaEntrega`).

**Objetivo general:**
Agregar el campo `FechaEstimadaEntrega` a nivel de cabecera en `tpPedido` para registrar la FEE confirmada post-pago, distinta de la FEE por partida ya existente en `tpPartidaPedido`.

**Objetivos específicos:**
- Verificar existencia: `SELECT 1 FROM sys.columns WHERE object_id = OBJECT_ID('dbo.tpPedido') AND name = 'FechaEstimadaEntrega'`.
- Si no existe: `ALTER TABLE dbo.tpPedido ADD FechaEstimadaEntrega datetime2(7) NULL`.
- Si ya existe: documentar el tipo y comportamiento actual; evaluar si coincide con el uso requerido en RE-028.
- Verificar que SPs, vistas y triggers dependientes de `tpPedido` no se ven afectados.
- Confirmar con Operaciones si la regla de FEE aplica a cabecera, partida o ambas (Brecha B4) antes de ejecutar.

**Resultado esperado:**
`tpPedido.FechaEstimadaEntrega datetime2(7) NULL` disponible para ser poblado en el post-envío del Paso 3, como FEE confirmada post-pago a nivel de cabecera del pedido.

**Entregables:**
- Script de verificación: `SELECT sys.columns` en RYNL010
- Script DDL condicional: `ALTER TABLE tpPedido ADD FechaEstimadaEntrega`
- Checklist de objetos dependientes verificados

**Scripts:**

```sql
-- =========================================================
-- Paso 1: Verificar existencia del campo en tpPedido
-- =========================================================
SELECT
    c.name        AS Columna,
    t.name        AS Tipo,
    c.max_length,
    c.is_nullable AS EsNullable
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.tpPedido')
  AND c.name = 'FechaEstimadaEntrega';
-- Si retorna 0 filas: ejecutar el ALTER a continuación.
-- Si retorna 1 fila:  validar tipo y uso; no ejecutar el ALTER.

-- =========================================================
-- Paso 2: Verificar FechaEstimadaEntrega en tpPartidaPedido (referencia)
-- =========================================================
SELECT
    c.name        AS Columna,
    t.name        AS Tipo,
    c.is_nullable AS EsNullable
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.tpPartidaPedido')
  AND c.name = 'FechaEstimadaEntrega';

-- =========================================================
-- Paso 3: ALTER condicional (ejecutar solo si no existe)
-- ⚠️ No ejecutar hasta resolver Brecha B4
-- =========================================================
IF NOT EXISTS (
    SELECT 1 FROM sys.columns
    WHERE object_id = OBJECT_ID('dbo.tpPedido')
      AND name = 'FechaEstimadaEntrega'
)
BEGIN
    ALTER TABLE dbo.tpPedido
        ADD FechaEstimadaEntrega datetime2(7) NULL;
END
GO
```

**Criterios de aceptación:**
- `tpPedido.FechaEstimadaEntrega datetime2(7) NULL` existe (preexistente o agregado por esta tarea).
- Ningún SP, vista ni trigger presenta errores tras el ALTER.
- ⚠️ La regla de cálculo y granularidad (B4) se documenta como pendiente hasta confirmar con Operaciones.

**Más información de la tarea:**
Ver sección *"ALTER TABLE tpPedido (condicional)"* en `R16A-RE-FU-028_BD.md` y Brecha B4 en `R16A-RE-FU-028-Back.md`. Ver nota de diseño FEE cabecera vs. partida en `R16A-RE-FU-028_BD.md`.

**Recursos:**
- `R16A-RE-FU-028_BD.md` — ALTER TABLE tpPedido (sección con nota de diseño y Brecha B4)
- `R16A-RE-FU-028-Back.md` — Brecha B4, sección B7.1

---

## TAREA 6

**[ RE-FU-028 ] [CREATE-SCRIPT-CONTROL] Crear vista vfccDocumentoFiscalCobro (consolidación estado Paso 3 por cliente)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3

**Consideraciones previas:**
- Vista operativa nueva. La Tarea 3 (`fccDocumentoFiscalCobro`) debe estar ejecutada antes de crear la vista.
- Consolida el estado del Paso 3 por cliente: navega desde `fccDocumentoFiscalCobro` hacia los registros de asociación del Paso 2, cobro, proforma, pedido, catálogos y CFDIs generados en una sola consulta.
- Finanzas la consume en dos escenarios: (a) al inicializar el Paso 3, para detectar si ya existen líneas previas y no reinicializar (Tarea 8); (b) al cerrar el wizard, para verificar que todas las líneas están en estado `ENVIADO` (Tarea 16).
- No requiere índices propios — los índices de las tablas base son suficientes.

**Objetivo general:**
Crear la vista `vfccDocumentoFiscalCobro` que consolida en una sola consulta el estado completo del Paso 3 por cliente, incluyendo datos de asociación, cobro, proforma, pedido, tipo de documento, estado de línea y CFDIs generados.

**Objetivos específicos:**
- Crear `CREATE VIEW dbo.vfccDocumentoFiscalCobro` con las columnas necesarias para que Finanzas renderice el estado actual del Paso 3 al reingresar.
- Unir: `fccDocumentoFiscalCobro` → `fccPagoFacturaPedido`/`fccPagoFacturaAdelanto` → `fccPagoCliente` → `tpProformaPedido`/`fccFactura` (RE-FU-015, antes `tpProformaAdelanto`) → `tpPedido` → `catTipoDocumentoFiscal` → `catDocumentoFiscalCobroEstado` → `CFDIGenerada` (Factura y Complemento).
- Incluir en la proyección: `IdFCCDocumentoFiscalCobro`, `IdFCCPagoCliente`, `IdCliente`, `TipoDocumento` (Clave del catálogo), `EstadoLinea` (Clave del catálogo), `IdCFDIGeneradaFactura`, `IdCFDIGeneradaComplemento`, `FechaGeneracion`, `FechaEnvio`, `FolioProforma`, `FolioFactura`, `FolioPedidoInterno`.
- Verificar que la vista compila sin errores y retorna datos correctos para líneas en los tres estados (`PENDIENTE`, `GENERADO`, `ENVIADO`).

**Resultado esperado:**
Vista `vfccDocumentoFiscalCobro` disponible en ProquifaDotNet, consultable por Finanzas para renderizar y validar el estado completo del Paso 3 por cliente.

**Entregables:**
- Script DDL: `CREATE VIEW vfccDocumentoFiscalCobro`
- Script de validación (`SELECT * FROM vfccDocumentoFiscalCobro` con datos de prueba)

**Scripts:**

```sql
-- Ejecutar DESPUÉS de CREATE TABLE fccDocumentoFiscalCobro y ALTERs de CFDIGenerada
CREATE VIEW [dbo].[vfccDocumentoFiscalCobro]
AS
SELECT
    p3l.IdFCCDocumentoFiscalCobro,
    tdf.Clave                    AS TipoDocumentoFiscal,
    est.Clave                    AS EstadoLinea,
    p3l.IdCatUsoCFDI,
    ufo.Clave                    AS UsoCFDIClave,
    ufo.Descripcion              AS UsoCFDIDescripcion,
    p3l.IdCatMetodoDePagoCFDI,
    mpc.Clave                    AS MetodoPagoClave,
    mpc.Descripcion              AS MetodoPagoDescripcion,
    -- Origen proforma (cuando IdFCCPagoFacturaPedido IS NOT NULL)
    p3l.IdFCCPagoFacturaPedido,
    pfp.IdFCCPagoCliente         AS IdFCCPagoCliente_PFP,
    pfp.IdTPProformaPedido,
    pp.Folio                     AS FolioProforma,
    pp.MontoTotal                AS MontoProforma,
    pp.MontoPendiente,
    e_pp.Prefijo                 AS EmpresaEmisoraProforma,
    -- Origen FAA (cuando IdFCCPagoFacturaAdelanto IS NOT NULL)
    p3l.IdFCCPagoFacturaAdelanto,
    pfa.IdFCCPagoCliente         AS IdFCCPagoCliente_PFA,
    pfa.IdFccFactura,            -- RE-FU-015 (antes: pfa.IdTPProformaAdelanto)
    fc.MontoTotal                 AS MontoFAA,
    cg_faa.UUID                  AS UUID_FAA,       -- UUID de la FAA para CFDIRelacionados
    -- Cobro (fuente de verdad: tabla de asociación Paso 2)
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
    -- Pedido (via cadena proforma → pedido)
    tp.IdTPPedido,
    tp.FolioPedidoInterno,
    tp.FechaEstimadaEntrega,
    -- CFDIs generados
    cg_f.IdCFDIGenerada          AS IdCFDIFactura,
    cg_f.UUID                    AS UUID_Factura,
    cg_f.Folio                   AS Folio_Factura,
    cg_f.Serie                   AS Serie_Factura,
    cg_f.FechaEmision            AS FechaEmision_Factura,
    cg_c.IdCFDIGenerada          AS IdCFDIComplemento,
    cg_c.UUID                    AS UUID_Complemento,
    cg_c.Folio                   AS Folio_Complemento,
    -- Estado legible
    CASE est.Clave
        WHEN 'ENVIADO'   THEN 'Enviado'
        WHEN 'GENERADO'  THEN 'Generado — pendiente envío'
        ELSE                  'Pendiente'
    END                          AS EstadoDescripcion,
    p3l.FechaGeneracion,
    p3l.FechaEnvio,
    p3l.FechaRegistro,
    p3l.FechaUltimaActualizacion
FROM dbo.fccDocumentoFiscalCobro p3l
INNER JOIN dbo.catTipoDocumentoFiscal tdf
    ON p3l.IdCatTipoDocumentoFiscal = tdf.IdCatTipoDocumentoFiscal
INNER JOIN dbo.catDocumentoFiscalCobroEstado est
    ON p3l.IdCatDocumentoFiscalCobroEstado = est.IdCatDocumentoFiscalCobroEstado
-- Asociación Paso 2 — proforma
LEFT JOIN dbo.fccPagoFacturaPedido pfp
    ON p3l.IdFCCPagoFacturaPedido = pfp.IdFCCPagoFacturaPedido
LEFT JOIN dbo.tpProformaPedido pp
    ON pfp.IdTPProformaPedido = pp.IdTPProformaPedido
LEFT JOIN dbo.Empresa e_pp
    ON pp.IdEmpresa = e_pp.IdEmpresa
-- Asociación Paso 2 — FAA
LEFT JOIN dbo.fccPagoFacturaAdelanto pfa
    ON p3l.IdFCCPagoFacturaAdelanto = pfa.IdFCCPagoFacturaAdelanto
LEFT JOIN dbo.fccFactura fc
    ON pfa.IdFccFactura = fc.IdFccFactura   -- RE-FU-015 (antes: LEFT JOIN dbo.tpProformaAdelanto pa ON pfa.IdTPProformaAdelanto = pa.IdTPProformaAdelanto)
LEFT JOIN dbo.CFDIGenerada cg_faa
    ON fc.IdCFDIGenerada = cg_faa.IdCFDIGenerada   -- UUID de la FAA para CFDIRelacionados
-- Cobro (resuelto desde la asociación Paso 2 activa)
LEFT JOIN dbo.fccPagoCliente fpc
    ON fpc.IdFCCPagoCliente = COALESCE(pfp.IdFCCPagoCliente, pfa.IdFCCPagoCliente)
-- Cliente
LEFT JOIN dbo.Cliente c
    ON fpc.IdCliente = c.IdCliente
LEFT JOIN dbo.DatosFacturacionCliente dfc
    ON fpc.IdCliente = dfc.IdCliente AND dfc.Activo = 1
-- Pedido (via proforma cuando aplica)
LEFT JOIN dbo.tpPedidoProformaPedido tpp_pp
    ON pp.IdTPProformaPedido = tpp_pp.IdTPProformaPedido AND tpp_pp.Activo = 1
LEFT JOIN dbo.tpPedido tp
    ON tpp_pp.IdTPPedido = tp.IdTPPedido
-- Catálogos SAT
LEFT JOIN dbo.catUsoCFDI ufo
    ON p3l.IdCatUsoCFDI = ufo.IdCatUsoCFDI
LEFT JOIN dbo.catMetodoDePagoCFDI mpc
    ON p3l.IdCatMetodoDePagoCFDI = mpc.IdCatMetodoDePagoCFDI
-- CFDIs generados en Paso 3
LEFT JOIN dbo.CFDIGenerada cg_f
    ON p3l.IdCFDIGeneradaFactura = cg_f.IdCFDIGenerada
LEFT JOIN dbo.CFDIGenerada cg_c
    ON p3l.IdCFDIGeneradaComplemento = cg_c.IdCFDIGenerada;
GO

-- Validación
SELECT TOP 10 * FROM dbo.vfccDocumentoFiscalCobro;
```

**Criterios de aceptación:**
- La vista compila sin errores.
- Retorna las columnas documentadas en `R16A-RE-FU-028_BD.md`.
- Navega correctamente para líneas originadas en `fccPagoFacturaPedido` y para líneas originadas en `fccPagoFacturaAdelanto`.
- Retorna resultados vacíos si no hay líneas del Paso 3 para el cliente (sin errores de NULL).

**Más información de la tarea:**
Ver sección *"CREATE VIEW vfccDocumentoFiscalCobro"* en `R16A-RE-FU-028_BD.md` y sección *"Parte A / A8"* en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028_BD.md` — CREATE VIEW vfccDocumentoFiscalCobro
- `R16A-RE-FU-028-Back.md` — Parte A, sección A8

---

## TAREA 7

**[ RE-FU-028 ] [IMP-EXIST-SERVICE] Extender endpoint de timbrado en Timbrado para soporte de tipo CFDI y cascada PPD**

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Timbrado — Endpoint por tipo de CFDI

**Consideraciones previas:**
- El endpoint de timbrado de Timbrado fue implementado en RE-FU-019 para Factura por Adelantado. Esta tarea lo extiende para soportar todos los tipos del Paso 3.
- La Tarea 2 debe estar ejecutada (`CFDIGenerada` tiene `IdCatTipoCFDI`).
- Por cada timbrado exitoso: INSERT `CFDIGenerada` con `IdCatTipoCFDI` resuelto desde `catTipoCFDI`, consumo atómico de folio en `EmpresaFolio` (UPDLOCK), llamada al PAC TurboPac, UPDATE `CFDIGenerada` con UUID/Folio/FechaEmision.
- Para el **Complemento en cascada PPD**: el `IdCFDIRelacionado` en `CFDIGenerada` se popula con el `IdCFDIGenerada` de la Factura PPD del mismo flujo. Finanzas envía este UUID como parte del request.
- El endpoint recibe: tipo de CFDI, datos emisor/receptor, partidas, `CFDIRelacionados` (NCs + UUID FAA si aplica), `IdCFDIRelacionado` (UUID Factura PPD para Complemento cascada).
- Retorna a Finanzas: UUID, Folio, Serie, FechaEmision del CFDI timbrado.
- ⚠️ Brecha B1: el tipo de relación SAT para Factura Anticipo de controlados (¿07?) está pendiente de confirmar con asesor fiscal.

**Objetivo general:**
Extender el endpoint de timbrado en ProquifaDotNet.Timbrado para soportar los cuatro tipos de CFDI del Paso 3 (FACTURA_PUE, FACTURA_PPD, FACTURA_ANTICIPO, COMPLEMENTO_PAGO), con inserción del discriminador `IdCatTipoCFDI` y soporte del campo `IdCFDIRelacionado` para el Complemento en cascada PPD.

**Objetivos específicos:**
- Extender el Command/Handler de timbrado para recibir `TipoCFDI` (clave de `catTipoCFDI`) en el request.
- Resolver `IdCatTipoCFDI` desde `catTipoCFDI` en el Handler y poblarlo en `CFDIGenerada` al INSERT.
- Recibir y persistir `IdCFDIRelacionado` en `CFDIGenerada` para el Complemento en cascada PPD (auto-referencia blanda, UUID de Factura PPD).
- Mantener el consumo atómico de folio en `EmpresaFolio` con UPDLOCK (patrón de RE-FU-019).
- Mantener la integración con PAC TurboPac sin cambios (mismo flujo de llamada y manejo de errores de RE-FU-019).
- Registrar en Serilog el tipo de CFDI, IdCFDIGenerada, folio y resultado por cada timbrado.

**Resultado esperado:**
El endpoint de timbrado en ProquifaDotNet.Timbrado acepta los cuatro tipos del Paso 3, inserta `IdCatTipoCFDI` e `IdCFDIRelacionado` en `CFDIGenerada` correctamente, y retorna UUID/Folio/Serie/FechaEmision a Finanzas.

**Entregables:**
- Extensión del Command + Handler de timbrado en Timbrado
- DTO de request actualizado con campos `TipoCFDI` e `IdCFDIRelacionado`
- Pruebas unitarias para los cuatro tipos de CFDI (incluyendo `IdCFDIRelacionado` para Complemento cascada)
- Prueba de no regresión para FAA (RE-FU-019)

**Criterios de aceptación:**
- Para `FACTURA_PUE`: INSERT `CFDIGenerada` con `IdCatTipoCFDI = FACTURA_PUE`, `IdCFDIRelacionado = NULL`.
- Para `FACTURA_PPD`: INSERT `CFDIGenerada` con `IdCatTipoCFDI = FACTURA_PPD`, `IdCFDIRelacionado = NULL`.
- Para `COMPLEMENTO_PAGO` en cascada PPD: INSERT `CFDIGenerada` con `IdCatTipoCFDI = COMPLEMENTO_PAGO`, `IdCFDIRelacionado` = UUID de la Factura PPD.
- Para `FACTURA_ANTICIPO`: INSERT `CFDIGenerada` con `IdCatTipoCFDI = FACTURA_ANTICIPO`, `IdCFDIRelacionado = NULL`.
- El folio se consume atómicamente de `EmpresaFolio` con UPDLOCK en todos los casos.
- El flujo FAA de RE-FU-019 no presenta regresiones.

**Más información de la tarea:**
Ver sección *"Parte C / C1"* en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte C, sección C1
- `R16A-RE-FU-019-Tareas.md` — Tarea 13 (FacturaAdelantadoGenerarService, patrón base a extender)

---

## TAREA 8

**[ RE-FU-028 ] [SERV-TRANSACT] Implementar inicialización del Paso 3 — creación de líneas fccDocumentoFiscalCobro**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 México

**Consideraciones previas:**
- Las Tareas 1, 3 y 6 deben estar ejecutadas (catálogos, tabla y vista disponibles en BD).
- Al avanzar desde el Paso 2 con la asociación cerrada, Finanzas crea una línea en `fccDocumentoFiscalCobro` por cada documento de la asociación.
- La **lógica condicional del tipo** por línea: `fccPagoFacturaPedido` + `HayControlados=0` → `FACTURA`; `fccPagoFacturaPedido` + `HayControlados=1` → `FACTURA_ANTICIPO`; `fccPagoFacturaAdelanto` → `COMPLEMENTO_PAGO`.
- Al reingresar al Paso 3 (si ya existen líneas para el cliente), Finanzas recupera el estado desde `vfccDocumentoFiscalCobro` sin reinicializar.
- Los datos del emisor/receptor necesarios para el timbrado se leen en este paso desde ProquifaDotNet: `DatosFacturacionCliente`, `Empresa`.

**Objetivo general:**
Implementar en Finanzas el servicio de inicialización del Paso 3 que, al avanzar desde el Paso 2, crea las líneas de `fccDocumentoFiscalCobro` con el tipo de documento correcto por línea, y al reingresar recupera el estado existente sin duplicar.

**Objetivos específicos:**
- Implementar `POST /api/v1/validate-collection/fiscalDocumentStep/initialize` en Finanzas.
- Crear `InitializeStep3Command` + Handler con lógica condicional de tipo por línea.
- Leer `fccPagoFacturaPedido` / `fccPagoFacturaAdelanto` + `tpProformaPedido.HayControlados` vía Scaffold Finanzas (`tpProformaPedido` movida a Finanzas 07/07/2026).
- `INSERT fccDocumentoFiscalCobro` por cada documento (estado inicial `PENDIENTE`, `IdCatUsoCFDI` = default del cliente desde `DatosFacturacionCliente`).
- Detectar re-entrada: `GET vfccDocumentoFiscalCobro WHERE IdCliente=@Id AND EstadoLinea != 'ENVIADO'` — si hay registros, retornar el estado existente sin reinicializar.
- Calcular y retornar en `Step3InitializedDto` el flag `CanGoBackSteps: bool` = `false` si **alguna** línea existente está en estado `GENERADO` o `ENVIADO`; `true` si todas las líneas están en `PENDIENTE`.
- Por cada línea en el response, incluir la lista de NCs aplicadas en el Paso 2 (campo `CreditNotes: List<AppliedCreditNoteDto>`) obtenida desde `fccNotaCredito` vía los FKs de la asociación (`fccPagoFacturaPedido` o `fccPagoFacturaAdelanto`). Este dato es necesario para la visualización en pantalla (sección E2 del requisito).
- DTO de response: `Step3InitializedDto` con la lista de líneas (tipo, estado, NCs aplicadas) y el flag `CanGoBackSteps`.

**Resultado esperado:**
Endpoint `POST .../fiscalDocumentStep/initialize` en Finanzas que crea las líneas del Paso 3 con el tipo correcto o recupera las existentes al reingresar, retornando el estado actual del Paso 3 para el cliente.

**Entregables:**
- Endpoint `POST /api/v1/validate-collection/fiscalDocumentStep/initialize`
- Command + Handler: `InitializeStep3Command`
- DTO: `Step3InitializedDto` (lista de líneas con tipo, estado, datos del documento)
- Pruebas unitarias para los tres tipos de línea (FACTURA, FACTURA_ANTICIPO, COMPLEMENTO_PAGO) y para la detección de re-entrada

**Criterios de aceptación:**
- Las líneas `fccPagoFacturaPedido` con `HayControlados=0` crean línea de tipo `FACTURA`.
- Las líneas `fccPagoFacturaPedido` con `HayControlados=1` crean línea de tipo `FACTURA_ANTICIPO`.
- Las líneas `fccPagoFacturaAdelanto` crean línea de tipo `COMPLEMENTO_PAGO`.
- Al reingresar con líneas previas no enviadas: no se duplican, se retorna el estado existente desde `vfccDocumentoFiscalCobro`.
- El estado inicial de toda línea nueva es `PENDIENTE`.
- `CanGoBackSteps = false` si alguna línea está en `GENERADO` o `ENVIADO`; `true` si todas están en `PENDIENTE`.
- Cada línea en `Step3InitializedDto` incluye la lista de NCs aplicadas desde `fccNotaCredito` (puede ser vacía si no hubo NCs en Paso 2).

**Más información de la tarea:**
Ver sección *"Parte B / B1"* en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte B, sección B1
- `R16A-RE-FU-028_BD.md` — fccDocumentoFiscalCobro (ciclo de vida), vfccDocumentoFiscalCobro

---

## TAREA 9

**[ RE-FU-028 ] [SERV-SIMPLE-PUT] Implementar auto-guardado de Uso CFDI y Método de Pago por línea**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 México

**Consideraciones previas:**
- La Tarea 3 debe estar ejecutada y la Tarea 8 debe haberse ejecutado al menos una vez para que existan líneas en `fccDocumentoFiscalCobro`.
- El usuario puede cambiar el Uso CFDI y el Método de Pago de una línea antes de timbrar; Finanzas persiste el cambio inmediatamente.
- Para líneas de tipo `COMPLEMENTO_PAGO`: el `IdCatMetodoDePagoCFDI` **no se persiste** (PPD es fijo e implícito por tipo).
- Escritura: `UPDATE fccDocumentoFiscalCobro SET IdCatUsoCFDI = @Id, IdCatMetodoDePagoCFDI = @Id, FechaUltimaActualizacion WHERE IdFCCDocumentoFiscalCobro = @Id`.

**Objetivo general:**
Implementar en Finanzas el endpoint de auto-guardado que persiste inmediatamente los cambios de Uso CFDI y Método de Pago al seleccionarlos en la pantalla del Paso 3.

**Objetivos específicos:**
- Implementar `PUT /api/v1/validate-collection/fiscalDocumentLine/{idLinea}/cfdiConfig` en Finanzas.
- Crear `UpdateLineConfigurationStep3Command` + Handler.
- Validar que la línea existe y pertenece al cliente activo.
- Para líneas `COMPLEMENTO_PAGO`: ignorar `IdCatMetodoDePagoCFDI` aunque se reciba en el payload.
- Retornar el estado actualizado de la línea.

**Resultado esperado:**
Endpoint `PUT .../lineas/{idLinea}/configuracion` que persiste Uso CFDI y Método de Pago de forma inmediata y silenciosa al usuario.

**Entregables:**
- Endpoint `PUT /api/v1/validate-collection/fiscalDocumentLine/{idLinea}/cfdiConfig`
- Command + Handler: `UpdateLineConfigurationStep3Command`
- DTO de request: `UpdateLineConfigurationDto` (IdCatUsoCFDI, IdCatMetodoDePagoCFDI nullable)
- Pruebas unitarias (incluyendo: línea COMPLEMENTO_PAGO no guarda IdCatMetodoDePagoCFDI, validación de pertenencia al cliente)

**Criterios de aceptación:**
- `UPDATE fccDocumentoFiscalCobro.IdCatUsoCFDI` se ejecuta correctamente.
- Para líneas `COMPLEMENTO_PAGO`: `IdCatMetodoDePagoCFDI` no se modifica aunque se envíe en el payload.
- El endpoint retorna error si la línea no pertenece al cliente activo.
- El endpoint retorna error si la línea ya está en estado `GENERADO` o `ENVIADO` (no editable).

**Más información de la tarea:**
Ver sección *"Parte B / B2"* en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte B, sección B2

---

## TAREA 10

**[ RE-FU-028 ] [IMP-EXIST-SERVICE] Implementar resolución de Notas de Crédito aplicadas en nodo CFDIRelacionados al timbrar**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 México

**Consideraciones previas:**
- Las NCs aplicadas en el Paso 2 se leen de `fccNotaCredito WHERE IdFCCPagoCliente IN (cobros de la línea) AND Aplicada=1`.
- Cada NC se incluye en el nodo `CFDIRelacionados` del XML con su UUID (`fccNotaCredito.IdCFDI`) y el tipo de relación SAT correspondiente.
- ⚠️ El tipo de relación SAT correcto para las NCs en la Factura Anticipo de controlados está vinculado a la Brecha B1 — pendiente confirmar con asesor fiscal.
- Este servicio es transversal: lo consume la Tarea 12 (timbrado) al armar el request para Timbrado.
- Para líneas sin NCs aplicadas: el nodo `CFDIRelacionados` se incluye vacío o se omite según la especificación del PAC.

**Objetivo general:**
Implementar el servicio en Finanzas que consulta las NCs aplicadas en el Paso 2 para una línea del Paso 3 y las mapea al nodo `CFDIRelacionados` del XML CFDI 4.0.

**Objetivos específicos:**
- Crear `ICreditNoteCfdiResolutionService` que, dado un `IdFCCDocumentoFiscalCobro`, retorna la lista de NCs aplicadas en formato `RelatedCfdiDto` (UUID, tipo relación SAT, monto).
- Leer `fccNotaCredito.IdCFDI` (UUID timbrado) + `Monto` de las NCs con `IdFCCPagoCliente` en los cobros de la línea y `Aplicada=1`, vía API ProquifaDotNet.
- Mapear al tipo de relación SAT según el tipo de línea (pendiente de confirmar para `FACTURA_ANTICIPO` — usar placeholder hasta resolver Brecha B1).
- Retornar lista vacía si no hay NCs aplicadas en la línea.

**Resultado esperado:**
Servicio `CreditNoteCfdiResolutionService` disponible para ser inyectado en el Handler de timbrado (Tarea 12), retornando la lista de NCs en formato `RelatedCfdiDto` lista para el request al endpoint de Timbrado.

**Entregables:**
- `ICreditNoteCfdiResolutionService` + `CreditNoteCfdiResolutionService`
- DTO: `RelatedCfdiDto` (UUID, TipoRelacionSAT, Monto)
- Pruebas unitarias (incluyendo: línea con NCs, línea sin NCs, tipo relación SAT por tipo de línea)

**Criterios de aceptación:**
- El servicio retorna correctamente los UUIDs de las NCs aplicadas en la línea.
- El `IdCFDI` de cada NC (UUID timbrado SAT) se mapea al campo UUID del `RelatedCfdiDto`.
- Para líneas sin NCs: retorna lista vacía (sin errores).
- ⚠️ El tipo de relación SAT para `FACTURA_ANTICIPO` queda como parámetro configurable hasta resolver Brecha B1.

**Más información de la tarea:**
Ver sección *"Parte B / B5"* en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte B, sección B5
- `R16A-RE-FU-026_BD.md` — fccNotaCredito (IdCFDI, Aplicada, IdFCCPagoCliente)

---

## TAREA 11

**[ RE-FU-028 ] [IMP-EXIST-SERVICE] Implementar previsualización PDF por línea: FACTURA y FACTURA_ANTICIPO**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 México

**Consideraciones previas:**
- `InvoicePdfMappingService.MapearPreviewAsync` (RE-FU-021) ya existe y genera el modelo sin `TimbreFiscalDigital`. Esta tarea lo invoca para las líneas de tipo `FACTURA` y `FACTURA_ANTICIPO`.
- Para líneas de tipo `COMPLEMENTO_PAGO`: la previsualización del PDF del Complemento se implementa en **R16A-RE-FU-030** — en esta tarea no se incluye.
- El PDF de previsualización se genera en memoria sin persistir en BD ni MinIO.
- Los templates `GOL/MUN/PRO/PQF_MEX_FAC` ya existen (RE-FU-021). El `TemplateKey` se resuelve dinámicamente desde `Empresa.Prefijo`.
- La Tarea 8 debe haberse ejecutado para que existan líneas en el Paso 3 que previsualizar.

**Objetivo general:**
Implementar en Finanzas el endpoint de previsualización PDF del Paso 3 para líneas de tipo `FACTURA` y `FACTURA_ANTICIPO`, invocando el servicio de mapping existente de RE-FU-021 y retornando el PDF en memoria al frontend.

**Objetivos específicos:**
- Implementar `POST /api/v1/validate-collection/fiscalDocumentLine/{idLinea}/pdfPreview` en Finanzas.
- Crear `GetStep3LinePreviewPdfQuery` + Handler.
- Invocar `InvoicePdfMappingService.MapearPreviewAsync(idCFDIGenerada)` para obtener el `InvoicePdfModel` sin `TimbreFiscalDigital`.
- Resolver `TemplateKey` dinámicamente (`GOL/MUN/PRO/PQF_MEX_FAC`) desde `Empresa.Prefijo`.
- Generar PDF en memoria vía DocumentBuilder y retornarlo como `byte[]` o stream al frontend.
- Para líneas `COMPLEMENTO_PAGO`: retornar error controlado indicando que la previsualización del Complemento corresponde a RE-FU-030.
- Sin escrituras en BD.

**Resultado esperado:**
Endpoint `GET .../lineas/{idLinea}/preview-pdf` que retorna el PDF de previsualización en memoria para líneas `FACTURA` y `FACTURA_ANTICIPO`, usando los templates existentes de RE-FU-021.

**Entregables:**
- Endpoint `POST /api/v1/validate-collection/fiscalDocumentLine/{idLinea}/pdfPreview`
- Query + Handler: `GetStep3LinePreviewPdfQuery`
- Pruebas unitarias para los 4 emisores (GOL/MUN/PRO/PQF) × 2 tipos (FACTURA / FACTURA_ANTICIPO)

**Criterios de aceptación:**
- El PDF de previsualización no contiene UUID, sellos ni QR (sin `TimbreFiscalDigital`).
- El branding (logo, colores) corresponde a la empresa emisora del pedido.
- Para líneas `COMPLEMENTO_PAGO`: el endpoint retorna error controlado (no intenta generar el preview).
- No hay persistencia en BD ni MinIO (preview en memoria únicamente).

**Más información de la tarea:**
Ver sección *"Parte B / B3"* en `R16A-RE-FU-028-Back.md`. Ver `InvoicePdfMappingService.MapearPreviewAsync` en `R16A-RE-FU-021-Back.md`.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte B, sección B3
- `R16A-RE-FU-021-Back.md` — InvoicePdfMappingService (MapearPreviewAsync)

---

## TAREA 12

**[ RE-FU-028 ] [ALG-COMPLX-LOGIC] Implementar timbrado por línea: 4 escenarios (PUE, PPD+cascada, Anticipo, Complemento FAA)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 México

**Consideraciones previas:**
- Las Tareas 7, 8, 9 y 10 deben estar ejecutadas (Timbrado extendido, líneas existentes, NCs resueltas).
- `ApiCallerStamping` (HttpClient + Polly) ya existe de RE-FU-018/019 — el Paso 3 usa `StampInvoiceAsync` (`POST /api/v1/stamp/invoice`) para Factura/Factura Anticipo y `StampPaymentComplementAsync` (`POST /api/v1/stamp/payment-complement`) para el Complemento.
- `PersistInvoicePdfService` ya existe de RE-FU-021 — se invoca post-timbrado para FACTURA y FACTURA_ANTICIPO.
- **Escenario B (PPD + cascada):** la Factura PPD se timbra primero; inmediatamente tras el éxito, Finanzas solicita el timbrado del Complemento enviando el UUID de la Factura PPD como `IdCFDIRelacionado`. Si el Complemento falla, la Factura PPD queda vigente (⚠️ Brecha B6 — política pendiente).
- **Escenario D (COMPLEMENTO_PAGO desde FAA):** Finanzas envía el UUID de la FAA existente (`fccFactura.IdCFDIGenerada`, RE-FU-015 — antes `tpProformaAdelanto.IdCFDIGenerada`) como referencia. La generación del PDF del Complemento corresponde a RE-FU-030.
- ⚠️ Brecha B1: tipo de relación SAT para FACTURA_ANTICIPO pendiente de confirmar.
- ⚠️ Brecha B6: política ante fallo del Complemento en cascada PPD pendiente de definir.

**Objetivo general:**
Implementar en Finanzas el servicio central de timbrado del Paso 3 con los cuatro escenarios: FACTURA PUE (1 CFDI), FACTURA PPD + Complemento en cascada (2 CFDIs), FACTURA_ANTICIPO (1 CFDI) y COMPLEMENTO_PAGO desde FAA existente (1 CFDI). Post-timbrado exitoso, invocar `PersistInvoicePdfService` para las Facturas y actualizar el estado de la línea a `GENERADO`.

**Objetivos específicos:**
- Implementar `POST /api/v1/validate-collection/fiscalDocumentLine/{idLinea}/stamp` en Finanzas.
- Crear `StampLineCommand` + Handler con detección del escenario por tipo de línea y Método de Pago.
- **Escenario A — FACTURA PUE:**
  1. Finanzas → Timbrado: request con `TipoCFDI=FACTURA_PUE`, datos emisor/receptor, NCs en `CFDIRelacionados`.
  2. Recibir UUID + XML timbrado.
  3. Invocar `PersistInvoicePdfService.PersistirAsync(IdCFDI, xmlTimbrado)`.
  4. `UPDATE fccDocumentoFiscalCobro SET EstadoLinea=GENERADO, IdCFDIGeneradaFactura, FechaGeneracion`.
  5. `UPDATE tpProformaPedido SET IdCFDIGenerada` directo vía EF Core (Scaffold Finanzas — movida a Finanzas 07/07/2026).
- **Escenario B — FACTURA PPD + Complemento en cascada:**
  1. Timbrar Factura PPD (igual que A, `TipoCFDI=FACTURA_PPD`).
  2. Invocar `PersistInvoicePdfService.PersistirAsync` para la Factura.
  3. Timbrar Complemento (`TipoCFDI=COMPLEMENTO_PAGO`, `IdCFDIRelacionado=UUID Factura PPD`). **El PDF del Complemento es responsabilidad de RE-FU-030.**
  4. `UPDATE fccDocumentoFiscalCobro SET EstadoLinea=GENERADO, IdCFDIGeneradaFactura, IdCFDIGeneradaComplemento, FechaGeneracion`.
- **Escenario C — FACTURA_ANTICIPO:** igual que A con `TipoCFDI=FACTURA_ANTICIPO` y tipo de relación 07 SAT en `CFDIRelacionados`.
- **Escenario D — COMPLEMENTO_PAGO desde FAA:**
  1. Timbrar Complemento (`TipoCFDI=COMPLEMENTO_PAGO`, `IdCFDIRelacionado=UUID FAA`).
  2. `UPDATE fccDocumentoFiscalCobro SET EstadoLinea=GENERADO, IdCFDIGeneradaFactura=@IdComplemento, FechaGeneracion`.
- Ante error del PAC: la línea permanece en `PENDIENTE`; Finanzas retorna el detalle del error para corrección y reintento.

**Resultado esperado:**
Endpoint `POST .../lineas/{idLinea}/timbrar` en Finanzas que ejecuta el escenario de timbrado correcto para cada tipo de línea, actualiza el estado a `GENERADO` y persiste el PDF de Factura (cuando aplica) post-timbrado exitoso.

**Entregables:**
- Endpoint `POST /api/v1/validate-collection/fiscalDocumentLine/{idLinea}/stamp`
- Command + Handler: `StampLineCommand` con los 4 escenarios
- Pruebas unitarias para los 4 escenarios (incluyendo: error PAC → línea queda PENDIENTE, fallo Complemento cascada con Factura PPD vigente)
- Prueba de integración básica con Timbrado mock

**Criterios de aceptación:**
- **Escenario A:** `fccDocumentoFiscalCobro.IdCFDIGeneradaFactura` poblado, estado = `GENERADO`, PDF en MinIO.
- **Escenario B:** ambos `IdCFDIGeneradaFactura` e `IdCFDIGeneradaComplemento` poblados, estado = `GENERADO`, PDF Factura en MinIO.
- **Escenario C:** igual que A con `IdCatTipoCFDI = FACTURA_ANTICIPO`.
- **Escenario D:** `IdCFDIGeneradaFactura` poblado con UUID del Complemento, estado = `GENERADO`.
- Error PAC: la línea permanece en `PENDIENTE`; el detalle del error se retorna al cliente.
- ⚠️ Tipo de relación SAT para FACTURA_ANTICIPO queda como parámetro configurable hasta resolver Brecha B1.
- ⚠️ Política ante fallo del Complemento en cascada PPD queda documentada como Brecha B6.
- **OBS-049:** El timbrado se procesa **línea por línea (uno a uno)** — no como proceso masivo. Cada CFDI se timbra en una llamada independiente al PAC. El endpoint `POST .../lineas/{idLinea}/timbrar` aplica a una sola línea por invocación.
- **OBS-050:** Antes de ejecutar en producción (o en plan de pruebas con el PAC real), **verificar los límites de concurrencia del PAC (SAP)**. Incluir en el plan de pruebas la validación de los límites del PAC para evitar rechazos por volumen en escenarios de carga.

**Más información de la tarea:**
Ver sección *"Parte B / B4"* y Brechas B1 y B6 en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte B, sección B4; Brechas B1 y B6
- `R16A-RE-FU-019-Tareas.md` — Tarea 13 (ApiCallerStamping, patrón base)
- `R16A-RE-FU-021-Back.md` — PersistInvoicePdfService

---

## TAREA 13

**[ RE-FU-028 ] [CREATE-PDF] Plantilla PDF Confirmación de Pedido Prepago México — 4 variantes (GOL/MUN/PRO/PQF_MEX_CDP)**

**Aplicativos:** DocumentBuilder

**Módulos:** DocumentBuilder — Plantillas Confirmación de Pedido

**Consideraciones previas:**
- Plantillas nuevas. No existen en DocumentBuilder. `TemplateKey` sigue el patrón `{Prefix}_MEX_CDP` (CDP = Confirmación de Pedido).
- Se requieren 4 variantes: `GOL_MEX_CDP`, `MUN_MEX_CDP`, `PRO_MEX_CDP`, `PQF_MEX_CDP` — una por empresa emisora del grupo México.
- Cada template requiere 3 archivos HTML: Header (`*_CDP_H.html`), Body (`*_CDP_B.html`), Footer (`*_CDP_F.html`).
- El registro en `DocumentTemplate` se inserta vía script SQL (patrón existente en `Scripts/`).
- El contenido de la Confirmación de Pedido incluye: datos del pedido (folio interno, fecha), datos del cliente y contacto, lista de partidas con cantidades, empresa emisora, datos bancarios, FEE confirmada, folio de confirmación y elementos de branding por empresa.
- ⚠️ Brecha B5: el formato exacto del folio de la Confirmación de Pedido está pendiente de confirmar con PMO/Operaciones. Diseñar el template con campo `FolioConfirmacion` configurable.
- Es **prerrequisito** de la Tarea 14 (al confirmar el envío, T14 invoca DocumentBuilder con estas plantillas para generar el PDF de Confirmación antes de despachar el correo).

**Objetivo general:**
Crear las 4 variantes de la plantilla HTML (Header, Body, Footer) y los registros en `DocumentTemplate` para la Confirmación de Pedido Prepago México, una por empresa emisora, con el branding correspondiente a cada empresa del grupo.

**Objetivos específicos (por variante — repetir para GOL, MUN, PRO, PQF):**
- Crear `{Prefix}_MEX_CDP_H.html` — cabecera con logo de la empresa, datos del emisor y título "Confirmación de Pedido".
- Crear `{Prefix}_MEX_CDP_B.html` — cuerpo con: folio de confirmación, fecha, datos del cliente y contacto, lista de partidas (producto, cantidad, unidad, precio), FEE confirmada, condiciones de entrega y datos bancarios.
- Crear `{Prefix}_MEX_CDP_F.html` — pie con información de contacto de la empresa, disclaimer y paginación "X de Y".
- Insertar el registro `DocumentTemplate` en BD para cada variante vía script SQL.

**Resultado esperado:**
Los 4 templates `GOL/MUN/PRO/PQF_MEX_CDP` registrados en DocumentBuilder y listos para generar el PDF de Confirmación de Pedido Prepago desde el post-envío del Paso 3.

**Entregables:**
- `GOL_MEX_CDP_H/B/F.html`, `MUN_MEX_CDP_H/B/F.html`, `PRO_MEX_CDP_H/B/F.html`, `PQF_MEX_CDP_H/B/F.html`
- Scripts SQL: `INSERT DocumentTemplate` para las 4 variantes
- Prueba de generación con datos de muestra para cada variante

**Criterios de aceptación:**
- Los 4 templates están registrados en `DocumentTemplate` y son invocables desde DocumentBuilder.
- El branding (logo, colores, datos de contacto) corresponde a cada empresa emisora.
- El campo `FolioConfirmacion` está presente en el Body y se renderiza correctamente.
- La paginación automática funciona correctamente.
- ⚠️ El formato del folio queda como parámetro configurable hasta confirmar con PMO (Brecha B5).

**Más información de la tarea:**
Ver sección *"Parte D"* en `R16A-RE-FU-028-Back.md` y sección *"Parte B / B7.2"*. Ver patrón de templates `*_MEX_FAC` en `R16A-RE-FU-021-Tareas.md` (Tareas 5–8) como referencia.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte D, sección B7.2
- `R16A-RE-FU-021-Tareas.md` — Tareas 5–8 (patrón de creación de templates)
- DocumentBuilder — `C:\Users\juan.garcia\Documents\DocumentBuilder-R14`

---

## TAREA 14

**[ RE-FU-028 ] [IMP-EXIST-SERVICE] Implementar modal de Envío y despacho vía ProquifaDotNet.EnvioCorreo (adjuntos CFDI + Confirmación de Pedido)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 México

**Consideraciones previas:**
- La Tarea 12 debe estar ejecutada (línea en estado `GENERADO` para poder enviar).
- La Tarea 13 debe estar ejecutada (templates `GOL/MUN/PRO/PQF_MEX_CDP` disponibles en DocumentBuilder).
- El modal de envío se abre al presionar "Enviar" en una línea en estado `GENERADO`. El usuario puede editar destinatarios antes de confirmar.
- **Destinatarios:** Para: contacto del pedido (`tpPedido.IdContacto`) — editable; CC: ESAC asignado al cliente — editable.
- ⚠️ Brecha B7: si el pedido no tiene contacto asignado, el comportamiento del modal (¿bloquea el envío?, ¿permite captura manual?) está pendiente de confirmar con negocio.
- ⚠️ Brecha B2: el asunto y cuerpo del correo para líneas `COMPLEMENTO_PAGO` están pendientes de confirmar con PMO (#31).
- **Orden de ejecución al confirmar envío:** (1) Generar PDF Confirmación de Pedido vía DocumentBuilder → subir a MinIO → obtener `RutaArchivoPDF`; (2) construir adjuntos (incluyendo el PDF Confirmación); (3) llamar al API de ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo — regla 7, sin cliente Brevo propio en Finanzas); (4) al éxito: `UPDATE fccDocumentoFiscalCobro SET EstadoLinea=ENVIADO` + persists DB. **El PDF debe generarse ANTES de llamar a ProquifaDotNet.EnvioCorreo** para poder adjuntarlo; si la generación falla, el envío no se despacha.
- La `RutaArchivoPDF` generada se pasa a la Tarea 15 para el `INSERT fccConfirmacionPedido`.
- Los adjuntos de Factura se sirven desde MinIO (ya persistidos en Tarea 12).

**Objetivo general:**
Implementar en Finanzas el endpoint del modal de Envío del Paso 3 que: (1) genera el PDF de Confirmación de Pedido vía DocumentBuilder y lo sube a MinIO, (2) despacha el correo vía ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo) con los adjuntos CFDI (PDF + XML) y el PDF de Confirmación, (3) actualiza el estado a `ENVIADO` y (4) dispara las acciones post-envío de la Tarea 15.

**Objetivos específicos:**
- Implementar `POST /api/v1/validate-collection/fiscalDocumentLine/{idLinea}/send` en Finanzas.
- Crear `SendLineCommand` + Handler.
- **Paso previo al correo — Generar Confirmación de Pedido:**
  - Resolver `TemplateKey` desde `Empresa.Prefijo` (`GOL/MUN/PRO/PQF_MEX_CDP`).
  - Invocar DocumentBuilder con el modelo del pedido (folio, cliente, contacto, partidas, empresa).
  - Subir el PDF generado a MinIO (bucket `confirmaciones`) y obtener `RutaArchivoPDF`.
  - Si la generación del PDF falla: retornar error controlado sin despachar el correo.
- **Asunto del correo por tipo de línea:**
  - `FACTURA_PUE` / `FACTURA_PPD`: `"{FolioPedidoInterno} — Factura {FolioFactura}"`.
  - `FACTURA_ANTICIPO`: `"{FolioPedidoInterno} — Factura Anticipo {FolioFacturaAdelanto}"`.
  - `COMPLEMENTO_PAGO`: ⚠️ pendiente de confirmar con PMO (Brecha B2).
- Construir el listado de adjuntos según el tipo de línea **incluyendo el PDF de Confirmación de Pedido** (ver tabla en `R16A-RE-FU-028-Back.md` Parte B/B6).
- Llamar al API de ProquifaDotNet.EnvioCorreo (`ApiCallerEnvioCorreo`, RE-FU-016) con los adjuntos, destinatarios y asunto correspondiente.
- Al envío exitoso: `UPDATE fccDocumentoFiscalCobro SET EstadoLinea=ENVIADO, FechaEnvio` + `INSERT CorreoEnviado` + `INSERT ArchivoCorreoEnviado` (x N adjuntos) + registrar en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo — regla 8).
- Pasar `RutaArchivoPDF` a la Tarea 15 para el `INSERT fccConfirmacionPedido`.
- Retornar confirmación del envío al frontend (incluyendo `RutaArchivoPDF` para T15).

**Resultado esperado:**
Endpoint `POST .../lineas/{idLinea}/enviar` que genera el PDF de Confirmación, lo adjunta junto con los CFDIs en el correo despachado vía ProquifaDotNet.EnvioCorreo, actualiza el estado a `ENVIADO` y deja `RutaArchivoPDF` disponible para la Tarea 15.

**Entregables:**
- Endpoint `POST /api/v1/validate-collection/fiscalDocumentLine/{idLinea}/send`
- Command + Handler: `SendLineCommand` (con generación de Confirmación de Pedido antes de llamar a ProquifaDotNet.EnvioCorreo)
- DTO de request: `SendLineRequestDto` (destinatarios editables)
- DTO de response: incluye `RutaArchivoPDF` para uso en T15
- Pruebas unitarias para los tipos de línea (adjuntos correctos por tipo, PDF Confirmación adjunto, UPDATE estado, INSERT CorreoEnviado, fallo generación PDF → no envío)

**Criterios de aceptación:**
- Para `FACTURA PUE`: adjuntos = PDF Factura + XML Factura + **PDF Confirmación de Pedido**.
- Para `FACTURA PPD + Complemento`: adjuntos = PDF Factura + XML Factura + XML Complemento + **PDF Confirmación** (PDF Complemento: RE-FU-030).
- Para `FACTURA_ANTICIPO`: adjuntos = PDF Factura Anticipo + XML Factura Anticipo + **PDF Confirmación**.
- Para `COMPLEMENTO_PAGO` desde FAA: adjuntos = XML Complemento + **PDF Confirmación** (PDF Complemento: RE-FU-030).
- El PDF de Confirmación de Pedido se genera con el template `{Prefijo}_MEX_CDP` correcto antes de llamar a ProquifaDotNet.EnvioCorreo.
- Si la generación del PDF de Confirmación falla: el correo **no** se despacha y se retorna error controlado.
- `fccDocumentoFiscalCobro.EstadoLinea = ENVIADO` tras el envío exitoso.
- `INSERT CorreoEnviado` + `INSERT ArchivoCorreoEnviado` correctamente registrados (N adjuntos incluye PDF Confirmación).
- La validación de cobro queda registrada en ProquifaDotNet.BitacoraCambios (regla 8).
- El asunto del correo es `"{FolioPedidoInterno} — Factura {FolioFactura}"` para FACTURA_PUE/PPD y `"{FolioPedidoInterno} — Factura Anticipo {FolioFacturaAdelanto}"` para FACTURA_ANTICIPO.
- ⚠️ Brecha B2 (asunto/plantilla correo Complemento) y Brecha B7 (contacto no disponible) documentados como pendientes.

**Más información de la tarea:**
Ver sección *"Parte B / B6"* y Brechas B2 y B7 en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte B, sección B6; Brechas B2 y B7

---

## TAREA 15

**[ RE-FU-028 ] [SERV-TRANSACT] Implementar acciones post-envío: FEE + registro trazabilidad Confirmación de Pedido**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 México (Post-envío)

**Consideraciones previas:**
- Las Tareas 4, 5 y 14 deben estar ejecutadas antes de implementar esta tarea.
- Se dispara automáticamente al confirmar el envío exitoso de cada línea (parte del flujo de Tarea 14), recibiendo la `RutaArchivoPDF` que T14 generó y subió a MinIO.
- **El PDF de Confirmación de Pedido lo genera y sube a MinIO la Tarea 14 (antes de despachar el correo).** Esta tarea solo persiste la referencia en `fccConfirmacionPedido` con la ruta ya disponible.
- ⚠️ **Brecha B4 BLOQUEANTE:** Las reglas de cálculo de la FEE y su granularidad están pendientes — implementar el UPDATE con el campo disponible pero con la regla como placeholder parametrizable hasta confirmar con Operaciones.
- Ambas acciones (FEE e INSERT Confirmación) se ejecutan como acciones post-envío; si alguna falla, la línea ya está en `ENVIADO` — manejar errores individualmente con log, sin revertir el estado de la línea.
- Solo aplica a México. Para Perú no se disparan estas acciones.
- Las transferencias ETL a Legacy (E1/E2/E3/E6) se implementan en la **Tarea 17** por tener un bloqueante independiente (Brecha B3).

**Objetivo general:**
Implementar en Finanzas las acciones post-envío del Paso 3 México: (1) establecer la FEE en `tpPedido` y (2) registrar la trazabilidad de la Confirmación de Pedido en `fccConfirmacionPedido` con la `RutaArchivoPDF` recibida de la Tarea 14.

**Objetivos específicos:**

**B7.1 — FEE:**
- `UPDATE tpPedido SET FechaEstimadaEntrega = @FEECalculada` vía API ProquifaDotNet.
- Implementar la regla de cálculo como parámetro configurable (parámetro de empresa o fijo hasta confirmar con Operaciones — Brecha B4).

**B7.2 — Registro trazabilidad Confirmación de Pedido:**
- `INSERT fccConfirmacionPedido (FolioConfirmacion, RutaArchivoPDF, FechaGeneracion)` usando la `RutaArchivoPDF` recibida de la Tarea 14.
- No invoca DocumentBuilder ni MinIO — el PDF ya fue generado y subido por T14.

**Resultado esperado:**
Al confirmar el envío exitoso de cada línea México: FEE actualizada en `tpPedido` y registro de trazabilidad de la Confirmación insertado en `fccConfirmacionPedido` con la ruta del PDF ya almacenado en MinIO por T14.

**Entregables:**
- Servicio `PostSendDeliveryConfirmationService` (FEE + INSERT trazabilidad Confirmación)
- `INSERT fccConfirmacionPedido` (FolioConfirmacion, RutaArchivoPDF recibida de T14, FechaGeneracion)
- `UPDATE tpPedido SET FechaEstimadaEntrega` (regla parametrizable)
- Pruebas unitarias para FEE e INSERT (incluyendo: fallo en una acción no revierte el estado ENVIADO)

**Criterios de aceptación:**
- `tpPedido.FechaEstimadaEntrega` se actualiza al enviar una línea México exitosamente.
- `fccConfirmacionPedido` contiene un nuevo registro con `RutaArchivoPDF` poblada (ruta generada por T14).
- Esta tarea **no** invoca DocumentBuilder ni MinIO — solo persiste la referencia.
- Si cualquier acción post-envío falla: la línea permanece en `ENVIADO`, el error se registra en Serilog y no se revierte.
- Para Perú: ninguna acción post-envío se dispara.
- ⚠️ Brecha B4 (regla FEE) queda documentada como pendiente.

**Más información de la tarea:**
Ver sección *"Parte B / B7.1 y B7.2"* en `R16A-RE-FU-028-Back.md`. Ver Brecha B4.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte B, secciones B7.1 y B7.2; Brecha B4
- `R16A-RE-FU-028_BD.md` — fccConfirmacionPedido DDL

---

## TAREA 16

**[ RE-FU-028 ] [SERV-SIMPLE-PUT] Implementar cierre automático del wizard al completar todas las líneas enviadas**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 3 México

**Consideraciones previas:**
- La Tarea 14 debe estar ejecutada. El cierre se verifica después de cada envío exitoso de línea.
- El wizard se cierra automáticamente cuando **todas** las líneas del cliente están en estado `ENVIADO`.
- La verificación se hace vía `vfccDocumentoFiscalCobro` con la consulta: `SELECT COUNT(*) FROM vfccDocumentoFiscalCobro WHERE IdCliente = @Id AND EstadoLinea != 'ENVIADO'`.
- Al cerrarse, el cliente sale del listado de pendientes de Validar Cobro.
- No requiere acción manual del usuario: el cierre es automático si todas las líneas están en `ENVIADO`.

**Objetivo general:**
Implementar en Finanzas la lógica de cierre automático del wizard del Paso 3, que detecta cuando todas las líneas del cliente han sido enviadas y cierra el wizard retornando al listado principal de Validar Cobro.

**Objetivos específicos:**
- Implementar `GET /api/v1/validate-collection/fiscalDocumentStep/{idCliente}/closingStatus` en Finanzas (o integrar en la respuesta del endpoint de envío de Tarea 14).
- Crear `VerifyStep3ClosureQuery` + Handler.
- Consultar `vfccDocumentoFiscalCobro WHERE IdCliente=@Id AND EstadoLinea != 'ENVIADO'`.
- Si `COUNT = 0`: retornar `{ closed: true }` — el frontend navega al listado principal.
- Si `COUNT > 0`: retornar `{ closed: false, pendingLines: N }` — el wizard permanece abierto.

**Resultado esperado:**
Lógica de cierre automático disponible en Finanzas: tras cada envío exitoso de línea, el frontend consulta si el wizard puede cerrarse, y al detectar que todas las líneas están en `ENVIADO`, navega al listado principal de Validar Cobro retirando al cliente de los pendientes.

**Entregables:**
- Endpoint o respuesta integrada en Tarea 14 con estado de cierre
- Query + Handler: `VerifyStep3ClosureQuery`
- DTO de response: `Step3ClosureStatusDto` (closed: bool, pendingLines: int)
- Pruebas unitarias (incluyendo: todas ENVIADO → closed=true; una PENDIENTE → closed=false)

**Criterios de aceptación:**
- Cuando todas las líneas del cliente están en `ENVIADO`: `closed = true`.
- Cuando al menos una línea no está en `ENVIADO`: `closed = false`, `pendingLines = N`.
- El cliente sale del listado de pendientes de Validar Cobro al detectar `closed = true`.
- La verificación usa `vfccDocumentoFiscalCobro` y no genera consultas adicionales a tablas base.

**Más información de la tarea:**
Ver sección *"Parte B / B8"* en `R16A-RE-FU-028-Back.md`.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte B, sección B8
- `R16A-RE-FU-028_BD.md` — vfccDocumentoFiscalCobro

---

## TAREA 17

**[ R16A-RE-FU-028 ] [QUERY-G] Análisis ETL Legacy — Mapeo de datos E1/E2/E3/E6 y resolución Brecha B3**

**Aplicativos:** ProquifaDotNet.Finanzas, Legacy (consulta)

**Módulos:** Validar Cobro — Paso 3 México (ETL Legacy — Análisis)

**Consideraciones previas:**
- ⚠️ **Brecha B3 BLOQUEANTE:** El canal de transferencia a Legacy (tabla ETL, cola RabbitMQ, API directa) no está definido. Esta tarea existe para resolverla formalmente con el equipo de arquitectura.
- Precede a T18 (implementación de builders) y T19 (canal definitivo). No puede avanzar implementación sin los resultados de este análisis.
- Requiere sesión de trabajo con el equipo de arquitectura y revisión con el equipo Legacy para definir tablas destino y campos de referencia cruzada.
- Alcance exclusivo: E1 (Buzón de Cobros), E2 (Proforma), E3 (Factura), E6 (PDF Factura). E4/E5/E7/E8 pertenecen a RE-030/032/034.

**Objetivo general:**
Resolver la Brecha B3 documentando el mecanismo de transferencia ETL a Legacy y mapeando con precisión los datos de origen (ProquifaDotNet) a destino (Legacy) para cada uno de los cuatro eventos E1, E2, E3 y E6.

**Objetivos específicos:**
- Definir el canal de transferencia con arquitectura: tabla ETL intermedia en BD, cola RabbitMQ o llamada API Legacy directa.
- E1: identificar el campo de referencia cruzada en Legacy que vincula el cobro de ProquifaDotNet con el registro en Legacy.
- E2: confirmar qué tablas Legacy reciben los datos de Proforma; diferenciar `fccPagoFacturaPedido` vs `fccPagoFacturaAdelanto`.
- E3: mapear UUID, Folio, Serie, Total del CFDI a columnas exactas en tablas Legacy (Factura, Pedidos, Partidas, Cobro).
- E6: definir si Legacy recibe bytes del PDF o ruta MinIO; confirmar formato y capacidad de almacenamiento.
- Documentar orden de ejecución E2 → E1 → E3+E6, atomicidad de E3+E6 y política de error ante fallo de canal.
- Entregar documento de análisis como insumo directo para T18 y T19.

**Resultado esperado:**
Brecha B3 resuelta. Documento de análisis ETL que incluye canal definitivo seleccionado, mapeo columna-a-columna para E1/E2/E3/E6, política de atomicidad, manejo de errores y acuerdos con arquitectura y Legacy.

**Entregables:**
- Documento de análisis ETL con: canal de transferencia seleccionado y justificación, mapeo origen → destino para E1/E2/E3/E6, política de atomicidad E3+E6, política de error (sin revertir `ENVIADO`), acuerdos documentados con arquitectura y Legacy.
- Actualización de Brecha B3 en `R16A-RE-FU-028-Back.md` como resuelta (con la decisión tomada).

**Criterios de aceptación:**
- El canal de transferencia está definido y acordado con el equipo de arquitectura.
- El campo de referencia cruzada Legacy-ProquifaDotNet para E1 está identificado.
- Los mapeos de campos para E2, E3 y E6 están documentados a nivel de tabla y columna en Legacy.
- Se definió si E6 transfiere bytes o ruta, con confirmación del equipo Legacy.
- La política de error (sin revertir `ENVIADO`) está confirmada por arquitectura.
- El documento de análisis está disponible y aprobado como prerequisito para T18 y T19.

**Más información de la tarea:**
Ver Brecha B3 y Parte E en `R16A-RE-FU-028-Back.md`. El análisis debe resolver específicamente las incógnitas marcadas en las secciones E1–E6 y en Consideraciones transversales ETL.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte E (E1–E6), Brecha B3, Consideraciones transversales ETL
- `R16A-RE-FU-028_BD.md` — tablas fuente: `fccPagoCliente`, `tpProformaPedido`, `CFDIGenerada`, `DatosFacturacionCliente`

---

## TAREA 18

**[ R16A-RE-FU-028 ] [SERV-COMPLEX-TRANSACT] Implementar builders de payloads ETL E1/E2/E3/E6 e interfaz IEtlLegacyTransferService**

**Aplicativos:** ProquifaDotNet.AplicativoNuevoLegacy

**Módulos:** Validar Cobro — Paso 3 México (ETL Legacy — Implementación)

**Consideraciones previas:**
- Predecesora: T17 — Análisis ETL (mapeo de datos y resolución Brecha B3).
- La Tarea 14 y la Tarea 15 deben estar ejecutadas antes de esta tarea.
- Se dispara automáticamente al confirmar el envío exitoso de cada línea, como parte del flujo post-envío junto con T15.
- Si la Brecha B3 aún no está resuelta al momento de implementar: la interfaz usa la implementación stub que registra el payload como `ETL_PENDIENTE` en Serilog sin bloquear el flujo.
- Al resolver B3: la implementación stub se reemplaza por la real en T19 sin cambiar la interfaz ni los callers.
- Solo aplica a México. Para Perú no hay ETL en este requisito.
- Alcance: E1 (Buzón de Cobros), E2 (Proforma), E3 (Factura), E6 (PDF Factura).

**Objetivo general:**
Implementar la capa de construcción de los cuatro payloads ETL (builders) y la interfaz `IEtlLegacyTransferService` con implementación stub, de forma que el flujo post-envío quede funcional independientemente del estado de la Brecha B3.

**Objetivos específicos:**
- Crear `IEtlLegacyTransferService` con método `SendAsync(EtlLegacyPayload payload)`.
- Crear `EtlLegacyTransferServiceStub`: implementación que loguea el payload en Serilog como `ETL_PENDIENTE` y retorna éxito simulado.
- Crear `EtlLegacyPayload` con discriminador de tipo (`EtlType`: E1/E2/E3/E6) y datos del payload.
- Implementar `EtlPaymentMailboxPayloadBuilder` (E1): construye desde `fccPagoCliente` + `fccBuzonCobro` con el campo de referencia cruzada definido en T17.
- Implementar `EtlQuotePayloadBuilder` (E2): construye desde `tpProformaPedido`; diferencia entre `fccPagoFacturaPedido` y `fccPagoFacturaAdelanto`.
- Implementar `EtlInvoicePayloadBuilder` (E3): construye desde `CFDIGenerada`, `DatosFacturacionCliente`, `Empresa`.
- Implementar `EtlInvoicePdfPayloadBuilder` (E6): construye referencia del PDF desde MinIO vía `CFDIGenerada.IdArchivoPdf`; omite para líneas `COMPLEMENTO_PAGO` sin Factura.
- Invocar los builders en el flujo post-envío de T15 en el orden definido en T17: E2 → E1 → E3+E6.

**Resultado esperado:**
Los cuatro builders construyen correctamente los payloads usando el mapeo de T17. La interfaz permite inyectar stub o implementación real sin cambiar callers. El flujo post-envío no se bloquea aunque B3 esté pendiente.

**Entregables:**
- `IEtlLegacyTransferService` (interfaz)
- `EtlLegacyTransferServiceStub` (implementación stub con log `ETL_PENDIENTE`)
- `EtlLegacyPayload` + enum `EtlType` (E1/E2/E3/E6)
- `EtlPaymentMailboxPayloadBuilder`, `EtlQuotePayloadBuilder`, `EtlInvoicePayloadBuilder`, `EtlInvoicePdfPayloadBuilder`
- Pruebas unitarias:
  - E1: payload con datos correctos de `fccPagoCliente` y referencia cruzada
  - E2: variante `fccPagoFacturaPedido` vs `fccPagoFacturaAdelanto`
  - E3: payload con UUID, Folio, RFC emisor/receptor correctos
  - E6: genera referencia para `FACTURA`; retorna vacío/null para `COMPLEMENTO_PAGO` sin Factura
  - Stub: payload logueado como `ETL_PENDIENTE`, flujo no interrumpido

**Criterios de aceptación:**
- Los 4 builders producen payloads correctos según el mapeo definido en T17.
- Para líneas `COMPLEMENTO_PAGO` sin Factura: `EtlInvoicePdfPayloadBuilder` retorna vacío sin lanzar excepción.
- La implementación stub loguea `ETL_PENDIENTE` en Serilog y retorna éxito; el flujo post-envío no revierte `ENVIADO`.
- La inyección de dependencias permite sustituir stub por implementación real sin cambiar callers.
- Todas las pruebas unitarias pasan con cobertura de los casos críticos.
- El código es revisado y aprobado por el equipo.

**Más información de la tarea:**
Ver Parte E / E1–E6 en `R16A-RE-FU-028-Back.md`. El mapeo columna-a-columna para construir los payloads proviene del análisis de T17.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte E, E1/E2/E3/E6; Brecha B3
- `R16A-RE-FU-028_BD.md` — tablas fuente ETL (`fccPagoCliente`, `tpProformaPedido`, `CFDIGenerada`, `DatosFacturacionCliente`)
- Análisis de T17 (mapeo de campos y campo de referencia cruzada E1)

---

## TAREA 19

**[ R16A-RE-FU-028 ] [SERV-TRANSACT] Canal ETL Legacy definitivo E1/E2/E3/E6 — integración, pruebas y documentación**

**Aplicativos:** ProquifaDotNet.AplicativoNuevoLegacy, Legacy (validación)

**Módulos:** Validar Cobro — Paso 3 México (ETL Legacy — Integración y Validación)

**Consideraciones previas:**
- Predecesoras: T17 (análisis y resolución B3) y T18 (builders + interfaz + stub). Esta tarea NO inicia sin ambas completadas.
- Requiere que la Brecha B3 esté resuelta (canal definido por arquitectura).
- Implementa `EtlLegacyTransferService` real que reemplaza `EtlLegacyTransferServiceStub`, sin modificar `IEtlLegacyTransferService` ni los callers.
- Si el canal es RabbitMQ: configurar cola, exchange, binding y política de reintento.
- Si el canal es tabla ETL: configurar proceso lector en Legacy (SSIS u otro) para consumir los registros.
- Si el canal es API Legacy directa: documentar endpoint, contrato y autenticación.
- Las pruebas de integración deben ejecutarse en ambiente de QA/staging con Legacy real.
- El fallo en el canal no revierte el estado `ENVIADO` de la línea; manejar con log, reencola o bandera de reintento según política definida en T17.
- Solo aplica a México. Para Perú no hay ETL en este requisito.

**Objetivo general:**
Implementar el canal ETL definitivo para las transferencias E1/E2/E3/E6 a Legacy, reemplazando la implementación stub de T18 por la real, ejecutar y documentar las pruebas de integración, y garantizar que los payloads lleguen correctamente a Legacy con trazabilidad completa.

**Objetivos específicos:**
- Implementar `EtlLegacyTransferService` real según el mecanismo definido en T17 (tabla ETL / RabbitMQ / API directa).
- Registrar la implementación real en el contenedor DI en sustitución del stub.
- Validar en Legacy que los registros de E1, E2, E3 y E6 se almacenan correctamente en las tablas destino.
- Ejecutar pruebas de integración end-to-end: flujo completo Validar Cobro → envío Paso 3 → payloads en Legacy.
- Verificar atomicidad de E3+E6: ambos llegan juntos o el fallo queda logueado para reintento.
- Verificar orden de ejecución: E2 → E1 → E3+E6.
- Documentar resultados de pruebas: casos ejecutados, evidencias, incidencias encontradas y resolución.
- Actualizar `R16A-RE-FU-028-Back.md` confirmando B3 como resuelta e implementada.

**Resultado esperado:**
Los payloads E1/E2/E3/E6 se transfieren exitosamente a Legacy a través del canal definitivo. Las pruebas de integración están ejecutadas y documentadas. La Brecha B3 queda formalmente cerrada.

**Entregables:**
- `EtlLegacyTransferService` real (implementación definitiva del canal)
- Configuración del canal según tecnología: DDL tabla ETL / configuración RabbitMQ / contrato API Legacy
- Pruebas de integración documentadas:
  - E1: cobro visible en Legacy vinculado al cliente y pedido
  - E2: proforma registrada en Legacy correctamente
  - E3: factura CFDI registrada con UUID, folio y datos fiscales
  - E6: PDF disponible en repositorio Legacy
  - Atomicidad E3+E6: si falla E6, se loguea y reencola sin revertir E3
  - Orden E2 → E1 → E3+E6: verificado en logs
  - Fallo de canal: estado `ENVIADO` no cambia, payload logueado o reencolado
- Documento de resultados de pruebas (casos, evidencias, incidencias y resolución)
- Actualización de Brecha B3 en `R16A-RE-FU-028-Back.md` como resuelta e implementada

**Criterios de aceptación:**
- Los 4 payloads (E1/E2/E3/E6) llegan a Legacy con datos correctos validados en tablas destino.
- La atomicidad E3+E6 se cumple: ambos se procesan exitosamente o el fallo queda logueado para reintento.
- El estado `ENVIADO` de la línea no se revierte ante fallos del canal ETL.
- Las pruebas de integración están ejecutadas con evidencias en ambiente QA/staging.
- La implementación real pasa revisión de código del equipo.
- Brecha B3 actualizada como resuelta en `R16A-RE-FU-028-Back.md`.

**Más información de la tarea:**
Esta tarea cierra el ciclo ETL de RE-028. Los ETL E4/E7 (Complemento de Pago) y E5/E8 (Notas de Crédito) se implementan en RE-030 y RE-032/034 respectivamente.

**Recursos:**
- `R16A-RE-FU-028-Back.md` — Parte E, Brecha B3 (resolución), Consideraciones transversales ETL
- `R16A-RE-FU-028_BD.md` — tablas fuente ETL
- Análisis de T17 (documento de mapeo y definición de canal)
- Entregables de T18 (interfaz `IEtlLegacyTransferService` + builders + stub)
