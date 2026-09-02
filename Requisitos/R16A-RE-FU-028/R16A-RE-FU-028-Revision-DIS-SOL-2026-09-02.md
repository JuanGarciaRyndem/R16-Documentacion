# Revisión DIS-SOL R16A-RE-FU-028 — 2026-09-02

**Documento revisado:** [R16A-RE-FU-028][DIS-SOL] Diseño de la solución — Google Docs, v1.4 (31-ago-2026)
Enlace: https://docs.google.com/document/d/1S_FzMndECy06361dG7l3eYm4w8u6ruLLk9xZIUtxapo/edit?tab=t.0
**Autor:** A. Javier Antúnez Estrada
**Revisor:** Juan David García Cruz
**Revisado por:** Claude (Cowork), a petición de Juan David García Cruz

## Valoración general

Documento sólido y con buena disciplina de trazabilidad: cada bloqueante resuelto (B1, B3-B6, B8) está justificado con su fuente (código, BD viva, confirmación directa con Juan David, DUDA-088). El propio autor ya marca en amarillo lo que falta (B9 y B10). Los comentarios de abajo se concentran en lo que aún no está cubierto.

## 1. Falta la integración con ProquifaDotNet.BitacoraCambios (hallazgo nuevo)

"Diseño y Desarrollo/Reglas al diseñar.md" (regla 8) exige que procesos como *"validar un cobro"* — el wizard de este requisito — registren en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo). El propio RE-FU-019, que este DIS dice seguir como patrón para Fase 1/Fase 2 (RT-18), sí lo hace explícitamente en su Back.md:

> "Registrar el guardado de la factura en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo — regla 8)"

Y los análisis previos de este mismo requisito ya lo contemplaban:
- `R16A-RE-FU-028-Back.md`, línea 273: "Registrar la validación de cobro en ProquifaDotNet.BitacoraCambios (Aplicativo Nuevo — Reglas al diseñar, regla 8)"
- `R16A-RE-FU-028-Tareas.md`, líneas 1176 y 1199: mismo registro esperado al enviar (`SendAsync`).

En el DIS-SOL actual (v1.4) no aparece ninguna mención — ni en Componentes involucrados, ni en Flujo 3/4, ni en Interfaces externas consumidas, ni en Impacto Técnico. Parece haberse perdido entre v1.0 y v1.4, no una decisión consciente de excluirlo.

**Acción sugerida:** confirmar con Javier si fue intencional. Si no, agregar el registro en Bitácora como parte de Fase 2 del Flujo 3 (timbrado) y/o Flujo 4 (envío), con su entrada correspondiente en Componentes involucrados e Interfaces externas consumidas.

## 2. B9 y B10 — de acuerdo en que bloquean, con una precisión de secuencia

Ambos bloqueantes son reales. Para **B10** (falla al invocar Logística tras cerrar el wizard) se recomienda resolverlo *antes* de construir el Flujo 4, no en paralelo: en ese punto el wizard ya cerró y los comprobantes ya están timbrados y enviados ante el SAT — es un estado irreversible, así que documentar el riesgo y seguir no es tan barato como en otros flujos reversibles del sistema.

## 3. RT-06 sin número de bloqueante ni dueño (RT-07 resuelta)

El banner de advertencia las mencionaba junto a B9/B10 como puntos a revisar, pero a diferencia de B1-B10 no tienen dueño asignado ni aparecen en la lista de "Bloqueantes activos".
- RT-06 (fallback de Uso CFDI cuando tpPedido y DatosFacturacionCliente son ambos NULL) afecta un campo obligatorio del CFDI. **Sigue pendiente.**
- RT-07 (si editar el Método de Pago por fila sobrescribe tpPedido.IdCatMetodoDePagoCFDI) — **RESUELTA en esta revisión (confirmado por Juan David García Cruz, 2026-09-02):** son dos datos independientes. `tpPedido.IdCatMetodoDePagoCFDI` queda congelado desde que se envía la Confirmación de Pedido al tramitar el pedido — ya no se puede modificar en ese punto. El Método de Pago de la fila del Paso 3 se ajusta al final según cómo pagó realmente el cliente, y es exclusivo de esa factura; no sobrescribe el dato del pedido. Ya se actualizó en el DIS-SOL local.

**Acción sugerida:** formalizar RT-06 como bloqueante con dueño, igual que B9/B10. RT-07 solo necesita reflejarse en la próxima versión oficial del Google Doc.

## 4. CA-EC1 (concurrencia) — riesgo alto por el contexto fiscal

Sin RowVersion, dos sesiones podrían timbrar dos CFDIs válidos para la misma fila ante el SAT — no se puede deshacer, solo corregir vía Nota de Crédito. La Estrategia de Pruebas propone "documentar el riesgo si se libera sin el candado".

**Acción sugerida:** tratarlo como condición de salida antes de producción (implementar RT-13), no como nota informativa.

## 5. Falta tarea explícita de backfill para CFDIGenerada.IdCatTipoCFDI

La columna nueva queda NULL en registros previos (Facturas Anticipo generadas antes de RE-FU-028) y el documento dice "requiere normalización posterior al alta de la columna", pero esa normalización no aparece como tarea en Impacto Técnico ni en Control de versiones — no tiene dueño ni criterio de validación.

## 6. Pregunta abierta: idempotencia del envío de correo

Si la llamada a Proquifa.Pqf2.Notificaciones tiene éxito pero la respuesta se pierde (timeout) antes de que Finanzas la reciba, ¿un reintento de `SendAsync` volvería a enviar el correo con los mismos adjuntos? El `ExternalReference` (IdFCCDocumentoFiscalCobro) es el candidato natural para deduplicar del lado de Notificaciones, pero el contrato descrito en B5 no aclara si ese servicio deduplica por esa referencia.

## Resumen de acciones sugeridas

| # | Hallazgo | Acción sugerida | Dueño propuesto |
| :- | :- | :- | :- |
| 1 | Falta integración con BitacoraCambios | Confirmar con Javier si fue intencional; si no, agregar al diseño | Javier Antúnez |
| 2 | B10 toca un flujo fiscal irreversible | Resolver antes de construir Flujo 4, no en paralelo | Diseño/Negocio |
| 3 | RT-06 sin dueño (RT-07 ya resuelta) | Formalizar RT-06 como bloqueante con número y dueño; reflejar la resolución de RT-07 en el Google Doc | Javier Antúnez |
| 4 | CA-EC1 concurrencia | Tratar como condición de salida antes de producción | Javier Antúnez |
| 5 | Backfill IdCatTipoCFDI | Agregar tarea explícita con dueño y criterio de validación | Javier Antúnez |
| 6 | Idempotencia de SendAsync | Confirmar con RE-FU-008/Notificaciones si dedupe por ExternalReference | Javier Antúnez |
