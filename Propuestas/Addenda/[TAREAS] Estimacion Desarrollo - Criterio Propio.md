# Estimación de Desarrollo — Criterio Propio (sin catálogo)

**Referencia:** Segunda lectura de `[TAREAS] Estimación Backend - Addenda de Factura.md`, con las mismas 10 tareas pero **sin aplicar las horas estándar del `Catalogo BackEnd.md`** — en su lugar, el criterio de un desarrollador senior que ya conoce el diseño completo (`[PROPUESTA] Adenda de Factura - Impacto en BD.md`, `[PROPUESTA] Adenda de Factura - Impacto en Backend.md` y `INFORMACIÓN DE NODOS DE ADENDAS.pdf`).

**Fecha:** 2026-08-31

**Por qué difiere del catálogo:** el catálogo dimensiona trabajo genérico con margen para descubrimiento e incertidumbre. Aquí esa incertidumbre ya se resolvió en el análisis previo (el esquema de tablas y los campos de cada formato de addenda están definidos campo por campo), así que la mayoría de las tareas bajan. La excepción es el motor de ensamblado, que subo por encima del catálogo: construir un motor de plantillas verdaderamente genérico es la única pieza de diseño realmente nueva de todo el proyecto, y de ella dependen las 4 plantillas de cliente.

**Advertencia:** esta es una estimación optimista de "todo sale como lo describe el diseño" — no lleva el colchón que normalmente carga una estimación de catálogo para imprevistos (datos reales no contemplados, retrabajo por code review, disponibilidad de ambientes, cambios de alcance de negocio). Úsala como piso de referencia y como argumento técnico para negociar, no como el número final a comprometer con negocio.

---

## Resumen

| Tarea | Descripción breve                                                             | Horas                                    |
| ----- | ----------------------------------------------------------------------------- | ---------------------------------------- |
| BD-1  | Construcción del modelo de datos genérico de Addenda                          | 12.00                                    |
| BD-2  | Actualización de vista dependiente y migración histórica                      | 12.00 (8 obligatorias + 4 condicionales) |
| BE-1  | Generalización de captura de Addenda en L03 y L04 (pqf)                       | 32.00                                    |
| BE-2  | Generalización de Addenda en L05 y catálogo de cliente (pqf)                  | 28.00                                    |
| BE-3  | Motor de ensamblado de Addenda + plantilla Mavi (Finanzas)                    | 48.00                                    |
| BE-4  | Definición de plantilla de Addenda — Pfizer                                   | 8.00                                     |
| BE-5  | Definición de plantilla de Addenda — Asofarma                                 | 8.00                                     |
| BE-6  | Definición de plantilla de Addenda — Sanofi / Sanofi Pasteur / Azteca Vacunas | 14.00                                    |
| BE-7  | Validación de datos obligatorios antes de timbrar                             | 10.00                                    |
| BE-8  | Endpoint de escritura de Addenda de pqf hacia Finanzas *(condicional)*        | 20.00                                    |

**Total obligatorio:** 168.00 h
**Total condicional (porción de BD-2 + BE-8):** 24.00 h
**Total general:** 192.00 h

*(Para contraste: la estimación basada en el catálogo de `[TAREAS] Estimación Backend - Addenda de Factura.md` da 278 h obligatorias + 48 h condicionales = 326 h totales.)*

---

## Detalle por tarea

### BD-1 — Construcción del modelo de datos genérico de Addenda · 12.00 h

**Descripción:** Crear las 3 piezas de esquema que soportan cualquier formato de addenda: el catálogo `catTipoAddenda` (`IdCatTipoAddenda`, `Clave`, `Descripcion`, `TieneDetallePorPartida`, `RequiereCorreoContacto`, `Activo`), la tabla `fccAddenda` (valores de cabecera por pedido, pares Clave/Valor) y `fccAddendaPartida` (valores de detalle por partida, pares Clave/Valor, sin depender de `fccAddenda`); más la alta de la columna `IdCatTipoAddenda` en `DatosFacturacionCliente`.

**Por qué esta cifra:** son 3 tablas chicas/medianas y una alteración de 1 columna, con el esquema ya definido columna por columna en el documento de BD — no hay diseño que descubrir, solo ejecutar scripts y crear las entidades EF conocidas.

---

### BD-2 — Actualización de vista dependiente y migración histórica · 12.00 h (8 obligatorias + 4 condicionales)

**Descripción:** Reescribir la vista `vTramitarPedidoPartidaDetalle` para que tome el detalle de addenda desde `fccAddendaPartida` en vez de las tablas específicas de Sanofi; y, de forma condicional (solo si negocio lo pide), migrar el histórico de `tpPartidaPedidoAddendaSanofi` (~1,221 filas) al modelo nuevo.

**Por qué esta cifra:** la vista es un cambio de origen de columnas, no una reescritura de lógica de negocio (8 h). La migración es un `INSERT...SELECT` de una sola tabla, no un proceso ETL complejo (4 h condicionales).

---

### BE-1 — Generalización de captura de Addenda en L03 y L04 (pqf) · 32.00 h

**Descripción:** Generalizar `PretramitarPromesaDeCompraTransaccionBO.cs` (L03) y `PretramitarPedidoTransaccionBO.cs`, `ppPedidoCorregidoBO.cs`, `ppPedidoTramitacionConErroresTransaccionBO.cs`/`ppPedidoOcNoAmparadaCorreoTransaccioBO.cs` (L04) para que capturen y reclonen addenda de cualquier cliente habilitado (vía `IdCatTipoAddenda`), redirigiendo la escritura de las tablas de Sanofi a `fccAddendaPartida`.

**Por qué esta cifra:** mantengo casi el mismo tamaño que el catálogo porque son 4 archivos de código legado en producción, con varios puntos de entrada (flujo normal, corrección de pedido, gestión de intramitables) y el riesgo real de regresión que ya identificó Osmar (el desfase de `LineaDeOrden` entre etapas). El diseño está claro, pero tocar transacciones existentes exige pruebas cuidadosas.

---

### BE-2 — Generalización de Addenda en L05 y catálogo de cliente (pqf) · 28.00 h

**Descripción:** Generalizar `tpPedidoTramitarTransaccionBO.cs` (renombrar `ProcesarAddendaSanofi` a `ProcesarAddenda`, fijar el valor final de `LineaDeOrden`, resolver `CorreoContacto` con su default, mantener la validación de clientes sin addenda) y actualizar `DatosFacturacionClienteFactory` para exponer `IdCatTipoAddenda` y sus características.

**Por qué esta cifra:** mismo tipo de riesgo que BE-1, pero concentrado en un solo archivo principal y con reglas ya numeradas una a una en el documento de Backend — menos superficie de cambio que L03+L04 juntas.

---

### BE-3 — Motor de ensamblado de Addenda + plantilla Mavi (Finanzas) · 48.00 h

**Descripción:** Construir, dentro del servicio de timbrado, el motor que carga la plantilla de un cliente por `IdCatTipoAddenda`, resuelve cada campo (fijo de plantilla / derivado de `fccFactura`-`fccFacturaPartida` / genuino de `fccAddenda`-`fccAddendaPartida`), repite el bloque de detalle por partida, serializa en atributos o elementos según el formato, e inserta el nodo `cfdi:Addenda`; validado de punta a punta con la plantilla de Mavi (el formato más simple).

**Por qué esta cifra:** es la única tarea donde subo el número por encima del catálogo. Un motor verdaderamente data-driven —que soporte ambas formas de serialización (atributos vs. elementos), namespaces, valores por default y repetición de detalle— es diseño nuevo real, no una adaptación de algo existente, y de él dependen las 4 plantillas de cliente.

---

### BE-4 — Definición de plantilla de Addenda — Pfizer · 8.00 h

**Descripción:** Configurar sobre el motor la plantilla de Pfizer (`Pfizer_Ebox` > `PfizerPO` > N × `Lineas`): atributos fijos `TipoAddenda="1"` e `InstruccionesAdicionales="N/A"`, y el detalle por partida (`AMOUNT`, `LINE_NO`, `PO_NUMBER`, `QUANTITY`, `TAX_CODE` con su regla por tasa de IVA).

**Por qué esta cifra:** con el motor ya construido, una plantilla es declarar campos y, en este caso, una sola regla condicional — no lógica nueva.

---

### BE-5 — Definición de plantilla de Addenda — Asofarma · 8.00 h

**Descripción:** Configurar la plantilla de Asofarma (`ASONIOSCOC` > `Partidas` > N × `Partida`): cabecera (`folio`, `noProveedor` con su mapeo por empresa emisora, `ordenCompra`, `serie` y `tipoProveedor` fijos) y detalle por partida (`Otros` y `ivaDevengado` fijos, `ivaAcreditable`, `noPartida`).

**Por qué esta cifra:** mismo caso que Pfizer — declarar campos más un mapeo simple de 2 valores.

---

### BE-6 — Definición de plantilla de Addenda — Sanofi / Sanofi Pasteur / Azteca Vacunas · 14.00 h

**Descripción:** Configurar la plantilla de Sanofi (namespace propio, `sanofi:header` con 13 campos incluyendo `CORREO_SANOFI` con su default, y N × `sanofi:details` con 15 campos por partida incluyendo `NUM_LINEA`).

**Por qué esta cifra:** más campos que declarar que Pfizer o Asofarma (28 en total) y el único caso con una regla de default real (`CORREO_SANOFI`), más el mapeo pendiente de unidad de medida — por eso queda arriba de las otras dos plantillas.

---

### BE-7 — Validación de datos obligatorios antes de timbrar · 10.00 h

**Descripción:** Antes de ensamblar el XML, verificar que existan todas las filas `fccAddenda`/`fccAddendaPartida` que la plantilla del cliente declara como obligatorias sin default, y rechazar el timbrado con un mensaje claro si falta alguna.

**Por qué esta cifra:** una vez que la plantilla declara qué es obligatorio, la validación es recorrer esa lista y comparar contra lo que existe — lógica corta y directa.

---

### BE-8 — Endpoint de escritura de Addenda de pqf hacia Finanzas *(condicional)* · 20.00 h

**Descripción:** Exponer en Finanzas el endpoint que recibe los valores de addenda capturados por pqf en L05, si `fccAddenda`/`fccAddendaPartida` viven en la base de datos de Finanzas y un INSERT directo cross-database no es viable; consumirlo desde `tpPedidoTramitarTransaccionBO.cs`.

**Por qué esta cifra:** aquí me quedo cerca del catálogo — es un endpoint nuevo con contrato, cliente HTTP y manejo de errores entre dos sistemas, trabajo estándar sin atajos por el diseño previo.
