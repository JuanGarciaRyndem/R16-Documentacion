# R16A-RE-FU-028 — Validar Cobro: Paso 3 México

| Campo                   | Valor                                                                                                                                                                                                                                                                                                                                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **ID**                  | R16A-RE-FU-028                                                                                                                                                                                                                                                                                                                                                                                                     |
| **Título**              | Validar Cobro: Paso 3 México                                                                                                                                                                                                                                                                                                                                                                                       |
| **Módulo / Épica**      | Validar Cobro                                                                                                                                                                                                                                                                                                                                                                                                      |
| **Historia de Usuario** | Yo como **Gestor de Cobranza**, quiero contar con la tercera pantalla del wizard de Validar Cobro (Paso 3 - Facturación y Envío) para previsualizar, timbrar y enviar los documentos fiscales de cada línea de la asociación, para cerrar el ciclo operativo de cobranza con todos los artefactos fiscales y operativos del cliente entregados. |
| **Prioridad**           | Alta                                                                                                                                                                                                                                                                                                                                                                                                               |
| **Estado**              | Propuesto                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Requisito asociado**  | R16.2M-RE-FU-002                                                                                                                                                                                                                                                                                                                                                                                                   |

---

## Requisito Funcional

El sistema debe contar con la tercera pantalla del wizard de Validar Cobro (Paso 3 - Facturación y Envío) para clientes con Región México, donde el sistema determina por cada documento de la asociación el tipo de documento fiscal a generar (Factura, Factura Anticipo o Complemento de Pago) y el usuario previsualiza, timbra y envía cada uno de forma individual. Al enviar cada documento en operaciones México, el sistema dispara automáticamente el establecimiento de la Fecha Estimada de Entrega del pedido, la transferencia a Legacy y la generación de las Confirmaciones de Pedido. El usuario debe completar todas las líneas del cliente antes de cerrar el ciclo. La estructura UI de esta pantalla se reutiliza para Perú con diferencias importantes y se documenta en requisito independiente.

---

## Alcance

### Aplica a

- Pantalla del Paso 3 del wizard de Validar Cobro: Facturación y Envío.
- Aplica a clientes con Región México exclusivamente (Perú se documenta en requisito independiente con diferencias significativas).
- Cabecera del cliente (estructura consistente con Paso 1 y Paso 2: logo, Alias, etiquetas preexistentes de clasificación, RFC/RUC, razón social legal, moneda de facturación).
- Barra de pasos del wizard: 1-CAPTURAR COBRO (✓), 2-ASOCIAR factura/PROFORMA (✓), 3-FACTURACIÓN Y ENVÍO (activo).
- Listado de líneas a procesar, una por cada documento de la asociación cerrada en el Paso 2.
- Lógica condicional por línea según el tipo de documento origen del Paso 2:
  - Proforma normal (sin productos controlados) → genera Factura nueva (CFDI Ingreso).
  - Proforma con productos controlados → genera Factura Anticipo (CFDI Ingreso). ~~con tipo de relación 07 SAT — Aplicación de Anticipo~~ **INCORRECTO — DUDA-088:** la Factura Anticipo NO lleva tipo de relación 07. La relación 07 se usa en la Factura Final (fuera de alcance de este requisito, se genera en Legacy) para referenciar hacia la Factura Anticipo. La Factura Anticipo se genera según `Guia_Tecnica_Facturas_Ingreso_MX.md` (sección 6): sin `CfdiRelacionados`, con `ClaveProdServ=84111506` (genérico), `ClaveUnidad=ACT`, descripción "Anticipo del bien o servicio".
  - Factura existente (de Prepago con Factura por Adelantado previa) con cobro asociado → genera Complemento de Pago (CFDI Pagos 2.0).
- Por línea, edición de Uso CFDI (combo del catálogo SAT c_UsoCFDI).
- Por línea, edición de Método de Pago únicamente para líneas que parten de proforma (PPD o PUE). Líneas de Complemento de Pago tienen Método de Pago fijo en PPD (no editable, conforme normativa SAT).
- Inclusión automática de las Notas de Crédito aplicadas en el Paso 2 dentro del nodo CFDIRelacionados del XML a timbrar (con UUID y monto correspondiente de cada NC, tipo de relación 01 o 07 según el caso).
- Estados por línea: Pendiente (estado inicial) → Factura Generada / Factura Anticipo Generada / Complemento Generado (después del timbrado exitoso) → Enviado (después del envío al cliente).
- Acciones por línea: Previsualizar PDF (modal con previsualización antes de timbrar), Generar/Timbrar (timbrado con PAC TurboPac), Enviar (modal de envío al cliente).
- Operación INDIVIDUAL por línea: no existen acciones masivas (no "Timbrar todo", no "Enviar todo"). El usuario procesa una línea a la vez.
- Modal Previsualizar: muestra el PDF representativo del documento a timbrar para validación visual del usuario antes del timbrado.
- Modal éxito de generación: tras timbrado exitoso muestra confirmación con folio fiscal (UUID) timbrado.
- Modal Enviar: permite al usuario disparar el envío del documento al cliente.
- Destinatario del envío: el contacto del cliente que envió el pedido (heredado del flujo de tramitación del pedido).
- Cuerpo del correo de envío para facturas: misma plantilla que el correo de envío de Factura por Adelantado.
- Cuerpo del correo de envío para complementos de pago: ** Pendiente confirmar la plantilla; propuesta inicial "`<Folio Pedido Interno> - <Folio Factura>`" como asunto, cuerpo por definir. **
- Al ENVIAR cada documento, el sistema dispara automáticamente las acciones post-envío (solo México):
  - Establecimiento de la Fecha Estimada de Entrega (FEE) del pedido asociado.
  - Transferencia del pedido y documentos generados (factura/anticipo/complemento, NCs aplicadas, info del cobro) a Legacy.
  - Generación de la Confirmación de Pedido (documento que el cliente confirma, análogo al existente hoy en ProquifaNet para crédito, ahora extendido a prepago en R16).
- Auto-guardado del Paso 3 consistente con Paso 1 y Paso 2.
- Persistencia del estado del Paso 3: si el usuario sale del wizard antes de finalizar todas las líneas, al volver a entrar al mismo cliente desde Validar Cobro el sistema lo regresa directamente al Paso 3 (no se reinicia el wizard) hasta que todas las líneas estén en estado Enviado.
- Inmutabilidad post-timbrado: una vez una línea está en estado Factura Generada / Factura Anticipo Generada / Complemento Generado, no se permite re-timbrar ni editar el documento. La única vía de modificación post-timbrado fiscal es vía el módulo Notas de Crédito (módulo independiente, fuera del Paso 3 de Validar Cobro).
- Manejo de errores del PAC TurboPac (timbrado fallido, PAC indisponible, datos inválidos, errores SAT): mismo comportamiento operativo que en el módulo Factura por Adelantado (mensaje de error al usuario con detalle del problema).
- Cierre del wizard: el wizard se considera cerrado para el cliente cuando todas las líneas del Paso 3 estén en estado Enviado.

### No aplica a

- Wizard de Validar Cobro para Región Perú: se documenta en requisito independiente con diferencias significativas.
- Cancelación fiscal de documentos ya timbrados en el Paso 3 (vía CFDI de cancelación SAT): NO está contemplada en R16 dentro del Paso 3 ni en el módulo Validar Cobro.
- Generación masiva (Timbrar todo / Enviar todo): NO contemplada. La operación es individual por línea. (Confirmado con cliente — DUDA-050.)
- Edición del Método de Pago en líneas de Complemento de Pago: NO permitida (PPD fijo conforme normativa SAT).
- Re-timbrado de un documento ya generado en el Paso 3: NO permitido. La línea queda inmutable post-timbrado.

---

## Reglas de Negocio

**Regla 1 — Aplicabilidad solo a Región México**
El Paso 3 del wizard de Validar Cobro opera exclusivamente sobre clientes con Región México. Los clientes con Región Perú son atendidos por el wizard equivalente Perú, con diferencias significativas en catálogos y reglas (requisito independiente).

**Regla 2 — Líneas a procesar derivadas del Paso 2**
El sistema genera una línea por cada documento (proforma o factura) asociado en el Paso 2 que requiera emisión de un documento fiscal. Cada línea referencia: tipo de documento origen (Proforma normal, Proforma con controlados, Factura existente), folio del documento origen, Pedido Interno, empresa emisora del grupo, cobros asociados, y NCs aplicadas (si las hubo).

**Regla 3 — Lógica condicional del tipo de documento fiscal a generar**
El tipo de documento fiscal a generar por línea depende del tipo de documento origen: si la línea parte de una Proforma normal (sin productos controlados), se genera una Factura nueva (CFDI Ingreso); si parte de una Proforma con productos controlados, se genera una Factura Anticipo (CFDI Ingreso). ~~con tipo de relación 07 SAT — Aplicación de Anticipo~~ si parte de una Factura existente (de Prepago con Factura por Adelantado previo) con cobro asociado, se genera un Complemento de Pago (CFDI Pagos 2.0) que se relaciona al UUID de la factura existente. **RESUELTO — DUDA-088 (corrige error anterior):** la Factura Anticipo NO usa tipo de relación 07 SAT — es incorrecto usar la relación 07 en la Factura Anticipo. La relación 07 aplica en la factura FINAL (cuando declara que aplica el anticipo de una Factura Anticipo previa), documento fuera de alcance de este requisito. La Factura Anticipo debe generarse conforme a `Guia_Tecnica_Facturas_Ingreso_MX.md`.

**Regla 4 — Edición del Uso CFDI por línea**
El campo Uso CFDI de cada línea es un combo del catálogo SAT c_UsoCFDI (ejemplo: "G01 - Adquisición de mercancías", "G03 - Gastos en general", "P01 - Por definir", etc.) editable por el usuario antes del timbrado. El valor por defecto corresponde al Uso CFDI configurado del cliente o capturado en el pedido original; el usuario puede ajustarlo previo al timbrado.

**Regla 5 — Edición del Método de Pago según tipo de documento**
El comportamiento del campo Método de Pago depende del tipo de documento a generar: si la línea genera Factura nueva o Factura Anticipo (parte de proforma), el Método de Pago es editable mediante radio button con dos opciones — PPD (Pago en parcialidades o diferido) o PUE (Pago en una sola exhibición); si la línea genera Complemento de Pago (parte de factura existente), el Método de Pago es siempre PPD no editable (conforme normativa SAT, los complementos solo aplican a CFDIs con método PPD).

**Regla 6 — Inclusión de Notas de Crédito en el XML al timbrar**
Cuando el usuario aplicó una o más NCs a la línea en el Paso 2, el sistema incluye las NCs aplicadas dentro del nodo CFDIRelacionados con: UUID de cada NC, monto correspondiente aplicado, y tipo de relación SAT (01 - Nota de crédito de los documentos relacionados, o 07 - CFDI por aplicación de anticipo, según el caso fiscal). Esto cumple con la normativa SAT (Apéndice 5 Anexo 20 versión 4.0).

**Regla 7 — Flujo operativo por línea: previsualizar, timbrar, enviar**
El flujo operativo por línea es: Previsualizar PDF (muestra el PDF representativo del documento que se generará, permite validar visualmente datos antes de timbrar); Generar/Timbrar (dispara el timbrado del documento con el PAC TurboPac: si la línea es Factura PUE desde proforma, timbra 1 CFDI; si es Factura PPD desde proforma, timbra en cascada 2 CFDIs — Factura PPD + Complemento de Pago asociado al cobro confirmado; si es Complemento de Pago desde factura existente, timbra 1 CFDI; si el timbrado es exitoso, la línea pasa al estado Generado correspondiente); y Enviar (abre el modal de envío con destinatario, CC al ESAC, asunto generado por sistema, adjuntos —PDF y XML de cada CFDI de la línea más Confirmación de Pedido del Pedido Prepago— y text area de notas extras opcional).

**Regla 8 — Estados por línea**
Los estados posibles de cada línea son: Pendiente (estado inicial al entrar al Paso 3, aún no se ha timbrado ni enviado); Generado (después del timbrado exitoso de los CFDIs que correspondan a la línea, 1 o 2 según aplique; los documentos están timbrados con UUID SAT pero aún no se enviaron al cliente); y Enviado (después del envío exitoso al cliente con todos los CFDIs adjuntos más Confirmación de Pedido; la línea está cerrada operativamente). Si el usuario timbra pero no envía en la misma sesión, la línea queda en estado intermedio Generado hasta el envío.

**Regla 9 — Operación INDIVIDUAL por línea**
No existe operación masiva en el Paso 3. La generación es individual (una línea a la vez) y el envío es individual (una línea a la vez). No hay botón "Timbrar todo" ni "Enviar todo".

**Regla 10 — Generación en cascada según Método de Pago en líneas con proforma origen**
Al confirmar el timbrado de una línea con proforma origen, el sistema actúa según el Método de Pago elegido: si es PUE, timbra únicamente la Factura PUE (1 CFDI), sin generar Complemento de Pago (conforme normativa SAT, las facturas PUE son fiscalmente autocontenidas); si es PPD, timbra primero la Factura PPD y, en cascada inmediata tras timbrado exitoso, timbra el Complemento de Pago asociado al cobro confirmado en el wizard (2 CFDIs).

**Regla 11 — Acciones automáticas al enviar (solo México)**
Cuando el usuario presiona Enviar en una línea y el envío es exitoso, el sistema dispara automáticamente (aplica únicamente a operaciones México): establecer la Fecha Estimada de Entrega (FEE) del pedido asociado al documento enviado, y transferir el pedido y los documentos generados (Factura, Complemento de Pago, NCs aplicadas, info del cobro) al sistema Legacy. La Confirmación de Pedido se genera al enviar (en el Paso 3 de Validar Cobro) y se adjunta en el mismo correo de envío de la línea. No se previsualiza: solo se genera y se muestra como archivo adjunto en el modal de envío. No existe candado bloqueante.

**Regla 12 — Destinatarios del envío (Para y CC)**
Al armar el modal de envío, el sistema determina los destinatarios: Para es el contacto del cliente del pedido (heredado del flujo de tramitación, editable por el usuario en el modal); CC es el ESAC asignado al cliente/pedido (sugerido por el sistema, editable por el usuario en el modal). ~~Pendiente confirmar el comportamiento si el contacto del pedido no está disponible o si hay múltiples contactos del cliente.~~ **RESUELTO — DUDA-089:** se usa el mismo mecanismo de envíos que el sistema actual ya tiene; no requiere desarrollo adicional (se descarta como funcionalidad nueva a construir).

**Regla 13 — Modal Previsualizar**
Al presionar "Previsualizar", el sistema abre un modal con la previsualización del PDF representativo del documento a generar (incluye todos los datos que tendrá la factura/anticipo/complemento). El usuario puede cerrar el modal sin acción, regresar a editar campos del Paso 3, o proceder a Timbrar.

**Regla 14 — Modal de éxito tras timbrado**
Cuando el timbrado de una línea es exitoso y el PAC TurboPac responde con UUID timbrado, el sistema muestra un modal de éxito con el folio fiscal (UUID) del documento timbrado y confirmación de generación. La línea pasa al estado correspondiente (Generada).

**Regla 15 — Modal Enviar**
Cuando la línea está en estado Generado y el usuario presiona "Enviar", el sistema abre el modal de envío con: Destinatario (Para) con el contacto del cliente del pedido (editable, default heredado del flujo de tramitación); Copia (CC) con el ESAC asignado (editable, default sugerido por el sistema); Asunto generado automáticamente según plantilla por tipo de documento de la línea (no editable); Adjuntos con el PDF y XML de cada CFDI timbrado en la línea más Confirmación de Pedido del Pedido Prepago (no editables); y Notas extras, text area editable por el usuario para capturar texto adicional libre opcional.

**Regla 16 — Persistencia del Paso 3 y navegación atrás según estado de timbrado**
Si el usuario sale del wizard antes de finalizar todas las líneas del Paso 3, al regresar al módulo Validar Cobro y seleccionar al mismo cliente el sistema lo redirige directamente al Paso 3 con el estado actual preservado (líneas pendientes, generadas, enviadas). La posibilidad de regresar a los Pasos 1 o 2 depende del estado de timbrado de cada línea: SÍ se permite regresar mientras la línea NO se haya timbrado (el documento está en borrador/corrección); NO se permite regresar una vez timbrada la línea (factura, anticipo o complemento), aunque falte enviarla, porque el documento timbrado es inmutable. La corrección posterior al timbrado es solo vía el módulo Notas de Crédito.

**Regla 17 — Inmutabilidad post-timbrado**
Cuando una línea está en estado Generado o Enviado, el sistema no permite editar el documento timbrado ni re-timbrarlo. La única vía de modificación post-timbrado fiscal es a través del módulo Notas de Crédito (módulo independiente). La cancelación fiscal vía CFDI de cancelación SAT no está contemplada en el Paso 3 ni en el módulo Validar Cobro.

**Regla 18 — Manejo de errores del PAC TurboPac**
Cuando el usuario presiona Timbrar y el PAC TurboPac responde con error, el sistema muestra el mensaje de error con el detalle del problema (PAC indisponible, datos inválidos, errores SAT, etc.) siguiendo el mismo comportamiento operativo que el módulo Factura por Adelantado. La línea permanece en estado Pendiente para que el usuario corrija (si aplica) e intente nuevamente.

**Regla 19 — Auto-guardado del Paso 3**
Cuando el usuario modifica Uso CFDI, Método de Pago, ejecuta acciones de previsualizar/timbrar/enviar, o navega, el sistema auto-guarda el estado del Paso 3 (estados de cada línea, valores editados, documentos timbrados, envíos confirmados). No existe botón "Guardar" manual.

**Regla 20 — Cierre del wizard**
Cuando todas las líneas del Paso 3 están en estado Enviado, al confirmar la finalización el sistema cierra el wizard de Validar Cobro para el cliente y retorna al listado principal de Validar Cobro. El cliente sale del listado de pendientes (al menos por los documentos procesados en esta sesión).

---

## Riesgos

**Riesgo 1 — Bloqueo de navegación tras timbrado**
Mientras una línea NO se haya timbrado, el usuario puede regresar a los Pasos 1 o 2 a corregir (el documento está en borrador). Una vez timbrada la línea, el documento es inmutable: no se puede regresar a los Pasos 1 o 2, solo resta enviarlo y, si hubo error, corregir vía el módulo Notas de Crédito. Esto implica que, si una línea quedó timbrada pero el ciclo necesita interrumpirse (cliente cancela a último minuto, error detectado tarde), no hay salida limpia del Paso 3 para esa línea sin pasar por Notas de Crédito. ** Pendiente definir vía operativa de excepción para casos donde el usuario necesita salir del Paso 3 con líneas pendientes sin timbrar. **

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL CLIENTE Y BARRA DE PASOS
═══════════════════════════════════════════════════════════════

**Criterio A1 — Cabecera del cliente**
Dado que el usuario entra al Paso 3 del wizard,
Cuando el sistema renderiza la cabecera,
Entonces deberá mostrar: logo del cliente (si existe), Alias, etiquetas preexistentes de clasificación del Catálogo de Clientes, identificador fiscal bajo la etiqueta unificada "RFC/RUC", razón social legal completa y moneda de facturación.

**Criterio A2 — Barra de pasos del wizard**
Dado que el usuario está en el Paso 3,
Cuando el sistema renderiza la barra de pasos,
Entonces deberá mostrar los tres pasos: "1 - CAPTURAR COBRO" (✓), "2 - ASOCIAR factura/PROFORMA" (✓), "3 - FACTURACIÓN Y ENVÍO" (activo).

═══════════════════════════════════════════════════════════════
SECCIÓN B — LISTADO DE LÍNEAS A PROCESAR
═══════════════════════════════════════════════════════════════

**Criterio B1 — Una línea por cada documento de la asociación**
Dado que el usuario avanzó al Paso 3 con la asociación cerrada en el Paso 2,
Cuando el sistema arma el listado del Paso 3,
Entonces deberá generar exactamente una línea por cada documento (proforma o factura) asociado en el Paso 2 que requiera emisión de un documento fiscal nuevo (factura, anticipo o complemento).

**Criterio B2 — Datos visibles de cada línea**
Dado que el sistema renderiza una línea del Paso 3,
Cuando muestra sus datos,
Entonces deberá presentar: tipo de documento origen (Proforma normal / Proforma con controlados / Factura existente), folio del documento origen, fecha, Pedido Interno, empresa emisora del grupo PROQUIFA México, monto total de la factura, tipo de cambio (cuando aplique), NCs aplicadas (si aplica), estado actual de la línea (Pendiente / Generado / Enviado), y los campos fiscales seleccionables de la línea según el caso (Uso CFDI y Método de Pago — ver Sección D).

═══════════════════════════════════════════════════════════════
SECCIÓN C — LÓGICA CONDICIONAL DEL TIPO DE DOCUMENTO FISCAL
═══════════════════════════════════════════════════════════════

**Criterio C1 — Proforma normal → Factura nueva**
Dado que una línea parte de una Proforma normal (sin productos controlados),
Cuando el sistema determina el tipo de documento fiscal a generar,
Entonces deberá configurar la línea para emitir una Factura nueva (CFDI Ingreso). La proforma se referencia como documento origen interno (no fiscal).

**Criterio C2 — Proforma con productos controlados → Factura Anticipo**
Dado que una línea parte de una Proforma con productos controlados (USP/EP/FEUM/Microbiologics u otros químicos-biológicos sujetos a control),
Cuando el sistema determina el tipo de documento fiscal a generar,
Entonces deberá configurar la línea para emitir una Factura Anticipo (CFDI Ingreso). ~~con tipo de relación 07 SAT — Aplicación de Anticipo~~ **INCORRECTO — DUDA-088:** ver corrección arriba; la Factura Anticipo no lleva relación 07. Esto se debe a que el pago se recibe antes de la entrega física del producto controlado, conforme práctica operativa de PROQUIFA.

**Criterio C3 — Factura existente con cobro → Complemento de Pago**
Dado que una línea parte de una Factura existente (emitida previamente desde el flujo de Prepago con Factura por Adelantado) con un cobro asociado en el Paso 2,
Cuando el sistema determina el tipo de documento fiscal a generar,
Entonces deberá configurar la línea para emitir un Complemento de Pago (CFDI Pagos 2.0) que se relaciona al UUID de la factura existente para registrar el pago recibido conforme normativa SAT.

═══════════════════════════════════════════════════════════════
SECCIÓN D — EDICIÓN DE USO CFDI Y MÉTODO DE PAGO
═══════════════════════════════════════════════════════════════

**Criterio D1 — Edición del Uso CFDI por línea**
Dado que el usuario opera una línea del Paso 3,
Cuando renderiza el campo Uso CFDI,
Entonces deberá ofrecer un combo del catálogo SAT c_UsoCFDI (ejemplo: "G01 - Adquisición de mercancías", "G03 - Gastos en general", "P01 - Por definir", etc.), editable antes del timbrado. El valor por defecto corresponde al Uso CFDI configurado del cliente o capturado en el pedido original.

**Criterio D2 — Método de Pago para líneas con proforma origen**
Dado que una línea parte de una proforma (genera Factura o Factura Anticipo),
Cuando el sistema renderiza el campo Método de Pago,
Entonces deberá ofrecer un control radio button con dos opciones editables por el usuario antes del timbrado: PPD (Pago en parcialidades o diferido) y PUE (Pago en una sola exhibición). El usuario selecciona según corresponda al caso.

**Criterio D3 — Método de Pago fijo PPD en Complementos**
Dado que una línea genera Complemento de Pago (parte de factura existente),
Cuando el sistema renderiza el campo Método de Pago,
Entonces deberá mostrar PPD como valor fijo no editable. Esto cumple con la normativa SAT: los Complementos de Pago aplican únicamente a CFDIs con método PPD.

═══════════════════════════════════════════════════════════════
SECCIÓN E — INCLUSIÓN DE NOTAS DE CRÉDITO EN EL XML
═══════════════════════════════════════════════════════════════

**Criterio E1 — NCs aplicadas en Paso 2 incluidas en CFDIRelacionados**
Dado que la línea tiene una o más NCs aplicadas desde el Paso 2,
Cuando el sistema arma el XML del CFDI a timbrar,
Entonces deberá incluir cada NC dentro del nodo CFDIRelacionados del XML con: UUID timbrado de la NC, monto aplicado al documento, y tipo de relación SAT correspondiente (01 - Nota de crédito de los documentos relacionados, o 07 - CFDI por aplicación de anticipo, según el caso fiscal).

**Criterio E2 — Visibilidad de NCs aplicadas en la línea del Paso 3**
Dado que la línea tiene NCs aplicadas,
Cuando el sistema renderiza la línea,
Entonces deberá mostrar visualmente las NCs que se incluirán en el XML (folio NC, UUID, monto aplicado) para que el usuario valide antes del timbrado.

═══════════════════════════════════════════════════════════════
SECCIÓN F — FLUJO PREVISUALIZAR, TIMBRAR, ENVIAR
═══════════════════════════════════════════════════════════════

**Criterio F1 — Acción Previsualizar**
Dado que una línea está en estado Pendiente,
Cuando el usuario presiona "Previsualizar",
Entonces el sistema deberá abrir un modal con el PDF representativo del documento a generar. El usuario puede cerrar sin acción, regresar a editar o proceder a Timbrar.

**Criterio F2 — Acción Timbrar (con generación en cascada cuando aplique)**
Dado que el usuario validó la previsualización y procede a timbrar,
Cuando presiona "Generar" o "Timbrar",
Entonces el sistema deberá actuar según el caso: si la línea es Factura PUE desde proforma, timbrar la Factura PUE vía PAC TurboPac (línea pasa a estado Generado); si la línea es Factura PPD desde proforma, timbrar primero la Factura PPD y en cascada el Complemento de Pago asociado (2 CFDIs, línea pasa a estado Generado); si la línea es Complemento de Pago desde factura existente, timbrar el Complemento (línea pasa a estado Generado). Tras timbrado exitoso se muestra el modal de éxito.

**Criterio F2.1 — Cascada Factura PPD + Complemento de Pago desde proforma**
Dado que el usuario elige PPD para una línea con proforma origen y presiona Timbrar,
Cuando el sistema procesa,
Entonces deberá: timbrar primero la Factura PPD; si exitoso, disparar inmediatamente el timbrado del Complemento de Pago asociado; persistir ambos CFDIs; habilitar el envío con destinatario, CC al ESAC, asunto, adjuntos (PDF y XML de ambos CFDIs + Confirmación de Pedido) y notas extras. Si el Complemento falla tras Factura PPD exitosa, notificar y permitir reintento; la Factura PPD permanece vigente.

**Criterio F2.2 — Factura PUE sin generación de Complemento de Pago**
Dado que el usuario elige PUE para una línea con proforma origen y presiona Timbrar,
Cuando el sistema procesa,
Entonces deberá timbrar únicamente la Factura PUE (conforme normativa SAT: facturas PUE son autocontenidas). El envío incluye PDF y XML de la Factura PUE + Confirmación de Pedido.

**Criterio F3 — Acción Enviar**
Dado que una línea está en estado Generado,
Cuando el usuario presiona "Enviar",
Entonces el sistema deberá abrir el modal de envío con: Destinatario (Para, editable, default contacto del pedido); CC (ESAC, editable, default sugerido); Asunto generado por sistema (no editable); Adjuntos (PDF y XML de cada CFDI + Confirmación de Pedido, no editables); Notas extras (text area opcional). Al confirmar, la línea pasa a Enviado.

**Criterio F4 — Operación INDIVIDUAL por línea**
Dado que el usuario opera el Paso 3,
Cuando ejecuta acciones de timbrado o envío,
Entonces deberá operar una línea a la vez. No existen botones de operación masiva.

═══════════════════════════════════════════════════════════════
SECCIÓN G — ESTADOS POR LÍNEA
═══════════════════════════════════════════════════════════════

**Criterio G1 — Estado inicial "Pendiente"**
Dado que el usuario entra al Paso 3,
Cuando el sistema renderiza el estado de cada línea,
Entonces el estado inicial de todas las líneas deberá ser "Pendiente".

**Criterio G2 — Estado tras timbrado exitoso**
Dado que el timbrado de una línea fue exitoso,
Cuando el sistema actualiza el estado,
Entonces la línea pasa a uno de los estados: "Factura Generada", "Factura Anticipo Generada" o "Complemento Generado" según el tipo de documento generado.

**Criterio G3 — Estado tras envío exitoso**
Dado que el envío de una línea al cliente fue exitoso,
Cuando el sistema actualiza el estado,
Entonces la línea pasa al estado "Enviado".

**Criterio G4 — Estados intermedios persistidos si el usuario interrumpe**
Dado que el usuario timbró una línea pero no la envió en la misma sesión,
Cuando regresa al Paso 3,
Entonces el sistema deberá conservar el estado Generado hasta que el usuario complete el envío.

═══════════════════════════════════════════════════════════════
SECCIÓN H — ACCIONES AUTOMÁTICAS AL ENVIAR (SOLO MÉXICO)
═══════════════════════════════════════════════════════════════

**Criterio H1 — Establecimiento de Fecha Estimada de Entrega (FEE)**
Dado que el envío de un documento fue exitoso,
Cuando el sistema confirma el envío,
Entonces deberá disparar automáticamente el establecimiento de la FEE del pedido asociado.

**Criterio H2 — Transferencia a Legacy**
Dado que el envío de un documento fue exitoso,
Cuando el sistema confirma el envío,
Entonces deberá transferir automáticamente al sistema Legacy: el pedido, el documento fiscal generado, las NCs aplicadas y la información del cobro asociado.

**Criterio H3 — Confirmación de Pedido generada y adjunta al envío**
Dado que el envío de una línea de un Pedido Prepago se realiza,
Cuando el sistema arma el correo,
Entonces deberá generar la Confirmación de Pedido e incluirla como adjunto en el mismo correo, junto con los demás CFDIs timbrados de la línea.

**Criterio H4 — Aplica solo a operaciones México**
Dado que las acciones automáticas H1, H2 y H3 se disparan al enviar,
Cuando el cliente es de Región México,
Entonces estas acciones aplicarán. Para Región Perú no aplican (flujos distintos documentados en requisito independiente).

═══════════════════════════════════════════════════════════════
SECCIÓN I — DESTINATARIO Y CUERPO DEL CORREO
═══════════════════════════════════════════════════════════════

**Criterio I1 — Destinatario del envío**
Dado que el sistema arma el correo de envío,
Cuando determina el destinatario,
Entonces deberá usar el contacto del cliente que envió el pedido (heredado del flujo de tramitación del pedido).

**Criterio I2 — Composición del modal de envío**
Dado el modal de envío del Paso 3,
Cuando el sistema lo renderiza,
Entonces deberá mostrar: Para (contacto del cliente, editable); CC (ESAC asignado, editable); Asunto generado por sistema según plantilla por tipo de documento (no editable); Adjuntos (PDF y XML de cada CFDI + Confirmación de Pedido, no editables); y Notas extras (text area opcional).

**Criterio I3 — Asunto del correo según tipo de documento**
Dado el modal de envío del Paso 3,
Cuando el sistema arma el asunto,
Entonces deberá generarlo automáticamente diferenciando: línea con Factura PUE, línea con Factura PPD + Complemento (cascada), y línea con solo Complemento (desde factura existente). ** Plantilla del asunto pendiente de confirmar (PMO #31). Propuesta: "<Folio Pedido Interno> - <Folio Factura>". **

═══════════════════════════════════════════════════════════════
SECCIÓN J — PERSISTENCIA E INMUTABILIDAD
═══════════════════════════════════════════════════════════════

**Criterio J1 — Auto-guardado del Paso 3**
Dado que el usuario opera en el Paso 3,
Cuando modifica Uso CFDI, Método de Pago, ejecuta acciones o navega,
Entonces el sistema deberá auto-guardar el estado del Paso 3. No existe botón "Guardar" manual.

**Criterio J2 — Persistencia del estado y navegación atrás según timbrado**
Dado que el usuario sale del wizard antes de finalizar todas las líneas,
Cuando regresa al módulo Validar Cobro y selecciona al mismo cliente,
Entonces el sistema deberá redirigirlo al Paso 3 con el estado preservado; permitir regresar a Pasos 1 o 2 solo para líneas NO timbradas; bloquear el regreso una vez timbada la línea (documento inmutable).

**Criterio J3 — Inmutabilidad post-timbrado**
Dado que una línea está en estado Generado o Enviado,
Cuando el usuario intenta editar el documento,
Entonces el sistema no deberá permitir edición ni re-timbrado. La única vía es Notas de Crédito (módulo independiente).

═══════════════════════════════════════════════════════════════
SECCIÓN K — MANEJO DE ERRORES Y CIERRE DEL WIZARD
═══════════════════════════════════════════════════════════════

**Criterio K1 — Errores del PAC TurboPac**
Dado que el usuario presiona Timbrar y el PAC responde con error,
Cuando el sistema procesa la respuesta,
Entonces deberá mostrar el mensaje de error con detalle (mismo comportamiento que Factura por Adelantado). La línea permanece en estado Pendiente.

**Criterio K2 — Cierre del wizard**
Dado que todas las líneas del Paso 3 están en estado Enviado,
Cuando el usuario confirma la finalización,
Entonces el sistema deberá cerrar el wizard y retornar al listado principal del módulo. El cliente sale del listado de pendientes.

---

## Notas Adicionales

- Esta fila documenta el Paso 3 del wizard de Validar Cobro (Facturación y Envío) para Región México exclusivamente. La estructura UI tiene diferencias significativas en Perú (sin Factura Anticipo SAT, sin Complemento de Pago 2.0, sin transferencia a Legacy ni Confirmaciones de Pedido, catálogos SUNAT en lugar de SAT) y se documenta en requisito independiente.
- El Paso 3 se invoca desde el Paso 2 al presionar "Continuar" con la asociación cerrada.
- Lógica condicional del tipo de documento fiscal a generar por línea:
  - Proforma normal → Factura nueva (CFDI Ingreso).
  - Proforma con productos controlados → Factura Anticipo (CFDI Ingreso). ~~con tipo de relación 07 SAT — Aplicación de Anticipo~~ **INCORRECTO — DUDA-088:** ver detalle en §resumen (arriba); la Factura Anticipo no lleva relación 07 — eso aplica a la Factura Final (fuera de alcance).
  - Factura existente (de Prepago con Factura por Adelantado previo) con cobro asociado → Complemento de Pago (CFDI Pagos 2.0).
- Productos controlados: PROQUIFA es distribuidor químico-biológico USP/EP/FEUM/Microbiologics. La práctica operativa exige pago anticipado para productos controlados, por lo que se factura como anticipo en lugar de factura estándar.
- Edición del Uso CFDI: combo del catálogo SAT c_UsoCFDI editable por línea antes del timbrado.
- Edición del Método de Pago: editable para proformas (PPD o PUE radio button); fijo PPD no editable para Complementos de Pago (conforme normativa SAT).
- Operación INDIVIDUAL por línea: no existen acciones masivas. Cada línea se previsualiza, timbra y envía una a la vez.
- Flujo operativo recomendado por línea: previsualizar PDF → timbrar (con PAC TurboPac) → enviar al cliente.
- Estados por línea: Pendiente (estado inicial), Factura Generada / Factura Anticipo Generada / Complemento Generado (después del timbrado), Enviado (después del envío al cliente).
- Inclusión automática de NCs en el XML: las NCs aplicadas en el Paso 2 se incluyen en el nodo CFDIRelacionados del CFDI a timbrar, con UUID y monto correspondiente, tipo de relación 01 o 07 según el caso. Conforme Apéndice 5 Anexo 20 versión 4.0 SAT.
- Destinatario del envío: contacto del cliente que envió el pedido (heredado del flujo de tramitación).
- Cuerpo del correo: para Facturas y Facturas Anticipo, misma plantilla que el correo de envío de Factura por Adelantado. Para Complementos de Pago, plantilla pendiente de confirmación.
- Al ENVIAR cada documento (no al timbrar), el sistema dispara automáticamente las acciones post-envío (SOLO MÉXICO):
  - FEE (Fecha Estimada de Entrega): establecimiento de la fecha estimada de entrega del pedido asociado conforme reglas operativas PROQUIFA México.
  - Transferencia a Legacy: envío del pedido, documento fiscal, NCs aplicadas e info del cobro al sistema legado para continuidad operativa (logística, surtido, entrega).
  - Confirmación de Pedido: concepto existente en ProquifaNet para crédito, extendido a prepago en R16. Viaja como adjunto en el correo de envío.
- Las tres acciones automáticas aplican únicamente a operaciones México. Perú tiene flujos distintos.
- Persistencia del Paso 3: si el usuario sale con líneas pendientes, al volver a entrar al mismo cliente el sistema lo redirige directamente al Paso 3 hasta cerrar todas las líneas.
- Inmutabilidad post-timbrado: una vez timbrado un documento, no se permite re-timbrar ni editar. La cancelación fiscal vía CFDI de cancelación SAT no está contemplada en el Paso 3 ni en Validar Cobro.
- Manejo de errores del PAC TurboPac: mismo comportamiento operativo que en Factura por Adelantado.
- Cierre del wizard: cuando todas las líneas están Enviadas, el sistema cierra el wizard y retorna al listado principal.
- ~~Pendiente confirmar el uso del tipo de relación 07 SAT para la Factura Anticipo generada desde proforma con productos controlados.~~ **RESUELTO — DUDA-088:** confirmado que es INCORRECTO usar la relación 07 en la Factura Anticipo; la relación 07 corresponde a la Factura Final (fuera de alcance). Ver `Guia_Tecnica_Facturas_Ingreso_MX.md`.
- ** Pendiente confirmar la plantilla del correo de envío para Complementos de Pago (asunto y cuerpo). Propuesta inicial: asunto "<Folio Pedido Interno> - <Folio Factura>". **
- ** Pendiente definir vía operativa de excepción para casos donde el usuario necesita salir del Paso 3 con líneas pendientes sin posibilidad de continuar (cliente cancela a último minuto, error operativo detectado tarde, etc.). **
- ** Pendiente definición formal de la política de indisponibilidad del PAC TurboPac (transversal con Factura por Adelantado). **
- **(Resuelto DUDA-047, 2026-08-21):** la denominación canónica del rol operativo en la documentación funcional es "Gestor de Cobranza"; se eliminan las menciones a "Analista de Cuentas por Cobrar" como nombre del rol.


---

## Cambios

| # | Fecha | Referencia | Descripción del cambio |
|---|-------|------------|------------------------|
| 1 | 2026-06-10 | OBS-048 | Revisado. La reanudación del wizard en el paso activo ya estaba cubierta en la nota de Persistencia del Paso 3. Sin cambios de contenido. |
| 2 | 2026-08-21 | DUDA-050 | Confirmado: timbrado uno a uno, no masivo. Ya reflejado en el documento; se agrega nota de trazabilidad. |
| 3 | 2026-08-21 | DUDA-088 | **Corrección de error fiscal:** la Factura Anticipo NO usa tipo de relación 07 SAT. Se tachan y corrigen todas las menciones que indicaban lo contrario. La relación 07 corresponde a la Factura Final (fuera de alcance), conforme `Guia_Tecnica_Facturas_Ingreso_MX.md`. |
| 4 | 2026-08-21 | DUDA-089 | Confirmado: el manejo de contacto del pedido para el envío usa el mismo mecanismo existente del sistema; no requiere desarrollo adicional. Se corrige la nota de "pendiente confirmar". |
| 5 | 2026-08-21 | DUDA-047 | Homologada la denominación del rol operativo a "Gestor de Cobranza"; se eliminan las menciones a "Analista de Cuentas por Cobrar" como nombre del rol. |
