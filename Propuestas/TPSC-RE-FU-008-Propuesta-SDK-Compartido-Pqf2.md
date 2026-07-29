# **Reglas de negocio y especificación funcional**

## Propuesta SDK Compartido Pqf2

| FORMATO | Arquitectura  |
| :---- | :---- |
| **PROYECTO** | N/A |
| **REFERENCIA** | AUI- FOR-01 |
| **VERSIÓN** | 1.0 |
| **FECHA** | 27 jul 2026 |
| **AUTOR** | [Carlos Iván Morales Carreón](mailto:carlos.morales@ryndem.mx) |
| **REVISOR** | [Juan David García Cruz](mailto:juan.garcia@ryndem.mx)[A. Javier Antúnez Estrada](mailto:agustin.antunez@ryndem.mx) |

## 

## Contenido

[**1\. Resumen ejecutivo	3**](#heading=)

[**2\. Contexto	3**](#heading=)

[2.1 El problema práctico	3](#heading=)

[2.2 El precedente que valida el modelo	5](#heading=)

[**3\. Propuesta	5**](#heading=)

[3.1 Tesis: dos SDKs complementarios por stack tecnológico	5](#heading=)

[3.2 Direccionalidad del flujo	7](#heading=)

[3.3 Restricción técnica que ya define el targeting	8](#heading=)

[**4\. Alcance del SDK	9**](#heading=)

[4.1 Nombre y ubicación	9](#heading=)

[4.2 Módulo inicial — Proquifa.Pqf2.Sdk.LegacyProxy	9](#heading=)

[4.3 Roadmap paraguas (no compromete, solo dimensiona)	12](#heading=)

[**5\. Hallazgo relevante para v1.2 del Estándar R16A-132	13**](#heading=)

[5.1 Lo que el PDF R16A-132 v1.1 prescribe (§3.4)	13](#heading=)

[5.2 Lo que el ecosistema real usa hoy	13](#heading=)

[5.3 Consecuencias directas	16](#heading=)

[5.4 Petición a los autores del estándar	16](#heading=)

[**6\. Gobernanza propuesta	17**](#heading=)

[**7\. Estrategia de adopción cuando el SDK no está publicado a tiempo	18**](#heading=)

[**8\. Impacto en el ecosistema	20**](#heading=)

[**9\. Qué necesitamos de los revisores	21**](#heading=)

[**10\. Referencias	22**](#heading=)

[**Control de versiones	23**](#control-de-versiones)

# **1\. Resumen ejecutivo**

Al alinear las primeras integraciones del ecosistema `proquifa.pqf2.*` hacia `ProquifaDotNet` al **[Estándar de Proxy hacia ProquifaDotNet (Anti-Corruption Layer)](https://docs.google.com/document/d/14EBkhp4jDpsiCAO7yhhf0NbJ9ZrNoxSGfkuvqe52Z_Y/edit?tab=t.0#heading=h.4utviquks2ew)** se detectó que la librería reutilizable prescrita por el estándar — nombrada literalmente `ProquifaDotNet.Finanzas.Infrastructure.ProquifaDotNetProxy/` — **no existe físicamente en ningún repo hoy**. Existe la ventana para decidir dónde vive, cómo se distribuye y quién es su dueño antes de que se cree.  
Al mismo tiempo, se identificaron **al menos 3 consumidores** del mismo patrón que no son Finanzas: `proquifa.pqf2.MailBot`, `proquifa.pqf2.LegacySync` y el caso original de `proquifa.pqf2.Finanzas`. Todo apunta a que serán más conforme el ecosistema `proquifa.pqf2.*` crezca.  
**Esta propuesta plantea 4 cosas:**

1. **Crear el SDK \`Proquifa.Pqf2.Sdk\`** como paquete NuGet interno publicado en GitHub Packages, usando el **mismo mecanismo que ya opera en producción para \`Core.Pqf\`**.  
2. Formularlo como **"dos SDKs complementarios por stack tecnológico"** — `Core.Pqf` sigue siendo el SDK del stack legacy (.NET Framework 4.8), `Proquifa.Pqf2.Sdk` es el SDK del stack nuevo (.NET 10). Separados por diseño, sin dependencia cruzada.  
3. Arrancar el SDK con **un solo módulo** — `Proquifa.Pqf2.Sdk.LegacyProxy` — alineado 1:1 con el Estándar R16A-132 v1.3, con roadmap del SDK para módulos futuros (`.Auth`, `.Observability`, `.Api`, `.Messaging`, `.Notifications`).  
4. Escalar un **hallazgo relevante para v1.2 del propio estándar**: el patrón de autenticación real del ecosistema (ya en producción) difiere de lo prescrito en R16A-132 §3.4. **Este hallazgo se detalla en §5** — es probablemente el punto más importante de esta propuesta para los revisores del estándar.

**Costo:** overhead de mantener el repo con disciplina de semver y changelog, mismo modelo que `Core.Pqf` acepta hoy.  
**Beneficio:** cero duplicación entre las 3+ soluciones, versionado independiente por consumidor, un lugar natural para las utilidades transversales que ya se están reinventando por solución.

# **2\. Contexto**

## **2.1 El problema práctico**

El Estándar de Proxy R16A-132 v1.1 prescribe una receta ACL: **Typed HttpClient vía \`IHttpClientFactory\` \+ \`Microsoft.Extensions.Http.Resilience\` \+ autenticación contra IdentityServer4 \+ excepción propia \`ProquifaDotNetProxyException\`**. La receta es sólida y aplica sin cambios a los consumidores identificados:

| Solución nueva | Consumo de PQF-R14 legacy |
| :---- | :---- |
| `proquifa.pqf2.MailBot` | Cliente hacia `WebApi.Catalogos/ArchivoExtensionsController` (`GET UrlSubirArchivo` \+ `PATCH MoverArchivoMinIO`) |
| `proquifa.pqf2.LegacySync` | Servicio de sincronización de archivos hacia endpoint de descarga en `ProquifaDotNet` |
| `proquifa.pqf2.Finanzas` | Caso original del estándar |

Sin decisión explícita, el default operativo sería **clonar la receta en cada solución**. Esto garantiza divergencia a mediano plazo: cuando el estándar evolucione a v1.2/v1.3, o cuando se descubra que un endpoint no es idempotente y hay que ajustar la política de retry, habrá que hacer el mismo cambio N veces con riesgo de drift entre implementaciones.  
**Diagrama — consumidores del Estándar R16A-132 v1.1:**  
![][image1]

```
flowchart LR    subgraph Ecosistema[".NET 10 — ecosistema proquifa.pqf2.*"]        MB["proquifa.pqf2.MailBot"]        LS["proquifa.pqf2.LegacySync"]        FZ["proquifa.pqf2.Finanzas<br/>caso original del estandar"]        FUT["+ N consumidores<br/>futuros del ecosistema"]    end    STD{{"Estandar R16A-132 v1.1<br/>Anti-Corruption Layer<br/>Typed HttpClient + Resilience<br/>+ Auth IdS4 + Exception"}}    LIB[/"Libreria prescrita en el PDF:<br/>ProquifaDotNet.Finanzas.Infrastructure<br/>.ProquifaDotNetProxy<br/><br/>NO EXISTE FISICAMENTE HOY"/]    PQF[("ProquifaDotNet<br/>.NET Framework 4.8<br/>WebApi.Catalogos")]    MB -.aplica.-> STD    LS -.aplica.-> STD    FZ -.aplica.-> STD    FUT -.aplica.-> STD    STD --> LIB    MB --> PQF    LS --> PQF    FZ --> PQF    style LIB stroke-dasharray: 5 5    style STD fill:#fff4e6    style PQF fill:#ffe6e6
```

## **2.2 El precedente que valida el modelo**

**\`Core.Pqf\` ya se empaqueta y publica como paquete NuGet en GitHub Packages** (mismo org, mismo pipeline, mismo `NuGet.Config` en los repos consumidores como `ProquifaDotNet`). Ese mecanismo lleva tiempo en producción, es conocido por Ryndem y por DevOps, y es exactamente el patrón que esta propuesta reutiliza sin infraestructura nueva.

# **3\. Propuesta**

## **3.1 Tesis: dos SDKs complementarios por stack tecnológico**

El ecosistema tiene dos stacks. Cada uno debería tener su SDK reconocible, con ciclo de vida propio, sin dependencia cruzada:

| Aspecto | \`Core.Pqf\` (existente) | \`Proquifa.Pqf2.Sdk\` (propuesto) |
| :---- | :---- | :---- |
| Stack objetivo | .NET Framework 4.8 | .NET 10 |
| Consumidores principales | `ProquifaDotNet` (monolito legacy) y otros sistemas legacy | Todas las soluciones nuevas: `MailBot`, `Finanzas`, `LegacySync`, `Notificaciones`, etc. |
| Propósito primario | Utilidades transversales del monolito legacy | Utilidades transversales del ecosistema nuevo; **primer módulo \= proxy ACL del estándar R16A-132 v1.1** |
| Distribución | NuGet vía GitHub Packages (en producción hoy) | NuGet vía GitHub Packages — **mismo mecanismo, mismo feed** |
| Relación | **Separados por diseño.** No comparten código ni versión; no hay dependencia cruzada. Cada uno vive en su stack | Idem |

Un desarrollador que entra al ecosistema sabe: *"estoy en .NET 4.8 → \`Core.Pqf\`; estoy en .NET 10 → \`*Proquifa.Pqf2.Sdk*\`"*. Sin ambigüedad, sin tentación de fusionar los dos ni de forzar multi-targeting.  
**Diagrama — dos SDKs complementarios por stack tecnológico:**  
![][image2]

```
flowchart TB
    subgraph Feed["GitHub Packages — feed NuGet privado (RYNDEM)"]
        direction LR
        PKG1[["Core.Pqf<br/>en produccion hoy<br/>net48"]]
        PKG2[["Proquifa.Pqf2.Sdk<br/>PROPUESTO<br/>net10.0"]]
    end

    subgraph Legacy["Stack legacy — .NET Framework 4.8"]
        direction TB
        PQF["ProquifaDotNet<br/>monolito legacy"]
        OTROS["otros sistemas<br/>legacy .NET 4.8"]
    end

    subgraph Nuevo["Stack nuevo — .NET 10 (ecosistema proquifa.pqf2.*)"]
        direction TB
        MB2["proquifa.pqf2.MailBot"]
        LS2["proquifa.pqf2.LegacySync"]
        FZ2["proquifa.pqf2.Finanzas"]
        NT2["proquifa.pqf2.Notificaciones"]
        MAS["+ soluciones futuras"]
    end

    PKG1 --consume--> PQF
    PKG1 --consume--> OTROS

    PKG2 --consume--> MB2
    PKG2 --consume--> LS2
    PKG2 --consume--> FZ2
    PKG2 --consume--> NT2
    PKG2 --consume--> MAS

    PKG1 -. sin dependencia cruzada .- PKG2

    style Legacy fill:#ffe6e6
    style Nuevo fill:#e6f2ff
    style PKG1 fill:#ffcccc
    style PKG2 fill:#cce0ff
```

## **3.2 Direccionalidad del flujo**

El eje de comunicación es **bidireccional**, pero cada dirección tiene su naturaleza propia:

| Aspecto | Nuevo → Legacy (proxy ACL, R16A-132) | Legacy → Nuevo (cliente HTTP normal) |
| :---- | :---- | :---- |
| Consumidor | .NET 10 (`proquifa.pqf2.*`) | .NET Framework 4.8 (`ProquifaDotNet` y otros) |
| Destino | `ProquifaDotNet` (.NET 4.8) | `proquifa.pqf2.*` (.NET 10\) |
| Modelo de dominio del destino | Ajeno (español, legacy) — **requiere traducción ACL** | Moderno, contrato limpio propio — **no requiere traducción** |
| Naming del patrón | Proxy (Anti-Corruption Layer) | Cliente HTTP tipado |
| **Vive en** | `Proquifa.Pqf2.Sdk.LegacyProxy` (esta propuesta) | Un módulo dentro de `Core.Pqf`, bajo demanda cuando aparezca el primer consumidor real (fuera del alcance de esta propuesta) |

Ambos módulos comparten la **disciplina y los patrones** (Typed HttpClient \+ resiliencia \+ excepción propia \+ tests con WireMock), aunque cada uno vive en su SDK y su stack. El sentido inverso (legacy → nuevo) se resuelve extendiendo `Core.Pqf` **bajo demanda cuando aparezca el primer consumidor concreto** — no requiere trabajo derivado en este DAR ni un tercer SDK.  
**Diagrama — bidireccionalidad del eje legacy ↔ nuevo:**  
![][image3]

```
flowchart LR
    subgraph LegacyStack["Stack legacy — .NET Framework 4.8"]
        PQF["ProquifaDotNet<br/>WebApi.Catalogos<br/>modelo de dominio<br/>en espanol / legacy"]
    end

    subgraph NuevoStack["Stack nuevo — .NET 10"]
        MB3["MailBot / LegacySync /<br/>Finanzas / etc."]
        API3["APIs nuevas<br/>proquifa.pqf2.*<br/>contrato limpio<br/>modelo propio"]
    end

    MB3 ==>|"Nuevo -> Legacy<br/>PROXY ACL<br/>traduccion de modelo<br/>Proquifa.Pqf2.Sdk.LegacyProxy<br/>(esta propuesta)"| PQF

    PQF ==>|"Legacy -> Nuevo<br/>cliente HTTP tipado<br/>sin traduccion<br/>Modulo dentro de Core.Pqf<br/>(bajo demanda, fuera de esta propuesta)"| API3

    style PQF fill:#ffe6e6
    style API3 fill:#e6f2ff
    style MB3 fill:#e6f2ff
```

Nota: en ambas direcciones la autenticación usa `client_credentials` \+ scope `internal.processes` (ver §5), con la convención `<origen>.<destino>.client` para el `ClientId`.

## **3.3 Restricción técnica que ya define el targeting**

El SDK propuesto sería **exclusivamente \`net10.0\`**, no multi-target. Justificación técnica:

* `Microsoft.Extensions.Http.Resilience` (pilar del estándar §3.2) requiere .NET 8+ — no compila contra `netstandard2.0` ni contra .NET Framework 4.8.  
* Los únicos consumidores realistas son las soluciones nuevas del ecosistema `proquifa.pqf2.*`, todas .NET 10 por construcción del proyecto R16.  
* Los sistemas legacy no van a migrar su stack HTTP al SDK nuevo — ya tienen sus wrappers manuales (`TokenAsynProquifaDotNet` en R14) y usarán el módulo simétrico en `Core.Pqf` cuando lo necesiten.

Convenciones del `.csproj`: `<TargetFramework>net10.0</TargetFramework>`, `<LangVersion>latest</LangVersion>`, `<Nullable>enable</Nullable>`. Cuando salga .NET 11 (nov/2026), la decisión de subir `net10.0` → `net11.0` se toma en su momento, no aquí.

# **4\. Alcance del SDK**

## **4.1 Nombre y ubicación**

* **Repo:** `ryndem/proquifa-pqf2-sdk` (kebab-case, mismo patrón que `ryndem/proquifa-pqf2-mailbot-backend`; sin sufijo `-backend` porque es librería, no servicio).  
* **Paquete NuGet / namespace .NET:** `Proquifa.Pqf2.Sdk` como paraguas (PascalCase con puntos, extensión natural de la convención `Proquifa.Pqf2.<Modulo>` ya establecida en `MailBot`/`LegacySync`/`Finanzas`/`Notificaciones`), con submódulos `Proquifa.Pqf2.Sdk.<Modulo>`.

## **4.2 Módulo inicial — `Proquifa.Pqf2.Sdk.LegacyProxy`**

Alineado 1:1 al Estándar R16A-132 v1.1.  
**Diagrama — arquitectura del módulo y su integración con un consumidor:**  
![][image4]

```
flowchart TB    subgraph Consumer["Consumidor (ej. una solucion del ecosistema)"]        APP["Application Layer<br/>llama IProquifaDotNetFileProxy"]        IMPL["ProquifaDotNetFileProxy<br/>implementa el contrato<br/>del consumidor"]        CFG["appsettings.Ambiente.json<br/>seccion ProquifaDotNetProxy"]    end    subgraph SDK["Proquifa.Pqf2.Sdk.LegacyProxy (net10.0)"]        direction TB        IFACE["IProquifaDotNet&lt;Recurso&gt;Proxy<br/>contratos por recurso"]        EXT["ServiceCollectionExtensions<br/>AddProquifaDotNetProxy&lt;T&gt;()"]        OPTS["ProquifaDotNetProxyOptions<br/>BaseUrl + Timeout +<br/>IdentityServer + Resilience"]        subgraph Auth["Auth/"]            TP["ProquifaDotNetTokenProvider<br/>client_credentials +<br/>IMemoryCache TTL = Expires - 300s"]        end        subgraph Config["Configuration/"]            RES["ResiliencePolicyDefaults<br/>retry backoff+jitter<br/>timeout circuit-breaker<br/>RetryOnGetOnly = true"]        end        EXC["ProquifaDotNetProxyException<br/>envuelve errores crudos<br/>de HttpClient"]    end    subgraph External["Externos"]        IDS["IdentityServer4<br/>configInternalClients.json<br/>scope internal.processes"]        PQF4["ProquifaDotNet<br/>WebApi.Catalogos"]    end    APP --> IMPL    IMPL --usa--> IFACE    IMPL --registro DI--> EXT    EXT --lee--> OPTS    OPTS <--valores--> CFG    EXT --configura pipeline--> RES    EXT --inyecta--> TP    IMPL --http request--> PQF4    TP --token endpoint--> IDS    IMPL --lanza--> EXC    style SDK fill:#cce0ff    style External fill:#ffe6e6
```

Estructura del repo:

```
Proquifa.Pqf2.Sdk/
├── src/
│   └── Proquifa.Pqf2.Sdk.LegacyProxy/           # módulo inicial
│       ├── IProquifaDotNet<Recurso>Proxy.cs      # interfaces por recurso
│       ├── ProquifaDotNetProxyException.cs       # excepción propia (estándar §3.3)
│       ├── Auth/
│       │   ├── IProquifaDotNetTokenProvider.cs
│       │   └── ProquifaDotNetTokenProvider.cs    # OAuth2 client_credentials + IMemoryCache (ver §5)
│       ├── Configuration/
│       │   ├── ProquifaDotNetProxyOptions.cs
│       │   └── ResiliencePolicyDefaults.cs       # retry backoff+jitter, timeout, circuit breaker (estándar §3.2)
│       └── Extensions/
│           └── ServiceCollectionExtensions.cs    # AddProquifaDotNetProxy<T>()
├── tests/
│   └── Proquifa.Pqf2.Sdk.LegacyProxy.Tests/     # WireMock.NET, cobertura >80% líneas
├── .github/workflows/
│   ├── build.yml
│   └── publish-nuget.yml                         # publica a GitHub Packages en cada tag semver
├── docs/
│   ├── README.md
│   ├── LegacyProxy.md
│   └── CHANGELOG.md
└── Directory.Packages.props                      # central package management
```

**Uso desde un consumidor** (ejemplo genérico):

```c#
services.AddProquifaDotNetProxy<IProquifaDotNetFileProxy, ProquifaDotNetFileProxy>(
    configuration.GetSection("ProquifaDotNetProxy"));
```

Con configuración en `appsettings.{Ambiente}.json`:

```json
{
  "ProquifaDotNetProxy": {
    "BaseUrl": "https://api.proquifadotnet.internal/",
    "TimeoutSeconds": 30,
    "IdentityServer": {
      "TokenUrl": "https://identity.proquifadotnet.internal/connect/token",
      "ClientId": "<feature>.<modulo>.client",
      "ClientSecret": "<vault>",
      "Scope": "internal.processes"
    },
    "Resilience": {
      "RetryOnGetOnly": true,
      "MaxRetryAttempts": 3
    }
  }
}
```

El `ClientId` sigue la convención `<feature>.<modulo>.client` documentada en `configInternalClients.json` del IdentityServer (ver §5).

## **4.3 Roadmap del SDK(no compromete, solo dimensiona)**

Cada módulo es un paquete NuGet independiente publicado desde el mismo repo (patrón monorepo usado por Serilog, Polly, ASP.NET Core). Un consumidor instala solo lo que necesita:

| Módulo | Propósito | Timing |
| :---- | :---- | :---- |
| `Proquifa.Pqf2.Sdk.LegacyProxy` | Proxy ACL hacia PQF-R14 (base del SDK, R16A-132 v1.1) | Fase actual (esta propuesta) |
| `Proquifa.Pqf2.Sdk.Auth` | Handlers y helpers de IdentityServer4 para el lado **servidor** (validación de JWT entrante, claims, políticas de rol en las APIs nuevas `proquifa.pqf2.*`). No confundir con `Auth/` dentro de `LegacyProxy` (§4.2), que es auth **saliente** — obtención de token vía `client_credentials` para llamar a `ProquifaDotNet` | Cuando aparezca el segundo consumidor concreto |
| `Proquifa.Pqf2.Sdk.Observability` | Serilog \+ enrichers (correlation ID, tenant), `HealthChecks` para MinIO/RabbitMQ/SQL Server | Cuando el patrón se estabilice en 2+ soluciones |
| `Proquifa.Pqf2.Sdk.Api` | `ProblemDetails` unificado, middleware de excepción a HTTP, contratos de paginación | Convenciones .NET 10 aplicables a todas las APIs del ecosistema |
| `Proquifa.Pqf2.Sdk.Messaging` | Abstracciones sobre RabbitMQ / Pub/Sub | Cuando surja el primer segundo consumidor de mensajería |
| `Proquifa.Pqf2.Sdk.Notifications` | Cliente tipado hacia `proquifa.pqf2.Notificaciones` | Sujeto a **3 precondiciones**: (1) contrato de la API publicado y estable (OpenAPI \+ semver por el equipo propietario); (2) ≥2 consumidores en producción con implementación in-repo (ya hay consumidores identificados en el ecosistema que cumplirán este criterio); (3) `LegacyProxy 1.0.0` validado en producción. **Timing esperado: probable 2027** |

**Ningún módulo se compromete en este release** salvo `LegacyProxy`. El roadmap justifica por qué el SDK es paraguas y no monofuncional.

# **5\. Hallazgo relevante para v1.2 del Estándar R16A-132**

*Este es probablemente el punto más importante de esta propuesta para los revisores del estándar. No es una crítica al PDF v1.1 — es evidencia del patrón que ya opera en producción y que difiere de lo prescrito.*

## **5.1 Lo que el PDF R16A-132 v1.1 prescribe (§3.4)**

**ROPC (\`grant\_type=password\`)** con cuenta de servicio, `DelegatingHandler` que adjunta el token como `Bearer`, argumentando que `AuthorizeProquifaDotNet` de `WebApi.Catalogos` exige roles asociados a usuario y por lo tanto *"\`client\_credentials\` puro no funciona"*.

## **5.2 Lo que el ecosistema real usa hoy**

**\`client\_credentials\` puro** con un scope dedicado `internal.processes` creado explícitamente para este propósito. Sin `DelegatingHandler` — token cacheado explícitamente en `IMemoryCache`. Evidencia concreta:  
**a) \`identity-server/IdentityServer/Config.cs\`:**

```c#
new ApiResource(Constants.DefaultScopeNames.InternalProcesses,
                "Procesos internos — Jobs, servicios e integraciones de ProquifaDotNet")
```

**b) \`identity-server/IdentityServer/configInternalClients.json\`** — archivo externo cargado en runtime (mismo patrón que `configCORS_*.json`), con la convención documentada en sus propios comentarios:

```c#
// Patron de clientId:  <feature>.<modulo>.client
//   Ejemplos:  logistica.sync.client  |  facturacion.batch.client
// El scope "internal.processes" es compartido por todos — no se modifica. 
```

 Un cliente ya operando en producción: **\`cargamasiva.catalogos.client\`** (consumido por `proquifa-pqf2-importador-worker`).  
**c) \`proquifa-pqf2-importador-worker/Infrastructure/Services/ImportNotificationService.cs\`** — implementación de referencia con `IdentityModel.Client`:

```c#
var tokenResponse = await identityClient.RequestClientCredentialsTokenAsync(
    new ClientCredentialsTokenRequest {
        Address = tokenUrl,
        ClientId = clientId,       // "cargamasiva.catalogos.client"
        ClientSecret = clientSecret,
        Scope = scope              // "internal.processes"
    }, ct);
// IMemoryCache con TTL = ExpiresIn - 300s (margen), mínimo 60s
// Llamada explícita: client.SetBearerToken(accessToken)
```

Consume `ProquifaDotNet` en `api/internal/notify` con este mecanismo, **sin problemas de \`401 Unauthorized\`**.  
**Diagrama — flujo de autenticación real (client\_credentials \+ IMemoryCache):**  
![][image5]

```
sequenceDiagram
    autonumber
    participant App as Aplicacion<br/>(Mailbot / Worker)
    participant Proxy as ProquifaDotNetFileProxy<br/>(SDK.LegacyProxy)
    participant TP as ProquifaDotNetTokenProvider<br/>(SDK.LegacyProxy)
    participant Cache as IMemoryCache
    participant IDS as IdentityServer4<br/>connect/token
    participant PQF as ProquifaDotNet<br/>WebApi.Catalogos

    App->>Proxy: GetUrlSubirArchivo(...)
    Proxy->>TP: GetAccessTokenAsync()
    TP->>Cache: TryGet("token")

    alt Cache HIT (token vigente)
        Cache-->>TP: accessToken
    else Cache MISS / expirado
        TP->>IDS: POST connect/token<br/>grant_type=client_credentials<br/>client_id={feature}.{modulo}.client<br/>scope=internal.processes
        IDS-->>TP: 200 OK {access_token, expires_in}
        TP->>Cache: Set("token", TTL = expires_in - 300s)
    end

    TP-->>Proxy: accessToken
    Proxy->>Proxy: SetBearerToken(accessToken)
    Proxy->>PQF: GET /api/catalogos/archivo/url-subir<br/>Authorization: Bearer {token}
    PQF-->>Proxy: 200 OK { urlPresigned }
    Proxy-->>App: urlPresigned
```

Nota clave: **no hay \`DelegatingHandler\`** (contra lo prescrito por R16A-132 v1.1 §3.4). El token se obtiene explícitamente por request via `IMemoryCache`, patrón `ImportNotificationService`.

## **5.3 Consecuencias directas**

1. **\`AuthorizeProquifaDotNet\` acepta \`client\_credentials\`** cuando el scope es `internal.processes` — verificado para el endpoint `api/internal/notify` (consumidor `ImportNotificationService`). **Pendiente de confirmar sobre el endpoint concreto de cada primera integración** (por ejemplo, `WebApi.Catalogos/ArchivoExtensionsController` para el consumidor de archivos): no hay evidencia directa de que todos los controllers protegidos con `AuthorizeProquifaDotNet` compartan la misma política de autorización, solo la inferencia de que cuelgan del mismo atributo. Recomendación: validar contra DEV (o revisar el código del atributo/filtro) antes de dar por cerrado el punto 2 para cada nuevo consumidor.  
2. **La cuenta de servicio se aprovisiona por consumidor** siguiendo la convención `<feature>.<modulo>.client`. Alta \= 5 líneas de JSON \+ secret (`openssl rand -base64 32`) \+ coordinar por ambiente con DevOps. Si la verificación del punto 1 confirma que el controller destino acepta el mismo scope, es tarea de \~15 min, no de sprint.  
3. **No se necesita \`DelegatingHandler\` de auth** — el patrón vigente usa `IMemoryCache` con TTL \= `ExpiresIn - 300s` (margen), llamada explícita a `SetBearerToken`.  
4. **Todos los clients internos comparten el mismo scope \`internal.processes\`** (el JSON dice literalmente *"no se modifica"*).

## **5.4 Petición a los autores del estándar**

**Publicar v1.2 del R16A-132 alineando la §3.4 al patrón real:** reemplazar el ejemplo ROPC \+ `DelegatingHandler` por `client_credentials` \+ `internal.processes` \+ `IMemoryCache` (patrón `ImportNotificationService`). Esta propuesta ya asume v1.2; documentarlo en el propio estándar cierra el círculo para futuros consumidores.  
Este hallazgo también **apunta a simplificar el aprovisionamiento de cuentas de servicio para cada consumidor**: de "hueco crítico bloqueante que requiere modelar ROPC con roles" a "tarea de \~15 min con DevOps" — **condicionado a la verificación del punto 1 de §5.3** contra el controller real de cada consumidor. Hasta confirmar eso, se recomienda tratar el aprovisionamiento como "probablemente simple, pendiente de una validación puntual".

# **6\. Gobernanza propuesta**

Modelo alineado al patrón ya establecido para el resto del ecosistema `proquifa.pqf2.*` (Mailbot, LegacySync, etc.) — el SDK es **una solución más del ecosistema**, no un producto especial con equipo dedicado.

| \# | Tema | Propuesta |
| :---- | :---- | :---- |
| G1 | **Owner del código** | Desarrolladores de Ryndem que trabajan el ecosistema R16 (los mismos que trabajan las soluciones consumidoras). **Propuesta de referentes técnicos con aprobación en PRs que toquen \`LegacyProxy\`:** Javier Antúnez \+ Juan David García (autores/revisor del estándar), para garantizar que la implementación no divergue del contrato de R16A-132 — **sujeto a su disponibilidad real**; si el rol de aprobador obligatorio no es viable para ellos, se puede degradar a revisión informativa (no bloqueante) sin afectar el resto de la propuesta |
| G2 | **DevOps** | Crea el repo `ryndem/proquifa-pqf2-sdk` cuando se solicita; administra credenciales del feed NuGet privado y el pipeline de release a Staging/Producción. Mismo modelo que aplica a todos los repos del ecosistema |
| G3 | **Modelo de contribución** | Fork+PR; revisor obligatorio del owner. Cambios breaking requieren migración en al menos 1 consumidor de referencia dentro del mismo PR |
| G4 | **Versionado** | Semver estricto. `1.0.0` \= versión inicial con `LegacyProxy` alineado a R16A-132 v1.2. Breaking → MAJOR; nuevos módulos → MINOR; fixes → PATCH |
| G5 | **Ciclo de release** | Bajo demanda: cada tag `vX.Y.Z` dispara el workflow de publicación a GitHub Packages |
| G6 | **Documentación mínima** | README por módulo, CHANGELOG central, ejemplos de uso ejecutables. Sin documentación no hay merge |
| G7 | **Testing** | \>80% cobertura de líneas. Tests de integración contra `WireMock.NET` para el proxy ACL |
| G8 | **Relación con \`Core.Pqf\`** | Separados por diseño y complementarios. Comparten únicamente el mecanismo de distribución (GitHub Packages, mismo org) |

# **7\. Estrategia de adopción cuando el SDK no está publicado a tiempo**

En la práctica, las primeras integraciones concretas que necesitan el patrón del estándar suelen aterrizar **antes** de que el SDK esté publicado y estable — es normal, dado que el propio SDK nace justamente de esas primeras integraciones. Forzar el timing (empujar el SDK a publicarse contra reloj para no bloquear la primera integración) genera riesgo alto y valor bajo: **el diseño se valida mejor contra un consumidor real** antes de empaquetarlo.  
Se propone una **estrategia de adopción en 3 fases** que evita bloquear a los primeros consumidores sin sacrificar el objetivo del SDK:  
**Diagrama — estrategia de adopción en 3 fases:**  
![][image6]

```
flowchart LR
    subgraph FaseA["Fase A — Implementacion in-repo"]
        A1["Primer consumidor<br/>implementa el patron<br/>directamente en su repo<br/>siguiendo la estructura §4.2"]
        A2["Alta de cuenta de servicio<br/>&lt;feature&gt;.&lt;modulo&gt;.client<br/>en configInternalClients.json"]
    end

    subgraph FaseB["Fase B — SDK en paralelo"]
        B1["Crear repo del SDK<br/>+ pipeline GitHub Packages<br/>(DevOps)"]
        B2["Portar las clases del<br/>primer consumidor<br/>al modulo LegacyProxy"]
        B3["Tests con WireMock<br/>cobertura >80%"]
        B4["Publicar 1.0.0-alpha<br/>iterar hasta 1.0.0 estable"]
    end

    subgraph FaseC["Fase C — Refactor + adopcion"]
        C1["Refactor mecanico<br/>del primer consumidor:<br/>mover archivos +<br/>PackageReference"]
        C2["Siguientes consumidores<br/>arrancan desde el dia 1<br/>con dotnet add package"]
    end

    A1 --> A2 --> B1
    B1 --> B2 --> B3 --> B4
    B4 --> C1 --> C2

    style FaseA fill:#fff4e6
    style FaseB fill:#e6f2ff
    style FaseC fill:#e6ffe6
```

**Descripción de cada fase:**

| Fase | Actividad | Riesgo |
| :---- | :---- | :---- |
| **A — Implementación in-repo** | El primer consumidor implementa el proxy (`ProquifaDotNetFileClient`, `ProquifaDotNetTokenProvider`, `ProquifaDotNetProxyException`, `ResiliencePolicyDefaults`, opciones) **directamente en su repo** siguiendo la misma estructura del §4.2 de esta propuesta. Único bloqueante de runtime: alta de la cuenta de servicio `<feature>.<modulo>.client` en `configInternalClients.json` (§5) — trabajo de \~15 min con DevOps | Muy bajo — el diseño ya está definido por esta propuesta y por el estándar |
| **B — SDK en paralelo** | En ventana holgada (sin sprint asignado): DevOps crea el repo del SDK \+ configura el pipeline (adaptando el de `Core.Pqf`); se portan las clases del primer consumidor al módulo `Proquifa.Pqf2.Sdk.LegacyProxy`; tests con WireMock; se publica `1.0.0-alpha` y se itera hasta `1.0.0` estable | Bajo — no está en la ruta crítica de ningún consumidor |
| **C — Refactor \+ adopción** | Refactor mecánico en el primer consumidor: mover archivos \+ cambiar namespaces \+ agregar `PackageReference`. **Los siguientes consumidores arrancan desde el día 1 consumiendo el SDK**, sin implementación in-repo | Bajo — refactor mecánico, típicamente \~200-300 LOC por consumidor |

**Nota de contingencia:** si la Fase B se atrasa respecto al arranque del segundo consumidor, ese segundo consumidor debería aplicar **la misma estrategia in-repo que el primero** (Fase A) en vez de bloquear su arranque. La deuda técnica temporal es acotada y trazable, con refactor de bajo riesgo cuando el SDK esté disponible.  
**Beneficios de esta estrategia:**

* La implementación in-repo del primer consumidor **valida el diseño del SDK contra un consumidor real** antes de empaquetarlo (mejor que diseñar en abstracto).  
* Los primeros consumidores no dependen de que DevOps corra en ventanas comprimidas.  
* Deuda técnica temporal acotada y trazable, con refactor de bajo riesgo.

# **8\. Impacto en el ecosistema**

Con la propuesta aceptada:

* **Cero duplicación entre consumidores.** El patrón vive en un solo lugar; cambios al estándar (v1.2, v1.3, …) se propagan vía nueva versión del paquete en vez de N PRs sincronizados en repos distintos.  
* **Versionado independiente por consumidor.** Cada solución elige cuándo actualizar su `PackageReference`; una versión problemática no rompe a los demás. Semver estricto (§6, G4) da garantías explícitas de compatibilidad.  
* **Un solo lugar donde ajustar la política de retry.** La preocupación de idempotencia (por ejemplo, no reintentar `PATCH`/`PUT` en endpoints no idempotentes como `MoverArchivoMinIO`) se decide una vez en `ResiliencePolicyDefaults`, con opción de opt-in por consumidor vía `ProquifaDotNetProxyOptions.Resilience`.  
* **Aprovisionamiento de cuentas de servicio simplificado** por la aplicación del hallazgo de §5: alta de `<feature>.<modulo>.client` en `configInternalClients.json` \+ secret con DevOps por ambiente — trabajo de \~15 min por consumidor, condicionado a la verificación puntual del punto 1 de §5.3.  
* **Los primeros consumidores no dependen de que el SDK esté publicado a tiempo** — la estrategia de adopción en 3 fases (§7) permite arranques in-repo con refactor mecánico posterior, sin bloquear ninguna ruta crítica.  
* **Escalabilidad a nuevos consumidores del ecosistema.** Cada solución nueva del ecosistema `proquifa.pqf2.*` que necesite consumir `ProquifaDotNet` puede sumarse con `dotnet add package` en vez de reimplementar el patrón desde cero — reduce time-to-market de las nuevas integraciones.  
* **Base transversal para utilidades futuras.** El roadmap paraguas (§4.3) da un lugar natural donde poner las utilidades transversales que hoy se reinventan por solución (auth, observability, ProblemDetails, messaging), sin forzar decisiones prematuras sobre módulos que aún no tienen consumidores concretos.

**Impacto colateral fuera del alcance directo del SDK (relevante pero no bloqueante):**

* **Estándar R16A-132 → v1.2:** actualizar §3.4 al patrón real `client_credentials` \+ `internal.processes` (§5 de esta propuesta).  
* **Módulo simétrico en \`Core.Pqf\`** para el sentido inverso (legacy → nuevo, ver §3.2): se implementa **bajo demanda**, cuando aparezca el primer consumidor legacy real del ecosistema nuevo. No requiere trabajo derivado por esta propuesta; solo queda documentada la posición para que el equipo dueño de `Core.Pqf` lo aterrice cuando corresponda.

# **9\. Qué necesitamos de los revisores**

Puntos concretos donde esta propuesta requiere aprobación o coordinación:

| \# | Necesidad | A quién | Cuándo |
| :---- | :---- | :---- | :---- |
| N1 | **Aprobación de Opción C** — SDK paraguas `Proquifa.Pqf2.Sdk` en repo nuevo `ryndem/proquifa-pqf2-sdk`, publicado a GitHub Packages siguiendo el modelo de `Core.Pqf` | Ryndem (dirección técnica) | Cuanto antes; no bloquea a los primeros consumidores dado que la estrategia de adopción de §7 corre sin esta aprobación (Fase A in-repo) |
| N2 | **Confirmación del modelo de gobernanza (§6)** — owners funcionales \= desarrolladores Ryndem del ecosistema `proquifa.pqf2.*`; referentes técnicos en PRs de `LegacyProxy` \= Javier Antúnez \+ Juan David García (rol exacto — aprobador obligatorio vs. revisión informativa — sujeto a lo que ellos puedan sostener, ver G1) | Ryndem \+ Javier \+ Juan David | Idem N1, sin fecha dura |
| N3 | **Publicación de R16A-132 v1.2** actualizando §3.4 al patrón `client_credentials` \+ `internal.processes` documentado en §5 de esta propuesta | Javier Antúnez \+ Juan David García | Antes de que el SDK publique `1.0.0` (Fase B/C de §7) |
| N4 | **DevOps: creación del repo \+ pipeline de release \+ credenciales del feed NuGet privado** (adaptando el pipeline de `Core.Pqf`) | DevOps | Fase B de §7 (ventana holgada, en paralelo a la implementación in-repo del primer consumidor) |
| N5 | **DevOps: alta de la cuenta de servicio del primer consumidor** — entrada `<feature>.<modulo>.client` en `configInternalClients.json` del IdentityServer real por ambiente (DEV/QA/PROD), 5 líneas de JSON \+ secret | DevOps \+ equipo IdentityServer4 | Antes del arranque del primer consumidor (Fase A de §7); análogo para cada consumidor subsecuente |
| N6 | **Confirmación del alcance de la simetría inversa** — módulo dentro de `Core.Pqf` bajo demanda para el sentido legacy → nuevo, sin trabajo derivado por esta propuesta | Ryndem \+ equipo owner de `Core.Pqf` (ProquifaDotNet) | Al aparecer el primer consumidor concreto (fuera de esta propuesta) |

# **10\. Referencias**

| Documento | Propósito |
| :---- | :---- |
| **Estándar de Proxy hacia ProquifaDotNet (ACL) R16A-132 v1.1**, 06/07/2026, Javier Antúnez / Juan David García | Documento base del que nace esta propuesta (Google Drive de R16) |
| Repo interno `proquifa-pqf2-importador-worker` (`Infrastructure/Services/ImportNotificationService.cs`) | Implementación de referencia del patrón `client_credentials` \+ `internal.processes` (§5) |
| Repo interno `identity-server` (`IdentityServer/Config.cs`, `configInternalClients.json`) | Evidencia del scope `internal.processes` y de la convención `<feature>.<modulo>.client` |
| Repo interno `Core.Pqf` | Precedente del mecanismo de publicación NuGet vía GitHub Packages que esta propuesta reutiliza (§2.2, §3.1) |

# 

# **Control de versiones** {#control-de-versiones}

| Versión | Fecha | Autor | Tipo de Cambio | Descripción del cambio | Aprobó |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 1.0 | 27 jul 2026 | [Carlos Iván Morales Carreón](mailto:carlos.morales@ryndem.mx) | Creación | Creación del documento. |  |
|  |  |  |  |  |  |
