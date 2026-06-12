# R16A-RE-FU-002

**Estatus:** ✅ Atendido

---

## Observaciones

- **(Criterios de aceptación - Regla 4 / Criterio 7)** ¿Qué sucede si el pendiente ya existe, está asignado a un Cobrador y el Coordinador de Tesorería cambia el cobrador asignado? ¿El pendiente cambia de cobrador o se mantiene con el cobrador que lo nació?
- Especificar que queda fuera del alcance colocar el campo *Cobrador* en el alta de cliente en *Cotizar lo Cotizable*, ya que esa alta está pensada únicamente para poder cotizar, no para gestión del cliente.

## Notas adicionales

- Cartera de cobradores.
- Estimar migrar Mailbot a un agente inteligente.
- Considerar que cuando se asigne un cobrador a un cliente que no tenía cobrador, se deben visualizar sus pendientes (manejo correcto de carteras) — *Criterio 5 → ahora Criterios C2 + C3*.

## Resumen de cambios aplicados

- HU simplificada.
- Reglas reescritas como enunciados declarativos: de 5 reglas *Cómo* pasaron a 5 reglas declarativas del *Qué*.
- **Decisión sobre cambio de Cobrador:** El filtrado de bandeja es dinámico — al reasignar el Cobrador de un cliente, todos sus pendientes y pagos vigentes se mueven inmediatamente a la bandeja del nuevo Cobrador. Documentado en Regla 4 (Filtrado dinámico) y Criterio C2 (Redistribución inmediata al reasignar).
- **Exclusión del campo Cobrador en Cotizar lo Cotizable:** agregada en Alcance "No aplica a".
- Criterios organizados en 3 secciones: **A** (Visibilidad y edición), **B** (Selector), **C** (Filtrado de bandeja).
