# R16A-RE-FU-024 — Pendientes

> Solo pendientes de **falta de definición o actualización**, no tareas de implementación.

## Glosario

| Término | Significado |
|---|---|
| **Anexo 20** | Especificación técnica del CFDI (SAT); define **rangos de tolerancia** para valores como el TC — fuera de rango exige confirmación o se rechaza |
| **BCRP** | Banco Central de Reserva del Perú; fuente oficial peruana de tipos de cambio |
| **CFF Art. 20** | Código Fiscal de la Federación: las conversiones a MXN usan el TC publicado en el DOF; sin publicación, el último publicado |
| **DOF** | Diario Oficial de la Federación. El FIX se publica ahí el día hábil siguiente a su determinación, cuando adquiere vigencia legal |
| **FIX** | TC de referencia USD/MXN que Banxico determina cada día hábil (~12:00); referencia oficial para obligaciones en dólares |
| **Lectura DOF** | Convención adoptada: el "TC del día D" es el FIX **publicado en el DOF** el día D (= el determinado el día hábil anterior); siempre disponible al capturar |
| **REP** | Recibo Electrónico de Pagos: CFDI tipo "Pago" (complemento de pagos 2.0) que se timbra al recibir pagos de facturas a crédito; contiene `TipoCambioP` y `EquivalenciaDR` |
| **`TipoCambioP`** | Atributo del REP: TC de la **moneda del pago vs MXN** del día del pago. Obligatorio si el pago ≠ MXN; **no se emite** si el pago es en MXN |
| **`EquivalenciaDR`** | Atributo del REP por documento relacionado: unidades de la moneda del documento por 1 unidad de la moneda del pago; se **deriva de los montos aplicados**, no de una tabla de TC |
| **SIE** | Sistema de Información Económica de Banxico: API pública de series de datos (SF43718 = FIX USD/MXN) |
| **SUNAT** | Autoridad fiscal peruana. Sin equivalente del REP: los pagos no generan documento fiscal |
| **Timbrado** | Certificación del CFDI ante el SAT; vuelve inmutable el cobro (`BloqueadoPorTimbrado`) |
| **KB** | Base de conocimiento del proyecto (documentación fuente R16, solo lectura) |

---

## Negocio / Cliente (PROQUIFA)

- **Catálogo de Tipos de Inconsistencia del Paso 1** — los 6 registros iniciales de `catTipoInconsistenciaCobro` son propuesta; el catálogo definitivo lo debe entregar PROQUIFA Tesorería antes del desarrollo. (Riesgo 1 del requisito)
- ~~**Foliador COB: global vs por región** — confirmar si el consecutivo es compartido MEX+PER (una fila en `fccPagoClienteConsecutivo`, como el foliador global de proforma) o independiente por región (una fila/prefijo por región).~~ **Resuelto — DUDA-072 (21/08/2026):** consecutivo **independiente por región**, con prefijo propio por región: México = `COM` (CO=Cobro, M=México), Perú = `COP` (CO=Cobro, P=Perú); mismo formato mmddaa-consecutivo en ambas, ej. "COM-071726-1" / "COP-071726-1". Prefijos intuitivos en pantalla. Impacta la Tarea 4 del KB (que cambia de SEQUENCE a tabla + SP) y a RE-025 — actualizar ambos para el esquema por región. *(El formato de longitud del consecutivo, si aplica, sigue lo ya resuelto: 4 dígitos — ver Decisiones en el contexto.)*
## Fiscal

- ~~**Resolver las dudas del documento de propuestas** — DUDA-073 y DUDA-074, pendiente validación del cliente.~~ **Resuelto:** **DUDA-073** (fuente oficial del TC = DOF/Banxico Oficial, sin margen 2.5%, de la fecha del comprobante, campo editable) y **DUDA-074** (dos TCs por cobro: MXN fiscal + moneda de facturación operativo) fueron confirmadas por el cliente — ver OBS-049 y OBS-050 en el requisito, ya reflejadas en el documento y en el impacto BD. Sigue pendiente únicamente lo ya anotado aparte: las **validaciones del campo Fecha del cobro** (fecha no futura; advertencia si es posterior a la recepción del correo), que no forman parte de estas dos dudas resueltas.

## Técnico

- ~~**Convivencia flags ↔ `IdCatCobroEstatus`** — catálogo adoptado, 16/07/2026 se decide la dirección: el estatus es la fuente de verdad, se retiran flags.~~ **Resuelto por el DIS-SOL (v1.0, Osmar Calderón, 2026-08-24):** `Confirmado`/`BloqueadoPorTimbrado`/`FechaBloqueoTimbrado` se retiran (decisión D2); **no se agrega estatus `TIMBRADO`** — se descartó (decisión D2b, confirmado por el cliente 20/07/2026) porque el cobro no se timbra (lo timbrado es el documento fiscal); la inmutabilidad la marcan los estatus terminales `COMPLETADO`/`SALDO_A_FAVOR`/`CANCELADO`/`CON_INCONSISTENCIA`. Seed final: **7 claves** (BORRADOR, CAPTURADO, ASOCIADO, SALDO_A_FAVOR, CON_INCONSISTENCIA, COMPLETADO, CANCELADO). Solo queda `Activo` como soft-delete puro (la inconsistencia ya no lo usa, D2c). El catálogo lo crea **RE-002** con el seed que define RE-024; la FK la pone **RE-008**.
  - **Queda abierto (⚠️ del propio DIS-SOL, D2b):** cobro aplicado a varios documentos con solo algunos timbrados — la inmutabilidad se deriva de la tabla de aplicación cobro-documento que define **RE-026**.
  - **Traza temporal de las transiciones de estatus** (¿historial?) — sigue pendiente, a coordinar con RE-002/RE-008 (así lo deja el propio DIS-SOL en su sección Pendientes).
- **🆕 Conflicto de formato de folio: DIS-SOL (`COB-`) vs DUDA-072 (`COM-`/`COP-`)** — el DIS-SOL v1.0 (fechado 20/07/2026, decisión D8) especifica `COB-MMDDYY-NNNN` con prefijo único "COB" y 4 dígitos; DUDA-072 se resolvió el 21/08/2026 (**posterior** a esa versión del diseño) con prefijos diferenciados por región: `COM` México / `COP` Perú. El DIS-SOL tampoco actualizó el seed provisional `PER/COB/0` del foliador. **Pendiente:** actualizar el diseño a `COM-MMDDYY-NNNN` (y el seed de Perú) antes de construcción, o confirmar explícitamente con el coordinador si DUDA-072 debía aplicar también al nombre técnico del foliador.
- **🆕 `IdRegion` nuevo en `fccCobroCliente`** (D8b, confirmado 10/08/2026) — se hereda de `Cliente.IdRegion` vía `IdCliente` y lo llena el Mailbot (RE-008) al crear la fila en BORRADOR; es el parámetro que usa el foliador para elegir la fila correcta del consecutivo.
- **🆕 `IdEmpresa` pasa a nullable** (D13, confirmado 10/08/2026) — antes NOT NULL; puede no conocerse al insertar la fila en BORRADOR.
- **🆕 `IdCorreoRecibidoClienteReferencia`** (D12, aprobado 10/08/2026) — nullable + índice único filtrado; llave idempotente contra reintentos de los 2 BOs legacy del Mailbot que insertan varias filas por correo (T7, RE-008/FU-008). Coordinación pendiente con RE-008.
- **🆕 Colisión de nombre `ExchangeRateService`** — el servicio nuevo de Finanzas colisiona de nombre con el servicio del importador de TC (solución aparte); el DIS-SOL deja pendiente distinguir por namespace o renombrar el servicio interno.
- **🆕 Prefijo y formato del folio COB para Perú** — sin confirmar (07/08/2026); el seed `PER/COB/0` es provisional. Compartido con RE-025 (B5). Ver también el conflicto de formato arriba.

## Fusión ACEPTADA — nueva tabla `fccCobroCliente`

> **Formalizada en el DIS-SOL v1.0 (2026-08-24, decisión D1)** — el diseño técnico back-end confirma la fusión con el DDL completo de `fccCobroCliente` (ver `[R16A-RE-FU-024][DIS-SOL] Diseño de la solución.md` en esta carpeta), incluyendo los campos nuevos que esta sección aún no tenía: `IdRegion`, `IdCorreoRecibidoClienteReferencia`, `TipoDeCambioFiscal`/`TipoDeCambioFacturacion` (rename + nuevo), y `IdEmpresa` ahora nullable. La tabla nace en `BORRADOR`, no en `RECIBIDO` — el DIS-SOL descarta el estatus `RECIBIDO` propuesto abajo (ver nota siguiente).

> **Estado: ACEPTADA (16/07/2026)** por el coordinador de desarrollo. `fccFolioPagoCliente` + `fccPagoCliente` se unifican en una sola tabla. Notas de la aceptación:
> - **Nombre de la tabla: `fccCobroCliente`** (no `fccPagoCliente`).
> - **Se agrega una columna para pintar el saldo a favor más fácil** (`SaldoAFavor`) — el listado del Paso 1 la lee directo, sin JOIN.
> - **La tabla `fccSaldoFavorCliente` sigue existiendo** (ciclo de vida propio, se consume tras el timbrado — ver el análisis del saldo a favor abajo). La nueva columna es solo para pintar.
> - Toca **RE-008** (Mailbot, dueño del Buzón), **RE-023** (pantalla principal), **RE-024** (Paso 1) y **RE-026** (Paso 2) — coordinar el impacto en esos requisitos.

### Por qué se sugiere fusionarlas

Las dos tablas describen **el mismo cobro en dos momentos**: `fccFolioPagoCliente` es lo que el **Mailbot** (RE-008) leyó del correo, y `fccPagoCliente` es el cobro que el **Gestor** capturó en el Paso 1. La relación es **1:1** — un correo genera exactamente un cobro.

Tres razones para unirlas:

1. **Están separadas sin necesidad** y la frontera es tan confusa que **ya causó un error**: la nueva `fccSaldoFavorCliente` (RE-026) apuntó su FK "cobro origen" a `fccFolioPagoCliente` (el correo, que **no tiene monto**) en vez de a `fccPagoCliente`. El nombre engaña: la tabla llamada *"FolioPagoCliente"* **no guarda el folio del cobro** — ese vive en `fccPagoCliente.Folio`.
2. **Duplican datos**: el importe está como `TotalMailBot` en una y `Monto` en la otra; la moneda como `MxnMailBot/UsdMailBot` y como `IdCatMoneda`. Es el mismo dato propuesto por el bot y luego confirmado por el Gestor.
3. **Ambas están vacías** (confirmado 16/07/2026): la funcionalidad no existe aún. Fusionar hoy es **cambiar el diseño, no migrar datos**.

### Las dos tablas hoy

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│ fccFolioPagoCliente         │        │ fccPagoCliente               │
│ (Buzón — lo llena el        │        │ (Cobro — lo captura el       │
│  Mailbot, RE-008)           │        │  Gestor en el Paso 1)        │
├─────────────────────────────┤        ├──────────────────────────────┤
│ PK IdFCCFolioPagoCliente    │◄──────┐│ PK IdFCCPagoCliente          │
│ FK IdCorreoRecibidoCliente  │       └┤ FK IdFCCFolioPagoCliente     │
│ FK IdArchivo (del correo)   │        │ FK IdCliente                 │
│    Folio (del Buzón)        │        │ FK IdEmpresa                 │
│    Consecutivo              │        │ FK IdContactoCliente         │
│    FechaRecepcion           │        │ FK IdCatCobroEstatus         │
│    Stp                      │        │    Monto                     │
│    SubtotalMailBot          │        │    FechaPago                 │
│    IvaMailBot               │        │ FK IdCatMoneda               │
│    TotalMailBot             │        │    MXN / USD                 │
│    MxnMailBot               │        │    TipoDeCambio              │
│    UsdMailBot               │        │ FK IdCatMedioDePago          │
│    Activo                   │        │ FK IdDatosBancarios          │
│    FechaRegistro            │        │ FK IdCatBanco                │
│    FechaUltimaActualizacion │        │    CuentaOrdenante           │
└─────────────────────────────┘        │    ReferenciaBancaria        │
                                        │    Broker                    │
   15 columnas · sin datos              │ FK IdCatBrokerCliente        │
                                        │    InformacionComplementoPago│
                                        │ FK IdCFDI                    │
                                        │    Folio (COB-mmddaa-NNNN)   │
                                        │    Serie                     │
                                        │ FK IdArchivo (comprobante)   │
                                        │    Confirmado                │
                                        │    FechaConfirmacion         │
                                        │ FK IdUsuarioConfirmacion     │
                                        │    Notas                     │
                                        │    Activo                    │
                                        │    FechaRegistro             │
                                        │    FechaUltimaActualizacion  │
                                        └──────────────────────────────┘
                                           31 columnas · sin datos
```

### La tabla `fccCobroCliente` — diccionario de campos

Une los campos de las dos tablas + las columnas nuevas de RE-024. La columna **Estatus** marca la disposición de cada campo. Los campos que se eliminan van **al final**.

**Leyenda:** ✅ en uso · 🆕 nueva (RE-024) · 🔤 renombrada · ❓ confirmar/duda · ❌ DROP (no pasa a la tabla nueva)

| Nombre | Tipo dato | Descripción | Estatus |
|---|---|---|---|
| `IdFCCCobroCliente` | uniqueidentifier PK | Identificador del cobro | 🔤 rename de `IdFCCPagoCliente`; las FK dependientes (`fccSaldoFavorCliente`, `fccPagoFacturaPedido`, `fccInconsistenciaCobro`) deben actualizarse |
| `IdCliente` | uniqueidentifier, NO | FK Cliente | ✅ en uso — ahora directo (el Buzón no lo tenía) |
| `IdEmpresa` | uniqueidentifier, NO | FK Empresa que recibe el cobro | ✅ en uso |
| `IdContactoCliente` | uniqueidentifier, SÍ | FK Contacto del cliente | ✅ en uso |
| `IdCatCobroEstatus` | uniqueidentifier, NO | FK `catCobroEstatus` — estatus del ciclo de vida | ✅ en uso (arranca en `RECIBIDO`) |
| `IdCorreoRecibidoCliente` | uniqueidentifier, NO | FK al correo del Buzón (origen Mailbot RE-008) | ✅ en uso — ancla al correo |
| `FechaRecepcion` | datetime, SÍ | Cuándo llegó el correo | ✅ en uso — ordena el bloque de recién llegados |
| `FolioCobro` | varchar(80), SÍ | `COB-mmddaa-NNNN` al confirmar; NULL en borrador | 🔤 rename de `Folio` — único folio de la tabla |
| `Monto` | decimal(18,4), NO | Monto recibido del cliente | ✅ en uso |
| `FechaPago` | datetime, SÍ | Fecha efectiva del pago (capturada; deriva el TC) | ✅ en uso |
| `IdCatMoneda` | uniqueidentifier, SÍ | FK `catMoneda` — moneda del cobro | ✅ en uso (reemplaza `MXN`/`USD`) |
| `TipoDeCambioFiscal` | decimal(18,6), SÍ | TC del pago vs moneda fiscal de la región (México: vs MXN; Perú: NULL) | 🔤 rename de `TipoDeCambio` + ✅ en uso (dos TC, DUDA-074) |
| `TipoDeCambioFacturacion` | decimal(18,6), SÍ | TC del pago vs moneda de facturación (=1 si coinciden) | 🆕 nueva (dos TC, DUDA-074) |
| `IdCatMedioDePago` | uniqueidentifier, SÍ | FK — forma de pago (c_FormaPago SAT) | ✅ en uso |
| `IdDatosBancarios` | uniqueidentifier, SÍ | FK — cuenta destino PROQUIFA | ✅ en uso |
| `CuentaOrdenante` | varchar(80), SÍ | Cuenta origen del cliente (texto libre) | ✅ en uso |
| `ReferenciaBancaria` | varchar(80), SÍ | Referencia bancaria — **clave de rastreo del comprobante** (llave de conciliación contra el banco) | ✅ en uso. *Hueco del KB a cerrar:* el formulario del Paso 1 debe capturarla (viene en el correo, junto al Ordenante) |
| `IdArchivoComprobante` | uniqueidentifier, SÍ | Comprobante de pago seleccionado del correo | ✅ en uso (era `IdArchivo` de `fccPagoCliente`) |
| `NotasDeCobro` | varchar(500), SÍ | Notas opcionales del formulario | 🔤 rename de `Notas` |
| `FechaConfirmacion` | datetime2, SÍ | Cuándo se capturó (rastro de la transición a `CAPTURADO`) | ✅ en uso — se conserva como auditoría (el estatus no guarda *cuándo*) |
| `IdUsuarioConfirmacion` | uniqueidentifier, SÍ | Quién capturó (rastro de la transición a `CAPTURADO`) | ✅ en uso — dato de auditoría no recuperable de otro lado |
| `IdCFDI` | uniqueidentifier, SÍ | FK al REP timbrado (lo llena el Paso 3) | ✅ en uso |
| `SaldoAFavor` | decimal(18,4), SÍ | Monto del saldo a favor a mostrar en el listado; NULL = sin saldo | 🆕 nueva. "Pinta" el saldo sin JOIN a `fccSaldoFavorCliente`, y resuelve **qué monto muestra la etiqueta** (el residual). ⚠️ duplica `fccSaldoFavorCliente.Monto` → definir quién la sincroniza |
| `Activo` | bit, NO | 1=activo; 0=inconsistencia (elimina el pendiente del Buzón) | ✅ en uso |
| `FechaRegistro` | datetime2(7), NO | Auditoría: alta del registro | ✅ control |
| `FechaUltimaActualizacion` | datetime2(7), NO | Auditoría: última modificación | ✅ control |

> **Origen de los datos:** columnas tomadas del `_BD.md` de RE-024 (v1.2) y del `ER-Finanzas.md` (08/07, que agrega `IdContactoCliente`, `IdCatCobroEstatus`, `Broker`, `IdCatBrokerCliente`, `InformacionComplementoPago`, `IdCFDI`, `Serie` — no listadas en el `_BD.md`). Las dos fuentes de la KB están desalineadas; esta tabla es la unión.

### Campos que se eliminaron (no pasan a `fccCobroCliente`)

Columnas de las dos tablas originales que **no** van a la tabla fusionada:

| Nombre | Tipo dato | Descripción | ❌ Motivo del DROP |
|---|---|---|---|
| `IdFCCFolioPagoCliente` | uniqueidentifier | FK a la antigua tabla del Buzón | Ya no existe tabla separada |
| `IdArchivoCorreo` | uniqueidentifier | Archivo del correo | Un correo trae **varios** archivos; ya existe `ArchivoCorreoRecibido` (1:N). Se alcanzan vía `IdCorreoRecibidoCliente → CorreoRecibido → ArchivoCorreoRecibido` |
| `FolioBuzon` | varchar | Folio del Buzón | Redundante; el único folio es `FolioCobro` |
| `ConsecutivoBuzon` | int | Consecutivo del folio Buzón | Dependía de `FolioBuzon` |
| `SubtotalMailBot` | decimal | Subtotal que leyó el bot | Un cobro es un solo `Monto`, no se desglosa (eso es estructura de factura) |
| `IvaMailBot` | decimal | IVA que leyó el bot | Idem |
| `TotalMailBot` | decimal | Total que leyó el bot | Mapea a `Monto`; solo serviría al auto-completado (no comprometido en R16) |
| `MxnMailBot` | bit | Moneda pesos que leyó el bot | La moneda es `IdCatMoneda` |
| `UsdMailBot` | bit | Moneda dólares que leyó el bot | Idem |
| `Confirmado` | bit | 0=borrador / 1=capturado | Redundante; `BORRADOR`/`CAPTURADO` ya son estatus de `catCobroEstatus`. Su rastro (quién/cuándo) se conserva en `FechaConfirmacion`/`IdUsuarioConfirmacion` |
| `IdCatBanco` | uniqueidentifier | Banco emisor del cliente | El formulario captura "Cuenta origen" como texto libre, sin combo de banco (confirmado por maqueta) |
| `Stp` | bit | 1 = cobro vía STP (Sistema de Transferencias y Pagos) | Banderita de un caso particular; la vía del pago se expresa con `IdCatMedioDePago` |
| `MXN` | bit | Bandera moneda pesos (legacy) | Reemplazada por `IdCatMoneda` |
| `USD` | bit | Bandera moneda dólares (legacy) | Idem |
| `Broker` | bit | ¿pagó un bróker/tercero? | El bróker es **dato del cliente** (`catBrokerCliente` vía `IdCliente`), no del cobro. *Residual:* si Tesorería confirma pago directo/vía-bróker mixto, cabría un flag per-pago — solo el flag, no el FK |
| `IdCatBrokerCliente` | uniqueidentifier | FK bróker/tercero | Identidad del bróker es dato del cliente; se obtiene vía `IdCliente → catBrokerCliente` |
| `InformacionComplementoPago` | bit | Sin uso claro | El REP se arma en el Paso 3 con datos reales; un bit suelto no aporta |
| `Serie` | varchar | Serie fiscal | El folio COB no lleva serie; la serie fiscal vive en el CFDI/`EmpresaFolio` |

### Propuesta: el estado del cobro vive solo en `catCobroEstatus` (16/07/2026)

Se retiran las banderas de estado (`BloqueadoPorTimbrado`, `FechaBloqueoTimbrado`, `Confirmado`) y su función se expresa con el estatus: se propone agregar `TIMBRADO` (cobro inmutable porque un documento asociado se timbró en el Paso 3) al ciclo `RECIBIDO → BORRADOR → CAPTURADO → ASOCIADO → TIMBRADO → COMPLETADO; SALDO_A_FAVOR; CON_INCONSISTENCIA / CANCELADO`; el `canEdit` pasa a derivarse del estatus (editable en `CAPTURADO`/`ASOCIADO`, inmutable desde `TIMBRADO`); solo queda `Activo` como soft-delete; el rastro de captura se conserva en `FechaConfirmacion`/`IdUsuarioConfirmacion`; **a definir** la traza temporal de las demás transiciones (¿historial de estatus?) y el alta de `TIMBRADO`, que la coordinan **RE-002** (catálogo) y **RE-008** (FK).

### Beneficios de la tabla propuesta

- **Un cobro = un registro.** Hoy, ver un solo cobro obliga a un JOIN 1:1 entre dos tablas; fusionadas es un `SELECT` directo.
- **Se acaba la confusión de nombres** que ya causó el error de la FK del saldo a favor.
- **`IdCliente` queda directo.** Hoy el Buzón no tiene el cliente — se resuelve navegando `IdCorreoRecibidoCliente → CorreoRecibidoCliente`.
- **Lo que propuso el bot y lo que capturó el Gestor quedan lado a lado** — se puede medir qué tan bien lee el bot y alimentar el auto-completado del formulario.
- **Mismo patrón que el proyecto ya adoptó** con `fccFactura`: una sola tabla escrita por varios requisitos en etapas distintas (RE-015 crea, RE-018/019/020 timbra, RE-026/027 cobra, RE-032 cancela), diferenciada por estado. *"Dos requisitos escriben la tabla"* dejó de ser un argumento aquí.

**En contra (evaluado):** si el Buzón se consulta como dominio propio (RE-008/023 listan correos sin importar el cobro), la fusión mezcla dominios. *(El otro argumento —columnas muertas pre-captura— casi desapareció: tras eliminar `FolioBuzon`/`ConsecutivoBuzon` y las 5 `*MailBot`, del Buzón solo sobreviven `IdCorreoRecibidoCliente`, `FechaRecepcion` y `Stp`.)* No pesa más que el 1:1 real + las tablas vacías.

### Cómo convive con el listado del Paso 1 (tres estados)

El listado muestra tres tipos de item: **recién llegados**, **capturados** y **saldo a favor**. Con la tabla única los tres son **una sola tabla filtrada por estatus** — hoy son una **UNIÓN de dos tablas** (por eso el Back de la KB habla de "dos bloques"). La fusión convierte esa unión en un `SELECT` ordenado por estatus:

| Lo que se ve en la lista | Estatus (`catCobroEstatus`) |
|---|---|
| **"Cobro sin capturar"** + "Fecha de Recepción de Correo" (gris) | `RECIBIDO` |
| Folio `COB-mmddaa-NNNN` + "Monto de Cobro" | `CAPTURADO` |
| Folio `COB-mmddaa-NNNN` + "Saldo a favor" | `SALDO_A_FAVOR` |

El item sin capturar **no usa un folio temporal "COB-N"** (confirmado por maqueta 16/07/2026): muestra el texto fijo "Cobro sin capturar" y la fecha de recepción del correo. Ambos ordenamientos siguen funcionando porque `FechaRecepcion` (recién llegado) y `FechaPago` (capturado) viven en el mismo registro.

> **Se agrega el estatus `RECIBIDO` al catálogo** (decisión 16/07/2026). El diseño de dos tablas le daba al recién-llegado un estado *implícito* ("existe en el Buzón, aún no en el cobro"), gratis. Con la tabla única ese correo es una fila del cobro con los campos aún en NULL y necesita su estado propio: `RECIBIDO` = correo clasificado por el Mailbot, pendiente de captura. Ciclo completo: **`RECIBIDO → BORRADOR → CAPTURADO → ASOCIADO / SALDO_A_FAVOR → COMPLETADO; CON_INCONSISTENCIA / CANCELADO`**. El catálogo lo crea **RE-002** y la FK la pone **RE-008** — coordinar el alta de `RECIBIDO` al validar esta propuesta.

### Saldo a favor en el Paso 1 — confirmar con RE-026

Ya resuelto: es el mismo cobro con la etiqueta del monto cambiada (Criterio B1), solo lectura en el Paso 1, y `isCreditBalance = IdCatCobroEstatus = SALDO_A_FAVOR` (sin JOIN). **Dos puntos a confirmar con RE-026** (dueño de `fccSaldoFavorCliente`): (1) qué monto muestra la etiqueta — el residual, no el del cobro; ¿desde la columna `SaldoAFavor` de `fccCobroCliente` o desde `fccSaldoFavorCliente.Monto`?; (2) corregir la FK de `fccSaldoFavorCliente` para que apunte al cobro (`fccCobroCliente`).

---

## Revisión

- **Revisar comentarios del coordinador de desarrollo** sobre [R16-RE024-PROPUESTAS-DUDAS-073-074.md](R16-RE024-PROPUESTAS-DUDAS-073-074.md) — ir atendiéndolos uno por uno. Atendidos:
  - 14/07/2026 — el "ruido" del TC del día en el Cómo de la DUDA-073 (línea de tiempo determinación vs publicación DOF).
  - 15/07/2026 — lote completo de la revisión: ejemplos con datos reales de BD (pago 14-jul + pago en inhábil); **corrección de la tolerancia del timbrador** (verificado contra el XLS oficial `catCFDI_V_4_20260703`: porcentaje de variación homologado en **500% para todas las monedas** → el ×1.025 NO falla; el argumento pasó a exactitud fiscal/contable); ejemplos de flujos multi-moneda en la DUDA-074 incluido el caso EUR/USD (SAT + dolarización exacta); sección "Tema relacionado" pedido→proforma/factura con la tabla de escenarios de moneda (fuera de alcance de Validar Cobro, anotado como pendiente en RE-013); "Resumen para el cliente" al final — un bloque por duda + segunda parte de la 073 (valor oficial vs venta), con `FechaPago` = fecha del pago según el comprobante (única fecha a la que el sistema tiene acceso); pasada de entendibilidad (nota de navegación, timbrador en bullets, jerga del proyecto glosada, sin repeticiones); y bloque "Dependencia operativa" en la 073 (la fecha la captura el usuario — mitigaciones + 2 validaciones propuestas del campo Fecha del cobro, **a espejear al contexto cuando el coordinador valide**).

## Transversal (compartido con RE-023)

- ~~**Denominación canónica del rol** — "Gestor de Cobranza" vs "Analista de Cuentas por Cobrar".~~ **(Resuelto DUDA-047, 2026-08-21):** denominación canónica del rol operativo = "Gestor de Cobranza".
