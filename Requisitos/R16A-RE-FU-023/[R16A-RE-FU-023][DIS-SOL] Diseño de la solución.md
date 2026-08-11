# 

# 

# 

# 

# 

# 

# **R16A-RE-FU-023\]\[DIS-SOL\] Diseño de la solución**

## Diseño de la solución

| FORMATO | N/A |
| :---- | :---- |
| **PROYECTO** | R16 \- Adquisiciones  |
| **REFERENCIA** | N/A |
| **VERSIÓN** | 1.0 |
| **FECHA** | 20/07/2026 |
| **AUTOR** | Osmar Calderón Vázquez |
| **REVISOR** | Valdemar Farina Sánchez |

## 

## Contenido

[**Importante	3**](#heading=)

[**Introducción	3**](#heading=)

[1\. Propósito del documento	3](#heading=)

[2\. Alcance	3](#heading=)

[Específicamente incluye:	3](#heading=)

[No se consideran:	3](#heading=)

[**Visión general del diseño	4**](#heading=)

[1\. Objetivo técnico	4](#heading=)

[2\. Componentes involucrados	4](#heading=)

[**Diseño funcional detallado	5**](#heading=)

[1\. Flujo técnico principal	5](#heading=)

[1.1 Listado de la pantalla principal	5](#heading=)

[1.2 Acción contextual	6](#heading=)

[1.3 Modal Gestionar Cobranza	6](#heading=)

[1.4 Cancelación de pedido por falta de pago (distribuida)	6](#heading=)

[2\. Criterios de aceptación del requisito	8](#heading=)

[3\. Reglas técnicas aplicadas	9](#heading=)

[4\. Endpoints nuevos/modificados	10](#heading=)

[**Diseño de componentes	11**](#heading=)

[1\. Responsabilidades por componente	11](#heading=)

[2\. Diagramas	12](#heading=)

[Diagrama 1 — Listado de la pantalla principal	12](#heading=)

[Diagrama 2 — Cancelación Caso A (sin CFDI)	13](#heading=)

[Diagrama 3 — Cancelación Caso B (con CFDI, Conducta 2\)	14](#heading=)

[**Diseño de Modelo de Datos	15**](#heading=)

[1\. Nuevas tablas	15](#heading=)

[fccFechaEstimadaPagoHistorial (OBS-044)	15](#heading=)

[2\. Tablas modificadas	15](#heading=)

[\~\~fccPagoCliente (×5)\~\~ — RETIRADO (absorbido por la fusión de RE-FU-024)	15](#heading=)

[tpPedido (cancelación)	16](#heading=)

[3\. Tablas eliminadas	16](#heading=)

[4\. Relaciones	16](#heading=)

[5\. Reglas de integridad	16](#heading=)

[**Decisiones Tomadas	17**](#heading=)

[**Impacto Técnico	17**](#heading=)

[1\. Impacto en código existente	17](#heading=)

[2\. Impacto en modelos	18](#heading=)

[3\. Impacto en despliegue	18](#heading=)

[**Manejo de Errores y Excepciones	18**](#heading=)

[**Estrategia de Pruebas (Diseño de las pruebas)	19**](#heading=)

[1\. Pruebas funcionales (Criterios de Aceptación en DEV)	19](#heading=)

[2\. Pruebas técnicas (unitarias e integración)	19](#heading=)

[3\. Casos críticos	19](#heading=)

[**Control de versiones	19**](#heading=)

# **Importante**

Preguntas rápidas para validar que el diseño está completo:

* ¿El programador sabe qué flujo implementar? → Sí: listado por cartera/región, acción contextual, modal Gestionar Cobranza y cancelación distribuida.  
* ¿Sabe qué pasa si algo falla? → Sí: sección *Manejo de Errores* (orquestación \+ idempotencia, sin transacción atómica única).  
* ¿Sabe qué reglas no puede romper? → Sí: sección *Reglas técnicas aplicadas*.  
* ¿Sabe qué pruebas debe pasar? → Sí: sección *Estrategia de Pruebas*.  
* ¿Sabe dónde impacta? → Sí: sección *Impacto Técnico*.

# **Introducción**

## **1\. Propósito del documento**

Definir el diseño de la solución técnica **back-end** para la pantalla principal del módulo **Validar Cobro**: el listado de clientes de la cartera del usuario (cobrador) con pendientes de cobro, la acción contextual por cliente, el modal **Gestionar Cobranza** (fecha estimada de pago \+ cancelación de pedido) y la **cancelación de pedido por falta de pago** orquestada de forma distribuida entre tres aplicativos.

## **2\. Alcance**

### **Específicamente incluye:**

* Endpoint de **listado de clientes** con conteos y saldo (patrón `QueryInfo` → `QueryResultDto`), filtrado por cartera y región resueltas del token.  
* Cálculo de campos derivados por cliente: cobros recibidos pendientes, proformas/facturas por cobrar, saldo total en USD, acción contextual, SLA.  
* Endpoints del **modal Gestionar Cobranza**: listado de pedidos con saldo, actualización de **fecha estimada de pago** (con historial) y **cancelación de pedido por falta de pago**.  
* Diseño de la **cancelación distribuida** (orquestada desde Finanzas): Caso A (sin CFDI) y Caso B (con CFDI, Conducta 2 \+ polling asíncrono).  
* Cambios de BD: `ALTER` de `tpPedido` y `CREATE` de `fccFechaEstimadaPagoHistorial`. **El \`ALTER fccPagoCliente\` se retira** — esa tabla se elimina en la fusión de RE-FU-024 y sus columnas nacen en el `CREATE TABLE fccCobroCliente`.

### **No se consideran:**

* El **wizard "Realizar Cobros"** (captura/confirmación del cobro) — pertenece a **RE-FU-024** y siguientes.  
* La generación del pendiente del Buzón, que **ya existe** en el Mailbot (RE-FU-008) y no se duplica. *(Con la fusión de RE-024 el Mailbot deja de escribir \`fccFolioPagoCliente\` y pasa a insertar \`fccCobroCliente\` con estatus \`BORRADOR\` — cambio propiedad de RE-008.)*  
* **El \`CREATE TABLE fccCobroCliente\` y el \`DROP\` de las dos tablas anteriores** — son propiedad de **RE-FU-024**; aquí solo se consumen.  
* La definición del catálogo `CatEstadoTpPedido` (⛔ OBS-027) y la representación de estado de la proforma (`catEstadoProforma`), que **son propiedad de RE-016**.  
* La reversión en Legacy (aún sin datos; a revisar cuando Legacy tenga datos).

# **Visión general del diseño**

## **1\. Objetivo técnico**

Exponer, desde **ProquifaDotNet.Finanzas** (.NET Core 10, EF Core Scaffold \+ CQRS), los servicios de lectura y acción que alimentan la pantalla principal de Validar Cobro, reutilizando el patrón Punchout (`QueryInfo`/`QueryResultDto`) del proyecto. La **cancelación de pedido por falta de pago** se implementa como una operación **distribuida y orquestada desde Finanzas**, con consistencia por orquestación e idempotencia (no transacción atómica única), repartida en Finanzas (proforma), ProquifaDotNet (`tpPedido`) y ProquifaDotNet.Timbrado (CFDI ante el SAT).

## **2\. Componentes involucrados**

| Componente | Responsabilidad | Ubicación |
| :---- | :---- | :---- |
| `PaymentValidationClientService` / `...Repository` | Listado de clientes de la cartera con conteos y saldo (patrón `QueryInfo`) | ProquifaDotNet.Finanzas (Application/Infrastructure) |
| CQRS Queries/Commands | `GetPendingPaymentFoliosQuery`, `ClosePaymentFolioCommand`, `GetClientPaymentQuery`, `UpsertClientPaymentCommand`, `GetClientProformaQuery`, `UpdateEstimatedPaymentDateCommand`, `CancelOrderForNonPaymentCommand` (orquestador) | ProquifaDotNet.Finanzas (Application) |
| Scaffold EF Core | **\`fccCobroCliente\`** (entidad única tras la fusión de RE-024 — reemplaza a `fccFolioPagoCliente` \+ `fccPagoCliente`), `catCobroEstatus`, `tpProformaPedido` (R/W), `tpPedido` (**solo lectura**) | ProquifaDotNet.Finanzas (Infrastructure) |
| Endpoint interno de cancelación de pedido | Escribe trazabilidad de cancelación en `tpPedido` | ProquifaDotNet (WebApi.Logistica) |
| Endpoint interno de cancelación de CFDI | Cancela el CFDI ante el SAT vía PAC (TurboPac) | ProquifaDotNet.Timbrado |
| `ConversorDivisas` (existente) | Dolariza el saldo pendiente (OBS-046) | ProquifaDotNet (reutilizado) |
| BD (SQL Server) | Persistencia; `ALTER`/`CREATE` de este requisito | Base de datos `ProquifaDotNet` |

# **Diseño funcional detallado**

## **1\. Flujo técnico principal**

### **1.1 Listado de la pantalla principal**

1. El usuario entra al módulo → el Front llama `POST /api/v1/validate-collection/client/search` con `QueryInfo` (único filtro: `textoBusqueda`).  
2. Finanzas resuelve **región** y **cartera (Cobrador)** desde el token (IdentityServer) — no como filtro explícito del body.  
3. Por cada cliente con al menos un pendiente, calcula:  
   * `PendingPaymentsReceived` \= COUNT `fccCobroCliente` con `IdCatCobroEstatus NOT IN (COMPLETADO, CANCELADO)` y `Activo = 1`. ⚠️ **El filtro por estatus es obligatorio:** tras la fusión de RE-024, los cobros ya capturados viven en la **misma tabla** que los sin capturar, así que un `COUNT` sin él inflaría el indicador.  
   * `PendingProformaInvoices` \= COUNT `tpProformaPedido` con `MontoPendiente > 0` y `IdcatEstadoProforma <> Cancelada`.  
   * `TotalPendingBalance` \= SUM `tpProformaPedido.MontoPendiente` dolarizado a USD (`ConversorDivisas`, OBS-046).  
   * `ContextualAction` \= `PROCESS_PAYMENTS` si `PendingPaymentsReceived > 0`; si no, `MANAGE_COLLECTIONS`.  
   * `OldestPendingPayment` \= MIN `fccCobroCliente.FechaRecepcion` con el **mismo filtro** (`IdCatCobroEstatus NOT IN (COMPLETADO, CANCELADO)`, `Activo=1`); sin él traería la fecha de un cobro ya cerrado.  
   * `HasSlaExpired` \= `true` si el cobro más antiguo lleva \> 72 h sin procesar.  
4. Orden por antigüedad del cobro recibido más antiguo (OBS-047, ASC); clientes sin cobros al final.  
5. El buscador aplica **trim automático** (OBS-041) antes de filtrar por nombre o identificador fiscal (RFC MEX / RUC PER).

### **1.2 Acción contextual**

* `PROCESS_PAYMENTS` → el Front navega al **wizard de 3 pasos** (pantalla nueva, RE-024+). Fuera de alcance de este documento.  
* `MANAGE_COLLECTIONS` → el Front abre el **modal Gestionar Cobranza** (sección 1.3).

### **1.3 Modal Gestionar Cobranza**

1. `GET /api/v1/validate-collection/proforma?clientId={id}` → lista los pedidos del cliente con `MontoPendiente > 0` y `IdcatEstadoProforma <> Cancelada`.  
2. **Editar fecha estimada de pago** por pedido → al Confirmar: `PUT /api/v1/validate-collection/proforma/estimated-payment-date`:  
   * `UPDATE tpProformaPedido.FechaPromesaPagoMonitoreoCobros`.  
   * `INSERT fccFechaEstimadaPagoHistorial` (append-only, OBS-044).  
3. **Cancelar Pedido** (acción inmediata) → `PUT /api/v1/validate-collection/orders/{orderId}/cancel-non-payment` → cancelación distribuida (sección 1.4).

### **1.4 Cancelación de pedido por falta de pago (distribuida)**

La cancelación se **orquesta desde Finanzas** y reparte las escrituras en tres aplicativos. **No** es una transacción de BD única: la consistencia se logra por **orquestación \+ idempotencia**. La **visibilidad del pedido en el listado depende solo de** `tpProformaPedido.IdcatEstadoProforma`, no de `tpPedido` ni del CFDI — esto es la base del manejo de fallos.

| Servicio                    | Escribe                                   | Lee                           | Endpoint                                                              |
| :-------------------------- | :---------------------------------------- | :---------------------------- | :-------------------------------------------------------------------- |
| **ProquifaDotNet.Finanzas** | `tpProformaPedido` (proforma) \+ orquesta | `tpPedido` (**solo lectura**) | `PUT /api/v1/validate-collection/orders/{orderId}/cancel-non-payment` |
| **ProquifaDotNet**          | `tpPedido` (trazabilidad de cancelación)  | —                             | `PUT /api/v1/orders/{orderId}/cancel-non-payment`                     |
| **ProquifaDotNet.Timbrado** | CFDI / SAT                                | —                             | `POST /api/v1/invoices/{invoiceId}/cancel`                            |

**1.4.1 Caso A — pedido SIN CFDI timbrado**  
Orden **remote-first**: la escritura remota idempotente (`tpPedido`) va primero; el commit local de Finanzas (proforma) es el último y el que decide.

1. Finanzas → ProquifaDotNet `PUT .../orders/{orderId}/cancel-non-payment`:  
   * ProquifaDotNet **idempotente**: si `tpPedido` ya está cancelado, no-op y responde OK; si no, `UPDATE tpPedido SET FechaCancelacionPorFaltaPago, IdUsuarioCancelacion`.  
   * Si la llamada falla → Finanzas aborta, mensaje "no se pudo cancelar, reintente" (proforma intacta).  
2. Finanzas `BEGIN TRAN` → `UPDATE tpProformaPedido SET IdcatEstadoProforma = Cancelada` → `COMMIT`.  
   * Si el commit falla → `ROLLBACK` \+ mensaje de reintento (el pedido **sigue** en el listado porque el estado de la proforma no quedó en Cancelada).  
3. Commit OK → el pedido sale del listado.

**1.4.2 Caso B — pedido CON CFDI timbrado (Conducta 2\)**  
Añade la cancelación del CFDI ante el SAT (externa, casi irreversible y posiblemente **asíncrona** — CFDI 4.0). **Conducta 2:** el pedido **desaparece de inmediato**; el desenlace del SAT se resuelve en segundo plano. Si el SAT **rechaza**, **no hay reapertura automática**: se **escala a soporte** por correo.  
Decisiones fijadas:

* **\`ClaveMotivo \= 03\`** (No se llevó a cabo la operación) — a confirmar con área fiscal.  
* **Orden CFDI-primero:** se inicia la cancelación del CFDI antes de tocar lo interno, para abortar limpio si el SAT rechaza de forma síncrona.  
* **Polling con Hangfire en Finanzas:** Finanzas orquesta el sondeo; Timbrado consulta al PAC; el correo a soporte lo envía Finanzas.  
1. Finanzas → Timbrado `POST /api/v1/invoices/{invoiceId}/cancel { claveMotivo = 03 }`:  
   * Timbrado **idempotente** (revisa `CFDICancelacion`): arma `requestCancelacion` firmado → `TurboPac.CancelaCfdi`; INSERT/UPDATE `CFDICancelacion`; responde `Cancelado | EnProceso | Rechazado(síncrono)`.  
   * Rechazado síncrono / error → Finanzas **aborta**, avisa al usuario, **no** toca lo interno.  
   * Cancelado o EnProceso → continúa.  
2. **Interno** (= mecanismo del Caso A): cancela `tpPedido` (idempotente) \+ `UPDATE tpProformaPedido.IdcatEstadoProforma = Cancelada` (commit). → El pedido **sale del listado de inmediato**.  
3. **Asíncrono** — Finanzas programa un job de Hangfire que consulta el estatus:  
   * `EnProceso` → re-programa el siguiente intento.  
   * `Cancelado` → fin (`CFDICancelacion.Estatus = 'Cancelado'`).  
   * `Rechazado` → Finanzas **envía correo a soporte** de ProquifaDotNet (atención manual); el pedido queda cancelado, sin reapertura automática.

## 

## **2\. Criterios de aceptación del requisito**

*Mapeados a las Reglas de Negocio (R1–R10) y decisiones (OBS-041/042/044/046/047) del contexto RE-023.*

| CA   | Descripción                                                                                            | Estado   | Justificación                                                          |
| :--- | :----------------------------------------------------------------------------------------------------- | :------- | :--------------------------------------------------------------------- |
| CA-1 | El listado muestra solo clientes de la cartera y región del usuario, con al menos un pendiente (R1–R3) | Cubierto | Región/cartera del token; filtro por conteos \> 0                      |
| CA-2 | Acción contextual `PROCESS_PAYMENTS` / `MANAGE_COLLECTIONS` según cobros recibidos (R4)                | Cubierto | Campo `ContextualAction` calculado                                     |
| CA-3 | Buscador por nombre o identificador fiscal con trim (R6, OBS-041)                                      | Cubierto | `textoBusqueda` con trim en `ApplyFilter`                              |
| CA-4 | Modal Gestionar Cobranza lista pedidos con saldo y permite editar fecha estimada (R7–R8)               | Cubierto | `GetClientProformaQuery` \+ `UpdateEstimatedPaymentDateCommand`        |
| CA-5 | La fecha estimada registra historial completo (OBS-044)                                                | Cubierto | `fccFechaEstimadaPagoHistorial` append-only                            |
| CA-6 | Cancelar Pedido saca el pedido del listado y registra trazabilidad (R9, OBS-042)                       | Cubierto | Cancelación distribuida \+ `tpPedido` cancelación \+ `CFDICancelacion` |
| CA-7 | Saldo pendiente siempre en USD (OBS-046)                                                               | Cubierto | `ConversorDivisas`                                                     |
| CA-8 | Orden por antigüedad del cobro más antiguo \+ indicador SLA 72h (OBS-047)                              | Cubierto | `OldestPendingPayment` / `HasSlaExpired`                               |

## **3\. Reglas técnicas aplicadas**

| Regla | Descripción |
| :---- | :---- |
| RT-01 | Región y cartera se resuelven **del token** (IdentityServer), nunca como filtro del body |
| RT-02 | El listado usa el patrón `QueryInfo` (`[HttpPost]` \+ `[FromBody] QueryInfo`) → `QueryResultDto<PaymentValidationClientDto>` |
| RT-03 | El saldo se muestra **siempre en USD** vía `ConversorDivisas` (OBS-046), sin importar la región |
| RT-04 | La visibilidad del pedido depende **solo** de `tpProformaPedido.IdcatEstadoProforma` |
| RT-05 | La cancelación es **distribuida** (orquestación \+ idempotencia), no transacción atómica única |
| RT-06 | Todos los endpoints internos de cancelación (`tpPedido`, CFDI) son **idempotentes** |
| RT-07 | El INSERT del pendiente del Buzón **no se duplica** — ya existe en Mailbot (con la fusión de RE-024 inserta `fccCobroCliente` en estatus `BORRADOR`) |
| RT-08 | Los contadores de cartera (`PendingPaymentsReceived`, `OldestPendingPayment`) **deben filtrar** `IdCatCobroEstatus NOT IN (COMPLETADO, CANCELADO)`: capturados y sin capturar comparten tabla tras la fusión |
| RT-08 | `tpPedido` entra al Scaffold de Finanzas como **solo lectura**; su escritura de cancelación la hace ProquifaDotNet |
| RT-09 | `catEstadoProforma` y `CatEstadoTpPedido` son **propiedad de RE-016 / OBS-027** respectivamente; este requisito los consume |
| RT-10 | `fccFechaEstimadaPagoHistorial` es **append-only** (solo INSERT) |

## **4\. Endpoints nuevos/modificados**

| Solución / Proyecto     | Endpoint                                                              | Tipo  | Parámetros                                      | Salida                                                |
| :---------------------- | :-------------------------------------------------------------------- | :---- | :---------------------------------------------- | :---------------------------------------------------- |
| ProquifaDotNet.Finanzas | `POST /api/v1/validate-collection/client/search`                      | Nuevo | `[FromBody] QueryInfo` (filtro `textoBusqueda`) | `QueryResultDto<PaymentValidationClientDto>`          |
| ProquifaDotNet.Finanzas | `GET /api/v1/validate-collection/mailbox/{clientId}/pending`          | Nuevo | `clientId` (ruta)                               | Pendientes activos del Buzón                          |
| ProquifaDotNet.Finanzas | `PUT /api/v1/validate-collection/mailbox/{id}/close`                  | Nuevo | `id` (ruta)                                     | Cierra pendiente (`Activo=0`) — se invoca en RE-024   |
| ProquifaDotNet.Finanzas | `GET /api/v1/validate-collection/payment/{id}`                        | Nuevo | `id` (ruta)                                     | Cobro por Id                                          |
| ProquifaDotNet.Finanzas | `GET /api/v1/validate-collection/payment?clientId={id}`               | Nuevo | `clientId` (query)                              | Cobros activos del cliente                            |
| ProquifaDotNet.Finanzas | `PUT /api/v1/validate-collection/payment`                             | Nuevo | `[FromBody]` borrador                           | Upsert borrador (Paso 1, RE-024)                      |
| ProquifaDotNet.Finanzas | `GET /api/v1/validate-collection/proforma?clientId={id}`              | Nuevo | `clientId` (query)                              | Pedidos pendientes del cliente (modal)                |
| ProquifaDotNet.Finanzas | `PUT /api/v1/validate-collection/proforma/estimated-payment-date`     | Nuevo | `[FromBody]` (proforma, fecha)                  | Actualiza fecha estimada \+ historial (OBS-044)       |
| ProquifaDotNet.Finanzas | `PUT /api/v1/validate-collection/orders/{orderId}/cancel-non-payment` | Nuevo | `orderId` (ruta)                                | **Orquestador** de cancelación (rollback si falla)    |
| ProquifaDotNet          | `PUT /api/v1/orders/{orderId}/cancel-non-payment`                     | Nuevo | `orderId` (ruta)                                | Interno: trazabilidad en `tpPedido`                   |
| ProquifaDotNet.Timbrado | `POST /api/v1/invoices/{invoiceId}/cancel`                            | Nuevo | `invoiceId` (ruta), `claveMotivo`               | Interno: cancela CFDI ante el SAT (asíncrono posible) |

**Nota:** *Endpoints nuevos siguen* `{ip}/api/v1/`*, kebab-case e inglés. Los recursos reutilizan el glosario del módulo (*`validate-collection`*,* `client/search`*,* `mailbox`*,* `payment`*,* `proforma`*,* `estimated-payment-date`*,* `cancel-non-payment`*).*

# **Diseño de componentes**

## **1\. Responsabilidades por componente**

* **\`PaymentValidationClientService\` / \`PaymentValidationClientRepository\`:** ejecuta la consulta del listado con `QueryableExtensions` (`ToPagedList`, `ApplyFilter`) y calcula los campos derivados por cliente.  
* **\`CancelOrderForNonPaymentCommand\`:** orquestador de la cancelación distribuida (Caso A / Caso B), incluyendo la programación del job de Hangfire para el polling del SAT.  
* **\`UpdateEstimatedPaymentDateCommand\`:** actualiza `FechaPromesaPagoMonitoreoCobros` e inserta el historial.  
* **Endpoint interno de ProquifaDotNet:** escribe la trazabilidad de cancelación en `tpPedido` de forma idempotente.  
* **Endpoint interno de ProquifaDotNet.Timbrado:** cancela el CFDI ante el SAT vía TurboPac y persiste en `CFDICancelacion`.  
* **\`ConversorDivisas\`:** dolariza el saldo (patrón ya usado en `FacturasPendientesClienteObj`).

## **2\. Diagramas**

### **Diagrama 1 — Listado de la pantalla principal**

Muestra cómo Finanzas resuelve región/cartera del token, calcula los campos derivados por cliente y devuelve el resultado paginado.  
![][image1]  
**Código Mermaid del diagrama:**  
`sequenceDiagram`  
    `participant FE as Front (Validar Cobro)`  
    `participant FIN as ProquifaDotNet.Finanzas`  
    `participant DB as BD (SQL Server)`  
    `FE->>FIN: POST /api/v1/validate-collection/client/search (QueryInfo)`  
    `FIN->>FIN: Resuelve región + cartera (token IdentityServer)`  
    `FIN->>DB: Query cartera: fccCobroCliente (estatus abierto) + tpProformaPedido`  
    `DB-->>FIN: Filas por cliente`  
    `FIN->>FIN: Calcula conteos, TotalPendingBalance (USD), ContextualAction, SLA`  
    `FIN-->>FE: QueryResultDto<PaymentValidationClientDto> (orden por cobro más antiguo)`

### 

### **Diagrama 2 — Cancelación Caso A (sin CFDI)**

Orden remote-first: `tpPedido` (idempotente) primero, commit de proforma al final como paso decisivo.  
![][image2]  
**Código Mermaid del diagrama:**  
`sequenceDiagram`  
    `participant FE as Front`  
    `participant FIN as Finanzas (orquestador)`  
    `participant PQF as ProquifaDotNet`  
    `participant DB as BD`  
    `FE->>FIN: PUT .../orders/{orderId}/cancel-non-payment`  
    `FIN->>PQF: PUT /api/v1/orders/{orderId}/cancel-non-payment`  
    `alt tpPedido ya cancelado`  
        `PQF-->>FIN: OK (no-op idempotente)`  
    `else no cancelado`  
        `PQF->>DB: UPDATE tpPedido (FechaCancelacionPorFaltaPago, IdUsuarioCancelacion)`  
        `PQF-->>FIN: OK`  
    `end`  
    `FIN->>DB: BEGIN TRAN → UPDATE tpProformaPedido.IdcatEstadoProforma = Cancelada → COMMIT`  
    `alt commit OK`  
        `FIN-->>FE: OK (el pedido sale del listado)`  
    `else commit falla`  
        `FIN-->>FE: Error "reintente" (pedido sigue visible)`  
    `end`

### 

### **Diagrama 3 — Cancelación Caso B (con CFDI, Conducta 2\)**

CFDI-primero; el pedido desaparece de inmediato; el desenlace del SAT se resuelve por Hangfire.  
![][image3]  
**Código Mermaid del diagrama:**  
`sequenceDiagram`  
    `participant FE as Front`  
    `participant FIN as Finanzas (orquestador)`  
    `participant TIM as ProquifaDotNet.Timbrado`  
    `participant PQF as ProquifaDotNet`  
    `participant HF as Hangfire (Finanzas)`  
    `FE->>FIN: PUT .../orders/{orderId}/cancel-non-payment`  
    `FIN->>TIM: POST /api/v1/invoices/{invoiceId}/cancel (claveMotivo=03)`  
    `alt Rechazado síncrono / error`  
        `TIM-->>FIN: Rechazado`  
        `FIN-->>FE: Error (no toca lo interno)`  
    `else Cancelado o EnProceso`  
        `TIM-->>FIN: Cancelado | EnProceso`  
        `FIN->>PQF: cancela tpPedido (idempotente)`  
        `FIN->>FIN: UPDATE tpProformaPedido = Cancelada (commit)`  
        `FIN-->>FE: OK (pedido sale del listado)`  
        `FIN->>HF: Programa polling del estatus SAT`  
        `loop hasta desenlace`  
            `HF->>TIM: Consulta estatus (PAC)`  
            `alt EnProceso`  
                `HF->>HF: Re-programa`  
            `else Cancelado`  
                `HF->>HF: Fin`  
            `else Rechazado`  
                `HF->>HF: Envía correo a soporte (atención manual)`  
            `end`  
        `end`  
    `end`

# **Diseño de Modelo de Datos**

## **1\. Nuevas tablas**

### **`fccFechaEstimadaPagoHistorial` (OBS-044)**

Historial append-only de cambios de la fecha estimada de pago.

| Campo | Tipo | Descripción |
| :---- | :---- | :---- |
| `IdFccFechaEstimadaPagoHistorial` | uniqueidentifier PK | DEFAULT NEWID() |
| `IdTpProformaPedido` | uniqueidentifier FK → `tpProformaPedido` | Requerido |
| `FechaEstimadaPagoAnterior` | datetime2 NULL | NULL en el primer cambio |
| `FechaEstimadaPagaNueva` | datetime2 NULL | Nuevo valor |
| `FechaCambio` | datetime2 NOT NULL | DEFAULT SYSUTCDATETIME() |
| `IdUsuarioCambio` | uniqueidentifier NOT NULL | Trazabilidad |
| `Motivo` | varchar(300) NULL | Justificación opcional |

*Índice* `IX_fccFechaEstimadaPagoHistorial_ProformaPedido (IdTpProformaPedido, FechaCambio DESC)`*. Solo INSERT.*

## **2\. Tablas modificadas**

### **\~\~`fccPagoCliente` (×5)\~\~ — RETIRADO (absorbido por la fusión de RE-FU-024)**

**La tabla \`fccPagoCliente\` se elimina.** *RE-FU-024 la fusiona con* `fccFolioPagoCliente` *en la tabla nueva* **\`fccCobroCliente\`** *(*`CREATE`*; ambas anteriores* `DROP`*). Las 5 columnas que RE-023 iba a agregar por* `ALTER` **nacen directamente en ese \`CREATE TABLE\`***, así que* **este \`ALTER\` ya no se ejecuta***. Los ALTER nunca llegaron a BD (verificado 14/07/2026), por lo que no hay nada que revertir.*  
Destino de cada columna en `fccCobroCliente`:

| Columna que RE-023 iba a agregar | Destino en \`fccCobroCliente\` |
| :---- | :---- |
| `Confirmado` bit NOT NULL DEFAULT(0) | ❌ **No existe** — el estado vive en `IdCatCobroEstatus` (`BORRADOR`/`CAPTURADO`/…) |
| `FechaConfirmacion` datetime2 NULL | ✅ Sobrevive, tipo **\`DateTime\`** (convención de tabla nueva) |
| `IdUsuarioConfirmacion` uniqueidentifier NULL | 🔤 Renombrada a **\`IdUsuarioCobrador\`** |
| `Notas` varchar(500) NULL | 🔤 Renombrada a **\`NotasDeCobro\`** |
| `IdCatMoneda` FK → `catMoneda` | ✅ Sobrevive sin cambios |

### **`tpPedido` (cancelación)**

| Campo | Tipo | Descripción |
| :---- | :---- | :---- |
| `FechaCancelacionPorFaltaPago` | datetime2 NULL | Trazabilidad de la cancelación |
| `IdUsuarioCancelacion` | uniqueidentifier NULL | Usuario que canceló |
| `FechaSolicitudCancelacion` | datetime2 NULL | CFDI — fecha de solicitud de cancelación (OBS-042) |
| `EstadoCancelacionCFDI` | varchar(50) NULL | CFDI — estado de la cancelación ante el SAT (OBS-042) |

## **3\. Tablas eliminadas**

Ninguna **por este requisito**. No obstante, **RE-FU-024 elimina \`fccPagoCliente\`, \`fccFolioPagoCliente\` y la vista \`vFCCFolioPagoCliente\`** al fusionarlas en `fccCobroCliente` — RE-023 las consume, así que sus queries y su Scaffold quedan apuntados a la tabla nueva (ver *Impacto Técnico*).

## **4\. Relaciones**

* `fccFechaEstimadaPagoHistorial.IdTpProformaPedido` → `tpProformaPedido` (N:1).  
* `fccCobroCliente.IdCatMoneda` → `catMoneda` (N:1) — la FK se declara en el `CREATE TABLE` de RE-FU-024.  
* `fccCobroCliente.IdCatCobroEstatus` → `catCobroEstatus` (N:1) — eje de los contadores de cartera.  
* `CFDICancelacion` (existente) referencia el CFDI cancelado — **no requiere ALTER**.

## **5\. Reglas de integridad**

* `fccFechaEstimadaPagoHistorial` es append-only: prohibido UPDATE/DELETE.  
* La visibilidad del pedido en el listado depende exclusivamente de `tpProformaPedido.IdcatEstadoProforma`.  
* `tpProformaPedido` (incluida la columna `IdcatEstadoProforma`) es **propiedad de RE-016**; este requisito la consume.

# **Decisiones Tomadas**

| ID | Decisión |
| :---- | :---- |
| D1 | La cancelación se **orquesta desde Finanzas** en 3 aplicativos (proforma R/W en Finanzas, CFDI en Timbrado, `tpPedido` en ProquifaDotNet — solo lectura en Finanzas). Operación distribuida con rollback, no transacción atómica única |
| D2 | **Conducta 2** en Caso B: el pedido desaparece de inmediato; el desenlace del SAT se resuelve por Hangfire; si el SAT rechaza, se escala a soporte por correo (sin reapertura automática) |
| D3 | `ClaveMotivo = 03` fija para la cancelación del CFDI (a confirmar con área fiscal) |
| D4 | El saldo pendiente se muestra **siempre en USD** (OBS-046), México y Perú por igual |
| D5 | La fecha estimada de pago registra **historial completo** append-only (OBS-044) |
| D6 | El estado de la proforma se representa con `IdcatEstadoProforma` (propiedad de RE-016), no con el bit `Cancelada` |
| D7 | El INSERT del pendiente del Buzón **no se duplica** (ya en Mailbot, RE-008) |

# **Impacto Técnico**

## **1\. Impacto en código existente**

* **Reutilización:** `ConversorDivisas` (dolarización), `CorreoRecibidoClienteToPagoBO` (INSERT del Buzón — no se toca), `CFDICancelacionBO` (persistencia de cancelación de CFDI), integración TurboPac existente.  
* **Nuevo en Finanzas:** servicio/repositorio del listado, queries/commands CQRS, orquestador de cancelación y job de Hangfire.  
* **Nuevo en ProquifaDotNet:** endpoint interno de cancelación de `tpPedido`.  
* **Nuevo en Timbrado:** endpoint interno de cancelación de CFDI.

## **2\. Impacto en modelos**

* Scaffold EF Core de **\`fccCobroCliente\`** y `catCobroEstatus` (post-`CREATE` de RE-FU-024 — una sola entidad, ya no `fccFolioPagoCliente` \+ `fccPagoCliente`), `tpProformaPedido` (R/W) y `tpPedido` (solo lectura).  
* Nueva entidad `fccFechaEstimadaPagoHistorial`.

## **3\. Impacto en despliegue**

* Scripts de BD: `ALTER tpPedido` (×4 columnas) y `CREATE fccFechaEstimadaPagoHistorial`. **El \`ALTER fccPagoCliente\` (×5) se retira** (absorbido por el `CREATE TABLE fccCobroCliente` de RE-FU-024).  
* ⚠️ **Orden de despliegue:** el `CREATE TABLE fccCobroCliente` \+ el seed de `catCobroEstatus` (RE-FU-002/024) son **prerrequisito** de los servicios de cartera de este requisito.  
* Configuración de **Hangfire** en Finanzas para el polling del SAT.  
* **Dependencia de RE-016:** el esquema de `tpProformaPedido` (con `IdcatEstadoProforma`, `MontoPendiente` inicializado) debe estar aplicado antes.

# **Manejo de Errores y Excepciones**

| Escenario | Comportamiento esperado |
| :---- | :---- |
| Falla la llamada a ProquifaDotNet (Caso A, paso 1\) | Finanzas aborta; proforma intacta; mensaje "no se pudo cancelar, reintente" |
| Falla el commit de la proforma (Caso A, paso 2\) | ROLLBACK; el pedido **sigue** en el listado; reintento seguro (ProquifaDotNet idempotente) |
| `tpPedido` cancelado pero proforma no | El pedido sigue visible; al reintentar, ProquifaDotNet no-op y Finanzas re-commitea la proforma; sin inconsistencia observable |
| SAT rechaza de forma síncrona (Caso B, paso 1\) | Finanzas aborta, avisa al usuario, **no** toca lo interno |
| SAT `EnProceso` (Caso B, async) | Hangfire re-programa el sondeo |
| SAT rechaza de forma asíncrona (Caso B, paso 3\) | Finanzas envía correo a soporte; el pedido queda cancelado; sin reapertura automática |
| Reintento de cancelación | Todos los endpoints internos son idempotentes → sin doble efecto |

# **Estrategia de Pruebas (Diseño de las pruebas)**

## **1\. Pruebas funcionales (Criterios de Aceptación en DEV)**

* Listado muestra solo clientes de la cartera/región del usuario, con pendientes (CA-1).  
* Acción contextual correcta según cobros recibidos (CA-2).  
* Buscador con trim por nombre y por RFC/RUC (CA-3).  
* Modal lista pedidos con saldo y edita fecha estimada \+ historial (CA-4/CA-5).  
* Cancelación saca el pedido del listado y registra trazabilidad (CA-6).  
* Saldo en USD para México y Perú (CA-7).  
* Orden por antigüedad \+ indicador SLA (CA-8).

## **2\. Pruebas técnicas (unitarias e integración)**

* Cálculo de campos derivados (`ContextualAction`, `HasSlaExpired`, dolarización).  
* Idempotencia de los endpoints internos (`tpPedido`, CFDI): doble llamada \= un solo efecto.  
* Orquestación Caso A: fallo del commit de proforma → rollback y visibilidad conservada.  
* Orquestación Caso B: rechazo síncrono aborta sin tocar lo interno; polling de Hangfire transita `EnProceso → Cancelado / Rechazado`.  
* Append-only de `fccFechaEstimadaPagoHistorial`.

## **3\. Casos críticos**

* Cancelación con CFDI cuando el receptor tarda hasta 72 h (async).  
* Concurrencia: dos cobradores sobre el mismo cliente/pedido.  
* Cliente de Perú con Buzón en 0 cobros (acción `MANAGE_COLLECTIONS`).  
* Fallo parcial (`tpPedido` cancelado, proforma no) y reintento.

# **Control de versiones**

| Versión | Fecha | Autor | Tipo de Cambio | Descripción del cambio | Aprobó |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 1.0 | 20/07/2026 | Osmar Calderón Vázquez | Creación | Creación del documento. | Valdemar Farina Sánchez |
|  |  |  |  |  |  |
|  |  |  |  |  |  |

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,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>