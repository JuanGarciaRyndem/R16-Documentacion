# Documento de Análisis y Recomendación (DAR)
### Mecanismo de Orquestación de Flujos para `Proquifa.Pqf2.LegacySync` — TPSC-RE-FU-008

| Campo         | Valor                                                                          |
| ------------- | ------------------------------------------------------------------------------ |
| **PROYECTO**  | R16 - Adquisiciones                                                            |
| **REQUISITO** | TPSC-RE-FU-008 — Buzón de Cobros (`Proquifa.Pqf2.LegacySync`, Canal ETLCobros) |
| **VERSIÓN**   | 1.3                                                                            |
| **FECHA**     | 22/06/2026                                                                     |
| **AUTOR**     | Carlos Ivan Morales Carreon                                                    |
| **REVISOR**   |                                                                                |
| **ESTADO**    | Decisión confirmada — Hangfire (2026-06-22)                                    |

---

## 1. Contexto y problema a resolver

`LegacySync.Worker` necesita un mecanismo que: (a) detecte registros pendientes de sincronizar a Legacy (`SyncControl`), (b) los ejecute con reintento automático clasificado (Transient reintenta con backoff, Permanent no reintenta y alerta), (c) ofrezca visibilidad operativa del estado de cada sincronización, y (d) sea sostenible de mantener por un solo desarrollador.

El DIS-SOL (`TPSC-RE-FU-008-DIS-SOL.md`, línea 810) ya especifica **Hangfire** para esto. Este DAR no parte de cero: existe porque se preguntó explícitamente si esa elección se sostiene frente a RabbitMQ y otras alternativas, dado que **RabbitMQ ya existe y está en uso real en el ecosistema** (ver sección 5) — no es una opción hipotética que se pueda descartar sin mirar.

---

## 2. Restricción de diseño que ya existe — esto no es un candado permanente, pero el costo de cambiar crece con el tiempo

`LegacySync` no ha iniciado construcción — LB-T1 todavía no arranca. Hoy, cambiar de mecanismo no cuesta nada porque no hay código que reescribir. Una vez que `SyncJobBase`, los atributos de reintento y el flujo de `SyncControl` estén implementados (LB-T1 a LB-T10), cambiar de Hangfire a otra cosa sí sería costoso — por eso conviene decidir con rigor ahora, no sobre la marcha.

---

## 3. Opciones descartadas sin entrar a comparativa

| Opción | Por qué se descarta |
|---|---|
| Quartz.NET | Mismo paradigma que Hangfire (scheduler in-process), pero sin dashboard ni persistencia incluidos por defecto — hay que agregarlos aparte. Hangfire ya cubre lo mismo con menos configuración, y ya tiene precedente real en este ecosistema (`TaskSchedulerPqf`); Quartz no |
| Azure Service Bus / AWS SQS | Introduciría una dependencia cloud nueva, fuera de la infraestructura on-premise actual (SQL Server, RabbitMQ on-prem). Sin ninguna ventaja sobre RabbitMQ, que ya resuelve el mismo problema y ya está desplegado |
| SQL Server Service Broker | Existe en el ecosistema — confirmado: colas reales (`SERVICE_QUEUE`) en `ProquifaDotNet` (ej. `dbo_cotCotizacion_..._Sender`/`_Receiver`). Pero es un mecanismo T-SQL de bajo nivel, sin tooling moderno de .NET, sin dashboard, de configuración frágil (manejo manual de conversaciones/certificados). Ninguna ventaja sobre Hangfire/RabbitMQ para código nuevo en .NET 10 |
| Cola interna en memoria (`System.Threading.Channels`, el mismo patrón que usa `Proquifa.Pqf2.MailbotWorker.sln` entre su API y su Worker) | No sobrevive un reinicio del proceso — inaceptable para sincronización financiera, donde perder un mensaje en memoria significa un cobro que nunca llega a Legacy |
| Kafka | Sin ningún precedente en el ecosistema — verificado: cero referencias en `ProquifaDotNet`, `Core.Pqf`, `Core.CrudTools`, `proquifa-punchout-backend`, `interfaces-proquifanet2` ni `SincronizadorPqfLegacy`. Pensado para streaming de eventos de alto volumen con múltiples consumidores y particionado — sobredimensionado para el volumen real de Cobros (una fracción de ~5,675 correos/mes de todo Mailbot). Introduciría infraestructura nueva (cluster Kafka + Zookeeper/KRaft) sin ninguna ventaja sobre RabbitMQ, que ya resuelve el mismo tipo de problema y ya está desplegado |

---

## 4. Opciones evaluadas

| Criterio                                     | Hangfire                                                                                                                                                                       | RabbitMQ                                                                                                                                                                                                                    |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Modelo de uso                                | Background/scheduled jobs in-process, con dashboard incluido                                                                                                                   | Message broker — pub/sub o cola punto a punto entre procesos distintos                                                                                                                                                      |
| Reintentos con backoff clasificado           | Nativo — `AutomaticRetryAttribute`, delays configurables (igual al patrón ya probado en `SincronizadorPqfLegacy`: 1m/5m/15m/1h/2h)                                             | No nativo — requiere TTL + Dead Letter Exchange configurado a mano, o una librería adicional (ej. MassTransit) encima                                                                                                       |
| Dashboard de monitoreo                       | Sí, incluido (`/hangfire`)                                                                                                                                                     | No nativo — requiere el plugin de management UI de RabbitMQ o tooling aparte                                                                                                                                                |
| Encaja con el patrón de trabajo de Canal ETLCobros  | Sí — "hay N registros pendientes en `SyncControl`, procesarlos" es naturalmente polling/batch                                                                                  | Requiere que algo más publique un evento "nuevo cobro a sincronizar" — hoy no hay un productor natural sin agregar lógica nueva a `MailbotWorker`                                                                           |
| Esfuerzo de implementación para este caso    | Bajo — es exactamente lo que el DIS-SOL ya describe (`SyncJobBase`, atributos de reintento)                                                                                    | Medio — ya existe una plantilla real y moderna que copiar (`proquifa-punchout-backend`, ver sección 5.1), pero igual falta agregar lo que hoy no existe: que algo publique el evento "nuevo cobro" (ver fila anterior)     |
| Ya probado y activo en este ecosistema       | **Sí** — `TaskSchedulerPqf` (BD real, esquema `HangFire`), jobs `Succeeded` hasta `2026-06-21`, app legacy `TaskSchedulerPqf.ppPedido.*`/`General.*` con recurring jobs reales | **Sí** — `Core.Pqf`/`ProquifaDotNet`, brokers reales configurados (hosts `172.24.32.x`), colas activas: `generadorDocumento` (PDF), `envioCorreo` (MailGun/SendInBlue), `PqfTimbrado` (CFDI), `embalar`, `transaccionesSTP` |
| Cercanía del precedente al dominio real del problema | **Muy cercana** — `SincronizadorPqfLegacy` ya resolvió, con Hangfire, el mismo problema (ETL hacia Legacy/PConnect, ~95-98% de éxito) que `LegacySync` ataca ahora, solo que para Cotizaciones en vez de Cobros | Lejana — los dos precedentes reales (`Core.Pqf`: documentos/correo; `proquifa-punchout-backend`: órdenes de un portal externo) resuelven problemas de otro dominio de negocio, no sincronización a Legacy |
| Dependencias operativas para un solo desarrollador | Una sola — reusa el mismo SQL Server donde ya vive `SyncControl`/`SyncJobLog`; nada nuevo que instalar, parchar o asegurar | Una adicional — un broker separado con su propio ciclo de vida y gestión de credenciales. Riesgo no hipotético: la implementación más reciente y mejor construida de RabbitMQ en este ecosistema (`proquifa-punchout-backend`, sección 5.1) tiene una contraseña real hardcodeada en el código fuente — evidencia de que ese mantenimiento extra no siempre se hace bien aquí |
| Coherencia con la arquitectura de "tres soluciones independientes" del DIS-SOL | Mantiene a `LegacySync` desacoplado — solo lee su propia tabla `SyncControl`, sin que `MailbotWorker` necesite saber que existe | Rompería ese desacoplamiento intencional — `MailbotWorker` tendría que publicar explícitamente hacia el consumidor de `LegacySync`, acoplando dos soluciones que el DIS-SOL diseñó para evolucionar por separado |
| Acoplamiento con lo ya escrito en el DIS-SOL | Ya es la elección vigente (línea 810) — sin costo de cambio                                                                                                                    | Cambiar implica rediseñar `SyncJobBase` y todo el flujo de reintentos de las secciones "Diseño de componentes" y "Flujo de sincronización" del DIS-SOL                                                                      |

---

## 5. Hallazgo relevante de esta sesión — RabbitMQ no es hipotético, pero tampoco encaja aquí

Verificado en el código real de `ProquifaDotNet`/`Core.Pqf` (no es un library reference sin usar): RabbitMQ está desplegado y activo, con hosts configurados y colas reales en uso (`generadorDocumento`, `envioCorreo`, `PqfTimbrado`, `embalar`, `transaccionesSTP`) — confirmado en `Core.Pqf/RabbitMQ/Abstract/AbstractRabbitMQWorker.cs` y las configuraciones de `_Worker.Servicios`, `_Worker.SendInBlue`, `WebApi.Catalogos`, `WebApi.Logistica`.

**Esto corrige, de paso, un dato impreciso en `TPSC-RE-FU-008-Contexto.md` (línea 113):** la Propuesta 1 del Mailbot (n8n + RabbitMQ) se descartó en su momento citando que *"PROQUIFA no usa n8n ni RabbitMQ actualmente"* — la parte de RabbitMQ era incorrecta, sí se usa, solo que para otra cosa (workers de documentos/correo, no para ingesta de Gmail). Esto **no cambia** la decisión de Mailbot (P2/Gmail Push+Pub/Sub sigue siendo superior por latencia y por reutilizar el GCP de P3, con o sin RabbitMQ ya existente) — se documenta aquí solo porque este DAR lo descubrió al verificar RabbitMQ a fondo, y vale la pena que quede corregido si alguien revisita esa tabla.

**Por qué, aun así, RabbitMQ no es la elección correcta para Canal ETLCobros de `LegacySync`:** el patrón real donde se usa RabbitMQ en este ecosistema es **productor en un proceso, consumidor en otro** (ej. `WebApi.Catalogos` publica en `generadorDocumento`, `_Worker.Servicios` lo consume) — comunicación entre procesos distintos. `LegacySync.Worker` no tiene ese problema: es un solo proceso que necesita revisar su propia tabla de control y reintentar su propio trabajo. Forzar RabbitMQ aquí significa construir desde cero el manejo de reintentos/DLQ que Hangfire ya da listo, sin ninguna ganancia real a cambio — y sin un productor natural que justifique el desacoplamiento entre procesos que RabbitMQ existe para resolver.

### 5.1 Segundo precedente, más sofisticado — `proquifa-punchout-backend`

Verificado contra el código real de `C:\Users\Ariel\Documents\proquifa-punchout-backend` (.NET reciente, Clean Architecture: API/Application/Domain/Infrastructure/Worker/Testing) — implementa RabbitMQ de forma mucho más completa que el patrón legacy de `Core.Pqf`. No es solo publicar/consumir: es un patrón productor/consumidor end-to-end con:

- **Publisher Confirms + Dead Letter Exchange/Queue con TTL** (72 horas) — `Infrastructure/RabbitMQ/RabbitMQClient.cs`.
- **Outbox pattern** (`OutboxEvent`) — registra cada intento de procesamiento con número de intento y mensaje de error, auditable.
- **Reintentos configurables vía tabla `AppSetting`** (`InboundOrderMaxAttempts`, default 5) — mismo espíritu que el `AppSettings` ya planeado para `LegacySync` en el DIS-SOL.
- **Idempotencia por estado** — antes de reprocesar revisa `order.Status`: si ya es `Processed`, hace ACK sin reprocesar; si es `Failed`, manda directo a Dead Letter sin reintentar. Es el mismo principio que LB-P4 ya definió para Canal ETLCobros (verificar contra destino antes de reinsertar).
- **Alerta crítica por correo (Brevo)** al alcanzar el máximo de intentos — mismo patrón que `BrevoNotificationService`, ya planeado en el DIS-SOL.

Esto es un ejemplo real, moderno y bien construido de "RabbitMQ con reintentos clasificados" — corrige la fila de "Esfuerzo de implementación" de la tabla de arriba: no habría que construirlo totalmente desde cero, hay una plantilla real que copiar.

**Lo que este hallazgo no cambia:** el patrón sigue siendo productor/consumidor entre procesos — la propia API de `proquifa-punchout-backend` publica a `po_queue` cuando llega una orden nueva, y su `Worker` reacciona a eso. Para que Canal ETLCobros siguiera este mismo patrón, `Proquifa.Pqf2.MailbotWorker.sln` tendría que publicar explícitamente un mensaje "nuevo cobro a sincronizar" cuando genera el pendiente — agregando una dependencia nueva entre dos soluciones que el DIS-SOL diseñó como independientes ("tres soluciones en paralelo"). Hangfire deja a `LegacySync` leyendo su propia tabla (`SyncControl`) sin que `MailbotWorker` necesite saber que `LegacySync` existe.

> **Observación de seguridad, fuera de alcance de este DAR:** `RabbitMQSettings.cs` en `proquifa-punchout-backend` tiene una contraseña real escrita como valor por defecto directamente en el código fuente (no en un archivo de configuración separado) — vale la pena reportarlo aparte; no se repite el valor literal en este documento.

---

## 6. Decisión confirmada

**Hangfire**, tal como ya especifica el DIS-SOL v2.6 — confirmado por Carlos el 2026-06-22 tras esta comparativa.

Justificación, en orden de peso:

1. **Encaje con el problema real (peso alto):** Canal ETLCobros es un patrón de "revisar pendientes y reintentar", no de "reaccionar a un evento publicado por otro proceso" — eso es exactamente lo que Hangfire resuelve nativo, y exactamente lo que RabbitMQ no resuelve sin trabajo adicional.
2. **Precedente en el mismo dominio del problema (peso alto):** no es solo que Hangfire ya esté probado en este ecosistema — está probado *para este mismo tipo de problema*. `SincronizadorPqfLegacy` ya resolvió, con Hangfire, ETL hacia Legacy con ~95-98% de éxito (ver `TPSC-RE-FU-008-Analisis-Reutilizacion-LegacySync.md`). Los dos precedentes reales de RabbitMQ (`Core.Pqf`, `proquifa-punchout-backend`) resuelven problemas de otro dominio — documentos/correo y órdenes de un portal externo, no sincronización a Legacy. El precedente más cercano al problema real pesa más que el precedente más sofisticado en abstracto.
3. **Menos piezas que operar y asegurar (peso alto):** Hangfire reutiliza el mismo SQL Server donde ya va a vivir `SyncControl`/`SyncJobLog` — nada nuevo que instalar, parchar o asegurar. RabbitMQ exige mantener un broker aparte, con su propio ciclo de vida y gestión de credenciales — algo que, en la práctica de este mismo equipo, no siempre se hace bien (la implementación más reciente y mejor construida de RabbitMQ que existe aquí, `proquifa-punchout-backend`, tiene una contraseña real hardcodeada en el código fuente). Para un proyecto sostenido por un solo desarrollador, cada pieza operativa adicional es un riesgo real, no teórico.
4. **Preserva el desacoplamiento que el propio DIS-SOL ya diseñó:** las "tres soluciones en paralelo" (`ProquifaDotNet`, `Proquifa.Pqf2.MailbotWorker.sln`, `LegacySync`) están pensadas para evolucionar por separado. Hangfire mantiene eso intacto — `LegacySync` solo lee su propia tabla. RabbitMQ rompería ese límite, obligando a `MailbotWorker` a publicar hacia un consumidor que hoy no necesita conocer.
5. **Cero costo de cambio:** es la elección ya escrita en el DIS-SOL — moverse a otra cosa significaría rediseñar `SyncJobBase` y el flujo de reintentos sin ninguna ganancia funcional a cambio.
6. **RabbitMQ queda descartado sin perder su lugar:** sigue siendo la herramienta correcta donde ya se usa (comunicación entre procesos distintos) — no hay ninguna recomendación de retirarlo de `ProquifaDotNet` ni de `proquifa-punchout-backend`, solo de no forzarlo dentro de `LegacySync.Worker`, donde no resuelve nada que Hangfire no resuelva ya mejor y con menos riesgo operativo.

---

## 7. Consecuencias

- Sin cambios al DIS-SOL más allá de la nota ya agregada en v2.6 — este DAR formaliza y documenta, con decisión confirmada, el porqué de una elección que ya estaba escrita, no introduce una nueva.
- `LB-T1` en adelante puede arrancar sobre Hangfire sin reabrir esta discusión, salvo que aparezca un escenario futuro genuinamente event-driven (ej. si algún día se quiere que `MailbotWorker` notifique a `LegacySync` en tiempo real en vez de que `LegacySync` haga polling sobre `SyncControl`) — en ese caso, RabbitMQ sería la herramienta a evaluar primero, ya que ya está desplegada, y `proquifa-punchout-backend` (sección 5.1) sería la plantilla concreta a seguir (Outbox + DLQ + reintentos configurables + alerta Brevo), no hay que diseñarla desde cero.
- Pendiente fuera de alcance de este DAR: corregir la imprecisión sobre RabbitMQ en `TPSC-RE-FU-008-Contexto.md` línea 113 (sección 5) — se deja anotada aquí, no se modifica ese archivo en esta sesión.
- Pendiente fuera de alcance de este DAR: reportar a quien corresponda la contraseña hardcodeada en `proquifa-punchout-backend/Infrastructure/RabbitMQ/RabbitMQSettings.cs` (sección 5.1) — observación de seguridad, no de arquitectura.

---

## Historial de Cambios

| Versión | Fecha | Descripción | Autor |
|---------|-------|--------------|-------|
| 1.0 | 22/06/2026 | Versión inicial — comparativa Hangfire vs RabbitMQ vs alternativas descartadas, verificada contra el código y la configuración real de `Core.Pqf`/`ProquifaDotNet` (RabbitMQ) y `TaskSchedulerPqf` (Hangfire). Recomendación: mantener Hangfire, sin cambios al DIS-SOL | Carlos Ivan Morales Carreon |
| 1.1 | 22/06/2026 | Agregado Kafka a opciones descartadas (sin precedente en el ecosistema). Agregada sección 5.1: segundo precedente de RabbitMQ, más sofisticado, verificado en `proquifa-punchout-backend` (Outbox + DLQ con TTL + reintentos configurables + alerta Brevo) — corrige a la baja el esfuerzo de implementación estimado para RabbitMQ, sin cambiar la recomendación (sigue faltando un productor natural del lado de `MailbotWorker`). Anotada observación de seguridad (contraseña hardcodeada), fuera de alcance de este DAR | Carlos Ivan Morales Carreon |
| 1.2 | 22/06/2026 | **Decisión confirmada por Carlos: Hangfire.** Reforzada la justificación con 3 criterios adicionales en la tabla comparativa y la sección 6: cercanía del precedente al dominio real del problema (ETL a Legacy, no documentos/correo/órdenes externas), menor cantidad de dependencias operativas para un solo desarrollador, y preservación del desacoplamiento entre las "tres soluciones en paralelo" del DIS-SOL. Sección 6 pasa de "Recomendación" a "Decisión confirmada" | Carlos Ivan Morales Carreon |
| 1.3 | 22/06/2026 | Renombradas todas las menciones de "Canal E1" → "Canal ETLCobros" (P19 de `Contexto.md`, decisión de Carlos) — el "E1" del cliente en `R16A-NO-FU-003`/RE-028 es un canal distinto. Sin cambio de contenido técnico | Carlos Ivan Morales Carreon |

---

*2026 RYNDEM SOFTWARE FACTORY. Todos los derechos reservados.*
