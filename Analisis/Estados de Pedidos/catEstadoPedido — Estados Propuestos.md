# catEstadoPedido — Estados Propuestos

**Referencia:** OBS-027 (bloqueante RE-FU-015) — desbloqueo de tareas T7 (BD) y T8 (Back).
**Alcance:** Catálogo del ciclo de vida del pedido (`tpPedido.IdCatEstadoPedido`), derivado del rastreo de flujos reales del aplicativo ProquifaDotNet y de la nueva arquitectura R16 (Facturas, Proforma, Validar Cobro, CFDI y transferencias a Legacy migran a **ProquifaDotNet.Finanzas** a partir del RE-FU-016).
**Fecha:** 2026-08-11 · **Actualizado 11/08/2026:** el estado genérico `INTRAMITABLE_GESTIONADO` (v1) se **separó en 4 estados** con evidencia directa de código (ver secciones 2, 3 y 7) — dos de ellos **no resuelven** el intramitable (`INTRAMITABLE_PENDIENTE_ACEPTACION`, `INTRAMITABLE_FEA_SOLICITADA`) y dos sí lo resuelven (`INTRAMITABLE_OC_ACEPTADA`, `INTRAMITABLE_TRAMITADO_CON_ERRORES`). La cancelación (antes agrupada ahí también) **no necesitó estado nuevo**: ya la cubre `CANCELADO_EN_PRETRAMITAR`.

---

## 1. Contexto

El catálogo `catEstadoPedido` está pendiente de propuesta del cliente desde el diseño del RE-FU-015. Los tres catálogos existentes en BD (`catEstadoPedido`, `catEstadoPartidaPedido`, `catEstadoPretramitacionPedido`) están vacíos en el código, sin FK a `tpPedido` y sin datos sembrados en el dump verificado (`ProquifaDotNet.sql`).

### Cambio de propietario por la arquitectura R16

A partir del **RE-FU-016**, las piezas de facturación, proforma, validar cobro, CFDI y transferencias a Legacy dejan de vivir en ProquifaDotNet (Venta Interna, .NET Framework 4.8) y pasan a **ProquifaDotNet.Finanzas** (.NET Core 10). Como consecuencia, el ciclo de vida del pedido se reparte entre dos aplicativos:

| Aplicativo | Etapas del ciclo | Requisitos R16 |
|---|---|---|
| **ProquifaDotNet** (Venta Interna) | Cotización → Promesa de Compra → Pretramitar → Tramitar (Liberar) | Legacy + RE-FU-013 / 014 / 015 |
| **ProquifaDotNet.Finanzas** | Proforma → Validar Cobro → Facturación (CFDI) → Transferencia a Legacy | RE-FU-016, 019, 020, 023, 024–029, 028, 030, 032, LegacySync |
| **ProquifaDotNet.Timbrado** | Timbrado del CFDI (fase interna de Facturación) | RE-FU-018, 028, 030, 032 |

Aunque el ciclo se ejecuta en dos aplicativos, la **fuente única de la verdad del estado del pedido cabecera** es `tpPedido.IdCatEstadoPedido` en la BD `ProquifaDotNet`. Finanzas **escribe este campo** mediante los servicios de Validar Cobro y Facturación (Scaffold EF Core sobre `tpPedido`, con guardias de idempotencia).

---

## 2. Fuente de la propuesta

Los estados se derivaron del rastreo de flujos en el aplicativo `ProquifaDotNet-R14` y del alcance funcional documentado en los requisitos R16:

| Módulo / Aplicativo | Ubicación | Aporte al ciclo |
|---|---|---|
| ProquifaDotNet · L03 Promesa de Compra | `PretramitarPromesaDeCompraTransaccionBO.cs` | Creación de `ppPedido` desde cotización confirmada |
| ProquifaDotNet · L04 Pretramitar Pedido | `ppPedidoBO.cs`, `PretramitarPedidoTransaccionBO.cs`, `TramitarPedidoBO.cs` | Flags `Tramitado`, `Intramitable`; bandejas de pendientes |
| ProquifaDotNet · L04 Gestionar Intramitables (resolución) | `ppPedidoAceptarOCInternaTransaccionBO.cs`, `ppPedidoTramitacionConErroresTransaccionBO.cs`, `ppPedidoCancelacionBO.cs` | Resolución (OC interna / tramitación con errores) o cancelación de intramitables |
| ProquifaDotNet · L04 Gestionar Intramitables (correo previo, pendiente de respuesta) | `ppPedidoIncidenciaCorreoTransaccionBO.cs`, `ppPedidoOcNoAmparadaCorreoTransaccioBO.cs` | Correo de aceptación enviado al cliente (tabla `ppPedidoAceptacionConError`: `Enviado`/`Aceptada`/`Procesado`/`Caducado`); no resuelve, solo notifica y espera respuesta |
| ProquifaDotNet · L04 Gestionar Intramitables (diferimiento) | `ppPedidosSolicitarFEATransaccionBO.cs` | Solicitud de **FEA** ("Fecha Estimada de Ajuste", `ppPedido.FechaEstimadaAjuste`) — el pedido **sigue Intramitable**, no se resuelve |
| ProquifaDotNet · L05 Tramitar Pedido | `tpPedidoTramitarTransaccionBO.cs`, `tpPedidoBO.cs` | `FechaTramitacion`, `Liberado=true` |
| **Finanzas · RE-FU-016 Proforma MEX** | `tpProformaPedido` (movida a Scaffold Finanzas 07/07/2026) | Proforma con `MontoPendiente > 0` dispara Validar Cobro |
| **Finanzas · RE-FU-015/019/020 Prepago con FAA** | `fccFactura` + `fccFacturaPartida` (nuevas R16) | Factura por Adelantado activa antes de Validar Cobro |
| **Finanzas · RE-FU-023 Validar Cobro (pantalla principal)** | Orquestador de cancelación distribuida | `CANCELADO_POR_FALTA_PAGO` con trazabilidad en `tpPedido` |
| **Finanzas · RE-FU-024–029 Wizard Validar Cobro** | `fccPagoCliente` / `fccCobroCliente` (fusión RE-024) | Cobro capturado → asociado → cerrado |
| **Finanzas · RE-FU-028 Facturación MEX / RE-029 Perú** | `CFDIGenerada`, `fccDocumentoFiscalCobro` | CFDI timbrado → estado `FACTURADO` |
| **Finanzas · RE-FU-030 Complemento de Pago** | `PaymentComplementCalculationService` | CP timbrado dentro de facturación PPD |
| **LegacySync** | ETLs E1–E8 (RE-FU-028) | Transferencia a Legacy → estado `TRANSFERIDO_A_LEGACY` |
| ProquifaDotNet · L06–L09 | OrdenDeCompra, Importaciones, Inspección, Embalar | Seguimiento post-facturación (Compra, Almacén, Entrega) |
| ProquifaDotNet · `ServicioLegacyBO` + `ExternalAppSettings.config` (`SeguimientoPedidoVD`) | Claves publicadas al cliente en Venta Digital | `encompra`, `almacenmatriz`, `rechazadoeninspeccion`, `enentrega`, `pedidoentregado` |

---

## 3. Estados propuestos

| #   | Clave                      | Descripción (`EstadoPedido`)                                                              | Aplicativo que lo escribe                                                                       | Requisito R16                             | Bandeja / evento                                                                                   | Terminal |
| --- | -------------------------- | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------- | -------- |
| 1   | `PEDIDO_GENERADO`          | Pedido generado (Promesa de Compra confirmada)                                            | **ProquifaDotNet** (L03)                                                                        | RE-FU-013 / 014                           | Al pasar cotización a `ppPedido`                                                                   | No       |
| 2   | `EN_PRETRAMITAR`           | En Pretramitar Pedido                                                                     | **ProquifaDotNet** (L04)                                                                        | RE-FU-013 / 014                           | `ppPedido` con `Tramitado=false, Intramitable=null`                                                | No       |
| 3   | `INTRAMITABLE`             | Intramitable — requiere gestión                                                           | **ProquifaDotNet** (L04)                                                                        | RE-FU-013 / 014                           | `ppPedido.Intramitable=true`                                                                       | No       |
| 4   | `INTRAMITABLE_PENDIENTE_ACEPTACION` | Correo de aceptación enviado al cliente (OC interna o tramitación con errores), pendiente de respuesta | **ProquifaDotNet** (L04)                                              | RE-FU-013 / 014                           | `ppPedidoAceptacionConError.Enviado=true, Aceptada=false`                                          | No       |
| 5   | `INTRAMITABLE_FEA_SOLICITADA` | Solicitud de FEA (Fecha Estimada de Ajuste) enviada — sigue Intramitable, solo se difiere/ajusta fecha | **ProquifaDotNet** (L04)                                                        | RE-FU-013 / 014                           | `ppPedido.FechaEstimadaAjuste` actualizada (`Intramitable` sin cambio)                             | No       |
| 6   | `INTRAMITABLE_OC_ACEPTADA` | Intramitable resuelto — cliente aceptó OC Interna, pedido tramitado                        | **ProquifaDotNet** (L04)                                                                        | RE-FU-013 / 014                           | `ppPedido.Intramitable=false, Tramitado=true, OcInterna=true`                                      | No       |
| 7   | `INTRAMITABLE_TRAMITADO_CON_ERRORES` | Intramitable resuelto — tramitación con errores aceptada, pedido tramitado (genera `tpPedido` + cotización) | **ProquifaDotNet** (L04)                                              | RE-FU-013 / 014                           | `ppPedido.Intramitable=false, Tramitado=true` (+ genera `tpPedido`/cotización)                     | No       |
| 8   | `CANCELADO_EN_PRETRAMITAR` | Cancelado en Pretramitar                                                                  | **ProquifaDotNet** (L04)                                                                        | RE-FU-013 / 014                           | `ppPedido.Cancelada=true`                                                                          | **Sí**   |
| 9   | `TRAMITADO`                | Tramitado — `tpPedido` creado y liberado (enviado a Compras / Producción)                 | **ProquifaDotNet** (L05)                                                                        | RE-FU-013 / 014 / 015                     | `tpPedido.Tramitado=true`, `FechaTramitacion` seteada, `Liberado=true`                             | No       |
| 10  | `PREPAGO_CON_FAA`          | Prepago con Factura por Adelantado pendiente de cobro                                     | **Finanzas**                                                                                    | RE-FU-015 / 019 / 020                     | Al activar FAA (`fccFactura` con estado `EMITIDA_SIN_COBRO`) — dispara pendiente en Validar Cobro  | No       |
| 11  | `EN_VALIDAR_COBRO`         | En Validar Cobro (proforma con saldo pendiente / cobro recibido no aplicado)              | **Finanzas**                                                                                    | RE-FU-016 / 023 / 024–027                 | Proforma con `MontoPendiente > 0` **o** correo de cobro en Buzón sin aplicar                       | No       |
| 12  | `FACTURADO`                | Facturado (CFDI timbrado; Complemento de Pago timbrado si aplica)                         | **Finanzas** (llama Timbrado)                                                                   | RE-FU-028 / 029 / 030                     | `CFDIGenerada` timbrado y (si PPD) `Complemento de Pago` timbrado                                  | No       |
| 13  | `TRANSFERIDO_A_LEGACY`     | Transferido a Legacy (ETLs de Factura + PDF + Complemento enviados)                       | **Finanzas · LegacySync**                                                                       | RE-FU-028 (ETLs E3/E6), RE-FU-030 (E4/E7) | Todos los ETLs del pedido/factura completados                                                      | No       |
| 14  | `EN_COMPRA`                | En Compra (OC generada, seguimiento del proveedor)                                        | ProquifaDotNet (L06/L07)                                                                        | Legacy                                    | `catSeguimientoPartidaPedido.Clave='encompra'`                                                     | No       |
| 15  | `EN_ALMACEN_MATRIZ`        | En Almacén Matriz                                                                         | ProquifaDotNet (L07/L08)                                                                        | Legacy                                    | `Clave='almacenmatriz'`                                                                            | No       |
| 16  | `RECHAZADO_EN_INSPECCION`  | Rechazado en Inspección                                                                   | ProquifaDotNet (L08)                                                                            | Legacy                                    | `Clave='rechazadoeninspeccion'`                                                                    | No       |
| 17  | `EN_ENTREGA`               | En Entrega (en ruta al cliente)                                                           | ProquifaDotNet (L09)                                                                            | Legacy                                    | `Clave='enentrega'`, `tpPedidoVD.MostrarEnRuta=true`                                               | No       |
| 18  | `ENTREGADO`                | Entregado al cliente                                                                      | ProquifaDotNet (L09)                                                                            | Legacy                                    | `Clave='pedidoentregado'`                                                                          | No       |
| 19  | `CANCELADO_POR_FALTA_PAGO` | Cancelado por falta de pago (Validar Cobro)                                               | **Finanzas** (orquestador) → **ProquifaDotNet** (trazabilidad `tpPedido`) → **Timbrado** (CFDI) | RE-FU-023                                 | `tpPedido.FechaCancelacionPorFaltaPago` seteada + `tpProformaPedido.IdcatEstadoProforma=Cancelada` | **Sí**   |
| 20  | `CANCELADO`                | Cancelado (cancelación operativa fuera de flujo de pago)                                  | ProquifaDotNet (L10 / `tpPedidoCancelacionController`)                                          | Legacy / RE-010                           | Cancelación desde bandeja de cancelaciones                                                         | **Sí**   |
| 21  | `FINALIZADO`               | Finalizado — pedido cerrado, entregado, facturado y transferido                           | **Finanzas** (evento final)                                                                     | Post RE-FU-028 + LegacySync               | Cuando se completa la cadena 13 + 18                                                                | **Sí**   |

> **Reordenamiento respecto a v1:** los estados de Validar Cobro (`EN_VALIDAR_COBRO`), Facturación (`FACTURADO`) y Legacy (`TRANSFERIDO_A_LEGACY`) se colocan **antes** de la cadena logística (Compra → Entrega) porque en la operación real el cobro se resuelve inmediatamente después del tramitado (proforma emitida con `MontoPendiente > 0`), y la facturación puede ocurrir en paralelo o antes que la salida física del almacén.

---

## 4. Transiciones observadas

```
[ProquifaDotNet — Venta Interna]
PEDIDO_GENERADO
   └─► EN_PRETRAMITAR
         ├─► INTRAMITABLE
         │      ├─► INTRAMITABLE_PENDIENTE_ACEPTACION (correo enviado, esperando al cliente)
         │      │      ├─► INTRAMITABLE_OC_ACEPTADA ──────────────┐
         │      │      └─► INTRAMITABLE_TRAMITADO_CON_ERRORES ────┤
         │      ├─► INTRAMITABLE_FEA_SOLICITADA (solo difiere fecha, sigue Intramitable)
         │      └─► CANCELADO_EN_PRETRAMITAR                                     [terminal]
         │                                                          │
         ├─► CANCELADO_EN_PRETRAMITAR                               │             [terminal]
         └─► TRAMITADO ◄────────────────────────────────────────────┘
               │
               │ ──────────── (Finanzas asume el ciclo) ────────────
               ▼
         [Finanzas]
         ├─► PREPAGO_CON_FAA ─► EN_VALIDAR_COBRO
         └─► EN_VALIDAR_COBRO
               ├─► CANCELADO_POR_FALTA_PAGO  (orquestador: Finanzas + PQF + Timbrado)   [terminal]
               └─► FACTURADO  (Finanzas + Timbrado)
                     └─► TRANSFERIDO_A_LEGACY  (LegacySync)
                           │
                           │ ──── (ProquifaDotNet reasume tracking físico) ────
                           ▼
                     [ProquifaDotNet — Logística]
                     └─► EN_COMPRA
                           └─► EN_ALMACEN_MATRIZ
                                 ├─► RECHAZADO_EN_INSPECCION ─► EN_COMPRA / EN_ALMACEN_MATRIZ (reproceso)
                                 └─► EN_ENTREGA
                                       └─► ENTREGADO
                                             └─► FINALIZADO  (Finanzas cierra ciclo)   [terminal]
```

> **Nota sobre la coordinación entre aplicativos:** las transiciones que involucran cambio de propietario (`TRAMITADO → PREPAGO_CON_FAA / EN_VALIDAR_COBRO`, `TRANSFERIDO_A_LEGACY → EN_COMPRA`, `ENTREGADO → FINALIZADO`) se ejecutan como escrituras cross-aplicativo sobre `tpPedido.IdCatEstadoPedido` desde Finanzas vía Scaffold EF Core (patrón declarado en `Base de datos nombramiento ProquifaDotNet.md` y en el diseño de RE-FU-023 sección 1.4).

---

## 5. Matriz de transiciones permitidas

Legenda: **✅** permitida · **✋** requiere autorización especial · **⛔** no permitida

> **Nota técnica (11/08/2026):** esta matriz se **reconstruyó desde la lógica del diagrama (sección 4) y la tabla de estados (sección 3)**, no por copia mecánica de la matriz v1 — se detectó que la fila `RECHAZADO_EN_INSPECCION` de v1 tenía una celda de menos (16 en vez de 17), inconsistencia que no convenía arrastrar ni tratar de reproducir.

| Origen \ Destino | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1. PEDIDO_GENERADO | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ⛔ |
| 2. EN_PRETRAMITAR | — | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ⛔ |
| 3. INTRAMITABLE | ⛔ | — | ✅ | ✅ | ⛔ | ⛔ | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ⛔ |
| 4. INTRAMITABLE_PENDIENTE_ACEPTACION | ⛔ | ⛔ | — | ⛔ | ✅ | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ |
| 5. INTRAMITABLE_FEA_SOLICITADA | ⛔ | ⛔ | ⛔ | — | ⛔ | ⛔ | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ |
| 6. INTRAMITABLE_OC_ACEPTADA | ✅ | ⛔ | ⛔ | ⛔ | — | ⛔ | ✅ | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ⛔ |
| 7. INTRAMITABLE_TRAMITADO_CON_ERRORES | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | — | ✅ | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ⛔ |
| 8. CANCELADO_EN_PRETRAMITAR | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ |
| 9. TRAMITADO | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ✅ | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ⛔ |
| 10. PREPAGO_CON_FAA | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ✋ | ⛔ |
| 11. EN_VALIDAR_COBRO | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ✋ | ⛔ |
| 12. FACTURADO | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ |
| 13. TRANSFERIDO_A_LEGACY | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ |
| 14. EN_COMPRA | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ✅ | ⛔ | ⛔ | ⛔ | ⛔ | ✋ | ⛔ |
| 15. EN_ALMACEN_MATRIZ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ✅ | ✅ | ⛔ | ⛔ | ✋ | ⛔ |
| 16. RECHAZADO_EN_INSPECCION | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ✅ | ✅ | — | ⛔ | ⛔ | ⛔ | ✋ | ⛔ |
| 17. EN_ENTREGA | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ✅ | ⛔ | ⛔ | ⛔ |
| 18. ENTREGADO | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ⛔ | ⛔ | ✅ |
| 19. CANCELADO_POR_FALTA_PAGO | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ⛔ | ⛔ |
| 20. CANCELADO | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — | ⛔ |
| 21. FINALIZADO | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | — |

> **Nota sobre cancelaciones:** las tres cancelaciones (8, 19, 20) se separan por origen y aplicativo para trazabilidad de auditoría. `CANCELADO_EN_PRETRAMITAR` aplica antes del tramitado (ProquifaDotNet L04, incluso directo desde `INTRAMITABLE` — el BO de cancelación no exige pasar por un estado de gestión previo); `CANCELADO_POR_FALTA_PAGO` es el flujo distribuido de RE-FU-023 (orquestador Finanzas + trazabilidad PQF + cancelación CFDI Timbrado); `CANCELADO` cubre cancelaciones operativas del `tpPedidoCancelacionController` en Logística.

---

## 6. Responsabilidades por aplicativo — escritura de `tpPedido.IdCatEstadoPedido`

| Aplicativo | Estados que escribe | Mecanismo |
|---|---|---|
| **ProquifaDotNet** (Venta Interna) | 1, 2, 3, 4, 5, 6, 7, 8, 9, 14, 15, 16, 17, 18, 20 | Escritura directa desde los BOs de L03–L10 con `Core.Pqf.ProquifaDotNetContext`. |
| **ProquifaDotNet.Finanzas** | 10, 11, 12, 13, 19 (parte proforma + trazabilidad), 21 | Scaffold EF Core sobre `tpPedido` (patrón declarado en `Base de datos nombramiento ProquifaDotNet.md`). En RE-FU-023 el orquestador de cancelación distribuida escribe también `tpProformaPedido.IdcatEstadoProforma = Cancelada`. |
| **ProquifaDotNet.Timbrado** | — (no escribe) | Solo cancela el CFDI ante el SAT (RE-FU-023 endpoint `POST /api/v1/invoices/{invoiceId}/cancel`); no toca `tpPedido`. |

> **Consistencia entre aplicativos:** los cambios de estado desde Finanzas usan la misma guardia de idempotencia que el orquestador de RE-FU-023 (leer estado actual, comparar, escribir solo si aplica la transición permitida). No hay transacciones cross-aplicativo únicas — la consistencia se logra por orquestación.

---

## 7. Observaciones y decisiones pendientes del cliente

1. **Ámbito del campo `IdCatEstadoPedido`:** confirmar si aplica solo en `tpPedido`, también en `ppPedido`, o en ambas. Los estados 1–8 son propios de `ppPedido` (viven en ProquifaDotNet L04); los 9–21 son de `tpPedido` (viven repartidos entre ProquifaDotNet L05/L06–L09 y Finanzas). Alternativas:
   - **(A)** Un solo catálogo, agregar `IdCatEstadoPedido` a ambas tablas.
   - **(B)** Un catálogo por tabla — reutilizar `catEstadoPretramitacionPedido` para `ppPedido` (1–8) y crear `catEstadoPedido` solo para `tpPedido` (9–21).
2. **~~`INTRAMITABLE_GESTIONADO`~~ — RESUELTO (11/08/2026):** se separó en 4 estados con evidencia de código (`INTRAMITABLE_PENDIENTE_ACEPTACION`, `INTRAMITABLE_FEA_SOLICITADA`, `INTRAMITABLE_OC_ACEPTADA`, `INTRAMITABLE_TRAMITADO_CON_ERRORES` — ver secciones 2 y 3). Agrupar los 3 escenarios originales perdía información de negocio relevante y, además, 2 de los 4 estados reales **ni siquiera resuelven** el intramitable.
3. **Reutilización de `catSeguimientoPartidaPedido`:** los estados 14–18 usan claves ya publicadas al cliente en Venta Digital (`SeguimientoPedidoVD`). Conviene mantener las mismas claves para no romper esa integración; o bien dejar `catSeguimientoPartidaPedido` para el detalle por partida y `catEstadoPedido` para la cabecera con las mismas claves espejadas.
4. **`RECHAZADO_EN_INSPECCION` como transición de reproceso:** hoy en código el pedido puede volver a `EN_COMPRA` o `EN_ALMACEN_MATRIZ` según el motivo del rechazo. Confirmar reglas.
5. **`PREPAGO_CON_FAA` vs `EN_VALIDAR_COBRO`:** conceptualmente son dos etapas del mismo flujo (FAA emitida → cobro por aplicar). Confirmar si conviene mantenerlas separadas o consolidarlas en `EN_VALIDAR_COBRO` con un flag `EsPrepago`.
6. **`TRANSFERIDO_A_LEGACY` como estado formal:** hoy no hay flag ni catálogo que lo marque explícitamente; con R16 (LegacySync coordinado desde Finanzas) esta etapa gana visibilidad. Se propone como estado observable — el cliente debe confirmar si prefiere verlo como estado formal o como bandera derivada.
7. **Transición 18 → 21 (`ENTREGADO → FINALIZADO`):** confirmar quién dispara este cierre y en qué momento. Propuesta: Finanzas cierra el pedido cuando confirma que ambas cadenas (facturación completa + entrega física) se cumplieron.

---

## 8. Impacto DDL (una vez aprobado)

> **Nota:** el catálogo `dbo.catEstadoPedido` **ya existe** en BD `ProquifaDotNet` con columnas `IdCatEstadoPedido uniqueidentifier`, `EstadoPedido varchar(50)`, `Clave varchar(150)`, `Activo bit`, `FechaUltimaActualizacion datetime` (verificado en `ProquifaDotNet.sql` líneas 29278–29289). Está vacío y sin FK desde `tpPedido`. Se requiere: (a) extender con las columnas `Orden`, `EsTerminal` y `Aplicativo`; (b) sembrar los 21 estados; (c) agregar FK en `tpPedido`.

```sql
-- 1. Extender el catálogo existente
ALTER TABLE dbo.catEstadoPedido
    ADD Orden int NULL,
        EsTerminal bit NOT NULL CONSTRAINT DF_catEstadoPedido_EsTerminal DEFAULT (0),
        Aplicativo varchar(30) NULL; -- 'ProquifaDotNet' | 'Finanzas' | 'Distribuido'

ALTER TABLE dbo.catEstadoPedido
    ADD CONSTRAINT UQ_catEstadoPedido_Clave UNIQUE (Clave);

-- 2. FK en tpPedido
ALTER TABLE dbo.tpPedido
    ADD IdCatEstadoPedido uniqueidentifier NULL
        CONSTRAINT FK_tpPedido_catEstadoPedido
            FOREIGN KEY REFERENCES dbo.catEstadoPedido (IdCatEstadoPedido);

-- 3. Seed (21 estados) — asume que la tabla está vacía
INSERT INTO dbo.catEstadoPedido (Clave, EstadoPedido, Orden, EsTerminal, Aplicativo, FechaUltimaActualizacion, Activo) VALUES
 ('PEDIDO_GENERADO',                    'Pedido generado',                                                     1, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('EN_PRETRAMITAR',                     'En Pretramitar Pedido',                                               2, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('INTRAMITABLE',                       'Intramitable',                                                        3, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('INTRAMITABLE_PENDIENTE_ACEPTACION',  'Intramitable — correo de aceptación enviado, pendiente de respuesta', 4, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('INTRAMITABLE_FEA_SOLICITADA',        'Intramitable — solicitud de FEA (Fecha Estimada de Ajuste) enviada',  5, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('INTRAMITABLE_OC_ACEPTADA',           'Intramitable resuelto — OC Interna aceptada, pedido tramitado',       6, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('INTRAMITABLE_TRAMITADO_CON_ERRORES', 'Intramitable resuelto — tramitación con errores aceptada, pedido tramitado', 7, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('CANCELADO_EN_PRETRAMITAR',           'Cancelado en Pretramitar',                                            8, 1, 'ProquifaDotNet', GETDATE(), 1),
 ('TRAMITADO',                          'Tramitado (enviado a Compras)',                                       9, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('PREPAGO_CON_FAA',                    'Prepago con Factura por Adelantado',                                 10, 0, 'Finanzas', GETDATE(), 1),
 ('EN_VALIDAR_COBRO',                   'En Validar Cobro',                                                   11, 0, 'Finanzas', GETDATE(), 1),
 ('FACTURADO',                          'Facturado',                                                          12, 0, 'Finanzas', GETDATE(), 1),
 ('TRANSFERIDO_A_LEGACY',               'Transferido a Legacy',                                               13, 0, 'Finanzas', GETDATE(), 1),
 ('EN_COMPRA',                          'En Compra',                                                          14, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('EN_ALMACEN_MATRIZ',                  'En Almacén Matriz',                                                  15, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('RECHAZADO_EN_INSPECCION',            'Rechazado en Inspección',                                            16, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('EN_ENTREGA',                         'En Entrega',                                                         17, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('ENTREGADO',                          'Entregado',                                                          18, 0, 'ProquifaDotNet', GETDATE(), 1),
 ('CANCELADO_POR_FALTA_PAGO',           'Cancelado por falta de pago',                                        19, 1, 'Distribuido', GETDATE(), 1),
 ('CANCELADO',                          'Cancelado',                                                          20, 1, 'ProquifaDotNet', GETDATE(), 1),
 ('FINALIZADO',                         'Finalizado',                                                         21, 1, 'Finanzas', GETDATE(), 1);
```

---

## 9. Requisitos R16 que consumen `catEstadoPedido`

| Requisito | Aplicativo | Uso |
|---|---|---|
| RE-FU-013 / RE-FU-014 | ProquifaDotNet | Asignación al tramitar pedido normal (1 → 2 → 9); gestión de intramitables (3 → 4/5 → 6/7 → 9) |
| RE-FU-015 | ProquifaDotNet + Finanzas | Asignación al tramitar Prepago con FAA (9 → 10) |
| RE-FU-016 | Finanzas | Transición a `EN_VALIDAR_COBRO` cuando la proforma se emite con `MontoPendiente > 0` (9 → 11) |
| RE-FU-019 / RE-FU-020 | Finanzas | Emisión de FAA MEX / Perú (`PREPAGO_CON_FAA`) |
| RE-FU-023 | Finanzas (orquestador) + ProquifaDotNet + Timbrado | Cancelación por falta de pago (10/11 → 19), lectura de estado para el listado principal |
| RE-FU-024 – RE-FU-027 | Finanzas | Consumo del estado para wizard Validar Cobro; cierre del pendiente (11 → 12) al confirmar cobro + facturación |
| RE-FU-028 / RE-FU-029 | Finanzas (llama Timbrado) | Transición a `FACTURADO` al timbrar CFDI |
| RE-FU-030 | Finanzas (llama Timbrado) | Complemento de Pago timbrado dentro de `FACTURADO` |
| LegacySync | Finanzas | Transición a `TRANSFERIDO_A_LEGACY` post-ETL |

---

## 10. Referencias

**Requisitos R16:**
- `Requisitos/R16A-RE-FU-015/R16A-RE-FU-015_BD.md` — OBS-027 bloqueante.
- `Requisitos/R16A-RE-FU-015/R16A-RE-FU-015-Tareas.md` — T7 (BD) y T8 (Back).
- `Requisitos/R16A-RE-FU-016/*` — Proforma MEX en Finanzas.
- `Requisitos/R16A-RE-FU-023/[R16A-RE-FU-023][DIS-SOL] Diseño de la solución.md` — cancelación distribuida.
- `Requisitos/R16A-RE-FU-024/*` — Wizard Validar Cobro Paso 1.
- `Requisitos/R16A-RE-FU-028/R16A-RE-FU-028-Back.md` — ETLs Facturación E1–E8.
- `Requisitos/R16A-RE-FU-030/DIS-SOL-backend-complemento-pago.md` — Complemento de Pago.

**Análisis y arquitectura:**
- `Analisis/Curacion-Datos-R16.md` — fila 5, backfill masivo bloqueante.
- `Database/Base de datos nombramiento ProquifaDotNet.md` — patrón Scaffold EF Core desde Finanzas sobre `tpPedido`.
- `Database/ER-Finanzas.md` — modelo de datos Finanzas.

**Fuente en aplicativos:**
- `Database/ProquifaDotNet.sql` — DDL de `catEstadoPedido`, `catEstadoPartidaPedido`, `catEstadoPretramitacionPedido`, `catSeguimientoPartidaPedido`, `tpPedido`.
- `ProquifaDotNet-R14/Logic.Pqf.Logistica/L04.PretramitarPedido/` y `L05.TramitarPedido/` — flujos observados.
- `ProquifaDotNet-R14/WebApi.Logistica/ExternalAppSettings.config` — claves `SeguimientoPedidoVD`.
