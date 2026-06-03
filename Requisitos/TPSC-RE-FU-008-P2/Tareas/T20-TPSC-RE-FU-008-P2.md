# [ TPSC-RE-FU-008 ] [SERV-COMPLEX-TRANSACT] Implementar Mailbot.Api — endpoints webhook + middleware + health

---

## Aplicativos

- MailbotWorker

## Módulos

- Mailbot

## Consideraciones previas

- Tareas T13 y T15 completadas (solución y EventoCorreoChannel listos).
- El endpoint webhook debe ser accesible desde internet (Google Cloud Pub/Sub hace push HTTPS).
- El middleware de validación de token evita que terceros inyecten eventos falsos.

## Objetivo general

Implementar la Minimal API que recibe las notificaciones push de Google Cloud Pub/Sub, valida el token, registra el evento en BD y lo publica en el Channel interno para procesamiento asíncrono por el Worker.

## Objetivos específicos

1. `Endpoints\WebhookGmailEndpoints.cs`:
   - `POST /webhook/gmail/{region}` — recibe payload Pub/Sub, decodifica base64, INSERT `MailbotEventoCorreo`, publica en `IEventoChannel`, retorna 200 OK.
2. `Endpoints\HealthEndpoints.cs`:
   - `GET /health` — health check básico.
   - `GET /metrics` — métricas básicas (eventos recibidos, procesados, fallidos).
3. `Endpoints\ReentrenamientoEndpoints.cs`:
   - `POST /reentrenamiento/trigger` — dispara proceso de reentrenamiento (placeholder inicial).
4. `Middleware\PubSubTokenValidationMiddleware.cs`:
   - Valida el Bearer token de Google en cada request al webhook.
   - Rechaza con 403 si el token no es válido.
5. Configurar `Program.cs` con DI, middleware y mapeo de endpoints.

## Resultado esperado

- La API recibe pushes de Pub/Sub, valida seguridad, registra eventos y los encola para el Worker.

## Entregables

- 3 archivos de endpoints + 1 middleware + Program.cs configurado.

## Criterios de aceptación

- [ ] `POST /webhook/gmail/mex` responde 200 OK con payload válido.
- [ ] `POST /webhook/gmail/mex` responde 403 con token inválido.
- [ ] El evento se registra en `MailbotEventoCorreo` con `Procesado = 0`.
- [ ] El evento se publica en el Channel interno.
- [ ] `GET /health` responde 200 OK.
- [ ] El proyecto compila y arranca sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-v2_Propuesta2.md` — secciones "Payload del Push" y "Configuración Gmail API Watch"
- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 2, sección 2.2

## Recursos

- Repositorio: MailbotWorker
