# Endpoints — LegacyBridge (ETL/SSIS hacia Sistema Legado)

LegacyBridge no expone endpoints HTTP propios. Es el mecanismo de transferencia de datos **unidireccional** desde `ProquifaDotNet.Finanzas` y `ProquifaDotNet` hacia el sistema legado (PCconnect) mediante procesos SSIS / ETL.

> **Dirección del flujo:** ProquifaDotNet → Legacy (PCconnect)
> **Tecnología:** SSIS (SQL Server Integration Services)
> **Canal:** Pendiente de definir con arquitectura para flujos R16 (tabla ETL, cola RabbitMQ, o llamada API Legacy directa) — Brecha B3 de RE-028
> **Alcance:** Solo México. Perú no ejecuta transferencias a Legacy (Regla 11 de RE-028).

---

## Transferencias documentadas

Las transferencias se disparan automáticamente desde `ProquifaDotNet.Finanzas` al completar acciones clave en el wizard de Validar Cobro (Pasos 1–3).

### E1 — ETL Datos Buzón de Cobros

| Campo | Detalle |
|-------|---------|
| **Disparador** | Al confirmar cobro en Paso 1 de Validar Cobro |
| **Datos transferidos** | Datos del `fccFolioPagoCliente` (folio, monto, fecha, cliente) |
| **Dependencia** | RE-008 (Buzón de Cobros) |
| **Implementado en** | RE-028 |
| **Estado** | Pendiente definición canal de transferencia |

---

### E2 — ETL Datos Proforma

| Campo | Detalle |
|-------|---------|
| **Disparador** | Al tramitar pedido Prepago en ProquifaDotNet |
| **Datos transferidos** | Datos de `tpProformaPedido` (folio, cliente, empresa, montos, fecha) |
| **Dependencia** | RE-016 (Proforma) |
| **Implementado en** | RE-028 |
| **Estado** | Pendiente definición canal de transferencia |

---

### E3 — ETL Datos Factura

| Campo | Detalle |
|-------|---------|
| **Disparador** | Al enviar línea en Paso 3 de Validar Cobro (tipo `FACTURA` o `FACTURA_ANTICIPO`) |
| **Datos transferidos** | UUID, Serie, Folio, RFC Emisor/Receptor, Total, FechaEmision de `CFDIGenerada` |
| **Dependencia** | RE-019 (FAA México) |
| **Implementado en** | RE-028 |
| **Estado** | Pendiente definición canal de transferencia |

---

### E4 — ETL Datos Complemento de Pago

| Campo | Detalle |
|-------|---------|
| **Disparador** | Al enviar línea en Paso 3 de Validar Cobro (tipo `COMPLEMENTO_PAGO`) |
| **Datos transferidos** | UUID CP, UUID Factura relacionada, monto pago, parcialidad, impuestos, `fccDocumentoFiscalCobro` campos CP |
| **Dependencia** | RE-030 (Complemento de Pago) |
| **Implementado en** | RE-030 |
| **Estado** | Fuera de alcance RE-028; implementado en RE-030 |

---

### E5 — ETL Datos Nota de Crédito

| Campo | Detalle |
|-------|---------|
| **Disparador** | Al enviar línea Paso 3 cuando hay NCs aplicadas al cobro |
| **Datos transferidos** | UUID NC, UUID Factura origen, Serie, Folio, Monto, Motivo, Estado de `fccNotaCredito` |
| **Dependencia** | RE-032 (NC México) |
| **Implementado en** | RE-032 |
| **Estado** | Fuera de alcance RE-028; implementado en RE-032 |

---

### E6 — Transferencia PDF Factura

| Campo | Detalle |
|-------|---------|
| **Disparador** | Al enviar línea Paso 3 (tipo Factura o Factura Anticipo) |
| **Datos transferidos** | Archivo PDF de la factura timbrada (desde MinIO) |
| **Dependencia** | RE-021 (Persistencia PDF Factura México) |
| **Implementado en** | RE-028 |
| **Estado** | Pendiente definición canal de transferencia |

---

### E7 — Transferencia PDF Complemento de Pago

| Campo | Detalle |
|-------|---------|
| **Disparador** | Al enviar línea Paso 3 (tipo Complemento de Pago) |
| **Datos transferidos** | Archivo PDF del Complemento de Pago (desde MinIO) |
| **Dependencia** | RE-030 (Complemento de Pago) |
| **Implementado en** | RE-030 |
| **Estado** | Fuera de alcance RE-028; implementado en RE-030 |

---

### E8 — Transferencia PDF Nota de Crédito

| Campo | Detalle |
|-------|---------|
| **Disparador** | Al enviar línea Paso 3 cuando hay NCs aplicadas |
| **Datos transferidos** | Archivo PDF de la Nota de Crédito México (desde MinIO) |
| **Dependencia** | RE-032 (NC México) |
| **Implementado en** | RE-032 (documentado en RE-034) |
| **Estado** | Fuera de alcance RE-028; implementado en RE-032 |

---

## Resumen de transferencias

| ID | Transferencia | Disparador | Req. origen | Implementado en |
|----|---------------|-----------|-------------|-----------------|
| E1 | ETL Datos Buzón de Cobros | Confirmar cobro Paso 1 | RE-008 | RE-028 |
| E2 | ETL Datos Proforma | Tramitar pedido Prepago | RE-016 | RE-028 |
| E3 | ETL Datos Factura | Enviar línea Paso 3 (Factura/FAA) | RE-019 | RE-028 |
| E4 | ETL Datos Complemento de Pago | Enviar línea Paso 3 (CP) | RE-030 | RE-030 |
| E5 | ETL Datos Nota de Crédito | Enviar línea Paso 3 (con NC) | RE-032 | RE-032 |
| E6 | Transferencia PDF Factura | Enviar línea Paso 3 (Factura/FAA) | RE-021 | RE-028 |
| E7 | Transferencia PDF Complemento de Pago | Enviar línea Paso 3 (CP) | RE-030 | RE-030 |
| E8 | Transferencia PDF Nota de Crédito | Enviar línea Paso 3 (con NC) | RE-032 | RE-032 |

---

## Transferencias previas (flujos existentes en ProquifaDotNet)

Flujos ETL ya existentes que se mantienen sin modificaciones en R16:

| Flujo | Descripción |
|-------|-------------|
| Marcas (Datos Generales) | Sincronización de marcas al sistema legado |
| Proveedores (Datos Generales) | Sincronización de proveedores |
| Productos (Datos Generales, Familia, Oferta) | Sincronización de catálogo de productos |
| Clientes (Datos Generales, Direcciones, Contactos, Datos Legales) | Sincronización completa de clientes |
| Buzones (Cotización, Pedidos, Pagos + adjuntos) | Transferencia de buzones |
| Envío de Cotizaciones (Cotización, Partidas, Fletes, PDF) | Al enviar cotización |
| PSC — Pedido Sin Crédito | Genera Pendientes en Legacy; nuevo en R16: continúa flujo en ProquifaDotNet y transfiere Factura, NC, Pendientes, Pedidos, Partidas, Cobro y PDF |
| PCC — Pedido Con Crédito | Genera Pendientes en Legacy; transfiere Pedido y Partidas al tramitar |

---

## Brechas y pendientes

| Brecha | Descripción | Responsable |
|--------|-------------|-------------|
| B3 (RE-028) | Canal de transferencia para E1–E3 y E6 sin definir (tabla ETL, RabbitMQ o API Legacy directa) | Arquitectura / TechLead |
| PCconnect estructura | Mapeo de tablas destino en PCconnect pendiente de documentación (SSIS) | Equipo Legacy |
