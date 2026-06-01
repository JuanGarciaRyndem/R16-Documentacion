# Diccionario de Datos — TPSC-RE-FU-003 Documentos Regulatorios del Cliente

| Campo | Valor |
|---|---|
| **Requisito** | TPSC-RE-FU-003 |
| **Nombre** | Mantenimiento de Catálogo de Clientes — Documentos Regulatorios |
| **Base de datos** | ProquifaDotNet |
| **Servidor** | RYNL010 |
| **Referencia Legacy** | R16.3M-RE-FU-004 |

---

## Resumen Ejecutivo

Habilitar la carga, visualización, reemplazo y eliminación de documentos PDF regulatorios
(Licencia Sanitaria y Aviso de Responsable Sanitario) por cliente en el Catálogo de Clientes.
La presencia de ambos documentos es requisito bloqueante para pretramitar pedidos con sustancias controladas.

---

## Modelo de Datos

```
Cliente
    └── FK IdCliente
ArchivoCliente  (IdCliente + IdArchivo + IdCatUsoArchivoSistema + Activo)
    └── FK IdArchivo
Archivo  (FileKey, FileBucket → MinIO)
    └── FK IdCatUsoArchivoSistema
catUsoArchivoSistema  (Tipo de documento: LicenciaSanitaria, AvisoResponsableSanitario)
```

---

## Entidades Afectadas

| Objeto | Tipo | Impacto | Descripción |
|---|---|---|---|
| `ArchivoCliente` | Tabla existente | Lectura / Escritura | Vincula documentos PDF con un cliente y su tipo de uso |
| `Archivo` | Tabla existente | Lectura / Escritura | Almacena la referencia al archivo físico en MinIO |
| `catUsoArchivoSistema` | Catálogo existente | Lectura / Datos iniciales | Clasifica el tipo de documento (Licencia Sanitaria / Aviso) |
| `Usuario` | Tabla existente | Lectura | Campo `GestorDeAsuntosRegulatorios` controla el acceso al rol Regulatorios |

---

## 1. ArchivoCliente

**Propósito:** Vincula documentos PDF con un cliente y su tipo de uso (Licencia Sanitaria / Aviso de Responsable Sanitario)
**Creada:** 01/06/2020

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdArchivoCliente` | `uniqueidentifier` | 16 | NO | PK. Default: `NEWID()` |
| `IdCliente` | `uniqueidentifier` | 16 | NO | FK → `Cliente` |
| `IdArchivo` | `uniqueidentifier` | 16 | SÍ | FK → `Archivo` (referencia al PDF en MinIO) |
| `IdCatUsoArchivoSistema` | `uniqueidentifier` | 16 | NO | FK → `catUsoArchivoSistema` (tipo de documento) |
| `Activo` | `bit` | 1 | NO | `1` = Documento vigente visible. `0` = Reemplazado / Eliminado. Default: `1` |

**Índices:**
- `PK_ArchivoCliente` (Clustered): `IdArchivoCliente`

**Reglas de negocio implementadas:**
- `Activo = 1`: Documento vigente (visible en pantalla).
- `Activo = 0`: Documento reemplazado o eliminado (baja lógica, Regla 3 del requisito).
- Un solo registro activo por `IdCliente` + `IdCatUsoArchivoSistema` garantiza una versión vigente por tipo.

---

## 2. Archivo

**Propósito:** Almacena la referencia al archivo físico en MinIO (`FileKey` + `FileBucket`)
**Creada:** 31/08/2020

| Columna                    | Tipo               | Longitud | Nulo | Descripción                               |
| -------------------------- | ------------------ | -------- | ---- | ----------------------------------------- |
| `IdArchivo`                | `uniqueidentifier` | 16       | NO   | PK. Default: `NEWID()`                    |
| `FileKey`                  | `varchar`          | 600      | NO   | Ruta / clave del archivo en MinIO         |
| `FileBucket`               | `varchar`          | 100      | NO   | Bucket de MinIO.                          |
| `Sincronizado`             | `bit`              | 1        | NO   | Indica si está sincronizado. Default: `1` |
| `IdProcesoSistema`         | `uniqueidentifier` | 16       | SÍ   | FK → `ProcesoSistema`                     |
| `IdRegion`                 | `uniqueidentifier` | 16       | SÍ   | FK → `Region`                             |
| `FechaRegistro`            | `datetime`         | 8        | NO   | Default: `GETDATE()`                      |
| `FechaUltimaActualizacion` | `datetime`         | 8        | NO   | Default: `GETDATE()`                      |
| `Activo`                   | `bit`              | 1        | NO   | Default: `1`                              |
## Almacenamiento de archivos: MinIO

Los archivos PDF no se almacenan en la base de datos relacional. Se almacenan en **MinIO** (almacenamiento de objetos). 

| Concepto                         | Descripción                                                                                                                        |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **Bucket sugerido**              | `documentos-regulatorios` (pendiente confirmar nombre con el equipo de infraestructura)                                            |
| **Clave del objeto**             | Formato sugerido: `{IdCliente}/{TipoDocumento}/{IdClienteDocumentoRegulatorio}.pdf`                                                |
| **Comportamiento al reemplazar** | El archivo físico anterior permanece en MinIO sin purga automática. Solo el registro lógico cambia de `Activo = 1` a `Activo = 0`. |
| **Comportamiento al eliminar**   | Igual que el reemplazo: se marca `Activo = 0` en BD. El archivo físico en MinIO no se elimina automáticamente.                     |


> **Nota crítica:** Al eliminar o reemplazar un documento desde la pantalla, se actualiza `ArchivoCliente.Activo = 0` (baja lógica del registro), pero el archivo físico permanece en MinIO sin purga automática. Comportamiento intencional del backend.

---

## 3. catUsoArchivoSistema (Catálogo — crítico)

**Propósito:** Clasifica el tipo de documento regulatorio asociado al cliente
**Creada:** 01/06/2020

| Columna | Tipo | Longitud | Descripción |
|---|---|---|---|
| `IdCatUsoArchivoSistema` | `uniqueidentifier` | 16 | PK |
| `UsoArchivoSistema` | `varchar` | 50 | Nombre del tipo de documento |
| `Activo` | `bit` | 1 | Tipo activo |

> **Pendiente:** Confirmar que existan (o agregar) los registros `LicenciaSanitaria` y `AvisoResponsableSanitario` (o sus equivalentes exactos) en el catálogo.

**Consulta de verificación:**

```sql
-- Verificar si existen los tipos requeridos
SELECT IdCatUsoArchivoSistema, UsoArchivoSistema, Activo
FROM   dbo.catUsoArchivoSistema
ORDER BY UsoArchivoSistema;
```

---

## 4. Usuario (Rol Regulatorios)

**Propósito:** Control de acceso — campo que identifica al rol Regulatorios

| Campo relevante | Tipo | Descripción |
|---|---|---|
| `GestorDeAsuntosRegulatorios` | `bit` | Rol Regulatorios: puede cargar, reemplazar, eliminar y visualizar documentos |
| `Activo` | `bit` | Solo usuarios activos tienen acceso |

**Mapeo de roles del requisito:**

| Rol en requisito | Campo en Usuario | Permiso |
|---|---|---|
| Regulatorios | `GestorDeAsuntosRegulatorios = 1` | Cargar, reemplazar, eliminar, visualizar |
| Cualquier otro rol | `GestorDeAsuntosRegulatorios = 0` | Solo visualizar (modo bloqueado) |

---

## Consultas de Referencia

### Documentos regulatorios vigentes de un cliente

```sql
SELECT
    ac.IdArchivoCliente,
    ac.IdCliente,
    cuas.UsoArchivoSistema    AS TipoDocumento,
    a.FileKey,
    a.FileBucket,
    a.FechaRegistro           AS FechaCarga
FROM       dbo.ArchivoCliente       ac
INNER JOIN dbo.Archivo              a    ON ac.IdArchivo              = a.IdArchivo
INNER JOIN dbo.catUsoArchivoSistema cuas ON ac.IdCatUsoArchivoSistema = cuas.IdCatUsoArchivoSistema
WHERE  ac.IdCliente = @IdCliente
  AND  ac.Activo    = 1
ORDER BY cuas.UsoArchivoSistema;
```

### Verificar si el cliente tiene ambos documentos cargados (consumidor: Pretramitar Pedido)

```sql
DECLARE @IdUsoLicenciaSanitaria UNIQUEIDENTIFIER;
DECLARE @IdUsoAvisoResponsable  UNIQUEIDENTIFIER;

SELECT
    @IdUsoLicenciaSanitaria = IdCatUsoArchivoSistema
FROM dbo.catUsoArchivoSistema WHERE UsoArchivoSistema = 'LicenciaSanitaria' AND Activo = 1;

SELECT
    @IdUsoAvisoResponsable  = IdCatUsoArchivoSistema
FROM dbo.catUsoArchivoSistema WHERE UsoArchivoSistema = 'AvisoResponsableSanitario' AND Activo = 1;

SELECT
    CASE
        WHEN COUNT(DISTINCT ac.IdCatUsoArchivoSistema) = 2 THEN 1
        ELSE 0
    END AS TieneDocumentosCompletos
FROM dbo.ArchivoCliente ac
WHERE ac.IdCliente = @IdCliente
  AND ac.Activo    = 1
  AND ac.IdCatUsoArchivoSistema IN (@IdUsoLicenciaSanitaria, @IdUsoAvisoResponsable);
```

### Reemplazo de documento regulatorio (lógica transaccional)

```sql
BEGIN TRANSACTION;
    -- Paso 1: Marcar documento anterior como inactivo
    UPDATE dbo.ArchivoCliente
    SET    Activo = 0
    WHERE  IdCliente              = @IdCliente
      AND  IdCatUsoArchivoSistema = @IdCatUsoArchivoSistema
      AND  Activo                 = 1;

    -- Paso 2: Registrar nuevo archivo en MinIO
    INSERT INTO dbo.Archivo (IdArchivo, FileKey, FileBucket, FechaRegistro, FechaUltimaActualizacion, Activo)
    VALUES (@IdArchivo, @FileKey, @FileBucket, GETDATE(), GETDATE(), 1);

    -- Paso 3: Vincular nuevo archivo con el cliente y tipo de documento
    INSERT INTO dbo.ArchivoCliente (IdArchivoCliente, IdCliente, IdArchivo, IdCatUsoArchivoSistema, Activo)
    VALUES (@IdArchivoCliente, @IdCliente, @IdArchivo, @IdCatUsoArchivoSistema, 1);
COMMIT TRANSACTION;
```

### Eliminación lógica de documento regulatorio

```sql
DECLARE @IdArchivoCliente UNIQUEIDENTIFIER;

UPDATE dbo.ArchivoCliente
SET    Activo = 0
WHERE  IdArchivoCliente = @IdArchivoCliente;
-- Nota: el archivo físico permanece en MinIO sin purga automática
```

### Clientes sin ambos documentos regulatorios cargados

```sql
DECLARE @IdUsoLicenciaSanitaria UNIQUEIDENTIFIER;
DECLARE @IdUsoAvisoResponsable  UNIQUEIDENTIFIER;

SELECT
    c.IdCliente,
    c.Nombre,
    MAX(CASE WHEN ac.IdCatUsoArchivoSistema = @IdUsoLicenciaSanitaria THEN 1 ELSE 0 END) AS TieneLicenciaSanitaria,
    MAX(CASE WHEN ac.IdCatUsoArchivoSistema = @IdUsoAvisoResponsable  THEN 1 ELSE 0 END) AS TieneAvisoResponsable
FROM       dbo.Cliente       c
LEFT JOIN  dbo.ArchivoCliente ac ON c.IdCliente = ac.IdCliente AND ac.Activo = 1
WHERE c.Activo = 1
GROUP BY c.IdCliente, c.Nombre
HAVING
    MAX(CASE WHEN ac.IdCatUsoArchivoSistema = @IdUsoLicenciaSanitaria THEN 1 ELSE 0 END) = 0
 OR MAX(CASE WHEN ac.IdCatUsoArchivoSistema = @IdUsoAvisoResponsable  THEN 1 ELSE 0 END) = 0
ORDER BY c.Nombre;
```

---

## Reglas de Negocio Implementadas

| Regla | Descripción | Implementación en BD |
|---|---|---|
| Regla 1 | Solo rol Regulatorios puede cargar / reemplazar / eliminar | `WHERE GestorDeAsuntosRegulatorios = 1` en capa aplicación |
| Regla 2 | Solo formato PDF | Validación en capa aplicación antes de insertar en `Archivo` |
| Regla 3 | Una sola versión vigente por tipo por cliente | `UPDATE Activo = 0` antes de `INSERT` nuevo registro |
| Regla 4 | Sin validación de vigencia ni contenido | No hay campos de fecha de vencimiento en el modelo |
| Regla 5 | Aplica México y Perú por igual | Sin distinción de país en `ArchivoCliente` |

---

## Análisis de Gaps

| Gap | Descripción | Acción requerida |
|---|---|---|
| `catUsoArchivoSistema` vacío | La tabla puede no tener los registros requeridos | Insertar: `LicenciaSanitaria` y `AvisoResponsableSanitario` |
| Denominación Perú | Pendiente confirmar si aplican mismas etiquetas | Confirmar con cliente (Riesgo 4 del requisito) |
| Sin `FechaRegistro` en `ArchivoCliente` | No hay fecha de carga en la tabla de vinculación | Usar `Archivo.FechaRegistro` como fecha de carga |
| Sin historial visual | Baja lógica oculta versiones anteriores | Comportamiento intencional documentado (Riesgo 2) |

---

## Módulo Consumidor

| Módulo | Requisito | Validación |
|---|---|---|
| Pretramitar Pedido | TPSC-RE-FU-009 | Ambos documentos `Activo = 1` requeridos para pedidos con sustancias controladas |

---

## Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| 1 | Documentos cargados pero vencidos legalmente | Revisión periódica offline por Regulatorios |
| 2 | Pérdida de trazabilidad histórica por reemplazo | Comportamiento intencional — sin historial visual en R16 |
| 3 | Eliminación accidental bloquea pretramitación | Confirmación explícita antes de eliminar en UI |
| 4 | Denominación distinta en Perú (DIGEMID/SUNAT) | Pendiente confirmación con cliente |

---

## Pendientes

| # | Pendiente | Responsable |
|---|---|---|
| P1 | Confirmar existencia de registros `LicenciaSanitaria` y `AvisoResponsableSanitario` en `catUsoArchivoSistema` | DBA |
| P2 | Confirmar denominación exacta de documentos para Perú (DIGEMID/SUNAT) y si requieren registros diferenciados en el catálogo | Funcional / Cliente |
| P3 | Definir si la purga física de archivos en MinIO al eliminar / reemplazar entra en alcance R16 | Product Owner |
