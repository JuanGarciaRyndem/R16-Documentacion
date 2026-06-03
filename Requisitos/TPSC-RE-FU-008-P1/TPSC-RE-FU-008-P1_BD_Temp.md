# Diccionario de Datos - Buzon de Cobros v2 (Mailbot .NET 10 + IA)
**Requisito:** TPSC-RE-FU-008-v2
**Base de Datos:** ProquifaDotNet
**Servidor:** RYNL010
**Version:** 1.0 - Propuesta Mailbot Worker .NET 10 con Agente IA
**Fecha:** 2026-06-02

---

## Resumen Ejecutivo
Rediseno del Mailbot actual (Framework 4.8 + Tarea Windows) a una solucion Worker en .NET 10
con lectura reactiva de correos via eventos (n8n/WebHook o Gmail Push) y clasificacion
mediante un Agente de Inteligencia Artificial flexible y expandible.
La clasificacion 'Cobro' se agrega al modelo existente (Cotizacion, Pedido, Otros).

---

## Comparativa Mailbot Actual vs Propuesta

| Aspecto             | Mailbot Actual                          | Propuesta v2                                  |
| ------------------- | --------------------------------------- | --------------------------------------------- |
| Framework           | .NET Framework 4.8                      | .NET 10 (Worker Service)                      |
| Ejecucion           | Tarea de Windows periodica              | Worker continuo reactivo                      |
| Lectura Gmail       | Polling (lectura por intervalo)         | Push/Evento (lectura al llegar)               |
| Clasificacion       | Reglas fijas en codigo                  | Agente IA entrenado y expandible              |
| Clasificaciones     | Cotizacion, Pedido, Cobro, Otros        | Cotizacion, Pedido, Cobro, Otros (extendible) |
| Adjuntos            | Guardado de archivo                     | Lectura e interpretacion por IA               |
| Datos pre-extraidos | SubtotalMailBot, TotalMailBot (parcial) | Monto, banco, cuenta, referencia (via IA)     |
| Cola de mensajes    | No                                      | RabbitMQ (Opcion 1) o Gmail Push (Opcion 2)   |
| Escalabilidad       | Limitada                                | Alta - Worker escalable horizontalmente       |

---

## Arquitectura de la Solucion Propuesta

### Capas del Worker Mailbot (.NET 10)

    MailbotWorker.sln
        Mailbot.Worker          <- Worker Service (punto de entrada, hosted services)
        Mailbot.Api             <- API REST opcional (healthcheck, reentrenamiento)
        Mailbot.Application     <- Casos de uso: clasificar, persistir, generar pendiente
        Mailbot.Domain          <- Entidades: Correo, Clasificacion, Pendiente, Regla IA
        Mailbot.Infrastructure  <- Gmail, RabbitMQ, BD (Scaffold ProquifaDotNet), IA Client
        Mailbot.Tests           <- Pruebas unitarias e integracion

### Scaffold de ProquifaDotNet (tablas relevantes)

    Tablas a incluir en el Scaffold:
        RegionConfiguracionMailBot  <- configuracion por region
        CorreoRecibido              <- correo entrante persistido
        CorreoRecibidoContenido     <- cuerpo del correo (texto/html)
        CorreoRecibidoCliente       <- clasificacion + cliente asignado
        CorreoRecibidoEstatus       <- estado lectura y procesado
        ArchivoCorreoRecibido       <- adjuntos del correo
        catClasificacionCorreoRecibido  <- catalogo de clasificaciones
        catProceso                  <- procesos del sistema
        catDominioCorreoRecibidoPendiente <- dominios en lista espera
        fccFolioPagoCliente         <- pendiente Validar Cobro (clasificacion Cobro)
        Region                      <- segregacion MEX/PER

---

## Opciones de Lectura Reactiva de Gmail

### Opcion 1 - n8n + RabbitMQ (WebHook intermediario)

    Gmail -> n8n (trigger por correo) -> publica evento en RabbitMQ -> Worker consume cola

| Aspecto | Detalle |
|---------|---------|
| Herramienta | n8n (orquestador no-code/low-code) |
| Cola | RabbitMQ - exchange/queue por region (MEX/PER) |
| Ventaja | Desacoplamiento total; n8n gestiona reintentos y filtros |
| Desventaja | Dependencia de n8n; infraestructura adicional (RabbitMQ) |
| Latencia | Baja (~segundos segun polling de n8n o webhook Gmail) |

### Opcion 2 - Gmail Push Notifications (Pub/Sub Google)

    Gmail -> Google Cloud Pub/Sub -> endpoint HTTP del Worker API -> Worker procesa

| Aspecto     | Detalle                                                                    |
| ----------- | -------------------------------------------------------------------------- |
| Herramienta | Gmail API (watch) + Google Cloud Pub/Sub                                   |
| Ventaja     | Nativo de Google; sin infraestructura adicional de cola                    |
| Desventaja  | Requiere cuenta GCP; el watch expira cada 7 dias (renovar automaticamente) |
| Latencia    | Muy baja (~1-2 segundos)                                                   |

---

## Agente de Inteligencia Artificial - Clasificador

### Responsabilidades del Agente

| Responsabilidad | Descripcion |
|-----------------|-------------|
| Clasificacion | Determina si el correo es Cotizacion, Pedido, Cobro u Otros |
| Extraccion de datos | Lee asunto, cuerpo texto, cuerpo HTML y adjuntos (PDF/imagen) |
| Identificacion de cliente | Infiere el cliente desde dominio emisor, datos del correo o adjunto |
| Pre-extraccion datos cobro | Monto, banco emisor, cuenta origen, fecha pago, referencia |
| Expandibilidad | Nuevas clasificaciones sin recompilar - via reentrenamiento del agente |

### Datos que el Agente puede pre-extraer para Cobros

| Dato | Campo BD destino | Tabla |
|------|-----------------|-------|
| Monto total | TotalMailBot | fccFolioPagoCliente |
| Subtotal | SubtotalMailBot | fccFolioPagoCliente |
| IVA | IvaMailBot | fccFolioPagoCliente |
| Moneda MXN | MxnMailBot | fccFolioPagoCliente |
| Moneda USD | UsdMailBot | fccFolioPagoCliente |
| Banco emisor | IdCatBanco | fccPagoCliente |
| Cuenta origen | CuentaOrdenante | fccPagoCliente |
| Referencia bancaria | ReferenciaBancaria | fccPagoCliente |
| Fecha del pago | FechaPago | fccPagoCliente |

### Diseño del Agente (flexible y expandible)

    Mailbot.Domain
        IClasificadorAgente.cs       <- interfaz del agente
        ClasificacionResultado.cs    <- resultado: clasificacion + datos extraidos + confianza
        ReglaClasificacion.cs        <- entidad de regla configurable

    Mailbot.Infrastructure
        AgentesIA/
            OpenAIClasificadorAgente.cs  <- implementacion con OpenAI/Azure OpenAI
            ClasificadorConfig.cs        <- configuracion: modelo, temperatura, prompt base
        Prompts/
            clasificacion_correo.txt     <- prompt base editable sin recompilar
            extraccion_cobro.txt         <- prompt extraccion datos cobro

---

## Modelo de Datos BD - Tablas Involucradas

### Flujo de persistencia del correo

    [Gmail / Cola RabbitMQ / Pub/Sub]
        -> Worker recibe evento
        -> Lee correo via Gmail API
        -> INSERT CorreoRecibidoContenido  (cuerpo texto y HTML)
        -> INSERT CorreoRecibido           (cabecera + IdRegion + IdCorreoRecibidoContenido)
        -> INSERT ArchivoCorreoRecibido    (por cada adjunto)
        -> Agente IA clasifica
        -> INSERT CorreoRecibidoCliente    (IdCatClasificacionCorreoRecibido + IdCliente)
        -> INSERT CorreoRecibidoEstatus    (estado inicial: Leido=0, Procesado=0)
        -> SI clasificacion = 'cobro':
               INSERT fccFolioPagoCliente  (pendiente en Validar Cobro con datos IA)

---

## 1. RegionConfiguracionMailBot
**Proposito:** Configuracion del Worker por region. Define la cuenta Gmail y etiquetas.
**Sin cambios estructurales en v2 - se reutiliza con el nuevo Worker.**

| Columna | Tipo | Longitud | Nulo | Descripcion |
|---------|------|----------|------|-------------|
| IdRegionConfiguracionMailBot | uniqueidentifier | 16 | NO | PK |
| IdRegion | uniqueidentifier | 16 | NO | FK - Region (MEX/PER) |
| ApplicationName | varchar | 100 | NO | Nombre de la app Gmail |
| Scopes | varchar | 500 | NO | Permisos OAuth2 Gmail |
| CorreoElectronico | varchar | 150 | NO | Cuenta monitoreada por el Worker |
| EtiquetasIds | varchar | MAX | SI | IDs de etiquetas Gmail a procesar |
| EtiquetasExcluidas | varchar | 500 | SI | Etiquetas a ignorar |
| TipoClienteOrigen | varchar | 500 | NO | Tipos de cliente origen aceptados |
| credPath | varchar | 500 | NO | Ruta credenciales OAuth2 |
| Activo | bit | 1 | NO | Configuracion activa |

> Para Opcion 2 (Gmail Push): agregar columna PubSubTopic varchar(300) para
> almacenar el topic de Google Cloud Pub/Sub por region.

---

## 2. CorreoRecibido
**Proposito:** Cabecera del correo recibido. IdRegion segrega MEX/PER.

| Columna | Tipo | Longitud | Nulo | Descripcion |
|---------|------|----------|------|-------------|
| IdCorreoRecibido | uniqueidentifier | 16 | NO | PK |
| IdCorreoRecibidoContenido | uniqueidentifier | 16 | NO | FK - cuerpo del correo |
| Asunto | varchar | 350 | SI | Asunto del correo |
| CorreoEmisor | varchar | 180 | NO | Remitente |
| CorreosReceptores | varchar | 800 | SI | Destinatarios |
| IdentificadorCorreo | varchar | 180 | SI | ID unico del mensaje en Gmail |
| FechaRecepcion | datetime | 8 | NO | Fecha de recepcion |
| IdRegion | uniqueidentifier | 16 | NO | FK - Region (MEX/PER) |
| Activo | bit | 1 | NO | Default: 1 |

---

## 3. CorreoRecibidoContenido
**Proposito:** Cuerpo del correo. Alimenta al Agente IA para clasificacion y extraccion.

| Columna | Tipo | Longitud | Nulo | Descripcion |
|---------|------|----------|------|-------------|
| IdCorreoRecibidoContenido | uniqueidentifier | 16 | NO | PK |
| Contenido | varchar | MAX | SI | Texto plano del cuerpo - entrada principal al Agente IA |
| ContenidoHtml | varchar | MAX | SI | HTML del cuerpo - entrada secundaria al Agente IA |

> El Agente IA usa Contenido y ContenidoHtml junto con los adjuntos (ArchivoCorreoRecibido)
> para determinar clasificacion y extraer datos del cobro.

---

## 4. ArchivoCorreoRecibido
**Proposito:** Adjuntos del correo. El Agente IA los lee para extraccion de datos.

| Columna | Tipo | Longitud | Nulo | Descripcion |
|---------|------|----------|------|-------------|
| IdArchivoCorreoRecibido | uniqueidentifier | 16 | NO | PK |
| IdCorreoRecibido | uniqueidentifier | 16 | NO | FK - CorreoRecibido |
| IdArchivo | uniqueidentifier | 16 | NO | FK - Archivo (PDF/imagen del comprobante) |
| Mostrar | bit | 1 | NO | Visible en pantalla |
| Activo | bit | 1 | NO | Default: 1 |

> Para cobros: el Agente IA procesa el PDF/imagen del comprobante de pago
> para extraer monto, banco, cuenta origen, referencia bancaria y fecha.

---

## 5. CorreoRecibidoCliente
**Proposito:** Vincula el correo con el cliente y la clasificacion del Agente IA.
**Tabla central del Buzon de Cobros.**

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdCorreoRecibidoCliente | uniqueidentifier | NO | PK |
| IdCatClasificacionCorreoRecibido | uniqueidentifier | NO | FK - clasificacion (cobro/pedido/etc) |
| IdCorreoRecibido | uniqueidentifier | NO | FK - CorreoRecibido |
| IdCliente | uniqueidentifier | SI | FK - Cliente identificado por IA (nullable si no identificado) |
| IdUsuario | uniqueidentifier | SI | FK - Gestor de Cobranza asignado |
| Leido | bit | NO | 0=No leido. Default: 0 |
| Procesado | bit | NO | 0=Pendiente. Default: 0 |
| Activo | bit | NO | 1=En buzon / 0=Reclasificado o cerrado |

**Reclasificacion manual:** UPDATE IdCatClasificacionCorreoRecibido al buzon destino.

---

## 6. catClasificacionCorreoRecibido
**Proposito:** Catalogo de clasificaciones del Agente IA. Define que rol ve cada tipo.
**Cambio v2:** Agregar o renombrar clasificacion 'Cobro' (actualmente 'Pago').

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| IdCatClasificacionCorreoRecibido | uniqueidentifier | PK |
| ClasificacionCorreoRecibido | varchar(180) | Nombre visible |
| Clave | varchar(150) | **Usada por el Agente IA para retornar la clasificacion** |
| AnalistaDeCuentasPorCobrar | bit | **1 = visible en Buzon de Cobros** |
| ClasificacionDefault | bit | Clasificacion por defecto (Otros=1) |
| Activo | bit | Clasificacion activa |

**Estado actual y cambio requerido:**

| Clasificacion | Clave | AnalistaDeCuentasPorCobrar | Estado |
|---------------|-------|---------------------------|--------|
| Pago | pago | SI | Existente - renombrar a 'Cobro'/'cobro' |
| Pedido | pedido | No | Existente - sin cambio |
| Requisicion | requisicion | No | Existente - sin cambio |
| Solicitud de informacion | solicituddeinformacion | No | Existente - sin cambio |
| Queja | queja | No | Existente - sin cambio |
| Sugerencia | sugerencia | No | Existente - sin cambio |
| Otros | otros | No | Existente - sin cambio |

**Script:**

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    UPDATE dbo.catClasificacionCorreoRecibido
    SET    ClasificacionCorreoRecibido = 'Cobro',
           Clave                       = 'cobro',
           FechaUltimaActualizacion    = GETDATE()
    WHERE  Clave = 'pago';

---

## 7. fccFolioPagoCliente
**Proposito:** Pendiente generado automaticamente en Validar Cobro.
**En v2 el Agente IA pre-rellena los campos MailBot con mayor precision.**

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdFCCFolioPagoCliente | uniqueidentifier | NO | PK |
| IdCorreoRecibidoCliente | uniqueidentifier | NO | FK - origen en el buzon |
| IdArchivo | uniqueidentifier | SI | Adjunto del comprobante |
| Folio | varchar(80) | SI | Folio generado |
| FechaRecepcion | datetime | NO | Fecha del correo |
| Stp | bit | NO | Pago via STP |
| **SubtotalMailBot** | **decimal** | **SI** | **Pre-extraido por Agente IA** |
| **IvaMailBot** | **decimal** | **SI** | **Pre-extraido por Agente IA** |
| **TotalMailBot** | **decimal** | **SI** | **Pre-extraido por Agente IA** |
| **MxnMailBot** | **bit** | **SI** | **Moneda MXN detectada por Agente IA** |
| **UsdMailBot** | **bit** | **SI** | **Moneda USD detectada por Agente IA** |
| Activo | bit | NO | 1=Pendiente activo / 0=Cerrado o inconsistencia |

---

## 8. CorreoRecibidoEstatus
**Proposito:** Historial de estados del correo. Sin cambios estructurales en v2.

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdCorreoRecibidoEstatus | uniqueidentifier | NO | PK |
| IdCorreoRecibido | uniqueidentifier | NO | FK - CorreoRecibido |
| IdCatClasificacionCorreoRecibido | uniqueidentifier | NO | FK - clasificacion aplicada |
| IdUsuario | uniqueidentifier | SI | Usuario que realizo el cambio |
| Leido | bit | SI | Estado de lectura |
| FechaLeido | datetime | SI | Fecha de lectura |
| Procesado | bit | SI | Estado de procesado |
| FechaProcesado | datetime | SI | Fecha de procesado |
| Activo | bit | NO | Registro activo |

---

## 9. Tablas Nuevas Propuestas para v2

### 9.1 MailbotEventoCorreo (NUEVA - recomendada)
**Proposito:** Cola interna de eventos de correo recibidos pendientes de procesar.
**Alternativa liviana a RabbitMQ para persistir eventos antes de procesarlos.**

| Columna Propuesta | Tipo | Descripcion |
|-------------------|------|-------------|
| IdMailbotEventoCorreo | uniqueidentifier | PK |
| IdRegion | uniqueidentifier | FK - Region (MEX/PER) |
| IdentificadorCorreoGmail | varchar(200) | ID del mensaje en Gmail |
| FechaEvento | datetime | Fecha de recepcion del evento |
| Procesado | bit | 0=Pendiente / 1=Procesado |
| FechaProcesado | datetime | Fecha de procesamiento |
| Intentos | int | Contador de reintentos |
| Error | varchar(MAX) | Detalle del error si fallo |
| FechaRegistro | datetime | Default: GETDATE() |
| Activo | bit | Default: 1 |

### 9.2 MailbotClasificacionLog (NUEVA - recomendada)
**Proposito:** Auditoria de clasificaciones del Agente IA para reentrenamiento.

| Columna Propuesta | Tipo | Descripcion |
|-------------------|------|-------------|
| IdMailbotClasificacionLog | uniqueidentifier | PK |
| IdCorreoRecibido | uniqueidentifier | FK - CorreoRecibido |
| ClasificacionIA | varchar(150) | Clasificacion propuesta por el Agente IA |
| Confianza | decimal(5,2) | Score de confianza del Agente (0.00-1.00) |
| ClasificacionFinal | varchar(150) | Clasificacion definitiva (puede diferir si usuario reclasifica) |
| ModeloVersion | varchar(50) | Version del modelo IA usado |
| FechaRegistro | datetime | Default: GETDATE() |
| Activo | bit | Default: 1 |

---

## Flujo Completo v2

    1. EVENTO: Gmail recibe correo
       - Opcion 1: n8n detecta -> publica en RabbitMQ -> Worker consume
       - Opcion 2: Gmail Pub/Sub notifica -> Worker API endpoint -> Worker procesa

    2. WORKER: Lee correo completo via Gmail API
       - Cabecera: asunto, emisor, fecha
       - Cuerpo: texto plano + HTML
       - Adjuntos: PDF / imagen del comprobante

    3. PERSISTENCIA INICIAL:
       INSERT CorreoRecibidoContenido (Contenido, ContenidoHtml)
       INSERT CorreoRecibido (IdRegion, IdentificadorCorreo, ...)
       INSERT ArchivoCorreoRecibido (por cada adjunto)
       INSERT MailbotEventoCorreo (Procesado=0)

    4. AGENTE IA CLASIFICA Y EXTRAE:
       - Entrada: asunto + Contenido + ContenidoHtml + adjuntos
       - Salida: { clasificacion, confianza, cliente?, datos_cobro? }
       INSERT MailbotClasificacionLog

    5. PERSISTENCIA SEGUN CLASIFICACION:
       INSERT CorreoRecibidoCliente (IdCatClasificacionCorreoRecibido, IdCliente)
       INSERT CorreoRecibidoEstatus

       SI clasificacion = 'cobro':
           INSERT fccFolioPagoCliente (SubtotalMailBot, IvaMailBot, TotalMailBot, MxnMailBot, UsdMailBot)

       UPDATE MailbotEventoCorreo SET Procesado=1

---

## Ciclo de Vida del Pendiente en Buzon de Cobros

| Estado | Trigger | Accion en BD |
|--------|---------|--------------|
| Aparece | Agente IA clasifica como 'cobro' | INSERT fccFolioPagoCliente (Activo=1) |
| Cierre | Cobro vinculado a proforma/factura en Validar Cobro | UPDATE fccFolioPagoCliente SET Activo=0 |
| Eliminacion | Cobro marcado como inconsistencia | UPDATE fccFolioPagoCliente SET Activo=0 |
| Reclasificacion | Gestor mueve a otro buzon | UPDATE CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido |

---

## Cambios BD Requeridos

| # | Cambio | Script | Prioridad |
|---|--------|--------|-----------|
| 1 | Renombrar 'Pago' a 'Cobro' en catClasificacionCorreoRecibido | Ver Seccion 6 | Alta |
| 2 | INSERT proceso 'Cobros' en catProceso | Script insert | Alta |
| 3 | CREATE TABLE MailbotEventoCorreo | Script nueva tabla | Alta |
| 4 | CREATE TABLE MailbotClasificacionLog | Script nueva tabla | Alta |
| 5 | Agregar PubSubTopic a RegionConfiguracionMailBot (Opcion 2 Gmail Push) | ALTER TABLE | Media |

---

## Consideraciones de Desarrollo

| Tema | Detalle |
|------|---------|
| Scaffold | Usar EF Core con db-first sobre ProquifaDotNet con las tablas de Seccion Scaffold |
| Reintentos | MailbotEventoCorreo.Intentos con max 3 reintentos y backoff exponencial |
| Reentrenamiento | MailbotClasificacionLog alimenta dataset de reentrenamiento del Agente |
| Expandibilidad | Nueva clasificacion = nuevo registro en catClasificacionCorreoRecibido + ajuste de prompt |
| Salud | Mailbot.Api expone /health y /metrics para monitoreo |
| Segregacion | CorreoRecibido.IdRegion + RegionConfiguracionMailBot.IdRegion garantizan MEX/PER |

---

## Gaps y Pendientes

| # | Gap | Descripcion | Accion |
|---|-----|-------------|--------|
| 1 | Decision opcion lectura | Opcion 1 (n8n+RabbitMQ) vs Opcion 2 (Gmail Push) | Validar con arquitectura e infraestructura |
| 2 | Modelo IA a usar | OpenAI, Azure OpenAI, modelo propio | Definir segun costo y privacidad de datos |
| 3 | Cliente no identificado | IdCliente nullable - quien atiende el correo | Confirmar con cliente |
| 4 | Clasificacion 'Pago' en uso | Verificar si 'pago' tiene registros activos antes de renombrar | Ejecutar SELECT COUNT antes del UPDATE |
| 5 | Prompt base del Agente | Texto del prompt de clasificacion y extraccion | Redactar con equipo de negocio |
| 6 | Clasificacion Peru | Aplicar misma mecanica para clientes PER | Sin cambio adicional - IdRegion ya segrega |

---

**Generado por:** GitHub Copilot in SSMS
**Version:** 1.0
**Base de Datos:** ProquifaDotNet
**Referencia:** TPSC-RE-FU-008 / TPSC-RE-FU-008 Revision
