# Impacto en Back — R16A-RE-FU-015
**Requisito:** Tramitación de pedidos Prepago (variante con Factura por Adelantado)
**Aplicativo:** ProquifaDotNet, ProquifaDotNet.Finanzas
**Módulo:** L05.TramitarPedido (orquestador) → ProquifaDotNet.Finanzas (nuevo servicio)
**Impacto:** Flujo preexistente — variante Prepago con FAA activada, genera pendiente directamente en el módulo Factura por Adelantado (NO en Validar Cobro, y sin generar proforma)

---

## ✅ OBS-027 RESUELTO — Estatus del pedido: catálogo `catEstadoPedido` + tabla `PedidoEstadoActual`

> **Resuelto (11/08/2026, actualizado 12/08/2026)** — ver `Analisis/Estados de Pedidos/catEstadoPedido — Estados Propuestos.md`.
>
> El cliente propuso el catálogo de 17 estados sobre `dbo.catEstadoPedido` (catálogo ya existente en BD, hoy vacío y sin FK — se extiende con `Orden`, `EsTerminal`, `Aplicativo`, `AliasOperativo`) y el catálogo nuevo `dbo.catMotivoCancelacion`. **Ya NO se crea el catálogo `CatEstadoTpPedido` ni se agrega `IdCatEstadoTpPedido` directo en `tpPedido`**, como se documentaba antes de esta actualización.
>
> En su lugar, el estatus del pedido (Criterio D5) se centraliza en la tabla nueva **`PedidoEstadoActual`**: un registro por pedido que referencia, según la etapa del flujo en la que se encuentre, a `pcPromesaDeCompra` (ocrecibida), `ppPedido` (Pretramitar/Intramitable/En Trámite) y/o `tpPedido` (Prepago con FAA/Confirmado/Logística/Entregado), más el estado vigente (`IdCatEstadoPedido`) y, si aplica, el motivo de cancelación (`IdCatMotivoCancelacion`). Esta tabla resuelve también la Observación #4 del catálogo propuesto (¿el campo va en `tpPedido`, en `ppPedido` o en ambas?): en vez de duplicar la FK en varias tablas, se centraliza en una sola.
>
> **T7 (BD) y T8 (Back) quedan DESBLOQUEADAS.** Ver diccionario de datos completo en `R16A-RE-FU-015_BD.md`.
>
> **Nota de alcance de este ajuste:** por indicación del cliente, este endpoint/tabla **no gestiona la cancelación** (no asigna `IdCatMotivoCancelacion`) — la columna existe en `PedidoEstadoActual` para cuando ese requisito se desarrolle, pero su lógica de escritura es un requisito aparte, fuera de este alcance.

---

## Resumen

> **Actualización de diseño (adopción del DIS-SOL v1.0):** este documento adopta la arquitectura definida en `[R16A-RE-FU-015][DIS-SOL] Diseño de la solución.pdf` (v1.0, 29/06/2026, revisado por Juan David García Cruz el 02/07/2026). La decisión de diseño **no reutiliza `tpProformaAdelanto`**; en su lugar, el pendiente FAA se modela con tres tablas nuevas propiedad de `ProquifaDotNet.Finanzas`: `fccFactura` (cabecera), `fccFacturaPartida` (detalle) y `fccFacturaReferenciaBancaria` (referencias bancarias). Queda pendiente el hallazgo H-01 de `R16A-RE-FU-015_DIS-SOL_Revision.md` (ver Riesgos abajo).

Este requisito es la variante Prepago sin controlados donde el ESAC **activa Factura por Adelantado**. A diferencia de RE-FU-013/014, este flujo **no genera proforma, PDF ni correo**: al tramitar, `ProquifaDotNet` actúa como orquestador (genera y commitea `FolioPedidoInterno`) y delega a `ProquifaDotNet.Finanzas`, que inserta el pendiente FAA y cierra el pendiente operativo de Tramitar Pedido.

- **Al tramitar NO genera pendiente en Validar Cobro** (no hay proforma que lo dispare)
- **Genera pendiente en módulo Factura por Adelantado** directamente al tramitar (INSERT en `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`, en `ProquifaDotNet.Finanzas`)
- El pendiente Validar Cobro se generará después, cuando FAA emita la factura PPD (RE-FU-018/019/020), reutilizando `fccFactura` con `EsFacturaPorAdelantado = 0`

**Otras diferencias:**
- `tpPedido.FacturaPorAdelantado = 1`
- Activación directa sin código de autorización (Regla 3 / RT-07)
- Datos de facturación bloqueados y fijados al activar FAA (snapshot en `fccFactura`)
- El cierre del pendiente operativo en Tramitar Pedido no implica que el pedido quede "tramitado en su totalidad" (Regla 13 / Criterio D3) — se realiza actualizando `PedidoEstadoActual.IdCatEstadoPedido` a `prepagoconfaa` (RT-05, OBS-027 resuelto — ver arriba)

---

## Arquitectura (adoptada del DIS-SOL)

| Componente | Responsabilidad | Ubicación |
|---|---|---|
| `ProquifaDotNet` — `L05.TramitarPedido` | Endpoint orquestador FAA: valida (`SinCredito=1`, `TieneControlados()=false`, `FAA=1`, `Remisión=0`, datos de facturación sin cambios), fija datos de facturación del catálogo del cliente, genera `FolioPedidoInterno` y lo asigna a `tpPedido` (commit), llama al endpoint interno de Finanzas | `WebApi.Logistica` / `Logic.Pqf.Logistica` |
| `ProquifaDotNet.Finanzas` | Lee `FolioPedidoInterno` desde `tpPedido`, INSERT atómico del pendiente FAA (`fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`), cierre del pendiente Tramitar Pedido | Nueva solución |
| BD (SQL Server) | Persistencia del pendiente FAA y estado del pedido | Base de datos existente (`ProquifaDotNet`) |

> `tpPedidoTramitarTransaccionBO.cs` **no se utiliza** en este flujo — el orquestador es un endpoint nuevo dedicado a la variante FAA.

### Endpoints nuevos/modificados

| Solución | Endpoint | Tipo | Parámetros | Salida |
|---|---|---|---|---|
| `ProquifaDotNet` | `POST /v1/api/invoices/advance-invoice/{orderId}` | Nuevo | `orderId` (Guid, ruta); body vacío | Confirmación: `orderId`, `internalOrderFolio`, `invoiceId`, `advanceInvoiceStatus`, `processOrderTaskClosed` |
| `ProquifaDotNet.Finanzas` (interno) | `POST /v1/api/invoices/advance-invoice/create/{orderId}` | Nuevo | `orderId` (Guid, ruta) | Confirmación de creación del pendiente |
| `ProquifaDotNet` | `GET /Logistica/vTramitarPedidoDetalle` | Reutilizado (RE-013 T8) | `idTPPedido` (Guid, query) | Modelo `vTramitarPedidoDetalle` (incluye `TieneProductosControlados` y `CorreoElectronicoEsac`) |
| `ProquifaDotNet` | `PUT /v1/api/orders/status` | Nuevo (OBS-027, T8) | Body: `idPcPromesaDeCompra`, `idPPPedido`, `idTPPedido` (Guid, opcionales — exactamente uno informado), `idCatEstadoPedido` (Guid, requerido) | Confirmación: `idPedidoEstadoActual`, `estadoAnterior`, `estadoNuevo`. No administra cancelación (no recibe `idCatMotivoCancelacion`) |

---

## Código Existente Relacionado

### Tramitación principal
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs` — **no se utiliza en este flujo** (el orquestador FAA es un endpoint nuevo, ver Arquitectura).

### Pendiente FAA — reemplazo de `tpProformaAdelanto`
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Anticipos\tpProformaAdelantoBO.cs` — **ya no aplica a este requisito.**

> Por decisión de diseño (DIS-SOL v1.0), el pendiente FAA de RE-015 **no se modela con `tpProformaAdelanto`**. Se reemplaza por tres tablas nuevas en `ProquifaDotNet.Finanzas`: `fccFactura` (cabecera, tabla única compartida con la factura final vía `EsFacturaPorAdelantado`), `fccFacturaPartida` (detalle de partidas) y `fccFacturaReferenciaBancaria` (referencias bancarias). `tpProformaAdelantoBO.cs` deja de ser código de referencia para este requisito; se conserva la mención únicamente para trazabilidad histórica.

---

## Análisis — Flujo compartido con RE-FU-014

| Componente | Se desarrolla en | Aplica a RE-FU-015 |
|------------|-------------------|----------------------|
| Verificación Perú | RE-FU-013 T6 | Sí — misma verificación |
| Validación Remisión Prepago | RE-FU-014 T1 | Sí — misma validación |
| Datos facturación solo lectura | RE-FU-014 T2 | Sí — misma validación |

> **Nota:** El foliador lineal global de proforma (RE-013 T1), la previsualización del PDF (RE-013 T3), el envío de correo (RE-013 T4) y la vinculación con DocumentBuilder (RE-013 T7) **ya no aplican a RE-015** — el requisito actualizado no genera proforma, PDF ni correo en este flujo.

---

## Riesgos

**Riesgo 1 — H-01 (`R16A-RE-FU-015_DIS-SOL_Revision.md`): `fccFactura` no modela campos fiscales de Perú**
El bloque de datos del receptor de `fccFactura` (snapshot de `DatosFacturacionCliente`) solo contempla catálogos SAT de México (`RegimenFiscalClaveSAT`, `UsoCFDIClaveSAT`, `MetodoDePagoClaveSAT`, `FormaDePagoClaveSAT`). La Regla 14 / Criterio A5 del requisito exige que para clientes Perú se persistan Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago SUNAT en su lugar, y el propio Alcance del DIS-SOL declara operación en México **y Perú**. Mientras no se resuelva este hallazgo (agregar columnas equivalentes a `fccFactura` o documentar explícitamente por qué no aplican), un pedido peruano con FAA activada no tendría dónde fijar sus datos fiscales correctos. **No bloquea la adopción de la tabla `fccFactura` en este documento, pero debe resolverse antes de cerrar el desarrollo de T2 (INSERT del pendiente FAA) para Perú.**

**Riesgo 2 — Campos de información fiscal originalmente configurados para México**
Los campos de información fiscal del módulo Tramitar Pedido están actualmente configurados conforme a las normas fiscales de México. Al operar pedidos peruanos, el ESAC podría experimentar confusión sobre qué campos aplican o cómo interpretarlos en el contexto fiscal peruano. Se espera capacitación al equipo operativo para clarificar el manejo de los campos fiscales en pedidos de la región Perú.

---

## Criterios de Aceptación

### Sección A — Tramitación, activación y opciones en pantalla

**Criterio A1 — Tramitación habilitada para Prepago sin controlados con Factura por Adelantado activada**
- **Dado** que un pedido pertenece a un cliente Prepago en México o Perú, sin productos controlados, y el ESAC activa la opción Factura por Adelantado,
- **Cuando** el ESAC opera el módulo Tramitar Pedido,
- **Entonces** el sistema deberá permitir la tramitación y, al ejecutarse, generar el pendiente FAA (`fccFactura` + detalle) — no se genera proforma (ver Notas: cláusula heredada de proforma pendiente de limpieza editorial en el requisito, ver `R16A-RE-FU-015_DIS-SOL_Revision.md` H-03).

**Criterio A2 — Activación de Factura por Adelantado desde Tramitar Pedido**
- **Dado** que un pedido pertenece a un cliente Prepago sin productos controlados,
- **Cuando** el ESAC visualiza el módulo Tramitar Pedido,
- **Entonces** el sistema deberá ofrecer la opción de activar Factura por Adelantado. La activación es directa y no requiere código de autorización (RT-07).

**Criterio A3 — Bloqueo de edición de datos de facturación al activar Factura por Adelantado**
- **Dado** que el ESAC activó la opción Factura por Adelantado en Tramitar Pedido,
- **Cuando** se muestra la pantalla del pedido,
- **Entonces** el botón "Editar Datos" para datos de facturación no debe aparecer disponible para este pedido. El sistema deberá mostrar los datos de facturación en modo solo lectura tomados del catálogo del cliente vigente al momento de la activación (fijados como snapshot en `fccFactura`).

**Criterio A4 — No visualización de Entrega con Remisión para Prepago**
- **Dado** que el pedido es de cliente Prepago,
- **Cuando** el ESAC visualiza la pantalla del pedido,
- **Entonces** el radio button de Entrega con Remisión no deberá aparecer en la pantalla, dado que esta opción no aplica para clientes prepago en ninguna variante.

**Criterio A5 — Composición regionalizada del panel de Información de Facturación**
- **Dado** que el ESAC visualiza el panel de Información de Facturación de un pedido en Tramitar Pedido,
- **Cuando** el sistema muestra el panel según la Región del cliente,
- **Entonces** para clientes México deberá mostrar Uso CFDI y Método de Pago (catálogos SAT); para clientes Perú deberá mostrar Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago Contado/Crédito SUNAT en su lugar; en ambas regiones deberá mostrar los campos comunes (Razón Social, RFC/RUC, Moneda, Quién Factura, Condiciones de Pago comerciales y Comentarios) y NO deberá mostrar Forma de Pago ni correo de envío.
- ⚠️ **No cubierto por el modelo de datos actual de `fccFactura`** — ver Riesgo 1 / H-01.

### Sección D — Pendientes generados y cierre

**Criterio D1 — Generación del pendiente Factura por Adelantado al tramitar**
- **Dado** un pedido prepago sin controlados con Factura por Adelantado activada,
- **Cuando** el ESAC ejecuta la acción Tramitar,
- **Entonces** el sistema deberá generar automáticamente un pendiente en el módulo Factura por Adelantado asociado al folio del pedido (INSERT atómico en `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`), para que Finanzas gestione posteriormente la emisión y timbrado de la factura.

**Criterio D2 — Momento de generación del pendiente Validar Cobro**
- **Dado** que el ESAC tramitó un pedido prepago con Factura por Adelantado activada,
- **Cuando** se completa la tramitación,
- **Entonces** el pendiente en Validar Cobro se generará posteriormente, cuando la factura se emita exitosamente desde el módulo Factura por Adelantado (RT-06).

**Criterio D3 — Desaparición del pendiente operativo en bandeja Tramitar Pedido**
- **Dado** que la acción de Tramitar Pedido se completó (con la generación del pendiente en Factura por Adelantado),
- **Cuando** se actualiza `PedidoEstadoActual.IdCatEstadoPedido` a `prepagoconfaa` vía `PUT /v1/api/orders/status` (RT-05, OBS-027 resuelto),
- **Entonces** el pedido no deberá seguir apareciendo como pendiente en la bandeja del módulo Tramitar Pedido del ESAC.

**Criterio D4 — Cancelación del pedido**
- **Dado** que un pedido tramitado tiene solicitud del cliente para cancelar,
- **Cuando** el ESAC ejecuta la acción Cancelar pedido en Tramitar Pedido,
- **Entonces** el sistema deberá presentar un modal de confirmación y requerir confirmación explícita antes de proceder (endpoint compartido con RE-FU-010).

**Criterio D5 — Estatus del pedido a lo largo del flujo**
- **Dado** que el sistema opera por pendientes (que aparecen y desaparecen de cada bandeja a medida que se trabajan) y que esos pendientes no reflejan por sí solos el estatus global del pedido,
- **Cuando** un pedido avanza por las distintas etapas del flujo,
- **Entonces** el sistema deberá mantener un estatus del pedido que refleje su punto en el flujo, persistido en `PedidoEstadoActual.IdCatEstadoPedido` contra el catálogo `catEstadoPedido` (OBS-027 resuelto — ver `catEstadoPedido — Estados Propuestos.md`).

---

## Gaps de Desarrollo Específicos de RE-FU-015

| # | Gap | Acción | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Generación pendiente FAA al tramitar con FAA=1 | Al confirmar la acción de tramitar (sin generación previa de proforma), INSERT atómico en `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria` (en `ProquifaDotNet.Finanzas`) con datos del pedido/cliente/empresa/monto/partidas/referencias bancarias | Medio |
| GAP-02 | ~~NO generar pendiente Validar Cobro cuando FAA=1~~ | **Ya no aplica.** Como este flujo no genera `tpProformaPedido`, no existe `MontoPendiente` que pudiera disparar un pendiente en Validar Cobro (RT-06) — no hay nada que suprimir | — |
| GAP-03 | Eliminar código de autorización para FAA | Buscar y eliminar validación de código de autorización para activar Factura por Adelantado (Regla 3 / RT-07: activación directa) | Bajo |
| GAP-04 | Bloquear datos facturación al activar FAA | Fijar datos de facturación del catálogo del cliente vigente al momento de activar FAA como snapshot en `fccFactura` (RFC, Razón Social, CP, Régimen Fiscal, Uso CFDI, Método de Pago, Forma de Pago) | Bajo |
| GAP-05 | Vinculación con módulo FAA (RE-FU-018/019/020) | Tarea para asegurar que el pendiente generado en `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria` sea consumido correctamente por el módulo FAA (RT-10: `fccFactura` es tabla única para FAA y factura final, diferenciadas por `EsFacturaPorAdelantado`) | Bajo |
| GAP-06 | Cancelación del pedido | Dependencia de R16A-RE-FU-010 (endpoint de cancelación) | Referencia |
| GAP-07 | Ausencia de documento/PDF disponible para TaskScheduler de Venta Digital | Confirmar si el job de TaskScheduler que transfiere PDFs a Legacy puede operar cuando no existe ningún PDF generado en Tramitar Pedido para este flujo (ver `R16A-RE-FU-015-Tareas.md`) | Medio |
| GAP-08 | H-01 — Campos fiscales de Perú en `fccFactura` | Antes de cerrar desarrollo, agregar a `fccFactura` (o tabla de extensión regional) los campos equivalentes a Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago SUNAT, siguiendo el mismo patrón snapshot que los campos SAT de México — o documentar explícitamente por qué no aplican | Medio |
| GAP-09 | Actualizar `PedidoEstadoActual` al cerrar el pendiente Tramitar Pedido (OBS-027 resuelto) | Al insertar el pendiente FAA (paso 3i), invocar `PUT /v1/api/orders/status` con `IdTPPedido` + `IdCatEstadoPedido=prepagoconfaa` para cerrar el pendiente operativo (ver T7/T8 en `-Tareas.md`) | Medio |

---

## Flujo Back Completo

```
0. Front llama GET /Logistica/vTramitarPedidoDetalle?idTPPedido=... (RE-013 T8)
   → Retorna TieneProductosControlados=false, CorreoElectronicoEsac
   → Front renderiza radio button FAA (disponible, no seleccionado por defecto)
   → ESAC activa FAA → datos de facturación pasan a solo lectura en pantalla

1. ESAC hace clic en "SOLICITAR FACTURA POR ADELANTADO"
   → Front muestra pantalla de resumen (datos entrega | datos facturación | partidas + totales)

2. ESAC hace clic en "ENVIAR SOLICITUD" → Front llama
   POST /v1/api/invoices/advance-invoice/{orderId} en ProquifaDotNet:
   a. Valida: SinCredito=1, TieneControlados()=false, FAA=1, Remisión=0,
      datos de facturación sin cambios
   b. Valida: FAA + controlados no permitido (validación defensiva — R13)
   c. Fija datos de facturación del catálogo del cliente vigente
      (DatosFacturacionCliente)
   d. Genera el FolioPedidoInterno y lo asigna a tpPedido → COMMIT
      → si falla el guardado: rollback aquí, NO se llama a Finanzas
      → si el folio ya existe en tpPedido (reintento): se reutiliza,
        NO se regenera (idempotente)
   e. Llama POST /v1/api/invoices/advance-invoice/create/{orderId}
      en ProquifaDotNet.Finanzas
      → si esta llamada falla: NO se hace rollback; tpPedido conserva
        el folio; el reintento es seguro

3. ProquifaDotNet.Finanzas — nuevo servicio:
   a. NO genera tpProformaPedido ni tpProformaPartidaPedido
   b. NO llama a DocumentBuilder — no hay PDF
   c. NO envía correo
   d. Lee FolioPedidoInterno desde tpPedido (ya persistido por ProquifaDotNet)
   e. INSERT fccFactura con EsFacturaPorAdelantado = 1 (cabecera, pendiente FAA)
   f. INSERT fccFacturaPartida — una por cada partida del pedido (snapshot)
   g. INSERT fccFacturaReferenciaBancaria — referencias bancarias
      (cuentas M.N./DLS del grupo PROQUIFA + ReferenciaCliente por Código Validador)
      → e, f, g en una sola transacción (atómico)
   h. NO genera pendiente Validar Cobro
   i. Llama PUT /v1/api/orders/status con IdTPPedido + IdCatEstadoPedido=prepagoconfaa
      → localiza y actualiza el registro de PedidoEstadoActual (ya tiene IdTPPedido
        poblado desde etapas anteriores del flujo) → cierra pendiente Tramitar Pedido
      (OBS-027 resuelto — ver catEstadoPedido — Estados Propuestos.md)
```

---

## Datos del pendiente FAA (`fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`)

| Dato | Origen |
|---|---|
| IdTPPedido | tpPedido.IdTPPedido |
| EsFacturaPorAdelantado | = 1 (fijo en este flujo) |
| IdCliente | tpPedido.IdCliente |
| IdEmpresa | Empresa emisora (Proquifa) |
| FolioPedidoInterno | tpPedido.FolioPedidoInterno |
| IdCatCondicionesDePago | tpPedido |
| IdCatMoneda / TipoCambio / FactorConversionUSD | Moneda de facturación (RT-09) |
| SubTotal / IVA / MontoTotal / MontoTotalLetras | Partidas del pedido |
| Datos del receptor (RFC, Razón Social, CP, Régimen Fiscal, Uso CFDI, Método de Pago, Forma de Pago) | Snapshot de `DatosFacturacionCliente` vigente |
| IdTPProformaPedido | NULL en este flujo (Prepago no genera proforma) — se puebla solo en el origen Crédito, RE-FU-012 |
| IdCFDIGenerada | NULL en la FAA — se puebla al timbrar la factura final (RT-10). Serie/Folio/FolioFiscal/Version/TipoDeComprobante/FechaCertificacion se leen de `CFDIGenerada` vía este FK, no se duplican en `fccFactura` |
| Enviada | 0 al generar el pendiente — se pone en 1 al enviar la factura final por correo (RE-FU-018/019/020) |
| fccFacturaPartida (1:N) | Snapshot de partidas del pedido (`ClaveProductoServicioSAT`, `ClaveUnidadSAT`, cantidades, importes, impuestos) |
| fccFacturaReferenciaBancaria (1:N) | Cuentas M.N./DLS del grupo PROQUIFA (`EmpresaDatosBancarios`/`DatosBancarios`/`catBanco`) + `ReferenciaCliente` (Código Validador, RE-006) |

---

## Transferencia a Legacy

- En Tramitar Pedido NO hay transferencia para Prepago
- La transferencia ocurre después de Validar Cobro (fuera de scope)
- **Punto abierto:** al no generarse ningún PDF en este flujo (ni proforma ni confirmación), queda pendiente confirmar qué documento, si alguno, puede transferir a Legacy el job de TaskScheduler de Venta Digital para pedidos Prepago con FAA (ver Tareas, tarea de validación VD)

---

## Dependencias

| Requisito | Relación |
|-----------|----------|
| R16A-RE-FU-006 | Referencia bancaria del documento fiscal (Código Validador) — origen de `fccFacturaReferenciaBancaria.ReferenciaCliente` |
| R16A-RE-FU-010 | Cancelación de pedido (endpoint compartido) |
| R16A-RE-FU-013 | Verificación Perú (única parte del flujo base aún compartida); arquitectura orquestador→Finanzas ya validada ahí |
| R16A-RE-FU-014 | Validaciones compartidas: Remisión Prepago, datos facturación |
| R16A-RE-FU-016 | Criterio E1 — las dos cuentas (M.N./DLS) del grupo PROQUIFA México en el CFDI, reutilizado en `fccFacturaReferenciaBancaria` |
| R16A-RE-FU-018/019/020 | Módulo FAA consume el pendiente generado aquí (`fccFactura` con `EsFacturaPorAdelantado=1` → se actualiza a `0` y se llenan campos fiscales al timbrar) |

> RE-FU-016/017 (generación de PDF de proforma en DocumentBuilder) **ya no aplica** a este requisito — no se genera proforma en este flujo.

---

## Conclusión

El requisito R16A-RE-FU-015 tiene **impacto medio** en desarrollo Back tras adoptar el DIS-SOL v1.0: introduce tres tablas nuevas (`fccFactura`, `fccFacturaPartida`, `fccFacturaReferenciaBancaria`) propiedad de `ProquifaDotNet.Finanzas` y un endpoint orquestador nuevo en `ProquifaDotNet`. Ya no reutiliza el flujo base de proforma de RE-FU-013 (foliador, previsualización, envío) — solo comparte con RE-FU-014 las validaciones de Remisión y datos de facturación. Lógica específica de RE-015:

1. **Estructura BD del pendiente FAA** — CREATE TABLE `fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria` (ver `_BD.md`)
2. **Pendiente FAA directo** (GAP-01) — INSERT atómico de las tres tablas al tramitar, sin proforma previa
3. **Sin código de autorización** (GAP-03) — eliminar validación anterior
4. **Bloqueo datos facturación** (GAP-04) — fijar al activar FAA
5. **Vinculación con FAA** (GAP-05) — asegurar consumo del pendiente por RE-FU-018/019/020
6. **Punto abierto de Venta Digital/Legacy** (GAP-07) — confirmar comportamiento de TaskScheduler sin PDF disponible
7. **H-01 pendiente** (GAP-08) — campos fiscales de Perú en `fccFactura`
8. **Estatus del pedido — OBS-027 resuelto** (GAP-09) — actualizar `PedidoEstadoActual` al cerrar el pendiente Tramitar Pedido, vía el nuevo endpoint compartido `PUT /v1/api/orders/status` (T7/T8)

El desarrollador debe implementar el endpoint orquestador en `ProquifaDotNet` y el servicio de creación del pendiente en `ProquifaDotNet.Finanzas` conforme al DIS-SOL v1.0. `tpProformaAdelantoBO.cs` ya no es código de referencia para este requisito. OBS-027 quedó resuelto con el catálogo `catEstadoPedido` (extendido) y la tabla nueva `PedidoEstadoActual` — ya no aplica el `ALTER TABLE tpPedido ADD IdCatEstadoTpPedido` documentado en versiones anteriores de este archivo.
