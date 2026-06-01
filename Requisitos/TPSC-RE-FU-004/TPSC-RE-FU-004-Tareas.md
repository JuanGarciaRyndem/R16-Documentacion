# Tareas — TPSC-RE-FU-004 Catálogo de Clientes — Información Fiscal

| Campo | Valor |
|---|---|
| **Requisito** | TPSC-RE-FU-004 |
| **Nombre** | Mantenimiento de Catálogo de Clientes — Información Fiscal |
| **Total de tareas** | 6 |
| **Revisión aplicada** | TPSC-RE-FU-004_Revision.md |

---

## Tarea 1

### TPSC-RE-FU-004   [ QUERY-CH ] Corrección de catálogo `catRegimenFiscal` México — desactivar códigos derogados y no estándar SAT CFDI 4.0 - GAP-02a

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogo de Clientes — Información Fiscal

**Consideraciones previas:**
Esta tarea está **bloqueada por el Pendiente P1**: confirmar con el cliente los códigos definitivos a desactivar o corregir. No ejecutar scripts sin confirmación.

**Descripción del problema:**
El catálogo `catRegimenFiscal` para México contiene registros inconsistentes respecto al catálogo SAT CFDI 4.0 vigente:
- Código **609** (Consolidación): derogado por el SAT desde 2014. Activo en BD.
- Códigos **628, 629, 630**: no existen en el catálogo SAT CFDI 4.0 oficial.

**Cambios requeridos (tras confirmación P1):**

```sql
BEGIN TRANSACTION;

-- Verificar si algún cliente activo usa estos códigos antes de desactivar
SELECT dfc.IdCliente, c.Nombre, rf.RegimenFiscal
FROM dbo.DatosFacturacionCliente dfc
INNER JOIN dbo.Cliente           c  ON dfc.IdCliente        = c.IdCliente
INNER JOIN dbo.catRegimenFiscal  rf ON dfc.IdCatRegimenFiscal = rf.IdCatRegimenFiscal
WHERE rf.RegimenFiscal IN ('609','628','629','630')
  AND dfc.Activo = 1
  AND c.Activo   = 1;

-- Solo si no hay clientes activos usando estos códigos (o previa coordinación):
UPDATE dbo.catRegimenFiscal SET Activo = 0
WHERE RegimenFiscal IN ('609','628','629','630');

-- Verificar resultado
SELECT RegimenFiscal, Descripcion, Activo
FROM dbo.catRegimenFiscal
WHERE RegimenFiscal IN ('609','628','629','630');

-- COMMIT;
-- ROLLBACK;
COMMIT;
```

**Criterios de aceptación:**
- [ ] Pendiente P1 resuelto y confirmado antes de ejecutar
- [ ] Verificación previa ejecutada: ningún cliente activo en producción usa los códigos a desactivar (o coordinado el cambio con el área)
- [ ] Códigos confirmados como inválidos quedan con `Activo = 0`
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-02 y Pendiente P1 del archivo `TPSC-RE-FU-004-Back.md`
- Ver archivo adjunto `TPSC-RE-FU-004_Equivalencias_MX_PE.xlsx` con el cruce de catálogos

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004.md`

---

## Tarea 2

### TPSC-RE-FU-004  [ QUERY-M ] Carga inicial de catálogos fiscales para Región Perú - GAP-02b 

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogo de Clientes — Información Fiscal

**Consideraciones previas:**
Esta tarea está **bloqueada por los Pendientes P2, P3 y P4**: confirmar con el cliente los catálogos definitivos de Régimen Fiscal PE (SUNAT), Tipo de Sociedad PE y la denominación oficial de S.A.C.S. No ejecutar scripts sin confirmación. Depende además de la resolución de la Tarea 3 (si aplica Opción A con campo `IdRegion`, los INSERT deben incluirlo).

**Descripción del problema:**
Los catálogos `catRegimenFiscal` y `catTipoSociedadMercantil` no tienen registros para la Región Perú. Sin estos datos, los selectores quedan vacíos para clientes peruanos y la funcionalidad es inoperante para esa región.

**Cambios requeridos (tras confirmación P2, P3, P4 y resolución de Tarea 3):**

```sql
BEGIN TRANSACTION;

-- Obtener IdRegion de Perú
DECLARE @IdRegionPeru UNIQUEIDENTIFIER;
SELECT @IdRegionPeru = IdRegion FROM dbo.Region WHERE ClaveISO = 'PER';

-- Régimen Fiscal Perú (valores orientativos — confirmar con cliente vía P2)
INSERT INTO dbo.catRegimenFiscal
    (IdCatRegimenFiscal, RegimenFiscal, Descripcion, Activo /*, IdRegion si aplica Tarea 3 Opción A */)
VALUES
    (NEWID(), 'RG',   'Régimen General',                    1),
    (NEWID(), 'RMT',  'Régimen MYPE Tributario',            1),
    (NEWID(), 'RER',  'Régimen Especial de Renta',          1),
    (NEWID(), 'NRUS', 'Nuevo Régimen Único Simplificado',   1),
    (NEWID(), 'RPN',  'Régimen para Personas Naturales',    1); -- Confirmar P2

-- Tipo de Sociedad Mercantil Perú (valores orientativos — confirmar con cliente vía P3, P4)
INSERT INTO dbo.catTipoSociedadMercantil
    (IdCatTipoSociedadMercantil, TipoSociedadMerdantil, Abreviatura, Activo)
VALUES
    (NEWID(), 'Sociedad Anónima',                             'S.A.',     1),
    (NEWID(), 'Sociedad Anónima Cerrada',                     'S.A.C.',   1),
    (NEWID(), 'Sociedad Anónima Abierta',                     'S.A.A.',   1),
    (NEWID(), 'Sociedad por Acciones Cerrada Simplificada',   'S.A.C.S.', 1), -- Confirmar P4
    (NEWID(), 'Sociedad Comercial de Responsabilidad Limitada','S.R.L.',  1),
    (NEWID(), 'Empresa Individual de Responsabilidad Limitada','E.I.R.L.',1);

-- Verificar
SELECT RegimenFiscal, Descripcion, Activo
FROM dbo.catRegimenFiscal
ORDER BY RegimenFiscal;

SELECT TipoSociedadMerdantil, Abreviatura, Activo
FROM dbo.catTipoSociedadMercantil
ORDER BY TipoSociedadMerdantil;

-- COMMIT;
-- ROLLBACK;
COMMIT;
```

> ⚠️ Los valores del script son orientativos. Los definitivos deben confirmarse vía P2, P3, P4.
> ⚠️ Si la Tarea 3 adopta Opción A (campo `IdRegion`), los INSERT deben incluir `@IdRegionPeru`.

**Criterios de aceptación:**
- [ ] Pendientes P2, P3 y P4 resueltos antes de ejecutar
- [ ] Registros de Régimen Fiscal Perú insertados con `Activo = 1`
- [ ] Registros de Tipo de Sociedad Perú insertados con `Activo = 1`
- [ ] Denominación S.A.C.S. confirmada (P4) antes de insertar
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-02 y Pendientes P2, P3, P4 del archivo `TPSC-RE-FU-004-Back.md`
- Bloqueante para que la Tarea 5 (endpoints) funcione correctamente para Perú
- Ver `TPSC-RE-FU-004_Equivalencias_MX_PE.xlsx` para el detalle del cruce

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004.md`

---

## Tarea 3

### TPSC-RE-FU-004  [ UPDATE-TABL-CH ] Decisión de arquitectura y migración: campo `IdRegion` en catálogos fiscales - GAP-03 

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogo de Clientes — Información Fiscal

**Descripción del problema:**
Los catálogos `catRegimenFiscal` y `catTipoSociedadMercantil` no tienen campo `IdRegion`. Sin esta distinción, no es posible que el backend filtre automáticamente las opciones por la región del usuario logueado. Requiere una decisión de arquitectura que desbloquea la Tarea 5.

**Opciones de solución:**

**Opción A — Agregar campo `IdRegion` a los catálogos (recomendada):**

```sql
BEGIN TRANSACTION;

ALTER TABLE dbo.catRegimenFiscal
    ADD IdRegion UNIQUEIDENTIFIER NULL
    REFERENCES dbo.Region(IdRegion);

ALTER TABLE dbo.catTipoSociedadMercantil
    ADD IdRegion UNIQUEIDENTIFIER NULL
    REFERENCES dbo.Region(IdRegion);

-- Asignar IdRegion México a todos los registros actuales
DECLARE @IdRegionMex UNIQUEIDENTIFIER;
SELECT @IdRegionMex = IdRegion FROM dbo.Region WHERE ClaveISO = 'MEX';

UPDATE dbo.catRegimenFiscal         SET IdRegion = @IdRegionMex;
UPDATE dbo.catTipoSociedadMercantil SET IdRegion = @IdRegionMex;

-- Verificar
SELECT TOP 5 IdCatRegimenFiscal, RegimenFiscal, IdRegion FROM dbo.catRegimenFiscal;
SELECT TOP 5 IdCatTipoSociedadMercantil, TipoSociedadMerdantil, IdRegion FROM dbo.catTipoSociedadMercantil;

-- COMMIT;
-- ROLLBACK;
COMMIT;
```

**Opción B — Crear vistas filtradas por Región (sin modificar tablas):**
- `vCatRegimenFiscalPorRegion`: JOIN de `catRegimenFiscal` con tabla de mapeo región-catálogo.
- `vCatTipoSociedadMercantilPorRegion`: ídem.

Actualizar modelos EF y lógica en Tarea 5 según la opción elegida.

**Criterios de aceptación:**
- [ ] Decisión de arquitectura documentada y aprobada (Opción A u Opción B)
- [ ] Script de migración revisado y aprobado por DBA
- [ ] Modelos EF actualizados si aplica Opción A
- [ ] Script incluido en el formulario de control de scripts del release
- [ ] Tarea 5 puede completar su implementación definitiva tras esta tarea
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-03 y Pendiente P8 del archivo `TPSC-RE-FU-004-Back.md`
- Prerequisito directo para la Tarea 5 (endpoints con filtro por región de usuario logueado)

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004_BD.md`

---

## Tarea 4

### TPSC-RE-FU-004  [ UPDATE-TABL-CH ] Evaluar y corregir typo `TipoSociedadMerdantil` en `catTipoSociedadMercantil` - GAP-04

**Aplicativos:**
ProquifaNet 2 — Base de datos ProquifaDotNet

**Módulos:**
Catálogo de Clientes — Información Fiscal

**Descripción del problema:**
La columna `TipoSociedadMerdantil` en `dbo.catTipoSociedadMercantil` tiene un typo (falta la "a" en "Mercantil"). La corrección requiere análisis de impacto completo en modelos EF, vistas, consultas y cualquier componente que referencie la columna por nombre.

**Cambios requeridos:**
1. Ejecutar análisis de impacto: buscar todas las referencias a `TipoSociedadMerdantil` en el codebase.
2. Si el impacto es manejable, ejecutar script de renombrado:

```sql
EXEC sp_rename 'dbo.catTipoSociedadMercantil.TipoSociedadMerdantil',
               'TipoSociedadMercantil', 'COLUMN';
```

3. Actualizar el modelo EF (`ProquifaDotNetContext`) y todas las referencias en código.
4. Si el impacto es alto, documentar como deuda técnica a resolver en release posterior y notificar a las Tareas 2 y 5 para que usen el nombre actual mientras tanto.

**Criterios de aceptación:**
- [ ] Análisis de impacto completo ejecutado y documentado
- [ ] Si se corrige: columna renombrada, modelo EF actualizado, build exitoso sin errores
- [ ] Si se difiere: decisión documentada como deuda técnica con justificación; Tareas 2 y 5 notificadas para usar nombre actual `TipoSociedadMerdantil`
- [ ] Script incluido en el formulario de control de scripts del release (si aplica)
- [ ] PR aprobado por líder técnico y DBA

**Más información de la tarea:**
- GAP-04 y Pendiente P7 del archivo `TPSC-RE-FU-004-Back.md`

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004_BD.md`

---

## Tarea 5

### TPSC-RE-FU-004  GAP-05  [ IMP-EXIST-SERVICE ] Aplicar filtro por región del usuario logueado en `catRegimenFiscalController` y `catTipoSociedadMercantilController` - GAP-05

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Información Fiscal

**Descripción del problema:**
Los controladores `catRegimenFiscalController` y `catTipoSociedadMercantilController` heredan de `ApiController` y exponen CRUD genérico sin filtro por Región. El selector de Régimen Fiscal y Tipo de Sociedad debe retornar únicamente las opciones válidas para la Región del usuario logueado, siguiendo el patrón ya establecido en el proyecto (`BaseApiController` + `ObtenerUsuarioAutenticado()` + `AsegurarFiltroRegion()`).

**Cambios requeridos:**

1. Cambiar la herencia de `ApiController` a `BaseApiController` en ambos controladores.
2. En el método `QueryResult` de cada controlador, obtener el usuario logueado y aplicar el filtro de región antes de ejecutar la consulta:

```csharp
[HttpPost]
[Route("catRegimenFiscal")]
public QueryResult<catRegimenFiscal> QueryResult([FromBody] QueryInfo info)
{
    var user = ObtenerUsuarioAutenticado();
    AsegurarFiltroRegion(info, user.IdRegion);
    var modelBO = new TablaGenericaBO<catRegimenFiscal>();
    return modelBO.QueryResult(info);
}
```

Aplicar el mismo patrón en `catTipoSociedadMercantilController`.

**Dependencias:**
- Requiere que la Tarea 3 haya resuelto cómo el campo `IdRegion` está disponible en los catálogos (Opción A: columna directa; Opción B: via vista) para que `AsegurarFiltroRegion` pueda filtrar correctamente.
- Requiere que la Tarea 1 y Tarea 2 hayan cargado los datos correctos en los catálogos.

**Archivos a modificar:**
- `WebApi.Catalogos\Controllers\Catalogos\catRegimenFiscalController.cs`
- `WebApi.Catalogos\Controllers\Catalogos\catTipoSociedadMercantilController.cs`

**Criterios de aceptación:**
- [ ] Ambos controladores heredan de `BaseApiController`
- [ ] El método `QueryResult` de cada controlador invoca `ObtenerUsuarioAutenticado()` y `AsegurarFiltroRegion(info, user.IdRegion)` antes de consultar
- [ ] Un usuario de Región México recibe solo opciones MX en ambos selectores
- [ ] Un usuario de Región Perú recibe solo opciones PE en ambos selectores (una vez cargados los datos por Tarea 2)
- [ ] Los demás métodos (`Obtener`, `GuardarOActualizar`, `Desactivar`) no son modificados
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-05 y Pendiente P8 del archivo `TPSC-RE-FU-004-Back.md`
- Patrón de referencia: `WebApi.Catalogos\Controllers\Catalogos\catRutaEntregaController.cs`
- Criterios D1 y D2 del requisito: selectores poblados por Región del usuario logueado

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004.md`

---

## Tarea 6

### TPSC-RE-FU-004  GAP-01  [ ALG-BASIC-LOGIC ] Implementar validación de formato RUC Perú (Módulo 11) en `DatosFacturacionClienteBO` - GAP-01

**Aplicativos:**
ProquifaNet 2

**Módulos:**
Catálogo de Clientes — Información Fiscal

**Consideraciones previas:**
Esta tarea está **bloqueada por el Pendiente P6**: confirmar con el cliente si la validación RUC Perú se implementa con algoritmo Módulo 11 o queda como captura libre. No iniciar desarrollo hasta que P6 esté resuelto.

**Descripción del problema:**
PQF2 ya tiene validación de formato RFC para México. No existe validación equivalente para el RUC de Perú. La Revisión indica que el formato de validación debe ser **paramétrico desde BD** (asociado a cada Región), para no hardcodear la expresión regular. Esto implica resolver previamente la estructura de configuración de validación por Región (relacionado con Tarea 3).

**Algoritmo Módulo 11 RUC Perú:**
- Pesos sobre los 10 primeros dígitos: `[5, 4, 3, 2, 7, 6, 5, 4, 3, 2]`
- Resto = Suma(dígito × peso) % 11
- Dígito verificador esperado = 11 − Resto (si 10 → 0; si 11 → 1)
- Comparar con el 11º dígito del RUC

**Cambios requeridos:**
1. Crear clase de extensión `DatosFacturacionClienteBO.Validacion.Extensions.cs` con método `ValidarIdentificadorFiscal(string rfc, Guid idRegion)`:
   - Para MEX: reutilizar lógica de validación RFC SAT existente (12 o 13 alfanuméricos).
   - Para PER: validar 11 dígitos numéricos, tipo de contribuyente (primeros 2 dígitos en `{10, 15, 17, 20}`) y dígito verificador por Módulo 11.
2. Invocar el método en `_GuardarOActualizar` de `DatosFacturacionClienteBO` antes de persistir.

**Archivos a modificar / crear:**
- `Logic.Pqf.Catalogos\Clientes\Configuracion\DatosFacturacionClientes\DatosFacturacionClienteBO.Validacion.Extensions.cs` (nuevo)
- `Logic.Pqf.Catalogos\Clientes\Configuracion\DatosFacturacionClientes\DatosFacturacionClienteBO.cs` (invocar validación en `_GuardarOActualizar`)

**Criterios de aceptación:**
- [ ] RUC válido (11 dígitos, tipo contribuyente correcto, dígito verificador Módulo 11 válido) es aceptado
- [ ] RUC con longitud distinta de 11, tipo contribuyente inválido o dígito verificador incorrecto es rechazado con mensaje descriptivo
- [ ] Validación RFC MX existente no es modificada ni afectada
- [ ] El método identifica la región del cliente para aplicar la validación correspondiente
- [ ] Bloqueada por P6 — no iniciar sin confirmación del cliente
- [ ] PR aprobado por líder técnico

**Más información de la tarea:**
- GAP-01 y Pendientes P5, P6 del archivo `TPSC-RE-FU-004-Back.md`
- Reglas R2 y R3 del requisito
- Revisión: debounce en campo de la UI es responsabilidad del frontend

**Recursos:**
- Análisis de impacto backend: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004-Back.md`
- Diccionario de datos: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004_BD.md`
- Requisito funcional: `Requisitos/TPSC-RE-FU-004/TPSC-RE-FU-004.md`