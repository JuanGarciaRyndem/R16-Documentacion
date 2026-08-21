# Impacto en BD - Diseno y Generacion PDF Proforma Mexico
**Requisito:** R16A-RE-FU-016
**Base de Datos:** ProquifaDotNet
**Version:** 3.0 (rev. 2026-07-21 — OBS-TC: tpProformaPedido.TipoCambio, nota coordinador)

---

## Resumen
Generacion del PDF de Proforma al tramitar pedido Prepago sin FAA para clientes Mexico.
Branding diferenciado por empresa emisora (GOL/MUN/PRO/PQF). Foliador global PRF.
PDF se genera bajo demanda, se persiste en Minio via tabla Archivo tras envio exitoso.
Logos se resuelven por Prefijo en DocumentBuilder (repositorio o base64), no en BD.

---

## Impacto en BD

| #   | Cambio                                                          | Tipo         | Prioridad |
| --- | --------------------------------------------------------------- | ------------ | --------- |
| 1   | ALTER TABLE tpProformaPedido ADD FolioProforma varchar(80) NULL | ALTER        | Alta      |
| 2   | CREATE SEQUENCE dbo.SeqFolioProforma                            | DDL Nuevo    | Alta      |
| 3   | CREATE VIEW dbo.vtpProformaPedido                               | Vista nueva  | Alta      |
| 4   | ClienteDatosBancarios (RE-FU-006)                               | Prerequisito | Alta      |
| 5   | EmpresaDatosBancarios.IdRegion (RE-FU-001)                      | Prerequisito | Alta      |
| 6   | ALTER TABLE tpProformaPedido ADD TipoCambio decimal(18,4) NULL  | ALTER        | Alta      |

> tpProformaPedido.Folio/Serie/Uuid son de la **factura CFDI**, no de la proforma.
> Se requiere campo **FolioProforma** nuevo para el foliador PRF global.
> Se reutiliza: tpPedido.IdArchivo -> Archivo (FileKey + FileBucket) para el PDF.
> Logos NO van en BD ni en Minio — se resuelven por Prefijo en el DocumentBuilder.

---

## Aclaracion: campos existentes en tpProformaPedido

| Campo existente                 | Uso REAL                                    | Es folio proforma? |
| ------------------------------- | ------------------------------------------- | ------------------ |
| Folio                           | Folio de la factura CFDI                    | NO                 |
| Serie                           | Serie de la factura CFDI                    | NO                 |
| Uuid                            | UUID del timbrado CFDI                      | NO                 |
| NumeroFactura                   | Numero de factura                           | NO                 |
| IdCFDI / IdCFDIGenerada         | FK al CFDI                                  | NO                 |
| **FolioProforma (NUEVO)**       | **Folio PRF global de la proforma**         | **SI**             |
| **ConsecutivoProforma (NUEVO)** | **Folio Consecutivo global de la proforma** | **SI**             |

---

## Script de cambios

    -- Created by GitHub Copilot in SSMS - review carefully before executing

    -- 1. Nuevo campo para folio de proforma
    ALTER TABLE dbo.tpProformaPedido
        ADD FolioProforma varchar(80) NULL;
    ALTER TABLE dbo.tpProformaPedido
        ADD ConsecutivoProforma int DEFAULT(0);

    -- 6. Tipo de cambio a nivel del documento (OBS-TC — nota coordinador 2026-07-21)
    --    Se setea al momento de generar la proforma (no heredar tpPedido.TipoCambioFacturacion que siempre = 1)
    ALTER TABLE dbo.tpProformaPedido
        ADD TipoCambio decimal(18,4) NULL;

    -- 2. Foliador global (ajustar START WITH al MAX consecutivo existente + 1)
    CREATE SEQUENCE dbo.SeqFolioProforma
        AS INT
        START WITH 1
        INCREMENT BY 1
        NO CYCLE;
    GO

    -- 3. Vista operativa con catalogos resueltos
    CREATE VIEW dbo.vtpProformaPedido
    AS
    SELECT
        pp.IdTPProformaPedido,
         --Poner Columna (FolioProforma+'-'+ConsecutivoProforma) as FolioProforma
		pp.IdCliente,
        c.Nombre              AS ClienteNombre,
        pp.IdEmpresa,
        e.Prefijo             AS EmpresaPrefijo,
        e.Alias               AS EmpresaAlias,
        pp.ReferenciaPago,
        pp.NumeroFactura,
        pp.MontoTotal,
        pp.MontoPagado,
        pp.MontoPendiente,
        pp.FechaCompromisoPago,
        pp.FechaPromesaPagoMonitoreoCobros,
        pp.FechaPagoCompleto,
        pp.FacturaFlete,
        pp.Factura,
        pp.MXN,
        pp.USD,
        pp.Folio              AS FolioFactura,
        pp.Serie              AS SerieFactura,
        pp.Uuid               AS UuidCFDI,
        pp.Cancelada,
        pp.Controlados,
        pp.Comentarios,
        pp.Publicaciones,
        pp.Revisada,
        pp.Contrarecibo,
        pp.NumeroOrdenDeCompra,
        pp.IdContactoCliente,
        pp.IdDireccionCliente,
        pp.IdCFDI,
        pp.IdCFDIGenerada,
        pp.IdTPProformaPedidoReemplazo,
        pp.FechaRegistro,
        pp.FechaUltimaActualizacion,
        pp.Activo,
        pp.PrecioFleteKPI,
        -- Datos del pedido vinculado
        tp.IdTPPedido,
        tp.FolioPedidoInterno,
        tp.IdRegion,
        r.Nombre              AS Region,
        r.ClaveISO            AS RegionClave,
        tp.IdCatCondicionesDePago,
        cdp.CondicionesDePago,
        tp.FacturaPorAdelantado,
        tp.EntregaConRemision,
        tp.Tramitado,
        tp.FechaTramitacion
    FROM dbo.tpProformaPedido pp
    LEFT  JOIN dbo.Cliente c            ON pp.IdCliente = c.IdCliente
    LEFT  JOIN dbo.Empresa e            ON pp.IdEmpresa = e.IdEmpresa
    LEFT  JOIN dbo.tpPedidoProformaPedido tpp ON pp.IdTPProformaPedido = tpp.IdTPProformaPedido
                                              AND tpp.Activo = 1
    LEFT  JOIN dbo.tpPedido tp          ON tpp.IdTPPedido = tp.IdTPPedido
    LEFT  JOIN dbo.Region r             ON tp.IdRegion = r.IdRegion
    LEFT  JOIN dbo.catCondicionesDePago cdp ON tp.IdCatCondicionesDePago = cdp.IdCatCondicionesDePago;

---

## Vista vtpProformaPedido - Columnas

| Columna              | Origen                                | Descripcion                                |
| -------------------- | ------------------------------------- | ------------------------------------------ |
| IdTPProformaPedido   | tpProformaPedido                      | PK                                         |
| FolioProforma        | FolioProforma+'-'+ConsecutivoProforma | FolioProforma(PreFijo-Fecha) + Consecutivo |
| ClienteNombre        | Cliente.Nombre                        | Nombre del cliente                         |
| EmpresaPrefijo       | Empresa.Prefijo                       | GOL/MUN/PRO/PQF (branding)                 |
| EmpresaAlias         | Empresa.Alias                         | Nombre corto empresa                       |
| MontoTotal           | tpProformaPedido                      | Monto total de la proforma                 |
| MontoPagado          | tpProformaPedido                      | Monto pagado (Validar Cobro)               |
| MontoPendiente       | tpProformaPedido                      | Monto pendiente de cobro                   |
| FolioFactura         | tpProformaPedido.Folio                | Folio de la factura CFDI                   |
| SerieFactura         | tpProformaPedido.Serie                | Serie de la factura CFDI                   |
| UuidCFDI             | tpProformaPedido.Uuid                 | UUID del timbrado                          |
| Controlados          | tpProformaPedido                      | 1=Sustancias controladas                   |
| IdTPPedido           | tpPedido (via tpPedidoProformaPedido) | Pedido vinculado                           |
| FolioPedidoInterno   | tpPedido                              | Folio interno del pedido                   |
| Region               | Region.Nombre                         | Mexico / Peru                              |
| RegionClave          | Region.ClaveISO                       | MEX / PER                                  |
| CondicionesDePago    | catCondicionesDePago                  | Texto condiciones                          |
| FacturaPorAdelantado | tpPedido                              | 1=FAA activa                               |
| EntregaConRemision   | tpPedido                              | 1=Remision                                 |
| Tramitado            | tpPedido                              | 1=Pedido tramitado                         |
| FechaTramitacion     | tpPedido                              | Fecha de tramitacion                       |

---

## Patron de Almacenamiento (existente)

    RegionConfiguracionMinioBucket
        BucketClave, BucketNombre, IdRegion, EsRegionalizable

    Archivo
        IdArchivo (PK)
        FileKey: varchar(600)   <- path en Minio
        FileBucket: varchar(100) <- bucket (default 'public/')
        IdRegion: FK Region
        Sincronizado: bit

    tpPedido.IdArchivo -> Archivo.IdArchivo  (PDF persistido)

---

## Persistencia del PDF

| Etapa | BD | Minio |
|-------|-----|-------|
| Previsualizacion | Nada | Nada (PDF en memoria) |
| Envio exitoso | INSERT Archivo + UPDATE tpPedido.IdArchivo + UPDATE tpProformaPedido.FolioProforma | Sube PDF al bucket 'pedidos' |
| Consulta historica | SELECT Archivo.FileKey, FileBucket | Descarga PDF |

**Flujo de persistencia:**

    1. NEXT VALUE FOR dbo.SeqFolioProforma -> @Consecutivo
    2. @FolioProforma = FORMAT(GETDATE(),'MMddyy') + '-' + CAST(@Consecutivo AS VARCHAR)
    3. INSERT dbo.Archivo (FileKey='proformas/PRF-{FolioProforma}.pdf', FileBucket='pedidos', IdRegion=MEX)
    4. UPDATE dbo.tpPedido SET IdArchivo = @IdArchivo
    5. UPDATE dbo.tpProformaPedido SET FolioProforma = @FolioProforma
    6. PDF binario sube a Minio bucket 'pedidos'

---

## Foliador Global PRF

| Aspecto            | Valor                                                 |
| ------------------ | ----------------------------------------------------- |
| Campo BD           | **tpProformaPedido.FolioProforma** (varchar 80) NUEVO |
| Campo BD           | **tpProformaPedido.ConsecutivoProforma** (int) NUEVO  |
| Formato interno BD | MMDDAA-Consecutivo (ej: 031826-691)                   |
| Formato visual PDF | PRF-MMDDAA-Consecutivo (prefijo PRF solo en render)   |
| Segmentacion       | Ninguna (global)                                      |
| Momento consumo    | Al confirmar envio exitoso (sin huecos)               |
| Mecanismo          | SQL SEQUENCE dbo.SeqFolioProforma                     |

---

## Branding

| Empresa | Prefijo | Resolucion |
|---------|---------|------------|
| Golocaer S.A. de C.V. | GOL | DocumentBuilder: repo versionado o base64 |
| Mungen S.A. de C.V. | MUN | DocumentBuilder: repo versionado o base64 |
| Proquifa S.A. de C.V. | PRO | DocumentBuilder: repo versionado o base64 |
| Proveedora Quimico Farmaceutica S.A. de C.V. | PQF | DocumentBuilder: repo versionado o base64 |

> DocumentBuilder recibe Empresa.Prefijo y traduce a logo + color.
> Assets viven en el repositorio de codigo, no en BD ni en Minio.

---

## Fuentes de Datos para el PDF

| Seccion PDF | Tabla Fuente | Campos |
|-------------|-------------|--------|
| Cabecera - Logo/Color | Empresa | Prefijo -> DocumentBuilder |
| Cabecera - Folio Proforma | tpProformaPedido | **FolioProforma** (NUEVO) |
| Cliente | Cliente + DatosFacturacionCliente | Alias/RazonSocial, RFC |
| Partidas | tpProformaPartidaPedido | IdProducto, NumeroDePiezas, PrecioUnitario |
| Descripcion producto | Producto + MarcaFamilia | Catalogo + Descripcion + Marca |
| Pago - Montos | tpProformaPedido | MontoTotal + calculo IVA |
| Pago - Moneda | DatosFacturacionCliente.IdCatMoneda | catMoneda.ClaveMoneda |
| Pago - Tipo cambio | tpProformaPedido | **TipoCambio** (NUEVO — seteado al generar la proforma). ~~tpPedido.TipoCambioFacturacion~~ no usar: siempre = 1 (OBS-TC) |
| Pago - Condiciones | catCondicionesDePago | CondicionesDePago texto |
| Bancarios - Cuentas | EmpresaDatosBancarios + DatosBancarios | Banco, Cuenta, CLABE, Sucursal |
| Bancarios - REF.CLIENTE | ClienteDatosBancarios (RE-FU-006) | CodigoValidador |
| Facturacion | DatosFacturacionCliente + DireccionCliente | RFC, RazonSocial, Direccion fiscal |
| Entrega | tpPedido + DireccionCliente + ContactoCliente | FolioPedido, Direccion, Contacto |
| Pie - Empresa | Empresa | RazonSocial legal, Direccion legal |

---

## Tablas Consultadas (Lectura)

| Tabla | Rol |
|-------|-----|
| tpPedido | FolioPedidoInterno, IdEmpresa, IdRegion, IdArchivo (**no leer TipoCambioFacturacion** — OBS-TC) |
| tpProformaPedido | FolioProforma, MontoTotal, ReferenciaPago, **TipoCambio** (NUEVO) |
| tpPedidoProformaPedido | Vinculacion pedido-proforma |
| tpProformaPartidaPedido | Partidas: IdProducto, Piezas, PrecioUnitario |
| tpPartidaPedido | Datos adicionales partida |
| Producto + MarcaFamilia | Catalogo, Descripcion, Marca |
| Cliente | Nombre, Alias |
| DatosFacturacionCliente | RFC, RazonSocial, IdCatMoneda, Correo |
| DireccionCliente | Direccion fiscal y de entrega |
| ContactoCliente | Contacto de entrega |
| Empresa | Prefijo, RazonSocial legal, Direccion legal |
| EmpresaDatosBancarios | Cuentas bancarias del grupo (MN + DLS) |
| DatosBancarios | NumeroDeCuenta, Clabe, Sucursal |
| catBanco | Nombre del banco |
| catMoneda | ClaveMoneda (MXN/USD) |
| catCondicionesDePago | CondicionesDePago texto |
| Region | Filtro MEX |
| Archivo | FileKey, FileBucket (persistencia PDF) |
| RegionConfiguracionMinioBucket | Bucket 'pedidos' |
| ClienteDatosBancarios (RE-FU-006) | CodigoValidador para REF.CLIENTE |

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | START WITH del SEQUENCE | Tecnico | Consultar MAX(consecutivo) en produccion |
| 2 | ~~Vigencia del documento~~ | Negocio | **Resuelto (DUDA-033, 2026-08-21):** 30 dias naturales desde la generacion |
| 3 | ~~Alias vs RazonSocial en seccion Cliente~~ | Negocio | **Resuelto (DUDA-034, 2026-08-21):** Razon Social |
| 4 | ~~Contacto de entrega: cual contacto~~ | Negocio | **Resuelto (DUDA-037, 2026-08-21):** Titulo+Contacto, referencia tabla Pedidos en Legacy |
| 5 | Certificaciones pie (ISO/NEEC) | Negocio | Confirmar vigencia |
| 6 | ~~Cuentas siempre MN+DLS~~ | Negocio | **Resuelto (DUDA-036, 2026-08-21):** dos cuentas activas mas recientes por Fecha de ultima actualizacion; si solo hay una, se muestra esa. Aplica tambien a Peru |
| 7 | ~~PUE siempre en Prepago~~ | Fiscal | **Resuelto (DUDA-035, 2026-08-21):** leyenda PUE fija, Prepago siempre asume PUE |
| 8 | ~~Prefijo PRF en BD o solo render~~ | Tecnico | **Resuelto (DUDA-032, 2026-08-21):** solo render, sin prefijo en BD (confirma recomendacion) |
| 9 | **OBS-TC** — `tpPedido.TipoCambioFacturacion` siempre = 1 (incorrecto) | Diseño/Bug | Ver nota abajo |
| 10 | ~~Momento de consumo del folio~~ | Tecnico | **Resuelto (DUDA-031, 2026-08-21):** al confirmar envio exitoso, sin huecos (ya reflejado en tabla "Foliador Global PRF") |
| 11 | ~~Tipo de almacenamiento del PDF~~ | Tecnico | **Resuelto (DUDA-039, 2026-08-21):** archivo/binario en Minio via tabla Archivo, sin regeneracion (ya reflejado en este documento) |

---

## OBS-TC — Tipo de Cambio en Documentos Generados

> **Nota del coordinador (2026-07-21):** `tpPedido.TipoCambioFacturacion` actualmente siempre se setea en 1, lo cual es incorrecto. Además, el TC no necesariamente pertenece al nivel de pedido: la Proforma y la Factura por Adelantado tienen su propio TC, y éste debe setearse al momento de generar cada documento para garantizar trazabilidad. El TC del documento debe ser el mismo que el utilizado en el cálculo del PDF.

**Decisión de diseño:**

El TC se setea **a nivel de documento** (no de pedido), al momento de generación:

| Documento | Tabla | Campo | Momento de set |
|-----------|-------|-------|----------------|
| Proforma (MEX/PER) | `tpProformaPedido` | `TipoCambio` decimal(18,4) **NUEVO** | Al generar el PDF (primer envío exitoso) |
| FAA Crédito (RE-FU-012) | `fccFactura` | `TipoCambio` decimal NULL | Al activar la FAA |
| FAA Prepago (RE-FU-015) | `fccFactura` | `TipoCambio` decimal(18,4) | Al activar la FAA |

**Sobre `tpPedido.TipoCambioFacturacion`:**
- Campo existente, actualmente siempre = 1 (bug legacy).
- Para Proforma y FAA: **no leer este campo para el PDF**; usar el campo `TipoCambio` del documento correspondiente.
- Si en el futuro se necesita el TC del pedido para otros fines, deberá setearse correctamente en el momento de tramitación — ese análisis está fuera del alcance de RE-FU-016.

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-001 | EmpresaDatosBancarios.IdRegion (filtrar cuentas MEX) |
| R16A-RE-FU-006 | ClienteDatosBancarios.CodigoValidador (REF.CLIENTE) |
| R16A-RE-FU-013 | Flujo Prepago con controlados (mismo PDF) |
| R16A-RE-FU-014 | Flujo Prepago sin controlados sin FAA (dispara este PDF) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
