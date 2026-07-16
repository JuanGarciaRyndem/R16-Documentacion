# Quitar Perú — Análisis Exhaustivo

Complemento del documento `Quitar Perú como análisis Inicial.md`. Aquí se mapea con precisión requisito por requisito (con foco en RE-027 a RE-035), sus dependencias cruzadas y las secciones concretas a modificar. Sirve como plan de ejecución antes de tocar el contenido de los requisitos.

Nomenclatura usada en la tabla de acción:

- **SALE** — el requisito completo queda fuera del alcance R16. Se marca como cancelado, se conserva el archivo con nota de cancelación y motivo.
- **SIMPLIFICA** — el requisito se conserva pero se retira lógica peruana fiscal (timbrado, NCs, documentos SUNAT). Puede quedar como requisito operativo interno.
- **CONSERVA** — el requisito sigue igual pero se retiran referencias cruzadas a los requisitos que salen y se ajustan notas de dependencia.
- **AJUSTA** — cambio menor: bloquear opción por región, filtrar por región, ajustar disclaimer.

---

## 1. Mapeo confirmado de requisitos 027–035

| ID | Título | Región | Acción | Motivo |
|---|---|---|---|---|
| RE-027 | Validar Cobro: Paso 2 Perú | Perú | **SIMPLIFICA** | Retirar aplicación de NCs, simplificar cálculo de saldo, quitar referencias a RE-020/022/033/035, ajustar Regla 2 (asociación siempre contra Proformas). |
| RE-028 | Validar Cobro: Paso 3 México | México | **CONSERVA** | Retirar referencias a "estructura reutilizada por Perú" (ya no aplica). |
| RE-029 | Validar Cobro: Paso 3 Perú | Perú | **SIMPLIFICA (mayor)** | Retirar timbrado SUNAT, generación de Factura Perú y envío de PDF/XML. Queda solo el envío de la Confirmación de Pedido y establecimiento de FEE. |
| RE-030 | Diseño y generación de Documentos: CDP México | México | **CONSERVA** | Sin cambios (Perú CDP ya estaba fuera). Retirar en No aplica la frase "requisito independiente si SUNAT exige documento equivalente". |
| RE-032 | Notas de Crédito: México | México | **CONSERVA** | Retirar referencias a "estructura reutilizada por Perú" y "R16A-RE-FU-033". |
| RE-033 | Notas de Crédito: Perú | Perú | **SALE** | Módulo NC Perú completo fuera de alcance. |
| RE-034 | Diseño y generación de Documentos: NDC México | México | **CONSERVA** | Retirar referencia a "requisito independiente" para Perú. |
| RE-035 | Diseño y generación de Documentos: NDC Perú | Perú | **SALE** | PDF NC Perú fuera de alcance. |

---

## 2. Requisitos 001–026 con impacto

| ID     | Título                                      | Acción                    | Detalle del cambio                                                                                                                                                                                                                                                   |
| ------ | ------------------------------------------- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RE-004 | Sección Entrega y Facturación               | **CONSERVA**              | Se conserva captura RUC, Tipo Sociedad, Régimen Fiscal Perú (propuesta). Verificar que no exista lógica de "envío a SUNAT" en el propio requisito.                                                                                                                   |
| RE-005 | Sección Cobros                              | **CONSERVA** con revisión | Se conservan Condición de Pago, Tipo Comprobante, Agente Retención IGV, Sujeto a Detracción como capturables. Revisar brechas B1–B5 (fiscal producto, GRE, Tipo de Operación, modelo bancario, OSE): marcarlas como "fuera de R16" en lugar de "brecha por definir". |
| RE-015 | Tramitación Prepago sin controlados con FpA | **AJUSTA**                | Bloquear/ocultar radio button de FpA cuando región cliente = Perú. Nota en Alcance/No aplica y regla nueva.                                                                                                                                                          |
| RE-017 | Proforma Perú (Diseño y PDF)                | **CONSERVA**              | Se conserva íntegro. Confirmar disclaimer SUNAT (B4) sigue vigente sin timbrado.                                                                                                                                                                                     |
| RE-018 | Factura por Adelantado: pantalla inicial    | **AJUSTA**                | Filtrar clientes región Perú del listado (no aparecen pedidos FpA de Perú porque no hay FpA en Perú).                                                                                                                                                                |
| RE-019 | Factura por Adelantado: Detalle México      | **CONSERVA**              | Sin cambios.                                                                                                                                                                                                                                                         |
| RE-020 | Factura por Adelantado: Detalle Perú        | **SALE**                  | Módulo completo fuera.                                                                                                                                                                                                                                               |
| RE-021 | Factura México (PDF CFDI)                   | **CONSERVA**              | Sin cambios.                                                                                                                                                                                                                                                         |
| RE-022 | Factura Perú (PDF CPE SUNAT)                | **SALE**                  | Módulo completo fuera.                                                                                                                                                                                                                                               |
| RE-023 | Validar Cobro: pantalla principal           | **CONSERVA** con revisión | Cliente Perú sí aparece en la pantalla principal (para operar Paso 1, Paso 2 simplificado y Paso 3 simplificado). Revisar filtros por región y confirmar que sigue habilitado para Perú.                                                                             |
| RE-024 | Validar Cobro: Paso 1 México                | **CONSERVA**              | Sin cambios.                                                                                                                                                                                                                                                         |
| RE-025 | Validar Cobro: Paso 1 Perú                  | **CONSERVA**              | Sin cambios. Confirmar que la vinculación cobro↔factura no depende de RE-022 (Facturas Perú no existen; solo Proformas).                                                                                                                                             |
| RE-026 | Validar Cobro: Paso 2 México                | **CONSERVA**              | Sin cambios. Verificar que las referencias cruzadas a "Paso 2 Perú (RE-027)" queden consistentes con la nueva simplificación.                                                                                                                                        |

---

## 3. Cambios concretos por requisito (secciones a editar)

### RE-027 — Validar Cobro: Paso 2 Perú (SIMPLIFICA)

- **Historia de Usuario**: reescribir eliminando "y aplico opcionalmente sus Notas de Crédito"; queda "para conciliar cobros contra Proformas y facturas históricas".
- **Requisito Funcional**: eliminar mención a "aplicar Notas de Crédito vigentes del cliente" y "el cálculo del saldo incluye la suma de NCs aplicadas".
- **Alcance / Aplica a**:
  - Retirar viñeta de aplicación opcional de NCs.
  - Ajustar viñeta de cálculo del saldo: `cobros aplicados − adeudo Proformas` (sin componente NC).
  - Retirar referencia a R16A-RE-FU-033/035.
  - Confirmar propuesta: la asociación opera contra **Proformas**; documentos históricos de facturas se conservan solo para conciliación de casos migrados.
- **Alcance / No aplica a**:
  - Añadir: "Aplicación de Notas de Crédito (fuera de alcance R16 en Perú)".
  - Añadir: "Cálculo de saldo con componente NC (fuera de alcance R16 en Perú)".
- **Reglas**: retirar las reglas que gobiernan aplicación de NCs, saldos remanentes de NC, aplicación en moneda distinta, etc. Simplificar Regla 2 (efecto fiscal): la asociación sigue siendo operativa.
- **Riesgos y pendientes**: cerrar los pendientes que referencian mecánica fiscal SUNAT de NC (heredados de RE-033).

### RE-028 — Validar Cobro: Paso 3 México (CONSERVA)

- **Requisito Funcional**: eliminar la frase final "La estructura UI de esta pantalla se reutiliza para Perú con diferencias importantes y se documenta en requisito independiente".
- **Alcance / Aplica a**: eliminar "Aplica a clientes con Región México exclusivamente (Perú se documenta en requisito independiente con diferencias significativas)" y dejar solo "Región México".
- **Alcance / No aplica a**: sustituir la primera viñeta por "Región Perú (Paso 3 se simplifica a envío de Confirmación de Pedido; ver RE-029 simplificado)".
- **Reglas**: quitar la referencia a "Regla 1 — Aplicabilidad solo a Región México" que hace remisión al equivalente de Perú.

### RE-029 — Validar Cobro: Paso 3 Perú (SIMPLIFICA — cambio mayor)

Este es el cambio más profundo. Aplica esencialmente:

- **Historia de Usuario**: reescribir a "quiero que el Paso 3 envíe la Confirmación de Pedido y establezca la Fecha Estimada de Entrega del pedido, para cerrar el ciclo operativo de cobranza para Perú sin timbrado fiscal".
- **Requisito Funcional**: reescribir eliminando timbrado ante SUNAT, generación de Factura electrónica CPE 01, PDF/XML de la factura. Queda: por cada línea de la asociación cerrada en el Paso 2, el sistema envía al cliente la Confirmación de Pedido y establece la FEE del pedido; no genera ni timbra documentos fiscales.
- **Alcance / Aplica a**:
  - Retirar todo lo referente a CPE tipo 01, UBL 2.1, IGV, Tipo de Operación, Condición de Pago, previsualización de PDF, timbrado, folio SUNAT, envío de PDF+XML.
  - Conservar: cabecera del cliente, listado de líneas, envío de Confirmación de Pedido con CC al ESAC, establecimiento de FEE, estados por línea (Pendiente → Enviado), auto-guardado, persistencia.
  - Modificar estados: `Pendiente → Enviado` (elimina "Factura Generada").
- **Alcance / Pendientes estructurales**: cerrar los 5 pendientes fiscales (modalidad SUNAT, PAC, plantilla correo Perú, Tipo de Operación, referencia NC) porque ya no aplican.
- **Alcance / No aplica a**: añadir "Timbrado, generación y envío de documentos fiscales (fuera de alcance R16 en Perú)".
- **Reglas**: eliminar las reglas 3, 4 (parcial), 5, 6 y todas las relativas a timbrado, catálogo SUNAT y NCs. Conservar Regla 1 (aplicabilidad a Perú) y Regla 2 (líneas del Paso 2). Añadir regla: "El Paso 3 no genera ni timbra documentos fiscales; su función es enviar la Confirmación de Pedido y establecer la FEE del pedido".
- **Riesgos**: sustituir riesgos de timbrado por el riesgo operativo de doble captura fuera del sistema.

### RE-030 — CDP México (CONSERVA)

- **Alcance / No aplica a**: eliminar la viñeta "Generación del Complemento de Pago para Perú (requisito independiente si SUNAT exige documento equivalente)". Reemplazar por: "Región Perú (fuera de alcance R16)".

### RE-032 — Notas de Crédito México (CONSERVA)

- **Requisito Funcional**: eliminar frase "La estructura funcional del módulo se reutiliza para Perú con diferencias significativas y se documenta en requisito independiente".
- **Alcance / Aplica a**: eliminar viñeta "Aplica a Región México; la estructura funcional se reutiliza para Perú...".
- **Alcance / No aplica a**: sustituir "Módulo NC para Región Perú (requisito independiente...)" por "Módulo NC para Región Perú (fuera de alcance R16)".
- **Reglas**: en Regla 1 quitar "La estructura funcional se reutiliza para Región Perú con sus diferencias específicas, documentada en requisito independiente".

### RE-033 — Notas de Crédito Perú (SALE)

- Marcar el archivo/carpeta como cancelado (renombrar carpeta a `R16A-RE-FU-033-Cancelado/` siguiendo patrón de RE-008-Cancelado y RE-031-Cancelado).
- Añadir bloque "Estado: Cancelado" con motivo "Timbrado y facturación de Perú fuera de alcance R16 por decisión de cliente".

### RE-034 — Diseño NDC México (PDF) (CONSERVA)

- **Requisito Funcional**: eliminar "La estructura se reutiliza para Perú con diferencias significativas y se documenta en requisito independiente".
- **Alcance / No aplica a**: sustituir "Generación de NC para Perú" por "Generación de NC para Perú (fuera de alcance R16)".

### RE-035 — Diseño NDC Perú (PDF) (SALE)

- Marcar el archivo/carpeta como cancelado (mismo patrón que RE-033).

### RE-020 — FpA Detalle Perú (SALE)

- Marcar carpeta como cancelada. Nota de motivo.

### RE-022 — Factura Perú PDF (SALE)

- Marcar carpeta como cancelada. Nota de motivo.

### RE-015 — Tramitación Prepago sin controlados con FpA (AJUSTA)

- **Alcance / Aplica a**: añadir nota "Radio button FpA no visible cuando cliente = Región Perú".
- **Alcance / No aplica a**: añadir "Región Perú (FpA fuera de alcance R16 en Perú)".
- **Reglas**: regla nueva: "Cuando el cliente es Región Perú, la opción de Factura por Adelantado NO se muestra en Tramitar Pedido".

### RE-018 — FpA pantalla inicial (AJUSTA)

- **Alcance / Aplica a**: añadir "El filtrado por cartera del usuario aplica sobre clientes México exclusivamente".
- **Reglas**: regla nueva: "Los clientes de Región Perú no aparecen en el listado (no hay pedidos con Factura por Adelantado en Perú)".

### RE-023 — Validar Cobro pantalla principal (CONSERVA con revisión)

- Verificar que la lógica de "usuario opera una sola región" siga aplicando para Perú (con Paso 2 y Paso 3 simplificados).
- Añadir en Reglas: "En Perú, el botón contextual sigue disponible con la nueva lógica simplificada del Paso 3".

### RE-005 — Sección Cobros (CONSERVA con revisión)

- Marcar brechas B1–B5 (datos fiscales producto, GRE, Tipo de Operación, modelo bancario, OSE) como "**Fuera de alcance R16 — se conservan capturables pero no se timbran**".
- No eliminar las banderas Agente Retención IGV y Detracción (se dejan capturables).

---

## 4. Impactos en soluciones nuevas

### ProquifaDotNet.Finanzas

- **Endpoints-Finanzas.md**: revisar y quitar endpoints o parámetros exclusivos a Perú (FpA Perú, Factura Perú, NC Perú). Los endpoints de Cobros y Validar Cobro se conservan pero con Paso 3 Perú sin ruta de timbrado.
- **ER-Finanzas.md**: revisar Scaffold EF Core. Confirmar si hay tablas exclusivas de facturación Perú (por ejemplo, series `F001`/`FC01`); si existen, marcarlas como "no utilizadas en R16".
- **Series EmpresaFolio**: quitar la serie `F001 GOLPERU` del alcance R16 (o dejarla capturable pero no consumida).
- **Diagramas de secuencia**: eliminar los que involucren Finanzas → Timbrado para Perú.

### ProquifaDotNet.Timbrado

- **Endpoints-Timbrado.md**: los 4 endpoints (stamp/invoice, stamp/payment-complement, stamp/credit-note, stamp/cancel) se conservan pero solo con SAT/TurboPac. Retirar toda referencia a SUNAT/OSE.
- **ER-Timbrado.md**: `AppSetting` conserva config para SAP TurboPac. Retirar configuraciones SUNAT si las hubiera.
- **StampingController**: sin cambios de código, pero validar que los DTOs no exijan campos SUNAT.

### ProquifaDotNet.LegacySync

- Sin impacto (Perú no transfiere a Legacy). Confirmar en la doc.

---

## 5. Impactos transversales

- **Matriz de requisitos**: actualizar cobertura por región. Bajan filas de RE-020, RE-022, RE-033, RE-035. RE-027 y RE-029 quedan como versiones simplificadas.
- **Contexto de proyecto / memoria de agente**: actualizar `r16_project_context.md` con el nuevo alcance (retirar 5 brechas SUNAT como "por analizar", marcar como "fuera de alcance R16").
- **Guía de Estimación (WBS Backend)**: quitar tareas de los requisitos que salen; ajustar tareas de los que se simplifican (RE-027 y RE-029 pierden tareas de timbrado, NCs).
- **Diagramas de secuencia**: eliminar los que involucren timbrado Perú, Factura Perú, NC Perú.
- **Análisis de brechas SUNAT (B1–B5, B7, B8, B10, B11, B12)**: mover del documento de brechas activas a un anexo "Brechas diferidas para timbrado Perú futuro".

---

## 6. Orden de ejecución recomendado (por qué en este orden)

1. **Punto 1 — Cancelar los "SALE" primero** (RE-020, RE-022, RE-033, RE-035): son los cambios más contundentes y todos los "CONSERVA/SIMPLIFICA" contienen referencias cruzadas a ellos. Cancelarlos primero facilita la limpieza posterior.
2. **Punto 2 — Simplificar RE-027 y RE-029**: son los que cambian sustancialmente y donde se materializa el nuevo flujo Perú.
3. **Punto 3 — Ajustar RE-015 y RE-018**: bloquear FpA Perú.
4. **Punto 4 — Retirar referencias cruzadas en RE-028, RE-030, RE-032, RE-034**: limpieza de menciones a "requisito Perú independiente".
5. **Punto 5 — Revisar RE-005, RE-017, RE-023, RE-025**: notas menores.
6. **Punto 6 — Actualizar transversales**: matriz de requisitos, memoria, guía de estimación, diagramas.

---

## 7. Consideraciones abiertas para confirmar con cliente antes de ejecutar

Antes de tocar los requisitos, conviene tener respuesta de PROQUIFA en:

- ¿La configuración fiscal capturable Perú (RUC, Régimen, Detracción, Retención IGV) se conserva como propuesto, o también se retira?
- ¿La Proforma Perú (RE-017) se mantiene sin cambios (disclaimer SUNAT y cuentas Golocaer Perú)?
- ¿En el Paso 3 Perú simplificado, el envío de la Confirmación de Pedido dispara alguna otra acción operativa (por ejemplo, cambio de estado del pedido, notificación a Almacén)?
- ¿La cancelación de facturas históricas Perú se maneja externamente sin ninguna interacción con el sistema?

---

## 8. Siguiente paso

Con este análisis listo, empezar por el **Punto 1 (cancelar RE-020, RE-022, RE-033, RE-035)**. Confirmar antes de proceder si:

- Se renombran las carpetas al patrón `-Cancelado/` (como RE-008 y RE-031), o solo se agrega un banner al inicio del `.md`.
- Se quiere conservar contenido histórico del archivo (recomendado, por si el timbrado Perú se retoma).
