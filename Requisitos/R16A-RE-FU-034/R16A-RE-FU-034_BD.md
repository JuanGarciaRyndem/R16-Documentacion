# Impacto en Base de Datos — R16A-RE-FU-034
**Requisito:** Diseño y generación de Documentos — NDC México (CFDI tipo E)
**Fecha:** 2026-06-09
**Aplicativo BD:** ProquifaDotNet | DocumentBuilder

---

## Resumen ejecutivo

Este requisito **no introduce cambios nuevos en base de datos**. Toda la estructura de BD necesaria para la generación del CFDI tipo E y el PDF de la NC México fue creada en el requisito **R16A-RE-FU-032** (Notas de Crédito México — Módulo). R16A-RE-FU-034 cubre únicamente la implementación del XML builder, el servicio de mapeo PDF y el diseño de los templates HTML.

---

## Prerrequisitos de BD — creados en R16A-RE-FU-032

Los siguientes cambios de BD son **prerrequisito** de las tareas de este requisito y deben estar implementados antes de iniciar:

| # RE-032 | Objeto                    | Cambio                                                        | Aplicativo             |
| -------- | ------------------------- | ------------------------------------------------------------- | ---------------------- |
| T1       | `fccNotaCredito`          | ALTER TABLE — 13 columnas R16                                 | ProquifaDotNet         |
| T2       | `fccNotaCreditoPartida`   | ALTER TABLE — 6 columnas R16                                  | ProquifaDotNet         |
| T3       | `catUsoCFDI`              | DML — INSERT clave G02                                        | ProquifaDotNet         |
| T4       | `catTipoCFDI`             | DML — INSERT clave NOTA_CREDITO                               | ProquifaDotNet         |
| T5       | `EmpresaFolio`            | DML — INSERT 4 filas Serie "P2" (GOL, MUN, PRO, PQF)          | ProquifaDotNetTimbrado |
| T6       | `DocumentTemplate`        | DML — INSERT 4 registros GOL/MUN/PRO/PQF\_MEX\_NC             | DocumentBuilder        |
| T12      | `catMotivoCancelacionSAT` | CREATE TABLE + DML — 4 claves c\_MotivoCancelacion SAT        | ProquifaDotNet         |

---

## Tablas consultadas por este requisito (lectura)

Las siguientes tablas existentes son leídas durante la generación del XML y el PDF. No requieren cambios estructurales.

| Tabla                       | Uso                                                                    |
| --------------------------- | ---------------------------------------------------------------------- |
| `fccNotaCredito`            | Origen de datos de cabecera de la NC                                   |
| `fccNotaCreditoPartida`     | Partidas de la NC (modalidad por partidas)                             |
| `CFDIGenerada`              | Factura origen relacionada — UUID, RFC emisor/receptor, Moneda         |
| `CFDIGeneradaConcepto`      | Conceptos originales de la factura origen — herencia de datos fiscales |
| `CFDIGeneradaRelacionado`   | Relación CFDI padre-hijo (TipoRelacion 01)                             |
| `catTipoRelacion`           | Catálogo tipo relación SAT (TipoRelacion=01)                           |
| `catUsoCFDI`                | Catálogo uso CFDI (G02 default)                                        |
| `catMotivoCancelacionSAT`   | Motivo cancelación SAT seleccionado en wizard Paso 2                   |
| `DocumentTemplate`          | Registro del template HTML a usar (GOL/MUN/PRO/PQF\_MEX\_NC)          |
| `Empresa`                   | Logo, paleta corporativa, RFC, CP, Régimen Fiscal emisor               |
| `Cliente`                   | RFC, Razón Social, CP fiscal, Régimen Fiscal receptor                  |

---

## Pendientes

| ID  | Descripción                                                                                                        | Impacto                                                     |
| --- | ------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| P1  | **Brecha B1 RE-032:** `fccNotaCredito.IdTPProformaPedido` NOT NULL — resolver antes de ejecutar RE-032 T1          | Bloquea escritura en `fccNotaCredito`                       |
| P2  | **Brecha B2 RE-032:** `fccNotaCreditoPartida.NumeroDePiezas int` vs `decimal` — resolver antes de RE-032 T2        | Bloquea escritura de cantidades fraccionarias               |
| P3  | **Campo `Descuento` comprobante root:** validar con asesor fiscal si se puebla o se omite (ver Regla 11 RE-034)    | Afecta estructura XML y totales del PDF — sin cambio de BD  |
| P4  | **`ObjetoImp` modalidad manual:** confirmar valor aplicable al concepto único ClaveProdServ=84111506               | Afecta nodo Impuestos del concepto — sin cambio de BD       |
| P5  | **Serie del foliador:** validar nombre de la serie distintiva para NC México en `EmpresaFolio`                     | Puede requerir actualizar el DML de RE-032 T5               |
