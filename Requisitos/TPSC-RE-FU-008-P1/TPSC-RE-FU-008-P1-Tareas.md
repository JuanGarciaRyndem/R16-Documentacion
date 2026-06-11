# Tareas — TPSC-RE-FU-008 Propuesta 1

> **Requisito:** TPSC-RE-FU-008 — Buzon de Cobros (Mailbot Rediseñado)
> **Propuesta:** P1 — Nueva solucion `MailbotWorker.sln` (.NET 10) + ajustes en ProquifaDotNet (Framework 4.8)
> **Analisis de impacto:** `TPSC-RE-FU-008-P1-Back.md`
> **Total de tareas:** 17

---

## Tarea 1

### [TPSC-RE-FU-008] [BD-OBJ-CH] Script de actualizacion de catalogos para Cobros

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Modulos:**
Mailbot — Clasificacion de correos recibidos

**Consideraciones previas:**
Este script cubre los cambios requeridos en catalogos existentes para habilitar la clasificacion de cobros en el Mailbot.
Existen dos opciones de implementacion (pendiente P2): Opcion A — renombrar el registro `Pago` a `Cobro` en `catClasificacionCorreoRecibido`; Opcion B — insertar una nueva clasificacion `Cobro` sin modificar `Pago`.
El script debe incluir ambas opciones comentadas y documentadas para que el TechLead o Product Owner seleccione la correcta antes de ejecutar.
Adicionalmente se requiere INSERT en `catProceso` para registrar el proceso de generacion de folio de cobro.
Este script debe ejecutarse **antes** que todas las demas tareas de esta propuesta.
Dependencia: ninguna.

**Objetivo general:**
Generar el script SQL que actualice `catClasificacionCorreoRecibido` y `catProceso` para soportar la clasificacion y proceso de cobros en el Mailbot.

**Objetivos especificos:**
- Preparar Opcion A: `UPDATE catClasificacionCorreoRecibido SET ClasificacionCorreoRecibido = 'Cobro', Clave = 'Cobro' WHERE Clave = 'Pago'`
- Preparar Opcion B: `INSERT INTO catClasificacionCorreoRecibido (Clave, ClasificacionCorreoRecibido, Activo) VALUES ('Cobro', 'Cobro', 1)`
- INSERT en `catProceso` para el proceso `GenerarFolioPagoCliente` si no existe
- Incluir consultas de verificacion post-ejecucion para cada cambio
- Documentar en el script cual opcion se selecciono y la fecha de ejecucion

**Resultado esperado:**
La tabla `catClasificacionCorreoRecibido` contiene el registro `Cobro` con su `Clave` correcta, y `catProceso` tiene el proceso de folio de cobro registrado, listos para ser consumidos por `ClasificacionCorreoRecibidoConstants` y `GeneradorProcesoMailBotBO`.

**Entregables:**
- Script SQL: `Scripts\R16\FU-008_CatalogosCobros.sql`

**Criterios de aceptacion:**
- [ ] El script incluye ambas opciones (A y B) documentadas y solo una activa
- [ ] La opcion seleccionada fue confirmada por Product Owner antes de ejecucion
- [ ] `catClasificacionCorreoRecibido` contiene registro con `Clave = 'Cobro'` tras ejecucion
- [ ] `catProceso` contiene el proceso de folio de cobro
- [ ] El script incluye consultas de verificacion post-ejecucion
- [ ] El script es idempotente (puede ejecutarse mas de una vez sin duplicar datos)
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- Pendiente P2 del `TPSC-RE-FU-008-P1-Back.md` debe resolverse antes de ejecutar este script
- Cambio relacionado con GAP-01 y GAP-02 del analisis de impacto
- La `Clave` del catalogo es la que consume `ClasificacionCorreoRecibidoConstants` — no el nombre display

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 2

### [TPSC-RE-FU-008] [BD-OBJ-M] Script de creacion de tablas de auditoria del Mailbot

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Modulos:**
Mailbot — Auditoria de eventos y clasificaciones

**Consideraciones previas:**
Este script crea las dos tablas nuevas requeridas por `MailbotWorker.sln`: `MailbotEventoCorreo` y `MailbotClasificacionLog`.
`MailbotEventoCorreo` registra cada correo procesado con su estado y numero de reintentos.
`MailbotClasificacionLog` registra cada respuesta del Agente IA con la confianza y tokens consumidos.
Ambas tablas tienen FK hacia `CorreoRecibido` y `RegionConfiguracionMailBot`.
Este script debe ejecutarse **antes** de las tareas de implementacion del Worker (.NET 10).
Dependencia: Tarea 1 completada.

**Objetivo general:**
Generar el script SQL que cree `MailbotEventoCorreo` y `MailbotClasificacionLog` con sus indices y restricciones de integridad referencial.

**Objetivos especificos:**
- CREATE TABLE `MailbotEventoCorreo`: `IdMailbotEvento` (PK), `IdCorreoRecibido` (FK), `IdRegion` (FK), `Estado`, `Intentos`, `FechaCreacion`, `FechaActualizacion`, `MensajeError`
- CREATE TABLE `MailbotClasificacionLog`: `IdClasificacionLog` (PK), `IdMailbotEvento` (FK), `ClasificacionResultado`, `Confianza`, `TokensUsados`, `ModeloIA`, `FechaClasificacion`, `PromptVersion`
- Crear indices en columnas de busqueda frecuente: `IdCorreoRecibido`, `Estado`, `IdRegion`, `FechaCreacion`
- Incluir consultas de verificacion de estructura (INFORMATION_SCHEMA) post-ejecucion
- Generar script de rollback (DROP TABLE con verificacion de existencia)

**Resultado esperado:**
Las tablas `MailbotEventoCorreo` y `MailbotClasificacionLog` existen en ProquifaDotNet listas para ser consumidas por el Worker .NET 10 y el scaffold de EF Core.

**Entregables:**
- Script SQL: `Scripts\R16\FU-008_TablaAuditoriaMailbot.sql`
- Script de rollback: `Scripts\R16\FU-008_TablaAuditoriaMailbot_Rollback.sql`

**Criterios de aceptacion:**
- [ ] `MailbotEventoCorreo` creada con PK, FKs e indices correctos
- [ ] `MailbotClasificacionLog` creada con PK, FK a `MailbotEventoCorreo` e indices correctos
- [ ] El script incluye verificacion de existencia previa antes de cada CREATE (idempotente)
- [ ] El script de rollback elimina las tablas en orden correcto (hija primero)
- [ ] Las consultas de verificacion post-ejecucion confirman estructura esperada
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- Definicion completa de columnas en `TPSC-RE-FU-008-P1_BD.md`
- El scaffold de EF Core (Tarea 6) leera estas tablas para generar las entidades

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1_BD.md`

---

## Tarea 3

### [TPSC-RE-FU-008] [IMP-EXIST-SERVICE] Refactorizar GeneradorProcesoMailBotBO y crear ClasificacionCorreoRecibidoConstants

**Aplicativos:**
ProquifaDotNet (Framework 4.8) — `Logic.Pqf.Logistica`

**Modulos:**
Mailbot — L11 Procesos — Clasificacion y enrutamiento de correos

**Consideraciones previas:**
Esta tarea corrige el **GAP-01** y crea el componente del **GAP-02** del analisis de impacto.
`GeneradorProcesoMailBotBO.cs` tiene el `case "Pago"` vacio — el correo queda marcado como procesado sin generar ningun pendiente, incumpliendo FU-008 Regla 1.
El switch evalua el nombre display de la clasificacion, lo cual es fragil ante renombrados en BD (Limitacion L3 y L4).
La refactorizacion debe cambiar el switch para evaluar la propiedad `Clave` del catalogo, usando `ClasificacionCorreoRecibidoConstants`.
Dependencia: Tarea 1 completada (la `Clave` debe existir en BD antes de probar).

**Objetivo general:**
Refactorizar `GeneradorProcesoMailBotBO.cs` para evaluar `Clave` en lugar del nombre display, crear `ClasificacionCorreoRecibidoConstants.cs` con las claves oficiales, e implementar el `case` de Cobro invocando `CorreoRecibidoClienteToPagoBO`.

**Objetivos especificos:**
- Crear `Logic.Pqf.Logistica\L11.MailBot\Constants\ClasificacionCorreoRecibidoConstants.cs` con constantes: `Requisicion`, `Pedido`, `Cobro`, `Otros`
- Modificar `GeneradorProcesoMailBotBO.cs`: cambiar `switch(catClasificacion.ClasificacionCorreoRecibido)` por `switch(catClasificacion.Clave)`
- Reemplazar literales de string en `case` por las constantes de `ClasificacionCorreoRecibidoConstants`
- Implementar `case ClasificacionCorreoRecibidoConstants.Cobro` invocando `CorreoRecibidoClienteToPagoBO.Ejecutar()`
- Verificar que `CorreoRecibidoClienteToPagoBO` genera `fccFolioPagoCliente` correctamente
- Asegurar que `Logic.Pqf.Logistica` compila sin errores

**Resultado esperado:**
`GeneradorProcesoMailBotBO` enruta correctamente los correos clasificados como Cobro hacia `CorreoRecibidoClienteToPagoBO`, generando el folio `fccFolioPagoCliente`. El switch es robusto ante renombrados de catalogo al usar `Clave`.

**Entregables:**
- Archivo nuevo: `Logic.Pqf.Logistica\L11.MailBot\Constants\ClasificacionCorreoRecibidoConstants.cs`
- Archivo modificado: `Logic.Pqf.Logistica\L11.MailBot\Procesos\GeneradorProcesoMailBotBO.cs`

**Criterios de aceptacion:**
- [ ] `ClasificacionCorreoRecibidoConstants` define constantes para `Requisicion`, `Pedido`, `Cobro` y `Otros`
- [ ] El `switch` evalua `catClasificacion.Clave` (no el nombre display)
- [ ] `case ClasificacionCorreoRecibidoConstants.Cobro` invoca `CorreoRecibidoClienteToPagoBO` y genera `fccFolioPagoCliente`
- [ ] Los `case` de Requisicion y Pedido siguen funcionando correctamente tras la refactorizacion
- [ ] No existen literales de string de clasificacion fuera de `ClasificacionCorreoRecibidoConstants`
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores (Framework 4.8)
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- GAP-01 y GAP-02 del archivo `TPSC-RE-FU-008-P1-Back.md`
- Limitaciones L3 y L4 corregidas por esta tarea
- El `case "Pago"` vacio es el bug principal que impide cumplir FU-008 Regla 1

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`
- Archivo objetivo: `Logic.Pqf.Logistica\L11.MailBot\Procesos\GeneradorProcesoMailBotBO.cs`

---

## Tarea 4

### [TPSC-RE-FU-008] [SERV-COMPLEX-TRANSACT] Crear CorreoRecibidoClienteBuzonCobrosBO

**Aplicativos:**
ProquifaDotNet (Framework 4.8) — `Logic.Pqf.Logistica`

**Modulos:**
Mailbot — L11 Cobros — Buzon de correos de cobro

**Consideraciones previas:**
Esta tarea cubre el **GAP-03** del analisis de impacto.
El BO debe devolver el listado paginado de correos clasificados como cobro, filtrado por cobrador e `IdRegion` del usuario autenticado.
La region y el cobrador se obtienen exclusivamente del contexto de autenticacion — **no se aceptan como parametros externos**.
El filtro de region debe aplicarse usando `AsegurarFiltroRegion(ResumeGroupQueryInfo, IdRegion)`.
Los correos MEX no deben aparecer en resultados para usuarios PER y viceversa.
Dependencia: Tareas 1 y 2 completadas.

**Objetivo general:**
Implementar `CorreoRecibidoClienteBuzonCobrosBO.cs` que retorne el listado paginado de correos de cobro para el cobrador e `IdRegion` del usuario autenticado.

**Objetivos especificos:**
- Crear `Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\CorreoRecibidoClienteBuzonCobrosBO.cs`
- Recibir `ResumeGroupQueryInfo` (paginacion y filtros) como parametro de entrada
- Extraer `IdRegion` e `IdUsuario` (cobrador) del contexto del BO — nunca de parametros directos
- Aplicar filtro de region obligatorio con `AsegurarFiltroRegion` antes de ejecutar la consulta
- Filtrar por `ClasificacionCorreoRecibido.Clave = 'Cobro'` usando join con el catalogo
- Retornar `PagedList<CorreoRecibidoClienteBuzonCobrosDTO>` con los campos requeridos
- Implementar el mapeo entre entidades de BD y el DTO de respuesta

**Resultado esperado:**
`CorreoRecibidoClienteBuzonCobrosBO` retorna el listado paginado de correos de cobro filtrados por region y cobrador, listo para ser consumido por `BuzonCobrosController`.

**Entregables:**
- Archivo nuevo: `Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\CorreoRecibidoClienteBuzonCobrosBO.cs`
- Archivo nuevo (DTO): `Logic.Pqf.Logistica\L11.MailBot\DTOs\CorreoRecibidoClienteBuzonCobrosDTO.cs`

**Criterios de aceptacion:**
- [ ] El BO aplica filtro de region obligatorio antes de ejecutar la consulta
- [ ] El `IdRegion` se obtiene del contexto del BO — no de parametros de entrada externos
- [ ] Los correos MEX no aparecen en resultados para usuarios PER y viceversa
- [ ] El listado esta paginado usando `ResumeGroupQueryInfo`
- [ ] El BO solo retorna correos con clasificacion `Clave = 'Cobro'`
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores (Framework 4.8)
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- GAP-03 del archivo `TPSC-RE-FU-008-P1-Back.md`
- La segregacion por region es un criterio critico: "Correos MEX no aparecen en buzon PER y viceversa"

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`
- Revision: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008 Revision.md`

---

## Tarea 5

### [TPSC-RE-FU-008] [LIST-PAG-MULT-FILTER] Crear BuzonCobrosController en WebApi.Logistica

**Aplicativos:**
ProquifaDotNet (Framework 4.8) — `WebApi.Logistica`

**Modulos:**
Mailbot — API — Buzon de Cobros

**Consideraciones previas:**
Esta tarea cubre el **GAP-04** del analisis de impacto.
El controller expone el endpoint `POST /BuzonCobros` que consume `CorreoRecibidoClienteBuzonCobrosBO`.
La region y el cobrador se extraen del token mediante `ObtenerUsuarioAutenticado()` — **no se aceptan como parametros en el body o query string**.
El controller debe seguir el patron `BaseApiController` con `TryExecute()` para manejo de excepciones.
Dependencia: Tarea 4 completada.

**Objetivo general:**
Crear `BuzonCobrosController.cs` en `WebApi.Logistica` que exponga `POST /BuzonCobros` con paginacion y multiples filtros, consumiendo `CorreoRecibidoClienteBuzonCobrosBO` con la region del usuario autenticado.

**Objetivos especificos:**
- Crear `WebApi.Logistica\Controllers\Mailbot\BuzonCobrosController.cs` heredando de `BaseApiController`
- Implementar accion `POST /BuzonCobros` que recibe `ResumeGroupQueryInfo` en el body
- Obtener `Usuario` autenticado con `ObtenerUsuarioAutenticado()` y pasar `IdRegion` al BO
- Invocar `CorreoRecibidoClienteBuzonCobrosBO` y retornar el resultado paginado
- Envolver la logica en `TryExecute()` para manejo uniforme de excepciones
- Aplicar atributo de autorizacion correspondiente al rol de cobrador

**Resultado esperado:**
El endpoint `POST /BuzonCobros` esta disponible en `WebApi.Logistica`, retorna el listado paginado de correos de cobro filtrado por la region e identidad del usuario del token.

**Entregables:**
- Archivo nuevo: `WebApi.Logistica\Controllers\Mailbot\BuzonCobrosController.cs`

**Criterios de aceptacion:**
- [ ] `POST /BuzonCobros` retorna `200 OK` con listado paginado en estructura `ResumeGroupResult`
- [ ] La region y el cobrador se extraen del token mediante `ObtenerUsuarioAutenticado()` — no de parametros externos
- [ ] El controller usa `TryExecute()` para manejo de excepciones
- [ ] El endpoint requiere autenticacion (rol de cobrador)
- [ ] El proyecto `WebApi.Logistica` compila sin errores (Framework 4.8)
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- GAP-04 del archivo `TPSC-RE-FU-008-P1-Back.md`
- Patron de referencia: `BaseApiController.ObtenerUsuarioAutenticado()` en `Core.WebApi\ControllerOperations\BaseApiController.cs`

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 6

### [TPSC-RE-FU-008] [ARQ-PROJ-NET] Crear solucion MailbotWorker.sln (.NET 10) — estructura base y scaffold de BD

**Aplicativos:**
MailbotWorker (nueva solucion .NET 10)

**Modulos:**
Mailbot — Arquitectura base — Todos los modulos del Worker

**Consideraciones previas:**
Esta tarea establece la estructura completa de la nueva solucion `MailbotWorker.sln` con Clean Architecture en .NET 10.
La solucion contiene los proyectos: `Mailbot.Domain`, `Mailbot.Application`, `Mailbot.Infrastructure`, `Mailbot.Worker`, `Mailbot.Api`.
El scaffold de EF Core debe generarse apuntando a la BD ProquifaDotNet (RYNL010).
Las interfaces de servicios de terceros (`IClasificadorAgente`, `IArchivoService`, `INotificacionService`) se definen en `Mailbot.Application` — las implementaciones van en `Mailbot.Infrastructure`.
Esta tarea es la base estructural: las Tareas 7-12 dependen de ella.
Dependencias: Tareas 1 y 2 completadas (las tablas nuevas deben existir para el scaffold).

**Objetivo general:**
Crear la solucion `MailbotWorker.sln` en .NET 10 con arquitectura limpia, definir las interfaces de abstraccion de servicios, realizar el scaffold de EF Core y configurar la inyeccion de dependencias base.

**Objetivos especificos:**
- Crear solucion `MailbotWorker.sln` con proyectos: `Mailbot.Domain`, `Mailbot.Application`, `Mailbot.Infrastructure`, `Mailbot.Worker`, `Mailbot.Api`
- Definir referencias entre proyectos siguiendo Clean Architecture (Domain <- Application <- Infrastructure <- Worker/Api)
- Generar scaffold de EF Core: `Mailbot.Infrastructure\Persistence\ProquifaDbContext.cs` apuntando a ProquifaDotNet
- Incluir en scaffold las entidades: `CorreoRecibido`, `CorreoRecibidoCliente`, `RegionConfiguracionMailBot`, `catClasificacionCorreoRecibido`, `fccFolioPagoCliente`, `MailbotEventoCorreo`, `MailbotClasificacionLog`
- Crear interfaces en `Mailbot.Application`: `IClasificadorAgente`, `IArchivoService`, `INotificacionService`, `IGmailService`
- Configurar `appsettings.json` con secciones: `ConnectionStrings`, `RabbitMQ`, `OpenAI`, `MinIO`, `Brevo`, `Gmail`
- Configurar `Program.cs` del Worker con registro de DI base
- Crear `Mailbot.Api\Endpoints\HealthEndpoints.cs` con `GET /health`

**Resultado esperado:**
La solucion `MailbotWorker.sln` compila en .NET 10 sin errores, con arquitectura limpia establecida, scaffold generado e interfaces de abstraccion definidas, lista para que las Tareas 7-12 implementen cada componente.

**Entregables:**
- Solucion nueva: `MailbotWorker.sln`
- Proyecto `Mailbot.Domain`
- Proyecto `Mailbot.Application` (con interfaces `IClasificadorAgente`, `IArchivoService`, `INotificacionService`, `IGmailService`)
- Proyecto `Mailbot.Infrastructure` (con `ProquifaDbContext.cs` scaffold)
- Proyecto `Mailbot.Worker` (con `Program.cs` y `appsettings.json`)
- Proyecto `Mailbot.Api` (con `HealthEndpoints.cs`)

**Criterios de aceptacion:**
- [ ] La solucion `MailbotWorker.sln` compila en .NET 10 sin errores
- [ ] La arquitectura limpia esta implementada (referencias unidireccionales entre proyectos)
- [ ] El scaffold de EF Core incluye todas las entidades requeridas
- [ ] `GET /health` retorna `200 OK`
- [ ] Las interfaces `IClasificadorAgente`, `IArchivoService`, `INotificacionService`, `IGmailService` estan definidas en `Mailbot.Application`
- [ ] `appsettings.json` tiene las secciones de configuracion para todos los servicios de terceros
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- Secciones 2 y 7 del archivo `TPSC-RE-FU-008-P1-Back.md`
- Pendientes P1, P4, P5, P6 deben estar resueltos antes de configurar `appsettings.json` con valores reales
- El scaffold debe excluir tablas que no pertenecen al dominio del Mailbot

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1_BD.md`

---

## Tarea 7

### [TPSC-RE-FU-008] [IMPL-THIRD-SERV] Integrar n8n + RabbitMQ — configuracion del orquestador y consumers del Worker

**Aplicativos:**
MailbotWorker (nueva solucion .NET 10) — Infraestructura de mensajeria

**Modulos:**
Mailbot — Orquestacion de eventos — Queues MEX y PER

**Consideraciones previas:**
n8n se activa mediante un Gmail Trigger (polling cada 1 min) y publica al exchange `mailbot.correos` en RabbitMQ.
El exchange enruta a dos queues: `mailbot.mex` y `mailbot.per`, cada una con su Dead Letter Queue (DLQ).
`Mailbot.Infrastructure\RabbitMQ\RabbitMQConsumer.cs` implementa el consumer que lee de las queues.
Los Workers `CorreoWorkerMex` y `CorreoWorkerPer` (Tarea 12) consumen este `RabbitMQConsumer`.
Pendiente P4 (licencia n8n: self-hosted vs cloud) debe resolverse antes de configurar el ambiente.
Pendiente P5 (cuentas Gmail MEX/PER) debe resolverse antes de configurar el Gmail Trigger en n8n.
Dependencia: Tarea 6 completada.

**Objetivo general:**
Implementar `RabbitMQConsumer.cs` en `Mailbot.Infrastructure`, configurar el exchange y queues en RabbitMQ, y documentar la configuracion del workflow de n8n para el Gmail Trigger.

**Objetivos especificos:**
- Implementar `Mailbot.Infrastructure\RabbitMQ\RabbitMQConsumer.cs` con conexion, canal, declaracion de exchange, queues y DLQ
- Implementar metodo `ConsumeAsync(string queue, Func<CorreoEventoMessage, Task> handler, CancellationToken ct)`
- Configurar retry policy para mensajes fallidos (maximo 3 intentos antes de enviar a DLQ)
- Serializar/deserializar mensajes usando `System.Text.Json`
- Registrar `RabbitMQConsumer` en DI del Worker (`Program.cs`)
- Documentar el workflow de n8n: Gmail Trigger -> publicacion al exchange RabbitMQ
- Agregar health check de RabbitMQ al endpoint `GET /health`

**Resultado esperado:**
`RabbitMQConsumer` puede conectarse a RabbitMQ, declarar el exchange `mailbot.correos` con queues `mailbot.mex` y `mailbot.per` y sus DLQs, y consumir mensajes de forma asincrona.

**Entregables:**
- Archivo nuevo: `Mailbot.Infrastructure\RabbitMQ\RabbitMQConsumer.cs`
- Archivo nuevo (mensaje): `Mailbot.Application\Models\CorreoEventoMessage.cs`
- Documento de configuracion n8n (workflow JSON exportado o markdown)

**Criterios de aceptacion:**
- [ ] `RabbitMQConsumer` conecta a RabbitMQ con las credenciales de `appsettings.json`
- [ ] El exchange `mailbot.correos` y las queues `mailbot.mex`, `mailbot.per` y DLQs se declaran correctamente
- [ ] Los mensajes fallidos (3 intentos) se enrutan a la DLQ correspondiente
- [ ] El health check de RabbitMQ reporta estado en `GET /health`
- [ ] La solucion `MailbotWorker.sln` compila en .NET 10 sin errores tras esta tarea
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- Seccion 2.2 del archivo `TPSC-RE-FU-008-P1-Back.md`
- Pendiente P4: confirmar licencia n8n antes de configurar ambiente
- Pendiente P5: confirmar cuentas Gmail MEX/PER antes de configurar Gmail Trigger

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`

---

## Tarea 8

### [TPSC-RE-FU-008] [ALG-COMPLX-LOGIC] Implementar Agente IA de clasificacion de correos

**Aplicativos:**
MailbotWorker (nueva solucion .NET 10) — `Mailbot.Infrastructure`

**Modulos:**
Mailbot — Clasificacion inteligente — Agente IA

**Consideraciones previas:**
Esta tarea implementa el Agente IA que reemplaza el modelo ML.NET estatico del Mailbot actual.
El agente lee asunto, cuerpo y adjuntos del correo para clasificarlo en: Cotizacion, Pedido, Cobro, Otros.
La implementacion es `OpenAIClasificadorAgente.cs` que implementa la interfaz `IClasificadorAgente` definida en Tarea 6.
Los prompts se almacenan en archivos `.txt` versionados para facilitar ajustes sin recompilar.
Cada clasificacion debe registrarse en `MailbotClasificacionLog` con confianza, tokens y version de prompt.
Pendiente P1 (proveedor IA: OpenAI vs Azure OpenAI) debe resolverse antes de implementar.
Pendiente P3 (correos reales de PROQUIFA para ajuste de prompts) debe resolverse antes de validar.
Dependencia: Tarea 6 completada.

**Objetivo general:**
Implementar `OpenAIClasificadorAgente.cs` que clasifica correos usando el Agente IA, con prompts versionados en archivos `.txt`, registrando cada clasificacion en `MailbotClasificacionLog`.

**Objetivos especificos:**
- Implementar `Mailbot.Infrastructure\IA\OpenAIClasificadorAgente.cs` implementando `IClasificadorAgente`
- Implementar `PromptBuilder.cs` que carga y compone el prompt desde archivos `.txt` con asunto, cuerpo y texto de adjuntos
- Crear prompt inicial: `Mailbot.Infrastructure\IA\Prompts\clasificacion_v1.txt`
- Parsear la respuesta del Agente IA para extraer: clasificacion, confianza y justificacion
- Registrar en `MailbotClasificacionLog`: clasificacion, confianza, tokens usados, modelo, version de prompt
- Implementar manejo de errores: timeout, rate limit, respuesta invalida -> excepcion tipificada
- Registrar `OpenAIClasificadorAgente` en DI del Worker

**Resultado esperado:**
El Agente IA clasifica correctamente correos de cobro, cotizacion y pedido usando asunto, cuerpo y adjuntos, y registra cada clasificacion con su metadata en `MailbotClasificacionLog`.

**Entregables:**
- Archivo nuevo: `Mailbot.Infrastructure\IA\OpenAIClasificadorAgente.cs`
- Archivo nuevo: `Mailbot.Infrastructure\IA\PromptBuilder.cs`
- Prompt inicial: `Mailbot.Infrastructure\IA\Prompts\clasificacion_v1.txt`

**Criterios de aceptacion:**
- [ ] El Agente IA clasifica correctamente correos de Cotizacion, Pedido, Cobro y Otros
- [ ] El agente lee adjuntos (PDF/imagen) ademas de asunto y cuerpo
- [ ] Cada clasificacion queda registrada en `MailbotClasificacionLog` con confianza, tokens y version de prompt
- [ ] Los prompts se cargan desde archivos `.txt` — no estan hardcodeados en el codigo
- [ ] Los errores (timeout, rate limit) lanzan excepciones tipificadas capturables por el Worker
- [ ] La solucion `MailbotWorker.sln` compila en .NET 10 sin errores tras esta tarea
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- Seccion 3 del archivo `TPSC-RE-FU-008-P1-Back.md`
- Pendiente P1: definir proveedor IA antes de implementar
- Pendiente P3: confirmar correos reales para ajuste de prompts

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 9

### [TPSC-RE-FU-008] [SERV-COMPLEX-TRANSACT] Implementar ProcesarCorreoUseCase y GenerarPendienteUseCase

**Aplicativos:**
MailbotWorker (nueva solucion .NET 10) — `Mailbot.Application`

**Modulos:**
Mailbot — Casos de uso — Procesamiento y generacion de pendientes

**Consideraciones previas:**
`ProcesarCorreoUseCase` orquesta el flujo completo: leer correo -> subir adjuntos a MinIO -> clasificar con Agente IA -> enrutar al caso de uso -> registrar evento en `MailbotEventoCorreo`.
`GenerarPendienteUseCase` genera el pendiente especifico segun la clasificacion: para Cobro invoca la logica equivalente a `CorreoRecibidoClienteToPagoBO` (Framework 4.8) adaptada a .NET 10.
Ambos casos de uso coordinan `IArchivoService`, `IClasificadorAgente`, `IGmailService` y `ProquifaDbContext` mediante inyeccion de dependencias.
Dependencias: Tareas 6, 7 y 8 completadas.

**Objetivo general:**
Implementar `ProcesarCorreoUseCase.cs` y `GenerarPendienteUseCase.cs` en `Mailbot.Application` que orquesten el flujo completo desde la recepcion del evento hasta la persistencia del pendiente en BD.

**Objetivos especificos:**
- Implementar `Mailbot.Application\UseCases\ProcesarCorreoUseCase.cs`:
  - Recibir `CorreoEventoMessage` (mensaje de RabbitMQ)
  - Leer correo completo via `IGmailService`
  - Subir adjuntos a MinIO via `IArchivoService`
  - Clasificar con `IClasificadorAgente`
  - Invocar `GenerarPendienteUseCase` con la clasificacion resultante
  - Registrar en `MailbotEventoCorreo` con estado y resultado
- Implementar `Mailbot.Application\UseCases\GenerarPendienteUseCase.cs`:
  - `case Cobro` -> INSERT en `fccFolioPagoCliente` + INSERT en `CorreoRecibidoCliente`
  - `case Cotizacion` -> flujo de pretramitacion de cotizacion
  - `case Pedido` -> flujo de pretramitacion de pedido
  - `case Otros` -> marcar como procesado sin pendiente
- Manejar transaccion: si cualquier paso falla, revertir y actualizar `MailbotEventoCorreo` con error e incrementar `Intentos`

**Resultado esperado:**
El flujo completo de procesamiento de un correo de cobro funciona de extremo a extremo: correo recibido -> adjuntos en MinIO -> clasificado como Cobro -> `fccFolioPagoCliente` generado -> evento registrado en `MailbotEventoCorreo`.

**Entregables:**
- Archivo nuevo: `Mailbot.Application\UseCases\ProcesarCorreoUseCase.cs`
- Archivo nuevo: `Mailbot.Application\UseCases\GenerarPendienteUseCase.cs`

**Criterios de aceptacion:**
- [ ] El flujo completo (RabbitMQ -> Gmail -> MinIO -> IA -> BD) funciona sin errores para correos de cobro
- [ ] `fccFolioPagoCliente` se genera correctamente para correos clasificados como Cobro
- [ ] `MailbotEventoCorreo` registra el estado final (exito o error) con numero de intentos
- [ ] Si el procesamiento falla, la transaccion se revierte y el evento queda en estado de error
- [ ] Los correos clasificados como Otros quedan marcados como procesados sin generar pendiente
- [ ] La solucion `MailbotWorker.sln` compila en .NET 10 sin errores tras esta tarea
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- Secciones 2 y 3 del archivo `TPSC-RE-FU-008-P1-Back.md`
- La logica de generacion de folio debe ser equivalente a la del Framework 4.8 (`CorreoRecibidoClienteToPagoBO`)

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`
- Referencia Framework 4.8: `Logic.Pqf.Logistica\L11.MailBot\Procesos\Pagos\CorreoRecibidoClienteToPagoBO.cs`

---

## Tarea 10

### [TPSC-RE-FU-008] [IMPL-THIRD-SERV] Integrar MinIO — almacenamiento de adjuntos del Mailbot

**Aplicativos:**
MailbotWorker (nueva solucion .NET 10) — `Mailbot.Infrastructure`

**Modulos:**
Mailbot — Almacenamiento — Adjuntos de correos de cobro

**Consideraciones previas:**
Esta tarea implementa `MinioArchivoService.cs` que implementa la interfaz `IArchivoService` definida en Tarea 6.
Los adjuntos de cada correo (PDF, imagenes) se suben al bucket `mailbot` en MinIO antes de persistir en BD.
La URL del objeto en MinIO se guarda en `ArchivoCorreoRecibido.UrlArchivo`.
Pendiente P6 (definir bucket MinIO y politica de retencion) debe resolverse antes de esta tarea.
Dependencia: Tarea 6 completada.

**Objetivo general:**
Implementar `MinioArchivoService.cs` en `Mailbot.Infrastructure` que suba adjuntos del correo al bucket `mailbot` en MinIO y retorne las URLs de los objetos almacenados.

**Objetivos especificos:**
- Implementar `Mailbot.Infrastructure\MinIO\MinioArchivoService.cs` implementando `IArchivoService`
- Conectar a MinIO usando las credenciales de `appsettings.json` (seccion `MinIO`)
- Implementar `SubirArchivoAsync(stream, nombreArchivo, contentType)` -> retorna URL del objeto
- Organizar objetos con path: `mailbot/{IdRegion}/{FechaCorreo:yyyyMMdd}/{IdCorreoRecibido}/{nombreArchivo}`
- Implementar manejo de errores: fallo en subida -> excepcion tipificada
- Registrar `MinioArchivoService` en DI del Worker
- Agregar health check de MinIO al endpoint `GET /health`

**Resultado esperado:**
Los adjuntos de los correos de cobro se suben correctamente a MinIO y sus URLs quedan disponibles para persistirse en `ArchivoCorreoRecibido.UrlArchivo` en la BD.

**Entregables:**
- Archivo nuevo: `Mailbot.Infrastructure\MinIO\MinioArchivoService.cs`

**Criterios de aceptacion:**
- [ ] `MinioArchivoService` sube archivos PDF e imagenes al bucket `mailbot`
- [ ] La URL retornada es accesible y apunta al objeto correcto en MinIO
- [ ] Los objetos se organizan con el path `mailbot/{IdRegion}/{Fecha}/{IdCorreoRecibido}/{archivo}`
- [ ] El health check de MinIO reporta estado en `GET /health`
- [ ] Los errores de conexion o subida lanzan excepciones tipificadas
- [ ] La solucion `MailbotWorker.sln` compila en .NET 10 sin errores tras esta tarea
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- Seccion 5 del archivo `TPSC-RE-FU-008-P1-Back.md`
- Pendiente P6: confirmar bucket y politica de retencion antes de implementar

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`

---

## Tarea 11

### [TPSC-RE-FU-008] [ATTACHED-EMAIL] Integrar Brevo — notificaciones de error del Mailbot

**Aplicativos:**
MailbotWorker (nueva solucion .NET 10) — `Mailbot.Infrastructure`

**Modulos:**
Mailbot — Notificaciones — Alertas de error al equipo de operaciones

**Consideraciones previas:**
Esta tarea implementa `BrevoNotificacionService.cs` que implementa la interfaz `INotificacionService` definida en Tarea 6.
Brevo envia alertas al equipo de operaciones cuando el Worker encuentra un error critico o un correo llega a la DLQ tras agotar los reintentos.
El email de alerta debe incluir: tipo de error, `IdCorreoRecibido`, region, intentos realizados y mensaje de error resumido.
Dependencia: Tarea 6 completada.

**Objetivo general:**
Implementar `BrevoNotificacionService.cs` en `Mailbot.Infrastructure` que envie emails de alerta al equipo de operaciones ante errores criticos del Worker, usando la API transaccional de Brevo.

**Objetivos especificos:**
- Implementar `Mailbot.Infrastructure\Brevo\BrevoNotificacionService.cs` implementando `INotificacionService`
- Conectar a la API transaccional de Brevo usando la API key de `appsettings.json` (seccion `Brevo`)
- Implementar `EnviarAlertaErrorAsync(ErrorMailbotDTO error)` con template configurable
- El email debe incluir: tipo de error, `IdCorreoRecibido`, `IdRegion`, `Intentos`, mensaje de error y timestamp
- Implementar throttling basico: no enviar mas de N alertas por hora para el mismo tipo de error
- Registrar `BrevoNotificacionService` en DI del Worker
- Implementar modo `DryRun` configurable para ambientes de desarrollo (log sin enviar email real)

**Resultado esperado:**
Ante un error critico del Worker, el equipo de operaciones recibe un email de alerta con el detalle via Brevo, sin saturacion por alertas repetidas.

**Entregables:**
- Archivo nuevo: `Mailbot.Infrastructure\Brevo\BrevoNotificacionService.cs`
- Archivo nuevo (DTO): `Mailbot.Application\Models\ErrorMailbotDTO.cs`

**Criterios de aceptacion:**
- [ ] El email de alerta llega al equipo de operaciones ante un error critico del Worker
- [ ] El email incluye: tipo de error, `IdCorreoRecibido`, region, intentos y mensaje de error
- [ ] El throttling basico evita spam de alertas repetidas del mismo tipo
- [ ] El modo `DryRun` loguea el email sin enviarlo (para ambiente DEV)
- [ ] La solucion `MailbotWorker.sln` compila en .NET 10 sin errores tras esta tarea
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- Seccion 6 del archivo `TPSC-RE-FU-008-P1-Back.md`
- Limitacion L6 del Mailbot actual corregida por esta tarea

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`

---

## Tarea 12

### [TPSC-RE-FU-008] [AUTOMATIC-JOB] Configurar CorreoWorkerMex y CorreoWorkerPer como IHostedService

**Aplicativos:**
MailbotWorker (nueva solucion .NET 10) — `Mailbot.Worker`

**Modulos:**
Mailbot — Workers — Procesamiento paralelo por region MEX y PER

**Consideraciones previas:**
Esta tarea implementa los dos Workers (`CorreoWorkerMex` y `CorreoWorkerPer`) que heredan de `BackgroundService` y consumen las queues de RabbitMQ de forma independiente por region.
Cada Worker invoca `ProcesarCorreoUseCase` al recibir un mensaje de su queue.
Los errores no manejados invocan `INotificacionService` para alertar al equipo de operaciones.
Esta es la tarea integradora final — requiere todas las tareas anteriores completadas (3-11).
Dependencias: Tareas 7, 9, 10 y 11 completadas.

**Objetivo general:**
Implementar `CorreoWorkerMex.cs` y `CorreoWorkerPer.cs` como `BackgroundService` que consuman sus queues de RabbitMQ de forma independiente e invoquen `ProcesarCorreoUseCase` para cada correo, con manejo de errores y alertas via Brevo.

**Objetivos especificos:**
- Implementar `Mailbot.Worker\Workers\CorreoWorkerMex.cs` heredando de `BackgroundService`:
  - Consumir queue `mailbot.mex` via `RabbitMQConsumer`
  - Por cada mensaje: invocar `ProcesarCorreoUseCase` con `IdRegion = MEX`
  - Ante excepcion no manejada: invocar `INotificacionService.EnviarAlertaErrorAsync()`
  - Respetar `CancellationToken` para graceful shutdown
- Implementar `Mailbot.Worker\Workers\CorreoWorkerPer.cs` con la misma estructura para queue `mailbot.per` y `IdRegion = PER`
- Registrar ambos Workers en `Program.cs` con `AddHostedService<CorreoWorkerMex>()` y `AddHostedService<CorreoWorkerPer>()`
- Configurar graceful shutdown timeout en `appsettings.json`
- Validar que los Workers procesan sus queues de forma paralela e independiente

**Resultado esperado:**
`CorreoWorkerMex` y `CorreoWorkerPer` procesan correos de sus respectivas regiones en paralelo, invocando el flujo completo (RabbitMQ -> Gmail -> MinIO -> IA -> BD -> Brevo en caso de error), y la solucion `MailbotWorker.sln` compila y ejecuta sin errores en .NET 10.

**Entregables:**
- Archivo nuevo: `Mailbot.Worker\Workers\CorreoWorkerMex.cs`
- Archivo nuevo: `Mailbot.Worker\Workers\CorreoWorkerPer.cs`
- `Mailbot.Worker\Program.cs` actualizado con registro de ambos Workers

**Criterios de aceptacion:**
- [ ] `CorreoWorkerMex` consume queue `mailbot.mex` y procesa correos de region MEX
- [ ] `CorreoWorkerPer` consume queue `mailbot.per` y procesa correos de region PER
- [ ] Ambos Workers procesan en paralelo sin bloquearse mutuamente
- [ ] Correos MEX no aparecen procesados en el contexto de PER y viceversa (segregacion por region)
- [ ] Los errores no manejados envian alerta via Brevo
- [ ] El graceful shutdown respeta el `CancellationToken` sin perder mensajes en proceso
- [ ] La solucion `MailbotWorker.sln` compila en .NET 10 sin errores
- [ ] `ProquifaDotNet` (Framework 4.8) compila sin errores tras los cambios de las tareas anteriores
- [ ] PR aprobado por lider tecnico

**Mas informacion de la tarea:**
- Secciones 2 y 7 del archivo `TPSC-RE-FU-008-P1-Back.md`
- Esta tarea valida la solucion completa de extremo a extremo

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`
- Revision: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008 Revision.md`

---

---

## Tarea 13

### [TPSC-RE-FU-008] [SERV-COMPLEX-TRANSACT] Crear CorreoRecibidoClienteBuzonCotizacionesBO

**Aplicativos:**
ProquifaDotNet (Framework 4.8) — `Logic.Pqf.Logistica`

**Módulos:**
Mailbot — L11 Cotizaciones — Buzón de correos de cotización

**Consideraciones previas:**
Esta tarea es el equivalente de la Tarea 4 para el buzón de Cotizaciones.
El BO devuelve el listado paginado de correos clasificados como Requisición/Cotización, filtrado por el agente comercial (gestor) e `IdRegion` del usuario autenticado.
La clave de clasificación a filtrar es `ClasificacionCorreoRecibidoConstants.Requisicion` (`"requisicion"`), que apunta a los correos procesados por `CorreoRecibidoClienteToCotizacionBO`.
La región y el gestor se obtienen exclusivamente del contexto de autenticación — **no se aceptan como parámetros externos**.
El filtro de región debe aplicarse usando `AsegurarFiltroRegion(ResumeGroupQueryInfo, IdRegion)`.
Dependencia: Tareas 1 y 2 completadas.

**Objetivo general:**
Implementar `CorreoRecibidoClienteBuzonCotizacionesBO.cs` que retorne el listado paginado de correos de cotización para el gestor e `IdRegion` del usuario autenticado.

**Objetivos específicos:**
- Crear `Logic.Pqf.Logistica\L11.MailBot\Procesos\Cotizaciones\CorreoRecibidoClienteBuzonCotizacionesBO.cs`
- Recibir `ResumeGroupQueryInfo` (paginación y filtros) como parámetro de entrada
- Extraer `IdRegion` e `IdUsuario` (gestor) del contexto del BO — nunca de parámetros directos
- Aplicar filtro de región obligatorio con `AsegurarFiltroRegion` antes de ejecutar la consulta
- Filtrar por `ClasificacionCorreoRecibido.Clave = ClasificacionCorreoRecibidoConstants.Requisicion`
- Retornar `PagedList<CorreoRecibidoClienteBuzonCotizacionesDTO>` con los campos requeridos por la vista
- Implementar el mapeo entre entidades de BD y el DTO de respuesta

**Resultado esperado:**
`CorreoRecibidoClienteBuzonCotizacionesBO` retorna el listado paginado de correos de cotización filtrados por región y gestor, listo para ser consumido por `BuzonCotizacionesController`.

**Entregables:**
- Archivo nuevo: `Logic.Pqf.Logistica\L11.MailBot\Procesos\Cotizaciones\CorreoRecibidoClienteBuzonCotizacionesBO.cs`
- Archivo nuevo (DTO): `Logic.Pqf.Logistica\L11.MailBot\DTOs\CorreoRecibidoClienteBuzonCotizacionesDTO.cs`

**Criterios de aceptación:**
- [ ] El BO aplica filtro de región obligatorio antes de ejecutar la consulta
- [ ] El `IdRegion` se obtiene del contexto del BO — no de parámetros de entrada externos
- [ ] Los correos MEX no aparecen en resultados para usuarios PER y viceversa
- [ ] El listado está paginado usando `ResumeGroupQueryInfo`
- [ ] El BO solo retorna correos con clasificación `Clave = 'requisicion'`
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores (Framework 4.8)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Patrón equivalente al GAP-03 del archivo `TPSC-RE-FU-008-P1-Back.md`, pero para clasificación `Requisicion`
- La clave `Requisicion` se define en `ClasificacionCorreoRecibidoConstants` (Tarea 3)
- Los correos en este buzón fueron pretramitados por `CorreoRecibidoClienteToCotizacionBO`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Referencia de patrón (cobros): `Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\CorreoRecibidoClienteBuzonCobrosBO.cs`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 14

### [TPSC-RE-FU-008] [LIST-PAG-MULT-FILTER] Crear BuzonCotizacionesController en WebApi.Logistica

**Aplicativos:**
ProquifaDotNet (Framework 4.8) — `WebApi.Logistica`

**Módulos:**
Mailbot — API — Buzón de Cotizaciones

**Consideraciones previas:**
Esta tarea es el equivalente de la Tarea 5 para el buzón de Cotizaciones.
El controller expone el endpoint `POST /BuzonCotizaciones` que consume `CorreoRecibidoClienteBuzonCotizacionesBO`.
La región y el gestor se extraen del token mediante `ObtenerUsuarioAutenticado()` — **no se aceptan como parámetros en el body o query string**.
El controller debe seguir el patrón `BaseApiController` con `TryExecute()` para manejo de excepciones, idéntico al `BuzonCobrosController` de Tarea 5.
Dependencia: Tarea 13 completada.

**Objetivo general:**
Crear `BuzonCotizacionesController.cs` en `WebApi.Logistica` que exponga `POST /BuzonCotizaciones` con paginación y múltiples filtros, consumiendo `CorreoRecibidoClienteBuzonCotizacionesBO` con la región del usuario autenticado.

**Objetivos específicos:**
- Crear `WebApi.Logistica\Controllers\Mailbot\BuzonCotizacionesController.cs` heredando de `BaseApiController`
- Implementar acción `POST /BuzonCotizaciones` que recibe `ResumeGroupQueryInfo` en el body
- Obtener `Usuario` autenticado con `ObtenerUsuarioAutenticado()` y pasar `IdRegion` al BO
- Invocar `CorreoRecibidoClienteBuzonCotizacionesBO` y retornar el resultado paginado
- Envolver la lógica en `TryExecute()` para manejo uniforme de excepciones
- Aplicar atributo de autorización correspondiente al rol de gestor/agente comercial

**Resultado esperado:**
El endpoint `POST /BuzonCotizaciones` está disponible en `WebApi.Logistica`, retorna el listado paginado de correos de cotización filtrado por la región e identidad del usuario del token.

**Entregables:**
- Archivo nuevo: `WebApi.Logistica\Controllers\Mailbot\BuzonCotizacionesController.cs`

**Criterios de aceptación:**
- [ ] `POST /BuzonCotizaciones` retorna `200 OK` con listado paginado en estructura `ResumeGroupResult`
- [ ] La región y el gestor se extraen del token mediante `ObtenerUsuarioAutenticado()` — no de parámetros externos
- [ ] El controller usa `TryExecute()` para manejo de excepciones
- [ ] El endpoint requiere autenticación (rol de gestor/agente comercial)
- [ ] El proyecto `WebApi.Logistica` compila sin errores (Framework 4.8)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Patrón equivalente al GAP-04 del archivo `TPSC-RE-FU-008-P1-Back.md`, pero para el buzón de cotizaciones
- Referencia de patrón: `BaseApiController.ObtenerUsuarioAutenticado()` en `Core.WebApi\ControllerOperations\BaseApiController.cs`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Referencia de patrón (cobros): `WebApi.Logistica\Controllers\Mailbot\BuzonCobrosController.cs`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 15

### [TPSC-RE-FU-008] [SERV-COMPLEX-TRANSACT] Crear CorreoRecibidoClienteBuzonPedidosBO

**Aplicativos:**
ProquifaDotNet (Framework 4.8) — `Logic.Pqf.Logistica`

**Módulos:**
Mailbot — L11 Pedidos — Buzón de correos de pedido

**Consideraciones previas:**
Esta tarea es el equivalente de la Tarea 4 para el buzón de Pedidos.
El BO devuelve el listado paginado de correos clasificados como Pedido, filtrado por el agente e `IdRegion` del usuario autenticado.
La clave de clasificación a filtrar es `ClasificacionCorreoRecibidoConstants.Pedido` (`"pedido"`), que apunta a los correos procesados por `CorreoRecibidoClienteToPretramitacionPedidoBO`.
La región y el agente se obtienen exclusivamente del contexto de autenticación — **no se aceptan como parámetros externos**.
El filtro de región debe aplicarse usando `AsegurarFiltroRegion(ResumeGroupQueryInfo, IdRegion)`.
Dependencia: Tareas 1 y 2 completadas.

**Objetivo general:**
Implementar `CorreoRecibidoClienteBuzonPedidosBO.cs` que retorne el listado paginado de correos de pedido para el agente e `IdRegion` del usuario autenticado.

**Objetivos específicos:**
- Crear `Logic.Pqf.Logistica\L11.MailBot\Procesos\Pedidos\CorreoRecibidoClienteBuzonPedidosBO.cs`
- Recibir `ResumeGroupQueryInfo` (paginación y filtros) como parámetro de entrada
- Extraer `IdRegion` e `IdUsuario` (agente) del contexto del BO — nunca de parámetros directos
- Aplicar filtro de región obligatorio con `AsegurarFiltroRegion` antes de ejecutar la consulta
- Filtrar por `ClasificacionCorreoRecibido.Clave = ClasificacionCorreoRecibidoConstants.Pedido`
- Retornar `PagedList<CorreoRecibidoClienteBuzonPedidosDTO>` con los campos requeridos por la vista
- Implementar el mapeo entre entidades de BD y el DTO de respuesta

**Resultado esperado:**
`CorreoRecibidoClienteBuzonPedidosBO` retorna el listado paginado de correos de pedido filtrados por región y agente, listo para ser consumido por `BuzonPedidosController`.

**Entregables:**
- Archivo nuevo: `Logic.Pqf.Logistica\L11.MailBot\Procesos\Pedidos\CorreoRecibidoClienteBuzonPedidosBO.cs`
- Archivo nuevo (DTO): `Logic.Pqf.Logistica\L11.MailBot\DTOs\CorreoRecibidoClienteBuzonPedidosDTO.cs`

**Criterios de aceptación:**
- [ ] El BO aplica filtro de región obligatorio antes de ejecutar la consulta
- [ ] El `IdRegion` se obtiene del contexto del BO — no de parámetros de entrada externos
- [ ] Los correos MEX no aparecen en resultados para usuarios PER y viceversa
- [ ] El listado está paginado usando `ResumeGroupQueryInfo`
- [ ] El BO solo retorna correos con clasificación `Clave = 'pedido'`
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores (Framework 4.8)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Patrón equivalente al GAP-03 del archivo `TPSC-RE-FU-008-P1-Back.md`, pero para clasificación `Pedido`
- La clave `Pedido` se define en `ClasificacionCorreoRecibidoConstants` (Tarea 3)
- Los correos en este buzón fueron pretramitados por `CorreoRecibidoClienteToPretramitacionPedidoBO`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Referencia de patrón (cobros): `Logic.Pqf.Logistica\L11.MailBot\Procesos\Cobros\CorreoRecibidoClienteBuzonCobrosBO.cs`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 16

### [TPSC-RE-FU-008] [LIST-PAG-MULT-FILTER] Crear BuzonPedidosController en WebApi.Logistica

**Aplicativos:**
ProquifaDotNet (Framework 4.8) — `WebApi.Logistica`

**Módulos:**
Mailbot — API — Buzón de Pedidos

**Consideraciones previas:**
Esta tarea es el equivalente de la Tarea 5 para el buzón de Pedidos.
El controller expone el endpoint `POST /BuzonPedidos` que consume `CorreoRecibidoClienteBuzonPedidosBO`.
La región y el agente se extraen del token mediante `ObtenerUsuarioAutenticado()` — **no se aceptan como parámetros en el body o query string**.
El controller debe seguir el patrón `BaseApiController` con `TryExecute()` para manejo de excepciones, idéntico al `BuzonCobrosController` de Tarea 5.
Dependencia: Tarea 15 completada.

**Objetivo general:**
Crear `BuzonPedidosController.cs` en `WebApi.Logistica` que exponga `POST /BuzonPedidos` con paginación y múltiples filtros, consumiendo `CorreoRecibidoClienteBuzonPedidosBO` con la región del usuario autenticado.

**Objetivos específicos:**
- Crear `WebApi.Logistica\Controllers\Mailbot\BuzonPedidosController.cs` heredando de `BaseApiController`
- Implementar acción `POST /BuzonPedidos` que recibe `ResumeGroupQueryInfo` en el body
- Obtener `Usuario` autenticado con `ObtenerUsuarioAutenticado()` y pasar `IdRegion` al BO
- Invocar `CorreoRecibidoClienteBuzonPedidosBO` y retornar el resultado paginado
- Envolver la lógica en `TryExecute()` para manejo uniforme de excepciones
- Aplicar atributo de autorización correspondiente al rol de agente comercial

**Resultado esperado:**
El endpoint `POST /BuzonPedidos` está disponible en `WebApi.Logistica`, retorna el listado paginado de correos de pedido filtrado por la región e identidad del usuario del token.

**Entregables:**
- Archivo nuevo: `WebApi.Logistica\Controllers\Mailbot\BuzonPedidosController.cs`

**Criterios de aceptación:**
- [ ] `POST /BuzonPedidos` retorna `200 OK` con listado paginado en estructura `ResumeGroupResult`
- [ ] La región y el agente se extraen del token mediante `ObtenerUsuarioAutenticado()` — no de parámetros externos
- [ ] El controller usa `TryExecute()` para manejo de excepciones
- [ ] El endpoint requiere autenticación (rol de agente comercial)
- [ ] El proyecto `WebApi.Logistica` compila sin errores (Framework 4.8)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Patrón equivalente al GAP-04 del archivo `TPSC-RE-FU-008-P1-Back.md`, pero para el buzón de pedidos
- Referencia de patrón: `BaseApiController.ObtenerUsuarioAutenticado()` en `Core.WebApi\ControllerOperations\BaseApiController.cs`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md`
- Referencia de patrón (cobros): `WebApi.Logistica\Controllers\Mailbot\BuzonCobrosController.cs`
- Requisito funcional: `Requisitos/TPSC-RE-FU-008/TPSC-RE-FU-008.md`

---

## Tarea 17

### [TPSC-RE-FU-008] [IMP-EXIST-SERVICE] Actualizar lógica de clasificación y enrutamiento para Cotizaciones y Pedidos

**Aplicativos:**
ProquifaDotNet (Framework 4.8) — `Logic.Pqf.Logistica` / MailbotWorker (nueva solución .NET 10) — `Mailbot.Application`

**Módulos:**
Mailbot — L11 — Clasificación y enrutamiento de correos: Cotizaciones y Pedidos

**Consideraciones previas:**
La Tarea 3 refactoriza el switch de `GeneradorProcesoMailBotBO` para usar `Clave` y corrige el `case Cobro` (vacío), pero los `case Requisicion` y `case Pedido` se marcan solo como "siguen funcionando". Esta tarea hace el análisis explícito de ambos BOs de procesamiento en Framework 4.8, verifica que producen los pendientes correctos, y detalla la implementación de los `case Cotizacion` y `case Pedido` en `GenerarPendienteUseCase` del Worker .NET 10.

Los BOs de Framework 4.8 ya existen:
- `CorreoRecibidoClienteToCotizacionBO` — crea el pendiente de cotización (pretramitación de requisición)
- `CorreoRecibidoClienteToPretramitacionPedidoBO` — crea el pendiente de pedido (pretramitación de pedido)

Se debe revisar que ambos BOs sean compatibles con el nuevo flujo (clasificación por `Clave` en lugar de nombre display) y que generen correctamente los registros esperados.

Para el Worker .NET 10, la Tarea 9 (`GenerarPendienteUseCase`) menciona los cases de Cotizacion y Pedido con una descripción genérica. Esta tarea implementa y valida el flujo completo de cada uno dentro de `GenerarPendienteUseCase`.

Dependencias: Tarea 3 completada (constantes y switch refactorizado), Tarea 9 completada (estructura base de `GenerarPendienteUseCase`).

**Objetivo general:**
Verificar y actualizar `CorreoRecibidoClienteToCotizacionBO` y `CorreoRecibidoClienteToPretramitacionPedidoBO` en Framework 4.8 para que sean compatibles con el nuevo sistema de clasificación por `Clave`, e implementar los `case Cotizacion` y `case Pedido` completos en `GenerarPendienteUseCase` del Worker .NET 10.

**Objetivos específicos:**

*Framework 4.8 — `Logic.Pqf.Logistica`:*
- Revisar `CorreoRecibidoClienteToCotizacionBO.Process()`: verificar que recibe `CorreoRecibidoCliente` y genera el pendiente de cotización (`cotCotizacionPendiente` o equivalente) sin depender del nombre display de la clasificación
- Revisar `CorreoRecibidoClienteToPretramitacionPedidoBO.Process()`: verificar que genera el pendiente de pedido correctamente y no depende del nombre display
- Confirmar que `GeneradorProcesoMailBotBO` (post-Tarea 3) enruta correctamente:
  - `case ClasificacionCorreoRecibidoConstants.Requisicion` → `CorreoRecibidoClienteToCotizacionBO`
  - `case ClasificacionCorreoRecibidoConstants.Pedido` → `CorreoRecibidoClienteToPretramitacionPedidoBO`
- Corregir cualquier referencia a nombre display o literal de clasificación que persista en los BOs revisados
- Prueba de integración: correo con clasificación `Clave='requisicion'` genera pendiente de cotización; correo con `Clave='pedido'` genera pendiente de pedido

*Worker .NET 10 — `Mailbot.Application`:*
- Implementar `case "cotizacion"` en `GenerarPendienteUseCase`:
  - INSERT en la tabla que corresponda al pendiente de cotización (equivalente a `cotCotizacionPendiente` en Framework 4.8)
  - INSERT en `CorreoRecibidoCliente` con el vínculo al correo procesado
  - Actualizar `MailbotEventoCorreo` con estado PROCESADO
- Implementar `case "pedido"` en `GenerarPendienteUseCase`:
  - INSERT en la tabla que corresponda al pendiente de pedido (equivalente a `ppPretramitacion` o tabla de pretramitación de pedido)
  - INSERT en `CorreoRecibidoCliente` con el vínculo al correo procesado
  - Actualizar `MailbotEventoCorreo` con estado PROCESADO
- Agregar al scaffold de EF Core (`ProquifaDbContext`) las tablas de pendientes de cotización y pedido si no están incluidas
- Validar que en caso de error en el INSERT, la transacción se revierte y `MailbotEventoCorreo` queda en estado ERROR con el mensaje correspondiente

**Resultado esperado:**
El Mailbot clasifica y enruta correctamente correos de Cotización y Pedido en ambos contextos: Framework 4.8 (enruta a los BOs existentes) y Worker .NET 10 (genera los pendientes directamente en BD). Un correo de cotización recibido produce el pendiente de cotización; un correo de pedido produce el pendiente de pretramitación de pedido.

**Entregables:**

*Framework 4.8:*
- Informe de revisión + commits con correcciones en `CorreoRecibidoClienteToCotizacionBO.cs` (si aplica)
- Informe de revisión + commits con correcciones en `CorreoRecibidoClienteToPretramitacionPedidoBO.cs` (si aplica)

*Worker .NET 10:*
- `Mailbot.Application\UseCases\GenerarPendienteUseCase.cs` actualizado con cases `cotizacion` y `pedido` completos
- Scaffold de EF Core actualizado si se requieren tablas adicionales de pendientes

**Criterios de aceptación:**
- [ ] `GeneradorProcesoMailBotBO` enruta `Clave='requisicion'` → `CorreoRecibidoClienteToCotizacionBO` y genera el pendiente de cotización correctamente (Framework 4.8)
- [ ] `GeneradorProcesoMailBotBO` enruta `Clave='pedido'` → `CorreoRecibidoClienteToPretramitacionPedidoBO` y genera el pendiente de pedido correctamente (Framework 4.8)
- [ ] Ningún BO de cotización o pedido referencia el nombre display de la clasificación
- [ ] `GenerarPendienteUseCase` `case "cotizacion"` inserta el pendiente de cotización en BD y actualiza `MailbotEventoCorreo` (Worker .NET 10)
- [ ] `GenerarPendienteUseCase` `case "pedido"` inserta el pendiente de pretramitación de pedido en BD y actualiza `MailbotEventoCorreo` (Worker .NET 10)
- [ ] En caso de error en el INSERT, la transacción se revierte y el evento queda en estado ERROR
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores (Framework 4.8)
- [ ] La solución `MailbotWorker.sln` compila en .NET 10 sin errores tras esta tarea
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Complementa la Tarea 3 (switch refactorizado) y la Tarea 9 (estructura base de `GenerarPendienteUseCase`)
- Las tablas de pendientes de cotización y pedido deben confirmarse con el TechLead antes de implementar el `case` en .NET 10 (pueden diferir del Framework 4.8)
- Si los BOs existentes en Framework 4.8 no requieren cambios, los entregables se reducen al informe de revisión y a la actualización de `GenerarPendienteUseCase`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-008-P1/TPSC-RE-FU-008-P1-Back.md` — Sección 3.1 (flujo de clasificación) y Sección 4.1
- `Logic.Pqf.Logistica\L11.MailBot\Procesos\Requisiciones\CorreoRecibidoClienteToCotizacionBO.cs`
- `Logic.Pqf.Logistica\L11.MailBot\Procesos\Pedidos\CorreoRecibidoClienteToPretramitacionPedidoBO.cs`
- `Mailbot.Application\UseCases\GenerarPendienteUseCase.cs` (producido por Tarea 9)
- Dependencias: Tarea 3 (T3) y Tarea 9 (T9)

---

## Resumen de tareas

|  #  | Clave                 | Descripcion                                                                         | Proyecto                                                   | Dependencias |
| :-: | --------------------- | ----------------------------------------------------------------------------------- | ---------------------------------------------------------- | ------------ |
|  1  | BD-OBJ-CH             | Script UPDATE catClasificacionCorreoRecibido + INSERT catProceso                    | BD ProquifaDotNet                                          | —            |
|  2  | BD-OBJ-M              | Script CREATE TABLE MailbotEventoCorreo + MailbotClasificacionLog                   | BD ProquifaDotNet                                          | 1            |
|  3  | IMP-EXIST-SERVICE     | Refactorizar GeneradorProcesoMailBotBO + crear ClasificacionCorreoRecibidoConstants | Logic.Pqf.Logistica (F4.8)                                 | 1            |
|  4  | SERV-COMPLEX-TRANSACT | Crear CorreoRecibidoClienteBuzonCobrosBO                                            | Logic.Pqf.Logistica (F4.8)                                 | 1, 2         |
|  5  | LIST-PAG-MULT-FILTER  | Crear BuzonCobrosController                                                         | WebApi.Logistica (F4.8)                                    | 4            |
|  6  | ARQ-PROJ-NET          | Crear MailbotWorker.sln — estructura base + scaffold EF Core                        | Nueva solución .NET 10                                     | 1, 2         |
|  7  | IMPL-THIRD-SERV       | Integrar n8n + RabbitMQ — consumer y queues                                         | Mailbot.Infrastructure (.NET 10)                           | 6            |
|  8  | ALG-COMPLX-LOGIC      | Implementar Agente IA (OpenAIClasificadorAgente + PromptBuilder)                    | Mailbot.Infrastructure (.NET 10)                           | 6            |
|  9  | SERV-COMPLEX-TRANSACT | Implementar ProcesarCorreoUseCase + GenerarPendienteUseCase                         | Mailbot.Application (.NET 10)                              | 6, 7, 8      |
| 10  | IMPL-THIRD-SERV       | Integrar MinIO (MinioArchivoService)                                                | Mailbot.Infrastructure (.NET 10)                           | 6            |
| 11  | ATTACHED-EMAIL        | Integrar Brevo (BrevoNotificacionService)                                           | Mailbot.Infrastructure (.NET 10)                           | 6            |
| 12  | AUTOMATIC-JOB         | Configurar CorreoWorkerMex + CorreoWorkerPer (IHostedService)                       | Mailbot.Worker (.NET 10)                                   | 7, 9, 10, 11 |
| 13  | SERV-COMPLEX-TRANSACT | Crear CorreoRecibidoClienteBuzonCotizacionesBO                                      | Logic.Pqf.Logistica (F4.8)                                 | 1, 2         |
| 14  | LIST-PAG-MULT-FILTER  | Crear BuzonCotizacionesController                                                   | WebApi.Logistica (F4.8)                                    | 13           |
| 15  | SERV-COMPLEX-TRANSACT | Crear CorreoRecibidoClienteBuzonPedidosBO                                           | Logic.Pqf.Logistica (F4.8)                                 | 1, 2         |
| 16  | LIST-PAG-MULT-FILTER  | Crear BuzonPedidosController                                                        | WebApi.Logistica (F4.8)                                    | 15           |
| 17  | IMP-EXIST-SERVICE     | Actualizar lógica de clasificación y enrutamiento: Cotizaciones y Pedidos           | Logic.Pqf.Logistica (F4.8) + Mailbot.Application (.NET 10) | 3, 9         |
