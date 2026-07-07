# Impacto en Back — R16A-RE-FU-012
**Requisito:** Tramitacion de pedidos Credito con Factura por Adelantado
**Aplicativo:** ProquifaDotNet
**Modulo:** L05.TramitarPedido
**Impacto:** Flujo tramitacion Credito preexistente + NUEVO pendiente FAA en tramitacion + validaciones

---

## Resumen

Este requisito reutiliza el flujo de tramitacion Credito existente (R16A-RE-FU-010) con la activacion opcional de **Factura por Adelantado (FAA)**. Al tramitar con FAA activa, se genera un pendiente FAA dentro de la transaccion.

> **Nota:** La emision de la factura, generacion del CFDI y timbrado fiscal se desarrollan en los requisitos **R16A-RE-FU-018**, **R16A-RE-FU-019** y **R16A-RE-FU-020**. Se generara una tarea para en su caso vincular el proceso de generacion del pendiente con el modulo de facturacion.

---

## Nota importante sobre codigo existente

> **El flujo anterior de Factura por Adelantado NO se reutilizara directamente.**
> **Actualización (06/07/2026):** el pendiente FAA de este requisito ya NO se modela con
> `tpProformaAdelanto`. Se unifica con el esquema `fccFactura` + `fccFacturaPartida` +
> `fccFacturaReferenciaBancaria` definido y creado en RE-FU-015, propiedad de
> `ProquifaDotNet.Finanzas`. La diferencia con Prepago (RE-015) es que aquí
> `fccFactura.IdTPProformaPedido` se puebla con la Confirmación de Pedido
> (`tpProformaPedido`) generada en paralelo — ver `R16A-RE-FU-015_BD.md`, sección
> "Migración de tpProformaAdelanto".
>
> El desarrollador asignado debera revisar las entidades y logica existentes para determinar que es aprovechable, con el entendido de que el destino final es `fccFactura` (Finanzas), no `tpProformaAdelanto` (ProquifaDotNet).
>
> **Archivos legacy a revisar (solo como referencia de lógica de negocio, ya no como destino de datos):**
> - `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Anticipos\tpProformaAdelantoBO.cs`
> - `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Anticipos\tpProformaAdelantoBO.Extensions.cs`
> - `WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\Facturas\tpProformaAdelantoController.cs`
> - `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Generadores\Anticipo\tpProformaAdelantoToCFDIGeneradaBO.cs`
> - `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Generadores\Anticipo\CFDIGeneradaConceptoAnticipoFactory.cs`
> - Tablas BD legacy (ya no aplican como destino): `tpProformaAdelanto`, `tpProformaAdelantoProformaPedido`
> - Tabla BD vigente (FK migrada): `fccPagoFacturaAdelanto` (ver RE-FU-026/027)
> - Entidades CFDI: `CFDIGenerada` (Finanzas, single source of truth), `CFDIGeneradaConcepto`

---

## Etapa 1 — Tramitar Pedido con FAA (Generacion del Pendiente)

### Clase principal
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

### Flujo al tramitar con FAA activa

| Paso | Accion                    | Detalle                                                      |
| ---- | ------------------------- | ------------------------------------------------------------ |
| 1    | ESAC activa FAA en UI     | Front envia `tpPedido.FacturaPorAdelantado = 1`              |
| 2    | Validaciones              | FAA solo Mexico, sin controlados, datos facturacion vigentes |
| 3    | Tramitacion normal        | Flujo Credito completo (folio, PDF, partidas, correo)        |
| 4    | Genera pendiente FAA      | INSERT atómico en `fccFactura`+`fccFacturaPartida`+`fccFacturaReferenciaBancaria` (Finanzas), en paralelo a `tpProformaPedido` |
| 5    | Bloquea datos facturacion | Se fijan del catalogo del cliente                            |
| 6    | Confirmacion de Pedido    | Se genera inmediatamente (no espera factura)                 |
| 7    | Transferencia Legacy      | Solo Mexico (independiente de FAA)                           |

### Datos del pendiente FAA a generar (en `fccFactura` + detalle)

| Dato | Origen |
|------|--------|
| IdTPPedido | tpPedido.IdTPPedido (FK directa) |
| IdTPProformaPedido | Id de `tpProformaPedido` (Confirmación de Pedido) generada en paralelo |
| Cliente | tpPedido.IdCliente |
| Empresa | tpPedido.IdEmpresa |
| Monto total | Calculado desde partidas del pedido |
| FolioPedidoInterno | tpPedido.FolioPedidoInterno |
| Datos facturacion | Snapshot de DatosFacturacionCliente vigente |
| Region | tpPedido.IdRegion (solo Mexico) |
| Moneda | MXN o USD segun pedido |
| IdCFDIGenerada | NULL (pendiente de emisión) |
| Enviada | 0 (pendiente de envío) |
| fccFacturaPartida | Snapshot de las partidas del pedido |
| fccFacturaReferenciaBancaria | Cuentas M.N./DLS del grupo PROQUIFA |

---

## Etapa 2 — Emision de Factura + CFDI + Timbrado (Referencia)

> **Este alcance se desarrolla en:**
> - **R16A-RE-FU-018** — Generacion de factura
> - **R16A-RE-FU-019** — Generacion de CFDI
> - **R16A-RE-FU-020** — Timbrado fiscal (PAC)
>
> **Nota para el desarrollador:** Se generara una tarea en este requisito para vincular el proceso de generacion del pendiente (Etapa 1) con el flujo de facturacion de los requisitos mencionados. El objetivo es asegurar que el pendiente FAA generado en la tramitacion sea consumido correctamente por el modulo de facturacion.
>
> **Referencia de codigo existente:** `tpProformaAdelantoToCFDIGeneradaBO.cs` contiene logica comentada de generacion de CFDI para anticipos que puede servir como guia.

---

## Seccion C — Bloqueo de Datos de Facturacion

### Comportamiento requerido

Al activar FAA:
- **Front:** oculta boton "Editar Datos" de facturacion
- **Back:** datos se toman del catalogo vigente del cliente y se fijan
- **Validacion Back:** rechazar actualizacion si `tpPedido.FacturaPorAdelantado = 1`

### Datos que se fijan (DatosFacturacionCliente) — regionalizados (Regla 9 — OBS sincronización matriz)

Los campos que se fijan y se exponen en el Panel de Información de Facturación dependen de la región del pedido. La estructura es la misma para ambas regiones, pero los campos fiscales difieren:

**Campos comunes (México y Perú):**

| Campo       | Tipo           | Descripción |
|-------------|----------------|-------------|
| RFC / RUC   | varchar(50)    | Identificador fiscal (RFC para MX, RUC para PE) — etiqueta unificada en UI |
| RazonSocial | varchar(120)   | Razón social legal del cliente |
| Correo      | varchar(200)   | Correo de envío de la factura |

**Campos fiscales México:**

| Campo                 | Tipo             | Descripción |
|-----------------------|------------------|-------------|
| IdCatUsoCFDI          | uniqueidentifier | Uso CFDI (catálogo SAT) |
| IdCatMetodoDePagoCFDI | uniqueidentifier | Método de Pago (catálogo SAT) |
| IdCatRegimenFiscal    | uniqueidentifier | Régimen Fiscal (catálogo SAT) |

**Campos fiscales Perú:**

| Campo            | Tipo             | Descripción |
|------------------|------------------|-------------|
| TipoOperacion    | varchar(50)      | Tipo de Operación (SUNAT) |
| CondicionPago    | varchar(50)      | Condición de Pago (SUNAT) |

> **Nota:** Los campos `Forma de Pago` y `correo de envío` NO se muestran en el Panel de Información de Facturación (Regla 9). La Forma de Pago se captura en Validar Cobro; el correo de envío se gestiona en el paso de envío de factura.

---

## Seccion D — Eliminacion del Codigo de Autorizacion

Regla 3: activacion directa sin codigo. Eliminar validacion si existe.

---

## Seccion E — Restriccion solo Mexico

Validacion Back: rechazar si `FAA=1` AND `region != Mexico`

---

## Seccion F — Cancelacion (Criterio C3)

Dependencia de R16A-RE-FU-010 T3.

---

## Gaps de Desarrollo

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Revision de tpProformaAdelanto existente (solo como referencia de lógica) | Desarrollador revisa codigo/tablas legacy para determinar qué lógica es aprovechable hacia `fccFactura` | Medio |
| GAP-02 | Generacion de pendiente FAA en transaccion de tramitacion | INSERT atómico en `fccFactura`+`fccFacturaPartida`+`fccFacturaReferenciaBancaria` (Finanzas) dentro de `GenerarCorreoTramitarPedido()` cuando FAA=1, en paralelo a `tpProformaPedido`, poblando `IdTPProformaPedido` | Medio |
| GAP-03 | Validacion Back: FAA solo Mexico | Rechazar si FAA=1 y region != Mexico | Bajo |
| GAP-04 | Eliminar codigo de autorizacion para FAA | Buscar y eliminar si existe | Bajo |
| GAP-05 | Bloquear edicion datos facturacion con FAA activa | Validacion en endpoint de edicion | Bajo |
| GAP-06 | Vinculacion con modulo de facturacion | Tarea para vincular pendiente FAA con flujo RE-FU-018/019/020 | Bajo |
| GAP-07 | Endpoint Cancelacion | Dependencia RE-FU-010 T3 | Referencia |

---

## Transferencia a Legacy

Sin cambios respecto a RE-FU-010. La FAA es un proceso paralelo independiente.

> **OBS-024 — PCE (Pago Contra Entrega) traduce como crédito en Legacy:**
> Cuando `catCondicionesDePago.Clave = 'pagocontraentrega'`, el payload builder (`EtlPedidoCreditoPayloadBuilder`) debe traducirlo como **crédito** en Legacy, **NO como prepago**. Aunque el nombre sugiera pago adelantado, Legacy lo procesa como flujo de crédito.

> **OBS-025 — PQF2 solo inserta datos planos; "Relacionar facturas" es responsabilidad de Legacy:**
> El payload builder de PQF2 únicamente inserta los datos planos del pedido en Legacy. La lógica de "Relacionar facturas" (asociación de facturas a pedido dentro de Legacy) es responsabilidad del proceso interno de Legacy, **no del payload builder** de ProquifaDotNet.

---

## Dependencias

| Requisito      | Relacion                                          |
| -------------- | ------------------------------------------------- |
| R16A-RE-FU-010 | Flujo base Credito + Endpoint Cancelacion         |
| R16A-RE-FU-011 | Restriccion: FAA NO compatible con controlados    |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria`/`vfccFactura` — este requisito solo consume, poblando `IdTPProformaPedido` |
| R16A-RE-FU-018 | Generacion de factura (consumo del pendiente FAA vía `vfccFactura`) |
| R16A-RE-FU-019 | Generacion de CFDI                                |
| R16A-RE-FU-020 | Timbrado fiscal (PAC)                             |
| R16A-RE-FU-026/027 | Migración de `fccPagoFacturaAdelanto.IdTPProformaAdelanto` → `IdFccFactura` |

---

## Conclusion

El requisito R16A-RE-FU-012 tiene **impacto medio** en desarrollo. Comprende:

**Bloque 1 (Tramitar Pedido):** Generar pendiente FAA en `fccFactura` (esquema de RE-FU-015) atomicamente con la tramitacion + validaciones (solo Mexico, sin controlados, bloqueo datos, sin codigo autorizacion). La diferencia con Prepago es que aquí `IdTPProformaPedido` se puebla con la Confirmación de Pedido generada en paralelo.

**Bloque 2 (Vinculacion):** Asegurar que el pendiente generado en `fccFactura` sea consumido por el modulo de facturacion desarrollado en RE-FU-018/019/020 (vía `vfccFactura`).

El desarrollador asignado debe revisar `tpProformaAdelantoToCFDIGeneradaBO.cs` (codigo comentado) como referencia de la logica anterior de generacion de CFDI para anticipos, adaptándola al destino `fccFactura`/`CFDIGenerada` en Finanzas.
