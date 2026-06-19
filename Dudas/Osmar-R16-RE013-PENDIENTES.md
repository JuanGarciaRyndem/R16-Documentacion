# R16A-RE-FU-013 — Pendientes y Decisiones por Tomar

---

## Decisión Global de Arquitectura
**Impacta: T0, T3, T4**

- [ ] Decidir entre:
  - **Opción 1** — ProquifaDotNet orquesta todo (BD + DocumentBuilder + MinIO + correo)
  - **Opción 2** — ProquifaDotNet delega a Api de Finanzas ← **bloqueante**: endpoints aún no definidos

---

## T1 — Foliador lineal global de proforma

- [x] Decidir opción de implementación:
  - ~~Opción 1 — Extender `FoliosConsecutivos<T>` + nueva tabla global~~
  - Opción 2 — `NEXT VALUE FOR` (SQL Server Sequence) 
	  - Nota: Revisar comportamiento con RollBack
	  - Opción 3 — Nueva clase `GeneradorFolioProforma` + nueva tabla ⭐ Recomendada
	  - Nota: se va integrar en la solucion de finanzas.
- [ ] **P-A** — Política del folio al cancelar la previsualización: si el ESAC cancela, el folio ya fue asignado. ¿Se conserva para el reintento o se descarta? ← Confirmar con cliente
	-No aplica cancelación porque los folios se generan hasta después de enviar.
- [ ] Confirmar formato exacto del folio (`PRF-MMDDYY-NNNN`) con cliente

---

## T3 — Previsualización obligatoria del PDF

- [ ] Decidir opción de implementación:
  - ~~Opción 1 — Reutilizar `ArchivoExportarPDFsController` (sube a MinIO siempre, genera archivos huérfanos en cancelaciones)~~
  - Opción 2 — Nuevo endpoint con `ByteArrayContent` sin MinIO ⭐ Recomendada En el Api de Finanzas opcion Seleccionaa
- [ ] **P-B** — ¿El PDF generado se almacena en MinIO durante la previsualización o se regenera al confirmar envío?
- [ ] Bloqueada por **T7** (contrato con DocumentBuilder debe estar definido primero)
Nota: El Pdf no lleva el Folio, especificar con Robert que diseño se utiliza


---

## T4 — Endpoint de envío de correo de proforma

- [ ] Confirmar qué ocurre exactamente tras el envío exitoso además de:
  - Pendiente Validar Cobro activo
  - Cierre pendiente bandeja Tramitar Pedido
  - ¿Cambios de estado en `tpPedido`? ¿Notificaciones internas?
- [ ] Confirmar asunto del correo: Regla 7 dice `"Proforma" + FolioPedidoInterno` — verificar con cliente si el folio de pedido interno es el dato correcto o si debe incluirse también el folio de proforma
	- Los folios sólo van en el asunto cuando se envían por correo.
- [ ] Bloqueada por decisión de arquitectura y por T3

---

## T5 — Validaciones regulatorias

Sin pendientes de decisión — implementación directa.

---

## T6 — Verificación operación Perú

- [ ] Confirmar en BD que existen registros `ProductoMarcaFamilia` con `Controlado = 1` para la región Perú
- [ ] Confirmar en BD que `ConfiguracionSendinBlue` tiene configuración para la región Perú

---

## T7 — Verificación integración con PDF (RE-FU-016/017)

- [ ] Confirmar contrato de datos con DocumentBuilder: modelo que recibe y lo que retorna (se asume `byte[]`)
- [ ] Confirmar compatibilidad del PDF retornado con T3 (previsualización `ByteArrayContent`)
- [ ] Confirmar que el PDF puede subirse a MinIO y adjuntarse al correo en T4
- **Bloqueante para T3 y T4** — debe resolverse primero

---

## Pendientes de Modelo de Datos

| # | Pendiente | Tarea relacionada |
|---|-----------|-------------------|
| P-A | Política del folio al cancelar previsualización | T1 |
| P-B | ¿PDF de proforma se persiste en MinIO o se regenera? | T3 |
| P-C | ReferenciaPago: ¿1 o 2 campos? — confirmar con RE-FU-006 | T0 / Modelo |
| P-D | Datos bancarios (Banca, Sucursal, Cuenta, CLABE): ¿existe tabla catálogo o se leen de otro origen? | T0 / Modelo |
| P-E | Tasa de IVA: ¿cuál es el origen/fuente de la tasa a aplicar? | T0 / Modelo |
| P-F | Campo Parciales: ¿de dónde se obtiene? | T0 / Modelo |
