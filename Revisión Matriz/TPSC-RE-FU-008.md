# TPSC-RE-FU-008

**Estatus:** ✅ Atendido

---

## Observaciones

- **(Aplica a)** Para la clasificación automática no hay criterios configurables; el mailbot se entrena con una base de conocimiento y si se requiere ajustar se debe realizar un nuevo entrenamiento.
- Eliminar menciones de envío de correo cuando se detecta una inconsistencia — ese es criterio de otro requisito. Aquí solo debe hacerse mención de que se elimina el buzón cuando se marca como inconsistencia.
- **Regla 1:** No existe una clasificación "No cobro". La clasificación ocurre de la misma forma que para los demás pendientes (Cotización, Pedido); solo se agrega la categoría *Cobro*.
- ¿Qué sucede si llega un correo de un cliente no registrado y el mailbot lo clasifica erróneamente como Cobro? ¿Quién lo ve?
- **Criterio 9:** Actualmente los Buzones permiten eliminar pendientes. ¿Estamos seguros de que para Cobros no puede eliminarse? Si no puede eliminar, indicar que tendrá la salida existente para clasificarlo como *Otros* y sea el ESAC quien pueda eliminarlo.

## Notas adicionales

- Realizar estimación de un nuevo mailbot integrando IA para la clasificación.

## Resumen de cambios aplicados

- La clasificación automática se documentó como ejecutada por el **Mailbot**, que agrega la categoría *Cobro* al modelo existente (Cotización, Pedido, Otros). Se eliminó la noción de clasificación "cobro/no-cobro" y la de "criterios configurables".
- Se retiró el detalle del envío de correo de notificación ante inconsistencia. Ese comportamiento pertenece al módulo Validar Cobro; en este requisito solo se contempla que el pendiente del Buzón se elimina automáticamente al marcar como inconsistencia.
- **Manejo de eliminación:** el Buzón de Cobros no ofrece eliminación directa al Gestor de Cobranza. La salida para correos mal clasificados es reclasificarlos al buzón de *Otros*; desde ahí, el rol ESAC puede eliminarlos. Documentado en Regla 9 y Criterio D2.
- Se reforzó el pendiente sobre correos de cobro cuyo cliente no es identificable o no está registrado. Queda pendiente de confirmar con el cliente.
- Se precisó el criterio de aplicabilidad regional (**Criterio E1**): el Buzón opera con la misma mecánica en ambas regiones pero segregando siempre por la región del cliente.
- Reglas reescritas como enunciados declarativos: de 8 a 10.
- Criterios organizados en 5 secciones: **A** (Clasificación y recepción), **B** (Reflejo y pendiente), **C** (Ciclo de vida), **D** (Acciones del Gestor y UX), **E** (Aplicabilidad regional).
- Riesgos renumerados consecutivamente (antes 3 y 4; ahora 1 y 2).
