# Impacto en Back — TPSC-RE-FU-012
**Requisito:** Tramitacion de pedidos Credito con Factura por Adelantado
**Aplicativo:** ProquifaDotNet
**Modulo:** L05.TramitarPedido
**Impacto:** Flujo tramitacion Credito preexistente + NUEVO pendiente FAA en tramitacion + validaciones

---

## Resumen

Este requisito reutiliza el flujo de tramitacion Credito existente (TPSC-RE-FU-010) con la activacion opcional de **Factura por Adelantado (FAA)**. Al tramitar con FAA activa, se genera un pendiente FAA dentro de la transaccion.

> **Nota:** La emision de la factura, generacion del CFDI y timbrado fiscal se desarrollan en los requisitos **TPSC-RE-FU-018**, **TPSC-RE-FU-019** y **TPSC-RE-FU-020**. Se generara una tarea para en su caso vincular el proceso de generacion del pendiente con el modulo de facturacion.

---

## Nota importante sobre codigo existente

> **El flujo anterior de Factura por Adelantado NO se reutilizara directamente.**
> El desarrollador asignado debera revisar las entidades y logica existentes para determinar que es aprovechable.
>
> **Archivos existentes a revisar:**
> - `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Anticipos\tpProformaAdelantoBO.cs`
> - `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Anticipos\tpProformaAdelantoBO.Extensions.cs`
> - `WebApi.Logistica\Controllers\Procesos\L05.TramitarPedido\Facturas\tpProformaAdelantoController.cs`
> - `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Generadores\Anticipo\tpProformaAdelantoToCFDIGeneradaBO.cs`
> - `Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Generadores\Anticipo\CFDIGeneradaConceptoAnticipoFactory.cs`
> - Tablas BD: `tpProformaAdelanto`, `tpProformaAdelantoProformaPedido`, `fccPagoFacturaAdelanto`
> - Entidades CFDI: `CFDIGenerada`, `CFDI`, `CFDIGeneradaConcepto`

---

## Etapa 1 — Tramitar Pedido con FAA (Generacion del Pendiente)

### Clase principal
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

### Flujo al tramitar con FAA activa

| Paso | Accion | Detalle |
|------|--------|---------|
| 1 | ESAC activa FAA en UI | Front envia `tpPedido.FacturaPorAdelantado = 1` |
| 2 | Validaciones | FAA solo Mexico, sin controlados, datos facturacion vigentes |
| 3 | Tramitacion normal | Flujo Credito completo (folio, PDF, partidas, correo) |
| 4 | Genera pendiente FAA | INSERT en tabla de pendientes FAA (atomico con tramitacion) |
| 5 | Bloquea datos facturacion | Se fijan del catalogo del cliente |
| 6 | Confirmacion de Pedido | Se genera inmediatamente (no espera factura) |
| 7 | Transferencia Legacy | Solo Mexico (independiente de FAA) |

### Datos del pendiente FAA a generar

| Dato | Origen |
|------|--------|
| Cliente | tpPedido.IdCliente |
| Empresa | tpPedido.IdEmpresa |
| Monto total | Calculado desde partidas del pedido |
| Orden de compra | tpPedido.OrdenDeCompra / FolioPedidoInterno |
| Datos facturacion | DatosFacturacionCliente vigente |
| Region | tpPedido.IdRegion (solo Mexico) |
| Moneda | MXN o USD segun pedido |
| Estado inicial | Pendiente de emision |

---

## Etapa 2 — Emision de Factura + CFDI + Timbrado (Referencia)

> **Este alcance se desarrolla en:**
> - **TPSC-RE-FU-018** — Generacion de factura
> - **TPSC-RE-FU-019** — Generacion de CFDI
> - **TPSC-RE-FU-020** — Timbrado fiscal (PAC)
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

Dependencia de TPSC-RE-FU-010 T3.

---

## Gaps de Desarrollo

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Revision de tpProformaAdelanto existente | Desarrollador revisa codigo/tablas para determinar aprovechabilidad | Medio |
| GAP-02 | Generacion de pendiente FAA en transaccion de tramitacion | INSERT pendiente dentro de GenerarCorreoTramitarPedido() cuando FAA=1 | Medio |
| GAP-03 | Validacion Back: FAA solo Mexico | Rechazar si FAA=1 y region != Mexico | Bajo |
| GAP-04 | Eliminar codigo de autorizacion para FAA | Buscar y eliminar si existe | Bajo |
| GAP-05 | Bloquear edicion datos facturacion con FAA activa | Validacion en endpoint de edicion | Bajo |
| GAP-06 | Vinculacion con modulo de facturacion | Tarea para vincular pendiente FAA con flujo RE-FU-018/019/020 | Bajo |
| GAP-07 | Endpoint Cancelacion | Dependencia RE-FU-010 T3 | Referencia |

---

## Transferencia a Legacy

Sin cambios respecto a RE-FU-010. La FAA es un proceso paralelo independiente.

---

## Dependencias

| Requisito      | Relacion                                          |
| -------------- | ------------------------------------------------- |
| TPSC-RE-FU-010 | Flujo base Credito + Endpoint Cancelacion         |
| TPSC-RE-FU-011 | Restriccion: FAA NO compatible con controlados    |
| TPSC-RE-FU-018 | Generacion de factura (consumo del pendiente FAA) |
| TPSC-RE-FU-019 | Generacion de CFDI                                |
| TPSC-RE-FU-020 | Timbrado fiscal (PAC)                             |

---

## Conclusion

El requisito TPSC-RE-FU-012 tiene **impacto medio** en desarrollo. Comprende:

**Bloque 1 (Tramitar Pedido):** Generar pendiente FAA atomicamente con la tramitacion + validaciones (solo Mexico, sin controlados, bloqueo datos, sin codigo autorizacion).

**Bloque 2 (Vinculacion):** Asegurar que el pendiente generado sea consumido por el modulo de facturacion desarrollado en RE-FU-018/019/020.

El desarrollador asignado debe revisar `tpProformaAdelantoToCFDIGeneradaBO.cs` (codigo comentado) como referencia de la logica anterior de generacion de CFDI para anticipos.
