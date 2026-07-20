# Análisis — Separación operativa PROQUIFA / GOLOCAER

> **Contexto:** Reorganización societaria previa al proyecto R16 (ProquifaDotNet). Se plantea separar la operación de PROQUIFA (facturación exclusiva de USP) de la de GOLOCAER (resto de marcas / proveedores), con el fin de mantener aislada la trazabilidad financiera y operativa ante una eventual auditoría.

---

## 1. Antecedentes del cambio

- **PROQUIFA** será la empresa que facture **USP**. Implica que:
  - Será la **única** empresa autorizada para comprarle al proveedor USP.
  - Será la **única** que comercialice toda la línea USP con los clientes.
- **GOLOCAER** absorberá la operación del **resto de marcas / proveedores** (los que hoy se distribuyen entre Proquifa, Proveedora, Golocaer, Ganilab, etc.).
- La operación de ambas empresas debe quedar **totalmente separada** para soportar auditorías (fiscales, regulatorias y de comercio exterior).
- La propuesta actual es **duplicar los sistemas**, generando una instancia de ProquifaDotNet por cada empresa comercializadora.

---

## 2. Situación actual — Matriz de combinación de partidas

Hoy ProquifaDotNet aplica reglas de combinación de productos dentro de una misma cotización en función de:

1. La **empresa que factura** (Proquifa, Proveedora, Golocaer, Mungen, Pharma, Ganilab, Golocaer SAC).
2. Si esa empresa **también factura publicaciones** (`Check` — TRUE / FALSE).
3. El **tipo del primer producto** agregado a la partida: Normal, Mundial/Nacional/Origen (Controlado), Servicios, Publicaciones, o Control N/A (todo lo que no es servicio ni publicación — Labware, Capacitaciones, Dispositivos Médicos).

### 2.1 Especialización por empresa

| Empresa                     | Rol comercial                                                                                                                                |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Proquifa**                | Normales                                                                                                                                     |
| **Proveedora**              | Controlados (Mundial / Nacional / Origen)                                                                                                    |
| **Golocaer**                | Publicaciones                                                                                                                                |
| **Ganilab**                 | Servicios                                                                                                                                    |
| **Golocaer SAC**            | Multiproducto — Perú (permite todo)                                                                                                          |
| **Mungen / Pharma / otros** | Se ven afectadas por el **tipo del primer producto**. Si es Normal o Control N/A, no cambia la razón social y se emite con formato Proquifa. |

### 2.2 Catálogo de mensajes de validación

| Código | Mensaje |
|---|---|
| **msg1** | No es posible agregar esta publicación, asegúrate de que la empresa que factura pueda facturar publicaciones. |
| **msg2** | No es posible agregar este producto, no puedes mezclar controlados y normales en la misma cotización. |
| **msg3** | No es posible agregar este producto, no puedes mezclar controlados y publicaciones. |
| **msg4** | No es posible agregar este producto, no puedes agregar otro producto que no sea publicación por el cambio de razón social a golocaer. |
| **msg5** | No es posible agregar este producto, no puedes agregar otro producto que no sea servicios. |
| **msg6** | No es posible agregar este producto, no puedes combinar productos de tipo servicios. |

### 2.3 Matriz consolidada (situación actual)

Leyenda: **1º** = se agrega primero; **✔** = se puede combinar; **✖(msgN)** = bloqueo con mensaje; **→X** = cambia quien factura a la empresa X al guardar.

| # | Quien factura | Check | Normal | Controlado (Mundial/Nal/Origen) | Servicios | Publicaciones | Control N/A | Cambio |
|---|---|---|---|---|---|---|---|---|
| 1 | Proquifa, Mungen, Pharma, otros | FALSE | 1º | ✖(msg2) | ✖(msg6) | ✖(msg1) | ✔ | N/A |
| 2 | Proquifa, Mungen, Proveedora, Pharma, Ganilab | FALSE | ✖(msg4) | ✖(msg3) | ✖(msg6) | 1º | ✔ | →Golocaer |
| 3 | Proquifa, Mungen, Proveedora, Pharma, otros | FALSE/TRUE | ✖(msg5) | ✖(msg5) | 1º | ✖(msg5) | ✖(msg5) | →Ganilab |
| 4 | Proquifa, Mungen, Pharma, otros | TRUE | ✔ | ✖(msg) | ✖(msg6) | ✔ | ✔ | N/A |
| 5 | Proquifa, Mungen, Golocaer, Pharma, Ganilab, otros | FALSE/TRUE | ✖(msg) | 1º | ✖(msg6) | ✖(msg) | ✔ | →Proveedora |
| 6 | Proveedora | FALSE | ✖(msg2) | ✔ | ✖(msg6) | ✖(msg3) | ✔ | N/A |
| 7 | Proveedora | TRUE | ✖(msg2) | ✔ | ✖(msg6) | ✔ | ✔ | N/A |
| 8 | Golocaer SAC (Perú) | TRUE/FALSE | ✔ | ✔ | ✔ | ✔ | ✔ | N/A |
| 9 | Ganilab | FALSE | ✖(msg5) | ✖(msg5) | 1º | ✖(msg5) | ✖(msg5) | N/A |
| 10 | Ganilab | TRUE | ✖(msg5) | ✖(msg5) | ✔ | ✖(msg5) | ✖(msg5) | N/A |
| 11 | Golocaer | N/A | ✖(msg) | 1º | ✖(msg6) | ✖(msg) | ✔ | →Proveedora |
| 12 | Golocaer | N/A | 1º | ✖(msg2) | ✖(msg6) | ✖(msg1) | ✔ | N/A |
| 13 | Golocaer | N/A | ✖(msg4) | ✖(msg3) | ✖(msg6) | 1º | ✔ | N/A |

> **Bloques indefinidos:** algunas celdas apuntan a `(*mensaje)` sin un código de mensaje formalizado. Corresponden a **indefiniciones o inconsistencias de texto** actuales: o no deberían aplicar, o deberían resolverse con un mensaje nuevo. Se debe cerrar este catálogo antes de la separación.

### 2.4 Comportamientos transversales relevantes

- **Servicios** es siempre excluyente: en cuanto se agrega un servicio, el carrito bloquea cualquier otro tipo (msg5 / msg6) y la razón social cambia a **Ganilab** al guardar.
- **Publicaciones** disparan la ruta *change_to_golocaer* cuando la empresa que factura no puede facturar publicaciones (`Check = FALSE`).
- **Controlados vs normales** nunca se combinan (msg2), independientemente de la empresa.
- **Control N/A** (Labware, Capacitaciones, Dispositivos Médicos) combina con la mayoría de escenarios excepto cuando la partida arranca con servicios.
- **Mungen y Pharma** son marcas comerciales que no cambian la razón social salvo que el primer producto obligue a ello.
- **El punto de guardado importa:** las validaciones difieren si la partida se guarda inmediatamente después del primer producto o al final. En las pruebas base el guardado se hizo al final.

### 2.5 Divergencia Front vs Back (importante)

- **Front:** aplica las validaciones descritas en §2.3 con la granularidad completa por tipo de producto.
- **Back:** cuando la **primera partida** agregada es de tipo **Control N/A** (Labware, Capacitación, Dispositivo Médico), el motor la trata como si fuera un **Normal**. Consecuencia: bloquea agregar posteriormente un controlado, una publicación u otro tipo que exigiría el cambio de razón social, aunque el Front sí lo permita.
- Este desfase Front / Back es un **riesgo abierto** que debe cerrarse antes o durante la separación, para que la matriz colapsada de PROQUIFA y la matriz vigente de GOLOCAER se comporten de forma determinística.

---

## 3. Impacto del cambio propuesto

### 3.1 Impacto funcional en cotización y facturación

- **PROQUIFA** debe quedar como una operación autocontenida centrada en la línea USP. Requiere:
  - Catálogo de marcas **reducido a USP**.
  - Reglas de combinación de partidas simplificadas (una sola línea de producto: la de USP).
  - Bloqueo de mezcla con cualquier producto que no pertenezca al catálogo autorizado.
- **GOLOCAER** hereda **todas las marcas restantes** y por lo tanto retiene la matriz de combinación actual (Normales, Controlados, Servicios, Publicaciones, Labware/Capacitaciones/Dispositivos Médicos).
- Desaparece — o al menos deja de ser necesaria — la lógica de *cambio de razón social* que hoy convierte una cotización Proquifa → Proveedora → Golocaer → Ganilab al guardar, salvo por el pivote entre marcas dentro de la operación GOLOCAER.

### 3.2 Impacto en sistemas (duplicación)

Se propone **duplicar la solución ProquifaDotNet** para tener dos instancias independientes:

| Aspecto | Instancia PROQUIFA (USP) | Instancia GOLOCAER (resto) |
|---|---|---|
| Base de datos | `ProquifaDotNet` (dedicada) | `ProquifaDotNet` (dedicada) |
| ERP / Legacy | Legado con RFC PROQUIFA | Legado con RFC GOLOCAER |
| Timbrado (CFDI) | Serie / certificado PROQUIFA | Serie / certificado GOLOCAER |
| Almacén / inventario | Sólo USP | Multi-marca |
| Compras | Sólo proveedor USP | Resto de proveedores |
| Cuentas por cobrar / cobros | Cartera PROQUIFA | Cartera GOLOCAER |
| Módulo Finanzas (.NET Core 10) | Instancia dedicada | Instancia dedicada |
| Módulo Timbrado (.NET Core 10) | Instancia dedicada | Instancia dedicada |
| DocumentBuilder | Plantillas con marca PROQUIFA | Plantillas con marca GOLOCAER |

### 3.3 Impacto en datos maestros

- **Clientes:** un mismo cliente puede comprar a las dos empresas; debe existir un mecanismo de conciliación (mismo RFC / mismo ID cliente en ambos entornos) para reportería consolidada.
- **Productos:** la línea USP se retira del catálogo GOLOCAER y viceversa. Requiere un **script de migración** que identifique claves USP y las marque como no comercializables en la instancia GOLOCAER.
- **Proveedores:** USP queda únicamente en PROQUIFA. Todos los demás proveedores se dan de baja en la instancia PROQUIFA.
- **Marcas:** partición estricta — USP en PROQUIFA; resto (Merck, Sigma, publicaciones, servicios, Labware, etc.) en GOLOCAER.

### 3.4 Impacto en integraciones y ETLs

Considerando las transferencias hoy existentes hacia el legado (Marcas, Proveedores, Productos, Clientes, Buzones, Cotizaciones, PSC, PCC, Facturas, Notas de Crédito, Cobros, PDF):

- Cada instancia requerirá su **propio pipeline de ETL** apuntando al legado correspondiente.
- Los **buzones** (Cotización, Pedidos, Pagos) deben duplicarse; recomendamos prefijar por empresa (`BZ_PQF_*`, `BZ_GOL_*`) para evitar colisiones.
- IdentityServer, Brevo, MinIO y RabbitMQ pueden mantenerse **compartidos** con particionado lógico (tenant / vhost por empresa) o duplicarse. La decisión se recomienda tomar por riesgo y costo (ver §5).

---

## 4. Impacto en la matriz de combinación

La matriz actual (§2.3) se **colapsa** de la siguiente forma:

- **PROQUIFA (USP):**
  - Sólo maneja la línea USP → la única regla efectiva es **"se agrega primero" para el producto USP** y **bloqueo total para cualquier producto que no sea USP**.
  - Se puede reutilizar msg5 (`no puedes agregar otro producto que no sea …`) con un nuevo texto: *"no puedes agregar otro producto que no sea USP en esta empresa"*.
  - Se **elimina** de esta instancia la lógica de cambio de razón social a Proveedora, Ganilab o Golocaer (renglones 2, 3, 5 de la matriz).
  - Los renglones 1, 4, 6 y 7 se reducen a un único caso trivial: sólo se aceptan productos USP.
- **GOLOCAER (resto de marcas):**
  - Conserva íntegramente la matriz actual **excepto la línea USP**.
  - Los renglones 1, 2, 3, 4, 5 (que hoy incluyen "Proquifa" como emisor) permanecen, pero Proquifa como razón social **desaparece o queda inactiva** dentro de esta instancia.
  - Debe decidirse si su rol (Normales) se **absorbe por Golocaer** como razón social por defecto, o si se conserva la marca "Proquifa" únicamente como identidad comercial sin efecto fiscal en esta instancia.
  - Los renglones 6–10 (Proveedora, Ganilab) y 8 (Golocaer SAC — Perú) se mantienen sin cambio.
  - Los renglones 11–13 (Golocaer) se mantienen sin cambio.
- **Bloques indefinidos (`*mensaje` sin código):** deben resolverse antes del corte para no arrastrar deuda de mensajería a dos entornos.
- **Divergencia Front / Back (§2.5):** debe cerrarse para que ambos entornos separados se comporten igual en Control N/A.

### 4.1 Escenarios a validar con negocio

1. ¿La línea USP incluye **controlados**, **servicios asociados**, **capacitaciones** o **publicaciones**? Si sí, la instancia PROQUIFA no puede ser sólo "normales" y debe replicar submatriz.
2. ¿Un cliente puede pedir en una **misma orden** productos USP y productos de otras marcas? Si sí, se necesita un flujo de **cotización doble** (una cotización por empresa, con un identificador de orden compartido).
3. Para **notas de crédito, devoluciones y refacturación**, ¿el flujo cruza empresas? Si sí, se requiere un contrato de servicios interempresa.
4. ¿Se conserva la operación **Perú (Golocaer SAC)** en la instancia GOLOCAER o se separa a un tercer entorno?

---

## 5. Riesgos y consideraciones

| # | Riesgo | Mitigación |
|---|---|---|
| R1 | **Duplicidad de datos maestros** (clientes, productos, marcas) genera inconsistencias | Implementar un **MDM ligero** (Master Data) que replique de una fuente única a las dos instancias vía RabbitMQ. |
| R2 | Cliente que compraba productos combinados hoy — ya no puede en un solo pedido | Definir política comercial y comunicación al cliente; permitir cotización paralela con folio maestro. |
| R3 | Costos de infraestructura duplicada (BD, VMs, licencias) | Evaluar si IdentityServer, MinIO, RabbitMQ, Brevo pueden compartirse (multi-tenant) o requieren separación estricta por auditoría. |
| R4 | Reportería financiera consolidada (grupo) requiere unificar información | Diseñar un **datawarehouse** que consuma de ambas instancias vía ETL. |
| R5 | Cambios regulatorios (SAT, comercio exterior) requieren aplicarse en dos entornos | Establecer pipeline CI/CD que promueva a ambos entornos desde un solo repositorio. |
| R6 | La lógica de "cambio de empresa al guardar" desaparece en PROQUIFA pero sigue viva en GOLOCAER | Refactorizar el módulo de cotización con un **feature flag por tenant** (`AllowMultiCompanyPivot`). |
| R7 | Timbrado fiscal con dos RFC obliga a mantener dos series y certificados | ProquifaDotNet.Timbrado debe soportar **multi-tenant** con configuración por empresa. |
| R8 | Auditoría exige **inmutabilidad** de la operación separada | Bloquear a nivel BD / Identity que un usuario opere transversalmente sin traza. |
| R9 | **Divergencia Front / Back para Control N/A** (§2.5) se arrastra a los dos entornos si no se resuelve antes del corte | Unificar la lógica de tipificación del primer producto en Back para que Control N/A no sea tratado como Normal. Cubrir con pruebas E2E antes del corte. |
| R10 | Bloques indefinidos (`*mensaje` sin código) en la matriz actual | Cerrar catálogo de mensajes (msg1–msg6 + nuevos) y documentarlos como constantes reutilizables por ambos tenants. |

---

## 6. Recomendaciones

1. **Confirmar alcance del catálogo USP** antes de definir el nivel de duplicación. Si USP es sólo "normales", el modelo simplificado es directo; si abarca varios tipos de producto, se necesita replicar submatriz.
2. **Evaluar arquitectura multi-tenant vs duplicación física**. La duplicación física es más segura ante auditoría; el multi-tenancy es más económico. Recomendamos:
   - **BD:** separada (una por empresa) — reduce riesgo de fuga de datos entre entidades.
   - **Servicios transversales** (IdentityServer, MinIO, RabbitMQ, Brevo): compartidos con particionado por tenant.
   - **API y Worker:** dos deployments independientes con el mismo binario y configuración por variable de entorno.
3. **Crear un requisito específico** dentro del proyecto R16 con clave `[TPSC-RE-FU-SEP]` para trackear la separación PROQUIFA / GOLOCAER, con tareas de:
   - Migración de catálogos (marcas, productos, proveedores).
   - Duplicación de esquema BD ProquifaDotNet.
   - Ajustes al módulo de cotización (colapso de matriz de combinación).
   - Configuración multi-tenant en Finanzas y Timbrado.
   - Duplicación de plantillas en DocumentBuilder.
   - ETLs por empresa hacia legado.
4. **Congelar la matriz actual** como línea base (este documento) y comparar con la matriz objetivo tras el cambio para dimensionar el impacto en pruebas.
5. **Diseñar un plan de migración de cotizaciones abiertas** al momento del corte: qué pasa con las que hoy mezclan productos USP y no USP.
6. **Definir el manejo de la operación Perú (Golocaer SAC)** — recomendamos dejarla en la instancia GOLOCAER dado que no comercializa USP.

---

## 7. Decisión inicial — MVP con infraestructura separada

> **Decisión de arranque:** el proyecto arranca con un **MVP donde la infraestructura de cada empresa está físicamente separada**. Esta decisión precede al detalle técnico y condiciona todas las secciones anteriores. Las preguntas abiertas de §8 se responden bajo este marco.

### 7.1 ¿Qué significa "infraestructura separada" en el MVP?

Dos entornos técnicos completamente independientes, uno por empresa (PROQUIFA y GOLOCAER), sin dependencias compartidas en el plano de datos ni de servicios.

| Componente                                         | PROQUIFA                                            | GOLOCAER                                            | Compartido                           |
| -------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------- | ------------------------------------ |
| Base de datos `ProquifaDotNet`                     | Instancia propia (servidor o catálogo distinto)     | Instancia propia                                    | ✖                                    |
| Servidor de aplicaciones (IIS / Kestrel)           | Deployment propio                                   | Deployment propio                                   | ✖                                    |
| ProquifaDotNet API (.NET Framework 4.8)            | Binario compartido, config propia                   | Binario compartido, config propia                   | Binario en repo                      |
| DocumentBuilder                                    | Instancia propia (plantillas con branding PROQUIFA) | Instancia propia (plantillas con branding GOLOCAER) | ✖                                    |
| IdentityServer                                     | Instancia propia                                    | Instancia propia                                    | ✖                                    |
| RabbitMQ                                           | Broker propio                                       | Broker propio                                       | ✖                                    |
| MinIO                                              | Bucket / bóveda propia                              | Bucket / bóveda propia                              | ✖                                    |
| Brevo                                              | Cuenta / API key propia                             | Cuenta / API key propia                             | ✖                                    |
| Certificados de timbrado (CSD)                     | Almacén propio, RFC PROQUIFA                        | Almacén propio, RFC GOLOCAER                        | ✖                                    |
| Repositorio de código                              | —                                                   | —                                                   | ✔ (un solo repo)                     |
| Pipeline CI/CD                                     | —                                                   | —                                                   | ✔ (un solo pipeline con dos targets) |
| Catálogos SAT (regímenes fiscales, uso CFDI, etc.) | Copia sembrada                                      | Copia sembrada                                      | ✖ (copia idéntica en cada BD)        |

### 7.2 Principios del MVP

- **Cero acoplamiento en runtime** entre PROQUIFA y GOLOCAER: un incidente en un entorno no afecta al otro.
- **Un solo binario, dos configuraciones**: el mismo build se despliega en ambos entornos con `Web.config` / `appsettings` distintos.
- **Un solo repositorio** de código, sin fork. Los flags de comportamiento (marcas permitidas, prefijo, RFC, CSD) viven en configuración.
- **Sin comunicación cross-tenant** en el MVP: no hay APIs entre PROQUIFA y GOLOCAER, no hay colas cruzadas, no hay conciliación automática.
- **Reportería consolidada queda fuera del MVP**: si negocio la requiere, se atiende manualmente (extractos exportados) hasta que se defina un DWH.

### 7.3 Alcance IN del MVP

| #      | Alcance                            | Descripción                                                                                         |
| ------ | ---------------------------------- | --------------------------------------------------------------------------------------------------- |
| MVP-1  | Duplicación de BD `ProquifaDotNet` | Dos instancias, esquema idéntico, catálogos sembrados independientes.                               |
| MVP-2  | Deployment aplicativo x2           | Dos entornos de ProquifaDotNet API, con `Web.config` por empresa.                                   |
| MVP-3  | Colapso de matriz PROQUIFA         | Restringir catálogo a USP; deshabilitar cambio de razón social; ajustar mensajes de validación.     |
| MVP-4  | Conservación de matriz GOLOCAER    | La instancia GOLOCAER conserva la matriz actual sin la línea USP.                                   |
| MVP-5  | Datos maestros por empresa         | Marcas, proveedores, productos, contratos cliente-marca filtrados por empresa al momento del corte. |
| MVP-6  | Timbrado por empresa               | CSD, series y folios propios por empresa.                                                           |
| MVP-7  | PDF por empresa                    | Plantillas DocumentBuilder con RFC / razón social / logotipo por empresa.                           |
| MVP-8  | ETL a legacy por empresa           | Cada instancia empuja a su legacy correspondiente (o etiqueta con `IdEmpresa`).                     |
| MVP-9  | Buzones por empresa                | Dominios / alias / carpetas distintas por empresa.                                                  |
| MVP-10 | Usuarios y permisos por empresa    | Cada usuario opera en un solo entorno; sin SSO cross-tenant en MVP.                                 |
| MVP-11 | Migración de catálogos y saldos    | Script one-shot al momento del corte para separar cotizaciones abiertas, cartera y pendientes.      |
| MVP-12 | Plan de corte y rollback           | Ventana definida, procedimiento de fallback y checklist de smoke tests.                             |

### 7.4 Alcance OUT del MVP (postergado para siguientes iteraciones)

- **Cotización cross-company** (folio maestro que agrupe dos cotizaciones).
- **Reportería consolidada** en tiempo real / DWH grupal.
- **MDM** (replicación de clientes y catálogos base entre entornos).
- **SSO** único con `tenant_id` en el token de IdentityServer.
- **Multi-tenant** en la BD (RLS por `IdEmpresa`).
- **Interfacturación** automática entre PROQUIFA y GOLOCAER.
- **Refacturación histórica** desde la razón social anterior.

### 7.5 Arquitectura objetivo del MVP

```
┌───────────────── Cliente / Usuario Interno ──────────────────┐
│                                                              │
│   https://app-proquifa.local          https://app-golocaer.local
│           │                                       │          │
└───────────┼───────────────────────────────────────┼──────────┘
            ▼                                       ▼
   ┌────────────────────┐               ┌────────────────────┐
   │ ProquifaDotNet API │               │ ProquifaDotNet API │
   │ (PROQUIFA)         │               │ (GOLOCAER)         │
   │ Web.config:        │               │ Web.config:        │
   │   Empresa=PROQUIFA │               │   Empresa=GOLOCAER │
   │   RFC=XXX          │               │   RFC=YYY          │
   └────────┬───────────┘               └────────┬───────────┘
            │                                    │
   ┌────────▼───────────┐               ┌────────▼───────────┐
   │ SQL Server         │               │ SQL Server         │
   │ ProquifaDotNet PQF │               │ ProquifaDotNet GOL │
   └────────────────────┘               └────────────────────┘
            │                                    │
   ┌────────▼───────────┐               ┌────────▼───────────┐
   │ RabbitMQ / MinIO   │               │ RabbitMQ / MinIO   │
   │ IdentityServer PQF │               │ IdentityServer GOL │
   │ DocumentBuilder PQF│               │ DocumentBuilder GOL│
   └────────┬───────────┘               └────────┬───────────┘
            │                                    │
   ┌────────▼───────────┐               ┌────────▼───────────┐
   │ Legacy (RFC PQF)   │               │ Legacy (RFC GOL)   │
   └────────────────────┘               └────────────────────┘
```

### 7.6 Trade-offs asumidos con el MVP

| Ventaja | Contra |
|---|---|
| Cumple auditoría de manera estricta desde día uno. | Duplica costo de infraestructura (BD, VMs, licencias, brokers). |
| Aísla incidentes: un fallo en PROQUIFA no impacta GOLOCAER. | Un cambio de esquema requiere migración doble. |
| Habilita rollback por entorno. | Duplica esfuerzo operativo (soporte, monitoreo, respaldos). |
| Deja libre a futuro consolidar o compartir servicios (reversible). | La reportería consolidada requiere trabajo adicional. |
| No requiere refactor de RLS ni de multi-tenant en el aplicativo. | Los usuarios que operen en ambas empresas necesitan dos cuentas hasta que exista SSO cross-tenant. |

### 7.7 Cambios mínimos en el aplicativo para soportar el MVP

Dado que el MVP corre el **mismo binario** en dos entornos, los cambios en código son los mínimos que permiten desacoplarlo del literal `"proquifa"` / `"golocaer"`:

1. **Externalizar identidad de empresa** — `Empresa.Clave == "golocaer"` deja de ser literal; se lee de la configuración `Empresa.Clave` de la instancia actual.
2. **Externalizar mensajes de validación** — msg1–msg6 pasan a `catMensajeValidacionCotizacion` con override por instancia.
3. **Deshabilitar cambio de razón social en PROQUIFA** — feature flag `AllowCompanyPivotOnSave` = `false` en PROQUIFA, `true` en GOLOCAER.
4. **Restringir catálogo por empresa** — la instancia PROQUIFA sólo muestra marcas USP; se implementa con `MarcaEmpresa` (N:N) o `Marca.IdEmpresaExclusiva`.
5. **Endpoints intactos** — no se agregan APIs entre PROQUIFA y GOLOCAER en el MVP.

Ver el análisis técnico complementario en `02 Impacto-Back-BD-PROQUIFA-GOLOCAER.md`.

### 7.8 Ruta de implementación sugerida

| Fase                        | Semanas (indicativas) | Entregables                                                                                                                             |
| --------------------------- | --------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| F0 — Definiciones           | 1–2                   | Respuestas §8.0 (Fundamentales), aprobación patrocinio, contrato USP, opinión del auditor.                                              |
| F1 — Preparación de BD      | 2                     | Enriquecer `Empresa` (`FacturaUSP`, `FacturaNormales`, `FacturaLabware`), crear `MarcaEmpresa`, crear `catMensajeValidacionCotizacion`. |
| F2 — Refactor mínimo Back   | 2–3                   | Externalizar mensajes y `Empresa.Clave`; feature flag `AllowCompanyPivotOnSave`; cerrar divergencia Front/Back Control N/A.             |
| F3 — Infraestructura x2     | 2                     | Provisionar servidores, BD, RabbitMQ, MinIO, IdentityServer, DocumentBuilder para cada empresa.                                         |
| F4 — Migración de catálogos | 2                     | Script one-shot: marcas USP → PROQUIFA; resto → GOLOCAER; separación de cotizaciones abiertas.                                          |
| F5 — Pruebas E2E            | 2                     | Suite parametrizable por empresa; smoke tests; simulacro de corte.                                                                      |
| F6 — Corte                  | 1                     | Ventana definida; comunicación a clientes/proveedores/empleados; monitoreo intensivo primeras 72 h.                                     |
| F7 — Estabilización         | 2                     | Ajustes finos; postmortem; plan para OUT del MVP.                                                                                       |

Total estimado: 12–14 semanas (a validar con negocio y equipo).

### 7.9 Riesgos específicos del MVP con infraestructura separada

| # | Riesgo | Mitigación |
|---|---|---|
| MVP-R1 | Un mismo usuario que opere las dos empresas necesita **dos cuentas** hasta que exista SSO cross-tenant. | Aceptado en el MVP. Priorizar SSO en fase 2. |
| MVP-R2 | Catálogos base (SAT, códigos postales, monedas) quedan **duplicados**; una actualización requiere aplicarla en ambos. | Script de sincronización manual o programada; convertir en tarea recurrente. |
| MVP-R3 | Cliente que pide productos USP y no-USP recibe **dos cotizaciones** sin folio maestro. | Aceptado; documentar procedimiento comercial para explicar al cliente. |
| MVP-R4 | Reportería del grupo queda **manual** durante el MVP. | Aceptado; usar extractos exportados con `IdEmpresa` visible; DWH en fase 2. |
| MVP-R5 | **Migración de saldos y pendientes** al momento del corte puede dejar registros sin destino claro (cotizaciones mezcladas USP/no-USP). | Reporte previo al corte listando esos casos; decisión caso-por-caso con negocio. |
| MVP-R6 | Costo de infraestructura duplicado. | Aceptado como trade-off por auditoría; revisar en fase 2 si algún componente puede compartirse sin comprometer separación. |
| MVP-R7 | Divergencia entre entornos si un despliegue falla en uno y no en el otro. | Pipeline atómico que promueve juntos o hace rollback; smoke tests obligatorios post-deploy. |

### 7.10 Métricas de éxito del MVP

- **Operativas:** 100 % de facturas emitidas post-corte con el RFC correcto por empresa; 0 cruces de datos entre entornos.
- **De auditoría:** posibilidad demostrable de extraer toda la operación de una sola empresa sin necesidad de filtrar la otra.
- **De estabilidad:** disponibilidad ≥ 99 % en cada entorno la primera semana post-corte; MTTR ≤ 4 h.
- **De negocio:** volumen de cotizaciones y pedidos post-corte comparable al periodo previo (ajustado por ventana de corte).

---

## 8. Preguntas abiertas para negocio

### 8.0 Fundamentales — antes de cualquier decisión técnica

> Preguntas de alto nivel que suelen darse por hechas pero que definen el marco de toda la separación. Vale la pena responderlas explícitamente aunque parezcan obvias.

**Gobierno y patrocinio**

1. ¿**Quién patrocina** esta iniciativa desde la dirección? ¿Está confirmado por escrito?
2. ¿**Quién decide** cuando haya conflicto entre PROQUIFA y GOLOCAER (catálogo, prioridad, presupuesto)?
3. ¿Está aprobado por **consejo de administración / accionistas**?
4. ¿Hay **fecha objetivo** o rango de fechas para el corte? ¿La fija negocio, legal, o el auditor?
5. ¿Existe un **presupuesto asignado** para el pre-proyecto?
6. ¿Qué **KPIs de éxito** se definieron? ¿Cómo se sabrá que la separación fue exitosa?
7. ¿La decisión es **reversible** si algo sale mal, o es un cambio permanente?
8. ¿Hay un **plan B** si USP no acepta la exclusividad con PROQUIFA?

**Legal y regulatorio**

9. ¿**GOLOCAER ya existe** legalmente como razón social operativa en México, con RFC, régimen fiscal y actas al día?
10. ¿PROQUIFA y GOLOCAER son la **misma persona moral** o entidades jurídicamente independientes?
11. ¿Ya está firmado el **contrato de exclusividad** entre PROQUIFA y USP?
12. ¿**USP está enterado** y de acuerdo con este esquema?
13. ¿Se consultó con el **auditor** cuál es el nivel mínimo de separación que exige? ¿Aceptaría segregación lógica, o exige separación física?
14. ¿Se consultó con **abogados fiscales / SAT** sobre implicaciones de reestructura?
15. ¿Hay **operaciones entre partes relacionadas** que deban documentarse por precios de transferencia?
16. ¿Se requieren **permisos regulatorios nuevos** (Cofepris, aduanas, comercio exterior) para PROQUIFA operando solo con USP?
17. ¿Hay **marcas registradas / propiedad intelectual** que se transfieran entre empresas?
18. ¿Cambia la **denominación social** de alguna de las empresas?
19. ¿Qué pasa con **litigios abiertos** — quedan en la razón social original o se transfieren?

**Contable y fiscal**

20. ¿PROQUIFA y GOLOCAER tienen el **mismo ejercicio fiscal** (enero-diciembre)?
21. ¿Comparten **políticas contables** (métodos de valuación, depreciación, provisiones)?
22. ¿Comparten **plan de cuentas** o cada una tiene el suyo?
23. ¿Consolidan financieramente para reportes de grupo, o cada una reporta por separado?
24. ¿Habrá **factoring / financiamiento** cruzado entre ellas?

**Organizacional — personas y espacios**

25. ¿PROQUIFA y GOLOCAER **comparten oficinas físicas**, o se separan también los espacios?
26. ¿Comparten **almacenes / warehouses**?
27. ¿Un mismo **empleado** puede trabajar para las dos empresas, o cada uno queda asignado a una?
28. ¿Habrá **movimientos de personal** (transferencias, recontratos, liquidaciones) para acomodar la nueva estructura?
29. ¿Se comparten **teléfonos, correos corporativos, dominios**?
30. ¿La **estructura organizacional** (director general, gerentes, comercial) es la misma o distinta?
31. ¿Se comparten proveedores de **servicios (nómina, contabilidad externa, TI)**?

**Comunicación**

32. ¿Se les **comunicará a los clientes** que la razón social con la que operan cambia? ¿Cuándo, cómo, con qué mensaje?
33. ¿Se les **comunicará a los proveedores** del cambio?
34. ¿Se les **comunicará a los empleados** internamente? ¿En qué momento?
35. ¿Hay un **plan de comunicación** aprobado, con etapas y responsables?
36. ¿Se contempla el **impacto en imagen de marca** durante la transición?

**Alcance temporal**

37. ¿La separación aplica **hacia adelante** (nuevas operaciones), o también **hacia atrás** (re-emisión de facturas históricas, migración de cartera)?
38. ¿Existe **fecha máxima legal** después de la cual no se puede operar bajo el modelo antiguo?
39. ¿Hay **ciclos de negocio críticos** (cierre trimestral, campañas comerciales, licitaciones abiertas) que condicionen la ventana de corte?

**Existencia de precedentes**

40. ¿Hay **casos previos** en el grupo de separación de razones sociales que sirvan de referencia?
41. ¿Existen **auditorías anteriores** cuyo hallazgo haya motivado esta separación?
42. ¿La separación es **preventiva** ante una eventual auditoría, o hay una **auditoría abierta** que la exige?

**Alcance geográfico**

43. La separación aplica sólo en **México**, o también en **Perú (Golocaer SAC)**?
44. Si es sólo México, ¿la operación Perú se ve afectada de algún modo?
45. ¿Se contemplan **otros países** a futuro que también deban aplicar la misma partición?

**Alcance sistémico**

46. ¿La separación aplica **sólo a ProquifaDotNet**, o también a **DocumentBuilder, sistemas legacy, ERP externo, contabilidad, nómina**?
47. ¿Hay **sistemas satélites** que sean invisibles hoy y que también deban separarse (facturación en Excel, CRM, mailing masivo)?
48. ¿Hay **integraciones con terceros** (bancos, PAC, plataformas B2B) que exijan un ID / RFC único por conexión?

### 8.1 Alcance comercial y funcional

1. ¿USP comercializa solamente productos "normales" u otras categorías (controlados, publicaciones, servicios, Labware, dispositivos médicos, capacitaciones)?
2. ¿La empresa PROQUIFA sigue existiendo como razón social para otros fines, o queda dedicada 100 % a USP?
3. ¿La operación Perú (Golocaer SAC) se separa también, o se mantiene bajo GOLOCAER en la instancia actual?
4. ¿Se requiere un puente de **cotización cross-company** (folio maestro que agrupe dos cotizaciones, una por empresa) cuando el cliente pida productos USP y no-USP en una sola orden?
5. ¿Cómo se manejarán las **cotizaciones y pedidos históricos** (previos al corte) que involucren la razón social anterior? ¿Se congelan, se migran, se renumeran?
6. ¿Existen **contratos vigentes** que incluyan productos USP y no-USP en una misma orden que deban migrarse o renegociarse?

### 8.2 Modelo de separación infraestructura vs lógica

1. ¿La separación es por **infraestructura** (dos entornos físicos: BD, VMs, servicios, redes), por **lógica** (una sola instancia con `IdEmpresa` como discriminador y RLS/tenancy en BD), o **híbrida** (BD compartida, aplicativos separados; o BD separada, aplicativos compartidos)?
2. ¿Los servicios transversales (IdentityServer, MinIO, RabbitMQ, Brevo) se **comparten** con particionado por tenant, o se **duplican** para no cruzar datos entre PROQUIFA y GOLOCAER?
3. ¿La auditoría exige **inmutabilidad física** entre entidades (no hay usuario con acceso cruzado, no hay conexión de red), o basta con **segregación lógica y bitácora**?
4. ¿Se necesita una **base consolidada** para reportería del grupo, o cada entidad reporta por su cuenta?
5. ¿Habrá **un único URL** con selector de empresa al login, o **dos URLs distintos** (uno por empresa)?
6. ¿El **backup / DRP** debe ser por empresa (restaurar sólo PROQUIFA sin tocar GOLOCAER) o compartido?

### 8.3 Datos maestros — Proveedores

1. ¿El proveedor **USP** queda exclusivamente en PROQUIFA, o algún otro proveedor también queda restringido a una sola empresa?
2. ¿Hay proveedores que hoy vendan a **más de una razón social** dentro del grupo? Si sí, ¿cómo se distribuyen post-corte?
3. ¿La relación `ProveedorEmpresa` se depura al corte, o se mantiene el histórico y solo se marca `Activo = 0` para las combinaciones inválidas?
4. ¿Las **condiciones comerciales con proveedor** (precios, descuentos, tiempos de entrega) son las mismas para PROQUIFA y GOLOCAER, o hay negociaciones diferenciadas?
5. ¿Los **contratos con USP** transfieren directo a PROQUIFA, o requieren firmas nuevas?

### 8.4 Datos maestros — Marcas y familias

1. Además de USP, ¿hay marcas que se **compartan** entre PROQUIFA y GOLOCAER, o cada marca se asigna a **una única empresa**?
2. Para las **familias de marcas** (`MarcaFamilia`), ¿se separan igual que la marca, o pueden pertenecer a varias?
3. ¿Las **campañas de marca** (`CampanaMarca`) siguen a la marca hacia su empresa, o se congelan al corte?
4. Los **contratos cliente-marca** (`ContratoClienteMarca`, `ContratoClienteMarcaFamilia`) que hoy incluyan marcas de las dos empresas — ¿se dividen en dos contratos, o se renegocian?
5. ¿Los **catálogos de precios por marca-cliente** se replican por empresa, o cada empresa negocia sus propios precios de USP y no-USP?

### 8.5 Datos maestros — Clientes

1. ¿Un cliente puede comprarle a **ambas empresas**, o se define un mapping "cliente ↔ empresa" que restrinja?
2. Si compra a ambas, ¿el **RFC/RUC** y **DatosFiscales** del cliente se replican en ambas instancias/particiones, o se comparten?
3. La **cartera** (`ClienteCartera`) — ¿es única del grupo, o cada empresa tiene su cartera?
4. Las **direcciones de entrega** — ¿se comparten (dirección única del cliente), o se replican por empresa?
5. Los **contactos comerciales** del cliente — ¿son los mismos para ambas empresas?
6. ¿El **cliente ve** las dos empresas como proveedores distintos o como una relación única?

### 8.6 Aplicativos front — VentaDigital, PunchOut

1. **VentaDigital** — ¿se separa en dos portales (uno para USP-PROQUIFA, otro para el resto en GOLOCAER), o se mantiene un solo portal con selector de empresa?
2. Si es un solo portal, ¿el catálogo mostrado depende del cliente y de la empresa seleccionada?
3. El **login** de VentaDigital — ¿un solo login (SSO con IdentityServer del grupo), o login independiente por empresa?
4. ¿La **navegación** (banner, colores, logotipos) cambia según la empresa mostrada?
5. **PunchOut** — el catálogo publicado a plataformas cliente (Ariba, Coupa, SAP SRM), ¿es uno solo o uno por empresa? La normativa PunchOut suele exigir un `punchout_url` distinto por proveedor.
6. Los **cXML/OCI** de PunchOut incluyen `Supplier.Name`, `Supplier.CorporateURL`, `PayloadID` — ¿cambian por empresa? Debe alinearse con el cliente PunchOut.
7. Los **pedidos entrantes** de PunchOut deben rutearse a PROQUIFA o GOLOCAER según catálogo — ¿cómo se determina el tenant destino?
8. ¿Hay aplicativos móviles (para vendedores, para clientes) que también deban separarse?

### 8.7 Productos controlados — combinación PROQUIFA vs GOLOCAER

1. ¿La línea USP incluye **productos controlados** (Mundial / Nacional / Origen)? Si sí, PROQUIFA no sólo maneja Normales.
2. Si un cliente pide **USP + un controlado no-USP**, ¿se **prohíbe** en una sola cotización, se **divide en dos cotizaciones** (una por empresa), o se **permite una cotización maestra con dos partidas facturadas por empresas distintas**?
3. Los **permisos regulatorios** (Cofepris, licencias sanitarias) — ¿son de PROQUIFA, de GOLOCAER, o de ambas? ¿Se transfieren o se piden nuevos para PROQUIFA?
4. Las **series y folios de facturas de controlados** (que hoy siguen normativa específica) — ¿PROQUIFA emite su propia serie o hereda la de la razón social anterior?
5. ¿Se requiere separar los **avisos de importación / padrón de importadores** por empresa? (`Empresa.Importador`, `Empresa.IdArchivoPadronImportador` ya modelan esto por empresa).

### 8.8 Documentos y PDF — DocumentBuilder / Logic.PDF

1. Cada empresa requiere **plantillas PDF propias** (cotización, confirmación de pedido, factura, nota de crédito, carta de disponibilidad, proforma, packing list) — ¿los diseños son idénticos con datos distintos, o el diseño gráfico también cambia?
2. ¿Los **logotipos, fuentes, colores** son distintos por empresa? ¿Hay guía de identidad para PROQUIFA vs GOLOCAER?
3. ¿La firma / sellos digitales de los PDF llevan datos de qué empresa? (relevante para PDFs jurídicos).
4. Los **avisos legales / avisos de privacidad** al pie del PDF — ¿son distintos por empresa? (comercio exterior, LFPDPPP, regulación sanitaria).
5. ¿Los PDFs generados históricamente se **re-emiten** por la nueva empresa cuando aplique (refacturaciones), o quedan intocables?

### 8.9 Nivel de separación visual y de UI

1. **Nivel visual en el aplicativo interno** (Venta Interna) — ¿el usuario ve una sola UI con un badge de empresa activa, o dos UIs completamente distintas?
2. Los **catálogos filtrados** por empresa (productos, marcas, clientes) — ¿se cargan automáticamente al elegir empresa, o siempre se muestra todo con filtros manuales?
3. Los **reportes internos** — ¿el usuario los ve por empresa, o consolidados con desglose?
4. ¿Los **avisos y notificaciones internos** (nuevas cotizaciones, aprobaciones pendientes) se filtran por la empresa del usuario, o los ve todos?
5. **Permisos por usuario** — ¿un usuario puede operar en las dos empresas, o cada usuario está asignado a una?
6. ¿Se requieren **roles diferenciados** por empresa (ej. "Comprador USP" separado de "Comprador General")?

### 8.10 Nivel de separación en Base de Datos *(en el MVP: dos BD físicamente separadas; preguntas remanentes sobre catálogos compartidos)*

1. ¿La BD `ProquifaDotNet` se **duplica físicamente** (dos servidores / dos catálogos), o se mantiene una sola con **`IdEmpresa` como discriminador transversal** (multi-tenant)?
2. Si se duplica físicamente, ¿los **catálogos compartidos** (SAT, códigos postales, regímenes fiscales, monedas, países) se sincronizan por script, se replican por CDC, o cada BD tiene copia independiente?
3. Si es una sola BD, ¿se implementa **Row-Level Security (RLS)** por `IdEmpresa`, o el filtrado queda a nivel aplicación?
4. Los **stored procedures** actuales que asumen "todo pertenece a un mismo grupo" — ¿se ajustan para respetar el `IdEmpresa` del contexto?
5. ¿Hay tablas que se puedan **desnormalizar por empresa** (`cotCotizacion_PROQUIFA`, `cotCotizacion_GOLOCAER`) para reducir contención de índices?
6. La **auditoría (bitácora, `FechaRegistro`, `FechaUltimaActualizacion`)** — ¿se conserva quién es el usuario que operó por empresa?

### 8.11 Buzones — comunicaciones entrantes y salientes

1. ¿Cada empresa tiene sus **propios buzones de correo** (cotización, pedidos, pagos, quejas), o se comparten con reglas de ruteo?
2. Si son separados, ¿cómo se distinguen — dominios distintos (`@proquifa.com`, `@golocaer.com`), alias, subdominios?
3. El **buzón de cotización entrante** (parsing automático de correos) — ¿procesa emails para las dos empresas o uno por instancia?
4. ¿Los **PDFs adjuntos** que llegan por buzón se rutean a la instancia PROQUIFA o GOLOCAER según remitente / cliente / asunto? ¿Qué regla decide?
5. Los **buzones de pago** (aviso de transferencia) — ¿cada empresa recibe sus propios avisos y los procesa contra su cartera?
6. Los **buzones de quejas / soporte** — ¿unificados con enrutamiento por empresa, o separados?
7. ¿Las **respuestas automáticas** (acuse de recibo, envío de cotización) llevan la identidad de la empresa que respondió?
8. Los **buzones de portales cliente** (VentaDigital, PunchOut) que hoy notifican al equipo interno — ¿se dividen por tenant destino?

### 8.12 Seguridad, cumplimiento y accesos

1. **IdentityServer** — ¿se comparte con `tenant_id` en el claim del token, o se despliegan dos IdentityServer distintos?
2. ¿Los **certificados de timbrado (CSD)** de PROQUIFA y GOLOCAER se guardan en la misma bóveda, o en bóvedas separadas por empresa?
3. Los **logs (Serilog)** — ¿se enrichecen con `IdEmpresa` y se filtran por empresa, o se agregan y se filtran a posteriori?
4. Los **respaldos** — ¿por empresa (para responder solicitudes de auditoría de sólo una), o unificados?
5. **Rotación de secretos / credenciales** — ¿cada empresa tiene su ciclo de rotación, o se rotan juntas?
6. **GDPR / LFPDPPP** — el derecho al olvido y el acceso a datos personales, ¿opera por empresa o por grupo?

### 8.13 Operación, soporte y desarrollo

1. ¿El **equipo de soporte** atenderá los dos entornos como uno solo, o hay dos guardias distintas?
2. El **pipeline CI/CD** — ¿un solo pipeline promoviendo a los dos entornos, o pipelines independientes?
3. ¿Se conserva **un solo repositorio de código** con branch/tenant flag, o se hace **fork** por empresa?
4. Las **migraciones de esquema BD** — ¿se ejecutan sincronizadamente en ambos entornos, o cada uno lleva su propio calendario?
5. El **monitoreo (APM, dashboards)** — ¿unificado con dimensión "empresa", o separado?
6. Las **pruebas E2E** — ¿un solo suite parametrizable por empresa, o dos suites?
7. ¿Los **entornos de QA / staging** también se separan por empresa, o basta con QA único con selector?

### 8.14 Ventana de corte y coexistencia

1. ¿Hay una fecha objetivo para el corte? ¿Se planifica **big-bang** o **transición progresiva** (por línea de producto, por cliente, por región)?
2. ¿Se opera un **período de coexistencia** en que ambos modelos convivan (nuevo modelo separado + modelo antiguo unificado), o el corte es duro?
3. Durante la ventana, ¿se **congelan pedidos** o se sigue operando?
4. Las **cotizaciones vigentes con vencimiento posterior al corte** — ¿mantienen la razón social original hasta expirar, o se re-emiten?
5. ¿Se requieren **facturas globales de cierre** para la razón social anterior?

### 8.15 Integraciones externas (SAT, PAC, bancos, aduanas)

1. Los **contratos con PAC (timbrado)** — ¿PROQUIFA y GOLOCAER usan el mismo PAC o distintos? ¿Se firman contratos nuevos?
2. Las **cuentas bancarias** por empresa (`EmpresaDatosBancarios`) — ¿ya existen las cuentas de PROQUIFA-USP-only y GOLOCAER-resto, o hay que aperturarlas?
3. Los **conectores bancarios (STP, referencias, cobranza)** — ¿un solo canal por empresa o compartido?
4. Las **cuentas de comercio exterior** (pedimentos, agentes aduanales) — ¿se separan por empresa? USP presumiblemente sólo por PROQUIFA.
5. Las **cuentas contables** (`CuentaEmpresa`) — ¿se replican por empresa como hoy o se consolidan?

---

## 9. Próximos pasos sugeridos

1. Sesión con negocio para responder §8.0 (Fundamentales) y validar el MVP definido en §7.
2. Con las respuestas, elaborar diseño técnico detallado (BD, servicios, ETLs, timbrado) siguiendo la ruta F0–F7 de §7.8.
3. Registrar el requisito `TPSC-RE-FU-SEP` en la Guía de Estimación (WBS Backend) con las fases del MVP.
4. Definir plan de corte y ventana de migración (fase F6).
5. Preparar plan de pruebas E2E que valide la matriz colapsada en PROQUIFA y la matriz vigente en GOLOCAER, parametrizable por empresa.
6. Provisionar entornos de infraestructura para las dos empresas (fase F3).

---

*Documento base — versión 0.1. Se actualizará conforme se confirmen los puntos abiertos con negocio.*
