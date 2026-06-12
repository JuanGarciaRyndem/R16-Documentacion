# Endpoints — ProquifaDotNet (.NET Framework 4.8)

Endpoints del aplicativo principal ProquifaDotNet (WebApi.Catalogos / WebApi.Logistica). Incluye controllers existentes y nuevos/modificados por el proyecto R16.

> **Base URL:** `https://{servidor}/api/` (WebApi.Catalogos) y `https://{servidor}/logistica/` (WebApi.Logistica)
> **Autenticación:** IdentityServer (token JWT en header `Authorization: Bearer {token}`)

---

## Índice

- [RE-001 — Datos Bancarios de Empresa](#re-001--datos-bancarios-de-empresa)
- [RE-002 — Cartera de Clientes / Reasignación de Cobrador](#re-002--cartera-de-clientes--reasignación-de-cobrador)
- [RE-003 — Documentos Regulatorios del Cliente](#re-003--documentos-regulatorios-del-cliente)
- [RE-004 — Régimen Fiscal y Tipo de Sociedad Mercantil](#re-004--régimen-fiscal-y-tipo-de-sociedad-mercantil)
- [RE-005 — Catálogos CFDI con Filtro de Región](#re-005--catálogos-cfdi-con-filtro-de-región)
- [RE-006 — Datos Bancarios del Cliente](#re-006--datos-bancarios-del-cliente)
- [RE-008 — Buzón de Cobros (Mailbot)](#re-008--buzón-de-cobros-mailbot)
- [RE-009 — Pretramitar Pedido](#re-009--pretramitar-pedido)
- [RE-010 — Tramitar Pedido (Crédito) — Cancelación](#re-010--tramitar-pedido-crédito--cancelación)
- [RE-016/018 — Delegación a Finanzas (ApiCallerFinanzas)](#re-016018--delegación-a-finanzas-apicallerfinanzas)
- [RE-023 — Cancelar Pedido por Falta de Pago](#re-023--cancelar-pedido-por-falta-de-pago)
- [NO-FU-002 — Bitácora Transaccional](#no-fu-002--bitácora-transaccional)

---

## RE-001 — Datos Bancarios de Empresa

**Controller:** `Controllers\Configuracion\Empresas\EmpresaDatosBancariosController.cs`
**Controller (nuevo):** `Controllers\Configuracion\Empresas\vEmpresaDatosBancariosController.cs`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| GET | `/EmpresaDatosBancarios` | Obtener cuenta bancaria de empresa por Id | `?id={guid}` query param | `EmpresaDatosBancariosDto` | Existente |
| PUT | `/EmpresaDatosBancarios` | Guardar o actualizar cuenta bancaria de empresa | Body: `EmpresaDatosBancariosDto` | `EmpresaDatosBancariosDto` actualizado | Existente |
| POST | `/EmpresaDatosBancarios` | Listado paginado de cuentas bancarias de empresa | Body: `QueryInfo` | `QueryResult<EmpresaDatosBancariosDto>` | Existente |
| DELETE | `/EmpresaDatosBancarios` | Desactivar cuenta bancaria de empresa | `?id={guid}` query param | `bool` | Existente |
| POST | `/EmpresaDatosBancariosDetalle` | Listado paginado de detalles de cuentas bancarias | Body: `QueryInfo` | `QueryResult<EmpresaDatosBancariosDetalleDto>` | Existente |
| POST | `/EmpresaDatosBancariosDetalle/GrupoLista` | Listado agrupado de detalles de cuentas bancarias | Body: `QueryInfo` | `List<GrupoEmpresaDatosBancarios>` | Existente |
| GET | `/vEmpresaDatosBancarios` | Detalle de cuenta bancaria empresa vigente filtrado por región del usuario autenticado | `?idEmpresaDatosBancarios={guid}` query param | `vEmpresaDatosBancariosDto` ó `204 NoContent` | **Nuevo** |
| POST | `/vEmpresaDatosBancarios` | Listado paginado de cuentas vigentes filtradas por región del usuario logueado | Body: `QueryInfo` | `QueryResult<vEmpresaDatosBancariosDto>` | **Nuevo** |

**Regla:** Los endpoints PUT/DELETE de `EmpresaDatosBancariosController` existen para Soporte a la Producción; ninguna pantalla de R16 los consume directamente.

---

## RE-002 — Cartera de Clientes / Reasignación de Cobrador

**Controller:** `Controllers\Configuracion\Clientes\Carteras\ClienteCarteraController.cs`
**Controller:** `Controllers\Configuracion\Usuarios\UsuarioController.cs`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| GET | `/ClienteCartera` | Obtener registro de cartera por Id | `?id={guid}` | `ClienteCarteraDto` | Existente |
| PUT | `/ClienteCartera` | Guardar o actualizar cartera | Body: `ClienteCarteraDto` | `ClienteCarteraDto` | Existente |
| POST | `/ClienteCartera` | Listado paginado de carteras | Body: `QueryInfo` | `QueryResult<ClienteCarteraDto>` | Existente |
| DELETE | `/ClienteCartera` | Desactivar cartera | `?id={guid}` | `bool` | Existente |
| PUT | `/ClienteCartera/ReasignarCobrador` | Reasigna cobrador de una cartera; valida que el solicitante sea Coordinador/Gerente de Tesorería y que el nuevo cobrador sea Gestor activo | Body: `{ IdClienteCartera, IdUsuarioCobradorNuevo }` | `ClienteCarteraDto` actualizado | **Nuevo** |
| GET | `/Usuario/GMUsuarioClienteCarteraDetalle` | Cartera completa del usuario (incluye `ClientesCarteraCobrador`) | `?idUsuario={guid}` | `GMUsuarioClienteCarteraDetalleDto` | Existente |
| POST | `/Usuario/ClienteCarteraCliente` | Asigna lista de clientes a una cartera | Body: `List<ClienteCarteraClienteDto>` | `bool` | Existente |
| POST | `/Usuario/ListaUsuariosCartera` | Lista usuarios con relaciones de cartera | Body: `QueryInfo` | `QueryResult<UsuarioCarteraDto>` | Existente |
| POST | `/Usuario` | Lista paginada de usuarios | Body: `QueryInfo` | `QueryResult<UsuarioDto>` | Existente |
| GET | `/Usuario/GestoresDeCobranza` | Lista usuarios con `AnalistaDeCuentasPorCobrar = true AND Activo = true` filtrado por región | `?idRegion={guid}` query param opcional | `List<UsuarioDto>` | **Nuevo** |

---

## RE-003 — Documentos Regulatorios del Cliente

**Controller:** `Controllers\Configuracion\Clientes\Relaciones\ArchivoClienteController.cs`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| POST | `/ArchivoCliente/DocumentoRegulatorio` | Carga PDF regulatorio del cliente; valida formato PDF, sube a MinIO y reemplaza registro activo anterior de forma atómica | Body: `multipart/form-data { idCliente, catUsoArchivoSistema, archivo }` | `ArchivoClienteDto` | **Nuevo** |
| GET | `/ArchivoCliente/DocumentosRegulatoriосs` | Retorna estado de documentos regulatorios vigentes por cliente (LicenciaSanitaria / AvisoResponsableSanitario, `Activo=1`) | `?idCliente={guid}` | `List<ArchivoClienteEstadoDto>` | **Nuevo** |
| GET | `/ArchivoCliente/DocumentoRegulatorio/{id}/pdf` | Descarga PDF de MinIO del documento regulatorio indicado | `id` en path (guid) | `application/pdf` con `Content-Disposition: inline` | **Nuevo** |

---

## RE-004 — Régimen Fiscal y Tipo de Sociedad Mercantil

**Controllers:** `catRegimenFiscalController`, `catTipoSociedadMercantilController`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| GET | `/catRegimenFiscal` | CRUD genérico — obtener régimen fiscal | `?id={guid}` | `catRegimenFiscalDto` | Existente |
| GET | `/catRegimenFiscal/PorRegion` | Retorna regímenes fiscales filtrados por región del cliente | `?idRegion={guid}` | `List<catRegimenFiscalDto>` | **Nuevo** |
| GET | `/catTipoSociedadMercantil/PorRegion` | Retorna tipos de sociedad mercantil filtrados por región del cliente | `?idRegion={guid}` | `List<catTipoSociedadMercantilDto>` | **Nuevo** |

---

## RE-005 — Catálogos CFDI con Filtro de Región

**Controllers existentes modificados** para heredar de `BaseApiController` e invocar `AsegurarFiltroRegion()` en cada query.

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| POST | `/catMetodoDePagoCFDI` | Listado de métodos de pago CFDI — **ahora con filtro de región** del usuario autenticado | Body: `QueryInfo` | `QueryResult<catMetodoDePagoCFDIDto>` | **Modificado** |
| POST | `/catUsoCFDI` | Listado de usos CFDI — **ahora con filtro de región** | Body: `QueryInfo` | `QueryResult<catUsoCFDIDto>` | **Modificado** |
| POST | `/catMedioDePago` | Listado de medios de pago — **ahora con filtro de región** | Body: `QueryInfo` | `QueryResult<catMedioDePagoDto>` | **Modificado** |

---

## RE-006 — Datos Bancarios del Cliente

**Controller nuevo:** `Controllers\Configuracion\Clientes\Relaciones\ClienteDatosBancariosController.cs`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| GET | `/ClienteDatosBancarios` | Obtener relación cliente — cuenta bancaria por Id | `?id={guid}` | `ClienteDatosBancariosDto` | **Nuevo** |
| PUT | `/ClienteDatosBancarios` | Guardar o actualizar relación cliente — cuenta bancaria; persiste `CodigoValidador` | Body: `ClienteDatosBancariosDto` | `ClienteDatosBancariosDto` actualizado | **Nuevo** |
| POST | `/ClienteDatosBancarios` | Listado paginado de relaciones cliente — cuenta bancaria | Body: `QueryInfo` | `QueryResult<ClienteDatosBancariosDto>` | **Nuevo** |
| DELETE | `/ClienteDatosBancarios` | Desactivar relación cliente — cuenta bancaria | `?id={guid}` | `bool` | **Nuevo** |
| GET | `/vClienteDatosBancarios` | Vista de consulta de datos bancarios del cliente (incluye datos del banco) | `?idCliente={guid}` | `List<vClienteDatosBancariosDto>` | Existente |
| GET | `/DatosBancarios` | CRUD de cuentas bancarias del grupo | `?id={guid}` | `DatosBancariosDto` | Existente |

---

## RE-008 — Buzón de Cobros (Mailbot)

**Controllers nuevos** en `WebApi.Logistica\Controllers\Procesos\Mailbot\`

| Método | Ruta                                        | Descripción                                                                                            | Parámetros entrada                                         | Respuesta                                | Estado         |
| ------ | ------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------- | ---------------------------------------- | -------------- |
| POST   | `/BuzonCobros`                              | Listado paginado del Buzón de Cobros filtrado por `IdUsuarioCobrador` y región del usuario autenticado | Body: `QueryInfo`                                          | `QueryResult<BuzonCobrosItemDto>`        | **Nuevo**      |
| POST   | `/BandejaCoordenadorTesoreria`              | Listado de la bandeja del Coordinador de Tesorería; acceso restringido por rol                         | Body: `QueryInfo`                                          | `QueryResult<BandejaCoordenadorItemDto>` | **Nuevo**      |
| GET    | `/api/buzon/cobros`                         | Lista paginada del Buzón de Cobros con filtros; patrón REST                                            | Body: `QueryInfo` (filtros, paginación)                    | `QueryResult<BuzonCobrosItemDto>`        | **Nuevo** (P2) |
| PUT    | `/api/buzon/cobros/{id}/reclasificar`       | Reclasifica correo del Buzón de Cobros a otro buzón                                                    | `id` en path; Body: `{ IdCatClasificacionCorreoRecibido }` | `204 NoContent`                          | **Nuevo** (P2) |
| PUT    | `/api/cobros/folio/{id}/cerrar`             | Cierra pendiente del Buzón al vincular cobro a proforma/factura                                        | `id` en path                                               | `204 NoContent`                          | **Nuevo** (P2) |
| GET    | `/api/buzon/cotizaciones`                   | Lista paginada del Buzón de Cotizaciones con filtros                                                   | Body: `QueryInfo`                                          | `QueryResult<BuzonCotizacionesItemDto>`  | **Nuevo** (P2) |
| PUT    | `/api/buzon/cotizaciones/{id}/reclasificar` | Reclasifica correo del Buzón de Cotizaciones                                                           | `id` en path; Body: `{ IdCatClasificacionCorreoRecibido }` | `204 NoContent`                          | **Nuevo** (P2) |
| GET    | `/api/buzon/pedidos`                        | Lista paginada del Buzón de Pedidos con filtros                                                        | Body: `QueryInfo`                                          | `QueryResult<BuzonPedidosItemDto>`       | **Nuevo** (P2) |
| PUT    | `/api/buzon/pedidos/{id}/reclasificar`      | Reclasifica correo del Buzón de Pedidos                                                                | `id` en path; Body: `{ IdCatClasificacionCorreoRecibido }` | `204 NoContent`                          | **Nuevo** (P2) |

---

## RE-009 — Pretramitar Pedido

**Controller:** `WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\`

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| POST | `/PretramitarPedido/transaccion` | Pretramita pedido; valida internamente vía `VerificarPedidoTramitableBO`; sin cambios en R16 | Body: `PretramitarPedidoRequest` | `PretramitarPedidoResponse` | Existente |
| POST | `/ValidarAjusteOC/transaccion` | Tramita pedido con OC corregida; mismo BO de validación | Body: `ValidarAjusteOCRequest` | `ValidarAjusteOCResponse` | Existente |

---

## RE-010 — Tramitar Pedido (Crédito) — Cancelación

**Controller nuevo:** `WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\tpPedidoCancelacionController.cs`

| Método | Ruta                 | Descripción                                                                                                                           | Parámetros entrada                        | Respuesta                      | Estado    |
| ------ | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- | ------------------------------ | --------- |
| PUT    | `/tpPedido/cancelar` | Cancela completamente el pedido: validaciones + cancelar `ppPedido` + cancelar partidas + inactivar `tpPedido` + registro en bitácora | Body: `{ IdTPPedido, MotivoCancelacion }` | `tpPedidoCancelacionResultDto` | **Nuevo** |

---

## RE-016/018 — Delegación a Finanzas (ApiCallerFinanzas)

ProquifaDotNet actúa como proxy/delegador invocando la API de `ProquifaDotNet.Finanzas` a través del cliente HTTP `ApiCallerFinanzas`. El frontend consume estos endpoints de ProquifaDotNet, que internamente delegan la lógica a Finanzas.

| Método | Ruta (ProquifaDotNet)                    | Delegado a (Finanzas)                 | Descripción                                                            | Req.   |
| ------ | ---------------------------------------- | ------------------------------------- | ---------------------------------------------------------------------- | ------ |
| POST   | `/api/proforma/generar-pdf`              | `POST /api/proforma/generar-pdf`      | Genera PDF de proforma; ProquifaDotNet reenvía la solicitud a Finanzas | RE-016 |
| GET    | `/api/proforma/{id}/pdf`                 | `GET /api/proforma/{id}/pdf`          | Descarga PDF histórico de proforma desde Finanzas/MinIO                | RE-016 |
| POST   | `/api/factura-adelantado/listar` (proxy) | `POST /api/factura-adelantado/listar` | Expone listado FAA al frontend delegando a Finanzas                    | RE-018 |

---

## RE-023 — Cancelar Pedido por Falta de Pago

**Controller:** `tpPedidoController` (extensión)

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| PUT | `/api/pedidos/{idTpPedido}/cancelar-falta-pago` | Cancela pedido por falta de pago: UPDATE `tpProformaPedido.Cancelada=1`, registra `tpPedido.FechaCancelacionPorFaltaPago`, solicita cancelación CFDI si hay factura vigente | `idTpPedido` en path | `204 NoContent` ó error con detalle | **Nuevo** |

---

## NO-FU-002 — Bitácora Transaccional

**Controller nuevo:** `BitacoraTransaccionController` (extensión de `BitacoraCRUDController`)

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| GET | `/BitacoraTransaccion/{idBitacoraTransaccion}` | Retorna encabezado de transacción más lista de `BitacoraCRUD` vinculados | `idBitacoraTransaccion` en path (guid) | `BitacoraTransaccionDetalleDto` | **Nuevo** |
| GET | `/BitacoraTransaccion` | Historial de transacciones por registro principal; filtra por tabla origen e Id del registro | `?tablaOrigen={string}&idRegistroOrigen={guid}` | `List<BitacoraTransaccionDto>` | **Nuevo** |

---

## Resumen de Controllers afectados por R16

| Controller | Módulo | Nuevos endpoints | Modificados | Req. |
|---|---|---|---|---|
| `vEmpresaDatosBancariosController` | Catálogos/Empresas | 2 | — | RE-001 |
| `ClienteCarteraController` | Catálogos/Clientes | 1 (`ReasignarCobrador`) | — | RE-002 |
| `UsuarioController` | Catálogos/Usuarios | 1 (`GestoresDeCobranza`) | — | RE-002 |
| `ArchivoClienteController` | Catálogos/Clientes | 3 | — | RE-003 |
| `catRegimenFiscalController` | Catálogos | 1 (`PorRegion`) | — | RE-004 |
| `catTipoSociedadMercantilController` | Catálogos | 1 (`PorRegion`) | — | RE-004 |
| `catMetodoDePagoCFDI` / `catUsoCFDI` / `catMedioDePago` | Catálogos | — | 3 (filtro región) | RE-005 |
| `ClienteDatosBancariosController` | Catálogos/Clientes | 4 | — | RE-006 |
| `BuzonCobrosController` | Logística/Mailbot | 5 | — | RE-008 |
| `BandejaCoordenadorTesoreriaController` | Logística/Mailbot | 1 | — | RE-008 |
| `tpPedidoCancelacionController` | Logística/Pedidos | 1 | — | RE-010 |
| `tpPedidoController` | Logística/Pedidos | 1 (`cancelar-falta-pago`) | — | RE-023 |
| `BitacoraTransaccionController` | Transversal | 2 | — | NO-FU-002 |
