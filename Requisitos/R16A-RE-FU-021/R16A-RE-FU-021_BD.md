# Impacto en BD - Diseno y Generacion Factura Mexico (CFDI 4.0 PDF)
**Requisito:** R16A-RE-FU-021
**Base de Datos:** ProquifaDotNet
**Version:** 2.0 - Consolidado: una sola tabla CFDIGenerada (propiedad de ProquifaDotNet.Finanzas), sin tabla CFDI separada

---

> **Nota de arquitectura:** la version previa de este documento describia una tabla `CFDI`
> (con `IdCFDI`, `UUID`, `FechaCertificacionSat`, `Serie`, `Folio`, `EfectoComprobante`,
> `IdArchivoXml`, `IdArchivoTimbre`, `Estatus`) por separado de `CFDIGenerada`, duplicando el
> mismo registro de negocio del CFDI en dos tablas. Se corrige: **existe una unica tabla,
> `CFDIGenerada`** (creada por `R16A-RE-FU-019`, extendida por `R16A-RE-FU-028` con
> `IdCatTipoCFDI`/`IdCFDIRelacionado`, y por `R16A-RE-FU-018` con las columnas tecnicas de
> timbrado — `IdCatUsoCFDI`, `IdCatMetodoDePagoCFDI`, `IdCatMoneda`, `TipoCambio`,
> `IdArchivoXml`, `Estado`, `MensajeError`, `FechaUltimaActualizacion`). `UUID`, `Serie`,
> `Folio` y `FechaEmision` ya existen desde la tabla base (RE-FU-019); no se duplican.
> `IdArchivoTimbre` no aplica (el timbre digital es parte del XML referenciado por
> `IdArchivoXml`, no un archivo separado). `Estatus`/`EfectoComprobante` corresponden a los
> campos `Estado` (RE-FU-018) y `TipoDocumento='I'` respectivamente.
>
> Este requisito (RE-FU-021) agrega a `CFDIGenerada` las columnas que faltan para completar
> el snapshot fiscal necesario para renderizar el PDF de la Factura CFDI 4.0: datos de emisor
> y receptor al momento de la emision (`RazonSocialEmisor`, `RegimenFiscal`,
> `LugarExpedicion`, `RazonSocialReceptor`, `RegimenFiscalReceptor`, `CodigoPostalReceptor`),
> `CondicionesPago`, `Subtotal`, `Exportacion` (obligatorio CFDI 4.0), y la referencia al PDF
> generado (`IdArchivoPdf`, `FechaCertificacionSat`). Estas columnas se agregan como
> snapshot porque el CFDI es un documento fiscal inmutable: debe reflejar los datos del
> emisor/receptor tal como estaban al momento del timbrado, no los datos actuales (que
> pueden cambiar despues) de `Empresa`/`Cliente`.

---

## Resumen
PDF de la Factura CFDI 4.0 generado al timbrar exitosamente ante el PAC.
Branding por empresa emisora (GOL/MUN/PRO/PQF). Logos en repo/base64.
Datos tecnicos SAT del TimbreFiscalDigital (UUID, sellos, cadena, QR) — UUID desde
`CFDIGenerada`, sellos/cadena original leidos directamente del XML del PAC (no se
persisten como columnas).
PDF persistido en Minio (bucket 'facturas') via tabla Archivo, referenciado desde
`CFDIGenerada.IdArchivoPdf`.
Aplica a FAA (RE-FU-019) y a Validar Cobro. Solo Region MEX.

---

## Impacto en BD

| #   | Cambio                                                                                                                                                                                                 | Tipo      | Prioridad | Estado      |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------- | --------- | ----------- |
| 1   | ALTER TABLE Producto ADD ClaveProdServSAT varchar(10) NULL                                                                                                                                             | DDL       | Alta      | ⏸ En espera |
| 2   | ALTER TABLE catUnidad ADD ClaveSAT varchar(10) NULL                                                                                                                                                    | DDL       | Alta      | ⏸ En espera |
| 3   | CREATE TABLE catClaveProdServSAT + INSERT catalogo c_ClaveProdServ SAT (claves relevantes PROQUIFA)                                                                                                    | DDL + DML | Media     | ⏸ En espera |
| 4   | ALTER TABLE CFDIGenerada ADD Exportacion varchar(2) NULL                                                                                                                                               | DDL       | Alta      | Pendiente   |
| 5   | ALTER TABLE CFDIGenerada ADD snapshot Emisor/Receptor (RazonSocialEmisor, RegimenFiscal, LugarExpedicion, RazonSocialReceptor, RegimenFiscalReceptor, CodigoPostalReceptor, CondicionesPago, Subtotal) | DDL       | Alta      | Pendiente   |
| 6   | ALTER TABLE CFDIGenerada ADD IdArchivoPdf uniqueidentifier NULL, FechaCertificacionSat datetime NULL (+ FK a Archivo)                                                                              | DDL       | Alta      | Pendiente   |

> **⏸ En espera (ítems 1, 2, 3):** `Producto.ClaveProdServSAT`, `catUnidad.ClaveSAT` y `catClaveProdServSAT` están pausados pendiente de definición/confirmación externa. Los ítems 4–6 sobre `CFDIGenerada` pueden ejecutarse independientemente.
> **ClaveProdServSAT** no existe en Producto — obligatorio para concepto del CFDI 4.0.
> **catUnidad.Clave** existe pero no es la clave SAT (c_ClaveUnidad) — agregar ClaveSAT.
> **CFDIGenerada** no tiene campo Exportacion — campo obligatorio CFDI 4.0.
> **CFDIGenerada** no tiene el snapshot Emisor/Receptor completo ni la referencia al PDF — se agregan en este requisito.
> El PDF se almacena via `Archivo` (patron existente) y se referencia desde `CFDIGenerada.IdArchivoPdf`.

---

## Tabla Existente Relevante

### CFDIGenerada (ProquifaDotNet) — propiedad de ProquifaDotNet.Finanzas

Registro central de negocio de todo CFDI emitido. Columnas ya existentes (creadas/extendidas
por requisitos previos) + columnas nuevas agregadas por este requisito (marcadas **NUEVO**).

| Columna | Tipo | Origen | Uso en Factura |
| --------------------- | ---------------- | ------------------ | ------------------------------------- |
| IdCFDIGenerada | uniqueidentifier | RE-FU-019 (base) | PK |
| RFCEmisor | varchar(13) | RE-FU-019 (base) | RFC de la empresa emisora |
| RFCReceptor | varchar(50) | RE-FU-019 (base) | RFC/RUC del cliente receptor |
| Serie | varchar(25) | RE-FU-019 (base) | Serie del CFDI |
| Folio | varchar(40) | RE-FU-019 (base) | Folio por empresa emisora |
| FechaEmision | datetime | RE-FU-019 (base) | Fecha/hora de emision |
| UUID | varchar(36) | RE-FU-019 (base) | Folio Fiscal SAT (36 chars) |
| Total | decimal(18,2) | RE-FU-019 (base) | Total del CFDI |
| IdCatTipoCFDI | uniqueidentifier | RE-FU-028 | FK -> catTipoCFDI |
| IdCFDIRelacionado | uniqueidentifier | RE-FU-028 | FK -> CFDIGenerada (CFDI relacionado, notas de credito) |
| IdCatUsoCFDI | uniqueidentifier | RE-FU-018 | FK -> catUsoCFDI |
| IdCatMetodoDePagoCFDI | uniqueidentifier | RE-FU-018 | FK -> catMetodoDePagoCFDI ('PPD') |
| IdCatMoneda | uniqueidentifier | RE-FU-018 | FK -> catMoneda |
| TipoCambio | decimal(18,6) | RE-FU-018 | TC del dia de emision |
| IdArchivoXml | uniqueidentifier | RE-FU-018 | FK -> Archivo (XML timbrado en Minio) |
| Estado | varchar(30) | RE-FU-018 | Pendiente/Timbrado/Error/Cancelado |
| MensajeError | varchar(max) | RE-FU-018 | Detalle de error de timbrado, si aplica |
| FechaUltimaActualizacion | datetime | RE-FU-018 | Ultima actualizacion de estatus |
| **Exportacion** | **varchar(2)** | **RE-FU-021 (NUEVO)** | **'01' No aplica (campo obligatorio CFDI 4.0)** |
| **RazonSocialEmisor** | **varchar(200)** | **RE-FU-021 (NUEVO)** | **Nombre legal del emisor (snapshot)** |
| **RegimenFiscal** | **varchar(3)** | **RE-FU-021 (NUEVO)** | **601 General de Ley PM (snapshot)** |
| **LugarExpedicion** | **varchar(5)** | **RE-FU-021 (NUEVO)** | **CP lugar de expedicion (snapshot)** |
| **RazonSocialReceptor** | **varchar(200)** | **RE-FU-021 (NUEVO)** | **Razon social del cliente (snapshot)** |
| **RegimenFiscalReceptor** | **varchar(3)** | **RE-FU-021 (NUEVO)** | **Regimen fiscal del cliente (snapshot)** |
| **CodigoPostalReceptor** | **varchar(5)** | **RE-FU-021 (NUEVO)** | **CP fiscal del cliente (CFDI 4.0, snapshot)** |
| **CondicionesPago** | **varchar(80)** | **RE-FU-021 (NUEVO)** | **PREPAGO 100%, 30 DIAS, etc.** |
| **Subtotal** | **decimal(18,2)** | **RE-FU-021 (NUEVO)** | **Suma de importes de partidas (previo a IVA)** |
| **IdArchivoPdf** | **uniqueidentifier** | **RE-FU-021 (NUEVO)** | **FK -> Archivo (PDF de la Factura en Minio)** |
| **FechaCertificacionSat** | **datetime** | **RE-FU-021 (NUEVO)** | **Fecha/hora de certificacion PAC** |

> Los sellos digitales, la cadena original y el numero de serie de certificados **no se
> persisten como columnas**: se leen directamente del `TimbreFiscalDigital` del XML
> referenciado por `IdArchivoXml` en el momento de generar el PDF.

---

## Cambios DDL Requeridos

### 1. Exportacion en CFDIGenerada (CFDI 4.0 obligatorio)

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    ALTER TABLE dbo.CFDIGenerada
        ADD Exportacion varchar(2) NULL;
    -- Valor por defecto: '01' = No aplica para operaciones nacionales

### 2. Snapshot Emisor/Receptor + CondicionesPago + Subtotal en CFDIGenerada

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Snapshot fiscal al momento de la emision (el CFDI es documento inmutable;
    -- no debe depender de datos actuales de Empresa/Cliente que puedan cambiar despues)
    ALTER TABLE dbo.CFDIGenerada
        ADD RazonSocialEmisor      varchar(200)   NULL,
            RegimenFiscal          varchar(3)     NULL,
            LugarExpedicion        varchar(5)     NULL,
            RazonSocialReceptor    varchar(200)   NULL,
            RegimenFiscalReceptor  varchar(3)     NULL,
            CodigoPostalReceptor   varchar(5)     NULL,
            CondicionesPago        varchar(80)    NULL,
            Subtotal               decimal(18,2)  NULL;
    GO

### 3. IdArchivoPdf + FechaCertificacionSat en CFDIGenerada

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    ALTER TABLE dbo.CFDIGenerada
        ADD IdArchivoPdf uniqueidentifier NULL,
            FechaCertificacionSat datetime NULL;
    GO
    ALTER TABLE dbo.CFDIGenerada
        ADD CONSTRAINT [FK_CFDIGenerada_ArchivoPdf]
            FOREIGN KEY (IdArchivoPdf) REFERENCES dbo.Archivo([IdArchivo]);
    GO

> Ver tambien `R16A-RE-FU-021-Back.md`, Parte "5. ALTER TABLE CFDIGenerada (IdArchivoPdf +
> FechaCertificacionSat)" — mismo script, documentado alli como GAP-15.

### 4. ClaveProdServSAT en Producto

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Clave del catalogo c_ClaveProdServ del SAT (obligatorio en concepto CFDI)
    ALTER TABLE dbo.Producto
        ADD ClaveProdServSAT varchar(10) NULL;

### 5. ClaveSAT en catUnidad (c_ClaveUnidad SAT)

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- catUnidad ya tiene Clave (varchar 150) pero no es la clave SAT
    -- Se agrega ClaveSAT como campo independiente
    ALTER TABLE dbo.catUnidad
        ADD ClaveSAT varchar(10) NULL;
    -- Ejemplos: 'KGM' Kilogramo, 'H87' Pieza, 'LTR' Litro, 'E48' Unidad de servicio

### 6. catClaveProdServSAT (catálogo c_ClaveProdServ)

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    CREATE TABLE [dbo].[catClaveProdServSAT](
        [IdCatClaveProdServSAT] uniqueidentifier NOT NULL
            CONSTRAINT [DF_catClaveProdServSAT_Id] DEFAULT (NEWID()),
        [Clave] varchar(10) NOT NULL,
        [Descripcion] varchar(300) NOT NULL,
        [Activo] bit NOT NULL
            CONSTRAINT [DF_catClaveProdServSAT_Activo] DEFAULT (1),
        CONSTRAINT [PK_catClaveProdServSAT] PRIMARY KEY CLUSTERED ([IdCatClaveProdServSAT]),
        CONSTRAINT [UQ_catClaveProdServSAT_Clave] UNIQUE ([Clave])
    );
    GO

    ALTER TABLE dbo.Producto
        ADD IdCatClaveProdServSAT uniqueidentifier NULL
            CONSTRAINT [FK_Producto_ClaveProdServSAT]
                FOREIGN KEY REFERENCES dbo.catClaveProdServSAT([IdCatClaveProdServSAT]);

### 7. INSERT catClaveProdServSAT (catalogo oficial SAT c_ClaveProdServ — Anexo 20 CFDI 4.0)

> Subconjunto del catalogo oficial `c_ClaveProdServ` del SAT, curado para el giro de PROQUIFA
> (distribucion de materias primas quimicas y productos farmaceuticos). Cubre: elementos y
> gases industriales, compuestos/mezclas quimicas de uso comun, y principios activos/insumos
> farmaceuticos frecuentes. **No es el catalogo completo** (el catalogo oficial supera 50,000
> claves) — se recomienda complementar bajo demanda usando el buscador oficial del SAT
> (http://pys.sat.gob.mx/PyS/catPyS.aspx) conforme se identifiquen productos sin clave asignada.
> Se incluye ademas la clave generica `01010101` para productos sin clasificacion especifica
> disponible (uso excepcional, no recomendado como practica habitual).

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Fuente: catalogo c_ClaveProdServ del SAT, Anexo 20 CFDI 4.0
    INSERT INTO [dbo].[catClaveProdServSAT] ([Clave], [Descripcion])
    VALUES
        -- Generico (uso excepcional)
        ('01010101', 'No existe en el catalogo'),

        -- Division 12 - Elementos y gases industriales (grupo 1214)
        ('12141806', 'Sodio Na'),
        ('12141804', 'Potasio K'),
        ('12141901', 'Cloro Cl'),
        ('12141903', 'Nitrogeno N'),
        ('12141904', 'Oxigeno O'),
        ('12141916', 'Yodo I'),
        ('12141909', 'Fosforo P'),
        ('12141912', 'Azufre S'),
        ('12141902', 'Hidrogeno H'),
        ('12142005', 'Gas helio He'),
        ('12142104', 'Gas dioxido de carbono (Anhidrido carbonico)'),
        ('12142109', 'Hielo seco'),
        ('12142105', 'Aire industrial'),

        -- Division 12 - Compuestos y mezclas quimicas (grupo 1235)
        ('12352104', 'Alcoholes o sus sustitutos (Alcohol etilico, Etanol, Alcohol absoluto, Alcohol desnaturalizado, Alcohol Isopropilico, 2-propanol, Alcohol metilico, Metanol)'),
        ('12352316', 'Hidroxido de sodio'),
        ('12352320', 'Hidroxido de potasio (Potasa caustica en escama)'),
        ('12352319', 'Hidroxido de calcio'),
        ('12352301', 'Acidos inorganicos'),
        ('12352106', 'Acidos organicos o sus sustitutos'),
        ('12352302', 'Sales metalicas inorganicas'),
        ('12352107', 'Sales organicas o sus sustitutos'),
        ('12352137', 'Glicol trietileno'),
        ('12352135', 'Glicol propileno'),
        ('12352209', 'Aminoacidos o sus derivados'),
        ('12352204', 'Enzimas'),
        ('12352202', 'Proteinas'),
        ('12352201', 'Carbohidratos o sus derivados'),
        ('12352211', 'Grasas o lipidos'),
        ('12352205', 'Nutrientes'),
        ('12352400', 'Mezclas'),
        ('12352401', 'Mezclas quimicas organicas'),
        ('12352402', 'Mezclas quimicas inorganicas'),
        ('12352309', 'Silice'),
        ('12352308', 'Silicatos'),
        ('12352310', 'Siliconas'),
        ('12352200', 'Productos bioquimicos'),

        -- Division 51 - Medicamentos y productos farmaceuticos (grupos 5110, 5119, 5114, 5116, etc.)
        ('51101500', 'Antibioticos'),
        ('51101511', 'Amoxicilina'),
        ('51101550', 'Cefalexina'),
        ('51101570', 'Eritromicina'),
        ('51102700', 'Antisepticos (Antimicrobianos)'),
        ('51102709', 'Peroxido de hidrogeno antiseptico (Agua oxigenada)'),
        ('51102713', 'Povidona yodada (Polividona yodada, Povidona, Yodopolivinilpirrolidona)'),
        ('51191900', 'Suplementos dieteticos y productos de terapia alimenticia (Suplementos alimenticios o nutricionales)'),
        ('51191905', 'Suplementos vitaminicos (Vitaminas y minerales)'),
        ('51142100', 'Farmacos antiinflamatorios no esteroideos (NSAID)'),
        ('51161700', 'Medicamentos para alteraciones del tracto respiratorio'),
        ('51102300', 'Medicamentos antivirales');
    GO

> **Nota de mantenimiento:** este catalogo debe revisarse y ampliarse conforme el area
> comercial identifique productos de PROQUIFA sin clave SAT asignada. La clave se determina
> por la naturaleza del producto facturado (responsabilidad del emisor del CFDI), no por el
> nombre comercial — un mismo producto quimico puede requerir distinta clave segun su
> presentacion, pureza o uso declarado.

---

## Fuentes de Datos para el PDF

| Seccion PDF | Tabla Fuente | Campo | Existe? |
|-------------|-------------|-------|---------|
| Cabecera Logo/Color | Empresa | Prefijo -> DocumentBuilder | SI |
| Datos Emisor - RFC | CFDIGenerada | RFCEmisor | SI |
| Datos Emisor - RazonSocial | CFDIGenerada | **RazonSocialEmisor (FALTA - ALTER)** | NO |
| Datos Emisor - RegimenFiscal | CFDIGenerada | **RegimenFiscal (FALTA - ALTER)** | NO |
| Datos Emisor - LugarExpedicion | CFDIGenerada | **LugarExpedicion (FALTA - ALTER)** | NO |
| Datos Receptor - RFC | CFDIGenerada | RFCReceptor | SI |
| Datos Receptor - RazonSocial | CFDIGenerada | **RazonSocialReceptor (FALTA - ALTER)** | NO |
| Datos Receptor - CP | CFDIGenerada | **CodigoPostalReceptor (FALTA - ALTER)** | NO |
| Datos Receptor - RegimenFiscal | CFDIGenerada | **RegimenFiscalReceptor (FALTA - ALTER)** | NO |
| CFDI - UUID (Folio Fiscal) | CFDIGenerada | UUID | SI |
| CFDI - Serie/Folio | CFDIGenerada | Serie, Folio | SI |
| CFDI - Fechas | CFDIGenerada | FechaEmision, **FechaCertificacionSat (FALTA - ALTER)** | Parcial |
| CFDI - MetodoPago/FormaPago | CFDIGenerada | IdCatMetodoDePagoCFDI ('PPD') | SI |
| CFDI - TipoComprobante | CFDIGenerada | Constante 'I' (Ingreso) | SI |
| CFDI - Moneda/TC | CFDIGenerada | IdCatMoneda, TipoCambio | SI |
| CFDI - Exportacion | CFDIGenerada | **Exportacion (FALTA - ALTER)** | NO |
| CFDI - CondPago | CFDIGenerada | **CondicionesPago (FALTA - ALTER)** | NO |
| Partidas - ClaveProdServSAT | Producto | **ClaveProdServSAT (FALTA - ALTER)** | NO |
| Partidas - ClaveUnidadSAT | catUnidad | **ClaveSAT (FALTA - ALTER)** | NO |
| Partidas - Cantidad/PU | tpPartidaPedido | NumeroDePiezas, PrecioUnitario | SI |
| Partidas - Descripcion | Producto + MarcaFamilia | Catalogo+Descripcion+Marca | SI |
| Partidas - Pedimento | **No definido** | **Pendiente confirmar** | NO |
| Partidas - Lote | tpPartidaPedido | FechaCaducidadStock | Parcial |
| Totales | CFDIGenerada | **Subtotal (FALTA - ALTER)**, Total | Parcial |
| IVA/Traslados | A calcular desde partidas | tasa 16% | Parcial |
| Datos bancarios | EmpresaDatosBancarios | Banco, Cuenta, CLABE | SI |
| REF.CLIENTE | ClienteDatosBancarios (RE-FU-006) | CodigoValidador | SI (pendiente) |
| Elementos SAT - UUID, Sellos, Cadena Original | XML del PAC (via CFDIGenerada.IdArchivoXml) | TimbreFiscalDigital — parseo directo, no columnas | SI |
| Elementos SAT - QR | Generado dinamicamente desde UUID+RFC | - | Logica app |
| Pie - Disclaimer | Constante fija | 'Representacion impresa CFDI 4.0' | SI |
| PDF persistido | Archivo, referenciado desde CFDIGenerada.IdArchivoPdf | FileKey, FileBucket='facturas' | **FALTA - ALTER** |

---

## Persistencia del PDF

    Al timbrar exitosamente (orquestado por ProquifaDotNet.Finanzas — CfdiService):
    1. PAC retorna XML timbrado con TimbreFiscalDigital
    2. INSERT Archivo (XML, FileBucket='facturas') -> UPDATE CFDIGenerada SET IdArchivoXml
    3. Generar PDF desde datos de CFDIGenerada + TimbreFiscalDigital (parseado del XML)
    4. INSERT Archivo (PDF, FileBucket='facturas') -> UPDATE CFDIGenerada SET IdArchivoPdf

    Al consultar historico:
    -> Descarga PDF desde Minio via Archivo.FileKey (sin regenerar)

---

## Datos Pendientes de Clarificar (Partidas)

| Dato Partida | Estado | Accion |
|-------------|--------|--------|
| ClaveProdServSAT (c_ClaveProdServ) | NO existe en BD | ALTER + poblar catalogo |
| ClaveSAT de unidad (c_ClaveUnidad) | NO existe en catUnidad | ALTER catUnidad |
| Pedimento aduanal | Sin definir origen | Confirmar con cliente |
| Lote del producto | tpPartidaPedido.FechaCaducidadStock (parcial) | Confirmar campo |
| Desglose IVA por linea | Calcular (16% sobre importe) | Logica aplicacion |

---

## Orden de Ejecucion de Scripts

| Paso | Script                                                                             | Prerequisito                          |
| ---- | ---------------------------------------------------------------------------------- | ------------------------------------- |
| 1    | ALTER TABLE CFDIGenerada ADD Exportacion                                           | CFDIGenerada debe existir (RE-FU-019) |
| 2    | ALTER TABLE CFDIGenerada ADD snapshot Emisor/Receptor + CondicionesPago + Subtotal | CFDIGenerada debe existir (RE-FU-019) |
| 3    | ALTER TABLE CFDIGenerada ADD IdArchivoPdf + FechaCertificacionSat (+ FK a Archivo) | CFDIGenerada debe existir (RE-FU-019) |
| 4    | ALTER TABLE catUnidad ADD ClaveSAT                                                 | Ninguno                               |
| 5    | CREATE TABLE catClaveProdServSAT                                                   | Ninguno                               |
| 6    | ALTER TABLE Producto ADD ClaveProdServSAT + FK                                     | Paso 5                                |
| 7    | INSERT catClaveProdServSAT (codigos SAT relevantes)                                | Paso 5                                |
| 8    | UPDATE catUnidad SET ClaveSAT (mapeo claves SAT)                                   | Paso 4                                |

---

## Gaps y Pendientes

| #   | Gap                                                                   | Tipo        | Accion                              | Estado      |
| --- | --------------------------------------------------------------------- | ----------- | ----------------------------------- | ----------- |
| 1   | ClaveProdServSAT en Producto                                          | DDL + Datos | ALTER + poblar SAT                  | ⏸ En espera |
| 2   | ClaveSAT en catUnidad                                                 | DDL + Datos | ALTER + mapear a c_ClaveUnidad      | ⏸ En espera |
| 3   | catClaveProdServSAT CREATE + INSERT catálogo                          | DDL + DML   | CREATE TABLE + INSERT claves SAT    | ⏸ En espera |
| 4   | Exportacion en CFDIGenerada (CFDI 4.0)                                | DDL         | ALTER - default '01'                | Pendiente   |
| 5   | Snapshot Emisor/Receptor + CondicionesPago + Subtotal en CFDIGenerada | DDL         | ALTER (ver script 2)                | Pendiente   |
| 6   | IdArchivoPdf + FechaCertificacionSat en CFDIGenerada                  | DDL         | ALTER + FK a Archivo (ver script 3) | Pendiente   |
| 7   | Pedimento aduanal en partidas                                         | Negocio     | Confirmar origen del dato           | Pendiente   |
| 8   | Lote del producto al facturar FAA                                     | Negocio     | FAA = antes del surtido             | Pendiente   |
| 9   | Referencia bancaria en Factura                                        | Negocio     | Misma logica Proforma o diferente   | Pendiente   |
| 10  | Almacenamiento PDF (BLOB vs snapshot)                                 | Tecnico     | Consistente con RE-FU-016           | Pendiente   |
| 11  | Factura Anticipo estructura                                           | Negocio     | Confirmar con asesor comercial      | Pendiente   |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-019 | CREATE TABLE CFDIGenerada (columnas base) |
| R16A-RE-FU-028 | ALTER CFDIGenerada — IdCatTipoCFDI, IdCFDIRelacionado |
| R16A-RE-FU-018 | ALTER CFDIGenerada — columnas tecnicas de timbrado (IdCatUsoCFDI, IdCatMetodoDePagoCFDI, IdCatMoneda, TipoCambio, IdArchivoXml, Estado, MensajeError, FechaUltimaActualizacion) |
| R16A-RE-FU-016 | Patron persistencia Minio (mismo bucket 'facturas') |
| R16A-RE-FU-006 | REF.CLIENTE seccion bancaria |
| R16A-RE-FU-001 | EmpresaDatosBancarios.IdRegion (cuentas MEX) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
