# Endpoints — ProquifaDotNet.Finanzas (.NET Core 10)

Endpoints de la solución nueva `ProquifaDotNet.Finanzas`. Arquitectura Clean Architecture con CQRS. Consumida por el frontend de ProquifaDotNet y por llamadas inter-API desde ProquifaDotNet (.NET 4.8).

> **Base URL:** `https://{servidor-finanzas}/api/`
> **Autenticación:** IdentityServer (token JWT en header `Authorization: Bearer {token}`)
> **Nota:** Las tablas de datos (`fcc*`, `CFDIGenerada`, etc.) residen físicamente en la BD ProquifaDotNet y son accedidas vía EF Core Scaffold en `Finanzas.Infrastructure`.

---

## Índice

- [RE-016 — Proforma México](#re-016--proforma-méxico)
- [RE-018 — FAA Listado (Factura por Adelantado)](#re-018--faa-listado-factura-por-adelantado)
- [RE-019 — FAA Detalle y Timbrado México](#re-019--faa-detalle-y-timbrado-méxico)
- [RE-020 — FAA Perú (adaptación regional)](#re-020--faa-perú-adaptación-regional)
- [RE-021 — Persistencia PDF Factura México](#re-021--persistencia-pdf-factura-méxico)
- [RE-022 — Persistencia CPE Factura Perú](#re-022--persistencia-cpe-factura-perú)
- [RE-023 — Validar Cobro: Buzón, Cobros y Proformas](#re-023--validar-cobro-buzón-cobros-y-proformas)
- [RE-023 — Validar Cobro: Pantalla Principal (Clientes)](#re-023--validar-cobro-pantalla-principal-clientes)
- [RE-024 — Validar Cobro Paso 1 México (Catálogos + Wizard)](#re-024--validar-cobro-paso-1-méxico-catálogos--wizard)
- [RE-026 — Validar Cobro Paso 2 México (Asociación Cobro-Documento)](#re-026--validar-cobro-paso-2-méxico-asociación-cobro-documento)
- [RE-027 — Validar Cobro Paso 2 Perú](#re-027--validar-cobro-paso-2-perú)
- [RE-028 — Validar Cobro Paso 3 México (Facturación)](#re-028--validar-cobro-paso-3-méxico-facturación)
- [RE-032 — Notas de Crédito México](#re-032--notas-de-crédito-méxico)
- [RE-033 — Notas de Crédito Perú](#re-033--notas-de-crédito-perú)

---

## RE-016 — Proforma México

**Controller:** `ProformaController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| POST | `/api/proforma/generar-pdf` | Genera PDF de proforma sin persistir (previsualización) o con persistencia; invoca DocumentBuilder con template `{EMPRESA}_MEX_PRO` | Body: `GenerarProformaPdfRequest { DatosProforma, IdRegion, Persistir }` | `byte[]` PDF |
| GET | `/api/proforma/{id}/pdf` | Descarga PDF histórico de proforma desde MinIO sin regenerar | `id` en path (guid) | `application/pdf` |

---

## RE-018 — FAA Listado (Factura por Adelantado)

**Controller:** `FacturaAdelantadoController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| POST | `/api/factura-adelantado/listar` | Listado agrupado por cliente, paginado, filtrado por cartera del cobrador autenticado | Body: `QueryInfo { SortField, SortDirection, Filters, PageSize, DesiredPage }` | `QueryResult<FacturaAdelantadoClienteDto>` |

---

## RE-019 — FAA Detalle y Timbrado México

**Controller:** `FacturaAdelantadoController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| POST | `/api/factura-adelantado/detalle` | Listado de pedidos del cliente con estado FAA | Body: `{ IdCliente, RegionClave: 'MEX' }` | `List<PedidoFAADto>` |
| POST | `/api/factura-adelantado/generar` | Orquesta: arma datos → llama Timbrado (`POST /api/timbrado/timbrar-faa`) → persiste resultado (PDF/XML en MinIO) | Body: `GenerarFAARequest { IdTPProformaAdelanto, DatosFactura }` | `FacturaAdelantadaGeneradaDto { IdCFDIGenerada, UUID, RutaPDF }` |
| POST | `/api/factura-adelantado/previsualizar-pdf` | Genera PDF preview sin timbrar para modal de previsualización; invoca DocumentBuilder | Body: `PrevisualizarFAAPdfRequest` | `byte[]` PDF |
| POST | `/api/factura-adelantado/enviar` | Envía correo con PDF + XML al cliente, marca `tpProformaAdelanto.Enviada=1`, ejecuta salida operativa | Body: `{ IdFAA, DatosCorreo }` | `EstadoEnvioDto` |

---

## RE-020 — FAA Perú (adaptación regional)

Reutiliza los mismos endpoints que México. El branch interno de Perú se activa cuando `RegionClave = 'PER'` en el body del request.

| Método | Ruta                                        | Variante Perú                                                             | Req.   |
| ------ | ------------------------------------------- | ------------------------------------------------------------------------- | ------ |
| POST   | `/api/factura-adelantado/detalle`           | `RegionClave='PER'` en request filtra pedidos Perú                        | RE-020 |
| POST   | `/api/factura-adelantado/generar`           | Si `PER` → arma UBL 2.1 → llama `/api/timbrado/timbrar-sunat` en Timbrado | RE-020 |
| POST   | `/api/factura-adelantado/previsualizar-pdf` | Si `PER` → usa template PDF `GOLPERU_PER_FAC`                             | RE-020 |
| POST   | `/api/factura-adelantado/enviar`            | Mismo flujo de envío, sin diferencias regionales                          | RE-020 |

---

## RE-021 — Persistencia PDF Factura México

**Controller:** `CFDIController` (en ProquifaDotNet, consumido por Finanzas)

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| PUT | `/api/cfdi/{IdCFDI}/archivo-pdf` | Finanzas actualiza el `IdArchivoPdf` del CFDI tras subir el PDF a MinIO | `IdCFDI` en path; Body: `{ IdArchivoPdf }` | `204 NoContent` |

---

## RE-022 — Persistencia CPE Factura Perú

**Controller:** `tpProformaAdelantoController` (en ProquifaDotNet, consumido por Finanzas)

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| PUT | `/api/tpProformaAdelanto/{Id}/cpe-generada` | Finanzas actualiza el CPE generado en la proforma adelanto Perú tras timbrado SUNAT | `Id` en path; Body: `CPEGeneradaDto` | `204 NoContent` |

---

## RE-023 — Validar Cobro: Buzón, Cobros y Proformas

**Controller:** `ValidarCobroController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| GET | `/api/validar-cobro/buzon/{idCliente}/pendientes` | Lista pendientes activos del Buzón de Cobros para el cliente | `idCliente` en path | `List<FolioPagoClientePendienteDto>` |
| PUT | `/api/validar-cobro/buzon/{id}/cerrar` | Cierra pendiente del Buzón (`Activo=0`) | `id` en path | `204 NoContent` |
| GET | `/api/validar-cobro/cobros/{id}` | Obtiene cobro por Id (`fccPagoCliente`) | `id` en path | `PagoClienteDto` |
| GET | `/api/validar-cobro/cobros` | Lista cobros activos del cliente | `?idCliente={guid}` | `List<PagoClienteDto>` |
| PUT | `/api/validar-cobro/cobros` | INSERT borrador (`Confirmado=0`) / UPDATE borrador del cobro (Paso 1) | Body: `PagoClienteDto` | `PagoClienteDto` con Id asignado |
| GET | `/api/validar-cobro/proformas` | Lista pedidos pendientes del cliente para modal Gestionar Cobranza (`MontoPendiente > 0 AND Cancelada = 0`) | `?idCliente={guid}` | `List<ProformaPedidoDto>` + header `MontoTotalPendiente` |
| PUT | `/api/validar-cobro/proformas/fecha-promesa` | Actualiza fecha estimada de pago; genera historial en `fccFechaEstimadaPagoHistorial` | Body: `List<{ IdTpProformaPedido, Fecha, IdUsuarioCambio, Motivo? }>` | `204 NoContent` |

---

## RE-023 — Validar Cobro: Pantalla Principal (Clientes)

**Controller:** `ValidarCobroController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| POST | `/api/validar-cobro/clientes` | Listado paginado de clientes con acción contextual (`REALIZAR_COBROS` o `GESTIONAR_COBRANZA`); incluye saldo pendiente, cobros pendientes y SLA vencido calculados | Body: `QueryInfo { SortField, SortDirection, Filters, PageSize, DesiredPage }` | `QueryResult<ClienteValidarCobroDto>` |

---

## RE-024 — Validar Cobro Paso 1 México (Catálogos + Wizard)

**Controller:** `ValidarCobroController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| GET | `/api/validar-cobro/catalogos/monedas` | Catálogo de monedas activas | — | `List<catMonedaDto>` |
| GET | `/api/validar-cobro/catalogos/medios-pago` | Catálogo de medios de pago (`ClaveFormaDePago IS NOT NULL` para SAT) | `?region=MEX` | `List<catMedioDePagoDto>` |
| GET | `/api/validar-cobro/catalogos/cuentas-destino` | Cuentas bancarias destino para México (GOL/MUN/PRO/PQF) | `?region=MEX` | `List<CuentaDestinoDto>` |
| GET | `/api/validar-cobro/inconsistencias/tipos` | Catálogo de tipos de inconsistencia filtrado por paso | `?paso=1` | `List<catTipoInconsistenciaCobroDto>` |
| POST | `/api/validar-cobro/clientes/{idCliente}/cobros/{idFCCPagoCliente}/inconsistencias` | Registra inconsistencia del cobro en el Paso 1; INSERT en `fccInconsistenciaCobro` | `idCliente`, `idFCCPagoCliente` en path; Body: tipo + datos de inconsistencia | `fccInconsistenciaCobroDto` |
| GET | `/api/validar-cobro/clientes/{idCliente}/estado-wizard` | Retorna estado actual del wizard para el cliente (permite reanudar en el último paso activo) | `idCliente` en path | `EstadoWizardDto { PasoActivo, DatosReanudacion }` |

---

## RE-026 — Validar Cobro Paso 2 México (Asociación Cobro-Documento)

**Controller:** `ValidarCobroPaso2Controller`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| GET | `/api/validar-cobro/paso2/{idCliente}/documentos` | Lista proformas y facturas pendientes del cliente para el Paso 2 | `idCliente` en path | `List<DocumentoPendienteDto>` |
| GET | `/api/validar-cobro/paso2/{idCliente}/notas-credito` | Lista NCs vigentes del cliente disponibles para aplicar (`Aplicada=0, Activo=1`) | `idCliente` en path | `List<NotaCreditoDisponibleDto>` |
| POST | `/api/validar-cobro/paso2/{idCliente}/asociacion` | Persiste la asociación cobro ↔ documento(s) transaccionalmente: INSERT `fccPagoFacturaPedido`/`fccPagoFacturaAdelanto` + UPDATE NCs aplicadas + INSERT saldo a favor si hay sobrepago | `idCliente` en path; Body: `AsociacionCobroDocumentoRequest` | `AsociacionCobroDocumentoResultDto` |
| POST | `/api/validar-cobro/clientes/{idCliente}/cobros/{id}/inconsistencias` | Registra inconsistencia del Paso 2; soporta `PAGO_INCOMPLETO_VENCIDO` (marca pedido con pendiente de cancelación) | Path: `idCliente`, `id`; Body: `InconsistenciaRequest` | `fccInconsistenciaCobroDto` |

---

## RE-027 — Validar Cobro Paso 2 Perú

Mismos endpoints que RE-026 con variaciones regionales: `region=PER`, moneda base PEN, sin trigger de Complemento de Pago, un único emisor (GOLPERU).

| Endpoint | Diferencia vs. México |
|----------|-----------------------|
| `GET /api/validar-cobro/paso2/{idCliente}/documentos` | Filtra por `IdRegion=PER` |
| `GET /api/validar-cobro/paso2/{idCliente}/notas-credito` | Filtra por `IdRegion=PER` |
| `POST /api/validar-cobro/paso2/{idCliente}/asociacion` | Sin generación de Complemento de Pago |

---

## RE-028 — Validar Cobro Paso 3 México (Facturación)

**Controller:** `ValidarCobroPaso3Controller`

| Funcionalidad | Descripción | Parámetros entrada | Respuesta |
|---------------|-------------|-------------------|-----------|
| **Inicialización Paso 3** — POST `/api/validar-cobro/paso3/inicializar` | Crea registros en `fccDocumentoFiscalCobro` por cada documento de la asociación del Paso 2 al avanzar desde Paso 2 | Body: `{ IdFCCPagoCliente }` | `List<fccDocumentoFiscalCobroDto>` |
| **Auto-guardado** — PUT `/api/validar-cobro/paso3/linea/{id}/cfdi-config` | UPDATE `fccDocumentoFiscalCobro.IdCatUsoCFDI` y/o `IdCatMetodoDePagoCFDI` al modificar campos en pantalla | `id` en path; Body: `{ IdCatUsoCFDI?, IdCatMetodoDePagoCFDI? }` | `204 NoContent` |
| **Previsualización PDF** — POST `/api/validar-cobro/paso3/linea/{id}/preview-pdf` | Genera PDF en memoria sin persistir vía DocumentBuilder; template según tipo de línea (`*_MEX_FAC` o `*_MEX_COP`) | `id` en path | `byte[]` PDF |
| **Timbrar línea** — POST `/api/validar-cobro/paso3/linea/{id}/timbrar` | Llama Timbrado → persiste PDF en MinIO → INSERT/UPDATE registros de timbrado; cascada PPD: Factura + Complemento en secuencia | `id` en path; Body: `TimbrarLineaRequest` | `TimbrarLineaResultDto` |
| **Enviar línea** — POST `/api/validar-cobro/paso3/linea/{id}/enviar` | Modal de envío y despacho con adjuntos según tipo de línea; usa EnvioCorreo | `id` en path; Body: `EnviarLineaRequest { DatosCorreo }` | `EstadoEnvioDto` |
| **Confirmar Pedido** — POST `/api/validar-cobro/paso3/confirmacion-pedido` | INSERT `fccConfirmacionPedido`; genera PDF de Confirmación vía DocumentBuilder y lo sube a MinIO | Body: `{ IdTpProformaPedido }` | `ConfirmacionPedidoDto` |
| **Cerrar wizard** — PUT `/api/validar-cobro/paso3/cerrar` | Marca el wizard como completado; actualiza estado global del cobro | Body: `{ IdFCCPagoCliente }` | `204 NoContent` |

---

## RE-032 — Notas de Crédito México

**Controller:** `NotaCreditoMexicoController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| GET | `/api/nc-mexico` | Pantalla principal — NCs agrupadas por cliente (Total NC, Vigentes, Por Aplicar, Monto Total) filtrado por cartera del cobrador autenticado | `?idCliente={guid}&moneda={str}&fechaDesde={date}&fechaHasta={date}` (opcionales) | `List<NCClienteResumenDto>` |
| GET | `/api/nc-mexico/cliente/{idCliente}` | Drill-down — lista NCs del cliente con folio, fecha, monto, estado y comprobante origen | `idCliente` en path | `List<NotaCreditoDto>` |
| GET | `/api/nc-mexico/facturas-elegibles` | Lista facturas vigentes de prepago del cliente elegibles como origen de NC (≤ 5 años, no canceladas) | `?idCliente={guid}` | `List<FacturaElegibleDto>` |
| GET | `/api/nc-mexico/partidas-factura` | Lista partidas de la factura origen (para wizard modalidad por partidas) | `?idCFDIGenerada={guid}` | `List<PartidaFacturaDto>` |
| GET | `/api/nc-mexico/preview-pdf` | Genera PDF preview de NC en memoria sin timbrar | `?idFCCNotaCredito={guid}` | `byte[]` PDF |
| POST | `/api/nc-mexico/timbrar` | Timbra NC México: llama Timbrado (`POST /api/timbrado/nota-credito-mexico`), persiste PDF/XML en MinIO, cancela factura origen si `CancelarFacturaOrigen=true`, envía correo | `?idFCCNotaCredito={guid}` | `NotaCreditoTimbradaDto` |
| POST | `/api/nc-mexico/{idFCCNotaCredito}/reenviar-correo` | Reenvía correo de NC usando PDF/XML ya existentes en MinIO | `idFCCNotaCredito` en path | `EstadoEnvioDto` |

---

## RE-033 — Notas de Crédito Perú

**Controller:** `NotaCreditoPeruController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta |
|--------|------|-------------|-------------------|-----------|
| GET | `/api/nc-peru` | Pantalla principal — NCs Perú agrupadas por cliente filtrado por cartera | Mismos query params que México | `List<NCClienteResumenDto>` |
| GET | `/api/nc-peru/cliente/{idCliente}` | Drill-down — lista NCs Perú del cliente | `idCliente` en path | `List<NotaCreditoDto>` |
| GET | `/api/catalogos/motivos-nota-credito-sunat` | Catálogo de motivos de NC SUNAT (catálogo 09) | — | `List<catMotivoCreditoSUNAT09Dto>` |
| GET | `/api/nc-peru/facturas-elegibles` | Lista facturas Perú elegibles como origen de NC | `?idCliente={guid}` | `List<FacturaElegibleDto>` |
| GET | `/api/nc-peru/partidas-factura` | Lista partidas de la factura CPE origen | `?idCFDIGenerada={guid}` | `List<PartidaFacturaDto>` |
| GET | `/api/nc-peru/preview-pdf` | Genera PDF preview de NC Perú en memoria sin timbrar | `?idFCCNotaCredito={guid}` | `byte[]` PDF |
| POST | `/api/nc-peru/timbrar` | Timbra NC Perú: llama Timbrado (`POST /api/timbrado/nota-credito-peru`), persiste PDF/XML en MinIO Perú, envía correo | `?idFCCNotaCredito={guid}` | `NotaCreditoTimbradaDto` |
| POST | `/api/nc-peru/{idFCCNotaCredito}/reenviar-correo` | Reenvío de correo NC Perú con PDF/XML existentes | `idFCCNotaCredito` en path | `EstadoEnvioDto` |

---

## Resumen de Controllers

| Controller | Requisito(s) | Endpoints |
|---|---|---|
| `ProformaController` | RE-016 | 2 |
| `FacturaAdelantadoController` | RE-018, RE-019, RE-020 | 4 (reutilizados con branches regionales) |
| `ValidarCobroController` | RE-023, RE-024 | 9 |
| `ValidarCobroPaso2Controller` | RE-026, RE-027 | 4 |
| `ValidarCobroPaso3Controller` | RE-028 | 7 |
| `NotaCreditoMexicoController` | RE-032 | 7 |
| `NotaCreditoPeruController` | RE-033 | 8 |
| **Total** | | **~41** |
