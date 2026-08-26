# Tareas BackEnd — R16A-RE-FU-025
**Requisito:** Validar Cobro: Paso 1 Perú — Captura del Cobro
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)

---

> **Orden de ejecución sugerido:** BD ProquifaDotNet (DML catálogos Perú — dependen de brechas) → TipoCambioPeruService → Adaptación catálogos en endpoints existentes RE-FU-024
> **Dependencias críticas:** RE-FU-024 debe estar completo (Tareas 1-10) antes de implementar la variante Perú. Las Tareas 1 y 2 de este requisito son BRECHA — su ejecución depende de que PROQUIFA proporcione los datos.

---

## Resumen de tareas

| # | Clave | Título simple | Tipo | Aplicativo | Bloqueante |
|---|-------|--------------|------|-----------|----------|
| 1 | UPDATE-TABL-CH | Insertar catálogo interno de medio de pago Perú en catMedioDePago | BD | ProquifaDotNet | ✅ RESUELTA (2026-08-24) |
| 2 | UPDATE-TABL-CH | Insertar cuentas bancarias de Golocaer S.A.C. Perú (GOLPERU) | BD | ProquifaDotNet | ⛔ BRECHA |
| 3 | ALG-COMPLX-LOGIC | Servicio — Tipo de Cambio del día automático para el cobro (Perú) | Back | ProquifaDotNet.Finanzas | ⛔ BRECHA |
| 4 | IMP-EXIST-SERVICE | Adaptar endpoints del Paso 1 (RE-FU-024) para variante Región Perú | Back | ProquifaDotNet.Finanzas | ⛔ BRECHA |

> **✅ Actualización 2026-08-24:** la Tarea 1 (catálogo `catMedioDePago` Perú) quedó **resuelta** — el cliente confirmó que el catálogo definitivo ya está cargado en BD. Ver el detalle actualizado más abajo en esta misma Tarea, y en `R16A-RE-FU-025_BD.md` / `R16A-RE-FU-025-Back.md`.
>
> ⚠️ **Nota adicional (DIS-SOL RE-025, 22/07/2026 — ver `R16A-RE-FU-025-Back.md`):** el DIS-SOL descartó el `TipoCambioPeruService` mencionado en las Tareas 3 y 4 de más abajo (decisión DP3) — el TC de Perú se resuelve con el mismo `ExchangeRateService` de RE-024, que ya determina la moneda base según la región del usuario. Las Tareas 3 y 4 de este documento describen un enfoque anterior al DIS-SOL y deben releerse junto con `R16A-RE-FU-025-Back.md` antes de ejecutarse.
>
> ⛔ Las Tareas 2, 3 y 4 siguen bloqueadas por: cuentas bancarias GOLPERU (RE-FU-006) y fuente del TC peruana. La Tarea 4 depende además de que la Tarea 3 esté resuelta.

---

## TAREA 1

**[ RE-FU-025 ] [UPDATE-TABL-CH] Insertar catálogo interno de medio de pago Perú en catMedioDePago**

**✅ RESUELTA (2026-08-24)** — ~~⛔ BRECHA~~

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 1 Perú

**Consideraciones previas (histórico — supuestos previos a la resolución):**
~~- `catMedioDePago` ya existe en BD con el campo `ClaveFormaDePago` varchar(2) **NULLABLE**. No requiere ALTER TABLE.
- Los registros Perú se insertan con `ClaveFormaDePago = NULL` (no es catálogo SAT; es control interno de Tesorería).
- El catálogo completo está **pendiente de definición con PROQUIFA Tesorería**. Los valores propuestos (Transferencia, Depósito, Cheque, Efectivo) son sugerencia.
- SUNAT no exige declarar el medio de pago en el comprobante; el campo sirve solo para control interno.
- Sin estos datos el combo "Medio de pago" del formulario Paso 1 Perú en Finanzas queda vacío.~~

**✅ Actualización 2026-08-24 — catálogo definitivo recibido y cargado:**
- El cliente confirmó (2026-08-24) que el catálogo ya está **actualizado/cargado en `catMedioDePago`** para la región Perú, con 4 registros: Depósito en cuenta (`001`), Transferencia de fondos (`003`), Tarjeta de débito (`005`), Otros medios de pago (`999`).
- ⚠️ **Corrección respecto al supuesto original:** `ClaveFormaDePago` **NO quedó NULL** para los registros Perú — tiene valores (`001`/`003`/`005`/`999`), pero son **códigos internos de 3 dígitos**, no equivalentes al catálogo SAT c_FormaPago de 2 dígitos usado en México. El discriminador correcto entre México y Perú es `IdRegion`, no "`ClaveFormaDePago` NULL vs NOT NULL" (ver Tarea 4, que debe actualizarse en consecuencia).
- ⚠️ **Corrección respecto al supuesto original:** no se usó prefijo `PER-` en las claves; las claves reales (`001`, `003`, `005`, `999`) siguen el mismo formato numérico usado en México, distinguiéndose únicamente por `IdRegion`.
- Los 4 registros tienen `ObligatorioEnCliente = 0`, lo que confirma en BD la resolución de DUDA-076 (medio de pago NO obligatorio en Perú).
- El listado de valores ya no incluye "Cheque" ni "Efectivo" (que eran sugerencia inicial) sino "Tarjeta de débito" y "Otros medios de pago".

**Objetivo general (cumplido):**
Insertar los registros del catálogo interno de medio de pago de Perú en `catMedioDePago`, para poblar el combo del formulario del Paso 1 Perú en Finanzas. El cliente reporta que la carga ya se realizó en el sistema.

**Objetivos específicos:**
- ~~Ejecutar el DML de INSERT con los valores confirmados por PROQUIFA Tesorería.~~ ✅ Completado por el cliente — ver script documentado en `R16A-RE-FU-025_BD.md`.
- ~~Usar claves con prefijo `PER-` para distinguir claramente los registros Perú de los México (SAT).~~ ❌ No aplica — la distinción real es por `IdRegion`, no por prefijo de clave.
- Verificar que no existan duplicados (comprobar claves/GUIDs existentes) — **pendiente de confirmación técnica**, ya que la carga fue reportada por el cliente, no ejecutada por el equipo.
- Validar que los registros se muestran correctamente en el combo del formulario filtrado por región.

**Resultado esperado:**
`catMedioDePago` con los 4 registros Perú insertados y disponibles para ser retornados por el endpoint de catálogo del formulario Paso 1 Perú en Finanzas.

**Entregables:**
- ~~Script DML: `INSERT catMedioDePago` (valores finales confirmados por Tesorería)~~ — script documentado post-hoc en `R16A-RE-FU-025_BD.md` (para trazabilidad; la carga ya fue reportada como hecha por el cliente).
- Script de validación (`SELECT` de registros Perú) — **pendiente de ejecutar** para confirmar contra el esquema real.

**Criterios de aceptación:**
- Los registros Perú existen en `catMedioDePago` — ✅ (4 registros: Depósito en cuenta, Transferencia de fondos, Tarjeta de débito, Otros medios de pago).
- ~~Las claves tienen prefijo `PER-` para distinguirlas de las claves SAT mexicanas.~~ ❌ Ya no aplica — se distinguen por `IdRegion`.
- No hay duplicados en la tabla — **pendiente de verificar**.
- El endpoint de catálogo de Finanzas retorna correctamente estos registros filtrados por región Perú — pendiente de implementar/validar en la Tarea 4.

**Más información de la tarea:**
Ver sección *"Parte A / A1 — INSERT catMedioDePago Perú"* en `R16A-RE-FU-025-Back.md` y sección *"Catálogo Medio de Pago Peru"* en `R16A-RE-FU-025_BD.md`.

**Recursos:**
- `R16A-RE-FU-025_BD.md` — Sección Catálogo Medio de Pago Peru
- `R16A-RE-FU-025-Back.md` — Parte A, sección A1

---

## TAREA 2

**[ RE-FU-025 ] [UPDATE-TABL-CH] Insertar cuentas bancarias de Golocaer S.A.C. Perú (GOLPERU)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 1 Perú

**Consideraciones previas:**
- `EmpresaDatosBancarios` para GOLPERU tiene **0 registros** actualmente en BD.
- Se requieren datos de PROQUIFA Perú: banco(s) peruano(s) de Golocaer S.A.C., números de cuenta CCI (20 dígitos), monedas PEN y/o USD.
- La Cuenta de Compensación Interbancaria (CCI) de Perú tiene 20 dígitos (equivalente a la CLABE mexicana de 18 dígitos). El campo `Clabe` en `DatosBancarios` se usa para el CCI.
- Verificar si `catMoneda` ya tiene PEN; si no, insertar.
- Si los bancos peruanos de Golocaer S.A.C. no están en `catBanco`, insertarlos.
- Bloqueante para el combo "Cuenta destino" del formulario Paso 1 Perú en Finanzas (ver RE-FU-006).

**Objetivo general:**
Poblar `catBanco`, `DatosBancarios` y `EmpresaDatosBancarios` con las cuentas bancarias de Golocaer S.A.C. Perú, para habilitar el combo "Cuenta destino" del formulario del Paso 1 Perú en Finanzas.

**Objetivos específicos:**
- Verificar existencia de `catMoneda` PEN; si no existe, ejecutar `INSERT catMoneda` para PEN.
- Insertar banco(s) peruano(s) de Golocaer S.A.C. en `catBanco` si no existen.
- Insertar en `DatosBancarios`: `NumeroDeCuenta`, `Clabe` (CCI 20 dígitos), `Sucursal`, `IdCatBanco`, `IdCatMoneda` (PEN / USD según cuentas disponibles).
- Insertar en `EmpresaDatosBancarios`: `IdEmpresa` → GOLPERU, `IdRegion` → PER, `IdDatosBancarios`.
- Validar que el combo de Finanzas retorna estas cuentas al filtrar por empresa GOLPERU y región PER.

**Resultado esperado:**
`EmpresaDatosBancarios` con al menos una cuenta de Golocaer S.A.C. Perú registrada y activa, disponible para ser retornada por el endpoint de catálogo del formulario Paso 1 Perú en Finanzas.

**Entregables:**
- Script DML: `INSERT catMoneda PEN` (si aplica) + `INSERT catBanco` (bancos GOLPERU) + `INSERT DatosBancarios` + `INSERT EmpresaDatosBancarios`
- Script de validación (`SELECT` de cuentas GOLPERU)

**Criterios de aceptación:**
- `EmpresaDatosBancarios` tiene al menos 1 registro activo para GOLPERU / región PER.
- `DatosBancarios` incluye `Clabe` (CCI) de 20 dígitos.
- `catMoneda` tiene el registro PEN.
- El endpoint de catálogo de cuentas destino de Finanzas retorna las cuentas GOLPERU filtradas por región PER.

**Más información de la tarea:**
Ver sección *"Parte A / A2"* en `R16A-RE-FU-025-Back.md` y sección *"Cuentas Bancarias GOLPERU (BRECHA)"* en `R16A-RE-FU-025_BD.md`. Ver también R16A-RE-FU-006.

**Recursos:**
- `R16A-RE-FU-025_BD.md` — Sección Cuentas Bancarias GOLPERU
- `R16A-RE-FU-025-Back.md` — Parte A, sección A2
- `R16A-RE-FU-006.md` — Catálogo de cuentas bancarias

---

## TAREA 3

**[ RE-FU-025 ] [ALG-COMPLX-LOGIC] Servicio — Tipo de Cambio del día automático para el cobro (Perú)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 Perú

**Consideraciones previas:**
- Paralelo a `TipoCambioService` (RE-FU-024 Tarea 9) pero con moneda base PEN en lugar de MXN.
- La fuente oficial del TC para Perú **no está definida**. No aplica el TC FIX Banxico/DOF mexicano.
- Sin la fuente del TC, este servicio no puede implementarse completamente. El campo se mostrará como N/A hasta que se resuelva la brecha.
- El TC capturado con el cobro se persiste en `fccPagoCliente.TipoDeCambio` y se reutiliza en el Paso 2 (conversiones operativas) y, si aplica, en el Paso 3.
- Lógica: si cobro=PEN y facturación=PEN → N/A; si cobro=PEN y facturación≠PEN → TC de la moneda de facturación vs PEN; si cobro≠PEN → TC de la moneda del cobro vs PEN.

**Objetivo general:**
Implementar en Finanzas el servicio `TipoCambioPeruService` que calcula el TC del día de la moneda no-PEN involucrada en el cobro Perú, con la misma lógica estructural que México pero con moneda base PEN y fuente peruana.

**Objetivos específicos:**
- Crear `ITipoCambioPeruService` con método `ObtenerTCDelDiaAsync(ClaveMonedaCobro, ClaveMonedaFacturacion)`.
- Implementar la lógica de selección de moneda (cobro vs facturación vs N/A) con base PEN.
- Integrar con la fuente del TC peruana (pendiente de confirmar con PROQUIFA).
- Exponer `GET /api/v1/validate-collection/exchangeRate?monedaCobro=USD&monedaFacturacion=PEN` en Finanzas.
- Registrar en logs Serilog si la fuente del TC no está disponible.

**Resultado esperado:**
Servicio `TipoCambioPeruService` en Finanzas que retorna el TC del día para cobros Perú según la regla 8 del requisito, listo para ser integrado en el formulario del Paso 1 Perú.

**Entregables:**
- `ITipoCambioPeruService` + `TipoCambioPeruService` en Infrastructure de Finanzas
- Endpoint `GET /api/v1/validate-collection/exchangeRate`
- Pruebas unitarias (3 escenarios: N/A, moneda facturación, moneda cobro)

**Criterios de aceptación:**
- Si cobro=PEN y facturación=PEN: retorna `{ "tipoCambio": null, "esNA": true }`.
- Si cobro=PEN y facturación≠PEN: retorna TC del día de la moneda de facturación vs PEN.
- Si cobro≠PEN: retorna TC del día de la moneda del cobro vs PEN.
- El valor es solo lectura (no modificable por el usuario en el formulario).
- Si la fuente no está disponible, se registra en Serilog y retorna error controlado.

**Más información de la tarea:**
Ver sección *"Parte B / B1 — TipoCambioPeruService"* en `R16A-RE-FU-025-Back.md`. Ver regla 8 y criterio D8 en `R16A-RE-FU-025.md`.

**Recursos:**
- `R16A-RE-FU-025-Back.md` — Parte B, sección B1
- `R16A-RE-FU-025.md` — Regla 8 y Criterio D8

---

## TAREA 4

**[ RE-FU-025 ] [IMP-EXIST-SERVICE] Adaptar endpoints del Paso 1 (RE-FU-024) para variante Región Perú**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 Perú

**Consideraciones previas:**
- Los endpoints del Paso 1 ya implementados en RE-FU-024 (Tareas 5-10) deben retornar catálogos distintos según la región del cliente.
- Las Tareas 1 y 2 de este requisito deben estar completas (catálogos Perú en BD disponibles).
- Los cambios son mínimos: diferenciar catálogos por región en los endpoints de combos (medio de pago, cuentas destino, TC).
- La cabecera del cliente muestra "RUC" en lugar de "RFC" para clientes Perú (el valor ya está en `DatosFacturacionCliente.RFC`; solo cambia la etiqueta en el Front).
- No se duplican endpoints: los mismos del Paso 1 detectan la región del cliente y retornan el catálogo correspondiente.
- La Tarea 3 (TipoCambioPeruService) debe estar completa para integrar el TC Perú.

**Objetivo general:**
Extender los endpoints del Paso 1 implementados en RE-FU-024 para soportar la variante Perú: retornar el catálogo interno de medio de pago Perú, las cuentas GOLPERU como cuenta destino y usar `TipoCambioPeruService` para el TC del día, todo condicionado a la región del cliente.

**Objetivos específicos:**
- Extender el endpoint de catálogo de medio de pago para filtrar por región usando **`IdRegion`** (⚠️ actualizado 2026-08-24: el filtro NO debe basarse en `ClaveFormaDePago IS NULL/IS NOT NULL` — desde la carga del catálogo Perú, `ClaveFormaDePago` tiene valores en ambas regiones (`001`/`003`/`005`/`999` en Perú, códigos SAT de 2 dígitos en México); el discriminador correcto es `IdRegion`).
- Extender el endpoint de cuentas destino para filtrar por empresa y región: si región=MEX → GOL/MUN/PRO/PQF; si región=PER → GOLPERU.
- Condicionar el servicio de TC del día: si región=MEX → `TipoCambioService`; si región=PER → `TipoCambioPeruService`.
- Validar que la cabecera del cliente retorna la etiqueta "RUC" para clientes Perú.
- Verificar que los demás endpoints del Paso 1 (listado cobros, detalle correo, auto-guardado, confirmar, inconsistencias) funcionan sin cambios para cobros Perú (usan las mismas tablas `fccPagoCliente`, `fccInconsistenciaCobro`, `SeqFolioCobro`).

**Resultado esperado:**
Los endpoints del Paso 1 detectan la región del cliente y sirven los catálogos correctos (Perú o México) sin duplicar lógica. El Paso 1 Perú funciona con la misma UI que México pero con sus catálogos propios.

**Entregables:**
- Modificación de handlers/servicios de catálogo (medio de pago, cuentas destino, TC) para soportar región como parámetro
- Pruebas unitarias de la bifurcación por región (MEX vs PER)

**Criterios de aceptación:**
- Para clientes Perú, el combo "Medio de pago" retorna solo registros con `IdRegion` = Perú (⚠️ actualizado 2026-08-24 — ya no se filtra por `ClaveFormaDePago IS NULL`, ver nota arriba).
- Para clientes Perú, el combo "Cuenta destino" retorna solo cuentas GOLPERU.
- Para clientes Perú, el TC del día usa `TipoCambioPeruService` (base PEN).
- La cabecera del cliente Perú muestra etiqueta "RUC".
- Para clientes México, los catálogos siguen funcionando igual que en RE-FU-024.

**Más información de la tarea:**
Ver sección *"Parte B / B2 — Adaptación de catálogos"* en `R16A-RE-FU-025-Back.md`. Ver sección *"Diferencias MEX vs PER en BD"* en `R16A-RE-FU-025_BD.md`.

**Recursos:**
- `R16A-RE-FU-025-Back.md` — Parte B, sección B2
- `R16A-RE-FU-025_BD.md` — Diferencias MEX vs PER
- `R16A-RE-FU-024-Tareas.md` — Tareas 5-10 (endpoints base a extender)
