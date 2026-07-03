# Revisión — [R16A-RE-FU-003][DIS-SOL] Diseño de la solución

| Campo | Valor |
|---|---|
| **Documento revisado** | `[R16A-RE-FU-003][DIS-SOL] Diseño de la solución.pdf` v1.3 (30 jun 2026) |
| **Autor del diseño** | Isai Amaury Garcia Flores |
| **Revisor del diseño (campo del documento)** | Valdemar Farina Sanchez |
| **Revisor de esta ronda** | Juan David García Cruz |
| **Fecha de revisión** | 2 jul 2026 |
| **Documento base de comparación** | `R16A-RE-FU-003-Propuesta1.md` (modelo genérico ArchivoDominio, aprobado 2026-06-26) |

---

## Resumen

El DIS-SOL v1.3 adoptó correctamente, a nivel narrativo y de diagramas, el modelo genérico de la Propuesta 1 (`ArchivoDominio` / `catDominioEntidad` / `CatalogoTipoDocumento`) mediante una capa de adaptación específica de Cliente que mantiene estable el contrato con Frontend — una buena decisión de diseño, bien justificada. Sin embargo, **la sección "Diseño de Modelo de Datos" no se actualizó cuando se aplicó la Propuesta 1**: el único script SQL del documento siembra el modelo viejo (`catUsoArchivoSistema`, `ArchivoCliente`) y contradice, en la misma página, la tabla de "Decisiones de diseño" que está justo debajo. Esto, sumado a la ausencia total del DDL de las 3 tablas nuevas y a que `R16A-RE-FU-003_BD.md` no se actualizó, deja al documento **sin script ejecutable para construir el modelo de datos real** — un desarrollador que solo lea el DIS-SOL no puede crear las tablas.

No recomiendo pasar a construcción hasta resolver los hallazgos H-01 a H-03.

---

## Hallazgos críticos (bloqueantes)

### H-01 — El script SQL de la sección "Modelo de datos" siembra el modelo viejo, no el nuevo

- **Sección del PDF:** Diseño funcional detallado · 1. Modelo de datos (página 8).
- **Lo que dice el script:** `INSERT INTO dbo.catUsoArchivoSistema (...)` con los valores `LicenciaSanitaria` / `AvisoResponsableSanitario`, y `CREATE UNIQUE INDEX UX_ArchivoCliente_ClienteUso ON ArchivoCliente (IdCliente, IdCatUsoArchivoSistema) WHERE Activo = 1`. Ambos objetos (`catUsoArchivoSistema`, `ArchivoCliente`) son el **modelo deprecado**. El comentario del script incluso conserva la etiqueta `GAP-05`, que en `R16A-RE-FU-003-Back.md` corresponde al diseño original (pre-Propuesta 1).
- **Por qué es una contradicción interna:** inmediatamente debajo, en la misma página, la tabla "Decisiones de diseño" dice textualmente *"Adoptar el modelo genérico ArchivoDominio/catDominioEntidad/CatalogoTipoDocumento del líder backend"*. Y en la página 18 ("Impacto en código existente"), el propio documento dice: *"índice único filtrado, ahora en ArchivoDominio (ya no en ArchivoCliente)"*.
- **Causa probable:** el script no se actualizó al pasar de v1.2 ("Se documenta diseño de la solución", 19-jun, previo a la Propuesta 1) a v1.3 ("Se actualiza documento tras ajustes en propuesta de diseño", 30-jun).
- **Acción:** Reemplazar el script por el seed real sobre el modelo nuevo: `INSERT` en `catDominioEntidad` (Cliente/Proveedor/Producto) y en `CatalogoTipoDocumento` (LicenciaSanitaria/AvisoResponsableSanitario, dominio Cliente), y el índice único filtrado sobre `ArchivoDominio` — ver Propuesta1 §2.2.1, §2.2.2 y §2.2.3.

### H-02 — Falta el DDL (`CREATE TABLE`) de las 3 tablas nuevas

- **Sección del PDF:** Diseño de Modelo de Datos / Diagramas.
- **Lo que sí está:** el diagrama ER (página 13) es correcto y muestra bien `catDominioEntidad` → `CatalogoTipoDocumento` → `ArchivoDominio` → `Archivo`, incluyendo el modelo deprecado para contexto de transición.
- **Lo que falta:** no hay ningún `CREATE TABLE` con tipos de dato, longitudes, PKs, FKs ni definición de índices (`UQ_ArchivoDominio_Vigente`, `IX_ArchivoDominio_Entidad`, `IX_ArchivoDominio_Tipo`, `UQ_CatalogoTipoDocumento_DominioClave`, `IX_CatalogoTipoDocumento_Dominio`, `UQ_catDominioEntidad_Clave`). Esa información completa solo existe en `R16A-RE-FU-003-Propuesta1.md` §2.2.
- **Acción:** Incorporar al DIS-SOL el DDL completo de las 3 tablas (copiado/adaptado de Propuesta1 §2.2.1-2.2.3), consistente con el estándar de diccionario de datos del proyecto (Nombre de tabla, Columnas, Relaciones, Índices, Consideraciones especiales).

### H-03 — `R16A-RE-FU-003_BD.md` no se actualizó tras la Propuesta 1

- **Documento:** `R16A-RE-FU-003_BD.md` (diccionario de datos de referencia).
- **Problema:** documenta exclusivamente el modelo viejo (`ArchivoCliente`, `Archivo`, `catUsoArchivoSistema`, `Usuario`) — no menciona `ArchivoDominio`, `catDominioEntidad` ni `CatalogoTipoDocumento` en ningún punto. Todas sus consultas de referencia (documentos vigentes, verificación de completitud, reemplazo transaccional, eliminación lógica) están escritas sobre `ArchivoCliente`/`catUsoArchivoSistema`.
- **Impacto:** es la causa raíz más probable de H-01 — si quien escribió el DIS-SOL consultó `_BD.md` como fuente para la sección de modelo de datos, encontró el modelo viejo documentado como si fuera el vigente.
- **Acción:** Reescribir `_BD.md` con el modelo `ArchivoDominio` como fuente de verdad (puede tomarse como base la Sección 2.2 de Propuesta1.md), y marcar `ArchivoCliente`/`catUsoArchivoSistema` como deprecados con plan de transición, igual que ya hace el propio DIS-SOL.

---

## Brechas — no bloqueantes pero deben corregirse

### H-04 — Falta el script de migración `ArchivoCliente` → `ArchivoDominio` en el DIS-SOL

- **Sección del PDF:** Impacto Técnico (página 18) menciona "+ migración" en la fila del script de BD, pero no incluye el script ni referencia explícita a dónde encontrarlo.
- **Acción:** Incluir el script de Propuesta1 §2.3 (o referenciarlo explícitamente) dentro del DIS-SOL, ya que es parte del entregable de Tarea 1.

### H-05 — Nombre del controlador inconsistente entre secciones

- **Sección del PDF:** "Componentes involucrados" (página 5) llama al componente **`DocumentosRegulatoriosClienteController`** (como si fuera una clase nueva).
- **Problema:** el resto del documento — Contratos de API (pág. 14), Impacto en código existente (pág. 18) y el diagrama de funcionalidad (pág. 14) — deja claro que el controlador sigue siendo **`ArchivoClienteController`** (existente, extendido; prefijo de ruta `/ArchivoCliente`) y que lo nuevo es el BO `DocumentosRegulatoriosClienteBO`.
- **Acción:** Corregir el nombre en la tabla de "Componentes involucrados" para que diga `ArchivoClienteController (existente, extendido)`, consistente con el resto del documento.

### H-06 — Datos iniciales de `CatalogoTipoDocumento` no replicados en el DIS-SOL

- Propuesta1 §2.2.2 especifica los valores iniciales (`FormatoAceptado`, `Orden`, `TamanioMaximoKB`) para `LicenciaSanitaria` y `AvisoResponsableSanitario`. El DIS-SOL no los incluye — deben agregarse junto con el DDL corregido (H-02).

### H-07 — Encabezado interior del documento con placeholder sin reemplazar

- El encabezado repetido en cada página interior (2 a 25) dice **"Diseño de la solución - Requisito R16A-RE-FU-XXX."** — el placeholder de plantilla no fue reemplazado por "R16A-RE-FU-003"; solo la portada (página 1) tiene el número correcto.
- Adicionalmente usa guión normal "-" en vez de "—", igual que se observó en la revisión del 006.
- **Acción:** Corregir el encabezado en todas las páginas interiores.

---

## Puntos que están bien y deben conservarse

1. **Capa de adaptación de Cliente** sobre el modelo genérico — decisión bien justificada (estabilidad de contrato para Frontend sin forzar rework) y consistente en todo el documento excepto por H-01/H-05.
2. **Diagrama ER (página 13)**: correcto, refleja el modelo nuevo y el deprecado con la relación de transición.
3. **Diagrama de funcionalidad por swimlanes (página 14)**: buen detalle de qué capa toca qué tabla, numerado alineado a los 5 endpoints.
4. **Criterios de aceptación mapeados 1:1 con el requisito** (A1, A2, B1, B2, C1, D1, D2, E1) — a diferencia del 006, aquí la numeración sí coincide exactamente con `R16A-RE-FU-003.md`.
5. **Manejo de Errores y Excepciones**: cubre el caso de archivo huérfano en MinIO si falla la transacción de carga — riesgo documentado explícitamente en vez de ignorado.
6. **Endpoint `Validar` extensible (Chain of Validators)**: bien alineado con Propuesta1 §3.4, mismo contrato de respuesta.

---

## Acciones para Isai Amaury (resumen accionable)

1. **Bloqueante:** Reemplazar el script SQL de la página 8 por el seed real sobre `catDominioEntidad` + `CatalogoTipoDocumento` + índice único filtrado en `ArchivoDominio` (H-01).
2. **Bloqueante:** Agregar el DDL completo (`CREATE TABLE`, índices, FKs) de `catDominioEntidad`, `CatalogoTipoDocumento` y `ArchivoDominio` al DIS-SOL (H-02).
3. **Bloqueante:** Coordinar la reescritura de `R16A-RE-FU-003_BD.md` sobre el modelo `ArchivoDominio` (H-03).
4. Incluir el script de migración `ArchivoCliente` → `ArchivoDominio` en el DIS-SOL (H-04).
5. Corregir el nombre del controlador en "Componentes involucrados" a `ArchivoClienteController` (H-05).
6. Agregar los datos iniciales de `CatalogoTipoDocumento` al DDL (H-06).
7. Corregir el encabezado de página con el número de requisito correcto y "—" en vez de "-" (H-07).

---

## Referencias

- Requisito: `R16A-RE-FU-003.md`
- Propuesta de rediseño (fuente del modelo adoptado): `R16A-RE-FU-003-Propuesta1.md`
- Diccionario de datos (desactualizado — ver H-03): `R16A-RE-FU-003_BD.md`
- Impacto Backend original (pre-Propuesta 1): `R16A-RE-FU-003-Back.md`
- Revisión del requisito (ya atendida): `R16A-RE-FU-003_Revision.md`
- Notas de la sesión de propuesta: `Cambios en revisión con valde.md`
