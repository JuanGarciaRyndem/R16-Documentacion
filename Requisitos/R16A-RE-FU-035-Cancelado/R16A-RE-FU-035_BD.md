# Impacto en Base de Datos — R16A-RE-FU-035
**Requisito:** Diseño y generación de Documentos — NDC Perú (CPE tipo 07 — UBL 2.1)
**Fecha:** 2026-06-09
**Aplicativo BD:** ProquifaDotNet | DocumentBuilder

---

## Resumen ejecutivo

Este requisito **no introduce cambios nuevos en base de datos**. Toda la estructura de BD necesaria para la generación del CPE tipo 07 y el PDF de la NC Perú fue creada en el requisito **R16A-RE-FU-033** (Notas de Crédito Perú — Módulo). R16A-RE-FU-035 cubre únicamente la implementación del XML builder (UBL 2.1), el servicio de mapeo PDF y el diseño del template HTML.

---

## Prerrequisitos de BD — creados en R16A-RE-FU-033

Los siguientes cambios de BD son **prerrequisito** de las tareas de este requisito y deben estar implementados antes de iniciar:

| # RE-033 | Objeto                        | Cambio                                                          | Aplicativo             |
| -------- | ----------------------------- | --------------------------------------------------------------- | ---------------------- |
| T1       | `fccNotaCredito`              | ALTER TABLE — 3 columnas Perú (ResponseCode, ResponseDescription, TipoCambioOrigen) | ProquifaDotNet  |
| T2       | `catMotivoCreditoSUNAT09`     | CREATE TABLE + DML — 11 motivos catálogo 09 SUNAT               | ProquifaDotNet         |
| T3       | `catTipoCFDI`                 | DML — INSERT clave NOTA_CREDITO_PERU (TipoDocumento='07')       | ProquifaDotNet         |
| T4       | `EmpresaFolio`                | DML — INSERT Serie NC Perú para Golocaer S.A.C.                 | ProquifaDotNet (Finanzas) |
| T5       | `DocumentTemplate`            | DML — INSERT template GOL\_PER\_NC (H/B/F)                      | DocumentBuilder        |

> **Nota adicional:** R16A-RE-FU-032 T1 también es prerrequisito (columnas genéricas de NC en `fccNotaCredito`, base para las columnas Perú de RE-033 T1).

---

## Tablas consultadas por este requisito (lectura)

Las siguientes tablas son leídas durante la generación del XML y el PDF. No requieren cambios estructurales.

| Tabla                         | Uso                                                                        |
| ----------------------------- | -------------------------------------------------------------------------- |
| `fccNotaCredito`              | Cabecera de la NC (serie-correlativo, moneda, tipo cambio origen, motivo)  |
| `fccNotaCreditoPartida`       | Partidas de la NC (modalidad por partidas)                                 |
| `CFDIGenerada`                | CPE origen — Serie-Correlativo, Moneda, TipoCambioOrigen                   |
| `CFDIGeneradaConcepto`        | Conceptos originales del CPE origen — herencia de datos fiscales           |
| `catMotivoCreditoSUNAT09`     | Código y descripción del motivo catálogo 09 seleccionado                   |
| `DocumentTemplate`            | Registro del template HTML (GOL\_PER\_NC)                                  |
| `Empresa`                     | Datos de Golocaer S.A.C. — RUC, razón social, domicilio fiscal, régimen   |
| `Cliente`                     | Datos del cliente — RUC/documento, razón social                            |

---

## Diferencias clave con RE-034 (México) en BD

| Aspecto                         | México (RE-034)                          | Perú (RE-035)                                      |
| ------------------------------- | ---------------------------------------- | -------------------------------------------------- |
| Empresas emisoras               | GOL, MUN, PRO, PQF (4 templates)        | Solo Golocaer S.A.C. (1 template)                  |
| Tabla catálogo motivo           | `catMotivoCancelacionSAT` (4 claves)     | `catMotivoCreditoSUNAT09` (11 motivos)             |
| Tabla de relación CFDI          | `CFDIGeneradaRelacionado` (UUID SAT)     | No aplica — referencia por serie-correlativo        |
| Columnas específicas en NC      | `ClaveMotivosCancelacion varchar(4)`     | `ResponseCode`, `ResponseDescription`, `TipoCambioOrigen` |

---

## Pendientes

| ID  | Descripción                                                                                              | Impacto                                                    |
| --- | -------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| P1  | **Brecha B1 RE-033:** Modalidad de emisión electrónica SUNAT no definida                                 | Bloquea timbrado end-to-end; XML builder + PDF pueden desarrollarse aislados |
| P2  | **Brecha B3 RE-033:** `fccNotaCredito.IdTPProformaPedido` NOT NULL — heredada de RE-032 B1              | Bloquea escritura en `fccNotaCredito`                      |
| P3  | **TipoCambioOrigen:** confirmar que la columna guarda el TC de la fecha de emisión del CPE origen        | Clave fiscal diferencial Perú — validar con asesor fiscal  |
| P4  | **Maquetas PDF NC Perú:** no disponibles aún; diseño se validará cuando lleguen                          | Puede requerir ajustes al template GOL\_PER\_NC            |
| P5  | **Toda la mecánica fiscal SUNAT** de este requisito está pendiente de validación con asesor fiscal peruano | Puede obligar a reestructurar el XML y/o el PDF           |
