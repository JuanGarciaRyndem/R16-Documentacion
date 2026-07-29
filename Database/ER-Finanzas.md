# Finanzas — Cobros, Facturación y Notas de Crédito

Tablas físicas en ProquifaDotNet gestionadas por la solución ProquifaDotNet.Finanzas via EF Core Scaffold: fcc*, CFDIGenerada, NC, foliador y catálogos SAT/SUNAT.

> **Migración (06/07/2026):** `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria` (agregadas aquí) reemplazan a la tabla legada `tpProformaAdelanto` (ver `ER-ProquifaDotNet.md`) como pendiente único de Factura por Adelantado, tanto de origen Prepago (RE-FU-015) como Crédito (RE-FU-012). `fccPagoFacturaAdelanto.IdFccFactura` reemplaza a `fccPagoFacturaAdelanto.IdTPProformaAdelanto`. Detalle completo en `R16A-RE-FU-015_BD.md`, sección "Migración de `tpProformaAdelanto`".

> **Actualización (07/07/2026):**
> - Se agrega el catálogo `catFacturaEstado` (RE-FU-015 v2.1) y la FK `fccFactura.IdCatFacturaEstado` — ciclo de vida de la factura: POR_GENERAR → ERROR_TIMBRADO/GENERADA → ENVIADA → PAGADA_PARCIAL/PAGADA → CANCELADA. También se agrega `fccFactura.FechaEnvio`.
> - **`EmpresaFolio` se mueve de `ProquifaDotNetTimbrado` a `ProquifaDotNet`, propiedad de Finanzas** (consumo directo con UPDLOCK atómico al timbrar, `EmpresaFolioRepository`). Timbrado no la toca — su BD queda solo con `AppSetting` + `StampingLog` (ver `ER-Timbrado.md`).
> - Se agrega `fccInconsistenciaCobro` (RE-FU-024/026) y las columnas de archivo/certificación de `CFDIGenerada` (RE-FU-021).
> - **`tpProformaPedido` y `tpProformaPartidaPedido` pasan a Finanzas**: se integran al Scaffold EF Core de `Finanzas.Infrastructure` (lectura/escritura directa — ya no se consumen vía API ProquifaDotNet). `tpProformaPedido` se agrega al diagrama con sus columnas ampliadas en R16 (FolioProforma, MontoPendiente, IdCFDIGenerada, ReferenciaPago, FechaEstimadaPago); `tpProformaPartidaPedido` es su detalle 1:N de partidas.

> **Actualización (08/07/2026):**
> - `fccFolioPagoCliente` y `fccPagoCliente` se corrigen a su **estructura real verificada contra BD** (`ProquifaDotNet.sql`, RE-023_BD). Se eliminan columnas que no existen (`FolioIdentificado`, `MontoIdentificado`, `Procesado` en el folio) y se agregan las reales (`Folio`, `Consecutivo`, `FechaRecepcion`, `Stp`, campos MailBot, `FechaUltimaActualizacion`).
> - Se agrega el catálogo `catCobroEstatus` (definido en RE-002) y la FK `fccPagoCliente.IdCatCobroEstatus` (ALTER RE-008) — ciclo de vida del cobro: BORRADOR → CAPTURADO → ASOCIADO / SALDO_A_FAVOR → COMPLETADO; en cualquier punto CON_INCONSISTENCIA o CANCELADO. `IdCatCobroEstatus` es la fuente de verdad del estado; `Confirmado` (bit) coexiste por compatibilidad y puede deprecarse. El pendiente del Buzón (`fccFolioPagoCliente`) no lleva catálogo de estatus: su estado es `Activo` (1 = abierto, 0 = cerrado).
> - `tpProformaPedido` y `tpProformaPartidaPedido` se corrigen a su **estructura real verificada contra BD**. Únicos cambios R16: ALTER `FolioProforma` + `ConsecutivoProforma` (RE-016). Se corrigen nombres: la fecha estimada de pago es la columna existente `FechaPromesaPagoMonitoreoCobros` (RE-023 la reutiliza, no crea columna) y el flag de controlados es `Controlados` (existente, RE-013/014) — no `HayControlados`.

```mermaid
erDiagram
    %% catImpuesto / catTipoFactorSat / catObjetoImpuestoSat / PerfilFiscal / FamiliaRegion
    %% viven en ProquifaDotNet — acceso Finanzas via Scaffold (R16A-RE-Cambio-PerfilFiscal)
    catImpuesto {
        uniqueidentifier IdCatImpuesto PK
        varchar Clave
        varchar Descripcion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catTipoFactorSat {
        uniqueidentifier IdCatTipoFactorSat PK
        varchar Clave
        varchar Descripcion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catObjetoImpuestoSat {
        uniqueidentifier IdCatObjetoImpuestoSat PK
        varchar Clave
        varchar Descripcion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    PerfilFiscal {
        uniqueidentifier IdPerfilFiscal PK
        uniqueidentifier IdRegion FK
        uniqueidentifier IdCatImpuesto FK
        uniqueidentifier IdCatTipoFactorSat FK
        decimal TasaOCuota
        uniqueidentifier IdCatObjetoImpuestoSat FK
        nvarchar Fundamento
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    FamiliaRegion {
        uniqueidentifier IdFamiliaRegion PK
        uniqueidentifier IdFamilia FK
        uniqueidentifier IdRegion FK
        uniqueidentifier IdPerfilFiscal FK
        varchar ClaveProdServSat
        varchar ClaveUnidadSat
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catTipoCFDI {
        uniqueidentifier IdCatTipoCFDI PK
        varchar Clave
        varchar Descripcion
        varchar TipoDocumentoSAT
        uniqueidentifier IdRegion FK
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catTipoDocumentoFiscal {
        uniqueidentifier IdCatTipoDocumentoFiscal PK
        varchar Clave
        varchar Descripcion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catDocumentoFiscalCobroEstado {
        uniqueidentifier IdCatDocumentoFiscalCobroEstado PK
        varchar Clave
        varchar Descripcion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catFacturaEstado {
        uniqueidentifier IdCatFacturaEstado PK
        varchar Clave UK
        nvarchar Descripcion
        int Orden
        bit EsTerminal
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catFormaPagoSAT {
        uniqueidentifier IdCatFormaPagoSAT PK
        varchar Clave
        varchar Descripcion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catTipoOperacionSUNAT {
        uniqueidentifier IdCatTipoOperacionSUNAT PK
        varchar Clave
        varchar Descripcion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catMotivoCancelacionSAT {
        uniqueidentifier IdCatMotivoCancelacionSAT PK
        varchar Clave
        varchar Descripcion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catMotivoCreditoSUNAT09 {
        uniqueidentifier IdCatMotivoCreditoSUNAT09 PK
        varchar Clave
        varchar Descripcion
        varchar Modalidad
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catTipoInconsistenciaCobro {
        uniqueidentifier IdCatTipoInconsistenciaCobro PK
        varchar TipoInconsistencia
        bit AplicaMarkPendienteCancelacion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    EmpresaFolio {
        uniqueidentifier IdEmpresaFolio PK
        uniqueidentifier IdEmpresa FK
        varchar Serie
        int UltimoFolio
        varchar FormatoFolio
        int LongitudMaxima
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    CFDIGenerada {
        uniqueidentifier IdCFDIGenerada PK
        varchar RFCEmisor
        varchar RFCReceptor
        varchar Serie
        varchar Folio
        datetime FechaEmision
        datetime FechaCertificacionSat
        uniqueidentifier IdCatTipoCFDI FK
        uniqueidentifier IdCFDIRelacionado FK
        varchar UUID
        decimal Total
        uniqueidentifier IdCatMetodoDePagoCFDI FK
        uniqueidentifier IdCatFormaPagoSAT FK
        varchar Exportacion
        varchar Estado
        uniqueidentifier IdArchivoXml FK
        uniqueidentifier IdArchivoPdf FK
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    CFDIGeneradaConcepto {
        uniqueidentifier IdCFDIGeneradaConcepto PK
        uniqueidentifier IdCFDIGenerada FK
        varchar ClaveProdServ
        varchar Descripcion
        decimal Cantidad
        decimal ValorUnitario
        decimal Importe
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    CFDIGeneradaRelacionado {
        uniqueidentifier IdCFDIGeneradaRelacionado PK
        uniqueidentifier IdCFDIGenerada FK
        varchar UUID
        varchar ClaveTipoRelacion
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    CFDICancelacion {
        uniqueidentifier IdCFDICancelacion PK
        uniqueidentifier IdCFDIGenerada FK
        varchar ClaveMotivo
        varchar UUID
        datetime FechaSolicitud
        varchar EstadoCancelacion
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    tpProformaPedido {
        uniqueidentifier IdTPProformaPedido PK
        uniqueidentifier IdCliente FK
        uniqueidentifier IdEmpresa FK
        varchar ReferenciaPago
        varchar NumeroFactura
        decimal MontoTotal
        decimal MontoPagado
        decimal MontoPendiente
        datetime FechaCompromisoPago
        datetime FechaPromesaPagoMonitoreoCobros
        datetime FechaPagoCompleto
        bit FacturaFlete
        uniqueidentifier IdCFDIGenerada FK
        bit Factura
        bit MXN
        bit USD
        varchar Folio
        varchar Serie
        varchar Uuid
        uniqueidentifier IdCFDI FK
        bit Cancelada
        uniqueidentifier IdTPProformaPedidoReemplazo FK
        decimal PrecioFleteKPI
        bit Revisada
        bit Contrarecibo
        bit Controlados
        varchar Comentarios
        bit Publicaciones
        uniqueidentifier IdContactoCliente FK
        uniqueidentifier IdDireccionCliente FK
        varchar NumeroOrdenDeCompra
        varchar FolioProforma
        int ConsecutivoProforma
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    tpProformaPartidaPedido {
        uniqueidentifier IdTPProformaPartidaPedido PK
        uniqueidentifier IdTPProformaPedido FK
        uniqueidentifier IdTPPartidaPedido FK
        uniqueidentifier IdEmpresaCompra FK
        uniqueidentifier IdProducto FK
        int NumeroDePiezas
        decimal PrecioUnitario
        uniqueidentifier IdLote FK
        varchar Pedimento
        decimal PrecioFleteParcialidad
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catCobroEstatus {
        uniqueidentifier IdCatCobroEstatus PK
        varchar Clave UK
        varchar Descripcion
        int Orden
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccFolioPagoCliente {
        uniqueidentifier IdFCCFolioPagoCliente PK
        uniqueidentifier IdCorreoRecibidoCliente FK
        uniqueidentifier IdArchivo FK
        varchar Folio
        int Consecutivo
        datetime FechaRecepcion
        bit Stp
        decimal SubtotalMailBot
        decimal IvaMailBot
        decimal TotalMailBot
        bit MxnMailBot
        bit UsdMailBot
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccPagoCliente {
        uniqueidentifier IdFCCPagoCliente PK
        uniqueidentifier IdCliente FK
        uniqueidentifier IdEmpresa FK
        uniqueidentifier IdContactoCliente FK
        uniqueidentifier IdFCCFolioPagoCliente FK
        uniqueidentifier IdCatCobroEstatus FK
        decimal Monto
        datetime FechaPago
        uniqueidentifier IdCatMoneda FK
        bit MXN
        bit USD
        decimal TipoDeCambio
        uniqueidentifier IdCatMedioDePago FK
        uniqueidentifier IdDatosBancarios FK
        uniqueidentifier IdCatBanco FK
        varchar CuentaOrdenante
        varchar ReferenciaBancaria
        bit Broker
        uniqueidentifier IdCatBrokerCliente FK
        bit InformacionComplementoPago
        uniqueidentifier IdCFDI FK
        varchar Folio
        varchar Serie
        uniqueidentifier IdArchivo FK
        bit Confirmado
        datetime FechaConfirmacion
        uniqueidentifier IdUsuarioConfirmacion FK
        varchar Notas
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccPagoFacturaPedido {
        uniqueidentifier IdFCCPagoFacturaPedido PK
        uniqueidentifier IdFCCPagoCliente FK
        uniqueidentifier IdTPProformaPedido FK
        decimal Monto
        decimal MontoPendienteAnterior
        int NumeroDeParcialidad
        datetime FechaAplicacion
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccFactura {
        uniqueidentifier IdFccFactura PK
        uniqueidentifier IdTPPedido FK
        uniqueidentifier IdTPProformaPedido FK
        uniqueidentifier IdCatFacturaEstado FK
        bit EsFacturaPorAdelantado
        bit Enviada
        datetime FechaEnvio
        uniqueidentifier IdCliente FK
        uniqueidentifier IdEmpresa FK
        varchar FolioPedidoInterno
        decimal MontoTotal
        uniqueidentifier IdCatMoneda FK
        uniqueidentifier IdCFDIGenerada FK
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccFacturaPartida {
        uniqueidentifier IdFccFacturaPartida PK
        uniqueidentifier IdFccFactura FK
        varchar Descripcion
        decimal Cantidad
        decimal ValorUnitario
        decimal Importe
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccFacturaReferenciaBancaria {
        uniqueidentifier IdFccFacturaReferenciaBancaria PK
        uniqueidentifier IdFccFactura FK
        uniqueidentifier IdCatMoneda FK
        varchar Banco
        varchar NumeroCuenta
        varchar Clabe
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccPagoFacturaAdelanto {
        uniqueidentifier IdFCCPagoFacturaAdelanto PK
        uniqueidentifier IdFCCPagoCliente FK
        uniqueidentifier IdTPProformaPedido FK
        uniqueidentifier IdFccFactura FK
        uniqueidentifier IdCFDIGenerada FK
        decimal Monto
        int NumeroParcialidad
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccDocumentoFiscalCobro {
        uniqueidentifier IdFCCDocumentoFiscalCobro PK
        uniqueidentifier IdFCCPagoFacturaPedido FK
        uniqueidentifier IdFCCPagoFacturaAdelanto FK
        uniqueidentifier IdCatTipoDocumentoFiscal FK
        uniqueidentifier IdCatDocumentoFiscalCobroEstado FK
        uniqueidentifier IdCatUsoCFDI FK
        uniqueidentifier IdCatMetodoDePagoCFDI FK
        uniqueidentifier IdCFDIGeneradaFactura FK
        uniqueidentifier IdCFDIGeneradaComplemento FK
        uniqueidentifier IdCatTipoOperacionSUNAT FK
        uniqueidentifier IdCatCondicionesDePago FK
        datetime FechaPagoCP
        uniqueidentifier IdCatFormaPagoSAT FK
        decimal TipoCambioP_CP
        int NumParcialidad
        decimal ImpSaldoAnt
        decimal ImpPagado
        decimal ImpSaldoInsoluto
        decimal EquivalenciaDR
        datetime FechaGeneracion
        datetime FechaEnvio
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccConfirmacionPedido {
        uniqueidentifier IdFCCConfirmacionPedido PK
        uniqueidentifier IdTPProformaPedido FK
        uniqueidentifier IdUsuario FK
        datetime FechaConfirmacion
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccSaldoFavorCliente {
        uniqueidentifier IdFCCSaldoFavorCliente PK
        uniqueidentifier IdCliente FK
        uniqueidentifier IdFCCPagoCliente FK
        varchar TipoSaldo
        decimal Monto
        bit MXN
        bit USD
        decimal TipoCambio
        bit Aplicado
        uniqueidentifier IdFCCPagoFacturaPedido FK
        varchar Observaciones
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccInconsistenciaCobro {
        uniqueidentifier IdFCCInconsistenciaCobro PK
        uniqueidentifier IdFCCPagoCliente FK
        uniqueidentifier IdCatTipoInconsistenciaCobro FK
        varchar Notas
        uniqueidentifier IdUsuario FK
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccNotaCredito {
        uniqueidentifier IdFCCNotaCredito PK
        uniqueidentifier IdTPProformaPedido FK
        uniqueidentifier IdCFDI FK
        uniqueidentifier IdCFDIGenerada FK
        uniqueidentifier IdEmpresa FK
        uniqueidentifier IdCliente FK
        decimal Monto
        varchar Serie
        varchar Modalidad
        varchar Motivo
        varchar Estado
        bit CancelarFacturaOrigen
        uniqueidentifier IdCatMotivoCancelacionSAT FK
        uniqueidentifier IdCFDIGeneradaFacturaOrigen FK
        varchar ConceptoManual
        varchar ObservacionesManual
        uniqueidentifier IdArchivoXml FK
        uniqueidentifier IdArchivoPdf FK
        varchar ResponseCode
        nvarchar ResponseDescription
        decimal TipoCambioOrigen
        uniqueidentifier IdFCCPagoCliente FK
        bit Activo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccNotaCreditoPartida {
        uniqueidentifier IdFCCNotaCreditoPartida PK
        uniqueidentifier IdFCCNotaCredito FK
        uniqueidentifier IdCFDIGeneradaConceptoOrigen FK
        decimal CantidadNC
        decimal Importe
        decimal Subtotal
        decimal IVA
        decimal Total
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    fccFechaEstimadaPagoHistorial {
        uniqueidentifier IdFccFechaEstimadaPagoHistorial PK
        uniqueidentifier IdTpProformaPedido FK
        datetime FechaEstimadaPagoAnterior
        datetime FechaEstimadaPagaNueva
        datetime FechaCambio
        uniqueidentifier IdUsuarioCambio FK
        varchar Motivo
        datetime FechaRegistro
        datetime FechaUltimaActualizacion
    }
    catTipoCFDI ||--o{ CFDIGenerada : "clasifica"
    CFDIGenerada ||--o{ CFDIGeneradaConcepto : "tiene"
    CFDIGenerada ||--o{ CFDIGeneradaRelacionado : "tiene"
    CFDIGenerada ||--o{ CFDICancelacion : "tiene"
    CFDIGenerada }o--o| CFDIGenerada : "relacionada con"
    catTipoDocumentoFiscal ||--o{ fccDocumentoFiscalCobro : "clasifica"
    catDocumentoFiscalCobroEstado ||--o{ fccDocumentoFiscalCobro : "estado"
    catFormaPagoSAT ||--o{ fccDocumentoFiscalCobro : "forma de pago CP"
    catTipoOperacionSUNAT ||--o{ fccDocumentoFiscalCobro : "tipo operacion Peru"
    catFacturaEstado ||--o{ fccFactura : "estado ciclo de vida"
    tpProformaPedido ||--o{ tpProformaPartidaPedido : "tiene"
    tpProformaPedido }o--o| tpProformaPedido : "reemplazada por"
    tpProformaPedido ||--o{ fccPagoFacturaPedido : "recibe cobro"
    tpProformaPedido ||--o{ fccPagoFacturaAdelanto : "referencia"
    tpProformaPedido ||--o{ fccFactura : "origen Credito"
    tpProformaPedido ||--o{ fccConfirmacionPedido : "confirma"
    tpProformaPedido ||--o{ fccNotaCredito : "referencia"
    tpProformaPedido ||--o{ fccFechaEstimadaPagoHistorial : "historial fecha pago"
    CFDIGenerada ||--o{ tpProformaPedido : "timbra"
    fccFolioPagoCliente ||--o{ fccPagoCliente : "origina"
    catCobroEstatus ||--o{ fccPagoCliente : "estatus ciclo de vida"
    fccPagoCliente ||--o{ fccPagoFacturaPedido : "aplica a"
    fccPagoCliente ||--o{ fccPagoFacturaAdelanto : "aplica a"
    fccPagoCliente ||--o{ fccInconsistenciaCobro : "marca"
    catTipoInconsistenciaCobro ||--o{ fccInconsistenciaCobro : "clasifica"
    fccFactura ||--o{ fccFacturaPartida : "tiene"
    fccFactura ||--o{ fccFacturaReferenciaBancaria : "tiene"
    fccFactura ||--o{ fccPagoFacturaAdelanto : "recibe cobro"
    CFDIGenerada ||--o{ fccFactura : "timbra"
    fccPagoCliente ||--o{ fccSaldoFavorCliente : "genera"
    fccPagoCliente ||--o{ fccNotaCredito : "asociado a"
    fccPagoFacturaPedido ||--o{ fccDocumentoFiscalCobro : "genera"
    fccPagoFacturaPedido ||--o{ fccSaldoFavorCliente : "referencia"
    fccPagoFacturaAdelanto ||--o{ fccDocumentoFiscalCobro : "genera"
    fccNotaCredito ||--o{ fccNotaCreditoPartida : "tiene"
    catMotivoCancelacionSAT ||--o{ fccNotaCredito : "motivo cancelacion"
    CFDIGeneradaConcepto ||--o{ fccNotaCreditoPartida : "concepto origen"
    catImpuesto ||--o{ PerfilFiscal : "impuesto"
    catTipoFactorSat ||--o{ PerfilFiscal : "tipo factor"
    catObjetoImpuestoSat ||--o{ PerfilFiscal : "objeto impuesto (MX)"
    PerfilFiscal ||--o{ FamiliaRegion : "perfil aplicado"
    PerfilFiscal ||--o{ CFDIGeneradaConcepto : "aplica perfil fiscal"
    catFormaPagoSAT ||--o{ CFDIGenerada : "forma de pago CFDI"
    catMetodoDePagoCFDI ||--o{ CFDIGenerada : "método de pago CFDI"
```

> `EmpresaFolio.IdEmpresa` referencia a `Empresa` (tabla existente de ProquifaDotNet, ver `ER-ProquifaDotNet.md`). El folio se consume con UPDLOCK atómico (`UPDATE ... SET UltimoFolio = UltimoFolio + 1 OUTPUT inserted.UltimoFolio WHERE IdEmpresa = @Id AND Serie = @Serie`) al timbrar Factura (serie por empresa), Complemento de Pago (Serie 'P') y Nota de Crédito (Serie 'P2'), y para el correlativo GOLPERU (Perú).

---

## Listado de tablas utilizadas por ProquifaDotNet.Finanzas

### 1. Tablas propias de Finanzas (BD ProquifaDotNet, Scaffold EF Core en `Finanzas.Infrastructure`)

| #   | Tabla                           | Estado                                                     | Requisito(s)                                                                                                                                                                    | Uso                                                                                                                                           |
| --- | ------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catFacturaEstado`              | ✨ Nueva R16                                                | RE-015 v2.1                                                                                                                                                                     | Catálogo de estados del ciclo de vida de la Factura (7 estados, transiciones en RE-015_BD)                                                    |
| 2   | `fccFactura`                    | ✨ Nueva R16                                                | RE-015 (dueño), RE-012, 018-022, 026-030                                                                                                                                        | Cabecera única FAA + factura final (`EsFacturaPorAdelantado`); reemplaza `tpProformaAdelanto`                                                 |
| 3   | `fccFacturaPartida`             | ✨ Nueva R16                                                | RE-015                                                                                                                                                                          | Partidas snapshot del pedido (1:N)                                                                                                            |
| 4   | `fccFacturaReferenciaBancaria`  | ✨ Nueva R16                                                | RE-015, RE-006, RE-016                                                                                                                                                          | Cuentas M.N./DLS + ReferenciaCliente (Código Validador)                                                                                       |
| 5   | `EmpresaFolio`                  | ✨ Nueva R16 — **movida a Finanzas (07/07/2026)**           | RE-019 (crea), RE-020 (GOLPERU), RE-028/030/032 (series)                                                                                                                        | Foliador por empresa/serie con UPDLOCK atómico; antes planeada en ProquifaDotNetTimbrado                                                      |
| 6   | `CFDIGenerada`                  | ✨ Nueva R16 (base RE-019, extendida)                       | RE-019 (crea), RE-018/021 (IdArchivoPdf, FechaCertificacionSat), RE-028 (IdCatTipoCFDI, IdCFDIRelacionado)                                                                      | Registro central de negocio de todo CFDI/CPE timbrado — single source of truth (Serie, Folio, UUID)                                           |
| 7   | `CFDIGeneradaConcepto`          | ✨ Nueva R16                                                | RE-021, RE-032                                                                                                                                                                  | Conceptos del CFDI (partidas fiscales)                                                                                                        |
| 8   | `CFDIGeneradaRelacionado`       | ✨ Nueva R16                                                | RE-028, RE-032                                                                                                                                                                  | Nodo CFDIRelacionados (NCs aplicadas, factura origen NC)                                                                                      |
| 9   | `CFDICancelacion`               | Existente pre-R16                                          | RE-032                                                                                                                                                                          | Cancelaciones ante SAT (motivo, estado, acuse)                                                                                                |
| 10  | `catTipoCFDI`                   | ✨ Nueva R16                                                | RE-028                                                                                                                                                                          | Clasifica el CFDI: FACTURA, FACTURA_ANTICIPO, COMPLEMENTO_PAGO, NOTA_CREDITO, FACTURA_CPE...                                                  |
| 11  | `catTipoDocumentoFiscal`        | ✨ Nueva R16                                                | RE-028                                                                                                                                                                          | Tipo de documento por línea del Paso 3 de Validar Cobro                                                                                       |
| 12  | `catDocumentoFiscalCobroEstado` | ✨ Nueva R16                                                | RE-028                                                                                                                                                                          | Estado de línea del wizard VC Paso 3: PENDIENTE → GENERADO → ENVIADO                                                                          |
| 13  | `fccDocumentoFiscalCobro`       | ✨ Nueva R16                                                | RE-028, RE-029, RE-030                                                                                                                                                          | Línea de documento fiscal a emitir por cobro (snapshot CFDI 4.0 / Pagos20)                                                                    |
| 14  | `fccConfirmacionPedido`         | ✨ Nueva R16                                                | RE-028                                                                                                                                                                          | Confirmación de pedido del Paso 3                                                                                                             |
| 15  | `fccFolioPagoCliente`           | ✅ Existente — sin cambios                                  | RE-008, RE-023                                                                                                                                                                  | Pendiente en Validar Cobro generado al clasificar correo como Cobro en el Buzón (Mailbot); incluye pre-extracción MailBot                     |
| 16  | `fccPagoCliente`                | ✨ Existente + ALTER                                        | RE-008 (IdCatCobroEstatus), RE-023 (Confirmado, FechaConfirmacion, IdUsuarioConfirmacion, Notas, IdCatMoneda), RE-024, RE-025                                                   | Cobro capturado en Validar Cobro Paso 1 (folio COB-mmddaa-#, inmutable al confirmar)                                                          |
| 17  | `fccPagoFacturaPedido`          | ✨ Nueva R16                                                | RE-026, RE-027                                                                                                                                                                  | Asociación N:N cobro ↔ proforma (Paso 2)                                                                                                      |
| 18  | `fccPagoFacturaAdelanto`        | ✨ Nueva R16                                                | RE-026, RE-027                                                                                                                                                                  | Asociación N:N cobro ↔ FAA (`IdFccFactura`)                                                                                                   |
| 19  | `fccSaldoFavorCliente`          | ✨ Nueva R16                                                | RE-026, RE-027                                                                                                                                                                  | Saldo a favor / tolerancia ≤100 MXN (Estado de Cuenta)                                                                                        |
| 20  | `catTipoInconsistenciaCobro`    | ✨ Nueva R16                                                | RE-024, RE-026                                                                                                                                                                  | Catálogo de tipos de inconsistencia (Pasos 1 y 2) — seed pendiente Tesorería                                                                  |
| 21  | `fccInconsistenciaCobro`        | ✨ Nueva R16                                                | RE-024, RE-026                                                                                                                                                                  | Inconsistencias marcadas sobre un cobro                                                                                                       |
| 22  | `fccNotaCredito`                | Existente, ampliada R16                                    | RE-026 (vínculo cobro), RE-032/033 (módulo NC)                                                                                                                                  | Notas de Crédito (México CFDI E / Perú CPE 07)                                                                                                |
| 23  | `fccNotaCreditoPartida`         | ✨ Nueva R16                                                | RE-032                                                                                                                                                                          | Detalle por partidas de la NC                                                                                                                 |
| 24  | `fccFechaEstimadaPagoHistorial` | ✨ Nueva R16                                                | RE-023                                                                                                                                                                          | Historial de fecha estimada de pago (Gestionar Cobranza)                                                                                      |
| 25  | `catFormaPagoSAT`               | ✨ Nueva R16                                                | RE-019                                                                                                                                                                          | Catálogo c_FormaPago SAT para captura del cobro                                                                                               |
| 26  | `catTipoOperacionSUNAT`         | ✨ Nueva R16                                                | RE-020, RE-029                                                                                                                                                                  | Catálogo 51 SUNAT (Tipo de Operación, Perú)                                                                                                   |
| 27  | `catMotivoCancelacionSAT`       | ✨ Nueva R16                                                | RE-032                                                                                                                                                                          | Motivos de cancelación SAT (01-04)                                                                                                            |
| 28  | `catMotivoCreditoSUNAT09`       | ✨ Nueva R16                                                | RE-033                                                                                                                                                                          | Catálogo 09 SUNAT (motivos de NC, Perú)                                                                                                       |
| 29  | `tpProformaPedido`              | Existente + ALTER R16 — **movida a Finanzas (07/07/2026)** | RE-016 (ALTER: FolioProforma, ConsecutivoProforma), RE-023 (usa FechaPromesaPagoMonitoreoCobros), RE-026/027 (usa MontoPendiente), RE-028/029 (usa Controlados, IdCFDIGenerada) | Proforma/Confirmación de Pedido — antes vía API ProquifaDotNet, ahora lectura/escritura directa vía Scaffold                                  |
| 30  | `tpProformaPartidaPedido`       | Existente — **movida a Finanzas (08/07/2026)**             | RE-013/014 (INSERT al tramitar), RE-016 (partidas del PDF proforma), RE-028 (partidas de la factura desde proforma)                                                             | Detalle 1:N de partidas de la proforma — mismo tratamiento que su cabecera: Scaffold directo                                                  |
| 31  | `catCobroEstatus`               | ✨ Nueva R16 (definida en RE-002)                           | RE-002 (crea), RE-008 (FK en fccPagoCliente), RE-026                                                                                                                            | Catálogo de estatus del ciclo de vida del cobro: BORRADOR → CAPTURADO → ASOCIADO / SALDO_A_FAVOR → COMPLETADO; CON_INCONSISTENCIA / CANCELADO |


> **Nota (R16A-RE-Cambio-PerfilFiscal):** Las tablas `catImpuesto`, `catTipoFactorSat`, `catObjetoImpuestoSat`, `PerfilFiscal` y `FamiliaRegion` **viven en ProquifaDotNet** (no son tablas propias de Finanzas). Finanzas las accede via Scaffold EF Core. Ver Sección 2.

### 2. Tablas existentes de ProquifaDotNet consumidas vía Scaffold (lectura / escritura puntual)

| Tabla                                                                    | Acceso    | Requisito(s)                    | Uso                                                                                                                                                                                                          |
| ------------------------------------------------------------------------ | --------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `catImpuesto`                                                            | Lectura   | RE-019 (Guía Técnica), RE-Cambio-PerfilFiscal | Catálogo c_Impuesto SAT: IVA, ISR, IEPS (renombrado de `catImpuestoSat`) |
| `catTipoFactorSat`                                                       | Lectura   | RE-019 (Guía Técnica), RE-Cambio-PerfilFiscal | Catálogo c_TipoFactor SAT: Tasa, Cuota, Exento |
| `catObjetoImpuestoSat`                                                   | Lectura   | RE-019 (Guía Técnica), RE-Cambio-PerfilFiscal | Catálogo c_ObjetoImp SAT: 01-No objeto, 02-Sí objeto, 03-Sí objeto no obligado, 04-Sí objeto IVA crédito IEPS |
| `PerfilFiscal`                                                           | Lectura   | RE-019 (Guía Técnica), RE-Cambio-PerfilFiscal | Perfil fiscal: combinación catImpuesto × catTipoFactorSat × TasaOCuota × catObjetoImpuestoSat × IdRegion |
| `FamiliaRegion`                                                          | Lectura   | RE-Cambio-PerfilFiscal          | Junction: Familia ↔ Region ↔ PerfilFiscal + ClaveProdServSat + ClaveUnidadSat |
| `Familia`                                                                | Lectura   | RE-019, RE-Cambio-PerfilFiscal  | Familia de productos (para resolución de ClaveProdServSat/ClaveUnidadSat via FamiliaRegion) |
| `Empresa`                                                                | Lectura   | RE-016, 019-022, 028-033        | Datos del emisor: `RFC`, `RazonSocial`, `Alias`, `Prefijo`, `IdCatRegimenFiscal` (→ `catRegimenFiscal.RegimenFiscal`), `IdDireccion` (→ `Direccion.CodigoPostal` = `LugarExpedicion`) + FK de `EmpresaFolio` |
| `Cliente`                                                                | Lectura   | RE-018+, 023-029                | Razón social, clave; joins de listados                                                                                                                                                                       |
| `DatosFacturacionCliente`                                                | Lectura   | RE-004, 012, 015, 019           | Snapshot fiscal del receptor al crear FAA/factura                                                                                                                                                            |
| `Archivo`                                                                | Escritura | RE-016, 019-022, 028-033        | Registro de PDF/XML subidos a MinIO (FileKey/FileBucket)                                                                                                                                                     |
| `RegionConfiguracionMinioBucket` / `Region`                              | Lectura   | RE-020, 022, 028-033            | Resolución de bucket por región; corte MEX/PER                                                                                                                                                               |
| `catMoneda`, `catUsoCFDI`, `catMetodoDePagoCFDI`, `catCondicionesDePago` | Lectura   | RE-005, 019, 024, 028           | Catálogos fiscales/comerciales existentes                                                                                                                                                                    |
| `CorreoEnviado` / `ArchivoCorreoEnviado`                                 | Escritura | RE-019, 020, 028, 030, 032, 033 | Trazabilidad de correos con PDF+XML adjuntos (envío vía ProquifaDotNet.EnvioCorreo)                                                                                                                          |
| `vUsuarioCartera`                                                        | Lectura   | RE-018, 023, 032                | Filtrado por cartera del Cobrador                                                                                                                                                                            |

### 3. Tablas con Controller/BO propio en ProquifaDotNet — consumo vía llamadas entre APIs (no Scaffold)

| Tabla | Requisito(s) | Uso |
|-------|--------------|-----|
| `tpPedido` | RE-012, 015, 028 | Folio interno, FacturaPorAdelantado, estado del pedido |
| `ClienteCartera` | RE-002, 023 | Cobrador asignado por cliente |

> Patrón de acceso (instrucciones del proyecto): las tablas que Finanzas lee/escribe directamente van al **Scaffold EF Core** de `Finanzas.Infrastructure`; las que ya tienen Controller/BO propio en ProquifaDotNet se consumen mediante **llamadas entre APIs**. Excepción decidida el 07-08/07/2026: `tpProformaPedido` y `tpProformaPartidaPedido` pasan al Scaffold de Finanzas (dejan de consumirse vía API).

---

## Diccionario de datos — tablas del Scaffold de Finanzas

> Una tabla por entidad: Columna, Tipo de Dato, Índice (PK/FK/UK) y Descripción. Tipos y llaves conforme al diagrama ER; el DDL completo vive en el `_BD.md` del requisito indicado en cada descripción.

### Tabla: `catImpuesto`

Catálogo c_Impuesto del SAT (IVA, ISR, IEPS). Renombrado de `catImpuestoSat` en R16A-RE-Cambio-PerfilFiscal. Tabla en ProquifaDotNet; Finanzas accede via Scaffold. Detalle: RE-019_BD (Guía Técnica).

| Columna                    | Tipo de Dato     | Índice | Descripción                  |
| -------------------------- | ---------------- | ------ | ---------------------------- |
| `IdCatImpuesto`            | uniqueidentifier | PK     | Identificador único (PK)     |
| `Clave`                    | varchar          | —      | Clave SAT (ej. `001` IVA, `002` ISR, `003` IEPS) |
| `Descripcion`              | varchar          | —      | Descripción legible          |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente) |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro   |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación |

### Tabla: `catTipoFactorSat`

Catálogo c_TipoFactor del SAT (Tasa, Cuota, Exento). Tabla en ProquifaDotNet; Finanzas accede via Scaffold. Detalle: RE-019_BD (Guía Técnica).

| Columna                    | Tipo de Dato     | Índice | Descripción                  |
| -------------------------- | ---------------- | ------ | ---------------------------- |
| `IdCatTipoFactorSat`       | uniqueidentifier | PK     | Identificador único (PK)     |
| `Clave`                    | varchar          | —      | Clave SAT (`Tasa`, `Cuota`, `Exento`) |
| `Descripcion`              | varchar          | —      | Descripción legible          |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente) |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro   |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación |

### Tabla: `catObjetoImpuestoSat`

Catálogo c_ObjetoImp del SAT (01-04). Tabla en ProquifaDotNet; Finanzas accede via Scaffold. Detalle: RE-019_BD (Guía Técnica).

| Columna                    | Tipo de Dato     | Índice | Descripción                  |
| -------------------------- | ---------------- | ------ | ---------------------------- |
| `IdCatObjetoImpuestoSat`   | uniqueidentifier | PK     | Identificador único (PK)     |
| `Clave`                    | varchar          | —      | Clave SAT (`01` No objeto, `02` Sí objeto, `03` Sí objeto no obligado, `04` Sí objeto IVA crédito IEPS) |
| `Descripcion`              | varchar          | —      | Descripción legible          |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente) |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro   |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación |

### Tabla: `PerfilFiscal`

Perfil fiscal reutilizable: combinación de catálogos SAT + región que define cómo se grava un concepto del CFDI (impuesto × factor × tasa × objeto de impuesto). Discriminado por `IdRegion` (MX/PE). Tabla en ProquifaDotNet; Finanzas accede via Scaffold. Detalle: RE-019_BD y R16A-RE-Cambio-PerfilFiscal_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                                                                                              |
| -------------------------- | ---------------- | ------ | -------------------------------------------------------------------------------------------------------- |
| `IdPerfilFiscal`           | uniqueidentifier | PK     | Identificador único (PK)                                                                                 |
| `IdRegion`                 | uniqueidentifier | FK     | FK → `Region` — discriminador MX/PE (NOT NULL)                                                           |
| `IdCatImpuesto`            | uniqueidentifier | FK     | FK → `catImpuesto` — tipo de impuesto (IVA/ISR/IEPS; NOT NULL)                                          |
| `IdCatTipoFactorSat`       | uniqueidentifier | FK     | FK → `catTipoFactorSat` — factor (Tasa/Cuota/Exento)                                                    |
| `TasaOCuota`               | decimal(6,6)     | —      | Tasa o cuota (`0.160000` para 16%); **NULL cuando TipoFactor = Exento**                                  |
| `IdCatObjetoImpuestoSat`   | uniqueidentifier | FK     | FK → `catObjetoImpuestoSat` — objeto del impuesto (01-04); solo MX                                      |
| `Fundamento`               | nvarchar         | —      | Referencia legal informativa (ej. `Art. 1 LIVA`, `Art. 2-A LIVA`, `Art. 9 LIVA`) — no afecta el cálculo |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente)                                                                             |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro                                                                               |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación                                                                             |

### Tabla: `FamiliaRegion`

Junction que asocia una Familia de productos a una Región con su PerfilFiscal correspondiente, `ClaveProdServSat` y `ClaveUnidadSat` por región. Tabla en ProquifaDotNet; Finanzas accede via Scaffold. Detalle: R16A-RE-Cambio-PerfilFiscal_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                                                  |
| -------------------------- | ---------------- | ------ | ------------------------------------------------------------ |
| `IdFamiliaRegion`          | uniqueidentifier | PK     | Identificador único (PK)                                     |
| `IdFamilia`                | uniqueidentifier | FK     | FK → `Familia`                                               |
| `IdRegion`                 | uniqueidentifier | FK     | FK → `Region` — discriminador MX/PE                          |
| `IdPerfilFiscal`           | uniqueidentifier | FK     | FK → `PerfilFiscal` — perfil fiscal aplicable en esta región |
| `ClaveProdServSat`         | varchar          | —      | Clave c_ClaveProdServ SAT para esta familia en esta región   |
| `ClaveUnidadSat`           | varchar          | —      | Clave c_ClaveUnidad SAT para esta familia en esta región     |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente)                                 |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro                                   |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación                                 |

### Tabla: `catTipoCFDI`

Catálogo de tipos de CFDI/CPE (FACTURA, FACTURA_ANTICIPO, COMPLEMENTO_PAGO, NOTA_CREDITO, FACTURA_CPE...). Detalle: RE-028_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                                 |
| -------------------------- | ---------------- | ------ | ------------------------------------------- |
| `IdCatTipoCFDI`            | uniqueidentifier | PK     | Identificador único (PK)                    |
| `Clave`                    | varchar          | —      | Clave programática                          |
| `Descripcion`              | varchar          | —      | Descripción legible                         |
| `TipoDocumentoSAT`         | varchar          | —      | Tipo de comprobante SAT (I/P/E) o CPE SUNAT |
| `IdRegion`                 | uniqueidentifier | FK     | FK → `Region`                               |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente)                |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro                  |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación                |

### Tabla: `catTipoDocumentoFiscal`

Catálogo de tipos de documento fiscal por línea del Paso 3 de Validar Cobro. Detalle: RE-028_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                  |
| -------------------------- | ---------------- | ------ | ---------------------------- |
| `IdCatTipoDocumentoFiscal` | uniqueidentifier | PK     | Identificador único (PK)     |
| `Clave`                    | varchar          | —      | Clave programática           |
| `Descripcion`              | varchar          | —      | Descripción legible          |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente) |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro   |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación |

### Tabla: `catDocumentoFiscalCobroEstado`

Catálogo de estados de línea del wizard VC Paso 3 (PENDIENTE → GENERADO → ENVIADO). Detalle: RE-028_BD.

| Columna                           | Tipo de Dato     | Índice | Descripción                  |
| --------------------------------- | ---------------- | ------ | ---------------------------- |
| `IdCatDocumentoFiscalCobroEstado` | uniqueidentifier | PK     | Identificador único (PK)     |
| `Clave`                           | varchar          | —      | Clave programática           |
| `Descripcion`                     | varchar          | —      | Descripción legible          |
| `Activo`                          | bit              | —      | Borrado lógico (1 = vigente) |
| `FechaRegistro`                   | datetime         | —      | Fecha de alta del registro   |
| `FechaUltimaActualizacion`        | datetime         | —      | Fecha de última modificación |

### Tabla: `catFacturaEstado`

Catálogo de estados del ciclo de vida de la Factura (7 estados con transiciones). Detalle: RE-015_BD v2.1.

| Columna                    | Tipo de Dato     | Índice | Descripción                                          |
| -------------------------- | ---------------- | ------ | ---------------------------------------------------- |
| `IdCatFacturaEstado`       | uniqueidentifier | PK     | Identificador único (PK)                             |
| `Clave`                    | varchar          | UK     | Clave programática (única)                           |
| `Descripcion`              | nvarchar         | —      | Descripción legible                                  |
| `Orden`                    | int              | —      | Orden natural del ciclo de vida (UI/reportes)        |
| `EsTerminal`               | bit              | —      | 1 = sin transiciones posteriores (PAGADA, CANCELADA) |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente)                         |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro                           |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación                         |

### Tabla: `catFormaPagoSAT`

Catálogo c_FormaPago del SAT para la captura del cobro. Detalle: RE-024_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                  |
| -------------------------- | ---------------- | ------ | ---------------------------- |
| `IdCatFormaPagoSAT`        | uniqueidentifier | PK     | Identificador único (PK)     |
| `Clave`                    | varchar          | —      | Clave programática           |
| `Descripcion`              | varchar          | —      | Descripción legible          |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente) |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro   |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación |

### Tabla: `catTipoOperacionSUNAT`

Catálogo 51 SUNAT — Tipo de Operación (Perú). Detalle: RE-020/029_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                  |
| -------------------------- | ---------------- | ------ | ---------------------------- |
| `IdCatTipoOperacionSUNAT`  | uniqueidentifier | PK     | Identificador único (PK)     |
| `Clave`                    | varchar          | —      | Clave programática           |
| `Descripcion`              | varchar          | —      | Descripción legible          |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente) |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro   |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación |

### Tabla: `catMotivoCancelacionSAT`

Catálogo de motivos de cancelación SAT (01-04). Detalle: RE-032_BD.

| Columna                     | Tipo de Dato     | Índice | Descripción                  |
| --------------------------- | ---------------- | ------ | ---------------------------- |
| `IdCatMotivoCancelacionSAT` | uniqueidentifier | PK     | Identificador único (PK)     |
| `Clave`                     | varchar          | —      | Clave programática           |
| `Descripcion`               | varchar          | —      | Descripción legible          |
| `Activo`                    | bit              | —      | Borrado lógico (1 = vigente) |
| `FechaRegistro`             | datetime         | —      | Fecha de alta del registro   |
| `FechaUltimaActualizacion`  | datetime         | —      | Fecha de última modificación |

### Tabla: `catMotivoCreditoSUNAT09`

Catálogo 09 SUNAT — motivos de Nota de Crédito (Perú). Detalle: RE-033_BD.

| Columna                     | Tipo de Dato     | Índice | Descripción                                |
| --------------------------- | ---------------- | ------ | ------------------------------------------ |
| `IdCatMotivoCreditoSUNAT09` | uniqueidentifier | PK     | Identificador único (PK)                   |
| `Clave`                     | varchar          | —      | Clave programática                         |
| `Descripcion`               | varchar          | —      | Descripción legible                        |
| `Modalidad`                 | varchar          | —      | Modalidad de la NC según catálogo 09 SUNAT |
| `Activo`                    | bit              | —      | Borrado lógico (1 = vigente)               |
| `FechaRegistro`             | datetime         | —      | Fecha de alta del registro                 |
| `FechaUltimaActualizacion`  | datetime         | —      | Fecha de última modificación               |

### Tabla: `catTipoInconsistenciaCobro`

Catálogo de tipos de inconsistencia de cobro (Pasos 1 y 2) — seed pendiente Tesorería. Detalle: RE-024/026_BD.

| Columna                          | Tipo de Dato     | Índice | Descripción                                                   |
| -------------------------------- | ---------------- | ------ | ------------------------------------------------------------- |
| `IdCatTipoInconsistenciaCobro`   | uniqueidentifier | PK     | Identificador único (PK)                                      |
| `TipoInconsistencia`             | varchar          | —      | Nombre del tipo de inconsistencia                             |
| `AplicaMarkPendienteCancelacion` | bit              | —      | 1 = marca el pedido "Pendiente cancelación por falta de pago" |
| `Activo`                         | bit              | —      | Borrado lógico (1 = vigente)                                  |
| `FechaRegistro`                  | datetime         | —      | Fecha de alta del registro                                    |
| `FechaUltimaActualizacion`       | datetime         | —      | Fecha de última modificación                                  |

### Tabla: `EmpresaFolio`

Foliador por empresa/serie; consumo UPDLOCK atómico al timbrar. Movida de ProquifaDotNetTimbrado a Finanzas (07/07/2026). Detalle: RE-019_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                                                             |
| -------------------------- | ---------------- | ------ | ----------------------------------------------------------------------- |
| `IdEmpresaFolio`           | uniqueidentifier | PK     | Identificador único (PK)                                                |
| `IdEmpresa`                | uniqueidentifier | FK     | FK → `Empresa`                                                          |
| `Serie`                    | varchar          | —      | Serie del foliador (factura por empresa; 'P' CP, 'P2' NC, F001 GOLPERU) |
| `UltimoFolio`              | int              | —      | Último folio consumido (UPDATE con UPDLOCK atómico)                     |
| `FormatoFolio`             | varchar          | —      | Formato de presentación del folio                                       |
| `LongitudMaxima`           | int              | —      | Longitud máxima del folio (varchar 6 en factura)                        |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente)                                            |
| `FechaRegistro`            | datetime        | —      | Fecha de alta del registro                                              |
| `FechaUltimaActualizacion` | datetime        | —      | Fecha de última actualización                                           |

### Tabla: `CFDIGenerada`

Registro central de negocio de todo CFDI/CPE timbrado — single source of truth (Serie, Folio, UUID). Detalle: RE-019/021/028_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                                                      |
| -------------------------- | ---------------- | ------ | ---------------------------------------------------------------- |
| `IdCFDIGenerada`           | uniqueidentifier | PK     | Identificador único (PK)                                                            |
| `RFCEmisor`                | varchar          | —      | RFC de la empresa emisora                                                           |
| `RFCReceptor`              | varchar          | —      | RFC del cliente receptor                                                            |
| `Serie`                    | varchar          | —      | Serie del documento                                                                 |
| `Folio`                    | varchar          | —      | Folio consecutivo                                                                   |
| `FechaEmision`             | datetime        | —      | Fecha de emisión del CFDI                                                           |
| `FechaCertificacionSat`    | datetime        | —      | Fecha de certificación (timbrado) por el PAC/SAT (RE-021)                           |
| `IdCatTipoCFDI`            | uniqueidentifier | FK     | FK → `catTipoCFDI`                                                                  |
| `IdCFDIRelacionado`        | uniqueidentifier | FK     | FK → `CFDIGenerada`                                                                 |
| `UUID`                     | varchar          | —      | Folio fiscal (UUID) asignado por SAT                                                |
| `Total`                    | decimal          | —      | Total del comprobante                                                               |
| `IdCatMetodoDePagoCFDI`    | uniqueidentifier | FK     | FK → `catMetodoDePagoCFDI` — c_MetodoPago SAT: `PPD` (FAA/crédito) o `PUE` (pago único) |
| `IdCatFormaPagoSAT`        | uniqueidentifier | FK     | FK → `catFormaPagoSAT` — c_FormaPago SAT (`99` en FAA; forma real en CP)           |
| `Exportacion`              | varchar(2)       | —      | c_Exportacion SAT — default `01` (No aplica); requerido en CFDI 4.0                |
| `Estado`                   | varchar          | —      | Estado técnico del timbrado ('Timbrado', 'Fallido', 'Cancelado')                    |
| `IdArchivoXml`             | uniqueidentifier | FK     | FK → `Archivo`                                                                      |
| `IdArchivoPdf`             | uniqueidentifier | FK     | FK → `Archivo`                                                                      |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente)                                                        |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro                                                          |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación                                                        |

### Tabla: `CFDIGeneradaConcepto`

Conceptos (partidas fiscales) del CFDI. Detalle: RE-021_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                  |
| -------------------------- | ---------------- | ------ | ---------------------------- |
| `IdCFDIGeneradaConcepto`   | uniqueidentifier | PK     | Identificador único (PK)     |
| `IdCFDIGenerada`           | uniqueidentifier | FK     | FK → `CFDIGenerada`          |
| `ClaveProdServ`            | varchar          | —      | Catálogo c_ClaveProdServ SAT |
| `Descripcion`              | varchar          | —      | Descripción legible          |
| `Cantidad`                 | decimal          | —      | Cantidad del concepto        |
| `ValorUnitario`            | decimal          | —      | Valor unitario               |
| `Importe`                  | decimal          | —      | Importe del concepto         |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro   |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación |

### Tabla: `CFDIGeneradaRelacionado`

Nodo CFDIRelacionados del CFDI (NCs aplicadas, factura origen). Detalle: RE-028/032_BD.

| Columna                     | Tipo de Dato     | Índice | Descripción                          |
| --------------------------- | ---------------- | ------ | ------------------------------------ |
| `IdCFDIGeneradaRelacionado` | uniqueidentifier | PK     | Identificador único (PK)             |
| `IdCFDIGenerada`            | uniqueidentifier | FK     | FK → `CFDIGenerada`                  |
| `UUID`                      | varchar          | —      | Folio fiscal (UUID) asignado por SAT |
| `ClaveTipoRelacion`         | varchar          | —      | c_TipoRelacion SAT (01, 03, 07...)   |
| `FechaRegistro`             | datetime         | —      | Fecha de alta del registro           |
| `FechaUltimaActualizacion`  | datetime         | —      | Fecha de última modificación         |

### Tabla: `CFDICancelacion`

Cancelaciones de CFDI ante el SAT (motivo, estado, acuse). Detalle: pre-R16 / RE-032_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                                |
| -------------------------- | ---------------- | ------ | ------------------------------------------ |
| `IdCFDICancelacion`        | uniqueidentifier | PK     | Identificador único (PK)                   |
| `IdCFDIGenerada`           | uniqueidentifier | FK     | FK → `CFDIGenerada`                        |
| `ClaveMotivo`              | varchar          | —      | Motivo de cancelación SAT (01-04)          |
| `UUID`                     | varchar          | —      | Folio fiscal (UUID) asignado por SAT       |
| `FechaSolicitud`           | datetime         | —      | Fecha de solicitud de cancelación          |
| `EstadoCancelacion`        | varchar          | —      | Estado del proceso de cancelación ante SAT |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro                 |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación               |

### Tabla: `tpProformaPedido`

Proforma/Confirmación de Pedido — tabla existente verificada contra BD; movida al Scaffold de Finanzas (07/07/2026). ALTER R16: `FolioProforma` + `ConsecutivoProforma` (RE-016). La fecha estimada de pago de Gestionar Cobranza (RE-023) reutiliza la columna existente `FechaPromesaPagoMonitoreoCobros`; el flag de controlados es la columna existente `Controlados`. Detalle: RE-016/023/026/028_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdTPProformaPedido` | uniqueidentifier | PK | Identificador único (PK) |
| `IdCliente` | uniqueidentifier | FK | FK → `Cliente` |
| `IdEmpresa` | uniqueidentifier | FK | FK → `Empresa` |
| `ReferenciaPago` | varchar | — | Referencia bancaria casada al PDF (snapshot inmutable, RE-006) |
| `NumeroFactura` | varchar | — | Número de factura asociada |
| `MontoTotal` | decimal | — | Monto total de la proforma |
| `MontoPagado` | decimal | — | Monto acumulado pagado |
| `MontoPendiente` | decimal | — | Saldo pendiente por cobrar; se reduce al asociar cobros (RE-026) |
| `FechaCompromisoPago` | datetime | — | Fecha compromiso de pago |
| `FechaPromesaPagoMonitoreoCobros` | datetime | — | Fecha estimada de pago — editable en Gestionar Cobranza; cada cambio genera INSERT en `fccFechaEstimadaPagoHistorial` (RE-023) |
| `FechaPagoCompleto` | datetime | — | Fecha en que se completó el pago |
| `FacturaFlete` | bit | — | 1 = incluye factura de flete |
| `IdCFDIGenerada` | uniqueidentifier | FK | FK → `CFDIGenerada` — CFDI timbrado de la proforma (RE-028) |
| `Factura` | bit | — | 1 = ya facturada |
| `MXN` | bit | — | Flag legacy: moneda pesos |
| `USD` | bit | — | Flag legacy: moneda dólares |
| `Folio` | varchar | — | Folio del documento |
| `Serie` | varchar | — | Serie del documento |
| `Uuid` | varchar | — | UUID fiscal asociado (legacy) |
| `IdCFDI` | uniqueidentifier | FK | FK → `CFDI` (legacy) |
| `Cancelada` | bit | — | 1 = proforma cancelada (Cancelar Pedido, RE-023) |
| `IdTPProformaPedidoReemplazo` | uniqueidentifier | FK | FK → `tpProformaPedido` — proforma que la reemplaza |
| `PrecioFleteKPI` | decimal | — | Precio de flete para KPI |
| `Revisada` | bit | — | 1 = revisada |
| `Contrarecibo` | bit | — | 1 = con contrarecibo |
| `Controlados` | bit | — | 1 = proforma con sustancias controladas → FACTURA_ANTICIPO (RE-013/014, RE-028) |
| `Comentarios` | varchar | — | Comentarios de la proforma |
| `Publicaciones` | bit | — | 1 = incluye publicaciones |
| `IdContactoCliente` | uniqueidentifier | FK | FK → `ContactoCliente` |
| `IdDireccionCliente` | uniqueidentifier | FK | FK → `DireccionCliente` |
| `NumeroOrdenDeCompra` | varchar | — | Orden de compra del cliente |
| `FolioProforma` | varchar | — | ✨ ALTER R16 — Folio global de proforma PRF-MMDDAA-# (RE-016, `SeqFolioProforma`) |
| `ConsecutivoProforma` | int | — | ✨ ALTER R16 — Consecutivo del foliador global (RE-016) — DEFAULT (0) |
| `Activo` | bit | — | Borrado lógico (1 = vigente) |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `tpProformaPartidaPedido`

Partidas de la proforma (1:N) — tabla existente verificada contra BD; movida al Scaffold de Finanzas (08/07/2026), sin cambios de estructura en R16. Detalle: RE-013/014_BD.

| Columna                     | Tipo de Dato     | Índice | Descripción                                        |
| --------------------------- | ---------------- | ------ | -------------------------------------------------- |
| `IdTPProformaPartidaPedido` | uniqueidentifier | PK     | Identificador único (PK)                           |
| `IdTPProformaPedido`        | uniqueidentifier | FK     | FK → `tpProformaPedido`                            |
| `IdTPPartidaPedido`         | uniqueidentifier | FK     | FK → `tpPartidaPedido` — partida del pedido origen |
| `IdEmpresaCompra`           | uniqueidentifier | FK     | FK → `Empresa` — empresa de compra                 |
| `IdProducto`                | uniqueidentifier | FK     | FK → `Producto`                                    |
| `NumeroDePiezas`            | int              | —      | Cantidad de piezas de la partida                   |
| `PrecioUnitario`            | decimal          | —      | Precio unitario                                    |
| `IdLote`                    | uniqueidentifier | FK     | FK → `Lote`                                        |
| `Pedimento`                 | varchar          | —      | Pedimento aduanal                                  |
| `PrecioFleteParcialidad`    | decimal          | —      | Precio de flete de la parcialidad                  |
| `Activo`                    | bit              | —      | Borrado lógico (1 = vigente)                       |
| `FechaRegistro`             | datetime         | —      | Fecha de alta del registro                         |
| `FechaUltimaActualizacion`  | datetime         | —      | Fecha de última modificación                       |

### Tabla: `catCobroEstatus`

Catálogo de estatus del ciclo de vida del cobro (BORRADOR → CAPTURADO → ASOCIADO / SALDO_A_FAVOR → COMPLETADO; CON_INCONSISTENCIA / CANCELADO). Detalle: RE-002_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                                      |
| -------------------------- | ---------------- | ------ | ------------------------------------------------ |
| `IdCatCobroEstatus`        | uniqueidentifier | PK     | Identificador único (PK)                         |
| `Clave`                    | varchar          | UK     | Clave programática (única)                       |
| `Descripcion`              | varchar          | —      | Descripción legible                              |
| `Orden`                    | int              | —      | Orden natural del ciclo de vida (UI/reportes)    |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente)                     |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro — DEFAULT GETDATE()   |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación — DEFAULT GETDATE() |

### Tabla: `fccFolioPagoCliente`

Pendiente en Validar Cobro generado automáticamente al clasificar un correo como Cobro en el Buzón (Mailbot o manual). Estructura verificada contra BD (✅ existente, sin cambios R16). El estado del pendiente se rastrea con `Activo` (1 = abierto, 0 = cerrado al vincular a proforma/factura o marcar inconsistencia); el estatus del ciclo de vida del cobro vive en `fccPagoCliente.IdCatCobroEstatus`. Detalle: RE-008/023_BD.

| Columna                    | Tipo de Dato     | Índice | Descripción                                                              |
| -------------------------- | ---------------- | ------ | ------------------------------------------------------------------------ |
| `IdFCCFolioPagoCliente`    | uniqueidentifier | PK     | Identificador único (PK) — DEFAULT NEWID()                               |
| `IdCorreoRecibidoCliente`  | uniqueidentifier | FK     | FK → `CorreoRecibidoCliente` — correo del Buzón que originó el pendiente |
| `IdArchivo`                | uniqueidentifier | FK     | FK → `Archivo` — adjunto del comprobante (nullable)                      |
| `Folio`                    | varchar          | —      | Folio de referencia interna                                              |
| `Consecutivo`              | int              | —      | Consecutivo del folio                                                    |
| `FechaRecepcion`           | datetime         | —      | Fecha de recepción del correo origen                                     |
| `Stp`                      | bit              | —      | 1 = cobro vía STP — DEFAULT (0)                                          |
| `SubtotalMailBot`          | decimal          | —      | Subtotal pre-extraído por Mailbot                                        |
| `IvaMailBot`               | decimal          | —      | IVA pre-extraído por Mailbot                                             |
| `TotalMailBot`             | decimal          | —      | Total pre-extraído por Mailbot                                           |
| `MxnMailBot`               | bit              | —      | Moneda MXN detectada por Mailbot                                         |
| `UsdMailBot`               | bit              | —      | Moneda USD detectada por Mailbot                                         |
| `Activo`                   | bit              | —      | 1 = pendiente abierto / **0 = cerrado** — DEFAULT (1)                    |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro — DEFAULT GETDATE()                           |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación — DEFAULT GETDATE()                         |

### Tabla: `fccPagoCliente`

Cobro capturado en Validar Cobro Paso 1 (folio COB-mmddaa-#; inmutable al confirmar). Tabla existente verificada contra BD + ALTERs R16: `IdCatCobroEstatus` (RE-008) y `Confirmado`, `FechaConfirmacion`, `IdUsuarioConfirmacion`, `Notas`, `IdCatMoneda` (RE-023). Detalle: RE-023/024_BD y RE-008_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFCCPagoCliente` | uniqueidentifier | PK | Identificador único (PK) — DEFAULT NEWID() |
| `IdCliente` | uniqueidentifier | FK | FK → `Cliente` |
| `IdEmpresa` | uniqueidentifier | FK | FK → `Empresa` — empresa PROQUIFA que recibe el cobro |
| `IdContactoCliente` | uniqueidentifier | FK | FK → `ContactoCliente` (nullable) |
| `IdFCCFolioPagoCliente` | uniqueidentifier | FK | FK → `fccFolioPagoCliente` — pendiente del Buzón que originó el cobro |
| `IdCatCobroEstatus` | uniqueidentifier | FK | FK → `catCobroEstatus` — estatus del ciclo de vida del cobro (ALTER RE-008) |
| `Monto` | decimal | — | Monto recibido del cliente |
| `FechaPago` | datetime | — | Fecha efectiva del pago |
| `IdCatMoneda` | uniqueidentifier | FK | FK → `catMoneda` — moneda oficial del cobro (ALTER RE-023; soporta PEN) |
| `MXN` | bit | — | Flag legacy: cobro en pesos — DEFAULT (0) |
| `USD` | bit | — | Flag legacy: cobro en dólares — DEFAULT (1) |
| `TipoDeCambio` | decimal | — | TC del día vs MXN — DEFAULT (0) |
| `IdCatMedioDePago` | uniqueidentifier | FK | FK → `catMedioDePago` — forma de pago (c_FormaPago SAT) |
| `IdDatosBancarios` | uniqueidentifier | FK | FK → `DatosBancarios` — cuenta PROQUIFA destino |
| `IdCatBanco` | uniqueidentifier | FK | FK → `catBanco` — banco emisor del cliente |
| `CuentaOrdenante` | varchar | — | Cuenta del cliente emisor |
| `ReferenciaBancaria` | varchar | — | Referencia bancaria del cobro |
| `Broker` | bit | — | 1 = cobro vía Broker — DEFAULT (0) |
| `IdCatBrokerCliente` | uniqueidentifier | FK | FK → `catBroker` — broker del cliente (cuando aplica) |
| `InformacionComplementoPago` | bit | — | 1 = requiere complemento de pago — DEFAULT (0) |
| `IdCFDI` | uniqueidentifier | FK | FK → `CFDI` (legacy) — documento fiscal asociado |
| `Folio` | varchar | — | Folio COB-mmddaa-NNNNNN — NULL en borrador; se llena al confirmar (RE-024) |
| `Serie` | varchar | — | Serie del comprobante fiscal |
| `IdArchivo` | uniqueidentifier | FK | FK → `Archivo` — comprobante de pago seleccionado del correo |
| `Confirmado` | bit | — | 0 = borrador / 1 = confirmado e inmutable (ALTER RE-023, RE-024) |
| `FechaConfirmacion` | datetime | — | Timestamp de confirmación del cobro (ALTER RE-023) |
| `IdUsuarioConfirmacion` | uniqueidentifier | FK | FK → `Usuario` — quién confirmó el cobro (ALTER RE-023) |
| `Notas` | varchar | — | Notas capturadas por el usuario (ALTER RE-023) |
| `Activo` | bit | — | 1 = vigente / **0 = inconsistencia** (cierra el pendiente del Buzón) |
| `FechaRegistro` | datetime | — | Fecha de alta del registro — DEFAULT GETDATE() |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación — DEFAULT GETDATE() |

### Tabla: `fccPagoFacturaPedido`

Asociación N:N cobro ↔ proforma (Paso 2). Detalle: RE-026_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFCCPagoFacturaPedido` | uniqueidentifier | PK | Identificador único (PK) |
| `IdFCCPagoCliente` | uniqueidentifier | FK | FK → `fccPagoCliente` |
| `IdTPProformaPedido` | uniqueidentifier | FK | FK → `tpProformaPedido` |
| `Monto` | decimal | — | Monto de la operación |
| `MontoPendienteAnterior` | decimal | — | Saldo pendiente de la proforma antes de aplicar este cobro |
| `NumeroDeParcialidad` | int | — | Número de parcialidad aplicada |
| `FechaAplicacion` | datetime | — | Fecha de aplicación del cobro al documento |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `fccFactura`

Cabecera única FAA + factura final (EsFacturaPorAdelantado); reemplaza tpProformaAdelanto. Detalle: RE-015_BD.

| Columna                  | Tipo de Dato     | Índice | Descripción                                    |
| ------------------------ | ---------------- | ------ | ---------------------------------------------- |
| `IdFccFactura`           | uniqueidentifier | PK     | Identificador único (PK)                       |
| `IdTPPedido`             | uniqueidentifier | FK     | FK → `tpPedido`                                |
| `IdTPProformaPedido`     | uniqueidentifier | FK     | FK → `tpProformaPedido`                        |
| `IdCatFacturaEstado`     | uniqueidentifier | FK     | FK → `catFacturaEstado`                        |
| `EsFacturaPorAdelantado` | bit              | —      | 1 = FAA, 0 = factura final (RT-10)             |
| `Enviada`                | bit              | —      | 1 = enviada al cliente con PDF+XML             |
| `FechaEnvio`             | datetime        | —      | Fecha y hora (UTC) del envío al cliente (v2.1) |
| `IdCliente`              | uniqueidentifier | FK     | FK → `Cliente`                                 |
| `IdEmpresa`              | uniqueidentifier | FK     | FK → `Empresa`                                 |
| `FolioPedidoInterno`     | varchar          | —      | Folio interno del pedido                       |
| `MontoTotal`             | decimal          | —      | Monto total                                    |
| `IdCatMoneda`            | uniqueidentifier | FK     | FK → `catMoneda`                               |
| `IdCFDIGenerada`           | uniqueidentifier | FK     | FK → `CFDIGenerada`                            |
| `Activo`                   | bit              | —      | Borrado lógico (1 = vigente)                   |
| `FechaRegistro`            | datetime         | —      | Fecha de alta del registro                     |
| `FechaUltimaActualizacion` | datetime         | —      | Fecha de última modificación                   |

### Tabla: `fccFacturaPartida`

Partidas snapshot del pedido (1:N de fccFactura). Detalle: RE-015_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFccFacturaPartida` | uniqueidentifier | PK | Identificador único (PK) |
| `IdFccFactura` | uniqueidentifier | FK | FK → `fccFactura` |
| `Descripcion` | varchar | — | Descripción legible |
| `Cantidad` | decimal | — | Cantidad de la partida |
| `ValorUnitario` | decimal | — | Valor unitario |
| `Importe` | decimal | — | Importe de la partida |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `fccFacturaReferenciaBancaria`

Cuentas M.N./DLS + referencia del cliente (Código Validador RE-006). Detalle: RE-015_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFccFacturaReferenciaBancaria` | uniqueidentifier | PK | Identificador único (PK) |
| `IdFccFactura` | uniqueidentifier | FK | FK → `fccFactura` |
| `IdCatMoneda` | uniqueidentifier | FK | FK → `catMoneda` |
| `Banco` | varchar | — | Banco (de catBanco) |
| `NumeroCuenta` | varchar | — | Número de cuenta |
| `Clabe` | varchar | — | CLABE interbancaria |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `fccPagoFacturaAdelanto`

Asociación N:N cobro ↔ Factura por Adelantado. Detalle: RE-026_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFCCPagoFacturaAdelanto` | uniqueidentifier | PK | Identificador único (PK) |
| `IdFCCPagoCliente` | uniqueidentifier | FK | FK → `fccPagoCliente` |
| `IdTPProformaPedido` | uniqueidentifier | FK | FK → `tpProformaPedido` |
| `IdFccFactura` | uniqueidentifier | FK | FK → `fccFactura` |
| `IdCFDIGenerada` | uniqueidentifier | FK | FK → `CFDIGenerada` |
| `Monto` | decimal | — | Monto de la operación |
| `NumeroParcialidad` | int | — | Número de parcialidad aplicada |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `fccDocumentoFiscalCobro`

Línea de documento fiscal a emitir por cobro — snapshot CFDI 4.0 / Pagos20 (VC Paso 3). Detalle: RE-028_BD.

| Columna                           | Tipo de Dato     | Índice | Descripción                                        |
| --------------------------------- | ---------------- | ------ | -------------------------------------------------- |
| `IdFCCDocumentoFiscalCobro`       | uniqueidentifier | PK     | Identificador único (PK)                           |
| `IdFCCPagoFacturaPedido`          | uniqueidentifier | FK     | FK → `fccPagoFacturaPedido`                        |
| `IdFCCPagoFacturaAdelanto`        | uniqueidentifier | FK     | FK → `fccPagoFacturaAdelanto`                      |
| `IdCatTipoDocumentoFiscal`        | uniqueidentifier | FK     | FK → `catTipoDocumentoFiscal`                      |
| `IdCatDocumentoFiscalCobroEstado` | uniqueidentifier | FK     | FK → `catDocumentoFiscalCobroEstado`               |
| `IdCatUsoCFDI`                    | uniqueidentifier | FK     | FK → `catUsoCFDI`                                  |
| `IdCatMetodoDePagoCFDI`           | uniqueidentifier | FK     | FK → `catMetodoDePagoCFDI`                         |
| `IdCFDIGeneradaFactura`           | uniqueidentifier | FK     | FK → `CFDIGenerada`                                |
| `IdCFDIGeneradaComplemento`       | uniqueidentifier | FK     | FK → `CFDIGenerada`                                |
| `IdCatTipoOperacionSUNAT`         | uniqueidentifier | FK     | FK → `catTipoOperacionSUNAT`                       |
| `IdCatCondicionesDePago`          | uniqueidentifier | FK     | FK → `catCondicionesDePago`                        |
| `FechaPagoCP`                     | datetime        | —      | Fecha de pago del Complemento (nodo Pagos)         |
| `IdCatFormaPagoSAT`               | uniqueidentifier | FK     | FK → `catFormaPagoSAT`                             |
| `TipoCambioP_CP`                  | decimal          | —      | Tipo de cambio P del Complemento de Pago           |
| `NumParcialidad`                  | int              | —      | Parcialidad del DoctoRelacionado (UPDLOCK, RE-030) |
| `ImpSaldoAnt`                     | decimal          | —      | Saldo anterior del DoctoRelacionado                |
| `ImpPagado`                       | decimal          | —      | Importe pagado del DoctoRelacionado                |
| `ImpSaldoInsoluto`                | decimal          | —      | Saldo insoluto del DoctoRelacionado                |
| `EquivalenciaDR`                  | decimal          | —      | EquivalenciaDR multi-divisa (Pagos20)              |
| `FechaGeneracion`                 | datetime        | —      | Fecha de timbrado exitoso de la línea              |
| `FechaEnvio`                      | datetime        | —      | Fecha de envío de la línea al cliente              |
| `Activo`                          | bit              | —      | Borrado lógico (1 = vigente)                       |
| `FechaRegistro`                   | datetime         | —      | Fecha de alta del registro                         |
| `FechaUltimaActualizacion`        | datetime         | —      | Fecha de última modificación                       |

### Tabla: `fccConfirmacionPedido`

Confirmación de pedido del Paso 3. Detalle: RE-028_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFCCConfirmacionPedido` | uniqueidentifier | PK | Identificador único (PK) |
| `IdTPProformaPedido` | uniqueidentifier | FK | FK → `tpProformaPedido` |
| `IdUsuario` | uniqueidentifier | FK | FK → `Usuario` |
| `FechaConfirmacion` | datetime | — | Fecha de confirmación del pedido |
| `Activo` | bit | — | Borrado lógico (1 = vigente) |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `fccSaldoFavorCliente`

Saldo a favor / tolerancia ≤100 MXN — Estado de Cuenta del cliente. Detalle: RE-026_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFCCSaldoFavorCliente` | uniqueidentifier | PK | Identificador único (PK) |
| `IdCliente` | uniqueidentifier | FK | FK → `Cliente` |
| `IdFCCPagoCliente` | uniqueidentifier | FK | FK → `fccPagoCliente` |
| `TipoSaldo` | varchar | — | 'SaldoFavor' (sobrepago) o 'ToleranciaAplicada' (≤100 MXN) |
| `Monto` | decimal | — | Monto de la operación |
| `MXN` | bit | — | 1 = saldo en pesos |
| `USD` | bit | — | 1 = saldo en dólares |
| `TipoCambio` | decimal | — | Tipo de cambio aplicado |
| `Aplicado` | bit | — | 1 = saldo ya aplicado en una sesión posterior |
| `IdFCCPagoFacturaPedido` | uniqueidentifier | FK | FK → `fccPagoFacturaPedido` |
| `Observaciones` | varchar | — | Observaciones del operador |
| `Activo` | bit | — | Borrado lógico (1 = vigente) |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `fccInconsistenciaCobro`

Inconsistencias marcadas sobre un cobro (Pasos 1 y 2). Detalle: RE-024/026_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFCCInconsistenciaCobro` | uniqueidentifier | PK | Identificador único (PK) |
| `IdFCCPagoCliente` | uniqueidentifier | FK | FK → `fccPagoCliente` |
| `IdCatTipoInconsistenciaCobro` | uniqueidentifier | FK | FK → `catTipoInconsistenciaCobro` |
| `Notas` | varchar | — | Notas capturadas por el usuario |
| `IdUsuario` | uniqueidentifier | FK | FK → `Usuario` |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `fccNotaCredito`

Notas de Crédito (México CFDI E / Perú CPE 07); vínculo a cobro y factura origen. Detalle: RE-026/032_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFCCNotaCredito` | uniqueidentifier | PK | Identificador único (PK) |
| `IdTPProformaPedido` | uniqueidentifier | FK | FK → `tpProformaPedido` |
| `IdCFDI` | uniqueidentifier | FK | FK → `CFDI (legacy)` |
| `IdCFDIGenerada` | uniqueidentifier | FK | FK → `CFDIGenerada` |
| `IdEmpresa` | uniqueidentifier | FK | FK → `Empresa` |
| `IdCliente` | uniqueidentifier | FK | FK → `Cliente` |
| `Monto` | decimal | — | Monto de la operación |
| `Serie` | varchar | — | Serie del documento |
| `Modalidad` | varchar | — | Modalidad de captura: 'Partidas' o 'Manual' |
| `Motivo` | varchar | — | Motivo de la NC (México texto / Perú catálogo 09) |
| `Estado` | varchar | — | Estado de la NC ('VIGENTE', 'Aplicada'...) |
| `CancelarFacturaOrigen` | bit | — | 1 = se solicitó cancelar la factura origen ante SAT |
| `IdCatMotivoCancelacionSAT` | uniqueidentifier | FK | FK → `catMotivoCancelacionSAT` |
| `IdCFDIGeneradaFacturaOrigen` | uniqueidentifier | FK | FK → `CFDIGenerada` |
| `ConceptoManual` | varchar | — | Concepto capturado en modalidad Manual |
| `ObservacionesManual` | varchar | — | Observaciones de la modalidad Manual |
| `IdArchivoXml` | uniqueidentifier | FK | FK → `Archivo` |
| `IdArchivoPdf` | uniqueidentifier | FK | FK → `Archivo` |
| `ResponseCode` | varchar | — | Código de respuesta del PAC/OSE |
| `ResponseDescription` | nvarchar | — | Descripción de la respuesta del PAC/OSE |
| `TipoCambioOrigen` | decimal | — | TC de la factura origen (aplicación multi-divisa) |
| `IdFCCPagoCliente` | uniqueidentifier | FK | FK → `fccPagoCliente` |
| `Activo` | bit | — | Borrado lógico (1 = vigente) |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `fccNotaCreditoPartida`

Detalle por partidas de la Nota de Crédito. Detalle: RE-032_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFCCNotaCreditoPartida` | uniqueidentifier | PK | Identificador único (PK) |
| `IdFCCNotaCredito` | uniqueidentifier | FK | FK → `fccNotaCredito` |
| `IdCFDIGeneradaConceptoOrigen` | uniqueidentifier | FK | FK → `CFDIGeneradaConcepto` |
| `CantidadNC` | decimal | — | Cantidad acreditada de la partida origen |
| `Importe` | decimal | — | Importe de la partida NC |
| `Subtotal` | decimal | — | Subtotal de la partida NC |
| `IVA` | decimal | — | IVA de la partida NC |
| `Total` | decimal | — | Total de la partida NC |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

### Tabla: `fccFechaEstimadaPagoHistorial`

Historial de cambios de la fecha estimada de pago (Gestionar Cobranza). Detalle: RE-023_BD.

| Columna | Tipo de Dato | Índice | Descripción |
|---------|--------------|--------|-------------|
| `IdFccFechaEstimadaPagoHistorial` | uniqueidentifier | PK | Identificador único (PK) |
| `IdTpProformaPedido` | uniqueidentifier | FK | FK → `tpProformaPedido` |
| `FechaEstimadaPagoAnterior` | datetime | — | Fecha estimada anterior |
| `FechaEstimadaPagaNueva` | datetime | — | Fecha estimada nueva |
| `FechaCambio` | datetime | — | Fecha del cambio |
| `IdUsuarioCambio` | uniqueidentifier | FK | FK → `Usuario` |
| `Motivo` | varchar | — | Motivo del cambio |
| `FechaRegistro` | datetime | — | Fecha de alta del registro |
| `FechaUltimaActualizacion` | datetime | — | Fecha de última modificación |

---

## Scaffold EF Core — `Finanzas.Infrastructure`

Comando para generar/regenerar el Scaffold con **todas las tablas y vistas que consume Finanzas** (secciones 1 y 2 del listado). Ejecutar desde la raíz de la solución `ProquifaDotNet.Finanzas`; es **incremental por requisito** — al agregar una tabla nueva, añadir su `--table` y re-ejecutar con `--force`.

```bash
dotnet ef dbcontext scaffold \
  "Data Source=RYNL010;Initial Catalog=ProquifaDotNet;Integrated Security=True;Persist Security Info=False;Pooling=False;MultipleActiveResultSets=False;Encrypt=False;TrustServerCertificate=True;" \
  Microsoft.EntityFrameworkCore.SqlServer \
  --project Infrastructure\Proquifa.Finanzas.Infrastructure.csproj \
  --context ProquifaDotNetContext \
  --context-dir Persistence\Context \
  --output-dir Models\ProquifaDotNet \
  --table catFacturaEstado \
  --table fccFactura \
  --table fccFacturaPartida \
  --table fccFacturaReferenciaBancaria \
  --table EmpresaFolio \
  --table CFDIGenerada \
  --table CFDIGeneradaConcepto \
  --table CFDIGeneradaRelacionado \
  --table CFDICancelacion \
  --table catTipoCFDI \
  --table catTipoDocumentoFiscal \
  --table catDocumentoFiscalCobroEstado \
  --table fccDocumentoFiscalCobro \
  --table fccConfirmacionPedido \
  --table fccFolioPagoCliente \
  --table fccPagoCliente \
  --table catCobroEstatus \
  --table fccPagoFacturaPedido \
  --table fccPagoFacturaAdelanto \
  --table fccSaldoFavorCliente \
  --table catTipoInconsistenciaCobro \
  --table fccInconsistenciaCobro \
  --table fccNotaCredito \
  --table fccNotaCreditoPartida \
  --table fccFechaEstimadaPagoHistorial \
  --table catFormaPagoSAT \
  --table catTipoOperacionSUNAT \
  --table catMotivoCancelacionSAT \
  --table catMotivoCreditoSUNAT09 \
  --table catImpuesto \
  --table catTipoFactorSat \
  --table catObjetoImpuestoSat \
  --table PerfilFiscal \
  --table FamiliaRegion \
  --table Familia \
  --table tpProformaPedido \
  --table tpProformaPartidaPedido \
  --table Empresa \
  --table Cliente \
  --table DatosFacturacionCliente \
  --table Archivo \
  --table Region \
  --table RegionConfiguracionMinioBucket \
  --table catMoneda \
  --table catUsoCFDI \
  --table catMetodoDePagoCFDI \
  --table catCondicionesDePago \
  --table CorreoEnviado \
  --table ArchivoCorreoEnviado \
  --table vfccFactura \
  --table vfccDocumentoFiscalCobro \
  --table vUsuarioCartera \
  --no-onconfiguring \
  --data-annotations \
  --no-pluralize \
  --verbose \
  --force
```

**Notas del comando:**

- **Conexión:** cadena estándar del proyecto (`RYNL010` / `ProquifaDotNet`, Integrated Security). En ambientes DEV/QA sustituir el `Data Source` por el servidor correspondiente (ej. `WIN-R17-DEV\DEV_R17_APPS`) sin alterar el resto de la cadena.
- **Vistas** (`vfccFactura`, `vfccDocumentoFiscalCobro`, `vUsuarioCartera`): el scaffold las genera como entidades sin llave (keyless, `[Keyless]`/`HasNoKey()`) — solo lectura.
- `--no-onconfiguring`: la cadena de conexión se inyecta por DI desde `appsettings.json`, no se incrusta en el contexto.
- `--data-annotations` + `--no-pluralize`: nombres de entidad 1:1 con la tabla (regla 1 — las columnas de BD conservan su nomenclatura en español; solo el código nuevo se escribe en inglés, regla 6).
- `--force`: sobrescribe los modelos generados — **no editar manualmente** los archivos de `Models\ProquifaDotNet`; cualquier extensión va en clases parciales fuera de esa carpeta.
- `tpPedido` y `ClienteCartera` **no** van en el Scaffold: se consumen vía llamadas entre APIs (sección 3 del listado).
