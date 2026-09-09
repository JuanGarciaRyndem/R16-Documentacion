# Impacto en Back — R16A-RE-FU-010
**Requisito:** Tramitacion de pedidos Credito
**Aplicativo:** ProquifaDotNet
**Modulo:** L05.TramitarPedido
**Impacto:** Flujo tramitacion preexistente + endpoint de Cancelacion de pedido (CONSTRUIDO, pendiente PR/evidencia)

---

## Resumen

Este requisito documenta el flujo **preexistente** de tramitacion de pedidos Credito (sin sustancias controladas, sin Factura por Adelantado), mas dos ajustes puntuales sobre ese flujo (Reglas 2/3 y 5), y el **endpoint de Cancelacion** en el modulo Tramitar Pedido que ejecuta el flujo completo de cancelacion (distinto del Desactivar/inactivar existente).

**Estado real al 2026-09-08** (ver seccion "Estado de implementacion" abajo): el codigo de las cuatro tareas ya esta escrito. T1/T2 (ajustes al flujo Credito existente) viven en una rama publicada sin Pull Request abierto. T3 (endpoint de Cancelacion) viaja dentro del Pull Request #269 de ProquifaDotNet, compartido con R16A-RE-FU-015 por una dependencia de catalogo. T4 (cierre de pendiente en bandeja) quedo verificado por construccion, sin desarrollo adicional.

---

## Seccion A — Flujo de Tramitacion Credito (Preexistente + ajustes T1/T2)

### Clase principal
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

### Metodo de entrada
`GenerarCorreoTramitarPedido(GMtpPedidoTramitarCorreo, out GMtpPedidoTramitarCorreoLiberado)`

### Ajuste T1/T2 sobre la decision de flujo (Reglas 2 y 3)

`Pago contra entrega` tiene `catCondicionesDePago.SinCredito = 1` en base de datos, pero por regla de negocio debe recorrer la rama de credito. El BO ya no evalua `SinCredito` directamente: delega en el nuevo helper puro `Logic.Pqf.Logistica\L05.TramitarPedido\Helpers\CondicionPagoFlujoHelper.cs`.

| Metodo | Devuelve `true` cuando |
|---|---|
| `EsPagoContraEntrega(condicionDePago)` | `Clave` (trim, case-insensitive) = `pagocontraentrega` |
| `EsCondicionCredito(condicionDePago)` | `!SinCredito` **o** `EsPagoContraEntrega(...)` |

La bandera `catCondicionesDePago.SinCredito` **no se modifica en BD** (decision D1 del helper): la traduccion vive solo en la capa de decision de flujo, para no impactar la transferencia a Legacy, reportes ni otros modulos que leen esa bandera directamente. El BO reemplaza cada verificacion de `SinCredito` por `!esCredito` / `esCredito` en los puntos que deciden folio, generacion de PDF y el incremento de consecutivo Legacy (solo Mexico).

### Ajuste T1/T2 sobre el ETL a Legacy (Regla 5 / CA-1 — operacion Peru)

El ETL del buzon hacia Legacy (`tpPedidoBO.ActualizarBuzonPedidoTramitadoLegacy`) ahora se ejecuta condicionado a `region.ProcesarEnETL`. Para clientes de Region Peru ese flag es falso: el ETL se omite y se deja rastro en el log (`Se omite el ETL de buzon hacia Legacy: la region {Clave} no transfiere a Legacy.`); la operacion termina con la confirmacion interna en PQF2, sin transferir el pedido a Legacy.

### Secuencia del flujo Credito (SinCredito = false / esCredito = true)

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
| 14   | Incremento de consecutivo Legacy (solo Mexico, si esCredito)      | SP Legacy via `GeneradorFoliosPedido`                                     |
| 15   | ETL de buzon a Legacy (solo si `region.ProcesarEnETL`)             | `spActualizarBuzonPedidoLegacy(Encolar)` — se omite para Peru             |
| 16   | Envio de correo via RabbitMQ/SendInBlue                           | Cola de mensajes                                                          |

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
| `catCondicionesDePago` | Determinar si es Credito (via `CondicionPagoFlujoHelper.EsCondicionCredito`) |
| `Region` | Clave region (MEX/PER), impuesto, `ProcesarEnETL` (T1/T2) |
| `DireccionCliente` / `DatosDireccionCliente` | Direccion entrega, AceptaParciales |
| `Cliente` | Region, configuracion |
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
- **Condicion regional (T1/T2, Regla 5):** el ETL solo se ejecuta si `region.ProcesarEnETL = true`. Para Peru se omite; el pedido no se transfiere a Legacy.

### Identificacion Credito vs Pago contra entrega

```
CondicionPagoFlujoHelper.EsCondicionCredito(catCDP) == true  -> rama de flujo Credito (incluye Pago contra entrega)
CondicionPagoFlujoHelper.EsPagoContraEntrega(catCDP) == true -> variante con marca de detencion en Legacy
catCondicionesDePago.SinCredito NO se modifica en BD en ningun caso
```

**Pendiente (no cambio en este ajuste):** el SP `spActualizarBuzonPedidoLegacy` en si no fue tocado por T1/T2 — solo se agrego el condicional regional alrededor de la llamada. Sigue sin evidencia levantada de que el SP incluya la marca de detencion para Pago contra entrega (ver T2 en Tareas).

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

## Seccion B — Cancelacion del Pedido desde Tramitar Pedido (CONSTRUIDO — T3, ticket R16A-1478)

### Estado

Ya **no** es una propuesta: el endpoint esta construido, probado por HTTP real (con autorizacion desactivada) y con pruebas automatizadas en verde, dentro del Pull Request [#269](https://github.com/ryndem/ProquifaDotNet/pull/269) de ProquifaDotNet — el mismo PR de R16A-RE-FU-015 (T7/T8, ticket R16A-1503), por la dependencia con el catalogo `catMotivoCancelacion` que esa tarea entrega. El seguimiento operativo de esta pieza vive en **R16A-1478** (Parte 2 de este requisito), no en R16A-1477.

Ya no aplica la comparacion contra `[HttpDelete] tpPedido` (`tpPedidoBO.Desactivar()`, solo inactiva) como el unico endpoint existente: el nuevo endpoint es un `PUT` distinto, con su propio flujo transaccional completo.

### Endpoint construido

```
PUT /api/tpPedido/Cancelar
Controller: WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\Cancelacion\tpPedidoCancelacionController.cs
Body (GMtpPedidoCancelacion):
{
  "idPPPedido": "guid",
  "motivoCancelacionClave": "solicitudcliente"   // clave de dbo.catMotivoCancelacion
}
```

El controller no valida nada por si mismo: delega todo — existencia, estado, dependencias, marcado, bitacora, transaccion — en `ppPedidoCancelacionBO.CancelacionOrdenDeCompra(idPPPedido, motivoCancelacionClave, inactivarPedidoTramitado: true)`, un **overload nuevo** de la clase que ya usaba Gestionar Intramitables. El overload original de un solo parametro sigue intacto y sin cambio de comportamiento (`motivoCancelacionClave: null`, `inactivarPedidoTramitado: false`), asi que Gestionar Intramitables no se ve afectado.

### Flujo real (dentro de `ppPedidoCancelacionBO`, una sola transaccion)

| Paso | Accion | Entidades / notas |
|------|--------|---------------------|
| 1 | Validar `idppPedido != Guid.Empty` **antes** de abrir el contexto de datos | corregido en este cambio (ver "Defectos") |
| 2 | Validar que el pedido existe | `ppPedido` (lectura) — si no existe, rechazo de validacion (ya no `NullReferenceException`, ver "Defectos") |
| 3 | Validar que no este ya cancelado (`ppPedido.Cancelada`) | mapea a HTTP 409 |
| 4 | Validar que no tenga dependencia en tpPedido (parcialidad) | `tpPedidoBO.ConDependenciaTpPedido()` — mapea a HTTP 409 |
| 5 | Validar que exista `catBitacoraAccion` con `Accion = "Cancelacion"` | lectura |
| 6 | Validar la clave de motivo contra `catMotivoCancelacion` (si viene informada) | `Clave` + `Activo = true`; clave inexistente = rechazo de validacion (HTTP 400) |
| 7 | Marcar `ppPedido.Cancelada = true` | UPDATE + SaveChanges |
| 8 | Marcar cada `ppPartidaPedido.Cancelada = true` + 1 `BitacoraCRUD` por partida | UPDATE + INSERT (detalle: catalogo del producto) |
| 9 | Inactivar el/los `tpPedido` asociados (`Activo = false`) — **solo si `inactivarPedidoTramitado = true`** | esto es lo que retira el pedido de la bandeja: `vTramitarPedido` filtra `tpPedido.Activo = 1` |
| 10 | Registrar 1 `BitacoraCRUD` para el pedido, con el motivo en el detalle si vino informado | `Detalle` incluye `cliente.Nombre` y `motivoCancelacion.Descripcion` |
| 11 | Commit | transaccion `IsolationLevel.ReadUncommitted` |

No existe un paso separado para "cerrar el pendiente de bandeja": el paso 9 (inactivar `tpPedido`) **es** el cierre del pendiente, dentro de la misma transaccion (ver T4 en Tareas).

### Mapeo de codigos HTTP (en el controller)

| Codigo | Cuando |
|---|---|
| 200 | Cancelacion exitosa |
| 409 (Conflict) | Pedido ya cancelado, o con dependencia de parcialidad |
| 400 (Bad Request) | Falta identificador, pedido no existe, o clave de motivo no existe en catalogo |
| 500 | Error interno |

### Entidades afectadas (escritura)

| Entidad | Tabla BD | Accion |
|---------|----------|--------|
| `ppPedido` | ppPedido | UPDATE (Cancelada=true) |
| `ppPartidaPedido` | ppPartidaPedido | UPDATE (Cancelada=true) |
| `tpPedido` | tpPedido | UPDATE (Activo=false) — solo si hay tpPedido asociado |
| `BitacoraCRUD` | BitacoraCRUD | INSERT (1 por pedido + 1 por partida) |

### Entidades de consulta (lectura)

| Entidad | Proposito |
|---------|-----------|
| `catBitacoraAccion` | Obtener accion "Cancelacion" |
| `catMotivoCancelacion` | Validar la clave de motivo recibida (catalogo de FU-015 T7, no de este requisito) |
| `ContactoCliente` | Obtener IdCliente del pedido |
| `Cliente` | Nombre del cliente para detalle bitacora |
| `Producto` | Catalogo del producto para detalle bitacora de cada partida |

### Validaciones de negocio confirmadas

- El pedido debe existir en BD.
- No debe estar previamente cancelado (`Cancelada = false`).
- No debe tener dependencia en tpPedido (parcialidad).
- La clave de motivo, si viene, debe existir y estar activa en `catMotivoCancelacion`.
- Requiere confirmacion explicita del ESAC — responsabilidad del Front, el endpoint no la valida.

### Defectos encontrados y su estado

| # | Defecto | Estado | Afecta tambien a |
|---|---|---|---|
| 1 | Pedido inexistente producia `NullReferenceException` en lugar de rechazo de validacion | **Corregido** | Gestionar Intramitables (codigo compartido) |
| 2 | El identificador vacio no se validaba antes de abrir el contexto de datos | **Corregido** | Gestionar Intramitables (codigo compartido) |
| 3 | En el `catch`, se invoca `Rollback()` sobre una transaccion que el bloque `using` ya desecho; ese `Rollback` lanza su propia excepcion y **enmascara la original** (p.ej. una violacion de FK llega al consumidor como `The underlying provider failed on Rollback`, la causa real solo queda en el log) | **NO corregido** — pendiente de decision aparte, por ser codigo compartido de L04 | Gestionar Intramitables (codigo compartido) |

### Lo que falta para cerrar T3

- **Probar la cancelacion de extremo a extremo con un token valido.** El flujo escribe en `BitacoraCRUD`, que exige `IdUsuario` (FK a `dbo.Usuario`) tomado del usuario autenticado; sin token las pruebas locales solo cubrieron los rechazos de validacion.
- Decidir si se corrige el defecto #3 (Rollback enmascarado) antes o despues de este release, dado que es codigo compartido.

### Punto abierto — relacion con `pedidoEstadoActual` (R16A-RE-FU-015)

`R16A-RE-FU-015` (T7/T8, ticket R16A-1503) crea la tabla `pedidoEstadoActual` y el endpoint generico `PUT /api/orders/status`, que **si** contempla un estado `cancelado` con motivo obligatorio (por acuerdo con Juan David, segun el comentario de T8). Este endpoint de Cancelacion (`PUT /api/tpPedido/Cancelar`, T3 de este requisito) **no** invoca ni actualiza `pedidoEstadoActual` — cancela via los campos existentes (`ppPedido.Cancelada`, `ppPartidaPedido.Cancelada`, `tpPedido.Activo`), que es un mecanismo distinto y anterior al de FU-015.

**Falta definir:** si al cancelar un pedido desde Tramitar Pedido tambien debe quedar reflejado el estado `cancelado` en `pedidoEstadoActual` (para que el estatus consolidado del pedido sea consistente), o si ambos mecanismos coexisten a proposito con alcances distintos. Sin esta definicion, un pedido cancelado por este endpoint puede seguir mostrando en `pedidoEstadoActual` el ultimo estado no terminal que tenia (p.ej. `entramite`), si algo mas adelante empieza a leer esa tabla como fuente de estatus.

---

## Impacto en Desarrollo

| Concepto | Detalle |
|----------|---------|
| Codigo T1/T2 (ajustes flujo Credito) | **Construido**, en rama `feature/r16-phase-02-R16A-RE-FU-010` (commit `c59d51f12`), publicada en origin, **sin Pull Request abierto** |
| Codigo T3 (endpoint Cancelacion) | **Construido y probado** (HTTP real sin token + 64 pruebas automatizadas), dentro del PR #269 (compartido con FU-015) |
| Codigo T4 (cierre de pendiente) | **Verificado por construccion** — no requirio desarrollo adicional (ver Seccion B) |
| Cambios en BD | **Si, pero no de este requisito**: `catMotivoCancelacion` y `pedidoEstadoActual` los crea FU-015 (T7). Este requisito solo *consume* `catMotivoCancelacion` |
| Cambios en API | Nuevo endpoint `PUT /api/tpPedido/Cancelar` (ya construido, no es el `PUT tpPedido/cancelar` en minuscula que se habia propuesto) |
| Cambios en ETL/Legacy | Condicional regional agregado (Regla 5); SP en si sin cambios — marca de detencion para Pago contra entrega sigue sin evidencia levantada |
| Testing | T1/T2: evidencia pendiente de levantar. T3: probado por HTTP sin token + automatizado; falta prueba end-to-end con token valido |

---

## Estado de implementacion (T1-T4) — actualizado 2026-09-08

Fuente: comentarios de Cristobal Sebastian Garcia Coss en R16A-1477 (T1/T2) y R16A-1503 (contexto T7/T8 de FU-015), revisados contra el codigo real en el repositorio `ProquifaDotNet`.

| Tarea | Ticket | Rama / PR | Estado |
|---|---|---|---|
| T1 — Verificacion flujo Credito | R16A-1477 | `feature/r16-phase-02-R16A-RE-FU-010` (commit `c59d51f12`), sin PR | Codigo listo, falta abrir PR y levantar evidencia |
| T2 — Verificacion Pago contra entrega + marca Legacy | R16A-1477 | idem | Codigo listo (helper + condicional regional), falta evidencia de la marca de detencion en el SP |
| T3 — Endpoint de Cancelacion | **R16A-1478** (no R16A-1477) | PR [#269](https://github.com/ryndem/ProquifaDotNet/pull/269), compartido con R16A-RE-FU-015 | Construido y probado; falta prueba end-to-end con token y decidir el defecto #3 |
| T4 — Cierre de pendiente en bandeja | R16A-1477 (Tarea 4 de este documento) | Sin desarrollo — es consecuencia del paso 9 de T3 | Verificado por construccion |

**Nota de convencion:** la rama de T1/T2 lleva el prefijo de fase (`r16-phase-02-`), mientras la convencion local del repositorio es `feature/<ID>` sin prefijo (asi se nombro la de R16A-RE-FU-015). No se recomienda renombrarla ya publicada, pero aplica para ramas futuras.

---

## Dependencias con otros requisitos

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-003 | Pretramitacion del pedido (genera `ppPedido` que alimenta este flujo) |
| R16A-RE-FU-006 | Referencia de pago bancaria en `tpProformaPedido` |
| R16A-RE-FU-009 | Validacion regulatoria (productos controlados — excluidos de este flujo) |
| R16A-RE-FU-015 | Entrega el catalogo `catMotivoCancelacion` (consumido por T3) y la tabla `pedidoEstadoActual` + endpoint `PUT /api/orders/status` (relacion con la cancelacion, punto abierto — ver Seccion B) |

---

## Conclusion

El requisito R16A-RE-FU-010 tiene dos componentes:

**Seccion A (Preexistente + ajustes T1/T2):** El flujo de tramitacion Credito ya estaba implementado en `tpPedidoTramitarTransaccionBO.GenerarCorreoTramitarPedido()`; T1/T2 aislaron la decision de flujo por condicion de pago en `CondicionPagoFlujoHelper` y condicionaron el ETL a Legacy por region. Codigo listo, publicado sin PR, falta evidencia.

**Seccion B (Cancelacion, T3):** El endpoint `PUT /api/tpPedido/Cancelar` esta construido y probado, reutilizando `ppPedidoCancelacionBO` mediante un overload que agrega motivo (via `catMotivoCancelacion`, catalogo de FU-015) e inactivacion de `tpPedido`. Corrigio dos defectos previos compartidos con Gestionar Intramitables y dejo uno sin corregir (Rollback enmascarado). Queda abierta la relacion entre esta cancelacion y `pedidoEstadoActual` de FU-015.

---

## Cambios

| # | Fecha | Observacion | Descripcion del cambio |
|---|---|---|---|
| 1 | 2026-09-08 | Revision R16A-1477 / R16A-1503 + codigo ProquifaDotNet | Se documenta el estado real de construccion: T1/T2 (helper `CondicionPagoFlujoHelper`, ETL condicional por region) en rama sin PR; T3 (endpoint `PUT /api/tpPedido/Cancelar`, ticket real R16A-1478) construido dentro del PR #269 compartido con FU-015; T4 verificado por construccion. Se agregan los tres defectos encontrados (2 corregidos, 1 pendiente) y el punto abierto sobre la relacion con `pedidoEstadoActual` de FU-015. |
