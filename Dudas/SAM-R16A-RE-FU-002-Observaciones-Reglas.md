# Observaciones a Reglas de Negocio — R16A-RE-FU-002

> Documento vivo para registrar observaciones, impedimentos y dudas que surjan sobre las **Reglas de Negocio** del requisito `R16A-RE-FU-002` (Asignación de Cobrador a Cliente) durante el análisis técnico o el desarrollo. Cada observación se numera de forma incremental y se mantiene con su estado hasta que se resuelve.
>
> No reemplaza a `R16A-RE-FU-002-Revision.md` (que documenta cambios ya aplicados tras observaciones de cliente cerradas). Este documento es para observaciones **nuevas, en curso**, que requieren definición antes de continuar.

---

## Índice de observaciones

| # | Regla afectada | Resumen | Estado |
|---|---|---|---|
| 1 | Regla 4 (Filtrado dinámico) / Criterios C2, C4 | Impedimento de secuencia: los "pendientes" que deben reasignarse pertenecen a estructuras de datos de otros requisitos aún no desarrolladas | 🔴 Abierta — pendiente de definición |

---

## Observación 1 — Dependencia de estructuras de datos de "pendientes" aún no desarrolladas

**Regla relacionada:** Regla 4 — Filtrado dinámico por cobrador asignado actualmente (y sus criterios derivados **C2** — Redistribución inmediata al reasignar Cobrador, y **C4** — Trazabilidad del trabajo realizado por el Cobrador anterior).

**Estado:** 🔴 Abierta — se pedirá definición.

### Planteamiento del impedimento

La Regla 4 establece que, al reasignar el Cobrador de un cliente, **los pendientes y pagos abiertos** del cliente deben redistribuirse a la bandeja del nuevo Cobrador, mientras que el trabajo **ya finalizado** por el Cobrador anterior debe preservarse en el histórico (Criterio C4).

Para poder ejecutar y/o probar esa reasignación a nivel de base de datos —incluyendo escribir el `Id` del nuevo cobrador donde corresponda y distinguir "abierto" de "trabajado"— se requiere que las **estructuras de datos de esos pendientes ya existan**, como mínimo, aunque su funcionalidad completa todavía no esté desarrollada. Actualmente varias de esas estructuras **no existen** o están **incompletas**, porque pertenecen a requisitos que siguen en estado "Propuesto" y cuyos scripts de `ALTER`/`CREATE TABLE` aparecen marcados como `❌ Pendiente` en sus propios documentos de impacto en BD.

Esto significa que el alcance de Regla 4 / Criterio C2 / Criterio C4 de R16A-RE-FU-002 **no se puede cerrar de forma independiente**: depende de que otros requisitos avancen primero (o al menos definan sus estructuras de datos).

### Pendientes (estructuras de datos) rastreados

#### Validar Cobro — R16A-RE-FU-023

| Tabla / objeto | Qué representa como "pendiente" | Estado en `_BD.md` |
|---|---|---|
| `fccFolioPagoCliente` | Cobro recibido pendiente de aplicar, generado al clasificar un correo del Buzón. Campo `Activo` = abierto(1)/cerrado(0) | Tabla existente, se usa para el conteo de pendientes |
| `fccPagoCliente` + columnas `Confirmado`, `FechaConfirmacion`, `IdUsuarioConfirmacion`, `Notas`, `IdCatMoneda` | Confirmación del cobro y **trazabilidad de quién lo trabajó** (relevante para Criterio C4) | ❌ Pendiente — `ALTER TABLE` no ejecutado |
| `tpPedido` + columnas `FechaCancelacionPorFaltaPago`, `IdUsuarioCancelacion`, `FechaSolicitudCancelacion`, `EstadoCancelacionCFDI` | Pedido pendiente de cobro gestionado desde "Gestionar Cobranza"; cancelación por falta de pago | ❌ Pendiente — `ALTER TABLE` no ejecutado |
| `fccFechaEstimadaPagoHistorial` (OBS-044) | Histórico de fecha estimada de pago por pedido pendiente | ❌ Pendiente — `CREATE TABLE` no ejecutado |
| `tpProformaPedido` (`MontoPendiente`, `Cancelada`, `Activo`) | Saldo pendiente de proformas/facturas por cobrar | Tabla existente, campos ya disponibles |

> Nota del propio documento `R16A-RE-FU-023_BD.md`: "Versión 3.0 (tablas ya existentes — CREATE TABLE removidos; todos los ALTER pendientes)" — confirma que ninguno de los `ALTER` requeridos se ha ejecutado todavía.

#### Buzón de Cobros — R16A-RE-FU-008-P1

| Tabla / objeto | Qué representa como "pendiente" | Estado en `_BD.md` |
|---|---|---|
| `MailbotEventoCorreo` | Evento de clasificación de correo entrante (origen de los pendientes de cobro) | ❌ Pendiente — `CREATE TABLE` no ejecutado, prioridad 🔴 Alta |
| `MailbotClasificacionLog` | Bitácora de clasificación del Agente IA / Mailbot | ❌ Pendiente — `CREATE TABLE` no ejecutado, prioridad 🔴 Alta |
| `GenerarPendienteUseCase.cs` → crea `fccFolioPagoCliente` | Lógica de negocio que origina el pendiente de cobro | No desarrollado (referenciado en el diseño, no implementado) |

#### Factura por Adelantado — R16A-RE-FU-018

| Tabla / objeto | Qué representa como "pendiente" | Estado en `_BD.md` |
|---|---|---|
| `tpProformaAdelanto` | "Pendiente FAA": Monto, `IdCFDIGenerada`, `FechaRegistro` — depende de que RE-012/RE-015 generen el pendiente | Tabla existente, pero el flujo que la alimenta depende de otros requisitos en curso |
| `AppSetting`, `TipoDocumentoFiscal`, `CFDI`, `TimbradoLog` (solución `ProquifaDotNet.Timbrado`) | Infraestructura nueva que sostiene el ciclo de vida del pendiente FAA hasta su timbrado | ❌ Pendiente — `CREATE TABLE`, prioridad Alta |

> El propio `_BD.md` de RE-018 ya documenta el mismo riesgo que RE-002: *"Cliente sin Cobrador asignado → Pendiente invisible"* — confirma el acoplamiento directo entre ambos requisitos.

#### Notas de Crédito México/Perú — R16A-RE-FU-032 / R16A-RE-FU-033

| Tabla / objeto | Qué representa como "pendiente" | Estado en `_BD.md` |
|---|---|---|
| `fccNotaCredito` + columnas R16 (empresa, cliente, modalidad, estado, fiscal) | Nota de Crédito en proceso, filtrable por cartera/Cobrador (OBS-004 de RE-002) | ❌ Pendiente — `ALTER TABLE` no ejecutado |
| `fccNotaCreditoPartida` + columnas R16 | Detalle de la Nota de Crédito | ❌ Pendiente — `ALTER TABLE` no ejecutado |
| `catMotivoCancelacionSAT` | Catálogo requerido para el flujo de cancelación | ❌ Pendiente — `CREATE TABLE` no ejecutado |
| RE-033 (Perú) | Comparte estructura con RE-032 | 🚫 Bloqueado además por definición de modalidad OSE/SUNAT |

### Conclusión del impedimento

El mecanismo de redistribución dinámica descrito en `R16A-RE-FU-002-Back.md` (JOIN sobre `ClienteCartera.IdUsuarioCobrador`, sin necesidad de migrar registros) **resuelve el Criterio C2 a nivel de Cliente/Cartera**, y en ese nivel no depende de las tablas anteriores. Sin embargo:

- El **Criterio C4** (trazabilidad de quién trabajó cada pendiente) sí depende de columnas que hoy no existen (`IdUsuarioConfirmacion` en `fccPagoCliente`, `IdUsuarioCancelacion` en `tpPedido`, equivalentes en NC), porque son las que registran "quién" ejecutó la acción sobre cada pendiente — sin ellas no hay forma de comprobar ni de auditar que el trabajo del Cobrador anterior se preservó.
- La prueba funcional end-to-end de Regla 4 (ver pendientes abiertos moverse de bandeja en cada módulo) **no puede ejecutarse hoy** en ningún módulo consumidor (Validar Cobro, Buzón de Cobros, Factura por Adelantado, Notas de Crédito) porque ninguno tiene aún sus estructuras de pendientes completas.

### Definición que se pedirá

1. ¿Se acepta cerrar el alcance de R16A-RE-FU-002 / Criterio C2 validando **únicamente** el JOIN dinámico a nivel `ClienteCartera`, dejando la verificación funcional completa (incluyendo Criterio C4) condicionada a que cada módulo consumidor entregue sus propias estructuras de "pendiente"?
2. ¿Cuál es el orden de ejecución esperado entre R16A-RE-FU-002 y los `ALTER`/`CREATE` pendientes de RE-008, RE-018, RE-023, RE-032 y RE-033 para poder dar trazabilidad real (Criterio C4) a las reasignaciones de Cobrador?
3. ¿Las columnas de trazabilidad de usuario (`IdUsuarioConfirmacion`, `IdUsuarioCancelacion`, etc.) que ya están definidas en los documentos `_BD.md` de cada módulo son las que se usarán para cumplir Criterio C4, o se requiere un campo adicional explícito tipo "trabajado por"?

---

## Plantilla para nuevas observaciones

```
## Observación N — <Título corto>

**Regla relacionada:** <Regla / Criterio>

**Estado:** 🔴 Abierta / 🟡 En definición / ✅ Resuelta

### Planteamiento del impedimento
<Descripción>

### Pendientes / estructuras rastreadas
<Tablas, requisitos relacionados, estado>

### Definición que se pedirá
<Preguntas concretas>
```
