# Endpoints — ProquifaDotNet.SendInBlue (.NET Core 10)

> ⚠️ **Corrección (07/07/2026):** este archivo conflaba dos aplicaciones distintas bajo un solo nombre ("EnvioCorreo / SendInBlue") y usaba rutas fabricadas (`/api/correo/enviar`, `/simple`, `/html`, `/plantilla-brevo`). Son **dos apps separadas** — ver sección final.

Endpoints de la solución `ProquifaDotNet.SendInBlue` (único requisito con especificación propia: **R16A-NO-FU-001**). Migra la infraestructura de envío de correo desde el Windows Service legado `_Worker.SendInBlue` (.NET 4.8) hacia una solución independiente en .NET Core 10, con CQRS, RabbitMQ, IdentityServer y soporte multi-región.

> **Base URL:** `https://{servidor-sendinblue}/api/`
> **Autenticación:** IdentityServer (token JWT en header `Authorization: Bearer {token}`)
> **Proveedor externo:** Brevo (ex-SendInBlue) — API transaccional
> **Cola:** RabbitMQ para envío asincrónico / failover; `Worker.SendMail` procesa la cola y sincroniza estado de entrega

---

## Contexto

`ProquifaDotNet` seguirá siendo el sistema de Venta Interna. `ProquifaDotNet.SendInBlue` se integra como servicio externo mediante **llamadas entre APIs**: ProquifaDotNet llama al API de SendInBlue para solicitar el envío. El endpoint legado `PATCH /EnviarCorreo` del controller `CorreoEnviadoEnviarController` (en `WebApi.Catalogos`) se **refactoriza** para delegar en `POST /api/v1/mail/send` en lugar de encolar directamente en RabbitMQ.

**Flujo de envío:**
```
Caller (ProquifaDotNet / Finanzas, si aplica)
    │
    └─ POST /api/v1/mail/send | /simple | /html | /template
            │
            ├─ Envío directo  → Brevo API → { brevoMessageId, enviado: true }
            └─ Envío/failover → encola en RabbitMQ → Worker.SendMail procesa
                                → resuelve plantilla XSLT / Brevo
                                → envía a Brevo
                                → actualiza estado + bitácora de envío
```

---

## Endpoints

**Controller:** `CorreoController` (4 endpoints)

### POST `/api/v1/mail/send`

Solicita envío de correo basado en un registro `CorreoEnviado` existente en ProquifaDotNet (flujo principal con plantilla XSLT). Encola en RabbitMQ → Worker resuelve la plantilla → envía a Brevo.

**Request:**
```json
{
  "idCorreoEnviado": "guid",
  "idProcesoSistema": "guid"
}
```

---

### POST `/api/v1/mail/simple`

Envío simple sin asociación a `CorreoEnviado`. Para notificaciones ad-hoc internas, usando `PlantillaCorreo` por clave lógica.

**Request:**
```json
{
  "idRegion": "guid",
  "receptores": [{ "nombre": "string", "correo": "string" }],
  "conCopia": [{ "nombre": "string", "correo": "string" }],
  "asunto": "string",
  "clavePlantilla": "string",
  "parametros": [{ "nombre": "string", "valor": "string" }]
}
```

**Comportamiento:** envío directo (sin cola) o failover a RabbitMQ si Brevo no responde.

---

### POST `/api/v1/mail/html`

Envío con contenido HTML explícito, para correos donde el caller ya generó el HTML (p. ej. Finanzas arma el cuerpo del correo de factura). Soporta adjuntos vía URL de MinIO.

**Request:**
```json
{
  "idRegion": "guid",
  "receptores": [{ "nombre": "string", "correo": "string" }],
  "conCopia": [{ "nombre": "string", "correo": "string" }],
  "asunto": "string",
  "htmlContent": "string",
  "adjuntos": [{ "nombre": "string", "urlMinIO": "string" }]
}
```

**Comportamiento:** envío directo (sin cola) o failover a RabbitMQ.

---

### POST `/api/v1/mail/template`

Envío usando una plantilla nativa de Brevo (renderizada en servidores Brevo). Se referencia por `clave` lógica (lookup en `CatalogoPlantillaBrevo`) o por `idTemplateBrevo` directo — debe estar presente exactamente uno de los dos.

**Request:**
```json
{
  "idRegion": "guid",
  "receptores": [{ "nombre": "string", "correo": "string" }],
  "conCopia": [{ "nombre": "string", "correo": "string" }],
  "clave": "BIENVENIDA_CLIENTE",
  "idTemplateBrevo": null,
  "params": { "nombre": "string", "empresa": "string", "urlAccion": "string" }
}
```

**Response (envío directo):**
```json
{ "brevoMessageId": "string", "enviado": true }
```

**Response (encolado):**
```json
{ "idSolicitudCorreo": "guid", "estado": "PENDIENTE" }
```

---

## Endpoint de Integración en ProquifaDotNet (Refactorizado)

| Método | Ruta (ProquifaDotNet) | Delegado a               | Descripción                                                                                     |
| ------ | --------------------- | ------------------------ | ----------------------------------------------------------------------------------------------- |
| PATCH  | `/EnviarCorreo`       | `POST /api/v1/mail/send` | Refactorizado: en lugar de encolar directamente en RabbitMQ, ahora llama al servicio SendInBlue |

Fuente: `R16A-NO-FU-001.md` líneas 205-254, 402-583.

---

## Resumen de Endpoints

| Endpoint | Tipo envío | Req. |
|---|---|---|
| `POST /api/v1/mail/send` | Basado en `CorreoEnviado` existente | NO-FU-001 |
| `POST /api/v1/mail/simple` | Ad-hoc con plantilla interna | NO-FU-001 |
| `POST /api/v1/mail/html` | HTML explícito + adjuntos MinIO | NO-FU-001 |
| `POST /api/v1/mail/template` | Plantilla nativa Brevo | NO-FU-001 |
| **Total API SendInBlue** | | **4** |

---

## ⚠️ ProquifaDotNet.EnvioCorreo — aplicación distinta, sin requisito propio aún

Los requisitos de Finanzas y Timbrado (RE-016 en adelante, "Reglas al diseñar — regla 7") referencian repetidamente un **Aplicativo Nuevo** llamado `ProquifaDotNet.EnvioCorreo` como el canal obligatorio para cualquier envío de correo desde Finanzas/Timbrado/Validar Cobro — explícitamente descrito como **independiente** de `ProquifaDotNet.SendInBlue` (que solo migra el envío de correo del sistema legado ProquifaDotNet/Venta Interna).

**No existe ningún requisito (RE-FU-XXX o NO-FU-XXX) que documente el contrato, los endpoints o la arquitectura de `ProquifaDotNet.EnvioCorreo` todavía.** Cada vez que un requisito de Finanzas/Timbrado lo menciona, lo hace como referencia forward a una integración pendiente de detallar (ver p. ej. `R16A-RE-FU-018-Back.md` línea 153: "el detalle de la integración (contrato, endpoint) aún no está documentado en un requisito propio").

Este archivo **no puede documentar las rutas de `ProquifaDotNet.EnvioCorreo`** hasta que exista esa fuente. Cuando se cree el requisito correspondiente, debe agregarse aquí como una sección nueva, separada de SendInBlue.
