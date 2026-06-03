# [ TPSC-RE-FU-008 ] [CONFIG-GMAIL] Configurar Gmail API watch por región (MEX/PER) con OAuth2

---

## Aplicativos

- MailbotWorker

## Módulos

- Mailbot

## Consideraciones previas

- Tarea T22 completada (Pub/Sub configurado — los topics existen).
- Se requieren credenciales OAuth2 para cada cuenta de correo monitoreada (MEX y PER).
- El watch expira cada 7 días — `GmailWatchRenovacionWorker` (T21) lo renueva cada 6.
- Las credenciales deben almacenarse fuera del repositorio (Key Vault / env vars / secrets).

## Objetivo general

Configurar la suscripción Gmail API watch para las cuentas de correo de México y Perú, de modo que Gmail notifique al Pub/Sub cada vez que llega un correo nuevo al INBOX.

## Objetivos específicos

1. Obtener/crear credenciales OAuth2 para la cuenta de correo MEX.
2. Obtener/crear credenciales OAuth2 para la cuenta de correo PER.
3. Ejecutar `Gmail API watch` para MEX apuntando al topic `projects/proquifa-prod/topics/mailbot-mex`.
4. Ejecutar `Gmail API watch` para PER apuntando al topic `projects/proquifa-prod/topics/mailbot-per`.
5. Registrar `WatchHistoryId` y `WatchExpiration` en `RegionConfiguracionMailBot` para cada región.
6. Configurar gestión de secretos (Key Vault / env vars) para `gmail_credentials.json` y tokens OAuth2.
7. Verificar que la primera notificación push llega al endpoint.

## Resultado esperado

- Gmail notifica automáticamente al Pub/Sub cuando llega un correo nuevo a cualquiera de las dos cuentas monitoreadas.

## Entregables

- Credenciales OAuth2 configuradas (fuera del repositorio).
- Watch activo para MEX y PER.
- Registros actualizados en `RegionConfiguracionMailBot`.
- Documentación del proceso de renovación manual (por si falla el Worker).

## Criterios de aceptación

- [ ] Al enviar un correo de prueba al INBOX MEX, el push llega a `POST /webhook/gmail/mex`.
- [ ] Al enviar un correo de prueba al INBOX PER, el push llega a `POST /webhook/gmail/per`.
- [ ] `RegionConfiguracionMailBot` tiene `WatchExpiration` registrado para ambas regiones.
- [ ] Las credenciales no están en el repositorio.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-v2_Propuesta2.md` — sección "Configuración Gmail API Watch"
- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 2, sección 2.5

## Recursos

- Proyecto GCP: proquifa-prod
- Cuentas de correo: confirmar con IT
