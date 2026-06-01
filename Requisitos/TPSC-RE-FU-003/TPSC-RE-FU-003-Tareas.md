# Tareas — TPSC-RE-FU-003 Catálogo de Clientes — Documentos Regulatorios

| Campo | Valor |
|---|---|
| **Requisito** | TPSC-RE-FU-003 |
| **Nombre** | Mantenimiento de Catálogo de Clientes — Documentos Regulatorios |
| **Total de tareas** | 6 |
| **Revisión aplicada** | TPSC-RE-FU-003_Revision.md |

---

## Tarea 1

### TPSC-RE-FU-003  GAP-01  [ IMP-EXIST-SERVICE ] Corregir lógica de reemplazo en `ArchivoClienteBO._GuardarOActualizar`

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Documentos Regulatorios

**Descripción del problema:**
El método `_GuardarOActualizar` en `ArchivoClienteBO` verifica duplicados por `IdArchivo` + `IdCliente`, pero no desactiva el registro anterior activo del mismo tipo de documento (`IdCatUsoArchivoSistema`) antes de insertar el nuevo. Esto viola la Regla R3: una sola versión vigente por tipo por cliente.

**Cambios requeridos:**
En `_GuardarOActualizar`, antes del `AddOrUpdate`:
1. Buscar si existe un `ArchivoCliente` activo con el mismo `IdCliente` + `IdCatUsoArchivoSistema`.
2. Si existe, actualizar su `Activo = false`.
3. Guardar el cambio.
4. Insertar el nuevo registro con `Activo = true`.

**Archivos a modificar:**
- `Logic.Pqf.Catalogos\Clientes\Relaciones\ArchivoClienteBO.cs`

**Criterios de aceptación:**
- [ ] Al cargar una segunda versión del mismo tipo para el mismo cliente, el registro anterior queda con `Activo = 0`
- [ ] Solo un registro activo por `IdCliente` + `IdCatUsoArchivoSistema` en todo momento
- [ ] El nuevo registro queda con `Activo = 1`
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-01 del archivo `TPSC-RE-FU-003-Back.md`
- Regla R3 del requisito: versión vigente única por tipo por cliente

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003-Back.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003.md`

---

## Tarea 2

### TPSC-RE-FU-003  GAP-02  [ SERV-COMPLEX-TRANSACT ] Endpoint de carga de documento regulatorio (upload + validación PDF + reemplazo atómico)

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Documentos Regulatorios

**Descripción del problema:**
No existe un endpoint dedicado que integre de forma atómica la validación de formato PDF, el upload a MinIO, el registro en `Archivo`, la baja lógica del documento anterior del mismo tipo y el alta del nuevo vínculo en `ArchivoCliente`.

**Cambios requeridos:**

Nuevo endpoint en `ArchivoClienteController`:

```
POST /ArchivoCliente/DocumentoRegulatorio/{idCliente}/{idCatUsoArchivoSistema}
Content-Type: multipart/form-data
```

Lógica a implementar:
1. Validar que el archivo recibido sea PDF (verificar `Content-Type` o extensión `.pdf`).
2. Rechazar la operación si no es PDF (sin escritura en BD ni MinIO).
3. Invocar `ArchivoBO.SubirArchivoDesdePeticion` para subir el archivo a MinIO y obtener el `Archivo` registrado.
4. Invocar el método corregido en `ArchivoClienteBO` (Tarea 1) para registrar el vínculo: desactiva el anterior e inserta el nuevo.

**Archivos a modificar / crear:**
- `WebApi.Catalogos\Controllers\Configuracion\Clientes\Relaciones\ArchivoClienteController.cs` (nuevo endpoint)
- `Logic.Pqf.Catalogos\Clientes\Relaciones\ArchivoClienteBO.cs` (nuevo método `CargarDocumentoRegulatorio`)

**Criterios de aceptación:**
- [ ] Un archivo PDF válido se sube correctamente a MinIO y queda registrado con `Activo = 1` en `ArchivoCliente`
- [ ] Un archivo no PDF es rechazado; no hay escritura en BD ni en MinIO
- [ ] Si existía un registro anterior activo del mismo tipo, queda con `Activo = 0` (reemplazo atómico)
- [ ] Depende de Tarea 1 (corrección de `_GuardarOActualizar`)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-02 del archivo `TPSC-RE-FU-003-Back.md`
- Criterios B1 y D1 del requisito
- Reutilizar `ArchivoBO.SubirArchivoDesdePeticion` existente en `Logic.Pqf.Catalogos\Archivos\ArchivoBO.Upload.Extensions.cs`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003.md`

---

## Tarea 3

### TPSC-RE-FU-003  GAP-03  [ LIST-MULT-FILTER ] Endpoint de consulta de documentos regulatorios vigentes por cliente

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Documentos Regulatorios

**Descripción del problema:**
No existe un endpoint que retorne el estado de los documentos regulatorios vigentes (`Activo = 1`) de un cliente, filtrados por los dos tipos definidos en `catUsoArchivoSistema`. El frontend lo necesita para mostrar si cada tipo de documento está cargado o no.

**Cambios requeridos:**

Nuevo endpoint en `ArchivoClienteController`:

```
GET /ArchivoCliente/DocumentosRegulatorios/{idCliente}
```

Retorna los registros de `ArchivoCliente` con `Activo = 1` del cliente dado, limitados a los `IdCatUsoArchivoSistema` correspondientes a `LicenciaSanitaria` y `AvisoResponsableSanitario`.

Nuevo método en `ArchivoClienteBO`: `ObtenerDocumentosRegulatoriosVigentes(Guid idCliente)`

**Archivos a modificar / crear:**
- `WebApi.Catalogos\Controllers\Configuracion\Clientes\Relaciones\ArchivoClienteController.cs` (nuevo endpoint)
- `Logic.Pqf.Catalogos\Clientes\Relaciones\ArchivoClienteBO.cs` (nuevo método)

**Criterios de aceptación:**
- [ ] El endpoint retorna los registros vigentes (`Activo = 1`) del cliente para los dos tipos de documento regulatorio
- [ ] Si un tipo no tiene documento cargado, no aparece en el resultado
- [ ] Depende de que `catUsoArchivoSistema` tenga los registros requeridos (Tarea 5)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-03 del archivo `TPSC-RE-FU-003-Back.md`
- Criterio A1 del requisito: visibilidad del estado de carga por tipo de documento

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003.md`

---

## Tarea 4

### TPSC-RE-FU-003  GAP-04  [ IMP-EXIST-SERVICE ] Endpoint de entrega del PDF al navegador

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Documentos Regulatorios

**Descripción del problema:**
No existe un endpoint que descargue el PDF desde MinIO y lo entregue al cliente HTTP con los headers correctos para que el navegador lo abra como PDF (Criterio C1 del requisito). El método `ArchivoBO.DescargarArregloBytes` ya existe y puede reutilizarse.

**Cambios requeridos:**

Nuevo endpoint en `ArchivoClienteController`:

```
GET /ArchivoCliente/DocumentoRegulatorio/{idArchivoCliente}/Pdf
```

Lógica:
1. Obtener el `ArchivoCliente` por `idArchivoCliente`.
2. Obtener el `IdArchivo` vinculado.
3. Invocar `ArchivoBO.DescargarArregloBytes(idArchivo)` para obtener los bytes del PDF desde MinIO.
4. Retornar `HttpResponseMessage` con:
   - `Content-Type: application/pdf`
   - `Content-Disposition: inline; filename="documento.pdf"`

**Archivos a modificar:**
- `WebApi.Catalogos\Controllers\Configuracion\Clientes\Relaciones\ArchivoClienteController.cs` (nuevo endpoint)

**Criterios de aceptación:**
- [ ] El endpoint retorna el contenido binario del PDF con `Content-Type: application/pdf`
- [ ] El header `Content-Disposition: inline` está presente
- [ ] Si el archivo no existe en MinIO, el endpoint retorna error controlado (no 500)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-04 del archivo `TPSC-RE-FU-003-Back.md`
- Criterio C1 del requisito: el sistema entrega el PDF al navegador
- Reutilizar `ArchivoBO.DescargarArregloBytes` existente en `Logic.Pqf.Catalogos\Archivos\ArchivoBO.DownloadExtensions.cs`
- Observación de la Revisión: apertura en pestaña nueva no garantizable — el header `inline` es la aproximación correcta

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003.md`

---

## Tarea 5

### TPSC-RE-FU-003  GAP-05  [ QUERY-CH ] Verificar y cargar datos iniciales en `catUsoArchivoSistema`

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogo de Clientes — Documentos Regulatorios

**Descripción del problema:**
La tabla `catUsoArchivoSistema` debe contener los registros `LicenciaSanitaria` y `AvisoResponsableSanitario` para que el módulo pueda clasificar y filtrar los documentos regulatorios. Si no existen, la funcionalidad completa queda inoperante.

**Cambios requeridos:**

1. Verificar en BD de producción (y desarrollo):

```sql
SELECT * FROM dbo.catUsoArchivoSistema
WHERE Clave IN ('LicenciaSanitaria', 'AvisoResponsableSanitario');
```

2. Si no existen, ejecutar script de inserción dentro de transacción:

```sql
BEGIN TRANSACTION;

INSERT INTO dbo.catUsoArchivoSistema (IdCatUsoArchivoSistema, Clave, Descripcion, Activo)
VALUES
    (NEWID(), 'LicenciaSanitaria',         'Licencia Sanitaria',             1),
    (NEWID(), 'AvisoResponsableSanitario', 'Aviso de Responsable Sanitario', 1);

-- Verificar
SELECT * FROM dbo.catUsoArchivoSistema
WHERE Clave IN ('LicenciaSanitaria', 'AvisoResponsableSanitario');

-- COMMIT;
-- ROLLBACK;
COMMIT;
```

3. Documentar los `IdCatUsoArchivoSistema` resultantes — son requeridos por las Tareas 2 y 3 para filtrar correctamente.

**Criterios de aceptación:**
- [ ] Los registros `LicenciaSanitaria` y `AvisoResponsableSanitario` existen en `catUsoArchivoSistema` con `Activo = 1`
- [ ] Los `IdCatUsoArchivoSistema` están documentados y disponibles para las Tareas 2 y 3
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] Pendiente P2 atendido: si Perú requiere denominaciones distintas, se insertan registros adicionales o se confirma que los mismos aplican

**Más información de la tarea:**
- GAP-05 y Pendiente P1 del archivo `TPSC-RE-FU-003-Back.md`
- Pendiente P2: confirmar con cliente si Perú (DIGEMID/SUNAT) requiere registros diferenciados
- Bloqueante para Tareas 2 y 3

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003.md`

---

## Tarea 6

### TPSC-RE-FU-003  DOC-GUIDE — Guía de resolución: Alta, Baja y Actualización de documentos regulatorios por cliente

**Aplicativos:**
ProquifaNet 2 — Soporte a la Producción

**Módulos:**
Documentación Operativa — Catálogo de Clientes — Documentos Regulatorios

**Consideraciones previas:**
Las operaciones de carga, reemplazo y eliminación de documentos regulatorios están disponibles a través de la UI para el rol Regulatorios. Sin embargo, en caso de incidencias o necesidades operativas que requieran intervención directa en BD, Soporte a la Producción debe disponer de esta guía.

**Objetivo general:**
Documentar los pasos para que Soporte a la Producción pueda gestionar de forma segura los registros de `ArchivoCliente` directamente en BD, sin romper la integridad referencial ni dejar duplicados activos.

---

### Precondiciones

| Dato requerido | Cómo obtenerlo |
|---|---|
| `IdCliente` del cliente | `SELECT IdCliente, Nombre FROM dbo.Cliente WHERE Nombre LIKE '%<nombre>%' AND Activo = 1` |
| `IdCatUsoArchivoSistema` del tipo de documento | `SELECT IdCatUsoArchivoSistema, Clave FROM dbo.catUsoArchivoSistema WHERE Activo = 1` |
| `IdArchivo` del archivo ya subido a MinIO | Previamente subido por la UI o proporcionado por Infraestructura |

---

### Consulta de estado actual de documentos regulatorios de un cliente

```sql
SELECT
    ac.IdArchivoCliente,
    ac.IdCliente,
    u.Clave              AS TipoDocumento,
    a.FileKey,
    a.FileBucket,
    a.FechaRegistro,
    ac.Activo
FROM dbo.ArchivoCliente          ac
INNER JOIN dbo.Archivo           a  ON ac.IdArchivo             = a.IdArchivo
INNER JOIN dbo.catUsoArchivoSistema u ON ac.IdCatUsoArchivoSistema = u.IdCatUsoArchivoSistema
WHERE ac.IdCliente = '<IdCliente>'
  AND u.Clave IN ('LicenciaSanitaria', 'AvisoResponsableSanitario')
ORDER BY u.Clave, a.FechaRegistro DESC;
```

---

### Alta / Reemplazo de documento regulatorio

```sql
BEGIN TRANSACTION;

DECLARE @IdCliente               UNIQUEIDENTIFIER = '<IdCliente>';
DECLARE @IdCatUsoArchivoSistema  UNIQUEIDENTIFIER = '<IdCatUsoArchivoSistema>';
DECLARE @IdArchivo               UNIQUEIDENTIFIER = '<IdArchivo del nuevo PDF ya en MinIO>';

-- 1. Desactivar registro anterior activo del mismo tipo para este cliente (si existe)
UPDATE dbo.ArchivoCliente
SET    Activo = 0
WHERE  IdCliente              = @IdCliente
  AND  IdCatUsoArchivoSistema = @IdCatUsoArchivoSistema
  AND  Activo                 = 1;

-- 2. Insertar el nuevo vínculo
INSERT INTO dbo.ArchivoCliente
    (IdArchivoCliente, IdCliente, IdArchivo, IdCatUsoArchivoSistema, Activo)
VALUES
    (NEWID(), @IdCliente, @IdArchivo, @IdCatUsoArchivoSistema, 1);

-- 3. Verificar que solo existe un registro activo del tipo para el cliente
SELECT ac.IdArchivoCliente, u.Clave, a.FileKey, ac.Activo
FROM dbo.ArchivoCliente          ac
INNER JOIN dbo.Archivo           a  ON ac.IdArchivo             = a.IdArchivo
INNER JOIN dbo.catUsoArchivoSistema u ON ac.IdCatUsoArchivoSistema = u.IdCatUsoArchivoSistema
WHERE ac.IdCliente              = @IdCliente
  AND ac.IdCatUsoArchivoSistema = @IdCatUsoArchivoSistema
  AND ac.Activo                 = 1;

-- COMMIT;
-- ROLLBACK;
COMMIT;
```

> ⚠️ El archivo físico debe estar previamente subido a MinIO. Este script solo gestiona el registro lógico en BD.

---

### Baja lógica de documento regulatorio

```sql
BEGIN TRANSACTION;

DECLARE @IdArchivoCliente UNIQUEIDENTIFIER = '<IdArchivoCliente>';

-- 1. Verificar estado actual
SELECT ac.IdArchivoCliente, u.Clave, a.FileKey, ac.Activo
FROM dbo.ArchivoCliente          ac
INNER JOIN dbo.Archivo           a  ON ac.IdArchivo             = a.IdArchivo
INNER JOIN dbo.catUsoArchivoSistema u ON ac.IdCatUsoArchivoSistema = u.IdCatUsoArchivoSistema
WHERE ac.IdArchivoCliente = @IdArchivoCliente;

-- 2. Aplicar baja lógica
UPDATE dbo.ArchivoCliente
SET    Activo = 0
WHERE  IdArchivoCliente = @IdArchivoCliente;

-- 3. Verificar resultado
SELECT IdArchivoCliente, Activo FROM dbo.ArchivoCliente
WHERE IdArchivoCliente = @IdArchivoCliente;

-- COMMIT;
-- ROLLBACK;
COMMIT;
```

> ⚠️ El archivo físico permanece en MinIO. No se realiza purga automática (comportamiento intencional en R16).
> ⚠️ Dar de baja ambos documentos de un cliente bloqueará la pretramitación de pedidos con sustancias controladas (TPSC-RE-FU-009).

---

### Reglas operativas

| Regla | Descripción |
|---|---|
| Sin DELETE físico | Nunca ejecutar `DELETE` sobre `ArchivoCliente`. Siempre baja lógica (`Activo = 0`) |
| Versión única activa | Nunca dejar dos registros con `Activo = 1` del mismo tipo para el mismo cliente |
| Transacción obligatoria | Toda operación dentro de `BEGIN TRANSACTION … COMMIT/ROLLBACK` |
| Verificación previa | Siempre consultar el estado actual antes de modificar |
| Verificación posterior | Confirmar resultado antes de `COMMIT` |
| Archivo físico previo | El `IdArchivo` debe existir en `Archivo` con `Activo = 1` antes del INSERT en `ArchivoCliente` |

---

**Criterios de aceptación de la tarea:**
- [ ] Documento de guía publicado en el repositorio de documentación de Soporte a la Producción
- [ ] Scripts SQL de alta/reemplazo y baja validados en ambiente de desarrollo
- [ ] Guía revisada y aprobada por el líder técnico

**Más información de la tarea:**
- Criterios D1 y D2 del requisito (reemplazo y eliminación)
- Pendiente P3 del Back: si la purga física de MinIO entra en alcance R16, esta guía deberá actualizarse

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003.md`
- Revisión funcional: `Requisitos/TPSC-RE-FU-003/TPSC-RE-FU-003_Revision.md`