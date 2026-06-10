# Impacto en BD — Notas de Crédito México (CFDI tipo E)
**Requisito:** TPSC-RE-FU-032
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
> — Serie "P2" en `EmpresaFolio`: formato del folio pendiente de validar con PMO (Regla 9).
> — `catTipoCFDI`: tabla creada en RE-028 como prerequisito; RE-032 inserta clave `NOTA_CREDITO`.
> — FormaPago en modalidad manual: pendiente de confirmar (PMO — Regla 7 del requisito).
> — ClaveProdServ y ClaveUnidad en modalidad manual: recomendación SAT 84111506/ACT pendiente
>   de confirmar con PROQUIFA.
> — Estructura PCconnect (Legacy): pendiente de recibir para documentar mapeo ETL completo.

---

## Impacto en BD

| #   | Cambio                                                                                          | Base de Datos          | Tipo      | Prioridad |
| --- | ----------------------------------------------------------------------------------------------- | ---------------------- | --------- | --------- |
| 1   | ALTER TABLE fccNotaCredito — ADD columnas R16 (empresa, cliente, modalidad, estado, fiscal)    | ProquifaDotNet         | DDL       | Alta      |
| 2   | ALTER TABLE fccNotaCreditoPartida — ADD columnas R16 (concepto origen, importes fiscales)      | ProquifaDotNet         | DDL       | Alta      |
| 3   | DML catUsoCFDI — INSERT clave G02 si no existe                                                  | ProquifaDotNet         | DML       | Alta      |
| 4   | DML catTipoCFDI — INSERT clave NOTA_CREDITO (prereq: RE-028 crea la tabla)                     | ProquifaDotNet         | DML       | Alta      |
| 5   | DML EmpresaFolio — INSERT filas Serie "P2" para GOL, MUN, PRO, PQF                             | ProquifaDotNetTimbrado | DML       | Alta      |
| 6   | DML DocumentTemplate — INSERT 4 templates PDF NC México                                         | DocumentBuilder        | DML       | Media     |
| 7   | DML RegionConfiguracionMinioBucket — INSERT bucket NC México si no existe                       | ProquifaDotNet         | DML       | Media     |
| 8   | CREATE TABLE catMotivoCancelacionSAT + DML 4 claves c_MotivoCancelacion SAT                    | ProquifaDotNet         | DDL + DML | Alta      |
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
| —   | Reutiliza: `EmpresaFolio` — estructura existente (RE-019); Serie "P2" en cambio #5             | ProquifaDotNetTimbrado | Existente | —         |
| —   | Reutiliza: `DocumentTemplate` — tabla existente (DocumentBuilder)                              | DocumentBuilder        | Existente | —         |
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
| `Activo`                | bit              | Registro activo                                |

**Columnas nuevas a agregar (RE-032):**

| Columna                        | Tipo              | Nulable | Descripción                                                                 |
| ------------------------------ | ----------------- | ------- | --------------------------------------------------------------------------- |
| `IdEmpresa`                    | uniqueidentifier  | No      | FK `Empresa` — empresa emisora (GOL, MUN, PRO, PQF)                        |
| `IdCliente`                    | uniqueidentifier  | No      | FK `Cliente` — cliente receptor de la NC                                   |
| `Serie`                        | varchar(10)       | Sí      | Serie del foliador interno. Valor: 'P2' (⚠️ pendiente validar con PMO)    |
| `Modalidad`                    | varchar(20)       | No      | 'POR_PARTIDAS' o 'MANUAL'                                                  |
| `Motivo`                       | varchar(50)       | Sí      | Clave del motivo principal. Ej: 'DEVOLUCION', 'DESCUENTO_BONIFICACION'     |
| `Estado`                       | varchar(20)       | No      | 'VIGENTE' \| 'CANCELADA'. Default 'VIGENTE'                                |
| `CancelarFacturaOrigen`        | bit               | No      | 1 si el usuario solicitó cancelar la factura origen ante el SAT. Default 0 |
| `ClaveMotivosCancelacion`      | varchar(4)        | Sí      | Clave SAT c_MotivoCancelacion ('01','02','03','04'). Null si no cancela     |
| `IdCFDIGeneradaFacturaOrigen`  | uniqueidentifier  | No      | FK `CFDIGenerada` — factura PPD que originó esta NC                        |
| `ConceptoManual`               | nvarchar(500)     | Sí      | Descripción de la materialidad fiscal. Solo modalidad MANUAL               |
| `ObservacionesManual`          | nvarchar(500)     | Sí      | Campo opcional de observaciones adicionales. Solo modalidad MANUAL         |
| `IdArchivoXml`                 | uniqueidentifier  | Sí      | FK `Archivo` — XML timbrado de la NC. Null hasta timbrado exitoso          |
| `IdArchivoPdf`                 | uniqueidentifier  | Sí      | FK `Archivo` — PDF representativo de la NC. Null hasta generación          |

**Relaciones:**

| Tabla relacionada   | Columna FK                      | Tipo relación | Descripción                               |
| ------------------- | ------------------------------- | ------------- | ----------------------------------------- |
| `Empresa`           | `IdEmpresa`                     | N:1           | Empresa emisora de la NC                  |
| `Cliente`           | `IdCliente`                     | N:1           | Cliente receptor                          |
| `CFDIGenerada`      | `IdCFDIGenerada`                | 1:1           | CFDI de la NC (TipoDocumento='E')         |
| `CFDIGenerada`      | `IdCFDIGeneradaFacturaOrigen`   | N:1           | Factura origen de la NC                   |
| `Archivo`           | `IdArchivoXml`                  | N:1           | XML CFDI timbrado                         |
| `Archivo`           | `IdArchivoPdf`                  | N:1           | PDF representativo                        |

**Índices nuevos:**

| Índice                                      | Columnas                          | Tipo         |
| ------------------------------------------- | --------------------------------- | ------------ |
| `IX_fccNotaCredito_IdCliente`               | `IdCliente`                       | Non-clustered |
| `IX_fccNotaCredito_IdCFDIGeneradaOrigen`    | `IdCFDIGeneradaFacturaOrigen`     | Non-clustered |
| `IX_fccNotaCredito_Estado`                  | `Estado`                          | Non-clustered |

---

### ALTER TABLE fccNotaCreditoPartida — Columnas R16

La tabla `fccNotaCreditoPartida` ya existe. Se agregan columnas para soportar la modalidad
por partidas de R16 con trazabilidad al concepto original de la factura.

**Columnas existentes relevantes:**

| Columna                      | Tipo             | Descripción                                               |
| ---------------------------- | ---------------- | --------------------------------------------------------- |
| `IdFCCNotaCreditoPartida`    | uniqueidentifier | PK                                                        |
| `IdFCCNotaCredito`           | uniqueidentifier | FK `fccNotaCredito`                                       |
| `IdTPProformaPartidaPedido`  | uniqueidentifier | FK partida del pedido origen (módulo legacy/pre-R16)      |
| `NumeroDePiezas`             | int              | Cantidad de piezas de la NC por partida                   |
| `PrecioUnitario`             | decimal(18,6)    | Precio unitario heredado de la factura origen             |

**Columnas nuevas a agregar (RE-032):**

| Columna                         | Tipo             | Nulable | Descripción                                                                 |
| ------------------------------- | ---------------- | ------- | --------------------------------------------------------------------------- |
| `IdCFDIGeneradaConceptoOrigen`  | uniqueidentifier | Sí      | FK `CFDIGeneradaConcepto` — concepto de la factura origen que se devuelve   |
| `CantidadNC`                    | decimal(18,6)    | Sí      | Cantidad a devolver (0, parcial o igual a cant. facturada). Decimal para MXN |
| `Importe`                       | decimal(18,6)    | Sí      | CantidadNC × ValorUnitario. En moneda de la factura origen                  |
| `Subtotal`                      | decimal(18,6)    | Sí      | Subtotal de la línea (sin IVA)                                              |
| `IVA`                           | decimal(18,6)    | Sí      | IVA calculado sobre el Subtotal de la línea                                 |
| `Total`                         | decimal(18,6)    | Sí      | Subtotal + IVA de la línea                                                  |

**Relaciones nuevas:**

| Tabla relacionada       | Columna FK                      | Tipo relación | Descripción                              |
| ----------------------- | ------------------------------- | ------------- | ---------------------------------------- |
| `CFDIGeneradaConcepto`  | `IdCFDIGeneradaConceptoOrigen`  | N:1           | Concepto original de la factura de origen |

---

### Tablas existentes — Uso en RE-032

#### CFDIGenerada (sin ALTER)

Se inserta una fila por cada NC timbrada con los siguientes valores fijos:

| Campo          | Valor en NC                                            |
| -------------- | ------------------------------------------------------ |
| `TipoDocumento`| `'E'` — Egreso                                        |
| `MetodoDePago` | `'PUE'` — fijo e inmutable (Regla 6 del requisito)    |
| `UsoCFDI`      | `'G02'` — default (Devoluciones, descuentos o bonif.) |
| `FormaPago`    | Heredada de la factura origen pagada (típicamente '03') |
| `Moneda`       | Heredada de la factura origen (MXN / USD / EUR)        |
| `TipoDeCambio` | TC del día del timbrado (null si MXN)                  |
| `Serie`        | 'P2' (⚠️ pendiente confirmar con PMO)                 |
| `Folio`        | Consecutivo del foliador `EmpresaFolio` Serie 'P2'     |

#### CFDIGeneradaRelacionado (sin ALTER)

Se inserta una fila al crear la NC, vinculando el UUID de la factura origen:

| Campo               | Valor                                                       |
| ------------------- | ----------------------------------------------------------- |
| `IdCFDIGenerada`    | IdCFDIGenerada de la NC (TipoDocumento='E')                 |
| `UUID`              | UUID SAT de la factura origen (de `CFDI.UUID`)              |
| `ClaveTipoRelacion` | `'01'` — Nota de crédito de los documentos relacionados    |

#### CFDICancelacion (sin ALTER, uso condicional)

Se inserta solo cuando el usuario activa "Cancelar Factura Origen" (Regla 8):
condición = NC por totalidad + dentro del mismo mes calendario de la factura.

| Campo          | Valor                                                       |
| -------------- | ----------------------------------------------------------- |
| `IdCFDI`       | IdCFDI de la **factura origen** (no de la NC)              |
| `ClaveMotivo`  | Clave SAT c_MotivoCancelacion seleccionada ('01','02','03','04') |
| `UUID`         | UUID SAT de la factura origen                               |
| `Estatus`      | 'CANCELADA' tras confirmación del PAC                       |

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

## DML — catTipoCFDI: INSERT clave NOTA_CREDITO

> **Prerequisito:** La tabla `catTipoCFDI` es creada por RE-028 (Tarea 1). RE-032 inserta
> la clave `NOTA_CREDITO` una vez que dicha tabla exista.

```sql
-- Ejecutar en ProquifaDotNet (después de RE-028 T1)
-- Verificar que no existe
SELECT * FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO';

INSERT INTO dbo.catTipoCFDI (IdCatTipoCFDI, Clave, Descripcion, TipoDocumentoSAT, Activo)
VALUES (NEWID(), 'NOTA_CREDITO', 'Nota de Crédito', 'E', 1);
GO

-- Validación
SELECT IdCatTipoCFDI, Clave, Descripcion, TipoDocumentoSAT, Activo
FROM dbo.catTipoCFDI WHERE Clave = 'NOTA_CREDITO';
```

---

## DML — EmpresaFolio: Serie "P2" para NC México

Las NCs de México usan Serie "P2" por empresa del grupo PROQUIFA México. Se insertan 4 filas
en `ProquifaDotNetTimbrado.EmpresaFolio`.

> ⚠️ **Brecha:** El esquema definitivo del foliador Serie "P2" (formato, longitud máxima)
> está pendiente de validar con PMO (Regla 9 del requisito).

```sql
-- Prerequisito: EmpresaFolio y las 4 empresas México deben existir (RE-019)
-- Ejecutar en ProquifaDotNetTimbrado
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

| Documento     | Ruta                                                    |
| ------------- | ------------------------------------------------------- |
| PDF NC        | `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID_NC}.pdf` |
| XML NC        | `notas-credito-mex/notas_credito/{anio}/{mes}/{UUID_NC}.xml` |

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

| Campo ProquifaDotNet                           | Descripción                                             |
| ---------------------------------------------- | ------------------------------------------------------- |
| `fccNotaCredito.IdFCCNotaCredito`              | Identificador interno (referencia cruzada)              |
| `fccNotaCredito.Serie` + `Folio`               | Folio interno de la NC (Serie P2 + número)              |
| `CFDI.UUID` (vía `IdCFDI`)                     | UUID SAT del CFDI de la NC                              |
| `CFDIGenerada.RFCEmisor`                       | RFC de la empresa emisora                               |
| `CFDIGenerada.RFCReceptor`                     | RFC del receptor (cliente)                              |
| `CFDIGenerada.FechaEmision`                    | Fecha de emisión y timbrado                             |
| `CFDIGenerada.Subtotal` / `Total`              | Importes de la NC                                       |
| `CFDIGenerada.Moneda` + `TipoDeCambio`         | Moneda y tipo de cambio                                 |
| `fccNotaCredito.Estado`                        | Estado: VIGENTE / CANCELADA                             |
| `fccNotaCredito.Modalidad`                     | Modalidad: POR_PARTIDAS / MANUAL                        |
| `fccNotaCredito.Motivo`                        | Motivo de la NC                                         |
| `CFDIGeneradaRelacionado.UUID`                 | UUID de la factura origen relacionada                   |
| `fccNotaCredito.CancelarFacturaOrigen`         | Indicador de cancelación de factura origen              |
| `fccNotaCredito.ClaveMotivosCancelacion`       | Motivo de cancelación SAT (si aplica)                   |
| `Archivo.FileKey` (IdArchivoPdf / IdArchivoXml)| Rutas MinIO del PDF y XML                               |

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
  en `fccNotaCredito`, `IdCFDI` poblado en `CFDIGenerada`).
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
| `catTipoCFDI.NOTA_CREDITO`         | RE-028 T1 | Discriminador de tipo CFDI; RE-032 inserta la clave (prereq: tabla de RE-028) |
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
| `FechaRegistro` | `datetime2(7)` | NO | `SYSUTCDATETIME()` | Fecha de inserción |

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

## Pendientes

| ID | Pendiente                                                                                                 | Sección afectada              |
| -- | --------------------------------------------------------------------------------------------------------- | ----------------------------- |
| P1 | Verificar existencia de `catUsoCFDI` G02 en BD real (query SSMS)                                        | DML catUsoCFDI                |
| P2 | Verificar bucket MinIO para NCs en `RegionConfiguracionMinioBucket` (query SSMS)                        | MinIO                         |
| P3 | Validar Serie "P2" y formato de folio con PMO (Regla 9 del requisito)                                   | DML EmpresaFolio              |
| P4 | Recibir estructura de tablas PCconnect para completar mapeo ETL SSIS                                    | ETL PCconnect                 |
| P5 | Confirmar FormaPago en modalidad manual (Regla 7 — ⚠️ '99' fiscalmente incorrecto para NC PUE)          | CFDIGenerada / fccNotaCredito |
| P6 | Confirmar ClaveProdServ y ClaveUnidad default para modalidad manual (candidato SAT: 84111506 / ACT)     | CFDIGeneradaConcepto          |
| P7 | Confirmar estructura de `catTipoCFDI` con RE-028 para validar columnas del INSERT de NOTA_CREDITO       | DML catTipoCFDI               |

---

## Brechas

| ID | Brecha                                                                                                              | Impacto                              |
| -- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| B1 | `fccNotaCredito.IdTPProformaPedido` es NOT NULL en la tabla existente. Las NCs R16 no vienen de un pedido directo — requiere confirmar si se puede poner NULL o si necesita un valor placeholder | ALTER TABLE, integridad referencial |
| B2 | `fccNotaCreditoPartida.NumeroDePiezas` es `int` (no decimal). Si un producto tiene cantidad fraccionaria necesita cambio de tipo | ALTER TABLE fccNotaCreditoPartida   |
| B3 | Estructura PCconnect desconocida — el paquete SSIS no puede diseñarse hasta tener el esquema destino              | ETL SSIS                             |
| B4 | No existe `catMotivoCancelacionSAT` como tabla catálogo — `CFDICancelacion.ClaveMotivo` es varchar libre. Si se requiere validación en UI, el front debe traer el catálogo SAT c_MotivoCancelacion desde config o hardcode | Validación de negocio               |
