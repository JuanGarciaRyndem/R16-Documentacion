# Revisión — [R16A-RE-FU-026][DIS-SOL] Diseño de la solución

**Documento revisado:** [Diseño de la solución](https://docs.google.com/document/d/1xAvNSwjJk4cuK7k5sdeZml7595GCbJm75bhLxOhD4Sk/edit?usp=drive_link) — v1.3, 12/08/2026, autor Isai Amaury Garcia Flores, revisor Valdemar Farina Sánchez.
**Fecha de revisión:** 2026-08-26
**Revisor:** Juan Garcia (con asistencia de Claude)
**Contexto considerado:** requisito funcional y análisis de impacto de esta misma carpeta (`R16A-RE-FU-026.md`, `-Back.md`, `_BD.md`) y el requisito del cual depende, `R16A-RE-FU-024` (Paso 1), incluyendo su DIS-SOL v1.0 y su archivo de pendientes.

Los comentarios están redactados para pegarse directamente como comentarios de Google Docs: cada uno indica el texto exacto a seleccionar en el documento antes de anclar el comentario.

---

## 🔴 Críticos (bloqueantes antes de construir)

### 1. Referencia a tabla extinta: `tpProformaAdelanto`

📍 **Seleccionar** (aparece en Flujo 1 y en S1): `SELECT tpProformaAdelanto WHERE IdCliente=@id`

💬 **Comentario:**
> `tpProformaAdelanto` fue migrada/retirada el 06/07/2026 según `R16A-RE-FU-026_BD.md` de esta misma carpeta: *"la columna IdTPProformaAdelanto (FK a la extinta tpProformaAdelanto) se retarget a IdFccFactura (FK a fccFactura, RE-FU-015)"*. Este diseño es de fecha posterior (v1.3, 12/08/2026) y sigue consultando la tabla vieja — hay que cambiar a `vfccFactura`/`fccFactura` (RE-FU-015) en el query, el erDiagram y las FK de `fccPagoFacturaAdelanto`/`fccNotaCreditoAdelanto`.

### 2. ¿Tabla nueva o ALTER de una existente?: `fccPagoFacturaPedido`

📍 **Seleccionar:** `fccPagoFacturaPedido | Nueva tabla (este requisito) | Asociación cobro↔proforma. FK IdFCCCobroCliente directa a fccCobroCliente. Ver DDL T5`

💬 **Comentario:**
> El DIS-SOL de RE-FU-024 (decisión D1) dice que al fusionar `fccPagoCliente`→`fccCobroCliente`, *"las FK dependientes (fccSaldoFavorCliente, **fccPagoFacturaPedido**, fccInconsistenciaCobro) deben actualizarse"* — lenguaje que asume que la tabla ya existe y solo se le ajusta la FK, no que se crea desde cero. Aquí se marca como tabla nueva con columnas distintas (`MontoAplicado` en vez de `Monto`, sin `NumeroDeParcialidad`/`MontoPendienteAnterior`). Confirmar con el dueño de RE-024 si realmente ya existe físicamente en BD antes de escribir el DDL — un `CREATE TABLE` sobre una tabla existente fallaría.

---

## 🟠 Importantes

### 3. Contradicción sobre el alcance del Modal de Inconsistencia

📍 **Seleccionar:** `Modal de Inconsistencia y registro de inconsistencias del Paso 2 — contemplado en R16A-RE-FU-024 (diseño aún no realizado; endpoint y lógica de fccInconsistenciaCobro / catTipoInconsistenciaCobro se definirán allí)`

💬 **Comentario:**
> Esto dice que el registro de inconsistencias NO es parte de este diseño (y el Flujo 4 lo repite: "Este flujo no se implementa en este requisito"). Pero el servicio **S4** más adelante lo implementa completo (validación, soft-delete de reedición, INSERT en fccInconsistenciaCobro, transición de estatus, marcado de tpPedido). ¿S4 es entregable de R026 o debería moverse a R024? Aclarar el ownership antes de repartir tareas de construcción.

### 4. Unificación de TC entre cobros de distinta moneda

📍 **Seleccionar:** `se usa el TC del primer cobro seleccionado como TC unificador de la sesión; los demás cobros se convierten a la moneda del primer cobro usando ese mismo TC`

💬 **Comentario:**
> Si los cobros seleccionados están en más de una moneda extranjera (ej. Cobro1 en USD, Cobro2 en EUR), aplicar el TC de Cobro1 (USD/MXN) sobre el monto de Cobro2 no tiene sentido dimensional. Incluso con solo USD/MXN, usar el TC de un cobro para convertir el monto de *otro* cobro (capturado en fecha distinta) contradice el criterio ya sentado en OBS-052 de RE-FU-024: cada pago se cubre con su propio TC del día de pago, no con el de un pago ajeno. Falta acotar esta regla o definir el tratamiento por pares de moneda.

### 5. Convención de rutas no sigue el estándar kebab-case

📍 **Seleccionar:** `/api/v1/validate-collection/client/{idCliente}/pendingDocument`

💬 **Comentario:**
> RE-FU-024 normalizó explícitamente sus rutas a kebab-case en inglés (`wizard-status`, `inconsistency-type`, `exchange-rate`), corrigiendo el camelCase que tenía el KB anterior. Estas rutas (`pendingDocument`, `balance/calculate`, `associationConfirmation`, `association/draft`) no siguen esa convención — alinear antes de implementar.

---

## 🟡 Menores / a verificar

### 6. Comentario SQL referencia un estatus descartado

📍 **Seleccionar:** `TIMBRADO+/COMPLETADO/CANCELADO = inmutable`

💬 **Comentario:**
> RE-FU-024 (decisión D2b) descartó explícitamente crear un estatus `TIMBRADO` — el catálogo final `catCobroEstatus` tiene 7 claves sin `TIMBRADO`. Este comentario referencia un estatus que no existe; puede confundir a quien lea el código después. Ajustar el comentario a los estatus reales (COMPLETADO/CANCELADO/CON_INCONSISTENCIA/SALDO_A_FAVOR).

### 7. Campos de `tpPedido` asumidos como existentes

📍 **Seleccionar:** `FechaCancelacionPorFaltaPago/IdUsuarioCancelacion ya están en el modelo base (ProquifaDotNet)`

💬 **Comentario:**
> El KB anterior de este mismo requisito (`R16A-RE-FU-026-Back.md`, sección B5) marcaba este campo como *"pendiente de confirmar en BD"*, no como confirmado. Verificar contra el esquema real de ProquifaDotNet antes de construir sobre este supuesto.

### 8. Mezcla de campo nuevo (R032) y bit legacy en `fccNotaCredito`

📍 **Seleccionar** (primera ocurrencia, en la lógica de S1): `nc.Estado='VIGENTE' AND nc.Aplicada=0`

💬 **Comentario:**
> Se combina `Estado='VIGENTE'` (campo nuevo que introduce R032) con `Aplicada=0` (bit legacy). Confirmar con el dueño de R032 que ambos campos van a coexistir en el esquema final antes de fijar este query como definitivo.

---

## ✅ Lo que está bien resuelto (no requiere comentario, solo referencia)

- Adopción correcta de `fccCobroCliente`/`catCobroEstatus` del DIS-SOL de RE-024 (a diferencia de los `.md` locales de esta carpeta, que quedaron desactualizados con el esquema viejo `fccPagoCliente`).
- Corrige el error histórico documentado en los pendientes de RE-024: la FK de `fccSaldoFavorCliente` ahora apunta correctamente a `fccCobroCliente`.
- Tolerancia parametrizada vía `appsettings.json` (`Finanzas:ToleranciaValidarCobroMXN`) en vez de hardcodear 100 MXN.
- Transacción ACID con rollback completo, guardas defensivas contra condiciones de carrera (`MontoPendiente >= @montoAplicado`), y gate fiscal fuera de la transacción (post-COMMIT).
- Tabla de criterios de aceptación exhaustiva y trazable a los endpoints/SQL concretos.
