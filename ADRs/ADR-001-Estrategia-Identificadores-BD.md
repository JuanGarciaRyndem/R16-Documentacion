# ADR-001 — Estrategia de identificadores en Base de Datos y Aplicativo

- **Estado:** Aprobado
- **Fecha:** 2026-08-11
- **Última actualización:** 2026-08-12
- **Autor:** Juan David
- **Revisores:** Valde (arquitectura) — aprobado. Gerardo (DBA) — aprobado con matices para catálogos.
- **Alcance:** **Aplicativos nuevos con base de datos nueva**. Este documento **NO aplica** a `ProquifaDotNet` (PQF2) ni a `ProquifaDotNet.Finanzas`, ya que ambos operan sobre la base de datos existente `ProquifaDotNet` y se rigen por sus propias convenciones. Aplica a soluciones que se creen con **su propia base de datos nueva** (por ejemplo, `ProquifaDotNetTimbrado` y futuras soluciones equivalentes).

---

## 1. Contexto

Al iniciar un aplicativo nuevo con su propia base de datos nueva es necesario definir de manera única y explícita la estrategia de generación de identificadores primarios. En proyectos anteriores hemos visto tres prácticas mezcladas — `int IDENTITY`, `uniqueidentifier` con `NEWID()` y `uniqueidentifier` generado desde el aplicativo —, lo cual produce:

- Fragmentación alta en índices clustered por uso de `NEWID()`.
- Necesidad de round-trips a BD para conocer el ID antes de publicar mensajes a colas o entre servicios.
- Inconsistencia de tipos entre tablas de una misma base, dificultando joins, referencias y trazabilidad.

Los aplicativos nuevos alcanzados por este ADR se construyen en **.NET Core 10**, usan **SQL Server** como motor de base de datos, publican eventos a **RabbitMQ** y se integran con otros servicios mediante APIs, por lo que requieren que los IDs de agregados estén disponibles **antes** de tocar la BD.

---

## 2. Decisión

Se adopta el siguiente estándar para aplicativos nuevos con base de datos nueva, diferenciando **tablas transaccionales/dominio** y **tablas de catálogo**.

### 2.1 Tablas transaccionales y de dominio

1. **La PK será `uniqueidentifier`**, sin excepción por tamaño ni tipo de entidad.
2. **El ID lo genera el aplicativo con `Guid.CreateVersion7()`** (nativo en .NET 9+, disponible en .NET Core 10).
3. **La columna tendrá `DEFAULT NEWSEQUENTIALID()`** como red de seguridad, para que cualquier `INSERT` que no envíe ID desde el aplicativo mantenga orden monótono y minimice fragmentación.
4. **La PK se define como CLUSTERED** salvo excepción documentada (tablas de altísima escritura por rango de fecha, donde el clustered index puede convenir sobre `FechaCreacion`).
5. Se centraliza la generación en un helper único por solución: `IIdGenerator.NewId()` en la capa `Domain` (o `Application`), implementado en `Infrastructure` con `Guid.CreateVersion7()`. Ninguna capa llama directamente a `Guid.CreateVersion7()` — así podemos sustituir la implementación (por ejemplo, para pruebas deterministas o si aparece UUIDv8).

### 2.2 Tablas de catálogo (regla del DBA)

Los catálogos **no** los administra el sistema en tiempo de ejecución: se cargan y actualizan por script y deben quedar **homologados entre ambientes** (Dev, QA, Prod, etc.). Por eso siguen reglas distintas:

1. **La PK será `uniqueidentifier`** (se mantiene la uniformidad de tipo con el resto de la BD).
2. **NO se usa `NEWSEQUENTIALID()` como DEFAULT** ni se genera el ID en runtime — el GUID lo **hardcodea el programador dentro del script de insert**. Así el mismo registro tiene **el mismo ID en todos los ambientes**, lo cual es indispensable para catálogos referenciados desde código, integraciones y migraciones.
3. **Todo catálogo debe llevar un `UNIQUE` (UK) sobre la columna `Clave`** (o el campo de negocio equivalente), para impedir que se inserte dos veces el mismo valor lógico.
4. **La PK sigue siendo CLUSTERED** salvo excepción documentada.
5. El GUID hardcodeado se genera **una sola vez** al crear el registro por primera vez en el script; queda registrado en el repositorio de scripts como parte del `INSERT` y no se regenera.

**Fuera de alcance:** las bases de datos existentes (`ProquifaDotNet`, entre otras) y los aplicativos que operan sobre ellas (`ProquifaDotNet` — PQF2 — y `ProquifaDotNet.Finanzas`) conservan sus propias convenciones y no se ven afectados por este ADR.

---

## 3. Alternativas consideradas

### 3.1 `uniqueidentifier` con `DEFAULT NEWID()`

**Ventajas:** global, sin coordinación, fácil de adoptar.
**Desventajas:** GUID totalmente aleatorio → **page splits constantes** en el clustered index → fragmentación alta, más I/O y peor rendimiento en `INSERT` masivos.
**Descartado.**

### 3.2 `uniqueidentifier` con `DEFAULT NEWSEQUENTIALID()` únicamente

**Ventajas:** GUIDs monótonos por instancia de SQL Server → baja fragmentación.
**Desventajas:** solo funciona como valor `DEFAULT` de columna — no se puede llamar en `SELECT`, ni desde .NET, ni publicar el ID en un mensaje antes de insertar. Reinicia su secuencia al reiniciar el servicio (no rompe unicidad, pero sí monotonía absoluta). No es único entre servicios distintos.
**Descartado como estrategia principal; conservado como red de seguridad.**

### 3.3 `Guid.NewGuid()` (UUIDv4) desde aplicativo

**Ventajas:** genera desde código, único globalmente.
**Desventajas:** totalmente aleatorio → misma fragmentación que `NEWID()`. Sin ventaja frente al DEFAULT de SQL.
**Descartado.**

### 3.4 **UUIDv7 desde aplicativo (`Guid.CreateVersion7()`) + `NEWSEQUENTIALID()` como default** — Decisión adoptada

**Ventajas:**
- 48 bits iniciales de timestamp Unix (ms) → GUIDs monótonos y ordenables por tiempo.
- Baja fragmentación en clustered index.
- Generado en cliente → agregados y eventos pueden construirse y publicarse a Rabbit sin tocar BD.
- Único globalmente entre bases y servicios sin coordinación.
- Estándar RFC 9562, soporte nativo en .NET 9+.
- `NEWSEQUENTIALID()` como `DEFAULT` protege contra inserciones que no envíen ID (scripts DDL/DML, mantenimientos, herramientas externas).

**Desventajas:**
- 16 bytes por columna — impacto en índices no clustered de tablas de altísimo volumen. Se mitigará evaluando índices no clustered caso por caso.
- El orden de bytes de GUID en SQL Server no coincide con el de .NET; UUIDv7 mantiene el orden porque los bytes de tiempo quedan al inicio en ambas representaciones, pero se validará con benchmark de fragmentación antes de estandarizar.
- Predecible en tiempo (expone timestamp de creación). Aceptable para IDs internos; para tokens u OTP se seguirá usando aleatoriedad criptográfica.

### 3.5 Tabla comparativa

| Criterio | `NEWID()` | `NEWSEQUENTIALID()` | UUIDv7 (aplicativo) | UUIDv7 + `NEWSEQUENTIALID()` default |
|---|---|---|---|---|
| Dónde se genera | SQL Server | SQL Server (solo DEFAULT) | Aplicativo (.NET) | Aplicativo, con red de seguridad en BD |
| Tamaño | 16 bytes | 16 bytes | 16 bytes | 16 bytes |
| Ordenado / monótono | No (aleatorio) | Sí (por instancia) | Sí (por timestamp ms) | Sí (siempre) |
| Fragmentación en clustered index | Alta | Baja | Baja | Baja |
| ID disponible antes del INSERT | No | No | Sí | Sí |
| Único entre servicios/BDs | Sí | Sí (con muy baja colisión) | Sí | Sí |
| Publicable a Rabbit sin round-trip | No | No | Sí | Sí |
| Se puede usar en `SELECT` / código | No aplica | No | Sí | Sí |
| Predecible (expone info) | No | Sí (orden) | Sí (timestamp) | Sí (timestamp) |
| Protege inserts fuera del aplicativo | — | Sí | No | Sí |
| Estándar | Propietario SQL | Propietario SQL | RFC 9562 | RFC 9562 + SQL |
| Recomendado para aplicativos nuevos | ❌ | Solo como default | ✅ | ✅ **Elegido (transaccionales)** |

**Nota sobre catálogos:** ninguna de las opciones anteriores aplica. Los catálogos usan **GUID hardcodeado en el script de carga** (sin DEFAULT, sin generación en runtime) más un UK sobre `Clave`, para garantizar el mismo `Id` en todos los ambientes.

---

## 4. Consecuencias

### 4.1 Positivas
- Estándar único y explícito para todo aplicativo nuevo con base de datos nueva.
- IDs disponibles antes de la persistencia → habilita mensajería asíncrona limpia con RabbitMQ y flujos por eventos.
- Menor fragmentación en índices clustered → mejor rendimiento de escritura y mantenimiento.
- Un único helper `IIdGenerator` facilita pruebas y evolución (v8, ULID, etc.).

### 4.2 Negativas / Trade-offs
- Mayor consumo de espacio en índices no clustered en tablas de muy alto volumen — se revisará individualmente.
- Requiere disciplina: nunca llamar `Guid.NewGuid()` para IDs de dominio; siempre `IIdGenerator.NewId()`.
- Convive con bases existentes (`ProquifaDotNet` y similares) que usan otro esquema — la frontera se cruza mediante APIs y ETLs, respetando los tipos existentes de cada lado.

### 4.3 Impactos operativos
- **Aplicativos alcanzados:** solo los que se crean con **base de datos nueva propia** (ej. `ProquifaDotNetTimbrado`).
- **Aplicativos no alcanzados:** `ProquifaDotNet` (PQF2) y `ProquifaDotNet.Finanzas` — ambos siguen las convenciones de la base `ProquifaDotNet` existente.
- **Diccionario de datos:** cada tabla del aplicativo nuevo debe declarar explícitamente:
  - Si es transaccional/dominio: PK `uniqueidentifier`, generada por aplicativo con UUIDv7, con `DEFAULT NEWSEQUENTIALID()`.
  - Si es catálogo: PK `uniqueidentifier` **hardcodeada en el script de carga**, sin DEFAULT, y con UK sobre `Clave`.
- **Scripts de catálogo:** deben versionarse en el repositorio de scripts con los GUIDs ya definidos; los mismos scripts se ejecutan en Dev, QA y Prod garantizando que un `Catalogo.Clave = 'X'` tenga el mismo `Id` en todos los ambientes.

---

## 4.4 Historial de revisión

| Fecha | Revisor | Rol | Resultado |
|---|---|---|---|
| 2026-08-12 | Valde | Arquitectura | Aprobado — luz verde a la propuesta adoptada. |
| 2026-08-12 | Gerardo | DBA | Aprobado con matices — para catálogos: GUID hardcodeado desde el script (no NEWSEQUENTIALID ni runtime), homologación de Ids entre ambientes y UK sobre `Clave`. Integrado en sección 2.2. |

---

## 5. Reglas de implementación

1. Cada solución define su propia interfaz en `Domain`:
   ```csharp
   public interface IIdGenerator
   {
       Guid NewId();
   }
   ```
2. Implementación única en `Infrastructure`:
   ```csharp
   internal sealed class UuidV7Generator : IIdGenerator
   {
       public Guid NewId() => Guid.CreateVersion7();
   }
   ```
3. Registro en el contenedor DI como `Singleton`.
4. Definición SQL estándar de la columna de PK **en tablas transaccionales/dominio**:
   ```sql
   [Id] UNIQUEIDENTIFIER NOT NULL
       CONSTRAINT DF_<Tabla>_Id DEFAULT (NEWSEQUENTIALID()),
       CONSTRAINT PK_<Tabla> PRIMARY KEY CLUSTERED ([Id])
   ```
5. Definición SQL estándar **en tablas de catálogo** (sin DEFAULT, con UK de negocio):
   ```sql
   CREATE TABLE [dbo].[<Catalogo>](
       [Id]           UNIQUEIDENTIFIER NOT NULL,
       [Clave]        VARCHAR(50)      NOT NULL,
       [Descripcion]  VARCHAR(200)     NOT NULL,
       -- resto de columnas del catálogo
       CONSTRAINT PK_<Catalogo>  PRIMARY KEY CLUSTERED ([Id]),
       CONSTRAINT UK_<Catalogo>_Clave UNIQUE ([Clave])
   );

   -- Los inserts hardcodean el GUID para homologar entre ambientes:
   INSERT INTO [dbo].[<Catalogo>] ([Id], [Clave], [Descripcion]) VALUES
       ('0198f2a3-1c5e-7c4a-9d2b-3f4e5a6b7c8d', 'CLAVE_A', 'Descripción A'),
       ('0198f2a3-1c5e-7c4a-9d2b-3f4e5a6b7c8e', 'CLAVE_B', 'Descripción B');
   ```
6. Prohibido `Guid.NewGuid()` para IDs de dominio (regla de revisión de código).
7. Prohibido dejar que un catálogo genere su GUID en runtime — el GUID viaja en el script para garantizar el mismo Id entre ambientes.

---

## 6. Validación pendiente

Antes de generalizar la decisión se realizará una prueba de rendimiento en una base de datos representativa:

- Tabla de prueba con 10M de filas insertadas con `NEWID()`, `NEWSEQUENTIALID()` y `Guid.CreateVersion7()`.
- Medir fragmentación (`sys.dm_db_index_physical_stats`), tiempo de `INSERT` por lote y tiempo de consulta por rango de fecha.
- Resultado esperado: UUIDv7 con fragmentación cercana a `NEWSEQUENTIALID()` (< 10%) y muy inferior a `NEWID()`.

---

## 7. Referencias

- RFC 9562 — Universally Unique IDentifiers (UUIDs), sección UUIDv7.
- .NET 9 — `Guid.CreateVersion7()` (`System.Guid`).
- SQL Server — `NEWSEQUENTIALID()`, ordenamiento de `uniqueidentifier`.
