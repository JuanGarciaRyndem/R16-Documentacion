# Adenda de Factura — Impacto en Base de Datos

**Referencia:** Continuación de `[PROPUESTA] Adenda de Factura.md` (Osmar), a la luz de `INFORMACIÓN DE NODOS DE ADENDAS.pdf` entregado por el equipo SAP/Finanzas.
**Fecha:** 2026-08-31
**Estado:** Propuesta — modelo de datos genérico validado contra los 4 formatos reales de addenda actualmente vigentes.

---

## 1. Objetivo de este documento

El análisis previo de Osmar (`[PROPUESTA] Adenda de Factura.md`) propuso un modelo de tablas **agnóstico al cliente** (`catTipoAddenda`, `fccAddenda`, `fccAddendaPartida`) basado en pares `Clave`/`Valor`, dejando pendiente validarlo contra formatos reales de addenda distintos a Sanofi.

El equipo SAP ahora entregó `INFORMACIÓN DE NODOS DE ADENDAS.pdf`, con la estructura XML exacta de **4 formatos** que cubren **6 clientes**:

| Formato | Clientes que lo usan |
|---|---|
| Mavi | Mavi |
| Pfizer | Pfizer |
| Asofarma | Asofarma |
| Sanofi | Sanofi, Sanofi Pasteur, Azteca Vacunas |

> Nota: la lista de clientes cambió respecto a la que manejaba Osmar como referencia (Amece/Asofarma/Mavi/Pfizer/Sanofi). El PDF de SAP **no menciona a Amece** y sí agrega **Sanofi Pasteur** y **Azteca Vacunas** (ambos con el mismo formato que Sanofi). Ver sección 6, pendiente de validar con negocio.

Este documento clasifica **cada campo de cada formato**, confirma que el modelo de tablas de Osmar sigue siendo válido y suficientemente genérico, y lo ajusta con los hallazgos nuevos.

---

## 2. Hallazgo principal

Al clasificar los ~50 campos que exigen los 4 formatos, la gran mayoría **no son datos nuevos que pqf deba capturar**. Se dividen en tres categorías:

1. **Fijo de plantilla** — un valor constante propio del formato del cliente (ej. `NumProveedor` siempre `704` para Mavi, el namespace de Sanofi, `tipoProveedor` siempre `2` para Asofarma). No cambia entre transacciones — vive en la **definición/plantilla** del lado de Finanzas, no en una tabla de pqf.
2. **Derivado de la factura** — un valor que Finanzas **ya tiene** al timbrar (`fccFactura` / `fccFacturaPartida`): moneda, montos, IVA, folio, serie, cantidad, precio unitario, unidad de medida, etc. No requiere que pqf lo capture ni lo guarde en una tabla de addenda — el motor de ensamblado lo toma directo de la factura.
3. **Genuino (requiere captura en pqf)** — un dato que **no existe en ningún otro lado** del modelo de datos de la factura y que solo tiene sentido en el contexto de la addenda de ese cliente.

Al aplicar esta clasificación a los 4 formatos, el resultado es que **el único dato verdaderamente nuevo que se repite entre clientes es el número de línea de la orden de compra del cliente** (lo que hoy ya es `LineaDeOrden` en las tablas de Sanofi) — y, únicamente para la familia Sanofi, el **correo de contacto de quien levantó la OC** (con su valor por default cuando no existe). Todo lo demás es fijo o derivable.

Esto confirma y **simplifica** la propuesta de Osmar: el modelo `fccAddenda`/`fccAddendaPartida` de pares `Clave`/`Valor` es correcto, pero en la práctica va a tener **muy pocas filas por transacción** — no un campo por cada nodo del XML.

---

## 3. Clasificación detallada por formato

Leyenda: 🔒 Fijo de plantilla · 🧮 Derivado de factura/partida · 🆕 Genuino (requiere fila en `fccAddenda`/`fccAddendaPartida`)

### 3.1 Mavi (solo a nivel trámite/cabecera — no repite por partida)

| Campo XML | Tipo | Clasificación | Regla / origen |
|---|---|---|---|
| `RfcProveedor` | elemento | 🧮 | RFC de la empresa emisora (Proveedora/Golocaer), dato ya conocido al timbrar |
| `NumProveedor` | elemento | 🔒 | Siempre `704` |
| `FechaFacturacion` | elemento | 🧮 | Fecha de la factura |
| `NumPedido` | elemento | 🧮 | Número de OC del cliente — ya existe en el pedido |
| `CodMoneda` | elemento | 🧮 | Moneda del trámite |
| `MontoTotal` | elemento | 🧮 | Total de la factura (incl. IVA) |
| `IVA` | elemento | 🧮 | IVA total de la factura |
| `PorcentajeIVA` | elemento | 🧮 | Tasa de IVA aplicable (LIVA) |
| `NumFactura` | elemento | 🧮 | Serie + folio concatenados |
| `Serie` | elemento | 🧮 | Regla fija: `"A"` si comprobante es Ingreso, `"P"` si es Pago — Finanzas ya sabe el tipo de comprobante |
| `Folio` | elemento | 🧮 | Folio de la factura |

**Conclusión Mavi:** cero campos genuinos. La addenda de Mavi se puede armar **100% del lado de Finanzas** con datos que ya tiene en `fccFactura`, sin depender de que pqf capture nada nuevo. No usa `fccAddendaPartida` (no hay detalle por línea).

### 3.2 Pfizer (`Pfizer_Ebox` > `PfizerPO` > N × `Lineas`, una por partida)

| Campo XML | Tipo | Nivel | Clasificación | Regla / origen |
|---|---|---|---|---|
| `TipoAddenda` (attr de `Pfizer_Ebox`) | atributo | Cabecera | 🔒 | Siempre `1` |
| `InstruccionesAdicionales` (attr de `PfizerPO`) | atributo | Cabecera | 🔒 | Siempre `"N/A"` |
| `AMOUNT` | atributo | Detalle | 🧮 | Importe de la partida |
| `LINE_NO` | atributo | Detalle | 🆕 | Número de línea de la orden del cliente — mismo concepto que `LineaDeOrden` de Sanofi |
| `PO_NUMBER` | atributo | Detalle | 🧮 | Número de OC del cliente (se repite en cada línea) |
| `QUANTITY` | atributo | Detalle | 🧮 | Cantidad de la partida |
| `TAX_CODE` | atributo | Detalle | 🧮 | Regla fija: `"F1"` si la tasa de impuesto de la partida es 0, si no `"F2"` — Finanzas ya conoce la tasa por partida |

**Conclusión Pfizer:** un solo campo genuino por partida (`LINE_NO` = `LineaDeOrden`). Usa `fccAddendaPartida`, no usa `fccAddenda` (no hay valores de cabecera genuinos).

### 3.3 Asofarma (`ASONIOSCOC` > `Partidas` > N × `Partida`)

| Campo XML | Tipo | Nivel | Clasificación | Regla / origen |
|---|---|---|---|---|
| `folio` | atributo | Cabecera | 🧮 | Folio de la factura |
| `noProveedor` | atributo | Cabecera | 🧮 | Mapeo fijo por empresa emisora: `"220476"` si Proveedora, `"221961"` si Golocaer — Finanzas ya sabe qué empresa emite |
| `ordenCompra` | atributo | Cabecera | 🧮 | Número de OC del cliente |
| `serie` | atributo | Cabecera | 🔒 | Siempre `"A"` |
| `tipoProveedor` | atributo | Cabecera | 🔒 | Siempre `"2"` |
| `Otros` (attr de `Partida`) | atributo | Detalle | 🔒 | Siempre `"0"` |
| `ivaAcreditable` | atributo | Detalle | 🧮 | Importe del impuesto trasladado de la partida |
| `ivaDevengado` | atributo | Detalle | 🔒 | Siempre `"0.00"` |
| `noPartida` | atributo | Detalle | 🆕 | = Número de línea de orden (mismo concepto que `LineaDeOrden`) |

**Conclusión Asofarma:** igual que Pfizer, un solo campo genuino por partida (`noPartida` = `LineaDeOrden`). El mapeo `noProveedor` por empresa emisora es una **regla fija de 2 valores**, no un dato por transacción — vive en la plantilla, no en `fccAddenda`.

### 3.4 Sanofi / Sanofi Pasteur / Azteca Vacunas (`sanofi:header` + N × `sanofi:details`)

**Cabecera (`sanofi:header`):**

| Campo | Clasificación | Regla / origen |
|---|---|---|
| `xmlns:Sanofi`, `version`, `xsi:schemaLocation` (attrs del nodo `sanofi`) | 🔒 | Constantes del namespace, fijas de plantilla |
| `TIPO_DOCTO` | 🔒 | Siempre `"01"` |
| `NUM_ORDEN` | 🧮 | Número de OC del cliente |
| `NUM_PROVEEDOR` | 🔒 | Siempre `"0001050470"` (constante propia de este cliente) |
| `FCTCONV` | 🔒 | Siempre `"1.000"` |
| `MONEDA` | 🧮 | Moneda del trámite |
| `CTA_CORREO` | 🔒 | Siempre `credito@proquifa.com.mx` |
| `IMP_RETENCION` | 🔒 | Siempre vacío (nodo autocerrado) |
| `IMP_TOTAL` | 🧮 | Total del trámite |
| `DISPONIBLE_1..4` | 🔒 | Siempre `"0.00"` |
| `CORREO_SANOFI` | 🆕 | Correo del contacto que levantó la OC del cliente; si no existe, usar default `Paola.Espinoza@sanofi.com` |

**Detalle (`sanofi:details`, uno por partida):**

| Campo | Clasificación | Regla / origen |
|---|---|---|
| `NUM_LINEA` | 🆕 | = Número de línea de orden (mismo concepto que `LineaDeOrden`) |
| `NUM_ENTRADA` | 🔒 | Siempre `"0000000000"` |
| `CUENTA_PUENTE` | ⚠️ Ver nota | El PDF dice que siempre va `"0000000000"` |
| `UNIDADES` | 🧮 | Cantidad de la partida |
| `PRECIO_UNITARIO` | 🧮 | Precio unitario de la partida |
| `IMPORTE` | 🧮 | Importe de la partida |
| `UNIDAD_MEDIDA` | 🧮 | Unidad de medida del producto — requiere mapear `IdCatUnidad` (catálogo interno) al texto que espera Sanofi (ver sección 6) |
| `TASA_IVA` | 🧮 | Tasa de IVA (LIVA) |
| `IMPORTE_IVA` | 🧮 | Monto de IVA de la partida |
| `DISPONIBLE_1..6` | 🔒 | Siempre `"0"` |

> ⚠️ **Nota — discrepancia a validar:** las tablas actuales de Sanofi en pqf tienen una columna real `CuentaPuente` (implica que históricamente se capturaba como valor variable), pero el PDF de SAP indica que `CUENTA_PUENTE` **siempre** debe ir `"0000000000"`. Antes de dejar de capturarlo como dato genuino, hay que confirmar con negocio/SAP si el campo realmente se volvió fijo o si el PDF está simplificando un caso particular. Mientras no se confirme, se recomienda mantenerlo como campo capturable en `fccAddendaPartida` (aunque hoy siempre reciba el mismo valor).

**Conclusión Sanofi:** dos campos genuinos — `CORREO_SANOFI` (cabecera, va en `fccAddenda`) y `NUM_LINEA` / `LineaDeOrden` (detalle, va en `fccAddendaPartida`) — más el pendiente de `CUENTA_PUENTE` a confirmar.

---

## 4. El patrón que emerge

En los 4 formatos, el **único dato de detalle verdaderamente genuino es el mismo concepto**: el número de línea de la orden de compra del cliente (`LineaDeOrden` / `LINE_NO` / `noPartida` / `NUM_LINEA`, según el cliente). Esto es exactamente lo que hoy ya captura Sanofi en `LineaDeOrden` — no es un caso especial de Sanofi, es **el atómo genérico de addenda por excelencia**, y se repite en 3 de los 4 formatos (todos menos Mavi, que no tiene detalle).

El único dato de cabecera verdaderamente genuino es `CORREO_SANOFI`, exclusivo de la familia Sanofi.

Esto valida la arquitectura de Osmar (pqf captura valores atómicos sueltos; Finanzas arma el XML con su propia plantilla por cliente) y reduce el riesgo de la migración: **no hay que rediseñar una captura compleja por cliente — hay que generalizar la captura de `LineaDeOrden` (que ya existe) a todos los clientes con addenda, y agregar la captura de `CorreoContacto` solo para Sanofi/Pasteur/Azteca Vacunas.**

---

## 5. Modelo de tablas propuesto (ajustado)

Se mantienen las 3 tablas de Osmar, con un ajuste a `catTipoAddenda` para que el motor de ensamblado sepa, sin adivinar, si un formato tiene nivel de detalle.

### `catTipoAddenda` (catálogo — ajustada)

| Campo | Tipo | Nota |
|---|---|---|
| `IdCatTipoAddenda` | uniqueidentifier PK | — |
| `Clave` | varchar(30) | `MAVI`, `PFIZER`, `ASOFARMA`, `SANOFI` (formato compartido por Sanofi, Sanofi Pasteur y Azteca Vacunas — ver sección 6.1) |
| `Descripcion` | varchar(100) | Nombre del formato |
| `TieneDetallePorPartida` | bit | **Nuevo.** `0` para Mavi (solo cabecera), `1` para Pfizer/Asofarma/Sanofi. Permite al motor de ensamblado y a las validaciones de pqf saber si debe exigir/permitir filas en `fccAddendaPartida` sin tener que inferirlo del catálogo de claves |
| `Activo` | bit | — |

### `fccAddenda` (valores genuinos a nivel pedido — cabecera)

Igual que la propuesta de Osmar: `IdFccAddenda`, `IdTPPedido` (FK), `IdCatTipoAddenda` (FK), `Clave`, `Valor`, `FechaGeneracion`, `FechaRegistro`, `FechaUltimaActualizacion`.

Con los formatos actuales, sus únicas filas esperadas son:

| Cliente | Clave | Ejemplo de `Valor` |
|---|---|---|
| Sanofi / Pasteur / Azteca Vacunas | `CorreoContacto` | `compras@cliente.com` (o el default si no existe) |

Mavi, Pfizer y Asofarma **no generan filas** en `fccAddenda` (todos sus campos de cabecera son fijos o derivados).

### `fccAddendaPartida` (valores genuinos a nivel partida — detalle)

Igual que la propuesta de Osmar: `IdFccAddendaPartida`, `IdTPPartidaPedido` (FK), `IdCatTipoAddenda` (FK), `Clave`, `Valor`, `FechaRegistro`, `FechaUltimaActualizacion`.

Con los formatos actuales, su única fila esperada por partida es:

| Cliente | Clave | Ejemplo de `Valor` |
|---|---|---|
| Pfizer, Asofarma, Sanofi / Pasteur / Azteca Vacunas | `LineaDeOrden` | `0034` |
| Sanofi / Pasteur / Azteca Vacunas (pendiente de confirmar, ver sección 3.4) | `CuentaPuente` | `0000000000` |

Mavi no genera filas (no tiene detalle por partida).

> El diseño **no cambia** de la propuesta original: sigue siendo genérico, sin columnas específicas de cliente, y agregar un cliente nuevo sigue sin requerir cambios de esquema — solo una fila en `catTipoAddenda` y, si aplica, nuevas `Clave`s de negocio.

---

## 6. Pendientes de validación con negocio / SAP

1. **Vigencia de clientes:** el PDF de SAP lista Mavi, Pfizer, Asofarma, Sanofi, Sanofi Pasteur y Azteca Vacunas. La propuesta previa de Osmar mencionaba también **Amece**, que no aparece aquí. Confirmar si Amece sigue vigente (y en tal caso pedir su formato) o si ya fue dado de baja.
2. **`catTipoAddenda` para la familia Sanofi:** dado que Sanofi, Sanofi Pasteur y Azteca Vacunas comparten *exactamente* el mismo formato XML, ¿conviene un solo registro `SANOFI` en el catálogo (con el cliente real resuelto por otro lado, ej. el RFC/razón social del receptor), o un registro por cliente (`SANOFI`, `SANOFI_PASTEUR`, `AZTECA_VACUNAS`) aunque la plantilla sea idéntica? Se recomienda **un registro por cliente** aunque compartan plantilla, porque valores fijos como `NUM_PROVEEDOR` (`"0001050470"`) o el correo default (`Paola.Espinoza@sanofi.com`) podrían no ser idénticos para Sanofi Pasteur o Azteca Vacunas — falta confirmarlo con SAP.
3. **`CUENTA_PUENTE` (Sanofi):** confirmar si en verdad siempre es `"0000000000"` o si el PDF simplificó un caso — ver nota en sección 3.4.
4. **`UNIDAD_MEDIDA` (Sanofi):** hoy pqf guarda `IdCatUnidad` (un GUID de catálogo interno, constante `02234CF0-230F-4F29-B686-E16E673CCE4B` en la muestra de Osmar). El XML de Sanofi espera un texto (`Pieza`, en el ejemplo del PDF). Falta definir la tabla/regla de mapeo `IdCatUnidad → texto esperado por Sanofi` (puede vivir del lado de Finanzas, como parte de la plantilla, ya que es una conversión de catálogo, no un dato nuevo por transacción).
5. **Mapeo `noProveedor` de Asofarma:** confirmar que los únicos 2 valores posibles siguen siendo `220476` (Proveedora) / `221961` (Golocaer), y si existe algún tercer emisor a futuro.
6. **`NumPedido` / `NUM_ORDEN` / `ordenCompra` / `PO_NUMBER`:** los 4 formatos usan el mismo concepto (número de OC del cliente) con nombre distinto. Confirmar que en todos los casos es el mismo campo que ya existe en el pedido (`OrdenCompra_Numero` o equivalente) y no una variante por cliente.

---

## 7. Migración de datos existentes

Las tablas actuales tienen datos reales de Sanofi (evidencia de Osmar): `ppPartidaPedidoAddendaSanofi` (1,231 filas), `tpPartidaPedidoAddendaSanofi` (1,221 filas), `tpPedidoAddendaSanofi` (0 filas, cabecera nunca usada).

Plan sugerido:

1. **No migrar histórico a `fccAddenda`/`fccAddendaPartida`** salvo que negocio necesite reprocesar facturas viejas — estas tablas nuevas nacen para nuevas transacciones a partir de la fecha de corte. El histórico se conserva solo para auditoría/consulta.
2. Si se requiere migrar: por cada fila de `tpPartidaPedidoAddendaSanofi` (la tabla "final", post Tramitar Pedido — ver hallazgo de Osmar sobre por qué la de Promesa de Compra puede quedar obsoleta), insertar en `fccAddendaPartida` una fila con `Clave='LineaDeOrden'`, `Valor=LineaDeOrden`, `IdCatTipoAddenda=SANOFI`. La columna `IdCatUnidad` no se migra como `Clave`/`Valor` si se confirma que es un valor fijo/mapeable (pendiente #4 arriba).
3. Los campos de correo (`CorreoContactoClienteAddenda`, `CorreoEmpresaAddenda`), al estar NULL en el 100% de la muestra histórica, no se migran — se capturan de cero hacia adelante bajo el nuevo requisito de `CorreoContacto`.

---

## 8. Recomendación sobre los "dos caminos" de Osmar

Osmar dejó pendiente elegir entre mantener las tablas Sanofi + un servicio de sincronización (Camino 1), o reemplazar de raíz (Camino 2). Con la evidencia de este PDF:

- El alcance real de "genuino" es mínimo (una `Clave` de detalle común a 3 de 4 formatos, una `Clave` de cabecera exclusiva de Sanofi). Esto **reduce el riesgo y el esfuerzo del Camino 2** frente a lo que se estimaba antes de tener el PDF — ya no hay que adivinar cuántos campos por cliente habrá que soportar.
- Mantener el Camino 1 (tablas Sanofi + servicio de sincronización) obligaría a crear una pieza nueva (el sincronizador) solo para traducir un modelo que, según este análisis, ya es casi genérico por sí mismo.
- **Se recomienda el Camino 2** (reemplazar `pp/tpPartidaPedidoAddendaSanofi` por `fccAddenda`/`fccAddendaPartida` directamente en las transacciones L03–L05), documentado con el detalle de implementación en `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`.
