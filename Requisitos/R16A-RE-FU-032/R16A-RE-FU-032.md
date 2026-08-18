# R16A-RE-FU-032 — Notas de Crédito: México

## Información general

| Campo | Valor |
| --- | --- |
| **ID** | R16A-RE-FU-032 |
| **ID Padre** | — |
| **Épica** | Notas de Crédito: México |
| **Módulo** | Notas de Crédito |
| **Tipo** | — |
| **Versión** | — |
| **Status** | Propuesto |
| **Fecha de Status** | — |
| **Prioridad** | — |
| **Dueño del Requisito (cliente)** | — |
| **Complejidad** | — |
| **ID Cliente** | R16.4M-RE-FU-001 |

---

## Historia de Usuario

Yo como **Analista de Cuentas por Pagar** (rol **Gestor de Cobranza**) de PROQUIFA México, quiero contar con un módulo independiente de Notas de Crédito que me permita consultar las NCs existentes y generar nuevas mediante un wizard, para emitir las NCs dentro del sistema cumpliendo la normativa SAT, dejarlas disponibles para su aplicación en Validar Cobro y eliminar el manejo actual de NCs por fuera del sistema.

---

## Requisito

El sistema debe contar con un módulo independiente de Notas de Crédito para Región México, aplicable exclusivamente a clientes Prepago en este release y operado por el área de Cuentas por Pagar. El módulo expone:

- Una **pantalla principal** de consulta de clientes con NCs (listado agrupado por cliente, con montos dolarizados) y drill-down al **detalle por cliente** con el listado individual de sus NCs.
- Un **wizard de generación de 3 pasos**: Paso 1 Buscar Factura, Paso 2 Capturar Datos, Paso 3 Confirmar (con previsualización del PDF antes de timbrar). Al terminar el timbrado, la interfaz navega a la **vista de detalle de la NC emitida**, que es la misma vista que se consulta desde el listado (sin duplicar pantallas con stepper).

La generación exige seleccionar el **motivo** desde un catálogo de tres opciones que determina automáticamente la modalidad de captura (por partidas o manual) y el tipo de relación con el CFDI origen. Cuando el motivo es **Devolución de mercancía**, el usuario elige entre TipoRelacion `01` y `03`; en los otros dos motivos el TipoRelacion se deriva automáticamente. El precio unitario en modalidad por partidas se hereda de la factura origen y no es editable.

La **forma de pago** de la NC se resuelve automáticamente mediante la comparación entre el monto de la NC y el **saldo pendiente** de la factura origen: si la NC es menor o igual al saldo pendiente se resuelve como **condonación** (FormaPago `15`) y reduce el saldo pendiente de la factura, de modo que en Validar Cobro se cobre únicamente la diferencia; si la NC excede el saldo pendiente, el **excedente** se elige como **devolución de dinero** (FormaPago real) o como **saldo a favor** disponible para consumo en facturas futuras (FormaPago `23` — Compensación).

El módulo se acopla a Validar Cobro: las NCs **con saldo a favor disponible** quedan disponibles para su aplicación durante la asociación de cobros; las NCs sin saldo a favor no aparecen. Las NCs históricas previas al go-live no se importan desde Legacy. **No existe módulo de Notas de Crédito para Región Perú** al no haber timbrado en su alcance.

---

## Alcance

### Aplica a

- Módulo independiente Notas de Crédito en PQF2 para Región México, operado por el área de Cuentas por Pagar (función **Analista de Cuentas por Pagar**, rol **Gestor de Cobranza**).
- Clientes Prepago en R16 (crédito fuera de scope este release).
- **Pantalla principal** de clientes con NCs (listado con montos dolarizados) y **drill-down al detalle por cliente** con su listado individual de NCs.
- **Wizard de generación de 3 pasos:** Paso 1 Buscar Factura, Paso 2 Capturar Datos, Paso 3 Confirmar (con previsualización del PDF antes de timbrar). La vista de la NC emitida no es un paso del wizard, es la misma vista de detalle que se consulta desde el listado.
- Selección obligatoria de cliente y de UNA factura vigente de prepago como origen de la NC (máximo 5 años de antigüedad); el listado de facturas candidatas incluye el **saldo pendiente** de cada factura entre sus datos visibles.
- **Catálogo de tres motivos** de generación con derivación automática de modalidad y TipoRelacion:
  - Devolución de mercancía → modalidad por partidas, TipoRelacion `01` o `03` elegido por el usuario.
  - Producto o piezas no entregados → modalidad por partidas, TipoRelacion `01` derivado.
  - Descuento / bonificación general → modalidad manual, TipoRelacion `01` derivado.
- **Dos modalidades de captura** derivadas del motivo, no editables:
  - **Por partidas** (devolución de mercancía; producto o piezas no entregados): tabla heredada de la factura origen con cálculo automático del monto; el **precio unitario se hereda y no es editable**; la captura por partida se hace **en piezas**.
  - **Manual** (descuento / bonificación general): captura libre de monto y concepto con materialidad fiscal.
- **Mecánica automática de forma de pago** (Sección G):
  - `NC ≤ saldo pendiente` → FormaPago `15` (Condonación) automática; reduce el saldo pendiente de la factura origen.
  - `NC > saldo pendiente` → excedente y elección del destino (devolución de dinero con FormaPago real, o saldo a favor con FormaPago `23`).
- **Consumo del saldo a favor** en facturas futuras del mismo cliente vía Validar Cobro (Sección M), con conversión de moneda cuando corresponda.
- Armado del XML conforme a CFDI 4.0 vigente (TipoDeComprobante=`E`, TipoRelacion `01` o `03` según motivo, UsoCFDI `G02` default, MetodoPago `PUE` fijo).
- **Herencia de moneda, tipo de cambio y estructura de impuestos** de la factura origen — en sustitución de la preservación del TC del día del timbrado.
- El **monto capturado** en la NC corresponde a la **base gravable** (antes de impuestos).
- **Bloqueo de NC contra el RFC genérico de público en general** (Regla 15 / Sección C).
- Timbrado vía PAC TurboPac con feedback visual (en progreso / éxito / error) y previsualización del PDF antes de timbrar.
- **El folio se consume únicamente al timbrar** (Regla 16 / Secciones I y J).
- **Foliado interno consecutivo continuo por empresa** con serie **"B2"** para las Notas de Crédito de PROQUIFA México, y UUID asignado por el SAT.
- **Envío del correo al cliente:** no automático; el envío se dispara desde el detalle de la NC con **contacto precargado del cliente que generó la factura origen** y opción de reenvío posterior (Regla 21 / Sección L).
- Soporte multi-divisa (MXN/USD/EUR) heredado de la factura origen.
- Persistencia y conservación de los XML timbrados por mínimo 5 años (Art. 30 CFF).
- Acoplamiento uni-direccional con Validar Cobro: **solo las NCs con saldo a favor disponible** (incluidas las parcialmente aplicadas) quedan disponibles en el Paso 2 del wizard de Validar Cobro.

### No aplica a

- **Región Perú:** no existe módulo de Notas de Crédito para esa región, al no haber timbrado en su alcance.
- NCs para clientes de CRÉDITO (fuera de scope R16; potencial extensión en futuras releases).
- NCs históricas pre-go-live: no se importan desde Legacy ni sistemas anteriores.
- **Cancelación de facturas y de Notas de Crédito:** no se ofrece esa función en el sistema para ningún documento fiscal. Si el cliente requiere cancelar una NC o una factura ante el SAT, se hace por fuera del sistema.
- **Corrección de errores de precio unitario mediante la modalidad por partidas:** el precio unitario se hereda de la factura origen y no es editable; una corrección de precio se maneja fuera de esta modalidad.
- **Captura por tasa cuando la factura origen tiene tasas mixtas de IVA:** escenario que no ocurre en la operación real y por lo tanto queda fuera del alcance.
- **Relación de una NC con más de una factura origen:** cada NC se emite contra exactamente una factura origen.
- **Diseño del PDF representativo de la NC** (layout, tipografía, secciones, branding, formato visual): se documenta en requisito independiente (análogo a R16A-RE-FU-021 Factura México diseño PDF). Esta fila cubre solo las funciones y la información del módulo.
- Generación o cancelación de NCs desde Validar Cobro (el acoplamiento es uni-direccional: Validar Cobro no genera ni cancela NCs).

---

## Criterios de Aceptación

### Reglas de Negocio

**Regla 1 — Módulo independiente operado por Cuentas por Pagar para prepago México**
El módulo NC aplica exclusivamente a Región México y es operado por el área de Cuentas por Pagar (función **Analista de Cuentas por Pagar**, rol **Gestor de Cobranza**). Aplica a clientes Prepago en este release.

**Regla 2 — Solo facturas vigentes prepago pueden originar NC**
El Paso 1 muestra como candidatas únicamente facturas en estado Vigente (no canceladas SAT) de clientes prepago, con un máximo de 5 años de antigüedad. El listado muestra por factura folio, UUID, fecha, total y **saldo pendiente**.

**Regla 3 — Catálogo de motivos y derivación automática de modalidad y TipoRelacion**
La generación exige seleccionar un motivo desde un catálogo de tres opciones. La **modalidad de captura** se deriva del motivo y no es editable; el **TipoRelacion** se deriva automáticamente salvo en el motivo de devolución de mercancía, donde el usuario elige entre `01` y `03`:

| Motivo | Modalidad (derivada) | TipoRelacion |
|---|---|---|
| Devolución de mercancía | Por partidas | `01` o `03` (elección del usuario) |
| Producto o piezas no entregados | Por partidas | `01` (derivado) |
| Descuento / bonificación general | Manual | `01` (derivado) |

**Regla 4 — Modalidad por partidas (devolución de mercancía; producto o piezas no entregados)**
En la modalidad por partidas el usuario ajusta la Cantidad NC por partida **en piezas** (0, parcial o total). El **precio unitario se hereda de la factura origen y no es editable**. El sistema calcula automáticamente el monto en tiempo real a partir de la cantidad y el precio unitario heredado.

**Regla 5 — Modalidad manual (descuento / bonificación general)**
En la modalidad manual el usuario captura libremente el monto y el concepto descriptivo de la materialidad fiscal. No hay tabla de partidas. Se usan por default las claves recomendadas por el SAT en su Apéndice 5 para conceptos genéricos.

**Regla 6 — Monto capturado como base gravable**
El monto capturado por el usuario (tanto en modalidad por partidas como en modalidad manual) corresponde a la **base gravable** (antes de impuestos). Los impuestos se calculan automáticamente sobre esa base heredando la estructura de la factura origen.

**Regla 7 — Herencia de moneda, tipo de cambio y estructura de impuestos**
La NC hereda de la factura origen la **moneda**, el **tipo de cambio** (cuando la moneda es distinta a MXN) y la **estructura de impuestos** (tasas y retenciones aplicadas a las partidas afectadas). No se recalculan tasas ni se preserva el TC del día del timbrado — el TC persiste al de la factura origen para consistencia con la referencia fiscal.

**Regla 8 — Determinación automática de la FormaPago (SaldoPendiente + Excedente)**
La FormaPago de la NC se determina automáticamente comparando el monto de la NC contra el **saldo pendiente** de la factura origen:

- Si `NC.Monto ≤ SaldoPendiente(factura)` → FormaPago = `15` (Condonación) automática; sin excedente.
- Si `NC.Monto > SaldoPendiente(factura)` → se calcula el **Excedente** = `NC.Monto − SaldoPendiente`; la FormaPago la determina la elección del destino del excedente (Regla 9).

**Regla 9 — Destino del excedente**
Cuando la NC genera excedente (Regla 8), el usuario elige el destino:

- **Devolución de dinero al cliente** → FormaPago real de la devolución (Transferencia, Efectivo, Tarjeta, etc.).
- **Saldo a favor** disponible para consumo en facturas futuras → FormaPago = `23` (Compensación).

Antes de avanzar al Paso 3 el sistema exige la elección del destino cuando aplica.

**Regla 10 — Campos fiscales fijos del XML**
Al armar el XML, el sistema fija: `TipoDeComprobante = E`, `CfdiRelacionados TipoRelacion = 01 o 03` (Regla 3) con UUID de la factura origen, `UsoCFDI receptor = G02` por default, y `MetodoPago = PUE` fijo inmutable según norma SAT.

**Regla 11 — Reducción del saldo pendiente de la factura origen por condonación**
Cuando la NC se resuelve como Condonación (FormaPago `15`), el sistema **reduce el saldo pendiente** de la factura origen por el monto de la NC. En Validar Cobro se cobrará únicamente la diferencia.

**Regla 12 — Inmutabilidad del total timbrado de la factura origen y múltiples NCs por factura**
La NC no cancela ni modifica el total timbrado de la factura origen; ese total permanece inmutable. Una misma factura puede recibir **más de una NC** mientras existan condiciones para hacerlo.

**Regla 13 — Estatus de aplicación de la NC**
Cada NC lleva un estatus de aplicación que refleja el consumo de su saldo a favor: Sin saldo a favor / Saldo disponible / Aplicada parcialmente / Aplicada totalmente. El estatus se actualiza al aplicarse la NC en Validar Cobro.

**Regla 14 — Consumo del saldo a favor**
El saldo a favor de una NC (FormaPago `23`) se consume contra facturas futuras del mismo cliente vía Validar Cobro. Cada consumo reduce el saldo disponible; el sistema controla el saldo remanente y evita sobrepasarlo.

**Regla 15 — Bloqueo de NC contra el RFC genérico de público en general**
El sistema no permite emitir NC contra el RFC genérico de público en general (`XAXX010101000`). El intento se bloquea en el Paso 1.

**Regla 16 — Consumo del folio únicamente al timbrar**
El folio interno de la NC se consume **únicamente al timbrar**. Un intento fallido de timbrado o el abandono del wizard antes de timbrar no consume folio.

**Regla 17 — Foliado consecutivo por empresa con serie B2**
El folio interno de la NC es consecutivo continuo por empresa del grupo PROQUIFA México, con **serie "B2"**. El UUID lo asigna el SAT al timbrar.

**Regla 18 — Sin función de cancelación en el sistema (facturas y NCs)**
El sistema no ofrece función de cancelación para ningún documento fiscal (ni facturas ni Notas de Crédito). Cualquier cancelación ante el SAT se gestiona por fuera del sistema.

**Regla 19 — Disponibilidad en Validar Cobro acotada a NCs con saldo a favor disponible**
Una NC generada queda disponible en el Paso 2 del wizard de Validar Cobro del mismo cliente **únicamente si tiene saldo a favor disponible** (incluidas las parcialmente aplicadas). Las NCs sin saldo a favor no aparecen. El acoplamiento es uni-direccional: NC alimenta a Validar Cobro, pero Validar Cobro no genera ni cancela NCs.

**Regla 20 — Conservación XML 5 años**
El XML timbrado de la NC se conserva por un mínimo de 5 años (Art. 30 CFF).

**Regla 21 — Envío del correo no automático, contacto precargado**
Al timbrar la NC el sistema **no envía el correo automáticamente**. El envío se dispara desde el detalle de la NC; el contacto **precargado** en el modal de envío es el del cliente que generó la factura origen (editable).

**Regla 22 — Manejo de errores PAC TurboPac**
Cuando el timbrado falla, el sistema muestra un mensaje de error con el detalle del problema; la NC no se persiste, no se consume folio y el usuario puede reintentar.

### Riesgos

**Riesgo 1 — Dependencia del PAC TurboPac**
Si TurboPac está indisponible, el timbrado se bloquea y las NCs no llegan a Validar Cobro.

**Riesgo 2 — Cancelación externa de una NC no visible para el sistema**
Como la cancelación de NCs se ejecuta por fuera del sistema (Regla 18), el estado de una NC cancelada externamente no se refleja automáticamente en el sistema. Requiere protocolo operativo para actualizar el estado cuando ocurra.

**Riesgo 3 — Múltiples NC sobre una misma factura sin validación por partida**
La Regla 12 permite múltiples NCs sobre una misma factura, pero el sistema no valida por partida cuánto se ha acreditado ya con NCs previas. Existe el riesgo de emitir NCs que en conjunto excedan el total de la factura por partida. **Pendiente** de definir el mecanismo de validación acumulada.

### Criterios Funcionales

#### Sección A — Pantalla principal (listado de clientes)

**Criterio A1 — Listado de clientes con NCs y dolarización de montos**
**Dado que** el usuario accede al módulo,
**Cuando** el sistema renderiza la pantalla principal,
**Entonces** deberá mostrar el listado de clientes con NCs con las columnas: Cliente, Total de NCs generadas, Vigentes, **Por Aplicar** (contabiliza las NCs con **saldo disponible**, incluidas las parcialmente aplicadas), Monto total dolarizado. Los montos se presentan **dolarizados** convirtiendo cada NC con el **tipo de cambio heredado de su factura origen**, para permitir un total único por cliente aunque las Notas de Crédito estén en distintas monedas.

**Criterio A2 — Filtros del listado principal**
**Dado que** el usuario está en la pantalla principal,
**Cuando** aplica filtros,
**Entonces** el sistema deberá ofrecer Cliente y Fecha. **No se ofrece filtro de Moneda** porque los montos ya se muestran dolarizados.

**Criterio A3 — Buscador por coincidencia**
**Dado que** el usuario está en la pantalla principal,
**Cuando** utiliza el buscador,
**Entonces** el sistema deberá filtrar por **coincidencia** sobre Cliente.

**Criterio A4 — Acción nueva NC desde listado de clientes**
**Dado que** el usuario está en la pantalla principal,
**Cuando** presiona la acción de nueva NC,
**Entonces** el sistema deberá invocar el wizard, y el Paso 1 **exigirá seleccionar el cliente** antes de continuar.

**Criterio A5 — Drill-down al detalle por cliente**
**Dado que** el usuario hace clic en un cliente del listado,
**Cuando** el sistema procesa la acción,
**Entonces** deberá navegar al detalle del cliente con su listado individual de NCs.

#### Sección B — Detalle por cliente

**Criterio B1 — Filtros del detalle por cliente**
**Dado que** el usuario está en el detalle de un cliente,
**Cuando** aplica filtros,
**Entonces** el sistema deberá ofrecer Fecha, Emisor, Estado, Moneda y buscador por folio o UUID (buscador por **coincidencia**).

**Criterio B2 — Columnas del detalle por cliente**
**Dado que** el usuario está en el detalle,
**Cuando** el sistema renderiza el listado,
**Entonces** deberá mostrar columnas: Fecha, Cobrador, Folio de la NC (acción → abre PDF), XML (acción → descarga archivo), Emisor, Monto con moneda, Factura asociada, Pedido Interno Asociado, Estado, Factura destino, Pedido destino.

**Criterio B3 — Acceso a la vista de detalle de la NC**
**Dado que** el usuario hace clic en una NC del listado,
**Cuando** el sistema procesa la acción,
**Entonces** deberá abrir la **vista de detalle de la NC** con todos los datos.

**Criterio B4 — Acción nueva NC desde detalle por cliente**
**Dado que** el usuario está en el detalle de un cliente,
**Cuando** presiona la acción de nueva NC,
**Entonces** el sistema deberá invocar el wizard con el **cliente preseleccionado**.

#### Sección C — Wizard Paso 1: Buscar Factura

**Criterio C1 — Selección obligatoria de cliente**
**Dado que** el usuario entra al Paso 1 desde la pantalla principal,
**Cuando** intenta listar facturas sin seleccionar cliente,
**Entonces** el sistema no deberá mostrar el listado de facturas hasta que se seleccione cliente. Si el wizard se abrió desde el detalle de un cliente, el cliente ya viene preseleccionado.

**Criterio C2 — Bloqueo contra RFC genérico de público en general**
**Dado que** el cliente seleccionado tiene RFC `XAXX010101000`,
**Cuando** el sistema evalúa la selección,
**Entonces** deberá **bloquear** el avance con un mensaje explicando que no se puede emitir NC contra el RFC genérico de público en general.

**Criterio C3 — Solo facturas vigentes prepago con datos y saldo pendiente**
**Dado que** se seleccionó cliente,
**Cuando** el sistema lista facturas,
**Entonces** solo deberá mostrar facturas en estado Vigente (no canceladas SAT) de prepago del cliente, con antigüedad máxima de 5 años, y por cada una deberá mostrar Folio, UUID, Fecha, Total con moneda y **Saldo pendiente**.

**Criterio C4 — Filtros y buscador del Paso 1**
**Dado que** existe el listado de facturas en el Paso 1,
**Cuando** el usuario aplica filtros o utiliza el buscador,
**Entonces** el sistema deberá ofrecer Fecha, Moneda y buscador por folio o UUID (buscador por **coincidencia**).

**Criterio C5 — Selección de una sola factura**
**Dado que** existe el listado de facturas,
**Cuando** el usuario selecciona,
**Entonces** solo deberá poder seleccionar UNA factura por NC (no múltiple).

#### Sección D — Wizard Paso 2: Capturar Datos (motivo y derivación)

**Criterio D1 — Datos de factura origen en solo lectura, con saldo pendiente**
**Dado que** el usuario entra al Paso 2,
**Cuando** el sistema renderiza la cabecera,
**Entonces** deberá mostrar de la factura origen (no editables): Folio, UUID, Tipo CFDI, RFC receptor, Razón Social, Fecha, Subtotal, IVA, Total con moneda, **Saldo pendiente**, Estado.

**Criterio D2 — Motivo desde catálogo con tres opciones**
**Dado que** existe el Paso 2,
**Cuando** el usuario captura Motivo,
**Entonces** el sistema deberá ofrecer un combo con tres opciones: Devolución de mercancía, Producto o piezas no entregados, Descuento o bonificación.

**Criterio D3 — Derivación automática de la modalidad**
**Dado que** existe la selección del Motivo,
**Cuando** el sistema renderiza el Paso 2,
**Entonces** deberá derivar automáticamente la modalidad de captura según la tabla de la Regla 3 (por partidas o manual) sin permitir edición.

**Criterio D4 — Derivación del TipoRelacion (y elección para devolución de mercancía)**
**Dado que** existe la selección del Motivo,
**Cuando** el sistema arma el XML,
**Entonces** deberá derivar automáticamente el TipoRelacion (Producto o piezas no entregados → `01`; Descuento / bonificación → `01`). En el motivo Devolución de mercancía el usuario elige entre `01` y `03`.

**Criterio D5 — Uso CFDI G02 default**
**Dado que** existe el Paso 2,
**Cuando** el sistema renderiza el campo Uso CFDI,
**Entonces** el valor por default deberá ser `G02` (Devoluciones, descuentos o bonificaciones).

#### Sección E — Modalidad por partidas

**Criterio E1 — Tabla heredada con Cantidad NC editable en piezas**
**Dado que** la modalidad es por partidas,
**Cuando** el sistema renderiza el Paso 2,
**Entonces** deberá mostrar la tabla de partidas heredada de la factura origen con columnas: **Clave del catálogo**, Producto, Cantidad Facturada, Cantidad NC (editable, en piezas), Precio Unitario (heredado, no editable), Subtotal, IVA, Total por línea.

**Criterio E2 — Captura de cantidad por partida en piezas**
**Dado que** existe la tabla de partidas,
**Cuando** el usuario edita Cantidad NC,
**Entonces** deberá poder capturar 0 (no incluida), parcial (entre 0 y Cantidad Facturada) o total (igual a Cantidad Facturada), siempre en **piezas**.

**Criterio E3 — Cálculo automático del Total NC**
**Dado que** el usuario edita Cantidad NC,
**Cuando** el sistema recalcula,
**Entonces** deberá actualizar Subtotal/IVA/Total por línea y Total NC al pie de la tabla en tiempo real.

**Criterio E4 — Concepto de la NC compuesto por partidas seleccionadas**
**Dado que** la modalidad es por partidas,
**Cuando** el sistema arma el contenido de la NC (XML y PDF),
**Entonces** el concepto deberá componerse a partir de las partidas seleccionadas con Cantidad NC > 0, heredando `ClaveProdServ`, `ClaveUnidad`, `NoIdentificacion` y `ValorUnitario` de la factura origen. Las partidas con Cantidad NC = 0 no se incluyen.

#### Sección F — Modalidad manual

**Criterio F1 — No mostrar tabla de partidas**
**Dado que** el motivo es Descuento o bonificación,
**Cuando** el sistema renderiza el Paso 2,
**Entonces** no deberá mostrar la tabla de partidas.

**Criterio F2 — Captura libre del monto (base gravable)**
**Dado que** la modalidad es manual,
**Cuando** el usuario captura el monto,
**Entonces** el sistema deberá ofrecer un campo editable para el monto (**base gravable**, antes de impuestos) en la moneda de la factura origen. El monto no debe ser mayor al Total de la factura origen.

**Criterio F3 — Concepto obligatorio con materialidad fiscal**
**Dado que** la modalidad es manual,
**Cuando** el usuario intenta avanzar al Paso 3,
**Entonces** el sistema deberá requerir que el campo Concepto esté lleno con la materialidad fiscal.

**Criterio F4 — Observaciones opcionales**
**Dado que** la modalidad es manual,
**Cuando** el usuario captura,
**Entonces** el sistema deberá ofrecer un campo opcional de Observaciones adicionales.

**Criterio F5 — ClaveProdServ y ClaveUnidad default (Apéndice 5 SAT)**
**Dado que** la modalidad es manual,
**Cuando** el sistema arma el XML,
**Entonces** deberá usar por default `ClaveProdServ = 84111506` (Servicios de facturación) y `ClaveUnidad = ACT` (Actividad), conforme al Apéndice 5 SAT.

#### Sección G — Resolución de la FormaPago (SaldoPendiente + Excedente)

**Criterio G1 — Condonación automática cuando NC ≤ SaldoPendiente**
**Dado que** el monto de la NC es menor o igual al saldo pendiente de la factura origen,
**Cuando** el sistema resuelve la FormaPago,
**Entonces** deberá asignar `FormaPago = 15` (Condonación) automáticamente, **reducir el saldo pendiente** de la factura origen por el monto de la NC y permitir avanzar al Paso 3.

**Criterio G2 — Cálculo del Excedente cuando NC > SaldoPendiente**
**Dado que** el monto de la NC es mayor al saldo pendiente de la factura origen,
**Cuando** el sistema resuelve la FormaPago,
**Entonces** deberá calcular el **Excedente = NC.Monto − SaldoPendiente** y solicitar al usuario el destino del excedente.

**Criterio G3 — Destino del excedente: devolución de dinero**
**Dado que** existe un excedente,
**Cuando** el usuario elige devolver el dinero al cliente,
**Entonces** el sistema deberá asignar la **FormaPago real** de la devolución (Transferencia, Efectivo, Tarjeta, etc.).

**Criterio G4 — Destino del excedente: saldo a favor**
**Dado que** existe un excedente,
**Cuando** el usuario elige dejarlo como saldo a favor,
**Entonces** el sistema deberá asignar `FormaPago = 23` (Compensación) y registrar el saldo a favor disponible para consumo en facturas futuras.

**Criterio G5 — Validación de avance al Paso 3**
**Dado que** existe un excedente,
**Cuando** el usuario intenta avanzar al Paso 3 sin elegir destino,
**Entonces** el sistema deberá bloquear el avance hasta que el destino esté definido.

#### Sección H — Campos fiscales del XML

**Criterio H1 — Campos obligatorios fijos**
**Dado que** existe la generación del XML,
**Cuando** el sistema arma el CFDI,
**Entonces** los campos fijos serán: `TipoDeComprobante = E` (Egreso), `CfdiRelacionados` con `TipoRelacion` según Regla 3 + UUID factura origen, `MetodoPago = PUE`.

**Criterio H2 — UsoCFDI receptor G02 default**
**Dado que** existe la generación del XML,
**Cuando** el sistema asigna UsoCFDI del receptor,
**Entonces** deberá usar `G02` (Devoluciones, descuentos o bonificaciones) por default.

**Criterio H3 — FormaPago según resolución automática**
**Dado que** existe la generación del XML,
**Cuando** el sistema asigna FormaPago,
**Entonces** deberá usar el valor resuelto por la Sección G (`15` Condonación, forma real de devolución, o `23` Compensación).

**Criterio H4 — Moneda, TipoCambio e Impuestos heredados de la factura origen**
**Dado que** existe la generación del XML,
**Cuando** el sistema asigna Moneda, TipoCambio y estructura de impuestos,
**Entonces** deberá heredarlos de la factura origen (no recalcula; no preserva el TC del día del timbrado).

**Criterio H5 — Régimen fiscal emisor y receptor**
**Dado que** existe la generación del XML,
**Cuando** el sistema asigna régimen fiscal,
**Entonces** el emisor será `601` (Personas Morales) según la empresa emisora; el receptor heredará el régimen fiscal capturado del receptor.

#### Sección I — Wizard Paso 3: Confirmar + previsualización

**Criterio I1 — Resumen completo de la NC**
**Dado que** existe el Paso 3,
**Cuando** el sistema renderiza,
**Entonces** deberá mostrar datos del emisor, datos del receptor, CFDI relacionado (folio + UUID factura origen), motivo, partidas o concepto manual según modalidad, e importes.

**Criterio I2 — Previsualización del PDF antes de timbrar**
**Dado que** existe el Paso 3,
**Cuando** el sistema renderiza,
**Entonces** el usuario deberá poder previsualizar el PDF de la NC tal como será timbrado, para validación visual previa al timbrado.

**Criterio I3 — Regreso al Paso 2 sin pérdida de datos**
**Dado que** existe el Paso 3,
**Cuando** el usuario regresa al Paso 2,
**Entonces** el sistema deberá conservar los datos capturados.

**Criterio I4 — Advertencia de irreversibilidad**
**Dado que** existe el Paso 3,
**Cuando** el sistema renderiza,
**Entonces** deberá mostrar una advertencia visible de que el timbrado es irreversible y generará un CFDI ante el SAT.

**Criterio I5 — Consumo del folio únicamente al timbrar**
**Dado que** existe el Paso 3,
**Cuando** el usuario presiona Timbrar,
**Entonces** el sistema deberá enviar el XML al PAC. El **folio se consume únicamente cuando el timbrado es exitoso**; ante error o abandono no se consume folio.

#### Sección J — Timbrado y feedback

**Criterio J1 — Feedback de progreso**
**Dado que** el usuario confirmó el timbrado,
**Cuando** el sistema envía el XML a TurboPac,
**Entonces** deberá mostrar al usuario feedback visual de progreso.

**Criterio J2 — Persistencia y consumo de folio al éxito**
**Dado que** el timbrado fue exitoso,
**Cuando** el sistema recibe el XML timbrado con sello SAT,
**Entonces** deberá persistir el XML y el PDF, **consumir el folio serie B2** y mostrar feedback de éxito.

**Criterio J3 — Navegación a la vista de detalle unificada**
**Dado que** el timbrado fue exitoso,
**Cuando** el sistema cierra el ciclo,
**Entonces** deberá navegar a la **vista de detalle de la NC** (misma vista que se consulta desde el listado), sin stepper adicional.

**Criterio J4 — Manejo de errores PAC (sin consumo de folio)**
**Dado que** el timbrado falló,
**Cuando** el sistema procesa la respuesta,
**Entonces** deberá mostrar un mensaje de error con detalle. La NC **no se persiste**, **no se consume folio** y el usuario puede reintentar.

#### Sección K — Vista de detalle de la NC (unificada con el post-timbrado)

**Criterio K1 — Información mostrada**
**Dado que** existe la vista de detalle (post-timbrado o consulta desde listado),
**Cuando** el sistema renderiza la vista,
**Entonces** deberá mostrar: UUID, datos del PAC (número de certificado SAT, RFC del SAT), Folio interno, Serie (`B2`), Tipo (`E - Egreso`), Versión CFDI 4.0, Motivo, TipoRelacion (`01` o `03`), Factura Original (folio + UUID), Receptor, Estado SAT, importes (Subtotal NC, IVA, Total NC con moneda), partidas o concepto manual, y nota legal de conservación 5 años (Art. 30 CFF).

**Criterio K2 — Acciones disponibles**
**Dado que** existe la vista de detalle,
**Cuando** el sistema renderiza,
**Entonces** deberá ofrecer las acciones: Descargar XML, Descargar PDF, Enviar/Reenviar por correo, Volver al listado.

#### Sección L — Envío del correo (no automático)

**Criterio L1 — Envío no automático desde el detalle**
**Dado que** existe la vista de detalle de la NC,
**Cuando** el usuario ejecuta la acción de enviar el correo,
**Entonces** el sistema deberá abrir el modal de envío. **No hay envío automático** al timbrar.

**Criterio L2 — Contacto precargado del cliente de la factura origen**
**Dado que** existe el modal de envío,
**Cuando** el sistema lo renderiza,
**Entonces** deberá pre-poblar Para con el **contacto del cliente que generó la factura origen** (editable); CC con el ESAC asignado y el analista de Cuentas por Cobrar (editable).

**Criterio L3 — Adjuntos automáticos**
**Dado que** existe el flujo de envío,
**Cuando** el sistema arma el correo,
**Entonces** deberá incluir automáticamente el PDF y el XML de la NC como adjuntos.

**Criterio L4 — Asunto y mensaje**
**Dado que** existe el flujo de envío,
**Cuando** el sistema arma el correo,
**Entonces** el asunto deberá incluir "Nota de Crédito + folio NC + folio de la factura relacionada" y el cuerpo deberá permitir edición libre.

#### Sección M — Consumo del saldo a favor en Validar Cobro

**Criterio M1 — Disponibilidad acotada a NCs con saldo a favor disponible**
**Dado que** se generó una NC con `FormaPago = 23` (saldo a favor),
**Cuando** se opera el Paso 2 del wizard de Validar Cobro con el mismo cliente,
**Entonces** la NC deberá aparecer disponible para asociación **solo si tiene saldo disponible** (incluidas las parcialmente aplicadas). Las NCs sin saldo a favor no aparecen.

**Criterio M2 — Control del saldo remanente**
**Dado que** se aplica una NC contra una factura futura,
**Cuando** el sistema procesa la asociación,
**Entonces** deberá reducir el saldo disponible de la NC por el monto aplicado y actualizar el estatus de aplicación (parcial o total).

**Criterio M3 — Conversión de moneda al aplicar el saldo**
**Dado que** la moneda de la NC es distinta a la moneda de la factura destino,
**Cuando** el sistema calcula el monto a aplicar,
**Entonces** deberá convertir usando el **tipo de cambio heredado por la NC de su factura origen** (Regla 7), sin usar TC del día ni del día de la aplicación.

**Criterio M4 — Acoplamiento uni-direccional**
**Dado que** existe la arquitectura del módulo,
**Cuando** se opera Validar Cobro,
**Entonces** Validar Cobro no deberá poder generar ni cancelar NCs.

#### Sección N — Foliado

**Criterio N1 — Foliado consecutivo por empresa con serie B2**
**Dado que** existe la generación de NC,
**Cuando** el sistema asigna folio interno,
**Entonces** deberá ser consecutivo continuo por empresa del grupo PROQUIFA México con serie **`B2`**.

**Criterio N2 — UUID asignado por SAT**
**Dado que** existe el timbrado,
**Cuando** el sistema recibe respuesta del SAT vía TurboPac,
**Entonces** deberá registrar el UUID único asignado por el SAT en el XML timbrado.

**Criterio N3 — Precisión interna y decimales mostrados**
**Dado que** existen cálculos y renderizado de importes,
**Cuando** el sistema opera,
**Entonces** deberá usar internamente de 6 a 8 decimales y mostrar al usuario máximo 4 decimales, respetando las reglas SAT sobre decimales en CFDI 4.0.

**Criterio N4 — Conservación XML 5 años**
**Dado que** el timbrado fue exitoso,
**Cuando** el sistema persiste,
**Entonces** deberá conservar el XML por un mínimo de 5 años (Art. 30 CFF).

---

## Observaciones

- Fila documenta el módulo NC para Región México. No existe módulo NC para Región Perú (no hay timbrado en su alcance).
- **Wizard de 3 pasos** (Paso 1 Buscar Factura, Paso 2 Capturar Datos, Paso 3 Confirmar + previsualización). La vista de la NC emitida **no es un paso del wizard**; es la misma vista de detalle que se consulta desde el listado (unificada para no mantener dos pantallas con el mismo contenido).
- Pantalla principal **agrupada por cliente** con montos **dolarizados** (TC heredado por cada NC de su factura origen) y drill-down al detalle por cliente.
- **Comportamiento diferenciado de la acción de nueva NC según pantalla de origen:** desde el listado de clientes se exige seleccionar el cliente en el Paso 1; desde el detalle de un cliente el cliente viene preseleccionado.
- **Buscadores por coincidencia** en las pantallas principal, detalle por cliente y Paso 1 del wizard.
- Dos modalidades de captura derivadas del motivo: por partidas (devolución de mercancía; producto o piezas no entregados; captura en piezas, precio unitario heredado no editable) y manual (descuento o bonificación). El concepto del CFDI en modalidad por partidas se compone únicamente de partidas con Cantidad NC > 0.
- **La corrección de errores de precio unitario mediante modalidad por partidas queda fuera de alcance:** el precio unitario se hereda y no es editable.
- **Captura por tasa con factura origen de tasas mixtas queda fuera de alcance:** escenario que no ocurre en la operación real.
- **Cancelación de facturas y NCs:** no se ofrece esa función en el sistema para ningún documento fiscal (Regla 18). Decisión documentada y acordada con el cliente.
- **Relación de una NC con más de una factura origen:** fuera de alcance; cada NC se emite contra exactamente una factura origen.
- Cumplimiento fiscal SAT validado contra Apéndice 5 Anexo 20 vigente: TipoDeComprobante `E`, TipoRelacion `01` o `03` según motivo con UUID factura origen, UsoCFDI `G02` default, MetodoPago `PUE` fijo.
- **FormaPago automática** vía comparación NC vs SaldoPendiente (Sección G): `15` Condonación (con reducción del saldo pendiente de la factura), forma real (devolución), o `23` Compensación (saldo a favor).
- **Herencia de moneda, TC y estructura de impuestos** de la factura origen (Regla 7).
- **Consumo del saldo a favor** en Validar Cobro (Sección M): disponibilidad acotada a NCs con saldo disponible; conversión con TC heredado.
- **Serie `B2`** para el foliado de las Notas de Crédito de PROQUIFA México. Folio consumido únicamente al timbrar.
- **Envío del correo:** no automático; contacto precargado del cliente que generó la factura origen.
- **Bloqueo contra RFC genérico de público en general** (`XAXX010101000`).
- **Estatus de aplicación de la NC**: Sin saldo a favor / Saldo disponible / Aplicada parcialmente / Aplicada totalmente.
- **Función operativa: Analista de Cuentas por Pagar; rol: Gestor de Cobranza.** (Cerrada la denominación canónica del rol.)
- **Cierre de dudas resueltas:**
  - FormaPago en modalidad manual: resuelto con la mecánica de SaldoPendiente + Excedente.
  - Claves de producto y unidad en la modalidad sin partidas: convención `84111506` / `ACT` documentada por el SAT (Apéndice 5).
  - Políticas de autorización por monto: resuelto — no aplica código de autorización.
- **Nota sobre la Guía Técnica de Notas de Crédito:** la Guía Técnica indica actualmente el TipoRelacion `03` como fijo para devolución de mercancía; según la Regla 3 el TipoRelacion es elección del usuario entre `01` y `03` en ese motivo. La Guía Técnica requiere actualización para reflejar esta regla.

### Documentos de referencia del cliente

- **Guía Técnica de Notas de Crédito México** — vive el detalle técnico del proceso, mecánica de FormaPago, aplicación del saldo a favor y estatus de la NC.

---

## Cambios (resumen del reenfoque)

| Bloque | Cambio principal |
|---|---|
| Motivos y modalidades | Catálogo de 3 motivos con derivación automática de modalidad y TipoRelacion (Regla 3). Devolución de mercancía permite elegir `01` o `03`; los otros dos motivos derivan `01`. Precio unitario heredado no editable; captura por partida en piezas. Se retira el TipoRelacion fijo en `01` que era incorrecto. |
| FormaPago / SaldoPendiente / Excedente | Mecánica automática (Sección G). Condonación reduce el saldo pendiente de la factura origen (Regla 11). Excedente con dos destinos: devolución (forma real) o saldo a favor (`23` Compensación). Se retira la regla de política de intentar NC antes que devolución. |
| Cancelación de documentos | Se retira la cancelación condicional de la factura origen. Regla 18 extendida: sin función de cancelación para facturas ni NCs. Sección de cancelación condicional (4 criterios) eliminada. |
| Región Perú | Se retira la referencia al módulo NC para Perú (no existe al no haber timbrado). |
| Wizard 4→3 pasos y vista unificada | Wizard de 3 pasos; la vista de la NC emitida se unifica con la vista de detalle. Sección K renombrada; criterios de banner y equivalencia retirados; Sección J navega a detalle. |
| Pantalla principal y detalle | Separación clara: listado de clientes (con dolarización) y detalle por cliente. Comportamiento diferenciado del "nueva NC" según origen. Buscadores por coincidencia. |
| Herencia, base gravable e impuestos | Regla 6 (monto = base gravable). Regla 7 (herencia de moneda/TC/impuestos de la factura origen). Se retira la preservación del TC del día del timbrado. |
| Bloqueos y foliado | Bloqueo RFC genérico público general (Regla 15 / Criterio C2). Folio consumido únicamente al timbrar (Regla 16 / Criterios I5/J2/J4). Serie **B2** confirmada (Regla 17). |
| Disponibilidad en Validar Cobro | Regla 19 / Sección M: acotada a NCs con saldo a favor disponible; parcialmente aplicadas cuentan. |
| Envío del correo | Regla 21 / Sección L: no automático; contacto precargado del cliente que generó la factura origen. |
| Datos visibles | Saldo pendiente en el Paso 1 (candidatas), en el Paso 2 (cabecera) y en el listado de clientes (dolarizado). Columna "Clave del catálogo" en tabla de partidas. |
| Riesgos | Riesgo 2 (cancelación externa no visible). Riesgo 3 (múltiples NC sobre misma factura sin validación por partida — pendiente). |
| Listado de clientes | Columna "Por Aplicar" contabiliza NCs con saldo disponible (incluidas parcialmente aplicadas). "Total" renombrado a "Total de NCs generadas". Montos **dolarizados** (TC heredado por cada NC de su factura origen). Filtro de moneda retirado. |
| Rol / función | Analista de Cuentas por Pagar (función) + Gestor de Cobranza (rol). |
| Cierres de dudas | FormaPago manual (mecánica saldo+excedente), claves producto/unidad (Apéndice 5 SAT), autorización por monto (no aplica), denominación canónica del rol, serie B2. |
| Otros | Exclusión de captura por tasas mixtas y de relación NC → N facturas origen. Referencia a la Guía Técnica de Notas de Crédito. |
