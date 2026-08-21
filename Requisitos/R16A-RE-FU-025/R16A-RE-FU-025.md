# R16A-RE-FU-025 — Validar Cobro: Paso 1 Perú

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-025 |
| **Título** | Validar Cobro: Paso 1 Perú |
| **Módulo / Épica** | Validar Cobro |
| **Historia de Usuario** | Yo como **Gestor de Cobranza**, quiero contar con la primera pantalla del wizard de Validar Cobro (Paso 1 - Captura del Cobro) para clientes con Región Perú, donde reviso el correo y el comprobante de pago recibidos y capturo los datos del cobro, para registrar de forma trazable los pagos recibidos de clientes Prepago de Golocaer S.A.C. y continuar con la asociación al documento por cobrar. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.2M-RE-FU-002 |

---

## Requisito Funcional

El sistema debe contar con la primera pantalla del wizard de Validar Cobro (Paso 1 - Captura del Cobro) para clientes con Región Perú, donde el usuario revisa el correo y el comprobante de pago recibidos y captura los datos del cobro. La estructura UI de esta pantalla es la misma que la de México (R16A-RE-FU-024); las diferencias entre regiones son exclusivamente los catálogos de opciones (medio de pago, cuentas destino de Golocaer S.A.C., fuente del tipo de cambio) y el identificador fiscal (RUC). El sistema contempla asistencia automatizada propuesta para el llenado del formulario a partir del análisis del correo y el comprobante, funcionalidad cuyo alcance técnico queda pendiente de definición. Un cobro capturado y confirmado no puede editarse posteriormente. A diferencia de México, en Perú la posterior vinculación del cobro a una factura (Pasos 2 y 3) NO tiene efecto fiscal —no genera Complemento de Pago ni se reporta a SUNAT, porque la factura peruana ya nació completa con su IGV—; la vinculación cobro↔factura se conserva como herramienta operativa interna de conciliación de cobranza, no como acto fiscal. El detalle se desarrolla en los Pasos 2 (R16A-RE-FU-027) y 3 (R16A-RE-FU-029).

---

## Alcance

### Aplica a

- Pantalla del Paso 1 del wizard de Validar Cobro: Captura del Cobro, para clientes con Región Perú.
- Estructura UI idéntica a la de México (R16A-RE-FU-024); las diferencias son exclusivamente catálogos y el identificador fiscal.
- Cabecera del cliente: logo del cliente, razón social, etiquetas preexistentes (clasificación), RUC, razón social legal, moneda de facturación.
- Barra de pasos del wizard visible en cabecera: 1-CAPTURAR COBRO (activo), 2-ASOCIAR FACTURA/PROFORMA, 3-FACTURACIÓN Y ENVÍO.
- Listado de cobros del cliente: items capturados arriba (ordenados por Fecha del Cobro ascendente) e items sin capturar abajo (ordenados por fecha de llegada del correo al Buzón, ascendente).
- Detalle del correo seleccionado: asunto, cuerpo, hora, fecha y adjuntos seleccionables con radio buttons (uno se marca como comprobante de pago oficial).
- Formulario de Datos del Cobro: folio del cobro (formato COB-mmddaa-consecutivo con la fecha efectiva del cobro), monto recibido, fecha del cobro, medio de pago (catálogo interno — ver No aplica a), cuenta origen (texto libre), cuenta destino (cuentas bancarias de Golocaer S.A.C. Perú), moneda del cobro, tipo de cambio del día (vs moneda de facturación), notas del cobro.
- Asistencia automatizada propuesta (no comprometida): auto-completado de los campos a partir del análisis del correo y del comprobante. ** Alcance técnico pendiente de definición. **
- Selección obligatoria del comprobante de pago entre los adjuntos del correo antes de continuar.
- Captura de múltiples cobros del cliente en la misma sesión antes de avanzar al Paso 2.
- Auto-guardado del Paso 1.
- Inmutabilidad del cobro una vez confirmado.
- Marcado de inconsistencias del cobro mediante modal (tipo de inconsistencia + comentario opcional).
- Navegación: Cancelar (regresa al listado principal) o Continuar (avanza al Paso 2 con los cobros capturados).

### No aplica a

- Wizard de Validar Cobro para Región México: se documenta en R16A-RE-FU-024.
- Catálogo SUNAT de medio de pago: SUNAT no exige declarar el medio de pago específico en el comprobante (a diferencia del catálogo SAT c_FormaPago mexicano). ~~En Perú el campo "Medio de pago" del Paso 1 se propone con un catálogo interno de PROQUIFA (transferencia, depósito, cheque, efectivo, etc.) que NO es requerido fiscalmente por SUNAT pero sirve para control interno de Tesorería.~~ *(Propuesta inicial superada — ver DUDA-076: no se trata de reutilizar sin más un catálogo genérico tipo México.)* **Resolución (DUDA-076, cerrada):** el campo se **regionaliza**: "Medio de pago"/"Forma de Pago" queda **obligatorio solo para México**; para Perú **deja de ser obligatorio**. El cliente entregó (18/08) la lista definitiva de medios de pago válidos para Perú, pero como **imagen**, no como texto/listado — por lo tanto el **catálogo exacto de valores para Perú aún no está disponible como texto** y queda **pendiente de captura formal** una vez transcrita esa imagen. ** Pendiente: transcribir el catálogo definitivo Perú desde la imagen del cliente (18/08) y cargarlo en catMedioDePago. **
- Cuentas destino: las cuentas de PROQUIFA México no aplican; se usan las cuentas bancarias de Golocaer S.A.C. Perú. ** Modelo de cuentas bancarias de Golocaer S.A.C. Perú pendiente de definir (ver R16A-RE-FU-006). **
- Fuente del tipo de cambio: el TC FIX Banxico/DOF no aplica; para Perú la fuente del tipo de cambio es la peruana. ** Fuente oficial del TC para Perú pendiente de definir (no aplica el DOF mexicano). **
- Generación de Complemento de Pago a partir de la vinculación del cobro: NO aplica a Perú (SUNAT no tiene Complemento de Pago). La vinculación cobro↔factura es operativa, no fiscal (se desarrolla en R16A-RE-FU-027 y R16A-RE-FU-029).
- Catálogo de Tipos de Inconsistencia (definición de las opciones del combo): ** pendiente del lado de PROQUIFA (Tesorería). **
- Detalle técnico de la asistencia automatizada propuesta: ** pendiente de definición, no comprometido como alcance R16. **
- Edición posterior de un cobro ya capturado y confirmado: NO se permite.

---

## Reglas de Negocio

**Regla 1 — Aplicabilidad solo a Región Perú**
El Paso 1 del wizard de Validar Cobro de este requisito opera exclusivamente sobre clientes con Región Perú. Los clientes con Región México son atendidos por el wizard equivalente de México (R16A-RE-FU-024). La UI es la misma; cambian los catálogos y el identificador fiscal.

**Regla 2 — Cabecera del cliente**
La cabecera del Paso 1 muestra los datos del cliente seleccionado: logo (si existe), Alias, etiquetas de clasificación preexistentes del Catálogo de Clientes, RUC, razón social legal completa y moneda de facturación.

**Regla 3 — Listado de cobros con identificación y orden según estado de captura**
Cada item del listado se muestra según su estado: el item sin capturar (pre-captura) muestra un identificador temporal "COB-N" (consecutivo simple por sesión/cliente) sin datos adicionales hasta que se capture el cobro; el item capturado muestra folio definitivo "COB-mmddaa-NNNN", Fecha del cobro y "Monto del cobro" con su moneda. Si tras la asociación del Paso 2 quedó residual no aplicado, la etiqueta del monto se actualiza a "Saldo a favor". El listado se ordena en dos bloques: primero los capturados (por Fecha del Cobro ascendente), después los sin capturar (por fecha de llegada del correo al Buzón, ascendente).

**Regla 4 — Detalle del correo seleccionado**
Al seleccionar un correo del listado, el detalle muestra: asunto, cuerpo, hora, fecha y los adjuntos del correo. Los adjuntos se presentan como opciones seleccionables (radio buttons) para identificar cuál corresponde al comprobante de pago oficial del cliente.

**Regla 5 — Selección obligatoria del comprobante de pago**
Para continuar al Paso 2, el sistema valida que el usuario haya seleccionado uno de los adjuntos del correo como comprobante de pago. Si no hay selección, el sistema bloquea la continuación y muestra el error correspondiente.

**Regla 6 — Datos del formulario del cobro**
El formulario de captura del cobro ofrece: Folio del cobro (COB-mmddaa-consecutivo, generado por el sistema con la fecha efectiva del cobro); Monto recibido (obligatorio); Fecha del cobro (obligatorio, datepicker, día efectivo del cobro); Medio de pago (~~obligatorio~~ **NO obligatorio para Perú — ver Regla 7 y DUDA-076**, combo de catálogo interno Perú); Cuenta origen (obligatorio, texto libre, cuenta del cliente); Cuenta destino (obligatorio, combo de las cuentas bancarias de Golocaer S.A.C. Perú); Moneda del cobro (obligatorio, combo); Tipo de Cambio (calculado automáticamente con el TC del día vs la moneda de facturación, no modificable); Notas del cobro (opcional, texto libre).

**Regla 7 — Medio de pago regionalizado: NO obligatorio en Perú, catálogo pendiente de transcripción (DUDA-076)**
~~El campo "Medio de pago" del Paso 1 Perú se captura con un catálogo interno de PROQUIFA (transferencia, depósito, cheque, efectivo, etc.) para control de Tesorería... se propone mantenerlo para conciliación interna del cobro.~~ *(Redacción original superada por la resolución de DUDA-076 — no se asume que el campo sea obligatorio ni que el catálogo mexicano/genérico aplique.)*

**Resolución DUDA-076 (cerrada):** el campo "Forma de Pago"/"Medio de pago" se **regionaliza**: es **obligatorio únicamente para México**; en Perú (05/08) **deja de ser obligatorio** — SUNAT no exige declarar el medio de pago específico en el comprobante, por lo que en Perú el dato pasa a ser opcional/informativo para Tesorería, no un requisito de captura. El 18/08 el cliente compartió la lista definitiva de medios de pago válidos para Perú, pero **entregada como imagen**, no como texto. En consecuencia:
- (a) el campo queda regionalizado (obligatorio solo en México);
- (b) para Perú **no es obligatorio**;
- (c) existe ya una lista definitiva del cliente, pero **su transcripción a valores discretos del catálogo (catMedioDePago) sigue pendiente** — no se cuenta con el listado exacto en texto.

** Pendiente/Gap abierto: transcribir el catálogo definitivo de medios de pago Perú a partir de la imagen entregada por el cliente (18/08) y cargarlo en BD. No se debe asumir ni inventar los valores mientras tanto. **

**Regla 8 — Tipo de cambio del día capturado con el cobro**
Cuando la moneda del cobro y/o la moneda de facturación del cliente involucran una moneda distinta a PEN, el sistema calcula automáticamente el TC del día de esa moneda no-PEN respecto a PEN: si el cobro es en PEN con cliente de facturación en moneda extranjera, captura el TC de la moneda de facturación vs PEN; si el cobro es en moneda extranjera, captura el TC de la moneda del cobro vs PEN; si el cobro es en PEN con facturación en PEN, el TC es N/A. El valor es solo lectura. Este TC capturado se conserva con el cobro y se utiliza en el Paso 2 (conversiones operativas) y en el Paso 3 cuando aplique. ** Fuente oficial del TC para Perú pendiente de definir (no aplica el DOF mexicano). **

**Regla 9 — Asistencia automatizada propuesta para captura del cobro**
Sobre un correo del Buzón con su adjunto seleccionado como comprobante de pago, el sistema ofrecería asistencia automatizada propuesta para auto-completar los campos del formulario (monto, fecha, medio de pago, cuenta origen, moneda) extraídos del correo y del comprobante. El usuario siempre tiene la última palabra; los datos sugeridos son editables antes de confirmar. ** El alcance técnico de esta asistencia queda pendiente de definición y NO está comprometido como alcance R16. **

**Regla 10 — Captura de múltiples cobros antes de avanzar**
Tras completar la captura de un cobro, el sistema permite seleccionar otro correo y capturar el siguiente cobro sin avanzar al Paso 2. El Paso 2 se activa cuando el usuario presiona "Continuar".

**Regla 11 — Avance al Paso 2 con o sin captura nueva**
Al presionar "Continuar", el sistema permite avanzar al Paso 2 si existe al menos un cobro registrado (capturado en esta sesión o auto-guardado de sesión previa). El sistema bloquea el avance si no existe ningún cobro registrado para el cliente.

**Regla 12 — Auto-guardado del Paso 1**
Cuando el usuario avanza entre correos, sale de la pantalla o navega a otra parte del sistema, el sistema auto-guarda el estado del Paso 1 (cobros capturados, selecciones de comprobantes, valores del formulario actual). No existe botón de "Guardar" manual; el guardado es transparente.

**Regla 13 — Inmutabilidad del cobro confirmado**
Una vez capturado y confirmado un cobro, el sistema no permite su edición posterior. Antes de confirmar el guardado definitivo, el sistema muestra una alerta indicando que los datos no podrán modificarse después.

**Regla 14 — Marcado de inconsistencia mediante modal**
Al presionar "Marcar Inconsistencia en Cobro", el sistema abre el modal "Inconsistencia de Pago" con dos campos: Tipo de Inconsistencia (combo del catálogo de tipos de inconsistencia) y Comentario adicional (opcional). Botones: Cancelar y Confirmar Inconsistencia.

**Regla 15 — Tipos de inconsistencia aplicables al Paso 1 (cobro como tal)**
La inconsistencia marcada en el Paso 1 se registra contra el cobro como tal (sin contexto de documento por cobrar). Los tipos aplicables corresponden a problemas intrínsecos del cobro recibido: datos incompletos, comprobante inválido, formato incorrecto, monto ilegible, etc. Las inconsistencias derivadas de la relación entre el cobro y un documento por cobrar se detectan en el Paso 2. ** Pendiente definir el catálogo completo de tipos de inconsistencia del Paso 1 del lado de PROQUIFA (Tesorería). **

**Regla 16 — Navegación: Cancelar y Continuar**
El Paso 1 ofrece dos acciones: Cancelar (regresa al listado principal de Validar Cobro, dejando el estado auto-guardado) y Continuar (avanza al Paso 2 con todos los cobros capturados disponibles).

**Regla 17 — La vinculación posterior del cobro no tiene efecto fiscal**
El cobro capturado en el Paso 1 se vinculará posteriormente a un documento (Paso 2) y se procesará en el Paso 3. En Perú esa vinculación NO genera Complemento de Pago ni se reporta a SUNAT (la factura peruana ya se emitió completa con su IGV); es un registro operativo interno de conciliación de cobranza. Esta naturaleza se desarrolla en R16A-RE-FU-027 y R16A-RE-FU-029.

---

## Riesgos

**Riesgo 1 — Catálogo de valores de medio de pago para Perú pendiente de transcripción (DUDA-076)**
DUDA-076 (cerrada) resolvió que el campo se regionaliza y deja de ser obligatorio para Perú, y que el cliente ya entregó (18/08) la lista definitiva de medios de pago válidos — pero como imagen. El riesgo remanente es exclusivamente operativo: transcribir esa imagen a valores discretos del catálogo (catMedioDePago) antes de poder cargarlo en BD; no persiste incertidumbre de negocio sobre obligatoriedad.

**Riesgo 2 — Cuentas bancarias de Golocaer S.A.C. Perú no disponibles**
Las cuentas destino del Paso 1 Perú son las de Golocaer S.A.C., cuyo modelo bancario no está registrado en el sistema (ver R16A-RE-FU-006).

**Riesgo 3 — Fuente del tipo de cambio para Perú no definida**
La fuente oficial del tipo de cambio para Perú no está definida (no aplica el DOF mexicano). Sin ella, el TC capturado con el cobro no puede calcularse automáticamente.

**Riesgo 4 — Catálogo de Tipos de Inconsistencia del Paso 1 pendiente**
El catálogo de tipos de inconsistencia del Paso 1 (cobro como tal) está pendiente de definición por PROQUIFA (Tesorería), transversal con México.

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL CLIENTE Y BARRA DE PASOS
═══════════════════════════════════════════════════════════════

**Criterio A1 — Cabecera del cliente**
Dado que el usuario entra al Paso 1 del wizard para un cliente con Región Perú,
Cuando el sistema renderiza la cabecera,
Entonces deberá mostrar: logo del cliente (si existe), razón social, etiquetas preexistentes del cliente, RUC, razón social legal completa y moneda de facturación.

**Criterio A2 — Barra de pasos del wizard**
Dado que el usuario está en el wizard,
Cuando el sistema renderiza la barra de pasos,
Entonces deberá mostrar los tres pasos: "1 - CAPTURAR COBRO" (activo), "2 - ASOCIAR FACTURA/PROFORMA", "3 - FACTURACIÓN Y ENVÍO".

═══════════════════════════════════════════════════════════════
SECCIÓN B — LISTADO DE COBROS DEL BUZÓN
═══════════════════════════════════════════════════════════════

**Criterio B1 — Identificación del item según estado de captura**
Dado el listado del Paso 1,
Cuando el sistema renderiza cada item,
Entonces si el cobro no ha sido capturado el item muestra únicamente "COB-N" (consecutivo simple pre-captura); y si el cobro ya fue capturado muestra folio definitivo "COB-mmddaa-NNNN", Fecha del cobro y "Monto del cobro" con moneda. Si en el Paso 2 quedó residual no aplicado, la etiqueta del monto cambia a "Saldo a favor".

**Criterio B2 — Orden mixto del listado según estado de captura**
Dado el listado del Paso 1,
Cuando el sistema lo presenta,
Entonces deberá ordenarlo en dos bloques: primero los capturados (por Fecha del Cobro ascendente) y después los sin capturar (por fecha de llegada del correo al Buzón ascendente). Al confirmar la captura, el item se desplaza al bloque de capturados según su Fecha del Cobro.

**Criterio B3 — Modo lectura del item capturado**
Dado un item en estado capturado,
Cuando el sistema lo renderiza,
Entonces deberá mostrarlo en modo lectura solamente, sin acción de edición (consistente con la inmutabilidad post-confirmación).

═══════════════════════════════════════════════════════════════
SECCIÓN C — DETALLE DEL CORREO Y SELECCIÓN DEL COMPROBANTE
═══════════════════════════════════════════════════════════════

**Criterio C1 — Datos del correo seleccionado**
Dado que el usuario selecciona un correo del listado,
Cuando el sistema renderiza el detalle,
Entonces deberá mostrar: contacto del cliente (nombre, correo, teléfono), asunto, cuerpo, fecha, hora y los adjuntos del correo.

**Criterio C2 — Adjuntos seleccionables como comprobante de pago**
Dado que el correo tiene uno o más adjuntos,
Cuando el sistema renderiza la sección de adjuntos,
Entonces deberá presentarlos como opciones seleccionables (radio buttons) para que el usuario identifique cuál corresponde al comprobante de pago oficial. El nombre del archivo de cada adjunto es visible.

**Criterio C3 — Selección obligatoria del comprobante**
Dado que el usuario intenta continuar al Paso 2,
Cuando el sistema valida la selección,
Entonces deberá bloquear la continuación si no ha seleccionado uno de los adjuntos como comprobante de pago.

═══════════════════════════════════════════════════════════════
SECCIÓN D — FORMULARIO DE DATOS DEL COBRO
═══════════════════════════════════════════════════════════════

**Criterio D1 — Folio del cobro**
Dado que el sistema renderiza el formulario,
Cuando muestra el folio,
Entonces deberá mostrar el folio formato COB-mmddaa-consecutivo, con la fecha del día efectivo del cobro. El folio se genera al confirmar la captura. ** Pendiente confirmar si el folio del cobro es por región (consecutivo independiente para México y Perú) o global. **

**Criterio D2 — Monto recibido**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un campo numérico obligatorio para el monto recibido del cliente.

**Criterio D3 — Fecha del cobro**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un datepicker obligatorio. La fecha corresponde al día efectivo en que el cliente realizó el pago (no la fecha de llegada del correo).

**Criterio D4 — Medio de pago (catálogo interno, NO obligatorio en Perú — DUDA-076)**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un combo **NO obligatorio** (regionalización DUDA-076: obligatorio solo en México) con el catálogo interno de medio de pago Perú. SUNAT no exige este dato fiscalmente. El cliente entregó el 18/08 la lista definitiva de valores para Perú, pero como imagen; ** el listado exacto de valores queda pendiente de transcripción formal a partir de esa imagen antes de poder cargar el catálogo en BD. **

**Criterio D5 — Cuenta origen**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un campo de texto libre obligatorio para la cuenta origen del pago (cuenta del cliente).

**Criterio D6 — Cuenta destino**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un combo obligatorio con las cuentas bancarias de Golocaer S.A.C. Perú (cuentas operativas destinadas a recibir cobros). ** Modelo de cuentas bancarias de Golocaer S.A.C. Perú pendiente de definir (ver R16A-RE-FU-006). **

**Criterio D7 — Moneda del cobro**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un combo obligatorio para la moneda en la que el cliente realizó el cobro.

**Criterio D8 — Tipo de cambio del día capturado con el cobro**
Dado que el usuario selecciona la moneda del cobro,
Cuando el sistema renderiza el campo Tipo de Cambio,
Entonces deberá comportarse según el caso: si la moneda del cobro es PEN y la de facturación es PEN, se muestra N/A; si la del cobro es PEN y la de facturación es distinta a PEN, calcula el TC del día de la moneda de facturación vs PEN; si la del cobro es distinta a PEN, calcula el TC del día de la moneda del cobro vs PEN. El valor es solo lectura. El TC capturado se conserva con el cobro y se usa en el Paso 2 y, cuando aplique, en el Paso 3. ** Fuente oficial del TC para Perú pendiente de definir (no aplica el DOF mexicano). **

**Criterio D9 — Notas del cobro**
Dado que el usuario captura el cobro,
Cuando el sistema renderiza el campo,
Entonces deberá ofrecer un campo de texto libre opcional para notas del cobro.

═══════════════════════════════════════════════════════════════
SECCIÓN E — INCONSISTENCIAS, INMUTABILIDAD Y NAVEGACIÓN
═══════════════════════════════════════════════════════════════

**Criterio E1 — Marcado de inconsistencia del cobro**
Dado que el usuario presiona "Marcar Inconsistencia en Cobro",
Cuando el sistema abre el modal "Inconsistencia de Pago",
Entonces deberá ofrecer Tipo de Inconsistencia (combo del catálogo) y Comentario adicional (opcional), con botones Cancelar y Confirmar Inconsistencia. ** Catálogo de tipos de inconsistencia del Paso 1 pendiente de definir (Tesorería). **

**Criterio E2 — Inmutabilidad del cobro confirmado**
Dado que el usuario confirma la captura de un cobro,
Cuando el sistema procesa la confirmación,
Entonces deberá mostrar una alerta indicando que los datos no podrán modificarse después, y una vez confirmado no permitir edición posterior del cobro.

**Criterio E3 — Auto-guardado del Paso 1**
Dado que el usuario navega entre correos, sale de la pantalla o va a otra parte del sistema,
Cuando ocurre la navegación,
Entonces el sistema deberá auto-guardar el estado del Paso 1 sin requerir acción manual del usuario.

**Criterio E4 — Navegación Cancelar y Continuar**
Dado que el usuario está en el Paso 1,
Cuando finaliza su trabajo,
Entonces el sistema deberá ofrecer: Cancelar (regresa al listado principal de Validar Cobro, estado auto-guardado) y Continuar (avanza al Paso 2 si hay al menos un cobro registrado; bloquea el avance si no hay ninguno).

---

## Notas Adicionales

- Fila para el Paso 1 (Captura del Cobro) del wizard de Validar Cobro de la Región Perú, contraparte de R16A-RE-FU-024 (México). Estado depende de la resolución de las brechas Perú.
- La estructura UI es idéntica a la de México; las diferencias son exclusivamente catálogos y el identificador fiscal (RUC).
- Diferencia de fondo vs México — vinculación cobro↔factura sin efecto fiscal: en Perú no existe el Complemento de Pago, por lo que asociar un cobro a una factura NO genera documento fiscal ni se reporta a SUNAT (la factura peruana ya se emitió completa con su IGV). La vinculación se mantiene como herramienta operativa interna de conciliación (qué cobro saldó qué factura, actualización de saldos), no como acto fiscal. Se desarrolla en R16A-RE-FU-027 (asociación) y R16A-RE-FU-029 (facturación/sin efecto fiscal).
- Diferencia — Medio de pago: en México se usa el catálogo SAT c_FormaPago y el campo es obligatorio; en Perú, por resolución de **DUDA-076 (cerrada)**, el campo se regionaliza y **deja de ser obligatorio**. El cliente entregó el 18/08 la lista definitiva de valores para Perú, pero como imagen. ** Pendiente: transcribir el catálogo definitivo Perú desde la imagen (18/08) y cargarlo en catMedioDePago; no inventar valores mientras tanto. **
- **Trazabilidad (2026-08-21):** actualización de este documento conforme a la resolución de DUDA-076 — "Medio de pago" regionalizado (obligatorio solo México, no obligatorio en Perú); catálogo Perú entregado por el cliente el 18/08 como imagen, transcripción de valores pendiente.
- Diferencia — Cuentas destino: cuentas de Golocaer S.A.C. Perú en lugar de PROQUIFA México. ** Modelo bancario pendiente (ver R16A-RE-FU-006). **
- Diferencia — Fuente del tipo de cambio: no aplica el DOF mexicano; fuente peruana pendiente de definir.
- El RUC del cliente proviene del campo `DatosFacturacionCliente.RFC` (tabla preexistente; R16A-RE-FU-004 cancelado — la validación de formato RUC queda pendiente de definir en el requisito que lo implemente).
- ** Pendiente — catálogo de tipos de inconsistencia del Paso 1 (Tesorería), transversal con México. **
- ** Pendiente — confirmar si el folio del cobro (COB-mmddaa-NNNN) es consecutivo por región o global. **
- ** Pendiente — alcance técnico de la asistencia automatizada propuesta (no comprometido en R16). **
- **(Resuelto DUDA-047, 2026-08-21):** la denominación canónica del rol operativo en la documentación funcional es "Gestor de Cobranza"; se eliminan las menciones a "Analista de Cuentas por Cobrar" como nombre del rol.
