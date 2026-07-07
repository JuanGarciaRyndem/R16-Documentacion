# Tareas BackEnd — R16A-RE-FU-030
**Requisito:** Diseño y generación de Documentos: CDP México — Complemento de Pago (CFDI tipo P / Pagos20 v2.0)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder

---

> **Orden de ejecución sugerido:** BD catFormaPagoSAT (T1) → BD DML CP01 (T2) → BD ALTER fccDocumentoFiscalCobro (T3) → BD EmpresaFolio Serie P (T4) → BD DocumentTemplate (T5) → BD ALTER vista v3.0 (T6) → Timbrado endpoint CP (T7) → Finanzas: MappingService + PersistirService (T8) → Finanzas: Preview PDF CP (T9) → Finanzas: Cascada PPD + Escenario D (T10) → DocumentBuilder: templates HTML *_MEX_CP (T11).
>
> **Dependencias externas:** R16A-RE-FU-028 completo (`catTipoCFDI.COMPLEMENTO_PAGO`, `fccDocumentoFiscalCobro` con `IdCFDIGeneradaComplemento`, `CFDIGenerada` con `IdCFDIRelacionado`, `catMetodoDePagoCFDI`). R16A-RE-FU-029 completo (`catTipoCFDI.IdRegion`, `vfccDocumentoFiscalCobro` v2.0). R16A-RE-FU-021 completo (`MexicoInvoicePdfMappingService`, `PersistMexicoInvoicePdfService` — patrón base de los nuevos servicios PDF del CP).
>
> **Brechas activas sin bloqueante:** B1 (hora FechaPago) y B5 (formato folio Serie P) son pendientes de bajo impacto que se documentan en código con TODO sin bloquear la implementación. B2 (plantilla correo) bloquea solo la configuración de la plantilla en ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo, regla 7); el despacho de adjuntos sí puede implementarse. B3 (política reintento CP) se implementa como "línea queda en PENDIENTE" hasta que PMO defina el flujo formal. **No hay brecha bloqueante en este requisito.**

---

## Resumen de tareas

| #   | Clave                | Título simple                                                                                      | Tipo | Aplicativo              |
| --- | -------------------- | -------------------------------------------------------------------------------------------------- | ---- | ----------------------- |
| 1   | CREATE-TABL-CH       | Crear catFormaPagoSAT (c_FormaPago SAT) con DDL + DML 22 claves                                   | BD   | ProquifaDotNet          |
| 2   | UPDATE-TABL-CH       | DML catUsoCFDI: INSERT clave CP01 (Pagos) — confirmado inexistente                                | BD   | ProquifaDotNet          |
| 3   | UPDATE-TABL-M        | Extender fccDocumentoFiscalCobro: ADD 8 columnas snapshot DR del Complemento de Pago              | BD   | ProquifaDotNet          |
| 4   | UPDATE-TABL-CH       | DML EmpresaFolio: INSERT 4 filas Serie "P" para GOL, MUN, PRO, PQF                                | BD   | ProquifaDotNetTimbrado  |
| 5   | UPDATE-TABL-CH       | DML DocumentTemplate: INSERT 4 templates PDF Complemento de Pago México                           | BD   | DocumentBuilder         |
| 6   | CREATE-SCRIPT-CONTROL| Actualizar vista vfccDocumentoFiscalCobro v3.0: columnas DR + JOIN catFormaPagoSAT                | BD   | ProquifaDotNet          |
| 7   | ALG-COMPLX-LOGIC     | Implementar endpoint timbrado CP (CFDI tipo P Pagos20 v2.0) en Timbrado — folio Serie P + TurboPac| Back | ProquifaDotNet.Timbrado |
| 8   | SERV-TRANSACT        | Implementar PaymentComplementPdfMappingService y PersistPaymentComplementPdfService (MinIO cobranza) | Back | ProquifaDotNet.Finanzas |
| 9   | IMP-EXIST-SERVICE    | Implementar previsualización PDF CP por línea en el Paso 3 (stub pendiente de RE-028 B3)          | Back | ProquifaDotNet.Finanzas |
| 10  | ALG-COMPLX-LOGIC     | Implementar generación automática del CP: cálculo DR, cascada PPD (Esc. B) y CP desde FAA (Esc. D)| Back | ProquifaDotNet.Finanzas |
| 11  | CREATE-PDF           | Diseñar e implementar templates HTML Complemento de Pago México: GOL/MUN/PRO/PQF_MEX_CP (H/B/F)  | Back | DocumentBuilder         |

---

## TAREA 1

**[ RE-FU-030 ] [CREATE-TABL-CH] Crear catFormaPagoSAT (c_FormaPago SAT) con DDL + DML 22 claves**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3 México — Complemento de Pago

**Consideraciones previas:**
- Catálogo nuevo. Confirmado que no existe ningún catálogo equivalente en ProquifaDotNet.
- Almacena las formas reales de cobro (campo `FormaDePagoP` del nodo Pago del XML Pagos20 v2.0). No confundir con `catMetodoDePagoCFDI` (PPD/PUE — método de pago del CFDI).
- Patrón estándar del proyecto: `uniqueidentifier PK` con `NEWID()`, `Clave varchar UNIQUE`, `Descripcion nvarchar`, `Activo bit DEFAULT(1)`, `FechaRegistro datetime2(7) DEFAULT SYSUTCDATETIME()`.
- Datos iniciales: 22 claves del catálogo c_FormaPago SAT vigente (01 Efectivo, 02 Cheque, 03 Transferencia, 04 Tarjeta de crédito, 28 Tarjeta de débito, entre otras).
- Es **prerrequisito** de la Tarea 3 (`fccDocumentoFiscalCobro` agrega FK `IdCatFormaPagoSAT` → `catFormaPagoSAT`).

**Objetivo general:**
Crear el catálogo `catFormaPagoSAT` en ProquifaDotNet con las 22 claves del c_FormaPago SAT, habilitando la FK que `fccDocumentoFiscalCobro` usará para registrar la forma real de cobro en el Complemento de Pago.

**Objetivos específicos:**
- Ejecutar `CREATE TABLE catFormaPagoSAT` con PK, UNIQUE en `Clave`, `Activo` y `FechaRegistro`.
- Insertar las 22 claves del catálogo c_FormaPago SAT vigente.
- Verificar PK, UNIQUE y que todos los registros tienen `Activo=1`.

**Resultado esperado:**
`catFormaPagoSAT` existe en ProquifaDotNet con las 22 claves del c_FormaPago SAT, listo para ser referenciado como FK en `fccDocumentoFiscalCobro` y consultado por Finanzas al armar el `PaymentComplementRequest`.

**Entregables:**
- Script DDL + DML: `CREATE TABLE catFormaPagoSAT` + `INSERT` 22 claves iniciales
- Script de validación (`SELECT` con conteo)

**Scripts:**

```sql
-- Ejecutar en ProquifaDotNet
CREATE TABLE [dbo].[catFormaPagoSAT](
    [IdCatFormaPagoSAT]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catFormaPagoSAT_Id]       DEFAULT (NEWID()),
    [Clave]              varchar(10)      NOT NULL,
        -- Código SAT c_FormaPago: '01', '02', '03', '04', '05', '06', '08', etc.
    [Descripcion]        nvarchar(150)    NOT NULL,
    [Activo]             bit              NOT NULL
        CONSTRAINT [DF_catFormaPagoSAT_Activo]   DEFAULT (1),
    [FechaRegistro]      datetime2(7)     NOT NULL
        CONSTRAINT [DF_catFormaPagoSAT_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catFormaPagoSAT]
        PRIMARY KEY CLUSTERED ([IdCatFormaPagoSAT]),
    CONSTRAINT [UQ_catFormaPagoSAT_Clave]
        UNIQUE ([Clave])
);
GO

-- Datos iniciales (catálogo c_FormaPago SAT vigente)
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
SELECT IdCatFormaPagoSAT, Clave, Descripcion, Activo FROM dbo.catFormaPagoSAT ORDER BY Clave;
```

**Criterios de aceptación:**
- `catFormaPagoSAT` existe con la estructura definida en `R16A-RE-FU-030_BD.md`.
- Contiene 22 registros iniciales; todos con `Activo=1`.
- PK y UNIQUE constraint activos.
- La consulta `SELECT * FROM catFormaPagoSAT WHERE Clave = '03'` retorna la fila de Transferencia electrónica de fondos.

**Más información de la tarea:**
Ver sección *"Catálogo: catFormaPagoSAT"* en `R16A-RE-FU-030_BD.md` y sección *"Parte A / A1"* en `R16A-RE-FU-030-Back.md`.

**Recursos:**
- `R16A-RE-FU-030_BD.md` — DDL + DML catFormaPagoSAT
- `R16A-RE-FU-030-Back.md` — Parte A, sección A1

---

## TAREA 2

**[ RE-FU-030 ] [UPDATE-TABL-CH] DML catUsoCFDI: INSERT clave CP01 (Pagos) — confirmado inexistente**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3 México — Complemento de Pago

**Consideraciones previas:**
- La tabla `catUsoCFDI` existe desde un requisito anterior. Confirmado mediante consulta SSMS que la clave `CP01` no está presente. Claves actuales: P01, G03, S01, G02, G01, N/A.
- `CP01` es obligatoria: el nodo Receptor del Complemento de Pago lleva `UsoCFDI=CP01` fijo según norma SAT (Regla 6 / Criterio C3 del requisito).
- Columnas de la tabla (confirmadas): `ClaveUso`, `Uso`, `Clave`, `Activo`. Verificar existencia de PK `IdCatUsoCFDI` antes de ejecutar.
- Ejecutar solo si `SELECT * FROM catUsoCFDI WHERE ClaveUso = 'CP01'` retorna vacío.

**Objetivo general:**
Insertar la clave `CP01` (Pagos) en `catUsoCFDI` para que Finanzas pueda asignarla como `UsoCFDI` del Receptor al armar el XML del Complemento de Pago.

**Objetivos específicos:**
- Verificar ausencia de `CP01` en `catUsoCFDI`.
- Ejecutar el `INSERT` con `ClaveUso='CP01'`, `Uso='CP01 Pagos'`, `Clave='CP01'`, `Activo=1`.
- Validar que la fila insertada es correcta.

**Resultado esperado:**
`catUsoCFDI` contiene la clave `CP01` con los valores correctos. Finanzas puede resolver `UsoCFDI=CP01` al armar el Receptor del Complemento de Pago.

**Entregables:**
- Script DML: `INSERT INTO catUsoCFDI` + script de validación

**Scripts:**

```sql
-- Ejecutar en ProquifaDotNet
-- ⚠️ Verificar que no existe CP01 antes de ejecutar:
SELECT * FROM dbo.catUsoCFDI WHERE ClaveUso = 'CP01';
-- Si retorna vacío, ejecutar el INSERT:

INSERT INTO dbo.catUsoCFDI (ClaveUso, Uso, Clave, Activo)
VALUES ('CP01', 'CP01 Pagos', 'CP01', 1);
GO

-- Validación
SELECT ClaveUso, Uso, Clave, Activo FROM dbo.catUsoCFDI WHERE ClaveUso = 'CP01';
```

**Criterios de aceptación:**
- `SELECT * FROM catUsoCFDI WHERE ClaveUso = 'CP01'` retorna exactamente 1 fila con `Activo=1`.
- No se modificaron ni eliminaron filas existentes.

**Más información de la tarea:**
Ver sección *"DML catUsoCFDI — INSERT clave CP01 (Pagos)"* en `R16A-RE-FU-030_BD.md` y sección *"Parte A / A2"* en `R16A-RE-FU-030-Back.md`.

**Recursos:**
- `R16A-RE-FU-030_BD.md` — DML catUsoCFDI

---

## TAREA 3

**[ RE-FU-030 ] [UPDATE-TABL-M] Extender fccDocumentoFiscalCobro: ADD 8 columnas snapshot DR del Complemento de Pago**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3 México — Complemento de Pago

**Consideraciones previas:**
- La tabla `fccDocumentoFiscalCobro` existe desde RE-028 y fue extendida en RE-029. Las 8 columnas nuevas son nullable — no afectan filas existentes.
- **Prerrequisito:** La Tarea 1 (`catFormaPagoSAT`) debe estar ejecutada antes de agregar la FK `IdCatFormaPagoSAT`.
- Las columnas almacenan el **snapshot fiscal inmutable** del nodo DoctoRelacionado y del nodo Pago del XML Pagos20 v2.0, en el momento del timbrado. No se recomputarán post-timbrado.
- Son NULL para todas las líneas no-CP (FACTURA, FACTURA_ANTICIPO, líneas Perú). La validación de presencia se hace en la capa de aplicación (Finanzas), no con CHECK CONSTRAINT.

**Columnas a agregar:**

| Columna | Tipo | Campo XML CP |
|---|---|---|
| `FechaPagoCP` | datetime2(7) NULL | `FechaPago` del nodo Pago |
| `IdCatFormaPagoSAT` | uniqueidentifier FK NULL | `FormaDePagoP` (FK → catFormaPagoSAT) |
| `TipoCambioP_CP` | decimal(18,6) NULL | `TipoCambioP` cuando MonedaP ≠ MXN |
| `NumParcialidad` | int NULL | `NumParcialidad` del DoctoRelacionado |
| `ImpSaldoAnt` | decimal(18,6) NULL | `ImpSaldoAnt` en MonedaDR |
| `ImpPagado` | decimal(18,6) NULL | `ImpPagado` en MonedaDR |
| `ImpSaldoInsoluto` | decimal(18,6) NULL | `ImpSaldoInsoluto` = ImpSaldoAnt − ImpPagado |
| `EquivalenciaDR` | decimal(18,6) NULL | `EquivalenciaDR` (1 si MonedaDR = MonedaP) |

**Objetivo general:**
Extender `fccDocumentoFiscalCobro` con las 8 columnas del snapshot fiscal del Complemento de Pago, permitiendo que Finanzas persista los valores del DoctoRelacionado como datos inmutables al momento del timbrado.

**Objetivos específicos:**
- Ejecutar los 9 `ALTER TABLE` (8 columnas + 1 FK constraint).
- Verificar que las 8 columnas existen y son nullable.
- Verificar que la FK `FK_fccDocumentoFiscalCobro_FormaPagoSAT` está activa.
- Confirmar que filas existentes siguen con NULL en las columnas nuevas.

**Resultado esperado:**
`fccDocumentoFiscalCobro` tiene las 8 columnas DR del CP. Finanzas puede hacer el `UPDATE` de snapshot tras timbrar exitosamente un CP.

**Entregables:**
- Scripts DDL: 8 `ALTER TABLE ADD COLUMN` + 1 `ADD CONSTRAINT FK`
- Script de validación (`sys.columns`)

**Scripts:**

```sql
-- Prerequisito: catFormaPagoSAT debe existir (Tarea 1)
-- Prerequisito: fccDocumentoFiscalCobro (RE-028 T3) y columnas Perú (RE-029 T3) deben existir
-- Ejecutar en ProquifaDotNet

-- 1. Fecha y hora del pago (snapshot nodo Pago del XML)
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
        -- NULL para líneas no-CP.
GO

-- 5. Saldo anterior (ImpSaldoAnt del DoctoRelacionado — en MonedaDR)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [ImpSaldoAnt] decimal(18,6) NULL;
        -- Saldo de la factura PPD antes de este cobro.
        -- Primer CP: igual al Total de la factura PPD.
        -- CPs subsecuentes: ImpSaldoInsoluto del CP anterior.
        -- NULL para líneas no-CP.
GO

-- 6. Importe pagado (ImpPagado del DoctoRelacionado — en MonedaDR)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [ImpPagado] decimal(18,6) NULL;
        -- Porción del cobro aplicada a esta factura. En MonedaDR.
        -- NULL para líneas no-CP.
GO

-- 7. Saldo insoluto (ImpSaldoInsoluto del DoctoRelacionado — en MonedaDR)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [ImpSaldoInsoluto] decimal(18,6) NULL;
        -- ImpSaldoAnt - ImpPagado. Inmutable tras timbrado.
        -- NULL para líneas no-CP.
GO

-- 8. Equivalencia DR (EquivalenciaDR del DoctoRelacionado)
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [EquivalenciaDR] decimal(18,6) NULL;
        -- Factor de conversión cuando MonedaDR ≠ MonedaP.
        -- Valor 1 cuando las monedas coinciden (obligatorio en el XML).
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

-- Validación FK
SELECT name AS Constraint_FK
FROM sys.foreign_keys
WHERE name = 'FK_fccDocumentoFiscalCobro_FormaPagoSAT';
```

**Criterios de aceptación:**
- Las 8 columnas existen en `fccDocumentoFiscalCobro` con los tipos correctos y `is_nullable=1`.
- FK `FK_fccDocumentoFiscalCobro_FormaPagoSAT` existe y referencia `catFormaPagoSAT.IdCatFormaPagoSAT`.
- Filas existentes no fueron modificadas (NULL en columnas nuevas).

**Más información de la tarea:**
Ver sección *"ALTER TABLE fccDocumentoFiscalCobro"* en `R16A-RE-FU-030_BD.md` y sección *"Parte A / A3"* en `R16A-RE-FU-030-Back.md`.

**Recursos:**
- `R16A-RE-FU-030_BD.md` — ALTER TABLE fccDocumentoFiscalCobro
- `R16A-RE-FU-030-Back.md` — Parte A, sección A3

---

## TAREA 4

**[ RE-FU-030 ] [UPDATE-TABL-CH] DML EmpresaFolio: INSERT 4 filas Serie "P" para GOL, MUN, PRO, PQF**

**Aplicativos:** ProquifaDotNetTimbrado

**Módulos:** Base de Datos — Timbrado — Foliador Complemento de Pago México

**Consideraciones previas:**
- La tabla `EmpresaFolio` existe desde RE-019 con las filas de Series de Facturas para las 4 empresas México. RE-030 agrega filas con Serie `'P'` (Complemento de Pago).
- La tabla `Empresa` en ProquifaDotNetTimbrado se filtra por `Prefijo` (confirmado: no existe columna `Region`). Los Prefijos México son `GOL`, `MUN`, `PRO`, `PQF`; Perú es `GOLPERU`.
- ⚠️ **Pendiente (Brecha B5 / Pendiente P2):** El formato definitivo del folio (`FormatoFolio`, `LongitudMaxima`) está pendiente de validar con PMO (Regla 12 del requisito). Valores propuestos: `FormatoFolio='P{folio:00000000}'`, `LongitudMaxima=8`. Ajustar al validar.
- Verificar que no existan filas previas con `Serie='P'` para estas empresas antes de insertar.

**Objetivo general:**
Registrar el foliador Serie "P" para las 4 empresas México en `EmpresaFolio`, habilitando el consumo atómico (UPDLOCK) que ProquifaDotNet.Timbrado ejecutará al generar cada Complemento de Pago.

**Objetivos específicos:**
- Verificar ausencia de filas `Serie='P'` para GOL, MUN, PRO, PQF.
- Insertar las 4 filas con `UltimoFolio=0` y `Activo=1`.
- Validar que la consulta de validación retorna exactamente 4 filas.

**Resultado esperado:**
`EmpresaFolio` contiene las 4 filas Serie "P" para México. ProquifaDotNet.Timbrado puede consumir el folio atómicamente al timbrar cada CP.

**Entregables:**
- Script DML: 4 `INSERT INTO EmpresaFolio` + script de validación

**Scripts:**

```sql
-- Prerequisito: EmpresaFolio y las 4 empresas México deben existir (RE-019)
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
WHERE e.Prefijo = 'MUN';

-- Proquifa México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa,
       'P',
       0,
       'P{folio:00000000}',  -- ⚠️ Formato pendiente confirmar con PMO
       8,
       1
FROM dbo.Empresa e
WHERE e.Prefijo = 'PRO';

-- Proveedora Quimico Farmaceutica México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa,
       'P',
       0,
       'P{folio:00000000}',  -- ⚠️ Formato pendiente confirmar con PMO
       8,
       1
FROM dbo.Empresa e
WHERE e.Prefijo = 'PQF';
GO

-- Validación: 4 filas Serie P, UltimoFolio=0
SELECT e.Prefijo, ef.Serie, ef.UltimoFolio, ef.FormatoFolio, ef.LongitudMaxima, ef.Activo
FROM dbo.EmpresaFolio ef
INNER JOIN dbo.Empresa e ON ef.IdEmpresa = e.IdEmpresa
WHERE ef.Serie = 'P'
ORDER BY e.Prefijo;
```

**Criterios de aceptación:**
- `SELECT * FROM EmpresaFolio WHERE Serie='P'` retorna exactamente 4 filas.
- Cada fila corresponde a una empresa diferente (GOL, MUN, PRO, PQF) con `UltimoFolio=0`, `Activo=1`.
- Las filas de Series de Facturas existentes no fueron modificadas.
- ⚠️ `FormatoFolio` y `LongitudMaxima` documentados como propuesta pendiente de validación con PMO.

**Más información de la tarea:**
Ver sección *"DML EmpresaFolio"* en `R16A-RE-FU-030_BD.md` y sección *"Parte A / A4"* en `R16A-RE-FU-030-Back.md`.

**Recursos:**
- `R16A-RE-FU-030_BD.md` — DML EmpresaFolio

---

## TAREA 5

**[ RE-FU-030 ] [UPDATE-TABL-CH] DML DocumentTemplate: INSERT 4 templates PDF Complemento de Pago México**

**Aplicativos:** DocumentBuilder

**Módulos:** Base de Datos — DocumentBuilder — Templates Complemento de Pago México

**Consideraciones previas:**
- La tabla `DocumentTemplate` existe en la base de datos `DocumentBuilder` (no en ProquifaDotNet). Estructura confirmada: `IdDocumentTemplate` (PK auto), `TemplateKey` (varchar 100), `HeaderTemplateFileName`, `BodyTemplateFileName`, `FooterTemplateFileName`, `HasHeaderTemplate`, `HasBodyTemplate`, `HasFooterTemplate`, `RegistrationDate`, `LastUpdateDate`, `IsActive`.
- Convención de nombres confirmada con datos existentes: `{TemplateKey}_{H/B/F}.html`. Los 3 archivos siempre se usan (`HasHeader/Body/Footer = 1`).
- Los 4 `TemplateKey` son: `GOL_MEX_CP`, `MUN_MEX_CP`, `PRO_MEX_CP`, `PQF_MEX_CP`.
- Los archivos HTML (H/B/F) se crean en la Tarea 11. Esta tarea solo registra los metadatos en BD.
- Verificar que no existan los TemplateKey antes de insertar.

**Objetivo general:**
Registrar los 4 templates del Complemento de Pago México en `DocumentBuilder.DocumentTemplate` para que `PersistPaymentComplementPdfService` pueda resolver el `TemplateKey` correcto por empresa emisora al generar el PDF.

**Objetivos específicos:**
- Verificar ausencia de los 4 `TemplateKey` en `DocumentTemplate`.
- Insertar las 4 filas con archivos `{TemplateKey}_H.html`, `{TemplateKey}_B.html`, `{TemplateKey}_F.html` y `HasHeader/Body/Footer=1`.
- Validar que los 4 registros existen y están activos.

**Resultado esperado:**
`DocumentBuilder.DocumentTemplate` contiene los 4 templates CP México. `PersistPaymentComplementPdfService` puede invocar DocumentBuilder con `TemplateKey={Prefijo}_MEX_CP`.

**Entregables:**
- Script DML (en DocumentBuilder): 4 `INSERT INTO DocumentTemplate` + script de validación

**Scripts:**

```sql
-- Ejecutar en DocumentBuilder
-- Verificar ausencia antes de insertar
SELECT TemplateKey FROM dbo.DocumentTemplate
WHERE TemplateKey IN ('GOL_MEX_CP','MUN_MEX_CP','PRO_MEX_CP','PQF_MEX_CP');
-- Debe retornar 0 filas antes de continuar

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

-- Validación: 4 registros activos con Has*=1
SELECT TemplateKey, BodyTemplateFileName,
       HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, IsActive
FROM dbo.DocumentTemplate
WHERE TemplateKey IN ('GOL_MEX_CP','MUN_MEX_CP','PRO_MEX_CP','PQF_MEX_CP')
ORDER BY TemplateKey;
```

**Criterios de aceptación:**
- `SELECT TemplateKey, IsActive FROM DocumentTemplate WHERE TemplateKey IN ('GOL_MEX_CP','MUN_MEX_CP','PRO_MEX_CP','PQF_MEX_CP')` retorna 4 filas con `IsActive=1`.
- Cada fila tiene `HasHeaderTemplate=1`, `HasBodyTemplate=1`, `HasFooterTemplate=1`.
- Los `BodyTemplateFileName` siguen la convención `{TemplateKey}_B.html`.

**Más información de la tarea:**
Ver sección *"DML DocumentTemplate"* en `R16A-RE-FU-030_BD.md` y sección *"Parte A / A5"* en `R16A-RE-FU-030-Back.md`.

**Recursos:**
- `R16A-RE-FU-030_BD.md` — DML DocumentTemplate
- `R16A-RE-FU-030-Back.md` — Parte A, sección A5

---

## TAREA 6

**[ RE-FU-030 ] [CREATE-SCRIPT-CONTROL] Actualizar vista vfccDocumentoFiscalCobro v3.0: columnas DR + JOIN catFormaPagoSAT**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 3 — Vista consolidada

**Consideraciones previas:**
- La vista `vfccDocumentoFiscalCobro` fue creada en RE-028 (v1.0) y extendida a v2.0 en RE-029 con JOINs Perú y corrección de resolución de catálogos.
- RE-030 la extiende a v3.0 de forma incremental: se agregan las 8 columnas DR del CP y `LEFT JOIN catFormaPagoSAT`.
- **Prerrequisitos:** Tarea 1 (catFormaPagoSAT) y Tarea 3 (columnas DR en fccDocumentoFiscalCobro) deben estar ejecutadas.
- El script v3.0 debe incluir la definición COMPLETA de v2.0 más los agregados de RE-030. Referencia v2.0: `R16A-RE-FU-029_BD.md`.
- Esta vista es la fuente principal de datos para Finanzas al renderizar el estado del Paso 3.

**Objetivo general:**
Actualizar `vfccDocumentoFiscalCobro` a v3.0 para exponer las 8 columnas del snapshot DR del CP y las columnas de forma de pago (`FormaPagoClave`, `FormaPagoDescripcion`) que Finanzas necesita al construir el PDF del Complemento de Pago.

**Objetivos específicos:**
- Construir el script `ALTER VIEW` completo (v2.0 base + agregados v3.0).
- Agregar `LEFT JOIN catFormaPagoSAT fpago ON p3l.IdCatFormaPagoSAT = fpago.IdCatFormaPagoSAT`.
- Agregar las 8 columnas DR del CP en el SELECT.
- Ejecutar el script y validar con `SELECT TOP 5 ... WHERE TipoDocumentoFiscal = 'COMPLEMENTO_PAGO'`.

**Resultado esperado:**
`vfccDocumentoFiscalCobro` v3.0 expone todas las columnas del Complemento de Pago. Finanzas puede consultar la vista para obtener el estado completo de una línea CP, incluyendo los valores fiscales del DoctoRelacionado.

**Entregables:**
- Script `ALTER VIEW vfccDocumentoFiscalCobro` (v3.0 completo, incluyendo base v2.0)
- Script de validación

**Scripts:**

```sql
-- Prerequisitos: Tarea 1 (catFormaPagoSAT) y Tarea 3 (columnas DR) deben estar ejecutadas
-- Ejecutar en ProquifaDotNet
-- El script completo debe incluir la definición COMPLETA de v2.0 (R16A-RE-FU-029_BD.md)
-- más los siguientes agregados de RE-030:

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

-- Validación post-ALTER
SELECT TOP 5
    TipoDocumentoFiscal, EstadoLinea, ClienteRegion,
    FormaPagoClave, NumParcialidad,
    ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto, EquivalenciaDR
FROM dbo.vfccDocumentoFiscalCobro
WHERE TipoDocumentoFiscal = 'COMPLEMENTO_PAGO';
```

**Criterios de aceptación:**
- La vista compila sin errores.
- `SELECT FormaPagoClave, NumParcialidad, ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto FROM vfccDocumentoFiscalCobro` ejecuta correctamente.
- Las columnas Perú (v2.0) siguen presentes y funcionales.
- `SELECT TOP 5 * FROM vfccDocumentoFiscalCobro WHERE TipoDocumentoFiscal = 'COMPLEMENTO_PAGO'` retorna filas para líneas CP existentes.

**Más información de la tarea:**
Ver sección *"ALTER VIEW vfccDocumentoFiscalCobro v3.0"* en `R16A-RE-FU-030_BD.md` y sección *"Parte A / A6"* en `R16A-RE-FU-030-Back.md`.

**Recursos:**
- `R16A-RE-FU-030_BD.md` — ALTER VIEW v3.0
- `R16A-RE-FU-029_BD.md` — ALTER VIEW v2.0 (base para el script completo)
- `R16A-RE-FU-030-Back.md` — Parte A, sección A6

---

## TAREA 7

**[ RE-FU-030 ] [ALG-COMPLX-LOGIC] Implementar endpoint timbrado CP (CFDI tipo P Pagos20 v2.0) en Timbrado — folio Serie P + TurboPac**

**Aplicativos:** ProquifaDotNet.Timbrado (.NET Core 10)

**Módulos:** Timbrado — Complemento de Pago México

**Consideraciones previas:**
- El endpoint de timbrado genérico (por tipo de CFDI) existe desde RE-019/028. RE-030 agrega el caso `COMPLEMENTO_PAGO` con la construcción del XML CFDI 4.0 Pagos20 v2.0.
- **Prerrequisitos:** Tarea 4 (EmpresaFolio Serie P) debe estar ejecutada para que el foliador tenga filas disponibles.
- La estructura del XML es radicalmente diferente a la Factura: `TipoDeComprobante=P`, `SubTotal=0`, `Total=0`, `Moneda=XXX`, concepto único fijo, sin `Impuestos` en la raíz; los impuestos van dentro del nodo `DoctoRelacionado.ImpuestosDR` (solo si `ObjetoImpDR=02`).
- El consumo del folio es atómico con UPDLOCK sobre `EmpresaFolio WHERE Serie='P'`.
- Datos del `PaymentComplementRequest` que llegan desde Finanzas: todos los valores ya calculados (NumParcialidad, ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto, EquivalenciaDR, FechaPago, IdCatFormaPagoSAT, ImpuestosDR si aplica).
- El manejo de errores PAC sigue el mismo patrón que Facturas (RE-019): si el PAC rechaza, retornar error sin INSERT en CFDIGenerada.

**Objetivo general:**
Implementar en ProquifaDotNet.Timbrado la generación del XML CFDI 4.0 Pagos20 v2.0, el consumo atómico del folio Serie P, la llamada al PAC TurboPac y la persistencia del resultado en `CFDIGenerada` con `IdCatTipoCFDI='COMPLEMENTO_PAGO'` e `IdCFDIRelacionado` al UUID de la factura relacionada.

**Objetivos específicos:**
- Implementar la construcción del XML Pagos20 v2.0 según los criterios B1–G3 del requisito:
  - Cabecera fija: `Version=4.0`, `TipoDeComprobante=P`, `Exportacion=01`, `SubTotal=0`, `Total=0`, `Moneda=XXX`.
  - Emisor: RFC, Nombre, `RegimenFiscal=601`.
  - Receptor: RFC, Nombre, DomicilioFiscal, RegimenFiscal, `UsoCFDI=CP01`.
  - Concepto único fijo: `ClaveProdServ=84111506`, `Cantidad=1`, `ClaveUnidad=ACT`, `Descripcion=Pago`, `ValorUnitario=0`, `Importe=0`, `ObjetoImp=01`.
  - Complemento Pagos20: Totales (MontoTotalPagos + IVA si aplica), 1 Pago con 1 DoctoRelacionado (+ ImpuestosDR si `ObjetoImpDR=02`).
- Implementar consumo atómico del folio: `UPDATE EmpresaFolio WITH (UPDLOCK) SET UltimoFolio = UltimoFolio + 1 OUTPUT inserted.UltimoFolio WHERE IdEmpresa = @Id AND Serie = 'P'`.
- Firmar el XML con el CSD de la empresa emisora y llamar al PAC TurboPac.
- Insertar en `CFDIGenerada` con `IdCatTipoCFDI='COMPLEMENTO_PAGO'`, `IdCFDIRelacionado`, UUID, Serie P, Folio, FechaEmision.
- Insertar en `StampingLog`.
- Retornar a Finanzas: UUID, Serie, Folio, FechaTimbre, XML timbrado con `TimbreFiscalDigital`.

**Resultado esperado:**
`ProquifaDotNet.Timbrado` puede generar un CFDI tipo P (Pagos20 v2.0) válido, timbrado por TurboPac, con folio Serie P consumido atómicamente y resultado registrado en `CFDIGenerada`.

**Entregables:**
- Clase/Handler para construcción de XML CFDI P (Pagos20 v2.0)
- Lógica de consumo de folio Serie P con UPDLOCK
- Endpoint/Command de timbrado CP extendido
- Tests unitarios: construcción XML con y sin ImpuestosDR; escenario MonedaP≠MXN; escenario MonedaDR≠MonedaP

**Criterios de aceptación:**
- El XML generado supera la validación de esquema XSD de CFDI 4.0 y del complemento Pagos20.
- El PAC TurboPac retorna UUID sin error para un CP de prueba.
- `CFDIGenerada` recibe la fila con `IdCatTipoCFDI='COMPLEMENTO_PAGO'` e `IdCFDIRelacionado` correcto.
- El folio en `EmpresaFolio Serie P` se incrementa en 1 por cada timbrado exitoso.
- Un XML con `SubTotal≠0` o `Total≠0` es rechazado por el PAC (validación SAT).

**Más información de la tarea:**
Ver sección *"Parte C / C1"* en `R16A-RE-FU-030-Back.md` y criterios B1–G3 en `R16A-RE-FU-030.md`.

**Recursos:**
- `R16A-RE-FU-030-Back.md` — Parte C, sección C1
- `R16A-RE-FU-030.md` — Secciones B, D, E, F, G (criterios XML)
- `R16A-RE-FU-028-Back.md` — Parte C (patrón base de timbrado)

---

## TAREA 8

**[ RE-FU-030 ] [SERV-TRANSACT] Implementar PaymentComplementPdfMappingService y PersistPaymentComplementPdfService (MinIO cobranza)**

**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10)

**Módulos:** Finanzas — PDF Complemento de Pago México

**Consideraciones previas:**
- Patrón de referencia: `MexicoInvoicePdfMappingService` y `PersistMexicoInvoicePdfService` de RE-021. Seguir la misma estructura.
- `PaymentComplementPdfMappingService` tiene dos modos: `MapearPreviewAsync()` (sin sello digital, NumParcialidad estimado) y `MapearAsync()` (con `TimbreFiscalDigital`, QR de verificación SAT).
- `PersistPaymentComplementPdfService` sube el PDF **y** el XML a MinIO bucket `cobranza` (MEX), resolviendo el bucket a través de `RegionConfiguracionMinioBucket` (BucketClave=`cobranza`, Región=MEX).
- Rutas MinIO: `cobranza/complementos/{anio}/{mes}/{UUID_CP}.pdf` y `.xml`.
- Tras subir, inserta en `Archivo` y actualiza `CFDIGenerada.IdArchivoPdf`.
- La resolución del `TemplateKey` es dinámica: `{Empresa.Prefijo}_MEX_CP` → `GOL_MEX_CP`, `MUN_MEX_CP`, etc. **Prerrequisito:** Tarea 5 (templates en DocumentTemplate).

**Objetivo general:**
Implementar los dos servicios de generación y persistencia del PDF del Complemento de Pago, siguiendo el patrón establecido en RE-021 para Facturas, con soporte para preview (sin sello) y definitivo (con sello + QR), y persistencia en MinIO bucket `cobranza`.

**Objetivos específicos:**
- Implementar `PaymentComplementPdfModel` con todas las secciones del PDF (Emisor, Receptor, Comprobante, Concepto fijo, Totales CP, Pago, DoctoRelacionado, ImpuestosDR, Sellos, QR).
- Implementar `PaymentComplementPdfMappingService.MapearPreviewAsync()` sin `TimbreFiscalDigital`.
- Implementar `PaymentComplementPdfMappingService.MapearAsync()` con `TimbreFiscalDigital` y QR SAT.
- Implementar `PersistPaymentComplementPdfService.PersistirAsync()`:
  - Resolver bucket `cobranza` MEX desde `RegionConfiguracionMinioBucket`.
  - Invocar DocumentBuilder con `TemplateKey={Prefijo}_MEX_CP`.
  - Subir PDF y XML a MinIO.
  - `INSERT Archivo` + `UPDATE CFDIGenerada.IdArchivoPdf`.

**Resultado esperado:**
Finanzas puede generar el PDF del CP en preview (antes de timbrar) y en versión definitiva (post-timbrado) con sellos SAT y QR, y persistirlo en MinIO bucket `cobranza` con el PDF y XML vinculados a `CFDIGenerada`.

**Entregables:**
- Clase `PaymentComplementPdfModel` con todas las secciones requeridas
- Clase `PaymentComplementPdfMappingService` (Preview + Async)
- Clase `PersistPaymentComplementPdfService` (DocumentBuilder + MinIO + Archivo)
- Tests unitarios: resolución de TemplateKey por empresa; mapping de ImpuestosDR presente/ausente; QR encoding correcto

**Criterios de aceptación:**
- El PDF generado incluye las secciones I4–I14 del requisito (Emisor, Receptor, Comprobante, Concepto, Totales, Pago, DoctoRelacionado, ImpuestosDR cuando aplica, Sellos, QR).
- `TemplateKey` se resuelve correctamente: GOL → `GOL_MEX_CP`, MUN → `MUN_MEX_CP`, etc.
- MinIO recibe los archivos en la ruta `cobranza/complementos/{anio}/{mes}/{UUID}.pdf` y `.xml`.
- `CFDIGenerada.IdArchivoPdf` queda actualizado tras persistir.

**Más información de la tarea:**
Ver sección *"Parte B / B6"* y *"Parte E"* en `R16A-RE-FU-030-Back.md` y criterios I4–I14 en `R16A-RE-FU-030.md`.

**Recursos:**
- `R16A-RE-FU-030-Back.md` — Parte B sección B6; Parte E (MinIO)
- `R16A-RE-FU-030.md` — Sección I (criterios PDF)
- `R16A-RE-FU-030_BD.md` — Sección MinIO (bucket cobranza)

---

## TAREA 9

**[ RE-FU-030 ] [IMP-EXIST-SERVICE] Implementar previsualización PDF CP por línea en el Paso 3 (stub pendiente de RE-028 B3)**

**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10)

**Módulos:** Finanzas — Validar Cobro Paso 3 México — Preview Complemento de Pago

**Consideraciones previas:**
- RE-028 B3 (paso 3) documentó explícitamente: *"Para líneas COMPLEMENTO_PAGO: la previsualización del PDF del Complemento se implementa en R16A-RE-FU-030"*. Esta tarea cierra ese stub.
- **Prerrequisito:** Tarea 8 (`PaymentComplementPdfMappingService.MapearPreviewAsync()`) debe estar implementada.
- El preview usa `NumParcialidad` estimado (SELECT COUNT + 1 **sin** UPDLOCK — solo informativo).
- `ImpSaldoAnt` estimado: si primer CP → Total de la factura relacionada; si CPs subsecuentes → `ImpSaldoInsoluto` del CP anterior consultado desde `fccDocumentoFiscalCobro`.
- El preview **no persiste** ningún dato en BD. Solo genera el PDF en memoria y lo retorna.
- El PDF de preview no incluye UUID, sello digital ni QR — se muestra un placeholder visual.

**Objetivo general:**
Implementar el endpoint/handler de previsualización del PDF del Complemento de Pago en el Paso 3 de Validar Cobro, cerrando el stub de RE-028 B3 para líneas de tipo `COMPLEMENTO_PAGO`.

**Objetivos específicos:**
- Leer datos de la línea desde `vfccDocumentoFiscalCobro`.
- Calcular `NumParcialidad` estimado (sin UPDLOCK).
- Calcular `ImpSaldoAnt` estimado según si es primer CP o no.
- Invocar `PaymentComplementPdfMappingService.MapearPreviewAsync()`.
- Generar PDF en memoria vía DocumentBuilder (TemplateKey `{Prefijo}_MEX_CP`).
- Retornar el PDF al frontend sin escrituras en BD.

**Resultado esperado:**
El usuario puede presionar "Previsualizar" en una línea `COMPLEMENTO_PAGO` del Paso 3 y ver el PDF con los datos del CP antes de timbrar, incluyendo valores estimados de parcialidad y saldos.

**Entregables:**
- Endpoint/handler de previsualización CP extendido en el Paso 3
- Tests unitarios: preview primer CP; preview CP subsecuente; verificación de no-escritura en BD

**Criterios de aceptación:**
- El endpoint retorna un PDF en memoria para líneas `COMPLEMENTO_PAGO` del Paso 3.
- El PDF incluye los datos del Emisor, Receptor, Pago y DoctoRelacionado con valores estimados.
- No se insertan ni actualizan filas en BD durante la previsualización.
- El PDF muestra placeholder en lugar de UUID y sello digital.

**Más información de la tarea:**
Ver sección *"Parte B / B1"* en `R16A-RE-FU-030-Back.md`.

**Recursos:**
- `R16A-RE-FU-030-Back.md` — Parte B, sección B1
- `R16A-RE-FU-028-Back.md` — Parte B, sección B3 (contexto del stub)

---

## TAREA 10

**[ RE-FU-030 ] [ALG-COMPLX-LOGIC] Implementar generación automática del CP: cálculo DR, cascada PPD (Escenario B) y CP desde FAA (Escenario D)**

**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10)

**Módulos:** Finanzas — Validar Cobro Paso 3 México — Generación Complemento de Pago

**Consideraciones previas:**
- Esta tarea implementa los pasos que RE-028 B4 dejó pendientes: **Escenario B pasos 4–6** (cascada PPD post-Factura) y **Escenario D pasos 2–3** (CP desde FAA existente).
- **Prerrequisitos:** Tareas 1, 3 y 6 (BD) + Tarea 7 (endpoint timbrado CP en Timbrado) + Tarea 8 (PersistPaymentComplementPdfService).
- El cálculo de `NumParcialidad` **debe** usar UPDLOCK en la misma transacción que el timbrado para evitar duplicados concurrentes.
- `ImpSaldoAnt`: primer CP = Total de la Factura PPD (de `CFDIGenerada`); CPs subsecuentes = `MAX(fccDocumentoFiscalCobro.ImpSaldoInsoluto)` para la misma factura relacionada.
- ⚠️ **Pendiente P1 (Brecha B1):** Hora de `FechaPago` (12:00:00 fija vs hora real). Implementar con `CAST(@FechaCobro AS date) + '12:00:00'` como convención provisional; documentar el TODO en código para ajustar al validar con asesor fiscal.
- **Escenario D diferencias:** `IdCFDIRelacionado` = UUID de la FAA (`fccFactura.IdCFDIGenerada`, RE-FU-015 — antes `tpProformaAdelanto.IdCFDIGenerada`); `NumParcialidad = 1` fijo; `ImpSaldoAnt` = Total de la FAA.
- **Política de fallo:** Si el CP falla tras la Factura PPD timbrada, la Factura PPD permanece vigente; la línea queda en estado `PENDIENTE` para reintento posterior. Hasta que PMO defina la política formal (Brecha B3), no se implementa lógica adicional de reintento automático.

**Objetivo general:**
Implementar la lógica de generación automática del Complemento de Pago en ProquifaDotNet.Finanzas: cálculo de los valores fiscales del DoctoRelacionado, solicitud de timbrado al Timbrado API, persistencia del PDF+XML en MinIO y actualización del snapshot en `fccDocumentoFiscalCobro`, tanto para la cascada PPD (Escenario B) como para el CP desde FAA (Escenario D).

**Objetivos específicos:**
- Implementar `PaymentComplementCalculationService`:
  - `CalcularNumParcialidad()` con UPDLOCK sobre `CFDIGenerada`.
  - `CalcularImpSaldoAnt()`: primer CP vs CPs subsecuentes.
  - `CalcularImpSaldoInsoluto()` = ImpSaldoAnt − ImpPagado.
  - `CalcularEquivalenciaDR()`: 1 si MonedaDR=MonedaP; factor de conversión si difieren.
  - `ConstruirFechaPago()`: fecha del cobro con hora 12:00:00 provisional (TODO documentado).
- Implementar `GeneratePaymentComplementService.GenerarEnCascadaPPDAsync()` (Escenario B):
  - Calcula valores DR.
  - Arma `PaymentComplementRequest` y llama a Timbrado API.
  - Invoca `PersistPaymentComplementPdfService`.
  - `UPDATE fccDocumentoFiscalCobro` con snapshot DR + `IdCFDIGeneradaComplemento` + `EstadoLinea='GENERADO'`.
  - Manejo de fallo: línea queda en PENDIENTE si CP falla.
- Implementar `GeneratePaymentComplementService.GenerarDesdeFacturaAdelantoAsync()` (Escenario D):
  - `NumParcialidad=1`, `ImpSaldoAnt` = Total FAA, misma lógica de persistencia.

**Resultado esperado:**
Al timbrar una Factura PPD en el Paso 3, Finanzas genera automáticamente el CP correspondiente, calcula los valores fiscales correctos, timbra a través de Timbrado API, persiste el PDF y XML en MinIO y actualiza el snapshot en `fccDocumentoFiscalCobro`. Ídem para CPs desde FAA existentes.

**Entregables:**
- Clase `PaymentComplementCalculationService` (cálculo NumParcialidad, saldos, EquivalenciaDR, FechaPago)
- Clase `GeneratePaymentComplementService` (Escenario B: cascada PPD + Escenario D: FAA)
- Integración con `ApiCallerStamping` y `PersistPaymentComplementPdfService`
- Tests unitarios: primer CP vs CP subsecuente; escenario multi-divisa; fallo del CP post-Factura PPD; Escenario D desde FAA

**Criterios de aceptación:**
- `NumParcialidad` se calcula con UPDLOCK; dos cobros concurrentes sobre la misma factura no generan el mismo número.
- `ImpSaldoInsoluto = ImpSaldoAnt - ImpPagado` exacto a 6 decimales.
- `EquivalenciaDR = 1` cuando MonedaDR=MonedaP (valor explícito en el request, no omitido).
- Tras timbrado exitoso del CP: `fccDocumentoFiscalCobro.EstadoLinea = 'GENERADO'`, `IdCFDIGeneradaComplemento` poblado, snapshot DR (8 columnas) completado.
- Si el timbrado del CP falla: la Factura PPD permanece válida; la línea queda en `PENDIENTE`; se loguea el error.
- Escenario D: `NumParcialidad=1`, `ImpSaldoAnt` = Total de la FAA.

**Más información de la tarea:**
Ver secciones *"Parte B / B2, B3, B4"* en `R16A-RE-FU-030-Back.md`. Contexto de Escenarios B y D en `R16A-RE-FU-028-Back.md` sección B4.

**Recursos:**
- `R16A-RE-FU-030-Back.md` — Parte B, secciones B2, B3, B4
- `R16A-RE-FU-030.md` — Reglas de Negocio 2, 7, 8, 9, 10, 11; Criterios D–G
- `R16A-RE-FU-028-Back.md` — Parte B, sección B4 (contexto cascada PPD Escenarios B y D)

---

## TAREA 11

**[ RE-FU-030 ] [CREATE-PDF] Diseñar e implementar templates HTML Complemento de Pago México: GOL/MUN/PRO/PQF_MEX_CP (H/B/F)**

**Aplicativos:** DocumentBuilder

**Módulos:** DocumentBuilder — Templates Complemento de Pago México

**Consideraciones previas:**
- Se requieren 4 conjuntos de 3 archivos HTML cada uno (Header, Body, Footer): `GOL_MEX_CP_H/B/F.html`, `MUN_MEX_CP_H/B/F.html`, `PRO_MEX_CP_H/B/F.html`, `PQF_MEX_CP_H/B/F.html`.
- Los templates `*_MEX_FAC` (RE-021) son la referencia de branding, tipografía, paleta corporativa e iconografía de certificaciones por empresa. El diseño del CP debe ser consistente con las Facturas (Criterio I3 del requisito).
- El PDF debe incluir todas las secciones definidas en los criterios I4–I14 del requisito: Emisor, Receptor, Comprobante, Concepto fijo, Totales CP, sección Pago, DoctoRelacionado con saldos y parcialidad, ImpuestosDR (condicional), Sellos digitales, QR SAT.
- **Prerrequisito:** Tarea 5 (registros en DocumentTemplate) para que DocumentBuilder pueda resolver los templates al ser invocado.
- ⚠️ **Pendiente:** Confirmar vigencia de iconografía de certificaciones del giro (Criterio I2 del requisito).

**Objetivo general:**
Diseñar e implementar los 12 archivos HTML (4 empresas × 3 archivos) de los templates del Complemento de Pago México en DocumentBuilder, con identidad visual consistente con las Facturas México por empresa emisora.

**Objetivos específicos:**
- Diseñar la estructura HTML/CSS de Header, Body y Footer para el CP.
- Header: logo empresa, datos del emisor, folio y UUID.
- Body: datos del receptor, concepto estructural fijo, totales del complemento, sección Pago, sección DoctoRelacionado (con saldos, parcialidad, EquivalenciaDR, impuestos DR condicionales), traslados resumen pago.
- Footer: sellos digitales (NoCertificadoSAT, NoCertificadoCSD, SelloSAT, SelloCFDI, CadenaOriginal), QR de verificación SAT, iconografía de certificaciones.
- Parametrizar el branding por empresa (logo + paleta de colores): Golocaer (naranja), Mungen (verde), Proquifa (cyan), Proveedora Quimico Farmaceutica (cyan).
- Registrar los 12 archivos en DocumentBuilder y verificar generación de PDF de prueba con datos mock.

**Resultado esperado:**
DocumentBuilder puede generar el PDF del Complemento de Pago para cada empresa emisora México, con la identidad visual correcta y todas las secciones fiscales obligatorias (I4–I14 del requisito).

**Entregables:**
- 12 archivos HTML (4 empresas × Header + Body + Footer)
- PDF de prueba generado por DocumentBuilder para cada empresa
- Validación visual de branding e información fiscal

**Criterios de aceptación:**
- DocumentBuilder genera un PDF válido para cada uno de los 4 TemplateKey (`GOL/MUN/PRO/PQF_MEX_CP`).
- El PDF incluye todas las secciones I4–I14: Emisor, Receptor, Comprobante, Concepto fijo, Totales CP, Pago, DoctoRelacionado (con NumParcialidad, ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto, EquivalenciaDR), ImpuestosDR (cuando aplica), Sellos, QR.
- Branding consistente con `*_MEX_FAC`: logo correcto por empresa, paleta de colores por empresa.
- La sección ImpuestosDR se oculta correctamente cuando `ObjetoImpDR=01` (sin impuestos).
- El QR SAT codifica la URL de verificación con los 5 campos requeridos.

**Más información de la tarea:**
Ver sección *"Parte D"* en `R16A-RE-FU-030-Back.md` y criterios I4–I14 en `R16A-RE-FU-030.md`.

**Recursos:**
- `R16A-RE-FU-030-Back.md` — Parte D
- `R16A-RE-FU-030.md` — Sección I, criterios I4–I14
- Templates `*_MEX_FAC` (RE-021) — referencia de branding por empresa
