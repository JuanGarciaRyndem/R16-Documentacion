# Impacto en Back y Base de Datos — Separación PROQUIFA / GOLOCAER

> Documento hermano de `Separacion-PROQUIFA-GOLOCAER.md`. Analiza el impacto **sobre el aplicativo actual ProquifaDotNet** (.NET Framework 4.8) y sobre la base de datos `ProquifaDotNet`, **sin considerar R16**. Sirve como base para dimensionar el cambio previo al proyecto.

**Alcance de fuentes revisadas**

- Base de datos: `C:\Users\juan.garcia\Documents\R16-Documentacion\Database\ProquifaDotNet.sql`
- Solución: `C:\Users\juan.garcia\Documents\ProquifaDotNet-R14`

---

## 1. Modelo actual de Empresa que factura

### 1.1 Tabla `dbo.Empresa` — campos que gobiernan la matriz

| Campo                                                                                                     | Tipo                       | Rol en la matriz                                                                |
| --------------------------------------------------------------------------------------------------------- | -------------------------- | ------------------------------------------------------------------------------- |
| `IdEmpresa`                                                                                               | uniqueidentifier           | PK — referenciada desde toda la operación                                       |
| `Prefijo`                                                                                                 | varchar(50)                | Identificador corto — hardcoded en Back: `"PRO"`, `"GOL"`                       |
| `Alias`                                                                                                   | varchar(50)                | Persistido en `cotCotizacion.QuienFacturaAlias`                                 |
| `RazonSocial`                                                                                             | varchar(50)                | Nombre fiscal                                                                   |
| `RFC`                                                                                                     | varchar(13)                | Emisor CFDI                                                                     |
| `Clave`                                                                                                   | varchar(150)               | Slug hardcoded en Back: `"proquifa"`, `"proveedora"`, `"golocaer"`, `"ganilab"` |
| `FacturaPublicaciones`                                                                                    | bit                        | **"Check"** de la matriz de combinación                                         |
| `FacturaControlados`                                                                                      | bit NULL                   | Habilita la ruta de controlados (Mundial/Nacional/Origen)                       |
| `FacturaServicios`                                                                                        | bit NULL                   | Habilita la ruta de servicios (Ganilab)                                         |
| `PHS`                                                                                                     | bit                        | Perfil especial para pedidos hospitalarios                                      |
| `FrenteComercial`                                                                                         | bit                        | Marca de frente comercial (Mungen / Pharma / otros)                             |
| `FacturacionMatriz`                                                                                       | bit                        | Empresa "cabeza" del grupo                                                      |
| `FacturacionElectronica`                                                                                  | bit                        | Habilita timbrado                                                               |
| `Comprador / Vendedor / Importador / Exportador`                                                          | bit                        | Roles operativos                                                                |
| `IdArchivoCertificadoFacturacion` / `IdArchivoLlavePrivadaFacturacion` / `ClaveLlavePrivadaCertificacion` | uniqueidentifier / varchar | Credenciales de timbrado                                                        |
| `Activo`                                                                                                  | bit                        | Baja lógica                                                                     |

**Empresas actuales por bandera (basado en código y datos):**

| Prefijo | Clave        | Rol                                                |
| ------- | ------------ | -------------------------------------------------- |
| `PRO`   | `proquifa`   | Normales                                           |
| —       | `proveedora` | Controlados (`FacturaControlados = 1`)             |
| `GOL`   | `golocaer`   | Publicaciones (`FacturaPublicaciones = 1`)         |
| —       | `ganilab`    | Servicios (`FacturaServicios = 1`)                 |
| —       | —            | Mungen, Pharma, Golocaer SAC (frentes comerciales) |

### 1.2 Foreign keys hacia `Empresa`

`IdEmpresa` está referenciada como FK desde al menos las siguientes tablas — cualquier estrategia de "quién factura" debe cubrirlas todas al momento del corte:

| Tabla | Columna(s) FK | Uso |
|---|---|---|
| `ClienteDatosSTP` | `IdEmpresa`, `IdEmpresaPublicaciones` | Cuenta STP por empresa emisora |
| `ClienteEmpresa` | `IdEmpresa` | Relación N:N Cliente ↔ Empresa |
| `ConfiguracionPagosDatosBancarios` | `IdEmpresa` | Datos bancarios para pagos |
| `ContratoCliente` | `IdEmpresa` | Contrato firmado por empresa |
| `cotCotizacion` | `IdEmpresa`, `IdEmpresaPublicaciones` | Empresa que factura la cotización y la que factura las publicaciones asociadas |
| `CuentaEmpresa` | `IdEmpresa` | Cuentas contables por empresa |
| `DatosFacturacionCliente` | `IdEmpresa` | Perfil fiscal cliente ↔ empresa |
| `EmpresaDatosBancarios` | `IdEmpresa` | Datos bancarios de la empresa |
| `EmpresaRegion` | `IdEmpresa` | Región donde opera la empresa |
| `EstrategiaInterfacturacion` | `IdEmpresaEmisor`, `IdEmpresaReceptor` | Interfacturación entre empresas del grupo |
| `fccPagoCliente` | `IdEmpresa` | Pago aplicado a empresa |
| `fiCFDI` | `IdEmpresaEmisor`, `IdEmpresaReceptor` | CFDI emitido/recibido |
| `fiPago` | `IdEmpresaBeneficiaria`, `IdEmpresaOrdenante` | Pago con empresas origen/destino |
| `fiPendienteInterfacturacion` | `IdEmpresaCompra`, `IdEmpresaVenta` | Interfacturación pendiente |
| `impOrdenDespacho` | `IdEmpresaExportador`, `IdEmpresaImportador` | Comercio exterior |
| `ocOrdenDeCompra` | `IdEmpresaCompra` (×2), `IdEmpresaEmbarque` | OC hacia proveedor |
| `ocPartida` | `IdEmpresa` | Partida de OC |
| `ppPartidaPedidoConfiguracion` | `IdEmpresa` | Configuración de partida en pedido |
| `ppPedidoConfiguracion` | `IdEmpresa` | Configuración de pedido |
| `ProveedorEmpresa` | `IdEmpresa` | Relación Proveedor ↔ Empresa (¡crítica para USP!) |
| `tpPartidaPedido` | `IdEmpresa` | Partida en pedido tramitado |
| `tpPedido` | `IdEmpresa` | Pedido tramitado |
| `tpProformaAdelanto` | `IdEmpresa` | Proforma de adelanto |
| `tpProformaPartidaPedido` | `IdEmpresaCompra` | Proforma de partida |
| `tpProformaPedido` | `IdEmpresa` | Proforma de pedido |

> Nota: el listado no es exhaustivo — hay más FKs de `IdEmpresa` en la BD; estos son los que impactan directamente el flujo Cotización → Pedido → Facturación → Cobro.

---

## 2. Impacto en Back (ProquifaDotNet, .NET Framework 4.8)

### 2.1 Reglas de combinación y validaciones — módulo Cotización

Archivo: `Logic.Pqf.Logistica/L01.Cotizacion/Actualizacion/ActualizarCotCotizacionTransaccionBO.Extensions.cs`

Lo que hace hoy:

- Define seis mensajes hardcoded como variables locales:
  - `validation_controlled_with_normal` → **msg2** de la matriz.
  - `validation_controlled_with_publications` → **msg3**.
  - `validation_change_to_golocaer` → **msg4**.
  - `validation_only_services` → **msg5/msg6**.
  - `validacion_company_does_not_invoice_publications` → **msg1**.
- Valida contra:
  - `Empresa.FacturaControlados`
  - `Empresa.FacturaPublicaciones`
  - `Empresa.FacturaServicios`
  - `Empresa.Clave == "golocaer"` (hardcoded)
  - `DatosFacturacionCliente.MismaEmpresaFacturaPublicaciones`
- Usa `TipoProductoClave` (`"publicaciones"`, `"servicios"`) y `ControlClave` (`Mundiales`, `Nacionales`, `Origen`, `Normal`) tomados del diccionario `diccionarioCatControl` para determinar la ruta.
- **Cambio automático de empresa al guardar:**
  - Si `productoEsControlado` → `empresa = empresaFacturaControlados` (busca `FacturaControlados == true`).
  - Si `productoEsPublicacion` → `empresa = empresaFacuraPublicaciones` (busca `FacturaPublicaciones == true`).
  - Si `productoEsServicio` → `empresa = empresaFacturaServicios` (busca `FacturaServicios == true`).

**Impacto del cambio propuesto:**

| Zona del código | Cambio requerido |
|---|---|
| Mensajes hardcoded (msg1–msg6) | Extraer a un catálogo (`catMensajeValidacionCotizacion` o resource file) para permitir dos textos distintos por tenant (PROQUIFA / GOLOCAER). |
| `Empresa.Clave == "golocaer"` | Reemplazar por bandera `Empresa.EsEmpresaPublicaciones` o parámetro de configuración — el string literal impide operar con dos empresas de publicaciones. |
| `Prefijo == "PRO"` / `Prefijo == "GOL"` | Eliminar comparación por Prefijo; ver §2.4. |
| Diccionario `diccionarioCatControl` | Verificar que USP no introduzca un nuevo tipo de control. Si sí, agregarlo. Si no, sin cambio. |
| Método `EmpresaFactura(idRegion)` | Aceptar como parámetro el **catálogo de marcas permitidas** por empresa, no sólo la región. |
| Cambio automático de empresa (ruta controlados/publicaciones/servicios) | En PROQUIFA no aplica — es única razón social; el bloque debe deshabilitarse por flag. En GOLOCAER conserva la lógica actual. |
| Divergencia Front / Back en Control N/A | Cerrar aquí (rama `todosControlNormal` incluye `ControlClave == "n/a"`). Se debe validar contra el catálogo `catTipoProducto`, no contra el string `"n/a"`. |

### 2.2 Tramitación de pedidos (PSC / PCC)

Archivo: `Logic.Pqf.Logistica/L04.PretramitarPedido/Tramite/TramitarPedidoBO.cs`

- Al tramitar un pedido, si un producto es publicación y el cliente NO tiene `MismaEmpresaFacturaPublicaciones`, se separa la publicación en un pedido aparte con `IdEmpresa` = `empresaBo.EmpresaFacturaPublicaciones().IdEmpresa`.
- La búsqueda de esa empresa se hace en `Logic.Pqf.Catalogos/Empresas/EmpresaBO.Extensions.cs`:

  ```csharp
  public Empresa EmpresaFacturaPublicaciones()
      => db.Empresa.FirstOrDefault(x => x.Activo && x.FacturaPublicaciones && x.Prefijo == "GOL");

  public Empresa EmpresaFacturaProquifa()
      => db.Empresa.FirstOrDefault(x => x.Activo && x.Prefijo == "PRO");

  public Empresa EmpresaFactura(Guid? idRegion)
  {
      if (region.Clave == Constants.Regiones.Mexico)
          return EmpresaFacturaProquifa();      // ← siempre PROQUIFA en México
      // ... por región Perú, etc.
  }
  ```

**Impacto:**

- El fallback `EmpresaFactura(idRegion)` **retorna siempre PROQUIFA para México** — con el nuevo esquema esto queda **inválido**: en México coexistirán PROQUIFA (USP) y GOLOCAER (resto). Debe recibir además el **conjunto de marcas de la cotización** para elegir empresa.
- Los `Prefijo == "PRO"` / `"GOL"` son literales frágiles. Deben migrarse a banderas o a un catálogo de "roles de facturación" con FKs.
- El split de pedidos por publicación pasa de un caso (publicación → Golocaer) a un caso más general (marca no-USP → Golocaer, marca USP → Proquifa).

### 2.3 Catálogo de constantes de control

Archivo: `cotCotizacionBO.Extensions.cs` y `Actualizarcot…Extensions.cs` — el diccionario `diccionarioCatControl` mapea claves de `catControl` a los strings `"mundiales"`, `"nacionales"`, `"origen"`, `"normal"`.

**Impacto:** revisar si USP introduce productos de control distinto al catálogo actual (`Origen` es el más probable para USP). Si USP incluye controlados, la instancia PROQUIFA deberá conservar la lógica de controlados.

### 2.4 Modelo de "empresa que factura" — de literal a configuración

Las comparaciones `Empresa.Clave == "…"` y `Empresa.Prefijo == "…"` están distribuidas en:

- `ActualizarCotCotizacionTransaccionBO.Extensions.cs`
- `EmpresaBO.Extensions.cs`
- `Logic.Pqf.Logistica/MonitorCifras/MonitoreoCifras.cs`
- `Logic.Pqf.Logistica/L01.Cotizacion/cotCotizacionBO.Extensions.cs`
- `Logic.Pqf.Logistica/L01.Cotizacion/Cliente/cotClienteCotizacionBO.cs`
- `Logic.Pqf.Logistica/L01.Cotizacion/Models/GMCotCotizacion.cs`
- `Logic.Pqf.Catalogos/Clientes/Configuracion/DatosFacturacionClientes/DatosFacturacionClienteFactory.cs`
- `Logic.Pqf.Catalogos/Clientes/Configuracion/DatosFacturacionClientes/DatosFacturacionClienteBO.cs`

**Recomendación:** introducir un catálogo `catRolFacturacion` (o columnas nuevas en `Empresa`) que exprese los roles funcionales (`FacturaNormales`, `FacturaControlados`, `FacturaPublicaciones`, `FacturaServicios`, `FacturaLabware`, `FacturaUSP`) y sustituir todos los `Clave == "…"` por consultas contra esos flags. Ver §3.2.

### 2.5 Controllers de API afectados

Archivos con impacto directo:

- `WebApi.Logistica/Controllers/Procesos/L01.Cotizacion/Validacion/ValidacionOfertaProductoController.cs` — endpoint que dispara la validación de combinación de partidas. **Debe seguir devolviendo el catálogo de mensajes**, ahora parametrizado por tenant.
- `WebApi.Catalogos/Controllers/Configuracion/Empresas/EmpresaController.cs` — expone Empresa; debe reflejar los nuevos flags/columnas.
- `WebApi.Catalogos/Controllers/Configuracion/Proveedores/Relaciones/ProveedorEmpresaController.cs` — administra la relación Proveedor ↔ Empresa; será el pivote para restringir USP a PROQUIFA.
- `WebApi.Logistica/Controllers/Procesos/L05.TramitarPedido/Facturas/tpProformaPedidoExtensionsController.cs` — usa `EmpresaFactura(idRegion)` para asignar empresa al proforma.

### 2.6 Generación de PDF (DocumentBuilder)

Archivos `Logic.PDF/**` construyen PDF por empresa (letterhead, logo, RFC, dirección):

- `Logic.PDF/Cotizaciones/**` — plantillas de cotización por empresa.
- `Logic.Pqf.Logistica/PDF/Cotizacion/ArchivoBOControladosNacionales.cs` — plantilla específica de controlados.
- `Logic.Pqf.Logistica/PDF/Pedido/**` — plantillas de confirmación de pedido.
- `Logic.Pqf.Logistica/PDF/CartaDeDisponibilidad/**` — carta de disponibilidad.

**Impacto:** cada plantilla lee `Empresa` para obtener RFC, razón social y logo. Con el corte, PROQUIFA emite sólo USP y GOLOCAER emite el resto — los logotipos, avisos legales y datos bancarios deben cambiar por empresa. No hay que crear plantillas nuevas — sólo asegurar que los datos correctos de `Empresa` lleguen a la plantilla.

### 2.7 Interfacturación entre empresas del grupo

Tablas `EstrategiaInterfacturacion` y `fiPendienteInterfacturacion` implementan hoy operaciones cruzadas entre las razones sociales del grupo. En el nuevo modelo:

- Si PROQUIFA compra USP y GOLOCAER vende al cliente final una operación que incluye USP, se requiere flujo de interfacturación entre PROQUIFA → GOLOCAER. **Se debe validar si esto ocurre** (ver preguntas abiertas en el análisis maestro).
- Alternativamente, si la venta USP la hace directamente PROQUIFA al cliente final, no hay interfacturación adicional — pero hay que **eliminar** los flujos actuales de interfacturación que hoy involucran a Proquifa/Proveedora/Golocaer como si fueran del mismo grupo.

---

## 3. Impacto en Base de Datos

### 3.1 Datos maestros a particionar

| Entidad                   | Acción                                                                                                                                                                     |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Empresa`                 | Marcar Proquifa como emisora sólo de USP. Confirmar activo/inactivo de Proveedora y verificar si sigue en el grupo GOLOCAER o desaparece.                                  |
| `Marca` + `MarcaFamilia`  | Etiquetar las marcas USP para restringir su comercialización a PROQUIFA. Requiere columna nueva `IdEmpresaExclusiva` (uniqueidentifier NULL) o tabla `MarcaEmpresa` (N:N). |
| `Proveedor`               | Restringir el proveedor USP a la instancia / empresa PROQUIFA. `ProveedorEmpresa` ya modela N:N — se puede depurar dejando USP sólo con Proquifa.                          |
| `ProveedorEmpresa`        | Depuración: dejar USP asociado sólo a PROQUIFA. Los demás proveedores quedan asociados sólo a GOLOCAER (y sus frentes).                                                    |
| `Cliente`                 | Sin partición — un cliente puede seguir comprando a las dos empresas. La partición se refleja en `ClienteEmpresa` (N:N).                                                   |
| `ClienteEmpresa`          | Poblar la relación para ambas empresas cuando aplique.                                                                                                                     |
| `DatosFacturacionCliente` | Duplicar por empresa cuando el cliente compre a ambas.                                                                                                                     |

### 3.2 Cambios de esquema propuestos (mínimos)

Todos como **script DDL/DML** a ejecutar previo al corte, en la BD `ProquifaDotNet`.

#### 3.2.1 Enriquecer `Empresa` con roles explícitos

```sql
-- Nuevos flags sobre Empresa
ALTER TABLE dbo.Empresa
    ADD FacturaNormales  BIT NULL,
        FacturaUSP        BIT NULL,
        FacturaLabware    BIT NULL;

-- Backfill: PROQUIFA sigue facturando Normales; luego se separa USP
UPDATE dbo.Empresa SET FacturaNormales = 1 WHERE Clave = 'proquifa';
UPDATE dbo.Empresa SET FacturaUSP      = 1 WHERE Clave = 'proquifa';   -- tras el corte
UPDATE dbo.Empresa SET FacturaNormales = 1, FacturaUSP = 0 WHERE Clave = 'golocaer';
```

Justificación: eliminar los `Empresa.Clave == "…"` y `Prefijo == "…"` del Back, quedándose con banderas.

#### 3.2.2 Marca ↔ Empresa

Opción A — columna exclusiva en `Marca`:

```sql
ALTER TABLE dbo.Marca
    ADD IdEmpresaExclusiva UNIQUEIDENTIFIER NULL;

ALTER TABLE dbo.Marca WITH CHECK
    ADD CONSTRAINT FK_Marca_EmpresaExclusiva
        FOREIGN KEY (IdEmpresaExclusiva) REFERENCES dbo.Empresa (IdEmpresa);
```

Opción B — tabla N:N `MarcaEmpresa` (permite marcas comercializables por más de una empresa con reglas por país):

```sql
CREATE TABLE dbo.MarcaEmpresa (
    IdMarcaEmpresa      UNIQUEIDENTIFIER NOT NULL,
    IdMarca             UNIQUEIDENTIFIER NOT NULL,
    IdEmpresa           UNIQUEIDENTIFIER NOT NULL,
    Exclusiva           BIT NOT NULL,
    FechaRegistro       DATETIME NOT NULL,
    Activo              BIT NOT NULL,
    CONSTRAINT PK_MarcaEmpresa PRIMARY KEY (IdMarcaEmpresa),
    CONSTRAINT FK_MarcaEmpresa_Marca   FOREIGN KEY (IdMarca)   REFERENCES dbo.Marca   (IdMarca),
    CONSTRAINT FK_MarcaEmpresa_Empresa FOREIGN KEY (IdEmpresa) REFERENCES dbo.Empresa (IdEmpresa)
);
```

Se recomienda la **Opción B** para no romper la posibilidad de que otras marcas sean comercializables por más de una razón social.

#### 3.2.3 Catálogo de mensajes de validación

Externalizar los seis mensajes hardcoded (más los indefinidos) del BO a un catálogo:

```sql
CREATE TABLE dbo.catMensajeValidacionCotizacion (
    IdCatMensaje         UNIQUEIDENTIFIER NOT NULL,
    Codigo               VARCHAR(80)      NOT NULL,   -- msg1, msg2, ...
    Mensaje              VARCHAR(500)     NOT NULL,
    IdEmpresa            UNIQUEIDENTIFIER NULL,       -- NULL = default, valor = override por tenant
    Activo               BIT              NOT NULL,
    FechaRegistro        DATETIME         NOT NULL,
    CONSTRAINT PK_catMensajeValidacionCotizacion PRIMARY KEY (IdCatMensaje),
    CONSTRAINT FK_catMensajeValidacionCotizacion_Empresa
        FOREIGN KEY (IdEmpresa) REFERENCES dbo.Empresa (IdEmpresa)
);
```

Poblado inicial con msg1–msg6 exactos del BO.

#### 3.2.4 Vista consolidada para reportería cruzada

Como PROQUIFA y GOLOCAER coexistirán en la BD (una sola instancia inicial), se debe crear una vista `vFacturacionConsolidada` que reporte por `IdEmpresa` sin exponer el detalle transversal en objetos que rompan auditoría.

### 3.3 Migración de datos

| Paso | Descripción |
|---|---|
| M1 | Congelar operación (ventana de corte). |
| M2 | Identificar `IdMarca` USP en `Marca` y todas sus familias en `MarcaFamilia` / `MarcaFamiliaProveedor`. |
| M3 | Insertar en `MarcaEmpresa` (o setear `IdEmpresaExclusiva`) las marcas USP → PROQUIFA. |
| M4 | Verificar `ProveedorEmpresa` — dejar el proveedor USP asociado únicamente a PROQUIFA. |
| M5 | Detectar **cotizaciones abiertas** que combinen productos USP con no-USP. Definir política: dividir en dos cotizaciones o cerrar y renegociar. |
| M6 | Migrar `cotCotizacion.IdEmpresa` a la empresa correcta según el catálogo de marcas de sus partidas. |
| M7 | Poblar `catMensajeValidacionCotizacion` con msg1–msg6 y nuevos. |
| M8 | Sembrar `Empresa.FacturaNormales`, `Empresa.FacturaUSP`, `Empresa.FacturaLabware`. |

### 3.4 Objetos derivados (Stored Procedures, Views, Triggers)

- Ejecutar en la BD:

  ```sql
  SELECT name, type_desc FROM sys.objects
   WHERE type_desc IN ('SQL_STORED_PROCEDURE','VIEW','SQL_TRIGGER')
     AND OBJECT_DEFINITION(object_id) LIKE '%FacturaPublicaciones%'
     UNION ALL
  SELECT name, type_desc FROM sys.objects
   WHERE type_desc IN ('SQL_STORED_PROCEDURE','VIEW','SQL_TRIGGER')
     AND OBJECT_DEFINITION(object_id) LIKE '%FacturaControlados%';
  ```

  para identificar SPs / vistas que también leen las banderas y actualizarlos.

---

## 4. Impacto en integraciones existentes hacia el Legacy

Con base en la sección "Integraciones ETL a Legacy" del proyecto, los pipelines actuales transfieren:

- Marcas (Datos Generales)
- Proveedores (Datos Generales)
- Productos (Datos Generales, Familia, Oferta)
- Clientes (Datos Generales, Direcciones, Contactos, Datos Legales)
- Buzones (Cotización, Pedidos, Pagos, adjuntos)
- Cotizaciones (con Partidas, Fletes, PDF)
- PSC / PCC (Pendientes, Pedidos, Partidas, Cobro, Factura, Nota de Crédito, PDF)

**Impacto de la separación (sin R16):**

| ETL | Cambio |
|---|---|
| Marcas | Debe respetar el filtro `MarcaEmpresa`: la instancia legacy PROQUIFA sólo recibe USP; la instancia legacy GOLOCAER recibe el resto. |
| Proveedores | Idem — filtrado por `ProveedorEmpresa`. |
| Productos | Idem — filtrado por marca. |
| Clientes | Se replican a ambos legacies si el cliente compra a las dos. |
| Buzones | Prefijar por empresa (`BZ_PQF_*`, `BZ_GOL_*`) para evitar colisiones al escribir en legacies distintos. |
| Cotizaciones | Enviar al legacy correspondiente según `cotCotizacion.IdEmpresa`. Las cotizaciones que hoy mezclan Proquifa/Golocaer deben dividirse (ver §3.3 M5). |
| PSC / PCC | Al tramitar, el pedido ya lleva la empresa correcta; la ETL rutea a legacy PROQUIFA o legacy GOLOCAER según `tpPedido.IdEmpresa`. |

Si por costo se decide **no duplicar el legacy**, el ETL debe **etiquetar** con `IdEmpresa` cada registro para permitir extracción por empresa desde la BD del legacy.

---

## 5. Estimación gruesa de esfuerzo

| Frente | Esfuerzo (relativo) | Comentario |
|---|---|---|
| Enriquecer `Empresa` con banderas nuevas | Bajo | 1 script DDL + backfill. |
| Modelar `MarcaEmpresa` (N:N) | Bajo–Medio | 1 tabla + FKs + backfill. |
| Externalizar mensajes de validación | Medio | Refactor local en `ActualizarCotCotizacionTransaccionBO.Extensions.cs`. |
| Reemplazar `Clave == "…"` / `Prefijo == "…"` por flags | Medio–Alto | Refactor amplio (10+ archivos identificados). |
| Cambiar `EmpresaFactura(idRegion)` para recibir catálogo de marcas | Alto | Cambia una firma pública consumida por Cotización y Tramitación. |
| Cerrar divergencia Front/Back Control N/A | Medio | Requiere revisar el diccionario `diccionarioCatControl` y agregar caso Control N/A explícito. |
| Refactor de ETLs | Alto | Requiere probar contra los dos legacies (o etiquetar por empresa). |
| Migración de cotizaciones abiertas mezcladas | Medio | Depende del volumen; se estima con reporte previo al corte. |
| Duplicación de infra (BDs, servicios) | Alto | Depende de decisión física vs multi-tenant (ver análisis maestro). |

---

## 6. Preguntas de definición pendientes para dimensionar el impacto

1. ¿USP incluye productos **controlados**, **publicaciones** o **servicios**, o sólo **Normales**? Determina si la matriz de PROQUIFA se colapsa a 1 caso o mantiene submatriz.
2. ¿La empresa **Proveedora** desaparece o se conserva como razón social de GOLOCAER para controlados?
3. ¿La empresa **Ganilab** (servicios) se conserva bajo GOLOCAER?
4. ¿Se conservan los frentes comerciales **Mungen / Pharma** en GOLOCAER? ¿PROQUIFA tendrá algún frente comercial?
5. ¿La operación de **Golocaer SAC (Perú)** se conserva en la misma BD o pasa a instancia aparte?
6. ¿Existe hoy **interfacturación** entre Proquifa y las demás razones? Si sí, ¿debe seguir existiendo entre PROQUIFA y GOLOCAER post-corte?
7. ¿Se conservará **una sola BD** con `IdEmpresa` como discriminador, o se **duplica físicamente** la BD `ProquifaDotNet`?
8. ¿Se admite el modelo de **cotización paralela** con folio maestro para clientes que hoy compran USP y no-USP en un mismo pedido?

Estas respuestas condicionan cuánto del refactor descrito arriba se aplica y qué frentes pueden postergarse.

---

*Documento base — versión 0.1. Complementa `Separacion-PROQUIFA-GOLOCAER.md`.*
