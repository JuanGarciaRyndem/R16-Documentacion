# Impacto en BD - Tramitacion Pedidos Prepago sin Controlados con FAA
**Requisito:** R16A-RE-FU-015
**Base de Datos:** ProquifaDotNet
**Version:** 2.1 (2.0 adopta el diseño del DIS-SOL v1.0 — reemplaza tpProformaAdelanto por fccFactura; 2.1 agrega el catálogo `catFacturaEstado`, la FK `fccFactura.IdCatFacturaEstado` y la columna `fccFactura.FechaEnvio`)

---

## Resumen
Prepago sin controlados con Factura por Adelantado activada.
Genera directamente el pendiente en Factura por Adelantado al tramitar
(sin generar proforma, PDF ni correo).
NO genera pendiente en Validar Cobro al tramitar.
El pendiente Validar Cobro se genera despues, al emitirse la factura PPD.

**CAMBIO ESTRUCTURAL (adopcion DIS-SOL v1.0):** el pendiente FAA ya NO se
modela con `tpProformaAdelanto`. Se introducen tres tablas nuevas, propiedad
de `ProquifaDotNet.Finanzas`: `fccFactura` (cabecera, tabla unica compartida
con la factura final via `EsFacturaPorAdelantado`), `fccFacturaPartida`
(detalle de partidas) y `fccFacturaReferenciaBancaria` (referencias bancarias).

---

## ⛔ BLOQUEANTE — OBS-027: CatEstadoTpPedido + IdCatEstadoTpPedido en tpPedido

> **BLOQUEANTE — En espera de propuesta del cliente.**
>
> Se requiere definir el catálogo `CatEstadoTpPedido` y el campo `IdCatEstadoTpPedido` en `tpPedido` para gestionar el estatus del pedido durante el flujo de tramitación y vida del pedido. Sin esta definición no es posible implementar las tareas de BD relacionadas con la transición de estados del pedido ni la lógica de transición en Backend.
>
> **Pendiente:**
> - Propuesta del cliente con los estados necesarios para `CatEstadoTpPedido` (clave, descripción, estados terminales, transiciones permitidas).
> - Confirmación de si `IdCatEstadoTpPedido` aplica en `tpPedido`, `ppPedido`, o ambas.
>
> **Las tareas de BD y Backend relacionadas a OBS-027 están BLOQUEADAS hasta recibir esta propuesta.** Esto NO bloquea la creación de `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`, que no dependen de `CatEstadoTpPedido`.

---

## ⚠️ Hallazgo abierto — H-01 (`R16A-RE-FU-015_DIS-SOL_Revision.md`)

`fccFactura` solo modela campos fiscales de México en el bloque "Datos del receptor" (`RegimenFiscalClaveSAT`, `UsoCFDIClaveSAT`, `MetodoDePagoClaveSAT`, `FormaDePagoClaveSAT`). La Regla 14 / Criterio A5 de `R16A-RE-FU-015.md` exige que, para clientes Perú, se persistan Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago SUNAT en su lugar. **Antes de cerrar el desarrollo para Perú**, debe agregarse a `fccFactura` (o a una tabla de extensión regional) las columnas equivalentes, o documentarse explícitamente por qué no aplican. No bloquea la creación de la tabla en esta fase.

---

## Impacto en BD: CUATRO TABLAS NUEVAS + ALTER pendiente (OBS-027)

> `tpProformaAdelanto` **ya NO aplica** a este requisito — reemplazada por
> `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`.
> `tpPedido.FacturaPorAdelantado = 1`.
> Al tramitar: pendiente en FAA (`fccFactura` + detalle), generado directamente
> sin proforma previa — NO en Validar Cobro.
> El pendiente en Validar Cobro lo genera el modulo FAA cuando emite la factura PPD.
> `tpProformaPedido` YA NO aplica a este requisito.

---

## Diccionario de Datos

### Tabla: `catFacturaEstado` (catálogo NUEVO)

**Descripción:** Catálogo de estados del ciclo de vida de la Factura (`fccFactura`): generación → timbrado → envío → cobro (parcial/total) → cancelación. Sigue el patrón de catálogos del proyecto (`catTipoCFDI`, `catDocumentoFiscalCobroEstado`). Propiedad de `ProquifaDotNet.Finanzas` (Scaffold EF Core en `Finanzas.Infrastructure`).

```sql
CREATE TABLE [dbo].[catFacturaEstado](
    [IdCatFacturaEstado]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catFacturaEstado_Id]       DEFAULT (NEWID()),
    [Clave]               varchar(30)      NOT NULL,
    [Descripcion]         nvarchar(150)    NOT NULL,
    [Orden]               int              NOT NULL,   -- orden natural del ciclo de vida (UI/reportes)
    [EsTerminal]          bit              NOT NULL
        CONSTRAINT [DF_catFacturaEstado_Terminal] DEFAULT (0),
    [Activo]              bit              NOT NULL
        CONSTRAINT [DF_catFacturaEstado_Activo]   DEFAULT (1),
    [FechaRegistro]       datetime     NOT NULL
        CONSTRAINT [DF_catFacturaEstado_FechaReg] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion] datetime NOT NULL
        CONSTRAINT [DF_catFacturaEstado_FechaUltimaActualizacion] DEFAULT (GETDATE()),
    CONSTRAINT [PK_catFacturaEstado] PRIMARY KEY CLUSTERED ([IdCatFacturaEstado]),
    CONSTRAINT [UQ_catFacturaEstado_Clave] UNIQUE ([Clave])
);
```

**Seed:**

```sql
INSERT INTO [dbo].[catFacturaEstado] (Clave, Descripcion, Orden, EsTerminal) VALUES
('POR_GENERAR',    N'Factura creada, pendiente de timbrado ante PAC/SUNAT',                                1, 0),
('ERROR_TIMBRADO', N'El PAC/SUNAT rechazó el timbrado; requiere corrección y reintento (Finanzas)',        2, 0),
('GENERADA',       N'Timbrada exitosamente (CFDI/CPE vigente); pendiente de envío al cliente',             3, 0),
('ENVIADA',        N'Enviada al cliente con PDF + XML adjuntos',                                           4, 0),
('PAGADA_PARCIAL', N'Con cobros aplicados parcialmente; saldo pendiente (PPD con complementos parciales)', 5, 0),
('PAGADA',         N'Cobro asociado y aplicado en su totalidad (Validar Cobro)',                           6, 1),
('CANCELADA',      N'Cancelada ante SAT/SUNAT (CFDICancelacion / NC según normativa)',                     7, 1);
```

**Columnas:**

| Nombre             | Tipo de dato     | Descripción                                                       |
| ------------------ | ---------------- | ------------------------------------------------------------------ |
| IdCatFacturaEstado | uniqueidentifier | PK, DEFAULT NEWID()                                                 |
| Clave              | varchar(30)      | Clave programática (UQ)                                             |
| Descripcion        | nvarchar(150)    | Descripción legible                                                 |
| Orden              | int              | Orden natural del ciclo de vida (UI/reportes)                       |
| EsTerminal         | bit              | 1 = sin transiciones posteriores (PAGADA, CANCELADA)                |
| Activo             | bit              | Borrado lógico, DEFAULT 1                                           |
| FechaRegistro      | datetime     | Alta del registro, DEFAULT GETDATE()                         |
| FechaUltimaActualizacion      | datetime     | Última modificación, DEFAULT GETDATE()                       |

**Relaciones:**

| Relación | Tipo |
|---|---|
| `fccFactura.IdCatFacturaEstado` → `catFacturaEstado.IdCatFacturaEstado` | N:1 |

**Índices:**

| Nombre | Columnas | Tipo |
|---|---|---|
| PK_catFacturaEstado | IdCatFacturaEstado | Clustered (PK) |
| UQ_catFacturaEstado_Clave | Clave | Unique nonclustered |

**Consideraciones especiales:**
- Transiciones válidas: POR_GENERAR → GENERADA \| ERROR_TIMBRADO; ERROR_TIMBRADO → GENERADA (reintento de Finanzas — Timbrado no reintenta, RE-FU-018); GENERADA → ENVIADA \| CANCELADA; ENVIADA → PAGADA_PARCIAL \| PAGADA \| CANCELADA; PAGADA_PARCIAL → PAGADA \| CANCELADA.
- PAGADA y CANCELADA son estados terminales (`EsTerminal = 1`).
- La inmutabilidad fiscal aplica desde GENERADA: la corrección posterior al timbrado es solo vía el módulo Notas de Crédito (RE-FU-032/033).
- No confundir con `catDocumentoFiscalCobroEstado` (estado de línea del wizard Validar Cobro Paso 3, RE-FU-028), con `CFDIGenerada.Estado` (estado técnico del timbrado) ni con el `EstadoFAA` calculado de `vfccFactura` (estado del pendiente FAA).
- La transición a PAGADA_PARCIAL/PAGADA la ejecuta Validar Cobro (RE-FU-026/027/028/029) al aplicar cobros; la transición a CANCELADA la ejecuta el flujo de cancelación (RE-FU-032, `POST /api/v1/stamp/cancel`).

---

### Tabla: `fccFactura`

**Descripción:** Cabecera única para la Factura por Adelantado (FAA) y para la factura final, diferenciadas por `EsFacturaPorAdelantado`. Los datos del receptor se fijan como snapshot de `DatosFacturacionCliente` al crear la FAA; los campos fiscales del timbrado SAT quedan `NULL` en la FAA y se llenan al timbrar la factura final (RT-10). Propiedad de `ProquifaDotNet.Finanzas` (Scaffold EF Core en `Finanzas.Infrastructure`).

**Columnas:**

| Nombre                                                  | Tipo de dato       | Descripción                                                              |
| ------------------------------------------------------- | ------------------ | ------------------------------------------------------------------------ |
| IdFccFactura                                            | uniqueidentifier   | PK                                                                       |
| IdTPPedido                                              | uniqueidentifier   | FK → `tpPedido.IdTPPedido`, requerido                                    |
| IdTPProformaPedido                                      | uniqueidentifier NULL | FK → `tpProformaPedido.IdTPProformaPedido` — poblado únicamente cuando la FAA se origina desde una Confirmación de Pedido ya emitida (flujo Crédito, RE-FU-012); `NULL` en el flujo Prepago (RE-FU-015), que no genera proforma (unifica el antiguo `tpProformaAdelantoProformaPedido`) |
| EsFacturaPorAdelantado                                  | bit                | 1 = FAA, 0 = factura final (bandera diferenciadora, RT-10)               |
| IdCatFacturaEstado                                      | uniqueidentifier   | FK → `catFacturaEstado.IdCatFacturaEstado`, requerido — estado del ciclo de vida de la factura; se asigna POR_GENERAR al crear el registro (la app resuelve el Id por `Clave`) |
| Enviada                                                 | bit                | 0 = no enviada / 1 = enviada al cliente con PDF+XML — determina, junto con `IdCFDIGenerada`, el estado calculado `EstadoFAA` en `vfccFactura` (equivalente a `tpProformaAdelanto.Enviada`, migrado de RE-FU-019) |
| FechaEnvio                                              | datetime NULL  | Fecha y hora (UTC) del envío de la factura al cliente — se asigna con GETDATE() en el mismo UPDATE que `Enviada = 1` / `IdCatFacturaEstado = ENVIADA`; `NULL` mientras no se envía |
| IdCliente                                               | uniqueidentifier   | FK, ← `tpPedido.IdCliente`                                               |
| IdEmpresa                                               | uniqueidentifier   | FK, empresa emisora (Proquifa)                                           |
| FolioPedidoInterno                                      | varchar            | ← `tpPedido.FolioPedidoInterno`                                          |
| IdCatCondicionesDePago                                  | uniqueidentifier   | FK, ← pedido                                                             |
| IdCatMoneda                                             | uniqueidentifier   | FK, moneda de facturación (los montos se expresan en esta moneda, RT-09) |
| TipoCambio                                              | decimal(18,4) NULL | TC al momento de generación de la FAA — seteado independientemente al activar la FAA. **No heredar `tpPedido.TipoCambioFacturacion`** (siempre = 1, bug legacy — OBS-TC, ver RE-FU-016_BD.md). ~~TC facturación↔tramitación, solo si difieren (NULL/1 si coinciden)~~ — descripción anterior incorrecta: el valor siempre debe reflejar el TC real vigente al momento de generación. |
| FactorConversionUSD                                     | decimal(18,6)      | Heredado de `tpPedido` — para monto en USD en reportes                   |
| SubTotal                                                | decimal(18,2)      | —                                                                        |
| IVA                                                     | decimal(18,2)      | Impuestos federales                                                      |
| MontoTotal                                              | decimal(18,2)      | Total                                                                    |
| MontoTotalLetras                                        | varchar            | Total en letra                                                           |
| RfcReceptor                                             | varchar            | Snapshot `DatosFacturacionCliente` — RFC del cliente                     |
| RazonSocialReceptor                                     | varchar            | Snapshot — Nombre/Razón Social                                           |
| CodigoPostalReceptor                                    | varchar            | Snapshot — CP / domicilio fiscal receptor                                |
| RegimenFiscalClaveSAT / RegimenFiscalLeyendaSAT         | varchar            | Catálogo `c_RegimenFiscal` (México)                                      |
| UsoCFDIClaveSAT / UsoCFDILeyendaSAT                     | varchar            | Catálogo `c_UsoCFDI` (México) — ⚠️ ver H-01, sin equivalente Perú        |
| MetodoDePagoClaveSAT / MetodoDePagoLeyendaSAT           | varchar            | Catálogo `c_MetodoPago` (México) — ⚠️ ver H-01, sin equivalente Perú     |
| FormaDePagoClaveSAT / FormaDePagoLeyendaSAT             | varchar            | Catálogo `c_FormaPago` (México)                                          |
| IdCFDIGenerada                                          | uniqueidentifier NULL | FK → `CFDIGenerada.IdCFDIGenerada` (Finanzas) — `NULL` mientras la factura no se ha timbrado (FAA pendiente); se llena al emitir/timbrar la factura final (RE-FU-018/019/020). Reemplaza el corte de `tpProformaAdelanto.IdCFDIGenerada` — **no se duplican** Serie/Folio/FolioFiscal/Version/TipoDeComprobante/FechaCertificacion en `fccFactura`: esos datos se leen de `CFDIGenerada` vía este FK (single source of truth, mismo criterio aplicado en RE-FU-018/019/020/021/022) |
| Activo                                                  | bit                | Control                                                                  |
| FechaRegistro                                           | datetime           | Control                                                                  |
| FechaUltimaActualizacion                                | datetime           | Control                                                                  |

**Relaciones:**

| Relación | Tipo |
|---|---|
| `fccFactura.IdTPPedido` → `tpPedido.IdTPPedido` | N:1 |
| `fccFactura.IdTPProformaPedido` → `tpProformaPedido.IdTPProformaPedido` | N:1, opcional (solo origen Crédito) |
| `fccFactura.IdCFDIGenerada` → `CFDIGenerada.IdCFDIGenerada` | N:1, opcional (NULL hasta timbrar) |
| `fccFactura.IdCatFacturaEstado` → `catFacturaEstado.IdCatFacturaEstado` | N:1, requerido |
| `fccFacturaPartida.IdFccFactura` → `fccFactura.IdFccFactura` | 1:N |
| `fccFacturaReferenciaBancaria.IdFccFactura` → `fccFactura.IdFccFactura` | 1:N |
| `fccPagoFacturaAdelanto.IdFccFactura` → `fccFactura.IdFccFactura` | N:1 (RE-FU-026/027/028/029/030 — reemplaza `fccPagoFacturaAdelanto.IdTPProformaAdelanto`) |

**Índices:**

| Nombre | Columnas | Tipo |
|---|---|---|
| PK_fccFactura | IdFccFactura | Clustered (PK) |
| IX_fccFactura_IdTPPedido | IdTPPedido | Nonclustered (FK, búsqueda por pedido) |
| IX_fccFactura_IdTPProformaPedido | IdTPProformaPedido | Nonclustered (FK, búsqueda por proforma — origen Crédito) |
| IX_fccFactura_IdCFDIGenerada | IdCFDIGenerada | Nonclustered (FK, búsqueda por CFDI timbrado) |
| IX_fccFactura_FolioPedidoInterno | FolioPedidoInterno | Nonclustered (búsqueda por folio) |
| IX_fccFactura_IdCatFacturaEstado | IdCatFacturaEstado | Nonclustered (FK, filtrado por estado) |

**Consideraciones especiales:**
- Un `fccFactura` debe tener exactamente un `IdTPPedido` válido.
- `EsFacturaPorAdelantado = 1` en el flujo RE-015 (al tramitar) y en el flujo RE-012 (Crédito, con `IdTPProformaPedido` poblado). El módulo FAA (RE-018/019/020) actualiza este registro a `EsFacturaPorAdelantado = 0` y puebla `IdCFDIGenerada` al emitir la factura final — no se crea un segundo registro (RT-10).
- `IdCFDIGenerada` debe permanecer `NULL` mientras `EsFacturaPorAdelantado = 1` y no se haya timbrado (`EstadoFAA = 'PendienteGenerar'` en `vfccFactura`).
- `IdTPProformaPedido` es `NULL` para pedidos Prepago (RE-FU-015, que no genera proforma) y está poblado para pedidos Crédito (RE-FU-012, cuya proforma/Confirmación de Pedido se genera en paralelo a `fccFactura` dentro de la misma transacción de tramitación).
- ⚠️ H-01 abierto: sin columnas para Tipo de Operación / Condición de Pago SUNAT (Perú).
- `IdCatFacturaEstado` sigue el ciclo de `catFacturaEstado` (ver catálogo arriba): POR_GENERAR al crear; GENERADA al timbrar (junto con `IdCFDIGenerada`); ENVIADA al enviar (junto con `Enviada = 1` y `FechaEnvio = GETDATE()`); PAGADA_PARCIAL/PAGADA desde Validar Cobro; CANCELADA desde el flujo de cancelación. Convive con `EstadoFAA` (calculado en `vfccFactura`, específico del pendiente FAA) sin sustituirlo.
- **Tabla única para el pendiente FAA, tanto en el origen Prepago (RE-015) como Crédito (RE-012)** — reemplaza `tpProformaAdelanto` en ambos flujos. Ver vista `vfccFactura` para el listado/estado calculado que antes ofrecía `vtpProformaAdelanto`.

---

### Vista: `vfccFactura` (reemplaza `vtpProformaAdelanto`)

**Descripción:** Vista de lectura sobre `fccFactura` (filtrada `WHERE EsFacturaPorAdelantado = 1`) que calcula el estado del pendiente FAA (`EstadoFAA`) y expone los datos de CFDI (vía `IdCFDIGenerada`) y del pedido/proforma de origen. Reemplaza `vtpProformaAdelanto` (creada originalmente en RE-FU-019 sobre `tpProformaAdelanto`). Consumida por RE-FU-018 (listado), RE-FU-019/020 (generación/envío) y RE-FU-026/027/028/029/030 (asociación de cobro y Complemento de Pago).

```sql
-- Reemplaza CREATE VIEW dbo.vtpProformaAdelanto (RE-FU-019) — ahora sobre fccFactura
CREATE VIEW dbo.vfccFactura
AS
SELECT
    f.IdFccFactura,
    f.IdTPPedido,
    f.IdTPProformaPedido,
    f.FolioPedidoInterno,
    f.SubTotal,
    f.IVA,
    f.MontoTotal,
    f.IdCatMoneda,
    f.IdCliente,
    c.Nombre                    AS ClienteNombre,
    f.RazonSocialReceptor       AS ClienteRazonSocial,
    f.RfcReceptor               AS ClienteRFC,
    f.IdEmpresa,
    e.Prefijo                   AS EmpresaPrefijo,
    e.Alias                     AS EmpresaAlias,
    f.IdCatFacturaEstado,
    fe.Clave                    AS FacturaEstadoClave,
    f.IdCFDIGenerada,
    cg.Folio                    AS FolioFactura,
    cg.Serie                    AS SerieFactura,
    cg.FechaEmision             AS FechaEmisionFactura,
    cg.Total                    AS TotalFactura,
    f.Enviada,
    f.FechaEnvio,
    f.FechaRegistro,
    f.FechaUltimaActualizacion,
    f.Activo,
    -- Estado calculado (mismo criterio que vtpProformaAdelanto.EstadoFAA)
    CASE
        WHEN f.IdCFDIGenerada IS NULL THEN 'PendienteGenerar'
        WHEN f.IdCFDIGenerada IS NOT NULL AND f.Enviada = 0 THEN 'PendienteEnviar'
        ELSE 'Completada'
    END                         AS EstadoFAA,
    tp.FechaTramitacion,
    tp.FacturaPorAdelantado,
    tp.IdRegion,
    r.Nombre                    AS Region,
    r.ClaveISO                  AS RegionClave,
    f.IdCatCondicionesDePago,
    cdp.CondicionesDePago,
    cdp.SinCredito              AS EsPrepago
FROM dbo.fccFactura f
LEFT JOIN dbo.Cliente c                     ON f.IdCliente = c.IdCliente
LEFT JOIN dbo.Empresa e                     ON f.IdEmpresa = e.IdEmpresa
LEFT JOIN dbo.catFacturaEstado fe           ON f.IdCatFacturaEstado = fe.IdCatFacturaEstado
LEFT JOIN dbo.CFDIGenerada cg               ON f.IdCFDIGenerada = cg.IdCFDIGenerada
LEFT JOIN dbo.tpPedido tp                   ON f.IdTPPedido = tp.IdTPPedido
LEFT JOIN dbo.Region r                      ON tp.IdRegion = r.IdRegion
LEFT JOIN dbo.catCondicionesDePago cdp      ON f.IdCatCondicionesDePago = cdp.IdCatCondicionesDePago
WHERE f.EsFacturaPorAdelantado = 1;
```

**Diferencia clave vs. `vtpProformaAdelanto`:** ya no hace falta la cadena `fccPagoFacturaAdelanto → tpProformaPedido → tpPedidoProformaPedido → tpPedido` para llegar al pedido — `fccFactura.IdTPPedido` es FK directa. Los datos del receptor (RFC, Razón Social) tampoco requieren JOIN a `DatosFacturacionCliente`: ya están fijados como snapshot en `fccFactura` (columnas `RfcReceptor`/`RazonSocialReceptor`).

**Columnas (equivalencia con `vtpProformaAdelanto`):**

| Columna | Origen | Equivalente en `vtpProformaAdelanto` |
|---|---|---|
| IdFccFactura | fccFactura | IdTPProformaAdelanto |
| IdTPProformaPedido | fccFactura | (vía `fccPagoFacturaAdelanto` → `tpProformaPedido`, indirecto) |
| ClienteRazonSocial / ClienteRFC | fccFactura (snapshot) | DatosFacturacionCliente (JOIN) |
| IdCFDIGenerada, FolioFactura, SerieFactura, FechaEmisionFactura, TotalFactura | fccFactura → CFDIGenerada (JOIN) | Igual |
| Enviada | fccFactura | tpProformaAdelanto.Enviada |
| FechaEnvio | fccFactura | (nuevo — sin equivalente; columna aditiva v2.1) |
| EstadoFAA | Calculado (misma fórmula) | Igual |
| IdCatFacturaEstado / FacturaEstadoClave | fccFactura → catFacturaEstado (JOIN) | (nuevo — sin equivalente; columnas aditivas v2.1) |
| IdTPPedido, FolioPedidoInterno, Region, RegionClave, CondicionesDePago, EsPrepago | fccFactura → tpPedido (JOIN directo) | Igual (antes vía cadena de 3 JOINs) |

---

### Tabla: `fccFacturaPartida`

**Descripción:** Detalle de partidas (1:N) de `fccFactura`, snapshot de las partidas del pedido al momento de generar el pendiente FAA. Propiedad de `ProquifaDotNet.Finanzas`.

**Columnas:**

| Nombre                                        | Tipo de dato          | Descripción                                  |
| --------------------------------------------- | --------------------- | -------------------------------------------- |
| IdFccFacturaPartida                           | uniqueidentifier      | PK                                           |
| IdFccFactura                                  | uniqueidentifier      | FK → `fccFactura.IdFccFactura`, requerido    |
| IdTPPedidoPartida                             | uniqueidentifier NULL | FK, partida origen del pedido (trazabilidad) |
| Numero                                        | int                   | Línea (#)                                    |
| Descripcion                                   | varchar               | Descripción del producto                     |
| NoIdentificacion                              | varchar               | Catálogo / Nº ID                             |
| Lote                                          | varchar NULL          | —                                            |
| Caducidad                                     | varchar NULL          | —                                            |
| ClaveProductoServicioSAT                      | varchar               | Catálogo `c_ClaveProdServ`                   |
| ClaveUnidadSAT                                | varchar               | Catálogo `c_ClaveUnidad`                     |
| UnidadMedida                                  | varchar               | —                                            |
| Cantidad                                      | decimal(18,2)         | —                                            |
| ValorUnitario                                 | decimal(18,2)         | —                                            |
| Importe                                       | decimal(18,2)         | —                                            |
| Pedimento                                     | varchar NULL          | —                                            |
| BaseImpuesto                                  | decimal(18,2)         | Base del traslado                            |
| TipoImpuestoClaveSAT / TipoImpuestoLeyendaSAT | varchar               | Catálogo `c_Impuesto`                        |
| TipoFactorSAT                                 | varchar               | Catálogo `c_TipoFactor` (campo único)        |
| TasaCuota                                     | decimal(18,6)         | —                                            |
| ImporteImpuesto                               | decimal(18,2)         | —                                            |
| Activo                                        | bit                   | Control                                      |
| FechaRegistro                                 | datetime              | Control                                      |
| FechaUltimaActualizacion                      | datetime              | Control                                      |

**Relaciones:**

| Relación | Tipo |
|---|---|
| `fccFacturaPartida.IdFccFactura` → `fccFactura.IdFccFactura` | N:1 |
| `fccFacturaPartida.IdTPPedidoPartida` → `tpPedidoPartida` (o tabla de partidas equivalente) | N:1, opcional (trazabilidad) |

**Índices:**

| Nombre | Columnas | Tipo |
|---|---|---|
| PK_fccFacturaPartida | IdFccFacturaPartida | Clustered (PK) |
| IX_fccFacturaPartida_IdFccFactura | IdFccFactura | Nonclustered (FK) |

**Consideraciones especiales:**
- Debe existir al menos una partida por cada `fccFactura` generado.
- El INSERT de partidas es atómico junto con la cabecera y las referencias bancarias (RT-04).
- `ClaveProductoServicioSAT` reutiliza el catálogo poblado en `catClaveProdServSAT` (RE-FU-021).

---

### Tabla: `fccFacturaReferenciaBancaria`

**Descripción:** Detalle de referencias bancarias (1:N) de `fccFactura`. El CFDI muestra siempre las dos cuentas (M.N. y DLS) del grupo PROQUIFA México (RE-FU-016, Criterio E1). El dato proviene de `EmpresaDatosBancarios`/`DatosBancarios`/`catBanco`; la `ReferenciaCliente` se arma con el Código Validador (RE-FU-006). Propiedad de `ProquifaDotNet.Finanzas`.

**Columnas:**

| Nombre                         | Tipo de dato     | Descripción                                     |
| ------------------------------ | ---------------- | ----------------------------------------------- |
| IdFccFacturaReferenciaBancaria | uniqueidentifier | PK                                              |
| IdFccFactura                   | uniqueidentifier | FK → `fccFactura.IdFccFactura`, requerido       |
| IdCatMoneda                    | uniqueidentifier | FK, moneda de la cuenta (MXN / USD)             |
| Banco                          | varchar          | de `catBanco`                                   |
| NumeroCuenta                   | varchar          | —                                               |
| Clabe                          | varchar          | CLABE interbancaria                             |
| Sucursal                       | varchar          | —                                               |
| ReferenciaCliente              | varchar          | REF. del cliente — Código Validador (RE-FU-006) |
| Activo                         | bit              | Control                                         |
| FechaRegistro                  | datetime         | Control                                         |
| FechaUltimaActualizacion       | datetime         | Control                                         |

**Relaciones:**

| Relación | Tipo |
|---|---|
| `fccFacturaReferenciaBancaria.IdFccFactura` → `fccFactura.IdFccFactura` | N:1 |
| `fccFacturaReferenciaBancaria.IdCatMoneda` → `catMoneda` | N:1 |

**Índices:**

| Nombre | Columnas | Tipo |
|---|---|---|
| PK_fccFacturaReferenciaBancaria | IdFccFacturaReferenciaBancaria | Clustered (PK) |
| IX_fccFacturaReferenciaBancaria_IdFccFactura | IdFccFactura | Nonclustered (FK) |

**Consideraciones especiales:**
- Deben insertarse ambas cuentas del grupo PROQUIFA México (M.N. y DLS) por cada `fccFactura` (RE-FU-016, Criterio E1).
- `ReferenciaCliente` se reconstruye a partir del Código Validador (RE-FU-006) — ver caso crítico en Tareas.

---

## ALTER pendiente en `tpPedido` (⛔ OBS-027)

| Campo | Tipo | Descripción |
|---|---|---|
| IdCatEstadoTpPedido | uniqueidentifier NULL, FK → `CatEstadoTpPedido` | Estado del pedido — cierra el pendiente Tramitar Pedido al asignarse (RT-05, ⛔ OBS-027, compartido con RE-013) |

> Bloqueado hasta recibir la propuesta del cliente sobre `CatEstadoTpPedido` (clave, descripción, estados terminales, transiciones permitidas).

---

## Tablas Involucradas

| Tabla                            | Rol                                                                   | Estado                                                              |
| -------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------- |
| tpPedido                         | Cabecera - FacturaPorAdelantado=1, Prepago                            | Existente - sin cambios estructurales (pendiente ALTER por OBS-027) |
| **catFacturaEstado**             | Catálogo de estados del ciclo de vida de la Factura                   | **NUEVA**                                                           |
| **fccFactura**                   | Cabecera del pendiente FAA (y de la factura final)                    | **NUEVA**                                                           |
| **fccFacturaPartida**            | Detalle de partidas del pendiente FAA                                 | **NUEVA**                                                           |
| **fccFacturaReferenciaBancaria** | Referencias bancarias del pendiente FAA                               | **NUEVA**                                                           |
| DatosFacturacionCliente          | Datos fiscales (solo lectura, se fijan como snapshot en `fccFactura`) | Existente - sin cambios                                             |

> `tpProformaAdelanto` **ya no aplica** a este requisito (reemplazada por `fccFactura`).
> `tpProformaPedido`, `tpPedidoProformaPedido`, `tpProformaPartidaPedido` y `tpPedidoCorreoEnviado` **ya no aplican** a este requisito — no se genera proforma ni correo en este flujo.

---

## Diferencia clave vs RE-FU-014 (sin FAA)

| Aspecto | RE-FU-014 (sin FAA) | RE-FU-015 (con FAA) |
|---------|--------------------|--------------------|
| tpPedido.FacturaPorAdelantado | = 0 | = 1 |
| Proforma/PDF/correo generado en Tramitar Pedido | SI | **NO** |
| Pendiente generado al tramitar | Validar Cobro | **Factura por Adelantado (`fccFactura` + detalle)** |
| Pendiente Validar Cobro | SI (inmediato) | NO (posterior, al emitir factura PPD) |
| Documento a cobrar | Proforma | Factura PPD |

---

## Flujo en BD al Tramitar Prepago con FAA

    1. tpPedido.FacturaPorAdelantado = 1
    2. tpPedido.FolioPedidoInterno = siguiente folio (generado y commiteado
       por ProquifaDotNet antes de llamar a Finanzas)
    3. Fija datos de facturacion del catalogo vigente del cliente
    4. ProquifaDotNet.Finanzas INSERT atomico:
       - fccFactura (EsFacturaPorAdelantado=1, IdCatFacturaEstado=POR_GENERAR,
         campos fiscales timbrado NULL)
       - fccFacturaPartida (una por partida del pedido)
       - fccFacturaReferenciaBancaria (cuentas M.N./DLS + ReferenciaCliente)
    5. NO se genera tpProformaPedido, PDF ni correo en este flujo
    6. NO INSERT pendiente Validar Cobro
    7. Asigna tpPedido.IdCatEstadoTpPedido -> cierra pendiente Tramitar Pedido
       (bloqueado hasta resolver OBS-027)

   === POSTERIORMENTE (fuera scope este requisito) ===
   8. Modulo FAA emite factura PPD -> INSERT CFDIGenerada (Finanzas) ->
      UPDATE fccFactura SET EsFacturaPorAdelantado=0, IdCFDIGenerada=@Id,
      IdCatFacturaEstado=GENERADA (si el PAC rechaza: ERROR_TIMBRADO, reintento de Finanzas)
      (Serie/Folio/FolioFiscal/Version/TipoDeComprobante/FechaCertificacion
      se leen de CFDIGenerada via este FK, no se duplican en fccFactura)
   8b. Al enviar la factura -> UPDATE fccFactura SET Enviada=1, FechaEnvio=GETDATE(),
       IdCatFacturaEstado=ENVIADA
   9. FAA genera pendiente Validar Cobro
   10. Validar Cobro aplica cobros -> IdCatFacturaEstado=PAGADA_PARCIAL o PAGADA;
       cancelacion (RE-032) -> IdCatFacturaEstado=CANCELADA

---

## Orden de Ejecución de Scripts

| #   | Script                                                                                              | Bloqueante                   |
| --- | --------------------------------------------------------------------------------------------------- | ---------------------------- |
| 1   | CREATE TABLE `CFDIGenerada` (si no existe aún — base, RE-FU-019)                                    | No (prerrequisito de 3 y 6)  |
| 2   | CREATE TABLE `catFacturaEstado` + seed (7 estados)                                                  | No (prerrequisito de 3)      |
| 3   | CREATE TABLE `fccFactura` (incluye FK `IdCFDIGenerada`, `IdTPProformaPedido`, `IdCatFacturaEstado`) | No (depende de 1 y 2)        |
| 4   | CREATE TABLE `fccFacturaPartida` (FK → `fccFactura`)                                                | No (depende de 3)            |
| 5   | CREATE TABLE `fccFacturaReferenciaBancaria` (FK → `fccFactura`)                                     | No (depende de 3)            |
| 6   | CREATE VIEW `vfccFactura` (reemplaza `vtpProformaAdelanto`, incluye JOIN a `catFacturaEstado`)      | No (depende de 1, 2, 3)      |
| 7   | CREATE TABLE `CatEstadoTpPedido` + seed                                                             | ⛔ Sí — OBS-027               |
| 8   | ALTER TABLE `tpPedido` ADD `IdCatEstadoTpPedido` (FK → `CatEstadoTpPedido`)                         | ⛔ Sí — OBS-027, depende de 7 |

> **Nota de migración (propagada a RE-FU-012/018/019/020/026/027/028/029/030):** estos requisitos consumían `tpProformaAdelanto` + `vtpProformaAdelanto`. A partir de la adopción de este esquema, deben apuntar a `fccFactura` + `vfccFactura`. Ver sección "Migración de `tpProformaAdelanto`" más abajo.

---

## Cambios de Comportamiento R16 (sin cambio en BD)

| Cambio | Antes | Ahora (R16) |
|--------|-------|-------------|
| Codigo autorizacion para FAA | Requeria codigo | Activacion directa |
| Boton Editar Datos | Visible | Oculto para Prepago siempre |
| Generacion de proforma/PDF/correo en Tramitar Pedido (con FAA) | N/A | Ya no aplica — el pendiente FAA se genera directo |
| Estructura del pendiente FAA | `tpProformaAdelanto` | `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria` |

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | Vinculacion FAA -> Validar Cobro posterior | Confirmar logica del modulo FAA (RE-018/019/020) sobre UPDATE de `fccFactura` |
| 2 | H-01 — Campos fiscales de Perú en `fccFactura` | Agregar columnas equivalentes a Tipo de Operación / Condición de Pago SUNAT, o documentar por qué no aplican |
| 3 | Documento disponible para TaskScheduler/Legacy | Confirmar si el job de Venta Digital puede operar sin ningun PDF generado en este flujo |
| 4 | Catalogo de estatus del pedido (OBS-027 / Criterio D5) | Pendiente propuesta del cliente |

> Ya no aplican: "Folio proforma lineal global" y "Politica de folio si ESAC cancela previsualizacion" — este flujo no genera proforma.

---

## Migración de `tpProformaAdelanto` (06/07/2026) — impacto en todo el proyecto

`fccFactura` (+ `fccFacturaPartida` + `fccFacturaReferenciaBancaria` + vista `vfccFactura`) reemplaza a `tpProformaAdelanto` (+ `tpProformaAdelantoProformaPedido` + `vtpProformaAdelanto`) como **tabla única del pendiente de Factura por Adelantado, tanto para el origen Prepago (este requisito) como Crédito (RE-FU-012)**. Se agregaron a `fccFactura` las columnas `IdTPProformaPedido` (nullable, origen Crédito) e `IdCFDIGenerada` (FK a `CFDIGenerada`) específicamente para soportar esta unificación — no formaban parte del diseño original del DIS-SOL v1.0 de este requisito, pero son necesarias para que RE-FU-012/018/019/020/026/027/028/029/030 (que consumían `tpProformaAdelanto`) puedan migrar sin perder funcionalidad.

**Tabla de equivalencias:**

| Objeto legacy | Objeto nuevo | Requisitos que lo consumen |
|---|---|---|
| `tpProformaAdelanto` | `fccFactura` (WHERE `EsFacturaPorAdelantado=1`) | RE-FU-012, 018, 019, 020, 026, 027, 028, 029, 030 |
| `tpProformaAdelanto.IdTPProformaAdelanto` (PK) | `fccFactura.IdFccFactura` (PK) | Todos los anteriores |
| `tpProformaAdelanto.IdCFDIGenerada` | `fccFactura.IdCFDIGenerada` | RE-FU-019, 020, 026, 027, 028, 029, 030 |
| `tpProformaAdelanto.Enviada` | `fccFactura.Enviada` (+ `FechaEnvio`, v2.1) | RE-FU-019, 020 |
| `tpProformaAdelantoProformaPedido` (tabla puente Crédito) | `fccFactura.IdTPProformaPedido` (columna directa) | RE-FU-012, 018 |
| `vtpProformaAdelanto` (vista, RE-FU-019) | `vfccFactura` (vista, este requisito) | RE-FU-018, 019, 020 |
| `fccPagoFacturaAdelanto.IdTPProformaAdelanto` | `fccPagoFacturaAdelanto.IdFccFactura` | RE-FU-026, 027, 028, 029 |

**Diferencia estructural de fondo:** `tpProformaAdelanto` no tenía FK directa a `tpPedido` (se vinculaba de forma ambigua vía `IdCliente + NumeroOrdenDeCompra`, gap sin resolver documentado en `R16A-RE-FU-012_BD.md`). `fccFactura.IdTPPedido` es FK directa y obligatoria, eliminando esa ambigüedad para ambos orígenes (Prepago y Crédito).

Ver el detalle de la migración en cada requisito afectado (`_BD.md`/`-Back.md`/`-Tareas.md` de RE-FU-012, 018, 019, 020, 026, 027, 028, 029, 030).

---

## Dependencias

| Requisito              | Relacion                                                                                                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R16A-RE-FU-006         | Referencia bancaria del documento fiscal (Codigo Validador) → `fccFacturaReferenciaBancaria.ReferenciaCliente`                                                            |
| R16A-RE-FU-012         | Variante Crédito de FAA — migrada a `fccFactura` con `IdTPProformaPedido` poblado (ver sección de migración)                                                              |
| R16A-RE-FU-013         | Arquitectura orquestador→Finanzas ya validada; folio interno                                                                                                              |
| R16A-RE-FU-014         | Flujo base Prepago sin FAA (este agrega FAA)                                                                                                                              |
| R16A-RE-FU-016         | Criterio E1 — dos cuentas bancarias del grupo PROQUIFA México                                                                                                             |
| R16A-RE-FU-018/019/020 | Módulo FAA — consulta `vfccFactura`, actualiza `fccFactura` a factura final (`EsFacturaPorAdelantado=0`, `IdCFDIGenerada`, `IdCatFacturaEstado`, `Enviada`, `FechaEnvio`) |
| R16A-RE-FU-021         | `catClaveProdServSAT` reutilizado por `fccFacturaPartida.ClaveProductoServicioSAT`                                                                                        |
| R16A-RE-FU-026/027     | Asociación de cobro — `fccPagoFacturaAdelanto.IdFccFactura`; transiciones PAGADA / PAGADA_PARCIAL                                                                         |
| R16A-RE-FU-028/029/030 | Complemento de Pago — JOIN `fccPagoFacturaAdelanto → fccFactura → CFDIGenerada` para `CFDIRelacionados`                                                                   |
| R16A-RE-FU-032         | Cancelación de factura origen — transición CANCELADA                                                                                                                      |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
