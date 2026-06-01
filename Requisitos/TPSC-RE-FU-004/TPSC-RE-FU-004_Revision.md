# TPSC-RE-FU-004

**Estatus:** ✅ Atendido

---

## Observaciones

- Validar los catálogos con el cliente, ya que actualmente en ProquifaNet 2 son diferentes.

## Notas adicionales

- ¿Hay validación de formato correcto de SAT actualmente?
- Considerar validación con expresión regular u otro mecanismo donde el formato esté en BD asociado a cada región para no hardcodearlo (e.g., `SELECT formatosValidacion WHERE id_region = cliente.idRegion`).
- Hacer validación de RFC con un *debounce* después de un tiempo sin escribir en el campo, similar a la búsqueda, para no saturar las consultas.
- Agregar catálogo de números de contribuyentes válidos para Perú (RUC, 11 dígitos).

## Resumen de cambios aplicados

- Reglas reescritas como enunciados declarativos: de 7 pasaron a 6. Reglas 5 y 6 originales (catálogos de Tipo de Sociedad y Régimen Fiscal) consolidadas en una sola **Regla 5** con redacción agnóstica al catálogo específico, hasta resolver el pendiente con el cliente.
- Criterios organizados en 5 secciones: **A** (Visualización y acceso), **B** (Validación RFC México), **C** (Validación RUC Perú), **D** (Selectores), **E** (Persistencia y consumo posterior).
- Criterios D1 y D2 reformulados sin listar opciones literales: el catálogo se consulta al sistema según Región. La lista de opciones se gestiona como dato paramétrico.
- Observación sobre catálogos PQF2 vs catálogos del cliente formalizada como pendiente en Observaciones. Se elaboró archivo adjunto `TPSC-RE-FU-004_Equivalencias_MX_PE.xlsx` con cruce de PQF2, archivo del cliente y catálogo SAT vigente.
- Riesgos renumerados consecutivamente (antes 1 y 3; ahora 1 y 2).
- **Pendientes preservados:** confirmación del régimen Perú "Régimen para Personas Naturales" y homoclave del RFC.
- **Nuevos pendientes formalizados:** validación local del RUC en PQF2, denominación oficial S.A.C.S. para Perú, consolidación de catálogos definitivos para ambas regiones.
