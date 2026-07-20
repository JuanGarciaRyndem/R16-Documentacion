# R16A-RE-FU-020 — Cancelado

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-020 |
| **Título** | Factura por Adelantado: Detalle Perú |
| **Módulo** | Factura por Adelantado |
| **Estado** | **CANCELADO** |
| **Fecha de cancelación** | 2026-07-20 |

---

## Motivo de cancelación

El timbrado fiscal y la facturación electrónica de Perú quedan **fuera del alcance de R16** por decisión del cliente. En Perú no existe Factura por Adelantado para pedidos Crédito (el timbrado peruano en R16 se limita a Prepago), y dado que el módulo de timbrado SUNAT/OSE completo sale del alcance, este requisito se cancela en su totalidad.

## Impacto

- Los pedidos Prepago de clientes Perú continúan su flujo hasta la Confirmación de Pedido sin generar factura electrónica ante SUNAT.
- El listado del módulo Factura por Adelantado (RE-FU-018) filtra clientes México exclusivamente.
- Las referencias a este requisito en RE-FU-027 y RE-FU-029 quedan eliminadas.

## Referencia

Ver análisis completo en `Analisis/Quitar Peru/03 Quitar Perú — Análisis.md` (acción: SALE).
