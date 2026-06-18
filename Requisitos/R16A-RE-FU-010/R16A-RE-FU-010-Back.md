# Impacto en Back — R16A-RE-FU-010
**Requisito:** Tramitacion de pedidos Credito
**Aplicativo:** ProquifaDotNet
**Modulo:** L05.TramitarPedido
**Impacto:** Flujo tramitacion preexistente + NUEVO endpoint de Cancelacion de pedido

---

## Resumen

Este requisito documenta el flujo **preexistente** de tramitacion de pedidos Credito (sin sustancias controladas, sin Factura por Adelantado). Adicionalmente, requiere un **nuevo endpoint de Cancelacion** en el modulo Tramitar Pedido que ejecute el flujo completo de cancelacion (distinto del Desactivar/inactivar existente).

---

## Seccion A — Flujo de Tramitacion Credito (Preexistente)

### Clase principal
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

### Metodo de entrada
`GenerarCorreoTramitarPedido(GMtpPedidoTramitarCorreo, out GMtpPedidoTramitarCorreoLiberado)`

### Secuencia del flujo Credito (SinCredito = false)

| Paso | Accion                                                            | Entidades afectadas                                                       |
| ---- | ----------------------------------------------------------------- | ------------------------------------------------------------------------- |
| 1    | Validaciones de configuracion (condiciones de pago, region, ESAC) | `catCondicionesDePago`, `Cliente`, `Region`, `ClienteCarteraCliente`      |
| 2    | Validaciones de pedido confirmacion                               | `tpPedido`, `tpPartidaPedido`                                             |
| 3    | Generacion de folio interno                                       | `VariableConfiguracion`, `RegionConsecutivoFoliosPedido` (Legacy SP)      |
| 4    | Generacion de PDF Pedido Confirmado                               | `Archivo` (MinIO)                                                         |
| 5    | Registro de contactos notificados entrega                         | `tpPedidoContactoNotificadoEntrega` (INSERT)                              |
| 6    | Actualizacion del pedido (Tramitado=1, Folio, FechaTramitacion)   | `tpPedido` (UPDATE)                                                       |
| 7    | Asignacion de stock y procesamiento de partidas                   | `tpPartidaPedido` (UPDATE)                                                |
| 8    | Creacion de pendientes de compra                                  | `ocPendienteCompraProducto` (INSERT)                                      |
| 9    | Addenda Sanofi (si aplica)                                        | `tpPartidaPedidoAddendaSanofi` (INSERT)                                   |
| 10   | Generacion de correo de confirmacion                              | `CorreoEnviado`, `ArchivoCorreoEnviado`, `tpPedidoCorreoEnviado` (INSERT) |
| 11   | Pendiente de stock (si hay partidas con stock)                    | `tpPendienteStock` (INSERT)                                               |
| 12   | Extracto Venta Digital                                            | `tpPedidoVD`, `tpPartidaPedidoVD` (INSERT/UPDATE)                         |
| 13   | Seguimiento de partidas                                           | `tpPartidaPedidoSeguimiento` (INSERT)                                     |
| 14   | Incremento de consecutivo Legacy (solo Mexico)                    | SP Legacy via `GeneradorFoliosPedido`                                     |
| 15   | Envio de correo via RabbitMQ/SendInBlue                           | Cola de mensajes                                                          |

---

## Modelo de Datos Involucrado (Entidades EF) — Seccion A

### Escritura (INSERT/UPDATE)

| Entidad | Tabla BD | Accion |
|---------|----------|--------|
| `tpPedido` | tpPedido | UPDATE (Tramitado=1, FolioPedidoInterno, FechaTramitacion, etc.) |
| `tpPartidaPedido` | tpPartidaPedido | UPDATE (stock, FEE, pendiente compra) |
| `ocPendienteCompraProducto` | ocPendienteCompraProducto | INSERT (pendiente de compra por partida) |
| `tpPedidoContactoNotificadoEntrega` | tpPedidoContactoNotificadoEntrega | INSERT |
| `CorreoEnviado` | CorreoEnviado | INSERT |
| `ArchivoCorreoEnviado` | ArchivoCorreoEnviado | INSERT |
| `tpPedidoCorreoEnviado` | tpPedidoCorreoEnviado | INSERT |
| `tpPendienteStock` | tpPendienteStock | INSERT (si hay partidas con stock) |
| `tpPartidaPedidoSeguimiento` | tpPartidaPedidoSeguimiento | INSERT |
| `tpPartidaPedidoAddendaSanofi` | tpPartidaPedidoAddendaSanofi | INSERT (si cliente Sanofi) |
| `VariableConfiguracion` | VariableConfiguracion | UPDATE (consecutivo Pedido) |
| `Archivo` | Archivo | INSERT (PDF confirmacion en MinIO) |

### Lectura (consulta)

| Entidad | Proposito |
|---------|-----------|
| `catCondicionesDePago` | Determinar si es Credito (SinCredito=0) |
| `DireccionCliente` / `DatosDireccionCliente` | Direccion entrega, AceptaParciales |
| `Cliente` | Region, configuracion |
| `Region` | Clave region (MEX/PER), impuesto |
| `ContactoCliente` / `CorreoElectronico` | Validacion de correo activo |
| `ppPedidoVD` / `ppPartidaPedidoVD` | Pedido pretramitado origen (Venta Digital) |
| `vProducto` | Datos producto, proveedor principal, controlado |
| `Proveedor` | Factor conversion, IdProveedor |
| `cotCotizacion` | Factor conversion USD, total moneda cliente |
| `tpPedidoFleteExpress` | Flete express por proveedor |
| `vDireccion` | Direccion facturacion y entrega |
| `catSeguimientoPartidaPedido` | Catalogo seguimiento (Orden=1) |

---

## Transferencia a Legacy

La transferencia se realiza post-commit mediante:
- **SP:** `spActualizarBuzonPedidoLegacy` / `spActualizarBuzonPedidoLegacyEncolar`
- **Datos transferidos:** Pedido (folio, cliente, montos), Partidas (producto, piezas, precios), Cobro
- **Marca de detencion (Pago contra entrega):** Se identifica por la clave de condicion de pago. Legacy usa esta marca para detener el pedido en fase de entrega hasta validar pago.

### Identificacion Credito vs Pago contra entrega

```
catCondicionesDePago.SinCredito = 0 -> Credito normal
catCondicionesDePago.Clave = 'pagocontraentrega' -> Variante con marca de detencion en Legacy
```

---

## Controlador Tramitacion

`WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\Liberar\tpPedidoTramitarController.cs`

---

## DTOs / Modelos de Transaccion

| Modelo | Ubicacion | Proposito |
|--------|-----------|-----------|
| `GMtpPedidoTramitarCorreo` | L05.TramitarPedido\Models\ | Input: pedido, partidas, contactos, comentarios |
| `GMtpPedidoTramitarCorreoLiberado` | L05.TramitarPedido\Models\ | Output: pedido liberado, partidas, correo, archivo |
| `GMtpPartidasPedido` | L05.TramitarPedido\Models\ | Partida + pendiente compra + addenda |
| `GMtpPedidoLiberado` | L05.TramitarPedido\Models\ | Pedido liberado |

---

## Seccion B — Cancelacion del Pedido desde Tramitar Pedido (DESARROLLO NUEVO)

### Situacion actual

El controlador `tpPedidoController.cs` solo cuenta con un endpoint `[HttpDelete] Route("tpPedido")` que ejecuta `tpPedidoBO.Desactivar()` — esto unicamente **inactiva** el registro (Activo=false), pero **no ejecuta el flujo completo de cancelacion** (validaciones, marcado de partidas, bitacora).

La logica de cancelacion completa ya existe en:
`Logic.Pqf.Logistica\L04.PretramitarPedido\GestionarIntramitables\ppPedidoCancelacionBO.CancelacionOrdenDeCompra()`

### Desarrollo requerido

Se necesita un **nuevo endpoint de Cancelacion** en el modulo L05.TramitarPedido que:

1. Reciba el `IdPPPedido` (o `IdTPPedido`) del pedido a cancelar
2. Ejecute el flujo completo de cancelacion (no solo inactivar)
3. Sea distinto del `Desactivar` existente

### Propuesta de implementacion

**Nuevo Controller (o extension del existente):**
```
WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\tpPedidoCancelacionController.cs
```

**Endpoint propuesto:**
```
[HttpPut]
[Route("tpPedido/cancelar")]
public Response CancelarPedido(Guid idppPedido)
```

**Flujo del endpoint:**

| Paso | Accion | Entidades afectadas |
|------|--------|---------------------|
| 1 | Validar que el pedido existe | `ppPedido` (lectura) |
| 2 | Validar que no este ya cancelado | `ppPedido.Cancelada` |
| 3 | Validar que no tenga dependencia en tpPedido (parcialidad) | `tpPedidoBO.ConDependenciaTpPedido()` |
| 4 | Validar catalogo de accion de bitacora | `catBitacoraAccion` (lectura) |
| 5 | Marcar pedido como cancelado | `ppPedido` (UPDATE: Cancelada=true) |
| 6 | Marcar cada partida como cancelada | `ppPartidaPedido` (UPDATE: Cancelada=true) |
| 7 | Inactivar el tpPedido asociado (si existe) | `tpPedido` (UPDATE: Activo=false) |
| 8 | Registrar en bitacora — partidas | `BitacoraCRUD` (INSERT por cada partida) |
| 9 | Registrar en bitacora — pedido | `BitacoraCRUD` (INSERT) |
| 10 | Eliminar pendiente de Tramitar Pedido | Cierre del pendiente en bandeja |
| 11 | Commit | Transaccion confirmada |

### Entidades afectadas (escritura)

| Entidad | Tabla BD | Accion |
|---------|----------|--------|
| `ppPedido` | ppPedido | UPDATE (Cancelada=true) |
| `ppPartidaPedido` | ppPartidaPedido | UPDATE (Cancelada=true) |
| `tpPedido` | tpPedido | UPDATE (Activo=false, o campo de cancelacion) |
| `BitacoraCRUD` | BitacoraCRUD | INSERT (1 por pedido + 1 por partida) |

### Entidades de consulta (lectura)

| Entidad | Proposito |
|---------|-----------|
| `catBitacoraAccion` | Obtener accion "Cancelacion" |
| `ContactoCliente` | Obtener IdCliente del pedido |
| `Cliente` | Nombre del cliente para detalle bitacora |
| `Producto` | Catalogo del producto para detalle bitacora |

### Validaciones de negocio
- El pedido debe existir en BD
- No debe estar previamente cancelado (Cancelada = false)
- No debe tener dependencia en tpPedido (no aplica si ya se tramito parcialidad)
- Requiere confirmacion explicita del ESAC (validada desde Front antes de invocar el endpoint)

### Referencia de logica existente
La logica base esta en `ppPedidoCancelacionBO.CancelacionOrdenDeCompra()`. Se puede:
- **Opcion A:** Reutilizar directamente esa clase desde el nuevo controller
- **Opcion B:** Crear un nuevo BO `tpPedidoCancelacionBO` que extienda la logica para incluir la inactivacion del tpPedido

---

## Impacto en Desarrollo

| Concepto | Detalle |
|----------|---------|
| Codigo nuevo requerido | **Endpoint de Cancelacion** (controller + posible BO nuevo) |
| Cambios en BD | **Ninguno** — tablas y campos ya existen |
| Cambios en API | **Nuevo endpoint** `[HttpPut] tpPedido/cancelar` |
| Cambios en ETL/Legacy | **Verificar** marca de detencion para Pago contra entrega |
| Testing | Validar flujo Credito + Pago contra entrega + Cancelacion + Inactivacion |

---

## Dependencias con otros requisitos

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-003 | Pretramitacion del pedido (genera `ppPedido` que alimenta este flujo) |
| R16A-RE-FU-006 | Referencia de pago bancaria en `tpProformaPedido` |
| R16A-RE-FU-009 | Validacion regulatoria (productos controlados — excluidos de este flujo) |

---

## Conclusion

El requisito R16A-RE-FU-010 tiene dos componentes:

**Seccion A (Preexistente):** El flujo de tramitacion Credito ya esta implementado en `tpPedidoTramitarTransaccionBO.GenerarCorreoTramitarPedido()`. Solo requiere verificacion de la marca de detencion para Pago contra entrega.

**Seccion B (Desarrollo Nuevo):** Se requiere un nuevo endpoint de **Cancelacion** que ejecute el flujo completo (validaciones + cancelar ppPedido + cancelar partidas + inactivar tpPedido + bitacora), distinto del `Desactivar` existente que solo marca Activo=false. La logica base existe en `ppPedidoCancelacionBO` y puede reutilizarse o extenderse.
