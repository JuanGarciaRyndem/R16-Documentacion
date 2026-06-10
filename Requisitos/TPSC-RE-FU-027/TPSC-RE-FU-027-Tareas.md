# Tareas BackEnd — TPSC-RE-FU-027
**Requisito:** Validar Cobro: Paso 2 Perú — Asociación
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)

---

> **Orden de ejecución sugerido:** BD ProquifaDotNet (ALTERs: fccNotaCredito + fccSaldoFavorCliente) → Extensiones Finanzas (listado documentos → catálogo NCs → motor de saldo → persistencia asociación → modal inconsistencia → auto-guardado)
> **Dependencias externas:** RE-FU-025 completo (cobros Perú capturados en Paso 1 disponibles) y RE-FU-026 completo (infraestructura Paso 2 México: fccSaldoFavorCliente, catTipoInconsistenciaCobro, motor de saldo, endpoints base). Tareas 1-2 de BD son prerrequisitos de las Tareas 5, 6 y 4 de Finanzas.
> **Nota:** Este requisito extiende la infraestructura del Paso 2 México (RE-FU-026) para Perú. La mayoría de los servicios de Finanzas son extensiones de los existentes; **no se crean nuevos servicios desde cero**. Diferencia clave: **sin Complemento de Pago** — la asociación es solo operativa.

---

## Resumen de tareas

| #   | Clave             | Título simple                                                                                   | Tipo | Aplicativo              |
| --- | ----------------- | ----------------------------------------------------------------------------------------------- | ---- | ----------------------- |
| 1   | UPDATE-TABL-CH    | Agregar PEN y MontoPEN a fccNotaCredito para soporte de NCs en soles peruanos                   | BD   | ProquifaDotNet          |
| 2   | UPDATE-TABL-CH    | Verificar y agregar campo PEN a fccSaldoFavorCliente si fue creada sin soporte PEN              | BD   | ProquifaDotNet          |
| 3   | IMP-EXIST-SERVICE | Extender endpoint de documentos pendientes del cliente para Región Perú (Paso 2)                | Back | ProquifaDotNet.Finanzas |
| 4   | IMP-EXIST-SERVICE | Extender endpoint de NCs vigentes para Región Perú (incluir campo PEN/MontoPEN)                 | Back | ProquifaDotNet.Finanzas |
| 5   | IMP-EXIST-SERVICE | Extender motor de cálculo del saldo para Región Perú (moneda base PEN, tolerancia pendiente)    | Back | ProquifaDotNet.Finanzas |
| 6   | IMP-EXIST-SERVICE | Extender persistencia transaccional de la asociación para Región Perú (sin Complemento de Pago) | Back | ProquifaDotNet.Finanzas |
| 7   | IMP-EXIST-SERVICE | Extender modal de inconsistencia Paso 2 para Región Perú                                        | Back | ProquifaDotNet.Finanzas |
| 8   | IMP-EXIST-SERVICE | Extender auto-guardado del Paso 2 para Región Perú                                              | Back | ProquifaDotNet.Finanzas |

---

## TAREA 1

**[ RE-FU-027 ] [UPDATE-TABL-CH] Agregar PEN y MontoPEN a fccNotaCredito para soporte de NCs en soles peruanos**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 2

**Consideraciones previas:**
- `fccNotaCredito` ya existe con campos `MXN bit`, `USD bit`, `MontoMXN`, `MontoUSD`. Se agrega soporte PEN con el mismo patrón.
- `PEN bit NOT NULL DEFAULT(0)`: identifica que la NC está emitida en soles peruanos.
- `MontoPEN decimal(18,4) NULL`: monto de la NC en soles; NULL para NCs existentes que no son PEN.
- Sin impacto en registros existentes: `PEN=0` y `MontoPEN=NULL` para todos los registros anteriores.
- Verificar que el ALTER no rompa SPs, vistas ni triggers dependientes de `fccNotaCredito`.
- Es prerrequisito de la Tarea 4 (endpoint NCs vigentes lee los nuevos campos).

**Objetivo general:**
Agregar soporte de moneda PEN (soles peruanos) a `fccNotaCredito` para que las Notas de Crédito de clientes Perú puedan registrarse y ser leídas correctamente en el Paso 2 de Validar Cobro Perú.

**Objetivos específicos:**
- Ejecutar `ALTER TABLE dbo.fccNotaCredito ADD PEN bit NOT NULL CONSTRAINT [DF_fccNotaCredito_PEN] DEFAULT (0)`.
- Ejecutar `ALTER TABLE dbo.fccNotaCredito ADD MontoPEN decimal(18,4) NULL`.
- Verificar que los registros existentes tienen `PEN=0` y `MontoPEN=NULL`.
- Verificar que el ALTER no rompe SPs, vistas ni triggers dependientes de `fccNotaCredito`.

**Resultado esperado:**
`fccNotaCredito` con campos `PEN bit` y `MontoPEN decimal(18,4)` disponibles para registrar y leer NCs de clientes Perú en el Paso 2 del wizard.

**Entregables:**
- Script DDL: `ALTER TABLE fccNotaCredito ADD PEN + MontoPEN`
- Script de validación (estructura + registros existentes sin afectación)

**Criterios de aceptación:**
- `fccNotaCredito.PEN bit NOT NULL DEFAULT(0)` existe.
- `fccNotaCredito.MontoPEN decimal(18,4) NULL` existe.
- Todos los registros existentes tienen `PEN=0` y `MontoPEN=NULL`.
- Ningún SP, vista ni trigger presenta errores tras el ALTER.

**Más información de la tarea:**
Ver sección *"Parte A / A1"* en `TPSC-RE-FU-027-Back.md` y sección *"ALTER TABLE fccNotaCredito (soporte PEN)"* en `TPSC-RE-FU-027_BD.md`.

**Recursos:**
- `TPSC-RE-FU-027_BD.md` — ALTER TABLE fccNotaCredito
- `TPSC-RE-FU-027-Back.md` — Parte A, sección A1

---

## TAREA 2

**[ RE-FU-027 ] [UPDATE-TABL-CH] Verificar y agregar campo PEN a fccSaldoFavorCliente si fue creada sin soporte PEN**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 2

**Consideraciones previas:**
- `fccSaldoFavorCliente` fue creada en RE-FU-026 (Tarea 1). Esta tarea es **condicional**: solo aplica si la tabla fue creada sin campo `PEN`.
- Si la tabla ya tiene `PEN bit`, esta tarea no aplica — verificar estructura antes de ejecutar.
- Se recomienda que RE-FU-026 incluya `PEN` desde la creación inicial para evitar esta dependencia.
- Esta tarea debe ejecutarse antes de la Tarea 6 (persistencia transaccional Perú escribe en `fccSaldoFavorCliente` con `PEN=1`).

**Objetivo general:**
Asegurar que `fccSaldoFavorCliente` tiene soporte para saldos a favor en soles peruanos (PEN), habilitando la correcta escritura de sobrepagos y tolerancias de clientes Perú en el Paso 2.

**Objetivos específicos:**
- Verificar estructura actual de `fccSaldoFavorCliente`: `SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='fccSaldoFavorCliente'`.
- Si NO existe campo `PEN`: ejecutar `ALTER TABLE dbo.fccSaldoFavorCliente ADD PEN bit NOT NULL CONSTRAINT [DF_fccSaldoFavorCliente_PEN] DEFAULT (0)`.
- Si ya existe `PEN`: esta tarea queda como "verificada/no aplica".
- Verificar que registros existentes (MEX) tienen `PEN=0`.

**Resultado esperado:**
`fccSaldoFavorCliente` con campo `PEN bit` disponible para registrar saldos a favor y tolerancias de clientes Perú en el Paso 2.

**Entregables:**
- Script de verificación de estructura
- Script DDL condicional: `ALTER TABLE fccSaldoFavorCliente ADD PEN` (si aplica)

**Criterios de aceptación:**
- `fccSaldoFavorCliente.PEN bit NOT NULL DEFAULT(0)` existe (ya sea preexistente o agregado por esta tarea).
- Registros existentes de México tienen `PEN=0`.
- Ningún objeto dependiente presenta errores.

**Más información de la tarea:**
Ver sección *"Parte A / A2"* en `TPSC-RE-FU-027-Back.md` y sección *"fccSaldoFavorCliente para Peru"* en `TPSC-RE-FU-027_BD.md`.

**Recursos:**
- `TPSC-RE-FU-027_BD.md` — fccSaldoFavorCliente para Perú
- `TPSC-RE-FU-027-Back.md` — Parte A, sección A2

---

## TAREA 3

**[ RE-FU-027 ] [IMP-EXIST-SERVICE] Extender endpoint de documentos pendientes del cliente para Región Perú (Paso 2)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 Perú

**Consideraciones previas:**
- El endpoint `GET /api/validar-cobro/clientes/{idCliente}/documentos-pendientes` fue implementado en RE-FU-026 (Tarea 4) para México.
- Extender para soportar Región Perú: agregar filtro `Region='PER'` y reflejar que el emisor es siempre Golocaer S.A.C. (GOLPERU) — sin mezcla de emisores.
- El campo análogo a `CFDIGenerada` en México es `CPEGenerada` en Perú (`tpProformaPedido.IdCPEGenerada`).
- No requiere paginación (todos los documentos pendientes del cliente en el panel del Paso 2).
- El listado sigue siendo proformas y FAA mezcladas, mismo patrón que México.

**Objetivo general:**
Extender en Finanzas el endpoint de documentos pendientes del Paso 2 para que soporte clientes de Región Perú, retornando proformas y FAA de GOLPERU con el campo `CPEGenerada` correspondiente.

**Objetivos específicos:**
- Extender `GetValidarCobroPaso2DocumentosQuery` + Handler para manejar `Region='PER'`.
- Agregar filtro `Region='PER'` al llamar a ProquifaDotNet.
- Incluir campo `CPEGenerada` (`tpProformaPedido.IdCPEGenerada`) en el DTO de respuesta para clientes Perú.
- Reflejar que `EmpresaEmisora = 'GOLPERU'` siempre (sin mezcla) en los documentos de Perú.
- Actualizar DTO si es necesario: `ValidarCobroPaso2DocumentoDto` con campo `CPEGenerada`.

**Resultado esperado:**
El endpoint `GET /api/validar-cobro/clientes/{idCliente}/documentos-pendientes` retorna correctamente el listado de documentos pendientes para clientes de Región Perú con `EmpresaEmisora='GOLPERU'` y `CPEGenerada` disponible.

**Entregables:**
- Extensión de `GetValidarCobroPaso2DocumentosQuery` + Handler
- Actualización de DTO: `ValidarCobroPaso2DocumentoDto` (campo `CPEGenerada`)
- Pruebas unitarias para Región Perú (emisor único GOLPERU, campo CPEGenerada)

**Criterios de aceptación:**
- Para clientes Perú: solo documentos con `Region='PER'`, `MontoPendiente > 0`, `Cancelada = 0`, `Activo = 1`.
- `EmpresaEmisora = 'GOLPERU'` en todos los documentos Perú.
- El campo `CPEGenerada` está disponible en el DTO de respuesta para clientes Perú.
- El endpoint sigue funcionando correctamente para clientes México (sin regresión).

**Más información de la tarea:**
Ver sección *"Parte B / B1"* en `TPSC-RE-FU-027-Back.md`. Ver regla 5 en `TPSC-RE-FU-027.md`.

**Recursos:**
- `TPSC-RE-FU-027-Back.md` — Parte B, sección B1
- `TPSC-RE-FU-026-Tareas.md` — Tarea 4 (endpoint base a extender)

---

## TAREA 4

**[ RE-FU-027 ] [IMP-EXIST-SERVICE] Extender endpoint de NCs vigentes para Región Perú (incluir campo PEN/MontoPEN)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 Perú

**Consideraciones previas:**
- El endpoint `GET /api/validar-cobro/clientes/{idCliente}/notas-credito-vigentes` fue implementado en RE-FU-026 (Tarea 5) para México.
- La Tarea 1 debe estar ejecutada: `fccNotaCredito` ya tiene campos `PEN` y `MontoPEN`.
- Para Perú el identificador fiscal es el UUID del CPE peruano, no `IdCFDI` del SAT.
- ⚠️ La mecánica fiscal de referencia de la NC peruana (catálogo 09 SUNAT) se desarrolla en RE-FU-033/035 y NO se implementa aquí; en este Paso solo se aplica operativamente al adeudo.

**Objetivo general:**
Extender en Finanzas el endpoint de NCs vigentes para que soporte clientes de Región Perú, retornando los campos `PEN` y `MontoPEN` y el identificador fiscal del CPE peruano en lugar del `IdCFDI` SAT.

**Objetivos específicos:**
- Extender `GetNotasCreditoVigentesQuery` + Handler para manejar `Region='PER'`.
- Incluir campos `PEN` y `MontoPEN` en el DTO de respuesta.
- Para clientes Perú, retornar el identificador del CPE peruano en lugar del `IdCFDI` SAT (pendiente confirmar nombre del campo en `fccNotaCredito`).
- Filtro: `Aplicada=0 AND Activo=1 AND IdCliente=@Id AND Region='PER'`.

**Resultado esperado:**
El endpoint de NCs vigentes retorna correctamente las NCs de clientes Perú con los campos `PEN` y `MontoPEN` disponibles para el cálculo del saldo en el Paso 2.

**Entregables:**
- Extensión de `GetNotasCreditoVigentesQuery` + Handler
- Actualización de DTO: `NotaCreditoVigenteDto` (campos `PEN`, `MontoPEN`, identificador CPE)
- Pruebas unitarias para Región Perú

**Criterios de aceptación:**
- Para clientes Perú: solo NCs con `Aplicada=0`, `Activo=1` y `Region='PER'`.
- Los campos `PEN` y `MontoPEN` están disponibles en el DTO.
- El endpoint sigue funcionando correctamente para clientes México (sin regresión).
- NCs ya aplicadas (`Aplicada=1`) NO aparecen en el catálogo.

**Más información de la tarea:**
Ver sección *"Parte B / B2"* en `TPSC-RE-FU-027-Back.md`. Ver regla 7 en `TPSC-RE-FU-027.md`.

**Recursos:**
- `TPSC-RE-FU-027-Back.md` — Parte B, sección B2
- `TPSC-RE-FU-026-Tareas.md` — Tarea 5 (endpoint base a extender)
- `TPSC-RE-FU-027_BD.md` — ALTER fccNotaCredito (campos PEN/MontoPEN)

---

## TAREA 5

**[ RE-FU-027 ] [IMP-EXIST-SERVICE] Extender motor de cálculo del saldo para Región Perú (moneda base PEN, tolerancia pendiente)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 Perú

**Consideraciones previas:**
- `SaldoAsociacionCalculatorService` fue implementado en RE-FU-026 (Tarea 6) con moneda base MXN y umbral de tolerancia 100 MXN.
- Para Perú: moneda base PEN, umbral de tolerancia **pendiente de definir** con PROQUIFA Tesorería (⚠️ Brecha B1).
- La fórmula base es idéntica: `SaldoAsociacion = (SumaCobrosAplicados + SumaNCAplicadas) - SumaAdeudoDocumentos`.
- El TC para conversión multi-divisa proviene de `fccPagoCliente.TipoDeCambio` del Paso 1 Perú — fuente pendiente (⚠️ Brecha B2).
- El servicio debe detectar la región del cliente y aplicar la lógica correspondiente (MXN base para México, PEN base para Perú).
- Mientras la tolerancia Perú no esté definida, la validación del escenario TOLERANCIA puede quedar parametrizable o retornar INSUFICIENTE por defecto.

**Objetivo general:**
Extender `SaldoAsociacionCalculatorService` en Finanzas para soportar Región Perú, usando moneda base PEN y el umbral de tolerancia de Perú cuando esté definido.

**Objetivos específicos:**
- Extender `ISaldoAsociacionCalculatorService` para aceptar parámetro de `Region` ('MEX' / 'PER').
- Implementar lógica de moneda base PEN para Perú en la conversión multi-divisa.
- Parametrizar el umbral de tolerancia por región: `toleranciaMXN = 100` para México; `toleranciaPEN = [PENDIENTE]` para Perú (configurar como parámetro, no hardcodeado).
- Actualizar escenarios para Perú: EXACTO / SOBREPAGO / TOLERANCIA_PEN / INSUFICIENTE.
- Actualizar el endpoint `POST /api/validar-cobro/clientes/{idCliente}/calcular-saldo` para retornar la moneda base del escenario (MXN o PEN).

**Resultado esperado:**
`SaldoAsociacionCalculatorService` en Finanzas calcula correctamente el saldo para clientes Perú con moneda base PEN y determina el escenario usando el umbral de tolerancia de Perú (parametrizable hasta confirmar el valor).

**Entregables:**
- Extensión de `ISaldoAsociacionCalculatorService` + `SaldoAsociacionCalculatorService`
- Umbral tolerancia Perú configurado como parámetro (no hardcodeado)
- Pruebas unitarias para los 4 escenarios Perú con moneda base PEN
- Prueba de no regresión para México (moneda base MXN, tolerancia 100 MXN)

**Criterios de aceptación:**
- Para Región Perú: todos los totales del panel se expresan en moneda del cobro con base PEN.
- `Escenario = EXACTO` cuando SaldoAsociacion = 0.
- `Escenario = SOBREPAGO` cuando SaldoAsociacion > 0.
- `Escenario = TOLERANCIA_PEN` cuando 0 > SaldoAsociacion AND ABS ≤ umbral Perú (parametrizable).
- `Escenario = INSUFICIENTE` cuando SaldoAsociacion < 0 AND ABS > umbral Perú.
- El cálculo México (tolerancia 100 MXN) no se ve afectado.
- ⚠️ El valor del umbral de tolerancia Perú quedará como parámetro hasta confirmar con Tesorería.

**Más información de la tarea:**
Ver sección *"Parte B / B3"* en `TPSC-RE-FU-027-Back.md`. Ver reglas 8-11 y 19-20 en `TPSC-RE-FU-027.md`.

**Recursos:**
- `TPSC-RE-FU-027-Back.md` — Parte B, sección B3
- `TPSC-RE-FU-026-Tareas.md` — Tarea 6 (motor base a extender)
- `TPSC-RE-FU-027.md` — Reglas 8-11, 19-20

---

## TAREA 6

**[ RE-FU-027 ] [IMP-EXIST-SERVICE] Extender persistencia transaccional de la asociación para Región Perú (sin Complemento de Pago)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 Perú

**Consideraciones previas:**
- `ConfirmarAsociacionPaso2Command` + Handler fueron implementados en RE-FU-026 (Tarea 7) para México.
- Para Perú: misma lógica transaccional pero **sin disparar generación de Complemento de Pago**. La asociación es solo operativa.
- La Tarea 2 debe estar ejecutada: `fccSaldoFavorCliente` tiene campo `PEN`.
- Al insertar en `fccSaldoFavorCliente` para Perú: `PEN=1`, no `MXN=1`.
- No se dispara ningún evento hacia Timbrado al avanzar al Paso 3 desde Perú.

**Objetivo general:**
Extender en Finanzas la persistencia transaccional del Paso 2 para Región Perú: mismas operaciones que México (INSERT asociaciones, UPDATE NCs, INSERT saldo a favor/tolerancia) pero sin disparar Complemento de Pago al avanzar al Paso 3.

**Objetivos específicos:**
- Extender `ConfirmarAsociacionPaso2Command` + Handler para manejar `Region='PER'`.
- Asegurar que para Región Perú NO se dispara generación de Complemento de Pago ni evento a Timbrado.
- Al insertar `fccSaldoFavorCliente` para Perú: usar `PEN=1` (no `MXN=1`).
- Transacción completa con rollback si cualquier operación falla, igual que México.
- Registrar en Serilog con contexto (usuario, módulo='ValidarCobroPaso2PER', operación).

**Resultado esperado:**
El endpoint `POST .../confirmar-asociacion` persiste correctamente la asociación del Paso 2 para clientes Perú en una sola transacción, sin disparar Complemento de Pago.

**Entregables:**
- Extensión de `ConfirmarAsociacionPaso2Command` + Handler para Región Perú
- Pruebas unitarias (incluyendo: rollback en error, bloqueo si escenario=INSUFICIENTE, ausencia de evento Complemento de Pago para Perú)

**Criterios de aceptación:**
- Las asociaciones se insertan correctamente en `fccPagoFacturaPedido` y/o `fccPagoFacturaAdelanto` para Perú.
- Las NCs aplicadas quedan con `Aplicada=1` e `IdFCCPagoCliente` poblado.
- Si hay sobrepago Perú: `INSERT fccSaldoFavorCliente` con `TipoSaldo='SaldoFavor'` y `PEN=1`.
- Si hay tolerancia Perú: `INSERT fccSaldoFavorCliente` con `TipoSaldo='ToleranciaAplicada'` y `PEN=1`.
- Si escenario=INSUFICIENTE: el endpoint retorna error y NO persiste nada.
- Si cualquier operación falla: rollback completo.
- **Para Región Perú: NO se dispara ningún evento de Complemento de Pago.**
- El flujo México (con trigger Complemento de Pago) no se ve afectado.

**Más información de la tarea:**
Ver sección *"Parte B / B4"* en `TPSC-RE-FU-027-Back.md`. Ver reglas 9-13 en `TPSC-RE-FU-027.md`.

**Recursos:**
- `TPSC-RE-FU-027-Back.md` — Parte B, sección B4
- `TPSC-RE-FU-026-Tareas.md` — Tarea 7 (persistencia base a extender)
- `TPSC-RE-FU-027_BD.md` — Tablas escritas runtime

---

## TAREA 7

**[ RE-FU-027 ] [IMP-EXIST-SERVICE] Extender modal de inconsistencia Paso 2 para Región Perú**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 Perú

**Consideraciones previas:**
- El modal de inconsistencia del Paso 2 fue implementado en RE-FU-026 (Tarea 8) para México.
- Para Perú: misma lógica, mismo catálogo `catTipoInconsistenciaCobro` con `AplicaPaso='2'`, mismos tipos `PAGO_INCOMPLETO_VENCIDO` y `PAGO_INSUFICIENTE`.
- Para el tipo `PAGO_INCOMPLETO_VENCIDO` en Perú: el marcado del pedido como "Pendiente de cancelación" es igual que en México — NO ejecuta cancelación fiscal. En Perú la cancelación fiscal sería vía Nota de Crédito SUNAT (RE-FU-033/035).
- ⚠️ Mecanismo de transferencia del estado "Pendiente de cancelación" para gestión externa en Perú pendiente de definir.
- La extensión es mínima: verificar que el filtro por región o cartera del usuario opere correctamente para cobros Perú.

**Objetivo general:**
Verificar y extender en Finanzas el modal de inconsistencias del Paso 2 para que opere correctamente para cobros de clientes Perú, sin diferencias funcionales respecto a México excepto el contexto de región.

**Objetivos específicos:**
- Verificar que `RegistrarInconsistenciaCobroCommand` detecta correctamente cobros de Región Perú.
- Confirmar que el catálogo `catTipoInconsistenciaCobro` con `AplicaPaso='2'` aplica igual para Perú.
- Para `PAGO_INCOMPLETO_VENCIDO` Perú: misma lógica de marcado de pedido; registrar en Serilog que la cancelación fiscal efectiva se gestiona externamente vía NC SUNAT.
- Actualizar registro en `fccInconsistenciaCobro` con contexto de región Perú.

**Resultado esperado:**
El modal de inconsistencias del Paso 2 opera correctamente para cobros de clientes Perú, con el mismo comportamiento que México y el contexto de región Perú registrado.

**Entregables:**
- Verificación y extensión mínima de `RegistrarInconsistenciaCobroCommand` para Región Perú
- Pruebas unitarias para Región Perú (incluyendo PAGO_INCOMPLETO_VENCIDO y PAGO_INSUFICIENTE)
- Prueba de no regresión para México

**Criterios de aceptación:**
- El combo del modal Paso 2 Perú muestra correctamente los tipos con `AplicaPaso='2'`.
- Para `PAGO_INCOMPLETO_VENCIDO` Perú: habilita la opción de marcar el pedido (igual que México).
- Para `PAGO_INSUFICIENTE` Perú: solo registra inconsistencia, sin marcado de pedido.
- El flujo México no se ve afectado.
- ⚠️ El mecanismo de transferencia del estado para cancelación efectiva Perú queda documentado como pendiente.

**Más información de la tarea:**
Ver sección *"Parte B / B5"* en `TPSC-RE-FU-027-Back.md`. Ver reglas 12-14 en `TPSC-RE-FU-027.md`.

**Recursos:**
- `TPSC-RE-FU-027-Back.md` — Parte B, sección B5
- `TPSC-RE-FU-026-Tareas.md` — Tarea 8 (modal base a extender)
- `TPSC-RE-FU-027.md` — Reglas 12-14

---

## TAREA 8

**[ RE-FU-027 ] [IMP-EXIST-SERVICE] Extender auto-guardado del Paso 2 para Región Perú**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 2 Perú

**Consideraciones previas:**
- El auto-guardado del Paso 2 fue implementado en RE-FU-026 (Tarea 9) para México.
- Para Perú: el comportamiento es idéntico — el borrador persiste cobros seleccionados, documentos con orden, NCs aplicadas e inconsistencias marcadas.
- Verificar que la estructura de borrador (`fccAsociacionBorrador` u otro patrón definido en RE-FU-026) soporta el contexto de Región Perú sin cambios o con un campo de región.
- Las asociaciones siguen siendo editables hasta que el usuario avanza al Paso 3 (Tarea 6 ejecutada).

**Objetivo general:**
Verificar y extender en Finanzas el mecanismo de auto-guardado del Paso 2 para que soporte el contexto de Región Perú, asegurando que el borrador de la asociación se preserva y recupera correctamente para cobros de clientes Perú.

**Objetivos específicos:**
- Verificar que `AutoGuardarAsociacionCommand` soporta el contexto `Region='PER'`.
- Si la tabla/estructura de borrador no tiene campo `Region`: agregar o ajustar para distinguir borradores México/Perú.
- Verificar que al retomar el Paso 2 Perú, el estado previo se carga correctamente.
- Prueba de no regresión: borradores México no se ven afectados.

**Resultado esperado:**
El auto-guardado del Paso 2 opera correctamente para clientes Perú, preservando y recuperando el borrador de la asociación de forma transparente.

**Entregables:**
- Verificación y extensión mínima de `AutoGuardarAsociacionCommand` para Región Perú
- Pruebas unitarias para Región Perú (guardado, recuperación, sobrescritura de borrador)
- Prueba de no regresión para México

**Criterios de aceptación:**
- El borrador de la asociación Perú se persiste correctamente al navegar fuera del Paso 2.
- Al retomar el Paso 2 Perú, el estado previo se carga desde el borrador.
- El borrador puede sobrescribirse hasta que la asociación se confirme con la Tarea 6.
- Borradores de clientes México no se ven afectados.

**Más información de la tarea:**
Ver sección *"Parte B / B6"* en `TPSC-RE-FU-027-Back.md`. Ver reglas 15 y 16 en `TPSC-RE-FU-027.md`.

**Recursos:**
- `TPSC-RE-FU-027-Back.md` — Parte B, sección B6
- `TPSC-RE-FU-026-Tareas.md` — Tarea 9 (auto-guardado base a extender)
- `TPSC-RE-FU-027.md` — Reglas 15 y 16
