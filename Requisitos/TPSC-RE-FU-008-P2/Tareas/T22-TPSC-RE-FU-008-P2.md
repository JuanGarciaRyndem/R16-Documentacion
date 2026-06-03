# [ TPSC-RE-FU-008 ] [SERVER-AMB] Configurar Google Cloud Pub/Sub — Topics, Subscriptions, DLQ, permisos IAM

---

## Aplicativos

- MailbotWorker (Infraestructura GCP)

## Módulos

- Mailbot

## Consideraciones previas

- Requiere cuenta GCP con proyecto definido (confirmar con IT).
- Requiere service account con rol `pubsub.publisher`.
- El endpoint HTTPS público de `Mailbot.Api` debe estar disponible (T20 + infraestructura de red).

## Objetivo general

Configurar la infraestructura de Google Cloud Pub/Sub necesaria para que Gmail envíe notificaciones push al Mailbot por región.

## Objetivos específicos

1. Crear Topic `mailbot-mex` en proyecto GCP.
2. Crear Topic `mailbot-per` en proyecto GCP.
3. Crear Topic `mailbot-dlq` (Dead Letter) para mensajes fallidos.
4. Crear Subscription Push `mailbot-mex-push`:
   - Topic: `mailbot-mex`
   - Endpoint: `https://mailbot.proquifa.mx/webhook/gmail/mex`
   - Ack deadline: 60 segundos
   - Max delivery attempts: 5
   - Dead letter topic: `mailbot-dlq`
5. Crear Subscription Push `mailbot-per-push` (análogo para PER).
6. Configurar permisos IAM:
   - Service account `gmail-api@proquifa-prod.iam.gserviceaccount.com` con rol `roles/pubsub.publisher` en ambos topics.

## Resultado esperado

- Pub/Sub recibe mensajes de Gmail y los entrega vía push HTTP al Mailbot.Api.
- Mensajes fallidos van al DLQ para revisión manual.

## Entregables

- Documentación de configuración GCP (screenshots o terraform/gcloud commands).
- Validación de conectividad push → endpoint.

## Criterios de aceptación

- [ ] Los topics existen y están activos en GCP.
- [ ] Las subscriptions push apuntan a los endpoints correctos.
- [ ] El DLQ recibe mensajes que exceden `max_delivery_attempts`.
- [ ] Los permisos IAM están correctamente asignados.
- [ ] Una prueba de push manual llega al endpoint de `Mailbot.Api`.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-v2_Propuesta2.md` — sección "Configuración Google Cloud Pub/Sub"
- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 2, sección 2.5

## Recursos

- Proyecto GCP: proquifa-prod (confirmar con IT)
- Dominio: mailbot.proquifa.mx (confirmar con infraestructura)
