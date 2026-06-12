# Endpoints — ProquifaDotNet.EnvioCorreo / SendInBlue (.NET Core 10)

Endpoints de la solución nueva `ProquifaDotNetSendInBlue` (EnvioCorreo). Centraliza el envío de notificaciones por correo electrónico vía Brevo para todos los aplicativos del ecosistema R16.

> **Base URL:** `https://{servidor-envio-correo}/api/`
> **Autenticación:** IdentityServer (token JWT en header `Authorization: Bearer {token}`)
> **Proveedor externo:** Brevo (ex-SendInBlue) — API transaccional
> **Cola:** RabbitMQ para envíos asincrónicos; Worker.EnvioCorreo procesa la cola

---

## Contexto

Esta solución reemplaza al `_Worker.SendInBlue` y al `_Ejecutable.SincronizadorSendInBlue` existentes en ProquifaDotNet. El endpoint `PATCH /EnviarCorreo` del controller `CorreoEnviadoEnviarController` en ProquifaDotNet se **refactoriza** para delegar en `POST /api/correo/enviar` de este servicio en lugar de encolar directamente en RabbitMQ.

**Flujo de envío asincrónico:**
```
Caller (Finanzas / ProquifaDotNet)
    │
    └─ POST /api/correo/enviar | /simple | /html | /plantilla-brevo
            │
            ├─ Envío inmediato  → Brevo API → { brevoMessageId, enviado: true }
            └─ Envío encolar    → INSERT SolicitudCorreo
                                → Publica en RabbitMQ
                                → Worker.EnvioCorreo procesa
                                → Resuelve plantilla XSLT / Brevo
                                → Envía a Brevo
                                → UPDATE SolicitudCorreo + INSERT BitacoraEnvioCorreo
```

---

## Endpoints

**Controller:** `CorreoController`

### POST `/api/correo/enviar`

Solicita envío de correo basado en un registro `CorreoEnviado` existente en ProquifaDotNet. Encola en RabbitMQ y el Worker resuelve la plantilla XSLT y envía a Brevo.

| Campo | Detalle |
|-------|---------|
| **Método** | POST |
| **Ruta** | `/api/correo/enviar` |
| **Req. origen** | NO-FU-001 |

**Request body:**
```json
{
  "idCorreoEnviado": "guid",
  "idProcesoSistema": "guid"
}
```

**Response:**
```json
{
  "idSolicitudCorreo": "guid",
  "estado": "PENDIENTE | ENVIADO"
}
```

---

### POST `/api/correo/simple`

Envío simple sin asociación a `CorreoEnviado`. Para notificaciones ad-hoc internas. Usa `PlantillaCorreo` por clave lógica.

| Campo | Detalle |
|-------|---------|
| **Método** | POST |
| **Ruta** | `/api/correo/simple` |
| **Req. origen** | NO-FU-001 |

**Request body:**
```json
{
  "idRegion": "guid",
  "receptores": ["correo@dominio.com"],
  "conCopia": ["cc@dominio.com"],
  "asunto": "Asunto del correo",
  "clavePlantilla": "CLAVE_PLANTILLA",
  "parametros": [
    { "clave": "NombreCliente", "valor": "Empresa ABC" }
  ]
}
```

**Response:**
```json
{
  "brevoMessageId": "string",
  "enviado": true
}
```
O si es asincrónico:
```json
{
  "idSolicitudCorreo": "guid",
  "estado": "PENDIENTE"
}
```

---

### POST `/api/correo/html`

Envío con HTML explícito. Para correos donde el caller ya generó el HTML (ej. Finanzas genera el cuerpo del correo de factura). Soporta adjuntos vía URL de MinIO.

| Campo | Detalle |
|-------|---------|
| **Método** | POST |
| **Ruta** | `/api/correo/html` |
| **Req. origen** | NO-FU-001 |

**Request body:**
```json
{
  "idRegion": "guid",
  "receptores": ["correo@dominio.com"],
  "conCopia": ["cc@dominio.com"],
  "asunto": "Asunto del correo",
  "htmlContent": "<html>...</html>",
  "adjuntos": [
    { "nombre": "factura.pdf", "urlMinio": "https://minio/.../factura.pdf" }
  ]
}
```

**Response:**
```json
{
  "brevoMessageId": "string",
  "enviado": true
}
```

---

### POST `/api/correo/plantilla-brevo`

Envío usando plantilla nativa de Brevo (renderizada en servidores Brevo). Se referencia por `clave` lógica (mapeada en `PlantillaCorreo`) o por `idTemplateBrevo` directo.

| Campo | Detalle |
|-------|---------|
| **Método** | POST |
| **Ruta** | `/api/correo/plantilla-brevo` |
| **Req. origen** | NO-FU-001 (T11-T12) |

**Request body:**
```json
{
  "idRegion": "guid",
  "receptores": ["correo@dominio.com"],
  "conCopia": ["cc@dominio.com"],
  "clave": "CLAVE_PLANTILLA_BREVO",
  "idTemplateBrevo": 42,
  "params": {
    "NombreCliente": "Empresa ABC",
    "NumeroFactura": "F-2024-001"
  }
}
```

**Response:**
```json
{
  "brevoMessageId": "string",
  "enviado": true
}
```
O si es asincrónico:
```json
{
  "idSolicitudCorreo": "guid",
  "estado": "PENDIENTE"
}
```

---

## Endpoint de Integración en ProquifaDotNet (Refactorizado)

El endpoint existente en ProquifaDotNet que ahora delega a EnvioCorreo:

| Método | Ruta (ProquifaDotNet) | Delegado a | Descripción |
|--------|-----------------------|-----------|-------------|
| PATCH | `/EnviarCorreo` | `POST /api/correo/enviar` | Refactorizado: en lugar de encolar directamente en RabbitMQ, ahora llama al servicio EnvioCorreo |

---

## Resumen de Endpoints

| Endpoint | Tipo envío | Req. |
|---|---|---|
| `POST /api/correo/enviar` | Basado en `CorreoEnviado` existente | NO-FU-001 |
| `POST /api/correo/simple` | Ad-hoc con plantilla interna | NO-FU-001 |
| `POST /api/correo/html` | HTML explícito + adjuntos MinIO | NO-FU-001 |
| `POST /api/correo/plantilla-brevo` | Plantilla nativa Brevo | NO-FU-001 |
| **Total API EnvioCorreo** | | **4** |
