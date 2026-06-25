# Revisión — [R16A-RE-FU-006][DIS-SOL] Diseño de la solución

| Campo | Valor |
|---|---|
| **Documento revisado** | `[R16A-RE-FU-006][DIS-SOL] Diseño de la solución.pdf` v1.0 (19 jun 2026) |
| **Autor del diseño** | Jose Armando Santiago Lorenzo |
| **Revisor** | Juan David García Cruz |
| **Fecha de revisión** | 24 jun 2026 |
| **Versión del requisito base** | R16A-RE-FU-006 v2 (post OBS-013/014/015) |

---

## Resumen

El diseño está bien estructurado (IEEE 1016, cubre flujos, BD, errores, pruebas) y la sección Impacto Técnico con el bug crítico de `cliente.Clave` es muy buena. **Hay 4 hallazgos críticos que bloquean el inicio de desarrollo** y 6 brechas menores. La causa raíz de los hallazgos críticos es que el diseño implementó solo el **nivel 2** de persistencia (snapshot en proforma) y se olvidó del **nivel 1** (referencia vigente del cliente), que es lo que Regla 4 ahora exige tras OBS-013.

Para alinear el diseño al requisito actualizado hay que actuar sobre estos puntos antes de pasarlo a construcción.

---

## Hallazgos críticos (bloqueantes)

### H-01 — Falta persistir `ReferenciaVigente` en `ClienteDatosBancarios`

- **Sección del PDF:** Diseño de Modelo de Datos · 1 (DDL de `ClienteDatosBancarios`) y Reglas técnicas · RT-07.
- **Lo que dice tu diseño:** *"La tabla ClienteDatosBancarios almacena los datos fuente (CodigoValidador, IdDatosBancarios), no la referencia resultante. GAP-A cerrado — no se requiere campo adicional en ClienteDatosBancarios."* (RT-07).
- **Lo que dice el requisito actualizado (Regla 4 nivel 1):** *"la combinación cliente-cuenta persiste en el modelo de datos el identificador de la cuenta bancaria, el identificador del cliente, el Código Validador capturado **y la referencia bancaria armada vigente**"*.
- **Acción:** GAP-A queda **reabierto**. Agregar a la tabla las columnas `ReferenciaVigente varchar(80) NULL` y `FechaReferenciaVigente datetime NULL`. Ver DDL actualizado en `R16A-RE-FU-006_BD.md` sección 1.

### H-02 — Flujo 1 no arma la referencia al CREATE/UPDATE

- **Sección del PDF:** Diseño funcional detallado · Flujo 1 (pasos 4-7).
- **Lo que dice tu diseño:** El Flujo 1 solo persiste `IdCliente`, `IdDatosBancarios`, `CodigoValidador`. La referencia no se arma aquí.
- **Lo que dice el requisito actualizado (Regla 5):** *"La referencia bancaria se arma al configurar/actualizar la cuenta del cliente, aplicando las reglas según el banco de la cuenta (ver Reglas 6 y 7), y se persiste como referencia vigente del cliente"*.
- **Acción:** Agregar paso en Flujo 1: tras validar duplicados y antes de persistir, invocar `ReferenciaBancariaBO.Construir(cliente, cuenta, codigoValidador)` y guardar el resultado en `ReferenciaVigente`. Mismo paso en Flujo de UPDATE de CódigoValidador.

### H-03 — Flujo 2 reconstruye la referencia en lugar de leer la vigente

- **Sección del PDF:** Diseño funcional detallado · Flujo 2 (pasos 3-7) y Criterio C1 mapeado en CA-6.
- **Lo que dice tu diseño:** El `tpProformaPedidoFactory` consulta `DatosBancarios`, `catBanco`, etc., y **reconstruye** la referencia desde fuentes.
- **Lo que dice el requisito actualizado (Criterio C1):** *"el sistema deberá **tomar la referencia vigente del cliente y casarla al documento**"*. El factory debe **leer**, no recalcular.
- **Acción:** Invertir el Flujo 2. Cambiar a: (1) consultar la asignación activa del cliente para la cuenta seleccionada del pedido; (2) copiar `ClienteDatosBancarios.ReferenciaVigente` a `tpProformaPedido.ReferenciaPago`; (3) si `ReferenciaVigente` es null (caso edge: cliente sin asignación o asignación pre-migración SSIS sin referencia armada), aplicar fallback documentado.

### H-04 — Falta lógica de regeneración por cambio en `Cliente.Nombre` o `Cliente.Clave`

- **Sección del PDF:** No cubierto.
- **Lo que dice el requisito actualizado (Regla 4 nivel 1):** *"solo se regenera si cambia un dato fuente (banco, cuenta, Código Validador o **datos del cliente que la componen**)"*.
- **Por qué importa:** Los segmentos S1-S3 dependen de `Cliente.Nombre` y S4 de `Cliente.Clave`. Si alguien actualiza esos campos en el catálogo de clientes y no se regenera la `ReferenciaVigente` de las asignaciones, la BD queda con referencia obsoleta y al generar la próxima proforma se casa un valor incorrecto.
- **Acción:** Documentar el mecanismo de regeneración en cascada. Opciones (recomendación: hook en `ClienteBO`):
  1. Hook en `ClienteBO._GuardarOActualizar`: si cambió `Nombre` o `Clave`, recorrer asignaciones activas del cliente y regenerar. **Recomendada**.
  2. Trigger BD `AFTER UPDATE` en `Cliente` (duplica lógica entre BD y .NET).
  3. Regeneración lazy en el factory (no escribe el campo en BD para clientes sin proforma).

---

## Brechas (no bloqueantes pero deben corregirse)

### H-05 — Selector de cuentas usa `DatosBancarios`, debe usar `EmpresaDatosBancarios`

- **Sección del PDF:** Criterios de aceptación · CA-2/CA-3.
- **Problema:** La Regla 2 dice *"catálogo de cuentas del grupo PROQUIFA"*. Tu diseño usa `GET /DatosBancarios?IdBanco={}` filtrado por `Activo = 1`, lo que devuelve **todas** las cuentas bancarias activas, no solo las del grupo. La fuente correcta es `EmpresaDatosBancarios` con `Activo = 1` cruzada con `DatosBancarios` por banco.

### H-06 — Constraint `UNIQUE` plano impide reasignar cuenta tras baja lógica

- **Sección del PDF:** Diseño de Modelo de Datos · 1 (DDL) y RT-02.
- **Problema:** El DDL aplica `UNIQUE (IdCliente, IdDatosBancarios)` plano. RT-02 dice *"La baja lógica (Activo = 0) no cuenta como duplicado — permite reasignar"*, pero el constraint UNIQUE plano **sí cuenta** los registros con `Activo = 0`. El comentario *"si se requiere... reemplazar por índice filtrado"* lo reconoce pero no aplica el fix. Debe ser índice único filtrado: `CREATE UNIQUE INDEX ... WHERE Activo = 1`. RT-02 quedaría coherente con el script.

### H-07 — Segmento S4 / `Carga_ClientesR1` como fuente productiva

- **Sección del PDF:** Diseño funcional detallado · Algoritmo Banamex — tabla S1-S7, y sección Impacto Técnico · GAP-E.
- **Punto bueno:** Detectaste correctamente que `dbo.Cliente` no tiene campo `Clave` y propusiste un fix. La sección está muy bien documentada.
- **Riesgo:** Apoyarse en `Carga_ClientesR1` (tabla ETL de migración) como fuente productiva de S4 es frágil — si esa tabla se trunca, se purga tras migración o no se pobla para clientes nuevos creados en R16, S4 = "0000" para esos clientes.
- **Acción:** Levantar como duda formal de arquitectura. Opciones:
  1. Agregar columna `ClaveLegacy` (o `Clave`) permanente a `dbo.Cliente`; el SSIS la pobla en migración y los clientes nuevos toman valor desde un secuencial.
  2. Mantener `Carga_ClientesR1` viva indefinidamente y documentarla como dependencia productiva.
  3. Recibir `IdClienteLegacy` como parámetro desde el factory (tu alternativa recomendada) — la mejor en cuanto a desacople del BO, pero deja igual la pregunta de dónde vive ese campo a largo plazo.

### H-08 — Diccionario de datos incompleto (directriz del proyecto)

- **Sección del PDF:** Diseño de Modelo de Datos · 1.
- **Falta:** índice `IX_ClienteDatosBancarios (IdCliente, IdDatosBancarios, Activo)` (sí está en `_BD.md` y debe replicarse en el diseño); índice único filtrado por `Activo = 1` (H-06); columnas `ReferenciaVigente` y `FechaReferenciaVigente` (H-01).

### H-09 — Numeración de Criterios de Aceptación no mapea con el requisito

- **Sección del PDF:** Criterios de aceptación del requisito · tabla.
- **Problema:** El requisito tiene 12 criterios numerados (A1–A4, B1–B5, C1–C3). El diseño usa CA-1…CA-8, CA-E1, CA-E2, CA-EC1, CA-EC2, CA-12 (nomenclatura mezclada y faltan CA-9/10/11).
- **Acción:** Renumerar 1:1 con los IDs del requisito para que el trazado sea directo.

### H-10 — Nomenclatura "PQF2" / "ProquifaDotNet-R14"

- **Sección del PDF:** Múltiples (mapeo PConnect → PQF2, ProquifaDotNet, etc.).
- **Acción:** Las directrices del proyecto piden referirse al sistema como **ProquifaDotNet** (no "PQF2" ni "ProquifaDotNet-R14"). Reemplazar globalmente.

---

## Puntos que están bien y deben conservarse

1. **Estructura IEEE 1016**: las 12 secciones cubren las 5 preguntas del checklist de la página 3.
2. **Tabla del algoritmo Banamex (S1-S7)**: muy clara, con fallback documentado por segmento y ejemplo concreto (`QUI2345002PABC`).
3. **Sección Impacto Técnico · GAP-E**: detección del bug `cliente.Clave` con código antes/después y alternativa recomendada. Esto está al nivel que se necesita para los hallazgos H-01 a H-04 también.
4. **Diagramas mermaid de Flujo 1 y Flujo 2**: muy útiles para el dev.
5. **Sección Manejo de Errores y Excepciones**: cubre los casos relevantes (cuenta inactiva, duplicado, CV vacío, banco no encontrado).
6. **Sección Pruebas**: buena cobertura unitaria/integración/críticos. Hay que extenderla para los nuevos casos de `ReferenciaVigente` y regeneración.
7. **Migración SSIS**: tablas, mapeo de campos y prueba post-migración bien definidos. Falta cerrar el mapeo `idCuenta → IdDatosBancarios` con tu apoyo.

---

## Acciones para José Armando (resumen accionable)

1. Agregar `ReferenciaVigente varchar(80) NULL` + `FechaReferenciaVigente datetime NULL` al DDL de `ClienteDatosBancarios` (H-01).
2. Modificar Flujo 1: tras validar duplicados, invocar `ReferenciaBancariaBO.Construir` y persistir resultado en `ReferenciaVigente` (H-02).
3. Modificar Flujo 2: cambiar de "reconstruye" a "lee `ReferenciaVigente` y copia a `ReferenciaPago`" (H-03).
4. Agregar Flujo 3 (o sub-flujo): regeneración de `ReferenciaVigente` ante cambio en `Cliente.Nombre`/`Clave` (H-04).
5. Cambiar selector de cuentas a `EmpresaDatosBancarios` (H-05).
6. Cambiar `UNIQUE` plano por índice único filtrado `WHERE Activo = 1` (H-06).
7. Levantar como duda formal: fuente productiva de la clave del cliente para S4 (H-07).
8. Completar diccionario de datos con índices nuevos (H-08).
9. Renumerar criterios de aceptación 1:1 con el requisito (H-09).
10. Reemplazar "PQF2" por "ProquifaDotNet" globalmente (H-10).

---

## Referencias

- Requisito actualizado: `R16A-RE-FU-006.md`
- Diccionario de datos: `R16A-RE-FU-006_BD.md`
- Impacto Backend (v2.0): `R16A-RE-FU-006-Back.md`
- Observaciones aplicadas: OBS-010 (migración Legacy), OBS-013 (persistencia en 2 niveles), OBS-014 (historial CV de 1 nivel), OBS-015 (PDF en firme = Legacy/Drobo).
