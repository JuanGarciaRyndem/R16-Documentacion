# Impacto en Back — R16A-RE-Cambio-PerfilFiscal

**Cambio:** Perfil Fiscal — Configuración fiscal por Familia de Producto (MX + PE)
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8)
**Módulo:** Logística — migración de cálculos de IVA en cotización, pedidos y trámite
**Impacto:** Actualización de Core.Pqf (NuGet) + migración de `region.Impuesto`/`GravaIVA` → `TasaOCuota`/`TipoFactor` en BOs de `Logic.Pqf.Catalogos` y `Logic.Pqf.Logistica`.

---

## 1. Arquitectura de ProquifaDotNet

ProquifaDotNet está compuesto por proyectos en **.NET Framework 4.8**:

```
ProquifaDotNet/
├── Core.Pqf  (NuGet v2.16.0.99 — NO editable aquí, repo separado)
│   ├── Core.Pqf.ProquifaDotNetContext   → ProquifaDotNetEntities (DbContext EF6)
│   │                                       + entidades/vistas EDMX generadas
│   │                                       (vProducto, vFlete, cotProductoOferta,
│   │                                        ppPartidaPedido, tpPartidaPedido, Region …)
│   └── Core.Pqf.BusinessBasicTools      → clases base: CrudBO<T>, ViewBO<T>,
│                                           ManagementResponse<T>, BusinessObjectBase …
├── Logic.Pqf.Catalogos   → BOs de catálogos (vProductoBO, vFleteBO …)
│   └── referencia Core.Pqf vía NuGet
├── Logic.Pqf.Logistica   → BOs de flujos L01–L05 (cotización, pedidos, trámite …)
│   └── referencia Core.Pqf vía NuGet
├── WebApi.Catalogos      → Controllers REST de catálogos
└── WebApi.Logistica      → Controllers REST de logística
```

**Implicaciones para este cambio:**

- Las entidades `vProducto`, `vFlete`, `cotProductoOferta`, etc. son clases **EDMX-generadas** en `Core.Pqf.dll`. Para que exporten los campos nuevos (`TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat`) **Core.Pqf debe actualizarse** (regenerar EDMX → nueva versión NuGet → `ActualizarNugget_Core.Pqf.bat <version>`).
- Los modelos extendidos como `vFleteObj : vFlete` copian propiedades con `AttributeCopycat<vFlete, vFleteObj>.CopyAttributes(...)` — los campos nuevos de la entidad base **se propagan automáticamente sin cambios** en el modelo extendido.
- **No se crean Controllers ni endpoints nuevos** — todo el impacto es en la capa de BOs de `Logic.Pqf.Catalogos` y `Logic.Pqf.Logistica`.

---

## 2. Prerrequisito — Actualizar Core.Pqf (NuGet)

| Paso | Acción |
|---|---|
| 1 | Ejecutar scripts DDL del `_BD.md` en la BD (CREATE TABLE + ALTER VIEW vProducto, vFlete) |
| 2 | En el repo de **Core.Pqf**: regenerar EDMX (Update Model from Database) |
| 3 | Publicar nueva versión del NuGet `Core.Pqf` |
| 4 | En ProquifaDotNet: `ActualizarNugget_Core.Pqf.bat <nueva-version>` |

Una vez actualizado, las entidades `vProducto` y `vFlete` exponen automáticamente:

| Campo | Tipo C# | Origen en BD |
|---|---|---|
| `TipoFactor` | `string` | `catTipoFactorSat.Clave` vía `PerfilFiscalConfiguracionFamilia → PerfilFiscal` |
| `TasaOCuota` | `decimal?` | `PerfilFiscal.TasaOCuota` — `null` cuando `TipoFactor = "Exento"` |
| `ClaveProdServSat` | `string?` | `PerfilFiscalConfiguracionFamilia.ClaveProdServSat` — solo MX |
| `ClaveUnidadSat` | `string?` | `PerfilFiscalConfiguracionFamilia.ClaveUnidadSat` — solo MX |

---

## 3. Patrón actual vs. patrón nuevo

**Patrón actual — fuente: `Region.Impuesto` + `producto.GravaIVA` (bool):**
```csharp
// DbContext lookup a Region para obtener la tasa
var gravaIVAPorRegionCliente = region.Impuesto > 0;
producto.GravaIVA = gravaIVAPorRegionCliente && producto.GravaIVA;

var valorPorcentajeIVA = producto.GravaIVA ? region.Impuesto : 0;
var precioIVA = precio * valorPorcentajeIVA;
```

**Patrón nuevo — fuente: `vProducto.TasaOCuota` + `vProducto.TipoFactor` (campos en la vista):**
```csharp
// Sin lookup a Region — la tasa ya viaja en la entidad de la vista
var gravaIVA = vProducto.TipoFactor != "Exento";

var valorPorcentajeIVA = gravaIVA ? (vProducto.TasaOCuota ?? 0m) : 0m;
var precioIVA = decimal.Round(precio * valorPorcentajeIVA, 6);
```

Para fletes (fuente: `vFlete.TipoFactor` / `vFlete.TasaOCuota`):
```csharp
var gravaIVAFlete = vFlete.TipoFactor != "Exento";
var valorIVAFlete = gravaIVAFlete ? (vFlete.TasaOCuota ?? 0m) : 0m;
```

> El lookup a la tabla `Region` para obtener `Impuesto` desaparece de todos los flujos de cálculo de IVA.

---

## 4. Entidades de partidas — campos disponibles tras actualizar Core.Pqf

Los campos nuevos quedan disponibles en las entidades de Core.Pqf que se derivan de las vistas actualizadas. Los modelos extendidos (`cotProductoOfertaObj`, `ppPartidaPedidoObj`, etc.) los heredan sin cambios adicionales.

| Entidad (Core.Pqf) | Vista fuente | Campos disponibles |
|---|---|---|
| `cotProductoOferta` | `vProducto` | `TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat` |
| `cotPartidaCotizacion` | `vProducto` | `TipoFactor`, `TasaOCuota` |
| `ppPartidaPedido` | `vProducto` | `TipoFactor`, `TasaOCuota` |
| `tpPartidaPedido` | `vProducto` | `TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat` |
| `pcPartidaPromesaDeCompra` | `vProducto` | `TipoFactor`, `TasaOCuota` |
| `cotCotizacionFleteExpress` | `vFlete` | `TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat` |
| `cotCotizacionFleteUltimaMilla` | `vFlete` | `TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat` |
| `pcPromesaDeCompraFleteUltimaMilla` | `vFlete` | `TipoFactor`, `TasaOCuota` |
| `ppPedidoFleteExpress` | `vFlete` | `TipoFactor`, `TasaOCuota` |
| `ppPedidoFleteUltimaMilla` | `vFlete` | `TipoFactor`, `TasaOCuota` |
| `tpPedidoFleteExpress` | `vFlete` | `TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat` |
| `tpPedidoFleteUltimaMilla` | `vFlete` | `TipoFactor`, `TasaOCuota`, `ClaveProdServSat`, `ClaveUnidadSat` |

> `tpPartidaPedido` y los fletes de tramitar necesitan `ClaveProdServSat` y `ClaveUnidadSat` porque son origen de los conceptos CFDI en `tpProformaPartidaPedidoToCFDIGeneradaConceptoBO.cs`.

**Cambio puntual en `vFleteBO.cs`** — la asignación explícita de `GravaIVA` no desaparece, pero los campos nuevos ya viajan en `x` (la entidad `vFlete`) gracias a `AttributeCopycat`:

```csharp
// vFleteBO.cs — ListaVFleteObj (sin cambios en el copycat; los campos nuevos ya se copian)
var vFleteObj = new vFleteObj(x);
vFleteObj.GravaIVA = x.GravaIVA;   // se mantiene por compatibilidad
// x.TipoFactor, x.TasaOCuota, x.ClaveProdServSat, x.ClaveUnidadSat ya disponibles en el obj
```

---

## 5. BOs a modificar en `Logic.Pqf.Logistica` — sin Controllers nuevos

Cambios en BOs de `Logic.Pqf.Logistica`: sustituir `region.Impuesto` por `TasaOCuota` y `GravaIVA` bool por `TipoFactor != "Exento"`. **No se agrega ningún Controller ni endpoint.**

#### L01 — Cotización

| Archivo | Cambio requerido |
|---|---|
| `L01.Cotizacion/Partidas/Adapters/RecalcularProductoOfertaBO.cs` | `gravaIVAPorRegionCliente = region.Impuesto > 0` → `gravaIVA = cotProductoOferta.TipoFactor != "Exento"` · `valorPorcentajeIVA = cotProductoOferta.TasaOCuota ?? 0` · eliminar lookup a `Region` |
| `L01.Cotizacion/Partidas/Desgloses/cotProductoOfertaBO.cs` | `producto.GravaIVA ? ... * region.Impuesto : 0` → `gravaIVA ? ... * (cotProductoOferta.TasaOCuota ?? 0) : 0` |
| `L01.Cotizacion/Fletes/cotCotizacionFleteExpressBO.Extensions.cs` | `impuesto = regionBo.ObtenerRegionPorDefecto(idRegion)?.Impuesto` → `impuesto = cotFlete.TasaOCuota ?? 0` · `GravaIVA` → `cotFlete.TipoFactor != "Exento"` |
| `L01.Cotizacion/Actualizacion/ActualizarCotCotizacionTransaccionBO.cs` | `ValorIVA = region.Impuesto` → `ValorIVA = cotProductoOferta.TasaOCuota ?? 0` |
| `L01.Cotizacion/CerrarOferta/SolicitarAjustesCerrarOfertaTransaccionBO.cs` | `GravaIVA = proveedorRegion.GravaIVAFleteExpress` → `GravaIVA = cotFlete.TipoFactor != "Exento"` · `Precio * region.Impuesto` → `Precio * (cotFlete.TasaOCuota ?? 0)` |

#### L02 — Ajustar Oferta

| Archivo | Cambio requerido |
|---|---|
| `L02.AjustarOferta/AjustesCotizacion/AutorizarAjusteOferta/AutorizarAjusteOfertaTransaccionBO.cs` | `porcentajeIVASistema = region.Impuesto` → `porcentajeIVASistema = cotProductoOferta.TasaOCuota ?? 0` · `gravaIVAPorRegionCliente = region.Impuesto > 0` → `gravaIVA = cotProductoOferta.TipoFactor != "Exento"` |

#### L03 — Promesa de Compra

| Archivo | Cambio requerido |
|---|---|
| `L03.PromesaDeCompra/Processors/PretramitarPromesaDeCompraTransaccionBO.cs` | `producto.GravaIVA ? ... * region.Impuesto : 0` → `(pcPartida.TipoFactor != "Exento") ? ... * (pcPartida.TasaOCuota ?? 0) : 0` |

#### L04 — Pretramitar Pedido

| Archivo | Cambio requerido |
|---|---|
| `L04.PretramitarPedido/Partidas/ppPartidaPedidoBO.cs` | `producto.GravaIVA ? ... * region.Impuesto : 0` → `(ppPartida.TipoFactor != "Exento") ? ... * (ppPartida.TasaOCuota ?? 0) : 0` |
| `L04.PretramitarPedido/Partidas/ppPartidaPedidoRecalculoBO.cs` | `!gravaIVA ? 0 : ... * region.Impuesto` (líneas 127, 151, 256) → `... * (partida.TasaOCuota ?? 0)` · `gravaIVA` derivado de `TipoFactor != "Exento"` |
| `L04.PretramitarPedido/Fabrica/Recalcular/ppPedidoRecalcularBO.cs` | `region.Impuesto > 0` (L151) → `TipoFactor != "Exento"` · `ValorIVA = region.Impuesto` (L352, L467) → `ValorIVA = vFlete.TasaOCuota ?? 0` · eliminar lookup a `Region` |

#### L05 — Tramitar Pedido

| Archivo | Cambio requerido |
|---|---|
| `L05.TramitarPedido/Facturas/Generadores/Partidas/tpProformaPartidaPedidoToCFDIGeneradaConceptoBO.cs` | `ClaveProductoServicio = familia.ClaveProductoServicioCFDI` → `ClaveProductoServicio = tpProformaPartidaPedido.ClaveProdServSat` · `ClaveUnidad = "H87"` → `ClaveUnidad = tpProformaPartidaPedido.ClaveUnidadSat` · `if (producto.GravaIVA)` → `if (tpProformaPartidaPedido.TipoFactor != "Exento")` · eliminar lookups a `vProductoBO` y `FamiliaBO` para datos fiscales |
| `Utils/CFDIGeneradas/Conceptos/IVA/CFDIGeneradaConceptoToCFDIGeneradaImpuestoIVABO.cs` | `TasaOCuota = 0.16M` (hard-code) → recibir tasa como parámetro · `Importe = 0.16M * source.Importe` → `Importe = tasa * source.Importe` |
| BO generador de concepto **Factura por Adelantada** (L05) | Resolver datos fiscales del concepto de anticipo consultando `PerfilFiscalConfiguracionFamilia WHERE ClaveTipoEntidad = 'factura-anticipo' AND IdFamilia IS NULL AND IdRegion = :idRegion` → obtener `ClaveProdServSat`, `ClaveUnidadSat`, `TipoFactor`, `TasaOCuota` · eliminar valores fijos de clave SAT que pueda tener hard-codeados |

#### Otros proyectos

| Archivo | Cambio requerido |
|---|---|
| `Logic.MailXslt/TramitarPedido/CorreoPedidoInterno.cs` | `region.Impuesto` en cálculo flete → `vFlete.TasaOCuota ?? 0` |
| `Logic.PDF/Pedido/ArchivoBOConfirmacionPedidoExtensionsPdf.cs` | `(region.Impuesto * 100M).ToString("0") + "%"` → `((tpPartida.TasaOCuota ?? 0) * 100M).ToString("0") + "%"` |

---

## 6. Campo `GravaIVA` — qué pasa con él

`GravaIVA` (bool) existe en múltiples entidades de Core.Pqf y en `ProductoDetalle` (modelo local de `Logic.Pqf.Catalogos`). **No se elimina** — Core.Pqf solo agrega los 4 campos nuevos; `GravaIVA` permanece para compatibilidad con otros flujos y validaciones de catálogo.

Lo que cambia es la **fuente de verdad para el cálculo del monto IVA**: deja de ser `region.Impuesto && GravaIVA` y pasa a ser `TipoFactor != "Exento"` + `TasaOCuota`. El campo se seguirá poblando por el EDMX, pero no se usará para derivar el porcentaje.

`ProveedorObj.GravaIVAFleteExpress` se mantiene como flag de configuración de proveedor; el monto de IVA de flete express usará `vFlete.TasaOCuota` en lugar de `region.Impuesto`.

---

## Recursos

- `R16A-RE-Cambio-PerfilFiscal_BD.md` — DDL/DML: `PerfilFiscalConfiguracionFamilia` + catálogos + ALTER TABLE Familia + ALTER VIEW vMarcaFamilia/vProducto/vFlete
- `R16A-RE-Cambio-PerfilFiscal-Back-Finanzas.md` — impacto en ProquifaDotNet.Finanzas: DTOs con campos fiscales + construcción dinámica de CfdiTraslado + Factura por Adelantada (`factura-anticipo`)
- `R16A-RE-FU-019_BD.md` — sección "Datos del producto — ClaveProdServ, ClaveUnidad y PerfilFiscal" (GAP-7/GAP-8 resueltos)
- `R16A-RE-FU-019-Back.md` — sección "Catálogos SAT y PerfilFiscal" (desbloquear ⏸ Pendiente)
- `R16A-RE-Cambio-PerfilFiscal/Perfil-Fiscal-REQ-00.md` — requisito funcional
