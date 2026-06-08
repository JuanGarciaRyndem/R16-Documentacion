# Tareas BackEnd — TPSC-RE-FU-025
**Requisito:** Validar Cobro: Paso 1 Perú — Captura del Cobro
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)

---

> **Orden de ejecución sugerido:** BD ProquifaDotNet (DML catálogos Perú — dependen de brechas) → TipoCambioPeruService → Adaptación catálogos en endpoints existentes RE-FU-024
> **Dependencias críticas:** RE-FU-024 debe estar completo (Tareas 1-10) antes de implementar la variante Perú. Las Tareas 1 y 2 de este requisito son BRECHA — su ejecución depende de que PROQUIFA proporcione los datos.

---

## Resumen de tareas

| # | Clave | Título simple | Tipo | Aplicativo | Bloqueante |
|---|-------|--------------|------|-----------|----------|
| 1 | UPDATE-TABL-CH | Insertar catálogo interno de medio de pago Perú en catMedioDePago | BD | ProquifaDotNet | ⛔ BRECHA |
| 2 | UPDATE-TABL-CH | Insertar cuentas bancarias de Golocaer S.A.C. Perú (GOLPERU) | BD | ProquifaDotNet | ⛔ BRECHA |
| 3 | ALG-COMPLX-LOGIC | Servicio — Tipo de Cambio del día automático para el cobro (Perú) | Back | ProquifaDotNet.Finanzas | ⛔ BRECHA |
| 4 | IMP-EXIST-SERVICE | Adaptar endpoints del Paso 1 (RE-FU-024) para variante Región Perú | Back | ProquifaDotNet.Finanzas | ⛔ BRECHA |

> ⛔ Todas las tareas bloqueadas por: catálogo de medio de pago Perú (Tesorería PROQUIFA), cuentas bancarias GOLPERU (RE-FU-006) y fuente del TC peruana. Las Tareas 3 y 4 solo pueden completarse cuando las Tareas 1 y 2 estén resueltas.

---

## TAREA 1

**[ RE-FU-025 ] [UPDATE-TABL-CH] Insertar catálogo interno de medio de pago Perú en catMedioDePago**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Validar Cobro Paso 1 Perú

**Consideraciones previas:**
- `catMedioDePago` ya existe en BD con el campo `ClaveFormaDePago` varchar(2) **NULLABLE**. No requiere ALTER TABLE.
- Los registros Perú se insertan con `ClaveFormaDePago = NULL` (no es catálogo SAT; es control interno de Tesorería).
- El catálogo completo está **pendiente de definición con PROQUIFA Tesorería**. Los valores propuestos (Transferencia, Depósito, Cheque, Efectivo) son sugerencia.
- SUNAT no exige declarar el medio de pago en el comprobante; el campo sirve solo para control interno.
- Sin estos datos el combo "Medio de pago" del formulario Paso 1 Perú en Finanzas queda vacío.

**Objetivo general:**
Insertar los registros del catálogo interno de medio de pago de Perú en `catMedioDePago` con `ClaveFormaDePago = NULL`, para poblar el combo del formulario del Paso 1 Perú en Finanzas.

**Objetivos específicos:**
- Ejecutar el DML de INSERT con los valores confirmados por PROQUIFA Tesorería.
- Usar claves con prefijo `PER-` para distinguir claramente los registros Perú de los México (SAT).
- Verificar que no se insertan duplicados (comprobar claves existentes antes del INSERT).
- Validar que los registros se muestran correctamente en el combo del formulario filtrado por región.

**Resultado esperado:**
`catMedioDePago` con los registros Perú insertados y disponibles para ser retornados por el endpoint de catálogo del formulario Paso 1 Perú en Finanzas.

**Entregables:**
- Script DML: `INSERT catMedioDePago` (valores finales confirmados por Tesorería)
- Script de validación (`SELECT` de registros Perú)

**Criterios de aceptación:**
- Los registros Perú existen en `catMedioDePago` con `ClaveFormaDePago = NULL`.
- Las claves tienen prefijo `PER-` para distinguirlas de las claves SAT mexicanas.
- No hay duplicados en la tabla.
- El endpoint de catálogo de Finanzas retorna correctamente estos registros filtrados por región Perú.

**Más información de la tarea:**
Ver sección *"Parte A / A1 — INSERT catMedioDePago Perú"* en `TPSC-RE-FU-025-Back.md` y sección *"Catálogo Medio de Pago Peru"* en `TPSC-RE-FU-025_BD.md`.

**Recursos:**
- `TPSC-RE-FU-025_BD.md` — Sección Catálogo Medio de Pago Peru
- `TPSC-RE-FU-025-Back.md` — Parte A, sección A1

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
Ver sección *"Parte A / A2"* en `TPSC-RE-FU-025-Back.md` y sección *"Cuentas Bancarias GOLPERU (BRECHA)"* en `TPSC-RE-FU-025_BD.md`. Ver también TPSC-RE-FU-006.

**Recursos:**
- `TPSC-RE-FU-025_BD.md` — Sección Cuentas Bancarias GOLPERU
- `TPSC-RE-FU-025-Back.md` — Parte A, sección A2
- `TPSC-RE-FU-006.md` — Catálogo de cuentas bancarias

---

## TAREA 3

**[ RE-FU-025 ] [ALG-COMPLX-LOGIC] Servicio — Tipo de Cambio del día automático para el cobro (Perú)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Paso 1 Perú

**Consideraciones previas:**
- Paralelo a `TipoCambioMexicoService` (RE-FU-024 Tarea 9) pero con moneda base PEN en lugar de MXN.
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
- Exponer `GET /api/validar-cobro/tipo-cambio-peru?monedaCobro=USD&monedaFacturacion=PEN` en Finanzas.
- Registrar en logs Serilog si la fuente del TC no está disponible.

**Resultado esperado:**
Servicio `TipoCambioPeruService` en Finanzas que retorna el TC del día para cobros Perú según la regla 8 del requisito, listo para ser integrado en el formulario del Paso 1 Perú.

**Entregables:**
- `ITipoCambioPeruService` + `TipoCambioPeruService` en Infrastructure de Finanzas
- Endpoint `GET /api/validar-cobro/tipo-cambio-peru`
- Pruebas unitarias (3 escenarios: N/A, moneda facturación, moneda cobro)

**Criterios de aceptación:**
- Si cobro=PEN y facturación=PEN: retorna `{ "tipoCambio": null, "esNA": true }`.
- Si cobro=PEN y facturación≠PEN: retorna TC del día de la moneda de facturación vs PEN.
- Si cobro≠PEN: retorna TC del día de la moneda del cobro vs PEN.
- El valor es solo lectura (no modificable por el usuario en el formulario).
- Si la fuente no está disponible, se registra en Serilog y retorna error controlado.

**Más información de la tarea:**
Ver sección *"Parte B / B1 — TipoCambioPeruService"* en `TPSC-RE-FU-025-Back.md`. Ver regla 8 y criterio D8 en `TPSC-RE-FU-025.md`.

**Recursos:**
- `TPSC-RE-FU-025-Back.md` — Parte B, sección B1
- `TPSC-RE-FU-025.md` — Regla 8 y Criterio D8

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
- Extender el endpoint de catálogo de medio de pago para filtrar por región: si región=MEX → `ClaveFormaDePago IS NOT NULL`; si región=PER → `ClaveFormaDePago IS NULL`.
- Extender el endpoint de cuentas destino para filtrar por empresa y región: si región=MEX → GOL/MUN/PRO/PQF; si región=PER → GOLPERU.
- Condicionar el servicio de TC del día: si región=MEX → `TipoCambioMexicoService`; si región=PER → `TipoCambioPeruService`.
- Validar que la cabecera del cliente retorna la etiqueta "RUC" para clientes Perú.
- Verificar que los demás endpoints del Paso 1 (listado cobros, detalle correo, auto-guardado, confirmar, inconsistencias) funcionan sin cambios para cobros Perú (usan las mismas tablas `fccPagoCliente`, `fccInconsistenciaCobro`, `SeqFolioCobro`).

**Resultado esperado:**
Los endpoints del Paso 1 detectan la región del cliente y sirven los catálogos correctos (Perú o México) sin duplicar lógica. El Paso 1 Perú funciona con la misma UI que México pero con sus catálogos propios.

**Entregables:**
- Modificación de handlers/servicios de catálogo (medio de pago, cuentas destino, TC) para soportar región como parámetro
- Pruebas unitarias de la bifurcación por región (MEX vs PER)

**Criterios de aceptación:**
- Para clientes Perú, el combo "Medio de pago" retorna solo registros con `ClaveFormaDePago IS NULL`.
- Para clientes Perú, el combo "Cuenta destino" retorna solo cuentas GOLPERU.
- Para clientes Perú, el TC del día usa `TipoCambioPeruService` (base PEN).
- La cabecera del cliente Perú muestra etiqueta "RUC".
- Para clientes México, los catálogos siguen funcionando igual que en RE-FU-024.

**Más información de la tarea:**
Ver sección *"Parte B / B2 — Adaptación de catálogos"* en `TPSC-RE-FU-025-Back.md`. Ver sección *"Diferencias MEX vs PER en BD"* en `TPSC-RE-FU-025_BD.md`.

**Recursos:**
- `TPSC-RE-FU-025-Back.md` — Parte B, sección B2
- `TPSC-RE-FU-025_BD.md` — Diferencias MEX vs PER
- `TPSC-RE-FU-024-Tareas.md` — Tareas 5-10 (endpoints base a extender)
