# R16A-RE-FU-009 — Impacto de Backend

| Campo | Valor |
|-------|-------|
| **Requisito** | R16A-RE-FU-009 |
| **Nombre** | Validación Regulatoria en Pretramitar Pedido |
| **Repositorio** | ProquifaDotNet (.NET Framework 4.8) |
| **Versión** | 1.0 |
| **Fecha** | 2026-06-02 |

---

## Resumen Ejecutivo

El requisito introduce una **validación de documentos regulatorios** en el módulo **Tramitar Pedido (L05)** cuando un pedido contiene sustancias controladas (Mundial, Nacional u Origen) y el cliente **no tiene registrados** los documentos regulatorios requeridos en su catálogo (`ArchivoCliente`).

> **OBS-023:** La validación se mueve de `L04.PretramitarPedido` (VerificarPedidoTramitableBO) a **`L05.TramitarPedido` (TramitarPedidoBO)**. La lógica en `VerificarPedidoTramitableBO` ya NO llama a `ValidarDocumentosRegulatoriosBO`.

> **⚠️ OBSOLETO (2026-08-14):** los puntos 1 y 5 (columnas `AceptaEntregasParciales`/`IdPedidoOrigenControlado` y la bifurcación por entregas parciales / pedido hijo) fueron **retirados**. El cliente confirmó que los pedidos mixtos (controladas + no controladas) no ocurren en la operación real; el requisito vigente (`R16A-RE-FU-009.md`, Regla 4 / Criterio B3) define que, ante documentación regulatoria faltante, el sistema **retiene siempre el pedido completo** en su folio original, sin bifurcación ni pedido hijo. Ver `R16A-RE-FU-009_DIS-SOL_Revision.md` H-02/H-03/H-04. El resto de este documento (puntos 2–4) sigue vigente.

El impacto en BackEnd incluye:
1. ~~Scripts BD: ALTER `ppPedido` (AceptaEntregasParciales) + ALTER `tpPedido` (IdPedidoOrigenControlado).~~ **Retirado.**
2. Scripts BD para catálogos y función.
3. Nueva clase `ValidarDocumentosRegulatoriosBO` en `L05.TramitarPedido\Validaciones\`.
4. DTO `ResultadoValidacionRegulatoria` (`EsValido`, `Mensaje`) — ya no requiere soporte de resultado parcial (partidas tramitables vs. retenidas), pues el pedido se retiene o se tramita como unidad completa.
5. ~~Bifurcación en `TramitarPedidoBO`: si `AceptaEntregasParciales=1`, tramitar elegibles + crear pedido hijo para controladas retenidas.~~ **Retirado.** `TramitarPedidoBO` solo necesita: si la validación falla, bloquear el pedido completo (excepción con mensaje genérico); si pasa, continuar el flujo existente.

---

## PARTE 1 — Impacto en Base de Datos (ProquifaDotNet — RYNL010)

### 1.1 Cambios requeridos

|  #  | Objeto                   | Tipo de cambio                                    | Prioridad | Dependencia                 |
| :-: | ------------------------ | ------------------------------------------------- | :-------: | --------------------------- |
|  1  | `catUsoArchivoSistema`   | INSERT — 'Licencia Sanitaria' (MEX)               |   Alta    | Prerequisito para RE-FU-003 |
|  2  | `catUsoArchivoSistema`   | INSERT — 'Aviso de Responsable Sanitario' (MEX)   |   Alta    | Prerequisito para RE-FU-003 |
|  3  | ~~`catUsoArchivoSistema`~~ | ~~INSERT — [docs DIGEMID PER]~~ **Descartado (DUDA-027)** |   —    | No aplica — Perú no soporta sustancias controladas en R16 (riesgo operativo asumido, sin bloqueo por sistema) |
|  4  | `fnEsProductoControlado` | ALTER FUNCTION — agregar clave `'origen'`         |   Alta    | Compartido con RE-FU-007    |

### 1.2 Tablas involucradas (sin cambios estructurales)

| Tabla                  | Rol en la validación                                           |
| ---------------------- | -------------------------------------------------------------- |
| `ppPedido`             | Cabecera — `IdRegion`, `IdContactoCliente`                     |
| `ppPartidaPedido`      | Partidas — `IdProducto` para detectar controlados              |
| `MarcaFamilia`         | Vincula Producto → Familia                                     |
| `Familia`              | `IdCatControl` determina si es controlado                      |
| `catControl`           | Clave IN (`mundiales`, `nacionales`, `origen`)                 |
| `ArchivoCliente`       | Documentos regulatorios del cliente — `IdCatUsoArchivoSistema` |
| `catUsoArchivoSistema` | Tipos de documento a validar                                   |
| `ContactoCliente`      | Vínculo `ppPedido` → `Cliente`                                 |
| `Cliente`              | Dueño de los documentos regulatorios                           |
| `Region`               | Determina qué documentos son requeridos (MEX vs PER)           |

### 1.3 Hallazgos críticos

| # | Hallazgo | Impacto |
|:-:|----------|---------|
| 1 | `ArchivoCliente` sin registros activos | La funcionalidad de carga de documentos (R16A-RE-FU-003) es **prerequisito** — sin ella, la validación siempre bloquearía |
| 2 | `catUsoArchivoSistema` sin registros | Deben insertarse los tipos de documento antes del desarrollo |
| 3 | `fnEsProductoControlado` incompleta | Solo detecta `mundiales` y `nacionales` — falta `origen` |
| 4 | `ProductoBO.EsControlado()` usa `ProductoMarcaFamilia.Controlado` | Verificar que este campo refleje la clave `origen` tras la actualización de la función |

---

## PARTE 2 — Impacto en Lógica de Negocio

### 2.1 Flujo actual de tramitación (sin cambios)

```
PretramitarPedidoTramitarController
  POST /PretramitarPedido/transaccion
    → PretramitarPedidoTransaccionBO.PretramitarPedidoTransaccion()
        → ... (validaciones existentes, configuración, partidas)
        → TramitarPedidoBO.Process(ppPedido)
            → VerificarPedidoTramitableBO.Procesar(ppPedido)  ← validaciones actuales
            → Separa partidas controladas / no controladas
            → Genera tpPedido(s)
```

### 2.2 Punto de inserción de la validación regulatoria (OBS-023)

> **Cambio OBS-023:** La validación se ejecuta ahora en **`TramitarPedidoBO.Process()`** (L05), NO en `VerificarPedidoTramitableBO.Procesar()` (L04).
> `VerificarPedidoTramitableBO` ya **no** llama a `ValidarDocumentosRegulatoriosBO`.

Dentro de `TramitarPedidoBO.Process()`:
1. Llamar a `ValidarDocumentosRegulatoriosBO.Validar(idPPPedido)`.
2. Si resultado = válido (no tiene controladas, o las tiene y los documentos están registrados) → tramitar normalmente el pedido completo (flujo existente).
3. Si resultado = inválido (tiene controladas y falta al menos un documento) → **bloquear el pedido completo** (excepción con mensaje genérico), sin excepción por composición del pedido ni entrega parcial.

> ~~Si resultado = controladas sin docs: Si `ppPedido.AceptaEntregasParciales = 1` → tramitar solo partidas no controladas + crear pedido hijo...~~ **Retirado (2026-08-14)** — ver nota de obsolescencia al inicio del documento.

### 2.3 Componentes nuevos (OBS-023)

| # | Archivo | Proyecto | Tipo | Descripción |
|:-:|---------|----------|------|-------------|
| 1 | `L05.TramitarPedido\Validaciones\ValidarDocumentosRegulatoriosBO.cs` | `Logic.Pqf.Logistica` | **NUEVO** | Lógica de validación regulatoria — retorna válido/inválido para el pedido completo (sin detalle de partidas tramitables/retenidas, ver obsolescencia) |
| 2 | `L05.TramitarPedido\Validaciones\Models\ResultadoValidacionRegulatoria.cs` | `Logic.Pqf.Logistica` | **NUEVO** | DTO: `EsValido`, `Mensaje` — ~~`PartidasTramitables`, `PartidasRetenidasPorControladas`~~ **retirado (2026-08-14)** |
| 3 | ~~`L05.TramitarPedido\Liberar\CrearPedidoHijoControladoBO.cs`~~ | ~~`Logic.Pqf.Logistica`~~ | ❌ **Retirado (2026-08-14)** | No se crean pedidos hijo — el pedido se retiene completo en su folio original |

### 2.4 Componentes existentes a modificar (OBS-023)

| # | Archivo | Proyecto | Cambio |
|:-:|---------|----------|--------|
| 1 | `L05.TramitarPedido\Liberar\TramitarPedidoBO.cs` | `Logic.Pqf.Logistica` | Llamar a `ValidarDocumentosRegulatoriosBO`; si inválido, bloquear el pedido completo (excepción con mensaje genérico) — ~~+ lógica de entrega parcial / pedido hijo~~ retirado |
| 2 | `L04.PretramitarPedido\Tramite\VerificarPedidoTramitableBO.cs` | `Logic.Pqf.Logistica` | **QUITAR** la llamada a `ValidarDocumentosRegulatoriosBO` (si se había agregado) — ahora está en L05 |
| 3 | `Productos\ProductoBO.TipoExtensions.cs` | `Logic.Pqf.Catalogos` | **Evaluar** — Verificar que `ProductoMarcaFamilia.Controlado` cubre `origen`. Si no, ajustar lógica o usar query directa a catControl. |

### 2.5 Lógica de `ValidarDocumentosRegulatoriosBO`

```csharp
// Pseudocódigo
public class ValidarDocumentosRegulatoriosBO
{
    public ResultadoValidacionRegulatoria Validar(Guid idPPPedido)
    {
        // 1. Verificar si el pedido tiene al menos 1 producto controlado
        //    (mundiales, nacionales, origen)
        bool tieneControlados = ExistenPartidasControladas(idPPPedido);
        if (!tieneControlados)
            return ResultadoValidacionRegulatoria.Valido(); // No requiere validación

        // 2. Obtener IdCliente del pedido
        //    ppPedido → ContactoCliente → Cliente
        Guid idCliente = ObtenerClienteDelPedido(idPPPedido);

        // 3. Obtener Región del pedido
        string claveRegion = ObtenerRegionDelPedido(idPPPedido);

        // 4. Determinar documentos requeridos según región
        List<string> docsRequeridos = ObtenerDocumentosRequeridosPorRegion(claveRegion);

        // 5. Verificar presencia en ArchivoCliente
        List<string> docsFaltantes = VerificarDocumentosRegistrados(idCliente, docsRequeridos);

        // 6. Si falta alguno → bloquear
        if (docsFaltantes.Any())
            return ResultadoValidacionRegulatoria.Invalido(
                "No es posible procesar el pedido porque el cliente no cuenta con la documentación regulatoria requerida para productos controlados. Revise la sección de documentos regulatorios en la configuración del cliente.");

        return ResultadoValidacionRegulatoria.Valido();
    }
}
```

### 2.6 Cadena de detección de producto controlado (consulta)

```sql
-- Partidas controladas del pedido
SELECT 1
FROM ppPartidaPedido pp
INNER JOIN MarcaFamilia mf ON pp.IdProducto = mf.IdProducto
INNER JOIN Familia f ON mf.IdFamilia = f.IdFamilia
INNER JOIN catControl cc ON f.IdCatControl = cc.IdCatControl
WHERE pp.IdPPPedido = @IdPPPedido
  AND pp.Activo = 1
  AND cc.Clave IN ('mundiales', 'nacionales', 'origen')
```

### 2.7 Cadena de validación de documentos

```sql
-- Verificar documentos regulatorios del cliente
SELECT ua.UsoArchivoSistema AS DocumentoRequerido,
       CASE WHEN ac.IdArchivoCliente IS NOT NULL THEN 1 ELSE 0 END AS Registrado
FROM catUsoArchivoSistema ua
LEFT JOIN ArchivoCliente ac ON ac.IdCatUsoArchivoSistema = ua.IdCatUsoArchivoSistema
                           AND ac.IdCliente = @IdCliente
                           AND ac.Activo = 1
WHERE ua.Activo = 1
  AND ua.UsoArchivoSistema IN (@Doc1, @Doc2)  -- Según región
```

---

## PARTE 3 — Impacto en API (WebApi.Logistica)

### 3.1 Endpoints afectados

| Endpoint | Controlador | Cambio |
|----------|-------------|--------|
| `POST /PretramitarPedido/transaccion` | `PretramitarPedidoTramitarController` | La validación se ejecuta internamente via `VerificarPedidoTramitableBO`; el controller ya maneja `Response.Status = false` con `BadRequest`. **No requiere cambios en el controller.** |
| `POST /ValidarAjusteOC/transaccion` | `PretramitarPedidoTramitarController` | Mismo flujo — usa `PretramitarPedidoTransaccionBO` que invoca `TramitarPedidoBO` → `VerificarPedidoTramitableBO`. **Automáticamente cubierto.** |

### 3.2 Puntos de entrada alternos a Tramitar (Riesgo 5 del requisito) — Resuelto vía DUDA-024

| Punto de entrada | Controlador probable | Nota |
|------------------|---------------------|------|
| Gestionar Intramitable → OC corregida → Validar Ajustes → Tramitar | `POST /ValidarAjusteOC/transaccion` | ✅ **Cubierto** — usa el mismo `PretramitarPedidoTransaccionBO` |
| "Tramitar con errores" desde Gestionar Intramitable | Por identificar | ✅ **Cubierto** — ver resolución DUDA-024 |
| Aceptación de OC Interna por el cliente | Por identificar | ✅ **Cubierto** — ver resolución DUDA-024 |

> **DUDA-024 (Resuelta, 2026-08-14 aprox.):** se cerró moviendo la validación regulatoria a Tramitar Pedido (último paso), no a Pretramitar. Como todos los caminos hacia Tramitar Pedido — Pretramitar, validar ajustes, aceptar orden de compra, pedido intramitable ("tramitar con errores") — pasan finalmente por Tramitar Pedido, validar ahí cubre los tres puntos de entrada con una sola validación, sobre un pedido ya limpio (inconsistencias de precio, cantidad, flete ya resueltas). Ya no queda pendiente confirmar en código si estos flujos hacen bypass: por diseño, todos convergen en el paso donde se ejecuta la validación.
>
> Nota de consistencia interna: esta sección todavía describe el punto de inserción como si dependiera de `VerificarPedidoTramitableBO.Procesar()` (L04). Según OBS-023 (§2.2 de este mismo documento) y el requisito vigente, el punto de inserción real es `TramitarPedidoBO.Process()` (L05) — el diagrama de flujo de PARTE 8 debe actualizarse para no citar `VerificarPedidoTramitableBO` como el punto único de validación.

---

## PARTE 4 — Impacto en ETL / Transferencia a Legacy

### 4.1 Análisis de impacto en ETLs

La validación regulatoria **no impacta directamente las ETLs** porque:

- **No modifica la estructura** de `ppPedido`, `ppPartidaPedido` ni `tpPedido`.
- **No agrega campos** a las tablas que se transfieren al sistema legado.
- Es una **validación de bloqueo** — si pasa, el flujo continúa exactamente igual; si no pasa, el pedido nunca llega a tramitación y por tanto nunca se transfiere.

| ETL existente | Impactado | Razón |
|---------------|:---------:|-------|
| Marcas (Datos Generales) | ❌ No | Sin relación |
| Proveedores (Datos Generales) | ❌ No | Sin relación |
| Productos (Datos Generales, Familia, Oferta) | ❌ No | Solo se lee `catControl` — sin cambios en transferencia |
| Clientes (Datos Generales, Direcciones, Contactos, Datos Legales) | ❌ No | `ArchivoCliente` no se transfiere a Legacy |
| Buzones (Cotización, Pedidos, Pagos) | ❌ No | Sin relación |
| Cotizaciones (Cotización, Partidas, Fletes, PDF) | ❌ No | Sin relación |
| Pedidos Sin Crédito (PSC → Legacy) | ❌ No | La validación actúa **antes** de generar el `tpPedido`; si pasa, el flujo PSC continúa sin cambios |
| Pedidos Con Crédito (→ Legacy) | ❌ No | Mismo razonamiento — la validación es pre-tramitación |

### 4.2 Consideración sobre flujo bloqueado

Si un pedido con controlados se bloquea por falta de documentación:
- **No se genera `tpPedido`** → no hay transferencia a Legacy.
- **No se generan pendientes en Legacy** (Pedidos, Partidas, Cobro).
- El pedido queda en estado "Pretramitado" hasta que el ESAC resuelva la documentación.

Esto es **comportamiento esperado** — no es un error ni un cambio en ETLs.

---

## PARTE 5 — Dependencias entre requisitos

| Requisito | Relación | Impacto en Backend |
|-----------|----------|-------------------|
| **R16A-RE-FU-003** | **Prerequisito** | Carga de documentos regulatorios en `ArchivoCliente`. Sin esto, la validación siempre bloquea porque no hay documentos registrados. |
| **R16A-RE-FU-007** | **Compartido** | Misma actualización de `fnEsProductoControlado` para agregar `origen`. El script de BD debe ejecutarse una sola vez. |
| **R16A-RE-FU-005** | **Referencia** | La `Region` del cliente determina qué documentos se requieren. |

---

## PARTE 6 — Resumen de tareas BackEnd

| # | Tarea | Clave Catálogo | Proyecto |
|:-:|-------|---------------|---------|
| T1 | Script BD: INSERT `catUsoArchivoSistema` (Licencia Sanitaria + Aviso Responsable Sanitario MEX) | `UPDATE-TABL-CH` | BD ProquifaDotNet |
| T2 | Script BD: ALTER FUNCTION `fnEsProductoControlado` — agregar `'origen'` (compartido con RE-FU-007) | `BD-OBJ-CH` | BD ProquifaDotNet |
| T3 | Crear `ValidarDocumentosRegulatoriosBO.cs` — lógica de validación regulatoria | `ALG-BASIC-LOGIC` | `Logic.Pqf.Logistica` |
| T4 | Crear `ResultadoValidacionRegulatoria.cs` — DTO de resultado | `ALG-BASIC-LOGIC` | `Logic.Pqf.Logistica` |
| T5 | Modificar `VerificarPedidoTramitableBO.cs` — integrar llamada a validación regulatoria | `IMP-EXIST-SERVICE` | `Logic.Pqf.Logistica` |
| T6 | Verificar/ajustar `ProductoBO.EsControlado()` para que cubra `origen` | `ALG-BASIC-LOGIC` | `Logic.Pqf.Catalogos` |
| ~~T7~~ | ~~Script BD: INSERT `catUsoArchivoSistema` (docs DIGEMID PER)~~ **Descartada (DUDA-027)** | — | — |

---

## PARTE 7 — Gaps y decisiones pendientes

| # | Gap | Descripción | Acción requerida | Responsable |
|:-:|-----|-------------|-----------------|-------------|
| 1 | ~~Denominación docs regulatorios Perú~~ **Descartado (DUDA-027)** | Ya no aplica: se confirmó que Perú no soporta sustancias controladas en R16; el riesgo de tramitar/facturar un controlado de Perú se asume como riesgo operativo, sin bloqueo técnico ni documentos regulatorios que insertar en `catUsoArchivoSistema` para esa región | Ninguna — cerrado | — |
| 2 | ArchivoCliente sin registros | La tabla está vacía — RE-FU-003 (carga de docs) debe estar operativo antes de probar esta validación | Coordinar orden de desarrollo con RE-FU-003 | PM |
| 3 | ~~Puntos de entrada alternativos a Tramitar~~ **Resuelto (DUDA-024)** | "Tramitar con errores" y "Aceptación OC Interna" pasan por Tramitar Pedido, punto único donde se ejecuta la validación regulatoria (último paso) | Ninguna — cerrado. Ver §3.2 | — |
| 4 | `ProductoMarcaFamilia.Controlado` vs `catControl` | Verificar si el campo `Controlado` de `ProductoMarcaFamilia` se actualiza cuando se agrega `origen` a la función | Revisar trigger/proceso que actualiza este campo | Dev BackEnd |
| 5 | fnEsProductoControlado compartida con RE-FU-007 | El ALTER se ejecuta una sola vez — coordinar para no duplicar scripts | Script idempotente | Dev BackEnd |

---

## PARTE 8 — Diagrama de flujo con validación regulatoria

```
ESAC → POST /PretramitarPedido/transaccion
         │
         ▼
  PretramitarPedidoTransaccionBO.PretramitarPedidoTransaccion()
         │
         ▼
  (... validaciones existentes, configuración, partidas ...)
         │
         ▼
  TramitarPedidoBO.Process(ppPedido)
         │
         ▼
  VerificarPedidoTramitableBO.Procesar(ppPedido)
         │
         ├── Validaciones existentes (datos facturación, configuración, partidas)
         │
         ├── ★ NUEVO: ValidarDocumentosRegulatoriosBO.Validar(idPPPedido) ★
         │         │
         │         ├── ¿Tiene partidas controladas? → NO → Continúa sin validar
         │         │
         │         ├── SÍ → Obtener Cliente + Región
         │         │
         │         ├── Verificar ArchivoCliente tiene docs requeridos según región
         │         │
         │         ├── ¿Todos registrados? → SÍ → Continúa
         │         │
         │         └── NO → throw ArgumentException("mensaje genérico regulatorio")
         │
         ▼
  (Continúa: separa controlados/no controlados → genera tpPedido)
         │
         ▼
  ETL → Legacy (Pedidos, Partidas, Cobro, Factura, etc.)
```

---

## PARTE 9 — Seguridad y consideraciones

| Aspecto | Nota |
|---------|------|
| Autorización | No se agregan nuevos endpoints — la validación opera dentro del flujo existente protegido por autenticación JWT |
| Datos sensibles | No se expone información de documentos al cliente (mensaje genérico) |
| Performance | La validación agrega 2 queries adicionales (partidas controladas + documentos). Impacto mínimo — ambas queries usan índices existentes |
| Idempotencia | La validación es de solo lectura — no modifica datos. Puede re-ejecutarse sin efecto secundario |
