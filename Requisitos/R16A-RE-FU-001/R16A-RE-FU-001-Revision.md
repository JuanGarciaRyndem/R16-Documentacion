# R16A-RE-FU-001

**Estatus:** ✅ Atendido

---

## Observaciones

- **(Historia de Usuario)** No es necesario especificar los módulos en los que se utilizará el catálogo de cuentas bancarias de Proquifa. Con mencionar que se necesita el catálogo para el sistema es suficiente.
- **(Requisito)** A nivel requisito no debería definirse la estructura técnica de la solución (se define el *qué*, no el *cómo*). Sugerencia: quitar la redacción sobre tablas, quitarlo también del alcance y de Reglas y Criterios. Así mismo, no debe tomarse como base sistemas legados para definir la estructura del catálogo.
- **(Requisito)** La redacción sobre que no tiene interfaz gráfica y que se gestiona desde BD son criterios de aceptación, no parte del requisito.
- **(Alcance)** ¿Por qué se está considerando en buzón de pagos? ¿Es por la propuesta de IA? — *Buzón de Pagos (identificación de pagos entrantes contra las cuentas del grupo).*
- **(Criterios de aceptación)** ¿Por qué en la Regla 1 se decide que debe ser modelado con la estructura de Legacy?
- **(Criterios de aceptación)** La Regla 2 y 3 deberían redactarse como un *qué*, no como un *cómo*, ya que la solución técnica puede ser diferente por diseño pero cumplir con la regla.
- **(Criterios de aceptación)** La Regla 5 y Criterio 5 no deberían especificarse. Se está entrando en temas de estrategia de borrado del sistema (borrado lógico). No es que se "filtre por cuentas activas" sino que son las cuentas que existen en sistema. Si técnicamente está inactiva, para negocio no existe. Puede confundir los términos *Existir* vs *Activo*. Sugerencia: cambiar a "Existe o no existe en sistema".
- Agregar requisito de mantenimiento post-go-live del catálogo (¿quién actualiza las cuentas y cómo si no hay UI?).
- 

## Notas adicionales

- Guía de resolución para dar de alta, baja o actualizar cuentas bancarias.

## Resumen de cambios aplicados

- **(Alcance)** El Buzón de Pagos se marcará como pendiente con `**` (depende de propuesta IA).
- **(Criterios - Regla 1)** Eliminada la referencia al modelado con estructura de Legacy. La referencia técnica a tabla `CuentaBanco` BD PConnect se conservó solo en Observaciones para el equipo interno.
- **(Mantenimiento post-go-live)** Actualizado y descrito de mejor manera en Criterios C1 y C2.
- HU simplificada (sin enumerar módulos consumidores).
- Requisito reescrito sin estructura técnica, sin referencia a Legacy, sin "no UI/gestión BD" (movido a Criterios).
- Reglas reescritas como enunciados declarativos: de 5 reglas *Cómo* pasaron a 4 reglas declarativas del *Qué*.
- Cambio "filtrar por cuentas activas" → "Existe vs No existe" aplicado en Regla 2 y Criterios B1, B2.
- Criterios organizados en 3 secciones: **A** (Disponibilidad), **B** (Estado de existencia), **C** (Gestión).
- "No tiene UI, se gestiona en BD" movido del Requisito a Criterios C1, C2, C3.
- Riesgos renumerados consecutivamente desde 1.
