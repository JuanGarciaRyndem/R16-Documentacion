# Estimación de Tareas — Proceso Genérico de Addenda de Factura

**Base de esta estimación:**
- `[PROPUESTA] Adenda de Factura - Impacto en BD.md`
- `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`
- Catálogo de estimación: `Catalogo BackEnd.md`
- Formato de tarea: `Estandar Redaccion Tarea.md`

**Fecha:** 2026-08-31 (v3 — tope de 45 h por tarea)
**Nota sobre el requisito:** los documentos de análisis dejan pendiente en qué requisito vive esta iniciativa (candidatos RE-018/019/020, o uno propio de "Addenda"). Mientras no se asigne, el título de cada tarea usa el marcador `[RE-PENDIENTE]` — sustitúyelo por el folio real una vez asignado.

**Criterio de unificación aplicado:**
- **Ninguna tarea rebasa 45 horas** — es el tope aplicado a todas las tareas, no solo a las de Base de Datos.
- Se agruparon únicamente los pares de trabajo cuya suma cabía dentro del tope; donde dos piezas de 24 h no podían combinarse sin pasarse (24+24=48 > 45), quedaron como tareas independientes en vez de forzar una agrupación que violara el límite.
- Se sumaron los tiempos y se agruparon las claves de catálogo de cada bloque combinado.
- Ningún total de horas cambió respecto a las versiones anteriores de esta estimación — solo se redistribuyó en más tareas, todas ≤ 45 h.

---

## Resumen de estimación (v3 — tope de 45 h)

| # | Tarea | Claves de catálogo agrupadas | Horas |
|---|---|---|---|
| BD-1 | Construcción del modelo de datos genérico de Addenda | CREATE-TABL-CH (×1) + CREATE-TABL-M (×2) + UPDATE-TABL-CH (×1) | 34.00 |
| BD-2 | Actualización de vista dependiente y migración histórica de Addenda | BD-OBJ-M (×1) + MIG-DATOS (×1, *condicional*) | 40.00 |
| BE-1 | Generalización de captura de Addenda en L03 y L04 (pqf) | IMP-EXIST-SERVICE (×1) + ALG-COMPLX-LOGIC (×1) | 44.00 |
| BE-2 | Generalización de Addenda en L05 y catálogo de cliente (pqf) | ALG-COMPLX-LOGIC (×1) + IMP-EXIST-SERVICE (×1) | 44.00 |
| BE-3 | Motor de ensamblado de Addenda + plantilla Mavi (Finanzas) | SERV-COMPLEX-TRANSACT (×1) + IMP-EXIST-SERVICE (×1) | 44.00 |
| BE-4 | Definición de plantilla de Addenda — Pfizer | ALG-BASIC-LOGIC (×1) | 24.00 |
| BE-5 | Definición de plantilla de Addenda — Asofarma | ALG-BASIC-LOGIC (×1) | 24.00 |
| BE-6 | Definición de plantilla de Addenda — Sanofi / Sanofi Pasteur / Azteca Vacunas | ALG-BASIC-LOGIC (×1) | 24.00 |
| BE-7 | Validación de datos obligatorios antes de timbrar | ALG-BASIC-LOGIC (×1) | 24.00 |
| BE-8 | Endpoint de escritura de Addenda de pqf hacia Finanzas *(condicional)* | IMPL-THIRD-SERV (×1) | 24.00 |

**Total obligatorio (BD-1, BE-1 a BE-7 + porción obligatoria de BD-2):** 278.00 h
**Total condicional (porción de BD-2 + BE-8):** 48.00 h
**Total con condicionales:** 326.00 h

*(Los totales no cambiaron respecto a las versiones anteriores — únicamente se reagruparon para que ninguna tarea supere 45 h.)*

---

## Detalle de tareas

### BD-1. Construcción del modelo de datos genérico de Addenda

## Título de la tarea

[RE-PENDIENTE] [CREATE-TABL-CH + CREATE-TABL-M ×2 + UPDATE-TABL-CH] Construcción del modelo de datos genérico de Addenda

## Descripción de la tarea

**Aplicativos:**
ProquifaNet 2 (pqf) / Finanzas

**Módulos:**
Addenda de Factura

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo. Se unifican en esta tarea las 4 piezas de esquema que conforman el modelo base — son interdependientes (las tablas de valores referencian al catálogo) y su tamaño combinado (34 h) no rebasa el tope de 45 h por tarea.

**Objetivo general:**
Crear el modelo de datos agnóstico al cliente que soporta cualquier formato de addenda, sin columnas específicas por cliente.

**Objetivos específicos:**
- Crear `catTipoAddenda` (`IdCatTipoAddenda`, `Clave`, `Descripcion`, `TieneDetallePorPartida`, `RequiereCorreoContacto`, `Activo`) y poblarlo con los 4 formatos vigentes: `MAVI`, `PFIZER`, `ASOFARMA`, `SANOFI`.
- Crear `fccAddenda` (valores de cabecera por pedido, pares Clave/Valor) con índice por `IdTPPedido`.
- Crear `fccAddendaPartida` (valores de detalle por partida, pares Clave/Valor), sin dependencia de `fccAddenda`.
- Agregar la columna `IdCatTipoAddenda` (nullable, FK) a `DatosFacturacionCliente`, en sustitución de los flags booleanos actuales (`AddendaDeLineaDeOrden`, `AddendaDeCorreo`).

**Resultado esperado:**
Un modelo de datos único, consultable, que permite a pqf y a Finanzas saber qué formato de addenda le corresponde a cada cliente y almacenar sus valores atómicos sin cambios de esquema al agregar un cliente nuevo.

**Entregables:**
Scripts de creación de las 3 tablas, script de carga del catálogo inicial, script de alteración de `DatosFacturacionCliente`, entidades EF correspondientes.

**Criterios de aceptación:**
- Las 3 tablas existen con las columnas, llaves e índices definidos.
- El catálogo queda cargado con las 4 filas iniciales, activas.
- `DatosFacturacionCliente` acepta `NULL` en `IdCatTipoAddenda` para clientes sin addenda, y queda correctamente vinculado para los clientes vigentes (Mavi, Pfizer, Asofarma, Sanofi, Sanofi Pasteur, Azteca Vacunas).

**Más información de la tarea:**
Ver sección 5 de `[PROPUESTA] Adenda de Factura - Impacto en BD.md` y sección 4.1 de `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`. Pendiente confirmar si Sanofi Pasteur y Azteca Vacunas comparten fila de catálogo con Sanofi (sección 6, pendiente #2 del documento de BD).

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura - Impacto en BD.md`
- Estándar de construcción SQL Server: `ryndem-standards:ryndem-sqlserver`

---

### BD-2. Actualización de vista dependiente y migración histórica de Addenda

## Título de la tarea

[RE-PENDIENTE] [BD-OBJ-M + MIG-DATOS] Actualización de vista dependiente y migración histórica de Addenda

## Descripción de la tarea

**Aplicativos:**
ProquifaNet 2 (pqf)

**Módulos:**
Addenda de Factura, Tramitar Pedido (dashboard)

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas y validación de conteos contra el origen. Se agrupan aquí la actualización de la vista dependiente y la migración de histórico porque ambas trabajan sobre el mismo origen de datos (las tablas actuales de Sanofi) y su combinación (16 h + 24 h = 40 h) se mantiene dentro del tope de 45 h por tarea. **La porción de migración de histórico (24 h) es condicional**: solo aplica si negocio requiere reprocesar/consultar addendas históricas bajo el modelo nuevo (ver sección 7 de `[PROPUESTA] Adenda de Factura - Impacto en BD.md`).

**Objetivo general:**
Dejar de depender de las tablas específicas de Sanofi en los objetos de consulta existentes, y decidir/ejecutar la migración de su histórico al modelo genérico.

**Objetivos específicos:**
- Reescribir `vTramitarPedidoPartidaDetalle` para tomar el detalle de addenda desde `fccAddendaPartida` (`Clave='LineaDeOrden'`) en vez de `pp/tpPartidaPedidoAddendaSanofi`.
- (Condicional) Insertar en `fccAddendaPartida` una fila por cada registro histórico de `tpPartidaPedidoAddendaSanofi` (1,221 filas, evidencia de Osmar), sin migrar los campos de correo (NULL en el 100% de la muestra histórica).

**Resultado esperado:**
El dashboard de Tramitar Pedido sigue funcionando sin romperse tras retirar las tablas específicas de Sanofi, y (si negocio lo decide) el histórico queda disponible bajo el nuevo modelo.

**Entregables:**
Script de alteración de vista; script de migración + reporte de conteo de filas migradas vs. origen (si se ejecuta la porción condicional).

**Criterios de aceptación:**
- La vista devuelve los mismos datos que hoy para pedidos con addenda Sanofi, usando el nuevo origen, y no referencia ya las tablas viejas.
- (Si aplica migración) El número de filas migradas coincide con el número de filas de origen con datos válidos.

**Más información de la tarea:**
Ver sección 2.1 y 5 de `[PROPUESTA] Adenda de Factura.md` (Osmar), sección 7 de `[PROPUESTA] Adenda de Factura - Impacto en BD.md`, y sección 7 (plan de rollout) de `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`.

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura.md`, `[PROPUESTA] Adenda de Factura - Impacto en BD.md`, `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`
- Estándar de construcción SQL Server: `ryndem-standards:ryndem-sqlserver`

---

### BE-1. Generalización de captura de Addenda en L03 y L04 (pqf)

## Título de la tarea

[RE-PENDIENTE] [IMP-EXIST-SERVICE + ALG-COMPLX-LOGIC] Generalización de captura de Addenda en L03 Promesa de Compra y L04 Pretramitar Pedido

## Descripción de la tarea

**Aplicativos:**
ProquifaNet 2 (pqf)

**Módulos:**
Promesa de Compra, Pretramitar Pedido, Gestión de Intramitables, Addenda de Factura

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo. Se integran L03 y L04 en una sola tarea por ser el mismo mecanismo (sustituir la escritura a tablas Sanofi por `fccAddendaPartida`) sobre transacciones consecutivas del mismo flujo; su combinación (12 h + 32 h = 44 h) se mantiene dentro del tope de 45 h.

**Objetivo general:**
Generalizar la captura y el reclonado de addenda en L03 y L04 para que operen sobre cualquier cliente habilitado, no solo Sanofi.

**Objetivos específicos:**
- L03 (`PretramitarPromesaDeCompraTransaccionBO.cs`): cambiar la condición de escritura a "el cliente tiene `IdCatTipoAddenda` no nulo y `TieneDetallePorPartida=true`"; redirigir el INSERT de `ppPartidaPedidoAddendaSanofi` a `fccAddendaPartida` (`Clave='LineaDeOrden'`).
- L04 (`PretramitarPedidoTransaccionBO.cs`): aplicar el mismo cambio de condición/destino.
- L04 — corrección de pedido (`ppPedidoCorregidoBO.cs`): clonar filas de `fccAddendaPartida` (por `Clave`) en vez de filas de la tabla Sanofi.
- L04 — gestión de intramitables (`ppPedidoTramitacionConErroresTransaccionBO.cs`, `ppPedidoOcNoAmparadaCorreoTransaccioBO.cs`): mismo cambio de origen/destino.

**Resultado esperado:**
Cualquier cliente con addenda de detalle por partida queda cubierto en Promesa de Compra y Pretramitar Pedido (incluida la corrección de pedidos e intramitables) sin lógica específica de Sanofi.

**Entregables:**
Código modificado en los 4 archivos + pruebas unitarias por escenario.

**Criterios de aceptación:**
- Un pedido de un cliente con addenda (cualquiera de los formatos con detalle) genera la fila correspondiente en `fccAddendaPartida` en L03 y L04.
- Al corregir un pedido con addenda, el nuevo registro conserva el valor de `LineaDeOrden` de la partida original.
- Un pedido de un cliente sin addenda no genera ninguna fila.

**Más información de la tarea:**
Ver secciones 4.2 y 4.3 de `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`. Recordar el hallazgo de Osmar: el valor de `LineaDeOrden` en esta etapa es intermedio y puede no coincidir con el valor final de L05 (ver tarea BE-2).

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`, `[PROPUESTA] Adenda de Factura.md`
- Estándar de construcción .NET: `ryndem-standards:ryndem-dotnet`

---

### BE-2. Generalización de Addenda en L05 y catálogo de cliente (pqf)

## Título de la tarea

[RE-PENDIENTE] [ALG-COMPLX-LOGIC + IMP-EXIST-SERVICE] Generalización de Addenda en L05 Tramitar Pedido y catálogo de cliente

## Descripción de la tarea

**Aplicativos:**
ProquifaNet 2 (pqf)

**Módulos:**
Tramitar Pedido, Datos de Facturación del Cliente, Addenda de Factura

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo. Se integra el ajuste al catálogo de cliente en la misma tarea de L05 porque L05 es quien primero consume `IdCatTipoAddenda` para decidir su comportamiento; su combinación (32 h + 12 h = 44 h) se mantiene dentro del tope de 45 h.

**Objetivo general:**
Generalizar `tpPedidoTramitarTransaccionBO.cs` para fijar el valor final de addenda de cualquier cliente, y asegurar que el catálogo de cliente exponga la información que esta transacción necesita.

**Objetivos específicos:**
- Renombrar/generalizar `ProcesarAddendaSanofi` a `ProcesarAddenda`, parametrizado por `IdCatTipoAddenda` del pedido.
- Recalcular y fijar en `fccAddendaPartida` el valor final de `LineaDeOrden` por partida cuando `TieneDetallePorPartida=true`.
- Resolver `CorreoContacto` (correo de quien levantó la OC del cliente, con default si no existe) cuando `RequiereCorreoContacto=true`, y guardarlo en `fccAddenda`.
- Mantener la validación existente: si el cliente no tiene addenda habilitada, ninguna partida debe traer valores de addenda.
- Inicializar `IdCatTipoAddenda` desde `DatosFacturacionClienteFactory` y exponer si el cliente requiere detalle por partida y/o correo de contacto.

**Resultado esperado:**
Al cerrar Tramitar Pedido, cualquier cliente con addenda habilitada queda con sus valores finales correctos en `fccAddenda`/`fccAddendaPartida`, leyendo un solo punto de verdad en el catálogo de cliente.

**Entregables:**
Código modificado en `tpPedidoTramitarTransaccionBO.cs` y `DatosFacturacionClienteFactory` + pruebas unitarias, incluyendo los casos de la tabla de la sección 6 de `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`.

**Criterios de aceptación:**
- El valor final de `LineaDeOrden` refleja el orden real de partidas después de cualquier reordenamiento.
- Cuando no existe correo de contacto capturado, se aplica el default definido para ese cliente.
- Un cliente configurado con `IdCatTipoAddenda` expone correctamente sus características de addenda; un cliente sin addenda expone `IdCatTipoAddenda=null` sin romper el flujo actual.

**Más información de la tarea:**
Ver secciones 4.1 y 4.4 de `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`.

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`
- Estándar de construcción .NET: `ryndem-standards:ryndem-dotnet`

---

### BE-3. Motor de ensamblado de Addenda + plantilla Mavi (Finanzas)

## Título de la tarea

[RE-PENDIENTE] [SERV-COMPLEX-TRANSACT + IMP-EXIST-SERVICE] Motor de ensamblado de Addenda y plantilla Mavi

## Descripción de la tarea

**Aplicativos:**
Finanzas (servicio de facturación/timbrado)

**Módulos:**
Facturación / Timbrado, Addenda de Factura

**Consideraciones previas:**
Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo. Se incluye la plantilla de Mavi en la misma tarea del motor porque es el formato más simple (solo cabecera) y sirve para probar el motor de punta a punta al construirlo; su combinación (32 h + 12 h = 44 h) se mantiene dentro del tope de 45 h.

**Objetivo general:**
Construir, dentro del servicio de timbrado, el motor genérico que arma el nodo `cfdi:Addenda` de cualquier cliente, y validarlo de punta a punta con el formato más simple (Mavi).

**Objetivos específicos:**
- Cargar la definición de plantilla por `IdCatTipoAddenda` del pedido facturado.
- Resolver cada campo según su origen (fijo de plantilla / derivado de `fccFactura`-`fccFacturaPartida` / genuino de `fccAddenda`-`fccAddendaPartida`, con `default` cuando aplique).
- Repetir el bloque de detalle una vez por partida facturada cuando corresponda, y serializar respetando la forma declarada (atributos XML vs. elementos hijos, namespace).
- Insertar el nodo `cfdi:Addenda` en la posición correcta del comprobante.
- Declarar la plantilla Mavi (11 campos de cabecera) como primer caso de uso del motor.

**Resultado esperado:**
Un motor único capaz de ensamblar la addenda de cualquier cliente, verificado con un caso real (Mavi) desde el primer momento.

**Entregables:**
Servicio de ensamblado + definición de plantilla Mavi + pruebas unitarias con el ejemplo del PDF (pedido `9586`, factura `A148237`).

**Criterios de aceptación:**
- El XML generado para Mavi coincide con el ejemplo de `INFORMACIÓN DE NODOS DE ADENDAS.pdf`.
- Una factura de un cliente sin addenda no genera nodo `cfdi:Addenda`.

**Más información de la tarea:**
Ver secciones 5.1 a 5.3 de `[PROPUESTA] Adenda de Factura - Impacto en Backend.md` y sección 3.1 de `[PROPUESTA] Adenda de Factura - Impacto en BD.md`.

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`, `[PROPUESTA] Adenda de Factura - Impacto en BD.md`
- Insumo de formatos: `INFORMACIÓN DE NODOS DE ADENDAS.pdf`
- Estándar de construcción .NET: `ryndem-standards:ryndem-dotnet`

---

### BE-4. Definición de plantilla de Addenda — Pfizer

## Título de la tarea

[RE-PENDIENTE] [ALG-BASIC-LOGIC] Definición de plantilla de Addenda — Pfizer

## Descripción de la tarea

**Aplicativos:**
Finanzas (servicio de facturación/timbrado)

**Módulos:**
Addenda de Factura

**Consideraciones previas:**
Depende de la tarea BE-3 (motor de ensamblado) ya construido. Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev. Queda como tarea independiente porque, al ser de 24 h, combinarla con cualquier otra plantilla de 24 h rebasaría el tope de 45 h (24+24=48).

**Objetivo general:**
Configurar la plantilla del formato Pfizer (`Pfizer_Ebox` > `PfizerPO` > N × `Lineas`, serializada por atributos XML) en el motor de ensamblado.

**Objetivos específicos:**
- Declarar los atributos fijos `TipoAddenda="1"` e `InstruccionesAdicionales="N/A"`.
- Declarar el bloque de detalle repetido por partida: `AMOUNT`, `LINE_NO` (= `LineaDeOrden`), `PO_NUMBER`, `QUANTITY`, `TAX_CODE`.
- Implementar la regla de `TAX_CODE` (`"F1"` si la tasa de IVA de la partida es 0, si no `"F2"`).

**Resultado esperado:**
Una factura de Pfizer con N partidas genera N nodos `Lineas` correctos dentro de `Pfizer_Ebox` > `PfizerPO`.

**Entregables:**
Definición de plantilla + pruebas unitarias con el ejemplo del PDF (3 líneas, `PO_NUMBER=9501315099`) y con una partida de tasa de IVA = 0.

**Criterios de aceptación:**
- El XML generado coincide con el ejemplo de la sección "Pfizer" de `INFORMACIÓN DE NODOS DE ADENDAS.pdf`.
- `TAX_CODE` resuelve correctamente en ambos escenarios de tasa de IVA.

**Más información de la tarea:**
Ver sección 3.2 de `[PROPUESTA] Adenda de Factura - Impacto en BD.md`.

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura - Impacto en BD.md`
- Insumo de formato: `INFORMACIÓN DE NODOS DE ADENDAS.pdf`

---

### BE-5. Definición de plantilla de Addenda — Asofarma

## Título de la tarea

[RE-PENDIENTE] [ALG-BASIC-LOGIC] Definición de plantilla de Addenda — Asofarma

## Descripción de la tarea

**Aplicativos:**
Finanzas (servicio de facturación/timbrado)

**Módulos:**
Addenda de Factura

**Consideraciones previas:**
Depende de la tarea BE-3 (motor de ensamblado) ya construido. Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev. Queda como tarea independiente porque, al ser de 24 h, combinarla con cualquier otra plantilla de 24 h rebasaría el tope de 45 h (24+24=48).

**Objetivo general:**
Configurar la plantilla del formato Asofarma (`ASONIOSCOC` > `Partidas` > N × `Partida`, serializada por atributos XML) en el motor de ensamblado.

**Objetivos específicos:**
- Declarar la cabecera `folio`, `noProveedor`, `ordenCompra`, `serie` (fijo `"A"`), `tipoProveedor` (fijo `"2"`).
- Implementar el mapeo de `noProveedor` por empresa emisora (`"220476"` Proveedora / `"221961"` Golocaer).
- Declarar el detalle repetido por partida: `Otros` (fijo `"0"`), `ivaAcreditable`, `ivaDevengado` (fijo `"0.00"`), `noPartida` (= `LineaDeOrden`).

**Resultado esperado:**
Una factura de Asofarma emitida por cualquiera de las dos empresas genera el nodo `ASONIOSCOC` correcto.

**Entregables:**
Definición de plantilla + pruebas unitarias con el ejemplo del PDF (folio `139732`, `ordenCompra=OC076015`) y con emisión desde Golocaer.

**Criterios de aceptación:**
- El XML generado coincide con el ejemplo de la sección "Asofarma" de `INFORMACIÓN DE NODOS DE ADENDAS.pdf`.
- `noProveedor` resuelve correctamente para ambas empresas emisoras.

**Más información de la tarea:**
Ver sección 3.3 de `[PROPUESTA] Adenda de Factura - Impacto en BD.md`. Confirmar con negocio si existe un tercer emisor (pendiente #5 del mismo documento).

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura - Impacto en BD.md`
- Insumo de formato: `INFORMACIÓN DE NODOS DE ADENDAS.pdf`

---

### BE-6. Definición de plantilla de Addenda — Sanofi / Sanofi Pasteur / Azteca Vacunas

## Título de la tarea

[RE-PENDIENTE] [ALG-BASIC-LOGIC] Definición de plantilla de Addenda — Sanofi / Sanofi Pasteur / Azteca Vacunas

## Descripción de la tarea

**Aplicativos:**
Finanzas (servicio de facturación/timbrado)

**Módulos:**
Addenda de Factura

**Consideraciones previas:**
Depende de la tarea BE-3 (motor de ensamblado) ya construido. Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev. Queda como tarea independiente porque, al ser de 24 h, combinarla con cualquier otra plantilla de 24 h rebasaría el tope de 45 h (24+24=48).

**Objetivo general:**
Configurar la plantilla del formato Sanofi (nodo `sanofi:sanofi` con namespace propio, `sanofi:header` + N × `sanofi:details`, serializada por elementos hijos) en el motor de ensamblado.

**Objetivos específicos:**
- Declarar los atributos de namespace fijos del nodo raíz (`xmlns:Sanofi`, `version`, `xsi:schemaLocation`).
- Declarar los 13 campos de `sanofi:header`, incluyendo los fijos (`TIPO_DOCTO`, `NUM_PROVEEDOR`, `FCTCONV`, `CTA_CORREO`, `IMP_RETENCION` vacío, `DISPONIBLE_1..4`) y el genuino `CORREO_SANOFI` con su default (`Paola.Espinoza@sanofi.com`).
- Declarar los 15 campos de `sanofi:details` repetidos por partida, incluyendo `NUM_LINEA` (= `LineaDeOrden`) y los fijos (`NUM_ENTRADA`, `DISPONIBLE_1..6`).
- Confirmar con negocio si Sanofi Pasteur y Azteca Vacunas usan los mismos valores fijos (`NUM_PROVEEDOR`, correo default) o requieren su propia plantilla (pendiente #2 de `[PROPUESTA] Adenda de Factura - Impacto en BD.md`).

**Resultado esperado:**
Una factura de cualquiera de los 3 clientes de la familia Sanofi genera el nodo `sanofi:sanofi` correcto, con o sin correo de contacto capturado.

**Entregables:**
Definición de plantilla + pruebas unitarias con el ejemplo del PDF (`NUM_ORDEN=C000191021`) y con un correo de contacto real capturado.

**Criterios de aceptación:**
- El XML generado coincide con el ejemplo de la sección "SANOFI" de `INFORMACIÓN DE NODOS DE ADENDAS.pdf`.
- `CORREO_SANOFI` usa el correo capturado cuando existe, y el default cuando no.

**Más información de la tarea:**
Ver sección 3.4 de `[PROPUESTA] Adenda de Factura - Impacto en BD.md`, incluyendo la nota sobre `CUENTA_PUENTE` (pendiente de confirmar si sigue siendo variable) y el mapeo de `IdCatUnidad` a `UNIDAD_MEDIDA`.

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura - Impacto en BD.md`
- Insumo de formato: `INFORMACIÓN DE NODOS DE ADENDAS.pdf`

---

### BE-7. Validación de datos obligatorios antes de timbrar

## Título de la tarea

[RE-PENDIENTE] [ALG-BASIC-LOGIC] Validación de datos obligatorios de Addenda antes de timbrar

## Descripción de la tarea

**Aplicativos:**
Finanzas (servicio de facturación/timbrado)

**Módulos:**
Facturación / Timbrado, Addenda de Factura

**Consideraciones previas:**
Depende de la tarea BE-3 (motor de ensamblado) ya construido. Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev. Queda como tarea independiente porque, al ser de 24 h, combinarla con cualquier plantilla de 24 h rebasaría el tope de 45 h (24+24=48).

**Objetivo general:**
Evitar que se timbre una factura con addenda incompleta cuando faltan datos obligatorios sin valor por default.

**Objetivos específicos:**
- Antes de ensamblar el XML, verificar que existan todas las filas `fccAddenda`/`fccAddendaPartida` que la plantilla del cliente declara como obligatorias sin default (ej. `LineaDeOrden` por cada partida).
- Rechazar el timbrado con un mensaje claro cuando falte algún dato, en vez de generar un CFDI con addenda incompleta.

**Resultado esperado:**
Ninguna factura con addenda incompleta llega a timbrarse; el usuario recibe un mensaje accionable.

**Entregables:**
Código de validación + pruebas unitarias (caso positivo y caso de partida sin `LineaDeOrden`).

**Criterios de aceptación:**
- Una factura con todos los datos obligatorios se timbra normalmente.
- Una factura a la que le falta un dato obligatorio es rechazada antes de generar el XML, con un mensaje que identifica el dato faltante.

**Más información de la tarea:**
Ver sección 5.4 de `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`.

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`
- Estándar de construcción .NET: `ryndem-standards:ryndem-dotnet`

---

### BE-8. Endpoint de escritura de Addenda de pqf hacia Finanzas *(condicional)*

## Título de la tarea

[RE-PENDIENTE] [IMPL-THIRD-SERV] Endpoint de escritura de Addenda de pqf hacia Finanzas

## Descripción de la tarea

**Aplicativos:**
ProquifaNet 2 (pqf) / Finanzas

**Módulos:**
Addenda de Factura

**Consideraciones previas:**
Tarea condicional, sin equivalente similar con el que integrarse: solo aplica si `fccAddenda`/`fccAddendaPartida` viven en la base de datos de Finanzas y no es viable un INSERT directo cross-database desde pqf (pendiente abierto tanto por Osmar como por el documento de Backend). Para esta actividad están contempladas su construcción, pruebas unitarias, aprobación del líder técnico mediante PR, liberación en dev, documentación sobre desarrollo.

**Objetivo general:**
Exponer en Finanzas el mecanismo por el cual pqf entrega los valores de addenda capturados en L05 Tramitar Pedido.

**Objetivos específicos:**
- Definir el contrato del endpoint (payload de cabecera y detalle, `IdTPPedido`, `IdCatTipoAddenda`).
- Consumir dicho endpoint desde `tpPedidoTramitarTransaccionBO.cs` (tarea BE-2) en vez de un INSERT directo.

**Resultado esperado:**
pqf y Finanzas quedan desacoplados a nivel de base de datos para la escritura de addenda.

**Entregables:**
Endpoint + cliente HTTP en pqf + pruebas de integración.

**Criterios de aceptación:**
- El endpoint persiste correctamente cabecera y detalle de addenda para un pedido dado.
- Un fallo de comunicación no deja el pedido en estado inconsistente (definir manejo de reintentos/errores).

**Más información de la tarea:**
Ver sección 8 (pendientes) de `[PROPUESTA] Adenda de Factura - Impacto en Backend.md` y sección 6 (pendientes) de Osmar (`[PROPUESTA] Adenda de Factura.md`).

**Recursos:**
- Especificación técnica: `[PROPUESTA] Adenda de Factura - Impacto en Backend.md`
- Estándar de construcción .NET: `ryndem-standards:ryndem-dotnet`
