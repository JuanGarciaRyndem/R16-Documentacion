# R16A-RE-FU-022 — Cancelado

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-022 |
| **Título** | Diseño y generación de Documentos: Factura Perú |
| **Módulo** | Factura por Adelantado |
| **Estado** | **CANCELADO** |
| **Fecha de cancelación** | 2026-07-20 |

---

## Motivo de cancelación

El timbrado fiscal y la facturación electrónica de Perú quedan **fuera del alcance de R16** por decisión del cliente. La generación del PDF del CPE tipo 01 (UBL 2.1) y el timbrado ante SUNAT/OSE para clientes Perú no se implementarán en este release.

## Impacto

- No se genera PDF de factura electrónica (CPE tipo 01) para clientes Perú.
- El módulo Factura por Adelantado (RE-FU-020) también se cancela.
- Las referencias a este requisito en RE-FU-025 y RE-FU-029 quedan eliminadas.
- Perú opera únicamente con Proforma (RE-FU-017) como documento de soporte al cobro.

## Referencia

Ver análisis completo en `Analisis/Quitar Peru/03 Quitar Perú — Análisis.md` (acción: SALE).
