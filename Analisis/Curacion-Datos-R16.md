# Curación de Datos R16 — Identificación y Estimación por Requisito

**Alcance:** Requisitos donde el cambio de reglas de negocio implica transformar, poblar o migrar datos existentes en ProquifaDotNet.  
**Fecha:** 2026-06-19

---

## ¿Qué es curación de datos en este contexto?

Un requisito requiere curación de datos cuando, además de los cambios DDL (CREATE/ALTER TABLE), existen **registros ya persistidos que deben modificarse** para ser compatibles con la nueva regla de negocio. Esto incluye:

- **Backfill de FK:** una nueva columna FK NOT NULL apunta a un catálogo nuevo; los registros existentes necesitan un valor válido.
- **Normalización de catálogos:** registros de catálogo existentes sin campo de región u otra dimensión nueva que debe asignarse.
- **Reclasificación de estados:** el sistema cambia la semántica de un campo (renombrar, recodificar).
- **Captura manual de datos maestros:** nuevos campos obligatorios en productos, unidades, empresas que solo el negocio puede llenar (códigos SAT, SUNAT, claves fiscales).
- **Integridad referencial bloqueante:** restricciones NOT NULL en tablas existentes que rompen si no se resuelven antes del ALTER TABLE.

---

## Clasificación de Requisitos

| #   | Requisito | Tipo de curación                                                                    | Riesgo     | Esfuerzo estimado                    |
| --- | --------- | ----------------------------------------------------------------------------------- | ---------- | ------------------------------------ |
| 1   | RE-001    | Backfill de IdRegion en `EmpresaDatosBancarios`                                     | 🟡 Medio   | Bajo (1 UPDATE automático)           |
| 2   | RE-005    | Normalización regional de 3 catálogos de facturación                                | 🔴 Alto    | Bajo-Medio (script + validación)     |
| 3   | RE-008    | Backfill `fccPagoCliente.IdCatCobroEstatus` + renombre clasificación correo         | 🔴 Alto    | Medio                                |
| 4   | RE-009    | `ppPedido.AceptaEntregasParciales` NOT NULL (DEFAULT automático)                    | 🟢 Bajo    | Mínimo                               |
| 5   | RE-015    | Backfill `tpPedido.IdEstatusPedido` en todos los pedidos históricos (OBS-027)       | 🔴 Crítico | Alto — BLOQUEANTE (pendiente cliente)|
| 6   | RE-019    | `tpProformaAdelanto.Enviada` NOT NULL DEFAULT(0) + revisión historial FAA           | 🟡 Medio   | Bajo                                 |
| 7   | RE-020    | Datos fiscales SUNAT en productos y unidades (curación manual)                      | 🔴 Alto    | Alto (negocio)                       |
| 8   | RE-021    | `ClaveProdServSAT` y `ClaveSAT` en productos y unidades (curación manual)           | 🔴 Alto    | Alto (negocio)                       |
| 9   | RE-023    | `tpPedido` campos de trazabilidad de cancelación (NULL — brecha de historial)       | 🟡 Medio   | Bajo (decisión sobre historial)      |
| 10  | RE-026    | UPDATE `catTipoInconsistenciaCobro.AplicaMarkPendienteCancelacion`                  | 🟡 Medio   | Mínimo                               |
| 11  | RE-028    | Backfill `CFDIGenerada.IdCatTipoCFDI` en facturas por adelantado históricas        | 🔴 Alto    | Medio (requiere decisión de negocio) |
| 12  | RE-029    | Backfill `catTipoCFDI.IdRegion` en entradas México + INSERT Perú                   | 🟡 Medio   | Bajo                                 |
| 13  | RE-032    | Brecha `fccNotaCredito.IdTPProformaPedido` NOT NULL vs NCs R16 sin pedido           | 🔴 Crítico | Medio-Alto (decisión arquitectónica) |

---

## Detalle por Requisito

---

### RE-001 — Mantenimiento de catálogos del sistema

**Tipo:** Backfill de dimensión regional en tabla existente.

**Contexto:** Se agrega `IdRegion` a `EmpresaDatosBancarios`. Todas las cuentas existentes pertenecen a México; se deben asignar antes de que la UI muestre datos por región.

**Script de curación:**
```sql
UPDATE dbo.EmpresaDatosBancarios
SET    IdRegion = (SELECT IdRegion FROM dbo.catRegion WHERE Clave = 'MEX'),
       FechaUltimaActualizacion = GETDATE()
WHERE  IdRegion IS NULL;
```

**Riesgo:** Bajo — el DEFAULT de México es correcto para todos los registros existentes. No hay ambigüedad.  
**Esfuerzo:** < 1 hora (script de validación + ejecución).  
**Prerequisito:** `catRegion` debe estar poblado y su clave `MEX` debe existir antes de ejecutar.

---

### RE-005 — Configuración de cobros y facturación (catálogos de facturación)

**Tipo:** Normalización regional de catálogos de facturación existentes.

**Contexto:** Se agrega `IdRegion` a tres catálogos que actualmente tienen entradas sin región. El cambio de regla es que a partir de R16, cada entrada de catálogo pertenece a una región específica. Los registros existentes son todos México.

**Tablas afectadas:**
- `catMetodoDePagoCFDI`
- `catUsoCFDI`
- `catMedioDePago`

**Script de curación:**
```sql
DECLARE @IdMexico uniqueidentifier = (SELECT IdRegion FROM dbo.catRegion WHERE Clave = 'MEX');

UPDATE dbo.catMetodoDePagoCFDI SET IdRegion = @IdMexico WHERE IdRegion IS NULL;
UPDATE dbo.catUsoCFDI          SET IdRegion = @IdMexico WHERE IdRegion IS NULL;
UPDATE dbo.catMedioDePago      SET IdRegion = @IdMexico WHERE IdRegion IS NULL;
```

**Riesgo:** Alto — si los catálogos no tienen región asignada, la UI no mostrará opciones para ningún cliente. Bloquea la facturación en producción.  
**Esfuerzo:** < 1 hora (script + verificación de conteos).  
**Prerequisito:** ALTER TABLE + catRegion poblada con `MEX` antes de ejecutar. Luego insertar entradas Perú por separado.

---

### RE-008 — Buzón de Cobros

**Tipo:** (a) Backfill de FK en tabla de cobros existente; (b) Reclasificación de nomenclatura en catálogo.

**Contexto A — `fccPagoCliente.IdCatCobroEstatus`:**  
Se introduce el catálogo `catCobroEstatus` y se agrega la FK `IdCatCobroEstatus` a `fccPagoCliente` como NOT NULL. Los registros existentes deben recibir el estatus que corresponde a su estado real. El DEFAULT del ALTER TABLE apunta a `BORRADOR`, pero los cobros confirmados históricos deberían tener `CAPTURADO` o `COMPLETADO`.

**Decisión de negocio requerida:** ¿Se acepta asignar `BORRADOR` a todos los registros históricos, o se hace un mapeo inteligente basado en `Confirmado` y existencia de registros en `fccPagoFacturaPedido`?

**Script de curación (opción con mapeo inteligente):**
```sql
DECLARE @IdBorrador    uniqueidentifier = (SELECT IdCatCobroEstatus FROM catCobroEstatus WHERE Clave = 'BORRADOR');
DECLARE @IdCapturado   uniqueidentifier = (SELECT IdCatCobroEstatus FROM catCobroEstatus WHERE Clave = 'CAPTURADO');
DECLARE @IdAsociado    uniqueidentifier = (SELECT IdCatCobroEstatus FROM catCobroEstatus WHERE Clave = 'ASOCIADO');
DECLARE @IdCompletado  uniqueidentifier = (SELECT IdCatCobroEstatus FROM catCobroEstatus WHERE Clave = 'COMPLETADO');

-- Cobros en borrador (no confirmados)
UPDATE fccPagoCliente SET IdCatCobroEstatus = @IdBorrador
WHERE Confirmado = 0 AND IdCatCobroEstatus = @IdBorrador; -- ya asignado por DEFAULT

-- Cobros confirmados sin asociación
UPDATE fccPagoCliente SET IdCatCobroEstatus = @IdCapturado
WHERE Confirmado = 1
  AND NOT EXISTS (SELECT 1 FROM fccPagoFacturaPedido WHERE IdFCCPagoCliente = fccPagoCliente.IdFCCPagoCliente)
  AND NOT EXISTS (SELECT 1 FROM fccPagoFacturaAdelanto WHERE IdFCCPagoCliente = fccPagoCliente.IdFCCPagoCliente);

-- Cobros confirmados con asociación
UPDATE fccPagoCliente SET IdCatCobroEstatus = @IdAsociado
WHERE Confirmado = 1
  AND EXISTS (SELECT 1 FROM fccPagoFacturaPedido WHERE IdFCCPagoCliente = fccPagoCliente.IdFCCPagoCliente);
```

**Contexto B — `catClasificacionCorreoRecibido`:**  
La clasificación `'pago'` debe renombrarse a `'cobro'` para alinearse con la nueva terminología. Si el valor se usa en otros módulos activos, se debe insertar un registro nuevo en lugar de actualizar.

**Esfuerzo total:** Medio (2–4 horas: validación de conteos, script de mapeo, verificación post-ejecución).  
**Riesgo:** Alto — el ALTER TABLE NOT NULL sin curación previa falla en producción si no hay DEFAULT. El DEFAULT está definido como BORRADOR, lo que es seguro para ejecutar pero deja datos históricos en estado incorrecto.

---

### RE-009 — Validación Regulatoria

**Tipo:** Columna NOT NULL con DEFAULT estático.

**Contexto:** Se agrega `AceptaEntregasParciales bit NOT NULL DEFAULT(0)` a `ppPedido`. SQL Server asigna el valor 0 automáticamente a los registros existentes. No requiere script adicional.

También se agrega `IdPedidoOrigenControlado` nullable a `tpPedido` como autorreferencia. Es NULL para registros históricos, lo que es correcto.

**Riesgo:** Bajo — ambas columnas tienen comportamiento seguro para datos existentes.  
**Esfuerzo:** Mínimo (validar conteos antes y después del ALTER).

---

### RE-015 — Tramitación de Pedidos Prepago con Factura por Adelantado (OBS-027)

**Tipo:** Backfill masivo de estatus en todos los pedidos históricos.

**Contexto:** Se requiere definir el catálogo `catEstatusPedido` y agregar `tpPedido.IdEstatusPedido` (FK) para gestionar el ciclo de vida del pedido en R16. El campo actualmente no existe — `tpPedido` solo tiene `Tramitado bit` y campos de auditoría. Cuando se implemente, **todos los pedidos existentes** (`tpPedido`) necesitarán que se les asigne el estatus que les corresponde.

**⚠️ Estado actual: BLOQUEANTE — OBS-027 pendiente de propuesta del cliente.**  
El cliente debe definir los estados, claves, estados terminales y transiciones permitidas antes de poder diseñar el script de curación.

**Complejidad del backfill (una vez desbloqueado):**

El mapeo base desde los campos actuales de `tpPedido` sería:

| Condición actual en tpPedido | Estado probable | Notas |
|---|---|---|
| `Tramitado = 0` | `PENDIENTE` o `EN_TRAMITACION` | Pre-pedido sin tramitar |
| `Tramitado = 1` + proforma activa + sin cobro | `TRAMITADO` | En espera de pago |
| `Tramitado = 1` + cobro registrado | `COBRADO` | Depende del diseño final |
| Pedido cancelado por falta de pago | `CANCELADO` | Actualmente no hay campo explícito en `tpPedido` |
| Pedido entregado | `ENTREGADO` | Requiere confirmar campo fuente en BD |

**Riesgo:** Crítico — `tpPedido` es la tabla central del sistema. Un backfill incorrecto afecta el comportamiento de todos los módulos que lean el nuevo `IdEstatusPedido`.  
**Esfuerzo:** Alto (diseño del mapeo: 4–8 horas; validación con negocio: 1–2 días; ejecución + QA: 2–4 horas).  
**Prerequisito:** Propuesta del cliente con definición de `catEstatusPedido` + confirmación de si aplica también a `ppPedido`.

---

### RE-023 — Validar Cobro (campos de cancelación en `tpPedido`)

**Tipo:** Columnas nullable nuevas — sin curación técnica obligatoria, pero con brecha de historial.

**Contexto:** Se agregan a `tpPedido` cuatro columnas de trazabilidad de cancelación, todas **NULL**:
- `FechaCancelacionPorFaltaPago datetime2 NULL`
- `IdUsuarioCancelacion uniqueidentifier NULL`
- `FechaSolicitudCancelacion datetime2 NULL` (OBS-042)
- `EstadoCancelacionCFDI varchar(50) NULL` (OBS-042)

El ALTER TABLE es seguro para datos existentes — SQL Server rellena NULL automáticamente. Sin embargo, los pedidos cancelados antes de R16 quedarán con estos campos en NULL indefinidamente, lo que genera una **brecha de historial** en reportes de auditoría y cancelaciones.

**Decisión de negocio requerida:** ¿Se necesita reconstruir el historial de cancelaciones previas a R16? Si sí:
- `FechaCancelacionPorFaltaPago` — requiere cruzar contra registros de cancelación en Legacy o tablas de auditoría.
- `EstadoCancelacionCFDI` — solo aplica a CFDIs ya cancelados ante el SAT; el estado final sería `'Cancelado'` pero no hay fuente confiable del estado intermedio.

**Riesgo:** Medio — el módulo de Validar Cobro funciona correctamente sin el backfill. El riesgo es de auditoría e informes históricos.  
**Esfuerzo:** Bajo si se acepta NULL para historial. Medio-alto si se requiere reconstrucción desde Legacy (consulta no trivial).

---

### RE-019 — Factura por Adelantado: Detalle México

**Tipo:** Columna NOT NULL con DEFAULT + posible curación de historial.

**Contexto:** Se agrega `Enviada bit NOT NULL DEFAULT(0)` a `tpProformaAdelanto`. SQL Server pone 0 automáticamente. Sin embargo, las facturas por adelantado que ya fueron enviadas antes de R16 quedarán con `Enviada = 0`, lo que puede afectar la vista de historial si el módulo muestra el estado de envío.

**Decisión de negocio requerida:** ¿Las FAA previas al release se consideran "enviadas" o no importa el historial? Si importa:

```sql
-- Marcar como enviadas las FAA que ya tienen IdCFDIGenerada (timbradas y enviadas)
UPDATE dbo.tpProformaAdelanto
SET Enviada = 1
WHERE IdCFDIGenerada IS NOT NULL;
```

**Riesgo:** Medio — el ALTER es seguro, pero el historial puede quedar con estado incorrecto si no se toma decisión sobre registros previos.  
**Esfuerzo:** Bajo (30 min de análisis + script simple).

---

### RE-020 — Factura por Adelantado: Detalle Perú

**Tipo:** Curación manual de datos maestros (catálogo SUNAT en productos y unidades).

**Contexto:** Para generar facturas electrónicas Perú se requieren:
- `Producto.CodigoSUNAT` — Catálogo 25 SUNAT (código de producto fiscal peruano)
- `catUnidad.ClaveSUNAT` — Catálogo 6 SUNAT (unidad de medida, ej: KGM, NIU, ZZ)
- `Producto.IdCatAfectacionIGV` — Catálogo 7 SUNAT (afectación al IGV: gravado, exonerado, inafecto)

Estos datos **no existen en el sistema** y deben ser provistos por el área de operaciones o el negocio peruano. El ALTER TABLE crea las columnas como NULL, pero si se genera una factura Perú sin estos valores, el XML CPE será inválido.

**No es curación técnica: es curación funcional por el negocio.**

**Estimación:**
- Número de productos activos que aplican a Perú: pendiente de determinar por el negocio
- Fuente: catálogos SUNAT publicados (sunat.gob.pe)
- Proceso: carga masiva (Excel → script UPDATE o pantalla de administración)

**Riesgo:** Alto — sin estos datos, el módulo de facturación Perú no puede operar. Es bloqueante para el Go Live de RE-020/022/025/027/029.  
**Esfuerzo:** Alto (negocio: 2–5 días de captura; IT: 2–4 horas para script de carga o pantalla de mantenimiento).

---

### RE-021 — Factura México (CFDI 4.0)

**Tipo:** Curación manual de datos maestros (catálogos SAT en productos y unidades).

**Contexto:** Para emitir CFDI 4.0 se requieren campos obligatorios del SAT:
- `Producto.ClaveProdServSAT` — Catálogo c_ClaveProdServ del SAT (~72,000 entradas)
- `catUnidad.ClaveSAT` — Catálogo c_ClaveUnidad del SAT (ej: KGM, H87, LTR)
- `CFDIGenerada.Exportacion` — Campo CFDI 4.0 obligatorio (DEFAULT `'01'` = no aplica, automático)

Los dos primeros requieren que el negocio o el área fiscal asigne la clave SAT a cada producto y unidad del catálogo.

**No es curación técnica: es curación funcional crítica para cumplimiento fiscal.**

**Estimación:**
- Catálogo de productos ProquifaDotNet: pendiente de contar (`SELECT COUNT(*) FROM Producto WHERE Activo=1`)
- Catálogo de unidades: pendiente de contar (`SELECT COUNT(*) FROM catUnidad WHERE Activo=1`)
- Fuente: SAT — catálogo c_ClaveProdServ publicado en sat.gob.mx
- Proceso recomendado: pantalla de mantenimiento o carga masiva desde Excel

**Riesgo:** Alto — sin `ClaveProdServSAT`, la generación de CFDI 4.0 falla en el servicio de timbrado SAP/PAC. Es bloqueante para México.  
**Esfuerzo:** Alto (negocio: 3–10 días según volumen; IT: 2–4 horas para herramienta de carga).

---

### RE-026 — Validar Cobro: Paso 2 México

**Tipo:** UPDATE de catálogo para activar nueva regla de negocio.

**Contexto:** Se agrega `AplicaMarkPendienteCancelacion bit` a `catTipoInconsistenciaCobro`. Solo el tipo `PAGO_INCOMPLETO_VENCIDO` debe activar el marcado de cancelación en `tpPedido`.

**Script de curación:**
```sql
UPDATE dbo.catTipoInconsistenciaCobro
SET AplicaMarkPendienteCancelacion = 1
WHERE Clave = 'PAGO_INCOMPLETO_VENCIDO';

UPDATE dbo.catTipoInconsistenciaCobro
SET AplicaMarkPendienteCancelacion = 0
WHERE AplicaMarkPendienteCancelacion IS NULL;
```

**Riesgo:** Medio — si el flag queda en NULL o en valor incorrecto, el módulo puede marcar o no marcar pedidos incorrectamente.  
**Esfuerzo:** Mínimo (< 30 minutos).

---

### RE-028 — Validar Cobro: Paso 3 México

**Tipo:** Backfill de FK discriminadora en registros históricos.

**Contexto:** Se agrega `IdCatTipoCFDI` a `CFDIGenerada` para discriminar el tipo de CFDI (PPD, PUE, Anticipo, Complemento). Las facturas por adelantado generadas en RE-019 (antes de R16) quedarán con `IdCatTipoCFDI = NULL`. El script de normalización debe asignarles el tipo correcto.

**Decisión de negocio requerida:** ¿Las FAA previas son PPD o PUE? La respuesta define el script de curación.

**Script de curación (validar con el negocio antes de ejecutar):**
```sql
-- Asignar tipo a CFDIs existentes (FAA = normalmente PPD)
UPDATE dbo.CFDIGenerada
SET IdCatTipoCFDI = (SELECT IdCatTipoCFDI FROM dbo.catTipoCFDI WHERE Clave = 'FACTURA_PPD')
WHERE IdCatTipoCFDI IS NULL;
-- ⚠️ Confirmar con el área fiscal si las FAA previas son PPD o PUE antes de ejecutar
```

**Riesgo:** Alto — una asignación incorrecta de tipo CFDI afecta la correcta generación del Complemento de Pago en el módulo RE-030.  
**Esfuerzo:** Medio (requiere consulta al área fiscal + validación de conteos).

---

### RE-029 — Validar Cobro: Paso 3 Perú

**Tipo:** Backfill de IdRegion en catálogo existente.

**Contexto:** `catTipoCFDI` se convierte en un catálogo regional. Los registros México insertados en RE-028 (`FACTURA_PPD`, `FACTURA_PUE`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO`) no tienen `IdRegion` porque la columna no existía. RE-029 agrega la columna y la puebla.

**Script de curación:**
```sql
-- Poblar IdRegion = México en entradas existentes
UPDATE dbo.catTipoCFDI
SET IdRegion = (SELECT IdRegion FROM dbo.catRegion WHERE Clave = 'MEX')
WHERE Clave IN ('FACTURA_PPD', 'FACTURA_PUE', 'FACTURA_ANTICIPO', 'COMPLEMENTO_PAGO');

-- Insertar entrada Perú
INSERT INTO dbo.catTipoCFDI (Clave, Descripcion, IdRegion)
SELECT 'FACTURA_CPE', 'Factura electrónica SUNAT — CPE tipo 01 UBL 2.1 (Perú)',
       IdRegion FROM dbo.catRegion WHERE Clave = 'PER';
```

**Riesgo:** Medio — sin el backfill, el módulo de Paso 3 Perú no puede filtrar tipos de documento por región.  
**Esfuerzo:** Bajo (< 1 hora).

---

### RE-032 — Notas de Crédito México

**Tipo:** Brecha de integridad referencial bloqueante.

**Contexto:** `fccNotaCredito.IdTPProformaPedido` es actualmente `NOT NULL` porque las NCs legacy siempre vienen de un pedido/proforma. Las NCs R16 se generan de forma independiente, sin pedido origen. Si la restricción NOT NULL permanece, el INSERT de una NC R16 fallará.

**⚠️ Esto NO es un UPDATE de curación: es una decisión de diseño que debe resolverse antes del ALTER TABLE.**

**Opciones:**
1. **Hacer la columna nullable** (recomendado): `ALTER TABLE fccNotaCredito ALTER COLUMN IdTPProformaPedido uniqueidentifier NULL` — permite NCs sin pedido origen. Requiere validar que ningún proceso legacy dependa de que siempre sea NOT NULL.
2. **Valor placeholder:** insertar un `tpProformaPedido` ficticio de "NC R16 sin pedido" y asignarlo a todas las NCs nuevas — no recomendado (ensucia los datos).

**Riesgo:** Crítico — sin resolver esta brecha, el módulo de Notas de Crédito R16 no puede persistir registros.  
**Esfuerzo:** Medio-Alto (análisis de impacto en módulos existentes que consumen NCs + decisión técnica + ejecución).

---

## Resumen Ejecutivo

### Curación técnica (scripts listos para ejecutar)

| Requisito | Tabla | Acción | Prerequisito |
|---|---|---|---|
| RE-001 | `EmpresaDatosBancarios` | UPDATE SET IdRegion = MEX | `catRegion` poblada |
| RE-005 | `catMetodoDePagoCFDI`, `catUsoCFDI`, `catMedioDePago` | UPDATE SET IdRegion = MEX | `catRegion` poblada; ALTER TABLE ejecutado |
| RE-008 | `fccPagoCliente` | UPDATE SET IdCatCobroEstatus según mapeo de Confirmado | `catCobroEstatus` creado y con datos |
| RE-008 | `catClasificacionCorreoRecibido` | UPDATE/INSERT reclasificación 'pago' → 'cobro' | Validar uso en otros módulos |
| RE-026 | `catTipoInconsistenciaCobro` | UPDATE SET AplicaMarkPendienteCancelacion | ALTER TABLE ejecutado |
| RE-029 | `catTipoCFDI` | UPDATE SET IdRegion = MEX en entradas MX | ALTER TABLE ejecutado |
| RE-029 | `catTipoCFDI` | INSERT entrada `FACTURA_CPE` Perú | `catRegion` con PER |

### Curación funcional (requiere acción del negocio)

| Requisito | Tabla | Campo | Responsable | Bloqueante para |
|---|---|---|---|---|
| RE-020 | `Producto` | `CodigoSUNAT` | Área fiscal Perú | Facturación Perú (RE-020/022/025/027/029) |
| RE-020 | `catUnidad` | `ClaveSUNAT` | Área fiscal Perú | Facturación Perú |
| RE-020 | `catAfectacionIGV` | Poblar catálogo | Área fiscal Perú | Facturación Perú |
| RE-021 | `Producto` | `ClaveProdServSAT` | Área fiscal México | Facturación México (RE-021/028) |
| RE-021 | `catUnidad` | `ClaveSAT` | Área fiscal México | Facturación México |

### Brechas bloqueantes sin resolver

| # | Brecha | Requisito | Acción requerida |
|---|---|---|---|
| B1 | `tpPedido.IdEstatusPedido`: catálogo y campo pendientes de propuesta del cliente — backfill masivo requerido | RE-015 | Propuesta del cliente → diseño de mapeo → script |
| B2 | `fccNotaCredito.IdTPProformaPedido` NOT NULL impide INSERT de NCs R16 | RE-032 | Decisión: hacer nullable o placeholder |
| B3 | `CFDIGenerada.IdCatTipoCFDI` backfill: ¿PPD o PUE para FAAs históricas? | RE-028 | Consultar área fiscal antes del script |
| B4 | `tpProformaAdelanto.Enviada`: ¿se marcan como enviadas las FAA históricas? | RE-019 | Consultar área operativa |
| B5 | `tpPedido` campos de cancelación: ¿se reconstruye historial pre-R16? | RE-023 | Decisión del negocio sobre auditoría histórica |
| B6 | Ningún proceso de mantenimiento de `ClaveProdServSAT` definido en la UI | RE-021 | Definir pantalla de administración o carga masiva |

---

*Generado: 2026-06-19 | Versión 1.0*
