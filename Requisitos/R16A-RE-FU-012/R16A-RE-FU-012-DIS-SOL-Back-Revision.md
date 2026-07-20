# Revisión — Diseño de Solución Backend R16A-RE-FU-012

| Campo | Valor |
|---|---|
| **Documento revisado** | R16A-RE-FU-012-DIS-SOL-Back.pdf |
| **Requisito de referencia** | R16A-RE-FU-012.md |
| **Fecha de revisión** | 2026-07-20 |
| **Revisor** | Juan David García Cruz |

---

## ✅ Bien cubierto

**Perú sin Legacy y sin FAA** — Ambas reglas del requisito están correctamente reflejadas:

- El helper `FacturaPorAdelantadoElegibilidadHelper` rechaza explícitamente `claveRegion != MX` con el mensaje adecuado, bloqueando FAA para Perú desde backend.
- El flowchart de LegacySync tiene el nodo `CReg -- No: Perú → PEComplete[Marcar Completado sin transferir a Legacy]`, cubriendo la Regla 7 y CA-C6.
- La prueba de "Corte Regional Perú" en §6.1 valida ambos escenarios.

**Bypass código de autorización (Regla 3 / PA-3)** — Documentado como impacto en `SolicitudAutorizacionBO.cs` y explicado en §1.3.

**Atomicidad del pendiente FAA** — La inserción en `fccFactura` / `fccFacturaPartida` / `fccFacturaReferenciaBancaria` dentro de la misma transacción cubre la Regla 5 y CA-B2.

**Bloqueo de datos fiscales** — §3.2 cubre CA-B1 con validación defensiva.

---

## ⚠️ Problemas y brechas

### 1. Dependencia de RE-FU-015 sin marcar como bloqueante

El diseño indica "se reutiliza el esquema creado en RE-FU-015" para la tabla `SyncControl`, pero RE-FU-015 no aparece en §1.3 como dependencia. Si RE-FU-012 se implementa antes que RE-FU-015, la tabla no existe.

**Acción:** Agregar `DEP-015` en §1.3 con impacto bloqueante.

---

### 2. CA-C4 (Pago contra entrega + marca de detención) no mapeado como bloqueante

El Criterio C4 del requisito exige que la transferencia a Legacy incluya la marca de detención para Pago contra entrega. El flowchart solo muestra `TransLegacy` sin anotar que este contrato sigue abierto — es el mismo bloqueante P1/PA-3 de RE-FU-010/011.

**Acción:** Marcarlo como `⚠️ Bloqueante compartido con RE-FU-010 PA-3` en el mapeo de CAs y en §1.3.

---

### 3. CA-C3 (Cancelación) sin prueba en §6

El Criterio C3 (modal de confirmación para cancelar) está listado en §5 como "Reusado sin cambio de RE-FU-010", pero §6 no incluye ningún caso de prueba para cancelación. En el diseño de RE-FU-010 sí existía.

**Acción:** Agregar caso de prueba para CA-C3 en §6.1.

---

### 4. CA-A4 (Panel Información de Facturación regionalizado) no mapeado

El Criterio A4 exige mostrar los campos SUNAT (Tipo de Operación, Condición de Pago) para clientes Perú. El diseño no lo menciona ni lo referencia como dependencia de RE-FU-005, que es donde se resuelve este dato (mismo patrón que en RE-FU-010 §3.3).

**Acción:** Agregar CA-A4 al mapeo de criterios con nota de dependencia RE-FU-005.

---

### 5. §3.1.1 Integración con Venta Digital — incompleta

El índice incluye esta subsección pero el contenido está truncado en el documento; solo aparece la última oración sobre el Windows Task Scheduler de Venta Digital. La integración VD es un punto técnico relevante que queda sin documentar.

**Acción:** Completar §3.1.1 con el flujo de integración VD.

---

### 6. Faltan las secciones de Manejo de Errores y Auto-chequeo (DoR)

Los diseños RE-FU-010 y RE-FU-011 tienen §6 Manejo de errores y §8 Auto-chequeo (Definition of Ready). El diseño de RE-FU-012 salta directo a pruebas. En particular, falta documentar:

- Qué ocurre si falla el INSERT en `SyncControl` post-commit (¿el job se pierde silenciosamente?).
- Comportamiento al agotar reintentos de Hangfire — PA-4 está abierto en §1.3 pero no tiene sección que lo recoja en errores.

**Acción:** Agregar §6 Manejo de errores y §8 Auto-chequeo siguiendo la estructura de RE-FU-010/011.

---

### 7. `SolicitudAutorizacionBO` bypass sin detalle de implementación

Aparece en la tabla de impacto técnico (#4) pero no hay código ni descripción de cómo se implementa el bypass. Siendo un cambio de comportamiento respecto al sistema actual (Regla 3), merece al menos una subsección equivalente a §4.1 / §4.2.

**Acción:** Agregar subsección §4.x con el detalle del bypass en `SolicitudAutorizacionBO.cs`.

---

### 8. `PedidoCreditoPayloadBuilder` sin detalle de mapping

Es un componente NUEVO que construye el DTO hacia Legacy, pero el diseño no describe las reglas de mapeo ni cómo maneja la variante "Pago contra entrega → Crédito" (Cambio 2 de RE-FU-010). Esto es especialmente relevante dado el PA-3 abierto sobre el contrato del payload Legacy.

**Acción:** Documentar las reglas de mapeo del payload en una subsección de §4.

---

## Resumen

| Área | Estado |
|---|---|
| Perú sin Legacy / sin FAA | ✅ Correcto |
| Atomicidad pendiente FAA | ✅ Correcto |
| Bloqueo datos fiscales | ✅ Correcto |
| Bypass código autorización | ⚠️ Sin detalle de implementación |
| DEP-015 (SyncControl) | ⚠️ No marcada como bloqueante |
| CA-C4 Pago contra entrega | ⚠️ No mapeado como bloqueante |
| CA-C3 Cancelación | ⚠️ Sin caso de prueba |
| CA-A4 Panel Facturación PE | ⚠️ No mapeado |
| Manejo de errores / DoR | ⚠️ Ausentes |
| §3.1.1 Integración VD | ⚠️ Truncada / incompleta |
| `PedidoCreditoPayloadBuilder` | ⚠️ Sin reglas de mapping |
