# R16A-RE-FU-019 — Factura por Adelantado: Detalle México

| Campo | Valor |
|-------|-------|
| **ID** | R16A-RE-FU-019 |
| **Título** | Factura por Adelantado: Detalle México |
| **Módulo / Épica** | Factura por Adelantado |
| **Historia de Usuario** | Yo como **Gestor de Cobranza**, quiero contar con una pantalla de detalle por cliente en Factura por Adelantado que me permita generar, timbrar y enviar la factura de cada pedido pendiente, para emitir oportunamente las facturas por adelantado y dar continuidad al flujo de cobro o transferencia a Legacy según el tipo de pedido. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.2M-RE-FU-001 |

---

## Requisito Funcional

El sistema debe contar con una pantalla de Detalle por cliente en el módulo Factura por Adelantado que muestre los datos del cliente y el listado de sus pedidos pendientes de facturar por adelantado, ofreciendo por cada pedido la acción de generar la factura o, si ya fue timbrada, de enviarla al cliente. El flujo de generación contempla la revisión de los datos fiscales, la validación previa al timbrado, la previsualización del PDF y el timbrado ante el SAT; tras un envío exitoso el pedido sale del listado de pendientes. La salida operativa posterior depende del tipo de pedido: los de Crédito y Pago contra entrega continúan por el flujo Crédito en Legacy, y los de Prepago generan el pendiente correspondiente en Validar Cobro. Esta funcionalidad no aplica a pedidos que contengan productos clasificados como Sustancias Controladas (tipo Mundial, Nacional u Origen).

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
- Salida operativa del pedido tras envío exitoso: transferencia a Legacy para pedidos Crédito y Pago contra entrega; generación del pendiente correspondiente en Validar Cobro para pedidos Prepago.
- Aplicación a clientes con Región México exclusivamente.

### No aplica a

- Pedidos que contengan productos clasificados como Sustancias Controladas tipo Mundial, Nacional u Origen. La Factura por Adelantado no es elegible para estos pedidos, independientemente del tipo (Crédito, Prepago o Pago contra entrega).
- Pedidos para clientes con Región Perú: quedan fuera del alcance de este release, al haberse retirado el timbrado de esa región (Decisión "Quitar Perú" 2026-07-17 / DUDA-049).
- Generación del PDF de la Factura como artefacto (estructura visual, secciones, datos a renderizar): se documenta en requisito independiente, análogo al del PDF de Proforma.
- Reenvío de la Factura tras envío exitoso al cliente: no se ofrece funcionalidad de reenvío; la Factura queda persistida y consultable desde Validar Cobro.
- Edición de datos fiscales del cliente desde esta pantalla: esos datos se administran en el Catálogo de Clientes. Si hay error de validación SAT por datos del cliente (ejemplo: Código Postal), el usuario debe ir al Catálogo a corregir y volver a reintentar.

---

## Reglas de Negocio

Regla 1 — Estados del pedido visibles en el listado
Cada pedido del cliente seleccionado muestra una acción contextual a su estado: "Generar Factura" si el pedido aún no tiene factura emitida (estado pendiente generar), o "Enviar Factura" si la factura ya fue generada y timbrada pero está pendiente de envío al cliente (estado pendiente enviar). Una vez la factura se envía exitosamente, el pedido desaparece del listado.

Regla 2 — Datos del cliente en cabecera del Detalle
La cabecera del Detalle del cliente muestra: Razón Social del cliente, identificador fiscal (RFC), moneda de facturación y la clasificación del cliente si está disponible. Son datos preexistentes del Catálogo de Clientes, en modo solo lectura, no editables desde este módulo.

Regla 3 — Datos del pedido visibles en el listado
Cada pedido del cliente en el listado muestra: Pedido Interno, Fecha del pedido, Condiciones de Pago (ejemplo: "PREPAGO 100%", "30 DIAS", "60 DIAS", "90 DIAS"), Empresa Emisora del pedido, Subtotal, IVA y Monto Total en la moneda del pedido.

Regla 4 — Modal de Generación con Uso CFDI como único editable
Al presionar "Generar Factura" en un pedido, el modal de Generación de Factura muestra en modo solo lectura: cliente (identificado por Razón Social, homologado con el resto de Facturación), Monto Total del pedido, Pedido Interno, Condiciones de Pago, datos del contacto del cliente (nombre, correo electrónico, teléfono), Datos Fiscales del Cliente (RFC, Razón Social, Código Postal, Régimen Fiscal, Correo electrónico, Moneda, Tipo de Cambio, Tipo de Comprobante, Método de Pago, Forma de Pago), Datos Fiscales del Emisor (RFC, Razón Social, Régimen Fiscal) y los Comentarios de Facturación. El único campo editable del modal es el Uso CFDI (combo de selección con catálogo SAT), que se precarga con el valor configurado en la ficha del cliente en el Catálogo de Clientes; contar con un valor válido de Uso CFDI es obligatorio para continuar el flujo.

Regla 5 — Forma de Pago, Método de Pago y Tipo de Comprobante forzados por normativa SAT
Por ser la Factura por Adelantado una factura PPD, el modal de Generación presenta como valores forzados (solo lectura): Método de Pago = "PPD - Pago en parcialidades o diferido", Forma de Pago = "99 - Por definir", Tipo de Comprobante = "I - Ingreso". Estos valores son obligatorios por normativa SAT para facturas PPD y no son modificables por el usuario. La Forma de Pago real se captura posteriormente en el Complemento de Pago del módulo Validar Cobro.

Regla 6 — Tipo de Cambio del día de generación
Cuando el cliente factura en una moneda distinta a pesos mexicanos, el modal de Generación muestra el Tipo de Cambio aplicable al día de generación de la factura. El valor es solo lectura, no modificable por el usuario.

Regla 7 — Generación en dos pasos: revisar datos, previsualizar PDF, timbrar
Tras confirmar los datos en el modal de Generación de Factura, el sistema muestra el modal de Previsualización con el PDF de la Factura tal como saldrá al cliente. El PDF previsualizado no incluye el folio fiscal ni los demás datos que el SAT asigna al timbrar (folio fiscal, sello digital, cadena original, código QR), dado que en este momento el documento aún no ha sido timbrado. El usuario revisa el documento y confirma con "Generar Factura" desde el modal de Previsualización para proceder con la validación previa y el timbrado real ante el PAC SAT. La acción "Cancelar" en cualquiera de los dos modales aborta el flujo sin timbrar.

Regla 8 — Manejo de errores de validación SAT o de timbrado
Cuando el timbrado ante el PAC falla por un error de validación (ejemplo: el Código Postal del cliente no coincide con el registrado en el SAT) o por un error del propio PAC (errores de decimales en totales, indisponibilidad del servicio, otros), el sistema muestra el modal de Alerta SAT con la descripción específica del error y una acción "Continuar" que cierra el modal y devuelve al usuario al estado previo. El usuario debe corregir los datos en el módulo correspondiente (típicamente el Catálogo de Clientes para errores de datos fiscales del cliente) y reintentar la generación desde el principio. El sistema no ofrece edición directa de los datos del cliente desde este modal.

Regla 9 — Validaciones previas al envío al PAC
Antes de enviar la Factura al PAC para su timbrado, el sistema valida: la compatibilidad del Uso CFDI seleccionado con el Régimen Fiscal del receptor, la ausencia de valores negativos en los importes, y la congruencia de los importes y totales del comprobante, conforme a la guía técnica de facturas de tipo Ingreso. Si alguna validación falla, el sistema no envía la solicitud al PAC y muestra el modal de Alerta SAT con la descripción del error (Regla 8).

Regla 10 — Almacenamiento de la Factura al timbrarse exitosamente
Cuando el timbrado se completa exitosamente, el sistema almacena la Factura como artefacto fiscal inmutable (incluye el documento timbrado completo: XML y representación visual PDF) y muestra al usuario el modal de confirmación de éxito de generación. A diferencia de la Proforma (que se almacena al envío), la Factura se almacena al timbrarse: una vez timbrada ya no se puede modificar.

Regla 11 — Folio de la Factura por empresa emisora
El folio de la Factura usa un contador consecutivo numérico independiente por empresa emisora (Golocaer, Mungen, Proquifa, Proveedora Quimico Farmaceutica). La serie de la Factura es "A2", distinta de la que usa el sistema anterior (Legacy), para evitar colisión de folios entre ambos sistemas.

Regla 12 — Modal de Envío de Factura
Cuando la Factura fue timbrada exitosamente y el usuario presiona "Enviar Factura", el modal de Envío presenta: Para (destinatario) con el contacto del cliente del pedido, editable, con default heredado del flujo de tramitación; CC con el ESAC asignado al cliente/pedido, editable, con default sugerido por el sistema; Asunto generado automáticamente con los folios del documento, no editable; Adjuntos con el PDF y XML de la Factura timbrada, no editables; y Notas extras, un campo editable opcional para texto adicional libre. El mecanismo de captura y edición del destinatario conserva el mismo utilizado actualmente por el sistema en sus envíos de correo, sin introducir un origen de datos distinto — confirmado por el cliente.

Regla 13 — Confirmación de envío exitoso y cierre del pendiente
Al confirmarse que el correo se envió exitosamente al cliente, el sistema muestra el modal de confirmación de éxito de envío, el pedido sale del listado del Detalle, y la factura queda almacenada y consultable desde el módulo Validar Cobro.

Regla 14 — Salida operativa post-envío diferenciada por tipo de pedido
Tras el envío exitoso de la Factura por Adelantado, el sistema actúa diferenciadamente según el tipo del pedido: si es Crédito o Pago contra entrega, la información de la Factura se transfiere a Legacy y el pedido continúa por el flujo Crédito en Legacy; si es Prepago, el sistema genera el pendiente correspondiente en el módulo Validar Cobro para que el equipo de Cobranza valide el pago del cliente contra esta Factura.

Regla 15 — Visibilidad filtrada por cartera del usuario
El acceso al Detalle de un cliente solo se permite si el cliente está asignado a la cartera del usuario (campo Cobrador del Catálogo de Clientes). Los clientes asignados a otros usuarios no son accesibles.

Regla 16 — Datos de la Factura provienen del Pedido
Los conceptos de la Factura se obtienen del Pedido: por cada partida, cantidad, descripción (catálogo + descripción + marca), precio unitario e importe. **La Factura por Adelantado no consigna lote ni pedimento**, conforme a la confirmación del cliente de que son datos de flujos posteriores: el pedimento se representa como N/A (comportamiento preexistente del sistema); el lote tampoco se incluye porque el surtido del pedido (asignación de lote en almacén) ocurre después de cobrar y facturar. Decisión confirmada por el cliente — OBS-039.

---

## Riesgos

Riesgo 1 — Indisponibilidad del PAC TurboPac
Si el PAC TurboPac (Quadrum Tecnologías) está caído o responde con timeout, no se puede timbrar la Factura. Ante esta situación, el sistema informa al usuario que debe esperar a que el servicio se restablezca; no ofrece timbrado alterno.

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
Dado que el cliente tiene pedidos Crédito, Prepago o Pago contra entrega sin Sustancias Controladas con Factura por Adelantado pendiente de generar o enviar,
Cuando el sistema renderiza el listado,
Entonces deberá mostrar una fila por cada pedido pendiente con: Pedido Interno, Fecha del pedido, Condiciones de Pago, Empresa Emisora del pedido, Subtotal, IVA, Monto Total y acción contextual.

Criterio A3 — Acción contextual por estado del pedido
Dado que un pedido del cliente aparece en el listado,
Cuando el sistema renderiza la acción del pedido,
Entonces deberá mostrar:
- "Generar Factura" si el pedido NO tiene factura emitida (estado: pendiente generar).
- "Enviar Factura" si el pedido tiene factura ya timbrada pero pendiente de envío (estado: pendiente enviar).

═══════════════════════════════════════════════════════════════
SECCIÓN B — REVISIÓN DE DATOS PARA GENERAR LA FACTURA
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

Criterio B7 — Uso CFDI como único campo editable, precargado y obligatorio
Dado que el modal de Generación está abierto,
Cuando el usuario interactúa con el modal,
Entonces el único campo modificable deberá ser Uso CFDI (combo de selección con el catálogo SAT correspondiente), precargado con el valor configurado en la ficha del cliente en el Catálogo de Clientes. Contar con un valor válido de Uso CFDI es obligatorio para continuar el flujo. Todos los demás datos son solo lectura.

Criterio B8 — Acciones del modal: Cancelar y Generar Factura
Dado que el usuario revisa los datos en el modal,
Cuando finaliza la revisión,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el flujo, cierra el modal sin generar factura) y Generar Factura (procede al siguiente paso: previsualización del PDF).

═══════════════════════════════════════════════════════════════
SECCIÓN C — PREVISUALIZACIÓN Y VALIDACIÓN PREVIA AL TIMBRADO
═══════════════════════════════════════════════════════════════

Criterio C1 — Apertura del modal de Previsualización tras confirmar el modal de Generación
Dado que el usuario presiona "Generar Factura" en el modal de Generación,
Cuando el sistema procesa la acción,
Entonces deberá abrir el modal de Previsualización mostrando el PDF de la Factura tal como saldrá al cliente, con los datos del pedido y del cliente integrados, sin el folio fiscal ni los demás datos que el SAT asigna al timbrar. En este momento la Factura aún NO se ha timbrado ante el PAC.

Criterio C2 — Acciones del modal: Cancelar y Generar Factura
Dado que el usuario revisa el PDF previsualizado,
Cuando finaliza la revisión,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el flujo, cierra el modal sin timbrar) y Generar Factura (procede con la validación previa y el timbrado real ante el PAC SAT).

Criterio C3 — Validación de compatibilidad Uso CFDI / Régimen Fiscal y de valores negativos
Dado que el usuario confirma "Generar Factura" desde el modal de Previsualización,
Cuando el sistema valida la Factura antes de enviarla al PAC,
Entonces deberá verificar la compatibilidad del Uso CFDI seleccionado con el Régimen Fiscal del receptor y la ausencia de valores negativos en los importes, conforme a la guía técnica de facturas de tipo Ingreso.

Criterio C4 — Validación de congruencia de importes y totales
Dado que el sistema valida la Factura antes de enviarla al PAC,
Cuando revisa los importes del comprobante,
Entonces deberá verificar la congruencia entre las partidas, el Subtotal, el IVA y el Monto Total. Si alguna de las validaciones de este criterio o del Criterio C3 falla, el sistema no envía la solicitud al PAC y muestra el modal de Alerta SAT con la descripción del error (Sección D).

═══════════════════════════════════════════════════════════════
SECCIÓN D — MANEJO DE ERRORES DE VALIDACIÓN O TIMBRADO
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
SECCIÓN E — TIMBRADO EXITOSO Y ALMACENAMIENTO DE LA FACTURA
═══════════════════════════════════════════════════════════════

Criterio E1 — Aviso mientras se procesa el timbrado
Dado que el usuario confirmó el timbrado desde el modal de Previsualización,
Cuando el sistema envía la solicitud de timbrado al PAC y espera respuesta,
Entonces deberá informar al usuario que la solicitud está siendo procesada (ejemplo: "Su solicitud está siendo atendida, por favor espere...").

Criterio E2 — Confirmación de éxito y cambio de estado del pedido
Dado que el timbrado se completa exitosamente,
Cuando el PAC retorna la confirmación,
Entonces el sistema deberá mostrar la confirmación de éxito (ejemplo: "¡Has generado una factura Exitosamente!") y almacenar la Factura. El pedido cambia de estado en el listado: ahora muestra la acción "Enviar Factura" en lugar de "Generar Factura".

Criterio E3 — Almacenamiento inmediato al timbrado exitoso
Dado que el timbrado fue exitoso,
Cuando el sistema procesa la respuesta del PAC,
Entonces deberá almacenar el artefacto fiscal completo (XML timbrado y PDF) como registro permanente inmutable. La Factura no puede modificarse después de este momento.

Criterio E4 — Folio por empresa emisora con serie "A2"
Dado que la Factura se timbra exitosamente,
Cuando el sistema le asigna el folio,
Entonces deberá usar el contador consecutivo independiente de la empresa emisora del pedido, con la serie "A2" — distinta de la que usa el sistema anterior (Legacy) — para evitar colisión de folios entre ambos sistemas.

═══════════════════════════════════════════════════════════════
SECCIÓN F — ENVÍO DE LA FACTURA AL CLIENTE
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
Entonces deberá pre-rellenar el asunto en un formato que incluya el folio de la Factura y el folio del Pedido Interno. **Brecha pendiente:** la plantilla exacta del asunto y del cuerpo del correo de envío está pendiente de definición; es transversal a los demás documentos del proyecto (Proforma, Factura, etc.) que hasta ahora la daban por definida.

Criterio F5 — Adjuntos automáticos PDF y XML
Dado que el modal de Envío está abierto,
Cuando el sistema muestra los adjuntos,
Entonces deberá incluir automáticamente los archivos PDF y XML de la Factura timbrada. Los adjuntos NO son removibles por el usuario.

Criterio F6 — Notas extras editables
Dado el modal de envío, cuando el sistema presenta el campo de notas extras, entonces deberá ser editable por el usuario para capturar texto adicional libre opcional.

Criterio F7 — Acciones del modal: Cancelar y Enviar
Dado que el usuario revisa el modal de Envío,
Cuando finaliza la edición,
Entonces el modal deberá ofrecer dos acciones: Cancelar (aborta el envío, cierra el modal; la Factura permanece timbrada y persistida con estado pendiente enviar) y Enviar (procede con el envío del correo al cliente).

═══════════════════════════════════════════════════════════════
SECCIÓN G — CONFIRMACIÓN DE ENVÍO Y SALIDA OPERATIVA
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
- Si el pedido es Crédito o Pago contra entrega: la información de la Factura se transfiere a Legacy. El pedido continúa por el flujo Crédito en Legacy. NO se genera pendiente en Validar Cobro.
- Si el pedido es Prepago: el sistema genera el pendiente correspondiente en el módulo Validar Cobro asociado a esta Factura, para que el equipo de Cobranza valide el pago del cliente.

═══════════════════════════════════════════════════════════════
SECCIÓN H — ORDEN DEL FLUJO COMPLETO
═══════════════════════════════════════════════════════════════

Criterio H1 — Orden canónico del flujo
Dado que el usuario inicia el flujo completo de generar y enviar una Factura,
Cuando avanza por las etapas del flujo,
Entonces éstas deberán seguir el siguiente orden funcional: revisión de los datos fiscales del cliente y del emisor (con Uso CFDI editable) → previsualización del PDF sin timbrar → validaciones previas al envío al PAC → timbrado ante el PAC (con manejo de error y reintento manual si falla) → almacenamiento de la Factura y cambio de estado a pendiente de envío → envío del correo al cliente con la Factura adjunta → confirmación de envío y salida operativa según el tipo de pedido (transferencia a Legacy para Crédito y Pago contra entrega; pendiente en Validar Cobro para Prepago).

---

## Notas Adicionales

- Esta fila documenta la pantalla de Detalle por cliente del módulo Factura por Adelantado y todo el flujo de generación, validación, timbrado y envío. La pantalla inicial del módulo (listado agrupado por cliente) se documenta en requisito independiente.
- El módulo Factura por Adelantado es funcionalidad NUEVA en PQF2 R16, para casos comerciales donde el cliente requiere la factura antes del ingreso de la mercancía. Aplica a pedidos Crédito, Prepago y Pago contra entrega, todos SIN Sustancias Controladas (tipo Mundial, Nacional u Origen); los pedidos Prepago con controlados se atienden vía Factura Anticipo desde Validar Cobro (artefacto y flujo independientes), y los pedidos Crédito con controlados no son elegibles.
- El PAC utilizado por PROQUIFA es TurboPac (Quadrum Tecnologías). Ante indisponibilidad del servicio, el sistema informa al usuario que debe esperar a que se restablezca (Riesgo 1); el reintento ante fallo de timbrado es responsabilidad de Finanzas (ver R16A-RE-FU-018 y `Diagramas/Diagrama Secuencia Encolamiento Finanzas y Timbrado Factura.md`): el pendiente permanece sin timbrar, se incrementa un contador de reintentos, y se notifica a soporte por correo si se supera el límite.
- El detalle del contenido y estructura del PDF de la Factura se documenta en requisito independiente, análogo al del PDF de Proforma. La Empresa Emisora que se muestra por pedido proviene de la empresa asignada al pedido (mismo origen que en la Proforma); no es seleccionable desde este módulo.
- El folio de la Factura es un consecutivo independiente por empresa emisora, con serie "A2" (distinta de la del sistema Legacy, para evitar colisión de folios entre ambos). Contrasta con el folio de la Proforma, que es global para todo el grupo.
- Para los pedidos Crédito y Pago contra entrega, la Factura se transfiere a Legacy tras el envío exitoso, y el seguimiento del cobro de esas facturas ocurre en Legacy, fuera del alcance de este módulo. Para los pedidos Prepago, el seguimiento del cobro ocurre en Validar Cobro, dentro de PQF2.
- **[Resuelto — Duda 047]** Denominación canónica: el rol operativo es **Gestor de Cobranza**; el puesto de trabajo es **Analista de Cuentas por Cobrar**.
- **[Resuelto — DUDA-050]** El timbrado de la Factura por Adelantado es uno a uno, documento por documento (Reglas 7-10, Criterio H1); no se implementa timbrado masivo ni por lote, conforme a lo aceptado por el cliente tras sesión de explicación del flujo.
- **Brecha pendiente:** la plantilla exacta del asunto y del cuerpo del correo de envío está pendiente de definición (Criterio F4); es transversal a los demás documentos del proyecto que hasta ahora la daban por definida.
- **Decisión OBS-039:** ninguna Factura por Adelantado consigna lote ni pedimento, conforme a la confirmación del cliente de que son datos de flujos posteriores al cobro y la facturación.

---

## Cambios

| # | Fecha | Referencia | Descripción del cambio |
|---|-------|------------|------------------------|
| 1 | 2026-06-10 | OBS-039 | Regla 15: lote y pedimento excluidos de la descripción de conceptos de la Factura por Adelantado. Descripción actualizada a "catálogo + descripción + marca". Pendiente de lote cerrado — decisión: ninguna Factura por Adelantado consigna lote ni pedimento. |
| 2 | 2026-08-21 | DUDA-048 | Regla 4: cerrado el pendiente "Razón Social o Alias" — el cliente se identifica por RAZÓN SOCIAL, homologado con el resto de Facturación. |
| 3 | 2026-08-21 | DUDA-049 | Se descarta el desarrollo de Factura por Adelantado para Región Perú (se cancela la facturación de Perú). Se tachan y cierran las menciones a formato de asunto/folio fiscal peruano pendientes (Criterio F4, Notas Adicionales) y el Riesgo 3 (brecha SUNAT/OSE). |
| 4 | 2026-08-21 | DUDA-050 | Se documenta explícitamente que el timbrado de la Factura por Adelantado es uno a uno (no masivo/por lote); el cliente aceptó esta propuesta tras sesión de explicación del flujo. |
| 5 | 2026-09-11 | Retiro de Región Perú del alcance / Cierre de dudas resueltas / Incorporación de las validaciones previas al timbrado / Definición de la serie de la Factura / Precisión del tipo de pedido en la salida operativa / Apertura del pendiente de la plantilla del correo / Correcciones de consistencia | Pasada integral: (1) Alcance/No aplica a: se sustituye la condición de la brecha de timbrado de Perú por su exclusión directa del release; se elimina el Riesgo de esa brecha (antes Riesgo 3) y la nota del formato del asunto del correo para esa región (Criterio F4). (2) Se elimina el Riesgo de solapamiento de denominación de rol (antes Riesgo 2); la denominación queda documentada en Notas Adicionales, corrigiendo el puesto de trabajo a **Analista de Cuentas por Cobrar** (no "por Pagar"). (3) Regla 4 y Criterio B7: el Uso CFDI se precarga con el valor de la ficha del cliente y es obligatorio para continuar. (4) Se incorpora una nueva Regla 9 (validaciones previas al envío al PAC: compatibilidad Uso CFDI/Régimen Fiscal, ausencia de valores negativos, congruencia de importes y totales) y los nuevos Criterios C3 y C4; se renumeran las Reglas 9–15 anteriores a 10–16. (5) Regla 11 (folio) y nuevo Criterio E4: se define la serie "A2" para la Factura, distinta de la del sistema Legacy, para evitar colisión de folios. (6) Requisito, Alcance, Regla 14 y Criterio G3: se incorpora Pago contra entrega como tipo de pedido propio, cuya Factura se transfiere a Legacy igual que la de Crédito. (7) Criterio F4: se marca como pendiente la plantilla exacta del asunto y del cuerpo del correo de envío, transversal a los demás documentos del proyecto. (8) Correcciones de consistencia: se retiran menciones a colores, indicador de carga (loader), texto de tipo de dato del folio y "text area"; Regla 10 (antes 9) y Criterio E3: "persiste en base de datos" → "almacena"; Regla 7 y Criterio C1: se precisa que el PDF previsualizado no incluye el folio ni los datos que asigna el SAT al timbrar; Riesgo 1: se retira el RFC del PAC; Criterio H1: se resume el orden del flujo en un criterio funcional único; se renombran las Secciones B, C, D, E, F y G conforme al flujo funcional; Regla 2 y Criterio A2: se precisa la redacción de los datos del cliente y el origen del listado. (9) Notas Adicionales reescritas por completo, incorporando el criterio de las validaciones previas, el origen de la Empresa Emisora y el seguimiento del cobro de las facturas transferidas a Legacy. |
