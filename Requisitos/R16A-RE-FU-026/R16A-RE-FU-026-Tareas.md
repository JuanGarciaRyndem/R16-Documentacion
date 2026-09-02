# Tareas BackEnd — R16A-RE-FU-026
**Requisito:** Validar Cobro: Paso 2 México — Asociación
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)

---

> **Orden de ejecución sugerido:** BD ProquifaDotNet (DDL: tabla + ALTERs) → Endpoints Finanzas (listado documentos → catálogo NCs → motor de saldo → persistencia asociación → modal inconsistencia Paso 2 → auto-guardado)
> **Dependencias externas:** RE-FU-024 completo (cobros capturados en Paso 1 disponibles). Tareas 1-3 de BD son prerrequisitos de todas las tareas de Finanzas.

---

## Resumen de tareas

| #   | Clave             | Título simple                                                                                  | Tipo | Aplicativo              |
| --- | ----------------- | ---------------------------------------------------------------------------------------------- | ---- | ----------------------- |
| 1   | CREATE-TABL-M     | Crear tabla fccSaldoFavorCliente (Estado de Cuenta / Auxiliar saldo a favor)                   | BD   | ProquifaDotNet          |
| 2   | UPDATE-TABL-CH    | Agregar IdFCCPagoCliente (FK) a fccNotaCredito para vínculo NC-cobro                           | BD   | ProquifaDotNet          |
| 3   | UPDATE-TABL-CH    | Agregar AplicaMarkPendienteCancelacion a catTipoInconsistenciaCobro                            | BD   | ProquifaDotNet          |
| 4   | LIST-NO-FILTER    | Endpoint y servicio — Listado de proformas y facturas pendientes del cliente (Paso 2)          | Back | ProquifaDotNet.Finanzas |
| 5   | LIST-NO-FILTER    | Endpoint y servicio — Catálogo de Notas de Crédito vigentes del cliente                        | Back | ProquifaDotNet.Finanzas |
| 6   | ALG-COMPLX-LOGIC  | Algoritmo — Motor de cálculo dinámico del saldo de la asociación (multi-divisa)                | Back | ProquifaDotNet.Finanzas |
| 7   | SERV-TRANSACT     | Endpoint y servicio — Persistencia transaccional de la asociación cobro↔documentos             | Back | ProquifaDotNet.Finanzas |
| 8   | IMP-EXIST-SERVICE | Extender modal de inconsistencia para Paso 2 (tipos AplicaPaso=2 + marcado pedido cancelación) | Back | ProquifaDotNet.Finanzas |
| 9   | SERV-SIMPLE-PUT   | Endpoint y servicio — Auto-guardado del estado de la asociación (Paso 2)                       | Back | ProquifaDotNet.Finanzas |

---

## TAREA 1

**[ RE-FU-026 ] [CREATE-TABL-M] Crear tabla fccSaldoFavorCliente (Estado de Cuenta / Auxiliar saldo a favor)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 2

**Consideraciones previas:**
- Tabla nueva. No existe en BD.
- Registra dos tipos de saldo: `SaldoFavor` (sobrepago: cobros+NCs > adeudo) y `ToleranciaAplicada` (diferencia ≤ 100 MXN).
- El campo `Aplicado` bit indica si el saldo a favor ya fue usado en una sesión futura de Validar Cobro.
- `IdFCCPagoFacturaPedido` nullable: se pobla cuando el saldo a favor se aplica a una futura proforma.
- FK hacia `Cliente` y `fccPagoCliente`.
- Es **prerrequisito** de la Tarea 7 (persistencia de la asociación inserta en esta tabla cuando hay sobrepago o tolerancia).

**Objetivo general:**
Crear la tabla `fccSaldoFavorCliente` para registrar el Estado de Cuenta/Auxiliar del cliente con saldos a favor por sobrepago y diferencias de tolerancia 100 MXN aplicadas en el Paso 2, disponibles para uso en sesiones futuras de Validar Cobro.

**Objetivos específicos:**
- Ejecutar el DDL `CREATE TABLE fccSaldoFavorCliente` con: PK NEWID, FK IdCliente, FK IdFCCPagoCliente, TipoSaldo varchar(30), Monto decimal(18,4), MXN bit, USD bit, TipoCambio decimal(18,6) NULL, Aplicado bit DEFAULT(0), IdFCCPagoFacturaPedido uniqueidentifier NULL, Observaciones varchar(500) NULL, Activo bit DEFAULT(1), FechaRegistro datetime2(7) DEFAULT SYSUTCDATETIME(), FechaUltimaActualizacion datetime2(7) DEFAULT SYSUTCDATETIME().
- Verificar que las dos FK están activas.
- Verificar que ningún objeto existente en BD se ve afectado.

**Resultado esperado:**
Tabla `fccSaldoFavorCliente` existente en ProquifaDotNet, lista para recibir los saldos a favor y tolerancias aplicadas en el Paso 2 del wizard.

**Entregables:**
- Script DDL: `CREATE TABLE fccSaldoFavorCliente`
- Script de validación (estructura + FKs)

**Criterios de aceptación:**
- La tabla existe con la estructura definida en `R16A-RE-FU-026_BD.md`.
- FK activa hacia `Cliente.IdCliente`.
- FK activa hacia `fccPagoCliente.IdFCCPagoCliente`.
- DEFAULT constraints correctamente configurados (Aplicado=0, Activo=1, fechas SYSUTCDATETIME()).
- Tabla vacía al crear.

**Más información de la tarea:**
Ver sección *"Parte A / A1"* en `R16A-RE-FU-026-Back.md` y sección *"Tabla Nueva: fccSaldoFavorCliente"* en `R16A-RE-FU-026_BD.md`.

**Recursos:**
- `R16A-RE-FU-026_BD.md` — DDL fccSaldoFavorCliente
- `R16A-RE-FU-026-Back.md` — Parte A, sección A1

---

## TAREA 2

**[ RE-FU-026 ] [UPDATE-TABL-CH] Agregar IdFCCPagoCliente (FK) a fccNotaCredito para vínculo NC-cobro**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 2

**Consideraciones previas:**
- `fccNotaCredito` ya existe con: `Aplicada` bit, `IdCFDI`, `Monto`, `MXN/USD`, `Folio`.
- Se agrega `IdFCCPagoCliente` nullable para registrar en qué sesión de cobro se aplicó la NC.
- NULL para no romper registros existentes (NCs ya aplicadas antes de este requisito quedan con NULL).
- Verificar que el ALTER no rompa SPs, vistas ni triggers dependientes de `fccNotaCredito`.

**Objetivo general:**
Agregar `IdFCCPagoCliente` (FK) a `fccNotaCredito` para vincular cada NC aplicada con el cobro de la sesión del Paso 2, habilitando la trazabilidad de qué cobro aplicó qué NC.

**Objetivos específicos:**
- `ALTER TABLE dbo.fccNotaCredito ADD IdFCCPagoCliente uniqueidentifier NULL CONSTRAINT [FK_fccNotaCredito_PagoCliente] FOREIGN KEY REFERENCES dbo.fccPagoCliente([IdFCCPagoCliente])`
- Verificar que el ALTER no rompa SPs, vistas ni triggers dependientes de `fccNotaCredito`.
- Validar que todos los registros existentes tienen `IdFCCPagoCliente = NULL`.

**Resultado esperado:**
`fccNotaCredito.IdFCCPagoCliente` disponible como FK nullable, lista para ser poblada al aplicar una NC en el Paso 2 del wizard.

**Entregables:**
- Script DDL: `ALTER TABLE fccNotaCredito ADD IdFCCPagoCliente`
- Script de validación + checklist de objetos dependientes

**Criterios de aceptación:**
- `fccNotaCredito.IdFCCPagoCliente uniqueidentifier NULL` existe.
- FK activa hacia `fccPagoCliente.IdFCCPagoCliente`.
- Todos los registros existentes tienen `IdFCCPagoCliente = NULL`.
- Ningún SP, vista ni trigger presenta errores tras el ALTER.

**Más información de la tarea:**
Ver sección *"Parte A / A2"* en `R16A-RE-FU-026-Back.md` y sección *"Vinculo NC con Cobro"* en `R16A-RE-FU-026_BD.md`.

**Recursos:**
- `R16A-RE-FU-026_BD.md` — ALTER fccNotaCredito
- `R16A-RE-FU-026-Back.md` — Parte A, sección A2

---

## TAREA 3

**[ RE-FU-026 ] [UPDATE-TABL-CH] Agregar AplicaMarkPendienteCancelacion a catTipoInconsistenciaCobro**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 2

**Consideraciones previas:**
- `catTipoInconsistenciaCobro` fue creada en RE-FU-024 (Tarea 2).
- Se agrega el campo `AplicaMarkPendienteCancelacion` bit DEFAULT(0) para indicar si el tipo activa la opción de marcar el pedido como "Pendiente de cancelación por falta de pago".
- Solo el tipo `pagoincompletovencido` tiene este flag en 1.
- La Tarea 8 de Finanzas lee este campo para decidir si mostrar la opción de marcado al usuario.

**Objetivo general:**
Extender `catTipoInconsistenciaCobro` con el campo `AplicaMarkPendienteCancelacion` y activarlo para el tipo `pagoincompletovencido`, habilitando la lógica de marcado del pedido en Finanzas.

**Objetivos específicos:**
- `ALTER TABLE dbo.catTipoInconsistenciaCobro ADD AplicaMarkPendienteCancelacion bit NOT NULL CONSTRAINT [DF_catTipoInconsistenciaCobro_Mark] DEFAULT (0)`
- `UPDATE dbo.catTipoInconsistenciaCobro SET AplicaMarkPendienteCancelacion = 1 WHERE Clave = 'pagoincompletovencido'`
- Verificar que el ALTER no rompa objetos dependientes del catálogo.
- Validar que todos los demás registros tienen `AplicaMarkPendienteCancelacion = 0`.

**Resultado esperado:**
`catTipoInconsistenciaCobro` con el campo `AplicaMarkPendienteCancelacion` disponible y configurado correctamente, listo para ser leído por Finanzas en el modal de inconsistencias del Paso 2.

**Entregables:**
- Script DDL: `ALTER TABLE catTipoInconsistenciaCobro ADD AplicaMarkPendienteCancelacion`
- Script DML: `UPDATE catTipoInconsistenciaCobro SET AplicaMarkPendienteCancelacion=1 WHERE Clave='pagoincompletovencido'`
- Script de validación

**Criterios de aceptación:**
- `AplicaMarkPendienteCancelacion bit NOT NULL DEFAULT(0)` existe en `catTipoInconsistenciaCobro`.
- Solo el registro `pagoincompletovencido` tiene `AplicaMarkPendienteCancelacion = 1`.
- Todos los demás registros tienen `AplicaMarkPendienteCancelacion = 0`.

**Más información de la tarea:**
Ver sección *"Parte A / A3"* en `R16A-RE-FU-026-Back.md` y sección *"Inconsistencias Paso 2"* en `R16A-RE-FU-026_BD.md`.

**Recursos:**
- `R16A-RE-FU-026_BD.md` — ALTER catTipoInconsistenciaCobro
- `R16A-RE-FU-026-Back.md` — Parte A, sección A3

---

## TAREA 4

**[ RE-FU-026 ] [LIST-NO-FILTER] Endpoint y servicio — Listado de proformas y facturas pendientes del cliente (Paso 2)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 México

**Consideraciones previas:**
- Tareas 1-3 de BD deben estar ejecutadas.
- El listado es el panel central del Paso 2: proformas y facturas mezcladas, sin filtros adicionales.
- Finanzas llama a ProquifaDotNet para obtener documentos de `tpProformaPedido` (proformas normales) y `vfccFactura` (FAA — RE-FU-015, antes `tpProformaAdelanto`).
- Pueden coexistir documentos de diferentes empresas emisoras del grupo (GOL, MUN, PRO, PQF) en el mismo listado.
- El orden de selección de los documentos por el usuario determina la prioridad de aplicación del cobro.
- No requiere paginación (todos los documentos pendientes del cliente en el panel del Paso 2).

**Objetivo general:**
Implementar en Finanzas el endpoint que retorna todas las proformas y facturas pendientes de cobrar del cliente, mezcladas en un listado único sin filtros, para el panel central del Paso 2.

**Objetivos específicos:**
- Implementar `GET /api/v1/validate-collection/client/{idCliente}/pendingDocument` en Finanzas.
- Crear `GetPaymentValidationStep2DocumentsQuery` + Handler.
- Llamar a ProquifaDotNet para obtener `tpProformaPedido` (proformas) + `vfccFactura` (FAA — RE-FU-015), unificados en un solo listado.
- Retornar por documento: Tipo (PROFORMA / FAA), Folio, PedidoInterno, EmpresaEmisora, ImporteTotal, SaldoPendiente, ClaveMoneda.
- DTO: `PendingDocumentDto`, `PendingDocumentsResponseDto`.

**Resultado esperado:**
Endpoint `GET /api/v1/validate-collection/client/{idCliente}/pendingDocument` en Finanzas que retorna el listado unificado de proformas y facturas pendientes del cliente para el panel central del Paso 2.

**Entregables:**
- Endpoint `GET /api/v1/validate-collection/client/{idCliente}/pendingDocument`
- Query + Handler: `GetPaymentValidationStep2DocumentsQuery`
- DTOs: `PendingDocumentDto`, `PendingDocumentsResponseDto`
- Pruebas unitarias del Handler

**Criterios de aceptación:**
- El listado incluye proformas normales y FAA del cliente mezcladas.
- Solo documentos con `MontoPendiente > 0`, `Cancelada = 0` y `Activo = 1`.
- Documentos de diferentes empresas del grupo coexisten en el mismo listado.
- El endpoint valida que el cliente pertenece a la cartera del usuario activo.

**Más información de la tarea:**
Ver sección *"Parte B / B1"* en `R16A-RE-FU-026-Back.md`. Ver regla 4 en `R16A-RE-FU-026.md`.

**Recursos:**
- `R16A-RE-FU-026-Back.md` — Parte B, sección B1
- `R16A-RE-FU-026_BD.md` — Tablas de asociación existentes

---

## TAREA 5

**[ RE-FU-026 ] [LIST-NO-FILTER] Endpoint y servicio — Catálogo de Notas de Crédito vigentes del cliente**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 México

**Consideraciones previas:**
- Una NC vigente tiene `Aplicada=0` y `Activo=1` en `fccNotaCredito`.
- La aplicación de NCs es OPCIONAL: el usuario puede aplicar cero, una o varias NCs por documento.
- Las NCs no aplicadas en la sesión siguen vigentes para sesiones futuras.
- Cada NC se aplica completa a un solo documento (no se distribuye entre múltiples documentos).
- El UUID timbrado de cada NC aplicada se preserva para su uso en el Paso 3 (nodo `CFDIRelacionados`).
- Tarea 2 debe estar ejecutada (campo `IdFCCPagoCliente` en `fccNotaCredito`).

**Objetivo general:**
Implementar en Finanzas el endpoint que retorna las Notas de Crédito vigentes del cliente disponibles para aplicar en el Paso 2, con los campos necesarios para su selección y posterior uso en el Paso 3.

**Objetivos específicos:**
- Implementar `GET /api/v1/validate-collection/client/{idCliente}/activeCreditNote` en Finanzas.
- Crear `GetActiveCreditNotesQuery` + Handler.
- Llamar a ProquifaDotNet: `SELECT FROM fccNotaCredito WHERE Aplicada=0 AND Activo=1 AND IdCliente=@Id`.
- Retornar: `IdFCCNotaCredito`, `Folio`, `IdCFDI` (UUID SAT), `Monto`, `ClaveMoneda`.
- DTO: `ActiveCreditNoteDto`, `ActiveCreditNotesResponseDto`.

**Resultado esperado:**
Endpoint `GET /api/v1/validate-collection/client/{idCliente}/activeCreditNote` en Finanzas que retorna las NCs vigentes del cliente listas para ser seleccionadas opcionalmente en el Paso 2.

**Entregables:**
- Endpoint `GET /api/v1/validate-collection/client/{idCliente}/activeCreditNote`
- Query + Handler: `GetActiveCreditNotesQuery`
- DTOs: `ActiveCreditNoteDto`, `ActiveCreditNotesResponseDto`
- Pruebas unitarias del Handler

**Criterios de aceptación:**
- Solo retorna NCs con `Aplicada=0` y `Activo=1`.
- El `IdCFDI` (UUID timbrado SAT) se incluye en el DTO para uso posterior en el Paso 3.
- NCs ya aplicadas (`Aplicada=1`) NO aparecen en el catálogo.
- El endpoint valida que el cliente pertenece a la cartera del usuario activo.

**Más información de la tarea:**
Ver sección *"Parte B / B2"* en `R16A-RE-FU-026-Back.md`. Ver regla 6 en `R16A-RE-FU-026.md`.

**Recursos:**
- `R16A-RE-FU-026-Back.md` — Parte B, sección B2
- `R16A-RE-FU-026_BD.md` — fccNotaCredito existente

---

## TAREA 6

**[ RE-FU-026 ] [ALG-COMPLX-LOGIC] Algoritmo — Motor de cálculo dinámico del saldo de la asociación (multi-divisa)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 México

**Consideraciones previas:**
- Este motor es la lógica central del Paso 2: se invoca cada vez que el usuario selecciona/deselecciona cobros, documentos o NCs.
- La fórmula base: `AssociationBalance = (SumAppliedPayments + SumAppliedCreditNotes) - SumSelectedDocumentsDebt`.
- **Multi-divisa:** cuando los documentos están en moneda distinta a la del cobro, Finanzas convierte el importe del documento a moneda del cobro usando `fccPagoCliente.TipoDeCambio` (TC capturado en Paso 1). Todos los totales del panel se expresan en moneda del cobro.
- El orden de selección de documentos determina la prioridad de aplicación del cobro (primer documento seleccionado = primera cobertura).
- La lógica de escenarios: exacto (=0), sobrepago (>0), tolerancia (≤100 MXN negativo), insuficiente (>100 MXN negativo).
- ⚠️ Pendiente confirmar tratamiento de tolerancia en monedas distintas a MXN.

**Objetivo general:**
Implementar en Finanzas el servicio `AssociationBalanceCalculatorService` que calcula el saldo dinámico de la asociación, aplica las conversiones multi-divisa con el TC del cobro, determina el escenario resultante y retorna el desglose completo para visualización en el panel de totales del Paso 2.

**Objetivos específicos:**
- Crear `IAssociationBalanceCalculatorService` con método `Calculate(selectedPayments, selectedDocuments, appliedCreditNotes, paymentExchangeRate)`.
- Implementar la conversión multi-divisa: importe documento en moneda distinta a cobro → convertir a moneda cobro usando TC del Paso 1.
- Calcular los totales del panel: DocumentsDebt (en moneda cobro), AppliedCreditNotes (en moneda cobro), AppliedPayments, AssociationBalance, escenario resultante.
- Determinar el escenario: EXACT, OVERPAYMENT, TOLERANCE_100_MXN, INSUFFICIENT.
- Exponer `POST /api/v1/validate-collection/client/{idCliente}/balance/calculate` para llamada del Front al seleccionar/deseleccionar elementos.
- DTO de request: `CalculateBalanceRequestDto` (cobros, documentos con orden de selección, NCs).
- DTO de response: `CalculateBalanceResponseDto` (totales por línea, AssociationBalance, Scenario, TC aplicado por documento).

**Resultado esperado:**
Servicio `AssociationBalanceCalculatorService` en Finanzas que calcula y retorna el saldo dinámico de la asociación con multi-divisa y determinación del escenario, listo para ser consumido por el panel de totales del Paso 2 en tiempo real.

**Entregables:**
- `IAssociationBalanceCalculatorService` + `AssociationBalanceCalculatorService` en Application de Finanzas
- Endpoint `POST /api/v1/validate-collection/client/{idCliente}/balance/calculate`
- DTOs: `CalculateBalanceRequestDto`, `CalculateBalanceResponseDto`
- Pruebas unitarias para los 4 escenarios (EXACT, OVERPAYMENT, TOLERANCE, INSUFFICIENT) y para conversión multi-divisa

**Criterios de aceptación:**
- `Scenario = EXACT` cuando AssociationBalance = 0.
- `Scenario = OVERPAYMENT` cuando AssociationBalance > 0.
- `Scenario = TOLERANCE_100_MXN` cuando 0 > AssociationBalance AND ABS ≤ 100 MXN.
- `Scenario = INSUFFICIENT` cuando AssociationBalance < 0 AND ABS > 100 MXN.
- Los importes de documentos en moneda distinta a la del cobro se convierten correctamente al TC del Paso 1.
- El TC aplicado por documento está disponible en el DTO de response para el tooltip del Front.

**Más información de la tarea:**
Ver sección *"Parte B / B3"* en `R16A-RE-FU-026-Back.md`. Ver reglas 7-11 y criterio de multi-divisa en `R16A-RE-FU-026.md`.

**Recursos:**
- `R16A-RE-FU-026-Back.md` — Parte B, sección B3
- `R16A-RE-FU-026.md` — Reglas 7-11, 19, 20

---

## TAREA 7

**[ RE-FU-026 ] [SERV-TRANSACT] Endpoint y servicio — Persistencia transaccional de la asociación cobro↔documentos**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 México

**Consideraciones previas:**
- Se ejecuta cuando el usuario avanza al Paso 3 con "Continuar" con la asociación cerrada.
- Solo se permite avanzar si el escenario es EXACTO, SOBREPAGO o TOLERANCIA_100_MXN (no INSUFICIENTE).
- Finanzas envía a ProquifaDotNet una transacción completa con todas las operaciones del Paso 2.
- Tareas 1-3 deben estar ejecutadas (tablas y campos disponibles en ProquifaDotNet).
- Las asociaciones son editables mientras el usuario esté en el Paso 2; al confirmar se fijan.

**Objetivo general:**
Implementar en Finanzas el endpoint transaccional que persiste en ProquifaDotNet toda la asociación del Paso 2 en una sola transacción: INSERT asociaciones cobro↔documentos, UPDATE NCs aplicadas, INSERT saldo a favor si aplica.

**Objetivos específicos:**
- Implementar `POST /api/v1/validate-collection/client/{idCliente}/associationConfirmation` en Finanzas.
- Crear `ConfirmAssociationStep2Command` + Handler.
- Validar en Finanzas que el escenario resultante no es INSUFICIENTE antes de enviar.
- Llamar a ProquifaDotNet en transacción:
  - `INSERT fccPagoFacturaPedido` por cada cobro↔proforma normal.
  - `INSERT fccPagoFacturaAdelanto` por cada cobro↔FAA.
  - `UPDATE fccNotaCredito SET Aplicada=1, IdFCCPagoCliente=@IdCobro` por cada NC aplicada.
  - `INSERT fccSaldoFavorCliente` si hay SOBREPAGO (`TipoSaldo='SaldoFavor'`) o TOLERANCIA (`TipoSaldo='ToleranciaAplicada'`).
- En caso de error, rollback completo de la transacción.
- Registrar en Serilog con contexto (usuario, módulo, operación).

**Resultado esperado:**
Endpoint `POST .../associationConfirmation` en Finanzas que persiste toda la asociación del Paso 2 en ProquifaDotNet en una sola transacción, lista para avanzar al Paso 3.

**Entregables:**
- Endpoint `POST /api/v1/validate-collection/client/{idCliente}/associationConfirmation`
- Command + Handler: `ConfirmAssociationStep2Command`
- DTO de request: `ConfirmAssociationRequestDto`
- Pruebas unitarias (incluyendo rollback en error y bloqueo si scenario=INSUFFICIENT)

**Criterios de aceptación:**
- Las asociaciones se insertan correctamente en `fccPagoFacturaPedido` y/o `fccPagoFacturaAdelanto`.
- Las NCs aplicadas quedan con `Aplicada=1` e `IdFCCPagoCliente` poblado.
- Si hay sobrepago, se inserta `fccSaldoFavorCliente` con `TipoSaldo='SaldoFavor'`.
- Si hay tolerancia, se inserta `fccSaldoFavorCliente` con `TipoSaldo='ToleranciaAplicada'`.
- Si el escenario es INSUFFICIENT, el endpoint retorna error y NO persiste nada.
- Si cualquier operación falla, la transacción hace rollback completo.

**Más información de la tarea:**
Ver sección *"Parte B / B4"* en `R16A-RE-FU-026-Back.md`. Ver reglas 8-11 en `R16A-RE-FU-026.md`.

**Recursos:**
- `R16A-RE-FU-026-Back.md` — Parte B, sección B4
- `R16A-RE-FU-026_BD.md` — Tablas de asociación + fccSaldoFavorCliente

---

## TAREA 8

**[ RE-FU-026 ] [IMP-EXIST-SERVICE] Extender modal de inconsistencia para Paso 2 (tipos AplicaPaso=2 + marcado pedido cancelación)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 México

**Consideraciones previas:**
- El modal de inconsistencia del Paso 1 fue implementado en RE-FU-024 (Tarea 10). Este requisito lo extiende para el Paso 2.
- Los tipos del Paso 2 tienen `AplicaPaso='2'` en el catálogo.
- Tarea 3 debe estar ejecutada (`AplicaMarkPendienteCancelacion` disponible en el catálogo).
- Para el tipo `pagoincompletovencido` (`AplicaMarkPendienteCancelacion=1`), Finanzas habilita la opción de marcar el pedido asociado.
- ⚠️ Pendiente definir el campo exacto en `tpPedido` para el estado "Pendiente de cancelación" y el mecanismo de transferencia al área de Finanzas.

**Objetivo general:**
Extender en Finanzas el modal de inconsistencias para el Paso 2: filtrar tipos por `AplicaPaso='2'`, registrar la inconsistencia contra el cobro, y para el tipo `pagoincompletovencido` habilitar y procesar el marcado del pedido como "Pendiente de cancelación por falta de pago".

**Objetivos específicos:**
- Extender el endpoint de catálogo de tipos de inconsistencia: `GET /api/v1/validate-collection/inconsistencyType?step=2`.
- Extender el endpoint de registro de inconsistencias para el Paso 2 con detección del flag `AplicaMarkPendienteCancelacion`.
- Si el tipo tiene `AplicaMarkPendienteCancelacion=1` y el usuario confirma el marcado: llamar a ProquifaDotNet para actualizar el estado del pedido.
- Registrar en Serilog las inconsistencias del Paso 2 con contexto de usuario, módulo y operación.

**Resultado esperado:**
El modal de inconsistencias del Paso 2 en Finanzas retorna los tipos correctos (`AplicaPaso='2'`) y procesa el marcado del pedido cuando el tipo es `pagoincompletovencido`.

**Entregables:**
- Extensión del endpoint `GET /api/v1/validate-collection/inconsistencyType?step=2`
- Extensión del Command `RegisterPaymentInconsistencyCommand` para manejar el flujo del Paso 2
- Pruebas unitarias (incluyendo: solo tipos AplicaPaso=2, flujo pagoincompletovencido con y sin marcado del pedido)

**Criterios de aceptación:**
- El combo del modal Paso 2 solo muestra tipos con `AplicaPaso='2'`.
- Los tipos del Paso 1 (`AplicaPaso='1'`) NO aparecen en el combo del Paso 2.
- Para el tipo `pagoincompletovencido`, se habilita la opción de marcar el pedido.
- El marcado del pedido se persiste correctamente en ProquifaDotNet.
- Para el tipo `pagoinsuficiente`, solo se registra la inconsistencia (sin marcado de pedido).

**Más información de la tarea:**
Ver sección *"Parte B / B5"* en `R16A-RE-FU-026-Back.md`. Ver reglas 12-14 en `R16A-RE-FU-026.md`.

**Recursos:**
- `R16A-RE-FU-026-Back.md` — Parte B, sección B5
- `R16A-RE-FU-026_BD.md` — catTipoInconsistenciaCobro + AplicaMarkPendienteCancelacion
- `R16A-RE-FU-024-Tareas.md` — Tarea 10 (base del modal a extender)

---

## TAREA 9

**[ RE-FU-026 ] [SERV-SIMPLE-PUT] Endpoint y servicio — Auto-guardado del estado de la asociación (Paso 2)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 México

**Consideraciones previas:**
- El auto-guardado del Paso 2 es transparente al usuario (sin botón "Guardar" manual).
- Se persiste: cobros seleccionados, documentos seleccionados con orden, NCs aplicadas, inconsistencias marcadas.
- Las asociaciones son editables mientras el usuario esté en el Paso 2 (pueden deshacerse y rehacer).
- Una vez el usuario avanza al Paso 3 (Tarea 7 ejecutada), las asociaciones quedan fijas.
- El estado se puede guardar en una tabla auxiliar de sesión o directamente en las tablas transaccionales con un flag "en progreso" — patrón a definir según la arquitectura de Finanzas.

**Objetivo general:**
Implementar en Finanzas el mecanismo de auto-guardado del estado de la asociación del Paso 2, preservando el progreso del usuario de forma transparente para que pueda retomar la sesión si navega fuera del Paso 2.

**Objetivos específicos:**
- Implementar `PUT /api/v1/validate-collection/client/{idCliente}/association/draft` en Finanzas.
- Crear `AutoSaveAssociationCommand` + Handler.
- Persistir en una estructura de sesión/borrador en ProquifaDotNet o en Finanzas: cobros seleccionados, documentos con orden, NCs seleccionadas, inconsistencias marcadas.
- Respetar la editabilidad: el borrador puede sobrescribirse hasta que se ejecute la Tarea 7 (confirmación).
- Definir el enfoque de persistencia del borrador (tabla auxiliar `fccAsociacionBorrador` u otro patrón).

**Resultado esperado:**
Mecanismo de auto-guardado del Paso 2 en Finanzas que preserva el estado de la asociación en progreso, permitiendo al usuario retomar la sesión del Paso 2 sin perder su trabajo.

**Entregables:**
- Endpoint `PUT /api/v1/validate-collection/client/{idCliente}/association/draft`
- Command + Handler: `AutoSaveAssociationCommand`
- DTO: `AutoSaveAssociationRequestDto`
- Pruebas unitarias (incluyendo sobrescritura de borrador previo)

**Criterios de aceptación:**
- El borrador de la asociación se persiste correctamente al navegar o salir del Paso 2.
- El borrador puede sobrescribirse hasta que la asociación se confirme con la Tarea 7.
- Al retomar el Paso 2, el estado previo se carga desde el borrador.
- No existe botón "Guardar" manual expuesto al usuario.

**Más información de la tarea:**
Ver sección *"Parte B / B6"* en `R16A-RE-FU-026-Back.md`. Ver reglas 15 y 16 en `R16A-RE-FU-026.md`.

**Recursos:**
- `R16A-RE-FU-026-Back.md` — Parte B, sección B6
- `R16A-RE-FU-026.md` — Reglas 15 