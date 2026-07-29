# **Estándar de Proxy hacia ProquifaDotNet** 

## Estándar de Proxy hacia ProquifaDotNet (*Anti-Corruption Layer*)

| FORMATO | N/A |
| :---- | :---- |
| **PROYECTO** | R16 \- Adquisiciones |
| **REFERENCIA** | [R16A-132: R16A-RE-FU-016 - \[DIS\] - Diseño de la solución Parte 1](https://newryndem.atlassian.net/browse/R16A-132) |
| **VERSIÓN** | 1.3 |
| **FECHA** | 6 jul 2026 |
| **AUTOR** | [A. Javier Antúnez Estrada](mailto:agustin.antunez@ryndem.mx) |
| **REVISOR** | [Juan David García Cruz](mailto:juan.garcia@ryndem.mx) |

## 

## Contenido

[**1\. Introducción	3**](#1.-introducción)

[1.1 Alternativas consideradas: ¿por qué no se comparó contra otros patrones arquitectónicos?	4](#1.1-alternativas-consideradas:-¿por-qué-no-se-comparó-contra-otros-patrones-arquitectónicos?)

[1.2 Alcance: ¿a qué sistemas externos aplica este estándar?	4](#1.2-alcance:-¿a-qué-sistemas-externos-aplica-este-estándar?)

[**2\. Patrón arquitectónico: Anti-Corruption Layer	6**](#2.-patrón-arquitectónico:-anti-corruption-layer)

[**3\. Mecanismo técnico	7**](#3.-mecanismo-técnico)

[3.1 Typed HttpClient vía IHttpClientFactory	7](#3.1-typed-httpclient-vía-ihttpclientfactory)

[3.2 Resiliencia (reintentos, timeouts, circuit breaker)	7](#3.2-resiliencia-\(reintentos,-timeouts,-circuit-breaker\))

[3.3 Manejo de errores	8](#3.3-manejo-de-errores)

[3.4 Autenticación (ROPC con cuenta de servicio)	8](#3.4-autenticación-\(ropc-con-cuenta-de-servicio\))

[**4\. Ubicación en el código	10**](#4.-ubicación-en-el-código)

[4.1 Incorporación de un nuevo sistema legacy	10](#4.1-incorporación-de-un-nuevo-sistema-legacy)

[**5\. Pendientes	12**](#5.-pendientes)

[**6\. Control de versiones	13**](#6.-control-de-versiones)

# **1\. Introducción** {#1.-introducción}

Todos los módulos de ProquifaDotNet.Finanzas (Proforma MX/PE, FAA MX/PE, etc.) necesitan, en algún momento, comunicarse con el sistema **ProquifaDotNet** para operaciones que no les corresponde implementar directamente — el caso inicial (que pertenece al requisito 016\) es la subida/descarga de archivos binarios (PDFs), que hoy vive en los endpoints SubirArchivo (UploadArchivoController) y DescargarArchivoBytes (ArchivoExtensionsController) de la API Catálogos de ProquifaDotNet, y que internamente resuelven contra MinIO.

Este estándar define **cómo** debe construirse cualquier proxy de este tipo, para que:

* No se duplique la lógica de comunicación HTTP, autenticación y reintentos en cada módulo.  
* ProquifaDotNet.Finanzas no quede acoplado a los detalles internos de ProquifaDotNet (nombres en español, estructuras legacy, mecanismo de almacenamiento).  
* El criterio sea consistente con los principios de diseño de Finanzas ya establecidos: idempotencia, reintentos con estrategia definida, manejo de errores estandarizado, independencia entre soluciones.

Requisito que originó este estándar: **R16A-RE-FU-016 (Proforma México)**, primer consumidor del proxy hacia SubirArchivo/DescargarArchivoBytes.

## **1.1 Alternativas consideradas: ¿por qué no se comparó contra otros patrones arquitectónicos?** {#1.1-alternativas-consideradas:-¿por-qué-no-se-comparó-contra-otros-patrones-arquitectónicos?}

No se realizó una investigación de mercado ni una comparativa formal contra otros patrones arquitectónicos (por ejemplo, *API Gateway*, *Backend for Frontend*, *Strangler Fig*, o un cliente HTTP directo sin capa de traducción) antes de adoptar *Anti-Corruption Layer*. El motivo fué:

* ACL es el patrón de DDD documentado específicamente para este problema: aislar un dominio nuevo (Finanzas) de un modelo legacy ajeno (ProquifaDotNet, en español, con sus propias convenciones). No es una elección entre arquitecturas equivalentes, sino la aplicación directa del patrón reconocido para este escenario exacto (Azure Architecture Center, [microservices.io](http://microservices.io/)).  
* Los patrones alternativos resuelven problemas distintos y no aplican al caso: API Gateway y BFF atienden enrutamiento/composición de múltiples servicios o agregación para un frontend específico; Strangler Fig atiende una migración incremental de un sistema completo. Aquí no hay múltiples servicios que enrutar ni una migración en curso — solo la necesidad de traducir un modelo de dominio legacy hacia uno nuevo.  
* A nivel de mecanismo técnico (ya dentro del patrón ACL elegido), tampoco se comparó mercado para la librería de resiliencia: se adoptó Microsoft.Extensions.Http.Resilience por ser la sucesora oficial de Microsoft.Extensions.Http.Polly en .NET 8+, recomendada por Microsoft para Typed HttpClient \+ IHttpClientFactory (ver sección 3.2) — no hay alternativa madura de mercado que compita en este espacio para .NET sin reintroducir Polly manualmente.

## **1.2 Alcance: ¿a qué sistemas externos aplica este estándar?** {#1.2-alcance:-¿a-qué-sistemas-externos-aplica-este-estándar?}

Este estándar (patrón Anti-Corruption Layer \+ librería ProquifaDotNetProxy) **aplica exclusivamente a sistemas legacy** con un modelo de dominio ajeno al de Finanzas — hoy, **ProquifaDotNet**. La razón de ser del ACL es traducir entre dos modelos de dominio distintos (legacy en español vs. Finanzas en inglés); si no hay traducción de modelo que hacer, no hay corrupción que prevenir y el patrón no aporta valor, solo burocracia.

**No aplica** a sistemas nuevos o "hermanos" que ya hablan el modelo de Finanzas o tienen su propio contrato moderno — por ejemplo:

• **DocumentBuilder**: recibe directamente ProformaDocumentDto (el DTO propio de Finanzas, en inglés). No hay traducción de dominio que hacer. Se consume mediante su propio Typed HttpClient (DocumentBuilderHttpClient, ver DIS-SOL de RE-FU-016 sección 5.4), **sin** pasar por esta librería de proxy ni por el framing de ACL.  
• **Microservicio de Bitácoras** (futuro): mismo criterio — al ser un sistema nuevo del propio ecosistema R16, su cliente HTTP se implementa de forma independiente, no a través de ProquifaDotNetProxy.

**Lo que SÍ se comparte con estos sistemas "no legacy"** es únicamente la **base técnica** de la sección "Mecanismo técnico" (Typed Client vía IHttpClientFactory, resiliencia con Microsoft.Extensions.Http.Resilience, excepción propia por cliente) — esa parte es una buena práctica general de comunicación HTTP saliente y se recomienda replicarla en cualquier cliente nuevo, pero cada sistema define su propio Typed Client independiente (DocumentBuilderHttpClient, BitacorasHttpClient, etc.), no uno compartido dentro de esta librería.

**Regla de decisión rápida:** ¿el sistema externo tiene un modelo de dominio propio y ajeno al de Finanzas que hay que traducir? → usar este estándar (ACL \+ librería compartida). ¿El sistema ya consume/produce el modelo de Finanzas o es parte del mismo ecosistema R16 nuevo? → Typed Client propio del módulo, sin ACL ni librería compartida.

# 

# **2\. Patrón arquitectónico: Anti-Corruption Layer** {#2.-patrón-arquitectónico:-anti-corruption-layer}

Se adopta el patrón **Anti-Corruption Layer (ACL)**, de Domain-Driven Design y documentado en el [Azure Architecture Center de Microsoft](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer) y en [microservices.io](https://microservices.io/patterns/refactoring/anti-corruption-layer.html).

**Motivo de la elección:** ProquifaDotNet es un sistema legacy con su propio modelo de datos, idioma (español) y convenciones (SubirArchivo, DescargarArchivoBytes, objeto Archivo). ProquifaDotNet.Finanzas es un sistema nuevo con sus propias convenciones (Clean Architecture, inglés, DTOs propios). El ACL crea una capa de traducción entre ambos modelos, evitando que los conceptos legacy se filtren al dominio nuevo.

**Regla general:** ningún módulo de Finanzas debe invocar directamente un endpoint de ProquifaDotNet desde su capa de Application o Domain. Toda comunicación pasa por una interfaz propia (en inglés, con el modelo de Finanzas) implementada en la capa Infrastructure, que internamente traduce hacia el contrato legacy.

# 

# **3\. Mecanismo técnico** {#3.-mecanismo-técnico}

## **3.1 Typed HttpClient vía IHttpClientFactory** {#3.1-typed-httpclient-vía-ihttpclientfactory}

A diferencia del patrón usado en venta-digital-backend (R14) — un IApiCallerProxy genérico con métodos Get/Post/Put/Delete parametrizados por string y un enum ProxyApiSelector — en Finanzas se usa un **Typed Client** por cada proxy, siguiendo la recomendación vigente de Microsoft ([HttpClient guidelines for .NET](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines)):

| public interface IProquifaDotNetArchivoProxy{    Task\<ArchivoDto\> SubirArchivoAsync(string bucketName, byte\[\] content, string fileName, CancellationToken ct \= default);    Task\<byte\[\]\> DescargarArchivoBytesAsync(Guid idArchivo, CancellationToken ct \= default);}public class ProquifaDotNetArchivoProxy : IProquifaDotNetArchivoProxy{    private readonly HttpClient \_httpClient;    *// ...*} |
| :---- |

Registro en Program.cs / Startup.cs:

| services.AddHttpClient\<IProquifaDotNetArchivoProxy, ProquifaDotNetArchivoProxy\>(client \=\>{    client.BaseAddress \= new Uri(configuration\["ProquifaDotNet:CatalogosBaseUrl"\]);}).AddStandardResilienceHandler(); *// ver sección siguiente* |
| :---- |

**Por qué Typed Client y no un caller genérico:** el Typed Client da IntelliSense, tipado fuerte y un punto único de configuración por proxy, evitando el anti-patrón de pasar nombres de endpoint como strings sueltos ("SubirArchivo", "DescargarArchivoBytes") por todo el código, como ocurre en el IApiCallerProxy legacy.

## **3.2 Resiliencia (reintentos, timeouts, circuit breaker)** {#3.2-resiliencia-(reintentos,-timeouts,-circuit-breaker)}

Se usa **Microsoft.Extensions.Http.Resilience**, sucesor de Microsoft.Extensions.Http.Polly para .NET 8+, construido sobre Polly v8 ([Build resilient HTTP apps — .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience)):

* Retry con backoff exponencial \+ **jitter** (evita que múltiples instancias reintenten al mismo tiempo tras una caída — [\[Implement HTTP call retries with exponential backoff with Polly](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/implement-http-call-retries-exponential-backoff-polly).  
* Timeout por intento, distinto del timeout total del HttpClient.  
* Circuit breaker para evitar seguir insistiendo si ProquifaDotNet está caído.

⚠️ **Precaución con idempotencia (aplicar caso por caso):** un retry automático sobre una operación no idempotente puede duplicar el efecto. SubirArchivoAsync debe revisarse: si dos intentos exitosos pero con timeout en la respuesta generan dos archivos distintos en MinIO, el retry produciría duplicados. Cada módulo consumidor debe validar este comportamiento contra el endpoint real de ProquifaDotNet antes de habilitar retry automático en operaciones de escritura.

*Ver sección 1.1 para la justificación de por qué no se comparó contra otras librerías de resiliencia.*

## **3.3 Manejo de errores** {#3.3-manejo-de-errores}

Excepción propia (ProquifaDotNetProxyException) para encapsular fallas de comunicación; el módulo consumidor la traduce al código HTTP que le corresponda (en RE-FU-016: 502 Bad Gateway). No se propaga la excepción cruda de HttpClient/Polly hacia capas superiores.

## **3.4 Autenticación (ROPC con cuenta de servicio)** {#3.4-autenticación-(ropc-con-cuenta-de-servicio)}

**Investigado en el código fuente de ProquifaDotNet-R14** (WebApi.Catalogos/App\_Start/WebApiConfig.cs, Core.Pqf.WebApi.AccessOperations.AuthorizeProquifaDotNet, Core.Pqf.BusinessBasicTools.IdentityServer4.ProquifaDotNetAccessManager):

* Todos los controllers de WebApi.Catalogos (incluidos ArchivoExtensionsController y UploadArchivoController, dueños de `DescargarArchivoBytes` y SubirArchivo) están protegidos por un filtro **global** (config.Filters.Add(new AuthorizeProquifaDotNet())), no por \[Authorize\] a nivel de acción.  
* El filtro valida el token contra IdentityServer4 y **exige que el token traiga roles** (ProquifaDotNetAccessManager.ValidateToken → tokenRequester.ObtenerRoles(Token); si no hay roles, lanza "Acceso no autorizado"). Los roles en IdentityServer4 se asocian a un **usuario**, no a un cliente.  
* Por eso, un token client\_credentials puro (M2M) **no es compatible tal cual** con este filtro: no trae un usuario ni roles asociados.  
* El consumidor legacy que ya funciona contra este mismo filtro (venta-digital-backend, clase TokenAsynProquifaDotNet \+ RequestNewToken) usa **grant\_type=password** (Resource Owner Password Credentials, ROPC) con una **cuenta de servicio** (username/password de configuración) que sí tiene roles asignados en IdentityServer4.

**Decisión (2026-07-01):** ProquifaDotNetArchivoProxy replica este mismo patrón ROPC — obtiene un token vía grant\_type=password con una cuenta de servicio dedicada para ProquifaDotNet.Finanzas, cachea el token hasta su expiración y lo adjunta como `Authorization: Bearer`. No se persigue client\_credentials/M2M puro porque requeriría modificar AuthorizeProquifaDotNet/ProquifaDotNetAccessManager en ProquifaDotNet (fuera del alcance de Finanzas) para que un cliente sin usuario pueda tener roles.

**Diferencia con la implementación legacy:** se reimplementa en la librería compartida como parte del DelegatingHandler/HttpMessageHandler del Typed Client (no como un wrapper manual de HttpClient como en R14), para que el refresco de token sea transparente a ProquifaDotNetArchivoProxy y a futuros proxies de la misma librería.

# 

# **4\. Ubicación en el código** {#4.-ubicación-en-el-código}

Librería compartida, referenciada por todos los módulos de Finanzas que necesiten hablar con ProquifaDotNet:

| ProquifaDotNet.Finanzas.Infrastructure.ProquifaDotNetProxy/├── IProquifaDotNetArchivoProxy.cs├── ProquifaDotNetArchivoProxy.cs├── Dto/│   └── ArchivoDto.cs└── ProquifaDotNetProxyException.cs |
| :---- |

Cada nuevo recurso de ProquifaDotNet que se necesite consumir (no solo Archivo) agrega su propia interfaz IProquifaDotNet`<Recurso>Proxy` dentro de esta misma librería, reutilizando el HttpClient tipado y la configuración de resiliencia.

## **4.1 Incorporación de un nuevo sistema legacy** {#4.1-incorporación-de-un-nuevo-sistema-legacy}

Si en el futuro aparece otro sistema **legacy** (según el criterio de la sección "Alcance": modelo de dominio propio y ajeno al de Finanzas), se crea una **librería nueva independiente**, no se agrega dentro de ProquifaDotNetProxy:

| ProquifaDotNet.Finanzas.Infrastructure.\<NombreSistemaLegacy\>Proxy/├── I\<NombreSistemaLegacy\>\<Recurso\>Proxy.cs├── \<NombreSistemaLegacy\>\<Recurso\>Proxy.cs├── Dto/└── \<NombreSistemaLegacy\>ProxyException.cs |
| :---- |

**Por qué una librería separada por sistema, y no una librería "genérica" compartida entre legacies:**  
**Aislamiento de dominio:** cada ACL traduce un modelo de dominio legacy distinto. Mezclar dos sistemas legacy en una sola librería reintroduce acoplamiento entre ellos — un cambio de contrato en SistemaX no debería poder romper algo de ProquifaDotNet.

* **Autenticación independiente:** no se puede asumir que todo sistema legacy futuro use ROPC como ProquifaDotNet. Podría usar API Key, client\_credentials real, certificados, Basic Auth, etc. Cada librería configura su propio mecanismo de auth sin condicionar a las demás.  
* **Configuración independiente:** cada sistema tiene su propia base URL, timeouts y política de reintentos (algunos endpoints legacy pueden ser más lentos o menos confiables que otros).

**Lo que SÍ se reutiliza** entre las distintas librerías de proxy hacia sistemas legacy es el **patrón** descrito en "Mecanismo técnico" (Typed Client vía IHttpClientFactory \+ Microsoft.Extensions.Http.Resilience \+ excepción propia por librería) — se replica la misma receta en cada una, no se comparte una implementación común forzada. Si con el tiempo aparece lógica realmente idéntica entre dos o más proxies legacy (por ejemplo, un helper de configuración de resiliencia), se extrae a un paquete técnico de bajo nivel (ej. ProquifaDotNet.Finanzas.Infrastructure.Http.Resilience), pero nunca se hace que una librería de proxy dependa del modelo de otra.

# 

# **5\. Pendientes** {#5.-pendientes}

| \# | Pendiente | Impacto |
| :---- | :---- | :---- |
| 1 | **Mecanismo de autenticación resuelto (ROPC), falta el aprovisionamiento:** crear la cuenta de servicio en IdentityServer4/ProquifaDotNet para ProquifaDotNet.Finanzas (usuario \+ roles necesarios para SubirArchivo/DescargarArchivoBytes), y obtener client\_id/username/password/scope para configurar el proxy. | Medio |
| 2 | Confirmar con el equipo de ProquifaDotNet si SubirArchivo es idempotente ante reintentos (ver nota de la sección Resiliencia). | Medio |

# 

# **6\. Control de versiones** {#6.-control-de-versiones}

| Versión | Fecha | Autor | Tipo de Cambio | Descripción del cambio | Aprobó |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 1.0 | 01/07/2026 | Javier Antúnez | Creación | Estándar inicial, originado en el diseño de RE-FU-016 (Proforma México) | Juan David |
| 1.1 | 01/07/2026 | Javier Antúnez | Aclaración de alcance | Agregada sección "Alcance": el patrón ACL aplica solo a sistemas legacy (ProquifaDotNet), no a sistemas nuevos/hermanos como DocumentBuilder o el futuro microservicio de Bitácoras, que usan su propio Typed HttpClient sin pasar por esta librería. |  |
| 1.2 | 01/07/2026 | Javier Antúnez | Autenticación definida | Investigado el código fuente de `WebApi.Catalogos` (ProquifaDotNet-R14): los endpoints están protegidos por un filtro global que exige roles de usuario, incompatible con client\_credentials puro. Se decide replicar el patrón ROPC (grant\_type=password con cuenta de servicio) usado por el consumidor legacy `venta-digital-backend`. Pendiente aprovisionar la cuenta. |  |
| 1.3 | 01/07/2026 | Javier Antúnez | Generalización | Agregada sección "Incorporación de un nuevo sistema legacy": cada sistema legacy futuro obtiene su propia librería independiente (`ProquifaDotNet.Finanzas.Infrastructure.<Sistema>Proxy`), no se agrega dentro de `ProquifaDotNetProxy`. Se comparte el patrón técnico, no la implementación. |  |

# 

