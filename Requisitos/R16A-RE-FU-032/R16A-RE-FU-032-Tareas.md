1# Tareas BackEnd — R16A-RE-FU-032
**Requisito:** Notas de Crédito México (CFDI tipo E — Egreso)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder + SSIS (ETL PCconnect)


---

## Resumen de tareas

| #   | Clave             | Título simple                                                                                 | Tipo | Aplicativo              |
| --- | ----------------- | --------------------------------------------------------------------------------------------- | ---- | ----------------------- |
| 1   | UPDATE-TABL-M     | Extender fccNotaCredito: ADD 13 columnas R16 (empresa, cliente, modalidad, estado, fiscal)    | BD   | ProquifaDotNet          |
| 2   | UPDATE-TABL-M     | Extender fccNotaCreditoPartida: ADD 6 columnas R16 (concepto origen, importes fiscales)       | BD   | ProquifaDotNet          |
| 3   | UPDATE-TABL-CH    | DML catUsoCFDI: INSERT clave G02 si no existe                                                 | BD   | ProquifaDotNet          |
| 4   | UPDATE-TABL-CH    | DML catTipoCFDI: INSERT clave NOTA_CREDITO                                                    | BD   | ProquifaDotNet          |
| 5   | UPDATE-TABL-CH    | DML EmpresaFolio: INSERT 4 filas Serie "P2" para GOL, MUN, PRO, PQF                           | BD   | ProquifaDotNet (Finanzas) |
| 6   | UPDATE-TABL-CH    | DML DocumentTemplate: INSERT 4 templates PDF NC México                                        | BD   | DocumentBuilder         |
| 7   | ALG-COMPLX-LOGIC  | Implementar endpoint timbrado NC México (CFDI tipo E) en Timbrado — folio Serie P2 + TurboPac | Back | ProquifaDotNet.Timbrado |
| 8   | IMP-EXIST-SERVICE | Implementar Wizard Paso 1 y Paso 2: búsqueda de facturas, captura por partidas y manual       | Back | ProquifaDotNet.Finanzas |
| 9   | SERV-TRANSACT     | Implementar timbrado NC, persistencia MinIO, cancelación condicional y correo automático      | Back | ProquifaDotNet.Finanzas |
| 10  | CREATE-TABL-CH    | Crear catMotivoCancelacionSAT con DDL + DML 4 claves SAT c_MotivoCancelacion                  | BD   | ProquifaDotNet          |
| 11  | LIST-NO-FILTER    | Endpoint GET /api/v1/cancellationReason para que Front obtenga claves SAT                     | Back | ProquifaDotNet.Finanzas |
| 12  | QUERY-G           | ETL PCconnect — Análisis de datos a transferir de NCs a Legacy                                | ETL  | SSIS / PCconnect        |
| 13  | BD-OBJ-G          | ETL PCconnect — Desarrollo paquete SSIS: transferencia de datos NC y PDF a Legacy             | ETL  | SSIS / PCconnect        |
| 14  | QUERY-M           | ETL PCconnect — Pruebas de validación de datos transferidos a Legacy                          | ETL  | SSIS / PCconnect        |

> **Nota:** Las filas marcadas *(→ RE-034)* corresponden a tareas de generación de documentos que pertenecen a **R16A-RE-FU-034**. Se listan aquí para trazabilidad del orden de ejecución. Las tareas propias de este requisito (RE-032) son T1–T9 y T10–T14.

---

## TAREA 1

**[ RE-FU-032 ] [UPDATE-TABL-M] Extender fccNotaCredito: ADD 13 columnas R16**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Notas de Crédito México — Tabla principal NC

**Consideraciones previas:**
- `fccNotaCredito` ya existe. Se agregan 13 columnas nuevas sin modificar las existentes.
- ⚠️ **Brecha B1 bloqueante:** `IdTPProformaPedido` es NOT NULL en la tabla existente. Las NCs R16 no provienen de un pedido. Confirmar con el equipo si la columna puede modificarse a NULL o se usa un valor placeholder antes de ejecutar este script.
- Las columnas `IdEmpresa` e `IdCFDIGeneradaFacturaOrigen` se agregan como NOT NULL — si hay registros pre-R16 en la tabla, se debe definir un valor default o ejecutar la migración en etapas.
- `ClaveMotivosCancelacion` sigue el patrón de `CFDICancelacion.ClaveMotivo varchar(4)` — sin FK a catálogo (el catálogo SAT c_MotivoCancelacion se gestiona desde config/frontend).

**Objetivo general:**
Agregar las 13 columnas necesarias para el módulo R16 de Notas de Crédito México en `fccNotaCredito`, habilitando la persistencia de empresa, cliente, modalidad, estado, datos fiscales, archivos MinIO y trazabilidad de cancelación.

**Objetivos específicos:**
- Agregar columnas de identificación: `IdEmpresa`, `IdCliente`.
- Agregar columnas de control: `Serie`, `Modalidad`, `Motivo`, `Estado`, `CancelarFacturaOrigen`, `ClaveMotivosCancelacion`.
- Agregar columna de origen: `IdCFDIGeneradaFacturaOrigen`.
- Agregar columnas de captura manual: `ConceptoManual`, `ObservacionesManual`.
- Agregar columnas de archivos: `IdArchivoXml`, `IdArchivoPdf`.
- Validar que las 13 columnas existan y sean del tipo correcto.

**Resultado esperado:**
`fccNotaCredito` contiene las 13 columnas R16. `ProquifaDotNet.Finanzas` puede persistir una NC completa en la tabla.

**Entregables:**
- Script DDL: 13 `ALTER TABLE fccNotaCredito ADD` + scripts de validación

**Scripts:**

```sql
-- ⚠️ Resolver Brecha B1 antes de ejecutar (IdTPProformaPedido NOT NULL)
-- Ejecutar en ProquifaDotNet

ALTER TABLE dbo.fccNotaCredito ADD [IdEmpresa] uniqueidentifier NULL;
    -- FK Empresa — empresa emisora de la NC (GOL, MUN, PRO, PQF)
GO
ALTER TABLE dbo.fccNotaCredito ADD [IdCliente] uniqueidentifier NULL;
    -- FK Cliente — cliente receptor de la NC
GO
ALTER TABLE dbo.fccNotaCredito ADD [Serie] varchar(10) NULL;
    -- Serie del foliador interno. Valor esperado: 'P2'. ⚠️ Pendiente validar con PMO.
GO
ALTER TABLE dbo.fccNotaCredito ADD [Modalidad] varchar(20) NULL;
    -- 'POR_PARTIDAS' | 'MANUAL'
GO
ALTER TABLE dbo.fccNotaCredito ADD [Motivo] varchar(50) NULL;
    -- Clave del motivo principal de la NC
GO
ALTER TABLE dbo.fccNotaCredito ADD [Estado] varchar(20) NULL
    CONSTRAINT [DF_fccNotaCredito_Estado] DEFAULT ('VIGENTE');
    -- 'VIGENTE' | 'CANCELADA'. Default VIGENTE.
GO
ALTER TABLE dbo.fccNotaCredito ADD [CancelarFacturaOrigen] bit NULL
    CONSTRAINT [DF_fccNotaCredito_CancelarFactura] DEFAULT (0);
    -- 1 si el usuario solicitó cancelar la factura origen ante el SAT
GO
ALTER TABLE dbo.fccNotaCredito ADD [ClaveMotivosCancelacion] varchar(4) NULL;
    -- Clave SAT c_MotivoCancelacion ('01','02','03','04'). Null si no cancela.
GO
ALTER TABLE dbo.fccNotaCredito ADD [IdCFDIGeneradaFacturaOrigen] uniqueidentifier NULL;
    -- FK CFDIGenerada — factura que originó esta NC
GO
ALTER TABLE dbo.fccNotaCredito ADD [ConceptoManual] nvarchar(500) NULL;
    -- Materialidad fiscal. Solo modalidad MANUAL.
GO
ALTER TABLE dbo.fccNotaCredito ADD [ObservacionesManual] nvarchar(500) NULL;
    -- Observaciones adicionales opcionales. Solo modalidad MANUAL.
GO
ALTER TABLE dbo.fccNotaCredito ADD [IdArchivoXml] uniqueidentifier NULL;
    -- FK Archivo — XML timbrado. Null hasta timbrado exitoso.
GO
ALTER TABLE dbo.fccNotaCredito ADD [IdArchivoPdf] uniqueidentifier NULL;
    -- FK Archivo — PDF representativo. Null hasta generación.
GO

-- Índices
CREATE NONCLUSTERED INDEX [IX_fccNotaCredito_IdCliente]
    ON dbo.fccNotaCredito ([IdCliente]);
GO
CREATE NONCLUSTERED INDEX [IX_fccNotaCredito_IdCFDIGeneradaFacturaOrigen]
    ON dbo.fccNotaCredito ([IdCFDIGeneradaFacturaOrigen]);
GO
CREATE NONCLUSTERED INDEX [IX_fccNotaCredito_Estado]
    ON dbo.fccNotaCredito ([Estado]);
GO

-- Validación
SELECT c.name AS Columna, t.name AS Tipo, c.is_nullable AS EsNulable
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.fccNotaCredito')
  AND c.name IN (
      'IdEmpresa','IdCliente','Serie','Modalidad','Motivo','Estado',
      'CancelarFacturaOrigen','ClaveMotivosCancelacion','IdCFDIGeneradaFacturaOrigen',
      'ConceptoManual','ObservacionesManual','IdArchivoXml','IdArchivoPdf'
  )
ORDER BY c.column_id;
-- Debe retornar 13 filas
```

**Criterios de aceptación:**
- Las 13 columnas existen en `fccNotaCredito` con los tipos correctos.
- Los defaults `Estado='VIGENTE'` y `CancelarFacturaOrigen=0` están activos.
- Los 3 índices nuevos existen.
- Los registros pre-R16 existentes no fueron afectados.

**Más información de la tarea:**
Ver sección *"ALTER TABLE fccNotaCredito — Columnas R16"* en `R16A-RE-FU-032_BD.md` y sección *"Parte A / A1"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032_BD.md` — ALTER TABLE fccNotaCredito

---

## TAREA 2

**[ RE-FU-032 ] [UPDATE-TABL-M] Extender fccNotaCreditoPartida: ADD 6 columnas R16**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Notas de Crédito México — Partidas de NC

**Consideraciones previas:**
- `fccNotaCreditoPartida` ya existe. Se agregan 6 columnas sin modificar las existentes.
- ⚠️ **Brecha B2:** `NumeroDePiezas` existente es `int`. Si hay productos con cantidad fraccionaria se requiere `ALTER COLUMN` a `decimal(18,6)`. Confirmar con el equipo antes de ejecutar.
- `IdCFDIGeneradaConceptoOrigen` vincula la partida de la NC con el concepto original de la factura en `CFDIGeneradaConcepto`, habilitando la herencia de ClaveProdServ, ClaveUnidad, Descripcion, etc.
- Las columnas de importes (`Importe`, `Subtotal`, `IVA`, `Total`) se calculan por Finanzas antes de persistir; no se recalculan en BD.
- Prerrequisito: Tarea 1 debe estar ejecutada.

**Objetivo general:**
Agregar las 6 columnas R16 a `fccNotaCreditoPartida` para soportar la modalidad por partidas con trazabilidad al concepto original de la factura e importes fiscales calculados.

**Objetivos específicos:**
- Agregar `IdCFDIGeneradaConceptoOrigen` para trazabilidad al concepto de la factura origen.
- Agregar `CantidadNC` en decimal para soportar cantidades fraccionarias.
- Agregar columnas de importes fiscales por línea: `Importe`, `Subtotal`, `IVA`, `Total`.
- Validar las 6 columnas nuevas.

**Resultado esperado:**
`fccNotaCreditoPartida` contiene las 6 columnas R16. Finanzas puede persistir cada partida de la NC con su trazabilidad fiscal completa.

**Entregables:**
- Script DDL: 6 `ALTER TABLE fccNotaCreditoPartida ADD` + script de validación

**Scripts:**

```sql
-- ⚠️ Resolver Brecha B2 antes si hay productos con cantidad fraccionaria
-- Ejecutar en ProquifaDotNet

ALTER TABLE dbo.fccNotaCreditoPartida ADD [IdCFDIGeneradaConceptoOrigen] uniqueidentifier NULL;
    -- FK CFDIGeneradaConcepto — concepto de la factura origen que se devuelve.
    -- Permite heredar ClaveProdServ, ClaveUnidad, NoIdentificacion, Descripcion.
GO
ALTER TABLE dbo.fccNotaCreditoPartida ADD [CantidadNC] decimal(18,6) NULL;
    -- Cantidad a devolver: 0 (no incluida), parcial o igual a Cant. Facturada.
    -- Decimal para soportar cantidades fraccionarias.
GO
ALTER TABLE dbo.fccNotaCreditoPartida ADD [Importe] decimal(18,6) NULL;
    -- CantidadNC × ValorUnitario. En moneda de la factura origen.
GO
ALTER TABLE dbo.fccNotaCreditoPartida ADD [Subtotal] decimal(18,6) NULL;
    -- Subtotal de la línea (sin IVA).
GO
ALTER TABLE dbo.fccNotaCreditoPartida ADD [IVA] decimal(18,6) NULL;
    -- IVA calculado sobre el Subtotal de la línea.
GO
ALTER TABLE dbo.fccNotaCreditoPartida ADD [Total] decimal(18,6) NULL;
    -- Subtotal + IVA de la línea.
GO

-- Validación
SELECT c.name AS Columna, t.name AS Tipo, c.is_nullable AS EsNulable
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.fccNotaCreditoPartida')
  AND c.name IN (
      'IdCFDIGeneradaConceptoOrigen','CantidadNC',
      'Importe','Subtotal','IVA','Total'
  )
ORDER BY c.column_id;
-- Debe retornar 6 filas
```

**Criterios de aceptación:**
- Las 6 columnas existen en `fccNotaCreditoPartida` con los tipos correctos.
- Las columnas son nullables (registros pre-R16 no afectados).

**Más información de la tarea:**
Ver sección *"ALTER TABLE fccNotaCreditoPartida — Columnas R16"* en `R16A-RE-FU-032_BD.md` y sección *"Parte A / A2"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032_BD.md` — ALTER TABLE fccNotaCreditoPartida

---

## TAREA 3

**[ RE-FU-032 ] [UPDATE-TABL-CH] DML catUsoCFDI: INSERT clave G02 si no existe**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogos SAT — UsoCFDI

**Consideraciones previas:**
- La NC usa `UsoCFDI='G02'` (Devoluciones, descuentos o bonificaciones) por default (Regla 6 / Criterio D4). Verificar existencia antes de insertar.
- La tabla `catUsoCFDI` existe con columnas: `IdCatUsoCFDI`, `Uso nvarchar(150)`, `ClaveUso nvarchar(3)`, `Activo bit`, `Clave varchar(150)`.
- Puede haberse insertado en otro requisito previo; el script verifica antes de actuar.

**Objetivo general:**
Garantizar que la clave `G02` existe en `catUsoCFDI` para que Finanzas pueda asignarla como `UsoCFDI` del receptor en el XML de la NC.

**Objetivos específicos:**
- Verificar si `G02` ya existe.
- Insertar si no existe.
- Validar el resultado.

**Resultado esperado:**
`catUsoCFDI` contiene la clave `G02` activa.

**Entregables:**
- Script DML: SELECT verificación + INSERT condicional + SELECT validación

**Scripts:**

```sql
-- Ejecutar en ProquifaDotNet
-- Paso 1: verificar
SELECT ClaveUso, Uso, Clave, Activo FROM dbo.catUsoCFDI WHERE ClaveUso = 'G02';
-- Si retorna 0 filas, ejecutar el INSERT siguiente

-- Paso 2: insertar si no existe
INSERT INTO dbo.catUsoCFDI (IdCatUsoCFDI, Uso, ClaveUso, Activo, Clave)
SELECT NEWID(), 'G02 Devoluciones, descuentos o bonificaciones', 'G02', 1, 'G02'
WHERE NOT EXISTS (SELECT 1 FROM dbo.catUsoCFDI WHERE ClaveUso = 'G02');
GO

-- Validación
SELECT ClaveUso, Uso, Activo FROM dbo.catUsoCFDI WHERE ClaveUso = 'G02';
-- Debe retornar 1 fila con Activo=1
```

**Criterios de aceptación:**
- `catUsoCFDI` contiene exactamente 1 fila con `ClaveUso='G02'` y `Activo=1`.
- No se duplicaron registros existentes.

**Más información de la tarea:**
Ver sección *"DML catUsoCFDI"* en `R16A-RE-FU-032_BD.md` y sección *"Parte A / A3"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032_BD.md` — DML catUsoCFDI

---

## TAREA 4

**[ RE-FU-032 ] [UPDATE-TABL-CH] DML catTipoCFDI: INSERT clave NOTA_CREDITO**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogos CFDI — TipoCFDI

**Consideraciones previas:**
- **Prerrequisito:** La tabla `catTipoCFDI` es creada por R16A-RE-FU-028 Tarea 1. Esta tarea no puede ejecutarse hasta que RE-028 T1 esté completa.
- Discriminador de tipo CFDI para las NCs en `CFDIGenerada.IdCatTipoCFDI`.
- Verificar la estructura exacta de `catTipoCFDI` (columnas y tipos) en RE-028 T1 antes de ejecutar el INSERT.
- ⚠️ Pendiente P7: confirmar columnas de `catTipoCFDI` con RE-028 para ajustar el INSERT si difiere.

**Objetivo general:**
Insertar la clave `NOTA_CREDITO` en `catTipoCFDI` para que Finanzas y Timbrado puedan discriminar las NCs en `CFDIGenerada`.

**Objetivos específicos:**
- Verificar que `catTipoCFDI` existe (RE-028 T1 completada).
- Verificar que `NOTA_CREDITO` no existe.
- Insertar la clave con `TipoDocumentoSAT='E'`.
- Validar el resultado.

**Resultado esperado:**
`catTipoCFDI` contiene la clave `NOTA_CREDITO` activa con `TipoDocumentoSAT='E'`.

**Entregables:**
- Script DML: SELECT verificación + INSERT + SELECT validación

**Scripts:**

```sql
-- Prerequisito: catTipoCFDI debe existir (RE-028 T1)
-- Ejecutar en ProquifaDotNet

-- Verificar que no existe
SELECT * FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO';

-- Insertar
INSERT INTO dbo.catTipoCFDI (IdCatTipoCFDI, Clave, Descripcion, TipoDocumentoSAT, Activo)
SELECT NEWID(), 'NOTA_CREDITO', 'Nota de Crédito', 'E', 1
WHERE NOT EXISTS (SELECT 1 FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO');
GO

-- Validación
SELECT IdCatTipoCFDI, Clave, Descripcion, TipoDocumentoSAT, Activo
FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO';
-- Debe retornar 1 fila con TipoDocumentoSAT='E' y Activo=1
```

**Criterios de aceptación:**
- `catTipoCFDI` contiene 1 fila con `Clave='NOTA_CREDITO'`, `TipoDocumentoSAT='E'`, `Activo=1`.

**Más información de la tarea:**
Ver sección *"DML catTipoCFDI"* en `R16A-RE-FU-032_BD.md` y sección *"Parte A / A4"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032_BD.md` — DML catTipoCFDI
- `R16A-RE-FU-028-Tareas.md` — Tarea 1 (catTipoCFDI)

---

## TAREA 5

**[ RE-FU-032 ] [UPDATE-TABL-CH] DML EmpresaFolio: INSERT 4 filas Serie "P2" para GOL, MUN, PRO, PQF**

**Aplicativos:** ProquifaDotNetTimbrado

**Módulos:** Base de Datos — Foliador Notas de Crédito México

**Consideraciones previas:**
- `EmpresaFolio` existe en `ProquifaDotNet` (propiedad Finanzas, creada en RE-019). Solo se insertan filas nuevas con Serie "P2".
- Las NCs de México usan Serie "P2" (distinta a Serie "P" del Complemento de Pago de RE-030). ⚠️ Pendiente validar formato y longitud con PMO (Regla 9 del requisito).
- El foliador usa UPDLOCK atómico en Timbrado para asignar folios consecutivos sin colisión.
- La tabla `Empresa` no tiene columna `Region` — se filtra por `Prefijo` (GOL, MUN, PRO, PQF).
- Verificar que no existan filas Serie "P2" antes de insertar para evitar duplicados.

**Objetivo general:**
Registrar las 4 filas de Serie "P2" en `EmpresaFolio` para que Timbrado pueda asignar folios consecutivos a las NCs de cada empresa México al timbrar.

**Objetivos específicos:**
- Verificar ausencia de filas Serie "P2" para las 4 empresas.
- Insertar las 4 filas con `UltimoFolio=0` y formato de folio propuesto.
- Validar que existen 4 filas con `Serie='P2'` y `UltimoFolio=0`.

**Resultado esperado:**
`EmpresaFolio` contiene 4 filas Serie "P2" (GOL, MUN, PRO, PQF) con `UltimoFolio=0`, listas para asignación UPDLOCK atómico en Timbrado.

**Entregables:**
- Script DML: 4 INSERT + script de validación

**Scripts:**

```sql
-- Prerequisito: EmpresaFolio y las 4 empresas México deben existir (RE-019)
-- Ejecutar en ProquifaDotNetTimbrado
-- ⚠️ Ajustar FormatoFolio y LongitudMaxima según validación con PMO.

-- Verificar ausencia
SELECT e.Prefijo, ef.Serie FROM dbo.EmpresaFolio ef
INNER JOIN dbo.Empresa e ON ef.IdEmpresa = e.IdEmpresa
WHERE ef.Serie = 'P2' AND e.Prefijo IN ('GOL','MUN','PRO','PQF');
-- Debe retornar 0 filas antes de continuar

-- Golocaer México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa, 'P2', 0, 'P2{folio:00000000}', 10, 1
FROM dbo.Empresa e WHERE e.Prefijo = 'GOL';

-- Mungen México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa, 'P2', 0, 'P2{folio:00000000}', 10, 1
FROM dbo.Empresa e WHERE e.Prefijo = 'MUN';

-- Proquifa México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa, 'P2', 0, 'P2{folio:00000000}', 10, 1
FROM dbo.Empresa e WHERE e.Prefijo = 'PRO';

-- Proveedora Quimico Farmaceutica México
INSERT INTO dbo.EmpresaFolio (IdEmpresa, Serie, UltimoFolio, FormatoFolio, LongitudMaxima, Activo)
SELECT e.IdEmpresa, 'P2', 0, 'P2{folio:00000000}', 10, 1
FROM dbo.Empresa e WHERE e.Prefijo = 'PQF';
GO

-- Validación
SELECT e.Prefijo, ef.Serie, ef.UltimoFolio, ef.FormatoFolio, ef.LongitudMaxima, ef.Activo
FROM dbo.EmpresaFolio ef
INNER JOIN dbo.Empresa e ON ef.IdEmpresa = e.IdEmpresa
WHERE ef.Serie = 'P2'
ORDER BY e.Prefijo;
-- Debe retornar 4 filas con UltimoFolio=0
```

**Criterios de aceptación:**
- 4 filas Serie "P2" existen en `EmpresaFolio` con `UltimoFolio=0` y `Activo=1`.
- Una fila por cada empresa: GOL, MUN, PRO, PQF.
- Las filas de otras Series no fueron afectadas.

**Más información de la tarea:**
Ver sección *"DML EmpresaFolio"* en `R16A-RE-FU-032_BD.md` y sección *"Parte A / A5"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032_BD.md` — DML EmpresaFolio

---

## TAREA 6

**[ RE-FU-032 ] [UPDATE-TABL-CH] DML DocumentTemplate: INSERT 4 templates PDF NC México**

**Aplicativos:** DocumentBuilder

**Módulos:** Base de Datos — DocumentBuilder — Templates Nota de Crédito México

**Consideraciones previas:**
- La tabla `DocumentTemplate` existe en `DocumentBuilder` (no en ProquifaDotNet). Columnas confirmadas: `TemplateKey`, `HeaderTemplateFileName`, `BodyTemplateFileName`, `FooterTemplateFileName`, `HasHeaderTemplate`, `HasBodyTemplate`, `HasFooterTemplate`, `RegistrationDate`, `LastUpdateDate`, `IsActive`.
- Convención de nombres: `{TemplateKey}_{H/B/F}.html`. Los 3 archivos siempre se usan (`Has*=1`).
- Los 4 `TemplateKey` son: `GOL_MEX_NC`, `MUN_MEX_NC`, `PRO_MEX_NC`, `PQF_MEX_NC`.
- Los archivos HTML se crean en la Tarea 11. Esta tarea solo registra metadatos en BD.

**Objetivo general:**
Registrar los 4 templates del Módulo NC México en `DocumentBuilder.DocumentTemplate` para que `PersistMexicoCreditNotePdfService` pueda resolver el `TemplateKey` correcto por empresa emisora.

**Objetivos específicos:**
- Verificar ausencia de los 4 TemplateKey.
- Insertar las 4 filas con archivos `{TemplateKey}_H/B/F.html` y `Has*=1`.
- Validar los 4 registros activos.

**Resultado esperado:**
`DocumentBuilder.DocumentTemplate` contiene los 4 templates NC México. Finanzas puede invocar DocumentBuilder con `TemplateKey={Prefijo}_MEX_NC`.

**Entregables:**
- Script DML (en DocumentBuilder): 4 INSERT + script de validación

**Scripts:**

```sql
-- Ejecutar en DocumentBuilder
-- Verificar ausencia
SELECT TemplateKey FROM dbo.DocumentTemplate
WHERE TemplateKey IN ('GOL_MEX_NC','MUN_MEX_NC','PRO_MEX_NC','PQF_MEX_NC');
-- Debe retornar 0 filas antes de continuar

INSERT INTO dbo.DocumentTemplate (TemplateKey, HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName, HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, RegistrationDate, LastUpdateDate, IsActive)
VALUES ('GOL_MEX_NC', 'GOL_MEX_NC_H.html', 'GOL_MEX_NC_B.html', 'GOL_MEX_NC_F.html', 1, 1, 1, GETDATE(), GETDATE(), 1);

INSERT INTO dbo.DocumentTemplate (TemplateKey, HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName, HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, RegistrationDate, LastUpdateDate, IsActive)
VALUES ('MUN_MEX_NC', 'MUN_MEX_NC_H.html', 'MUN_MEX_NC_B.html', 'MUN_MEX_NC_F.html', 1, 1, 1, GETDATE(), GETDATE(), 1);

INSERT INTO dbo.DocumentTemplate (TemplateKey, HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName, HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, RegistrationDate, LastUpdateDate, IsActive)
VALUES ('PRO_MEX_NC', 'PRO_MEX_NC_H.html', 'PRO_MEX_NC_B.html', 'PRO_MEX_NC_F.html', 1, 1, 1, GETDATE(), GETDATE(), 1);

INSERT INTO dbo.DocumentTemplate (TemplateKey, HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName, HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, RegistrationDate, LastUpdateDate, IsActive)
VALUES ('PQF_MEX_NC', 'PQF_MEX_NC_H.html', 'PQF_MEX_NC_B.html', 'PQF_MEX_NC_F.html', 1, 1, 1, GETDATE(), GETDATE(), 1);
GO

-- Validación
SELECT TemplateKey, BodyTemplateFileName, HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, IsActive
FROM dbo.DocumentTemplate
WHERE TemplateKey IN ('GOL_MEX_NC','MUN_MEX_NC','PRO_MEX_NC','PQF_MEX_NC')
ORDER BY TemplateKey;
-- Debe retornar 4 filas con Has*=1 e IsActive=1
```

**Criterios de aceptación:**
- 4 registros activos con `IsActive=1` y `Has*Template=1`.
- `BodyTemplateFileName` sigue la convención `{TemplateKey}_B.html`.

**Más información de la tarea:**
Ver sección *"DML DocumentTemplate"* en `R16A-RE-FU-032_BD.md` y sección *"Parte A / A6"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032_BD.md` — DML DocumentTemplate

---

## TAREA 7

**[ RE-FU-032 ] [ALG-COMPLX-LOGIC] Implementar endpoint timbrado NC México en Timbrado**

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Timbrado — Nota de Crédito México — CFDI tipo E

**Consideraciones previas:**
- Mismo patrón que el endpoint de Factura México (RE-019/021) y Complemento de Pago (RE-030 C1).
- El XML es CFDI 4.0 TipoDocumento='E', MetodoPago='PUE', CfdiRelacionados TipoRelacion='01'.
- El folio se obtiene con UPDLOCK atómico sobre `EmpresaFolio` Serie "P2" (prereq: Tarea 5).
- Si la cancelación de factura origen fue solicitada (`CancelarFacturaOrigen=1`), Finanzas llama a `POST /api/v1/stamp/cancel` en Timbrado (vía `POST /api/v1/cfdi/{id}/cancel` de Finanzas — endpoint de cancelación ya existente desde RE-FU-018/021/028, no se crea uno nuevo).
- Prerrequisitos: Tareas 4 y 5.

**Objetivo general:**
Ampliar el `StampingController` (endpoint de Notas de Crédito `POST /api/v1/stamp/credit-note` creado en RE-FU-018) para que, al recibir `CreditNoteMexicoRequest` de Finanzas, construya el XML CFDI E, lo timbre vía PAC TurboPac, inserte en `CFDIGenerada` + `CFDI`, actualice `EmpresaFolio` y retorne el CFDI timbrado.

**Objetivos específicos:**
- Recibir y validar `CreditNoteMexicoRequest` (datos del emisor, receptor, conceptos, CFDI relacionado).
- Construir XML CFDI 4.0 TipoDocumento='E' con todos los nodos requeridos.
- Obtener folio con UPDLOCK atómico Serie "P2".
- Enviar XML al PAC TurboPac y recibir XML timbrado con `TimbreFiscalDigital`.
- INSERT `CFDIGenerada` + `CFDI` con UUID SAT.
- UPDATE `EmpresaFolio.UltimoFolio`.
- Retornar `CreditNoteMexicoResponse` a Finanzas.
- Manejo de errores PAC: retornar error con detalle sin persistir.

**Resultado esperado:**
Endpoint de Notas de Crédito (`POST /api/v1/stamp/credit-note`) funcional que timbra NCs México vía TurboPac con folios Serie "P2" únicos y consecutivos.

**Entregables:**
- Ampliación de `StampingController` (`POST /api/v1/stamp/credit-note`) para el flujo de Nota de Crédito México
- `CreditNoteMexicoRequest` / `CreditNoteMexicoResponse` DTOs
- Unit tests del servicio de construcción XML

**Criterios de aceptación:**
- NC timbrada tiene UUID único asignado por el SAT.
- `EmpresaFolio.UltimoFolio` incrementa atómicamente sin colisiones concurrentes.
- Error del PAC retorna detalle sin registros en `CFDIGenerada`.
- Folio + Serie "P2" quedan en `CFDIGenerada.Folio` / `Serie`.

**Más información de la tarea:**
Ver sección *"Parte C / C1"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032-Back.md` — Parte C, sección C1
- Endpoint análogo: `POST /api/v1/stamp/invoice` (mismo StampingController, endpoint por tipo de documento — RE-019/021)

---

## TAREA 8

**[ RE-FU-032 ] [IMP-EXIST-SERVICE] Implementar Wizard Paso 1 y Paso 2: búsqueda de facturas, captura por partidas y manual**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Notas de Crédito México — Wizard Pasos 1 y 2

**Consideraciones previas:**
- **Paso 1:** Lista facturas vigentes de prepago del cliente con antigüedad máxima 5 años (Regla 3/10). Filtra sobre `CFDIGenerada` TipoDocumento='I', no canceladas en `CFDICancelacion`, del cliente seleccionado.
- **Paso 2 Modalidad por partidas:** Carga partidas de la factura origen desde `CFDIGeneradaConcepto`. El cálculo de subtotal/IVA/total se hace en tiempo real en el frontend (o endpoint de recálculo). La cancelación condicional de factura se evalúa en backend: totalidad + mismo mes calendario.
- **Paso 2 Modalidad manual:** Captura libre de monto (máximo = Total factura origen), concepto obligatorio, observaciones opcionales.
- ⚠️ Pendiente P4: FormaPago en modalidad manual. Pendiente P5: ClaveProdServ/ClaveUnidad default modalidad manual.
- Prerrequisitos: Tarea 1 ejecutada (columnas R16 disponibles).

**Objetivo general:**
Implementar los endpoints de Finanzas para el Wizard Paso 1 (búsqueda de facturas elegibles) y Paso 2 (captura de datos en ambas modalidades), incluyendo la evaluación de la cancelación condicional.

**Objetivos específicos:**
- Endpoint GET `/api/v1/client/{id}/eligibleInvoice`: retorna facturas vigentes prepago del cliente (máx. 5 años).
- Endpoint GET `/api/v1/cfdi/{id}/lineItem`: retorna partidas de la factura origen para modalidad por partidas.
- Lógica de evaluación de cancelación condicional: `NC por totalidad && MONTH(FechaFactura) == MONTH(GETDATE())`.
- Endpoint POST `/api/v1/creditNote`: persiste `fccNotaCredito` + `fccNotaCreditoPartida` (si aplica) en estado `PENDIENTE` antes de timbrar.
- Validaciones: monto manual ≤ Total factura origen, concepto obligatorio en manual.

**Resultado esperado:**
El frontend puede construir el Wizard Paso 1 y Paso 2 completo consumiendo los endpoints de Finanzas, con evaluación correcta de la cancelación condicional.

**Entregables:**
- Endpoints GET facturas elegibles, GET partidas, POST borrador
- Lógica de evaluación de cancelación condicional
- Unit tests de validaciones

**Criterios de aceptación:**
- Solo facturas Vigente de prepago aparecen en el Paso 1 (no canceladas SAT, máx. 5 años).
- La opción "Cancelar Factura" solo aparece cuando NC = totalidad + mismo mes.
- El monto manual no puede superar el Total de la factura origen.
- El concepto en modalidad manual es obligatorio para avanzar al Paso 3.

**Más información de la tarea:**
Ver secciones *"Parte B / B1, B2, B3"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032-Back.md` — Parte B, secciones B1–B3

---

## TAREA 9

**[ RE-FU-032 ] [SERV-TRANSACT] Timbrado NC, persistencia MinIO, cancelación condicional y correo automático**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Notas de Crédito México — Wizard Paso 3 acción Timbrar + Persistencia + Correo

**Consideraciones previas:**
- Orquesta la secuencia completa post-timbrado: Finanzas → Timbrado → (condicional) Cancelación factura → DocumentBuilder → MinIO → BD → Correo.
- `PersistMexicoCreditNotePdfService` es implementado en **R16A-RE-FU-034 T6**. Esta tarea lo consume; asegurarse de que RE-034 T6 esté completada antes de integrarse end-to-end.
- El bucket MinIO se resuelve de `RegionConfiguracionMinioBucket` donde `BucketClave='notas_credito'` y `Region.Clave='MEX'` (⚠️ verificar existencia — Pendiente P2).
- Correo automático al timbrar: Para=contacto cliente (editable), CC=ESAC+CxC (editable). ⚠️ Plantilla pendiente PMO #31.
- Prerrequisitos: T1, T2, T7, T8, **RE-034 T1–T6** completos.

**Objetivo general:**
Implementar el flujo completo de Finanzas post-timbrado: llamada a Timbrado, (condicional) cancelación de factura origen, generación PDF final, subida a MinIO, actualización de BD y envío automático del correo al cliente con PDF y XML adjuntos.

**Objetivos específicos:**
- Endpoint POST `/api/v1/creditNote/{id}/stamp`: orquesta toda la secuencia.
- Llamada al API de Timbrado (`POST /api/v1/stamp/credit-note`, vía `ApiCallerStamping.StampCreditNoteAsync`) y manejo de error/éxito (Criterios J1, J5).
- Si `CancelarFacturaOrigen=1`: llamada al endpoint de cancelación CFDI.
- `PersistMexicoCreditNotePdfService.PersistirAsync()`: DocumentBuilder → MinIO → INSERT Archivo × 2 → INSERT CFDIGeneradaRelacionado → UPDATE fccNotaCredito → INSERT fccNotaCreditoPartida (si por partidas).
- Envío de correo con INSERT `CorreoEnviado` + `ArchivoCorreoEnviado` (PDF + XML).
- Navegación al Paso 4 NC Emitida con datos completos.

**Resultado esperado:**
Al presionar "Timbrar" en el Paso 3, la NC queda en estado VIGENTE en BD, el PDF y XML están en MinIO, y el correo es enviado automáticamente al cliente.

**Entregables:**
- Endpoint POST timbrar
- `PersistMexicoCreditNotePdfService`
- Flujo de correo automático
- Unit tests + integration tests del flujo completo

**Criterios de aceptación:**
- NC timbrada tiene `Estado='VIGENTE'`, `IdCFDIGenerada`, `IdArchivoPdf`, `IdArchivoXml` poblados.
- PDF y XML accesibles en MinIO en la ruta `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID}.pdf/.xml`.
- Si cancelación: `CFDICancelacion` tiene registro con `ClaveMotivo` y `Estatus='CANCELADA'`.
- Correo enviado registrado en `CorreoEnviado` con adjuntos en `ArchivoCorreoEnviado`.
- Error PAC: NC permanece en estado previo, no se persiste como VIGENTE.

**Más información de la tarea:**
Ver sección *"Parte B / B6, B7, B8"* y *"Parte E"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032-Back.md` — Parte B sección B6, Parte E
- `PersistMexicoInvoicePdfService` (RE-021) — patrón base
- `PersistPaymentComplementPdfService` (RE-030) — patrón base

---

## TAREA 10

**[ RE-FU-032 ] [CREATE-TABL-CH] Crear catMotivoCancelacionSAT con DDL + DML 4 claves SAT**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogos SAT — Motivos de Cancelación CFDI

**Consideraciones previas:**
- La tabla `catMotivoCancelacionSAT` es nueva. El campo `CFDICancelacion.ClaveMotivo varchar(4)` ya existe pero no tiene FK — esta tarea la establece como catálogo formal.
- Contiene las 4 claves oficiales del catálogo **c_MotivoCancelacion** del SAT para CFDI 4.0.
- Patrón estándar del proyecto: `uniqueidentifier PK` con `NEWID()`, `Clave varchar UNIQUE`, `Descripcion nvarchar`, `Activo bit DEFAULT(1)`, `FechaRegistro datetime2(7) DEFAULT SYSUTCDATETIME()`.
- El endpoint de Tarea 11 expone este catálogo al frontend. El frontend envía la `Clave` elegida (no el GUID) al backend al confirmar cancelación.

**Objetivo general:**
Crear el catálogo `catMotivoCancelacionSAT` con las 4 claves del SAT para que el frontend pueda mostrar el selector de motivo al cancelar una Nota de Crédito y enviar la clave correspondiente.

**Objetivos específicos:**
- Ejecutar `CREATE TABLE catMotivoCancelacionSAT` con PK, UNIQUE en `Clave`.
- Insertar las 4 claves c_MotivoCancelacion SAT vigentes.
- Validar los 4 registros activos.

**Resultado esperado:**
`catMotivoCancelacionSAT` existe en ProquifaDotNet con las 4 claves SAT oficiales, lista para ser consultada por el endpoint de catálogos.

**Entregables:**
- Script DDL + DML: `CREATE TABLE catMotivoCancelacionSAT` + `INSERT` 4 claves iniciales

**Scripts:**

```sql
-- Ejecutar en ProquifaDotNet

CREATE TABLE dbo.catMotivoCancelacionSAT (
    [IdCatMotivoCancelacionSAT] uniqueidentifier NOT NULL
        CONSTRAINT [PK_catMotivoCancelacionSAT] PRIMARY KEY DEFAULT NEWID(),
    [Clave]       varchar(4)      NOT NULL,
    [Descripcion] nvarchar(150)   NOT NULL,
    [Activo]      bit             NOT NULL CONSTRAINT [DF_catMotivoCancelacionSAT_Activo] DEFAULT (1),
    [FechaRegistro] datetime2(7)  NOT NULL CONSTRAINT [DF_catMotivoCancelacionSAT_FechaRegistro] DEFAULT SYSUTCDATETIME(),
    CONSTRAINT [UQ_catMotivoCancelacionSAT_Clave] UNIQUE ([Clave])
);
GO

-- DML — 4 claves oficiales SAT c_MotivoCancelacion (CFDI 4.0)
INSERT INTO dbo.catMotivoCancelacionSAT (Clave, Descripcion) VALUES
('01', 'Comprobante emitido con errores con relación'),
('02', 'Comprobante emitido con errores sin relación'),
('03', 'No se llevó a cabo la operación'),
('04', 'Operación nominativa relacionada en una factura global');
GO

-- Validación
SELECT Clave, Descripcion, Activo
FROM dbo.catMotivoCancelacionSAT
ORDER BY Clave;
-- Debe retornar 4 filas con Activo=1
```

**Diccionario de datos — catMotivoCancelacionSAT:**

| Columna                     | Tipo                  | Descripción                                         |
| --------------------------- | --------------------- | --------------------------------------------------- |
| `IdCatMotivoCancelacionSAT` | `uniqueidentifier` PK | Identificador único                                 |
| `Clave`                     | `varchar(4)` UNIQUE   | Clave SAT c_MotivoCancelacion ('01','02','03','04') |
| `Descripcion`               | `nvarchar(150)`       | Descripción oficial SAT del motivo                  |
| `Activo`                    | `bit` DEFAULT(1)      | Control de vigencia                                 |
| `FechaRegistro`             | `datetime2(7)`        | Fecha de inserción                                  |

**Criterios de aceptación:**
- Tabla `catMotivoCancelacionSAT` existe con PK y UNIQUE en `Clave`.
- 4 filas con claves '01','02','03','04' y `Activo=1`.
- El endpoint de Tarea 11 puede consultar la tabla correctamente.

**Más información de la tarea:**
Ver sección *"catMotivoCancelacionSAT"* en `R16A-RE-FU-032_BD.md`.

**Recursos:**
- `R16A-RE-FU-032_BD.md` — catMotivoCancelacionSAT

---

## TAREA 11

**[ RE-FU-032 ] [LIST-NO-FILTER] Endpoint GET /api/v1/cancellationReason**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Catálogos — Motivos de Cancelación SAT — API Finanzas

**Consideraciones previas:**
- Endpoint de solo lectura: retorna las 4 claves del catálogo `catMotivoCancelacionSAT` sin filtro.
- El frontend lo consume al abrir el selector de motivo en el modal de cancelación de NC.
- El frontend envía únicamente la `Clave` varchar(4) de regreso en el request de cancelación — no el GUID.
- Puede cachear el resultado (catálogo estático); vida útil de caché recomendada: 24 h.
- Prerrequisito: Tarea 10 ejecutada.

**Objetivo general:**
Exponer el catálogo `catMotivoCancelacionSAT` vía API REST para que el frontend construya el selector de motivo de cancelación y envíe la clave elegida al confirmar la cancelación de una NC.

**Objetivos específicos:**
- Endpoint `GET /api/v1/cancellationReason`: retorna lista de `{ clave, descripcion }`.
- Query sobre `catMotivoCancelacionSAT WHERE Activo=1 ORDER BY Clave`.
- DTO de respuesta: `CancellationReasonDto { Clave: string, Descripcion: string }`.

**Resultado esperado:**
El frontend puede obtener las 4 claves SAT y mostrarlas en el selector. Al confirmar cancelación, envía la `Clave` elegida en el body del request.

**Entregables:**
- Endpoint `GET /api/v1/cancellationReason`
- `CancellationReasonDto`
- Query / Repository method

**Ejemplo de respuesta:**
```json
[
  { "clave": "01", "descripcion": "Comprobante emitido con errores con relación" },
  { "clave": "02", "descripcion": "Comprobante emitido con errores sin relación" },
  { "clave": "03", "descripcion": "No se llevó a cabo la operación" },
  { "clave": "04", "descripcion": "Operación nominativa relacionada en una factura global" }
]
```

**Criterios de aceptación:**
- `GET /api/v1/cancellationReason` retorna HTTP 200 con las 4 claves activas ordenadas por `Clave`.
- La respuesta no expone el GUID interno (`IdCatMotivoCancelacionSAT`).
- El campo `clave` en la respuesta es `varchar(4)` compatible con lo que `fccNotaCredito.ClaveMotivosCancelacion` espera.

**Más información de la tarea:**
Tarea 12 es prerrequisito. La `Clave` retornada se usa como valor de `ClaveMotivosCancelacion` en `fccNotaCredito` al cancelar.

**Recursos:**
- `R16A-RE-FU-032_BD.md` — catMotivoCancelacionSAT
- `R16A-RE-FU-032-Back.md` — Parte B, sección B6 (cancelación condicional)

---

## TAREA 12

**[ RE-FU-032 ] [QUERY-G] ETL PCconnect — Análisis de datos a transferir de NCs a Legacy**

**Aplicativos:** SSIS / PCconnect (Legacy)

**Módulos:** ETL — Transferencia Notas de Crédito a PCconnect

**Consideraciones previas:**
- La estructura de tablas destino en PCconnect aún no ha sido proporcionada. Esta tarea es el punto de partida del ETL.
- El objetivo es documentar el mapeo completo: qué campos de ProquifaDotNet corresponden a qué campos en PCconnect.
- Incluye el análisis de los 4 eventos que disparan transferencia: NC timbrada, partidas NC, cancelación de factura origen, aplicación de NC a cobro.
- Esta tarea debe completarse antes de iniciar el desarrollo del paquete SSIS (Tarea 13 → ahora T13).
- Referencia de datos fuente ya documentada en `R16A-RE-FU-032-Back.md` sección F3.

**Objetivo general:**
Analizar y documentar el mapeo completo de datos entre ProquifaDotNet y PCconnect para las Notas de Crédito, incluyendo tablas destino, transformaciones requeridas y mecanismo de control de transferencias (columna de control vs tabla SSIS).

**Objetivos específicos:**
- Obtener y revisar la estructura de tablas destino en PCconnect para NCs.
- Mapear columna a columna: cabecera NC, partidas NC, cancelación factura, aplicación a cobro.
- Identificar transformaciones de datos requeridas (conversiones de tipo, lookups, valores default).
- Definir el mecanismo de control de transferencias (campo `TransferidoPCConnect` en `fccNotaCredito` vs tabla de control SSIS).
- Definir la clave natural de idempotencia (UUID SAT de la NC).
- Documentar el trigger del paquete SSIS (polling periódico vs evento RabbitMQ).
- Documentar escenario de cancelación: actualización de estado de factura en PCconnect cuando `CancelarFacturaOrigen=1`.

**Resultado esperado:**
Documento de mapeo ETL completo aprobado por el equipo, listo para iniciar el desarrollo del paquete SSIS en Tarea 13.

**Entregables:**
- Documento de análisis de mapeo ETL NC → PCconnect (tablas, columnas, transformaciones)
- Decisión documentada sobre mecanismo de control de transferencias
- Diagrama de flujo del paquete SSIS (alto nivel)

**Criterios de aceptación:**
- Mapeo cubre los 4 eventos de transferencia (NC, partidas, cancelación, aplicación).
- Clave de idempotencia definida (UUID SAT).
- Mecanismo de control acordado y documentado.
- Documento aprobado por el equipo técnico y el área de Legacy.

**Más información de la tarea:**
Ver sección *"Parte F"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032-Back.md` — Parte F (ETL PCconnect)
- Estructura de PCconnect (pendiente de recibir)

---

## TAREA 13

**[ RE-FU-032 ] [BD-OBJ-G] ETL PCconnect — Desarrollo paquete SSIS: transferencia de datos NC y PDF a Legacy**

**Aplicativos:** SSIS / PCconnect (Legacy)

**Módulos:** ETL — Paquete SSIS Notas de Crédito

**Consideraciones previas:**
- **Prerrequisito bloqueante:** Tarea 12 completada con mapeo aprobado.
- **Prerrequisito bloqueante:** Tarea 9 completada (NCs timbradas disponibles en ProquifaDotNet con estado VIGENTE).
- Paquete SSIS nuevo — no reutiliza paquetes de Facturas o Pedidos.
- Si se decidió agregar `TransferidoPCConnect` en `fccNotaCredito`, la Tarea 1 debe incluir esa columna (o agregar un ALTER TABLE adicional).
- La transferencia incluye PDF (ruta MinIO o binario) y XML de la NC.
- El paquete debe ser idempotente: UUID SAT como clave natural en PCconnect.

**Objetivo general:**
Desarrollar el paquete SSIS que transfiere las NCs timbradas de ProquifaDotNet a PCconnect, incluyendo cabecera, partidas, estado de cancelación de factura y rutas de archivos PDF/XML.

**Objetivos específicos:**
- Implementar flujo de extracción de NCs en estado VIGENTE no transferidas desde ProquifaDotNet.
- Implementar transformaciones según el mapeo de Tarea 12.
- Implementar inserción/actualización en tablas destino de PCconnect (idempotente por UUID SAT).
- Implementar transferencia del PDF (ruta MinIO) y XML.
- Implementar lógica de cancelación: si `CancelarFacturaOrigen=1`, actualizar estado de factura en PCconnect.
- Implementar marcado de registro como transferido en ProquifaDotNet.
- Manejo de errores y logging del paquete.

**Resultado esperado:**
Paquete SSIS funcional que transfiere NCs timbradas a PCconnect de forma idempotente, con manejo de errores y registro de transferencias completadas.

**Entregables:**
- Paquete SSIS `.dtsx` para transferencia de NCs
- Script de control de transferencias (ALTER TABLE o tabla SSIS según decisión de Tarea 12)
- Documentación de configuración del paquete (connection strings, variables, schedule)

**Criterios de aceptación:**
- Ejecución del paquete transfiere NCs VIGENTE no transferidas a PCconnect.
- UUID SAT usado como clave de idempotencia: segunda ejecución no duplica registros.
- NCs con `CancelarFacturaOrigen=1` actualizan estado de factura origen en PCconnect.
- Registros transferidos quedan marcados en ProquifaDotNet.
- Errores en transferencia individual no abortan el paquete completo (manejo por fila).

**Más información de la tarea:**
Ver sección *"Parte F / F2, F3, F4"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032-Back.md` — Parte F
- Análisis de mapeo ETL (entregable de Tarea 12 de este requisito)
- Paquetes SSIS existentes de referencia (Facturas, Pedidos)

---

## TAREA 14

**[ RE-FU-032 ] [QUERY-M] ETL PCconnect — Pruebas de validación de datos transferidos a Legacy**

**Aplicativos:** SSIS / PCconnect (Legacy)

**Módulos:** ETL — Validación transferencia NC a PCconnect

**Consideraciones previas:**
- **Prerrequisito:** Tarea 13 completada (paquete SSIS desarrollado).
- Las pruebas deben cubrir los 4 escenarios de transferencia (NC normal, NC con cancelación de factura, NC modalidad manual, NC por partidas).
- Validar tanto en ProquifaDotNet (marcado como transferido) como en PCconnect (datos recibidos correctamente).
- Incluir prueba de idempotencia (ejecutar el paquete dos veces, verificar sin duplicados).
- Las pruebas deben ejecutarse en ambiente de QA/pruebas, no en producción.

**Objetivo general:**
Validar que los datos de las NCs transferidas por SSIS a PCconnect son correctos, completos y consistentes con los datos origen en ProquifaDotNet, y que el paquete es idempotente.

**Objetivos específicos:**
- Validar cabecera NC: folio, UUID, RFC emisor/receptor, montos, moneda, estado.
- Validar partidas NC (modalidad por partidas): cantidades, precios, importes, IVA.
- Validar concepto manual (modalidad manual): descripción, monto, IVA.
- Validar cancelación de factura origen: estado en PCconnect actualizado correctamente.
- Validar transferencia de rutas PDF/XML.
- Prueba de idempotencia: segunda ejecución del paquete sin duplicados.
- Prueba de error controlado: registrar fallo de una NC y verificar que el paquete continúa con las demás.

**Resultado esperado:**
Paquete SSIS validado con datos correctos en PCconnect para todos los escenarios. Evidencia de pruebas documentada y aprobada.

**Entregables:**
- Documento de resultados de pruebas (escenarios ejecutados, resultados esperados vs reales)
- Scripts de validación SQL en ProquifaDotNet y PCconnect
- Evidencia de prueba de idempotencia

**Scripts de validación (ProquifaDotNet):**
```sql
-- Verificar NCs marcadas como transferidas
SELECT nc.IdFCCNotaCredito, nc.Folio, nc.Serie, nc.Estado,
       cfdi.UUID AS UUID_NC, nc.FechaRegistro
FROM dbo.fccNotaCredito nc
INNER JOIN dbo.CFDIGenerada cg ON nc.IdCFDIGenerada = cg.IdCFDIGenerada
INNER JOIN dbo.CFDI cfdi       ON cg.IdCFDI = cfdi.IdCFDI
WHERE nc.Estado = 'VIGENTE'
ORDER BY nc.FechaRegistro DESC;
-- Contrastar con registros en PCconnect usando UUID_NC como clave
```

**Criterios de aceptación:**
- 100% de NCs VIGENTE en ProquifaDotNet están presentes en PCconnect con UUID correcto.
- Montos, RFC, fechas y estados son idénticos en origen y destino.
- Segunda ejecución del paquete no genera duplicados en PCconnect.
- NCs con `CancelarFacturaOrigen=1` tienen factura origen cancelada en PCconnect.
- Pruebas aprobadas y firmadas por el área de Legacy/PCconnect.

**Más información de la tarea:**
Ver sección *"Parte F"* en `R16A-RE-FU-032-Back.md`.

**Recursos:**
- `R16A-RE-FU-032-Back.md` — Parte F
- Resultados de Tarea 13 (paquete SSIS desarrollado)
