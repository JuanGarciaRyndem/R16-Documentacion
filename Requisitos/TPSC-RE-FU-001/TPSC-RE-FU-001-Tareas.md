# Tareas — TPSC-RE-FU-001 Catálogo de Cuentas Bancarias del Grupo PROQUIFA

| Campo | Valor |
|---|---|
| **Requisito** | TPSC-RE-FU-001 |
| **Nombre** | Mantenimiento de catálogos del sistema |
| **Total de tareas** | 7 |
| **Revisión aplicada** | TPSC-RE-FU-001-Revision.md |

### Cambios respecto a la versión anterior de Tareas

| #   | Cambio                                                                                                                              | Origen                            |
| --- | ----------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| 1   | Terminología `Activo = true/false` → **existe / no existe vigente en sistema** en todos los criterios y descripciones               | Alineación con Back revisado      |
| 2   | Tarea 4: ya no referencia GAP-04 (eliminado del Back). Ahora referencia **Pendiente P1** del Back y Criterios C1/C2 del requisito   | Back revisado eliminó GAP-04      |
| 3   | Tarea 5: ya no referencia GAP-05 (eliminado del Back). Ahora referencia **Regla 4** y Sección 2 del Back revisado                   | Back revisado: GAP-05 consolidado |
| 4   | Tarea 6: validación pasa a vista `vEmpresaDatosBancariosVigentes` y **Pendiente P4** del Back                                       | Consistencia con Back actualizado |
| 5   | Tarea 7 agregada: Guía de resolución operativa para alta, baja y actualización de cuentas bancarias (Notas adicionales de Revisión) | TPSC-RE-FU-001-Revision.md        |

---
## Tarea 1

### [ TPSC-RE-FU-001 ] [ BD-OBJ-CH ] Construcción de vista vEmpresaDatosBancariosVigentes

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — Base de Datos ProquifaDotNet

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Construir la vista `vEmpresaDatosBancariosVigentes` en la BD `ProquifaDotNet` que centralice el filtro de cuentas que existen vigentes en sistema del grupo PROQUIFA México. Evita que cada módulo consumidor duplique la lógica de filtrado. Cumple la **Regla 4** del requisito: cualquier módulo que requiera cuentas del grupo consulta el catálogo centralizado.

> **Nota:** Esta tarea ya no referencia un GAP numerado del Back (GAP-05 eliminado en la revisión). La necesidad se mantiene y se basa en la **Regla 4** y el análisis de filtrado de prefijos documentados en la Sección 2 del `TPSC-RE-FU-001-Back.md`.

**Objetivos específicos:**
- Leer el diccionario de datos `TPSC-RE-FU-001_BD.md` y la Sección 2 del análisis backend `TPSC-RE-FU-001-Back.md` (Regla 4)
- Diseñar y construir la vista con JOIN entre `EmpresaDatosBancarios`, `Empresa`, `DatosBancarios`, `catBanco` y `catMoneda`
- Aplicar filtros: `EmpresaDatosBancarios.Activo = 1` (existe vigente), `Empresa.Activo = 1`, `Empresa.Prefijo IN ('GOL','MUN','PRO','PQF')`
- Verificar que la vista excluye registros de `GOLPERU` y cuentas que no existen vigentes
- Documentar la vista en el diccionario de datos del requisito

**Resultado esperado:**
`vEmpresaDatosBancariosVigentes` retorna únicamente las cuentas que existen vigentes en sistema de las empresas mexicanas del grupo. Los módulos consumidores hacen un `SELECT` simple sin replicar los filtros.

**Entregables:**
- Script de creación de la vista `vEmpresaDatosBancariosVigentes`
- Vista registrada en el script de control de BD del release

**Criterios de aceptación:**
- [ ] La vista retorna únicamente cuentas que **existen vigentes en sistema** (`Activo = 1`) de empresas con `Prefijo IN ('GOL','MUN','PRO','PQF')`
- [ ] La vista excluye registros de `GOLPERU` y cuentas no vigentes
- [ ] La vista expone al menos: `Prefijo`, `Alias`, `Banco`, `NumeroDeCuenta`, `Clabe`, `ClaveMoneda`, `FechaRegistro`
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Referencia: **Regla 4** y Sección 2 del archivo `TPSC-RE-FU-001-Back.md`
- Tablas involucradas: `EmpresaDatosBancarios`, `Empresa`, `DatosBancarios`, `catBanco`, `catMoneda`
- La vista es de solo lectura; la gestión del catálogo sigue siendo responsabilidad de Soporte a la Producción directamente en BD (Criterios C1 y C2 del requisito)

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---
## Tarea 2

### [ TPSC-RE-FU-001 ] [ IMP-EXIST-SERVICE ] Corrección de ObtenerTodos() para excluir cuentas no vigentes

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Corregir el método `ObtenerTodos()` en `EmpresaDatosBancariosBO.Extensions.cs` para que retorne únicamente cuentas que **existen vigentes en sistema** (`Activo = true`), cumpliendo la Regla 2 y el Criterio B1 del requisito.

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-001 (GAP-01)
- Localizar el método `ObtenerTodos()` en `EmpresaDatosBancariosBO.Extensions.cs`
- Agregar el filtro `Where(x => x.Activo)` al query
- Verificar que ningún módulo consumidor dependa del comportamiento anterior (sin filtro)
- Ejecutar pruebas unitarias sobre el método corregido

**Resultado esperado:**
`ObtenerTodos()` retorna únicamente las cuentas que existen vigentes en sistema. Las cuentas no vigentes (`Activo = false`) no son devueltas a los módulos consumidores pero se conservan en BD para trazabilidad histórica.

**Entregables:**
- `EmpresaDatosBancariosBO.Extensions.cs` modificado con filtro `Where(x => x.Activo)`
- Pruebas unitarias del método

**Criterios de aceptación:**
- [ ] `ObtenerTodos()` retorna únicamente cuentas que **existen vigentes en sistema**
- [ ] Las cuentas que no existen vigentes no aparecen en el resultado
- [ ] Las cuentas no vigentes se conservan en BD (sin DELETE físico) para trazabilidad histórica
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs`
- GAP documentado: GAP-01 del archivo `TPSC-RE-FU-001-Back.md`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 3

### [ TPSC-RE-FU-001 ] [ IMP-EXIST-SERVICE ] Agregar método ObtenerVigentesPorEmpresa() en vEmpresaDatosBancariosBO

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — Logic.Pqf.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el método `ObtenerVigentesPorEmpresa(string prefijo)` en `vEmpresaDatosBancariosBO.Extensions.cs` para que los módulos consulten cuentas que **existen vigentes en sistema** filtradas por empresa emisora del grupo, cumpliendo el Criterio A2 del requisito (GAP-02).

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-001 (GAP-02)
- Implementar el método `ObtenerVigentesPorEmpresa(string prefijo)`
- Validar que el prefijo esté dentro del set permitido: `GOL`, `MUN`, `PRO`, `PQF`
- Aplicar filtro de existencia vigente (`Activo = true`) en `Empresa` y `EmpresaDatosBancarios`
- Retornar lista vacía si el prefijo no es válido o la empresa no existe en sistema
- Ejecutar pruebas unitarias con cada prefijo válido e inválido

**Resultado esperado:**
El método retorna las cuentas que existen vigentes en sistema de la empresa cuyo prefijo coincide con el solicitado. Prefijos fuera del alcance R16 (ej. `GOLPERU`) retornan lista vacía.

**Entregables:**
- `vEmpresaDatosBancariosBO.Extensions.cs` con el nuevo método
- Pruebas unitarias para los 4 prefijos válidos y casos inválidos

**Criterios de aceptación:**
- [ ] `ObtenerVigentesPorEmpresa("GOL")` retorna cuentas que existen vigentes de Golocaer
- [ ] `ObtenerVigentesPorEmpresa("GOLPERU")` retorna lista vacía
- [ ] Solo se retornan cuentas que existen vigentes en sistema (`Activo = true`)
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `Logic.Pqf.Catalogos\Empresas\DatosBancarios\EmpresaDatosBancariosBO.Extensions.cs`
- GAP documentado: GAP-02 del archivo `TPSC-RE-FU-001-Back.md`
- Prefijos válidos R16: `GOL`, `MUN`, `PRO`, `PQF`. Excluir `GOLPERU`.

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---

## Tarea 4

### [ TPSC-RE-FU-001 ] [ LIST-NO-FILTER ] Agregar endpoint GET /vEmpresaDatosBancarios/Vigentes/{prefijo}

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — WebApi.Catalogos

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo (si aplica).

**Objetivo general:**
Agregar el endpoint `GET /vEmpresaDatosBancarios/Vigentes/{prefijo}` en `EmpresaDatosBancariosController.cs` para exponer las cuentas que existen vigentes en sistema, filtradas por empresa emisora, a los módulos consumidores. Cumple el Criterio A2 del requisito (GAP-03).

**Objetivos específicos:**
- Leer el análisis de impacto backend del requisito TPSC-RE-FU-001 (GAP-03)
- Agregar el método `ObtenerVigentesPorEmpresa(string prefijo)` en el controller
- Consumir el método del mismo nombre implementado en la Tarea 2 (`EmpresaDatosBancariosBO`)
- Documentar el endpoint en Swagger con descripción de prefijos válidos R16
- Verificar que el endpoint no requiera UI (solo consumo interno entre módulos)

**Resultado esperado:**
`GET /vEmpresaDatosBancarios/Vigentes/{prefijo}` responde con la lista de cuentas que existen vigentes en sistema de la empresa solicitada. La respuesta excluye automáticamente cuentas no vigentes y empresas fuera del alcance R16.

**Entregables:**
- `vEmpresaDatosBancariosController.cs` con el nuevo endpoint documentado en Swagger
- Prueba manual del endpoint en ambiente de desarrollo

**Criterios de aceptación:**
- [ ] `GET /vEmpresaDatosBancarios/Vigentes/GOL` retorna cuentas que existen vigentes de Golocaer
- [ ] `GET /vEmpresaDatosBancarios/Vigentes/GOLPERU` retorna lista vacía
- [ ] El endpoint está documentado en Swagger con descripción de prefijos válidos
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Archivo a modificar: `WebApi.Catalogos\Controllers\Configuracion\Empresas\vEmpresaDatosBancariosController.cs`
- GAP documentado: GAP-03 del archivo `TPSC-RE-FU-001-Back.md`
- Depende de Tarea 2 (método `ObtenerVigentesPorEmpresa` en BO)

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`

---
## Tarea 5

### [ TPSC-RE-FU-001 ] [ MIG-DATOS ] Carga inicial de cuentas bancarias del grupo PROQUIFA

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Cuentas Bancarias — Base de Datos ProquifaDotNet

**Consideraciones previas:**
Para esta actividad están contempladas la recepción de datos por parte de PROQUIFA, construcción del script de carga, validación, aprobación del líder técnico mediante PR y liberación en dev.

> ⚠️ **Bloqueante:** Esta tarea está bloqueada hasta recibir el detalle completo de cuentas por empresa del grupo (banco, número de cuenta, CLABE, moneda, códigos). Pendiente entrega por PROQUIFA — ver **Pendiente P4** en `TPSC-RE-FU-001-Back.md`.

**Objetivo general:**
Cargar los datos iniciales de las cuentas bancarias que existen vigentes en sistema de las cuatro empresas del grupo PROQUIFA México (`GOL`, `MUN`, `PRO`, `PQF`) en las tablas `EmpresaDatosBancarios` y `DatosBancarios` de la BD `ProquifaDotNet`, como requisito previo al funcionamiento del catálogo en cualquier módulo consumidor.

**Objetivos específicos:**
- Recibir y validar el archivo de datos de cuentas bancarias entregado por PROQUIFA
- Verificar que cada cuenta tenga: empresa emisora, banco, número de cuenta, CLABE, moneda
- Construir script `INSERT` para `DatosBancarios` y `EmpresaDatosBancarios` con `Activo = 1` (existen vigentes en sistema)
- Validar que los `IdEmpresa` referenciados corresponden a `GOL`, `MUN`, `PRO`, `PQF` en la tabla `Empresa`
- Validar que los `IdCatBanco` e `IdCatMoneda` existan en los catálogos correspondientes
- Ejecutar script en ambiente de desarrollo y verificar resultados con la vista `vEmpresaDatosBancariosVigentes`
- Registrar el script en el formulario de control de scripts del release

**Resultado esperado:**
Las cuentas bancarias del grupo existen vigentes en sistema con `Activo = 1`. Los módulos consumidores pueden consultar el catálogo correctamente a través de la vista `vEmpresaDatosBancariosVigentes`.

**Entregables:**
- Script de carga inicial `INSERT` para `DatosBancarios` y `EmpresaDatosBancarios`
- Script registrado en el formulario de control de scripts del release
- Evidencia de validación en ambiente de desarrollo

**Criterios de aceptación:**
- [ ] Las cuatro empresas del grupo (`GOL`, `MUN`, `PRO`, `PQF`) tienen al menos una cuenta que **existe vigente en sistema** (`Activo = 1`)
- [ ] Todos los registros tienen `IdCatBanco` e `IdCatMoneda` válidos
- [ ] La vista `vEmpresaDatosBancariosVigentes` retorna los registros cargados correctamente
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- Pendiente P4 del archivo `TPSC-RE-FU-001-Back.md`: entrega del detalle completo de cuentas por PROQUIFA
- Tablas destino: `DatosBancarios` (detalle bancario) y `EmpresaDatosBancarios` (vínculo con empresa)
- Depende de la Tarea 5 (vista `vEmpresaDatosBancariosVigentes`) para la validación final

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`


---

## Tarea 6

### [ TPSC-RE-FU-001 ] [ DOC-GUIDE ] Guía de resolución — Alta, Baja y Actualización de cuentas bancarias del grupo PROQUIFA

**Aplicativos:**
ProquifaNet 2 — Soporte a la Produccion

**Modulos:**
Documentacion Operativa — Catalogo de Cuentas Bancarias del Grupo PROQUIFA

**Consideraciones previas:**
No existe pantalla en ProquifaNet 2 que permita al usuario final gestionar el catalogo de cuentas bancarias del grupo. Toda operacion de alta, baja o actualizacion debe ser ejecutada directamente en base de datos por el equipo de **Soporte a la Produccion**, siguiendo esta guia.

Esta tarea consiste en **redactar y publicar la guia operativa** en el repositorio de documentacion del equipo.

**Objetivo general:**
Crear el documento de guia de resolucion que permita a Soporte a la Produccion realizar de forma segura y trazable las operaciones de alta, baja logica y actualizacion de cuentas bancarias del grupo PROQUIFA directamente en BD, sin afectar registros historicos ni la integridad referencial.

---

### Precondiciones

Antes de ejecutar cualquier operacion, Soporte a la Produccion debe contar con:

| Dato requerido | Como obtenerlo |
|---|---|
| `IdEmpresa` de la empresa objetivo | `SELECT IdEmpresa, Prefijo, Alias FROM dbo.Empresa WHERE Prefijo IN ('GOL','MUN','PRO','PQF') AND Activo = 1` |
| `IdCatBanco` del banco | `SELECT IdCatBanco, Banco FROM dbo.catBanco WHERE Activo = 1` |
| `IdCatMoneda` de la moneda | `SELECT IdCatMoneda, ClaveMoneda FROM dbo.catMoneda WHERE Activo = 1` |
| Numero de cuenta y CLABE | Proporcionados por PROQUIFA (Pendiente P4 del Back) |

---

### Paso 0 — Verificar estado actual

Antes de realizar cualquier operacion, ejecutar la consulta de cuentas vigentes para conocer el estado actual:

```sql
SELECT
    edb.IdEmpresaDatosBancarios,
    e.Prefijo,
    e.Alias                AS Empresa,
    b.Banco,
    db.NumeroDeCuenta,
    db.Clabe,
    m.ClaveMoneda          AS Moneda,
    edb.FechaRegistro,
    edb.Activo
FROM dbo.EmpresaDatosBancarios edb
INNER JOIN dbo.Empresa         e   ON edb.IdEmpresa        = e.IdEmpresa
INNER JOIN dbo.DatosBancarios  db  ON edb.IdDatosBancarios = db.IdDatosBancarios
LEFT  JOIN dbo.catBanco        b   ON db.IdCatBanco        = b.IdCatBanco
LEFT  JOIN dbo.catMoneda       m   ON db.IdCatMoneda       = m.IdCatMoneda
WHERE e.Prefijo IN ('GOL','MUN','PRO','PQF')
ORDER BY e.Prefijo, edb.Activo DESC, db.NumeroDeCuenta;
```

---

### Alta de cuenta bancaria

Se requieren dos inserciones en secuencia: primero en `DatosBancarios` (detalle bancario), luego en `EmpresaDatosBancarios` (vinculo con la empresa).

```sql
BEGIN TRANSACTION;

-- 1. Insertar el detalle bancario
DECLARE @IdDatosBancarios UNIQUEIDENTIFIER = NEWID();
DECLARE @IdCatBanco       UNIQUEIDENTIFIER = '<reemplazar con IdCatBanco>';
DECLARE @IdCatMoneda      UNIQUEIDENTIFIER = '<reemplazar con IdCatMoneda>';

INSERT INTO dbo.DatosBancarios
    (IdDatosBancarios, IdCatBanco, NumeroDeCuenta, Clabe, IdCatMoneda, Activo)
VALUES
    (@IdDatosBancarios, @IdCatBanco, '<NumeroDeCuenta>', '<Clabe>', @IdCatMoneda, 1);

-- 2. Vincular con la empresa
DECLARE @IdEmpresaDatosBancarios UNIQUEIDENTIFIER = NEWID();
DECLARE @IdEmpresa               UNIQUEIDENTIFIER = '<reemplazar con IdEmpresa>';

INSERT INTO dbo.EmpresaDatosBancarios
    (IdEmpresaDatosBancarios, IdEmpresa, IdDatosBancarios, Activo, FechaRegistro)
VALUES
    (@IdEmpresaDatosBancarios, @IdEmpresa, @IdDatosBancarios, 1, GETDATE());

-- 3. Verificar el alta
SELECT
    e.Prefijo, b.Banco, db.NumeroDeCuenta, db.Clabe, m.ClaveMoneda AS Moneda, edb.Activo
FROM dbo.EmpresaDatosBancarios edb
INNER JOIN dbo.Empresa        e   ON edb.IdEmpresa        = e.IdEmpresa
INNER JOIN dbo.DatosBancarios db  ON edb.IdDatosBancarios = db.IdDatosBancarios
LEFT  JOIN dbo.catBanco       b   ON db.IdCatBanco        = b.IdCatBanco
LEFT  JOIN dbo.catMoneda      m   ON db.IdCatMoneda       = m.IdCatMoneda
WHERE edb.IdEmpresaDatosBancarios = @IdEmpresaDatosBancarios;

-- Si el resultado es correcto, confirmar. Si no, revertir.
-- COMMIT;
-- ROLLBACK;

COMMIT;
```

> ⚠️ **Nunca** ejecutar `INSERT` sin envolver en transaccion. Verificar el resultado antes de confirmar.

---

### Baja logica de cuenta bancaria

No se realiza `DELETE` fisico. La baja consiste en marcar `Activo = 0` en `EmpresaDatosBancarios`. El registro se conserva para trazabilidad historica.

```sql
BEGIN TRANSACTION;

DECLARE @IdCuenta UNIQUEIDENTIFIER = '<reemplazar con IdEmpresaDatosBancarios>';

-- 1. Verificar que la cuenta existe y esta vigente
SELECT edb.IdEmpresaDatosBancarios, e.Prefijo, db.NumeroDeCuenta, edb.Activo
FROM dbo.EmpresaDatosBancarios edb
INNER JOIN dbo.Empresa        e   ON edb.IdEmpresa        = e.IdEmpresa
INNER JOIN dbo.DatosBancarios db  ON edb.IdDatosBancarios = db.IdDatosBancarios
WHERE edb.IdEmpresaDatosBancarios = @IdCuenta;

-- 2. Aplicar baja logica
UPDATE dbo.EmpresaDatosBancarios
SET    Activo                   = 0,
       FechaUltimaActualizacion = GETDATE()
WHERE  IdEmpresaDatosBancarios  = @IdCuenta;

-- 3. Verificar resultado
SELECT edb.IdEmpresaDatosBancarios, e.Prefijo, db.NumeroDeCuenta, edb.Activo
FROM dbo.EmpresaDatosBancarios edb
INNER JOIN dbo.Empresa        e   ON edb.IdEmpresa        = e.IdEmpresa
INNER JOIN dbo.DatosBancarios db  ON edb.IdDatosBancarios = db.IdDatosBancarios
WHERE edb.IdEmpresaDatosBancarios = @IdCuenta;

-- Si Activo = 0, confirmar. Si no, revertir.
-- COMMIT;
-- ROLLBACK;

COMMIT;
```

> ⚠️ Si la cuenta esta referenciada en ordenes de pago activas (`fcppOrdenDePago.Activo = 1`), coordinar con el area correspondiente antes de dar de baja.

---

### Actualizacion de datos de cuenta bancaria

Permite corregir o actualizar: banco, numero de cuenta, CLABE o moneda.

```sql
BEGIN TRANSACTION;

DECLARE @IdEmpresaDatosBancarios UNIQUEIDENTIFIER = '<reemplazar con IdEmpresaDatosBancarios>';

-- 1. Obtener el IdDatosBancarios vinculado
DECLARE @IdDatosBancarios UNIQUEIDENTIFIER;
SELECT @IdDatosBancarios = IdDatosBancarios
FROM dbo.EmpresaDatosBancarios
WHERE IdEmpresaDatosBancarios = @IdEmpresaDatosBancarios;

-- 2. Verificar estado actual antes de modificar
SELECT db.NumeroDeCuenta, db.Clabe, b.Banco, m.ClaveMoneda, edb.Activo
FROM dbo.DatosBancarios db
LEFT  JOIN dbo.catBanco  b   ON db.IdCatBanco  = b.IdCatBanco
LEFT  JOIN dbo.catMoneda m   ON db.IdCatMoneda = m.IdCatMoneda
INNER JOIN dbo.EmpresaDatosBancarios edb ON edb.IdDatosBancarios = db.IdDatosBancarios
WHERE db.IdDatosBancarios = @IdDatosBancarios;

-- 3. Aplicar actualizacion (solo los campos que cambian; omitir los que no cambian)
UPDATE dbo.DatosBancarios
SET    NumeroDeCuenta = '<NuevoNumero>',
       Clabe          = '<NuevaClabe>',
       IdCatBanco     = '<NuevoIdBanco>',
       IdCatMoneda    = '<NuevoIdMoneda>'
WHERE  IdDatosBancarios = @IdDatosBancarios;

-- 4. Verificar resultado
SELECT db.NumeroDeCuenta, db.Clabe, b.Banco, m.ClaveMoneda
FROM dbo.DatosBancarios db
LEFT JOIN dbo.catBanco  b ON db.IdCatBanco  = b.IdCatBanco
LEFT JOIN dbo.catMoneda m ON db.IdCatMoneda = m.IdCatMoneda
WHERE db.IdDatosBancarios = @IdDatosBancarios;

-- Si el resultado es correcto, confirmar. Si no, revertir.
-- COMMIT;
-- ROLLBACK;

COMMIT;
```

---

### Paso final — Validar en vista de vigentes

Despues de cualquier operacion, confirmar que el estado del catalogo es correcto consultando la vista:

```sql
SELECT * FROM dbo.vEmpresaDatosBancariosVigentes
ORDER BY Prefijo, NumeroDeCuenta;
```

> La vista `vEmpresaDatosBancariosVigentes` es creada por la Tarea 5 de este mismo requisito.

---

### Reglas operativas

| Regla | Descripcion |
|---|---|
| Sin DELETE fisico | Nunca ejecutar `DELETE` sobre `EmpresaDatosBancarios` ni `DatosBancarios` |
| Transaccion obligatoria | Toda operacion debe ir dentro de `BEGIN TRANSACTION … COMMIT/ROLLBACK` |
| Verificacion previa | Siempre consultar el estado actual antes de modificar |
| Verificacion posterior | Siempre confirmar el resultado antes de hacer `COMMIT` |
| Solo empresas del grupo | Operar unicamente sobre prefijos `GOL`, `MUN`, `PRO`, `PQF` |
| Sin empresas fuera de alcance | No operar sobre `GOLPERU` ni otras empresas fuera del grupo Mexico en R16 |

---

**Criterios de aceptacion de la tarea:**
- [ ] Documento de guia publicado en el repositorio de documentacion de Soporte a la Produccion
- [ ] Scripts SQL de alta, baja y actualizacion validados en ambiente de desarrollo antes de su publicacion
- [ ] Guia revisada y aprobada por el lider tecnico
- [ ] El documento referencia la vista `vEmpresaDatosBancariosVigentes` (Tarea 5) para validacion post-operacion

**Mas información de la tarea:**
- Pendiente P1 del archivo `TPSC-RE-FU-001-Back.md`: restriccion formal o documentacion de uso interno de PUT/DELETE
- Criterio C2 del requisito: gestion de cuentas por Soporte a la Produccion directamente en BD, sin UI en R16
- No hay UI en ProquifaNet 2 para esta operacion. Esta guia es el mecanismo de control operativo.

**Recursos:**
- Analisis de impacto backend: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001.md`
- Revision funcional: `Requisitos/TPSC-RE-FU-001/TPSC-RE-FU-001-Revision.md`