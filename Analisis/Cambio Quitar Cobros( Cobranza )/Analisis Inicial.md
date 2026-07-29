# Análisis — Cambio de Alcance: Quitar Cobros (Cobranza) de R16

| Campo | Valor |
|---|---|
| **Fecha** | 2026-07-29 |
| **Tipo** | Análisis de impacto de cambio de alcance |
| **Estado** | Inicial |

---

## Contexto

Se decide quitar el módulo de Cobros (Cobranza) del alcance del proyecto R16.

---

## Scope resultante

### Requisitos que permanecen

| Requisito | Nombre                                                 | Observación                                                                                                                                                                                                                      |
| --------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RE-001    | Mantenimiento de catálogos del sistema                 | Sin cambios                                                                                                                                                                                                                      |
| RE-002    | Mantenimiento de Catálogo de Clientes                  | Sin cambios                                                                                                                                                                                                                      |
| RE-003    | Catálogo de Clientes — Documentos Regulatorios         | Sin cambios                                                                                                                                                                                                                      |
| RE-005    | Configuración de Cobros y Facturación                  | Solo la parte de **configuración de facturación**; la configuración de cobros pierde propósito                                                                                                                                   |
| RE-006    | Referencia de Pago y Código Validador                  | Se mantiene como dato informativo en la Proforma — ver nota                                                                                                                                                                      |
| RE-008    | Mailbot v2                                             | Cambio de alcance - No se va a generar el buzón de cobros, pero si la solución y la nueva funcionalidad sincronizar los buzones Cotizaciones y Pedidos<br>Buzones (Gmail Push Notifications + MailBot Worker + clasificación IA) |
| RE-009    | Validación Regulatoria                                 | Sin cambios                                                                                                                                                                                                                      |
| RE-010    | Tramitación de Pedidos Crédito                         | Sin cambios                                                                                                                                                                                                                      |
| RE-011    | Tramitación de Pedidos Crédito con Controlados         | Sin cambios                                                                                                                                                                                                                      |
| RE-012    | Tramitación de Pedidos Crédito con FAA                 | Sin cambios                                                                                                                                                                                                                      |
| RE-013    | Tramitación de Pedidos Prepago con Controlados         | Sin cambios                                                                                                                                                                                                                      |
| RE-014    | Tramitación de Pedidos Prepago sin Controlados sin FAA | Sin cambios                                                                                                                                                                                                                      |
| RE-015    | Tramitación de Pedidos Prepago con FAA                 | Sin cambios                                                                                                                                                                                                                      |
| RE-016    | Proforma México                                        | Sin cambios                                                                                                                                                                                                                      |
| RE-017    | Proforma Perú                                          | Sin cambios                                                                                                                                                                                                                      |
| RE-018    | Factura por Adelantado — Listado                       | Sin cambios                                                                                                                                                                                                                      |
| RE-019    | Factura por Adelantado — Detalle México                | **Ampliado**: incluye registro de pago recibido y trigger de CDP                                                                                                                                                                 |
| RE-020    | Factura por Adelantado — Detalle Perú                  | **Ampliado**: análogo a RE-019 para Perú                                                                                                                                                                                         |
| RE-021    | Documento Factura México                               | Sin cambios                                                                                                                                                                                                                      |
| RE-022    | Documento Factura Perú                                 | Sin cambios                                                                                                                                                                                                                      |
| RE-030    | CDP México                                             | Permanece — trigger movido a RE-019 (ver sección CDP)                                                                                                                                                                            |

### Cancelación de Factura

La cancelación de CFDI ante el SAT **no vive en Finanzas ni dentro del requisito de Facturar**. El endpoint de cancelación se define dentro de **`ProquifaDotNet.Timbrado`**. Finanzas únicamente actualiza el estado de `CFDIGenerada.Estado` y registra el acuse en `CFDICancelacion` cuando Timbrado notifica el resultado.

### Requisitos eliminados

| Requisito | Nombre                                                        |
| --------- | ------------------------------------------------------------- |
| RE-023    | Validar Cobro — módulo principal (incluye Gestionar Cobranza) |
| RE-024    | Validar Cobro Paso 1 México                                   |
| RE-025    | Validar Cobro Paso 1 Perú                                     |
| RE-026    | Validar Cobro Paso 2 México                                   |
| RE-027    | Validar Cobro Paso 2 Perú                                     |
| RE-028    | Validar Cobro Paso 3 México                                   |
| RE-029    | Validar Cobro Paso 3 Perú                                     |
| RE-032    | Notas de Crédito México                                       |
| RE-033    | Notas de Crédito Perú                                         |
| RE-034    | Documento NDC México                                          |
| RE-035    | Documento NDC Perú                                            |

**Total eliminados: 12 requisitos.**

---

## Impacto en el CDP (Complemento de Pago)

En el diseño original el CDP se generaba desde **Validar Cobro Paso 3 (RE-028)**, con el cobro capturado en el Paso 1 como origen. Al eliminar Validar Cobro, el trigger del CDP queda sin hogar.

**Decisión:** el registro del pago y la generación del CDP se incorporan dentro del **requisito RE-019 (Factura por Adelantado — Detalle México)**. Esto implica que RE-019 amplía su alcance para incluir:

- Registro simplificado del pago recibido (monto, fecha, forma de pago SAT, parcialidad, tipo de cambio si aplica).
- Asociación del pago a la FAA correspondiente (`fccPagoFacturaAdelanto`).
- Actualización del estado de `fccFactura` a `PAGADA_PARCIAL` / `PAGADA`.
- Trigger de generación y timbrado del CDP vía `ProquifaDotNet.Timbrado`.

---

## Impacto en la solución técnica

### `ProquifaDotNet.Finanzas` — alcance reducido

| Responsabilidad | Estado |
|---|---|
| FAA (generación, foliado, timbrado) | ✅ Se mantiene |
| Factura final | ✅ Se mantiene |
| CDP (trigger desde RE-019, pago mínimo) | ✅ Se mantiene |
| Cancelación (recibe notificación de Timbrado) | ✅ Se mantiene |
| Wizard Validar Cobro (3 pasos) | ❌ Eliminado |
| Notas de Crédito | ❌ Eliminadas |
| Gestionar Cobranza | ❌ Eliminado |

### `ProquifaDotNet.Timbrado`

| Responsabilidad                              | Estado        |
| -------------------------------------------- | ------------- |
| Timbrado FAA                                 | ✅ Sin cambios |
| Timbrado CDP                                 | ✅ Sin cambios |
| **Endpoint de cancelación de CFDI ante SAT** | ✅             |

### `proquifa.pqf2.MailBot` — alcance reducido, no eliminado

El Worker .NET 10 con Gmail Push Notifications y clasificación por Agente IA **permanece en el ecosistema**. Su implementación se realiza dentro del requisito RE-008.

| Buzón               | Estado                                                  |
| ------------------- | ------------------------------------------------------- |
| Buzón de Cotización | ✅ Se mantiene — implementado en RE-008                  |
| Buzón de Pedidos    | ✅ Se mantiene — implementado en RE-008                  |
| Buzón de Cobros     | ❌ Eliminado — ya no hay flujo de Validar Cobros en PQF2 |

Lo que desaparece de MailBot es únicamente la clasificación y el procesamiento de correos de cobro (`fccFolioPagoCliente`). El Worker en sí continúa operando para los buzones de Cotización y Pedido.

---

## Tablas de BD — impacto

### Tablas que permanecen en `ProquifaDotNet.Finanzas`

| Tabla                          | Rol                                                                 |
| ------------------------------ | ------------------------------------------------------------------- |
| `EmpresaFolio`                 | Foliador atómico (FAA, CDP)                                         |
| `CFDIGenerada`                 | Registro central de todo CFDI timbrado                              |
| `CFDIGeneradaConcepto`         | Conceptos del CFDI                                                  |
| `CFDIGeneradaRelacionado`      | Nodo CFDIRelacionados                                               |
| `CFDICancelacion`              | Acuse de cancelación ante SAT                                       |
| `catTipoCFDI`                  | Clasifica el tipo de comprobante                                    |
| `catFacturaEstado`             | Ciclo de vida de la FAA                                             |
| `catFormaPagoSAT`              | Catálogo c_FormaPago SAT (para CDP)                                 |
| `catMotivoCancelacionSAT`      | Motivos de cancelación 01-04 (consumido por Timbrado)               |
| `fccFactura`                   | Cabecera FAA / factura final                                        |
| `fccFacturaPartida`            | Partidas snapshot de la FAA                                         |
| `fccFacturaReferenciaBancaria` | Cuentas MN/DLS + referencia del cliente                             |
| `fccPagoFacturaAdelanto`       | Vínculo pago registrado ↔ FAA (trigger CDP)                         |
| `fccDocumentoFiscalCobro`      | Snapshot del CDP antes de timbrar                                   |
| `fccPagoCliente`               | Posible tabla para comunicar tabla con aplicativo Externo de cobros |

### Tablas que desaparecen

| Tabla                           | Motivo                                     |
| ------------------------------- | ------------------------------------------ |
| `fccFolioPagoCliente`           | Exclusiva del Buzón de Cobros              |
| `fccPagoFacturaPedido`          | Asociación cobro ↔ proforma de crédito     |
| `fccSaldoFavorCliente`          | Sin validación de cobro                    |
| `fccInconsistenciaCobro`        | Sin Paso 1 de Validar Cobro                |
| `catCobroEstatus`               | Ciclo de vida del cobro                    |
| `catTipoInconsistenciaCobro`    | Sin inconsistencias de cobro               |
| `fccFechaEstimadaPagoHistorial` | Sin Gestionar Cobranza                     |
| `fccConfirmacionPedido`         | Exclusiva de Validar Cobro Paso 3          |
| `fccNotaCredito`                | Sin módulo de Notas de Crédito             |
| `fccNotaCreditoPartida`         | Sin módulo de Notas de Crédito             |
| `catMotivoCreditoSUNAT09`       | Exclusivo de NC Perú                       |

---

## Nota — RE-006 Código Validador

El propósito original del Código Validador es identificar pagos automáticamente en el Buzón de Cobros. Al eliminarse el Buzón, este propósito desaparece.

Sin embargo, la **referencia bancaria construida con el Código Validador sigue apareciendo en la Proforma** (RE-016 Criterio E2) como dato informativo para que el cliente sepa a qué cuenta depositar. Por eso RE-006 permanece en el scope, aunque con propósito reducido a dato informativo del documento.

**Pendiente de confirmar:** si el campo REF. CLIENTE de la Proforma se mantiene o se elimina dado que ya no hay identificación automática del pago en ProquifaDotNet.

## Diagramas

### Flujo Actual

![Flujo entradas y salidas](/Analisis/Cambio%20Quitar%20Cobros(%20Cobranza%20)/Imagenes/flujo_entradas_salidas_cobros.png)

### Componentes o Módulos que se quitan
![[flujo_cobros_whiteboard_digital.png]]

### Flujo nuevo con Aplicativo Externo de Cobranza
![[flujo_cobros_aplicativo_externo.png]]

---

## AplicativoExternoCobranza — Integración con ProquifaDotNet

### Contexto

Al eliminar el módulo de Validar Cobros de R16, el flujo de cobranza (captura de pago, asociación con documentos y seguimiento) pasa a ser responsabilidad de un **aplicativo externo aún no definido**, denominado provisionalmente **AplicativoExternoCobranza**. Este aplicativo ya contaría con todo el flujo de cobranza implementado.

ProquifaDotNet.Finanzas **no gestiona cobros**. Su punto de entrada es el registro del pago ya procesado, que se materializa en la tabla `fccPagoCliente`. A partir de ahí, Finanzas continúa de forma autónoma el flujo de facturación.

---

### Tabla pivote — `fccPagoCliente`

`fccPagoCliente` es el contrato entre AplicativoExternoCobranza y ProquifaDotNet.Finanzas. Independientemente de la opción de integración que se elija, el resultado debe ser un registro en esta tabla con los datos mínimos necesarios para disparar la facturación:

| Campo                         | Descripción                                       |
| ----------------------------- | ------------------------------------------------- |
| `Monto`                       | Importe del pago recibido                         |
| `Fecha`                       | Fecha valor del pago                              |
| `FormaPago`                   | Clave SAT (c_FormaPago)                           |
| `Parcialidad`                 | Número de parcialidad (para CDP)                  |
| `TipoCambio`                  | Tipo de cambio si el pago es en moneda extranjera |
| `IdFccFactura` / `IdProforma` | Referencia al documento que se liquida            |

---

### Opciones de integración

Se identificaron tres mecanismos posibles para que AplicativoExternoCobranza transfiera los datos de pago a ProquifaDotNet.Finanzas. La elección depende de las capacidades de integración que exponga el aplicativo externo.

#### Opción A — AplicativoExternoCobranza llama la API de PQF2 *(preferida)*

AplicativoExternoCobranza, una vez que confirma el cobro, realiza una llamada HTTP al endpoint de ProquifaDotNet.Finanzas. El endpoint recibe los datos del pago, crea el registro en `fccPagoCliente` y dispara el flujo de facturación (CDP / actualización FAA).

**Ventajas:** desacoplamiento total, control de errores en PQF2, auditoría centralizada, sin dependencia de horarios.  
**Requisito:** AplicativoExternoCobranza debe poder hacer llamadas salientes a la API de PQF2 y autenticarse con IdentityServer.

#### Opción B — Worker PQF2 monitorea AplicativoExternoCobranza *(alternativa)*

Se crea un **Worker en ProquifaDotNet.Finanzas** (o en un servicio dedicado) que consulta periódicamente AplicativoExternoCobranza — ya sea por API o por una vista/tabla expuesta — para detectar cobros confirmados que aún no estén registrados en `fccPagoCliente`. Al detectar uno nuevo, lo importa y dispara el flujo.

**Ventajas:** no requiere que AplicativoExternoCobranza haga llamadas salientes.  
**Desventajas:** latencia según frecuencia del polling, mayor complejidad operativa, riesgo de duplicados si no se maneja idempotencia.

#### Opción C — PQF2 lee directamente la BD de AplicativoExternoCobranza *(menos probable)*

ProquifaDotNet.Finanzas se conecta mediante Scaffold (EF Core) o consulta SQL directa a la base de datos de AplicativoExternoCobranza para leer los cobros confirmados.

**Desventajas:** alto acoplamiento, dependencia del esquema interno del aplicativo externo, problemas de seguridad y gobernanza de datos. Se descarta salvo que sea la única opción técnicamente viable.

---

### Vínculo con los flujos documentados

| Flujo                         | Relación con AplicativoExternoCobranza                                                                                                                                                                            |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **AS-IS — Validar Cobros**    | El módulo completo (Pasos 1-3 + Gestionar Cobranza + Auxiliar Contable) es reemplazado por AplicativoExternoCobranza. Todo lo tachado en el diagrama AS-IS pasa a ser responsabilidad del aplicativo externo.     |
| **TO-BE — Facturación**       | ProquifaDotNet.Finanzas recibe el pago vía `fccPagoCliente` (resultado de cualquiera de las 3 opciones) y continúa el flujo: actualiza estado de FAA, genera CDP y lo envía a timbrar en ProquifaDotNet.Timbrado. |
| **RE-019 FAA Detalle México** | El registro simplificado de pago (trigger del CDP) que se amplió en RE-019 asume que `fccPagoCliente` ya existe — llenado por AplicativoExternoCobranza. RE-019 solo consume ese dato; no lo captura.             |
| **RE-030 CDP México**         | Sin cambios en su lógica. El trigger sigue siendo la existencia de un pago registrado en `fccPagoCliente` asociado a una FAA.                                                                                     |

---

### Pendientes de definición

- Identificar qué aplicativo es AplicativoExternoCobranza y si ya existe en el ecosistema Proquifa o es un tercero.
- Confirmar qué opciones de integración expone (API, BD, eventos).
- Definir contrato de datos mínimos para `fccPagoCliente` en conjunto con el equipo del aplicativo externo.
- Determinar si PQF2 necesita exponer un nuevo endpoint de recepción de cobros (Opción A) o si el Worker es suficiente (Opción B).
- Evaluar si RE-006 (Código Validador / Referencia Bancaria) sigue siendo relevante como dato informativo en la Proforma una vez que AplicativoExternoCobranza gestiona la identificación del pago.