# [CAMBIO] Generación de Servicio de Envío de Correo — Descripción para JIRA

**Título sugerido:** [ CAMBIO ] [ R16A-NO-FU-001 ] Generación de Servicio de Envío de Correo — Solución proquifa.pqf2.notificaciones

```
h2. Descripción del cambio

Se genera una nueva solución denominada *proquifa.pqf2.notificaciones*, desarrollada en .NET Core 10, que centraliza el servicio de envío de correo electrónico vía Brevo (SendInBlue), reemplazando la infraestructura actual de envío de ProquifaDotNet (.NET Framework 4.8): _Worker.SendInBlue, _Ejecutable.SincronizadorSendInBlue, ConfiguracionSendinBlueBO y CorreoGenericoBO con plantillas en disco.

El cambio incluye:

* *Base de datos nueva Pqf2Notificaciones*, dedicada exclusivamente al servicio de correo. No comparte tablas con ProquifaDotNet y el servicio no lee datos de otros sistemas: la comunicación es exclusivamente vía llamadas entre APIs — el modelo de datos del correo y los archivos adjuntos viajan dentro de la petición.
* *Solución nueva pqf2.notificaciones* con el mismo arquetipo: Domain, Application (CQRS con MediatR), Infrastructure (EF Core, Brevo, RabbitMQ, IdentityServer, Serilog), API, Worker.Notificaciones y Testing.
* *Integración con ProquifaDotNet* mediante llamadas entre APIs autenticadas con IdentityServer, manteniendo las firmas públicas actuales para no impactar a los módulos consumidores.

h2. Alcance

*Base de datos (Pqf2Notificaciones):*

* Tablas de configuración: Similar a ConfiguracionSendInBlue (credenciales Brevo cifradas, una fila por región México/Perú), PlantillaCorreo (plantillas HTML centralizadas, hoy en disco) y AppSettings (parámetros operativos, hoy en App.config del Worker).
* Tablas operativas: SolicitudCorreo (cola persistente de trabajo con estados PENDIENTE/PROCESANDO/ENVIADO/ERROR/CANCELADO y reintentos con backoff exponencial) y BitacoraEnvioCorreo (auditoría de cada intento: HTTP status, messageId, respuesta Brevo, duración).

*Solución pqf2.notificaciones:*

* Domain: 5 entidades mapeadas 1:1 a BD, interfaces de repositorios, IBrevoMailService y enums TipoEnvioCorreo (TEMPLATE, SIMPLE, HTML, BREVO_TEMPLATE) y EstadoSolicitudCorreo.
* Application: 4 Commands (EnviarCorreo, EnviarCorreoSimple, EnviarCorreoHtml, SincronizarEstadoCorreo) + 2 Queries, DTOs y validators FluentValidation.
* Infrastructure: NotificacionesDbContext (base de datos propia Pqf2Notificaciones) + repositorios — sin Scaffold ni DbContext de lectura hacia ProquifaDotNet: el servicio no consulta datos de otros sistemas; BrevoMailService (HTTP, API key por región, timeout configurable); XsltTemplateRenderer (migración de Logic.MailXslt a .NET Core 10, renderiza a partir del modelo de datos recibido en la petición) y HtmlTemplateRenderer; RabbitMQPublisher (cola queueSendInBlue); IdentityServer y Serilog.
* API: 4 endpoints REST protegidos con Bearer token — POST /api/correo/enviar (reemplaza PATCH /EnviarCorreo), /simple y /html (reemplazan CorreoGenericoBO) y /plantilla-brevo (templateId nativo de Brevo). El modelo de parámetros de envío incluye el modelo de datos del correo (receptores, asunto, datos para la plantilla) y los archivos adjuntos dentro de la misma petición — el servicio no recibe solo un Id para consultar los datos por su cuenta. Respuestas estandarizadas, catálogo de errores SENDINBLUE-001 a 010 y Swagger documentado.
* Worker.Notificaciones: NotificacionesWorker (reemplaza _Worker.SendInBlue) consume queueSendInBlue con reintentos por backoff exponencial (FechaProximoIntento = Now + 2^Intentos min; al MaxIntentos → CANCELADO + log crítico), procesando cada solicitud con el modelo de datos y adjuntos persistidos con la propia SolicitudCorreo — sin lecturas a ProquifaDotNet ni descargas externas; el flujo IdCFDI para correos de factura recibe el XML + PDF como adjuntos en la petición; SincronizacionWorker (reemplaza _Ejecutable.SincronizadorSendInBlue) con cron configurable notifica el estado de entrega a ProquifaDotNet vía API (sin acceso directo a su base de datos).
* Plantillas y pruebas: migración de los 10+ XSLT de Logic.MailXslt con soporte regional México/Perú, tests unitarios (coverage > 70% en Application) y de integración del flujo completo.

*Ajustes en ProquifaDotNet (.NET Framework 4.8):*

* CorreoEnviadoEnviarController.EnviarCorreo: Actualizar el encolado RabbitMQ directo por llamada HTTP autenticada al nuevo API (URL configurable por appSettings). ProquifaDotNet arma el payload completo — modelo de datos del correo y archivos adjuntos — y lo envía dentro de la petición. Si el API no responde, retorna false sin excepción.
* CorreoGenericoBO.GenerarCorreo<T>: delega al nuevo API manteniendo la firma pública sin cambios — los 9 módulos dependientes (autorizaciones, cotizaciones, investigación, incidencias, FEA, promesa de compra, cartera, clientes) no se modifican.

h2. Política de reintentos

* Los reintentos de *procesos* de PQF, Finanzas u otro sistema/servicio se llevan por el worker del sistema correspondiente o por RabbitMQ, dependiendo del sistema — no son responsabilidad del servicio de notificaciones.
* Los reintentos de *envíos* que gestiona pqf2.notificaciones son únicamente aquellos que no necesitan un Id de Envío para vincular el envío con una tabla en específico. Cuando el envío requiere vinculación por Id de Envío con una tabla del sistema de origen, el reintento lo gobierna dicho sistema.

h2. Fuera de alcance

* Cambios funcionales en los 9 módulos consumidores de CorreoGenericoBO (solo se verifica compilación y funcionamiento).
* GET /ObtenerHtmlCorreo de ProquifaDotNet (mantiene su reflección local).
* Acceso del servicio a bases de datos de otros sistemas (Scaffold o DbContext de lectura sobre ProquifaDotNet): el servicio solo persiste en su propia base de datos Pqf2Notificaciones.
* Cambios en el contenido o diseño de las plantillas de correo (solo migración).
* Reintentos de procesos de PQF, Finanzas u otros sistemas/servicios: se gestionan en el worker o RabbitMQ del sistema correspondiente (ver Política de reintentos).

h2. Justificación

* Desacoplar el envío de correo del monolito .NET Framework 4.8 hacia un servicio moderno, escalable y mantenible en .NET Core 10.
* Centralizar credenciales (cifradas, por región), plantillas y parámetros operativos en base de datos, eliminando configuración en disco y App.config.
* Trazabilidad completa del envío: cola persistente con reintentos automáticos y bitácora de cada intento.
* Alinear el servicio al arquetipo y estándares transversales del proyecto R16 (CQRS, IdentityServer, RabbitMQ, Serilog, catálogo de errores).

h2. Resultado esperado

Todo correo de ProquifaDotNet (flujo principal y correos genéricos) se envía a través de proquifa.pqf2.notificaciones: el sistema de origen manda en la petición el modelo de datos del correo y sus adjuntos, la solicitud queda encolada en SolicitudCorreo, se procesa por NotificacionesWorker con reintentos automáticos, se registra cada intento en BitacoraEnvioCorreo y el estado de entrega (lectura/spam) se notifica de vuelta a ProquifaDotNet vía API. Los módulos actuales funcionan sin cambios. Los reintentos de envío del servicio aplican solo a envíos sin vinculación por Id de Envío; los reintentos de procesos permanecen en el worker o RabbitMQ del sistema de origen.
```
