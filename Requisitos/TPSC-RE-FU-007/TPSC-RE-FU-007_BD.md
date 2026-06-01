# Diccionario de Datos — TPSC-RE-FU-007 Notificación Regulatoria en Cotización Definitiva

| Campo | Valor |
|-------|-------|
| **Requisito** | TPSC-RE-FU-007 |
| **Base de Datos** | ProquifaDotNet |
| **Servidor** | RYNL010 |
| **Versión** | 1.0 |
| **Exploración BD** | 2026-06-01 |

---

## Resumen Ejecutivo

Agregar automáticamente una leyenda regulatoria al PDF de la cotización definitiva cuando al menos una partida contiene un producto clasificado como Sustancia Controlada tipo Mundial, Nacional u Origen. La leyenda es informativa y no bloqueante. Único cambio funcional de R16 en el módulo Cotizar lo Cotizable.

---

## Hallazgo Crítico — GAP en función `fnEsProductoControlado`

> La función existente `fnEsProductoControlado` solo detecta Mundiales y Nacionales.
> El requisito exige también detectar Origen (clave `origen` en `catControl`).
> La función debe actualizarse **antes** del desarrollo.

**Función actual (detecta Mundiales y Nacionales — incompleta):**

```sql
WHERE Clave IN ('mundiales','nacionales')
```

**Función requerida por TPSC-RE-FU-007 (debe incluir Origen):**

```sql
WHERE Clave IN ('mundiales','nacionales','origen')
```

---

## Modelo de Datos — Cadena de Detección

```
cotCotizacion  (CotizacionDeInvestigacion = 0 = definitiva, IdRegion)
    FK IdCotCotizacion
cotPartidaCotizacion
    FK IdCotProductoOferta
cotProductoOferta
    FK IdProducto
Producto
    (via MarcaFamilia -> Familia)
MarcaFamilia
    FK IdFamilia
Familia
    FK IdCatControl -> catControl  (Clave IN 'mundiales','nacionales','origen' = controlado)

Función de apoyo:
fnEsProductoControlado(@IdMarcaFamilia) -> BIT
    [REQUIERE ACTUALIZACIÓN: agregar clave 'origen']
```

---

## Entidades Afectadas

| Objeto                        | Tipo      | Estado                       | Descripción                                                             |
| ----------------------------- | --------- | ---------------------------- | ----------------------------------------------------------------------- |
| `cotCotizacion`               | Tabla     | Existente — sin cambios      | Campo `CotizacionDeInvestigacion` distingue definitiva vs investigación |
| `cotPartidaCotizacion`        | Tabla     | Existente — sin cambios      | Partidas de la cotización con referencia al producto                    |
| `cotProductoOferta`           | Tabla     | Existente — sin cambios      | Oferta de producto con `IdProducto`                                     |
| `Familia`                     | Tabla     | Existente — sin cambios      | Tiene `IdCatControl` — determina si el producto es controlado           |
| `catControl`                  | Catálogo  | Existente — sin cambios      | Tipos de control: Mundiales, Nacionales, Origen, Normal, N/A            |
| `MarcaFamilia`                | Tabla     | Existente — sin cambios      | Vincula Producto con Familia                                            |
| `catClasificacionRegulatoria` | Catálogo  | Existente — sin cambios      | Clasificación regulatoria del producto                                  |
| *`fnEsProductoControlado`*    | *Función* | ***REQUIERE ACTUALIZACIÓN*** | *Agregar clave `origen` a la detección*                                 |
| `ProductoMarcaFamilia`        | Tabla     | Existente — referencia       | Campo `Controlado` (bit) y `IdCatClasificacionRegulatoria`              |

---

## 1. `cotCotizacion` — Campos Clave

**Propósito:** Cabecera de la cotización. Determina si es definitiva o de investigación.

| Columna | Tipo | Nulo | Descripción |
|---------|------|------|-------------|
| `IdCotCotizacion` | uniqueidentifier | NO | PK |
| `IdCliente` | uniqueidentifier | NO | FK — cliente al que pertenece la cotización |
| `IdRegion` | uniqueidentifier | NO | FK — región del cliente (MEX / PER) |
| `CotizacionDeInvestigacion` | bit | NO | **0 = definitiva** (aplica leyenda) / 1 = investigación (no aplica) |
| `Folio` | varchar | SÍ | Folio de la cotización |
| `FechaCotizacion` | datetime | NO | Fecha de creación |
| `Activo` | bit | NO | Default: 1 |

---

## 2. `catControl` — Catálogo de Tipos de Control

**Propósito:** Clasifica si un producto requiere control sanitario especial.
**Detonante de la leyenda:** `Clave IN ('mundiales','nacionales','origen')`.

| Columna | Tipo | Longitud | Descripción |
|---------|------|----------|-------------|
| `IdCatControl` | uniqueidentifier | 16 | PK |
| `Control` | varchar | 180 | Nombre del tipo de control |
| `Controlado` | bit | 1 | Flag de producto controlado |
| `Clave` | varchar | 150 | Clave interna usada en lógica de detección |
| `Activo` | bit | 1 | Default: 1 |

**Registros actuales en BD:**

| Control | Clave | Controlado | Aplica Leyenda R16 |
|---------|-------|------------|--------------------|
| Mundiales | mundiales | No * | ✅ SÍ |
| Nacionales | nacionales | No * | ✅ SÍ |
| Origen | origen | No * | ✅ SÍ |
| Normal | normal | No * | ❌ NO |
| N/A | n/a | No * | ❌ NO |

> (*) El campo bit `Controlado = 0` para todos los registros en BD.
> La detección de controlados se hace por `Clave`, no por el flag `Controlado`.
> Verificar con el equipo si el flag `Controlado` debe actualizarse.

---

## 3. `Familia` — Campo `IdCatControl`

**Propósito:** Las familias de productos tienen asignado un tipo de control.
**Cadena:** `Producto` → `MarcaFamilia` → `Familia` → `catControl`

| Columna | Tipo | Nulo | Descripción |
|---------|------|------|-------------|
| `IdFamilia` | uniqueidentifier | NO | PK |
| `IdCatControl` | uniqueidentifier | NO | FK — `catControl` (tipo de control de la familia) |
| `IdCatTipoProducto` | uniqueidentifier | NO | FK — tipo de producto |
| `IdCatSubtipoProducto` | uniqueidentifier | NO | FK — subtipo de producto |
| `ClaveProductoServicioCFDI` | varchar | SÍ | Clave SAT del producto/servicio |

---

## 4. `fnEsProductoControlado` — Función Existente (REQUIERE ACTUALIZACIÓN)

**Propósito:** Determina si un producto (por `IdMarcaFamilia`) es sustancia controlada.
**Problema:** Solo detecta Mundiales y Nacionales. Falta Origen.

**Parámetros:**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `@IdMarcaFamilia` | uniqueidentifier | ID de la relación Marca-Familia del producto |

**Retorno:** `BIT` (1 = controlado, 0 = no controlado)

**Lógica actual (incompleta — sin `origen`):**

```sql
SELECT 1 FROM dbo.catControl
WHERE Clave IN ('mundiales','nacionales')
  AND IdCatControl IN (
      SELECT IdCatControl FROM Familia
      WHERE IdFamilia IN (
          SELECT IdFamilia FROM MarcaFamilia
          WHERE IdMarcaFamilia = @IdMarcaFamilia
      )
  )
```

**Actualización requerida para R16:**

```sql
ALTER FUNCTION [dbo].[fnEsProductoControlado]
(
    @IdMarcaFamilia UNIQUEIDENTIFIER
)
RETURNS BIT
AS
BEGIN
    DECLARE @EsControlado BIT = 0;
    IF EXISTS(
        SELECT 1 FROM dbo.catControl
        WHERE Clave IN ('mundiales','nacionales','origen')  -- ORIGEN AGREGADO
          AND IdCatControl IN (
            SELECT IdCatControl FROM dbo.Familia
            WHERE IdFamilia IN (
                SELECT IdFamilia FROM dbo.MarcaFamilia
                WHERE IdMarcaFamilia = @IdMarcaFamilia
            )
          )
    )
    BEGIN
        SET @EsControlado = 1;
    END
    RETURN @EsControlado;
END;
```

---

## 5. `catClasificacionRegulatoria` — Referencia

**Propósito:** Clasificación regulatoria del producto (Narcóticos, Psicotrópicos, Precursores, etc.)

> Este catálogo clasifica el tipo de sustancia controlada. No es el detonante de la leyenda (el detonante es `catControl`), pero es complementario.

| Descripción | Clave | Activo |
|-------------|-------|--------|
| Espectros de referencia | espectrosdereferencia | ✅ Sí |
| N/A | n/a | ✅ Sí |
| Narcóticos | narcoticos | ✅ Sí |
| Productos precursores | productosprecursores | ✅ Sí |
| Sustancias psicotrópicas | sustanciaspsicotropicos | ✅ Sí |

---

## 6. Leyenda Regulatoria por Región

**Propósito:** Texto que se imprime una sola vez en el PDF de la cotización definitiva cuando existe al menos una partida con producto controlado.

| Región | ClaveISO | Texto de la Leyenda | Estado |
|--------|----------|---------------------|--------|
| México | MEX | 'Producto sujeto a regulación sanitaria. Para procesar el pedido se requiere: Licencia Sanitaria vigente y Aviso de Responsable Sanitario.' | ⚠️ PENDIENTE confirmar texto final con cliente/UX |
| Perú | PER | [placeholder DIGEMID] | ⚠️ PENDIENTE: confirmar denominación exacta con cliente |

> El texto de la leyenda no se almacena en BD — es una constante en la capa de aplicación.
> Se recomienda modelarlo como parámetro configurable en BD para facilitar actualizaciones sin deploy.

---

## Consultas SQL de Referencia

### Verificar si una cotización definitiva tiene partidas controladas

```sql
DECLARE @IdCotCotizacion UNIQUEIDENTIFIER;

SELECT
    cot.IdCotCotizacion,
    cot.Folio,
    cot.CotizacionDeInvestigacion,
    r.ClaveISO AS Region,
    CASE
        WHEN cot.CotizacionDeInvestigacion = 1 THEN 0  -- Investigación: NO aplica leyenda
        WHEN EXISTS (
            SELECT 1
            FROM dbo.cotPartidaCotizacion cp
            INNER JOIN dbo.cotProductoOferta cpo ON cp.IdCotProductoOferta = cpo.IdCotProductoOferta
            INNER JOIN dbo.MarcaFamilia mf ON cpo.IdProducto = mf.IdProducto
            INNER JOIN dbo.Familia f ON mf.IdFamilia = f.IdFamilia
            INNER JOIN dbo.catControl cc ON f.IdCatControl = cc.IdCatControl
            WHERE cp.IdCotCotizacion = cot.IdCotCotizacion
              AND cp.Activo = 1
              AND cc.Clave IN ('mundiales','nacionales','origen')
        ) THEN 1
        ELSE 0
    END AS DebeIncluirLeyenda
FROM dbo.cotCotizacion cot
INNER JOIN dbo.Region r ON cot.IdRegion = r.IdRegion
WHERE cot.IdCotCotizacion = @IdCotCotizacion;
```

### Listar partidas controladas de una cotización definitiva

```sql
DECLARE @IdCotCotizacion UNIQUEIDENTIFIER;

SELECT
    cp.IdCotPartidaCotizacion,
    cp.Numero                  AS NumeroPartida,
    cpo.IdProducto,
    cc.Control                 AS TipoControl,
    cc.Clave                   AS ClaveControl,
    cr.Descripcion             AS ClasificacionRegulatoria
FROM dbo.cotPartidaCotizacion cp
INNER JOIN dbo.cotProductoOferta      cpo ON cp.IdCotProductoOferta  = cpo.IdCotProductoOferta
INNER JOIN dbo.MarcaFamilia           mf  ON cpo.IdProducto          = mf.IdProducto
INNER JOIN dbo.Familia                f   ON mf.IdFamilia            = f.IdFamilia
INNER JOIN dbo.catControl             cc  ON f.IdCatControl          = cc.IdCatControl
LEFT  JOIN dbo.ProductoMarcaFamilia   pmf ON mf.IdMarcaFamilia       = pmf.IdMarcaFamilia
LEFT  JOIN dbo.catClasificacionRegulatoria cr
                                           ON pmf.IdCatClasificacionRegulatoria = cr.IdCatClasificacionRegulatoria
WHERE cp.IdCotCotizacion = @IdCotCotizacion
  AND cp.Activo          = 1
  AND cc.Clave IN ('mundiales','nacionales','origen')
ORDER BY cp.Numero;
```

### Cotizaciones definitivas con leyenda regulatoria (últimos 30 días)

```sql
SELECT
    cot.Folio,
    c.Nombre                       AS Cliente,
    r.ClaveISO                     AS Region,
    cot.FechaCotizacion,
    COUNT(DISTINCT cc.Clave)       AS TiposControlEncontrados
FROM dbo.cotCotizacion cot
INNER JOIN dbo.Cliente               c   ON cot.IdCliente           = c.IdCliente
INNER JOIN dbo.Region                r   ON cot.IdRegion            = r.IdRegion
INNER JOIN dbo.cotPartidaCotizacion  cp  ON cot.IdCotCotizacion     = cp.IdCotCotizacion AND cp.Activo = 1
INNER JOIN dbo.cotProductoOferta     cpo ON cp.IdCotProductoOferta  = cpo.IdCotProductoOferta
INNER JOIN dbo.MarcaFamilia          mf  ON cpo.IdProducto          = mf.IdProducto
INNER JOIN dbo.Familia               f   ON mf.IdFamilia            = f.IdFamilia
INNER JOIN dbo.catControl            cc  ON f.IdCatControl          = cc.IdCatControl
WHERE cot.CotizacionDeInvestigacion = 0
  AND cot.Activo                   = 1
  AND cot.FechaCotizacion          >= DATEADD(DAY, -30, GETDATE())
  AND cc.Clave IN ('mundiales','nacionales','origen')
GROUP BY cot.Folio, c.Nombre, r.ClaveISO, cot.FechaCotizacion
ORDER BY cot.FechaCotizacion DESC;
```

---

## Análisis de Gaps

| # | Gap | Descripción | Acción | Prioridad |
|---|-----|-------------|--------|-----------|
| 1 | `fnEsProductoControlado` incompleta | Solo detecta Mundiales y Nacionales — falta Origen | Ejecutar `ALTER FUNCTION` (script Sección 4) | Alta |
| 2 | `catControl.Controlado = 0` en todos | El flag bit `Controlado` está en 0 para todos los registros | Verificar si debe actualizarse o si la lógica usa solo `Clave` | Media |
| 3 | Texto leyenda Perú indefinido | Denominación DIGEMID no confirmada por cliente | Confirmar con cliente antes de desarrollo | Alta |
| 4 | Texto leyenda México sin confirmar | Texto definitivo es decisión UX/Marketing del cliente | Confirmar texto exacto con cliente | Media |
| 5 | Ubicación leyenda en PDF sin definir | Encabezado, sección dedicada o pie de página | Definir en sprint de diseño UI | Media |
| 6 | Leyenda no parametrizable en BD | Texto de leyenda como constante en código — difícil de actualizar | Evaluar tabla `catLeyendaRegulatoriaRegion` | Baja |

---

## Reglas de Negocio

| Regla | Descripción | Implementación en BD |
|-------|-------------|----------------------|
| Regla 1 | Solo cotizaciones definitivas | `cotCotizacion.CotizacionDeInvestigacion = 0` |
| Regla 2 | Detonante: al menos 1 partida controlada | EXISTS partida con catControl.Clave IN ('mundiales','nacionales','origen') |
| Regla 3 | Leyenda una sola vez por documento | Lógica en capa de aplicación — no por partida |
| Regla 4 | Texto según Región del cliente | `cotCotizacion.IdRegion` → `Region.ClaveISO` (MEX / PER) |
| Regla 5 | No bloqueante | PDF se genera siempre — leyenda es aditiva |
| Regla 6 | Sin consulta al catálogo del cliente | No JOIN con `ArchivoCliente` ni documentos regulatorios |

---

## Riesgos

| # | Riesgo | Mitigación |
|---|--------|------------|
| 1 | Leyenda redundante para clientes con documentos cargados | Comportamiento intencional R16 — documentado |
| 2 | Denominación Perú no definida | **No iniciar desarrollo para PE** hasta confirmar con cliente |

---

## Módulos Relacionados

| Módulo | Requisito | Relación |
|--------|-----------|----------|
| Catálogo de Clientes — Docs Regulatorios | TPSC-RE-FU-003 | La leyenda avisa al cliente que debe tener estos docs cargados |
| Pretramitar Pedido | TPSC-RE-FU-009 | Valida bloqueantemente que el cliente tenga los docs cargados |

---

**Versión:** 1.0 — **Base de Datos:** ProquifaDotNet — **Alcance:** México y Perú (texto Perú pendiente)
