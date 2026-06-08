# Impacto en BD - Diseno y Generacion Factura Peru (CPE UBL 2.1 PDF)
**Requisito:** TPSC-RE-FU-022
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
PDF del CPE tipo 01 (Factura) generado al timbrar ante SUNAT para clientes Peru.
Branding de Golocaer S.A.C. Logo en repo/base64. IGV 18%, RUC, CCI.
PDF persistido en Minio (bucket 'facturas', IdRegion=PER) via tabla Archivo.
Se requiere tabla nueva CPEGenerada: CFDIGenerada tiene campos SAT que no aplican a Peru.

---

## Decision: CPEGenerada vs reutilizar CFDIGenerada

| Aspecto | CFDIGenerada (existente) | CPEGenerada (nueva) |
|---------|--------------------------|---------------------|
| FormaPago / MetodoPago | Campos SAT (PPD/99) | **No aplica en Peru** |
| UsoCFDI | Campo SAT | **No aplica en Peru** |
| RegimenFiscal / CP Receptor | Campos SAT | Equivalentes SUNAT distintos |
| TipoOperacion (cat.51 SUNAT) | No existe | **Obligatorio en Peru** |
| Totales SUNAT (ISC, ICBPER, etc.) | No existen | **Necesarios en Peru** |
| RUC (11 digitos) vs RFC (13 chars) | varchar(15) - cabe | varchar(11) - mas preciso |

> **Recomendacion: CREATE TABLE CPEGenerada** — los campos son demasiado distintos.
> Reutilizar CFDIGenerada implicaria NULLs masivos y confusion en ORM/scaffold.

---

## Impacto en BD

| #   | Cambio                                                           | Tipo                   | Prioridad      |     |
| --- | ---------------------------------------------------------------- | ---------------------- | -------------- | --- |
| 1   | CREATE TABLE CPEGenerada                                         | DDL                    | Alta           |     |
| 2   | ALTER TABLE Producto ADD CodigoSUNAT (ya propuesto en RE-FU-020) | DDL                    | **BLOQUEANTE** |     |
| 3   | ALTER TABLE catUnidad ADD ClaveSUNAT (ya propuesto en RE-FU-020) | DDL                    | **BLOQUEANTE** |     |
| 4   | CREATE TABLE catAfectacionIGV (ya propuesto en RE-FU-020)        | DDL                    | **BLOQUEANTE** |     |
| 5   | Reutiliza: Archivo (bucket 'facturas', IdRegion=PER)             | Existente              | -              |     |
| 6   | Reutiliza: EmpresaFolio GOLPERU (RE-FU-020)                      | ProquifaDotNetTimbrado | Existente      | -   |

> Los cambios 2-4 son compartidos con RE-FU-020 — solo se ejecutan una vez.

---

## CREATE TABLE CPEGenerada

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    CREATE TABLE [dbo].[CPEGenerada](
        [IdCPEGenerada]       uniqueidentifier NOT NULL CONSTRAINT [DF_CPEGenerada_Id]          DEFAULT (NEWID()),
        [RUCEmisor]           varchar(11)      NOT NULL,
        [RazonSocialEmisor]   varchar(200)     NOT NULL,
        [DireccionEmisor]     varchar(300)     NULL,
        [UbigeoEmisor]        varchar(6)       NULL,
        [RUCReceptor]         varchar(11)      NOT NULL,
        [RazonSocialReceptor] varchar(200)     NOT NULL,
        [DireccionReceptor]   varchar(300)     NULL,
        [TipoComprobante]     varchar(2)       NOT NULL CONSTRAINT [DF_CPEGenerada_TipoCPE] DEFAULT ('01'),
        [TipoOperacion]       varchar(4)       NULL,    -- catalogo 51 SUNAT (ej: '0101')
        [Serie]               varchar(4)       NOT NULL, -- ej: 'F001' o 'E001'
        [Correlativo]         varchar(8)       NOT NULL, -- hasta 8 digitos
        [CondicionPago]       varchar(50)      NULL,    -- Contado / Credito
        [Moneda]              varchar(3)       NOT NULL, -- PEN, USD
        [TipoCambio]          decimal(18,6)    NULL,
        [ValorVenta]          decimal(18,2)    NOT NULL,
        [IGV]                 decimal(18,2)    NOT NULL,
        [ISC]                 decimal(18,2)    NOT NULL CONSTRAINT [DF_CPEGenerada_ISC]     DEFAULT (0),
        [ICBPER]              decimal(18,2)    NOT NULL CONSTRAINT [DF_CPEGenerada_ICBPER]  DEFAULT (0),
        [OtrosTributos]       decimal(18,2)    NOT NULL CONSTRAINT [DF_CPEGenerada_Otros]   DEFAULT (0),
        [Total]               decimal(18,2)    NOT NULL,
        [Observaciones]       varchar(500)     NULL,
        [FechaEmision]        datetime2(7)     NOT NULL,
        [Activo]              bit              NOT NULL CONSTRAINT [DF_CPEGenerada_Activo]   DEFAULT (1),
        [FechaRegistro]       datetime2(7)     NOT NULL CONSTRAINT [DF_CPEGenerada_FechaReg] DEFAULT (SYSUTCDATETIME()),
        [FechaUltimaActualizacion] datetime2(7) NOT NULL CONSTRAINT [DF_CPEGenerada_FechaUpd] DEFAULT (SYSUTCDATETIME()),
        CONSTRAINT [PK_CPEGenerada] PRIMARY KEY CLUSTERED ([IdCPEGenerada])
    );

---

## Columnas de CPEGenerada

| Columna | Tipo | Descripcion | Equivalente en CFDIGenerada |
|---------|------|-------------|----------------------------|
| RUCEmisor | varchar(11) | RUC de Golocaer SAC | RFCEmisor |
| RazonSocialEmisor | varchar(200) | Golocaer S.A.C. | RazonSocialEmisor |
| DireccionEmisor | varchar(300) | Direccion + Ubigeo | LugarExpedicion (CP) |
| UbigeoEmisor | varchar(6) | Ubigeo SUNAT | No existe |
| RUCReceptor | varchar(11) | RUC del cliente Peru | RFCReceptor |
| RazonSocialReceptor | varchar(200) | Razon social cliente | RazonSocialReceptor |
| TipoComprobante | varchar(2) | '01' = Factura | TipoDocumento |
| TipoOperacion | varchar(4) | Cat.51 SUNAT: '0101' | No existe (UsoCFDI es SAT) |
| Serie | varchar(4) | ej: F001, E001 | Serie |
| Correlativo | varchar(8) | hasta 8 digitos | Folio |
| CondicionPago | varchar(50) | Contado / Credito | CondicionesPago |
| Moneda | varchar(3) | PEN, USD | Moneda |
| TipoCambio | decimal(18,6) | TC SUNAT del dia | TipoDeCambio |
| ValorVenta | decimal(18,2) | Subtotal sin IGV | Subtotal |
| IGV | decimal(18,2) | 18% calculado | No existe (era solo Total) |
| ISC | decimal(18,2) | Default 0 | No existe |
| ICBPER | decimal(18,2) | Default 0 | No existe |
| OtrosTributos | decimal(18,2) | Default 0 | No existe |
| Total | decimal(18,2) | ValorVenta + IGV + otros | Total |

---

## Vinculacion CPEGenerada con tpProformaAdelanto

    tpProformaAdelanto
        IdCFDIGenerada -> CFDIGenerada (MEX) ya existe como FK
        ** Para PER: agregar IdCPEGenerada -> CPEGenerada **

**Script adicional:**

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    ALTER TABLE dbo.tpProformaAdelanto
        ADD IdCPEGenerada uniqueidentifier NULL
            CONSTRAINT [FK_tpProformaAdelanto_CPEGenerada]
                FOREIGN KEY REFERENCES dbo.CPEGenerada([IdCPEGenerada]);

---

## Diferencias vs Factura Mexico (RE-FU-021)

| Aspecto                 | Mexico (CFDI 4.0)        | Peru (CPE UBL 2.1)               |
| ----------------------- | ------------------------ | -------------------------------- |
| Tabla de datos fiscales | CFDIGenerada             | **CPEGenerada (nueva)**          |
| ID fiscal emisor        | RFC (13 chars)           | RUC (11 digits)                  |
| ID fiscal receptor      | RFC                      | RUC                              |
| Impuesto                | IVA 16%                  | IGV 18%                          |
| Tipo operacion          | UsoCFDI SAT              | TipoOperacion cat.51 SUNAT       |
| Metodo/Forma Pago       | PPD + 99                 | **NO existen**                   |
| Complemento de Pago     | SI (modulo VC)           | **NO existe**                    |
| Elementos tecnicos      | UUID, sellos, cadena SAT | QR, firma digital, resumen SUNAT |
| Totales extra           | No                       | ISC, ICBPER, OtrosTributos       |
| Folio fiscal            | UUID 36 chars (SAT)      | No hay UUID (es CPE)             |
| Serie                   | varchar(80)              | 4 chars alfanumerica             |
| Correlativo             | varchar(6) numerico      | 8 digits                         |

---

## Fuentes de Datos para el PDF Peru

| Seccion PDF | Tabla Fuente | Campo | Existe? |
|-------------|-------------|-------|---------|
| Branding Emisor | Empresa (GOLPERU) | Prefijo -> DocumentBuilder | SI |
| Datos Emisor - RUC | CPEGenerada | RUCEmisor | NUEVA |
| Datos Emisor - RazonSocial | CPEGenerada | RazonSocialEmisor | NUEVA |
| Datos Emisor - Direccion | CPEGenerada | DireccionEmisor | NUEVA |
| Datos Receptor - RUC | CPEGenerada | RUCReceptor | NUEVA |
| Serie + Correlativo | CPEGenerada | Serie, Correlativo | NUEVA |
| TipoOperacion | CPEGenerada | TipoOperacion (cat.51) | NUEVA |
| Moneda/TC | CPEGenerada | Moneda, TipoCambio | NUEVA |
| Totales SUNAT | CPEGenerada | ValorVenta, IGV, ISC, ICBPER, Total | NUEVA |
| Partidas - CodigoSUNAT | Producto.CodigoSUNAT | BRECHA | NO |
| Partidas - ClaveSUNAT UdM | catUnidad.ClaveSUNAT | BRECHA | NO |
| Partidas - AfectacionIGV | catAfectacionIGV | BRECHA | NO |
| QR/Firma SUNAT | CFDI.IdArchivoXml | XML del timbrado SUNAT | SI (patron) |
| PDF persistido | Archivo (FileBucket='facturas', IdRegion=PER) | FileKey | SI |

---

## Persistencia del PDF

    Al timbrar exitosamente ante SUNAT/OSE:
    1. OSE retorna CDR de aceptacion + XML firmado
    2. INSERT CPEGenerada (datos fiscales del CPE)
    3. INSERT Archivo (XML, FileBucket='facturas', IdRegion=PER)
    4. Generar PDF desde CPEGenerada + datos fiscales producto
    5. INSERT Archivo (PDF, FileBucket='facturas', IdRegion=PER)
    6. UPDATE tpProformaAdelanto SET IdCPEGenerada = @id

---

## Brechas Criticas

| # | Brecha | Bloqueante | Accion |
|---|--------|-----------|--------|
| B1 | Datos fiscales SUNAT del producto | **SI** | RE-FU-020: ALTER Producto + catAfectacionIGV |
| B2 | OSE/PSE SUNAT no definido | **SI** | RE-FU-020: AppSetting + proveedor |
| B3 | RUC/ubigeo/certificado Golocaer SAC | SI | Recopilar y UPDATE Empresa |
| B4 | Branding/disclaimer legal Peru | SI | Confirmar con Golocaer SAC |
| B5 | Cuentas bancarias Peru en PDF | No | Confirmar si aplica |

---

## Orden de Ejecucion de Scripts

| Paso | Script | Prerequisito |
|------|--------|--------------|
| 1 | CREATE TABLE CPEGenerada | Ninguno |
| 2 | ALTER tpProformaAdelanto ADD IdCPEGenerada | Paso 1 |
| 3 | Pasos RE-FU-020: CodigoSUNAT + catAfectacionIGV | RE-FU-020 |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| TPSC-RE-FU-020 | Dispara generacion de este PDF (timbrado SUNAT) |
| TPSC-RE-FU-021 | PDF Factura MEX (patron identico, tabla diferente) |
| TPSC-RE-FU-005 | Brechas Peru (B1 bloqueante productos SUNAT) |
| TPSC-RE-FU-017 | Patron Minio PER (bucket 'facturas' region PER) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
