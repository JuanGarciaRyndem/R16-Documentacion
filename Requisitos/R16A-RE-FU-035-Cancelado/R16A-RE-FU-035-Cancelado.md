# R16A-RE-FU-035 — Cancelado

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-035 |
| **Título** | Diseño y generación de Documentos: NDC Perú |
| **Módulo** | Notas de Crédito |
| **Estado** | **CANCELADO** |
| **Fecha de cancelación** | 2026-07-20 |

---

## Motivo de cancelación

El timbrado fiscal y la facturación electrónica de Perú quedan **fuera del alcance de R16** por decisión del cliente. La generación del PDF del CPE tipo 07 (Nota de Crédito Electrónica, UBL 2.1) y el timbrado ante SUNAT para clientes Perú no se implementarán en este release.

## Impacto

- No se genera PDF ni CPE tipo 07 para Notas de Crédito de clientes Perú.
- El módulo NC Perú (RE-FU-033) también se cancela.
- La referencia a este requisito en RE-FU-034 queda eliminada.
- La conservación de XMLs por 5 años (R.S. 117-2017/SUNAT) no aplica al no generarse documentos.

## Referencia

Ver análisis completo en `Analisis/Quitar Peru/03 Quitar Perú — Análisis.md` (acción: SALE).
