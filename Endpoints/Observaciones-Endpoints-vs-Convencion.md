# Observaciones — Endpoints propuestos vs Convención del proyecto

| Campo | Valor |
|---|---|
| **Documentos revisados** | `Endpoints-Finanzas.md`, `Endpoints-Timbrado.md`, `Endpoints-ProquifaDotNet.md`, `Endpoints-LegacySync.md`, `Endpoints-EnvioCorreo.md` |
| **Convención de referencia** | `api/v1/{resource}/{id}/{subresource}` — recurso singular en inglés, CRUD por método HTTP, subrecurso solo para acciones especiales |
| **Fecha de revisión** | 2026-07-17 |
| **Revisor** | Juan David García |
| **Veredicto general** | **Cumple parcialmente** — las soluciones nuevas siguen la convención bien; el módulo Validar Cobro y algunos subrecursos rompen el patrón; los endpoints legacy de ProquifaDotNet (.NET 4.8) son otra historia (aplica regla de "actualización de proceso → español") |

---

## 1. Resumen ejecutivo por aplicativo

| Aplicativo | Naturaleza | Regla de idioma aplicable | Cumplimiento convención de rutas |
|---|---|---|---|
| `ProquifaDotNet.Finanzas` | Solución nueva | **Inglés total** | Alto (con excepciones — ver Obs. 2.x) |
| `ProquifaDotNet.Timbrado` | Solución nueva | **Inglés total** | Alto (4 endpoints) |
| `ProquifaDotNet` (WebApi.Catalogos + WebApi.Logistica) | Aplicativo existente ampliado | Español (Catálogos + actualización Logística); inglés (nuevos endpoints Logística) | Bajo — pero es lo esperado por ser aplicativo legado |
| `LegacySync` | Mecanismo ETL/SSIS | N/A (no expone HTTP) | N/A |
| `ProquifaDotNet.SendInBlue` | Solución nueva | **Inglés total** | Alto (4 endpoints) |
| `ProquifaDotNet.EnvioCorreo` | Aplicativo Nuevo referenciado | **Inglés total** (cuando exista) | **Sin documentación** — bloqueo transversal |

**Total de endpoints propuestos:** ~62 nuevos (Finanzas ~50, Timbrado 4, ProquifaDotNet ~14 nuevos/modificados, SendInBlue 4). El grueso está en Finanzas.

---

## 2. Observaciones sobre `Endpoints-Finanzas.md`

### 2.1 — El módulo Validar Cobro usa un agrupador que rompe el patrón

**Convención:** `api/v1/{resource}/{id}/{subresource}` — el segmento posterior a `/api/v1/` debe ser el **recurso principal**, no un agrupador de módulo.

**Lo que se propone:** todo el wizard de Validar Cobro (RE-023 a RE-030) vive bajo `/api/v1/validate-collection/*`, con el recurso real ubicado **después** del agrupador. Ejemplos:
- `/api/v1/validate-collection/mailbox/{id}/close`
- `/api/v1/validate-collection/payment/{id}`
- `/api/v1/validate-collection/quote/promiseDate`
- `/api/v1/validate-collection/fiscalDocumentLine/{id}/stamp`
- `/api/v1/validate-collection/client/{idCliente}/pendingDocument`

**Análisis:** el archivo mismo lo reconoce y lo llama *"convención propia del módulo"*, confirmada por el equipo. Es una desviación deliberada de la regla, no un descuido.

**Problema:** la convención general no admite agrupadores de módulo. Si esta excepción se acepta para Validar Cobro, invita a otros módulos futuros a hacer lo mismo (`/api/v1/credit-management/*`, `/api/v1/collection-cycle/*`, etc.), y la convención pierde su valor como estándar. El equivalente estándar sería:

| Actual (con agrupador) | Estándar (sin agrupador) |
|---|---|
| `GET /api/v1/validate-collection/mailbox/{id}/pending` | `GET /api/v1/mailbox/{id}/pending` |
| `GET /api/v1/validate-collection/payment/{id}` | `GET /api/v1/payment/{id}` |
| `POST /api/v1/validate-collection/fiscalDocumentLine/{id}/stamp` | `POST /api/v1/fiscalDocumentLine/{id}/stamp` |
| `POST /api/v1/validate-collection/client/search` | `POST /api/v1/client/search` |

**Acción requerida:** decidir formalmente una de tres:
1. **Retirar el agrupador `validate-collection`** y usar solo el recurso — alinea con la convención general.
2. **Mantener el agrupador como excepción** justificada (los recursos `payment`, `mailbox`, `quote`, `client` son compartidos por otros módulos y necesitan agruparse por contexto), pero **documentar la excepción en la convención misma** — evita que otros módulos lo copien sin criterio.
3. **Actualizar la convención global** para admitir `api/v1/{module}/{resource}/{id}/{subresource}` como patrón oficial y aplicarlo consistente en todos los módulos.

La opción 2 es la menos disruptiva pero exige actualizar el documento de reglas. La opción 1 es la más limpia pero implica renombrar ~15-20 endpoints ya propuestos.

### 2.2 — Recurso `cfdi` en minúsculas rompe camelCase del resto

**Convención:** singular en inglés — el resto de la solución usa camelCase (`paymentComplement`, `advanceInvoice`, `creditNote`, `fiscalDocumentLine`).

**Lo que se propone:** `POST /api/v1/cfdi`, `GET /api/v1/cfdi/{id}/xml`, `POST /api/v1/cfdi/search`. Uso de `cfdi` como sigla en minúsculas.

**Análisis:** `cfdi` es un acrónimo (Comprobante Fiscal Digital por Internet), no una palabra. Similar a `xml`, `pdf`, `pdf/preview`. Convención habitual en REST: acrónimos ampliamente conocidos en minúsculas son aceptables (`cfdi`, `xml`, `pdf`). Sin embargo:
- El proyecto ya usa `paymentComplement` en vez de `payment-complement` o `paymentcomplement` → decidió camelCase por consistencia.
- Si se mantiene `cfdi`, hay que **documentarlo como excepción** para que nadie lo cambie a `cFDI` o `Cfdi` por error.

**Recomendación:** dejar `cfdi` en minúsculas (es un acrónimo del dominio fiscal), pero registrarlo como excepción explícita a la regla de camelCase.

### 2.3 — Subrecurso `search` vs `list` inconsistente

**Lo que se propone:**
- `POST /api/v1/cfdi/search` (subrecurso `search`)
- `POST /api/v1/advanceInvoice/search` (`search`)
- `POST /api/v1/proforma/list` (`list`)
- `POST /api/v1/client/search` (dentro de `validate-collection`, `search`)

**Problema:** dos verbos ("search" vs "list") para la misma operación semántica — listado paginado con `QueryInfo`. Rompe la consistencia interna de la solución.

**Acción requerida:** elegir uno (recomendado `search` porque acepta filtros complejos, no solo listado plano) y aplicarlo uniforme. Renombrar `POST /api/v1/proforma/list` → `POST /api/v1/proforma/search`.

### 2.4 — Confusión entre CRUD y acción especial en RE-023 Cobros

**Lo que se propone:**
- `GET /api/v1/validate-collection/payment/{id}` — obtener cobro por Id. CRUD estándar. ✓
- `GET /api/v1/validate-collection/payment` — lista cobros del cliente. Usa `?idCliente={guid}` en query. Colisión potencial con "listar todo".
- `PUT /api/v1/validate-collection/payment` — INSERT o UPDATE del cobro (Paso 1). **Sin `{id}`** — usa PUT como upsert.

**Problema:** `PUT /api/v1/payment` sin ID como INSERT rompe la convención — `PUT` en REST es idempotente sobre un recurso identificado. La convención propia del proyecto exige `POST` para crear y `PUT /{id}` para actualizar. Este endpoint se comporta como "upsert" en la misma ruta sin identificador.

**Acción requerida:**
- Separar `POST /api/v1/validate-collection/payment` (crear borrador) y `PUT /api/v1/validate-collection/payment/{id}` (actualizar borrador existente).
- Para el listado por cliente, usar subrecurso: `GET /api/v1/client/{idCliente}/payment` (más semántico que query string opcional).

### 2.5 — Camino `promiseDate` a nivel colección sin ID

**Lo que se propone:** `PUT /api/v1/validate-collection/quote/promiseDate` — actualiza fecha estimada de pago para **N** proformas (recibe lista en el body).

**Problema:** `PUT` sobre `/quote/promiseDate` (sin `{id}`) actualiza múltiples entidades en batch. La convención espera `PUT /api/v1/quote/{id}/promiseDate` para actualizar la promesa de una proforma específica. Para el batch, opciones más semánticas:
- `POST /api/v1/quote/promiseDate/batch`
- `POST /api/v1/quote/promiseDate` (POST implica creación de N cambios, más aceptable que PUT sin ID)

**Acción requerida:** decidir si el caso de negocio es realmente batch (varios cambios simultáneos), y si sí, usar POST con `/batch` explícito.

### 2.6 — Rutas de RE-032/033 inconsistentes con el resto (sin agrupador `client`)

**Lo que se propone:**
- `GET /api/v1/creditNote` (recurso principal) ✓
- `GET /api/v1/client/{idCliente}/creditNote` (subrecurso de cliente) ✓
- `GET /api/v1/client/{id}/eligibleInvoice` (subrecurso de cliente) ✓
- `GET /api/v1/cfdi/{id}/lineItem` (subrecurso de CFDI) ✓
- `GET /api/v1/creditNote/{id}/pdf/preview` — subrecurso doble `pdf/preview`

**Problema con `pdf/preview`:** el patrón `{resource}/{id}/{subresource}` admite **un** subrecurso, no una cadena `pdf/preview`. La convención sería `GET /api/v1/creditNote/{id}/pdfPreview` o `GET /api/v1/creditNote/{id}/preview?format=pdf`.

Además, en el mismo archivo existe `POST /api/v1/proforma/{id}/pdf` (RE-016) — solo `pdf`, sin `/preview` — para el mismo comportamiento ("generar sin persistir"). Falta consistencia entre requisitos hermanos.

**Acción requerida:** unificar convención de "generar preview de PDF":
- Opción A: `POST /api/v1/{resource}/{id}/pdf?persistir=false` (query string controla persistencia).
- Opción B: `POST /api/v1/{resource}/{id}/pdfPreview` (subrecurso dedicado, más explícito).
- Aplicar a `proforma`, `creditNote`, `advanceInvoice`, `paymentComplement`.

### 2.7 — Método HTTP incorrecto para "cerrar pendiente"

**Lo que se propone:** `PUT /api/v1/validate-collection/mailbox/{id}/close` — cierra pendiente del Buzón (`Activo=0`).

**Análisis:** el subrecurso `close` es una acción, no un recurso. La convención propia del proyecto ejemplifica `POST api/v1/invoice/{id}/cancel` para acciones especiales — la acción se hace con `POST`, no con `PUT`. Consistentemente:
- `POST /api/v1/paymentComplement/stamp` ✓ (usa POST)
- `PUT /api/v1/validate-collection/mailbox/{id}/close` ✗ (debería ser POST)

**Acción requerida:** cambiar a `POST /api/v1/mailbox/{id}/close` (o mantener agrupador si Obs. 2.1 se resuelve por Opción 2).

Mismo caso: `PUT /api/v1/validate-collection/payment/{id}/edit` (RE-024) — la acción `edit` es un `PUT` sobre el recurso, no un subrecurso. Debería ser `PUT /api/v1/payment/{id}` directamente, sin el subrecurso `edit`.

### 2.8 — `PUT /api/v1/validate-collection/client/{idCliente}/association/draft` — semántica dudosa

**Lo que se propone:** auto-guardado del estado en progreso de la asociación cobro↔documento(s).

**Problema:** `association/draft` es un subrecurso compuesto (`association` es la relación, `draft` es su estado). La convención admite un subrecurso, no dos anidados.

**Acción requerida:** aplanar a `PUT /api/v1/client/{idCliente}/associationDraft` o `POST /api/v1/client/{idCliente}/association/save`.

---

## 3. Observaciones sobre `Endpoints-Timbrado.md`

### 3.1 — Cumplimiento general: ✓ correcto

Los 4 endpoints propuestos **cumplen la convención**:

| Endpoint | Cumple |
|---|---|
| `POST /api/v1/stamp/invoice` | ✓ Sujeto `stamp` con subrecurso `invoice` (tipo de documento) |
| `POST /api/v1/stamp/payment-complement` | ⚠ ver Obs. 3.2 (kebab-case vs camelCase) |
| `POST /api/v1/stamp/credit-note` | ⚠ ver Obs. 3.2 |
| `POST /api/v1/stamp/cancel` | ✓ Acción de negocio |

### 3.2 — Kebab-case vs camelCase (inconsistencia interna del proyecto)

**Lo que se propone:** `POST /api/v1/stamp/payment-complement` y `POST /api/v1/stamp/credit-note` (kebab-case).

**Inconsistencia:** en `Endpoints-Finanzas.md` los mismos conceptos son `paymentComplement` y `creditNote` (camelCase). El mismo dominio tiene **dos representaciones** según qué aplicativo lo expone.

**Acción requerida:** unificar. La convención del proyecto ejemplifica `paymentComplement`, `creditNote` en camelCase — Timbrado debería alinearse:
- `POST /api/v1/stamp/paymentComplement`
- `POST /api/v1/stamp/creditNote`

### 3.3 — Verbo `stamp` como recurso principal

**Análisis:** el patrón `/api/v1/{resource}/{id}/{subresource}` espera un sustantivo en `{resource}`. Aquí `stamp` es un verbo. El propio archivo lo reconoce y lo justifica: *"Servicio técnico, no expone un recurso de negocio: las rutas usan una acción (stamp) más el tipo de documento en vez de un sustantivo CRUD, ya que no hay una entidad persistida detrás."*

**Alternativas semánticamente correctas:**
- `POST /api/v1/stampingRequest` con `type: "invoice" | "paymentComplement" | "creditNote"` en body (un solo endpoint discriminado).
- `POST /api/v1/invoice/stamp`, `POST /api/v1/paymentComplement/stamp`, `POST /api/v1/creditNote/stamp` — pero esto **choca con Finanzas**, que ya expone esos recursos como propios.

**Recomendación:** aceptar la excepción actual `stamp/{documentType}` documentándola en las reglas del proyecto como patrón válido para servicios técnicos sin recurso persistente. La justificación del archivo es válida — es mejor mantener un contrato/validador por tipo que un endpoint monolítico discriminado en el body.

---

## 4. Observaciones sobre `Endpoints-ProquifaDotNet.md`

### 4.1 — Aplican reglas distintas por ser aplicativo existente

**Reglas aplicables al aplicativo:**
- WebApi.Catalogos: **codificación en español** (no cambia con R16).
- WebApi.Logistica actualización de proceso: **español**.
- WebApi.Logistica nuevo endpoint / nuevo proceso: **inglés** (controller y modelo).

**Cumplimiento observado:**
- Todos los endpoints en `Endpoints-ProquifaDotNet.md` están en español (`/EmpresaDatosBancarios`, `/ClienteCartera/ReasignarCobrador`, `/ArchivoCliente/DocumentoRegulatorio`, `/tpPedido/cancelar`, `/PretramitarPedido/transaccion`).
- **Ninguno usa `api/v1/`** — es aplicativo legado, no aplica la convención nueva.
- Los "nuevos" endpoints están en español (`ReasignarCobrador`, `GestoresDeCobranza`, `DocumentoRegulatorio`, `PorRegion`) — **contradicen** la regla *"si es nuevo endpoint en Logística, controller/modelo en inglés"*.

**Análisis por caso:**

| Endpoint nuevo | Área | Cumple regla de idioma |
|---|---|---|
| `/vEmpresaDatosBancarios` | Catálogos | ✓ (Catálogos = español) |
| `/ClienteCartera/ReasignarCobrador` | Catálogos | ✓ (Catálogos = español) |
| `/Usuario/GestoresDeCobranza` | Catálogos | ✓ (Catálogos = español) |
| `/ArchivoCliente/DocumentoRegulatorio` | Catálogos | ✓ (Catálogos = español) |
| `/catRegimenFiscal/PorRegion` | Catálogos | ✓ |
| `/catTipoSociedadMercantil/PorRegion` | Catálogos | ✓ |
| `/ClienteDatosBancarios` (CRUD nuevo) | Catálogos | ✓ |
| `/tpPedido/cancelar` (RE-010) | Logística — **nuevo endpoint** | ✗ debería ser `/tpPedido/cancel` o mejor `/api/v1/order/{id}/cancel` (en inglés) |
| `/pedidos/{idTpPedido}/cancelar-falta-pago` (RE-023) | Logística — **nuevo endpoint** | ✗ debería ser `POST /api/v1/order/{id}/cancelForNonPayment` |

**Acción requerida:**
1. **Confirmar con arquitectura** si los endpoints nuevos en `WebApi.Logistica` deben migrar a inglés cuando son "nuevo proceso" — RE-010 y RE-023 son claramente nuevos.
2. Si sí: renombrar `tpPedido/cancelar` y `pedidos/{id}/cancelar-falta-pago` a inglés siguiendo la convención `api/v1/order/{id}/{action}`.
3. Si no: **actualizar la regla del proyecto** para reflejar que Logística nuevo también puede estar en español si se integra con controllers legacy del mismo módulo.

### 4.2 — Uso de query string vs path para IDs

**Lo que se propone:** muchos endpoints existentes de Catálogos usan `?id={guid}` en query string en vez de `/{id}` en path:
- `GET /EmpresaDatosBancarios?id={guid}` (en vez de `/EmpresaDatosBancarios/{id}`)
- `DELETE /ClienteCartera?id={guid}`
- `GET /Usuario/GMUsuarioClienteCarteraDetalle?idUsuario={guid}`

**Análisis:** convención heredada del aplicativo legado. No aplica la regla del proyecto (que es para nuevos endpoints/aplicativos), y no vale la pena refactorizar endpoints existentes solo por estilo.

**Acción requerida:** ninguna. Documentar como convención legada aceptada. Los nuevos endpoints de Catálogos/Logística deberían seguir esta misma convención por consistencia interna del aplicativo, no la nueva.

---

## 5. Observaciones sobre `Endpoints-LegacySync.md`

### 5.1 — No aplica convención HTTP

**Cumplimiento:** LegacySync no expone endpoints HTTP — solo ETL/SSIS. La revisión de convención no aplica.

**Observación válida:** el archivo enumera 8 transferencias (E1-E8) con estado *"Pendiente definición canal de transferencia"* en la mayoría. Esto es una **brecha real**, no una observación de estilo: sin canal definido (tabla ETL, RabbitMQ, API Legacy directa), los flujos R16 no pueden implementarse.

**Acción requerida:** priorizar la definición del canal con arquitectura (brecha B3 de RE-028, ya reconocida).

---

## 6. Observaciones sobre `Endpoints-EnvioCorreo.md`

### 6.1 — Convención de rutas: ✓ correcto (`ProquifaDotNet.SendInBlue`)

Los 4 endpoints de SendInBlue **cumplen** la convención:

| Endpoint | Cumple |
|---|---|
| `POST /api/v1/mail/send` | ✓ |
| `POST /api/v1/mail/simple` | ✓ |
| `POST /api/v1/mail/html` | ✓ |
| `POST /api/v1/mail/template` | ✓ |

Recurso singular en inglés (`mail`), subrecursos como acciones (`send`, `simple`, `html`, `template`), prefijo `/api/v1/`.

**Nota menor:** `simple`, `html`, `template` son **variantes del mismo tipo de envío**, no subrecursos en el sentido REST estricto. Alternativa: `POST /api/v1/mail/send` único con discriminador `mode: "simple" | "html" | "template"` en el body. Sin embargo, la separación actual permite validadores dedicados por tipo (mismo patrón que Timbrado con `invoice`/`paymentComplement`/`creditNote`) y es aceptable.

### 6.2 — Bloqueo transversal — `ProquifaDotNet.EnvioCorreo` no está documentado

**Problema crítico:** el archivo dedica su sección final a aclarar que *"`ProquifaDotNet.EnvioCorreo`... no existe ningún requisito (RE-FU-XXX o NO-FU-XXX) que documente el contrato, los endpoints o la arquitectura"* de este aplicativo — pese a que **múltiples requisitos de Finanzas y Timbrado lo referencian como canal obligatorio** de envío de correo desde el módulo Validar Cobro y otros flujos.

**Impacto en los otros archivos:**
- `Endpoints-Finanzas.md` menciona *"despacha correo con CFDI + Confirmación vía ProquifaDotNet.EnvioCorreo"* en RE-028 sin especificar el endpoint.
- `Endpoints-Timbrado.md` no menciona correo (correcto — es servicio técnico).
- El requisito RE-030 (revisado en el archivo `Observaciones-DIS-SOL-vs-RE-030.md`) tiene la misma brecha en Observación 7.3.

**Acción requerida:** priorizar la creación del requisito de `ProquifaDotNet.EnvioCorreo` (RE-FU o NO-FU) con:
- Endpoints propuestos (esperablemente similares a los de SendInBlue: `POST /api/v1/mail/send`, `POST /api/v1/mail/html`, etc.).
- Diferencia clara vs SendInBlue: si SendInBlue es solo para migración del envío legacy de Venta Interna, EnvioCorreo debe justificar por qué es una app distinta y no un endpoint más de SendInBlue.
- Contrato para adjuntos de MinIO (multi-región).
- Contrato para plantillas Brevo (PMO #31 pendiente).

Sin este requisito, **todos los criterios de "envío de correo tras timbrado"** de RE-019, RE-028, RE-029, RE-030, RE-032, RE-033 quedan sin fundamento técnico.

---

## 7. Inventario de desviaciones y consolidado

| # | Área | Endpoint(s) afectado(s) | Tipo de desviación | Prioridad |
|---|---|---|---|---|
| D1 | Finanzas | Todos los `/api/v1/validate-collection/*` (~15 rutas) | Agrupador de módulo no previsto por la convención | Alta — decisión formal |
| D2 | Finanzas | `POST /api/v1/proforma/list` vs `POST /api/v1/cfdi/search` | `list` vs `search` (dos verbos para lo mismo) | Media |
| D3 | Finanzas | `PUT /api/v1/validate-collection/payment` (sin ID) | PUT como upsert sin identificador | Alta |
| D4 | Finanzas | `PUT /api/v1/validate-collection/quote/promiseDate` | PUT batch sin `{id}` | Media |
| D5 | Finanzas | `GET /api/v1/creditNote/{id}/pdf/preview` | Subrecurso anidado `pdf/preview` | Baja |
| D6 | Finanzas | `PUT /api/v1/validate-collection/mailbox/{id}/close` y `.../payment/{id}/edit` | Método HTTP incorrecto (PUT en vez de POST para acción; PUT sobre subrecurso `edit` en vez de sobre el recurso) | Media |
| D7 | Finanzas | `PUT /api/v1/validate-collection/client/{idCliente}/association/draft` | Subrecurso doble anidado | Baja |
| D8 | Finanzas / Timbrado | `paymentComplement` (camelCase) vs `payment-complement` (kebab-case) | Inconsistencia case entre aplicativos | Media |
| D9 | Finanzas | `cfdi` en minúsculas vs resto en camelCase | Excepción no documentada (acrónimo) | Baja |
| D10 | Timbrado | `POST /api/v1/stamp/...` — `stamp` como recurso | Verbo como recurso principal (excepción justificada) | Baja — documentar |
| D11 | ProquifaDotNet | `/tpPedido/cancelar`, `/pedidos/{id}/cancelar-falta-pago` (Logística nuevos) | En español pese a ser "nuevo endpoint Logística" | Media — confirmar regla |
| D12 | ProquifaDotNet.EnvioCorreo | **Sin documentación** | Bloqueo transversal — apps que lo referencian quedan colgadas | **Crítica** |
| D13 | LegacySync | Canal de transferencia sin definir (E1-E3, E6) | Brecha de arquitectura, no de convención | Alta |

---

## 8. Recomendaciones finales

En orden de prioridad para cerrar brechas de convención:

1. **Definir el requisito de `ProquifaDotNet.EnvioCorreo`** (D12) — desbloquea RE-019, RE-028, RE-029, RE-030, RE-032, RE-033. Es la brecha más costosa de todo el conjunto.
2. **Decidir formalmente sobre el agrupador `validate-collection`** (D1) — actualizar la convención global o retirar el agrupador, no dejar la excepción implícita.
3. **Unificar case entre aplicativos** (D8) — `paymentComplement` o `payment-complement`, no ambos. Recomendado: camelCase para alinear con Finanzas.
4. **Corregir métodos HTTP mal aplicados** (D3, D4, D6) — `PUT` sin ID como upsert, batch con `PUT`, acciones con `PUT` en vez de `POST`.
5. **Unificar `search` vs `list`** (D2) — usar `search` en todos los listados paginados con `QueryInfo`.
6. **Confirmar regla de idioma para Logística nuevo** (D11) — decisión con arquitectura antes de crear más endpoints en español.
7. **Documentar excepciones aceptadas** (D9, D10) — `cfdi` en minúsculas y `stamp/{documentType}` en Timbrado — como reglas explícitas del proyecto, no como excepciones ad hoc.
8. Aplanar subrecursos anidados (D5, D7) — cambios cosméticos, se pueden abordar en PR de limpieza.

**Conclusión:** el conjunto de endpoints está mayoritariamente alineado con la convención propuesta, pero contiene 13 desviaciones que oscilan entre incidencias menores de estilo (case, subrecursos anidados) y bloqueos duros (falta de documentación de `ProquifaDotNet.EnvioCorreo`). Las soluciones nuevas (`Finanzas`, `Timbrado`, `SendInBlue`) son las mejor alineadas; el módulo Validar Cobro es donde más se concentran las desviaciones y merece una revisión formal antes de codificar. Los endpoints legacy de `ProquifaDotNet` juegan con reglas propias que no invalidan la convención, pero exigen definir el criterio de idioma para "nuevos endpoints en Logística" antes de seguir agregando.
