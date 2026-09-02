# 📐 Nombramiento — ProquifaDotNet

## Convenciones Base

### Prefijos de Módulo en Tablas

| Prefijo     | Módulo                     | Ejemplo                                 |
| ----------- | -------------------------- | --------------------------------------- |
| *(ninguno)* | Entidades core             | `Cliente`, `Producto`, `Proveedor`      |
| `cat`       | Catálogos                  | `catMoneda`, `catPais`                  |
| `ajOf`      | Ajuste de Oferta           | `ajOfEstrategiaCotizacion`              |
| `cot`       | Cotización                 | `cotCotizacion`, `cotPartidaCotizacion` |
| `emb`       | Embalaje                   | `embPartida`, `embPaquete`              |
| `fcc`       | Facturación Cobro Cliente  | `fccPagoCliente`                        |
| `fcpp`      | Facturación Pago Proveedor | `fcppOrdenDePago`                       |
| `fi`        | Interfacturación           | `fiCFDI`, `fiPago`                      |
| `fpp`       | Flujo Pago Proveedor       | `fppSeguimientoPagoFactura`             |
| `imp`       | Importación                | `impOrdenDespacho`                      |
| `insp`      | Inspección                 | `inspPartida`, `inspPieza`              |
| `oc`        | Orden de Compra            | `ocOrdenDeCompra`, `ocPartida`          |
| `pc`        | Promesa de Compra          | `pcPromesaDeCompra`                     |
| `pp`        | Pedido Procesado           | `ppPedido`, `ppPartidaPedido`           |
| `seg`       | Seguridad / Visitantes     | `segVisitante`                          |
| `tp`        | Tramitación de Pedido      | `tpPedido`, `tpPartidaPedido`           |

### Prefijos por Tipo de Objeto

| Tipo | Prefijo | Ejemplo |
|---|---|---|
| Tabla / Catálogo | `prefijo módulo` + PascalCase | `cotCotizacion`, `catMoneda` |
| Vista | `v` | `vCliente`, `vProducto` |
| Stored Procedure | `sp` | `spActualizaArchivosPedido` |
| Función (con prefijo) | `fn` | `fnEsValidoCliente` |
| Función (TVF sin prefijo) | PascalCase + sufijo `Tabla` | `PrecioProquifaNetClienteTabla` |
| Trigger | `tr` | `trActualizarControlado` |
| ETL (SP especial) | `etl` | `etlClienteContactoProcesarLegacy` |

---

## 📏 Reglas Generales

| Regla | Detalle |
|---|---|
| **Idioma** | Español en todos los objetos (salvo siglas técnicas como `ETL`, `CFDI`, `VD`) |
| **Case** | PascalCase tras el prefijo de módulo |
| **Número** | Singular |
| **Esquema** | `dbo` por defecto |
| **Guiones / Underscores** | No permitidos dentro de los nombres |
| **Prefijo `sp_`** | Prohibido en objetos propios — reservado por SQL Server |
| **Prefijo `usp_`** | No aplica en este proyecto |
| **Abreviaciones** | Usar siglas establecidas del negocio (`OC`, `VD`, `ETL`, `CFDI`) en mayúsculas |

---

## 🔑 Formato de Claves de Catálogo (columna `Clave`)

La columna `Clave` de toda tabla de catálogo (prefijo `cat`) es un valor de **dato**, no un identificador de objeto de BD — pero tiene su propia regla de formato:

| Regla | Detalle |
|---|---|
| **Case** | Minúsculas |
| **Espacios** | No permitidos |
| **Guion medio (`-`)** | No permitido |
| **Guion bajo (`_`)** | No permitido |

**Ejemplo:** `OC_RECIBIDA` → `ocrecibida`

Aplica solo a la columna `Clave` (el valor de negocio que identifica el registro del catálogo). No aplica a otras columnas del mismo catálogo que sí llevan mayúsculas o guiones por diseño (p. ej. `AliasOperativo`, `Aplicativo`).

---

## ❌ Patrones No Permitidos

| Patrón | Ejemplo no permitido | Motivo |
|---|---|---|
| `sp_NombreProcedimiento` | `sp_ActualizarFecha` | Prefijo reservado por SQL Server |
| `usp_NombreProcedimiento` | `usp_DesactivateUserTokens` | No corresponde a la convención del proyecto |
| `Nombre_Con_Underscores` | `Carga_ClientesR1` | Rompe la convención PascalCase |
| `vw_NombreVista` | `vw_cotPartidaCotizacion` | Prefijo incorrecto; usar `v` |
| `Trigger_Accion_Tabla` | `Trigger_AfterInsert_Productos` | No sigue el patrón `tr` + PascalCase |
| Nombres en inglés | `UserRegistrationToken` | Se debe mantener consistencia en español |
| Typos en nombres | `Buscardor` | Errores ortográficos en objetos de BD |

## 📁 Archivos de Scripts SQL

Los archivos de modificación, creación, inserción o actualización deben seguir esta convención de nombre:

```
/Requisito/Seq_NombreObjeto_Accion.sql
```

| Segmento         | Descripción                                 | Ejemplo                                                |
| ---------------- | ------------------------------------------- | ------------------------------------------------------ |
| **Requisito**    | Número del requisito que se está trabajando | `REQ-42`                                               |
| **Seq**          | Número secuencial de 3 dígitos              | `001`, `002`, `015`                                    |
| **NombreObjeto** | Nombre del objeto de base de datos          | `Cliente`, `Producto`, `vCliente`, `fnCalculaProducto` |
| **Accion**       | Acción que realiza el script                | `CREATE`, `ALTER`, `INSERT`, `UPDATE`, `DELETE`        |

**Ejemplos:**

```
✅ /REQ-42/001_Cliente_CREATE.sql
✅ /REQ-42/002_Producto_ALTER.sql
✅ /REQ-42/003_vCliente_CREATE.sql
✅ /REQ-42/004_fnCalculaProducto_CREATE.sql
✅ /REQ-42/005_Pedido_INSERT.sql
✅ /REQ-42/006_Inventario_UPDATE.sql
```

---

## ❌ Patrones No Permitidos

| Patrón                    | Motivo                                                    |
| ------------------------- | --------------------------------------------------------- |
| `sp_NombreProcedimiento`  | Prefijo reservado por SQL Server para objetos del sistema |
| `tbl_NombreTabla`         | Prefijo innecesario; rompe la convención PascalCase       |
| `vw_NombreVista`          | Prefijo incorrecto; usar `v` sin guión bajo               |
| Nombres en español        | Se debe mantener consistencia en inglés                   |
| Nombres en plural         | Todos los objetos deben ser en singular                   |
| Abreviaciones no estándar | Dificultan la legibilidad y mantenimiento                 |
