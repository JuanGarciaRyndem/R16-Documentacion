# Impacto en BD - Diseno y Generacion Factura Mexico (CFDI 4.0 PDF)
**Requisito:** R16A-RE-FU-021
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
PDF de la Factura CFDI 4.0 generado al timbrar exitosamente ante el PAC.
Branding por empresa emisora (GOL/MUN/PRO/PQF). Logos en repo/base64.
Datos tecnicos SAT del TimbreFiscalDigital (UUID, sellos, cadena, QR).
PDF persistido en Minio (bucket 'facturas') via tabla Archivo.
Aplica a FAA (RE-FU-019) y a Validar Cobro. Solo Region MEX.

---

## Impacto en BD

| # | Cambio | Tipo | Prioridad |
|---|--------|------|-----------|
| 1 | ALTER TABLE Producto ADD ClaveProdServSAT varchar(10) NULL | DDL | Alta |
| 2 | ALTER TABLE catUnidad ADD ClaveSAT varchar(10) NULL | DDL | Alta |
| 3 | ALTER TABLE CFDIGenerada ADD Exportacion varchar(2) NULL | DDL | Alta |
| 4 | CREATE TABLE catClaveProdServSAT | DDL | Media |

> **ClaveProdServSAT** no existe en Producto — obligatorio para concepto del CFDI 4.0.
> **catUnidad.Clave** existe pero no es la clave SAT (c_ClaveUnidad) — agregar ClaveSAT.
> **CFDIGenerada** no tiene campo Exportacion — campo obligatorio CFDI 4.0.
> El PDF se almacena via CFDI.IdArchivoXml + tabla Archivo (patron existente).

---

## Tablas Existentes Relevantes

### CFDI (ProquifaDotNet) - ya tiene los archivos

| Columna               | Tipo             | Uso en Factura                        |
| --------------------- | ---------------- | ------------------------------------- |
| IdCFDI                | uniqueidentifier | PK                                    |
| UUID                  | varchar(80)      | Folio Fiscal SAT (36 chars)           |
| FechaCertificacionSat | datetime         | Fecha/hora de certificacion PAC       |
| FechaEmision          | datetime         | Fecha/hora de emision                 |
| Serie                 | varchar(80)      | Serie del CFDI                        |
| Folio                 | varchar(80)      | Folio por empresa emisora             |
| EfectoComprobante     | varchar(80)      | 'I - Ingreso'                         |
| IdArchivoXml          | uniqueidentifier | FK -> Archivo (XML timbrado en Minio) |
| IdArchivoTimbre       | uniqueidentifier | FK -> Archivo (XML del timbre)        |
| Estatus               | varchar(30)      | Vigente / Cancelado                   |

### CFDIGenerada (ProquifaDotNet) - datos fiscales de la factura

| Columna | Tipo | Uso en Factura |
|---------|------|----------------|
| IdCFDIGenerada | uniqueidentifier | PK |
| RFCEmisor | varchar(15) | RFC de la empresa emisora |
| RazonSocialEmisor | varchar(200) | Nombre legal emisor |
| RegimenFiscal | varchar(3) | 601 General de Ley PM |
| LugarExpedicion | varchar(5) | CP lugar de expedicion |
| RFCReceptor | varchar(15) | RFC del cliente |
| RazonSocialReceptor | varchar(200) | Razon Social del cliente |
| RegimenFiscalReceptor | varchar(3) | Regimen fiscal del cliente |
| CodigoPostalReceptor | varchar(5) | CP fiscal del cliente (CFDI 4.0) |
| UsoCFDI | varchar(3) | S01, G03, etc. (catUsoCFDI) |
| FormaPago | varchar(2) | '99' - Por definir (forzado) |
| MetodoDePago | varchar(3) | 'PPD' (forzado) |
| TipoDocumento | varchar(1) | 'I' - Ingreso |
| CondicionesPago | varchar(80) | PREPAGO 100%, 30 DIAS, etc. |
| Moneda | varchar(3) | MXN, USD, etc. |
| TipoDeCambio | decimal | TC del dia de emision |
| Subtotal | decimal | Suma importes partidas |
| Total | decimal | Subtotal + IVA |
| Folio | varchar(80) | Folio consecutivo por empresa |
| Serie | varchar(80) | Serie (ej: 'A') |
| FechaEmision | datetime | Timestamp emision |
| **Exportacion** | **varchar(2) - FALTA** | **'01' No aplica (campo CFDI 4.0)** |

---

## Cambios DDL Requeridos

### 1. Exportacion en CFDIGenerada (CFDI 4.0 obligatorio)

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    ALTER TABLE dbo.CFDIGenerada
        ADD Exportacion varchar(2) NULL;
    -- Valor por defecto: '01' = No aplica para operaciones nacionales

### 2. ClaveProdServSAT en Producto

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Clave del catalogo c_ClaveProdServ del SAT (obligatorio en concepto CFDI)
    ALTER TABLE dbo.Producto
        ADD ClaveProdServSAT varchar(10) NULL;

### 3. ClaveSAT en catUnidad (c_ClaveUnidad SAT)

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- catUnidad ya tiene Clave (varchar 150) pero no es la clave SAT
    -- Se agrega ClaveSAT como campo independiente
    ALTER TABLE dbo.catUnidad
        ADD ClaveSAT varchar(10) NULL;
    -- Ejemplos: 'KGM' Kilogramo, 'H87' Pieza, 'LTR' Litro, 'E48' Unidad de servicio

### 4. catClaveProdServSAT (catálogo c_ClaveProdServ)

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

---

## Fuentes de Datos para el PDF

| Seccion PDF | Tabla Fuente | Campo | Existe? |
|-------------|-------------|-------|---------|
| Cabecera Logo/Color | Empresa | Prefijo -> DocumentBuilder | SI |
| Datos Emisor - RFC | CFDIGenerada | RFCEmisor | SI |
| Datos Emisor - RazonSocial | CFDIGenerada | RazonSocialEmisor | SI |
| Datos Emisor - RegimenFiscal | CFDIGenerada | RegimenFiscal (601) | SI |
| Datos Emisor - LugarExpedicion | CFDIGenerada | LugarExpedicion | SI |
| Datos Receptor - RFC | CFDIGenerada | RFCReceptor | SI |
| Datos Receptor - RazonSocial | CFDIGenerada | RazonSocialReceptor | SI |
| Datos Receptor - CP | CFDIGenerada | CodigoPostalReceptor | SI |
| Datos Receptor - RegimenFiscal | CFDIGenerada | RegimenFiscalReceptor | SI |
| CFDI - UUID (Folio Fiscal) | CFDI | UUID | SI |
| CFDI - Serie/Folio | CFDIGenerada + CFDI | Serie, Folio | SI |
| CFDI - Fechas | CFDI | FechaEmision, FechaCertificacionSat | SI |
| CFDI - MetodoPago/FormaPago | CFDIGenerada | MetodoDePago='PPD', FormaPago='99' | SI |
| CFDI - TipoComprobante | CFDIGenerada | TipoDocumento='I' | SI |
| CFDI - Moneda/TC | CFDIGenerada | Moneda, TipoDeCambio | SI |
| CFDI - Exportacion | CFDIGenerada | **Exportacion (FALTA - ALTER)** | NO |
| CFDI - CondPago | CFDIGenerada | CondicionesPago | SI |
| Partidas - ClaveProdServSAT | Producto | **ClaveProdServSAT (FALTA - ALTER)** | NO |
| Partidas - ClaveUnidadSAT | catUnidad | **ClaveSAT (FALTA - ALTER)** | NO |
| Partidas - Cantidad/PU | tpPartidaPedido | NumeroDePiezas, PrecioUnitario | SI |
| Partidas - Descripcion | Producto + MarcaFamilia | Catalogo+Descripcion+Marca | SI |
| Partidas - Pedimento | **No definido** | **Pendiente confirmar** | NO |
| Partidas - Lote | tpPartidaPedido | FechaCaducidadStock | Parcial |
| Totales | CFDIGenerada | Subtotal, Total | SI |
| IVA/Traslados | A calcular desde partidas | tasa 16% | Parcial |
| Datos bancarios | EmpresaDatosBancarios | Banco, Cuenta, CLABE | SI |
| REF.CLIENTE | ClienteDatosBancarios (RE-FU-006) | CodigoValidador | SI (pendiente) |
| Elementos SAT - UUID, Sellos | CFDI | IdArchivoTimbre -> XML PAC | SI |
| Elementos SAT - QR | Generado dinamicamente desde UUID+RFC | - | Logica app |
| Pie - Disclaimer | Constante fija | 'Representacion impresa CFDI 4.0' | SI |
| PDF persistido | Archivo (IdArchivoXml en CFDI) | FileKey, FileBucket='facturas' | SI |

---

## Persistencia del PDF

    Al timbrar exitosamente:
    1. PAC retorna XML timbrado con TimbreFiscalDigital
    2. INSERT Archivo (XML, FileBucket='facturas') -> CFDI.IdArchivoXml
    3. Generar PDF desde datos del CFDI + TimbreFiscalDigital
    4. INSERT Archivo (PDF, FileBucket='facturas') -> referencia separada

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

| Paso | Script | Prerequisito |
|------|--------|--------------|
| 1 | ALTER TABLE CFDIGenerada ADD Exportacion | Ninguno |
| 2 | ALTER TABLE catUnidad ADD ClaveSAT | Ninguno |
| 3 | CREATE TABLE catClaveProdServSAT | Ninguno |
| 4 | ALTER TABLE Producto ADD ClaveProdServSAT + FK | Paso 3 |
| 5 | INSERT catClaveProdServSAT (codigos SAT relevantes) | Paso 3 |
| 6 | UPDATE catUnidad SET ClaveSAT (mapeo claves SAT) | Paso 2 |

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | ClaveProdServSAT en Producto | DDL + Datos | ALTER + poblar SAT |
| 2 | ClaveSAT en catUnidad | DDL + Datos | ALTER + mapear a c_ClaveUnidad |
| 3 | Exportacion en CFDIGenerada (CFDI 4.0) | DDL | ALTER - default '01' |
| 4 | Pedimento aduanal en partidas | Negocio | Confirmar origen del dato |
| 5 | Lote del producto al facturar FAA | Negocio | FAA = antes del surtido |
| 6 | Referencia bancaria en Factura | Negocio | Misma logica Proforma o diferente |
| 7 | Almacenamiento PDF (BLOB vs snapshot) | Tecnico | Consistente con RE-FU-016 |
| 8 | Factura Anticipo estructura | Negocio | Confirmar con asesor comercial |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-019 | FAA MEX: dispara generacion de este PDF |
| R16A-RE-FU-016 | Patron persistencia Minio (mismo bucket 'facturas') |
| R16A-RE-FU-006 | REF.CLIENTE seccion bancaria |
| R16A-RE-FU-001 | EmpresaDatosBancarios.IdRegion (cuentas MEX) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
