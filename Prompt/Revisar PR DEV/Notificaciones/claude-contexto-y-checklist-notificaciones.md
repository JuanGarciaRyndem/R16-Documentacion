# Contexto + Checklist + Workflow para revisión de PRs — Notificaciones (R16)

---

## Bloque 1 — Contexto del proyecto

### Repositorio de trabajo

Repo local a revisar para el proyecto de Notificaciones (R16 - Envío de Correos):

```
C:\Users\juan.garcia\Documents\proquifa-pqf2-notificaciones-api
```

Este repositorio corresponde a la API de notificaciones
(proquifa-pqf2-notificaciones-api) y será la referencia base para el
análisis/revisión del proyecto de Notificaciones.

- Organización/owner en GitHub: `ryndem`
- Nombre exacto del repo en GitHub: `ryndem/proquifa-pqf2-notificaciones-api`
- Ramas base observadas: `main`, `project/r16-pedidos-sin-credito`,
  `qa/r16-pedidos-sin-credito`, `uat`. **No asumir que la base de un PR es
  `main`**: el PR #2, por ejemplo, tenía como base real
  `project/r16-pedidos-sin-credito`, no `main`. Verificar siempre con:

```bash
gh api repos/ryndem/proquifa-pqf2-notificaciones-api/pulls/{N} --jq "{base: .base.ref}"
```

_Nota: esta ruta está en la máquina local del usuario (equipo "rynl010"). Para que
una sesión en la nube pueda leer estos archivos directamente, es necesario conectar
la carpeta correspondiente desde la app de escritorio de Claude (botón "Add folder")
y solicitar acceso a la carpeta que contiene este repo._

### Herramientas disponibles en esta máquina

- `git` instalado y con acceso al remoto (fetch/pull habilitados).
- GitHub CLI (`gh`) instalado en `C:\Program Files\GitHub CLI\gh.exe`, con sesión
  iniciada como `JuanGarciaRyndem` (verificar con `gh auth status`) para poder
  leer/comentar PRs sin exponer tokens en texto plano.

---

## Bloque 2 — Checklist de revisión de PR (.NET Core 10, Clean Architecture)

Plantilla estándar a usar para revisar Pull Requests del repositorio
`ryndem/proquifa-pqf2-notificaciones-api` (proyecto R16 - Envío de Correos /
Notificaciones).

### Rol

Actuar como revisor de código senior (Staff Engineer / Tech Lead) especializado en
.NET Core y Clean Architecture, revisando un PR de GitHub para una aplicación .NET
Core 10 estructurada bajo Clean Architecture (capas: Domain, Application,
Infrastructure, API/Presentation).

### Contexto del proyecto

- Framework: .NET Core 10
- Arquitectura: Clean Architecture (Domain / Application / Infrastructure / API)
- Completar si aplica: patrón CQRS con MediatR, EF Core, FluentValidation,
  AutoMapper/Mapster, xUnit/NUnit/MSTest, convenciones internas del equipo, etc.

### Checklist de análisis

**1. Cumplimiento de Clean Architecture**
- ¿Se respeta la regla de dependencia (las capas internas —Domain, Application— no
  dependen de capas externas —Infrastructure, API—)?
- ¿La lógica de negocio está en Domain/Application y no filtrada hacia
  Infrastructure o Controllers?
- ¿Las interfaces están definidas en la capa correcta (ej. puertos en Application,
  implementaciones en Infrastructure)?
- ¿Hay fugas de detalles de infraestructura (EF Core, HttpClient, etc.) hacia
  Domain o Application?

**2. Diseño y buenas prácticas de .NET**
- Uso correcto de async/await (sin bloqueos con `.Result` o `.Wait()`,
  `ConfigureAwait` donde aplique).
- Manejo de nulabilidad (nullable reference types habilitados y respetados).
- Inyección de dependencias correcta (lifetimes: Scoped/Transient/Singleton
  apropiados).
- Uso adecuado de records, pattern matching, y features nuevas de C#/.NET 10
  cuando aportan claridad.
- Nombres de clases, métodos y variables claros y consistentes con el resto del
  código.

**3. Manejo de errores y validaciones**
- ¿Las excepciones se manejan de forma consistente (excepciones de dominio vs.
  errores técnicos)?
- ¿Existe validación de entrada (FluentValidation, Data Annotations) en la capa
  correcta?
- ¿Los errores se traducen correctamente a respuestas HTTP (ProblemDetails,
  códigos de estado apropiados)?

**4. Persistencia y acceso a datos**
- Revisión de queries EF Core: N+1, uso de `AsNoTracking` donde corresponda,
  proyecciones innecesarias.
- Transacciones y consistencia de datos.
- Migraciones incluidas y coherentes con los cambios de modelo.

**5. Seguridad**
- Validación de inputs contra inyección, exposición de datos sensibles en logs o
  respuestas.
- Autenticación/autorización aplicada correctamente en endpoints nuevos o
  modificados.
- Manejo seguro de secretos/configuración.

**6. Testing**
- ¿Hay pruebas unitarias para la lógica de Domain/Application?
- ¿Se cubren casos límite y de error, no solo el camino feliz?
- ¿Los tests son independientes, legibles y no dependen de estado compartido?

**7. Rendimiento y mantenibilidad**
- Complejidad innecesaria, código duplicado, violaciones de SOLID.
- Impacto en rendimiento de los cambios (loops, allocations innecesarias,
  serialización).

**8. Consistencia con el estilo del proyecto**
- Convenciones de nombres, organización de carpetas/namespaces.
- Uso de patrones ya establecidos en el repo (Result pattern, Specification
  pattern, etc.) en lugar de introducir nuevos sin justificación.
- El proyecto migró sus comentarios/identificadores de tests a inglés (ver commits
  `refactor: use English in comments` y `refactor(tests): use English test
  identifiers`) — todo código nuevo debe seguir esa convención; señalar cualquier
  comentario que quede en español por descuido.

**9. Análisis estático tipo SonarQube**

- Code Smells:
  - Nombres de propiedades/tipos duplicados o ambiguos (ej. propiedad y tipo con
    el mismo nombre, como pasó con `BounceType` en `EmailRecipientDomain`).
  - Código muerto: clases, métodos, campos o imports sin ningún call site (ej.
    `Errors.Validation.Failed` quedó huérfano tras migrar a
    `Errors.Notifications.ValidationFailed`).
  - Complejidad cognitiva/ciclomática alta en un solo método.
  - Métodos con demasiados parámetros (umbral típico mayor a 7).
  - Duplicación de código entre archivos o dentro del mismo archivo (incluye
    constantes/comentarios idénticos copiados en más de un lugar, ej. la
    constante `NoJitter` repetida en dos archivos de test).

- Bugs potenciales:
  - Tipos que pueden desbordar (`int` donde el dominio permite valores grandes,
    ej. `SizeBytes` de un adjunto).
  - Nulabilidad mal manejada (referencias nulas no verificadas, forzado sin
    justificación).
  - Parámetros de fecha/hora sin validar su "kind"/zona horaria cuando el nombre o
    la documentación asumen un valor específico (ej. `utcNow` sin verificar
    `DateTimeKind.Utc`).
  - Comparaciones de igualdad exacta sobre valores de punto flotante o derivados
    (riesgo de redondeo, regla S1244).
  - Comparaciones o conversiones que puedan lanzar excepciones no controladas.

- Vulnerabilidades / Security Hotspots:
  - Secretos o referencias a credenciales expuestas en código o logs.
  - Validación de entrada insuficiente antes de persistir o exponer datos.
  - Deserialización insegura, inyección (SQL, log, comandos, etc.).

- Mantenibilidad:
  - Colecciones mutables expuestas sin control de acceso (getter público que
    devuelve una lista mutable en vez de exponer una interfaz de solo lectura
    más un método de mutación, ej. `Recipients`/`Attempts`/`Attachments` en
    `EmailRequestDomain`).
  - Encapsulación inconsistente entre entidades del mismo agregado.

- Cobertura de tests:
  - ¿El nuevo código tiene tests correspondientes, o quedó sin cubrir?
  - ¿Los tests cubren tanto el camino feliz como los casos límite/erróneos?

**10. Convenciones de rutas de EndPoints (API)**

Regla de la ruta:

```
api/v1/{resource}/{id}/{subresource}
```

Componentes:
- `api/v1/` → versión base de la API.
- `{resource}` → entidad principal, **en singular y en inglés** (ej. `invoice`,
  `payment`, `creditNote`).
- `{id}` → identificador del recurso.
- `{subresource}` → opcional; solo cuando la acción **no** es CRUD estándar (ej.
  `cancel`, `stamp`, `report`, `export`).

Reglas a verificar en el PR:
- El recurso va en **singular** (`invoice`, no `invoices`).
- El nombre del recurso está en **inglés**, consistente con el resto de la API.
- Las operaciones CRUD estándar se expresan con el **verbo HTTP** (GET, POST, PUT,
  DELETE), no con palabras en la ruta (ej. no `api/v1/invoice/create`).
- Las acciones especiales que no encajan en CRUD puro se modelan como
  subrecurso/cambio de estado (ej. `api/v1/invoice/{id}/cancel`), y solo se
  agregan cuando el caso de uso realmente lo requiere — no crear subrecursos para
  todo.
- Prefijo `api/v1/` presente y consistente para versionado en todos los endpoints
  nuevos.

Ejemplos de referencia:

| Caso de uso                      | Endpoint                     | Método |
| --------------------------------- | ----------------------------- | ------ |
| Crear factura                     | `api/v1/invoice`               | POST   |
| Obtener factura                   | `api/v1/invoice/{id}`          | GET    |
| Actualizar factura                | `api/v1/invoice/{id}`          | PUT    |
| Eliminar factura                  | `api/v1/invoice/{id}`          | DELETE |
| Cancelar factura (caso especial)  | `api/v1/invoice/{id}/cancel`   | POST   |
| Registrar pago                    | `api/v1/payment`               | POST   |
| Obtener pago                      | `api/v1/payment/{id}`          | GET    |
| Generar nota de crédito           | `api/v1/creditNote`            | POST   |
| Obtener nota de crédito           | `api/v1/creditNote/{id}`       | GET    |

Si un PR introduce o modifica endpoints, señalar en "Hallazgos menores" cualquier
ruta que use plural, español, verbos en la URL para CRUD estándar, o subrecursos
innecesarios.

### Formato de salida esperado

1. **Resumen general** (2-4 líneas): qué hace el PR y una valoración global.
2. **Hallazgos críticos** (bloquean el merge): lista con archivo/línea,
   descripción del problema y sugerencia de corrección.
3. **Hallazgos menores / sugerencias**: mejoras opcionales de estilo,
   legibilidad, rendimiento o hallazgos tipo SonarQube (sección 9).
4. **Preguntas para el autor**: dudas que requieren contexto adicional antes de
   aprobar.
5. **Veredicto**: Aprobar / Aprobar con comentarios / Solicitar cambios.

### Regla importante

No asumir contexto que no esté en el diff; si falta información para evaluar algo
(por ejemplo, si una regla de negocio es correcta), indicarlo explícitamente en
"Preguntas para el autor" en lugar de suponer.

### Notas de uso

- Al invocar esta revisión, se debe proporcionar: el diff del PR, el link del PR
  de GitHub, o los archivos relevantes modificados.
- Repo local de referencia: ver Bloque 1 de este documento.

---

## Bloque 3 — Workflow técnico para obtener el diff real de un PR

**1. Traer las referencias del PR directamente desde GitHub** (funciona aunque el
repo sea privado, siempre que `gh`/git tengan credenciales):

```bash
git fetch origin +refs/pull/{N}/head:refs/pr{N}/head
```

**2. No asumir que la base es `main`.** Confirmar la base real configurada en el
PR (recordar: en este repo ya hubo un caso donde la base real era
`project/r16-pedidos-sin-credito`, no `main`):

```bash
gh api repos/ryndem/proquifa-pqf2-notificaciones-api/pulls/{N} --jq "{base: .base.ref, base_sha: .base.sha, head_sha: .head.sha}"
```

**3. Calcular el punto de divergencia real y diffear contra ese punto** (no contra
el tip actual de la base, que puede haber avanzado):

```bash
git merge-base origin/{BASE_REAL} refs/pr{N}/head
git diff --stat {MERGE_BASE_SHA} refs/pr{N}/head
git diff {MERGE_BASE_SHA} refs/pr{N}/head > pr{N}_diff.txt
```

**4. Señales de que la base asumida está mal:** el diff aparece gigantesco (miles
de líneas) o muestra archivos/entidades **eliminándose** que en realidad ya
existen en el código — eso indica que se está comparando contra un ancestro
demasiado lejano.

---

## Bloque 4 — Publicar comentarios de revisión con `gh` (sin exponer tokens)

Requisito previo: `gh auth status` debe mostrar sesión iniciada (usuario
`JuanGarciaRyndem`) con permisos `repo`. Nunca pegar el token en la conversación
ni en archivos versionados.

**1. Armar un JSON con los comentarios** (usar el SHA del head real del PR como
`commit_id`, obtenido de `gh api .../pulls/{N} --jq ".head.sha"`):

```json
{
  "commit_id": "{HEAD_SHA}",
  "event": "COMMENT",
  "body": "",
  "comments": [
    {
      "path": "ruta/relativa/al/archivo.ext",
      "line": 10,
      "side": "RIGHT",
      "body": "comentario en markdown"
    }
  ]
}
```

**2. Publicar:**

```bash
"C:\Program Files\GitHub CLI\gh.exe" api repos/ryndem/proquifa-pqf2-notificaciones-api/pulls/{N}/reviews -X POST --input review.json
```

**3. Limitación importante de la API:** solo se pueden anclar comentarios a
líneas que formen parte del "hunk" del diff (líneas realmente tocadas o su
contexto inmediato). Código preexistente sin cambiar, aunque sea relevante para
el comentario, dará el error `"Line could not be resolved"`. Solución: anclar el
comentario a la línea más cercana que sí sea parte del diff (por ejemplo, la
línea donde se define el reemplazo de algo que quedó obsoleto) y mencionar en el
texto la línea original de referencia.

**4. Si un intento falla a medias**, puede quedar un review en estado `PENDING`
huérfano que bloquea el siguiente intento con el error `"User can only have one
pending review per pull request"`. Verificar y limpiar:

```bash
"C:\Program Files\GitHub CLI\gh.exe" api repos/ryndem/proquifa-pqf2-notificaciones-api/pulls/{N}/reviews --jq ".[] | {id, state}"
"C:\Program Files\GitHub CLI\gh.exe" api repos/ryndem/proquifa-pqf2-notificaciones-api/pulls/{N}/reviews/{REVIEW_ID} -X DELETE
```

**5. Usar `"event": "COMMENT"`** (no `APPROVE` ni `REQUEST_CHANGES`) si el
veredicto final lo va a dar la persona manualmente en GitHub. Dejar `"body": ""`
si solo se quieren los comentarios de línea, sin resumen general.
