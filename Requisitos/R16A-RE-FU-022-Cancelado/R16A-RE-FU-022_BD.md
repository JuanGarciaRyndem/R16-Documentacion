# Impacto en BD - Diseno y Generacion Factura Peru (CPE UBL 2.1 PDF)
**Requisito:** R16A-RE-FU-022
**Base de Datos:** ProquifaDotNet
**Version:** 2.0 - Consolidado: se elimina la tabla CPEGenerada duplicada, se reutiliza CFDIGenerada (propiedad de ProquifaDotNet.Finanzas)

---

> **Nota de arquitectura:** la version previa de este documento proponia **crear una tabla
> nueva `CPEGenerada`**, argumentando que los campos SUNAT eran "demasiado distintos" de
> `CFDIGenerada`. Esto contradice la decision ya tomada en `R16A-RE-FU-020_BD.md`
> (seccion "Reutilizacion de RE-FU-018/019"): *"`CFDIGenerada` (tabla) — Misma tabla que MEX,
> distinto XML/CDR SUNAT — propiedad de Finanzas"*. RE-FU-020 ya factura Peru (timbrado FAA)
> escribiendo en `CFDIGenerada`, no en una tabla separada. Crear `CPEGenerada` aqui hubiera
> duplicado el registro de negocio del CPE en dos tablas para el mismo pedido.
>
> Se corrige: **no se crea `CPEGenerada`**. Este requisito **extiende `CFDIGenerada`** con las
> columnas especificas de SUNAT que no existen todavia (`UbigeoEmisor`, `DireccionEmisor`,
> `DireccionReceptor`, `TipoOperacion`, `ISC`, `ICBPER`, `OtrosTributos`), y **reutiliza** las
> columnas ya existentes (creadas por RE-FU-019/018/028/021) para los campos equivalentes:
> `RFCEmisor`/`RFCReceptor` (RUC cabe en los mismos `varchar`), `RazonSocialEmisor`/
> `RazonSocialReceptor` (RE-FU-021), `Serie`/`Folio` (Folio almacena el Correlativo SUNAT),
> `FechaEmision`, `Total`, `Subtotal` (RE-FU-021, equivalente a ValorVenta), `IdCatMoneda`,
> `TipoCambio`, `CondicionesPago` (RE-FU-021, equivalente a CondicionPago), `Estado`,
> `IdArchivoXml`, `IdArchivoPdf`, `FechaCertificacionSat` (RE-FU-021/018).
> El `TipoComprobante` ('01' = Factura) no requiere columna: este requisito alcanza
> unicamente CPE tipo 01, se maneja como constante en la capa de aplicacion (igual que
> `TipoDocumento='I'` en Mexico, ver `R16A-RE-FU-021_BD.md`).
>
> `fccFactura` (RE-FU-015, antes `tpProformaAdelanto`) **no requiere un FK nuevo**: reutiliza
> `IdCFDIGenerada` (columna movida a RE-FU-015 el 06/07/2026, originalmente agregada a
> `tpProformaAdelanto` por RE-FU-019), igual que Mexico. No se agrega `IdCPEGenerada`.

---

## Resumen
PDF del CPE tipo 01 (Factura) generado al timbrar ante SUNAT para clientes Peru.
Branding de Golocaer S.A.C. Logo en repo/base64. IGV 18%, RUC.
PDF persistido en Minio (bucket 'facturas', IdRegion=PER) via tabla Archivo, referenciado
desde `CFDIGenerada.IdArchivoPdf` (misma tabla y mecanismo que Mexico, RE-FU-021).
Datos fiscales del CPE en `CFDIGenerada` (misma tabla que Mexico — sin tabla nueva).

---

## Impacto en BD

| # | Cambio | Tipo | Prioridad |
|---|--------|------|-----------|
| 1 | ALTER TABLE CFDIGenerada ADD UbigeoEmisor, DireccionEmisor, DireccionReceptor, TipoOperacion, ISC, ICBPER, OtrosTributos | DDL | Alta |
| 2 | ALTER TABLE Producto ADD CodigoSUNAT (ya propuesto en RE-FU-020) | DDL | **BLOQUEANTE** |
| 3 | ALTER TABLE catUnidad ADD ClaveSUNAT (ya propuesto en RE-FU-020) | DDL | **BLOQUEANTE** |
| 4 | CREATE TABLE catAfectacionIGV (ya propuesto en RE-FU-020) | DDL | **BLOQUEANTE** |
| 5 | Reutiliza: Archivo (bucket 'facturas', IdRegion=PER) | Existente | - |
| 6 | Reutiliza: EmpresaFolio GOLPERU (RE-FU-020) | ProquifaDotNet (Finanzas) | Existente |
| 7 | Reutiliza: CFDIGenerada (RE-FU-019 base + RE-FU-018 + RE-FU-028 + RE-FU-021) | Existente | - |
| 8 | Reutiliza: fccFactura.IdCFDIGenerada (RE-FU-015, antes tpProformaAdelanto.IdCFDIGenerada de RE-FU-019) | Existente | - |

> Los cambios 2-4 son compartidos con RE-FU-020 — solo se ejecutan una vez.

---

## ALTER TABLE CFDIGenerada — columnas especificas SUNAT

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
-- Ejecutar sobre ProquifaDotNet. Prerequisito: CFDIGenerada debe existir (RE-FU-019)
ALTER TABLE dbo.CFDIGenerada
    ADD UbigeoEmisor      varchar(6)     NULL, -- Ubigeo SUNAT del emisor (6 digitos)
        DireccionEmisor   varchar(300)   NULL, -- Direccion fiscal completa del emisor (Peru)
        DireccionReceptor varchar(300)   NULL, -- Direccion fiscal completa del receptor (Peru)
        TipoOperacion     varchar(4)     NULL, -- Catalogo 51 SUNAT (ej: '0101')
        ISC               decimal(18,2)  NOT NULL
            CONSTRAINT [DF_CFDIGenerada_ISC]     DEFAULT (0),
        ICBPER            decimal(18,2)  NOT NULL
            CONSTRAINT [DF_CFDIGenerada_ICBPER]  DEFAULT (0),
        OtrosTributos     decimal(18,2)  NOT NULL
            CONSTRAINT [DF_CFDIGenerada_Otros]   DEFAULT (0);
GO
```

> Los defaults en `0` para `ISC`/`ICBPER`/`OtrosTributos` evitan que los registros de Mexico
> (donde estas columnas no aplican) requieran valores explicitos.

---

## Columnas de CFDIGenerada usadas para el CPE Peru

| Columna | Origen | Uso en Peru | Equivalente/uso en Mexico (RE-FU-021) |
|---------|--------|--------------|----------------------------------------|
| RFCEmisor | RE-FU-019 (base) | RUC de Golocaer S.A.C. (11 digitos, cabe en varchar(13)) | RFC (13 chars) |
| RFCReceptor | RE-FU-019 (base) | RUC del cliente Peru | RFC del cliente |
| RazonSocialEmisor | RE-FU-021 | "GOLOCAER S.A.C." | Razon social emisor MEX |
| RazonSocialReceptor | RE-FU-021 | Razon social del cliente Peru | Razon social receptor MEX |
| Serie | RE-FU-019 (base) | Serie SUNAT (ej: 'F001', 'E001') | Serie CFDI |
| Folio | RE-FU-019 (base) | Correlativo SUNAT (hasta 8 digitos) | Folio CFDI |
| FechaEmision | RE-FU-019 (base) | Fecha/hora de emision del CPE | Fecha de emision CFDI |
| Total | RE-FU-019 (base) | Importe Total del CPE | Total CFDI |
| Subtotal | RE-FU-021 | Valor de Venta (previo a IGV) | Subtotal CFDI |
| IdCatMoneda | RE-FU-018 | PEN / USD | MXN / USD |
| TipoCambio | RE-FU-018 | TC SUNAT del dia | TC del dia SAT |
| CondicionesPago | RE-FU-021 | Contado / Credito | PREPAGO 100%, 30 DIAS, etc. |
| Estado | RE-FU-018 | Pendiente/Timbrado/Error/Cancelado | Igual |
| IdArchivoXml | RE-FU-018 | FK -> Archivo (XML/CDR SUNAT) | FK -> Archivo (XML timbrado SAT) |
| IdArchivoPdf | RE-FU-021 | FK -> Archivo (PDF del CPE) | FK -> Archivo (PDF de la Factura) |
| FechaCertificacionSat | RE-FU-021 | Fecha/hora de aceptacion del OSE/PSE | Fecha de certificacion PAC |
| **UbigeoEmisor** | **RE-FU-022 (NUEVO)** | **Ubigeo SUNAT del emisor** | No aplica |
| **DireccionEmisor** | **RE-FU-022 (NUEVO)** | **Direccion fiscal completa emisor** | No aplica (usa LugarExpedicion=CP) |
| **DireccionReceptor** | **RE-FU-022 (NUEVO)** | **Direccion fiscal completa receptor** | No aplica |
| **TipoOperacion** | **RE-FU-022 (NUEVO)** | **Catalogo 51 SUNAT** | No aplica (UsoCFDI es SAT) |
| **ISC** | **RE-FU-022 (NUEVO)** | **Impuesto Selectivo al Consumo** | No aplica |
| **ICBPER** | **RE-FU-022 (NUEVO)** | **Impuesto a bolsas plasticas** | No aplica |
| **OtrosTributos** | **RE-FU-022 (NUEVO)** | **Otros tributos SUNAT** | No aplica |

> Columnas SAT que **no aplican** a Peru y quedan en `NULL`: `IdCatUsoCFDI`,
> `IdCatMetodoDePagoCFDI` (PPD/99 no existen en SUNAT), `RegimenFiscal`, `LugarExpedicion`,
> `RegimenFiscalReceptor`, `CodigoPostalReceptor`, `Exportacion`.

---

## Vinculacion con fccFactura

> **Migración (06/07/2026):** esta sección se actualizó porque `tpProformaAdelanto` fue reemplazada por `fccFactura` (RE-FU-015) como pendiente FAA unificado.

    fccFactura
        IdCFDIGenerada -> CFDIGenerada (MEX y PER, misma columna — movida a RE-FU-015, originalmente en RE-FU-019)

No se requiere ALTER adicional: `fccFactura.IdCFDIGenerada` ya cubre ambas regiones.

---

## Diferencias vs Factura Mexico (RE-FU-021)

| Aspecto | Mexico (CFDI 4.0) | Peru (CPE UBL 2.1) |
| ------- | ------------------ | -------------------- |
| Tabla de datos fiscales | CFDIGenerada | **CFDIGenerada (misma tabla)** |
| ID fiscal emisor | RFC (13 chars) — columna RFCEmisor | RUC (11 digitos) — misma columna RFCEmisor |
| ID fiscal receptor | RFC — columna RFCReceptor | RUC — misma columna RFCReceptor |
| Impuesto | IVA 16% (calculado de partidas) | IGV 18% (calculado de partidas) |
| Tipo operacion | IdCatUsoCFDI (SAT) | TipoOperacion cat.51 SUNAT (columna nueva) |
| Metodo/Forma Pago | IdCatMetodoDePagoCFDI (PPD+99) | **NO existen — quedan NULL** |
| Complemento de Pago | Si (modulo Validar Cobro) | **No existe** |
| Elementos tecnicos | UUID, sellos, cadena SAT (via XML en IdArchivoXml) | QR, firma digital, resumen SUNAT (via XML en IdArchivoXml) |
| Totales extra | No | ISC, ICBPER, OtrosTributos (columnas nuevas) |
| Folio fiscal | UUID (columna UUID) | No hay UUID — Serie+Folio(Correlativo) identifican el CPE |
| Direccion fiscal | LugarExpedicion (solo CP) | DireccionEmisor/DireccionReceptor + UbigeoEmisor (columnas nuevas) |

---

## Fuentes de Datos para el PDF Peru

| Seccion PDF | Tabla Fuente | Campo | Existe? |
|-------------|-------------|-------|---------|
| Branding Emisor | Empresa (GOLPERU) | Prefijo -> DocumentBuilder | SI |
| Datos Emisor - RUC | CFDIGenerada | RFCEmisor | SI |
| Datos Emisor - RazonSocial | CFDIGenerada | RazonSocialEmisor | SI (RE-FU-021) |
| Datos Emisor - Direccion | CFDIGenerada | **DireccionEmisor (FALTA - ALTER)** | NO |
| Datos Emisor - Ubigeo | CFDIGenerada | **UbigeoEmisor (FALTA - ALTER)** | NO |
| Datos Receptor - RUC | CFDIGenerada | RFCReceptor | SI |
| Datos Receptor - RazonSocial | CFDIGenerada | RazonSocialReceptor | SI (RE-FU-021) |
| Datos Receptor - Direccion | CFDIGenerada | **DireccionReceptor (FALTA - ALTER)** | NO |
| Serie + Correlativo | CFDIGenerada | Serie, Folio | SI |
| TipoOperacion | CFDIGenerada | **TipoOperacion (FALTA - ALTER)** | NO |
| Moneda/TC | CFDIGenerada | IdCatMoneda, TipoCambio | SI |
| CondicionPago | CFDIGenerada | CondicionesPago | SI (RE-FU-021) |
| Totales SUNAT | CFDIGenerada | Subtotal, **ISC (FALTA)**, **ICBPER (FALTA)**, **OtrosTributos (FALTA)**, Total | Parcial |
| Partidas - CodigoSUNAT | Producto.CodigoSUNAT | BRECHA (RE-FU-020) | NO |
| Partidas - ClaveSUNAT UdM | catUnidad.ClaveSUNAT | BRECHA (RE-FU-020) | NO |
| Partidas - AfectacionIGV | catAfectacionIGV | BRECHA (RE-FU-020) | NO |
| QR/Firma/Resumen SUNAT | XML del OSE/PSE (via CFDIGenerada.IdArchivoXml) | Parseo directo — no columnas | SI (patron) |
| PDF persistido | Archivo, referenciado desde CFDIGenerada.IdArchivoPdf | FileKey, FileBucket='facturas' | SI (RE-FU-021) |

---

## Persistencia del PDF

    Al timbrar exitosamente ante SUNAT/OSE (orquestado por ProquifaDotNet.Finanzas — CfdiService,
    igual patron que RE-FU-021 / RE-FU-020):
    1. OSE retorna CDR de aceptacion + XML firmado
    2. INSERT Archivo (XML, FileBucket='facturas', IdRegion=PER) -> UPDATE CFDIGenerada SET IdArchivoXml
    3. Generar PDF desde CFDIGenerada + datos fiscales de producto + TimbreFiscalDigital/CDR (parseado del XML)
    4. INSERT Archivo (PDF, FileBucket='facturas', IdRegion=PER) -> UPDATE CFDIGenerada SET IdArchivoPdf

    Al consultar historico:
    -> Descarga PDF desde Minio via Archivo.FileKey (sin regenerar)

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
| 1 | ALTER TABLE CFDIGenerada ADD UbigeoEmisor, DireccionEmisor, DireccionReceptor, TipoOperacion, ISC, ICBPER, OtrosTributos | CFDIGenerada debe existir (RE-FU-019) |
| 2 | Pasos RE-FU-020: CodigoSUNAT + catAfectacionIGV | RE-FU-020 |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-020 | Dispara generacion de este PDF (timbrado SUNAT); CFDIGenerada ya se usa alli para Peru |
| R16A-RE-FU-019 | CREATE TABLE CFDIGenerada (columnas base) |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura` — `IdCFDIGenerada` (antes en `tpProformaAdelanto`, movida el 06/07/2026) |
| R16A-RE-FU-018 | ALTER CFDIGenerada — columnas tecnicas de timbrado (IdCatMoneda, TipoCambio, IdArchivoXml, Estado, etc.) |
| R16A-RE-FU-021 | PDF Factura MEX (mismo patron, misma tabla) — aporta RazonSocialEmisor/Receptor, Subtotal, CondicionesPago, IdArchivoPdf, FechaCertificacionSat, reutilizados aqui |
| R16A-RE-FU-005 | Brechas Peru (B1 bloqueante productos SUNAT) |
| R16A-RE-FU-017 | Patron Minio PER (bucket 'facturas' region PER) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
