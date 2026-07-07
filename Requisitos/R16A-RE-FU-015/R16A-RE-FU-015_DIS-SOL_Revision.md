# Revisión DIS-SOL — R16A-RE-FU-015

| Campo | Valor |
|---|---|
| **Documento revisado** | `[R16A-RE-FU-015][DIS-SOL] Diseño de la solución.pdf` (v1.0, 29/06/2026) |
| **Requisito contra el que se valida** | `R16A-RE-FU-015.md` — versión actualizada por matriz cliente (post "Actualizar respecto al requisito por ahora no al diseño") |
| **Autor** | Osmar Calderón Vázquez |
| **Revisor** | Juan David García Cruz |
| **Fecha de revisión** | 02 jul 2026 |
| **Estatus** | ⚠️ Diseño adoptado en `Back.md`/`_BD.md`/`Tareas.md` (06/07/2026) — H-01 (Perú) sigue abierto como gap documentado (GAP-08 / Riesgo 1), no bloquea el desarrollo de México |

---

## Resumen

El DIS-SOL de 015 ya está alineado con la actualización de alcance del requisito: confirma explícitamente que **no se genera proforma, PDF ni correo** (RT-01), coincide con el "No aplica a" del requisito, y reutiliza el patrón de arquitectura ya validado en 013 (ProquifaDotNet como orquestador que genera y commitea `FolioPedidoInterno` antes de delegar a `ProquifaDotNetFinanzas`). También decide **no reutilizar `tpProformaAdelanto`** y en su lugar modela tres tablas nuevas (`fccFactura`, `fccFacturaPartida`, `fccFacturaReferenciaBancaria`) — una decisión de diseño legítima que, por instrucción tuya, no se adoptó todavía en `Back.md`/`_BD.md`/`Tareas.md`.

El hallazgo crítico es que el modelo de datos (`fccFactura`) **solo contempla campos fiscales de México** (Uso CFDI, Método de Pago, Régimen Fiscal — catálogos SAT) y no incluye los campos fiscales que la Regla 14 / Criterio A5 del requisito exigen para Perú (Tipo de Operación catálogo 51 SUNAT, Condición de Pago SUNAT). Dado que el propio alcance del DIS-SOL declara operación en México **y Perú**, esto deja sin soporte de datos la mitad regional del requisito.

---

## Hallazgos críticos (bloqueantes)

### H-01 — `fccFactura` no modela los campos fiscales de Perú requeridos por Regla 14 / Criterio A5

**Sección del PDF:** Diseño de Modelo de Datos → 1. Nuevas tablas → `fccFactura` (cabecera)

**Lo que dice el diseño:** El bloque "Datos del receptor (snapshot de `DatosFacturacionCliente`)" de `fccFactura` incluye únicamente: `RfcReceptor`, `RazonSocialReceptor`, `CodigoPostalReceptor`, `RegimenFiscalClaveSAT/Leyenda`, `UsoCFDIClaveSAT/Leyenda`, `MetodoDePagoClaveSAT/Leyenda`, `FormaDePagoClaveSAT/Leyenda` — todos catálogos SAT (México).

**Lo que dice el requisito:** Regla 14 establece que el panel de Información de Facturación es regionalizado: para México se fijan Uso CFDI y Método de Pago; **para Perú esos campos se reemplazan por Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago SUNAT (Contado/Crédito)** — un campo fiscal distinto de las Condiciones de Pago comerciales. El Criterio A5 exige explícitamente que el sistema muestre estos campos peruanos "en su lugar". El propio Alcance del requisito ("Aplica a") y el Alcance del DIS-SOL ("Operación en México y Perú... flujo idéntico en ambas regiones para este módulo") confirman que 015 debe operar en ambos países.

**Impacto:** Tal como está modelada, `fccFactura` no tiene dónde persistir Tipo de Operación SUNAT ni Condición de Pago SUNAT para un pedido peruano con FAA. Si el ESAC activa FAA en un pedido de Perú, el sistema no tendría los campos fiscales correctos fijados en el pendiente, lo que rompe tanto el Criterio A5 (los datos deben "fijarse del catálogo vigente") como la futura emisión del CFDI/comprobante peruano.

**Acción:** Antes de iniciar desarrollo, agregar a `fccFactura` (o a una tabla de extensión regional) los campos equivalentes a Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago SUNAT, siguiendo el mismo patrón snapshot que los campos SAT de México. Alternativamente, si existe una razón técnica para diferir esto (p. ej. Perú no vive en 015 sino en el catálogo de cliente y no requiere persistirse en la FAA), debe documentarse explícitamente esa decisión en el DIS-SOL para no dejarlo como una omisión.

---

## Brechas (no bloqueantes)

### H-02 — IDs de Criterios de Aceptación del DIS-SOL no corresponden a los IDs oficiales del requisito

**Sección del PDF:** Diseño funcional detallado → 2. Criterios de aceptación del requisito

El DIS-SOL usa su propio esquema de IDs (A1-A3, B1-B3, C1-C2) que no coincide con los IDs oficiales de `R16A-RE-FU-015.md` (Sección A: A1-A5; Sección C: C1-C4 — proforma, ver H-03; Sección D: D1-D5). Mapeo inferido:

| DIS-SOL | Requisito | Coincide |
|---|---|---|
| A1 | Criterio A1 | Sí (con matiz, ver nota de contexto abajo) |
| A2 | Criterio A2 | Sí |
| A3 | Criterio A4 | Sí |
| B1 | (cubierto por Regla 7 / Alcance, sin Criterio propio) | — |
| B2 | Criterio D1 | Sí |
| B3 | Criterio A3 | Sí |
| C1 | Criterio D3 | Sí (bloqueado por OBS-027) |
| C2 | Criterio D2 | Sí |
| — | Criterio A5 | No cubierto (ver H-01) |
| — | Criterio D4 (cancelación) | No mencionado — presumiblemente cubierto por R16A-RE-FU-010, confirmar |
| — | Criterio D5 (estatus del pedido) | No mencionado — pendiente de catálogo cliente (fuera de alcance PQF2 según el propio requisito) |

**Acción:** Renombrar o mapear explícitamente la tabla de criterios del DIS-SOL contra los IDs oficiales del requisito, para trazabilidad. No bloquea desarrollo, pero facilita auditoría futura.

### H-03 — Sección C del requisito (proforma) no se menciona en el DIS-SOL — contradicción heredada, no del diseño

La Sección C del requisito (Criterios C1-C4: previsualización y envío de proforma) y la cláusula final del Criterio A1 ("generar automáticamente la proforma asociada al pedido") contradicen el Alcance actualizado del propio requisito ("No aplica a": no se genera proforma). Esta contradicción fue dejada intencionalmente en `R16A-RE-FU-015.md` por instrucción tuya ("dejarlo tal cual me lo pasaste") al actualizar el documento con la matriz del cliente.

El DIS-SOL no cae en esa contradicción: su tabla de criterios ni siquiera enumera C1-C4, y RT-01 declara explícitamente "RE-015 no genera proforma, PDF ni correo". Es decir, el diseño interpretó correctamente el Alcance vigente por encima del texto residual de la Sección C. No requiere ninguna acción sobre el diseño — se documenta aquí solo para que quede constancia de que la Sección C del requisito sigue pendiente de limpieza editorial en algún momento futuro.

### H-04 — Encabezado de página con folio incorrecto

Todas las páginas del PDF (excepto la portada) muestran el encabezado "R16A-RE-FU-013." en vez de "R16A-RE-FU-015." — artefacto de copiar la plantilla del diseño de 013. Cosmético, pero puede generar confusión si el documento circula fuera de este contexto. También hay una discrepancia menor entre la VERSIÓN de portada (1.0) y la fila de Control de versiones (1.1).

### H-05 — `fccFactura` como tabla compartida con la factura final (RT-10) ata a RE-FU-018/019/020, aún no diseñados

RT-10 decide que `fccFactura` es una tabla única para la FAA y la factura final, diferenciadas por `EsFacturaPorAdelantado`. Es una decisión de diseño razonable, pero compromete de antemano el esquema que usarán RE-FU-018/019/020 (Factura por Adelantado — emisión/timbrado), que todavía no tienen su propio DIS-SOL. Recomendación: cuando se diseñe 018/019/020, validar que esta decisión siga siendo válida antes de construir sobre ella, para evitar rediseño de `fccFactura` a mitad de implementación.

---

## Puntos que están bien

- **RT-01 / Alcance:** el diseño refleja fielmente la reducción de alcance del requisito actualizado — sin proforma, PDF ni correo.
- **Arquitectura orquestador → Finanzas:** reutiliza el mismo patrón ya validado y aprobado en 013 (folio generado y commiteado en `tpPedido` por ProquifaDotNet antes de delegar a Finanzas; reintento idempotente si Finanzas falla). Consistente entre 013/014/015.
- **`ProquifaDotNetFinanzas` como ubicación de la nueva lógica:** ya confirmado como arquitectura legítima en la revisión de 013/014, aunque los requisitos 013-015 son anteriores al corte formal "a partir del 016" para nuevas soluciones.
- **Numeración de GAP-03** ("elimina validación de código de autorización") coincide con la numeración usada en `R16A-RE-FU-015-Back.md`, lo que indica consistencia entre ambos documentos.
- **RT-06 / Criterio C2:** confirma correctamente que no se genera pendiente Validar Cobro al tramitar porque no existe `tpProformaPedido.MontoPendiente` que lo dispare — mismo razonamiento que documentamos al actualizar `_BD.md`.
- **OBS-027 (`IdCatEstadoTpPedido`/`CatEstadoTpPedido`):** identificado como bloqueante compartido con 013, consistente con lo ya documentado en `Back.md`/`_BD.md`/`Tareas.md` de este requisito.
- **Manejo de errores:** la tabla de escenarios (rollback en ProquifaDotNet si falla el guardado del folio; sin rollback pero reintento seguro si falla Finanzas; rollback atómico de las tres tablas si falla el INSERT en Finanzas) es clara y completa.

---

## Notas de contexto

- **Actualización 06/07/2026:** la arquitectura de este DIS-SOL (`fccFactura` + `fccFacturaPartida` + `fccFacturaReferenciaBancaria`, endpoints `/v1/api/invoices/advance-invoice/...`) ya fue adoptada en `R16A-RE-FU-015-Back.md`, `R16A-RE-FU-015_BD.md` y `R16A-RE-FU-015-Tareas.md` (segunda pasada de actualización mencionada abajo). `tpProformaAdelanto` ya no aparece en esos documentos como entidad del pendiente FAA de RE-015. H-01 (campos fiscales de Perú) se conservó como gap documentado (GAP-08 en `Back.md`, hallazgo abierto en `_BD.md`) en lugar de bloquear la adopción del resto del diseño; H-02/H-04/H-05 quedan como estaban, sin acción adicional requerida sobre el DIS-SOL.

---

## Referencias

- `R16A-RE-FU-015.md` (versión actualizada por matriz cliente)
- `[R16A-RE-FU-015][DIS-SOL] Diseño de la solución.pdf`
- `R16A-RE-FU-015-Back.md`, `R16A-RE-FU-015_BD.md`, `R16A-RE-FU-015-Tareas.md` (documentos complementarios, alineados al requisito, no al diseño)
- `R16A-RE-FU-013_DIS-SOL_Revision.md` (arquitectura orquestador→Finanzas ya validada ahí)
