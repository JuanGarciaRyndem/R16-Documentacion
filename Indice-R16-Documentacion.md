# Índice y Resumen — R16-Documentacion

> Generado automáticamente a partir del contenido completo de la carpeta `R16-Documentacion` (repo Obsidian/git). Sirve como mapa de navegación: para el detalle exacto siempre conviene abrir el archivo fuente citado.
> Fecha de generación: 2026-08-17.

## 0. Qué es el proyecto R16

Documentación técnica del proyecto **R16 — Adquisiciones / Tramitación de Pedidos** de **ProquifaDotNet**. Cubre análisis, diseño y desarrollo de los módulos de Tramitación de Pedidos, Cobranza, Facturación, Buzones y Notas de Crédito para las regiones **México** y **Perú**. Extiende el sistema de Venta Interna (.NET Framework 4.8) e incorpora soluciones nuevas en **.NET Core 10** (Finanzas, Timbrado, LegacySync, EnvíoCorreo). Fuente: `Inicio.md`, `README.md`.

---

## 1. Estado de los Requisitos — Cancelados vs. Activos

Esto responde directamente a "cuáles ya se cancelaron". Se verificó cruzando (a) el sufijo `-Cancelado` en el nombre de la carpeta, (b) el contenido de cada requisito, y (c) el historial de git del repo (fechas de commits) para confirmar que la cancelación sigue vigente y no fue revertida.

### 1.1 Requisitos confirmados como CANCELADOS

| Requisito | Nombre | Motivo de cancelación | Evidencia |
|---|---|---|---|
| **R16A-RE-FU-004** | Actualización de catálogos Régimen Fiscal y Tipo de Sociedad Mercantil (Cliente) | Cancelado explícitamente al liberar el Paquete 01 | `Analisis/Liberaciones/Liberacion_Paquete_01.md`: *"Requisito 004 — CANCELADO... no forma parte del alcance del Paquete 01"* |
| **R16A-RE-FU-008 (propuesta original / v1)** | Buzón de Cobros — versión inicial (Mailbot clásico) | Superseded — se rediseñó dos veces | Carpeta `R16A-RE-FU-008-Cancelado/` |
| **R16A-RE-FU-008 Propuesta 1** | Buzón de Cobros — n8n + RabbitMQ + Worker .NET 10 | Superseded por la Propuesta 2 (Gmail Push + Pub/Sub), que es la versión **activa hoy** en `R16A-RE-FU-008/` | Carpeta `R16A-RE-FU-008-P1-Cancelado/` |
| **R16A-RE-FU-020** | Factura por Adelantado — Detalle Perú | Se decidió sacar el timbrado fiscal y la facturación electrónica de Perú del alcance de R16 | Carpeta `R16A-RE-FU-020-Cancelado/`, `R16A-RE-FU-020-Cancelado.md` |
| **R16A-RE-FU-022** | Diseño y generación de Documentos: Factura Perú | Mismo motivo — Perú fuera de alcance | Carpeta `R16A-RE-FU-022-Cancelado/` |
| **R16A-RE-FU-031** | Diseño y generación de Documentos: CDP Perú | Mismo motivo — Perú fuera de alcance | Carpeta `R16A-RE-FU-031_Cancelado/` |
| **R16A-RE-FU-033** | Notas de Crédito: Perú | Mismo motivo — Perú fuera de alcance | Carpeta `R16A-RE-FU-033-Cancelado/` |
| **R16A-RE-FU-035** | Diseño y generación de Documentos: NDC Perú | Mismo motivo — Perú fuera de alcance | Carpeta `R16A-RE-FU-035-Cancelado/` |

**Resumen:** 6 requisitos numerados cancelados de forma definitiva (004, 020, 022, 031, 033, 035) — los 5 últimos son consecuencia directa de la decisión de **sacar Perú del alcance de timbrado/facturación electrónica** (ver `Analisis/Quitar Peru/`). Además, el requisito 008 tuvo **dos intentos de diseño cancelados** (v1 y Propuesta 1) antes de llegar a la versión activa actual (Propuesta 2).

### 1.2 Propuesta de cancelación descartada (el cliente NO la aceptó)

Existe un documento — `Analisis/Cambio Quitar Cobros( Cobranza )/Analisis Inicial.md` (fecha 2026-07-29, **Estado: Inicial**, borrador) — que **proponía eliminar 12 requisitos** del módulo de Cobranza/Validar Cobro y Notas de Crédito, delegando esa función a un "AplicativoExternoCobranza" aún sin definir:

> RE-023, RE-024, RE-025, RE-026, RE-027, RE-028, RE-029, RE-032, RE-033, RE-034, RE-035

**Confirmado (17-ago-2026): el cliente no aceptó este cambio de alcance.** Esto coincide con lo que ya se veía en el historial de git — las carpetas de estos requisitos nunca tomaron el sufijo `-Cancelado`, y hubo commits posteriores a esa fecha (3, 5, 6, 11, 12, 13 y 14 de agosto de 2026) actualizando activamente RE-FU-006, RE-FU-028 y RE-FU-034. **Se continúa con el módulo de Validar Cobro y Notas de Crédito (023–029, 032, 034) tal como estaba documentado antes de esta propuesta** — no hay cancelación aquí, todos estos requisitos siguen activos.

### 1.3 Todos los demás requisitos (001–003, 005–007, 009–019, 021, 023–030, 032, 034 y los NO-FU/Cambio-PerfilFiscal) están **activos**, con trabajo documentado reciente (julio–agosto 2026).

---

## 2. Índice de Análisis (`Analisis/`)

Documentos de análisis funcional/técnico, guías fiscales y decisiones de alcance. 43 archivos.

**Nivel raíz de Analisis:**
- `Borrador - Analisis de flujo y reglas de Tramitacion.md` — Define el comportamiento del sistema según tipo de cliente (Crédito/Prepago); referencia RE-FU-002/006/011/015.
- `Curacion-Datos-R16.md` (2026-06-19) — Identifica requisitos que requieren migrar/curar datos ya persistidos, no solo cambios DDL.
- `Estatus-Cobro-Ciclo-Vida.md` — Ciclo de vida del cobro desde que llega al Buzón hasta el cierre del Paso 3 de Validar Cobro (RE-008 → RE-023 → RE-024/025 → RE-026/027 → RE-028/029).
- `Foliados-Documentos.md` — Referencia centralizada de esquemas de foliado por tipo de documento en Finanzas.
- `Guia_Tecnica_Facturas_Ingreso_MX.md` / `...-Valde.md` (78K) — Guía técnica CFDI de Ingreso México (dos versiones/borradores).
- `Guia_Tecnica_Perfil_Fiscal_IVA_MX.md` — Explica los 2 ejes que determinan el tratamiento de IVA por línea facturada.
- `R16 - Análisis Consolidado de Sesiones.md` — Consolida las 6 sesiones de entendimiento (marzo–abril 2026).
- `R16 - Análisis de Documentos de Referencia.md` — Lista los documentos fuente analizados (checklists, Pyx4, etc.).
- `R16M_Resumen.md` (40K) — **Resumen ejecutivo de los 35 requisitos R16A-RE-FU-001 a 035**, formato ADP-FOR-13. Buen punto de partida para overview rápido.
- `Simulador-Recursos-R16-Servidor-20.22.md` — Estimación de impacto en CPU/RAM del release sobre el servidor 20.22.
- `Buzon-Cobros-P1-vs-P2.pptx`, `DAR-BuzonCobros-008.docx`, `[R16A-RE-FU][DIS-SOL] Diseño de la solución [Plantilla].pdf` — no-markdown (pptx/docx/pdf).

**Subcarpetas de Analisis:**
- `Cambio Quitar Cobros( Cobranza )/` — Ver sección 1.2 arriba (propuesta de eliminar el módulo de Cobranza, descartada por el cliente).
- `Estados de Pedidos/catEstadoPedido — Estados Propuestos.md` — Catálogo del ciclo de vida del pedido, con actividad muy reciente (11–14 ago 2026, últimos commits del repo).
- `Factura Empresa/` (3 docs) — Análisis de separación operativa PROQUIFA/GOLOCAER y su impacto en Back/BD, más contraste contra RE-FU-010.
- `Facturación/` (13 docs) — El bloque más grande: guías técnicas de Notas de Crédito, Facturas de Ingreso y Perfil Fiscal (MX y PE), diseño unificado `PerfilFiscal` MX+PE, addendas genéricas CFDI, flujo NC↔Validar Cobro, dudas pendientes para el cliente.
- `Liberaciones/Liberacion_Paquete_01.md` — Ver sección 1.
- `Quitar Peru/` (7 docs, incl. 3 docx) — El paquete de análisis que sustenta las cancelaciones de Perú (sección 1.1): decisión de dejar fuera timbrado/facturación Perú, mapeo requisito por requisito (RE-027 a 035), glosario de términos.

---

## 3. Índice de Requisitos (`Requisitos/`) — 39 carpetas, ~240 archivos .md

Cada requisito sigue el patrón: `RXXX.md` (historia de usuario), `-Back.md` (impacto backend), `_BD.md` (diccionario de datos), `-Tareas.md` (desglose de tareas), `[DIS-SOL]` (diseño de la solución, a veces docx/pdf), `_Revision.md` / `_DIS-SOL_Revision.md` (observaciones de revisión).

**No funcionales / transversales:**
- `R16A-NO-FU-001` — Migración de Envío de Correo a `ProquifaDotNet.SendInBlue` (.NET Core 10).
- `R16A-NO-FU-002` — Auditoría transaccional: `IdTransaccion` y agrupación de `BitacoraCRUD`.
- `R16A-RE-Cambio-PerfilFiscal` — Cambio transversal: configuración fiscal de producto a nivel Familia (MX+PE), afecta a RE-FU-000/019.

**Bloque Catálogos y Cliente (001–009):**
- `000` Perfil Fiscal · `001` Catálogo Cuentas Bancarias PROQUIFA · `002` Catálogo Clientes — campo Cobrador · `003` Documentos Regulatorios del Cliente · `004` **CANCELADO** (catálogos Régimen Fiscal/Tipo Sociedad) · `005` Catálogos Forma de Pago/Uso CFDI · `006` Referencia de Pago y Código Validador (muy activo, actividad hasta 2026-08-11/12) · `007` Notificación Regulatoria en Cotización Definitiva · `008` Buzón de Cobros/Mailbot v2 (activo — Propuesta 2 vigente; v1 y P1 cancelados) · `009` Validación Regulatoria en Tramitar Pedido.

**Bloque Tramitación de Pedidos (010–015):**
- `010` Crédito (flujo base) · `011` Crédito con sustancias controladas · `012` Crédito con Factura por Adelantado · `013` Prepago con controlados · `014` Prepago sin controlados sin FAA · `015` Prepago con FAA.

**Bloque Documentos/Proforma/Factura (016–022, 030, 034–035):**
- `016` Proforma México · `017` Proforma Perú · `018` Factura por Adelantado (pantalla inicial) · `019` Factura por Adelantado Detalle México · `020` **CANCELADO** (Detalle Perú) · `021` Documento Factura México · `022` **CANCELADO** (Documento Factura Perú) · `030` CDP México · `031` **CANCELADO** (CDP Perú) · `034` Documento NDC México · `035` **CANCELADO** (Documento NDC Perú).

**Bloque Validar Cobro (023–029) — activo; ver nota 1.2 (propuesta de quitarlo, descartada por el cliente):**
- `023` Pantalla principal · `024`/`025` Paso 1 (captura del cobro) MX/PE · `026`/`027` Paso 2 (asociación) MX/PE · `028`/`029` Paso 3 (facturación y envío) MX/PE.

**Bloque Notas de Crédito (032–033) — 032 activo; ver nota 1.2:**
- `032` Notas de Crédito México · `033` **CANCELADO** (Notas de Crédito Perú).

**Otros archivos sueltos en `Requisitos/`:** `Requisitos.MD` (índice general con links), `Catalogo BackEnd.md`, `Estandar Redaccion Tarea.md` (plantilla de formato de tareas), `historial_cambios/2026-08-14_reenfoques_paquete1.md` (reenfoques consolidados del Paquete 01 para 001–007, 009, 032 — confirma que 032 sigue vivo).

> Nota técnica: dentro de `Requisitos/` hay varios archivos `.fuse_hidden...` (lock files temporales de edición, sin contenido útil) — se ignoraron en este índice.

---

## 4. Otras carpetas del proyecto (resumen breve)

- **ADRs/** — `ADR-001-Estrategia-Identificadores-BD.md` (Aprobado, 2026-08-11): estrategia de identificadores en BD/aplicativo para proyectos nuevos.
- **Database/** — Diagramas ER (`.md` + `.mermaid`) por solución: ProquifaDotNet, Finanzas, Timbrado, LegacySync, PCconnect, EnvioCorreo; reglas de nombramiento; `ProquifaDotNet.sql` (dump de 8.7 MB).
- **Diagramas/** — 11 flowcharts Mermaid + carpeta `Canvas/` (12 `.canvas` de Obsidian + SVG/mmd) del flujo de tramitación, cobranza y buzones.
- **Diseño y Desarrollo/** — `Reglas al diseñar.md`: convenciones de idioma/codificación por tipo de BD.
- **Endpoints/** — Documentación de endpoints por aplicativo (ProquifaDotNet, Finanzas, Timbrado, LegacySync, EnvíoCorreo) + observaciones de convención (~62 endpoints nuevos propuestos).
- **Planeacion/** — `Dependencias-Requisitos.md`: análisis de dependencias y orden de ejecución de todos los requisitos.
- **Propuestas/** — Estándar de Proxy/Anti-Corruption Layer, propuesta de SDK compartido (RE-FU-008), inventario de puertos locales, y la carpeta `Bitacora/` + `Notificaciones/` (diseños de soluciones nuevas, C4).
- **ProquifaNet/** — Reglas de negocio de ProquifaDotNet, descripción de módulos Catálogos/Logística, changelog de rama `develop-pack04`.
- **Revisión Matriz/** — `Actualizacion de Matriz de Requisitos.md` (319 KB, la matriz completa) y `Observaciones Cliente.md` (87 KB) — las fuentes crudas más grandes del repo; `TareasModificadas.md` y `Reporte_Revision_Matriz_2026-05-26.md` resumen el trabajo de revisión.
- **Sesiones de Entendimiento/** — 6 actas (marzo–abril 2026) de las sesiones de entendimiento del módulo Tramitar Pedido sin Crédito.
- **Soluciones Nuevas/** — Ficha de alcance de las 4 soluciones .NET Core 10 nuevas: EnvioCorreo, Finanzas, LegacySync, Timbrado.

---

## 5. Cómo usar este índice

Este documento es un mapa, no un sustituto del contenido — los extractos son de ~300 caracteres. Cuando necesites el detalle real de un requisito, análisis o diseño, dime cuál y abro el archivo completo. Dado que el repo se sigue actualizando activamente (últimos commits: 14-ago-2026), conviene regenerar este índice cada cierto tiempo si quieres que siga reflejando el estado más reciente.
