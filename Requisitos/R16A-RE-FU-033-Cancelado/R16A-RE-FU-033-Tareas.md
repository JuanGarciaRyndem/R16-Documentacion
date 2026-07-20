# Tareas BackEnd — R16A-RE-FU-033
**Requisito:** Notas de Crédito Perú (CPE tipo 07 — UBL 2.1)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder

---

## Resumen de tareas

| #             | Clave             | Título simple                                                                                      | Tipo | Aplicativo              |
| ------------- | ----------------- | -------------------------------------------------------------------------------------------------- | ---- | ----------------------- |
| 1             | UPDATE-TABL-CH    | Extender fccNotaCredito: ADD 3 columnas Perú (ResponseCode, ResponseDescription, TipoCambioOrigen) | BD   | ProquifaDotNet          |
| 2             | CREATE-TABL-CH    | Crear catMotivoCreditoSUNAT09 con DDL + DML 11 motivos catálogo 09 SUNAT                           | BD   | ProquifaDotNet          |
| 3             | UPDATE-TABL-CH    | DML catTipoCFDI: INSERT clave NOTA_CREDITO_PERU (TipoDocumento='07')                               | BD   | ProquifaDotNet          |
| 4             | UPDATE-TABL-CH    | DML EmpresaFolio: INSERT Serie NC Perú para Golocaer S.A.C.                                        | BD   | ProquifaDotNet (Finanzas) |
| 5             | UPDATE-TABL-CH    | DML DocumentTemplate: INSERT template GOL\_PER\_NC                                                 | BD   | DocumentBuilder         |
| 6             | LIST-NO-FILTER    | Endpoint GET /api/v1/creditNoteReasonSunat para catálogo 09                            | Back | ProquifaDotNet.Finanzas |
| 7             | IMP-EXIST-SERVICE | Implementar Wizard Paso 1 y Paso 2: búsqueda CPE origen, captura por partidas y manual             | Back | ProquifaDotNet.Finanzas |
| 8             | ALG-COMPLX-LOGIC  | Implementar endpoint timbrado NC Perú (CPE tipo 07) en Timbrado ⚠️ bloqueado Brecha B1             | Back | ProquifaDotNet.Timbrado |
| 9             | SERV-TRANSACT     | Implementar timbrado NC Perú, persistencia MinIO y correo automático ⚠️ bloqueado Brecha B1        | Back | ProquifaDotNet.Finanzas |

> **Nota:** Las filas marcadas *(→ RE-035)* corresponden a tareas de generación de documentos que pertenecen a **R16A-RE-FU-035**. Se listan aquí para trazabilidad del orden de ejecución.

---

## TAREA 1

**[ RE-FU-033 ] [UPDATE-TABL-CH] Extender fccNotaCredito: ADD 3 columnas Perú**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Notas de Crédito Perú — Tabla principal NC

**Consideraciones previas:**
- **Prerrequisito:** R16A-RE-FU-032 T1 debe estar ejecutado (columnas genéricas de NC ya existen en `fccNotaCredito`).
- ⚠️ **Brecha B3 heredada:** `fccNotaCredito.IdTPProformaPedido` es NOT NULL. Si RE-032 B1 no fue resuelta, esta tarea también está bloqueada. Confirmar antes de ejecutar.
- Las 3 columnas son exclusivas de la NC Perú: no tienen equivalente en la NC México. Los registros MEX tendrán estos campos en NULL.
- `TipoCambioOrigen` es la diferencia fiscal clave de Perú: la NC hereda el tipo de cambio de la fecha de emisión de la factura origen (no el del día del timbrado), conforme a la normativa SUNAT.
- `ResponseCode` referencia lógicamente `catMotivoCreditoSUNAT09.Clave` (sin FK formal — misma decisión de diseño que `catMotivoCancelacionSAT` en RE-032).

**Objetivo general:**
Agregar las 3 columnas específicas de Perú a `fccNotaCredito` para persistir el motivo catálogo 09, el sustento del motivo y el tipo de cambio heredado de la factura origen.

**Objetivos específicos:**
- Agregar `ResponseCode varchar(2) NULL` — código catálogo 09 SUNAT.
- Agregar `ResponseDescription nvarchar(500) NULL` — sustento cbc:Description.
- Agregar `TipoCambioOrigen decimal(18,6) NULL` — TC heredado de la factura origen.
- Validar las 3 columnas.

**Resultado esperado:**
`fccNotaCredito` contiene las 3 columnas Perú. Finanzas puede persistir el motivo, sustento y tipo de cambio de una NC CPE.

**Entregables:**
- Script DDL: 3 `ALTER TABLE fccNotaCredito ADD` + script de validación

**Scripts:**

```sql
-- Ejecutar en ProquifaDotNet
-- Prerrequisito: RE-032 T1 ejecutado + Brecha B3 resuelta

ALTER TABLE dbo.fccNotaCredito ADD [ResponseCode] varchar(2) NULL;
    -- Clave catálogo 09 SUNAT (cbc:ResponseCode). Ej: '01','06','04'.
    -- NULL para registros MEX.
GO
ALTER TABLE dbo.fccNotaCredito ADD [ResponseDescription] nvarchar(500) NULL;
    -- Sustento del motivo (cbc:Description). Obligatorio en XML UBL 2.1.
    -- NULL para registros MEX.
GO
ALTER TABLE dbo.fccNotaCredito ADD [TipoCambioOrigen] decimal(18,6) NULL;
    -- TC de la fecha de emisión de la factura origen (Perú).
    -- Diferencia clave vs MEX: en Perú la NC hereda el TC de la operación original.
    -- NULL si moneda=PEN o para registros MEX.
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

**Criterios de aceptación:**
- Las 3 columnas existen con los tipos correctos y son nullable.
- Los registros pre-existentes (incluyendo NCs México) no fueron afectados.

**Más información de la tarea:**
Ver sección *"ALTER TABLE fccNotaCredito — Columnas Perú"* en `R16A-RE-FU-033_BD.md`.

**Recursos:**
- `R16A-RE-FU-033_BD.md`
- `R16A-RE-FU-032-Tareas.md` — T1 (columnas genéricas, prerrequisito)

---

## TAREA 2

**[ RE-FU-033 ] [CREATE-TABL-CH] Crear catMotivoCreditoSUNAT09 con DDL + DML 11 motivos**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogos SUNAT — Motivos NC Perú

**Consideraciones previas:**
- Tabla nueva. Equivalente peruano de `catMotivoCancelacionSAT` (RE-032) pero con 11 claves y columna adicional `Modalidad`.
- La columna `Modalidad ('POR_PARTIDAS'|'MANUAL')` es clave: el frontend la usa para determinar automáticamente qué formulario mostrar al seleccionar el motivo, sin lógica hardcoded en el cliente.
- El motivo `01` (Anulación) es `POR_PARTIDAS` con todas las partidas pre-cargadas no editables.
- ⚠️ Pendiente confirmar con el cliente cuáles de los 11 motivos se habilitan en R16. El campo `Activo` permite desactivar motivos sin eliminar registros.
- Es prerrequisito del endpoint de Tarea 6.

**Objetivo general:**
Crear el catálogo `catMotivoCreditoSUNAT09` con las 11 claves del catálogo 09 SUNAT para que Finanzas pueda exponer los motivos al frontend y el usuario pueda seleccionar el motivo de la NC.

**Objetivos específicos:**
- Ejecutar `CREATE TABLE catMotivoCreditoSUNAT09` con PK, UNIQUE en `Clave`, CHECK en `Modalidad`.
- Insertar los 11 motivos con su `Modalidad` correcta.
- Validar 11 filas con `Activo=1`.

**Resultado esperado:**
`catMotivoCreditoSUNAT09` existe con 11 claves activas y columna `Modalidad` correcta. El endpoint T6 puede consultarla.

**Entregables:**
- Script DDL + DML: `CREATE TABLE catMotivoCreditoSUNAT09` + 11 INSERT + validación

**Scripts:**

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

**Criterios de aceptación:**
- Tabla existe con PK y UNIQUE en `Clave`.
- 11 filas con `Activo=1` y `Modalidad` correcta por motivo.
- CHECK constraint `Modalidad` activo.

**Más información de la tarea:**
Ver sección *"CREATE TABLE catMotivoCreditoSUNAT09"* en `R16A-RE-FU-033_BD.md`.

**Recursos:**
- `R16A-RE-FU-033_BD.md`

---

## TAREA 3

**[ RE-FU-033 ] [UPDATE-TABL-CH] DML catTipoCFDI: INSERT clave NOTA_CREDITO_PERU**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogos CFDI — TipoCFDI

**Consideraciones previas:**
- **Prerrequisito:** `catTipoCFDI` creada por RE-028 T1. Confirmar estructura antes de ejecutar.
- `TipoDocumentoSAT='07'` (CPE tipo 07 SUNAT). El campo se llama `TipoDocumentoSAT` por convención del proyecto aunque para Perú es un código SUNAT.
- RE-032 ya insertó `NOTA_CREDITO` (TipoDocumento='E'). Esta tarea inserta `NOTA_CREDITO_PERU` (TipoDocumento='07') — son claves distintas.

**Objetivo general:**
Registrar la clave `NOTA_CREDITO_PERU` en `catTipoCFDI` para que Finanzas y Timbrado puedan discriminar las NCs Perú en `CFDIGenerada`.

**Entregables:**
- Script DML: SELECT verificación + INSERT + SELECT validación

**Scripts:**

```sql
-- Prerrequisito: catTipoCFDI debe existir (RE-028 T1)
-- Ejecutar en ProquifaDotNet

SELECT * FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO_PERU';
-- 0 filas antes de continuar

INSERT INTO dbo.catTipoCFDI (IdCatTipoCFDI, Clave, Descripcion, TipoDocumentoSAT, Activo)
SELECT NEWID(), 'NOTA_CREDITO_PERU', 'Nota de Crédito Electrónica Perú (CPE tipo 07)', '07', 1
WHERE NOT EXISTS (SELECT 1 FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO_PERU');
GO

-- Validación
SELECT IdCatTipoCFDI, Clave, Descripcion, TipoDocumentoSAT, Activo
FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO_PERU';
-- 1 fila con TipoDocumentoSAT='07' y Activo=1
```

**Criterios de aceptación:**
- 1 fila `Clave='NOTA_CREDITO_PERU'`, `TipoDocumentoSAT='07'`, `Activo=1`.
- No afecta la clave `NOTA_CREDITO` (México RE-032) ni otras existentes.

**Más información de la tarea:**
Ver sección *"DML catTipoCFDI"* en `R16A-RE-FU-033_BD.md`.

**Recursos:**
- `R16A-RE-FU-033_BD.md`
- `R16A-RE-FU-028-Tareas.md` — T1 (catTipoCFDI creado ahí)

---

## TAREA 4

**[ RE-FU-033 ] [UPDATE-TABL-CH] DML EmpresaFolio: INSERT Serie NC Perú para Golocaer S.A.C.**

**Aplicativos:** ProquifaDotNetTimbrado

**Módulos:** Base de Datos — Foliador NC Perú

**Consideraciones previas:**
- Solo Golocaer S.A.C. emite NCs Perú (a diferencia de México que tiene 4 empresas).
- ⚠️ Serie y formato del folio pendientes de validar con PMO (Regla 9 — candidato: Serie='FC01', correlativo 8 dígitos: `FC01{folio:00000000}`).
- El foliador usa UPDLOCK atómico en Timbrado — misma mecánica que RE-030 T4 y RE-032 T5.
- La empresa Golocaer S.A.C. puede tener ya una Serie de factura Perú (RE-029 T?). Verificar que no colisione la clave.

**Objetivo general:**
Registrar la fila de Serie NC Perú en `EmpresaFolio` para que Timbrado pueda asignar folios consecutivos a las NCs de Golocaer S.A.C.

**Entregables:**
- Script DML: INSERT + script de validación

**Scripts:**

```sql
-- Prerrequisito: EmpresaFolio existe (RE-019), Golocaer S.A.C. y Region PER existen
-- Ejecutar en ProquifaDotNetTimbrado
-- ⚠️ Validar Serie y FormatoFolio con PMO antes de ejecutar

-- Verificar ausencia
SELECT e.Prefijo, ef.Serie FROM dbo.EmpresaFolio ef
INNER JOIN dbo.Empresa e ON ef.IdEmpresa = e.IdEmpresa
WHERE e.Prefijo = 'GOL' AND ef.Serie LIKE 'FC%';
-- 0 filas antes de continuar

-- Golocaer S.A.C. Perú — NC Serie FC01
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa, 'FC01', 0, 'FC01{folio:00000000}', 12, 1
FROM dbo.Empresa e
INNER JOIN dbo.Region r ON e.IdRegion = r.IdRegion
WHERE e.Prefijo = 'GOL' AND r.Clave = 'PER';
GO

-- Validación
SELECT e.Prefijo, ef.Serie, ef.UltimoFolio, ef.FormatoFolio, ef.LongitudMaxima, ef.Activo
FROM dbo.EmpresaFolio ef
INNER JOIN dbo.Empresa e ON ef.IdEmpresa = e.IdEmpresa
WHERE e.Prefijo = 'GOL' AND ef.Serie LIKE 'FC%';
-- Debe retornar 1 fila con UltimoFolio=0
```

**Criterios de aceptación:**
- 1 fila Serie 'FC01' con `UltimoFolio=0` y `Activo=1` para Golocaer S.A.C. Perú.
- No colisiona con series de Factura Perú (RE-029).

**Más información de la tarea:**
Ver sección *"DML EmpresaFolio"* en `R16A-RE-FU-033_BD.md`.

**Recursos:**
- `R16A-RE-FU-033_BD.md`

---

## TAREA 5

**[ RE-FU-033 ] [UPDATE-TABL-CH] DML DocumentTemplate: INSERT template GOL_PER_NC**

**Aplicativos:** DocumentBuilder

**Módulos:** Base de Datos — DocumentBuilder — Template NC Perú

**Consideraciones previas:**
- Solo 1 template (Golocaer S.A.C.) a diferencia de los 4 de México (RE-032 T6).
- Misma convención de nombres: `{TemplateKey}_{H/B/F}.html` → `GOL_PER_NC_H.html`, `GOL_PER_NC_B.html`, `GOL_PER_NC_F.html`.
- Los archivos HTML se crean en la Tarea 11. Esta tarea solo registra metadatos.
- Prerrequisito de RE-035 T3 (CreditNotePeruPdfMappingService necesita el TemplateKey).

**Objetivo general:**
Registrar el template `GOL_PER_NC` en `DocumentBuilder.DocumentTemplate` para que `PersistPeruCreditNotePdfService` resuelva el template correcto.

**Entregables:**
- Script DML: INSERT + validación

**Scripts:**

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
-- 1 fila con Has*=1 e IsActive=1
```

**Criterios de aceptación:**
- 1 registro activo con `IsActive=1`, `Has*Template=1`, archivos con convención `GOL_PER_NC_{H/B/F}.html`.

**Más información de la tarea:**
Ver sección *"DML DocumentTemplate"* en `R16A-RE-FU-033_BD.md`.

**Recursos:**
- `R16A-RE-FU-033_BD.md`

---

## TAREA 6

**[ RE-FU-033 ] [LIST-NO-FILTER] Endpoint GET /api/v1/creditNoteReasonSunat**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Catálogos — Motivos NC SUNAT — API Finanzas

**Consideraciones previas:**
- Endpoint de solo lectura. Retorna los motivos activos del `catMotivoCreditoSUNAT09` con su `Modalidad`.
- El frontend usa `Modalidad` para determinar qué formulario mostrar en el Paso 2 sin lógica hardcoded.
- El frontend envía la `Clave` del motivo de regreso al backend (no el GUID).
- Prerrequisito: Tarea 2.

**Objetivo general:**
Exponer el catálogo 09 SUNAT vía API para que el frontend construya el selector de motivo y determine automáticamente la modalidad de captura.

**Objetivos específicos:**
- `GET /api/v1/creditNoteReasonSunat`: lista activos por Clave ASC.
- DTO: `SunatCreditReasonDto { Clave: string, Descripcion: string, Modalidad: string }`.
- Query: `catMotivoCreditoSUNAT09 WHERE Activo=1 ORDER BY Clave`.

**Resultado esperado:**
Frontend obtiene los motivos con su `Modalidad` y activa el formulario por partidas o manual según corresponda.

**Entregables:**
- Endpoint `GET /api/v1/creditNoteReasonSunat`
- `SunatCreditReasonDto`

**Ejemplo de respuesta:**
```json
[
  { "clave": "01", "descripcion": "Anulación de la operación",       "modalidad": "POR_PARTIDAS" },
  { "clave": "04", "descripcion": "Descuento global",                 "modalidad": "MANUAL" },
  { "clave": "06", "descripcion": "Devolución total",                 "modalidad": "POR_PARTIDAS" }
]
```

**Criterios de aceptación:**
- HTTP 200 con los motivos activos ordenados por `Clave`.
- La respuesta incluye el campo `modalidad` (no expone el GUID interno).
- Un motivo desactivado (`Activo=0`) no aparece en la respuesta.

**Más información de la tarea:**
Ver sección *"Parte B / B1"* en `R16A-RE-FU-033-Back.md`.

**Recursos:**
- `R16A-RE-FU-033-Back.md` — Parte B, sección B1
- Endpoint análogo: `GET /api/v1/cancellationReason` (RE-032 T13)

---

## TAREA 7

**[ RE-FU-033 ] [IMP-EXIST-SERVICE] Implementar Wizard Paso 1 y Paso 2: búsqueda CPE origen, captura por partidas y manual**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Notas de Crédito Perú — Wizard Pasos 1 y 2

**Consideraciones previas:**
- Reutiliza la estructura del Wizard NC México (RE-032 T8) adaptando los campos Perú.
- **Paso 1:** facturas elegibles = CPEs tipo 01 (Factura) vigentes de Golocaer S.A.C., cliente prepago Región Perú, sin NC de anulación previa. El estado "vigente" del CPE depende de la definición de RE-029.
- **Paso 2 por partidas (motivos 01,02,03,05,06,07):**
  - Hereda `CFDIGeneradaConcepto` del CPE origen.
  - Motivo 01: todas las partidas pre-cargadas con `CantidadNC = CantidadFacturada`, no editables.
  - Motivos 05/07: el usuario edita `CantidadNC`. Partidas con `CantidadNC=0` se excluyen.
  - Cálculo: `IGV = Subtotal × 0.18`, `Total = Subtotal + IGV`.
- **Paso 2 manual (motivos 04,08,09,10,13):** monto libre, concepto obligatorio (mapea a `cbc:Description`).
- `TipoCambioOrigen`: si la factura origen está en moneda distinta a PEN, se lee el TC de la fecha de emisión del CPE origen.
- ⚠️ Pendiente P3: origen del texto `cbc:Description` (auto-generado o capturado). Implementar como capturado hasta confirmar.

**Objetivo general:**
Implementar los endpoints del Wizard Paso 1 y Paso 2 de la NC Perú, incluyendo la carga dinámica de la tabla de partidas del CPE origen y los cálculos de IGV.

**Objetivos específicos:**
- `GET /api/v1/client/{id}/eligibleInvoice` — CPEs tipo 01 vigentes prepago Golocaer S.A.C. Perú.
- `GET /api/v1/cfdi/{id}/lineItem` — partidas del CPE origen para modalidad por partidas.
- `POST /api/v1/creditNote/draft` — persiste `fccNotaCredito` + `fccNotaCreditoPartida` (si aplica) en estado PENDIENTE.
- Lógica motivo 01: pre-carga todas las partidas, campo no editable.
- Validaciones: monto manual ≤ Total CPE origen; concepto obligatorio en modalidad manual.

**Resultado esperado:**
El frontend puede construir el Wizard Paso 1 y Paso 2 completo de NC Perú.

**Entregables:**
- Endpoints GET facturas elegibles, GET partidas, POST borrador
- Lógica de modalidad por motivo (reutiliza catMotivoCreditoSUNAT09.Modalidad)
- Unit tests de validaciones

**Criterios de aceptación:**
- Solo CPEs tipo 01 vigentes Golocaer S.A.C. Perú aparecen en el Paso 1.
- El motivo 01 pre-carga todas las partidas con `CantidadNC = CantidadFacturada` no editable.
- El monto manual no puede superar el Total del CPE origen.
- El concepto en modalidad manual es obligatorio para avanzar al Paso 3.
- `TipoCambioOrigen` se lee de la fecha de emisión de la factura origen (no del día actual).

**Más información de la tarea:**
Ver secciones *"Parte B / B2, B3, B4"* en `R16A-RE-FU-033-Back.md`.

**Recursos:**
- `R16A-RE-FU-033-Back.md` — Parte B, secciones B2–B4
- `R16A-RE-FU-032-Tareas.md` — T8 (patrón Wizard NC México)

---

## TAREA 8

**[ RE-FU-033 ] [ALG-COMPLX-LOGIC] Implementar endpoint timbrado NC Perú en Timbrado ⚠️ BLOQUEADO B1**

> *(Tarea original T9 — renumerada a T8 tras mover CreditNotePeruXmlBuilder y template a RE-035)*

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Timbrado — Nota de Crédito Perú — CPE tipo 07

**Consideraciones previas:**
- ⚠️ **Brecha B1 bloqueante:** La modalidad de emisión electrónica ante SUNAT (OSE/directa) no está definida. Este endpoint no puede implementarse hasta que RE-029 resuelva la brecha de timbrado SUNAT.
- Mismo patrón que el endpoint de timbrado Factura Perú (RE-029 T5/T9) — una vez resuelta la brecha.
- Folio con UPDLOCK atómico sobre `EmpresaFolio` Serie NC Perú (prereq: Tarea 4).
- INSERT `CFDIGenerada` con `TipoDocumento='07'`, `IdCatTipoCFDI`=NOTA_CREDITO_PERU (prereq: Tarea 3).

**Objetivo general:**
Ampliar el `StampingController` (endpoint de Notas de Crédito `POST /api/v1/stamp/credit-note` creado en RE-FU-018) para que, al recibir `CreditNotePeruStampingRequest` de Finanzas, envíe el XML UBL 2.1 al proveedor SUNAT, persista el resultado en `CFDIGenerada` y retorne la constancia.

**Objetivos específicos:**
- Recibir y validar `CreditNotePeruStampingRequest`.
- Obtener folio UPDLOCK atómico Serie NC Perú.
- Enviar XML al proveedor SUNAT (pendiente modalidad B1).
- INSERT `CFDIGenerada` con TipoDocumento='07', folio, serie, constancia SUNAT.
- UPDATE `EmpresaFolio.UltimoFolio`.
- Retornar `CreditNotePeruStampingResponse` con constancia a Finanzas.
- Manejo de errores SUNAT.

**Resultado esperado:**
Endpoint de Notas de Crédito (`POST /api/v1/stamp/credit-note`) funcional que timbra NCs Perú con folios consecutivos únicos y constancia SUNAT.

**Entregables:**
- Ampliación de `StampingController` (`POST /api/v1/stamp/credit-note`) para el flujo de Nota de Crédito Perú
- `CreditNotePeruStampingRequest` / `CreditNotePeruStampingResponse` DTOs
- Unit tests del servicio de construcción del XML a enviar al proveedor

**Criterios de aceptación:**
- NC timbrada tiene constancia SUNAT con número de orden (cuando proveedor esté definido).
- `EmpresaFolio.UltimoFolio` incrementa atómicamente sin colisiones.
- Error SUNAT retorna detalle sin registros en `CFDIGenerada`.

**Más información de la tarea:**
Ver sección *"Parte C / C1"* en `R16A-RE-FU-033-Back.md`.

**Recursos:**
- `R16A-RE-FU-033-Back.md` — Parte C
- Endpoint Factura Perú (RE-029) — patrón base una vez resuelta B1

---

## TAREA 9

**[ RE-FU-033 ] [SERV-TRANSACT] Timbrado NC Perú, persistencia MinIO y correo automático ⚠️ BLOQUEADO B1**

> *(Tarea original T10 — renumerada a T9 tras mover CreditNotePeruXmlBuilder y template a RE-035)*

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Notas de Crédito Perú — Wizard Paso 3 acción Timbrar + Persistencia + Correo

**Consideraciones previas:**
- ⚠️ **Brecha B1 bloqueante:** Depende de T8 (endpoint Timbrado definido).
- `PersistPeruCreditNotePdfService` sigue el patrón de `PersistMexicoCreditNotePdfService` (RE-032 T10) con rutas MinIO Perú.
- Bucket MinIO Perú: `RegionConfiguracionMinioBucket` con `Region='PER'`, `BucketClave='notas_credito'` (⚠️ verificar — Pendiente P4).
- Rutas MinIO: `notas-credito-per/notas_credito/{anio}/{mes}/{serie}-{correlativo}.pdf/.xml`.
- **Sin cancelación SAT** — la anulación en Perú es vía NC motivo 01 (no hay llamada a cancelación separada).
- Correo automático vía ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo — regla 7): plantilla NC Perú pendiente de PMO (Pendiente P9).
- Prerrequisitos: Tareas 1, 7, RE-035 T1–T3, 8.

**Objetivo general:**
Implementar el flujo completo de Finanzas post-timbrado NC Perú: llamada a Timbrado, generación PDF final, subida a MinIO, actualización de BD y envío automático del correo al cliente.

**Objetivos específicos:**
- `POST /api/v1/creditNote/{id}/stamp` — orquesta la secuencia completa.
- Llamada al endpoint de Notas de Crédito de Timbrado (`POST /api/v1/stamp/credit-note`, vía `ApiCallerStamping.StampCreditNoteAsync`).
- `PersistPeruCreditNotePdfService.PersistirAsync()`: DocumentBuilder → MinIO → INSERT Archivo × 2 → UPDATE `fccNotaCredito` (Estado='VIGENTE', IdArchivoPdf, IdArchivoXml, IdCFDIGenerada) → INSERT CFDIGeneradaConcepto (si por partidas) → INSERT fccNotaCreditoPartida.
- Envío de correo vía ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo, regla 7) con INSERT `CorreoEnviado` + `ArchivoCorreoEnviado`.
- Registro del guardado de la NC en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo, regla 8).
- Navegación al Paso 4 (NC Emitida).

**Resultado esperado:**
Al presionar "Timbrar" en el Paso 3, la NC queda en estado VIGENTE, el PDF y XML están en MinIO y el correo es enviado al cliente.

**Entregables:**
- Endpoint POST timbrar
- `PersistPeruCreditNotePdfService`
- Flujo de correo automático
- Integration tests del flujo completo

**Criterios de aceptación:**
- NC timbrada con `Estado='VIGENTE'`, `IdCFDIGenerada`, `IdArchivoPdf`, `IdArchivoXml` poblados.
- PDF y XML accesibles en MinIO en la ruta `notas-credito-per/notas_credito/{anio}/{mes}/{serie}-{correlativo}.pdf/.xml`.
- Correo enviado registrado en `CorreoEnviado` con adjuntos en `ArchivoCorreoEnviado`.
- Error SUNAT: NC permanece en estado previo, sin persistir como VIGENTE.

**Más información de la tarea:**
Ver secciones *"Parte B / B7, B8, B9"* y *"Parte E"* en `R16A-RE-FU-033-Back.md`.

**Recursos:**
- `R16A-RE-FU-033-Back.md` — Parte B B7–B9, Parte E
- `PersistMexicoCreditNotePdfService` (RE-032 T10) — patrón base

---

