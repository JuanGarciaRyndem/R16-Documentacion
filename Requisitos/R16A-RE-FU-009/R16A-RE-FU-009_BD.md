# Impacto en BD - Validacion Regulatoria en Pretramitar Pedido
**Requisito:** R16A-RE-FU-009
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
Validacion de documentos regulatorios al **Tramitar** el pedido (L05 — OBS-023: movida desde L04).
Cuando el pedido tiene sustancias controladas y el cliente **no tiene** documentos regulatorios registrados, el sistema **retiene el pedido completo** en su folio original — sin bifurcación.

> **⚠️ OBSOLETO (2026-08-14) — retirada la bifurcación por entregas parciales.** El cliente confirmó que los pedidos mixtos (controladas + no controladas) no ocurren en la operación real; se retiró la mecánica de `AceptaEntregasParciales` / pedido hijo descrita originalmente aquí (ver detalle de columnas más abajo, marcadas igualmente como obsoletas). Ver `R16A-RE-FU-009.md` Regla 4 y Criterio B3 (bloqueo total, folio original) y `R16A-RE-FU-009_DIS-SOL_Revision.md` H-02.
>
> **OBS-023 (vigente):** La validación regulatoria se mueve de `L04.PretramitarPedido` (VerificarPedidoTramitableBO) a `L05.TramitarPedido` (TramitarPedidoBO).

---

## Cadena de Deteccion - Producto Controlado

    ppPedido (pedido pretramitado, IdRegion)
        -> ppPartidaPedido (IdProducto)
            -> MarcaFamilia (IdFamilia)
                -> Familia (IdCatControl)
                    -> catControl (Clave IN 'mundiales','nacionales','origen')

    O usar funcion existente:
    dbo.fnEsProductoControlado(@IdMarcaFamilia) = 1
    ** REQUIERE ACTUALIZACION: agregar 'origen' (ver R16A-RE-FU-007) **

## Cadena de Validacion - Documentos Regulatorios

    ppPedido -> ppPedidoConfiguracion -> IdEmpresa
    ppPedido -> IdContactoCliente -> ContactoCliente -> IdCliente
    Cliente -> ArchivoCliente (IdCatUsoArchivoSistema = docs regulatorios)

---

## Cambios Estructurales (OBS-023) — ⚠️ OBSOLETO, ver nota arriba

| Tabla | Campo | Tipo | Estado | Descripción |
|-------|-------|------|--------|-------------|
| `ppPedido` | `AceptaEntregasParciales` | `bit NOT NULL DEFAULT(0)` | ❌ **Retirado (2026-08-14)** | Ya no aplica — no hay bifurcación por entregas parciales; el pedido se retiene completo |
| `tpPedido` | `IdPedidoOrigenControlado` | `uniqueidentifier NULL` | ❌ **Retirado (2026-08-14)** | Ya no aplica — no se generan pedidos hijo para controladas retenidas |

**Scripts (NO ejecutar — mecánica retirada, se conservan solo como referencia histórica):**

```sql
-- OBSOLETO (2026-08-14): bifurcación por entregas parciales retirada.
-- Ver R16A-RE-FU-009.md Regla 4 / Criterio B3 (bloqueo total, folio original).
-- ALTER TABLE dbo.ppPedido
--     ADD AceptaEntregasParciales bit NOT NULL
--         CONSTRAINT DF_ppPedido_AceptaEntregasParciales DEFAULT (0);
--
-- ALTER TABLE dbo.tpPedido
--     ADD IdPedidoOrigenControlado uniqueidentifier NULL
--         CONSTRAINT FK_tpPedido_PedidoOrigenControlado
--             FOREIGN KEY REFERENCES dbo.tpPedido(IdTpPedido);
```

---

## Entidades Afectadas

| Objeto | Tipo | Estado | Rol en Validacion |
|--------|------|--------|-------------------|
| ppPedido | Tabla | Existente — sin ALTER | Cabecera del pedido - IdRegion, Tramitado |
| ppPartidaPedido | Tabla | Existente | Partidas con IdProducto - detectar si es controlado |
| MarcaFamilia | Tabla | Existente | Vincula Producto -> Familia |
| Familia | Tabla | Existente | IdCatControl determina si es controlado |
| catControl | Catalogo | Existente | Clave IN mundiales/nacionales/origen |
| fnEsProductoControlado | Funcion | Existente - ACTUALIZAR | Agregar 'origen' |
| ArchivoCliente | Tabla | Existente - SIN REGISTROS | Almacena docs regulatorios del cliente |
| catUsoArchivoSistema | Catalogo | Existente - SIN REGISTROS | Tipos de archivo (Licencia, Aviso, etc.) |
| Cliente | Tabla | Existente | RestringirVentaSustanciasControladas (bit) |

---

## Hallazgos Criticos de BD

| #   | Hallazgo                             | Impacto                                                                         |
| --- | ------------------------------------ | ------------------------------------------------------------------------------- |
| 1   | ArchivoCliente sin registros activos | La funcionalidad de carga de docs (R16A-RE-FU-003) es prerequisito              |
| 2   | catUsoArchivoSistema sin registros   | Deben insertarse los tipos 'Licencia Sanitaria' y 'Aviso Responsable Sanitario' |
| 3   | fnEsProductoControlado incompleta    | Solo detecta mundiales/nacionales - falta 'origen'                              |
| 4   | ppPedido.IdRegion existe             | Permite diferenciar reglas MEX vs PER                                           |
| 5   | ppPartidaPedido.IdProducto existe    | Permite rastrear familia->control del producto                                  |

---

## Tablas Clave

### ppPedido (cabecera — sin ALTER; ver nota de obsolescencia arriba)

| Columna | Tipo | Uso en Validacion |
|---------|------|-------------------|
| IdPPPedido | uniqueidentifier | PK del pedido |
| IdRegion | uniqueidentifier | Determina docs requeridos (MEX vs PER) |
| Tramitado | bit | 0=En pretramitacion / 1=Tramitado |
| IdContactoCliente | uniqueidentifier | -> ContactoCliente -> IdCliente |
| IdCatEstadoPretramitacionPedido | uniqueidentifier | Estado del pedido |
| ~~AceptaEntregasParciales~~ | ~~bit NOT NULL DEFAULT(0)~~ | ❌ **Retirado (2026-08-14)** — no aplica, ver nota de obsolescencia |

### tpPedido (sin ALTER; ver nota de obsolescencia arriba)

| Columna | Tipo | Uso |
|---------|------|-----|
| ~~IdPedidoOrigenControlado~~ | ~~uniqueidentifier NULL~~ | ❌ **Retirado (2026-08-14)** — no se generan pedidos hijo |

### ppPartidaPedido (partidas - sin cambios estructurales)

| Columna | Tipo | Uso en Validacion |
|---------|------|-------------------|
| IdPPPartidaPedido | uniqueidentifier | PK de la partida |
| IdPPPedido | uniqueidentifier | FK al pedido |
| IdProducto | uniqueidentifier | -> MarcaFamilia -> Familia -> catControl |
| Activo | bit | Solo partidas activas |

### ArchivoCliente (docs regulatorios - sin cambios estructurales)

| Columna | Tipo | Uso en Validacion |
|---------|------|-------------------|
| IdArchivoCliente | uniqueidentifier | PK |
| IdCliente | uniqueidentifier | FK al cliente del pedido |
| IdArchivo | uniqueidentifier | FK al archivo fisico |
| IdCatUsoArchivoSistema | uniqueidentifier | TIPO de documento: Licencia Sanitaria / Aviso RS |
| Activo | bit | 1=Documento registrado |

### catUsoArchivoSistema (tipos de documento - REQUIERE INSERTS)

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| IdCatUsoArchivoSistema | uniqueidentifier | PK |
| UsoArchivoSistema | varchar(50) | Nombre del tipo de documento |
| Activo | bit | Default: 1 |

**Registros a insertar:**

| UsoArchivoSistema | Region | Descripcion |
|-------------------|--------|-------------|
| Licencia Sanitaria | MEX | Documento COFEPRIS requerido para controlados |
| Aviso de Responsable Sanitario | MEX | Documento COFEPRIS requerido para controlados |
| ~~[Pendiente confirmar]~~ | ~~PER~~ | **Descartado (DUDA-027)** — Perú no soporta sustancias controladas en R16; riesgo operativo asumido, sin documentos regulatorios que registrar |
| ~~[Pendiente confirmar]~~ | ~~PER~~ | **Descartado (DUDA-027)** — ídem |

---

## Logica de Validacion (pseudocodigo)

    FUNCION ValidarDocumentosRegulatorios(@IdPPPedido)
    BEGIN
        -- 1. Verificar si el pedido tiene al menos 1 producto controlado
        SI NO EXISTS (
            SELECT 1 FROM ppPartidaPedido pp
            INNER JOIN MarcaFamilia mf ON pp.IdProducto = mf.IdProducto
            INNER JOIN Familia f ON mf.IdFamilia = f.IdFamilia
            INNER JOIN catControl cc ON f.IdCatControl = cc.IdCatControl
            WHERE pp.IdPPPedido = @IdPPPedido
              AND pp.Activo = 1
              AND cc.Clave IN ('mundiales','nacionales','origen')
        )
        ENTONCES RETORNAR TRUE  -- No requiere validacion

        -- 2. Obtener IdCliente del pedido
        @IdCliente = (SELECT c.IdCliente FROM ppPedido p
                      JOIN ContactoCliente cc ON p.IdContactoCliente = cc.IdContactoCliente
                      JOIN Cliente c ON cc.IdCliente = c.IdCliente
                      WHERE p.IdPPPedido = @IdPPPedido)

        -- 3. Obtener Region del pedido
        @ClaveRegion = (SELECT r.ClaveISO FROM ppPedido p
                        JOIN Region r ON p.IdRegion = r.IdRegion
                        WHERE p.IdPPPedido = @IdPPPedido)

        -- 4. Verificar documentos segun region
        SI @ClaveRegion = 'MEX' ENTONCES
            VERIFICAR que existan en ArchivoCliente:
              - IdCatUsoArchivoSistema = 'Licencia Sanitaria'
              - IdCatUsoArchivoSistema = 'Aviso de Responsable Sanitario'
        -- DESCARTADO (DUDA-027): Perú no soporta sustancias controladas en R16;
        -- no se construye rama de validación para PER. La detección de controlados
        -- y su tramitación/facturación en Perú se asumen como riesgo operativo.

        -- 5. Si falta alguno -> BLOQUEAR
        SI falta algun documento ENTONCES
            RETORNAR FALSE + mensaje generico
        SINO
            RETORNAR TRUE
    END

---

## Consulta SQL - Verificar si pedido tiene controlados

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    DECLARE @IdPPPedido UNIQUEIDENTIFIER;

    SELECT CASE WHEN EXISTS (
        SELECT 1
        FROM dbo.ppPartidaPedido pp
        INNER JOIN dbo.MarcaFamilia mf ON pp.IdProducto = mf.IdProducto
        INNER JOIN dbo.Familia f        ON mf.IdFamilia = f.IdFamilia
        INNER JOIN dbo.catControl cc    ON f.IdCatControl = cc.IdCatControl
        WHERE pp.IdPPPedido = @IdPPPedido
          AND pp.Activo     = 1
          AND cc.Clave IN ('mundiales','nacionales','origen')
    ) THEN 1 ELSE 0 END AS TieneControlados;

## Consulta SQL - Verificar documentos regulatorios del cliente

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    DECLARE @IdCliente UNIQUEIDENTIFIER;

    SELECT
        ua.UsoArchivoSistema   AS DocumentoRequerido,
        CASE WHEN ac.IdArchivoCliente IS NOT NULL THEN 'Registrado' ELSE 'FALTANTE' END AS Estado
    FROM dbo.catUsoArchivoSistema ua
    LEFT JOIN dbo.ArchivoCliente ac ON ac.IdCatUsoArchivoSistema = ua.IdCatUsoArchivoSistema
                                   AND ac.IdCliente = @IdCliente
                                   AND ac.Activo    = 1
    WHERE ua.Activo = 1
      AND ua.UsoArchivoSistema IN ('Licencia Sanitaria','Aviso de Responsable Sanitario')
    ORDER BY ua.UsoArchivoSistema;

---

## Cambios BD Requeridos

| # | Cambio | Tipo | Prioridad | Dependencia |
|---|--------|------|-----------|-------------|
| 1 | INSERT catUsoArchivoSistema (Licencia Sanitaria) | INSERT | Alta | Prerequisito para carga docs (RE-FU-003) |
| 2 | INSERT catUsoArchivoSistema (Aviso Resp. Sanitario) | INSERT | Alta | Prerequisito para carga docs (RE-FU-003) |
| 3 | ALTER FUNCTION fnEsProductoControlado (agregar 'origen') | ALTER | Alta | Compartido con RE-FU-007 |
| ~~4~~ | ~~INSERT catUsoArchivoSistema (docs DIGEMID Peru)~~ **Descartado (DUDA-027)** | — | — | Perú no soporta controlados en R16 |

---

## Dependencias entre Requisitos

| Requisito      | Relacion     | Descripcion                                     |
| -------------- | ------------ | ----------------------------------------------- |
| R16A-RE-FU-003 | Prerequisito | Carga de docs regulatorios en ArchivoCliente    |
| R16A-RE-FU-007 | Compartido   | Misma fnEsProductoControlado + agregar 'origen' |
| R16A-RE-FU-005 | Referencia   | Region del cliente determina docs requeridos    |

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | catUsoArchivoSistema vacio | Insertar tipos de documento MEX antes de desarrollo |
| 2 | ArchivoCliente sin registros | Prerequisito: RE-FU-003 debe estar operativo |
| 3 | ~~Denominacion docs Peru no definida~~ **Descartado (DUDA-027)** | Cerrado — Perú no soporta sustancias controladas en R16, no se insertan documentos regulatorios para esa región |
| 4 | fnEsProductoControlado sin 'origen' | Ejecutar ALTER FUNCTION |
| 5 | ~~Puntos de entrada alternos a Tramitar~~ **Resuelto (DUDA-024)** | Cerrado — la validación se ejecuta en Tramitar Pedido (último paso), punto por el que convergen todos los caminos (Pretramitar, validar ajustes, aceptar OC, pedido intramitable). No requiere acción adicional |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
