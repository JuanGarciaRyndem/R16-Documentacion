---
tags:
  - moc
---

# R16 — Adquisiciones

Documentación técnica del proyecto R16 de ProquifaDotNet. Cubre el análisis, diseño y desarrollo de los módulos de Tramitación de Pedidos, Cobranza, Facturación, Buzones y Notas de Crédito para las regiones México y Perú. El proyecto extiende el sistema de Venta Interna (.NET Framework 4.8) e incorpora nuevas soluciones (.NET Core 10) para Finanzas y Timbrado.

**Proyecto:** R16
**PM:** Irma Andrade Aguado
**Autor técnico Back:** Juan David García Cruz
**Revisora:** Rosa Ríos Gómez

---

## 🗺️ Mapa de Contenido

### 🏢 Proyecto

---

#### 📊 Análisis

- [Borrador — Análisis de flujo y reglas de Tramitación](Analisis/Borrador%20-%20Analisis%20de%20flujo%20y%20reglas%20de%20Tramitacion.md)
- [R16 — Análisis Consolidado de Sesiones](Analisis/R16%20-%20Análisis%20Consolidado%20de%20Sesiones.md)
- [R16 — Análisis de Documentos de Referencia](Analisis/R16%20-%20Análisis%20de%20Documentos%20de%20Referencia.md)
- [Soluciones Nuevas](Analisis/Soluciones%20Nuevas.md)

---

#### 🆕 Soluciones Nuevas

- [ProquifaDotNet.Finanzas](Soluciones%20Nuevas/ProquifaDotNet.Finanzas.md)
- [ProquifaDotNet.Timbrado](Soluciones%20Nuevas/ProquifaDotNet.Timbrado.md)
- [ProquifaDotNet.LegacySync](Soluciones%20Nuevas/ProquifaDotNet.LegacySync.md)
- [ProquifaDotNet.EnvioCorreo](Soluciones%20Nuevas/ProquifaDotNet.EnvioCorreo.md)

---

#### 🗂️ Diagramas

- [01 — Flujo General de Tramitación](Diagramas/01%20-%20Flujo%20General%20de%20Tramitacion.md)
- [02 — Flujo de Cobranza Prepago — Validar Pago](Diagramas/02%20-%20Flujo%20de%20Cobranza%20Prepago%20-%20Validar%20Pago.md)
- [03 — Flujo de Factura por Adelantado](Diagramas/03%20-%20Flujo%20de%20Factura%20por%20Adelantado.md)
- [04 — Flujo del Buzón de Pagos](Diagramas/04%20-%20Flujo%20del%20Buzón%20de%20Pagos.md)
- [05 — Resumen de Rutas por Tipo de Cliente](Diagramas/05%20-%20Resumen%20de%20Rutas%20por%20Tipo%20de%20Cliente.md)
- [06 — Roles y Responsabilidades por Módulo](Diagramas/06%20-%20Roles%20y%20Responsabilidades%20por%20Modulo.md)
- [07 — Tramitación de Pedidos — Crédito](Diagramas/07%20-%20Tramitacion%20de%20Pedidos%20-%20Credito.md)
- [08 — Tramitación de Pedidos — Prepago](Diagramas/08%20-%20Tramitacion%20de%20Pedidos%20-%20Prepago.md)

##### Canvas — Diagramas

- [01 — Flujo General de Tramitación](Diagramas/Canvas/01%20-%20Flujo%20General%20de%20Tramitacion.canvas)
- [02 — Flujo de Cobranza Prepago](Diagramas/Canvas/02%20-%20Flujo%20de%20Cobranza%20Prepago.canvas)
- [03 — Flujo de Factura por Adelantado](Diagramas/Canvas/03%20-%20Flujo%20de%20Factura%20por%20Adelantado.canvas)
- [04 — Flujo del Buzón de Pagos](Diagramas/Canvas/04%20-%20Flujo%20del%20Buzon%20de%20Pagos.canvas)
- [05 — Resumen de Rutas por Tipo de Cliente](Diagramas/Canvas/05%20-%20Resumen%20de%20Rutas%20por%20Tipo%20de%20Cliente.canvas)
- [06 — Roles y Responsabilidades](Diagramas/Canvas/06%20-%20Roles%20y%20Responsabilidades.canvas)
- [07 — Tramitación de Pedidos — Crédito](Diagramas/Canvas/07%20-%20Tramitacion%20de%20Pedidos%20-%20Credito.canvas)
- [08 — Tramitación de Pedidos — Prepago](Diagramas/Canvas/08%20-%20Tramitacion%20de%20Pedidos%20-%20Prepago.canvas)

---

#### 🗄️ Base de Datos — Diagramas ER

- [ER — ProquifaDotNet](Database/ER-ProquifaDotNet.md)
- [ER — Finanzas](Database/ER-Finanzas.md)
- [ER — Timbrado](Database/ER-Timbrado.md)
- [ER — EnvioCorreo](Database/ER-EnvioCorreo.md)
- [ER — LegacySync](Database/ER-LegacySync.md)
- [ER — PCconnect](Database/ER-PCconnect.md)
- [Script DDL — ProquifaDotNet](Database/ProquifaDotNet.sql)

---

#### 🔌 Endpoints

- [Endpoints — ProquifaDotNet](Endpoints/Endpoints-ProquifaDotNet.md)
- [Endpoints — Finanzas](Endpoints/Endpoints-Finanzas.md)
- [Endpoints — Timbrado](Endpoints/Endpoints-Timbrado.md)
- [Endpoints — EnvioCorreo](Endpoints/Endpoints-EnvioCorreo.md)
- [Endpoints — LegacySync](Endpoints/Endpoints-LegacySync.md)

---

#### 📋 Requisitos

- [Índice de Requisitos](Requisitos/Requisitos.MD)

---

#### 🔍 Revisión Matriz

- [Observaciones del Cliente](Revisi%C3%B3n%20Matriz/Observaciones%20Cliente.md)
- [Actualización de Matriz de Requisitos](Revisi%C3%B3n%20Matriz/Actualizacion%20de%20Matriz%20de%20Requisitos.md)
- [Reporte Revisión Matriz 2026-05-26](Revisi%C3%B3n%20Matriz/Reporte_Revision_Matriz_2026-05-26.md)
- [Tareas Modificadas](Revisi%C3%B3n%20Matriz/TareasModificadas.md)

---

#### 🎙️ Sesiones de Entendimiento

- [01 — R16 Adquisiciones — 1ra Sesión de Entendimiento (2026-03-20)](Sesiones%20de%20Entendimiento/01%20-%20R16%20Adquisiciones%20-%201ra%20sesión%20de%20entendimiento_%202026_03_20%2009_58%20CST%20-%20Notas%20de%20Gemini.md)
- [03 — R16 Tramitar Pedido sin Crédito — 3ra Sesión de Entendimiento (2026-04-06)](Sesiones%20de%20Entendimiento/03%20R16%20Tramitar%20Pedido%20sin%20Crédito%20-%203ra%20Sesión%20de%20Entendimiento_Presentación%20de%20pantallas_%202026_04_06%2014_55%20CST%20-%20Notas%20de%20Gemini.md)
- [04 — R16 Tramitar Pedido sin Crédito — Revisión de pantallas continuación (2026-04-08)](Sesiones%20de%20Entendimiento/04%20-%20R16%20Tramitar%20Pedido%20sin%20Crédito%20-%20Revisión%20de%20pantallas%20continuación_%202026_04_08%2015_52%20CST%20-%20Notas%20de%20Gemini.md)
- [05 — R16 Tramitar Pedido sin Crédito — Revisión de pantallas continuación (2026-04-09)](Sesiones%20de%20Entendimiento/05%20-%20R16%20Tramitar%20Pedido%20sin%20Crédito%20-%20Revisión%20de%20pantallas%20continuación_%202026_04_09%2014_59%20CST%20-%20Notas%20de%20Gemini.md)
- [06 — R16 Tramitar Pedido sin Crédito — Sesión de entendimiento (2026-04-14)](Sesiones%20de%20Entendimiento/06%20-%20R16%20Tramitar%20Pedido%20sin%20Crédito%20-%20Sesión%20de%20entendimiento_%202026_04_14%2015_52%20CST%20-%20Notas%20de%20Gemini.md)
- [07 — R16 Tramitar Pedido sin Crédito — 7ma Sesión de entendimiento (2026-04-15)](Sesiones%20de%20Entendimiento/07%20-%20R16%20Tramitar%20Pedido%20sin%20Crédito%20-%207ma%20Sesión%20de%20entendimiento_%202026_04_15%2014_55%20CST%20-%20Notas%20de%20Gemini.md)
- [R16 — Análisis de Documentos de Referencia](Sesiones%20de%20Entendimiento/R16%20-%20Análisis%20de%20Documentos%20de%20Referencia.md)
