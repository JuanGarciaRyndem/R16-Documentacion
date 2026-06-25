# R16A-RE-FU-024 — Validar Cobro: Paso 1 México

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-024 |
| **Título** | Validar Cobro: Paso 1 México |
| **Módulo / Épica** | Validar Cobro |
| **Historia de Usuario** | Yo como **Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente resolver)**, quiero contar con la primera pantalla del wizard de Validar Cobro (Paso 1 - Captura del Cobro) para revisar los correos del Buzón de Cobros del cliente y capturar los datos del cobro recibido o marcar sus inconsistencias, para registrar formalmente cada cobro y prepararlo para su asociación en el Paso 2. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.2M-RE-FU-002 |

---

## Requisito Funcional

El sistema debe contar con la primera pantalla del wizard de Validar Cobro (Paso 1 — Captura del Cobro) para clientes con Región México, donde el usuario revisa el correo y el comprobante de pago recibidos y captura los datos del cobro. El sistema contempla asistencia automatizada propuesta para el llenado del formulario a partir del análisis del correo y el comprobante, funcionalidad cuyo alcance técnico queda pendiente de definición. Un cobro capturado se guarda en modo lectura y, mientras el documento asociado no haya sido timbrado, presenta un botón **Editar** que permite modificar el formulario completo del cobro (monto, fecha, forma de pago, cuentas, moneda, comprobante seleccionado, notas) para corregir errores de captura, aun si el cobro ya está asociado en el Paso 2. La inmutabilidad del cobro se alcanza al timbrar el documento asociado en el Paso 3: a partir de ese momento el cobro queda solo lectura y desaparece el botón Editar. Los cobros con saldo a favor visibles en el Paso 1 corresponden a cobros ya timbrados en sesiones previas, por lo que tampoco presentan botón Editar. La estructura UI de esta pantalla se reutiliza idénticamente para clientes Región Perú; las diferencias entre regiones son exclusivamente los catálogos de opciones y se documentan en requisito independiente.

---

## Alcance

### Aplica a

- Pantalla del Paso 1 del wizard de Validar Cobro: Captura del Cobro.
- Aplica a clientes con Región México exclusivamente (los catálogos y la operación Perú con la misma UI se documentan en requisito independiente).
- Cabecera del cliente (estructura consistente con Factura por Adelantado): logo del cliente, razón social, etiquetas preexistentes (BAJO, REST, etc.), RFC/RUC, razón social legal, moneda de facturación.
- Barra de pasos del wizard visible en cabecera: 1-CAPTURAR COBRO (activo), 2-ASOCIAR FACTURA/PROFORMA, 3-FACTURACIÓN Y ENVÍO.
- Listado de cobros del cliente: items capturados arriba (ordenados por Fecha del Cobro ascendente) e items sin capturar abajo (ordenados por fecha de llegada del correo al Buzón, ascendente).
- Detalle del correo seleccionado: asunto, cuerpo del correo, hora, fecha y adjuntos seleccionables con radio buttons (uno de los adjuntos se marca como comprobante de pago oficial).
- Formulario de Datos del Cobro: folio del cobro (formato COB-mmddaa-consecutivo donde la fecha corresponde al día efectivo del cobro), monto recibido, fecha del cobro, forma de pago (catálogo SAT c_FormaPago), cuenta origen (texto libre), cuenta destino (Catálogo de Cuentas Bancarias del grupo PROQUIFA México), moneda del cobro, tipo de cambio del día (vs moneda de facturación), notas del cobro.
- Asistencia automatizada propuesta (no comprometida): el sistema propondría el auto-completado de los campos del formulario a partir del análisis del correo y del comprobante de pago seleccionado. ** Alcance técnico pendiente de definición. **
- Selección obligatoria del comprobante de pago entre los adjuntos del correo antes de poder continuar.
- Captura de múltiples cobros del cliente en la misma sesión del Paso 1 antes de avanzar al Paso 2.
- Auto-guardado del Paso 1 para preservar lo capturado si el usuario sale o navega entre pantallas.
- Editabilidad del cobro hasta el timbrado: un cobro capturado se guarda en modo lectura y presenta un botón **Editar** mientras el documento asociado no haya sido timbrado (aun si ya está asociado en el Paso 2). Al timbrar en el Paso 3 el cobro queda inmutable y sin botón Editar. Los cobros con "Saldo a favor" visibles en el Paso 1 corresponden a cobros ya timbrados y tampoco presentan botón Editar.
- Marcado de inconsistencias del cobro mediante modal: tipo de inconsistencia (combo del catálogo de tipos de inconsistencia) y comentario opcional.
- Las inconsistencias del Paso 1 aplican únicamente al cobro como tal (datos incompletos, comprobante inválido, formato incorrecto, etc.). Las inconsistencias relativas a la relación entre el cobro y una proforma o factura (como "Pago Incompleto Vencido") se documentan en el Paso 2 cuando ya hay contexto del documento a cobrar.
- Navegación: Cancelar (regresa al listado principal de Validar Cobro) o Continuar (avanza al Paso 2 con los cobros capturados).

### No aplica a

- Wizard de Validar Cobro para Región Perú: se documenta en requisito independiente con la misma estructura UI pero catálogos específicos de Perú (Forma de Pago SUNAT, cuentas destino Golocaer Perú, fuente del TC SBS, etc.).
- Catálogo de Tipos de Inconsistencia (definición de las opciones del combo): ** pendiente del lado de PROQUIFA (Tesorería). **
- Detalle técnico de la asistencia automatizada propuesta (modelo de IA, motor de extracción de datos, integraciones técnicas): ** pendiente de definición, no está comprometido como alcance R16. **
- Edición de un cobro cuyo documento asociado ya fue timbrado en el Paso 3: NO se permite (el cobro queda inmutable a partir del timbrado).

---

## Reglas de Negocio

**Regla 1 — Aplicabilidad solo a Región México**
El Paso 1 del wizard de Validar Cobro opera exclusivamente sobre clientes con Región México. Los clientes con Región Perú son atendidos por el wizard equivalente Perú con la misma UI pero catálogos específicos de Perú (requisito independiente).

**Regla 2 — Cabecera del cliente**
La cabecera del Paso 1 muestra los datos del cliente seleccionado: logo (si existe), Alias, etiquetas de clasificación preexistentes del Catálogo de Clientes, identificador fiscal bajo la etiqueta unificada "RFC/RUC" (con el valor correspondiente según la región del cliente), razón social legal completa y moneda de facturación.

**Regla 3 — Listado de cobros con identificación y orden según estado de captura**
Cada item del listado de cobros se muestra según su estado: el item sin capturar (pre-captura) muestra un identificador temporal "COB-N" (consecutivo simple por sesión/cliente, ej. COB-1, COB-2, COB-3) sin datos adicionales hasta que se capture el cobro; el item capturado muestra folio definitivo "COB-mmddaa-NNNN" (donde mmddaa es la fecha efectiva del cobro), Fecha del cobro y "Monto del cobro" con su moneda. Si tras la asociación del Paso 2 quedó residual no aplicado, la etiqueta del monto se actualiza a "Saldo a favor". El listado se ordena en dos bloques: primero los items capturados, ordenados por Fecha del Cobro de la más antigua a la más reciente; después los items sin capturar, ordenados por fecha de llegada del correo al Buzón de la más antigua a la más reciente.

**Regla 4 — Detalle del correo seleccionado**
Al seleccionar un correo del listado izquierdo, el detalle del correo muestra: asunto, cuerpo, hora, fecha y los adjuntos del correo. Los adjuntos se presentan como opciones seleccionables (radio buttons) para identificar cuál corresponde al comprobante de pago oficial del cliente.

**Regla 5 — Selección obligatoria del comprobante de pago**
Para continuar al Paso 2, el sistema valida que el usuario haya seleccionado uno de los adjuntos del correo como comprobante de pago. Si no hay selección, el sistema bloquea la continuación y muestra el error correspondiente.

**Regla 6 — Datos del formulario del cobro**
El formulario de captura del cobro ofrece los siguientes campos: Folio del cobro (COB-mmddaa-consecutivo, generado por el sistema con la fecha efectiva del cobro); Monto recibido (obligatorio); Fecha del cobro (obligatorio, datepicker; corresponde al día efectivo del cobro); Forma de pago (obligatorio, combo del catálogo SAT c_FormaPago: ejemplo "01 - Efectivo", "02 - Cheque nominativo", "03 - Transferencia electrónica de fondos", etc.); Cuenta origen (obligatorio, texto libre, es la cuenta del cliente); Cuenta destino (obligatorio, combo del Catálogo de Cuentas Bancarias del grupo PROQUIFA México); Moneda del cobro (obligatorio, combo); Tipo de Cambio (calculado automáticamente con el TC del día vs la moneda de facturación del cliente, no modificable por el usuario); Notas del cobro (opcional, texto libre).

**Regla 7 — Tipo de cambio del día capturado con el cobro**
Cuando la moneda del cobro y/o la moneda de facturación del cliente involucran una moneda distinta a MXN, el sistema calcula automáticamente el TC del día actual de esa moneda no-MXN respecto a MXN: si el cobro es en MXN con cliente de facturación en moneda extranjera, captura el TC de la moneda de facturación vs MXN; si el cobro es en moneda extranjera, captura el TC de la moneda del cobro vs MXN; si el cobro es en MXN con cliente de facturación en MXN, el TC es N/A. El valor es solo lectura, no modificable por el usuario. Este TC capturado se conserva con el cobro y se utiliza en el Paso 2 (conversiones operativas) y en el Paso 3 (TipoCambio del CFDI fiscal cuando aplique).

**Regla 8 — Asistencia automatizada propuesta para captura del cobro**
Sobre un correo del Buzón con su adjunto seleccionado como comprobante de pago, el sistema ofrecería asistencia automatizada propuesta para proponer valores de auto-completado de los campos del formulario (monto, fecha, forma de pago, cuenta origen, moneda) extraídos del contenido del correo y del comprobante. El usuario siempre tiene la última palabra sobre los valores capturados; los datos sugeridos son editables antes de confirmar. ** El alcance técnico de esta asistencia (modelo de IA, motor de extracción, integraciones, precisión esperada, comportamiento ante baja confianza) queda pendiente de definición y NO está comprometido como alcance R16. **

**Regla 9 — Captura de múltiples cobros antes de avanzar**
Tras completar la captura de un cobro, el sistema permite al usuario seleccionar otro correo del listado y capturar el siguiente cobro sin necesidad de avanzar al Paso 2. El Paso 2 se activa cuando el usuario explícitamente presiona "Continuar".

**Regla 10 — Avance al Paso 2 con o sin captura nueva**
Al presionar "Continuar", el sistema permite el avance al Paso 2 si existe al menos un cobro registrado (capturado en esta sesión o auto-guardado de sesión previa); no se requiere capturar un cobro nuevo en la sesión actual si ya hay cobros previos disponibles. El sistema bloquea el avance si no existe ningún cobro registrado para el cliente.

**Regla 11 — Auto-guardado y reanudación del wizard en el paso donde se quedó**
Cuando el usuario avanza entre correos del listado, sale de la pantalla o navega a otra parte del sistema, el sistema auto-guarda el estado del Paso 1 (cobros capturados, selecciones de comprobantes, valores del formulario actual) para preservar el progreso. No existe botón de "Guardar" manual; el guardado es transparente al usuario. Al regresar al wizard del mismo cliente (desde "Realizar Cobros" en el listado principal), el sistema retoma la sesión desde el último paso activo en que se encontraba el usuario, no necesariamente desde el Paso 1. Si la sesión previa estaba en el Paso 2 o Paso 3, el wizard abre en ese paso directamente. Decisión confirmada por el cliente — OBS-048.

**Regla 12 — Editabilidad del cobro hasta el timbrado e inmutabilidad post-timbrado**
Un cobro capturado se guarda en modo lectura. Mientras el documento asociado (factura/proforma con el cobro aplicado) NO haya sido timbrado en el Paso 3, el item del listado del Paso 1 presenta un botón **Editar** que abre el formulario del cobro en modo edición y permite modificar cualquier dato del cobro (monto recibido, fecha del cobro, forma de pago, cuenta origen, cuenta destino, moneda, tipo de cambio recalculado según moneda, comprobante de pago seleccionado, notas del cobro). La edición está disponible incluso si el cobro ya está asociado en el Paso 2, para permitir la corrección de errores de captura sin necesidad de eliminar la asociación. Al confirmarse el timbrado del documento asociado en el Paso 3, el cobro queda inmutable, se elimina el botón Editar y el sistema muestra el cobro únicamente en modo lectura. Los cobros visibles en el Paso 1 con etiqueta "Saldo a favor" provienen de cobros ya timbrados en sesiones previas y, por lo tanto, tampoco presentan botón Editar. El sistema no requiere alerta de confirmación al guardar la captura del cobro (la captura no es definitiva hasta el timbrado).

**Regla 13 — Marcado de inconsistencia mediante modal**
Al presionar "Marcar Inconsistencia en Cobro", el sistema abre el modal "Inconsistencia de Pago" con dos campos: Tipo de Inconsistencia (combo del catálogo de tipos de inconsistencia) y Comentario adicional (opcional, texto libre). Botones del modal: Cancelar y Confirmar Inconsistencia.

**Regla 14 — Tipos de inconsistencia aplicables al Paso 1 (cobro como tal)**
La inconsistencia marcada en el Paso 1 se registra contra el cobro como tal (sin contexto de pedido o documento por cobrar, ya que el cobro aún no se ha asociado a ninguna proforma o factura). Los tipos aplicables al Paso 1 corresponden a problemas intrínsecos del cobro recibido: datos incompletos, comprobante inválido, formato incorrecto, monto ilegible, etc. Las inconsistencias derivadas de la relación entre el cobro y un documento a cobrar (por ejemplo, "Pago Incompleto Vencido", "Monto Incorrecto vs Proforma") se detectan en el Paso 2 (Asociación), no aquí. ** Pendiente definir el catálogo completo de tipos de inconsistencia aplicables al Paso 1 (cobro como tal) del lado de PROQUIFA (Tesorería). **

**Regla 15 — Navegación: Cancelar y Continuar**
El Paso 1 ofrece dos acciones: Cancelar (botón al pie de la pantalla) regresa al listado principal de Validar Cobro, dejando el estado del Paso 1 auto-guardado para que el usuario pueda regresar posteriormente; Continuar avanza al Paso 2 (Asociación de Factura/Proforma) con todos los cobros capturados disponibles.

---

## Riesgos

**Riesgo 1 — Catálogo de Tipos de Inconsistencia del Paso 1 pendiente**
El catálogo de tipos de inconsistencia aplicables al Paso 1 (cobro como tal) está pendiente de definición por parte de PROQUIFA (Tesorería). Los tipos aplicables al Paso 1 corresponden a problemas intrínsecos del cobro recibido (datos incompletos, comprobante inválido, formato incorrecto, monto ilegible, etc.). El tipo "Pago Incompleto Vencido" no aplica al Paso 1; se marca en el Paso 2 cuando ya hay contexto del documento a cobrar. ** Pendiente solicitar el catálogo completo al cliente antes del desarrollo. **

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL CLIENTE Y BARRA DE PASOS
═══════════════════════════════════════════════════════════════

**Criterio A1 — Cabecera del cliente**
Dado que el usuario entra al Paso 1 del wizard desde "Realizar Cobros",
Cuando el sistema renderiza la cabecera del cliente,
Entonces deberá mostrar: logo del cliente (si existe), razón social, etiquetas preexistentes del cliente (clasificación), identificador fiscal bajo la etiqueta unificada "RFC/RUC" (con el valor correspondiente al cliente según su región), razón social legal completa y moneda de facturación.

**Criterio A2 — Barra de pasos del wizard**
Dado que el usuario está en el wizard,
Cuando el sistema renderiza la barra de pasos,
Entonces deberá mostrar los tres pasos del wizard: "1 - CAPTURAR COBRO" (activo), "2 - ASOCIAR FACTURA/PROFORMA", "3 - FACTURACIÓN Y ENVÍO".

═══════════════════════════════════════════════════════════════
SECCIÓN B — LISTADO DE COBROS DEL BUZÓN
═══════════════════════════════════════════════════════════════

**Criterio B1 — Identificación del item en el listado según estado de captura**
Dado el listado del Paso 1,
Cuando el sistema renderiza cada item,
Entonces si el cobro no ha sido capturado el item muestra únicamente "COB-N" (consecutivo simple pre-captura por sesión/cliente, ej. COB-1, COB-2); y si el cobro ya fue capturado el item muestra folio definitivo "COB-mmddaa-NNNN", Fecha del cobro y "Monto del cobro" con moneda. Si en el Paso 2 quedó residual no aplicado, la etiqueta del monto cambia a "Saldo a favor".

**Criterio B2 — Orden mixto del listado según estado de captura**
Dado el listado del Paso 1,
Cuando el sistema lo presenta al usuario,
Entonces deberá ordenarlo en dos bloques visualmente continuos: primero los items capturados (ordenados por Fecha del Cobro de la más antigua a la más reciente) y después los items sin capturar (COB-N, ordenados por fecha de llegada del correo al Buzón de la más antigua a la más reciente). Al confirmar la captura de un item, éste se desplaza al bloque de capturados según su Fecha del Cobro.

**Criterio B3 — Modo lectura del item capturado con botón Editar condicionado al estado de timbrado**
Dado un item del listado en estado capturado,
Cuando el sistema lo renderiza,
Entonces deberá mostrarlo en modo lectura. Si el documento asociado al cobro NO ha sido timbrado en el Paso 3, el item deberá presentar un botón **Editar** que abre el formulario del cobro en modo edición y permite modificar cualquier dato (monto, fecha, forma de pago, cuentas, moneda, comprobante, notas), aun si el cobro ya está asociado en el Paso 2. Si el documento asociado ya fue timbrado o si el item corresponde a un cobro con "Saldo a favor" (cobro timbrado previamente), el sistema NO deberá ofrecer el botón Editar y el item permanecerá únicamente en lectura, consistente con la regla de inmutabilidad post-timbrado.

═══════════════════════════════════════════════════════════════
SECCIÓN C — DETALLE DEL CORREO Y SELECCIÓN DEL COMPROBANTE
═══════════════════════════════════════════════════════════════

**Criterio C1 — Datos del correo seleccionado**
Dado que el usuario selecciona un correo del listado,
Cuando el sistema renderiza el detalle,
Entonces deberá mostrar: contacto del cliente (nombre, correo electrónico, teléfono), asunto del correo, cuerpo del correo, fecha, hora y los adjuntos del correo.

**Criterio C2 — Adjuntos seleccionables como comprobante de pago**
Dado que el correo tiene uno o más adjuntos,
Cuando el sistema renderiza la sección de adjuntos,
Entonces deberá presentarlos como opciones seleccionables (radio buttons) para que el usuario identifique cuál corresponde al comprobante de pago oficial. El nombre del archivo de cada adjunto es visible para que el usuario decida.

**Criterio C3 — Selección obligatoria del comprobante**
Dado que el usuario intenta continuar al Paso 2,
Cuando el sistema valida la selección,
Entonces deberá bloquear la continuación si el usuario no ha seleccionado uno de los adjuntos como comprobante de pago.

═══════════════════════════════════════════════════════════════
SECCIÓN D — FORMULARIO DE DATOS DEL COBRO
═══════════════════════════════════════════════════════════════

**Criterio D1 — Folio del cobro**
Dado que el sistema renderiza el formulario,
Cuando muestra el folio,
Entonces deberá mostrar el folio formato COB-mmddaa-consecutivo, donde la fecha se construye con el día efectivo del cobro (campo Fecha del cobro del formulario). El folio se genera al confirmar la captura del cobro. ** Pendiente confirmar si el folio del cobro es por región (consecutivo independiente para México y Perú) o global. **

**Criterio D2 — Monto recibido**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un campo numérico obligatorio para el monto recibido del cliente.

**Criterio D3 — Fecha del cobro**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un datepicker obligatorio. La fecha capturada corresponde al día efectivo en que el cliente realizó el pago (no la fecha de llegada del correo al Buzón).

**Criterio D4 — Forma de pago**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un combo obligatorio del catálogo SAT c_FormaPago (ejemplo de opciones: "01 - Efectivo", "02 - Cheque nominativo", "03 - Transferencia electrónica de fondos", etc.). No aplica la restricción "99 - Por definir" usada en facturas PPD; aquí se captura la forma efectiva del pago realizado.

**Criterio D5 — Cuenta origen**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un campo de texto libre obligatorio para la cuenta origen del pago (cuenta bancaria del cliente desde la cual se realizó el pago).

**Criterio D6 — Cuenta destino**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un combo obligatorio con las cuentas bancarias del Catálogo de Cuentas Bancarias del grupo PROQUIFA México (las cuentas operativas del grupo destinadas a recibir cobros).

**Criterio D7 — Moneda del cobro**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un combo obligatorio para que el usuario seleccione la moneda en la que el cliente realizó el cobro.

**Criterio D8 — Tipo de cambio del día capturado con el cobro**
Dado que el usuario selecciona la moneda del cobro,
Cuando el sistema renderiza el campo Tipo de Cambio,
Entonces deberá comportarse según el caso: si la moneda del cobro es MXN y la moneda de facturación del cliente es MXN, el campo se muestra como N/A; si la moneda del cobro es MXN y la moneda de facturación del cliente es distinta a MXN, el sistema calcula automáticamente el TC del día de la moneda de facturación del cliente respecto a MXN, capturado anticipadamente para conversiones operativas en el Paso 2 con documentos en esa moneda; si la moneda del cobro es distinta a MXN, el sistema calcula automáticamente el TC del día de la moneda del cobro respecto a MXN. El valor del TC es solo lectura, no modificable por el usuario. El TC capturado se conserva con el cobro y se utiliza posteriormente en el Paso 2 (conversiones operativas) y en el Paso 3 (TipoCambio del CFDI fiscal cuando aplique). ** Pendiente confirmar la fuente oficial del TC (propuesta: TC FIX Banxico publicado en DOF). **

**Criterio D9 — Notas del cobro**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un campo de texto libre opcional para notas adicionales del cobro.

═══════════════════════════════════════════════════════════════
SECCIÓN E — ASISTENCIA AUTOMATIZADA PROPUESTA
═══════════════════════════════════════════════════════════════

**Criterio E1 — Propuesta de auto-completado del formulario**
Dado que el usuario seleccionó un comprobante de pago del correo,
Cuando el sistema procesa la información del correo y del comprobante,
Entonces deberá ofrecer asistencia automatizada propuesta (no comprometida) para auto-completar los campos del formulario: monto recibido, fecha del cobro, forma de pago, cuenta origen, moneda. ** El alcance técnico de esta asistencia (motor de extracción, modelo IA, precisión esperada, integraciones) queda pendiente de definición y no está comprometido como alcance R16. **

**Criterio E2 — Edición de los datos sugeridos**
Dado que el sistema propuso valores de auto-completado,
Cuando el usuario revisa los datos,
Entonces deberá poder editar libremente cualquiera de los valores antes de confirmar. La asistencia es propuesta, no obligatoria. El usuario tiene la última palabra sobre los datos capturados.

═══════════════════════════════════════════════════════════════
SECCIÓN F — AUTO-GUARDADO, EDITABILIDAD HASTA EL TIMBRADO E INMUTABILIDAD POST-TIMBRADO
═══════════════════════════════════════════════════════════════

**Criterio F1 — Auto-guardado y reanudación del wizard en el paso donde se quedó**
Dado que el usuario captura datos en el formulario,
Cuando avanza entre correos del listado, sale de la pantalla o navega el sistema,
Entonces el sistema deberá auto-guardar el estado del Paso 1 (cobros capturados, selecciones de comprobantes, valores del formulario actual). No existe botón "Guardar" manual; el guardado es transparente. Al regresar al wizard del mismo cliente, el sistema deberá retomar la sesión desde el último paso activo en que se encontraba el usuario: si la sesión previa había avanzado al Paso 2 o Paso 3, el wizard deberá abrirse directamente en ese paso sin forzar al usuario a recorrer los pasos anteriores nuevamente. Decisión confirmada por el cliente — OBS-048.

**Criterio F2 — Captura de múltiples cobros antes de avanzar**
Dado que el cliente tiene varios correos pendientes,
Cuando el usuario finaliza la captura de un cobro,
Entonces el sistema deberá permitir al usuario seleccionar otro correo del listado y capturar el siguiente cobro. El Paso 2 se activa solo al presionar "Continuar".

**Criterio F3 — Edición de un cobro capturado mientras el documento asociado no haya sido timbrado**
Dado un cobro ya capturado y guardado en el listado del Paso 1,
Cuando el documento asociado a ese cobro aún NO ha sido timbrado en el Paso 3 (sin importar si el cobro ya está asociado en el Paso 2 o todavía no),
Entonces el item deberá presentar un botón **Editar** que al presionarse abre el formulario del cobro en modo edición y permite modificar todos los campos del cobro (monto recibido, fecha del cobro, forma de pago, cuenta origen, cuenta destino, moneda, tipo de cambio recalculado según la moneda, selección de comprobante de pago, notas del cobro). Al guardar los cambios el item regresa a modo lectura y los nuevos valores se reflejan en cualquier asociación vigente del Paso 2. El sistema no deberá exigir alerta de confirmación al guardar (la captura no es definitiva hasta el timbrado).

**Criterio F4 — Inmutabilidad del cobro al timbrar el documento asociado**
Dado un cobro asociado a un documento que ha sido timbrado en el Paso 3,
Cuando el usuario regresa al Paso 1 o consulta el listado de cobros del cliente,
Entonces el sistema NO deberá ofrecer botón Editar para ese cobro y el item permanecerá únicamente en modo lectura. Aplica el mismo comportamiento a los cobros con etiqueta "Saldo a favor" visibles en el Paso 1, ya que provienen de cobros timbrados previamente.

═══════════════════════════════════════════════════════════════
SECCIÓN G — MODAL DE INCONSISTENCIA DE PAGO
═══════════════════════════════════════════════════════════════

**Criterio G1 — Apertura del modal**
Dado que el usuario detecta inconsistencias en el cobro,
Cuando presiona "Marcar Inconsistencia en Cobro" en el pie de la pantalla,
Entonces el sistema deberá abrir el modal "Inconsistencia de Pago".

**Criterio G2 — Campos del modal**
Dado que el modal está abierto,
Cuando el sistema renderiza los campos,
Entonces deberá ofrecer: Tipo de Inconsistencia (obligatorio, combo del catálogo de tipos de inconsistencia aplicables al Paso 1) y Comentario adicional (opcional, texto libre para describir detalle adicional al cliente). ** El catálogo del Paso 1 está pendiente de definición por PROQUIFA. Los tipos aplicables al Paso 1 son inconsistencias intrínsecas del cobro (datos incompletos, comprobante inválido, formato incorrecto, etc.); no incluye "Pago Incompleto Vencido" que se marca en el Paso 2. **

**Criterio G3 — Acciones del modal**
Dado que el modal está abierto,
Cuando el usuario finaliza la captura,
Entonces deberá ofrecer dos acciones: Cancelar (cierra el modal sin guardar la inconsistencia) y Confirmar Inconsistencia (registra la inconsistencia en el cobro).

**Criterio G4 — Alcance de las inconsistencias del Paso 1**
Dado que el usuario marca una inconsistencia en el Paso 1,
Cuando confirma la inconsistencia,
Entonces el sistema deberá registrar la inconsistencia contra el cobro como tal. Las inconsistencias del Paso 1 no incluyen el caso "Pago Incompleto Vencido" ni otras inconsistencias que requieran conocer el documento contra el que se aplicaría el cobro (proforma o factura); esas inconsistencias se marcan en el Paso 2 (Asociación) cuando ya hay contexto del documento a cobrar. ** El catálogo específico de tipos de inconsistencia aplicables en cada paso queda pendiente de definición por PROQUIFA (Tesorería). **

═══════════════════════════════════════════════════════════════
SECCIÓN H — NAVEGACIÓN DEL PASO 1
═══════════════════════════════════════════════════════════════

**Criterio H1 — Botón Cancelar**
Dado que el usuario opera el Paso 1,
Cuando presiona "Cancelar" al pie de la pantalla,
Entonces el sistema deberá regresar al listado principal de Validar Cobro. El estado del Paso 1 queda auto-guardado para permitir retomar la captura posteriormente.

**Criterio H2 — Botón Continuar: condiciones de habilitación**
Dado el Paso 1,
Cuando el sistema evalúa el botón "Continuar",
Entonces si existe al menos un cobro registrado (capturado en esta sesión o auto-guardado de sesión previa) el botón está habilitado y al presionarlo avanza al Paso 2 con todos los cobros disponibles para asociación; si no existe ningún cobro registrado para el cliente el botón está deshabilitado. La captura nueva en la sesión actual no es obligatoria si ya hay cobros previos capturados.

---

## Notas Adicionales

- Esta fila documenta el Paso 1 del wizard de Validar Cobro (Captura del Cobro) para Región México exclusivamente. La estructura UI de la pantalla se reutiliza idénticamente para clientes Región Perú; las únicas diferencias son los catálogos de opciones (Forma de Pago SAT vs SUNAT, cuentas bancarias grupo PROQUIFA México vs Golocaer Perú, fuente del TC, etc.) y se documentan en requisito independiente.
- El wizard se invoca desde la pantalla principal de Validar Cobro (Listado de clientes con pendientes) al presionar "Realizar Cobros" en un cliente. Es una pantalla nueva, no modal.
- La cabecera del cliente sigue la estructura consistente del proyecto (logo, Alias, etiquetas preexistentes de clasificación, identificador fiscal bajo etiqueta unificada "RFC/RUC", razón social legal, moneda de facturación), análoga a la cabecera de Factura por Adelantado.
- El listado de correos del Buzón de Cobros del cliente se ordena del más antiguo al más reciente, por fecha de llegada al Buzón.
- El folio del cobro tiene formato COB-mmddaa-consecutivo (confirmado por PMO #40), donde la fecha corresponde al día efectivo del cobro (no a la fecha de llegada del correo al Buzón). Esto genera una ambigüedad operativa en cómo identificar el correo en el listado antes de capturar el cobro (cuando aún no se conoce la fecha efectiva). ** Decisión pendiente. **
- Selección obligatoria del comprobante de pago: el correo del Buzón puede traer múltiples adjuntos (comprobante real + otros archivos como notas o referencias); el usuario debe identificar explícitamente cuál es el comprobante oficial mediante radio button.
- Forma de pago del cobro: catálogo SAT c_FormaPago (no aplica la restricción "99 - Por definir" que rige facturas PPD; aquí se captura la forma efectiva del pago realizado).
- Cuenta origen: texto libre (cuenta del cliente desde donde pagó).
- Cuenta destino: combo del Catálogo de Cuentas Bancarias del grupo PROQUIFA México (mismo catálogo usado en Proforma y Factura).
- Tipo de Cambio: del día, de la moneda no-MXN involucrada respecto a MXN (moneda base SAT, requisito del CFDI). El sistema decide qué moneda capturar según el caso: la moneda del cobro si es distinta a MXN, o la moneda de facturación del cliente si el cobro es MXN pero el cliente factura en otra moneda. Si ambas son MXN, el campo se muestra como N/A. El TC capturado sirve como referencia única para las conversiones del Paso 2 y para el TipoCambio del CFDI en el Paso 3. La fuente estándar fiscal mexicana es el TC FIX publicado por Banxico en el DOF.
- ** DUDA ABIERTA — Moneda base del TC capturado: actualmente se utiliza MXN como moneda base de todas las conversiones (consistente con la regla SAT del CFDI). Pendiente validar con asesor fiscal y PROQUIFA si esta es la opción correcta o si en algún escenario operativo conviene capturar el TC en sentido distinto (por ejemplo, vs moneda de facturación del cliente). **
- Asistencia automatizada propuesta: funcionalidad exploratoria para auto-completar los datos del formulario a partir del correo y el comprobante. No está comprometida como alcance R16; el alcance técnico queda pendiente de definición. El usuario siempre tiene la última palabra sobre los valores capturados.
- Auto-guardado del Paso 1: el progreso se preserva si el usuario sale o navega entre pantallas. No existe botón "Guardar" manual; el guardado es transparente al usuario.
- Editabilidad del cobro hasta el timbrado: un cobro capturado se guarda en modo lectura y, mientras el documento asociado no haya sido timbrado en el Paso 3, presenta un botón **Editar** que permite modificar el formulario completo del cobro (monto, fecha, forma de pago, cuentas, moneda, comprobante, notas) para corregir errores de captura, aun si el cobro ya está asociado en el Paso 2. Al timbrar en el Paso 3, el cobro queda inmutable y se elimina el botón Editar. Los cobros con etiqueta "Saldo a favor" en el Paso 1 corresponden a cobros ya timbrados en sesiones previas y, por lo tanto, tampoco presentan botón Editar.
- Captura de múltiples cobros antes de avanzar: el usuario puede capturar varios cobros del cliente en la misma sesión del Paso 1 sin avanzar al Paso 2 hasta que decida explícitamente.
- Modal "Inconsistencia de Pago" del Paso 1: permite marcar inconsistencias intrínsecas del cobro (datos incompletos, comprobante inválido, formato incorrecto, monto ilegible, etc.). Tipo de inconsistencia (combo) y comentario opcional. El catálogo de tipos del Paso 1 está pendiente de definición por PROQUIFA (Tesorería).
- Las inconsistencias del Paso 1 aplican únicamente al cobro como tal. Las inconsistencias derivadas de la relación entre el cobro y un documento a cobrar (proforma o factura), como "Pago Incompleto Vencido", se marcan en el Paso 2 (Asociación), no aquí.
- ** Pendiente confirmar si el folio del cobro (COB-mmddaa-consecutivo) es por región (consecutivo independiente para México y Perú) o global del grupo. **
- ** Pendiente clarificar la identificación visual del correo en el listado antes de la captura (cuando aún no se conoce la fecha efectiva del cobro). Propuestas: usar la fecha de llegada del correo, mostrar identificador temporal del Buzón, generar el folio definitivo solo al confirmar la captura, asignar folio provisional, u otra alternativa. **
- ** Pendiente definir el catálogo completo de Tipos de Inconsistencia aplicables al Paso 1 del lado de PROQUIFA (Tesorería). Tipos aplicables al Paso 1: inconsistencias intrínsecas del cobro (datos incompletos, comprobante inválido, formato incorrecto, etc.). El tipo "Pago Incompleto Vencido" no aplica al Paso 1 (se marca en el Paso 2 cuando ya hay contexto del documento a cobrar). **
- ** Pendiente definir el alcance técnico de la asistencia automatizada propuesta para auto-completar el formulario (modelo de IA o motor de extracción, integración, precisión esperada, comportamiento ante baja confianza). No comprometido como alcance R16. **
- ** Pendiente confirmar la fuente oficial del Tipo de Cambio del día para México (propuesta estándar fiscal: TC FIX publicado por Banxico en el DOF). Confirmar si PROQUIFA utiliza esta fuente o tiene una fuente propia. **
- ** Pendiente resolver formalmente la denominación canónica del rol operativo entre "Gestor de Cobranza" y "Analista de Cuentas por Cobrar" (transversal). **

---

## Cambios

| # | Fecha | Referencia | Descripción del cambio |
|---|-------|------------|------------------------|
| 1 | 2026-06-10 | OBS-048 | Regla 11 y Criterio F1 actualizados: el wizard reanuda desde el último paso activo al volver al cliente, no desde el Paso 1. |
| 2 | 2026-06-23 | Actualización funcional | Cambio en la editabilidad del cobro del Paso 1: la inmutabilidad pasa de "al confirmar la captura" a "al timbrar". Un cobro capturado se guarda en modo lectura y, mientras el documento asociado no haya sido timbrado, presenta un botón **Editar** que permite editar el formulario completo (monto, fecha, forma de pago, cuentas, moneda, comprobante, notas) aun si ya está asociado en el Paso 2. Al timbrar en el Paso 3 queda inmutable y sin botón Editar. Los cobros con "Saldo a favor" visibles en el Paso 1 corresponden a cobros ya timbrados y tampoco presentan botón Editar. Afecta: Requisito Funcional, Regla 12, Criterios B3, F3, F4 y encabezado de la Sección F. |
