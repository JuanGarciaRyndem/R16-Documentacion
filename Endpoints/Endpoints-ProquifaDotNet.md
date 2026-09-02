# Endpoints — ProquifaDotNet (.NET Framework 4.8)

Endpoints del aplicativo principal ProquifaDotNet (WebApi.Catalogos / WebApi.Logistica). Incluye controllers existentes y nuevos/modificados por el proyecto R16.

> **Base URL:** `https://{servidor}/Catalogos/` (WebApi.Catalogos) y `https://{servidor}/logistica/` (WebApi.Logistica)
> **Autenticación:** IdentityServer (token JWT en header `Authorization: Bearer {token}`)

---

## Índice

- [RE-001 — Datos Bancarios de Empresa](#re-001--datos-bancarios-de-empresa)
- [RE-002 — Cartera de Clientes / Reasignación de Cobrador](#re-002--cartera-de-clientes--reasignación-de-cobrador)
- [RE-003 — Documentos Regulatorios del Cliente](#re-003--documentos-regulatorios-del-cliente)
- [RE-004 — Régimen Fiscal y Tipo de Sociedad Mercantil](#re-004--régimen-fiscal-y-tipo-de-sociedad-mercantil)
- [RE-005 — Catálogos CFDI con Filtro de Región](#re-005--catálogos-cfdi-con-filtro-de-región)
- [RE-006 — Datos Bancarios del Cliente](#re-006--datos-bancarios-del-cliente)
- [RE-009 — Pretramitar Pedido](#re-009--pretramitar-pedido)
- [RE-010 — Tramitar Pedido (Crédito) — Cancelación](#re-010--tramitar-pedido-crédito--cancelación)
- [RE-016/018 — Delegación a Finanzas (ApiCallerFinanzas)](#re-016018--delegación-a-finanzas-apicallerfinanzas)
- [RE-023 — Cancelar Pedido por Falta de Pago](#re-023--cancelar-pedido-por-falta-de-pago)

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
| GET | `/vEmpresaDatosBancarios` | Detalle de cuenta bancaria empresa vigente filtrado por región del usuario autenticado | `?idEmpresaDatosBancarios={guid}` query param | `vEmpresaDatosBancariosDto` ó `204 NoContent` | **Nuevo** — sigue activo (Catálogos/Venta Interna); consumido directamente (sin endpoint propio) por el wizard de Validar Cobro en Finanzas para el combo Cuenta destino — ver `Endpoints-Finanzas.md` (RE-024) |
| POST | `/vEmpresaDatosBancarios` | Listado paginado de cuentas vigentes filtradas por región del usuario logueado | Body: `QueryInfo` | `QueryResult<vEmpresaDatosBancariosDto>` | **Nuevo** — sigue activo (Catálogos/Venta Interna); consumido directamente (sin endpoint propio) por el wizard de Validar Cobro en Finanzas para el combo Cuenta destino — ver `Endpoints-Finanzas.md` (RE-024) |

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
| POST | `/catMedioDePago` | Listado de medios de pago — **ahora con filtro de región** | Body: `QueryInfo` | `QueryResult<catMedioDePagoDto>` | **Modificado** — sigue activo (Catálogos/Venta Interna); consumido directamente (sin endpoint propio) por el wizard de Validar Cobro en Finanzas para el combo Medio de pago — ver `Endpoints-Finanzas.md` (RE-024) |

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

# RE-023 — Cancelar Pedido por Falta de Pago (interno — solo trazabilidad en `tpPedido`)

**Controller:** `tpPedidoController` (extensión)

> **Rol:** endpoint **interno** invocado únicamente por el **orquestador de Finanzas** (`PUT /api/v1/validate-collection/orders/{orderId}/cancel-non-payment`). Solo escribe trazabilidad de cancelación en `tpPedido` — NO cancela la proforma (`tpProformaPedido.IdcatEstadoProforma` la escribe Finanzas), NO cancela el CFDI (lo hace ProquifaDotNet.Timbrado). Es **idempotente**: si `tpPedido` ya está cancelado, no-op y responde `200 OK`. Ver diseño distribuido en `[R16A-RE-FU-023][DIS-SOL] Diseño de la solución.md` sección 1.4.

| Método | Ruta                                          | Descripción                                                                                                                                                                                                                             | Parámetros entrada                                                  | Respuesta                                                                        | Estado    |
| ------ | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- | -------------------------------------------------------------------------------- | --------- |
| PUT    | `/api/v1/orders/{orderId}/cancel-non-payment` | Escribe trazabilidad de cancelación en `tpPedido`: `FechaCancelacionPorFaltaPago = GETDATE()`, `IdUsuarioCancelacion = @IdUsuario`. **Idempotente** (si ya está cancelado, retorna OK sin cambios). NO toca `tpProformaPedido` ni CFDI. | `orderId` (=`IdTpPedido`) en path; Body: `{ IdUsuarioCancelacion }` | `200 OK` (cancelado o ya estaba cancelado — no-op idempotente) / `404 Not Found` | **Nuevo** |

---

## RE-024 — Catálogo de Monedas (también consumido por Finanzas — Validar Cobro Paso 1)

**Controller:** catálogo existente en Área Catálogos (mismo patrón `TablaGenericaBO<T>` que `catMedioDePago`, RE-005)

| Método | Ruta | Descripción | Parámetros entrada | Respuesta | Estado |
|--------|------|-------------|-------------------|-----------|--------|
| POST | `/Catalogos/catMoneda` | Listado de monedas activas | Body: `QueryInfo (Activo=1)` | `QueryResult<catMonedaDto>` | Existente — Catálogos/Venta Interna, no deprecado |

> Endpoint de Catálogos, sigue activo y en uso por Venta Interna. El wizard de Validar Cobro en Finanzas lo consume **directamente** para el combo Moneda del Paso 1 — sin endpoint propio en Finanzas — ver `Endpoints-Finanzas.md` (RE-024).

---

## RE-024 — Buzón de Correo: Detalle y Adjuntos (también consumido por Finanzas — Validar Cobro Paso 1)

**Controllers:** catálogos existentes en Área Catálogos, confirmados por captura de tráfico HTTP real (07/07/2026)

| Método | Ruta                        | Descripción                                     | Parámetros entrada | Respuesta | Estado |
| ------ | --------------------------- | ------------------------------------------------ | ------------------- | --------- | ------ |
| GET | `/Catalogos/CorreoRecibido` | Datos del correo: asunto, fecha, hora, contacto | `?idCorreoRecibido={guid}` | `CorreoRecibidoDto` | Existente — no deprecado |
| GET | `/Catalogos/CorreoRecibidoContenido` | Cuerpo/contenido del correo | `?idCorreoRecibidoContenido={guid}` | `CorreoRecibidoContenidoDto` | Existente — no deprecado |
| POST | `/Catalogos/ArchivoCorreoRecibido` | Lista de adjuntos del correo, candidatos a comprobante de pago | Body: `{ Filters: [{Activo:true},{IdCorreoRecibido:{guid}},{Mostrar:true}], GroupColumn }` | `QueryResult<ArchivoCorreoRecibidoDto>` | Existente — no deprecado |
| GET | `/Catalogos/Archivo` | Detalle/descarga de un adjunto específico | `?idArchivo={guid}` | `ArchivoDto` | Existente — no deprecado |

> **Confirmado por captura de tráfico HTTP real (07/07/2026):** el frontend de Validar Cobro (Finanzas) consume estos 4 endpoints **directamente** — Finanzas no crea un endpoint propio de "detalle de correo y adjuntos". Siguen activos y en uso por Venta Interna, sin deprecar — ver `Endpoints-Finanzas.md` (RE-024, sección B2).

---

## Resumen de Controllers afectados por R16

| Controller                                                                                          | Módulo             | Nuevos endpoints                                                      | Modificados                                        | Req.   |
| --------------------------------------------------------------------------------------------------- | ------------------ | --------------------------------------------------------------------- | -------------------------------------------------- | ------ |
| `vEmpresaDatosBancariosController`                                                                  | Catálogos/Empresas | 2                                                                     | —                                                  | RE-001 |
| `ClienteCarteraController`                                                                          | Catálogos/Clientes | 1 (`ReasignarCobrador`)                                               | —                                                  | RE-002 |
| `UsuarioController`                                                                                 | Catálogos/Usuarios | 1 (`GestoresDeCobranza`)                                              | —                                                  | RE-002 |
| `ArchivoClienteController`                                                                          | Catálogos/Clientes | 3                                                                     | —                                                  | RE-003 |
| `catRegimenFiscalController`                                                                        | Catálogos          | 1 (`PorRegion`)                                                       | —                                                  | RE-004 |
| `catTipoSociedadMercantilController`                                                                | Catálogos          | 1 (`PorRegion`)                                                       | —                                                  | RE-004 |
| `catMetodoDePagoCFDI` / `catUsoCFDI` / `catMedioDePago`                                             | Catálogos          | —                                                                     | 3 (filtro región)                                  | RE-005 |
| `ClienteDatosBancariosController`                                                                   | Catálogos/Clientes | 4                                                                     | —                                                  | RE-006 |
| `tpPedidoCancelacionController`                                                                     | Logística/Pedidos  | 1                                                                     | —                                                  | RE-010 |
| `tpPedidoController`                                                                                | Logística/Pedidos  | 1 (`cancel-non-payment` — interno, invocado por orquestador Finanzas) | —                                                  | RE-023 |
| `catMoneda` (Área Catálogos)                                                                        | Catálogos          | —                                                                     | Sin cambios — consumido directamente por Finanzas  | RE-024 |
| `CorreoRecibido` / `CorreoRecibidoContenido` / `ArchivoCorreoRecibido` / `Archivo` (Área Catálogos) | Catálogos/Buzón    | —                                                                     | Sin cambios — consumidos directamente por Finanzas | RE-024 |
