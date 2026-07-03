# R16A-RE-FU-003 — Propuesta 1: Modelo genérico ArchivoDominio + Endpoints reutilizables

| Campo | Valor |
|---|---|
| **Requisito base** | R16A-RE-FU-003 — Mantenimiento de Catálogo de Clientes — Documentos Regulatorios |
| **Tipo** | Propuesta de rediseño técnico |
| **Versión** | 1.0 |
| **Fecha** | 2026-06-26 |
| **Aplicativo** | ProquifaDotNet |
| **Base de datos** | ProquifaDotNet |

---

## 1. Propósito de la Propuesta

Rediseñar la capa de datos y los endpoints de RE-FU-003 para que la gestión de documentos:

1. **Sea reutilizable** para Cliente, Proveedor, Producto y futuras entidades, sin crear tablas específicas por dominio (hoy: `ArchivoCliente`; mañana: `ArchivoProveedor`, `ArchivoProducto`…).
2. **Centralice la metadata del catálogo** (mimeTypes aceptados, orden de visualización, tamaño máximo KB) en BD en vez de hardcodear en código.
3. **Separe responsabilidades**: un endpoint subir-archivo genérico (independiente del dominio) y otro vincular-archivo-a-dominio.
4. **Habilite un endpoint Validar extensible** — hoy valida documentos regulatorios del cliente; mañana puede incorporar otras reglas de bloqueo de tramitación.

> **Alcance regional — OBS-007:** sigue aplicando únicamente a Región México. Sustancias Controladas no habilitadas para Región Perú en R16.

---

## 2. Impacto en Base de Datos

### 2.1 Modelo propuesto

```
catDominioEntidad                    (Cliente / Proveedor / Producto / …)
    └── 1:N
CatalogoTipoDocumento                (Licencia Sanitaria, Aviso Responsable, Ficha Técnica, …)
    └── 1:N
ArchivoDominio                       (vínculo archivo ↔ entidad de dominio)
    └── FK IdArchivo
Archivo                              (FileKey, FileBucket → MinIO — existente, sin cambios)
```

**Sustituciones respecto al modelo actual:**

| Modelo actual (R16A-RE-FU-003) | Modelo propuesto (Propuesta 1) |
|---|---|
| `ArchivoCliente` (específico Cliente) | `ArchivoDominio` (genérico — Cliente / Proveedor / Producto / …) |
| `catUsoArchivoSistema` (sin scope de dominio, sin metadata UI) | `CatalogoTipoDocumento` (scoped por `IdcatDominioEntidad`, con FormatoAceptado / Orden / TamanioMaximoKB) |
| (no existe) | `catDominioEntidad` (catálogo de tipos de entidad de dominio) |

---

### 2.2 Diccionario de Datos — Tablas nuevas

#### 2.2.1 `catDominioEntidad` (catálogo nuevo)

**Propósito:** Define los tipos de entidad de dominio que pueden tener archivos asociados (Cliente, Proveedor, Producto, …).

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `IdcatDominioEntidad` | `uniqueidentifier` | NO | PK. Default `NEWID()` |
| `Clave` | `varchar(50)` | NO | Identificador lógico (`Cliente`, `Proveedor`, `Producto`) |
| `Descripcion` | `varchar(200)` | NO | Descripción legible para administración |
| `Activo` | `bit` | NO | Default `1` |
| `FechaRegistro` | `datetime` | NO | Default `GETDATE()` |
| `FechaUltimaActualizacion` | `datetime` | NO | Default `GETDATE()` |

**Índices:**
- `PK_catDominioEntidad` (Clustered): `IdcatDominioEntidad`
- `UQ_catDominioEntidad_Clave` (Unique): `Clave`

**Datos iniciales (R16):**

| IdcatDominioEntidad | Clave | Descripcion |
|---|---|---|
| `NEWID()` | `Cliente` | Maneja los archivos del cliente |
| `NEWID()` | `Proveedor` | Maneja los archivos del proveedor |
| `NEWID()` | `Producto` | Maneja los archivos del producto |

---

#### 2.2.2 `CatalogoTipoDocumento` (sustituye `catUsoArchivoSistema` para este dominio)

**Propósito:** Catálogo de tipos de documento permitidos, scopeados por dominio de entidad, con metadata para UI (mimeTypes, orden, tamaño máximo).

| Columna                    | Tipo               | Nulo | Descripción                                                                                                                                                   |
| -------------------------- | ------------------ | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `IdCatalogoTipoDocumento`  | `uniqueidentifier` | NO   | PK. Default `NEWID()`                                                                                                                                         |
| `Clave`                    | `varchar(80)`      | NO   | Identificador lógico (`LicenciaSanitaria`, `AvisoResponsableSanitario`, `FichaTecnica`, …)                                                                    |
| `Nombre`                   | `varchar(200)`     | NO   | Nombre para mostrar en UI                                                                                                                                     |
| `Descripcion`              | `varchar(500)`     | SÍ   | Descripción extendida (tooltips / ayuda)                                                                                                                      |
| `IdcatDominioEntidad`      | `uniqueidentifier` | NO   | FK → `catDominioEntidad`. Define a qué dominio aplica este tipo                                                                                               |
| `FormatoAceptado`          | `varchar(500)`     | NO   | Lista de mimeTypes separados por coma (p. ej. `application/pdf,image/jpeg,image/png`). El Frontend usa este campo para configurar el input file y validación. |
| `Orden`                    | `int`              | NO   | Orden de despliegue en pantalla (UI lista por dominio)                                                                                                        |
| `TamanioMaximoKB`          | `int`              | NO   | Tamaño máximo aceptado en KB. `0` = sin límite                                                                                                                |
| `Activo`                   | `bit`              | NO   | Default `1`                                                                                                                                                   |
| `FechaRegistro`            | `datetime`         | NO   | Default `GETDATE()`                                                                                                                                           |
| `FechaUltimaActualizacion` | `datetime`         | NO   | Default `GETDATE()`                                                                                                                                           |

**Índices:**
- `PK_CatalogoTipoDocumento` (Clustered): `IdCatalogoTipoDocumento`
- `UQ_CatalogoTipoDocumento_DominioClave` (Unique filtrado): `(IdcatDominioEntidad, Clave) WHERE Activo = 1`
- `IX_CatalogoTipoDocumento_Dominio` (NonClustered): `(IdcatDominioEntidad, Orden) WHERE Activo = 1`

**Relaciones:**
- FK `IdcatDominioEntidad` → `catDominioEntidad.IdcatDominioEntidad`

**Datos iniciales (R16 — solo México, OBS-007):**

| Clave | Nombre | Dominio | FormatoAceptado | Orden | TamanioMaximoKB |
|---|---|---|---|---|---|
| `LicenciaSanitaria` | Licencia Sanitaria | Cliente | `application/pdf` | 1 | 10240 |
| `AvisoResponsableSanitario` | Aviso de Responsable Sanitario | Cliente | `application/pdf` | 2 | 10240 |

> Pendiente confirmar tamaño máximo (10 MB propuesto) con Regulatorios.

---

#### 2.2.3 `ArchivoDominio` (sustituye `ArchivoCliente`)

**Propósito:** Vincula un `Archivo` (físico en MinIO) con una **entidad de dominio** (Cliente, Proveedor, Producto, …) y su **tipo de documento**.

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `IdArchivoDominio` | `uniqueidentifier` | NO | PK. Default `NEWID()` |
| `IdArchivo` | `uniqueidentifier` | NO | FK → `Archivo` |
| `IdCatalogoTipoDocumento` | `uniqueidentifier` | NO | FK → `CatalogoTipoDocumento` (define tipo + dominio implícito) |
| `IdDominioEntidadArchivo` | `uniqueidentifier` | NO | GUID de la entidad del dominio (IdCliente / IdProveedor / IdProducto según corresponda). No es FK rígida — se valida en aplicación por la `Clave` de `catDominioEntidad`. |
| `Activo` | `bit` | NO | `1` = vigente. `0` = reemplazado / eliminado. Default `1` |
| `FechaRegistro` | `datetime` | NO | Default `GETDATE()` |
| `FechaUltimaActualizacion` | `datetime` | NO | Default `GETDATE()` |

**Índices:**
- `PK_ArchivoDominio` (Clustered): `IdArchivoDominio`
- `UQ_ArchivoDominio_Vigente` (Unique filtrado): `(IdDominioEntidadArchivo, IdCatalogoTipoDocumento) WHERE Activo = 1` — garantiza una sola versión vigente por tipo por entidad (Regla R3)
- `IX_ArchivoDominio_Entidad` (NonClustered): `(IdDominioEntidadArchivo, Activo)`
- `IX_ArchivoDominio_Tipo` (NonClustered): `(IdCatalogoTipoDocumento, Activo)`

**Relaciones:**
- FK `IdArchivo` → `Archivo.IdArchivo`
- FK `IdCatalogoTipoDocumento` → `CatalogoTipoDocumento.IdCatalogoTipoDocumento`
- `IdDominioEntidadArchivo`: **referencia lógica polimorfa** (no FK física). El BO valida que el GUID corresponda al dominio del `CatalogoTipoDocumento.IdcatDominioEntidad` (p. ej. si tipo aplica al dominio `Cliente`, el GUID debe existir en `Cliente`).

**Consideraciones especiales:**
- No se aplica FK física sobre `IdDominioEntidadArchivo` porque referenciaría tablas distintas según el dominio. La integridad se garantiza por validación en el BO (Tarea 1).
- Baja lógica obligatoria: nunca `DELETE`. Siempre `Activo = 0`.
- El archivo físico en MinIO no se purga automáticamente al hacer baja lógica (mismo comportamiento que el modelo actual).

---

### 2.3 Migración desde modelo actual

Si `ArchivoCliente` ya tiene datos en producción, migrar a `ArchivoDominio`:

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
-- Asume que catDominioEntidad y CatalogoTipoDocumento ya estan poblados
BEGIN TRANSACTION;

DECLARE @IdDominioCliente UNIQUEIDENTIFIER =
    (SELECT IdcatDominioEntidad FROM dbo.catDominioEntidad WHERE Clave = 'Cliente');

INSERT INTO dbo.ArchivoDominio
    (IdArchivoDominio, IdArchivo, IdCatalogoTipoDocumento, IdDominioEntidadArchivo, Activo, FechaRegistro, FechaUltimaActualizacion)
SELECT
    NEWID(),
    ac.IdArchivo,
    ctd.IdCatalogoTipoDocumento,
    ac.IdCliente,
    ac.Activo,
    GETDATE(),
    GETDATE()
FROM dbo.ArchivoCliente             ac
INNER JOIN dbo.catUsoArchivoSistema u   ON ac.IdCatUsoArchivoSistema = u.IdCatUsoArchivoSistema
INNER JOIN dbo.CatalogoTipoDocumento ctd ON ctd.Clave = u.Clave AND ctd.IdcatDominioEntidad = @IdDominioCliente;

-- Verificar conteos
SELECT COUNT(*) AS OrigenArchivoCliente FROM dbo.ArchivoCliente;
SELECT COUNT(*) AS DestinoArchivoDominio FROM dbo.ArchivoDominio WHERE IdDominioEntidadArchivo IN (SELECT IdCliente FROM dbo.Cliente);

-- COMMIT;
-- ROLLBACK;
COMMIT;
```

> `ArchivoCliente` y `catUsoArchivoSistema` permanecen vivas durante la transición; se retiran cuando los consumidores migren al nuevo modelo. No se eliminan en la misma release.

---

## 3. Impacto en Back

### 3.1 Resumen de endpoints

| #     | Método     | Ruta                                                           | Propósito                                                                                                      |
| ----- | ---------- | -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **1** | **`POST`** | **`/Archivo/Subir`**                                           | **Subir archivo genérico (file + bucket + region) → retorna `Archivo` Revisar esto con lo generado con Sebas** |
| 2     | `PUT`      | `/ArchivoDominio`                                              | Vincular `Archivo` ya subido a una entidad de dominio (con lógica de reemplazo `Activo=0`)                     |
| 3     | `POST`     | `/ArchivoDominio/QueryResult`                                  | Consultar documentos por entidad y/o tipo usando `QueryInfo` (filtros, paginación, orden, búsqueda)            |
| 4     | `GET`      | `/ArchivoDominio/{id}/Pdf`                                     | Descargar contenido binario del archivo (PDF u otro mimeType)                                                  |
| 5     | `GET`      | `/Cliente/Validar/{idCliente}`                                 | Validar si el cliente cumple por ahora con los documentos regulatorios requeridos                              |

---

### 3.2 Endpoint detallado — `POST /Archivo/Subir`

**Propósito:** Subir un archivo a MinIO y registrar el `Archivo` en BD. **No vincula a ningún dominio** — su responsabilidad termina al retornar el `Archivo` creado.

**Request (multipart/form-data):**
- `file` — archivo binario
- `bucket` — nombre del bucket de MinIO destino
- `region` — `IdRegion` (Guid) que se asigna al `Archivo`

**Response (200 OK):**
```json
{
  "idArchivo": "guid",
  "fileKey": "ruta/generada/del/archivo",
  "fileBucket": "documentos-regulatorios",
  "idRegion": "guid",
  "fechaRegistro": "2026-06-26T10:00:00",
  "activo": true
}
```

**Validaciones:**
- `file` no vacío.
- `bucket` debe existir en `RegionConfiguracionMinioBucket` para la `region` indicada (o ser bucket válido configurado).
- No valida mimeType aquí (lo hace el consumidor antes de invocar; mantiene endpoint genérico).

**BO:** `ArchivoBO.Subir(file, bucket, region)` — reutiliza la lógica existente de `SubirArchivoDesdePeticion` y la generaliza.

---

### 3.3 Endpoint detallado — `POST /ArchivoDominio`

**Propósito:** Vincular un `Archivo` ya subido a una entidad de dominio. Implementa la lógica de **reemplazo atómico** (baja lógica del registro anterior activo del mismo tipo para la misma entidad).

**Request (JSON):**
```json
{
  "idArchivo": "guid (resultado de POST /Archivo/Subir)",
  "idCatalogoTipoDocumento": "guid",
  "idDominioEntidadArchivo": "guid (IdCliente / IdProveedor / IdProducto)"
}
```

**Response (200 OK):** `ArchivoDominio` creado.

**Lógica del BO (`ArchivoDominioBO._GuardarOActualizar`):**
1. Validar que `CatalogoTipoDocumento.IdcatDominioEntidad` corresponda al dominio del `idDominioEntidadArchivo` (p. ej. si tipo aplica a `Cliente`, verificar que el GUID exista en `Cliente`).
2. Validar que el `FormatoAceptado` del `CatalogoTipoDocumento` incluya el mimeType del `Archivo` referenciado (consulta `Archivo` para inferir extensión o pedir mimeType en el payload).
3. Validar `TamanioMaximoKB` contra el tamaño del archivo.
4. Buscar registro activo con mismo `(IdDominioEntidadArchivo, IdCatalogoTipoDocumento)`.
5. Si existe → `UPDATE Activo = 0`.
6. `INSERT` nuevo `ArchivoDominio` con `Activo = 1`.

**Cubre las Reglas R2 (formato), R3 (versión única vigente) y R4 (baja lógica) del requisito original, sobre el nuevo modelo.**

---

### 3.4 Endpoint detallado — `GET /Cliente/Validar/{idCliente}`

**Propósito:** Validar si un cliente cumple con los documentos regulatorios requeridos (Licencia Sanitaria + Aviso de Responsable Sanitario activos). **Diseño extensible** para incorporar futuras reglas de bloqueo de tramitación.

**Response (200 OK):**
```json
{
  "esValido": true,
  "reglas": [
    {
      "clave": "DOCUMENTOS_REGULATORIOS",
      "cumple": true,
      "detalle": [
        { "clave": "LicenciaSanitaria",         "cumple": true,  "idArchivoDominio": "guid" },
        { "clave": "AvisoResponsableSanitario", "cumple": false, "idArchivoDominio": null }
      ],
      "mensaje": "Falta cargar Aviso de Responsable Sanitario"
    }
  ]
}
```

**Implementación inicial (R16):** una sola regla — verificar que existan `ArchivoDominio` activos para los `CatalogoTipoDocumento` con clave `LicenciaSanitaria` y `AvisoResponsableSanitario` del dominio `Cliente`, vinculados al `idCliente`.

**Diseño extensible — Patrón Chain of Validators:**

```csharp
// Logic.Pqf.Catalogos\Validacion\IValidadorCliente.cs
public interface IValidadorCliente
{
    string Clave { get; }
    Task<ResultadoRegla> Validar(Guid idCliente);
}

// Logic.Pqf.Catalogos\Validacion\ValidadorDocumentosRegulatorios.cs
public class ValidadorDocumentosRegulatorios : IValidadorCliente
{
    public string Clave => "DOCUMENTOS_REGULATORIOS";
    public async Task<ResultadoRegla> Validar(Guid idCliente) { ... }
}

// ValidacionClienteController invoca todos los IValidadorCliente registrados
// y consolida en una sola respuesta. Para R16 solo hay uno; en el futuro se
// suman más sin tocar el controller ni el contrato de respuesta.
```

**Consumidores:**
- `Pretramitar Pedido` (R16A-RE-FU-009) consulta este endpoint antes de permitir un pedido con sustancias controladas.
- En el futuro: validación de límite de crédito, validación de zona, etc.

---

### 3.5 Endpoint detallado — `POST /ArchivoDominio/QueryResult`

**Propósito:** Listar archivos vinculados a entidades de dominio usando el patrón estándar del proyecto `QueryInfo` / `QueryResult<T>`. Soporta filtros dinámicos, paginación, orden y búsqueda — equivalente al patrón usado en otros controladores del catálogo (`ClienteDatosBancariosController.QueryResult`, etc.).

**Request:**

```
POST /ArchivoDominio/QueryResult
Content-Type: application/json
```

**Body (`QueryInfo`):**

```json
{
  "filters": [
    { "field": "IdDominioEntidadArchivo", "operator": "Equal", "value": "<guid>" },
    { "field": "CatalogoTipoDocumento.Clave", "operator": "Equal", "value": "LicenciaSanitaria" },
    { "field": "Activo", "operator": "Equal", "value": true }
  ],
  "orderBy": [
    { "field": "CatalogoTipoDocumento.Orden", "direction": "Asc" },
    { "field": "FechaRegistro", "direction": "Desc" }
  ],
  "skip": 0,
  "take": 50,
  "search": null
}
```

**Filtros típicos esperados desde el frontend:**

| Caso de uso | Filtros sugeridos |
|---|---|
| Documentos vigentes de un cliente | `IdDominioEntidadArchivo = {idCliente}` + `Activo = true` |
| Un tipo específico vigente | + `CatalogoTipoDocumento.Clave = "LicenciaSanitaria"` |
| Historial completo (incluye inactivos) | `IdDominioEntidadArchivo = {idCliente}` (sin filtro `Activo`) |
| Documentos por dominio (Cliente / Proveedor / Producto) | `CatalogoTipoDocumento.catDominioEntidad.Clave = "Cliente"` |

**Response (`QueryResult<ArchivoDominioDto>`):**

```json
{
  "data": [
    {
      "idArchivoDominio": "guid",
      "idArchivo": "guid",
      "fileKey": "ruta/del/archivo.pdf",
      "fileBucket": "documentos-regulatorios",
      "idCatalogoTipoDocumento": "guid",
      "catalogoTipoDocumento": {
        "clave": "LicenciaSanitaria",
        "nombre": "Licencia Sanitaria",
        "formatoAceptado": "application/pdf",
        "orden": 1
      },
      "idDominioEntidadArchivo": "guid",
      "dominioEntidad": "Cliente",
      "activo": true,
      "fechaRegistro": "2026-06-26T10:00:00",
      "fechaUltimaActualizacion": "2026-06-26T10:00:00"
    }
  ],
  "total": 2,
  "skip": 0,
  "take": 50
}
```

**Notas de implementación:**

- El BO `ArchivoDominioBO.QueryResult(QueryInfo info)` hereda del patrón `TablaGenericaBO<T>.QueryResult` ya usado en el proyecto. El `Include` debe traer `Archivo` y `CatalogoTipoDocumento` (con su `catDominioEntidad`) para que el `Dto` retorne datos enriquecidos sin necesidad de N+1 consultas adicionales.
- Mantiene consistencia con el resto de los controladores del Catálogo: mismo verbo HTTP (`POST` con `QueryInfo` en body), misma estructura de respuesta (`QueryResult<T>`).
- Permite que el frontend reutilice el componente de grid/tabla genérico que ya consume `QueryResult`.

---

### 3.6 Endpoint detallado — `GET /ArchivoDominio/{id}/Pdf`

**Propósito:** Entregar el contenido binario del archivo vinculado (idéntico al GAP-04 del Back original, adaptado al nuevo modelo).

**Response:** `HttpResponseMessage` con `Content-Type: application/pdf` (o el mimeType correspondiente) y `Content-Disposition: inline`.

---

### 3.7 Impacto en clases existentes

| Archivo | Cambio |
|---|---|
| `Logic.Pqf.Catalogos\Archivos\ArchivoBO.cs` | Nuevo método `Subir(file, bucket, region)` que generaliza `SubirArchivoDesdePeticion` |
| `Logic.Pqf.Catalogos\Dominio\ArchivoDominioBO.cs` | **Nuevo** — CRUD + lógica de reemplazo Activo=0 + validación de dominio polimorfo |
| `Logic.Pqf.Catalogos\Dominio\CatalogoTipoDocumentoBO.cs` | **Nuevo** — CRUD genérico |
| `Logic.Pqf.Catalogos\Dominio\catDominioEntidadBO.cs` | **Nuevo** — CRUD genérico |
| `Logic.Pqf.Catalogos\Validacion\IValidadorCliente.cs` | **Nuevo** — interfaz para chain of validators |
| `Logic.Pqf.Catalogos\Validacion\ValidadorDocumentosRegulatorios.cs` | **Nuevo** — primer validador |
| `WebApi.Catalogos\Controllers\Archivos\ArchivoController.cs` | Agregar `POST /Archivo/Subir` |
| `WebApi.Catalogos\Controllers\Dominio\ArchivoDominioController.cs` | **Nuevo** — CRUD `/ArchivoDominio` + `/Pdf` |
| `WebApi.Catalogos\Controllers\Validacion\ValidacionClienteController.cs` | **Nuevo** — `GET /Validar/Cliente/{id}/DocumentosRegulatorios` |
| `Logic.Pqf.Catalogos\Clientes\Relaciones\ArchivoClienteBO.cs` | Marcado como **deprecado** durante transición; consumidores migran al nuevo modelo. Sin DELETE en esta release. |

---

## 4. Tareas (manteniendo numeración 1–6 del 003 original)

> Cada tarea de la propuesta reemplaza/redefine la tarea con el mismo número del archivo `R16A-RE-FU-003-Tareas.md` original, adaptada al modelo `ArchivoDominio`.

---

### Tarea 1

#### R16A-RE-FU-003 GAP-01 [ BD-OBJ-CH ] Crear modelo `ArchivoDominio` + `catDominioEntidad` + `CatalogoTipoDocumento` y `ArchivoDominioBO` con lógica de reemplazo

**Aplicativos:** ProquifaDotNet (BD + Logic)

**Módulos:** Catálogo de Clientes — Documentos Regulatorios + Dominio Genérico de Archivos

**Descripción:**
Crear las 3 tablas nuevas (`catDominioEntidad`, `CatalogoTipoDocumento`, `ArchivoDominio`) con sus índices, constraints y datos iniciales. Implementar `ArchivoDominioBO` con la lógica de reemplazo atómico (`Activo = 0` del registro anterior + `INSERT` del nuevo) en `_GuardarOActualizar`, incluyendo validación de dominio polimorfo (`IdDominioEntidadArchivo` debe existir en la tabla del dominio indicado por `CatalogoTipoDocumento.IdcatDominioEntidad`).

**Cambios requeridos:**
1. Scripts DDL para `catDominioEntidad`, `CatalogoTipoDocumento`, `ArchivoDominio` (ver Sección 2.2).
2. Índice único filtrado `UQ_ArchivoDominio_Vigente (IdDominioEntidadArchivo, IdCatalogoTipoDocumento) WHERE Activo = 1`.
3. INSERTs iniciales en `catDominioEntidad` (Cliente / Proveedor / Producto) y `CatalogoTipoDocumento` (LicenciaSanitaria / AvisoResponsableSanitario para dominio Cliente).
4. BO `ArchivoDominioBO` con `_GuardarOActualizar` que valida dominio + mimeType + tamaño + aplica baja lógica del registro anterior + INSERT.
5. Migración de datos desde `ArchivoCliente` (script de Sección 2.3).

**Archivos a modificar / crear:**
- BD: scripts DDL + DML
- `Logic.Pqf.Catalogos\Dominio\ArchivoDominioBO.cs` (nuevo)
- `Logic.Pqf.Catalogos\Dominio\CatalogoTipoDocumentoBO.cs` (nuevo)
- `Logic.Pqf.Catalogos\Dominio\catDominioEntidadBO.cs` (nuevo)

**Criterios de aceptación:**
- [ ] Las 3 tablas existen con PK, FKs e índices definidos
- [ ] Índice único filtrado garantiza una sola versión vigente por (entidad, tipo)
- [ ] Datos iniciales de `catDominioEntidad` y `CatalogoTipoDocumento` cargados
- [ ] Migración SSIS/SQL de `ArchivoCliente` → `ArchivoDominio` ejecutada y conteos validados
- [ ] `ArchivoDominioBO._GuardarOActualizar` rechaza inserción si `IdDominioEntidadArchivo` no existe en la tabla del dominio
- [ ] `ArchivoDominioBO._GuardarOActualizar` rechaza inserción si mimeType del archivo no está en `FormatoAceptado`
- [ ] `ArchivoDominioBO._GuardarOActualizar` rechaza inserción si tamaño excede `TamanioMaximoKB`
- [ ] Al cargar una segunda versión del mismo tipo para la misma entidad, el registro anterior queda `Activo = 0` y el nuevo `Activo = 1`
- [ ] PR aprobado por líder técnico

**Más información:**
- Sustituye la corrección puntual de `ArchivoClienteBO._GuardarOActualizar` del GAP-01 original por la lógica equivalente en el nuevo BO genérico.
- `ArchivoClienteBO` queda deprecado pero vivo durante la transición.

**Recursos:**
- `Requisitos/R16A-RE-FU-003/R16A-RE-FU-003-Propuesta1.md` (este documento) §2 y §3.7
- `Requisitos/R16A-RE-FU-003/R16A-RE-FU-003.md` (Reglas R3, R4)

---

### Tarea 2

#### R16A-RE-FU-003 GAP-02 [ SERV-COMPLEX-TRANSACT ] Endpoint genérico `POST /Archivo/Subir`

**Aplicativos:** ProquifaDotNet

**Módulos:** Archivos — Upload Genérico

**Descripción:**
Crear un endpoint genérico para subir un archivo a MinIO y registrarlo en `Archivo`, **independiente del dominio que lo consuma**. Recibe `file + bucket + region`, retorna el objeto `Archivo` creado. Su responsabilidad termina al retornar — no vincula a ninguna entidad.

**Cambios requeridos:**

```
POST /Archivo/Subir
Content-Type: multipart/form-data

file:    [binary]
bucket:  string
region:  Guid (IdRegion)

Response 200:
{
  "idArchivo": "guid",
  "fileKey": "string",
  "fileBucket": "string",
  "idRegion": "guid",
  "fechaRegistro": "datetime",
  "activo": true
}
```

Implementación:
1. Validar que `file` no esté vacío.
2. Validar que `bucket` esté configurado (existe en `RegionConfiguracionMinioBucket` o lista permitida).
3. Generar `FileKey` único (estrategia: `{region}/{yyyy}/{mm}/{Guid}.{ext}`).
4. Subir a MinIO mediante `MinioStorageService`.
5. `INSERT` en `Archivo` con `Activo = 1`.
6. Retornar el `Archivo` creado.

**Archivos a modificar / crear:**
- `Logic.Pqf.Catalogos\Archivos\ArchivoBO.cs` (nuevo método `Subir(file, bucket, region)`)
- `WebApi.Catalogos\Controllers\Archivos\ArchivoController.cs` (nuevo endpoint `POST /Archivo/Subir`)

**Criterios de aceptación:**
- [ ] Un archivo válido se sube a MinIO en el bucket indicado y queda registrado en `Archivo` con `Activo = 1`
- [ ] Se devuelve el `Archivo` con `IdArchivo`, `FileKey`, `FileBucket`, `IdRegion`
- [ ] Si el bucket no es válido o `region` no existe, retorna 400 con mensaje claro
- [ ] El endpoint NO valida mimeType (el consumidor lo hace antes); endpoint es genérico
- [ ] PR aprobado por líder técnico

**Más información:**
- Sustituye el endpoint específico de carga regulatoria del GAP-02 original. La especificidad de "documento regulatorio del cliente" pasa a la Tarea 3 (`POST /ArchivoDominio`).
- Reutiliza la infraestructura existente de MinIO de `SubirArchivoDesdePeticion`.

**Recursos:**
- `Requisitos/R16A-RE-FU-003/R16A-RE-FU-003-Propuesta1.md` §3.2

---

### Tarea 3

#### R16A-RE-FU-003 GAP-03 [ SERV-COMPLEX-TRANSACT ] Endpoint `PUT /ArchivoDominio` (vincular) y `POST /ArchivoDominio/QueryResult` (consulta con `QueryInfo`)

**Aplicativos:** ProquifaDotNet

**Módulos:** Dominio Genérico de Archivos

**Descripción:**
Crear los endpoints REST sobre `ArchivoDominio` que consumen el resultado de `POST /Archivo/Subir` (Tarea 2) y aplican la lógica de reemplazo del BO (Tarea 1). Incluye endpoint de consulta usando el patrón estándar `QueryInfo` / `QueryResult<T>` del proyecto (mismo patrón que `ClienteDatosBancariosController.QueryResult`).

**Cambios requeridos:**

```
PUT /ArchivoDominio
Body: { idArchivo, idCatalogoTipoDocumento, idDominioEntidadArchivo }
Response 200: ArchivoDominio creado

POST /ArchivoDominio/QueryResult
Body: QueryInfo (filters, orderBy, skip, take, search)
Response 200: QueryResult<ArchivoDominioDto> con Archivo + CatalogoTipoDocumento enriquecidos
```

Implementación:
1. `PUT` invoca `ArchivoDominioBO._GuardarOActualizar` (Tarea 1), que valida dominio + mimeType + tamaño y aplica reemplazo atómico.
2. `POST .../QueryResult` invoca `ArchivoDominioBO.QueryResult(QueryInfo info)` que hereda del patrón `TablaGenericaBO<T>.QueryResult`. El BO debe incluir el `Include` de `Archivo` y `CatalogoTipoDocumento` (con su `catDominioEntidad`) para evitar N+1 y poder filtrar por campos relacionados (`CatalogoTipoDocumento.Clave`, `CatalogoTipoDocumento.catDominioEntidad.Clave`).

**Archivos a modificar / crear:**
- `WebApi.Catalogos\Controllers\Dominio\ArchivoDominioController.cs` (nuevo controller con `PUT` y `POST /QueryResult`)
- `Logic.Pqf.Catalogos\Dominio\ArchivoDominioBO.cs` (sobrescribir `QueryResult` si se requiere personalizar `Include`/proyección al DTO)
- `Logic.Pqf.Catalogos\Dominio\Dto\ArchivoDominioDto.cs` (DTO de respuesta enriquecido — opcional si se prefiere retornar la entidad cruda)

**Criterios de aceptación:**
- [ ] `PUT /ArchivoDominio` crea registro nuevo y desactiva anterior del mismo tipo para la misma entidad
- [ ] `PUT /ArchivoDominio` retorna 400 si dominio, mimeType o tamaño no validan
- [ ] `POST /ArchivoDominio/QueryResult` recibe `QueryInfo` y retorna `QueryResult<ArchivoDominioDto>` con paginación
- [ ] El `QueryResult` permite filtrar por `IdDominioEntidadArchivo`, `Activo`, `CatalogoTipoDocumento.Clave` y `CatalogoTipoDocumento.catDominioEntidad.Clave`
- [ ] El `QueryResult` permite ordenar por `CatalogoTipoDocumento.Orden` y `FechaRegistro`
- [ ] La respuesta incluye datos enriquecidos (`Archivo.FileKey`, `CatalogoTipoDocumento.Nombre`, etc.) sin requerir consultas adicionales del frontend (sin N+1)
- [ ] Depende de Tarea 1 (BO) y Tarea 2 (subir archivo)
- [ ] PR aprobado por líder técnico

**Más información:**
- Sustituye el endpoint específico de consulta de documentos regulatorios del GAP-03 original con uno genérico aplicable a Cliente / Proveedor / Producto / …

**Recursos:**
- `Requisitos/R16A-RE-FU-003/R16A-RE-FU-003-Propuesta1.md` §3.3 y §3.5

---

### Tarea 4

#### R16A-RE-FU-003 GAP-04 [ IMP-EXIST-SERVICE ] Endpoint `GET /ArchivoDominio/{id}/Pdf` para entregar archivo binario

**Aplicativos:** ProquifaDotNet

**Módulos:** Dominio Genérico de Archivos

**Descripción:**
Endpoint que descarga el archivo desde MinIO y lo entrega al cliente HTTP con el `Content-Type` correcto (PDF o el mimeType correspondiente) para que el navegador lo abra inline.

**Cambios requeridos:**

```
GET /ArchivoDominio/{idArchivoDominio}/Pdf

Response 200:
Content-Type: application/pdf
Content-Disposition: inline; filename="documento.pdf"
Body: bytes del archivo
```

Implementación:
1. Obtener `ArchivoDominio` por id.
2. Obtener `IdArchivo` vinculado.
3. Invocar `ArchivoBO.DescargarArregloBytes(idArchivo)` (existente).
4. Determinar mimeType desde el `CatalogoTipoDocumento.FormatoAceptado` (si tiene un solo formato, usar ese; si tiene varios, inferir por extensión del `FileKey`).
5. Retornar `HttpResponseMessage` con `Content-Type` y `Content-Disposition: inline`.

**Archivos a modificar / crear:**
- `WebApi.Catalogos\Controllers\Dominio\ArchivoDominioController.cs` (endpoint adicional `/Pdf`)

**Criterios de aceptación:**
- [ ] El endpoint retorna el contenido binario con el `Content-Type` correcto
- [ ] El header `Content-Disposition: inline` está presente
- [ ] Si el archivo no existe en MinIO, retorna 404 controlado (no 500)
- [ ] Si el `ArchivoDominio` está `Activo = 0`, opcionalmente se sirve igual (decisión: sí, para descarga del histórico)
- [ ] Reutiliza `ArchivoBO.DescargarArregloBytes` existente
- [ ] PR aprobado por líder técnico

**Más información:**
- Sustituye el endpoint específico `GET /ArchivoCliente/.../Pdf` del GAP-04 original con uno genérico aplicable a cualquier dominio.

**Recursos:**
- `Requisitos/R16A-RE-FU-003/R16A-RE-FU-003-Propuesta1.md` §3.6

---

### Tarea 5

#### R16A-RE-FU-003 GAP-05 [ SERV-COMPLEX-TRANSACT ] Endpoint `GET /Validar/Cliente/{idCliente}/DocumentosRegulatorios` (extensible)

**Aplicativos:** ProquifaDotNet

**Módulos:** Validación de Cliente — Reglas Bloqueantes de Tramitación

**Descripción:**
Endpoint que valida si un cliente cumple con los documentos regulatorios requeridos para tramitar pedidos con sustancias controladas. **Diseñado con patrón Chain of Validators** para incorporar reglas adicionales en releases futuras sin tocar el contrato de salida.

**Cambios requeridos:**

```
GET /Validar/Cliente/{idCliente}/DocumentosRegulatorios

Response 200:
{
  "esValido": true,
  "reglas": [
    {
      "clave": "DOCUMENTOS_REGULATORIOS",
      "cumple": true,
      "detalle": [
        { "clave": "LicenciaSanitaria",         "cumple": true,  "idArchivoDominio": "guid" },
        { "clave": "AvisoResponsableSanitario", "cumple": false, "idArchivoDominio": null }
      ],
      "mensaje": "Falta cargar Aviso de Responsable Sanitario"
    }
  ]
}
```

Implementación:
1. Definir interfaz `IValidadorCliente` con método `Task<ResultadoRegla> Validar(Guid idCliente)` y propiedad `Clave`.
2. Implementar `ValidadorDocumentosRegulatorios : IValidadorCliente`:
   - Consultar `CatalogoTipoDocumento` activo del dominio `Cliente` con claves `LicenciaSanitaria` y `AvisoResponsableSanitario`.
   - Verificar que existan `ArchivoDominio` activos para el `idCliente` con esos tipos.
   - Retornar detalle por tipo + `cumple` agregado.
3. Controller `ValidacionClienteController` recibe la lista de `IValidadorCliente` (vía DI / contenedor existente), los ejecuta y consolida en la respuesta.
4. Hoy se registra solo `ValidadorDocumentosRegulatorios`; en el futuro se agregan más implementaciones sin tocar controller ni contrato.

**Archivos a modificar / crear:**
- `Logic.Pqf.Catalogos\Validacion\IValidadorCliente.cs` (nuevo)
- `Logic.Pqf.Catalogos\Validacion\ResultadoRegla.cs` (nuevo DTO)
- `Logic.Pqf.Catalogos\Validacion\ValidadorDocumentosRegulatorios.cs` (nuevo)
- `WebApi.Catalogos\Controllers\Validacion\ValidacionClienteController.cs` (nuevo)
- Registro DI / contenedor para descubrir todos los `IValidadorCliente` automáticamente

**Criterios de aceptación:**
- [ ] El endpoint retorna `esValido = true` cuando el cliente tiene ambos documentos activos
- [ ] El endpoint retorna `esValido = false` cuando falta alguno, con detalle por tipo
- [ ] El contrato de respuesta permite agregar nuevas reglas (nuevos `IValidadorCliente`) sin cambios para el consumidor
- [ ] Pretramitar Pedido (R16A-RE-FU-009) puede consumir este endpoint como reemplazo de su consulta directa a la tabla
- [ ] PR aprobado por líder técnico

**Más información:**
- Reemplaza la responsabilidad implícita que Pretramitar Pedido tenía de consultar directamente `ArchivoCliente`. Ahora consume un servicio dedicado y extensible.
- Sustituye la Tarea 5 original (carga de datos iniciales en `catUsoArchivoSistema`) — los datos iniciales del nuevo modelo ya quedan en la Tarea 1 (`catDominioEntidad` + `CatalogoTipoDocumento`).

**Recursos:**
- `Requisitos/R16A-RE-FU-003/R16A-RE-FU-003-Propuesta1.md` §3.4

---

### Tarea 6

#### R16A-RE-FU-003 DOC-GUIDE — Guía operativa sobre `ArchivoDominio` + plan de deprecación de `ArchivoCliente`

**Aplicativos:** ProquifaDotNet — Soporte a la Producción

**Módulos:** Documentación Operativa — Dominio Genérico de Archivos

**Descripción:**
Actualizar la guía de Soporte a la Producción (heredada del DOC-GUIDE original) para que opere sobre el nuevo modelo `ArchivoDominio` y documentar el plan de deprecación de `ArchivoCliente` / `catUsoArchivoSistema`.

**Cambios requeridos:**

1. **Guía operativa actualizada** con:
   - Cómo consultar estado actual de documentos por entidad (Cliente / Proveedor / Producto) usando `ArchivoDominio` + `CatalogoTipoDocumento`.
   - Scripts de alta / reemplazo manual sobre `ArchivoDominio` (con validación previa de dominio polimorfo).
   - Scripts de baja lógica.
   - Reglas operativas (sin DELETE, versión única activa, transacciones).

2. **Plan de deprecación** documentado:
   - Release N: ambos modelos vivos. Nuevos consumidores usan `ArchivoDominio`. `ArchivoCliente` solo lectura.
   - Release N+1: migrar consumidores restantes de `ArchivoCliente` a `ArchivoDominio` (especialmente Pretramitar Pedido).
   - Release N+2: marcar `ArchivoCliente` y `catUsoArchivoSistema` como obsoletas. Validar que no hay escritura.
   - Release N+3 (o ventana acordada): DROP de las tablas y limpieza de código deprecado.

3. **Mapeo `ArchivoCliente` ↔ `ArchivoDominio`** documentado para que Soporte pueda hacer correlación de incidencias entre modelos.

**Criterios de aceptación:**
- [ ] Documento publicado en el repositorio de Soporte a la Producción
- [ ] Scripts validados en ambiente de desarrollo
- [ ] Plan de deprecación revisado y aprobado por arquitectura y líder técnico
- [ ] Mapeo de tablas / endpoints viejo↔nuevo documentado

**Más información:**
- Mantiene el espíritu de la Tarea 6 original (DOC-GUIDE) pero adaptado al modelo de la propuesta.
- Pendiente: si la purga física de MinIO entra en alcance R16, esta guía deberá actualizarse.

**Recursos:**
- `Requisitos/R16A-RE-FU-003/R16A-RE-FU-003-Propuesta1.md` (este documento)
- `Requisitos/R16A-RE-FU-003/R16A-RE-FU-003-Tareas.md` (Tarea 6 original, como referencia)

---

## 5. Pendientes / Decisiones abiertas de la Propuesta

| # | Pendiente | Responsable |
|---|---|---|
| P1 | Confirmar `TamanioMaximoKB` por tipo de documento (propuesta: 10 MB / 10240 KB para los regulatorios). | Regulatorios / Negocio |
| P2 | Confirmar nombre del bucket de MinIO para documentos regulatorios. | Infraestructura |
| P3 | Decidir si la validación de dominio polimorfo (`IdDominioEntidadArchivo` existe en la tabla correcta) se hace en BO o vía trigger BD. Propuesta: BO. | Arquitectura |
| P4 | Decidir ventana de transición vs. deprecación de `ArchivoCliente` y `catUsoArchivoSistema` (cuántos releases conviven). | Arquitectura / Tech Lead |
| P5 | Confirmar si Pretramitar Pedido (R16A-RE-FU-009) migra a consumir `GET /Validar/Cliente/...` en esta release o en la siguiente. | Backend |
| P6 | Confirmar política de DI para descubrir `IValidadorCliente` (atributo + reflexión vs. registro explícito). | Arquitectura |

---

## 6. Beneficios de la Propuesta vs. modelo actual

| Aspecto | Modelo actual | Propuesta 1 |
|---|---|---|
| Reutilización entre dominios | Una tabla por dominio (`ArchivoCliente`, `ArchivoProveedor`, …) | Una sola tabla `ArchivoDominio` parametrizada |
| Metadata UI en BD | Hardcodeada en código FE/BE | En `CatalogoTipoDocumento` (FormatoAceptado, Orden, TamanioMaximoKB) |
| Endpoint upload | Acoplado al dominio | Genérico (`POST /Archivo/Subir`) |
| Validación de bloqueo | Acoplada en Pretramitar Pedido | Endpoint `Validar` extensible (Chain of Validators) |
| Agregar nueva regla bloqueante | Requiere cambios en cada consumidor | Implementar nuevo `IValidadorCliente`, registrar en DI |
| Agregar nuevo tipo de documento | ALTER + datos + cambios en código | INSERT en `CatalogoTipoDocumento` |
| Trazabilidad | Solo activos vs. inactivos | Misma + posibilidad de filtrar por dominio o tipo en consulta unificada |

---

## 7. Riesgos de la Propuesta

| # | Riesgo | Mitigación |
|---|---|---|
| 1 | `IdDominioEntidadArchivo` sin FK física puede degradar integridad si el BO falla | Validación en BO obligatoria + tests automatizados; opcional: trigger BD como segunda capa |
| 2 | Convivencia de `ArchivoCliente` y `ArchivoDominio` durante la transición puede dejar datos inconsistentes | Marcar `ArchivoCliente` como solo lectura desde release N; auditoría periódica de paridad |
| 3 | El endpoint `Validar` puede crecer en latencia al sumar reglas | Cada `IValidadorCliente` debe ser idempotente y rápido (<200 ms); cache por idCliente con TTL corto si crece |
| 4 | La validación de mimeType depende de `FormatoAceptado` correctamente poblado | Tarea 1 incluye datos iniciales; validación de coherencia en QA |

---

## 8. Resumen para decisión

Esta propuesta **mantiene los 6 entregables del 003 original** (alineado en numeración) pero los reorienta sobre un modelo genérico reutilizable. Los criterios de aceptación de cada tarea siguen cubriendo las Reglas R1–R5 del requisito funcional. El esfuerzo neto adicional vs. el modelo actual está en:

- +1 tabla nueva (`catDominioEntidad`) y +1 catálogo (`CatalogoTipoDocumento` vs `catUsoArchivoSistema`).
- +1 endpoint genérico (`/Archivo/Subir`) que reemplaza la lógica de upload inline.
- +1 endpoint nuevo (`/Validar`) extensible para reglas futuras.
- Migración de datos `ArchivoCliente` → `ArchivoDominio` (1 script).

El beneficio se materializa cuando entren los siguientes módulos que necesiten documentos por dominio (Proveedor, Producto), evitando duplicar 3 tablas + 4 endpoints por cada uno.
