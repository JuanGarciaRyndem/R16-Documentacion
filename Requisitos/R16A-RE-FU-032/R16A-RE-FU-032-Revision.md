# R16A-RE-FU-032 — Revisión de cobertura — Requisito vs Back vs BD

**Fecha:** 2026-07-27
**Documentos revisados:** `R16A-RE-FU-032.md` (funcional) contra `R16A-RE-FU-032-Back.md` y `R16A-RE-FU-032_BD.md`.

---

## 1. Inconsistencias entre Back y BD (corregir)

### 1.1 `catMotivoCancelacionSAT` — el Back dice que no existe; la BD ya la crea ✅ APLICADO

**Resuelto (2026-07-27):** se agregó la sección A7 al Back (CREATE TABLE + DML, referenciando el BD cambio #8), el combo de B2 lee del catálogo vía el nuevo endpoint B9 (`GET /api/v1/credit-note/cancellation-reasons`, servicio de solo lectura en Finanzas), y la Brecha B4 quedó marcada como resuelta.

El documento de BD incluye el cambio #8 (`CREATE TABLE catMotivoCancelacionSAT` + DML con las 4 claves SAT y diccionario de datos completo), pero el Back mantiene la **Brecha B4** diciendo que la tabla no existe y que el front debe hardcodear el catálogo. Además, la Parte A del Back (A1–A6) no tiene sección para este cambio.

**Acción:** agregar sección A7 en el Back (CREATE TABLE + DML catMotivoCancelacionSAT), actualizar el combo del Criterio G3/B2 para leer de esta tabla, y eliminar o marcar resuelta la Brecha B4.

### 1.2 Estado de la NC — dominio incompleto en BD ✅ APLICADO

**Resuelto (2026-07-27):** se creó el catálogo `catNotaCreditoEstado` (BD cambio #9, Back sección A8) con 4 estados: PENDIENTE (default, capturada sin timbrar), VIGENTE (timbrada), ENVIADA (correo enviado) y CANCELADA (terminal). `fccNotaCredito.Estado` se reemplazó por FK `IdCatNotaCreditoEstado`; se actualizaron el flujo B6 (VIGENTE al persistir, ENVIADA tras el correo), las queries de B7/F3 y el ciclo mencionado en Infraestructura reutilizada.

- Back B2/B3: la NC se crea en estado **PENDIENTE** (pre-timbrado) y pasa a VIGENTE tras persistencia post-timbrado.
- BD `fccNotaCredito.Estado`: dominio documentado `'VIGENTE' | 'CANCELADA'`, default `'VIGENTE'`.

El dominio de BD no incluye `PENDIENTE`, y el default VIGENTE contradice el flujo (una NC recién capturada quedaría "vigente" sin timbrar). La fila de "Infraestructura reutilizada" del Back menciona además un ciclo `PENDIENTE → GENERADA → ENVIADA` que no coincide con ninguno de los dos.

**Acción:** unificar el ciclo de estados (propuesta: `PENDIENTE → VIGENTE → CANCELADA`, default `PENDIENTE`) en ambos documentos y en el CHECK/documentación de la columna.

### 1.3 Base de datos del DML `EmpresaFolio` — encabezado contradictorio ✅ APLICADO

**Resuelto (2026-07-27):** el script se corrigió a "Ejecutar en ProquifaDotNet" y se agregó nota aclaratoria: la base de datos es una sola (`ProquifaDotNet`); "propiedad Finanzas" significa que la tabla la consume la solución Finanzas vía su Scaffold EF Core, no que exista una base separada.

**Complemento:** `ProquifaDotNetTimbrado` sí es base de datos aparte (solución Timbrado). Se documentó su uso en RE-032: `AppSetting` (lectura de configuración) y `StampingLog` (INSERT por cada Stamp/Cancel de la NC, con `CfdiGeneradaId` como referencia cross-database sin FK); los flujos C1/C2 del Back registran ahora el log correspondiente.

En el documento de BD, la sección del DML Serie "P2" dice en prosa "Se insertan 4 filas en `ProquifaDotNet.EmpresaFolio`", pero el comentario del script dice "Ejecutar en **ProquifaDotNetTimbrado**". El Back (A5) dice ProquifaDotNet (propiedad Finanzas).

**Acción:** confirmar la base real de `EmpresaFolio` (según RE-019) y corregir el script.

### 1.4 Pendiente P7 del Back ya resuelto en el propio Back ✅ APLICADO

**Resuelto (2026-07-27):** P7 marcado como resuelto en los Pendientes del Back — el endpoint `POST /api/v1/stamp/cancel` pertenece a RE-FU-018 (allí se crea como tarea propia) y RE-032 solo lo referencia como dependencia/reutilización en C2, conforme a la directriz de no duplicar tareas entre requisitos.

El Back lista P7 "Confirmar si endpoint Cancelar CFDI ya existe en Timbrado o debe crearse", pero la sección C2 ya afirma que `POST /api/v1/stamp/cancel` existe desde RE-FU-018 y se reutiliza. **Acción:** cerrar P7 o quitar la afirmación de C2.

### 1.5 Bucket MinIO sin sección en Parte A del Back ✅ APLICADO

**Resuelto (2026-07-27):** se agregó la sección A9 (DML `RegionConfiguracionMinioBucket`) y se ajustó el patrón de subida: Finanzas no accede a MinIO directamente — envía la petición a `PQF.Catalogos.SubirArchivo` (WebApi.Catalogos de ProquifaDotNet) con la clave del bucket y el archivo; ProquifaDotNet resuelve el bucket, sube a MinIO, inserta `Archivo` y retorna el `IdArchivo`. Actualizados B6 paso 4, Parte E (E1/E3) y la tabla de distribución.

La tabla de distribución del Back incluye "BD — DML bucket `RegionConfiguracionMinioBucket`", y el DML completo vive en el doc de BD, pero la Parte A del Back no tiene sección (A1–A6 no lo cubren). **Acción:** agregar la referencia como A8 (o dentro de E1) por trazabilidad.

---

## 2. Elementos del requisito sin cobertura explícita en Back/BD (brechas funcionales)

### 2.1 Columna "Cobrador" y usuario creador de la NC

El Criterio B2 exige columna Cobrador en el drill-down y el Criterio A5 filtra por cartera del Cobrador, pero las columnas nuevas de `fccNotaCredito` no incluyen ningún usuario (creador ni cobrador). No está definida la fuente del dato "Cobrador" por NC (¿usuario autenticado que la generó? ¿cobrador de cartera del cliente al momento de emisión?). **Acción:** definir fuente; si es el usuario creador, agregar columna (ej. `IdUsuarioRegistro`) al ALTER de A1.

### 2.2 Cabecera del Paso 2 (Criterio D1) — datos no retornados por B1

D1 exige mostrar de la factura origen: Tipo CFDI, RFC receptor, Razón Social, IVA y Estado, pero el retorno documentado del endpoint B1 (Paso 1) solo incluye Folio, UUID, FechaEmision, Subtotal, Total y Moneda. **Acción:** ampliar el retorno de B1 o definir endpoint de detalle de factura para el Paso 2.

### 2.3 Endpoint de detalle de NC (Paso 4 / Criterios K1, K4)

K1 exige la vista completa (datos del PAC: número de certificado SAT, RFC del SAT, versión CFDI, etc.) y K4 exige que la consulta desde el listado use la misma vista. El Back documenta la navegación al Paso 4 (B6 paso 7) y el drill-down (B7), pero no el endpoint de consulta del detalle completo de una NC ya emitida. **Acción:** agregarlo a B7 o como B9.

### 2.4 Filtros de listados (Criterios A2, B1, C3)

Los filtros exigidos (pantalla principal: Cliente/Moneda/Fecha; drill-down: Fecha/Emisor/Estado/Moneda/buscador folio-UUID; Paso 1: Fecha/Moneda/buscador) están solo parcialmente mencionados en B7 ("Filtros adicionales: Moneda, Fecha") y ausentes en B1. **Acción:** documentar los parámetros de filtrado de cada endpoint.

### 2.5 FormaPago '15 — Condonación' (Criterio H3)

H3 contempla el escenario de factura origen no pagada → FormaPago '15'. El Back B4 solo documenta la herencia del '03'. **Acción:** agregar el caso a B4 (aunque sea escenario raro en R16).

### 2.6 Reglas SAT de decimales (Criterio N5 — PMO #55)

N3/N4/N5 (precisión interna 6–8 decimales, 4 decimales en UI, reglas SAT de redondeo en CFDI 4.0) no aparecen en el Back ni en sus pendientes. Los tipos `decimal(18,6)` de BD dan 6 decimales (dentro del rango N3). **Acción:** agregar PMO #55 a los Pendientes del Back y validar el redondeo en la sección B4 (armado del XML).

### 2.7 Multi-divisa EUR (alcance "MXN/USD/EUR")

`fccNotaCredito` existente tiene indicadores `MXN`/`USD` (bit) y montos convertidos `MontoUSD`/`MontoMXN`; no hay representación para EUR. La moneda real viaja en `CFDIGenerada.Moneda`, pero si los indicadores/columnas de conversión se siguen usando en consultas (pantalla principal agrupa por moneda), EUR queda sin soporte. **Acción:** confirmar si los bits de moneda son legacy sin uso en R16 o si se requiere ajuste.

### 2.8 Catálogo de motivos de la NC (Criterio D2)

El combo de Motivo (Devolución de mercancía / Descuento o bonificación) no tiene catálogo en BD — `fccNotaCredito.Motivo` es varchar con claves sugeridas ('DEVOLUCION', 'DESCUENTO_BONIFICACION'). Aceptable como dominio fijo en front/back, pero conviene declararlo explícitamente (mismo criterio que se aplicó a `Modalidad`).

---

## 3. Cobertura verificada (sin observación)

| Sección del requisito | Cobertura |
| --- | --- |
| A1/A3/A4 — Pantalla principal agrupada + wizard + drill-down | Back B7 (query con `vUsuarioCartera`, OBS-004) |
| C1–C5 — Paso 1 Buscar Factura (vigentes, prepago, 5 años, una sola factura) | Back B1 |
| D2–D5 — Motivo, bifurcación de modalidad, G02, TipoRelacion 01 | Back B2/B3/B4 |
| E1–E5 — Modalidad por partidas, cálculo en tiempo real, herencia de conceptos, solo Cant. NC > 0 | Back B2 + B4; BD `fccNotaCreditoPartida` (columnas R16) |
| F1–F5 — Modalidad manual, monto máximo, concepto obligatorio, claves default | Back B3 + B4; BD `ConceptoManual`/`ObservacionesManual` |
| G1–G4 — Cancelación condicional (totalidad + mismo mes, siempre total) | Back B2 + B6 paso 3 + C2; BD `CancelarFacturaOrigen`, `ClaveMotivosCancelacion`, `CFDICancelacion` |
| H1–H2, H4–H7 — Campos fiscales del XML | Back B4 + contrato C1.1 (`CreditNoteRequest` sin IDs de negocio) |
| I1–I3, I6 — Paso 3, previsualización PDF, Timbrar | Back B5 (preview sin sello/UUID) + B6 |
| J1–J5 — Timbrado, feedback, persistencia, correo, errores PAC | Back B6 + C1; BD `Archivo`, MinIO, `CorreoEnviado` |
| K2–K3 — Banner y acciones (descargas, reenvío) | Back B6/B8 + MinIO E2 |
| L1–L4 — Reenvío de correo | Back B8 + B6 paso 5 (asunto pendiente PMO #31) |
| M1–M2 — Acoplamiento uni-direccional Validar Cobro | Back (distribución) + BD `fccNotaCreditoPedido` sin cambios |
| N1–N2 — Foliado Serie P2, UUID SAT | Back C1 + BD DML `EmpresaFolio` (pendiente P3 PMO) |
| N6 / Regla 14 — Conservación XML 5 años | MinIO + nota legal en PDF (B5); sin política de retención explícita, cubierta por persistencia permanente |
| Reglas 3, 10, 11, 12, 16 | B1 (filtros), sin importación Legacy (no hay ETL inverso), inmutabilidad (sin endpoints de edición), C1 (manejo de errores) |
| Pendientes funcionales (FormaPago manual, claves manual, PMO #31, foliador) | Reflejados en Pendientes del Back (P3, P4, P5, P8) y BD (P3, P5, P6) |

---

## 4. Resumen de acciones

1. Back: agregar A7 `catMotivoCancelacionSAT` y cerrar Brecha B4 (inconsistencia con BD #8).
2. Ambos: unificar ciclo de estados de `fccNotaCredito` incluyendo `PENDIENTE` (default) y corregir la mención `PENDIENTE → GENERADA → ENVIADA`.
3. BD: corregir base de datos destino del DML `EmpresaFolio` (ProquifaDotNet vs ProquifaDotNetTimbrado).
4. Back: cerrar Pendiente P7 (endpoint cancel ya existe según C2).
5. Back: definir fuente de "Cobrador" por NC y, si aplica, columna de usuario creador en A1.
6. Back: ampliar retorno de B1 (o endpoint de detalle de factura) para la cabecera del Paso 2 (D1).
7. Back: documentar endpoint de detalle de NC (K1/K4) y los filtros de A2/B1/C3.
8. Back: agregar FormaPago '15' (H3) y PMO #55 decimales (N5) a B4/Pendientes.
9. Confirmar soporte EUR frente a los indicadores `MXN`/`USD` existentes.
