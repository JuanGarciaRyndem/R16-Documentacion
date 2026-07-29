# **Inventario técnico**

## Puertos locales — proyectos .NET (concentración de referencia)

| FORMATO | Arquitectura  |
| :---- | :---- |
| **PROYECTO** | N/A |
| **REFERENCIA** | AUI- FOR-01 |
| **VERSIÓN** | 1.0 |
| **FECHA** | 27 jul 2026 |
| **AUTOR** | [Carlos Iván Morales Carreón](mailto:carlos.morales@ryndem.mx) |
| **REVISOR** |  |

## 

## Contenido

[**Resumen ejecutivo	3**](#heading=)

[**1\. ASP.NET Core / .NET 5+ — launchSettings.json	3**](#heading=)

[**2\. Hallazgo — puertos duplicados heredados del arquetipo	4**](#heading=)

[2.1 Esquema aplicado a Mailbot y LegacySync	4](#heading=)

[2.2 Propuesta para los dos repos externos/históricos (pendiente de aplicar)	5](#heading=)

[2.3 Vista consolidada	5](#heading=)

[**3\. Workers sin puerto HTTP (BackgroundService puro, sin applicationUrl)	6**](#heading=)

[**4\. .NET Framework clásico — bindings de IIS Express (local a esta máquina)	6**](#heading=)

[**5\. Rangos ya ocupados (referencia para el próximo scaffold)	7**](#heading=)

[**6\. Convención de referencia — segmentación por bloques (compartida por Carlos, 24/07/2026)	7**](#heading=)

[6.1 Identificador por proyecto (dígito dentro de cada rango común)	8](#heading=)

[6.2 Bloques completos por propósito	8](#heading=)

[6.3 Aplicación a este ecosistema hoy	9](#heading=)

[**Control de versiones	10**](#control-de-versiones)

# **Resumen ejecutivo**

* **Qué es esto.** Un inventario de qué puerto usa cada proyecto .NET local del ecosistema PQF2, para ver de un vistazo si dos soluciones chocan antes de correrlas juntas. Fuente: `launchSettings.json` de cada repo (versionado, portable entre máquinas) y, para los proyectos .NET Framework clásicos, el `applicationhost.config` de IIS Express de esta máquina (no versionado, puede diferir en otro equipo).  
* **Qué no es.** No es un ajuste que se aplicó a los repos ni una convención obligatoria — es una fotografía del estado actual. Los puertos aquí documentados no se tocaron salvo donde se indica explícitamente (Mailbot y LegacySync, §2.1).  
* **Hallazgo principal.** Tres proyectos (`microservice-clean-architecture-template`, `proquifa-punchout-backend`, `ExchangeRateService`) heredan del arquetipo los mismos cuatro puertos sin cambiarlos — chocan si se corren dos a la vez. Mailbot y LegacySync, los dos derivados que el equipo controla hoy, ya se corrigieron con un esquema formal (`80XY`, §2.1). Los otros dos son repos externos/históricos, fuera de alcance directo del equipo — este inventario les propone valores dentro del mismo esquema (§2.2), pendientes de que un desarrollador con acceso a cada repo los aplique cuando le convenga. Aparte, hay colisiones ya presentes en IIS Express clásico (`WebSite1:8080` compartido por `ProquifaDotNet` y `Core.Pqf`, §4) que no se tocan, solo se registran.  
* **Qué hacer si vas a levantar un proyecto nuevo del arquetipo.** Seguí el esquema `80XY` (§2.1) y tomá el siguiente identificador realmente libre (`5`, ver §5 — los ids `3` y `4` ya están propuestos para los repos externos en §2.2). Si en cambio vas a correr en local uno de los repos con puertos heredados sin cambiar, revisá primero §5 para evitar los valores ya ocupados.

# **1\. ASP.NET Core / .NET 5+ — `launchSettings.json`**

| Repo | Proyecto | IIS Express (http / sslPort) | Perfil http | Perfil https |
| :---- | :---- | :---- | :---- | :---- |
| `microservice-clean-architecture-template` | `API` | `32254` / `44385` | `5180` | `7190` |
| `proquifa-pqf2-mailbot-backend` | `Mailbot.Api` | `8001` / `8011` | `8021` | `8031` |
| `proquifa-punchout-backend` | `API` | `32254` / `44385` | `5180` | `7190` |
| `ExchangeRateService` | `API` | `32254` / `44385` | `5180` | `7190` |
| `proquifa-pqf2-legacysync-backend` | `LegacySync.Api` | `8002` / `8012` | `8022` | `8032` |
| `venta-digital-backend` | `VentaDigitalPQF` | `51071` / `44375` | `5052` | `7101` |
| `identity-server` | `IdentityServer` | `5000` / `44333` | `5000` (perfil `IdentityServer`, sin https propio) | `44333` (solo vía perfil `IIS Express`) |
| `TaskSchedulerPqf` | (raíz del repo) | `17212` / `0` (sin SSL) | `5174` | — |

# **2\. Hallazgo — puertos duplicados heredados del arquetipo**

`microservice-clean-architecture-template/API`, `proquifa-punchout-backend/API` y `ExchangeRateService/API` comparten exactamente los mismos 4 valores: `32254` / `44385` / `5180` / `7190`. No es coincidencia — son los puertos que Visual Studio generó una sola vez al crear el arquetipo `microservice-clean-architecture-template`, y cada proyecto derivado copió el `launchSettings.json` sin cambiarlos (el renombrado transversal `Microservicio.*` → `<Producto>.*` nunca cubrió este archivo). Si se necesita correr dos de estos tres al mismo tiempo en local, van a chocar.  
`Mailbot.Api` y `LegacySync.Api` tenían el mismo problema — son los dos únicos derivados del arquetipo que el equipo controla hoy (los otros dos son repos externos/históricos, fuera de alcance). Se corrigieron con un esquema formal, no con valores ad-hoc; ver §2.1.

## **2.1 Esquema aplicado a Mailbot y LegacySync**

Bloque `8000-8999` (backend APIs), patrón `80XY`:

* `X` \= capa: `0` IIS Express http, `1` IIS Express sslPort, `2` perfil `http`, `3` perfil `https`.  
* `Y` \= identificador de proyecto: `1` Mailbot, `2` LegacySync.

| Proyecto | IIS Express http | IIS Express sslPort | Perfil http | Perfil https |
| :---- | :---- | :---- | :---- | :---- |
| Mailbot.Api (id 1\) | `8001` | `8011` | `8021` | `8031` |
| LegacySync.Api (id 2\) | `8002` | `8012` | `8022` | `8032` |

Historial: LB-T2 primero asignó a `LegacySync.Api` valores únicos pero ad-hoc (`53210`/`44395`/`5290`/`7290`, commit `b7457f9`) solo para dejar de chocar con Mailbot. Carlos pidió después una convención real, aplicable a ambos proyectos; el esquema `80XY` la reemplaza (`LegacySync.Api` commit `3b90d52`, `Mailbot.Api` commit `c7135d7` — PR \[proquifa-pqf2-mailbot-backend\#4\](https://github.com/ryndem/proquifa-pqf2-mailbot-backend/pull/4), ya que T13-A estaba cerrada y fusionada). El siguiente proyecto derivado del arquetipo (p. ej. Finanzas/T5) tomaría el identificador `5` — ver §2.2 y §5. Este esquema `80XY` es la aplicación concreta, solo sobre el bloque de APIs backend, de la convención general de §6.

## **2.2 Propuesta para los dos repos externos/históricos (pendiente de aplicar)**

Los dos proyectos que hoy comparten los puertos duplicados del arquetipo (`proquifa-punchout-backend`, `ExchangeRateService`) quedan fuera del control directo del equipo, pero pueden adoptar el mismo esquema `80XY` de §2.1 si algún desarrollador con acceso al repo lo considera conveniente. Esto es una **propuesta de este inventario, no un cambio aplicado** — a diferencia de Mailbot y LegacySync (§2.1), acá no se tocó ningún `launchSettings.json`.

| Proyecto | Id propuesto | IIS Express http | IIS Express sslPort | Perfil http | Perfil https |
| :---- | :---- | :---- | :---- | :---- | :---- |
| `proquifa-punchout-backend/API` | 3 | `8003` | `8013` | `8023` | `8033` |
| `ExchangeRateService/API` | 4 | `8004` | `8014` | `8024` | `8034` |

El arquetipo (`microservice-clean-architecture-template/API`) no recibe id propio en esta propuesta: es la plantilla fuente, no una instancia que corra en local en paralelo con las demás — el problema real está en los derivados que la copiaron sin ajustarla.

## **2.3 Vista consolidada**

![][image1]

```
flowchart TB    subgraph Duplicados["Puertos duplicados heredados del arquetipo<br/>32254 / 44385 / 5180 / 7190"]        T["microservice-clean-architecture-template<br/>(arquetipo fuente, sin id propio)"]        P["proquifa-punchout-backend"]        Ex["ExchangeRateService"]    end    subgraph Resuelto["Esquema 80XY aplicado — §2.1"]        M["Mailbot.Api — id 1<br/>8001/8011/8021/8031"]        L["LegacySync.Api — id 2<br/>8002/8012/8022/8032"]    end    subgraph Propuesto["Esquema 80XY propuesto, pendiente — §2.2"]        P2["proquifa-punchout-backend → id 3<br/>8003/8013/8023/8033"]        Ex2["ExchangeRateService → id 4<br/>8004/8014/8024/8034"]    end    P -.-> P2    Ex -.-> Ex2
```

# **3\. Workers sin puerto HTTP (`BackgroundService` puro, sin `applicationUrl`)**

Estos no exponen HTTP y por lo tanto no pueden chocar por puerto entre sí ni con las APIs de arriba:

* `microservice-clean-architecture-template/Worker`  
* `proquifa-pqf2-mailbot-backend/Mailbot.Worker`  
* `proquifa-pqf2-legacysync-backend/LegacySync.Worker`  
* `proquifa-punchout-backend/Worker`  
* `proquifa-pqf2-importador-worker/Worker`  
* `WorkerTraking-backend/WorkerTracking`

# **4\. .NET Framework clásico — bindings de IIS Express (local a esta máquina)**

Estos proyectos no usan `launchSettings.json`; los puertos quedan en `.vs/<Solución>/config/applicationhost.config`, que **no se versiona** y Visual Studio puede regenerar distinto en otra máquina. Se documentan aquí solo como fotografía del estado actual en este equipo:

| Repo | Sitio IIS Express | Puerto local |
| :---- | :---- | :---- |
| `ProquifaDotNet` | `WebSite1` | `8080` (http) |
| `ProquifaDotNet` | `WebApi.Catalogos` | `3851` (http) |
| `ProquifaDotNet` | `WebApi.Finanzas` | `9763` (http) |
| `ProquifaDotNet` | `WebApi.Logistica` | `3148` (http) |
| `ProquifaDotNet` | `WebApi.Stp` | `3940` (http) / `44326` (https) |
| `Core.Pqf` | `WebSite1` | `8080` (http) |
| `Core.Pqf` | `ElNugetero` | `2088` (http) / `44326` (https) |

`ProquifaDotNet-R14` se omite de este inventario: es un fork de `ProquifaDotNet`, no una solución aparte.  
**Colisiones ya presentes en esta máquina** (no se tocan, solo se registran):

* `WebSite1:8080` se repite en `ProquifaDotNet` y `Core.Pqf` — no se pueden tener ambas soluciones abiertas en Visual Studio con IIS Express corriendo al mismo tiempo sin que una falle al bindear el puerto.  
* `44326` (https) se repite entre `ProquifaDotNet/WebApi.Stp` y `Core.Pqf/ElNugetero`.

`Core.CrudTools`, `interfaces-proquifanet2`, `proquifa-pqf2-finanzas-api` (solo `README.md` por ahora, ver `TPSC-RE-FU-008-Implementacion-T5.md`) y otros repos de la lista no tienen `launchSettings.json` ni `applicationhost.config` con sitios propios — no son proyectos Web o no se han ejecutado localmente todavía.

# **5\. Rangos ya ocupados (referencia para el próximo scaffold)**

Si se deriva una nueva solución del arquetipo (`proquifa-pqf2-finanzas-api`/T5, o cualquier otra) y el equipo decide seguir el esquema `80XY` de §2.1, el siguiente identificador **realmente libre es 5** (`8005`/`8015`/`8025`/`8035`): los ids `3` y `4` quedan reservados por la propuesta pendiente de §2.2 para `proquifa-punchout-backend` y `ExchangeRateService` (aunque todavía no se hayan aplicado). Si en cambio se conserva sin cambios el `launchSettings.json` copiado del arquetipo, estos son los valores que **ya están en uso** en el resto del ecosistema y hay que evitar:

* IIS Express: `32254`, `51071`, `5000`, `17212`  
* SSL: `44385`, `44375`, `44333`, `44326`  
* Perfil `http`: `5180`, `5052`, `5174`  
* Perfil `https`: `7190`, `7101`  
* Bloque `8000-8999` (esquema `80XY`): `8001`/`8011`/`8021`/`8031` (Mailbot) y `8002`/`8012`/`8022`/`8032` (LegacySync) ya asignados y aplicados; `8003`/`8013`/`8023`/`8033` (punchout-backend) y `8004`/`8014`/`8024`/`8034` (ExchangeRateService) propuestos en §2.2, pendientes de aplicar  
* IIS Express clásico (.NET Framework): `8080`, `3851`, `9763`, `3148`, `3940`, `2088`

# **6\. Convención de referencia — segmentación por bloques (compartida por Carlos, 24/07/2026)**

Regla general para organizar puertos de múltiples proyectos sin que choquen, más allá de los dos proyectos .NET de §2.1 — sirve como marco de referencia si el ecosistema suma frontend, microfrontends o bases de datos en contenedor:

## **6.1 Identificador por proyecto (dígito dentro de cada rango común)**

Cada proyecto recibe un dígito identificador que se repite en todas sus capas:

| Proyecto | Frontend | Backend | Base de datos |
| :---- | :---- | :---- | :---- |
| Proyecto 1 (id 0/1) | `3000` | `8000` | `5432` |
| Proyecto 2 (id 2\) | `3002` | `8002` | `5434` |
| Proyecto 3 (id 3\) | `3003` | `8003` | `5435` |

## **6.2 Bloques completos por propósito**

| Bloque / Rango | Propósito | Ejemplo |
| :---- | :---- | :---- |
| `3000-3999` | Frontend general | `3001` (Proy A), `3002` (Proy B) |
| `4000-4999` | Microfrontends / Angular | `4200` (Main), `4201` (Auth) |
| `5000-5999` | Bases de datos locales (contenedores Docker, para no saturar las nativas) | — |
| `8000-8999` | APIs y Gateways (backend) | `8080` (Microservicio 1), `8081` (Microservicio 2\) |
| `9000-9999` | Herramientas de DevOps (paneles, webhooks, proxies, métricas) | — |

Vista de los bloques en orden numérico:  
![][image2]

```
flowchart LR    B3["3000-3999<br/>Frontend general<br/>(reservado, sin asignar)"]    B4["4000-4999<br/>Microfrontends / Angular<br/>(reservado, sin asignar)"]    B5["5000-5999<br/>Bases de datos locales<br/>(reservado, sin asignar)"]    B8["8000-8999<br/>APIs y Gateways<br/>(en uso — esquema 80XY)"]    B9["9000-9999<br/>DevOps<br/>(reservado, sin asignar)"]    B3 --- B4 --- B5 --- B8 --- B9
```

## **6.3 Aplicación a este ecosistema hoy**

Todos los proyectos de §1 son backend .NET puro, sin frontend ni contenedor de BD propio — por eso solo está en uso el bloque `8000-8999` (esquema `80XY` de §2.1). Los bloques `3000-3999`, `4000-4999`, `5000-5999` y `9000-9999` quedan reservados sin asignar: se activarían si el ecosistema PQF2 suma un frontend propio (más allá de `FrontProquifaNet`, que no vive en este equipo), microfrontends, bases de datos en contenedor local, o paneles/herramientas de DevOps.

# **Control de versiones** {#control-de-versiones}

| Versión | Fecha | Autor | Tipo de Cambio | Descripción del cambio | Aprobó |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 1.0 | 27 jul 2026 | [Carlos Iván Morales Carreón](mailto:carlos.morales@ryndem.mx) | Creación | Creación del documento. |  |
|  |  |  |  |  |  |
