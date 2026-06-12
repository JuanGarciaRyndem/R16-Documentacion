# Análisis de Impacto Backend — R16A-RE-FU-003
# Catálogo de Clientes — Documentos Regulatorios (Licencia Sanitaria y Aviso de Responsable Sanitario)

| Campo | Valor |
|---|---|
| **Requisito** | R16A-RE-FU-003 |
| **Nombre** | Mantenimiento de Catálogo de Clientes — Documentos Regulatorios |
| **Aplicativo** | ProquifaNet 2 |
| **Base de datos** | ProquifaDotNet |
| **Referencia Legacy** | R16.3M-RE-FU-004 |
| **Revisión aplicada** | R16A-RE-FU-003_Revision.md |
| **Fecha** | 2026-05-29 |

---

## Resumen Ejecutivo

Habilitar la carga, visualización, reemplazo y eliminación de documentos PDF regulatorios (Licencia Sanitaria y Aviso de Responsable Sanitario) por cliente en el módulo Catálogo de Clientes, con acceso exclusivo para el rol Regulatorios. Los documentos se almacenan en MinIO y se referencian en BD mediante las tablas `Archivo` y `ArchivoCliente`. La presencia de ambos documentos es requisito bloqueante para pretramitar pedidos con sustancias controladas (R16A-RE-FU-009).

> **Alcance regional — OBS-007:** Esta funcionalidad aplica **únicamente a Región México**. Las Sustancias Controladas no están habilitadas para Región Perú en R16; por tanto, la sección de Documentos Regulatorios y su validación condicionante en Pretramitar Pedido no deben activarse para Perú.

---

## Reglas de Negocio (declarativas)

| #   | Regla                                                                                                                                                                                                                        |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1  | Solo el usuario con `GestorDeAsuntosRegulatorios = 1` en la tabla `Usuario` puede cargar, reemplazar, eliminar y visualizar documentos regulatorios. Otros roles acceden a la sección en modo bloqueado (lectura de estado). |
| R2  | Solo se aceptan archivos en formato PDF. Cualquier otro formato debe rechazarse en la capa de aplicación antes de realizar el upload a MinIO.                                                                                |
| R3  | Existe una sola versión vigente por tipo de documento por cliente. Al cargar una nueva versión se desactiva (`Activo = 0`) el registro anterior en `ArchivoCliente` antes de insertar el nuevo.                              |
| R4  | La eliminación es lógica: se marca `Activo = 0` en `ArchivoCliente`. El archivo físico en MinIO se conserva sin purga automática (comportamiento existente del backend).                                                     |
| R5  | El sistema no valida fecha de vigencia, contenido, autenticidad ni firmas del documento. El documento se considera vigente por el solo hecho de estar cargado con `Activo = 1`.                                              |

---

## Entidades Afectadas

| Entidad | Tipo | Impacto | Descripción |
|---|---|---|---|
| `ArchivoCliente` | Tabla existente | R / W | Vincula documento PDF con cliente y tipo de uso |
| `Archivo` | Tabla existente | R / W | Referencia al archivo físico en MinIO (`FileKey` + `FileBucket`) |
| `catUsoArchivoSistema` | Catálogo existente | R / Datos iniciales | Clasifica el tipo de documento (LicenciaSanitaria, AvisoResponsableSanitario) |
| `Usuario` | Tabla existente | R | Campo `GestorDeAsuntosRegulatorios` controla el acceso al rol Regulatorios |
| MinIO | Almacenamiento externo | W | Almacenamiento físico de los PDFs cargados |

---

## Análisis de GAPs

### GAP-01 — `ArchivoClienteBO`: lógica de reemplazo (versión vigente única)

**Estado:** ❌ No implementado

**Descripción:**
El método `_GuardarOActualizar` en `ArchivoClienteBO` actualmente verifica si ya existe un registro con el mismo `IdArchivo` e `IdCliente` para evitar duplicados, pero **no desactiva el registro vigente anterior** del mismo tipo de documento (`IdCatUsoArchivoSistema`) antes de insertar el nuevo.

Para garantizar la Regla R3 (una sola versión vigente por tipo por cliente), el método debe:
1. Buscar si existe un registro activo con el mismo `IdCliente` + `IdCatUsoArchivoSistema`.
2. Si existe, marcarlo como `Activo = 0` (baja lógica del anterior).
3. Insertar el nuevo registro con `Activo = 1`.

**Archivos afectados:**
- `Logic.Pqf.Catalogos\Clientes\Relaciones\ArchivoClienteBO.cs`

---

### GAP-02 — Endpoint de carga de documento regulatorio con validación PDF y reemplazo atómico

**Estado:** ❌ No implementado

**Descripción:**
No existe un endpoint dedicado que integre de forma atómica:
1. Validación de formato PDF (antes del upload a MinIO).
2. Upload del archivo a MinIO mediante `ArchivoBO.SubirArchivoDesdePeticion`.
3. Registro en `Archivo`.
4. Baja lógica del registro anterior en `ArchivoCliente` (si existe uno activo del mismo tipo).
5. Alta del nuevo registro en `ArchivoCliente`.

El controlador existente `ArchivoClienteController` expone CRUD genérico pero no implementa esta lógica de negocio específica.

**Archivos afectados:**
- `WebApi.Catalogos\Controllers\Configuracion\Clientes\Relaciones\ArchivoClienteController.cs` (nuevo endpoint)
- `Logic.Pqf.Catalogos\Clientes\Relaciones\ArchivoClienteBO.cs` (nuevo método de extensión)

---

### GAP-03 — Endpoint de consulta de documentos regulatorios vigentes por cliente

**Estado:** ❌ No implementado

**Descripción:**
No existe un endpoint que retorne los documentos regulatorios vigentes (`Activo = 1`) de un cliente, filtrados por los dos tipos de uso del sistema (`LicenciaSanitaria` y `AvisoResponsableSanitario`).

El frontend necesita este endpoint para mostrar el estado de cada tipo de documento (cargado / no cargado) en la sección de Documentos Regulatorios del Catálogo de Clientes.

**Archivos afectados:**
- `WebApi.Catalogos\Controllers\Configuracion\Clientes\Relaciones\ArchivoClienteController.cs` (nuevo endpoint)
- `Logic.Pqf.Catalogos\Clientes\Relaciones\ArchivoClienteBO.cs` (nuevo método de extensión)

---

### GAP-04 — Endpoint de entrega del PDF al navegador

**Estado:** ❌ No implementado

**Descripción:**
No existe un endpoint dedicado que descargue el contenido binario de un documento regulatorio desde MinIO y lo entregue al cliente HTTP con el header `Content-Type: application/pdf` y `Content-Disposition: inline`, para que el navegador lo abra directamente (Criterio C1 del requisito). `ArchivoBO.DescargarArregloBytes` ya existe y puede reutilizarse.

**Archivos afectados:**
- `WebApi.Catalogos\Controllers\Configuracion\Clientes\Relaciones\ArchivoClienteController.cs` (nuevo endpoint)

---

### GAP-05 — Datos iniciales en `catUsoArchivoSistema`

**Estado:** ⚠️ Pendiente de verificación (P1)

**Descripción:**
La tabla `catUsoArchivoSistema` debe contener los registros `LicenciaSanitaria` y `AvisoResponsableSanitario`. Si no existen, el módulo no podrá clasificar ni filtrar los documentos regulatorios. Requiere verificación en BD de producción y, si no existen, ejecutar script de carga inicial.

---

## Pendientes

| # | Pendiente | Responsable |
|---|---|---|
| P1 | Verificar existencia de registros `LicenciaSanitaria` y `AvisoResponsableSanitario` en `catUsoArchivoSistema` en BD de producción | DBA |
| ~~P2~~ | ~~Confirmar denominación exacta de documentos para Perú (DIGEMID/SUNAT) y si requieren registros diferenciados en `catUsoArchivoSistema`~~ | **Cerrado — OBS-007:** Perú fuera de alcance en R16. No aplica. |
| P3 | Definir si la purga física de archivos en MinIO al eliminar/reemplazar entra en alcance R16 | Product Owner |
| P4 | Confirmar bucket de MinIO y estructura de `FileKey` que se usará para documentos regulatorios de clientes | DBA / Infraestructura |

---

## Criterios de Aceptación Técnica

| # | Criterio |
|---|---|
| C1 | Un usuario con `GestorDeAsuntosRegulatorios = 1` puede cargar un PDF; el sistema lo sube a MinIO y registra el vínculo en `ArchivoCliente` con `Activo = 1`. |
| C2 | Al cargar una nueva versión del mismo tipo de documento para el mismo cliente, el registro anterior queda con `Activo = 0` y el nuevo con `Activo = 1`. Solo un registro activo por tipo por cliente. |
| C3 | Si el archivo subido no es PDF, el endpoint rechaza la operación y no se realiza ninguna escritura en BD ni en MinIO. |
| C4 | El endpoint de consulta retorna el estado de ambos tipos de documento (`LicenciaSanitaria` y `AvisoResponsableSanitario`) para un cliente dado, con `Activo = 1`. |
| C5 | El endpoint de visualización descarga el PDF desde MinIO y lo entrega con `Content-Type: application/pdf` y `Content-Disposition: inline`. |
| C6 | La eliminación de un documento marca `Activo = 0` en `ArchivoCliente`. El archivo físico en MinIO se conserva sin purga. |

---

## Módulo Consumidor

| Módulo | Requisito | Dependencia |
|---|---|---|
| Pretramitar Pedido | R16A-RE-FU-009 | Requiere `Activo = 1` para ambos tipos de documento del cliente para pedidos con sustancias controladas — **solo Región México** (OBS-007) |

---

## Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| R1 | Documentos cargados pero vencidos legalmente | Revisión periódica offline por Regulatorios — el sistema no valida vigencia |
| R2 | Eliminación accidental bloquea pretramitación de pedidos con sustancias controladas | Confirmación explícita antes de eliminar en UI (Criterio D2 del requisito) |
| ~~R3~~ | ~~Denominación distinta en Perú (DIGEMID/SUNAT) puede requerir registros diferenciados~~ | **Eliminado — OBS-007:** Perú fuera de alcance en R16. Si en una entrega futura se habilitan Sustancias Controladas para Perú, se deberá extender el modelo. |
| R4 | Bucket o `FileKey` no definido para documentos regulatorios de clientes | Pendiente P4 — confirmar con DBA/Infraestructura |