TPSC-RE-FU-032		Notas de Crédito: México	Notas de Crédito	Yo como ** Usuario de Tesorería de PROQUIFA México **, quiero contar con un módulo independiente de Notas de Crédito que me permita consultar las NCs existentes y generar nuevas mediante un wizard, para emitir las NCs dentro del sistema cumpliendo la normativa SAT, dejarlas disponibles para su aplicación en Validar Cobro y eliminar el manejo actual de NCs por fuera del sistema.	El sistema debe contar con un módulo independiente de Notas de Crédito para Región México, aplicable exclusivamente a clientes Prepago en este release y operado por el área de Tesorería. El módulo expone una pantalla de consulta de NCs agrupada por cliente y un wizard de generación que permite seleccionar la factura origen, capturar el motivo y los datos de la NC (en modalidad por partidas o manual), confirmar con previsualización del PDF y emitir el documento timbrado. El módulo se acopla a Validar Cobro: las NCs generadas quedan disponibles para su aplicación durante la asociación de cobros. Las NCs históricas previas al go-live no se importan desde Legacy. La estructura funcional del módulo se reutiliza para Perú con diferencias significativas y se documenta en requisito independiente.			Propuesto					R16.4M-RE-FU-001	"## Aplica a
- Módulo independiente Notas de Crédito en PQF2, separado de Validar Cobro (módulos distintos; NC alimenta a Validar Cobro pero no es sub-módulo), para Región México y operado por el área de Tesorería.
- Clientes prepago en R16 (crédito fuera de scope este release).
- Pantalla principal de Consulta de NCs agrupada por cliente, con drill-down al detalle por cliente.
- Wizard de generación de 4 pasos: Paso 1 Buscar Factura, Paso 2 Capturar Datos, Paso 3 Confirmar (con previsualización del PDF antes de timbrar), Paso 4 NC Emitida (misma vista que el detalle de cualquier NC ya generada).
- Selección obligatoria de cliente y de UNA factura vigente de prepago como origen de la NC (máximo 5 años de antigüedad).
- Dos modalidades de captura: por partidas (devolución de mercancía total o parcial, con cálculo automático del monto) y manual (descuento, bonificación o error, con captura libre de monto y concepto con materialidad fiscal).
- Cancelación condicional de la factura origen ante el SAT, ofrecida solo cuando la NC es por totalidad y dentro del mismo mes calendario de emisión. La cancelación es siempre total, nunca parcial.
- Armado del XML conforme a CFDI 4.0 vigente (TipoDeComprobante=E, TipoRelacion=01 con UUID de la factura origen, UsoCFDI=G02 default, MetodoPago=PUE fijo, FormaPago heredada de la factura origen pagada).
- Timbrado vía PAC TurboPac con feedback visual (en progreso / éxito / error) y previsualización del PDF antes de timbrar.
- Envío automático del correo al cliente al timbrar y opción de reenvío posterior, con PDF y XML adjuntos.
- Soporte multi-divisa (MXN/USD/EUR) preservando el tipo de cambio del día del timbrado.
- Foliado interno consecutivo continuo por empresa del grupo PROQUIFA México y UUID asignado por el SAT.
- Persistencia y conservación de los XML timbrados por mínimo 5 años (Art. 30 CFF).
- Acoplamiento uni-direccional con Validar Cobro: las NCs vigentes quedan disponibles automáticamente en el Paso 2 del wizard de Validar Cobro para aplicación a cobros nuevos del mismo cliente.
- Aplica a Región México; la estructura funcional se reutiliza para Perú con diferencias significativas, documentada en requisito independiente.

## No aplica a
- Módulo NC para Región Perú (requisito independiente, con catálogos SUNAT, sin CFDI/SAT ni PAC TurboPac).
- NCs para clientes de CRÉDITO (fuera de scope R16; potencial extensión en futuras releases).
- NCs históricas pre-go-live: no se importan desde Legacy ni sistemas anteriores. Solo viven en PQF2 las NCs generadas a partir del go-live; las históricas se gestionan por fuera del sistema durante la transición.
- Flujo de Devolución de Dinero al cliente: no contemplado en R16 (Política #1 PROQUIFA: retener al cliente con saldo a favor vía NC). Queda en stand-by como flujo alterno separado para evaluación posterior.
- Cancelación parcial de facturas: las facturas se cancelan siempre en su totalidad.
- Diseño del PDF representativo de la NC (layout, tipografía, secciones, branding, formato visual): se documenta en requisito independiente (análogo a TPSC-RE-FU-021 Factura México diseño PDF). Esta fila cubre solo las funciones y la información del módulo.
- Generación o cancelación de NCs desde Validar Cobro (el acoplamiento es uni-direccional: Validar Cobro no genera ni cancela NCs).
- Políticas de autorización por monto (umbrales que requieran autorización de coordinador o director): ** PMO #54. **"	"## Reglas de Negocio

Regla 1 — Módulo independiente operado por Tesorería para prepago México
El módulo NC para Región México es independiente de Validar Cobro (módulos distintos; NC alimenta a Validar Cobro pero no es sub-módulo), operado por el área de Tesorería, y aplica a clientes prepago en este release. La estructura funcional se reutiliza para Región Perú con sus diferencias específicas, documentada en requisito independiente.

Regla 2 — Política #1 PROQUIFA: NC antes que devolución de dinero
Ante un caso resoluble con NC o con devolución de dinero, la política es siempre intentar la NC primero para retener al cliente con saldo a favor. La devolución de dinero es un flujo alterno separado, no contemplado en este módulo ni en el alcance de R16.

Regla 3 — Solo facturas vigentes prepago pueden originar NC
El Paso 1 muestra como candidatas únicamente facturas en estado Vigente (no canceladas SAT) de clientes prepago, con un máximo de 5 años de antigüedad.

Regla 4 — Selección por partidas para devolución de mercancía
En la modalidad por partidas, el usuario ajusta la Cant. NC por partida (0, parcial o total) y el sistema calcula automáticamente el monto en tiempo real.

Regla 5 — Modalidad manual para bonificación/descuento
En la modalidad manual, el usuario captura libremente el monto y el concepto descriptivo de la materialidad fiscal. No hay tabla de partidas.

Regla 6 — Campos fiscales fijos del XML
Al armar el XML, el sistema fija: TipoDeComprobante=E, CfdiRelacionados TipoRelacion=01 con UUID de la factura origen, UsoCFDI receptor=G02 por default, y MetodoPago=PUE fijo inmutable según norma SAT.

Regla 7 — FormaPago heredada de factura origen pagada
El FormaPago de la NC se hereda de la factura origen pagada (en R16 prepago, típicamente 03 - Transferencia electrónica). No se usa 99 - Por definir (esa forma solo aplica a PPD). ** Confirmar el comportamiento del FormaPago en modalidad manual. **

Regla 8 — Cancelación condicional de factura origen (totalidad + mismo mes)
Cuando la NC es por totalidad y la fecha actual está dentro del mismo mes calendario de emisión de la factura, el sistema ofrece en el Paso 2 la opción de cancelar también la factura ante el SAT con motivo del catálogo SAT c_MotivoCancelacion. La cancelación es siempre total. La restricción del mismo mes es política interna PROQUIFA para optimización del IVA mensual.

Regla 9 — Foliado consecutivo por empresa con serie distintiva
El folio interno de la NC es consecutivo continuo por empresa del grupo PROQUIFA México, con serie ""P2"" propuesta. El UUID lo asigna el SAT al timbrar. ** Esquema del foliador con serie ""P2"" pendiente de validar. **

Regla 10 — Límite de 5 años para generar NC
El Paso 1 lista como candidatas únicamente facturas de hasta 5 años de antigüedad (obligación fiscal SAT).

Regla 11 — NCs históricas no se importan
Las NCs históricas pre-implementación no se importan a PQF2 desde Legacy. Solo viven en PQF2 las NCs generadas a partir del go-live.

Regla 12 — Inmutabilidad post-timbrado
Una NC timbrada no puede modificarse en su contenido; el sistema no permite la edición.

Regla 13 — NCs disponibles para aplicación en Validar Cobro
Una NC vigente generada queda disponible en el Paso 2 del wizard de Validar Cobro del mismo cliente para asociación a cobros nuevos. El acoplamiento es uni-direccional: NC alimenta a Validar Cobro, pero Validar Cobro no genera ni cancela NCs.

Regla 14 — Conservación XML 5 años
El XML timbrado de la NC se conserva por un mínimo de 5 años (Art. 30 CFF).

Regla 15 — Destinatarios del correo de envío de la NC
Al enviar el correo de la NC timbrada al cliente, el sistema arma los destinatarios: Para con el contacto del cliente del flujo (editable); CC con el ESAC asignado al cliente más el analista de Cuentas por Cobrar (editable).

Regla 16 — Manejo de errores PAC TurboPac
Cuando el timbrado falla, el sistema muestra un mensaje de error con el detalle del problema (mismo comportamiento que Factura por Adelantado y Validar Cobro); la NC no se persiste y el usuario puede reintentar.

## Riesgos

Riesgo 1 — Dependencia del PAC TurboPac
Si TurboPac está indisponible, el timbrado se bloquea y las NCs no llegan a Validar Cobro.

Riesgo 2 — Restricción ""mismo mes"" para cancelar factura
La política PROQUIFA del mismo mes es interna, no normativa SAT (el SAT permite cancelar dentro de todo el ejercicio fiscal). Si el cliente requiere cancelar una factura en un mes posterior, deberá gestionarlo por fuera del sistema.

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — PANTALLA PRINCIPAL DE CONSULTA (AGRUPADA POR CLIENTE)
═══════════════════════════════════════════════════════════════

Criterio A1 — Vista agrupada por cliente
Dado que el usuario accede al módulo,
Cuando el sistema renderiza la pantalla principal,
Entonces deberá mostrar la lista de clientes con NCs, con columnas Cliente, Total NC, Vigentes, Por Aplicar, Monto total de Notas de Crédito, Moneda del monto total. Adicionalmente deberá mostrar agregados al pie (total de clientes y total de NCs en el sistema).

Criterio A2 — Filtros del listado principal
Dado que el usuario está en la pantalla principal,
Cuando aplica filtros,
Entonces el sistema deberá ofrecer Cliente, Moneda y Fecha.

Criterio A3 — Acción nueva NC
Dado que el usuario está en la pantalla principal,
Cuando presiona la acción de nueva NC,
Entonces el sistema deberá invocar el wizard de generación de 4 pasos.

Criterio A4 — Drill-down al detalle por cliente
Dado que el usuario hace clic en un cliente del listado,
Cuando el sistema procesa la acción,
Entonces deberá navegar al detalle del cliente con su listado individual de NCs.

═══════════════════════════════════════════════════════════════
SECCIÓN B — DETALLE POR CLIENTE (DRILL-DOWN)
═══════════════════════════════════════════════════════════════

Criterio B1 — Filtros del detalle por cliente
Dado que el usuario está en el detalle de un cliente,
Cuando aplica filtros,
Entonces el sistema deberá ofrecer Fecha, Emisor, Estado, Moneda y buscador por folio o UUID.

Criterio B2 — Columnas del detalle por cliente
Dado que el usuario está en el detalle,
Cuando el sistema renderiza el listado,
Entonces deberá mostrar columnas: Fecha, Cobrador, Folio de la NC (acción → abre PDF), XML (acción → descarga archivo), Emisor, Monto con moneda, Factura asociada, Pedido Interno Asociado, Estado, Factura destino, Pedido destino.

Criterio B3 — Acceso al detalle completo de NC
Dado que el usuario hace clic en una NC del listado detalle,
Cuando el sistema procesa la acción,
Entonces deberá abrir la vista del Paso 4 NC Emitida con todos los datos de la NC.

═══════════════════════════════════════════════════════════════
SECCIÓN C — WIZARD PASO 1: BUSCAR FACTURA
═══════════════════════════════════════════════════════════════

Criterio C1 — Selección obligatoria de cliente
Dado que el usuario entra al Paso 1,
Cuando intenta listar facturas sin seleccionar cliente,
Entonces el sistema no deberá mostrar el listado de facturas hasta que se seleccione cliente.

Criterio C2 — Solo facturas vigentes prepago
Dado que se seleccionó cliente,
Cuando el sistema lista facturas,
Entonces solo deberá mostrar facturas en estado Vigente (no canceladas SAT) de prepago del cliente, con antigüedad máxima de 5 años.

Criterio C3 — Filtros del Paso 1
Dado el listado de facturas en el Paso 1,
Cuando el usuario aplica filtros,
Entonces el sistema deberá ofrecer Fecha, Moneda y buscador por folio o UUID.

Criterio C4 — Datos visibles por factura
Dado el listado de facturas,
Cuando el sistema renderiza,
Entonces deberá mostrar por factura: Folio, UUID SAT, Fecha de emisión, Total con moneda.

Criterio C5 — Selección de una sola factura
Dado el listado de facturas,
Cuando el usuario selecciona,
Entonces solo deberá poder seleccionar UNA factura por NC (no múltiple).

═══════════════════════════════════════════════════════════════
SECCIÓN D — WIZARD PASO 2: CAPTURAR DATOS (COMÚN A AMBAS MODALIDADES)
═══════════════════════════════════════════════════════════════

Criterio D1 — Datos de factura origen en solo lectura
Dado que el usuario entra al Paso 2,
Cuando el sistema renderiza la cabecera,
Entonces deberá mostrar de la factura origen (no editables): Folio, UUID, Tipo CFDI, RFC receptor, Razón Social, Fecha, Subtotal, IVA, Total con moneda, Estado.

Criterio D2 — Motivo desde catálogo SAT
Dado el Paso 2,
Cuando el usuario captura Motivo,
Entonces el sistema deberá ofrecer un combo con motivos principales: Devolución de mercancía (modalidad por partidas) y Descuento o bonificación (modalidad manual).

Criterio D3 — Bifurcación de modalidad según motivo
Dado la selección del Motivo,
Cuando el sistema renderiza el Paso 2,
Entonces deberá bifurcar la captura: si el motivo es Devolución de mercancía, modalidad por partidas; si el motivo es Descuento o bonificación, modalidad manual.

Criterio D4 — Uso CFDI G02 default
Dado el Paso 2,
Cuando el sistema renderiza el campo Uso CFDI,
Entonces el valor por default deberá ser G02 (Devoluciones, descuentos o bonificaciones).

Criterio D5 — Tipo de Relación 01 fijo
Dado el Paso 2,
Cuando el sistema renderiza el campo Tipo de Relación,
Entonces deberá mostrar 01 (Nota de crédito de los documentos relacionados) fijo no editable.

═══════════════════════════════════════════════════════════════
SECCIÓN E — MODALIDAD POR PARTIDAS (DEVOLUCIÓN DE MERCANCÍA)
═══════════════════════════════════════════════════════════════

Criterio E1 — Tabla heredada con Cant. NC editable
Dado motivo = Devolución de mercancía,
Cuando el sistema renderiza el Paso 2,
Entonces deberá mostrar la tabla de partidas heredada de la factura origen con columnas: Código, Producto, Cant. Facturada, Cant. NC (editable), Precio Unitario, Subtotal, IVA, Total por línea.

Criterio E2 — Captura de cantidad por partida
Dado la tabla de partidas,
Cuando el usuario edita Cant. NC,
Entonces deberá poder capturar 0 (no incluida), parcial (entre 0 y Cant. Facturada) o total (igual a Cant. Facturada).

Criterio E3 — Cálculo automático del Total NC
Dado que el usuario edita Cant. NC,
Cuando el sistema recalcula,
Entonces deberá actualizar Subtotal/IVA/Total por línea y Total NC al pie de la tabla en tiempo real.

Criterio E4 — XML hereda datos fiscales del concepto original por cada partida seleccionada
Dado modalidad por partidas,
Cuando el sistema arma el XML para timbrar,
Entonces por cada partida con Cant. NC > 0 deberá heredar del concepto original de la factura origen: ClaveProdServ, ClaveUnidad, NoIdentificacion, ValorUnitario, Descripción y configuración de impuestos (tasa IVA aplicable). Los importes y cantidades se recalculan según la Cant. NC capturada.

Criterio E5 — Concepto de la NC compuesto por partidas seleccionadas
Dado modalidad por partidas,
Cuando el sistema arma el contenido de la NC (XML y PDF representativo),
Entonces el concepto deberá componerse a partir de las partidas seleccionadas con Cant. NC > 0 (no de la totalidad de las partidas de la factura origen). Cada partida seleccionada genera un nodo Concepto en el XML con: ClaveProdServ, ClaveUnidad, NoIdentificacion heredados del concepto original de la factura; Cantidad = Cant. NC capturada por el usuario; ValorUnitario heredado; Importe = Cant. NC × ValorUnitario; Descripción del producto heredada; impuestos trasladados recalculados sobre el nuevo importe. Las partidas con Cant. NC = 0 no se incluyen en la NC.

═══════════════════════════════════════════════════════════════
SECCIÓN F — MODALIDAD MANUAL (DESCUENTO / BONIFICACIÓN)
═══════════════════════════════════════════════════════════════

Criterio F1 — No mostrar tabla de partidas
Dado motivo = Descuento o bonificación,
Cuando el sistema renderiza el Paso 2,
Entonces no deberá mostrar la tabla de partidas.

Criterio F2 — Captura libre del Monto Total NC
Dado modalidad manual,
Cuando el usuario captura el monto,
Entonces el sistema deberá ofrecer un campo editable para el Monto Total NC en la moneda de la factura origen. El monto no debe ser mayor al Total de la factura origen.

Criterio F3 — Concepto obligatorio con materialidad fiscal
Dado modalidad manual,
Cuando el usuario intenta avanzar al Paso 3,
Entonces el sistema deberá requerir que el campo Concepto esté lleno (texto libre describiendo la materialidad fiscal del descuento o bonificación).

Criterio F4 — Observaciones opcionales
Dado modalidad manual,
Cuando el usuario captura,
Entonces el sistema deberá ofrecer un campo opcional de Observaciones adicionales.

Criterio F5 — ClaveProdServ y ClaveUnidad default modalidad manual
Dado modalidad manual,
Cuando el sistema arma el XML,
Entonces deberá usar por default ClaveProdServ=84111506 (Servicios de facturación) y ClaveUnidad=ACT (Actividad). ** Confirmar las claves con el cliente. **

═══════════════════════════════════════════════════════════════
SECCIÓN G — CANCELACIÓN CONDICIONAL DE FACTURA ORIGEN
═══════════════════════════════════════════════════════════════

Criterio G1 — Opción aparece solo si totalidad + mismo mes
Dado modalidad por partidas,
Cuando todas las Cant. NC son iguales a Cant. Facturada (NC por totalidad) y la fecha actual está dentro del mismo mes calendario de emisión de la factura,
Entonces el sistema deberá mostrar la opción ""Cancelar Factura"" con combo de Motivo de cancelación SAT.

Criterio G2 — Opción no aparece si parcial o mes anterior
Dado NC parcial o factura origen en mes anterior al actual,
Cuando el sistema renderiza el Paso 2,
Entonces no deberá mostrar la opción de cancelar la factura.

Criterio G3 — Motivo de cancelación SAT obligatorio
Dado que el usuario activa cancelar factura,
Cuando intenta avanzar al Paso 3,
Entonces el sistema deberá requerir la selección del Motivo de cancelación desde el catálogo SAT c_MotivoCancelacion.

Criterio G4 — Cancelación siempre total
Dado que el usuario activa cancelar factura,
Cuando el sistema ejecuta la cancelación al timbrar,
Entonces deberá cancelar la factura origen en su totalidad (nunca parcialmente).

═══════════════════════════════════════════════════════════════
SECCIÓN H — CAMPOS FISCALES DEL XML
═══════════════════════════════════════════════════════════════

Criterio H1 — Campos obligatorios fijos
Dado la generación del XML,
Cuando el sistema arma el CFDI,
Entonces los campos fijos serán: TipoDeComprobante=E (Egreso), CfdiRelacionados con TipoRelacion=01 + UUID factura origen, MetodoPago=PUE (Pago en Una Exhibición).

Criterio H2 — UsoCFDI receptor G02 default
Dado la generación del XML,
Cuando el sistema asigna UsoCFDI del receptor,
Entonces deberá usar G02 (Devoluciones, descuentos o bonificaciones) por default.

Criterio H3 — FormaPago heredada de factura origen pagada
Dado NC en R16 prepago,
Cuando el sistema asigna FormaPago,
Entonces deberá heredar de la factura origen pagada (típicamente 03 - Transferencia electrónica). Si la factura origen estuviera no pagada (escenario raro en R16), sería 15 - Condonación. ** Confirmar el comportamiento en modalidad manual. **

Criterio H4 — Moneda heredada de factura origen
Dado la generación del XML,
Cuando el sistema asigna Moneda,
Entonces deberá heredar de la factura origen (no editable).

Criterio H5 — TipoCambio del día capturado
Dado moneda extranjera,
Cuando el sistema timbra,
Entonces deberá capturar el TipoCambio del día del timbrado y preservarlo en el XML.

Criterio H6 — Régimen fiscal emisor y receptor
Dado la generación del XML,
Cuando el sistema asigna régimen fiscal,
Entonces el emisor será 601 (Personas Morales) según la empresa emisora; el receptor heredará el régimen fiscal capturado del receptor.

Criterio H7 — Impuestos trasladados calculados automáticamente
Dado la generación del XML,
Cuando el sistema calcula impuestos,
Entonces deberá calcular automáticamente IVA al 16% o la tasa correspondiente al producto.

═══════════════════════════════════════════════════════════════
SECCIÓN I — WIZARD PASO 3: CONFIRMAR + PREVISUALIZACIÓN
═══════════════════════════════════════════════════════════════

Criterio I1 — Resumen completo de la NC
Dado el Paso 3,
Cuando el sistema renderiza,
Entonces deberá mostrar datos del emisor, datos del receptor, CFDI relacionado (folio + UUID factura origen), motivo, partidas o concepto manual según modalidad, e importes (subtotal, IVA, total con moneda y TC).

Criterio I2 — Indicador de cancelación de factura origen
Dado que se eligió cancelar factura,
Cuando el sistema renderiza el Paso 3,
Entonces deberá mostrar un indicador visible de la cancelación más el motivo SAT seleccionado.

Criterio I3 — Previsualización del PDF antes de timbrar
Dado el Paso 3,
Cuando el sistema renderiza,
Entonces el usuario deberá poder previsualizar el PDF de la NC tal como será timbrado, para validación visual previa al timbrado (consistente con Factura por Adelantado y Validar Cobro).

Criterio I4 — Regreso al Paso 2 sin pérdida de datos
Dado el Paso 3,
Cuando el usuario regresa al Paso 2,
Entonces el sistema deberá conservar los datos capturados (modalidad, motivo, partidas/cantidades, monto, concepto, observaciones, opción de cancelar factura).

Criterio I5 — Advertencia de irreversibilidad
Dado el Paso 3,
Cuando el sistema renderiza,
Entonces deberá mostrar una advertencia visible de que el timbrado es irreversible y generará un CFDI ante el SAT.

Criterio I6 — Acción Timbrar dispara envío al PAC
Dado el Paso 3,
Cuando el usuario presiona Timbrar,
Entonces el sistema deberá enviar el XML al PAC TurboPac.

═══════════════════════════════════════════════════════════════
SECCIÓN J — TIMBRADO Y FEEDBACK
═══════════════════════════════════════════════════════════════

Criterio J1 — Feedback de progreso
Dado que el usuario confirmó el timbrado,
Cuando el sistema envía el XML a TurboPac,
Entonces deberá mostrar al usuario feedback visual de progreso.

Criterio J2 — Persistencia al éxito
Dado timbrado exitoso,
Cuando el sistema recibe el XML timbrado con sello SAT,
Entonces deberá persistir el XML y el PDF en PQF2 y mostrar feedback de éxito.

Criterio J3 — Envío automático del correo al timbrar
Dado timbrado exitoso,
Cuando el sistema confirma,
Entonces deberá enviar automáticamente correo al cliente con PDF y XML adjuntos: Para = contacto del cliente vinculado a la factura, CC = ESAC asignado + analista de Cuentas por Cobrar.

Criterio J4 — Navegación automática al Paso 4
Dado timbrado y envío exitosos,
Cuando el sistema cierra el ciclo,
Entonces deberá navegar automáticamente al Paso 4 NC Emitida con todos los datos del CFDI timbrado.

Criterio J5 — Manejo de errores PAC
Dado timbrado fallido,
Cuando el sistema procesa la respuesta,
Entonces deberá mostrar un mensaje de error con detalle (PAC indisponible, datos inválidos, errores SAT) consistente con Factura por Adelantado y Validar Cobro. La NC no se persiste como vigente y el usuario puede reintentar.

═══════════════════════════════════════════════════════════════
SECCIÓN K — PASO 4: NC EMITIDA (= VISTA DE DETALLE DE NC)
═══════════════════════════════════════════════════════════════

Criterio K1 — Información mostrada
Dado timbrado exitoso (o consulta de NC desde listado),
Cuando el sistema renderiza la vista,
Entonces deberá mostrar: UUID, datos del PAC (número de certificado SAT, RFC del SAT), Folio interno, Serie, Tipo (E - Egreso), Versión CFDI 4.0, Motivo, Tipo de Relación (01), Factura Original (folio + UUID con indicación si fue cancelada SAT), Receptor, Estado SAT (Vigente / Cancelada), importes (Subtotal NC, IVA, Total NC con moneda), partidas o concepto manual según modalidad, y nota legal de conservación 5 años (Art. 30 CFF).

Criterio K2 — Banner de estado de la operación
Dado la vista de NC Emitida,
Cuando el sistema renderiza,
Entonces deberá mostrar un banner del estado: NC timbrada exitosamente; si además se canceló la factura origen, un indicador adicional con el motivo SAT de cancelación.

Criterio K3 — Acciones disponibles en Paso 4
Dado la vista de NC Emitida,
Cuando el sistema renderiza,
Entonces deberá ofrecer las acciones: Descargar XML, Descargar PDF, Reenviar por correo, Volver al listado.

Criterio K4 — Misma vista para consulta de NC ya generada
Dado que el usuario consulta una NC desde el listado detalle por cliente,
Cuando el sistema renderiza el detalle,
Entonces deberá presentar exactamente la misma vista que el Paso 4 NC Emitida (consistencia de vista de detalle).

═══════════════════════════════════════════════════════════════
SECCIÓN L — REENVÍO DE CORREO
═══════════════════════════════════════════════════════════════

Criterio L1 — Destinatarios pre-poblados editables
Dado el modal de envío del correo de la NC,
Cuando el sistema lo renderiza,
Entonces deberá pre-poblar: Para con el contacto del cliente del flujo (editable); CC con el ESAC asignado más el analista de Cuentas por Cobrar (editable).

Criterio L2 — Adjuntos automáticos
Dado el flujo de envío,
Cuando el sistema arma el correo,
Entonces deberá incluir automáticamente el PDF y el XML de la NC como adjuntos.

Criterio L3 — Asunto pre-poblado
Dado el flujo de envío,
Cuando el sistema arma el asunto,
Entonces deberá incluir Nota de Crédito + folio NC + folio de la factura relacionada. ** Confirmar plantilla final (PMO #31). **

Criterio L4 — Campo de mensaje libre editable
Dado el flujo de envío,
Cuando el sistema renderiza el modal,
Entonces deberá ofrecer un campo de mensaje libre editable por el usuario.

═══════════════════════════════════════════════════════════════
SECCIÓN M — ACOPLAMIENTO CON VALIDAR COBRO
═══════════════════════════════════════════════════════════════

Criterio M1 — NCs vigentes disponibles automáticamente en Validar Cobro
Dado que se generó una NC vigente en el módulo NC,
Cuando se opera el Paso 2 del wizard de Validar Cobro con el mismo cliente,
Entonces la NC deberá aparecer disponible para asociación a cobros nuevos.

Criterio M2 — Acoplamiento uni-direccional
Dado la arquitectura del módulo,
Cuando se opera Validar Cobro,
Entonces Validar Cobro no deberá poder generar ni cancelar NCs.

═══════════════════════════════════════════════════════════════
SECCIÓN N — MULTI-DIVISA, FOLIADO, DECIMALES
═══════════════════════════════════════════════════════════════

Criterio N1 — Foliado consecutivo por empresa
Dado la generación de NC,
Cuando el sistema asigna folio interno,
Entonces deberá ser consecutivo continuo por empresa del grupo PROQUIFA México con serie ""P2"" propuesta. ** Esquema del foliador pendiente de validar. **

Criterio N2 — UUID asignado por SAT
Dado timbrado,
Cuando el sistema recibe respuesta del SAT vía TurboPac,
Entonces deberá registrar el UUID único asignado por el SAT en el XML timbrado.

Criterio N3 — Precisión interna de cálculo
Dado cálculos del sistema,
Cuando ejecuta operaciones,
Entonces deberá usar internamente de 6 a 8 decimales para los cálculos.

Criterio N4 — Decimales mostrados al usuario
Dado el renderizado de importes,
Cuando el sistema muestra montos al usuario,
Entonces deberá mostrar un máximo de 4 decimales (ajuste sobre el estándar actual de 2).

Criterio N5 — Reglas SAT de decimales
Dado el timbrado,
Cuando el sistema arma el XML,
Entonces deberá respetar las reglas específicas SAT sobre decimales en CFDI 4.0. ** PMO #55. **

Criterio N6 — Conservación XML 5 años
Dado timbrado exitoso,
Cuando el sistema persiste,
Entonces deberá conservar el XML por un mínimo de 5 años (Art. 30 CFF)."								"- Fila documenta el módulo NC completo para Región México. Perú en requisito independiente.
- Wizard de 4 pasos: 1. Buscar Factura, 2. Capturar Datos, 3. Confirmar (con previsualización del PDF antes de timbrar), 4. NC Emitida. El Paso 4 es la misma vista que el detalle de cualquier NC ya generada consultada desde el listado.
- Pantalla principal agrupada por cliente con drill-down al detalle por cliente.
- Dos modalidades de captura: por partidas (devolución de mercancía total o parcial) y manual (descuento o bonificación).
- El concepto del CFDI en modalidad por partidas se compone únicamente de partidas con Cant. NC > 0 (no de la totalidad de partidas de la factura origen).
- Cancelación condicional de factura origen ofrecida solo si la NC es por totalidad y dentro del mismo mes calendario de emisión. Cancelación siempre total, nunca parcial.
- La restricción ""mismo mes"" para cancelar factura es política interna PROQUIFA (no normativa SAT estricta; el SAT permite cancelar dentro de todo el ejercicio fiscal). Razón: optimización mensual de IVA.
- Cumplimiento fiscal SAT validado contra Apéndice 5 Anexo 20 vigente: TipoDeComprobante=E, TipoRelacion=01 con UUID factura origen, UsoCFDI=G02 default, MetodoPago=PUE fijo (regla SAT inmutable, las NCs siempre son PUE), FormaPago heredada de factura origen pagada (típicamente 03 - Transferencia electrónica).
- NCs históricas pre-go-live no se importan desde Legacy ni sistemas anteriores. Solo viven en PQF2 las NCs generadas a partir del go-live.
- Multi-divisa MXN/USD/EUR: hereda Moneda de la factura origen, preserva TipoCambio del día del timbrado en el XML.
- Foliado interno consecutivo continuo por empresa del grupo PROQUIFA México con serie ""P2"" propuesta (pendiente validación final).
- Acoplamiento uni-direccional con Validar Cobro: las NCs vigentes quedan disponibles automáticamente en el Paso 2 del wizard de Validar Cobro para aplicación a cobros nuevos. Validar Cobro no genera ni cancela NCs.
- El diseño del PDF representativo de la NC se documenta en requisito independiente (análogo a TPSC-RE-FU-021 Factura México). Esta fila cubre solo funciones e información del módulo.
- ** Pendiente (alta): FormaPago en modalidad manual. El mockup muestra ""99 - Por definir"", que es fiscalmente incorrecto (99 solo aplica a PPD, y la NC es PUE). Confirmar el valor correcto. **
- ** Pendiente (media): ClaveProdServ y ClaveUnidad en modalidad manual. Recomendación SAT 84111506/ACT; PROQUIFA puede preferir otra clave. Confirmar. **
- ** Pendiente: PMO #31, plantilla del cuerpo del correo de envío (transversal Proforma/Factura/NC/Complemento/Inconsistencia). **
- ** Pendiente: PMO #54, políticas de autorización por monto. **
- ** Pendiente: esquema definitivo del foliador con serie ""P2"". **
- ** Pendiente: denominación canónica del rol operativo (transversal con Validar Cobro y Factura por Adelantado). **"									