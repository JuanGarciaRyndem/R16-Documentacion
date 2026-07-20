# R16A-RE-FU-033 — Cancelado

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-033 |
| **Título** | Notas de Crédito: Perú |
| **Módulo** | Notas de Crédito |
| **Estado** | **CANCELADO** |
| **Fecha de cancelación** | 2026-07-20 |

---

## Motivo de cancelación

El timbrado fiscal y la facturación electrónica de Perú quedan **fuera del alcance de R16** por decisión del cliente. El módulo de Notas de Crédito para Región Perú (CPE tipo 07, UBL 2.1, timbrado ante SUNAT) no se implementará en este release.

## Impacto

- No se implementa el wizard de generación de NCs electrónicas para clientes Perú.
- El módulo NC Perú no alimenta el Paso 2 de Validar Cobro para Perú (RE-FU-027 se simplifica: sin aplicación de NCs).
- Las referencias a este requisito en RE-FU-027, RE-FU-032 y RE-FU-035 quedan eliminadas.
- La estructura funcional del módulo (reutilizada de RE-FU-032 México) no se adapta para Perú.

## Referencia

Ver análisis completo en `Analisis/Quitar Peru/03 Quitar Perú — Análisis.md` (acción: SALE).
