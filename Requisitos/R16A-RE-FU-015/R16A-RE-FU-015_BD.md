# Impacto en BD - Tramitacion Pedidos Prepago sin Controlados con FAA
**Requisito:** R16A-RE-FU-015
**Base de Datos:** ProquifaDotNet
**Version:** 2.2 (2.0 adopta el diseño del DIS-SOL v1.0 — reemplaza tpProformaAdelanto por fccFactura; 2.1 agrega el catálogo `catFacturaEstado`, la FK `fccFactura.IdCatFacturaEstado` y la columna `fccFactura.FechaEnvio`; 2.2 resuelve OBS-027 — adopta `catEstadoPedido` extendido + `catMotivoCancelacion` propuestos por el cliente y agrega la tabla `PedidoEstadoActual`; ya NO se crea `CatEstadoTpPedido` ni se altera `tpPedido`)

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

## ✅ OBS-027 RESUELTO — catálogo `catEstadoPedido` + tabla `PedidoEstadoActual`

> **Resuelto (11/08/2026, actualizado 12/08/2026)** — propuesta del cliente: `Analisis/Estados de Pedidos/catEstadoPedido — Estados Propuestos.md` (17 estados, catálogo `catMotivoCancelacion`, matriz de transiciones).
>
> **Decisión de diseño de este documento (adopción del catálogo):**
> - El catálogo `dbo.catEstadoPedido` **ya existe** en BD (hoy vacío, sin FK) y se extiende con `Orden`, `EsTerminal`, `Aplicativo`, `AliasOperativo`, más el seed de los 17 estados — ver catálogo propuesto, sección 8.
> - Se crea el catálogo nuevo `dbo.catMotivoCancelacion` (5 motivos) — ver catálogo propuesto, sección 8.
> - **Confirmación del ámbito (Observación #4 del catálogo propuesto):** en vez de agregar `IdCatEstadoPedido` directamente en `tpPedido` y/o `ppPedido` (opciones A/B planteadas por el catálogo), este documento centraliza el estatus del pedido en una tabla nueva, **`PedidoEstadoActual`**, con FK nullable hacia `pcPromesaDeCompra`, `ppPedido` y `tpPedido` según la etapa del flujo. Esto evita duplicar la FK en varias tablas y da una única fuente de verdad consultable independientemente de en qué tabla vive el pedido en un momento dado.
> - **Ya NO aplica** el `ALTER TABLE tpPedido ADD IdCatEstadoTpPedido` documentado en versiones anteriores de este archivo, ni la creación de un catálogo `CatEstadoTpPedido` propio de este requisito.
>
> **Las tareas T7 (BD) y T8 (Back) quedan DESBLOQUEADAS.**
>
> **Fuera de alcance de este ajuste:** la lógica de cancelación (asignación de `IdCatMotivoCancelacion`) no se implementa aquí — la columna existe en `PedidoEstadoActual` para cuando ese requisito se desarrolle.

---

## ⚠️ Hallazgo abierto — H-01 (`R16A-RE-FU-015_DIS-SOL_Revision.md`)

`fccFactura` solo modela campos fiscales de México en el bloque "Datos del receptor" (`RegimenFiscalClaveSAT`, `UsoCFDIClaveSAT`, `MetodoDePagoClaveSAT`, `FormaDePagoClaveSAT`). La Regla 14 / Criterio A5 de `R16A-RE-FU-015.md` exige que, para clientes Perú, se persistan Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago SUNAT en su lugar. **Antes de cerrar el desarrollo para Perú**, debe agregarse a `fccFactura` (o a una tabla de extensión regional) las columnas equivalentes, o documentarse explícitamente por qué no aplican. No bloquea la creación de la tabla en esta fase.

---

## Impacto en BD: CUATRO TABLAS NUEVAS + catálogo/tabla de estatus (OBS-027 resuelto)

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
('porgenerar',    N'Factura creada, pendiente de timbrado ante PAC/SUNAT',                                1, 0),
('errortimbrado', N'El PAC/SUNAT rechazó el timbrado; requiere corrección y reintento (Finanzas)',        2, 0),
('generada',       N'Timbrada exitosamente (CFDI/CPE vigente); pendiente de envío al cliente',             3, 0),
('enviada',        N'Enviada al cliente con PDF + XML adjuntos',                                           4, 0),
('pagadaparcial', N'Con cobros aplicados parcialmente; saldo pendiente (PPD con complementos parciales)', 5, 0),
('pagada',         N'Cobro asociado y aplicado en su totalidad (Validar Cobro)',                           6, 1),
('cancelada',      N'Cancelada ante SAT/SUNAT (CFDICancelacion / NC según normativa)',                     7, 1);
```

**Columnas:**

| Nombre             | Tipo de dato     | Descripción                                                       |
| ------------------ | ---------------- | ------------------------------------------------------------------ |
| IdCatFacturaEstado | uniqueidentifier | PK, DEFAULT NEWID()                                                 |
| Clave              | varchar(30)      | Clave programática (UQ)                                             |
| Descripcion        | nvarchar(150)    | Descripción legible                                                 |
| Orden              | int              | Orden natural del ciclo de vida (UI/reportes)                       |
| EsTerminal         | bit              | 1 = sin transiciones posteriores (pagada, cancelada)                |
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
- Transiciones válidas: porgenerar → generada \| errortimbrado; errortimbrado → generada (reintento de Finanzas — Timbrado no reintenta, RE-FU-018); generada → enviada \| cancelada; enviada → pagadaparcial \| pagada \| cancelada; pagadaparcial → pagada \| cancelada.
- pagada y cancelada son estados terminales (`EsTerminal = 1`).
- La inmutabilidad fiscal aplica desde generada: la corrección posterior al timbrado es solo vía el módulo Notas de Crédito (RE-FU-032/033).
- No confundir con `catDocumentoFiscalCobroEstado` (estado de línea del wizard Validar Cobro Paso 3, RE-FU-028), con `CFDIGenerada.Estado` (estado técnico del timbrado) ni con el `EstadoFAA` calculado de `vfccFactura` (estado del pendiente FAA).
- La transición a pagadaparcial/pagada la ejecuta Validar Cobro (RE-FU-026/027/028/029) al aplicar cobros; la transición a cancelada la ejecuta el flujo de cancelación (RE-FU-032, `POST /api/v1/stamp/cancel`).

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
| IdCatFacturaEstado                                      | uniqueidentifier   | FK → `catFacturaEstado.IdCatFacturaEstado`, requerido — estado del ciclo de vida de la factura; se asigna porgenerar al crear el registro (la app resuelve el Id por `Clave`) |
| Enviada                                                 | bit                | 0 = no enviada / 1 = enviada al cliente con PDF+XML — determina, junto con `IdCFDIGenerada`, el estado calculado `EstadoFAA` en `vfccFactura` (equivalente a `tpProformaAdelanto.Enviada`, migrado de RE-FU-019) |
| FechaEnvio                                              | datetime NULL  | Fecha y hora (UTC) del envío de la factura al cliente — se asigna con GETDATE() en el mismo UPDATE que `Enviada = 1` / `IdCatFacturaEstado = enviada`; `NULL` mientras no se envía |
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
- `IdCatFacturaEstado` sigue el ciclo de `catFacturaEstado` (ver catálogo arriba): porgenerar al crear; generada al timbrar (junto con `IdCFDIGenerada`); enviada al enviar (junto con `Enviada = 1` y `FechaEnvio = GETDATE()`); pagadaparcial/pagada desde Validar Cobro; cancelada desde el flujo de cancelación. Convive con `EstadoFAA` (calculado en `vfccFactura`, específico del pendiente FAA) sin sustituirlo.
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

## Estatus del pedido (OBS-027 resuelto) — `catEstadoPedido` extendido + `catMotivoCancelacion` + `PedidoEstadoActual`

> Reemplaza lo documentado en versiones anteriores de este archivo (`CatEstadoTpPedido` nuevo + `ALTER TABLE tpPedido ADD IdCatEstadoTpPedido`). Ver catálogo propuesto por el cliente: `Analisis/Estados de Pedidos/catEstadoPedido — Estados Propuestos.md`.

### Extensión de `catEstadoPedido` (catálogo EXISTENTE)

**Descripción:** `dbo.catEstadoPedido` ya existe en BD `ProquifaDotNet` (`IdCatEstadoPedido uniqueidentifier`, `EstadoPedido varchar(50)`, `Clave varchar(150)`, `Activo bit`, `FechaUltimaActualizacion datetime` — verificado en `ProquifaDotNet.sql` líneas 29278–29289), vacío y sin FK desde `tpPedido`/`ppPedido`. Se extiende y se puebla con los 17 estados del ciclo de vida del pedido (ver catálogo propuesto, secciones 3 y 8).

```sql
ALTER TABLE dbo.catEstadoPedido
    ADD Orden int NULL,
        EsTerminal bit NOT NULL CONSTRAINT DF_catEstadoPedido_EsTerminal DEFAULT (0),
        Aplicativo varchar(30) NULL,        -- 'ProquifaDotNet' | 'Finanzas' | 'Distribuido'
        AliasOperativo varchar(50) NULL,    -- ej. 'PEDIDO_ABIERTO' para pedidoconfirmado
        FechaRegistro datetime NULL;        -- falta en la definición original; se agrega para completar los 3 campos de control del estándar

ALTER TABLE dbo.catEstadoPedido
    ADD CONSTRAINT UQ_catEstadoPedido_Clave UNIQUE (Clave);
```

**Columnas nuevas:**

| Nombre | Tipo de dato | Descripción |
|---|---|---|
| Orden | int NULL | Orden natural del ciclo de vida (UI/reportes) |
| EsTerminal | bit | 1 = estado terminal (entregado, cancelado) |
| Aplicativo | varchar(30) NULL | Qué aplicativo escribe ese estado: `ProquifaDotNet` \| `Finanzas` \| `Distribuido` |
| AliasOperativo | varchar(50) NULL | Alias de negocio (`PEDIDO_ABIERTO` para `pedidoconfirmado`, `PEDIDO_CERRADO` para `entregado`) |
| FechaRegistro | datetime NULL | Completa los 3 campos de control estándar del proyecto — la definición original solo tenía `FechaUltimaActualizacion` |

**Seed:** 17 estados (script tomado del catálogo propuesto, sección 8), con GUIDs hardcodeados por ambiente conforme al estándar de identificadores para catálogos (no `NEWID()` en el script de seed final; se muestra aquí sin GUID explícito porque `IdCatEstadoPedido` toma el default de la PK al insertar):

```sql
-- Seed catEstadoPedido (17 estados) — asume que la tabla está vacía
INSERT INTO dbo.catEstadoPedido (Clave, EstadoPedido, Orden, EsTerminal, Aplicativo, AliasOperativo, FechaUltimaActualizacion, Activo) VALUES
 ('ocrecibida',                       'OC recibida en Atender Promesa de Compra',                           1, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('enpretramite',                     'En Pretramite',                                                       2, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('intramitable',                      'Intramitable',                                                        3, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('intramitablependienteaceptacion', 'Intramitable — correo de aceptación enviado, pendiente de respuesta', 4, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('intramitablefeasolicitada',       'Intramitable — solicitud de FEA (Fecha Estimada de Ajuste) enviada',  5, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('ocajustadarecibida',              'OC ajustada recibida — Validar ajustes OC',                           6, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('entramite',                        'En Tramite',                                                          7, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('prepagoconfaa',                   'Prepago con Factura por Adelantado',                                  8, 0, 'Finanzas',       NULL,             GETDATE(), 1),
 ('prepagoencobro',                  'Prepago en Cobro',                                                    9, 0, 'Finanzas',       NULL,             GETDATE(), 1),
 ('facturado',                         'Facturado',                                                          10, 0, 'Finanzas',       NULL,             GETDATE(), 1),
 ('pedidoconfirmado',                 'Pedido Confirmado / Abierto',                                        11, 0, 'ProquifaDotNet', 'PEDIDO_ABIERTO', GETDATE(), 1),
 ('encompra',                         'En Compra',                                                          12, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('enalmacenmatriz',                 'En Almacen Matriz',                                                  13, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('rechazadoeninspeccion',           'Rechazado en Inspeccion',                                            14, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('enentrega',                        'En Entrega',                                                         15, 0, 'ProquifaDotNet', NULL,             GETDATE(), 1),
 ('entregado',                         'Entregado / Cerrado',                                                16, 1, 'ProquifaDotNet', 'PEDIDO_CERRADO', GETDATE(), 1),
 ('cancelado',                         'Cancelado (motivo en catMotivoCancelacion)',                         17, 1, 'Distribuido',    NULL,             GETDATE(), 1);
```

> **Nota:** el script no incluye `IdCatEstadoPedido` (usa el default `NEWID()` de la PK) ni `FechaRegistro` (columna agregada en este documento sobre el catálogo original); si se decide poblarla en el seed final, agregar `FechaRegistro = GETDATE()` a la lista de columnas y valores.

**Consideraciones especiales:**
- Este documento **consume** el catálogo tal como lo definió el cliente; el seed de las 17 filas se reproduce arriba, pero no se repite aquí la matriz de transiciones — ver el archivo de catálogo para esa matriz y el historial de cambios.
- Las Observaciones #2 (cancelaciones desde estados posteriores a `pedidoconfirmado`), #3 (evento de entrada de `ocajustadarecibida`) y #6 (paso "folio apartado" antes de `pedidoconfirmado`) del catálogo propuesto siguen abiertas — no bloquean T7/T8, pero deben confirmarse con negocio antes de cerrar el desarrollo de los flujos que ejercitan esos tramos (no es este requisito el que los ejercita, ver Gaps).

---

### Tabla: `catMotivoCancelacion` (catálogo NUEVO)

**Descripción:** Motivo de cancelación del pedido, desacoplado del estado terminal único `cancelado` de `catEstadoPedido` (opción B de la Observación #1 del catálogo propuesto — elegida por trazabilidad y para que negocio administre motivos sin release). Propiedad de `ProquifaDotNet`.

```sql
CREATE TABLE dbo.catMotivoCancelacion (
    IdCatMotivoCancelacion uniqueidentifier NOT NULL
        CONSTRAINT PK_catMotivoCancelacion PRIMARY KEY CLUSTERED
        CONSTRAINT DF_catMotivoCancelacion_Id DEFAULT (NEWID()),
    Clave varchar(50) NOT NULL,
    Descripcion varchar(200) NOT NULL,
    Aplicativo varchar(30) NULL,
    Activo bit NOT NULL CONSTRAINT DF_catMotivoCancelacion_Activo DEFAULT (1),
    FechaRegistro datetime NOT NULL CONSTRAINT DF_catMotivoCancelacion_FechaRegistro DEFAULT (GETDATE()),
    FechaUltimaActualizacion datetime NOT NULL CONSTRAINT DF_catMotivoCancelacion_Fecha DEFAULT (GETDATE()),
    CONSTRAINT UQ_catMotivoCancelacion_Clave UNIQUE (Clave)
);
```

**Seed:** 5 motivos (`intramitable`, `ocnoajustada`, `faltapago`, `operativo`, `solicitudcliente`) — ver catálogo propuesto, sección 8.

**Relaciones:**

| Relación | Tipo |
|---|---|
| `PedidoEstadoActual.IdCatMotivoCancelacion` → `catMotivoCancelacion.IdCatMotivoCancelacion` | N:1, opcional |

**Consideraciones especiales:**
- La asignación de `IdCatMotivoCancelacion` (lógica de escritura del flujo de cancelación) **no se desarrolla en este requisito** — queda fuera de alcance, para el requisito de cancelación correspondiente. Aquí solo se deja modelada la relación.

---

### Tabla: `PedidoEstadoActual` (NUEVA)

**Descripción:** Estatus actual del pedido a lo largo de todo su ciclo de vida (Criterio D5), consolidado en un único registro por pedido que se va enriqueciendo y actualizando conforme el pedido avanza — **no es una bitácora histórica de transiciones**, sino el estado vigente. Referencia, según la etapa en la que se encuentre el pedido, a `pcPromesaDeCompra` (recepción en Atender Promesa de Compra / Buzón), `ppPedido` (Pretramitar) y/o `tpPedido` (Tramitar); las tres FK son nullable porque se van poblando progresivamente, nunca las tres a la vez desde el origen. Propiedad de `ProquifaDotNet` (vive en la misma BD que `pcPromesaDeCompra`/`ppPedido`/`tpPedido`, a diferencia de `fccFactura` que es de Finanzas).

```sql
CREATE TABLE dbo.PedidoEstadoActual (
    IdPedidoEstadoActual   uniqueidentifier NOT NULL
        CONSTRAINT PK_PedidoEstadoActual PRIMARY KEY CLUSTERED
        CONSTRAINT DF_PedidoEstadoActual_Id DEFAULT (NEWID()),
    IdPcPromesaDeCompra    uniqueidentifier NULL
        CONSTRAINT FK_PedidoEstadoActual_pcPromesaDeCompra
            FOREIGN KEY REFERENCES dbo.pcPromesaDeCompra (IdPcPromesaDeCompra),
    IdPPPedido             uniqueidentifier NULL
        CONSTRAINT FK_PedidoEstadoActual_ppPedido
            FOREIGN KEY REFERENCES dbo.ppPedido (IdPPPedido),
    IdTPPedido             uniqueidentifier NULL
        CONSTRAINT FK_PedidoEstadoActual_tpPedido
            FOREIGN KEY REFERENCES dbo.tpPedido (IdTPPedido),
    IdCatEstadoPedido      uniqueidentifier NOT NULL
        CONSTRAINT FK_PedidoEstadoActual_catEstadoPedido
            FOREIGN KEY REFERENCES dbo.catEstadoPedido (IdCatEstadoPedido),
    IdCatMotivoCancelacion uniqueidentifier NULL
        CONSTRAINT FK_PedidoEstadoActual_catMotivoCancelacion
            FOREIGN KEY REFERENCES dbo.catMotivoCancelacion (IdCatMotivoCancelacion),
    FechaRegistro              datetime NOT NULL
        CONSTRAINT DF_PedidoEstadoActual_FechaRegistro DEFAULT (GETDATE()),
    FechaUltimaActualizacion   datetime NOT NULL
        CONSTRAINT DF_PedidoEstadoActual_FechaUltimaActualizacion DEFAULT (GETDATE()),
    Activo                     bit NOT NULL
        CONSTRAINT DF_PedidoEstadoActual_Activo DEFAULT (1)
);
```

**Columnas:**

| Nombre                   | Tipo de dato          | Descripción                                                                                                                                                                                         |
| ------------------------ | --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IdPedidoEstadoActual     | uniqueidentifier      | PK, DEFAULT NEWID()                                                                                                                                                                                 |
| IdPcPromesaDeCompra      | uniqueidentifier NULL | FK → `pcPromesaDeCompra.IdPcPromesaDeCompra` — se puebla cuando la OC del cliente se baja del Buzón y se registra en Atender Promesa de Compra (L03, estado `ocrecibida`)                          |
| IdPPPedido               | uniqueidentifier NULL | FK → `ppPedido.IdPPPedido` — se puebla cuando la promesa de compra pasa a pedido (Pretramitar, L04)                                                                                                 |
| IdTPPedido               | uniqueidentifier NULL | FK → `tpPedido.IdTPPedido` — se puebla cuando se manda crear como pedido activo (Tramitar, L05)                                                                                                     |
| IdCatEstadoPedido        | uniqueidentifier      | FK → `catEstadoPedido.IdCatEstadoPedido`, requerido — el estado vigente; cambia conforme el pedido avanza en el flujo                                                                               |
| IdCatMotivoCancelacion   | uniqueidentifier NULL | FK → `catMotivoCancelacion.IdCatMotivoCancelacion` — solo tiene valor cuando `IdCatEstadoPedido` corresponde al estado terminal `cancelado`; su asignación queda fuera de alcance de este requisito |
| FechaRegistro            | datetime              | Alta del registro (momento en que se crea, típicamente en `ocrecibida`)                                                                                                                            |
| FechaUltimaActualizacion | datetime              | Última vez que cambió `IdCatEstadoPedido` (o se agregó una de las FK de origen)                                                                                                                     |
| Activo                   | bit                   | Control (borrado lógico)                                                                                                                                                                            |

**Relaciones:**

| Relación | Tipo |
|---|---|
| `PedidoEstadoActual.IdPcPromesaDeCompra` → `pcPromesaDeCompra.IdPcPromesaDeCompra` | N:1, opcional |
| `PedidoEstadoActual.IdPPPedido` → `ppPedido.IdPPPedido` | N:1, opcional |
| `PedidoEstadoActual.IdTPPedido` → `tpPedido.IdTPPedido` | N:1, opcional |
| `PedidoEstadoActual.IdCatEstadoPedido` → `catEstadoPedido.IdCatEstadoPedido` | N:1, requerido |
| `PedidoEstadoActual.IdCatMotivoCancelacion` → `catMotivoCancelacion.IdCatMotivoCancelacion` | N:1, opcional |

**Índices:**

| Nombre | Columnas | Tipo |
|---|---|---|
| PK_PedidoEstadoActual | IdPedidoEstadoActual | Clustered (PK) |
| IX_PedidoEstadoActual_IdPcPromesaDeCompra | IdPcPromesaDeCompra | Nonclustered (FK, filtrado) |
| IX_PedidoEstadoActual_IdPPPedido | IdPPPedido | Nonclustered (FK, filtrado) |
| IX_PedidoEstadoActual_IdTPPedido | IdTPPedido | Nonclustered (FK, filtrado — es el más consultado, por ser la etapa final) |
| IX_PedidoEstadoActual_IdCatEstadoPedido | IdCatEstadoPedido | Nonclustered (filtrado por estado, tableros/consultas) |

**Consideraciones especiales:**
- **Un registro por pedido, no una bitácora.** El nombre (`PedidoEstadoActual`) y los tres campos de control reflejan un registro que se actualiza in-place: se crea una vez (normalmente en `ocrecibida`) y se actualiza conforme el pedido avanza — no se inserta una fila nueva por cada cambio de estado. Si el cliente requiere el historial completo de transiciones (no solo el estado vigente) para auditoría o Power BI, es una tabla adicional fuera de alcance de este ajuste (ver Gaps).
- **Alta del registro:** ocurre en el punto del flujo donde nace el pedido — normalmente al bajar la OC del Buzón (`pcPromesaDeCompra`, `ocrecibida`). Los pedidos que ingresan sin ese paso (origen Venta Digital u otro) deben identificar su propio punto de alta; confirmar con el flujo real de cada origen (fuera de alcance de este requisito, ver Gaps).
- **Enriquecimiento progresivo:** cuando el pedido pasa de una etapa a otra (`pcPromesaDeCompra` → `ppPedido` → `tpPedido`), el mismo registro se localiza por la FK que ya tiene poblada y se actualiza para agregar la FK de la nueva etapa — no se crea un registro nuevo. Este ajuste puntual (T7/T8) cubre la etapa que corresponde a este requisito (cierre de Tramitar Pedido con FAA, `IdTPPedido` ya poblado desde antes por RE-013/014); los puntos de alta y enriquecimiento en etapas anteriores (Buzón → Pretramitar → Tramitar) son responsabilidad de los requisitos que implementan esas etapas (RE-013/014 y anteriores) y no se tocan en este ajuste.
- `IdCatMotivoCancelacion` solo debe tener valor cuando `IdCatEstadoPedido` referencia la clave `cancelado` de `catEstadoPedido` — no se declara como restricción de verificación en este documento porque `cancelado` es un dato (una fila de catálogo), no un valor fijo de esquema; queda como regla de negocio a validar en el servicio que escribe (T8) o, si se prefiere blindarlo en BD, como disparador (`trValidaMotivoCancelacion`) a evaluar en implementación.
- El endpoint que actualiza `IdCatEstadoPedido` (ver `-Back.md`, T8) localiza el registro por la FK de mayor jerarquía informada, en orden descendente del flujo: `IdTPPedido` → `IdPPPedido` → `IdPcPromesaDeCompra`.

---

## Migración de los estados de los pedidos actuales (backfill de `PedidoEstadoActual`)

> Objetivo: sembrar `PedidoEstadoActual` con un registro por pedido en curso o cerrado, para que el catálogo `catEstadoPedido` tenga sustento desde el día uno y no solo aplique a pedidos nuevos. Es un script de datos (idempotente por clave de negocio, no por identificador — ver `ryndem-sqlserver`, cap. 11): valida por combinación de `IdPcPromesaDeCompra`/`IdPPPedido`/`IdTPPedido` antes de insertar, para poder reejecutarse sin duplicar.

**Regla de mapeo (mejor esfuerzo con las columnas verificadas en `ProquifaDotNet.sql`):**

| Origen / condición | Clave `catEstadoPedido` resultante |
|---|---|
| `pcPromesaDeCompra` con `IdPPPedido IS NULL` | `ocrecibida` |
| `ppPedido.Cancelada = 1` | `cancelado` (motivo sin determinar — ver nota) |
| `ppPedido.Tramitado = 0` y `ppPedido.Intramitable = 1` | `intramitable` (sub-estado ACEPTACION/FEA sin determinar — ver nota) |
| `ppPedido.Tramitado = 0` y `ppPedido.Intramitable` es `0`/`NULL` | `enpretramite` |
| `ppPedido.Tramitado = 1` y no existe `tpPedido` asociado aún | `entramite` |
| `tpPedido.Finalizado = 1` | `entregado` |
| `tpPedido.Liberado = 1` y `Finalizado` no es `1` | `pedidoconfirmado` |
| `tpPedido.FacturaPorAdelantado = 1` y `Liberado = 0` | `prepagoconfaa` o `prepagoencobro` según exista o no un pendiente FAA histórico ya emitido (sin determinar — ver nota) |
| `tpPedido.Liberado = 0`, sin FAA, con partidas en seguimiento logístico | según `MAX(catSeguimientoPartidaPedido.Clave)` de las partidas del pedido (`encompra` → `encompra`, `almacenmatriz` → `enalmacenmatriz`, `rechazadoeninspeccion` → `rechazadoeninspeccion`, `enentrega` → `enentrega`) |

```sql
-- Ejemplo simplificado — paso 1: pedidos aun en pcPromesaDeCompra (no llegaron a ppPedido)
INSERT INTO dbo.PedidoEstadoActual (IdPcPromesaDeCompra, IdCatEstadoPedido)
SELECT pc.IdPcPromesaDeCompra, ce.IdCatEstadoPedido
FROM dbo.pcPromesaDeCompra pc
CROSS JOIN (SELECT IdCatEstadoPedido FROM dbo.catEstadoPedido WHERE Clave = 'ocrecibida') ce
WHERE pc.IdPPPedido IS NULL
  AND pc.Activo = 1
  AND NOT EXISTS (
      SELECT 1 FROM dbo.PedidoEstadoActual pea WHERE pea.IdPcPromesaDeCompra = pc.IdPcPromesaDeCompra
  );

-- Paso 2: pedidos en ppPedido (sin tpPedido aun) — enpretramite / intramitable / entramite / cancelado
-- (repetir el patron NOT EXISTS por IdPPPedido, con el CASE de la tabla de mapeo)

-- Paso 3: pedidos con tpPedido — prepagoconfaa / pedidoconfirmado / logistica / entregado
-- (repetir el patron NOT EXISTS por IdTPPedido, con el CASE de la tabla de mapeo y el JOIN a catSeguimientoPartidaPedido)
```

**Nota — piezas que requieren confirmación de negocio antes de correr en PROD (no bloquean T7/T8, sí la ejecución del script en el ambiente productivo):**
- Sub-estados de `intramitable` (`intramitablependienteaceptacion`, `intramitablefeasolicitada`, `ocajustadarecibida`) requieren inspeccionar tablas adicionales (`ppPedidoAceptacionConError`, `ppPedido.FechaEstimadaAjuste`) no cubiertas por este requisito.
- Distinguir `prepagoconfaa` de `prepagoencobro` para pedidos históricos requiere revisar el pendiente FAA legado (`tpProformaAdelanto`, previo a este requisito) en vez de `fccFactura` (que nace vacía con este requisito).
- El motivo de cancelación (`IdCatMotivoCancelacion`) de pedidos ya cancelados no se puede reconstruir de forma confiable con las columnas actuales (`Cancelada bit`, sin motivo) — se deja `NULL` en la migración y se documenta como dato no migrable.

---

## Tablas Involucradas

| Tabla                            | Rol                                                                   | Estado                                                              |
| --------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------- |
| tpPedido                         | Cabecera - FacturaPorAdelantado=1, Prepago                            | Existente - sin cambios estructurales (OBS-027 resuelto sin ALTER — el estatus se centraliza en `PedidoEstadoActual`) |
| ppPedido / pcPromesaDeCompra     | Origen del pedido en etapas previas a Tramitar                        | Existentes - sin cambios estructurales                              |
| **catFacturaEstado**             | Catálogo de estados del ciclo de vida de la Factura                   | **NUEVA**                                                           |
| **fccFactura**                   | Cabecera del pendiente FAA (y de la factura final)                    | **NUEVA**                                                           |
| **fccFacturaPartida**            | Detalle de partidas del pendiente FAA                                 | **NUEVA**                                                           |
| **fccFacturaReferenciaBancaria** | Referencias bancarias del pendiente FAA                               | **NUEVA**                                                           |
| **catEstadoPedido**              | Catálogo del ciclo de vida del pedido (17 estados, OBS-027)           | **EXISTENTE, EXTENDIDA** (`Orden`, `EsTerminal`, `Aplicativo`, `AliasOperativo` + seed) |
| **catMotivoCancelacion**         | Catálogo de motivos de cancelación (OBS-027)                          | **NUEVA**                                                           |
| **PedidoEstadoActual**           | Estatus vigente del pedido, centralizado (Criterio D5, OBS-027)       | **NUEVA**                                                           |
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
       - fccFactura (EsFacturaPorAdelantado=1, IdCatFacturaEstado=porgenerar,
         campos fiscales timbrado NULL)
       - fccFacturaPartida (una por partida del pedido)
       - fccFacturaReferenciaBancaria (cuentas M.N./DLS + ReferenciaCliente)
    5. NO se genera tpProformaPedido, PDF ni correo en este flujo
    6. NO INSERT pendiente Validar Cobro
    7. Actualiza PedidoEstadoActual.IdCatEstadoPedido = prepagoconfaa (localizado por
       IdTPPedido, via PUT /v1/api/orders/status) -> cierra pendiente Tramitar Pedido
       (OBS-027 resuelto)

   === POSTERIORMENTE (fuera scope este requisito) ===
   8. Modulo FAA emite factura PPD -> INSERT CFDIGenerada (Finanzas) ->
      UPDATE fccFactura SET EsFacturaPorAdelantado=0, IdCFDIGenerada=@Id,
      IdCatFacturaEstado=generada (si el PAC rechaza: errortimbrado, reintento de Finanzas)
      (Serie/Folio/FolioFiscal/Version/TipoDeComprobante/FechaCertificacion
      se leen de CFDIGenerada via este FK, no se duplican en fccFactura)
   8b. Al enviar la factura -> UPDATE fccFactura SET Enviada=1, FechaEnvio=GETDATE(),
       IdCatFacturaEstado=enviada
   9. FAA genera pendiente Validar Cobro
   10. Validar Cobro aplica cobros -> IdCatFacturaEstado=pagadaparcial o pagada;
       cancelacion (RE-032) -> IdCatFacturaEstado=cancelada

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
| 7   | ALTER TABLE `catEstadoPedido` ADD `Orden`, `EsTerminal`, `Aplicativo`, `AliasOperativo` + seed (17 estados) | No — OBS-027 resuelto (prerrequisito de 9) |
| 8   | CREATE TABLE `catMotivoCancelacion` + seed (5 motivos)                                              | No — OBS-027 resuelto (prerrequisito de 9) |
| 9   | CREATE TABLE `PedidoEstadoActual` (FK → `pcPromesaDeCompra`, `ppPedido`, `tpPedido`, `catEstadoPedido`, `catMotivoCancelacion`) | No (depende de 7 y 8) |
| 10  | Script de datos — backfill de `PedidoEstadoActual` con los pedidos existentes                       | No (depende de 9; ver sección de migración) |

> **Nota de migración (propagada a RE-FU-012/018/019/020/026/027/028/029/030):** estos requisitos consumían `tpProformaAdelanto` + `vtpProformaAdelanto`. A partir de la adopción de este esquema, deben apuntar a `fccFactura` + `vfccFactura`. Ver sección "Migración de `tpProformaAdelanto`" más abajo.

---

## Cambios de Comportamiento R16 (sin cambio en BD)

| Cambio | Antes | Ahora (R16) |
|--------|-------|-------------|
| Codigo autorizacion para FAA | Requeria codigo | Activacion directa |
| Boton Editar Datos | Visible | Oculto para Prepago siempre |
| Generacion de proforma/PDF/correo en Tramitar Pedido (con FAA) | N/A | Ya no aplica — el pendiente FAA se genera directo |
| Estructura del pendiente FAA | `tpProformaAdelanto` | `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria` |
| Política de folio de proforma en reintento (DUDA-030, 2026-08-21) | Pendiente definir | No aplica a este requisito (ya no genera proforma). Resolución documentada para trazabilidad: donde sí aplica (RE-FU-013/014), el folio se consume hasta el envío correcto — se reintenta con el mismo folio hasta éxito, sin descartarlo ni reasignar uno nuevo |

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | Vinculacion FAA -> Validar Cobro posterior | Confirmar logica del modulo FAA (RE-018/019/020) sobre UPDATE de `fccFactura` |
| 2 | H-01 — Campos fiscales de Perú en `fccFactura` | Agregar columnas equivalentes a Tipo de Operación / Condición de Pago SUNAT, o documentar por qué no aplican |
| 3 | Documento disponible para TaskScheduler/Legacy | Confirmar si el job de Venta Digital puede operar sin ningun PDF generado en este flujo |
| 4 | ~~Catalogo de estatus del pedido (OBS-027 / Criterio D5)~~ | **Resuelto** — `catEstadoPedido` extendido + `catMotivoCancelacion` + `PedidoEstadoActual` (ver sección "Estatus del pedido" arriba) |
| 5 | Sub-estados de `intramitable` y distinción `prepagoconfaa`/`prepagoencobro` para el backfill histórico | Confirmar con negocio antes de correr la migración en PROD (ver sección de migración) |
| 6 | ¿`PedidoEstadoActual` necesita historial completo de transiciones, o basta el estado vigente? | El nombre y diseño actual asumen solo estado vigente (no bitácora) — confirmar si Power BI u otro consumidor requiere historial; sería tabla adicional fuera de este alcance |

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
| `catEstadoPedido — Estados Propuestos.md` | Fuente de la propuesta que resuelve OBS-027 (17 estados, matriz de transiciones, `catMotivoCancelacion`) — `Analisis/Estados de Pedidos/` |
| R16A-RE-FU-006         | Referencia bancaria del documento fiscal (Codigo Validador) → `fccFacturaReferenciaBancaria.ReferenciaCliente`                                                            |
| R16A-RE-FU-012         | Variante Crédito de FAA — migrada a `fccFactura` con `IdTPProformaPedido` poblado (ver sección de migración)                                                              |
| R16A-RE-FU-013         | Arquitectura orquestador→Finanzas ya validada; folio interno                                                                                                              |
| R16A-RE-FU-014         | Flujo base Prepago sin FAA (este agrega FAA)                                                                                                                              |
| R16A-RE-FU-016         | Criterio E1 — dos cuentas bancarias del grupo PROQUIFA México                                                                                                             |
| R16A-RE-FU-018/019/020 | Módulo FAA — consulta `vfccFactura`, actualiza `fccFactura` a factura final (`EsFacturaPorAdelantado=0`, `IdCFDIGenerada`, `IdCatFacturaEstado`, `Enviada`, `FechaEnvio`) |
| R16A-RE-FU-021         | `catClaveProdServSAT` reutilizado por `fccFacturaPartida.ClaveProductoServicioSAT`                                                                                        |
| R16A-RE-FU-026/027     | Asociación de cobro — `fccPagoFacturaAdelanto.IdFccFactura`; transiciones pagada / pagadaparcial                                                                         |
| R16A-RE-FU-028/029/030 | Complemento de Pago — JOIN `fccPagoFacturaAdelanto → fccFactura → CFDIGenerada` para `CFDIRelacionados`                                                                   |
| R16A-RE-FU-032         | Cancelación de factura origen — transición cancelada                                                                                                                      |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
