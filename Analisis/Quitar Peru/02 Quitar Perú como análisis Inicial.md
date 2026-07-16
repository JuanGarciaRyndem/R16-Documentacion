# Quitar Perú — Análisis Inicial

Resumen ejecutivo de los cambios necesarios en la documentación de requisitos R16 a partir de la decisión de dejar fuera la Facturación y el Timbrado de Perú (ver `Dejar Fuera Facturacion y Timbrado en Peru.md`).

Este documento es el **punto de partida**. El análisis exhaustivo por requisito (mapeo RE-027 a RE-035, dependencias, secciones concretas a editar) se elabora en un documento posterior.

---

## Punto 1 — Requisitos que salen completos del alcance

- **RE-020** — Factura por Adelantado: Detalle Perú (módulo completo).
- **RE-022** — Factura Perú (Diseño y PDF CPE SUNAT).
- **Notas de Crédito Perú** — módulo y PDF (verificar mapeo exacto a RE-032 / RE-033 / RE-034 / RE-035).
- **Complemento de Pago Perú** — ya estaba fuera; solo confirmar y anotar.

---

## Punto 2 — Requisitos que se simplifican (retirar lógica Perú fiscal)

- **RE-015** — Tramitación Prepago sin controlados con FpA → bloquear/no mostrar radio button de FpA cuando región = Perú.
- **RE-018** — Factura por Adelantado, pantalla inicial → filtrar clientes Perú del listado.
- **Validar Cobro Paso 2 Perú** (verificar si es RE-027 u otro) → retirar aplicación de Notas de Crédito, simplificar cálculo de saldo a `adeudo Proformas − cobros aplicados`, eliminar reglas NC por documento / conversión NC en moneda distinta / aplicar/remover NC.
- **Validar Cobro Paso 3 Perú** (verificar mapeo) → simplificar a solo envío de Confirmación de Pedido (sin timbrado, sin generación de documentos fiscales).
- **RE-023** — Validar Cobro pantalla principal → revisar si el botón contextual o los filtros dependen de FpA/Factura Perú.

---

## Punto 3 — Requisitos que se conservan sin cambios

- **RE-004** — Sección Entrega y Facturación (RUC, Tipo Sociedad, Régimen Fiscal Perú) — propuesta: dejar capturable.
- **RE-005** — Sección Cobros Perú (Condición de Pago, Tipo Comprobante, Retención IGV, Detracción) — propuesta: dejar capturable.
- **RE-017** — Proforma Perú (diseño y PDF) — se conserva íntegro.
- **RE-025** — Validar Cobro Paso 1 Perú (captura del cobro) — se conserva íntegro.

---

## Punto 4 — Ajustes propuestos a confirmar con cliente

- Asociación Paso 2 Perú siempre contra **Proformas** (no contra Facturas, porque Facturas Perú ya no existen).
- Conservación de la configuración fiscal Perú capturable en Catálogo de Clientes (Cobros, Entrega y Facturación) y en Validar Cobro, para no perder el trabajo de análisis y quedar listos si el timbrado se habilita más adelante.

---

## Punto 5 — Impactos en soluciones nuevas

- **ProquifaDotNet.Finanzas** — quitar del alcance módulos FpA Perú, Factura Perú, NC Perú; ajustar Scaffold/EF Core y endpoints (`Endpoints-Finanzas.md`, `ER-Finanzas.md`).
- **ProquifaDotNet.Timbrado** — validar que no haya referencias a SUNAT/OSE en `Endpoints-Timbrado.md`, `ER-Timbrado.md`, `StampingController`. En principio TurboPac México ya era lo único activo.
- **ProquifaDotNet.LegacySync** — sin impacto (Perú no transfiere a Legacy).

---

## Punto 6 — Impactos transversales

- **Matriz de requisitos** — actualizar cobertura por región (bajan filas Perú fiscales).
- **Diagramas de secuencia** — quitar los que involucren timbrado Perú.
- **Contexto de proyecto / memoria** — actualizar brechas OSE, catálogos SUNAT, Golocaer Perú (pasan a "fuera de R16").
- **Guía de Estimación** — quitar tareas de los requisitos que salen; ajustar tareas de los que se simplifican.
- **Requisitos 027–035** — verificar sus títulos/objetivos exactos para mapear cuáles son Paso 2 Perú, Paso 3 Perú, NC Perú, NC PDF, etc. (Punto A del análisis siguiente).

---

## Siguientes pasos

- **A. Análisis exhaustivo** — mapear RE-027 a RE-035, identificar dependencias por requisito, secciones concretas a editar y orden de aplicación de cambios.
- **B. Integración por punto** — arrancar por el Punto 1 (los que salen completos) porque son los cambios más contundentes y bloquean el resto.
