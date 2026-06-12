# R16A-RE-FU-019 — Factura por Adelantado: Detalle México

| Campo | Valor |
|-------|-------|
| **ID** | R16A-RE-FU-019 |
| **Título** | Factura por Adelantado: Detalle México |
| **Módulo / Épica** | Factura por Adelantado |
| **Historia de Usuario** | Yo como ** Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver) **, quiero contar con una pantalla de detalle por cliente en Factura por Adelantado que me permita generar, timbrar y enviar la factura de cada pedido pendiente, para emitir oportunamente las facturas por adelantado y dar continuidad al flujo de cobro o transferencia a Legacy según el tipo de pedido. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.2M-RE-FU-001 |

---

## Requisito Funcional

El sistema debe contar con una pantalla de Detalle por cliente en el módulo Factura por Adelantado que muestre los datos del cliente y el listado de sus pedidos pendientes de facturar por adelantado, ofreciendo por cada pedido la acción de generar la factura o, si ya fue timbrada, de enviarla al cliente. El flujo de generación contempla la revisión de los datos fiscales, la previsualización del PDF y el timbrado ante el SAT; tras un envío exitoso el pedido sale del listado de pendientes. La salida operativa posterior depende del tipo de pedido: los de Crédito continúan por el flujo Crédito y los de Prepago generan el pendiente correspondiente en Validar Cobro. Esta funcionalidad no aplica a pedidos que contengan productos clasificados como Sustancias Controladas (tipo Mundial, Nacional u Origen).

---

## Alcance

### Aplica a

- Pantalla de Detalle por cliente del módulo Factura por Adelantado.
- Listado de pedidos del cliente seleccionado con pendiente de generar o enviar Factura por Adelantado.
- Acciones contextuales por pedido: "Generar Factura" (pedido sin factura todavía) o "Enviar Factura" (pedido con factura ya generada pendiente de envío).
- Modal de Generación de Factura (revisión de datos fiscales del cliente, del emisor y del pedido; único campo editable: Uso CFDI).
- Modal de Previsualización del PDF de la Factura antes del timbrado SAT.
- Modal de Alerta SAT con descripción del error de validación o timbrado, cuando aplique.
- Modal de éxito de generación cuando el timbrado se completa exitosamente.
- Modal de Envío de Factura (destinatario, CC, asunto, adjuntos PDF + XML, notas del correo).
- Modal de éxito de envío cuando el correo se envía exitosamente.
- Cambio de estado del pedido en el listado conforme avanza el ciclo (pendiente generar → pendiente enviar → desaparece tras envío).
- Salida operativa del pedido tras envío exitoso: transferencia a Legacy para pedidos Crédito; generación del pendiente correspondiente en Validar Cobro para pedidos Prepago.
- Aplicación a clientes con Región México exclusivamente.

### No aplica a

- Pedidos que contengan productos clasificados como Sustancias Controladas tipo Mundial, Nacional u Origen. La Factura por Adelantado no es elegible para estos pedidos, independientemente del tipo (Crédito o Prepago).
- Pedidos para clientes con Región Perú. La habilitación para Perú depende de la resolución de la brecha de timbrado SUNAT/OSE y de las consideraciones fiscales peruanas; se documenta en requisito independiente cuando esté disponible.
- Generación del PDF de la Factura como artefacto (estructura visual, secciones, datos a renderizar): se documenta en requisito independiente, análogo al del PDF de Proforma.
- Reenvío de la Factura tras envío exitoso al cliente: no se ofrece funcionalidad de reenvío; la Factura queda persistida y consultable desde Validar Cobro.
- Edición de datos fiscales del cliente desde esta pantalla: esos datos se administran en el Catálogo de Clientes. Si hay error de validación SAT por datos del cliente (ejemplo: Código Postal), el usuario debe ir al Catálogo a corregir y volver a reintentar.

---

## Reglas de Negocio

Regla 1 — Estados del pedido visibles en el listado
Cada pedido del cliente seleccionado muestra una acción contextual a su estado: "Generar Factura" si el pedido aún no tiene factura emitida (estado pendiente generar), o "Enviar Factura" si la factura ya fue generada y timbrada pero está pendiente de envío al cliente (estado pendiente enviar). Una vez la factura se envía exitosamente, el pedido desaparece del listado.

Regla 2 — Datos del cliente en cabecera del Detalle
La cabecera del Detalle del cliente muestra: Razón Social del cliente, identificador fiscal (RFC), moneda de facturación y la clasificación crediticia del cliente si está disponible. Son datos preexistentes en el sistema, en modo solo lectura, no editables desde este módulo.

Regla 3 — Datos del pedido visibles en el listado
Cada pedido del cliente en el listado muestra: Pedido Interno, Fecha del pedido, Condiciones de Pago (ejemplo: "PREPAGO 100%", "30 DIAS", "60 DIAS", "90 DIAS"), Empresa Emisora del pedido, Subtotal, IVA y Monto Total en la moneda del pedido.

Regla 4 — Modal de Generación con Uso CFDI como único editable
Al presionar "Generar Factura" en un pedido, el modal de Generación de Factura muestra en modo solo lectura: cliente (Razón Social o Alias), Monto Total del pedido, Pedido Interno, Condiciones de Pago, datos del contacto del cliente (nombre, correo electrónico, teléfono), Datos Fiscales del Cliente (RFC, Razón Social, Código Postal, Régimen Fiscal, Correo electrónico, Moneda, Tipo de Cambio, Tipo de Comprobante, Método de Pago, Forma de Pago), Datos Fiscales del Emisor (RFC, Razón Social, Régimen Fiscal) y los Comentarios de Facturación. El único campo editable del modal es el Uso CFDI (combo de selección con catálogo SAT). ** Pendiente confirmar si el cliente se identifica por Razón Social o Alias. **

Regla 5 — Forma de Pago, Método de Pago y Tipo de Comprobante forzados por normativa SAT
Por ser la Factura por Adelantado una factura PPD, el modal de Generación presenta como valores forzados (solo lectura): Método de Pago = "PPD - Pago en parcialidades o diferido", Forma de Pago = "99 - Por definir", Tipo de Comprobante = "I - Ingreso". Estos valores son obligatorios por normativa SAT para facturas PPD y no son modificables por el usuario. La Forma de Pago real se captura posteriormente en el Complemento de Pago del módulo Validar Cobro.

Regla 6 — Tipo de Cambio del día de generación
Cuando el cliente factura en una moneda distinta a pesos mexicanos, el modal de Generación muestra el Tipo de Cambio aplicable al día de generación de la factura. El valor es solo lectura, no modificable por el usuario.

Regla 7 — Generación en dos pasos: revisar datos, previsualizar PDF, timbrar
Tras confirmar los datos en el modal de Generación de Factura, el sistema muestra el modal de Previsualización con el PDF de la Factura tal como saldrá al cliente. El usuario revisa el documento y confirma con "Generar Factura" desde el modal de Previsualización para proceder con el timbrado real ante el PAC SAT. La acción "Cancelar" en cualquiera de los dos modales aborta el flujo sin timbrar.

Regla 8 — Manejo de errores de validación SAT o de timbrado
Cuando el timbrado ante el PAC falla por un error de validación (ejemplo: el Código Postal del cliente no coincide con el registrado en el SAT) o por un error del propio PAC (errores de decimales en totales, indisponibilidad del servicio, otros), el sistema muestra el modal de Alerta SAT con la descripción específica del error y un botón "Continuar" que cierra el modal y devuelve al usuario al estado previo. El usuario debe corregir los datos en el módulo correspondiente (típicamente el Catálogo de Clientes para errores de datos fiscales del cliente) y reintentar la generación desde el principio. El sistema no ofrece edición directa de los datos del cliente desde este modal.

Regla 9 — Persistencia de la Factura al timbrarse exitosamente
Cuando el timbrado se completa exitosamente, el sistema persiste la Factura en base de datos como artefacto fiscal inmutable (incluye el documento timbrado completo: XML y representación visual PDF) y muestra al usuario el modal de confirmación de éxito de generación. A diferencia de la Proforma (que se persiste al envío), la Factura se persiste al timbrarse: una vez timbrada ya no se puede modificar.

Regla 10 — Folio de la Factura por empresa emisora
El folio de la Factura usa un contador consecutivo numérico independiente por empresa emisora (Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica). El folio de la Factura es de tipo varchar(6).

Regla 11 — Modal de Envío de Factura
Cuando la Factura fue timbrada exitosamente y el usuario presiona "Enviar Factura", el modal de Envío presenta: Para (destinatario) con el contacto del cliente del pedido, editable, con default heredado del flujo de tramitación; CC con el ESAC asignado al cliente/pedido, editable, con default sugerido por el sistema; Asunto generado automáticamente con los folios del documento, no editable; Adjuntos con el PDF y XML de la Factura timbrada, no editables; y Notas extras, un campo de texto editable opcional para texto adicional libre.

Regla 12 — Confirmación de envío exitoso y cierre del pendiente
Al confirmarse que el correo se envió exitosamente al cliente, el sistema muestra el modal de confirmación de éxito de envío, el pedido sale del listado del Detalle, y la factura queda persistida y consultable desde el módulo Validar Cobro.

Regla 13 — Salida operativa post-envío diferenciada por tipo de pedido
Tras el envío exitoso de la Factura por Adelantado, el sistema actúa diferenciadamente según el tipo del pedido: si es Crédito, la información de la Factura se transfiere a Legacy y el pedido continúa por el flujo Crédito en Legacy; si es Prepago, el sistema genera el pendiente correspondiente en el módulo Validar Cobro para que el equipo de Cobranza valide el pago del cliente contra esta Factura.

Regla 14 — Visibilidad filtrada por cartera del usuario
El acceso al Detalle de un cliente solo se permite si el cliente está asignado a la cartera del usuario (campo Cobrador del Catálogo de Clientes). Los clientes asignados a otros usuarios no son accesibles.

Regla 15 — Datos de la Factura provienen del Pedido
Los conceptos de la Factura se obtienen del Pedido: por cada partida, cantidad, descripción (catálogo + descripción + marca), precio unitario e importe. **La Factura por Adelantado no consigna lote ni pedimento.** El pedimento se representa como N/A (comportamiento preexistente del sistema); el lote tampoco se incluye porque el surtido del pedido (asignación de lote en almacén) ocurre después de cobrar y facturar. Decisión confirmada por el cliente — OBS-039.

---

## Riesgos

Riesgo 1 — Indisponibilidad del PAC TurboPac
Si el PAC TurboPac (RFC QSO100827UB0, Quadrum Tecnologías) está caído o responde con timeout, no se puede timbrar la Factura.

Riesgo 2 — Solapamiento de denominación de rol
La denominación del rol que opera este módulo aparece como "Gestor de Cobranza" en la matriz del cliente y como "Analista de Cuentas por Cobrar" en sesiones de revisión de pantallas Factura por Adelantado. ** Pendiente resolver formalmente la denominación canónica antes del desarrollo. **

Riesgo 3 — Brecha de timbrado para Perú
La habilitación para Región Perú depende de la integración con OSE/SUNAT autorizado, brecha mayor no resuelta del proyecto documentada en R16A-RE-FU-005 (Brecha 5). Mientras no se resuelva, los clientes Perú no pueden usar este módulo.

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL CLIENTE Y LISTADO DE PEDIDOS
═══════════════════════════════════════════════════════════════

Criterio A1 — Datos del cliente en cabecera
Dado que el usuario navega al Detalle de un cliente desde el listado del módulo,
Cuando el sistema renderiza la cabecera,
Entonces deberá mostrar Razón Social del cliente, identificador fiscal (RFC), moneda de facturación, y la clasificación preexistente del cliente si está disponible.

Criterio A2 — Listado de pedidos pendientes
Dado que el cliente tiene facturas pendientes de generar o enviar Factura por Adelantado,
Cuando el sistema renderiza el listado,
Entonces deberá mostrar una fila por cada pedido pendiente con: Pedido Interno, Fecha del pedido, Condiciones de Pago, Empresa Emisora del pedido, Subtotal, IVA, Monto Total y acción contextual.

Criterio A3 — Acción contextual por estado del pedido
Dado que un pedido del cliente aparece en el listado,
Cuando el sistema renderiza la acción del pedido,
Entonces deberá mostrar:
- "Generar Factura" si el pedido NO tiene factura emitida (estado: pendiente generar).
- "Enviar Factura" si el pedido tiene factura ya timbrada pero pendiente de envío (estado: pendiente enviar).

═══════════════════════════════════════════════════════════════
SECCIÓN B — MODAL DE GENERACIÓN DE FACTURA
═══════════════════════════════════════════════════════════════

Criterio B1 — Apertura del modal al presionar "Generar Factura"
Dado que el usuario presiona "Generar Factura" en un pedido del listado,
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal de Generación de Factura con los datos del pedido y del cliente cargados.

Criterio B2 — Datos del pedido en cabecera del modal
Dado que el modal de Generación se abre,
Cuando el sistema arma la cabecera del modal,
Entonces deberá mostrar: Razón Social del cliente, Monto Total del pedido (visualmente prominente), Pedido Interno, Condiciones de Pago.

Criterio B3 — Datos del contacto del cliente (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los datos del contacto del cliente,
Entonces deberá mostrar en modo solo lectura: nombre del contacto, correo electrónico, teléfono con extensión cuando aplique.

Criterio B4 — Datos Fiscales del Cliente (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los datos fiscales del cliente,
Entonces deberá mostrar en modo solo lectura: RFC, Razón Social, Código Postal, Régimen Fiscal, Correo electrónico, Moneda, Tipo de Cambio, Tipo de Comprobante (siempre "I - Ingreso"), Método de Pago (siempre "PPD - Pago en parcialidades o diferido"), Forma de Pago (siempre "99 - Por definir").

Criterio B5 — Datos Fiscales del Emisor (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los datos fiscales del emisor,
Entonces deberá mostrar en modo solo lectura los datos visibles al usuario:
- RFC del emisor.
- Emisor (identificador comercial: Golocaer, Mungen, Proquifa, Proveedora).
- Razón Social del emisor (razón social legal completa: Golocaer S.A. de C.V., Mungen S.A. de C.V., Proquifa S.A. de C.V., Proveedora Quimico Farmaceutica S.A. de C.V.).

Criterio B6 — Comentarios de Facturación (solo lectura)
Dado que el modal de Generación está abierto,
Cuando el sistema muestra los comentarios de facturación,
Entonces deberá mostrar el texto del campo Comentarios de Facturación del pedido en modo solo lectura.

Criterio B7 — Uso CFDI como único campo editable
Dado que el modal de Generación está abierto,
Cuando el usuario interactúa con el modal,
Entonces el único campo modificable deberá ser Uso CFDI (combo de selección con el catálogo SAT correspondiente). Todos los demás datos son solo lectura.

Criterio B8 — Acciones del modal: Cancelar y Generar Factura
Dado que el usuario revisa los datos en el modal,
Cuando finaliza la revisión,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el flujo, cierra el modal sin generar factura) y Generar Factura (procede al siguiente paso: previsualización del PDF).

═══════════════════════════════════════════════════════════════
SECCIÓN C — MODAL DE PREVISUALIZACIÓN DEL PDF
═══════════════════════════════════════════════════════════════

Criterio C1 — Apertura del modal de Previsualización tras confirmar el modal de Generación
Dado que el usuario presiona "Generar Factura" en el modal de Generación,
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal de Previsualización mostrando el PDF de la Factura tal como saldrá al cliente, con todos los datos integrados. En este momento la Factura aún NO se ha timbrado ante el PAC.

Criterio C2 — Acciones del modal: Cancelar y Generar Factura
Dado que el usuario revisa el PDF previsualizado,
Cuando finaliza la revisión,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el flujo, cierra el modal sin timbrar) y Generar Factura (procede con el timbrado real ante el PAC SAT).

═══════════════════════════════════════════════════════════════
SECCIÓN D — MODAL DE ALERTA SAT (errores de validación o timbrado)
═══════════════════════════════════════════════════════════════

Criterio D1 — Aparición del modal de Alerta ante error
Dado que el sistema intenta timbrar la Factura y recibe respuesta de error del PAC SAT (errores de validación previa o errores del propio servicio de timbrado),
Cuando el error se identifica,
Entonces deberá mostrar el modal de Alerta SAT con la descripción específica del error (ejemplo: "El código postal del cliente no coincide con el registrado en el SAT. Verifica y actualiza el código postal para poder timbrar").

Criterio D2 — Acción del modal: Continuar
Dado que el modal de Alerta está abierto,
Cuando el usuario lo cierra,
Entonces el sistema deberá cerrar el modal con la acción "Continuar" y devolver al usuario al estado previo (típicamente el listado del Detalle). El sistema NO ofrece edición directa de los datos del cliente desde este modal: el usuario debe ir al módulo correspondiente (Catálogo de Clientes) a corregir y reintentar la generación desde el principio.

═══════════════════════════════════════════════════════════════
SECCIÓN E — MODAL DE ÉXITO DE GENERACIÓN (timbrado exitoso)
═══════════════════════════════════════════════════════════════

Criterio E1 — Loader durante el timbrado
Dado que el usuario confirmó el timbrado desde el modal de Previsualización,
Cuando el sistema envía la solicitud de timbrado al PAC y espera respuesta,
Entonces deberá mostrar un indicador de carga (loader) con mensaje informativo (ejemplo: "Su solicitud está siendo atendida, por favor espere...").

Criterio E2 — Modal de confirmación de éxito
Dado que el timbrado se completa exitosamente,
Cuando el PAC retorna la confirmación,
Entonces el sistema deberá mostrar el modal de confirmación de éxito (ejemplo: "¡Has generado una factura Exitosamente!") y persistir la Factura en base de datos. El pedido cambia de estado en el listado: ahora muestra acción "Enviar Factura" (verde) en lugar de "Generar Factura" (azul).

Criterio E3 — Persistencia inmediata al timbrado exitoso
Dado que el timbrado fue exitoso,
Cuando el sistema procesa la respuesta del PAC,
Entonces deberá persistir el artefacto fiscal completo (XML timbrado y PDF) en base de datos como registro permanente inmutable. La Factura no puede modificarse después de este momento.

═══════════════════════════════════════════════════════════════
SECCIÓN F — MODAL DE ENVÍO DE FACTURA
═══════════════════════════════════════════════════════════════

Criterio F1 — Apertura del modal de Envío al presionar "Enviar Factura"
Dado que un pedido tiene Factura ya timbrada en estado pendiente enviar y el usuario presiona "Enviar Factura",
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal de Envío con los campos pre-rellenados.

Criterio F2 — Contacto destinatario pre-rellenado y editable
Dado que el modal de Envío está abierto,
Cuando el sistema muestra el campo destinatario,
Entonces deberá pre-rellenar el correo del contacto del cliente. El campo es editable por el usuario si fuese necesario modificar el destinatario.

Criterio F3 — CC editable con default ESAC
Dado el modal de envío, cuando el sistema renderiza el campo CC, entonces deberá pre-rellenarlo con el ESAC asignado al cliente/pedido como default, manteniéndolo editable por el usuario.

Criterio F4 — Asunto pre-rellenado con folios
Dado que el modal de Envío está abierto,
Cuando el sistema muestra el campo Asunto,
Entonces deberá pre-rellenar el asunto en un formato que incluya el folio de la Factura y el folio del Pedido Interno (formato canónico definido para correos de Factura de México). ** Para Perú, la lógica del asunto podría ser análoga, pero la numeración del folio fiscal es distinta (serie SUNAT F### + correlativo) y el formato final queda pendiente de confirmar con PROQUIFA y su asesor peruano. **

Criterio F5 — Adjuntos automáticos PDF y XML
Dado que el modal de Envío está abierto,
Cuando el sistema muestra los adjuntos,
Entonces deberá incluir automáticamente los archivos PDF y XML de la Factura timbrada. Los adjuntos NO son removibles por el usuario.

Criterio F6 — Notas extras editables
Dado el modal de envío, cuando el sistema renderiza el text area de notas extras, entonces deberá ser editable por el usuario para capturar texto adicional libre opcional.

Criterio F7 — Acciones del modal: Cancelar y Enviar
Dado que el usuario revisa el modal de Envío,
Cuando finaliza la edición,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el envío, cierra el modal; la Factura permanece timbrada y persistida con estado pendiente enviar) y Enviar (procede con el envío del correo al cliente).

═══════════════════════════════════════════════════════════════
SECCIÓN G — MODAL DE ÉXITO DE ENVÍO Y SALIDA OPERATIVA
═══════════════════════════════════════════════════════════════

Criterio G1 — Confirmación de envío exitoso
Dado que el sistema confirma que el correo de la Factura fue enviado exitosamente al cliente,
Cuando el envío se completa,
Entonces deberá mostrar el modal de confirmación de éxito (ejemplo: "¡Has enviado una factura Exitosamente!").

Criterio G2 — Salida del pedido del listado
Dado que el envío fue exitoso,
Cuando el sistema actualiza el estado del pedido,
Entonces el pedido deberá desaparecer del listado del Detalle del cliente. Si era el último pedido del cliente, el cliente puede salir también del listado agrupado de la pantalla inicial del módulo.

Criterio G3 — Salida operativa diferenciada por tipo de pedido
Dado que la Factura fue enviada exitosamente,
Cuando el sistema procesa la salida operativa,
Entonces deberá actuar diferenciadamente:
- Si el pedido es Crédito: la información de la Factura se transfiere a Legacy. El pedido continúa por el flujo Crédito en Legacy. NO se genera pendiente en Validar Cobro.
- Si el pedido es Prepago: el sistema genera el pendiente correspondiente en el módulo Validar Cobro asociado a esta Factura, para que el equipo de Cobranza valide el pago del cliente.

═══════════════════════════════════════════════════════════════
SECCIÓN H — ORDEN DEL FLUJO COMPLETO
═══════════════════════════════════════════════════════════════

Criterio H1 — Orden canónico del flujo
Dado que el usuario inicia el flujo completo de generar y enviar una Factura,
Cuando avanza paso a paso,
Entonces deberá seguirse el siguiente orden:
1. Usuario navega al Detalle del cliente y presiona "Generar Factura" en un pedido.
2. Sistema abre el modal de Generación con los datos del pedido y del cliente.
3. Usuario revisa, edita Uso CFDI si requiere, y presiona "Generar Factura" del modal.
4. Sistema abre el modal de Previsualización del PDF.
5. Usuario revisa el PDF previsualizado y presiona "Generar Factura" del modal de Previsualización.
6. Sistema envía la solicitud al PAC. Si hay error: muestra modal de Alerta SAT y termina aquí (el usuario corrige y vuelve a empezar). Si es exitoso: continúa.
7. Sistema muestra modal de loader durante el timbrado y luego modal de éxito de generación.
8. Sistema persiste la Factura en base de datos. El pedido cambia de estado a pendiente enviar.
9. Usuario regresa al listado y presiona "Enviar Factura" en el pedido.
10. Sistema abre el modal de Envío con datos pre-rellenados.
11. Usuario revisa, notas al correo si requiere, y presiona "Enviar".
12. Sistema envía el correo al cliente con adjuntos PDF y XML.
13. Sistema muestra modal de éxito de envío.
14. Pedido sale del listado. Salida operativa según tipo (Legacy para Crédito; pendiente en Validar Cobro para Prepago).

---

## Notas Adicionales

- Esta fila documenta la pantalla de Detalle por cliente del módulo Factura por Adelantado, junto con todos los modales del flujo: Generación de Factura, Previsualización del PDF, Alerta SAT, Éxito de generación, Envío de Factura y Éxito de envío. La pantalla inicial del módulo (listado agrupado por cliente) se documenta en requisito independiente.
- El módulo Factura por Adelantado es funcionalidad NUEVA en PQF2 R16. La incorporación de Factura por Adelantado atiende casos comerciales donde el cliente requiere la factura antes del ingreso de la mercancía.
- Aplica exclusivamente a pedidos SIN productos clasificados como Sustancias Controladas (tipo Mundial, Nacional u Origen). Los pedidos Prepago con controlados se atienden vía Factura Anticipo desde Validar Cobro (artefacto y flujo independientes). Los pedidos Crédito con controlados no son elegibles para Factura por Adelantado.
- Aplica a pedidos Crédito SIN controlados y Prepago SIN controlados. La diferenciación post-envío:
  - Crédito → transferencia a Legacy.
  - Prepago → pendiente en Validar Cobro.
- La generación de la Factura se realiza en DOS pasos: (1) modal de revisión de datos fiscales del cliente y del emisor con único editable Uso CFDI; (2) modal de previsualización del PDF antes del timbrado real ante el PAC SAT.
- El PAC utilizado por PROQUIFA es TurboPac (operado por Quadrum Tecnologías S.A. de C.V., RFC QSO100827UB0). El detalle técnico de la integración con el PAC se documenta en requisito independiente.
- El detalle del contenido y estructura del PDF de la Factura (qué información va, en qué secciones, branding por empresa emisora, etc.) se documenta en requisito independiente, análogo al del PDF de Proforma.
- Datos fiscales clave forzados por normativa SAT y no modificables:
  - Método de Pago = "PPD - Pago en parcialidades o diferido".
  - Forma de Pago = "99 - Por definir" (la Forma de Pago real se captura en el Complemento de Pago del módulo Validar Cobro).
  - Tipo de Comprobante = "I - Ingreso".
  - Tipo de Cambio = el del día de generación (regla SAT confirmada por el cliente, en factura no se permite modificar el TC).
- El folio de la Factura es un consecutivo numérico independiente POR empresa emisora (varchar 6). Esto contrasta con el folio de la Proforma que es global lineal para todo el grupo. Cada empresa del grupo PROQUIFA México (Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica) mantiene su propio contador.
- A diferencia de la Proforma (que se persiste al envío exitoso del correo), la Factura se persiste al timbrarse exitosamente ante el PAC. Una vez timbrada, la Factura es artefacto fiscal inmutable que no puede modificarse.
- Los errores más comunes reportados por PROQUIFA en el proceso de facturación electrónica actual y que deben manejarse en el modal de Alerta SAT son: (a) cambio de Código Postal del cliente respecto al registrado en el SAT (requiere corrección en el Catálogo de Clientes); (b) errores en decimales al calcular totales de la factura.
- El modal de Alerta SAT NO ofrece edición directa de los datos del cliente. El usuario debe corregir en el Catálogo de Clientes y reintentar la generación desde el principio. Esto preserva fuente única de verdad para los datos del cliente.
- El modal de Envío incluye copia automática a ESAC por regla canónica del proyecto.
- El asunto del correo de envío se pre-rellena con un formato canónico que incluye folio de la Factura y folio del Pedido Interno (México). ** Para Perú, el formato del asunto y la numeración del folio fiscal (serie SUNAT F### + correlativo, distinta del folio mexicano) quedan pendientes de confirmar con PROQUIFA y su asesor peruano cuando se habilite la región. **
- La sección Cliente del módulo (cabecera del Detalle) muestra etiquetas de clasificación del cliente que son datos preexistentes del Catálogo de Clientes y solo lectura desde este módulo.
- Cobertura geográfica de esta fila: clientes con Región México exclusivamente. La habilitación para Perú depende de la integración con OSE/SUNAT autorizado (brecha mayor del proyecto) y se documenta en requisito independiente cuando esté disponible.
- **Decisión OBS-039:** ninguna Factura por Adelantado consigna lote ni pedimento. El pedimento ya se manejaba como N/A (comportamiento preexistente); el lote sigue la misma decisión — el surtido (asignación de lote en almacén) ocurre después del cobro y la facturación, por lo que el lote no está disponible al timbrar. Decisión cerrada y confirmada por el cliente.
- ** Pendiente confirmar la política ante indisponibilidad del PAC TurboPac (reintento automático, encolamiento, fallback). **
- ** Pendiente resolver formalmente la denominación canónica del rol operativo entre "Gestor de Cobranza" y "Analista de Cuentas por Cobrar". **

---

## Cambios

| # | Fecha | Referencia | Descripción del cambio |
|---|-------|------------|------------------------|
| 1 | 2026-06-10 | OBS-039 | Regla 15: lote y pedimento excluidos de la descripción de conceptos de la Factura por Adelantado. Descripción actualizada a "catálogo + descripción + marca". Pendiente de lote cerrado — decisión: ninguna Factura por Adelantado consigna lote ni pedimento. |
