# Impacto en BD — Validar Cobro: Paso 3 Perú (Facturación y Envío)
**Requisito:** TPSC-RE-FU-029
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (lectura/escritura)
**Versión:** 1.0

---

## Resumen

Paso 3 del wizard Validar Cobro Perú: timbrado ante SUNAT y envío individual de la Factura
electrónica (CPE tipo 01, UBL 2.1) por cada línea derivada de la asociación cerrada en el
Paso 2. A diferencia de México, Perú tiene un único tipo de documento (Factura electrónica),
sin Complemento de Pago ni cascada PPD ni transferencia a Legacy.

Las estructuras de control de estado del Paso 3 (`fccDocumentoFiscalCobro`,
`fccConfirmacionPedido`, `catTipoDocumentoFiscal`, `catDocumentoFiscalCobroEstado`,
`tpPedido.FechaEstimadaEntrega`) fueron creadas en RE-FU-028 y se **reutilizan** para Perú.

Las estructuras de timbrado Peru (`CFDIGenerada`, `EmpresaFolio` con fila GOLPERU,
`catCondicionesDePago`) fueron creadas en RE-FU-018/019/020 y también se **reutilizan**.

RE-029 agrega únicamente: el catálogo de Tipo de Operación SUNAT (catálogo 51), la clave
`FACTURA_CPE` en `catTipoCFDI`, y extiende `fccDocumentoFiscalCobro` con los dos campos
fiscales de Perú (TipoOperacion y CondicionPago). La vista consolidada se actualiza para
incluir las columnas Perú.

> **Nota — corrección a RE-028:**
> La descripción de `fccConfirmacionPedido` en RE-FU-028 indicaba "exclusivamente México".
> RE-029 confirma que la Confirmación de Pedido **también aplica a Perú** (Reglas 10 y 11
> del requisito). La tabla `fccConfirmacionPedido` es compartida; solo se requiere registrar
> el template DocumentBuilder `{PrefijoPeru}_PER_CDP` vía DML en `DocumentTemplate`.

> ⚠️ **Brecha mayor — Modalidad de emisión electrónica SUNAT:**
> La integración con SUNAT (SEE-SOL, SEE del Contribuyente, SEE-OSE o Facturador SUNAT)
> está pendiente de definición (ver Brecha 5 de TPSC-RE-FU-005 y Brechas B1–B4 de RE-FU-020).
> El timbrado Perú está bloqueado hasta resolver estas brechas; las estructuras BD se diseñan
> desde ahora para ser compatibles con cualquier modalidad.

---

## Impacto en BD

| #   | Cambio                                                                                     | Base de Datos          | Tipo      | Prioridad |
| --- | ------------------------------------------------------------------------------------------ | ---------------------- | --------- | --------- |
| 1   | CREATE TABLE catTipoOperacionSUNAT (catálogo 51 SUNAT)                                     | ProquifaDotNet         | DDL + DML | Alta      |
| 2   | ALTER TABLE catTipoCFDI — ADD IdRegion + UPDATE entradas MEX + INSERT `FACTURA_CPE` (PER)  | ProquifaDotNet         | DDL + DML | Alta      |
| 3   | ALTER TABLE fccDocumentoFiscalCobro — ADD columnas Perú (TipoOperacion, CondicionPago)     | ProquifaDotNet         | DDL       | Alta      |
| 4   | ALTER VIEW vfccDocumentoFiscalCobro — extender con JOINs Perú                              | ProquifaDotNet         | DDL       | Media     |
| 5   | DML DocumentTemplate — registrar template `{PrefijoPeru}_PER_CDP`                          | ProquifaDotNet         | DML       | Media     |
| —   | Reutiliza: CFDIGenerada (RE-018/019; Peru CPE se almacena con Clave=`FACTURA_CPE`)         | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: EmpresaFolio GOLPERU (fila insertada en RE-020; Serie F001)                     | ProquifaDotNetTimbrado | Existente | —         |
| —   | Reutiliza: catCondicionesDePago (RE-018/019; Contado/Crédito ya existen)                   | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: fccDocumentoFiscalCobro (estructura base RE-028)                                | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: fccConfirmacionPedido (RE-028; aplica también a Perú)                           | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: catTipoDocumentoFiscal (clave `FACTURA` ya insertada en RE-028)                 | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: catDocumentoFiscalCobroEstado (ciclo PENDIENTE/GENERADO/ENVIADO RE-028)         | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: tpPedido.FechaEstimadaEntrega (agregada en RE-028)                              | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: tpProformaPedido.IdCFDIGenerada (RE-026; campo existente, Perú lo puebla igual) | ProquifaDotNet         | Existente | —         |
| —   | Reutiliza: fccPagoFacturaPedido / fccPagoFacturaAdelanto (RE-026)                          | ProquifaDotNet         | Existente | —         |

---

## Catálogo Nuevo: catTipoOperacionSUNAT

**Descripción:** Catálogo de tipos de operación SUNAT (catálogo 51) utilizados en la emisión
de la Factura electrónica peruana. Se consigna por línea en el Paso 3 en lugar del Uso CFDI
mexicano. El valor configurable por el operador o fijo por el sistema está pendiente de
confirmar (ver Regla 5 del requisito y Brecha B5 de TPSC-RE-FU-020).

```sql
CREATE TABLE [dbo].[catTipoOperacionSUNAT](
    [IdCatTipoOperacionSUNAT]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catTipoOperacionSUNAT_Id]     DEFAULT (NEWID()),
    [Clave]                    varchar(10)      NOT NULL,
        -- Código catálogo 51 SUNAT: '0101', '0112', '0200', '1001', '2001', etc.
    [Descripcion]              nvarchar(200)    NOT NULL,
    [Activo]                   bit              NOT NULL
        CONSTRAINT [DF_catTipoOperacionSUNAT_Activo] DEFAULT (1),
    [FechaRegistro]            datetime2(7)     NOT NULL
        CONSTRAINT [DF_catTipoOperacionSUNAT_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catTipoOperacionSUNAT]
        PRIMARY KEY CLUSTERED ([IdCatTipoOperacionSUNAT]),
    CONSTRAINT [UQ_catTipoOperacionSUNAT_Clave]
        UNIQUE ([Clave])
);
GO
-- Datos iniciales — catálogo 51 SUNAT (operaciones frecuentes para distribución B2B)
INSERT INTO dbo.catTipoOperacionSUNAT (Clave, Descripcion) VALUES
    ('0101', 'Venta interna'),
    ('0112', 'Venta interna — sustento de traslado de mercancía'),
    ('0200', 'Exportación de bienes'),
    ('0201', 'Exportación de servicios'),
    ('1001', 'Operación sujeta a detracción'),
    ('1002', 'Operación sujeta a detracción con liquidación'),
    ('2001', 'Operación sujeta a percepción');
-- Extender con los códigos adicionales del catálogo 51 que Golocaer S.A.C. requiera.
```

### Diccionario de datos — catTipoOperacionSUNAT

| Nombre de tabla | Descripción |
|-----------------|-------------|
| catTipoOperacionSUNAT | Catálogo 51 SUNAT — tipos de operación para facturas electrónicas Perú. Equivalente al `catUsoCFDI` del SAT en México. |

**Columnas:**

| Nombre | Tipo | Descripción |
|--------|------|-------------|
| IdCatTipoOperacionSUNAT | uniqueidentifier PK | Identificador único del catálogo |
| Clave | varchar(10) NOT NULL UNIQUE | Código catálogo 51 SUNAT (ej. `0101`, `1001`) |
| Descripcion | nvarchar(200) NOT NULL | Descripción legible del tipo de operación |
| Activo | bit NOT NULL | 1 = vigente |
| FechaRegistro | datetime2(7) NOT NULL | Fecha de inserción |

**Índices:**

| Nombre | Columnas | Tipo |
|--------|----------|------|
| PK_catTipoOperacionSUNAT | IdCatTipoOperacionSUNAT | PRIMARY KEY CLUSTERED |
| UQ_catTipoOperacionSUNAT_Clave | Clave | UNIQUE non-clustered |

---

## ALTER TABLE catTipoCFDI — Agregar IdRegion + FACTURA_CPE

`catTipoCFDI` fue creado en RE-FU-028 exclusivamente para México. RE-029 es el primer
requisito que usa el catálogo en una segunda región (Perú), por lo que se agrega la columna
`IdRegion` para que el wizard pueda filtrar los tipos de documento por región del cliente y
evitar combinaciones inválidas (ej. que un CPE aparezca como opción para una línea México).

```sql
-- Prerrequisito: catTipoCFDI debe existir (RE-FU-028); catRegion debe existir

-- 1. Agregar columna (nullable primero para no romper filas existentes)
ALTER TABLE dbo.catTipoCFDI
    ADD [IdRegion] uniqueidentifier NULL;
GO

ALTER TABLE dbo.catTipoCFDI
    ADD CONSTRAINT [FK_catTipoCFDI_Region]
        FOREIGN KEY ([IdRegion]) REFERENCES dbo.catRegion([IdRegion]);
GO

-- 2. Poblar las entradas México de RE-FU-028
UPDATE dbo.catTipoCFDI
SET IdRegion = (SELECT IdRegion FROM dbo.catRegion WHERE Clave = 'MEX')
WHERE Clave IN ('FACTURA_PPD', 'FACTURA_PUE', 'FACTURA_ANTICIPO', 'COMPLEMENTO_PAGO');
GO

-- 3. Insertar entrada Perú
INSERT INTO dbo.catTipoCFDI (Clave, Descripcion, IdRegion)
SELECT 'FACTURA_CPE',
       'Factura electrónica SUNAT — CPE tipo 01 UBL 2.1 (Perú)',
       IdRegion
FROM dbo.catRegion
WHERE Clave = 'PER';
GO
```

> **Consideración:** `IdRegion` se deja nullable (no se convierte a NOT NULL) para permitir
> posibles entradas genéricas o multi-región en el futuro. A nivel de aplicación, al guardar
> o enviar una línea se valida que `catTipoCFDI.IdRegion` coincida con la región del cliente.

**Columna agregada a catTipoCFDI:**

| Nombre | Tipo | Descripción |
|--------|------|-------------|
| IdRegion | uniqueidentifier FK NULL | Región a la que aplica este tipo de documento. FK a `catRegion`. `'MEX'` para CFDI SAT; `'PER'` para CPE SUNAT. NULL = multi-región (reservado) |

> **Nota de diseño:** `CFDIGenerada` es la tabla compartida para documentos fiscales tanto de
> México (CFDI SAT) como de Perú (CPE SUNAT). Para un CPE peruano los campos se mapean así:



| Campo `CFDIGenerada` | Valor México                            | Valor Perú (CPE tipo 01)               |
| -------------------- | --------------------------------------- | -------------------------------------- |
| UUID                 | UUID SAT (36 chars)                     | `NULL` (SUNAT no genera UUID)          |
| Folio                | Folio numérico SAT                      | Correlativo 8 dígitos (ej. `00000001`) |
| Serie                | Serie alfanumérica SAT                  | Serie SUNAT (ej. `F001`)               |
| FechaEmision         | Fecha timbrado SAT                      | Fecha emisión SUNAT                    |
| Total                | Total CFDI (MXN)                        | Total CPE (PEN o USD)                  |
| IdCatTipoCFDI        | FACTURA_PPD / FACTURA_PUE / etc.        | `FACTURA_CPE`                          |
| IdCFDIRelacionado    | UUID CFDI relacionado (Complemento PPD) | `NULL`                                 |

---

## ALTER TABLE fccDocumentoFiscalCobro — Columnas Perú

La tabla `fccDocumentoFiscalCobro` (RE-FU-028) maneja el ciclo de vida de cada línea del
Paso 3. Sus columnas actuales para México (`IdCatUsoCFDI`, `IdCatMetodoDePagoCFDI`) son
nullable. RE-029 agrega dos columnas nullable para las líneas Perú, que son los equivalentes
peruanos de esos campos.

El CPE timbrado se referencia mediante la columna existente `IdCFDIGeneradaFactura` (igual
que México usa para la Factura), con `IdCatTipoCFDI = 'FACTURA_CPE'` en `CFDIGenerada`
como discriminador. No se agrega una columna separada para el CPE.

```sql
-- Prerrequisito: catTipoOperacionSUNAT debe existir; catCondicionesDePago ya existe (RE-018/019)
-- Ejecutar en ProquifaDotNet
ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD [IdCatTipoOperacionSUNAT] uniqueidentifier NULL,
        -- FK a catTipoOperacionSUNAT. NULL para líneas México; valor para líneas Perú.
        -- Equivalente al IdCatUsoCFDI de México.
        [IdCatCondicionesDePago]  uniqueidentifier NULL;
        -- FK a catCondicionesDePago (tabla existente de RE-018/019; CONTADO/CRÉDITO).
        -- NULL para líneas México; valor para líneas Perú.
        -- Equivalente al IdCatMetodoDePagoCFDI de México.
GO

ALTER TABLE dbo.fccDocumentoFiscalCobro
    ADD CONSTRAINT [FK_fccDocumentoFiscalCobro_TipoOperacionSUNAT]
        FOREIGN KEY ([IdCatTipoOperacionSUNAT])
        REFERENCES dbo.catTipoOperacionSUNAT([IdCatTipoOperacionSUNAT]),
    CONSTRAINT [FK_fccDocumentoFiscalCobro_CondicionesDePago]
        FOREIGN KEY ([IdCatCondicionesDePago])
        REFERENCES dbo.catCondicionesDePago([IdCatCondicionesDePago]);
```

**Columnas agregadas a fccDocumentoFiscalCobro:**

| Nombre | Tipo | Descripción |
|--------|------|-------------|
| IdCatTipoOperacionSUNAT | uniqueidentifier FK NULL | Tipo de operación catálogo 51 SUNAT. NULL para líneas México. Análogo a `IdCatUsoCFDI` |
| IdCatCondicionesDePago | uniqueidentifier FK NULL | Condición de pago (CONTADO / CRÉDITO) de `catCondicionesDePago`. NULL para líneas México. Análogo a `IdCatMetodoDePagoCFDI` |

**Consideraciones especiales:**
- Ambas columnas son nullable para no romper los INSERT existentes de México.
- El CPE timbrado de una línea Perú se almacena en `CFDIGenerada` y se referencia mediante `IdCFDIGeneradaFactura` (columna preexistente), con `catTipoCFDI.Clave = 'FACTURA_CPE'` como discriminador. No se necesita columna adicional.
- La validación de que líneas México lleven `IdCatUsoCFDI`/`IdCatMetodoDePagoCFDI` y líneas Perú lleven `IdCatTipoOperacionSUNAT`/`IdCatCondicionesDePago` se implementa a nivel de aplicación (Finanzas), no mediante CHECK CONSTRAINT en BD.

---

## ALTER VIEW vfccDocumentoFiscalCobro — Extensión Perú (v2.0)

La vista creada en RE-FU-028 consolida el estado del Paso 3 México. RE-029 la extiende con:
- JOINs a `catTipoOperacionSUNAT` y `catCondicionesDePago` para las columnas Perú.
- `INNER JOIN` a `catTipoDocumentoFiscal` y `catDocumentoFiscalCobroEstado` (corrección: en
  RE-028 la vista accedía a `p3l.TipoDocumentoFiscal` y `p3l.EstadoLinea` como si fueran
  columnas de texto, cuando en realidad son FKs; aquí se resuelven correctamente vía JOIN).
- `c.Region` como discriminador México/Perú.

```sql
-- Prerrequisito: todos los ALTERs de este requisito deben estar aplicados
-- Ejecutar en ProquifaDotNet
ALTER VIEW [dbo].[vfccDocumentoFiscalCobro]
AS
SELECT
    p3l.IdFCCDocumentoFiscalCobro,
    -- Tipo y estado (resueltos desde catálogos)
    tdoc.Clave                   AS TipoDocumentoFiscal,
    tdoc.Descripcion             AS TipoDocumentoFiscalDescripcion,
    est.Clave                    AS EstadoLinea,
    est.Descripcion              AS EstadoDescripcion,
    -- Región del cliente (para filtrar México vs. Perú)
    c.Region                     AS ClienteRegion,
    -- ── CAMPOS MÉXICO ──────────────────────────────────────────────────────────
    p3l.IdCatUsoCFDI,
    ufo.Clave                    AS UsoCFDIClave,
    ufo.Descripcion              AS UsoCFDIDescripcion,
    p3l.IdCatMetodoDePagoCFDI,
    mpc.Clave                    AS MetodoPagoClave,
    mpc.Descripcion              AS MetodoPagoDescripcion,
    -- CFDIs México (Factura + Complemento cascada PPD)
    p3l.IdCFDIGeneradaFactura,
    cg_f.UUID                    AS UUID_Factura,
    cg_f.Folio                   AS Folio_Factura,
    cg_f.Serie                   AS Serie_Factura,
    cg_f.FechaEmision            AS FechaEmision_Factura,
    p3l.IdCFDIGeneradaComplemento,
    cg_c.UUID                    AS UUID_Complemento,
    cg_c.Folio                   AS Folio_Complemento,
    -- ── CAMPOS PERÚ ────────────────────────────────────────────────────────────
    p3l.IdCatTipoOperacionSUNAT,
    tos.Clave                    AS TipoOperacionSUNATClave,
    tos.Descripcion              AS TipoOperacionSUNATDescripcion,
    p3l.IdCatCondicionesDePago,
    cdp.Clave                    AS CondicionPagoClave,
    cdp.Descripcion              AS CondicionPagoDescripcion,
    -- CPE Perú: mismo JOIN que cg_f; Serie SUNAT y Folio = Correlativo
    -- IdCFDIGeneradaFactura reutilizado; catTipoCFDI.Clave = 'FACTURA_CPE' discrimina
    cg_f.Serie                   AS CPE_Serie,       -- 'F001' para Perú; NULL para México hasta timbrar
    cg_f.Folio                   AS CPE_Correlativo, -- '00000001' para Perú
    -- ── CAMPOS COMPARTIDOS ─────────────────────────────────────────────────────
    -- Origen proforma
    p3l.IdFCCPagoFacturaPedido,
    pfp.IdFCCPagoCliente         AS IdFCCPagoCliente_PFP,
    pfp.IdTPProformaPedido,
    pp.Folio                     AS FolioProforma,
    pp.MontoTotal                AS MontoProforma,
    pp.MontoPendiente,
    e_pp.Prefijo                 AS EmpresaEmisoraProforma,
    -- Origen FAA
    p3l.IdFCCPagoFacturaAdelanto,
    pfa.IdFCCPagoCliente         AS IdFCCPagoCliente_PFA,
    pfa.IdTPProformaAdelanto,
    pa.Monto                     AS MontoFAA,
    cg_faa.UUID                  AS UUID_FAA,
    -- Cobro
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
    -- Pedido
    tp.IdTPPedido,
    tp.FolioPedidoInterno,
    tp.FechaEstimadaEntrega,
    -- Timestamps
    p3l.FechaGeneracion,
    p3l.FechaEnvio,
    p3l.FechaRegistro,
    p3l.FechaUltimaActualizacion
FROM dbo.fccDocumentoFiscalCobro p3l
-- Catálogos compartidos (tipo y estado — JOIN para resolver Clave correctamente)
INNER JOIN dbo.catTipoDocumentoFiscal tdoc
    ON p3l.IdCatTipoDocumentoFiscal = tdoc.IdCatTipoDocumentoFiscal
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
LEFT JOIN dbo.tpProformaAdelanto pa
    ON pfa.IdTPProformaAdelanto = pa.IdTPProformaAdelanto
LEFT JOIN dbo.CFDIGenerada cg_faa
    ON pa.IdCFDIGenerada = cg_faa.IdCFDIGenerada
-- Cobro
LEFT JOIN dbo.fccPagoCliente fpc
    ON fpc.IdFCCPagoCliente = COALESCE(pfp.IdFCCPagoCliente, pfa.IdFCCPagoCliente)
-- Cliente y datos de facturación
LEFT JOIN dbo.Cliente c
    ON fpc.IdCliente = c.IdCliente
LEFT JOIN dbo.DatosFacturacionCliente dfc
    ON fpc.IdCliente = dfc.IdCliente AND dfc.Activo = 1
-- Pedido
LEFT JOIN dbo.tpPedidoProformaPedido tpp_pp
    ON pp.IdTPProformaPedido = tpp_pp.IdTPProformaPedido AND tpp_pp.Activo = 1
LEFT JOIN dbo.tpPedido tp
    ON tpp_pp.IdTPPedido = tp.IdTPPedido
-- ── JOINs México ──────────────────────────────────────────────────────────────
LEFT JOIN dbo.catUsoCFDI ufo
    ON p3l.IdCatUsoCFDI = ufo.IdCatUsoCFDI
LEFT JOIN dbo.catMetodoDePagoCFDI mpc
    ON p3l.IdCatMetodoDePagoCFDI = mpc.IdCatMetodoDePagoCFDI
LEFT JOIN dbo.CFDIGenerada cg_f
    ON p3l.IdCFDIGeneradaFactura = cg_f.IdCFDIGenerada
LEFT JOIN dbo.CFDIGenerada cg_c
    ON p3l.IdCFDIGeneradaComplemento = cg_c.IdCFDIGenerada
-- ── JOINs Perú ────────────────────────────────────────────────────────────────
LEFT JOIN dbo.catTipoOperacionSUNAT tos
    ON p3l.IdCatTipoOperacionSUNAT = tos.IdCatTipoOperacionSUNAT
LEFT JOIN dbo.catCondicionesDePago cdp
    ON p3l.IdCatCondicionesDePago = cdp.IdCatCondicionesDePago;
```

**Columnas nuevas en v2.0:**

| Columna | Origen | Descripción |
|---------|--------|-------------|
| **ClienteRegion** | `Cliente.Region` | `'MEX'` o `'PER'` — discrimina líneas por región |
| TipoDocumentoFiscal / EstadoLinea | JOIN a catálogos (corrección RE-028) | Ahora resueltos vía `INNER JOIN`; antes se accedían como columnas directas |
| IdCatTipoOperacionSUNAT / TipoOperacionSUNATClave | catTipoOperacionSUNAT | Tipo de operación catálogo 51 SUNAT; NULL para México |
| IdCatCondicionesDePago / CondicionPagoClave | catCondicionesDePago | CONTADO / CRÉDITO; NULL para México |
| CPE_Serie / CPE_Correlativo | alias de `cg_f.Serie` / `cg_f.Folio` | Reutiliza el JOIN `cg_f` (IdCFDIGeneradaFactura); `catTipoCFDI.Clave='FACTURA_CPE'` discrimina |

---

## DML: Registro de template DocumentBuilder Perú (CDP)

La Confirmación de Pedido aplica también a Perú (Regla 10 del requisito). Se requiere un
template `{PrefijoPeru}_PER_CDP` en DocumentBuilder análogo a los templates
`GOL/MUN/PRO/PQF_MEX_CDP` de México. El prefijo de Golocaer S.A.C. (`GOLPERU`) se
toma de la tabla `Empresa` donde ya existe la fila para RE-020.

```sql
-- Confirmar el TemplateKey exacto con el equipo DocumentBuilder antes de ejecutar.
-- El Prefijo de Golocaer S.A.C. en la tabla Empresa es 'GOLPERU' (RE-020).

-- INSERT INTO dbo.DocumentTemplate (Clave, Descripcion, TemplateHeader, TemplateBody, TemplateFooter, Activo)
-- VALUES (
--     'GOLPERU_PER_CDP',
--     'Confirmación de Pedido Prepago Perú — Golocaer S.A.C.',
--     'GOLPERU_PER_CDP_H.html',
--     'GOLPERU_PER_CDP_B.html',
--     'GOLPERU_PER_CDP_F.html',
--     1
-- );

-- ⚠️ La creación del template HTML (GOLPERU_PER_CDP_H/B/F.html) es una tarea DocumentBuilder
--    independiente — ver Tarea CREATE-PDF en TPSC-RE-FU-029-Tareas.md.
```

---

## Tablas Leídas (Perú)

| Tabla | Datos leídos | Diferencia vs México |
|-------|-------------|----------------------|
| fccDocumentoFiscalCobro | Todas las columnas | Mismo ciclo de vida; columnas Perú nuevas |
| fccPagoFacturaPedido | IdFCCPagoCliente, IdTPProformaPedido, Monto | Sin diferencia |
| fccPagoFacturaAdelanto | IdFCCPagoCliente, IdTPProformaAdelanto, Monto | Perú: sin generación de CPE (conciliación interna — Regla 4) |
| fccPagoCliente | Folio, Monto, MXN, USD, TipoDeCambio, IdCliente | Sin diferencia |
| tpProformaPedido | Folio, MontoTotal, MontoPendiente, IdEmpresa | Sin `HayControlados` (no aplica en Perú) |
| tpProformaAdelanto | Monto, IdEmpresa | Sin generación de CPE en RE-029 |
| DatosFacturacionCliente | RazonSocial, RFC (RUC en Perú), RegimenFiscal | RFC = RUC 11 dígitos |
| Empresa (GOLPERU) | Prefijo, RazonSocial, RUC emisor | Solo GOLPERU; datos pendientes de B3 RE-020 |
| CFDIGenerada | UUID (NULL), Folio (Correlativo), Serie (F001), FechaEmision | Misma tabla; `IdCatTipoCFDI = 'FACTURA_CPE'` |
| catTipoDocumentoFiscal | Clave `FACTURA` | Solo un tipo en Perú |
| catDocumentoFiscalCobroEstado | Clave, Descripcion | Sin diferencia |
| catTipoOperacionSUNAT | IdCatTipoOperacionSUNAT, Clave, Descripcion | **Nueva — RE-029** |
| catCondicionesDePago | IdCatCondicionesDePago, Clave (CONTADO/CRÉDITO) | Existente desde RE-018/019 |
| catTipoCFDI | Clave `FACTURA_CPE` | Nueva clave insertada en RE-029 |
| fccNotaCredito | Id, Monto | NCs Perú; mecánica de referencia catálogo 09 SUNAT pendiente (RE-033/035) |
| tpPedido | FolioPedidoInterno, IdContacto | Sin diferencia |
| EmpresaFolio GOLPERU (ProquifaDotNetTimbrado) | Serie (`F001`), UltimoFolio | Misma tabla con fila GOLPERU de RE-020 |

---

## Tablas Escritas (runtime — Perú)

| Tabla | Momento | Operación |
|-------|---------|-----------|
| fccDocumentoFiscalCobro | Al iniciar Paso 3 | INSERT línea: `TipoDocumentoFiscal = 'FACTURA'`, `EstadoLinea = 'PENDIENTE'` |
| fccDocumentoFiscalCobro | Auto-guardado TipoOperacion / CondicionPago | UPDATE `IdCatTipoOperacionSUNAT`, `IdCatCondicionesDePago` |
| fccDocumentoFiscalCobro | Timbrado exitoso | UPDATE `EstadoLinea = 'GENERADO'`, `IdCFDIGeneradaFactura = @IdCPE`, `FechaGeneracion` |
| fccDocumentoFiscalCobro | Envío exitoso | UPDATE `EstadoLinea = 'ENVIADO'`, `FechaEnvio` |
| CFDIGenerada | Timbrado exitoso | INSERT (`Serie='F001'`, `Folio=Correlativo`, `UUID=NULL`, `IdCatTipoCFDI='FACTURA_CPE'`) vía servicio Timbrado Perú |
| EmpresaFolio GOLPERU | Timbrado exitoso | UPDATE `UltimoFolio + 1` (UPDLOCK atómico — misma mecánica México) |
| tpProformaPedido | Timbrado exitoso | UPDATE `IdCFDIGenerada = @IdCPE` (marca proforma como facturada; mismo campo que México) |
| fccConfirmacionPedido | Envío exitoso | INSERT (FolioConfirmacion, RutaArchivoPDF) — igual que México |
| tpPedido | Envío exitoso | UPDATE `FechaEstimadaEntrega` (FEE — aplica también a Perú) |
| CorreoEnviado | Envío exitoso | INSERT (registro del correo enviado) |
| ArchivoCorreoEnviado | Envío exitoso | INSERT x N: PDF CPE + XML CPE + PDF Confirmación de Pedido |

---

## Flujo de Datos (Perú)

```
1. INICIAR PASO 3 (al avanzar desde Paso 2 — cliente Región Perú)
   Lee:  fccPagoFacturaPedido, fccPagoFacturaAdelanto (asociación Paso 2)
         DatosFacturacionCliente (RUC receptor), Empresa GOLPERU (emisora)
   Escribe: fccDocumentoFiscalCobro INSERT una línea por documento
            IdCatTipoDocumentoFiscal → 'FACTURA' (único tipo en Perú)
            IdCatDocumentoFiscalCobroEstado → 'PENDIENTE'

2. EDITAR LÍNEA (Tipo Operación SUNAT / Condición de Pago)
   Lee:  catTipoOperacionSUNAT, catCondicionesDePago
   Escribe: fccDocumentoFiscalCobro UPDATE IdCatTipoOperacionSUNAT, IdCatCondicionesDePago

3. PREVISUALIZAR
   Lee:  vfccDocumentoFiscalCobro, fccNotaCredito
   Sin escrituras. PDF generado en memoria (template {Prefijo}_PER_FAC de RE-020).

4. TIMBRAR (⚠️ bloqueado por brechas RE-020: B1 datos SUNAT producto, B2 OSE/PSE)
   ProquifaDotNet.Timbrado (módulo Perú):
     INSERT CFDIGenerada (Serie='F001', Folio=Correlativo, UUID=NULL, IdCatTipoCFDI='FACTURA_CPE')
     UPDATE EmpresaFolio GOLPERU SET UltimoFolio + 1 (UPDLOCK)
   ProquifaDotNet:
     UPDATE tpProformaPedido SET IdCFDIGenerada = @IdCPE
     UPDATE fccDocumentoFiscalCobro SET EstadoLinea='GENERADO', IdCFDIGeneradaFactura, FechaGeneracion

5. ENVIAR
   ProquifaDotNet.Finanzas:
     Genera PDF Confirmación de Pedido vía DocumentBuilder (GOLPERU_PER_CDP) → sube a MinIO
     INSERT fccConfirmacionPedido (FolioConfirmacion, RutaArchivoPDF)
     INSERT CorreoEnviado + ArchivoCorreoEnviado (PDF CPE + XML CPE + PDF Confirmación)
     UPDATE fccDocumentoFiscalCobro SET EstadoLinea='ENVIADO', FechaEnvio
     UPDATE tpPedido SET FechaEstimadaEntrega (FEE)
     [SIN transferencia a Legacy — exclusiva de México]
```

---

## Diferencias clave México vs. Perú en BD

| Aspecto | México (RE-028) | Perú (RE-029) |
|---------|-----------------|----------------|
| Tipos de documento | FACTURA, FACTURA_ANTICIPO, COMPLEMENTO_PAGO | Solo FACTURA |
| Campo catálogo 1 | `IdCatUsoCFDI` → `catUsoCFDI` (SAT) | `IdCatTipoOperacionSUNAT` → `catTipoOperacionSUNAT` (catálogo 51 SUNAT) |
| Campo catálogo 2 | `IdCatMetodoDePagoCFDI` → `catMetodoDePagoCFDI` (PPD/PUE) | `IdCatCondicionesDePago` → `catCondicionesDePago` (CONTADO/CRÉDITO) |
| Tabla comprobante | `CFDIGenerada` con `UUID`, `IdCatTipoCFDI = FACTURA_PPD/PUE/ANTICIPO` | `CFDIGenerada` con `UUID=NULL`, `IdCatTipoCFDI = FACTURA_CPE` |
| Serie / folio | Serie SAT (alfanumérica), Folio numérico | Serie SUNAT (`F001`), Folio = Correlativo 8 dígitos |
| Control de series | `EmpresaFolio` GOL/MUN/PRO/PQF | `EmpresaFolio` GOLPERU (fila ya insertada en RE-020) |
| Cascada documentos | Factura PPD → Complemento (2 CFDIs) | Solo 1 CPE por línea |
| FEE y Confirmación de Pedido | Sí (`*_MEX_CDP`) | Sí (`GOLPERU_PER_CDP`) |
| Transferencia Legacy | Sí (E1/E2/E3/E6 en RE-028 T17) | No |
| Timbrado | PAC TurboPac (SAT México) | OSE/PSE/SUNAT (modalidad pendiente) |
