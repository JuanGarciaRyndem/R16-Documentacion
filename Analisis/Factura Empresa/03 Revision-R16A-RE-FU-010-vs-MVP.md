# Revisión — R16A-RE-FU-010 vs MVP Separación PROQUIFA / GOLOCAER

> Contraste del diseño back del requisito **R16A-RE-FU-010** (Tramitación de pedidos Crédito, con Pago contra entrega y flujo de Cancelación) contra el MVP de separación por infraestructura definido en `01 Separacion-PROQUIFA-GOLOCAER.md` §7.
>
> **Documento base revisado:** `Requisitos/R16A-RE-FU-010/R16A-RE-FU-010-DIS-SOL-Back.md` (versión 1.0, 02-jul-2026).

## 0. Contexto — Legacy sólo existe en México

> **Regla de oro:** **no existe Legacy para Perú.** Nada de la operación PE se transfiere a Legacy: el ciclo del pedido termina en ProquifaDotNet (PQF2). Esto es precisamente lo que RE-010 formaliza en su Cambio 1.
>
> Por lo tanto, cuando en el resto del documento se hable de "legacies separados por empresa", se refiere exclusivamente a **dos legacies mexicanos**:
>
> - **Legacy MX PROQUIFA** — recibe pedidos MX cuya empresa emisora es PROQUIFA (USP).
> - **Legacy MX GOLOCAER** — recibe pedidos MX cuya empresa emisora es GOLOCAER (resto de marcas).
>
> **Perú (Golocaer SAC) no tiene destino Legacy** en ninguna de las dos empresas del MVP. Toda la lógica de ruteo Legacy es un problema México-only.

---

## 1. Resumen ejecutivo

RE-FU-010 es **funcionalmente compatible** con el MVP: el mismo binario corre en las dos instancias (PROQUIFA y GOLOCAER) y los tres cambios (Cambio 1 exclusión PE, Cambio 2 Pago contra entrega, Cambio 3 endpoint Cancelación) no dependen de qué empresa esté ejecutando.

Existen, sin embargo, **cinco puntos técnicos** que deben quedar explícitos antes de codear, porque el diseño actual asume una sola empresa emisora y no diferencia PROQUIFA vs GOLOCAER en los puntos de contacto con Legacy, folios y catálogos regionalizados. Si no se atienden, el MVP separado emite pedidos con el RFC correcto pero puede rutear el buzón / consecutivo Legacy a la instancia incorrecta.

Semáforo:

| # | Punto | Estado |
|---|---|---|
| 1 | Rutear `spActualizarBuzonPedidoLegacyLegacy` al legacy correcto por empresa | 🟡 Ajuste requerido |
| 2 | `ObtenerFolioPorRegion` — series por empresa/RFC | 🟡 Ajuste requerido |
| 3 | Literal `region.Clave == Constants.Regiones.Mexico` — mismo antipatrón que `Empresa.Clave == "…"` | 🟢 Compatible, oportunidad de refactor |
| 4 | `CondicionPagoFlujoHelper.EsCondicionCredito` — helper puro | 🟢 Compatible sin cambio |
| 5 | Endpoint `tpPedidoCancelacionController` | 🟢 Compatible — un endpoint por instancia |
| 6 | Panel Información de Facturación regionalizado (CA-4) | 🟢 Se despliega, PROQUIFA-MX simplemente no usa el catálogo PE |
| 7 | Correo post-commit (RabbitMQ → Brevo) | 🟢 Compatible — cada instancia tiene su propio broker y cuenta Brevo |
| 8 | PDF Confirmación de Pedido | 🟢 Compatible — plantillas por instancia en DocumentBuilder |
| 9 | P6 — `spActualizarBuzonPedidoLegacyLegacy` provisional (será reemplazado por RE-012 LegacySync) | 🔴 Oportunidad crítica — RE-012 debe nacer con multi-empresa desde el diseño |
| 10 | P5 — Perú + controlados como riesgo | 🟢 Aplica sólo a GOLOCAER; PROQUIFA no maneja PE |

---

## 2. Análisis por punto

### 2.1 🟡 Ruteo del ETL de buzón Legacy por empresa (sólo MX)

**Diseño actual (RE-010 §4.2):**

```csharp
if (region.Clave == Constants.Regiones.Mexico)
{
    tpPedidoBO.ActualizarBuzonPedidoTramitadoLegacy(
        GMtpPedidoTramitarCorreo.tpPedido.IdCliente);
}
```

**Situación en MVP separado (recordando §0):** la rama del `if` **sólo se ejecuta para pedidos MX** — para PE la instancia GOLOCAER-SAC nunca invoca al SP y no hay Legacy destino. El ruteo por empresa aplica únicamente al camino MX.

**Cómo queda por instancia:**

| Instancia MVP | Pedido MX | Pedido PE |
|---|---|---|
| PROQUIFA (USP, MX) | Escribe a Legacy MX PROQUIFA por connection string | N/A — PROQUIFA no comercializa PE |
| GOLOCAER (resto, MX + PE) | Escribe a Legacy MX GOLOCAER por connection string | Omite ETL — sin Legacy PE |

**Recomendación:**

- En el MVP cada instancia MX apunta a **su propio** Legacy vía `Web.config` (connection string distinta). El ruteo por empresa queda **implícito por config**, sin cambio en el código.
- **Verificar** que `spActualizarBuzonPedidoLegacyLegacy` no cruce datos entre RFCs: si el SP hoy lee tablas asumiendo "todos los pedidos son del mismo grupo", debe filtrar por `IdEmpresa` del pedido o por el RFC del legacy destino.
- La instancia GOLOCAER que reciba un pedido PE **ya está cubierta** por la exclusión de Cambio 1 — no hay que hacer nada adicional.
- Documentar en RE-010 §4.2 la nota: *"En MVP separado, cada deployment MX apunta a su Legacy vía connection string; el SP no debe cruzar RFCs. PE nunca invoca al SP."*

### 2.2 🟡 `ObtenerFolioPorRegion` — series de folios por empresa

**Diseño actual:** `ObtenerFolioPorRegion(db, region, esCredito)` en L214.

**Problema en MVP separado:** las **series de folios fiscales** son **por RFC** (PROQUIFA emite con serie A propia; GOLOCAER con serie B propia). Hoy la función depende sólo de `region` y `esCredito`. Si dos instancias comparten el mismo generador, hay riesgo de colisión.

**En el MVP separado hay dos BD**, cada una con su tabla de consecutivos, así que **el aislamiento es automático a nivel BD**. Pero:

- Verificar que la función **no** consulte una tabla que se replique entre entornos sin partición.
- Confirmar que la **serie fiscal** (no sólo el folio interno) también se resuelva por empresa. Los certificados CSD y series los da `Empresa.IdArchivoCertificadoFacturacion`, `Empresa.ClaveLlavePrivadaCertificacion` — así que sí quedan por empresa.
- Documentar en RE-010 §4.1 la nota: *"En MVP separado, la serie fiscal se resuelve por `IdEmpresa` del pedido, disponible en `tpPedido.IdEmpresa`."*

**Riesgo residual:** si `ObtenerFolioPorRegion` decide una **serie** basada sólo en región (`MX-CR-…` vs `PE-CR-…`) sin considerar RFC, en producción PROQUIFA y GOLOCAER podrían emitir con la misma serie. Revisar antes de codear.

### 2.3 🟢 Literal `region.Clave == Constants.Regiones.Mexico` — mismo antipatrón que `Empresa.Clave`

**Diseño actual (§4.2):**

```csharp
if (region.Clave == Constants.Regiones.Mexico) { … }
```

**Observación:** es el mismo patrón que en el aplicativo actual con `Empresa.Clave == "golocaer"`. Compatible con el MVP (el mismo binario en ambos entornos evalúa la constante), pero es una **oportunidad** de aplicar el mismo principio propuesto en `02 Impacto-Back-BD-PROQUIFA-GOLOCAER.md` §2.4: reemplazar literales por flags configurables (`region.TransfiereABuzonLegacy` como columna nueva, por ejemplo).

**No es bloqueante** para RE-010, pero conviene registrarlo como deuda técnica alineada con la separación.

### 2.4 🟢 `CondicionPagoFlujoHelper.EsCondicionCredito` — helper puro

Sin dependencia de empresa. El binario compartido lo evalúa igual en las dos instancias. **Compatible sin cambio.**

**Nota de diseño (D1 en RE-010 §2.2):** propone mover el helper a componente compartido si otros módulos R16 lo necesitan. **Coherente** con el patrón "un binario, dos configuraciones" del MVP: cualquier helper compartido queda automáticamente disponible en ambas instancias.

### 2.5 🟢 Endpoint `tpPedidoCancelacionController`

El controller nuevo `POST /TramitarPedido/Cancelar` se despliega tal cual en las dos instancias — cada una expone su propio endpoint bajo su URL:

- `https://app-proquifa.local/TramitarPedido/Cancelar`
- `https://app-golocaer.local/TramitarPedido/Cancelar`

**Compatible sin cambio.** El Front (R16A-173) invoca al endpoint correspondiente según la instancia activa.

### 2.6 🟢 Panel Información de Facturación regionalizado (CA-4)

RE-010 depende de RE-FU-005 para exponer campos regionalizados de facturación (incluye el catálogo 51 SUNAT para PE). En el MVP separado:

- **PROQUIFA (USP, MX only)** — no comercializa PE. Los campos SUNAT existen en el binario pero no se usan. Pedidos PE tampoco llegan aquí.
- **GOLOCAER (MX + PE)** — usa los campos SUNAT para el subconjunto de clientes PE. Recordar §0: los pedidos PE **no** transfieren a Legacy — el ciclo termina en PQF2. La facturación PE se emite y se archiva localmente, sin ETL saliente.

**Sin bloqueo para RE-010.** La brecha 3 de RE-FU-005 (catálogo SUNAT) sigue viva pero afecta sólo a la instancia GOLOCAER en su operación PE.

### 2.7 🟢 Correo post-commit (RabbitMQ → Brevo)

Cada instancia del MVP tiene **su propio broker RabbitMQ** y **su propia cuenta Brevo** (§7.1 de la separación). El correo se enruta automáticamente por config. Sin cambio en el código de RE-010.

**Recomendación:** verificar que la plantilla del correo (asunto, remitente, firma) incluya la identidad de la empresa emisora. En Brevo, cada cuenta tiene su remitente verificado (`no-reply@proquifa.com` vs `no-reply@golocaer.com`).

### 2.8 🟢 PDF Confirmación de Pedido

Se genera dentro del BO `tpPedidoTramitarTransaccionBO`. En el MVP separado, cada instancia de DocumentBuilder monta las plantillas de su empresa (logotipo, RFC, dirección, avisos legales). Sin cambio en RE-010.

**Recomendación:** confirmar que la plantilla actual de Confirmación de Pedido puede parametrizarse por empresa (leer `IdEmpresa` del pedido y renderizar con esos datos). Ya debería estar así en la implementación actual, dado que `tpPedido.IdEmpresa` existe hoy.

### 2.9 🔴 P6 — `spActualizarBuzonPedidoLegacyLegacy` y RE-012 LegacySync (sólo MX)

Este es el punto **más crítico** para el MVP.

RE-010 marca el nombre `spActualizarBuzonPedidoLegacyLegacy` como **provisional** y anuncia que RE-012 (LegacySync) lo reemplazará. Si se aprueba el MVP con infraestructura separada, **RE-012 debe nacer con multi-empresa (dos legacies MX) en su diseño**, no adaptarse después.

**Alcance de RE-012 bajo el MVP:**

- Sólo enruta pedidos **MX** — PE ya está excluido por Cambio 1 y no tiene Legacy destino.
- Debe distinguir entre **Legacy MX PROQUIFA** y **Legacy MX GOLOCAER** según `IdEmpresa` del pedido.
- **No** requiere considerar un tercer destino para PE.

**Recomendaciones concretas para RE-012:**

- La API/worker de LegacySync debe recibir (o inferir de la conexión activa) el `IdEmpresa` o `RFC` de la empresa emisora del pedido y rutear al Legacy MX correspondiente.
- El contrato debe permitir dos endpoints físicamente distintos (uno por empresa) sin que el aplicativo cliente lo sepa — la config del `HttpClient` / connection string decide.
- Los mensajes en RabbitMQ deben ir en colas particionadas por empresa (`legacysync-mx-pqf`, `legacysync-mx-gol`) o en un único broker con `routing_key` que incluya el RFC destino.
- El worker de LegacySync debe correr en dos instancias (una por empresa) o ser tenant-aware. **Ninguna instancia procesa pedidos PE.**
- Si en el diseño de RE-012 alguien propone un endpoint / cola / SP para PE, es señal de error — debe rechazarse por contradecir §0.

**Acción sugerida:** agregar una nota en RE-010 §1.4 (P6) o directamente en RE-012 apuntando a este documento (§0 y §2.9) y a `02 Impacto-Back-BD-PROQUIFA-GOLOCAER.md` §4.

### 2.10 🟢 P5 — Perú + controlados

Riesgo transversal. En el MVP separado:

- **PROQUIFA (USP-MX)** — no maneja PE, no aplica.
- **GOLOCAER** — sigue expuesto al riesgo si mantiene operación Perú. Consecuencias regulatorias, si las hubiera, se contienen dentro de PQF2 dado que PE no tiene canal Legacy (§0).

**Sin cambio** en el alcance de RE-010; sólo se refina el ámbito de afectación.

---

## 3. Efecto de RE-010 sobre el MVP (impacto inverso)

Además de verificar si RE-010 es compatible con el MVP, conviene evaluar si RE-010 **afecta** al MVP o cambia alguna decisión de la separación.

| Aspecto | ¿RE-010 obliga a cambiar algo del MVP? |
|---|---|
| Alcance IN del MVP | No — los MVP-1 a MVP-12 no se ven afectados. |
| Alcance OUT del MVP | No — RE-010 no requiere cotización cross-company, DWH, MDM, SSO ni interfacturación. |
| Ruta F0–F7 | No — RE-010 puede implementarse antes o durante F1–F2 sin dependencias con F3 (infra x2). |
| Cambios mínimos en código MVP (§7.7) | Alineado — RE-010 ya introduce un helper puro y respeta el patrón de un binario. |
| Riesgos MVP-R1 a R7 | Sin nuevos riesgos aportados por RE-010. |

**Conclusión:** RE-010 puede implementarse en el aplicativo **antes de la duplicación de infraestructura** y quedará automáticamente disponible en ambas instancias cuando se dispare la fase F3 del MVP.

---

## 4. Puntos que RE-010 debería documentar explícitamente

Sugerencias de edición al documento `R16A-RE-FU-010-DIS-SOL-Back.md`:

1. **§1.2 Alcance — "No incluye"** → agregar: *"Separación de razones sociales PROQUIFA / GOLOCAER (documentada en `Analisis/Factura Empresa/`). Este diseño asume que el binario opera indistintamente en cualquiera de las dos instancias del MVP."*
2. **§1.4 P6** → ampliar: *"En el marco del MVP de separación por infraestructura, RE-012 debe rutear a dos legacies distintos (uno por RFC). Ver `Analisis/Factura Empresa/03 Revision-R16A-RE-FU-010-vs-MVP.md` §2.9."*
3. **§4.1 (helper esCredito)** → nota: *"El helper es puro (sin dependencias de tenant). Se despliega igual en las dos instancias del MVP."*
4. **§4.2 (condición región ETL Legacy)** → nota: *"En MVP separado, cada instancia apunta a su legacy vía connection string. Verificar que `spActualizarBuzonPedidoLegacyLegacy` no cruce datos entre RFCs."*
5. **§5 Impacto técnico** → agregar fila: *"No hay impacto en el modelo `Empresa` ni en `MarcaEmpresa`. El requisito RE-010 es tenant-agnostic."*
6. **§7.3 Casos críticos** → agregar caso: *"En MVP separado, tramitar un pedido en la instancia PROQUIFA debe generar folio con serie PROQUIFA; el mismo pedido en la instancia GOLOCAER debe generar folio con serie GOLOCAER. Ambos deben poder coexistir sin colisión."*

---

## 5. Preguntas nuevas que abre RE-010 para el MVP

Se agregan al banco de preguntas del análisis maestro:

1. ¿La función `ObtenerFolioPorRegion` decide la **serie fiscal** o sólo el **folio interno**? Si decide serie, ¿se puede refactorizar para leer la serie de `Empresa`?
2. En el MVP separado, ¿el consecutivo `IncrementarConsecutivoLegacy` es una **tabla por empresa** en cada BD MX, o vive en el legacy MX (que también se duplica)? (No aplica a PE — §0.)
3. Post-corte, ¿los **pedidos históricos** con `IdEmpresa` de la razón social anterior se pueden cancelar desde la nueva instancia, o quedan cancelables sólo en modo lectura?
4. El **modal de confirmación de cancelación** (R16A-173, Front) — ¿debe mostrar el RFC de la empresa que está cancelando para evitar confusión?
5. La cancelación de un pedido en PROQUIFA — ¿debe notificar de algún modo a GOLOCAER (si son del mismo grupo cliente), o el aislamiento es total?
6. Para pedidos **PE cancelados**: ¿existe alguna comunicación externa (contable, fiscal PE) que deba dispararse aunque no exista canal Legacy?

---

## 6. Conclusión y recomendación

RE-010 puede **avanzar sin bloqueos** en la línea de tiempo del MVP. Los tres cambios propuestos son tenant-agnostic y el helper `EsCondicionCredito` sigue el patrón "un binario, dos configuraciones" que exige el MVP.

**Acciones recomendadas antes de codear RE-010:**

1. Añadir al documento `R16A-RE-FU-010-DIS-SOL-Back.md` las cinco notas de §4 de este archivo.
2. Al momento de diseñar RE-012 (LegacySync), incorporar el patrón multi-empresa desde el arranque — no como adaptación posterior.
3. Antes de implementar §4.1 (helper), verificar en código si `ObtenerFolioPorRegion` decide serie fiscal o sólo folio interno (§2.2 de este documento).
4. Confirmar en QA que las plantillas de PDF y correo respetan `tpPedido.IdEmpresa` — ya debería estar así, pero validarlo.

---

*Documento de revisión — versión 0.1. Complementa `01 Separacion-PROQUIFA-GOLOCAER.md` y `02 Impacto-Back-BD-PROQUIFA-GOLOCAER.md`.*
