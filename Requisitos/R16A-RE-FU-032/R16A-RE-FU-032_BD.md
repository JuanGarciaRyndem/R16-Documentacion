# Impacto en BD — Notas de Crédito México (CFDI tipo E)
**Requisito:** R16A-RE-FU-032
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (lectura/escritura) + DocumentBuilder (escritura)
**Versión:** 1.0

---

## Resumen

RE-032 habilita el módulo independiente de Notas de Crédito para Región México operado por
Tesorería. Genera CFDIs de tipo E (Egreso) con TipoRelacion=01 vinculados a la factura origen
pagada, con dos modalidades de captura (por partidas / manual) y cancelación condicional de
la factura origen.

Las tablas `fccNotaCredito` y `fccNotaCreditoPartida` **ya existen** en ProquifaDotNet como
estructura heredada del módulo de cobros. RE-032 las extiende con columnas R16 mediante
`ALTER TABLE`.

La tabla `CFDIGenerada` (estructura creada en RE-019, extendida en RE-028) almacena el CFDI
de la NC como `TipoDocumento='E'`. `CFDIGeneradaConcepto` almacena las partidas del XML.
`CFDIGeneradaRelacionado` registra el UUID de la factura origen con `ClaveTipoRelacion='01'`.

Las NCs timbradas se transfieren al sistema legado PCconnect mediante **SSIS** (SQL Server
Integration Services). La estructura destino de PCconnect está pendiente de documentar; esta
sección reserva el análisis de impacto ETL.

> ⚠️ **Pendientes que afectan BD:**
> — `catUsoCFDI` G02: existencia pendiente de verificar con query SSMS.
> — Bucket MinIO para NCs México: pendiente de verificar en `RegionConfiguracionMinioBucket`.
> — ~~Serie "P2" en `EmpresaFolio`: formato del folio pendiente de validar con PMO (Regla 9).~~ **RESUELTO (2026-08-21, DUDA-101):** la serie confirmada para las NC de México es **`B2`**, no `P2`. Resuelto en conjunto con DUDA-113 (fuera de este batch); el formato/longitud exactos del foliador siguen sin detallarse en este documento — ver esa duda antes de ejecutar el DML. Todo el DML y los ejemplos de esta sección que muestran `'P2'` deben leerse como `'B2'` una vez validado el detalle.
> — `catTipoCFDI`: tabla creada en RE-028 como prerequisito; RE-032 inserta clave `notacredito`.
> — ~~FormaPago en modalidad manual: pendiente de confirmar (PMO — Regla 7 del requisito).~~ **RESUELTO (2026-08-21, DUDA-098):** no se hereda un valor fijo de la factura origen — se calcula comparando `NC.Monto` contra el `SaldoPendiente` de la factura (`Factura.Total − Factura.TotalCobrado`, antes de aplicar la NC). `NC ≤ SaldoPendiente` → `FormaPago = 15` (Condonación) siempre; `NC > SaldoPendiente` → forma real de devolución o `FormaPago = 23` (Novación, no Compensación) si queda como saldo a favor. `FormaPago = 99` nunca es válido en una NC (exige `MetodoPago=PPD`, y la NC siempre es `PUE`). Ver `Guia_Tecnica_Notas_de_Credito_MX.md` para el detalle completo.
> — ~~ClaveProdServ y ClaveUnidad en modalidad manual: recomendación SAT 84111506/ACT pendiente de confirmar con PROQUIFA.~~ **RESUELTO (2026-08-21, DUDA-100):** `84111506` / `ACT` es la convención documentada en el Apéndice 5 SAT para NC sin partidas reales — no es una decisión fiscal abierta, queda fija.
> — Estructura PCconnect (Legacy): pendiente de recibir para documentar mapeo ETL completo.
> — Políticas de autorización por monto (PMO #54): **confirmado (2026-08-21, DUDA-102)** que no aplican a las NC en R16 — no requieren cambios de BD.

---

## Impacto en BD

| #   | Cambio                                                                                          | Base de Datos          | Tipo      | Prioridad |
| --- | ----------------------------------------------------------------------------------------------- | ---------------------- | --------- | --------- |
| 1   | ALTER TABLE fccNotaCredito — ADD columnas R16 (empresa, cliente, modalidad, estado, fiscal)    | ProquifaDotNet         | DDL       | Alta      |
| 2   | ALTER TABLE fccNotaCreditoPartida — ADD columnas R16 (concepto origen, importes fiscales)      | ProquifaDotNet         | DDL       | Alta      |
| 3   | DML catUsoCFDI — INSERT clave G02 si no existe                                                  | ProquifaDotNet         | DML       | Alta      |
| 4   | DML catTipoCFDI — INSERT clave notacredito (prereq: RE-028 crea la tabla)                     | ProquifaDotNet         | DML       | Alta      |
| 5   | DML EmpresaFolio — INSERT filas Serie "P2" para GOL, MUN, PRO, PQF                             | ProquifaDotNet (Finanzas) | DML       | Alta      |
| 6   | DML DocumentTemplate — INSERT 4 templates PDF NC México                                         | DocumentBuilder        | DML       | Media     |
| 7   | DML RegionConfiguracionMinioBucket — INSERT bucket NC México si no existe                       | ProquifaDotNet         | DML       | Media     |
| 8   | CREATE TABLE catMotivoCancelacionSAT + DML 4 claves c_MotivoCancelacion SAT                    | ProquifaDotNet         | DDL + DML | Alta      |
| 9   | CREATE TABLE catNotaCreditoEstado + DML 4 estados (PENDIENTE, VIGENTE, ENVIADA, CANCELADA)     | ProquifaDotNet         | DDL + DML | Alta      |
| —   | Reutiliza: `fccNotaCredito` — estructura existente; RE-032 agrega columnas (cambio #1)         | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `fccNotaCreditoPartida` — estructura existente; RE-032 agrega columnas (cambio #2)  | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `fccNotaCreditoPedido` — registro de aplicación de NC a pedidos en Validar Cobro   | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `CFDIGenerada` — INSERT NC como TipoDocumento='E', MetodoDePago='PUE'               | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `CFDIGeneradaConcepto` — INSERT conceptos de la NC (por partidas o manual)          | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `CFDIGeneradaRelacionado` — INSERT UUID factura origen, ClaveTipoRelacion='01'      | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `CFDICancelacion` — INSERT condicional si usuario cancela factura origen (ClaveMotivo) | ProquifaDotNet      | Existente | —         |
| —   | Reutiliza: `CFDI` — poblado por Timbrado tras timbre SAT (UUID, sello, certificado)            | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `Archivo` — INSERT PDF representativo y XML timbrado de la NC                       | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `CorreoEnviado` + `ArchivoCorreoEnviado` — trazabilidad de correo al cliente        | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `catUsoCFDI` — tabla existente; G02 se inserta en cambio #3                         | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: `EmpresaFolio` — estructura existente (RE-019); Serie "P2" en cambio #5             | ProquifaDotNet (Finanzas) | Existente | —         |
| —   | Reutiliza: `DocumentTemplate` — tabla existente (DocumentBuilder)                              | DocumentBuilder        | Existente | —         |
| —   | Reutiliza: `AppSetting` (lectura) + `StampingLog` (INSERT por Stamp/Cancel de la NC)           | ProquifaDotNetTimbrado | Existente | —         |
| —   | ETL SSIS: transferencia de NCs timbradas a PCconnect (Legacy) — mapeo pendiente estructura PCconnect | PCconnect (Legacy) | ETL      | Alta      |

---

## Diccionario de Datos

### ALTER TABLE fccNotaCredito — Columnas R16

La tabla `fccNotaCredito` ya existe. Se agregan las siguientes columnas para soportar el
módulo R16 de Notas de Crédito México.

**Columnas existentes relevantes:**

| Columna                 | Tipo             | Descripción                                    |
| ----------------------- | ---------------- | ---------------------------------------------- |
| `IdFCCNotaCredito`      | uniqueidentifier | PK                                             |
| `IdTPProformaPedido`    | uniqueidentifier | FK al pedido de origen (módulo legacy/pre-R16) |
| `IdCFDI`                | uniqueidentifier | FK a `CFDI` (post-timbrado — UUID SAT)         |
| `IdCFDIGenerada`        | uniqueidentifier | FK a `CFDIGenerada` (CFDI de la NC)            |
| `Monto`                 | decimal(18,6)    | Monto total de la NC                           |
| `TipoDeCambio`          | decimal(18,6)    | TC del día del timbrado (null si MXN)          |
| `MXN` / `USD`           | bit              | Indicadores de moneda                          |
| `Folio`                 | varchar(80)      | Folio interno de la NC                         |
| `Aplicada`              | bit              | Indica si la NC ha sido aplicada a un cobro    |
| `MontoUSD` / `MontoMXN` | decimal(18,6)    | Montos convertidos                             |
| `FechaRegistro`         | datetime         | Auditoría                                      |
| `FechaUltimaActualizacion`         | datetime         | Auditoría                                      |
| `Activo`                | bit              | Registro activo                                |

**Columnas nuevas a agregar (RE-032):**

| Columna                       | Tipo             | Nulable | Descripción                                                                                                                                 |
| ----------------------------- | ---------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `IdEmpresa`                   | uniqueidentifier | No      | FK `Empresa` — empresa emisora (GOL, MUN, PRO, PQF)                                                                                         |
| `IdCliente`                   | uniqueidentifier | No      | FK `Cliente` — cliente receptor de la NC                                                                                                    |
| `Serie`                       | varchar(10)      | Sí      | Serie del foliador interno. ~~Valor: 'P2' (⚠️ pendiente validar con PMO)~~ **Resuelto (DUDA-101): 'B2'** — formato/longitud exactos vinculados a DUDA-113 |
| `Modalidad`                   | varchar(20)      | No      | 'POR_PARTIDAS' o 'MANUAL'                                                                                                                   |
| `Motivo`                      | varchar(50)      | Sí      | Clave del motivo principal. Ej: 'DEVOLUCION', 'DESCUENTO_BONIFICACION'                                                                      |
| `IdCatNotaCreditoEstado`      | uniqueidentifier | No      | FK `catNotaCreditoEstado` — estado de la NC. Default: estado PENDIENTE. Ciclo: PENDIENTE → VIGENTE (timbrada) → ENVIADA; CANCELADA terminal |
| `CancelarFacturaOrigen`       | bit              | No      | 1 si el usuario solicitó cancelar la factura origen ante el SAT. Default 0                                                                  |
| `ClaveMotivosCancelacion`     | varchar(4)       | Sí      | Clave SAT c_MotivoCancelacion ('01','02','03','04'). Null si no cancela                                                                     |
| `IdCFDIGeneradaFacturaOrigen` | uniqueidentifier | No      | FK `CFDIGenerada` — factura PPD que originó esta NC                                                                                         |
| `ConceptoManual`              | nvarchar(500)    | Sí      | Descripción de la materialidad fiscal. Solo modalidad MANUAL                                                                                |
| `ObservacionesManual`         | nvarchar(500)    | Sí      | Campo opcional de observaciones adicionales. Solo modalidad MANUAL                                                                          |
| `IdArchivoXml`                | uniqueidentifier | Sí      | FK `Archivo` — XML timbrado de la NC. Null hasta timbrado exitoso                                                                           |
| `IdArchivoPdf`                | uniqueidentifier | Sí      | FK `Archivo` — PDF representativo de la NC. Null hasta generación                                                                           |

**Relaciones:**

| Tabla relacionada | Columna FK                    | Tipo relación | Descripción                       |
| ----------------- | ----------------------------- | ------------- | --------------------------------- |
| `Empresa`         | `IdEmpresa`                   | N:1           | Empresa emisora de la NC          |
| `Cliente`         | `IdCliente`                   | N:1           | Cliente receptor                  |
| `CFDIGenerada`    | `IdCFDIGenerada`              | 1:1           | CFDI de la NC (TipoDocumento='E') |
| `CFDIGenerada`    | `IdCFDIGeneradaFacturaOrigen` | N:1           | Factura origen de la NC           |
| `catNotaCreditoEstado` | `IdCatNotaCreditoEstado` | N:1           | Estado de la NC (catálogo)        |
| `Archivo`         | `IdArchivoXml`                | N:1           | XML CFDI timbrado                 |
| `Archivo`         | `IdArchivoPdf`                | N:1           | PDF representativo                |

**Índices nuevos:**

| Índice                                      | Columnas                          | Tipo         |
| ------------------------------------------- | --------------------------------- | ------------ |
| `IX_fccNotaCredito_IdCliente`               | `IdCliente`                       | Non-clustered |
| `IX_fccNotaCredito_IdCFDIGeneradaOrigen`    | `IdCFDIGeneradaFacturaOrigen`     | Non-clustered |
| `IX_fccNotaCredito_IdCatNotaCreditoEstado`  | `IdCatNotaCreditoEstado`          | Non-clustered |

---

### ALTER TABLE fccNotaCreditoPartida — Columnas R16

La tabla `fccNotaCreditoPartida` ya existe. Se agregan columnas para soportar la modalidad
por partidas de R16 con trazabilidad al concepto original de la factura.

**Columnas existentes relevantes:**

| Columna                     | Tipo             | Descripción                                          |
| --------------------------- | ---------------- | ---------------------------------------------------- |
| `IdFCCNotaCreditoPartida`   | uniqueidentifier | PK                                                   |
| `IdFCCNotaCredito`          | uniqueidentifier | FK `fccNotaCredito`                                  |
| `IdTPProformaPartidaPedido` | uniqueidentifier | FK partida del pedido origen (módulo legacy/pre-R16) |
| `NumeroDePiezas`            | int              | Cantidad de piezas de la NC por partida              |
| `PrecioUnitario`            | decimal(18,6)    | Precio unitario heredado de la factura origen        |

**Columnas nuevas a agregar (RE-032):**

| Columna                        | Tipo             | Nulable | Descripción                                                                  |
| ------------------------------ | ---------------- | ------- | ---------------------------------------------------------------------------- |
| `IdCFDIGeneradaConceptoOrigen` | uniqueidentifier | Sí      | FK `CFDIGeneradaConcepto` — concepto de la factura origen que se devuelve    |
| `CantidadNC`                   | decimal(18,6)    | Sí      | Cantidad a devolver (0, parcial o igual a cant. facturada). Decimal para MXN |
| `Importe`                      | decimal(18,6)    | Sí      | CantidadNC × ValorUnitario. En moneda de la factura origen                   |
| `Subtotal`                     | decimal(18,6)    | Sí      | Subtotal de la línea (sin IVA)                                               |
| `IVA`                          | decimal(18,6)    | Sí      | IVA calculado sobre el Subtotal de la línea                                  |
| `Total`                        | decimal(18,6)    | Sí      | Subtotal + IVA de la línea                                                   |

**Relaciones nuevas:**

| Tabla relacionada       | Columna FK                      | Tipo relación | Descripción                              |
| ----------------------- | ------------------------------- | ------------- | ---------------------------------------- |
| `CFDIGeneradaConcepto`  | `IdCFDIGeneradaConceptoOrigen`  | N:1           | Concepto original de la factura de origen |

---

### Tablas existentes — Uso en RE-032

#### CFDIGenerada (sin ALTER)

Se inserta una fila por cada NC timbrada con los siguientes valores fijos:

| Campo           | Valor en NC                                             |
| --------------- | ------------------------------------------------------- |
| `TipoDocumento` | `'E'` — Egreso                                          |
| `MetodoDePago`  | `'PUE'` — fijo e inmutable (Regla 6 del requisito)      |
| `UsoCFDI`       | `'G02'` — default (Devoluciones, descuentos o bonif.)   |
| `FormaPago`     | ~~Heredada de la factura origen pagada (típicamente '03')~~ **Resuelto (DUDA-098):** `15` (Condonación) si `NC ≤ SaldoPendiente`; forma real o `23` (Novación) si `NC > SaldoPendiente` — nunca `99` (ver sección de FormaPago en el diccionario y `Guia_Tecnica_Notas_de_Credito_MX.md`) |
| `Moneda`        | Heredada de la factura origen (MXN / USD / EUR)         |
| `TipoDeCambio`  | TC del día del timbrado (null si MXN)                   |
| `Serie`         | ~~'P2' (⚠️ pendiente confirmar con PMO)~~ **Resuelto (DUDA-101): 'B2'** |
| `Folio`         | Consecutivo del foliador `EmpresaFolio` Serie ~~'P2'~~ **'B2'**      |

#### CFDIGeneradaRelacionado (sin ALTER)

Se inserta una fila al crear la NC, vinculando el UUID de la factura origen:

| Campo               | Valor                                                   |
| ------------------- | ------------------------------------------------------- |
| `IdCFDIGenerada`    | IdCFDIGenerada de la NC (TipoDocumento='E')             |
| `UUID`              | UUID SAT de la factura origen (de `CFDI.UUID`)          |
| `ClaveTipoRelacion` | `'01'` — Nota de crédito de los documentos relacionados |

#### CFDICancelacion (sin ALTER, uso condicional)

Se inserta solo cuando el usuario activa "Cancelar Factura Origen" (Regla 8):
condición = NC por totalidad + dentro del mismo mes calendario de la factura.

| Campo         | Valor                                                            |
| ------------- | ---------------------------------------------------------------- |
| `IdCFDI`      | IdCFDI de la **factura origen** (no de la NC)                    |
| `ClaveMotivo` | Clave SAT c_MotivoCancelacion seleccionada ('01','02','03','04') |
| `UUID`        | UUID SAT de la factura origen                                    |
| `Estatus`     | 'CANCELADA' tras confirmación del PAC                            |

#### fccNotaCreditoPedido (sin ALTER)

Tabla existente que registra la aplicación de una NC a un pedido durante el flujo de
Validar Cobro (acoplamiento uni-direccional Regla 13). No requiere cambios en RE-032;
el INSERT lo realiza el módulo de Validar Cobro al aplicar la NC.

---

## DML — catUsoCFDI: INSERT clave G02

La NC usa `UsoCFDI='G02'` (Devoluciones, descuentos o bonificaciones) por default (Regla 6).

> ⚠️ **Verificar antes de ejecutar:**
```sql
-- Ejecutar en ProquifaDotNet
SELECT ClaveUso, Uso, Clave, Activo FROM dbo.catUsoCFDI WHERE ClaveUso = 'G02';
-- Si retorna 0 filas, ejecutar el INSERT siguiente
```

```sql
-- Ejecutar en ProquifaDotNet solo si G02 no existe
INSERT INTO dbo.catUsoCFDI (IdCatUsoCFDI, Uso, ClaveUso, Activo, Clave)
VALUES (NEWID(), 'G02 Devoluciones, descuentos o bonificaciones', 'G02', 1, 'G02');
GO

-- Validación
SELECT ClaveUso, Uso, Activo FROM dbo.catUsoCFDI WHERE ClaveUso = 'G02';
```

---

## DML — catTipoCFDI: INSERT clave notacredito

> **Prerequisito:** La tabla `catTipoCFDI` es creada por RE-028 (Tarea 1). RE-032 inserta
> la clave `notacredito` una vez que dicha tabla exista.

```sql
-- Ejecutar en ProquifaDotNet (después de RE-028 T1)
-- Verificar que no existe
SELECT * FROM dbo.catTipoCFDI WHERE Clave = 'notacredito';

INSERT INTO dbo.catTipoCFDI (IdCatTipoCFDI, Clave, Descripcion, TipoDocumentoSAT, Activo)
VALUES (NEWID(), 'notacredito', 'Nota de Crédito', 'E', 1);
GO

-- Validación
SELECT IdCatTipoCFDI, Clave, Descripcion, TipoDocumentoSAT, Activo
FROM dbo.catTipoCFDI WHERE Clave = 'notacredito';
```

---

## DML — EmpresaFolio: Serie ~~"P2"~~ "B2" para NC México

> ⚠️ **Corrección (2026-08-21, DUDA-101):** la serie confirmada es **`B2`**, no `P2`. El
> script SQL de esta sección todavía usa el literal `'P2'` heredado de la versión anterior del
> requisito — se conserva sin editar porque el formato y la longitud máxima definitivos del
> foliador quedan resueltos en conjunto con DUDA-113 (fuera de este batch), y no corresponde
> inventar aquí ese detalle. Antes de ejecutar el DML, reemplazar todo literal `'P2'` por
> `'B2'` y confirmar `FormatoFolio`/`LongitudMaxima` con la resolución de DUDA-113.

Las NCs de México usan Serie ~~"P2"~~ **"B2"** por empresa del grupo PROQUIFA México. Se insertan 4 filas
en `ProquifaDotNet.EmpresaFolio` (propiedad Finanzas).

> **Nota:** la base de datos es una sola — `ProquifaDotNet`. "Propiedad Finanzas" indica que la tabla
> la consume/gestiona la solución ProquifaDotNet.Finanzas vía su Scaffold EF Core, no que exista
> una base de datos separada.

> ⚠️ **Brecha (parcialmente resuelta):** ~~El esquema definitivo del foliador Serie "P2" (formato, longitud máxima) está pendiente de validar con PMO (Regla 9 del requisito).~~ La serie ya está confirmada (`B2`, DUDA-101/DUDA-113); lo que sigue pendiente es únicamente el formato y la longitud máxima exactos del foliador.

```sql
-- Prerequisito: EmpresaFolio y las 4 empresas México deben existir (RE-019)
-- Ejecutar en ProquifaDotNet
-- ⚠️ Ajustar FormatoFolio y LongitudMaxima según validación con PMO.

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

-- Validación: 4 filas Serie P2
SELECT e.Prefijo, ef.Serie, ef.UltimoFolio, ef.FormatoFolio, ef.LongitudMaxima, ef.Activo
FROM dbo.EmpresaFolio ef
INNER JOIN dbo.Empresa e ON ef.IdEmpresa = e.IdEmpresa
WHERE ef.Serie = 'P2'
ORDER BY e.Prefijo;
```

---

## DML — DocumentTemplate: Templates PDF NC México

Se registran 4 templates en `DocumentBuilder.DocumentTemplate`, uno por empresa emisora
México. Convención de clave: `{PREFIJO}_MEX_NC`.

```sql
-- Ejecutar en DocumentBuilder
-- Verificar ausencia antes de insertar
SELECT TemplateKey FROM dbo.DocumentTemplate
WHERE TemplateKey IN ('GOL_MEX_NC','MUN_MEX_NC','PRO_MEX_NC','PQF_MEX_NC');
-- Debe retornar 0 filas antes de continuar

-- Golocaer México
INSERT INTO dbo.DocumentTemplate (
    TemplateKey,
    HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName,
    HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate,
    RegistrationDate, LastUpdateDate, IsActive
)
VALUES ('GOL_MEX_NC', 'GOL_MEX_NC_H.html', 'GOL_MEX_NC_B.html', 'GOL_MEX_NC_F.html', 1, 1, 1, GETDATE(), GETDATE(), 1);

-- Mungen México
INSERT INTO dbo.DocumentTemplate (
    TemplateKey,
    HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName,
    HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate,
    RegistrationDate, LastUpdateDate, IsActive
)
VALUES ('MUN_MEX_NC', 'MUN_MEX_NC_H.html', 'MUN_MEX_NC_B.html', 'MUN_MEX_NC_F.html', 1, 1, 1, GETDATE(), GETDATE(), 1);

-- Proquifa México
INSERT INTO dbo.DocumentTemplate (
    TemplateKey,
    HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName,
    HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate,
    RegistrationDate, LastUpdateDate, IsActive
)
VALUES ('PRO_MEX_NC', 'PRO_MEX_NC_H.html', 'PRO_MEX_NC_B.html', 'PRO_MEX_NC_F.html', 1, 1, 1, GETDATE(), GETDATE(), 1);

-- Proveedora Quimico Farmaceutica México
INSERT INTO dbo.DocumentTemplate (
    TemplateKey,
    HeaderTemplateFileName, BodyTemplateFileName, FooterTemplateFileName,
    HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate,
    RegistrationDate, LastUpdateDate, IsActive
)
VALUES ('PQF_MEX_NC', 'PQF_MEX_NC_H.html', 'PQF_MEX_NC_B.html', 'PQF_MEX_NC_F.html', 1, 1, 1, GETDATE(), GETDATE(), 1);
GO

-- Validación
SELECT TemplateKey, BodyTemplateFileName, HasHeaderTemplate, HasBodyTemplate, HasFooterTemplate, IsActive
FROM dbo.DocumentTemplate
WHERE TemplateKey IN ('GOL_MEX_NC','MUN_MEX_NC','PRO_MEX_NC','PQF_MEX_NC')
ORDER BY TemplateKey;
```

---

## MinIO — Configuración de Bucket NC México

Los archivos de la NC (PDF representativo y XML timbrado) se almacenan en MinIO.

> ⚠️ **Verificar antes de ejecutar:**
```sql
-- Ejecutar en ProquifaDotNet
SELECT rcmb.BucketClave, rcmb.BucketNombre, r.Clave AS Region, rcmb.Activo
FROM dbo.RegionConfiguracionMinioBucket rcmb
INNER JOIN dbo.Region r ON rcmb.IdRegion = r.IdRegion
WHERE r.Clave = 'MEX' AND rcmb.Activo = 1
ORDER BY rcmb.BucketClave;
-- Verificar si existe un bucket 'notas_credito' o 'facturas' para MEX
```

Si no existe un bucket dedicado para NCs, insertar:

```sql
-- Ejecutar en ProquifaDotNet solo si no existe bucket 'notas_credito' para MEX
INSERT INTO dbo.RegionConfiguracionMinioBucket (
    IdRegionConfiguracionMinioBucket, IdRegion,
    BucketClave, BucketNombre, BucketDescripcion,
    FechaRegistro, FechaUltimaActualizacion, Activo, EsRegionalizable
)
SELECT NEWID(), r.IdRegion,
    'notas_credito',
    'notas-credito-mex',
    'Bucket para PDF y XML de Notas de Crédito México',
    GETDATE(), GETDATE(), 1, 1
FROM dbo.Region r WHERE r.Clave = 'MEX';
GO

-- Validación
SELECT rcmb.BucketClave, rcmb.BucketNombre, r.Clave AS Region
FROM dbo.RegionConfiguracionMinioBucket rcmb
INNER JOIN dbo.Region r ON rcmb.IdRegion = r.IdRegion
WHERE rcmb.BucketClave = 'notas_credito' AND r.Clave = 'MEX';
```

### Estructura de rutas en MinIO

| Documento | Ruta                                                         |
| --------- | ------------------------------------------------------------ |
| PDF NC    | `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID_NC}.pdf` |
| XML NC    | `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID_NC}.xml` |

### Consulta de resolución de bucket (Finanzas)

```sql
-- Ejecutar en ProquifaDotNet
SELECT rcmb.BucketNombre
FROM dbo.RegionConfiguracionMinioBucket rcmb
INNER JOIN dbo.Region r ON rcmb.IdRegion = r.IdRegion
WHERE rcmb.BucketClave = 'notas_credito'
  AND r.Clave           = 'MEX'
  AND rcmb.Activo       = 1
```

---

## ETL — Transferencia a PCconnect (Legacy) vía SSIS

Las Notas de Crédito timbradas en ProquifaDotNet deben transferirse al sistema legado
**PCconnect** mediante **SQL Server Integration Services (SSIS)**, siguiendo el mismo
patrón de transferencia que ya existe para otros documentos fiscales (Facturas, Pedidos,
Cobros).

> ⚠️ **Pendiente de documentar completamente:** La estructura de tablas destino en PCconnect
> no ha sido compartida. Una vez disponible se documentará el mapeo columna a columna.
> Esta sección reserva el análisis de impacto y define los datos que ProquifaDotNet
> debe exponer para la transferencia.

### Alcance de la transferencia

| Evento que dispara el ETL                       | Datos a transferir                         |
| ----------------------------------------------- | ------------------------------------------ |
| NC timbrada exitosamente (`fccNotaCredito`)     | Cabecera de la NC (ver tabla abajo)        |
| NC con modalidad por partidas                   | Partidas de la NC (`fccNotaCreditoPartida`) |
| Factura origen cancelada por la NC              | Estado de cancelación de la factura origen |
| NC aplicada a cobro en Validar Cobro            | Aplicación de NC a pedido (`fccNotaCreditoPedido`) |

### Datos de cabecera NC a transferir

| Campo ProquifaDotNet                            | Descripción                                |
| ----------------------------------------------- | ------------------------------------------ |
| `fccNotaCredito.IdFCCNotaCredito`               | Identificador interno (referencia cruzada) |
| `fccNotaCredito.Serie` + `Folio`                | Folio interno de la NC (Serie P2 + número) |
| `CFDI.UUID` (vía `IdCFDI`)                      | UUID SAT del CFDI de la NC                 |
| `CFDIGenerada.RFCEmisor`                        | RFC de la empresa emisora                  |
| `CFDIGenerada.RFCReceptor`                      | RFC del receptor (cliente)                 |
| `CFDIGenerada.FechaEmision`                     | Fecha de emisión y timbrado                |
| `CFDIGenerada.Subtotal` / `Total`               | Importes de la NC                          |
| `CFDIGenerada.Moneda` + `TipoDeCambio`          | Moneda y tipo de cambio                    |
| `catNotaCreditoEstado.Clave` (vía `IdCatNotaCreditoEstado`) | Estado: VIGENTE / ENVIADA / CANCELADA      |
| `fccNotaCredito.Modalidad`                      | Modalidad: POR_PARTIDAS / MANUAL           |
| `fccNotaCredito.Motivo`                         | Motivo de la NC                            |
| `CFDIGeneradaRelacionado.UUID`                  | UUID de la factura origen relacionada      |
| `fccNotaCredito.CancelarFacturaOrigen`          | Indicador de cancelación de factura origen |
| `fccNotaCredito.ClaveMotivosCancelacion`        | Motivo de cancelación SAT (si aplica)      |
| `Archivo.FileKey` (IdArchivoPdf / IdArchivoXml) | Rutas MinIO del PDF y XML                  |

### Datos de partidas NC a transferir (modalidad por partidas)

| Campo ProquifaDotNet                                | Descripción                              |
| --------------------------------------------------- | ---------------------------------------- |
| `fccNotaCreditoPartida.IdFCCNotaCredito`            | FK a cabecera                            |
| `CFDIGeneradaConcepto.ClaveProductoServicio`        | Clave SAT del producto                   |
| `CFDIGeneradaConcepto.NoIdentificacion`             | Código interno del producto              |
| `CFDIGeneradaConcepto.Descripcion`                  | Descripción del concepto                 |
| `fccNotaCreditoPartida.CantidadNC`                  | Cantidad devuelta                        |
| `fccNotaCreditoPartida.PrecioUnitario`              | Precio unitario heredado de la factura   |
| `fccNotaCreditoPartida.Importe`                     | Importe de la línea                      |
| `fccNotaCreditoPartida.IVA`                         | IVA de la línea                          |
| `fccNotaCreditoPartida.Total`                       | Total de la línea                        |

### Consideraciones SSIS

- **Trigger de transferencia:** Después del timbrado exitoso de la NC (estado `VIGENTE`
  del catálogo `catNotaCreditoEstado` en `fccNotaCredito`, `IdCFDI` poblado en `CFDIGenerada`).
- **Paquete SSIS nuevo:** Se requiere un nuevo paquete SSIS específico para NCs
  (no se reutiliza el de Facturas, ya que el tipo de documento y las tablas destino difieren).
- **Manejo de cancelación:** Si la factura origen se cancela simultáneamente, el paquete
  debe actualizar el estado de la factura en PCconnect además de insertar la NC.
- **Idempotencia:** El paquete debe manejar reenvíos sin duplicar registros en PCconnect
  (clave natural sugerida: UUID SAT de la NC).
- **Mapeo completo:** Pendiente hasta recibir estructura de tablas PCconnect.

---

## Estructuras reutilizadas — Resumen de uso en RE-032

| Estructura                         | Origen    | Uso en RE-032                                                                  |
| ---------------------------------- | --------- | ------------------------------------------------------------------------------ |
| `fccNotaCredito`                   | Pre-R16   | Base extendida con columnas R16 (ALTER TABLE, cambio #1)                      |
| `fccNotaCreditoPartida`            | Pre-R16   | Base extendida con columnas R16 (ALTER TABLE, cambio #2)                      |
| `fccNotaCreditoPedido`             | Pre-R16   | Registro de aplicación de NC a pedido en Validar Cobro (sin cambios)          |
| `CFDIGenerada`                     | RE-019    | INSERT NC como TipoDocumento='E', MetodoDePago='PUE'                          |
| `CFDIGeneradaConcepto`             | RE-019    | INSERT conceptos de la NC (por partidas o concepto manual)                    |
| `CFDIGeneradaRelacionado`          | Pre-R16   | INSERT UUID factura origen con ClaveTipoRelacion='01'                         |
| `CFDICancelacion`                  | Pre-R16   | INSERT condicional si se cancela la factura origen (ClaveMotivo SAT)          |
| `CFDI`                             | Pre-R16   | Poblado por Timbrado con UUID SAT, sello y certificado tras timbrar           |
| `catTipoCFDI.notacredito`         | RE-028 T1 | Discriminador de tipo CFDI; RE-032 inserta la clave (prereq: tabla de RE-028) |
| `EmpresaFolio` (estructura)        | RE-019    | Foliador UPDLOCK atómico; RE-032 agrega filas Serie 'P2'                      |
| `Archivo`                          | Pre-R16   | PDF + XML de la NC almacenados en MinIO                                       |
| `CorreoEnviado`                    | Pre-R16   | Trazabilidad del correo automático con NC al timbrar y reenvíos               |
| `ArchivoCorreoEnviado`             | Pre-R16   | PDF + XML adjuntos al correo de la NC                                         |
| `RegionConfiguracionMinioBucket`   | Pre-R16   | Resolución del bucket 'notas_credito' (MEX) al subir PDF/XML                 |
| PAC TurboPac                       | RE-019    | Mismo cliente/servicio para timbrar la NC vía ProquifaDotNet.Timbrado         |

---

## Diccionario de Datos — catMotivoCancelacionSAT (tabla nueva)

**Base de datos:** ProquifaDotNet
**Descripción:** Catálogo oficial SAT c_MotivoCancelacion para CFDI 4.0. Almacena las 4 claves permitidas al cancelar un CFDI ante el SAT. El frontend consulta este catálogo para mostrar el selector de motivo al cancelar una Nota de Crédito.

### Columnas

| Nombre | Tipo | Nulable | Default | Descripción |
|---|---|---|---|---|
| `IdCatMotivoCancelacionSAT` | `uniqueidentifier` | NO | `NEWID()` | PK — Identificador único |
| `Clave` | `varchar(4)` | NO | — | Clave SAT c_MotivoCancelacion UNIQUE ('01','02','03','04') |
| `Descripcion` | `nvarchar(150)` | NO | — | Descripción oficial SAT del motivo |
| `Activo` | `bit` | NO | `1` | Control de vigencia |
| `FechaRegistro` | `datetime` | NO | `GETDATE()` | Fecha de inserción |
| `FechaUltimaActualizacion` | `datetime` | NO | `GETDATE()` | Fecha de última modificación |

### Índices

| Nombre | Columnas | Tipo |
|---|---|---|
| `PK_catMotivoCancelacionSAT` | `IdCatMotivoCancelacionSAT` | PK Clustered |
| `UQ_catMotivoCancelacionSAT_Clave` | `Clave` | Unique NonClustered |

### Relaciones

Ninguna FK saliente. La `Clave` es referenciada lógicamente por `fccNotaCredito.ClaveMotivosCancelacion varchar(4)` (sin FK formal — mismo patrón que `CFDICancelacion.ClaveMotivo`).

### Datos iniciales (c_MotivoCancelacion SAT — CFDI 4.0)

| Clave | Descripcion |
|---|---|
| `01` | Comprobante emitido con errores con relación |
| `02` | Comprobante emitido con errores sin relación |
| `03` | No se llevó a cabo la operación |
| `04` | Operación nominativa relacionada en una factura global |

### Consideraciones especiales

- Catálogo **estático** — las claves son definidas por el SAT y cambian excepcionalmente. No requiere administración en la aplicación.
- La clave `01` ("con relación") aplica cuando existe un CFDI sustituto. La clave `02` aplica cuando no. Las claves `03` y `04` son para otros escenarios de cancelación. Para NCs el motivo más común es `01`.
- El frontend debe enviar la `Clave` (no el GUID) en el body del request de cancelación. El backend la guarda en `fccNotaCredito.ClaveMotivosCancelacion`.

---

## ProquifaDotNetTimbrado — Uso en RE-032

`ProquifaDotNetTimbrado` **sí es una base de datos aparte**, propia de la solución ProquifaDotNet.Timbrado. RE-032 no crea tablas nuevas en ella — reutiliza las existentes:

### AppSetting (lectura)

Configuración operativa de la solución Timbrado (parámetros del PAC, timeouts, reintentos).

| Columna | Tipo | Descripción |
|---|---|---|
| `Id` | uniqueidentifier | PK |
| `Name` | varchar | Nombre del parámetro |
| `Value` | varchar | Valor del parámetro |
| `Description` | varchar | Descripción |
| `CreatedAt` / `UpdatedAt` | datetime | Auditoría |
| `IsActive` | bit | Vigencia |

### StampingLog (escritura — un INSERT por operación)

Bitácora de las operaciones de timbrado y cancelación. RE-032 inserta una fila por cada llamada de la NC: timbrado (`Action='Stamp'`, endpoint C1) y cancelación de la factura origen (`Action='Cancel'`, endpoint C2).

| Columna | Tipo | Descripción |
|---|---|---|
| `Id` | uniqueidentifier | PK |
| `CfdiGeneradaId` | uniqueidentifier | Referencia informativa cross-database a `ProquifaDotNet.CFDIGenerada` (sin FK) |
| `Action` | varchar | `Stamp` \| `Cancel` |
| `PreviousStatus` | varchar | Estado previo de la operación |
| `NewStatus` | varchar | `Pending` \| `Stamped` \| `Failed` |
| `Request` | varchar | Payload enviado al PAC |
| `Response` | varchar | Respuesta del PAC |
| `ErrorMessage` | varchar | Detalle del error (si aplica) |
| `DurationMs` | int | Duración de la operación |
| `CreatedAt` | datetime | Fecha de la operación |
| `IsActive` | bit | Vigencia |

**Consideraciones:**

- `CfdiGeneradaId` es referencia cruzada entre bases (ProquifaDotNetTimbrado → ProquifaDotNet), por lo que no lleva FK formal.
- En un timbrado fallido (Regla 16), la NC no se persiste en ProquifaDotNet pero el intento **sí** queda registrado en `StampingLog` con `NewStatus='Failed'` y el detalle del PAC en `Response`/`ErrorMessage`.

---

## Diccionario de Datos — catNotaCreditoEstado (tabla nueva)

**Base de datos:** ProquifaDotNet
**Descripción:** Catálogo de estados del ciclo de vida de la Nota de Crédito R16. Reemplaza el dominio en texto libre de la columna `Estado` — `fccNotaCredito` referencia este catálogo vía `IdCatNotaCreditoEstado`.

### Columnas

| Nombre                   | Tipo               | Nulable | Default     | Descripción                                                           |
| ------------------------ | ------------------ | ------- | ----------- | --------------------------------------------------------------------- |
| `IdCatNotaCreditoEstado` | `uniqueidentifier` | NO      | `NEWID()`   | PK — Identificador único                                              |
| `Clave`                  | `varchar(20)`      | NO      | —           | Clave del estado UNIQUE ('PENDIENTE','VIGENTE','ENVIADA','CANCELADA') |
| `Descripcion`            | `nvarchar(150)`    | NO      | —           | Descripción del estado                                                |
| `Activo`                 | `bit`              | NO      | `1`         | Control de vigencia                                                   |
| `FechaRegistro`          | `datetime`        | NO      | `GETDATE()` | Fecha de inserción                                                    |
| `FechaUltimaActualizacion`          | `datetime`        | NO      | `GETDATE()` | Fecha de última modificación |

### Índices

| Nombre | Columnas | Tipo |
|---|---|---|
| `PK_catNotaCreditoEstado` | `IdCatNotaCreditoEstado` | PK Clustered |
| `UQ_catNotaCreditoEstado_Clave` | `Clave` | Unique NonClustered |

### Relaciones

Referenciada por `fccNotaCredito.IdCatNotaCreditoEstado` (FK formal, N:1).

### Datos iniciales

| Clave       | Descripcion                                |
| ----------- | ------------------------------------------ |
| `PENDIENTE` | NC capturada, pendiente de timbrar         |
| `VIGENTE`   | NC timbrada ante el SAT (vigente)          |
| `ENVIADA`   | NC timbrada y correo enviado al cliente    |
| `CANCELADA` | NC cancelada ante el SAT — estado terminal |

### Consideraciones especiales

- **Ciclo de vida:** `PENDIENTE → VIGENTE (timbrada) → ENVIADA`; `CANCELADA` es terminal y puede alcanzarse desde VIGENTE o ENVIADA.
- Default de `fccNotaCredito.IdCatNotaCreditoEstado`: estado `PENDIENTE` (la NC se crea pre-timbrado en el Paso 2 del wizard).
- Las NCs disponibles para aplicación en Validar Cobro y para transferencia SSIS a PCconnect son las de estado `VIGENTE` o `ENVIADA` (timbradas, no canceladas).
- Si el timbrado falla, la NC permanece en `PENDIENTE` y el usuario puede reintentar (Regla 16).

---

## Pendientes

| ID | Pendiente                                                                                                 | Sección afectada              |
| -- | --------------------------------------------------------------------------------------------------------- | ----------------------------- |
| P1 | Verificar existencia de `catUsoCFDI` G02 en BD real (query SSMS)                                        | DML catUsoCFDI                |
| P2 | Verificar bucket MinIO para NCs en `RegionConfiguracionMinioBucket` (query SSMS)                        | MinIO                         |
| P3 | ~~Validar Serie "P2" y formato de folio con PMO (Regla 9 del requisito)~~ — **RESUELTO PARCIAL (2026-08-21, DUDA-101):** serie confirmada `B2`; solo queda pendiente el formato/longitud exacto, vinculado a DUDA-113 | DML EmpresaFolio              |
| P4 | Recibir estructura de tablas PCconnect para completar mapeo ETL SSIS                                    | ETL PCconnect                 |
| P5 | ~~Confirmar FormaPago en modalidad manual (Regla 7 — ⚠️ '99' fiscalmente incorrecto para NC PUE)~~ — **RESUELTO (2026-08-21, DUDA-098):** se resuelve por comparación NC vs SaldoPendiente — `15` Condonación / forma real / `23` Novación; `99` queda formalmente excluido | CFDIGenerada / fccNotaCredito |
| P6 | ~~Confirmar ClaveProdServ y ClaveUnidad default para modalidad manual (candidato SAT: 84111506 / ACT)~~ — **RESUELTO (2026-08-21, DUDA-100):** convención confirmada, `84111506` / `ACT` (Apéndice 5 SAT) | CFDIGeneradaConcepto          |
| P7 | Confirmar estructura de `catTipoCFDI` con RE-028 para validar columnas del INSERT de notacredito       | DML catTipoCFDI               |
| P8 | Políticas de autorización por monto (PMO #54) — **RESUELTO (2026-08-21, DUDA-102):** no aplica código de autorización para NC en R16; sin impacto en BD | — |

---

## Brechas

| ID | Brecha                                                                                                              | Impacto                              |
| -- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| B1 | `fccNotaCredito.IdTPProformaPedido` es NOT NULL en la tabla existente. Las NCs R16 no vienen de un pedido directo — requiere confirmar si se puede poner NULL o si necesita un valor placeholder | ALTER TABLE, integridad referencial |
| B2 | `fccNotaCreditoPartida.NumeroDePiezas` es `int` (no decimal). Si un producto tiene cantidad fraccionaria necesita cambio de tipo | ALTER TABLE fccNotaCreditoPartida   |
| B3 | Estructura PCconnect desconocida — el paquete SSIS no puede diseñarse hasta tener el esquema destino              | ETL SSIS                             |
| B4 | No existe `catMotivoCancelacionSAT` como tabla catálogo — `CFDICancelacion.ClaveMotivo` es varchar libre. Si se requiere validación en UI, el front debe traer el catálogo SAT c_MotivoCancelacion desde config o hardcode | Validación de negocio               |
