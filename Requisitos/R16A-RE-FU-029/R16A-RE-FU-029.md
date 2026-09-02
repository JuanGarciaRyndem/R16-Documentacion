# R16A-RE-FU-029 — Validar Cobro: Paso 3 Perú

> ⚠️ **FUERA DE ALCANCE DE R16 — Cancelación de facturación Perú (2026-08-21)**
> Este requisito queda fuera de alcance de R16: la facturación electrónica de Perú (timbrado ante SUNAT, Factura electrónica CPE tipo 01) se CANCELÓ POR COMPLETO. Así lo documenta la resolución/respuesta de las dudas cerradas DUDA-092, DUDA-093 y DUDA-094 (hoja "R16 - Dudas a Cliente", todas Descartadas por este mismo motivo raíz). Todo el contenido de este documento —incluyendo Reglas de Negocio, Riesgos y Criterios de Aceptación referidos al timbrado/envío de facturas en Perú— queda sin efecto y pendiente de re-evaluación de alcance (¿se mantiene únicamente el envío de la Confirmación de Pedido para Perú, sin documentos fiscales? — ver DUDA-094). El campo "Estado" de la tabla siguiente no ha sido actualizado a nivel de gestión de requisito; se deja constancia aquí para no perder trazabilidad.

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-029 |
| **Título** | Validar Cobro: Paso 3 Perú |
| **Módulo / Épica** | Validar Cobro |
| **Historia de Usuario** | Yo como **Gestor de Cobranza / Analista de Cuentas por Cobrar (denominación pendiente de resolver)**, quiero contar con la tercera pantalla del wizard de Validar Cobro (Paso 3 - Facturación y Envío) para clientes con Región Perú, donde por cada documento de la asociación cerrada en el Paso 2 genero, timbro ante SUNAT y envío la factura electrónica de forma individual, para emitir los comprobantes de los cobros conciliados de Golocaer S.A.C. y cerrar el ciclo de cobranza cumpliendo la normativa SUNAT. |
| **Prioridad** | Alta |
| **Estado** | Propuesto |
| **Requisito asociado** | R16.2M-RE-FU-002 |

---

## Requisito Funcional

El sistema debe contar con la tercera pantalla del wizard de Validar Cobro (Paso 3 - Facturación y Envío) para clientes con Región Perú, donde por cada documento de la asociación cerrada en el Paso 2 el usuario previsualiza, timbra ante SUNAT y envía la factura electrónica de forma individual. A diferencia de México, en Perú el único tipo de documento fiscal que se genera es la Factura electrónica (CPE tipo 01): no existe Factura Anticipo con tipo de relación SAT ni Complemento de Pago (SUNAT no tiene esos mecanismos). Al enviar cada documento, el sistema establece la Fecha Estimada de Entrega (FEE) del pedido y genera la Confirmación de Pedido; NO ejecuta transferencia a Legacy (de Perú nada va a Legacy). El usuario debe completar todas las líneas del cliente antes de cerrar el ciclo. La estructura UI de esta pantalla es la misma que la de México (R16A-RE-FU-028); las diferencias son los catálogos SUNAT y las simplificaciones del modelo peruano.

---

## Alcance

### Aplica a

- Pantalla del Paso 3 del wizard de Validar Cobro: Facturación y Envío, para clientes con Región Perú.
- Estructura UI idéntica a la de México (R16A-RE-FU-028); las diferencias son catálogos SUNAT y las simplificaciones del modelo peruano.
- Cabecera del cliente (consistente con Paso 1 y Paso 2: logo, Alias, etiquetas preexistentes, RUC, razón social legal, moneda de facturación).
- Barra de pasos del wizard: 1-CAPTURAR COBRO (✓), 2-ASOCIAR factura/PROFORMA (✓), 3-FACTURACIÓN Y ENVÍO (activo).
- Listado de líneas a procesar, una por cada documento de la asociación cerrada en el Paso 2.
- Tipo de documento fiscal único: Factura electrónica (CPE tipo 01, UBL 2.1). En Perú NO hay lógica condicional de tres caminos como en México: todas las líneas que requieren emisión generan una Factura electrónica normal.
- Por línea: Tipo de Operación (catálogo 51 SUNAT) y Condición de Pago (Contado/Crédito), en lugar del Uso CFDI y Método de Pago del SAT.
- Operación INDIVIDUAL por línea: previsualizar, timbrar y enviar una línea a la vez. No existen acciones masivas.
- Modal Previsualizar: PDF representativo del documento antes del timbrado.
- Modal de éxito de generación: tras timbrado exitoso, confirmación con Serie y Correlativo timbrados.
- Modal Enviar: envío del documento al cliente (PDF + XML de la factura + Confirmación de Pedido generada al enviar), con destinatario el contacto del cliente y CC al ESAC.
- Al ENVIAR cada documento, el sistema establece la Fecha Estimada de Entrega (FEE) del pedido como dato operativo y genera la Confirmación de Pedido (que se adjunta en el correo, sin previsualización).
- Timbrado del CPE ante SUNAT.
- Estados por línea: Pendiente → Factura Generada (tras timbrado) → Enviado (tras envío).
- Auto-guardado y persistencia del Paso 3 (si el usuario sale, regresa al Paso 3 al volver al cliente, hasta que todas las líneas estén en estado Enviado).
- Navegación atrás según estado de timbrado: se permite regresar a los Pasos 1 o 2 mientras la línea NO se haya timbrado; una vez timbrada, no se permite regresar (documento inmutable).
- Inmutabilidad post-timbrado: una factura timbrada no se re-timbra ni se edita; la única vía de corrección posterior es la Nota de Crédito SUNAT (R16A-RE-FU-033/035).
- Cierre del wizard cuando todas las líneas están en estado Enviado.

### Pendientes estructurales de esta fila (a validar con el cliente)

- ** Escenario de cobro contra factura ya emitida: al no existir Complemento de Pago en Perú, no hay documento que generar ni enviar; pendiente confirmar qué acciones debe ofrecer el sistema (solo conciliación interna, alguna constancia no fiscal, u otra). Ver Regla 4. **
- ** Parámetros que se configuran al pasar de proforma a factura en Perú: candidatos Condición de Pago y Tipo de Operación; confirmar si hay otros (detracción/percepción cuando aplique, tipo de cambio). Ver Regla 5. **
- ** Mecánica de referencia de las Notas de Crédito peruanas (catálogo 09 SUNAT) en la emisión: se define en R16A-RE-FU-033/035. Ver Regla 6. **
- ** Modalidad de emisión electrónica ante SUNAT pendiente de definir. NO se da por hecho el uso de un OSE: SUNAT ofrece cuatro modalidades (SEE-SOL portal gratuito, SEE del Contribuyente, SEE-OSE con Operador de Servicios Electrónicos, y Facturador SUNAT); la elección depende, entre otros factores, de si Golocaer S.A.C. es gran contribuyente. La modalidad y, en su caso, el proveedor, quedan pendientes; no se reutiliza el PAC de México. Brecha mayor (ver Brecha 5 de R16A-RE-FU-005). **
- ** Formato del asunto y plantilla del cuerpo del correo de envío para Perú. **

### No aplica a

- Wizard de Validar Cobro para Región México: se documenta en R16A-RE-FU-028.
- Factura Anticipo con tipo de relación 07 SAT: NO existe en Perú. La mecánica peruana de anticipos (glosa "Pago Anticipado" + PrepaidAmount) es distinta y, conforme al análisis del proyecto, no se requiere en este flujo porque la factura de venta peruana no depende del dato aduanero (la DUA es trámite separado). Ver análisis en R16A-RE-FU-020.
- Complemento de Pago (CFDI Pagos 2.0): NO existe en Perú. Cuando el cobro corresponde a una factura ya emitida (Factura por Adelantado previa), en Perú NO se genera ningún documento fiscal: solo se registra la conciliación interna del cobro contra esa factura (operativa, no fiscal). Ver Regla 4.
- Uso CFDI y Método de Pago PPD/PUE: conceptos del SAT que no aplican a SUNAT.
- Transferencia a Legacy: de Perú nada se transfiere a Legacy.
- PAC TurboPac: no se reutiliza la solución de México; la modalidad de emisión electrónica para Perú está pendiente de definir (ver Pendientes estructurales).
- Inclusión de NCs en nodo CFDIRelacionados SAT: la mecánica de referencia de la NC peruana (catálogo 09 SUNAT) se define en R16A-RE-FU-033/035.
- Cancelación fiscal de documentos timbrados: en Perú se realiza vía Nota de Crédito SUNAT (R16A-RE-FU-033/035), fuera del Paso 3.
- Generación masiva (Timbrar todo / Enviar todo): la operación es individual por línea.

---

## Reglas de Negocio

**Regla 1 — Aplicabilidad solo a Región Perú**
El Paso 3 de este requisito opera exclusivamente sobre clientes con Región Perú. Los clientes con Región México son atendidos por el wizard equivalente de México (R16A-RE-FU-028). La UI es la misma; cambian los catálogos y las simplificaciones del modelo peruano.

**Regla 2 — Líneas a procesar derivadas del Paso 2**
El Paso 3 genera una línea por cada documento (proforma o factura) asociado en el Paso 2 que requiera emisión de un documento fiscal. Cada línea referencia: tipo de documento origen, folio del documento origen, fecha, Pedido Interno, emisor (Golocaer S.A.C.), monto total, tipo de cambio (cuando aplique) y NCs aplicadas (si las hubo).

**Regla 3 — Tipo de documento fiscal único: Factura electrónica**
A diferencia de México (que tiene tres caminos: Factura, Factura Anticipo y Complemento de Pago), en Perú el único documento fiscal que se genera es la Factura electrónica (CPE tipo 01, UBL 2.1). No existe Factura Anticipo con relación 07 SAT ni Complemento de Pago. Toda línea que parte de una proforma genera una Factura electrónica normal.

**Regla 4 — Vinculación de cobro a factura existente sin documento fiscal**
Cuando el cobro del Paso 2 corresponde a una factura ya emitida (por ejemplo, una Factura por Adelantado previa), en Perú NO se genera ningún documento fiscal en el Paso 3 (no hay Complemento de Pago). El sistema solo registra internamente la conciliación del cobro contra esa factura y actualiza el saldo. ** Pendiente confirmar con el cliente qué acciones debe ofrecer el sistema en este escenario, ya que no hay documento que generar ni enviar: ¿solo registrar la conciliación y cerrar la línea sin acción de envío?, ¿enviar al cliente alguna constancia/notificación interna no fiscal de que el pago fue aplicado? Actualmente este escenario quedaría sin acciones de salida hacia el cliente. **

**Regla 5 — Tipo de Operación y Condición de Pago por línea**
Por cada línea, el sistema consigna el Tipo de Operación (catálogo 51 SUNAT) y la Condición de Pago (Contado/Crédito), en lugar del Uso CFDI y el Método de Pago del SAT. ** Pendiente definir si el Tipo de Operación es configurable por el operador o lo fija el sistema (ver R16A-RE-FU-020). Pendiente también confirmar qué parámetros se configuran al pasar de proforma a factura en Perú: candidatos Condición de Pago y Tipo de Operación; confirmar si hay otros (ej. detracción/percepción cuando aplique, tipo de cambio). **

**Regla 6 — Inclusión de Notas de Crédito aplicadas**
Cuando en el Paso 2 se aplicaron Notas de Crédito a un documento, estas se reflejan en la emisión de la factura conforme a la mecánica SUNAT. ** La forma de referenciar la NC peruana (catálogo 09 SUNAT) en el comprobante se define en R16A-RE-FU-033/035 y queda pendiente de validar. **

**Regla 7 — Flujo operativo por línea: previsualizar, timbrar, enviar**
Por cada línea, el usuario puede: Previsualizar el PDF del documento antes de timbrar; Generar/Timbrar el documento ante SUNAT/OSE; y Enviar el documento al cliente. El flujo es secuencial: no se puede enviar sin haber timbrado.

**Regla 8 — Estados por línea**
Cada línea transita por los estados: Pendiente (inicial) → Factura Generada (tras timbrado exitoso) → Enviado (tras envío al cliente). En Perú solo existe el estado "Factura Generada" (no "Factura Anticipo Generada" ni "Complemento Generado", que son de México).

**Regla 9 — Operación individual por línea**
El sistema no ofrece acciones masivas (no "Timbrar todo", no "Enviar todo"). El usuario procesa una línea a la vez.

**Regla 10 — Acciones automáticas al enviar**
Al enviar cada documento, el sistema establece la Fecha Estimada de Entrega (FEE) del pedido asociado como dato operativo y genera la Confirmación de Pedido, que se adjunta en el mismo correo (no se previsualiza; solo se genera y se muestra como adjunto en el modal). A diferencia de México, esto NO se acompaña de transferencia a Legacy (de Perú nada va a Legacy).

**Regla 11 — Sin transferencia a Legacy**
En Perú, el envío del documento NO dispara transferencia a Legacy. La FEE y la Confirmación de Pedido SÍ se generan (igual que en México). Solo la transferencia a Legacy es exclusiva del flujo mexicano.

**Regla 12 — Destinatarios del envío**
El modal de envío presenta como destinatario (Para) el contacto del cliente del pedido (editable) y como CC el ESAC asignado al cliente/pedido (editable). Adjuntos: PDF y XML de la factura timbrada y la Confirmación de Pedido (no removibles). Asunto generado automáticamente con Serie y Correlativo de la factura y el Pedido Interno. ** Formato del asunto y plantilla del cuerpo para Perú pendientes de confirmar. **

**Regla 13 — Modal Previsualizar**
Antes de timbrar, el usuario puede abrir el modal de Previsualización que muestra el PDF representativo del documento para validación visual. En este punto la factura aún no se ha timbrado ante SUNAT/OSE.

**Regla 14 — Modal de éxito tras timbrado**
Tras el timbrado exitoso, el sistema muestra un modal de confirmación con la Serie y Correlativo timbrados. NO se muestra Folio Fiscal UUID (es un concepto del SAT).

**Regla 15 — Persistencia del Paso 3 y navegación atrás según estado de timbrado**
El estado del Paso 3 se auto-guarda. La posibilidad de regresar a los Pasos 1 o 2 depende del estado de timbrado de las líneas: SÍ se permite regresar mientras la(s) línea(s) NO se hayan timbrado (el documento está en borrador/corrección); NO se permite regresar una vez timbrada la factura, aunque falte enviarla (la factura timbrada es inmutable). Si el usuario sale del wizard, al volver a entrar al mismo cliente desde Validar Cobro el sistema lo regresa al Paso 3 con el estado preservado.

**Regla 16 — Inmutabilidad post-timbrado**
Una vez una línea está en estado Factura Generada, no se permite re-timbrar ni editar el documento. La única vía de corrección posterior es la Nota de Crédito SUNAT (R16A-RE-FU-033/035).

**Regla 17 — Manejo de errores de timbrado SUNAT/OSE**
Cuando el timbrado falla (error de validación, servicio indisponible, datos inválidos, error SUNAT), el sistema muestra un mensaje de error al usuario con el detalle del problema, mismo comportamiento operativo que el módulo Factura por Adelantado (R16A-RE-FU-020).

**Regla 18 — Auto-guardado del Paso 3**
El sistema auto-guarda el estado del Paso 3 (líneas procesadas, estados, datos) sin requerir acción manual del usuario.

**Regla 19 — Cierre del wizard**
El wizard se considera cerrado para el cliente cuando todas las líneas del Paso 3 están en estado Enviado.

---

## Riesgos

**Riesgo 1 — Escenario de cobro contra factura existente sin acciones definidas**
Cuando el cobro corresponde a una factura ya emitida, en Perú no hay Complemento de Pago que generar ni enviar, por lo que el escenario podría quedar sin acciones de salida hacia el cliente. Si no se define qué debe hacer el sistema, la línea podría quedar sin un cierre claro (ver Regla 4).

**Riesgo 2 — Brecha de timbrado SUNAT no resuelta**
La habilitación para Región Perú depende de la integración con la modalidad de emisión electrónica que defina Golocaer S.A.C. ante SUNAT, brecha mayor no resuelta del proyecto (ver Brecha 5 de R16A-RE-FU-005). La modalidad está pendiente de definir entre las cuatro disponibles (SEE-SOL, SEE del Contribuyente, SEE-OSE, Facturador SUNAT); no se da por hecho el uso de un OSE ni se reutiliza el PAC de México. Mientras no se resuelva, no es posible timbrar.

**Riesgo 3 — Parámetros de configuración proforma→factura no definidos**
No está confirmado qué parámetros se configuran al pasar de proforma a factura en Perú (Condición de Pago, Tipo de Operación, y posibles otros). Sin esa definición, el Paso 3 no puede armar correctamente el comprobante.

**Riesgo 4 — Bloqueo de navegación tras timbrado**
Una vez timbrada una línea, no se puede regresar a los Pasos 1 o 2 (la factura es inmutable); solo resta enviarla y, si hay error, corregir vía Nota de Crédito SUNAT. Si una línea quedó timbrada pero el envío no puede completarse, el cliente podría quedar detenido en el Paso 3. Antes de timbrar, en cambio, el usuario sí puede regresar a corregir.

---

## Criterios de Aceptación

═══════════════════════════════════════════════════════════════
SECCIÓN A — CABECERA DEL CLIENTE Y BARRA DE PASOS
═══════════════════════════════════════════════════════════════

**Criterio A1 — Cabecera del cliente**
Dado que el usuario entra al Paso 3 para un cliente con Región Perú,
Cuando el sistema renderiza la cabecera,
Entonces deberá mostrar: logo del cliente (si existe), razón social, etiquetas preexistentes, RUC, razón social legal y moneda de facturación.

**Criterio A2 — Barra de pasos del wizard**
Dado que el usuario está en el Paso 3,
Cuando el sistema renderiza la barra de pasos,
Entonces deberá mostrar "1 - CAPTURAR COBRO" (✓), "2 - ASOCIAR factura/PROFORMA" (✓) y "3 - FACTURACIÓN Y ENVÍO" (activo).

═══════════════════════════════════════════════════════════════
SECCIÓN B — LISTADO DE LÍNEAS A PROCESAR
═══════════════════════════════════════════════════════════════

**Criterio B1 — Una línea por cada documento de la asociación**
Dado que la asociación del Paso 2 quedó cerrada,
Cuando el sistema arma el Paso 3,
Entonces deberá mostrar una línea por cada documento que requiera emisión de factura, con emisor Golocaer S.A.C.

**Criterio B2 — Datos visibles de cada línea**
Dado que el sistema renderiza una línea,
Cuando muestra sus datos,
Entonces deberá mostrar: tipo de documento origen, folio del documento origen, fecha, Pedido Interno, emisor (Golocaer S.A.C.), monto total de la factura, tipo de cambio (cuando aplique), NCs aplicadas (si las hubo), estado actual de la línea (Pendiente / Factura Generada / Enviado), y los campos fiscales seleccionables de la línea (Tipo de Operación y Condición de Pago — ver Sección D, pendientes de validar para Perú).

═══════════════════════════════════════════════════════════════
SECCIÓN C — TIPO DE DOCUMENTO FISCAL (ÚNICO: factura)
═══════════════════════════════════════════════════════════════

**Criterio C1 — Toda línea con emisión genera Factura electrónica**
Dado que una línea requiere emisión de documento fiscal,
Cuando el sistema determina el tipo de documento,
Entonces deberá generar una Factura electrónica (CPE tipo 01, UBL 2.1). En Perú no hay lógica condicional de tres caminos: no se generan Factura Anticipo (relación 07 SAT) ni Complemento de Pago.

**Criterio C2 — Cobro contra factura existente: sin documento fiscal**
Dado que el cobro corresponde a una factura ya emitida (por ejemplo Factura por Adelantado previa),
Cuando el sistema procesa la línea,
Entonces NO deberá generar ningún documento fiscal (no hay Complemento de Pago); solo registrará la conciliación interna del cobro contra esa factura y actualizará el saldo. ** Pendiente confirmar qué acciones ofrece el sistema en este escenario, dado que no hay documento que generar ni enviar (ver Regla 4). **

═══════════════════════════════════════════════════════════════
SECCIÓN D — TIPO DE OPERACIÓN Y CONDICIÓN DE PAGO
═══════════════════════════════════════════════════════════════

**Criterio D1 — Tipo de Operación por línea**
Dado que el sistema arma la factura de una línea,
Cuando incluye los datos fiscales,
Entonces deberá consignar el Tipo de Operación (catálogo 51 SUNAT), en lugar del Uso CFDI mexicano. ** Pendiente definir si es configurable por el operador o lo fija el sistema (ver R16A-RE-FU-020). **

**Criterio D2 — Condición de Pago por línea**
Dado que el sistema arma la factura de una línea,
Cuando incluye los datos fiscales,
Entonces deberá consignar la Condición de Pago (Contado/Crédito), en lugar del Método de Pago PPD/PUE del SAT. ** Pendiente confirmar qué parámetros se configuran al pasar de proforma a factura en Perú (candidatos: Condición de Pago, Tipo de Operación; confirmar si hay otros como detracción/percepción o tipo de cambio). **

═══════════════════════════════════════════════════════════════
SECCIÓN E — NOTAS DE CRÉDITO APLICADAS
═══════════════════════════════════════════════════════════════

**Criterio E1 — Reflejo de las NCs aplicadas en el Paso 2**
Dado que en el Paso 2 se aplicaron Notas de Crédito a un documento,
Cuando el sistema emite la factura,
Entonces deberá reflejar las NCs aplicadas conforme a la mecánica SUNAT. ** La forma de referenciar la NC peruana (catálogo 09 SUNAT) se define en R16A-RE-FU-033/035 y queda pendiente de validar. **

═══════════════════════════════════════════════════════════════
SECCIÓN F — FLUJO PREVISUALIZAR, TIMBRAR, ENVIAR
═══════════════════════════════════════════════════════════════

**Criterio F1 — Acción Previsualizar**
Dado que una línea está en estado Pendiente,
Cuando el usuario presiona Previsualizar,
Entonces el sistema deberá mostrar el PDF representativo del documento antes del timbrado. La factura aún no se ha timbrado ante SUNAT/OSE.

**Criterio F2 — Acción Timbrar**
Dado que el usuario confirma el timbrado de una línea,
Cuando el sistema envía la solicitud a SUNAT/OSE,
Entonces deberá timbrar la Factura electrónica. No hay generación en cascada (no existe Complemento de Pago en Perú).

**Criterio F3 — Acción Enviar**
Dado que una línea está timbrada (estado Factura Generada),
Cuando el usuario presiona Enviar,
Entonces el sistema deberá abrir el modal de envío para mandar el documento (PDF + XML de la factura + Confirmación de Pedido) al cliente.

**Criterio F4 — Operación individual por línea**
Dado que hay varias líneas en el Paso 3,
Cuando el usuario opera,
Entonces el sistema deberá procesar una línea a la vez; no existen acciones masivas (no "Timbrar todo", no "Enviar todo").

═══════════════════════════════════════════════════════════════
SECCIÓN G — ESTADOS POR LÍNEA
═══════════════════════════════════════════════════════════════

**Criterio G1 — Estado inicial "Pendiente"**
Dado que una línea se crea en el Paso 3,
Cuando aún no se timbra,
Entonces su estado deberá ser "Pendiente".

**Criterio G2 — Estado tras timbrado exitoso**
Dado que una línea se timbra exitosamente,
Cuando SUNAT/OSE confirma,
Entonces su estado deberá pasar a "Factura Generada" (único estado de generación en Perú; no existen "Factura Anticipo Generada" ni "Complemento Generado").

**Criterio G3 — Estado tras envío exitoso**
Dado que una línea timbrada se envía al cliente,
Cuando el envío se completa,
Entonces su estado deberá pasar a "Enviado".

**Criterio G4 — Estados persistidos si el usuario interrumpe**
Dado que el usuario interrumpe el Paso 3,
Cuando regresa al cliente,
Entonces el sistema deberá conservar el estado de cada línea (Pendiente / Factura Generada / Enviado) y regresarlo al Paso 3.

═══════════════════════════════════════════════════════════════
SECCIÓN H — ACCIONES AUTOMÁTICAS AL ENVIAR
═══════════════════════════════════════════════════════════════

**Criterio H1 — Establecimiento de la Fecha Estimada de Entrega (FEE)**
Dado que el usuario envía un documento,
Cuando el envío se completa,
Entonces el sistema deberá establecer la Fecha Estimada de Entrega (FEE) del pedido asociado como dato operativo.

**Criterio H2 — Generación y envío de la Confirmación de Pedido**
Dado que el usuario envía un documento,
Cuando el sistema arma el correo de envío,
Entonces deberá generar la Confirmación de Pedido del pedido asociado y adjuntarla en el mismo correo, igual que en México. La Confirmación de Pedido no se previsualiza: solo se genera y se muestra como archivo adjunto en el modal de envío.

**Criterio H3 — Sin transferencia a Legacy**
Dado que el usuario envía un documento en Perú,
Cuando se ejecutan las acciones post-envío,
Entonces el sistema NO deberá transferir el pedido ni los documentos a Legacy. La transferencia a Legacy es exclusiva del flujo mexicano; de Perú nada va a Legacy.

═══════════════════════════════════════════════════════════════
SECCIÓN I — DESTINATARIO Y CUERPO DEL CORREO
═══════════════════════════════════════════════════════════════

**Criterio I1 — Destinatario y CC del envío**
Dado que el usuario abre el modal de envío,
Cuando el sistema arma los destinatarios,
Entonces deberá poner como Para el contacto del cliente del pedido (editable) y como CC el ESAC asignado (editable).

**Criterio I2 — Adjuntos y asunto**
Dado que el usuario abre el modal de envío,
Cuando el sistema arma el correo,
Entonces deberá adjuntar el PDF y el XML de la factura timbrada y la Confirmación de Pedido (no removibles), y pre-rellenar el asunto con Serie, Correlativo y Pedido Interno. ** Formato del asunto y plantilla del cuerpo para Perú pendientes de confirmar. **

═══════════════════════════════════════════════════════════════
SECCIÓN J — PERSISTENCIA, INMUTABILIDAD, ERRORES Y CIERRE
═══════════════════════════════════════════════════════════════

**Criterio J1 — Auto-guardado, persistencia y navegación atrás según timbrado**
Dado que el usuario opera o interrumpe el Paso 3,
Cuando navega o sale,
Entonces el sistema deberá auto-guardar el estado; permitir regresar a los Pasos 1 o 2 únicamente mientras la línea NO se haya timbrado; bloquear el regreso una vez timbrada (factura inmutable); y, al volver a entrar al cliente, llevarlo de nuevo al Paso 3 con el estado preservado.

**Criterio J2 — Inmutabilidad post-timbrado**
Dado que una línea está en estado Factura Generada,
Cuando el usuario intenta modificarla,
Entonces el sistema NO deberá permitir re-timbrar ni editar el documento. La única vía de corrección posterior es la Nota de Crédito SUNAT (R16A-RE-FU-033/035).

**Criterio J3 — Manejo de errores de timbrado SUNAT/OSE**
Dado que el timbrado de una línea falla,
Cuando SUNAT/OSE retorna error (validación, indisponibilidad, datos inválidos),
Entonces el sistema deberá mostrar un mensaje de error con el detalle, mismo comportamiento que el módulo Factura por Adelantado.

**Criterio J4 — Cierre del wizard**
Dado que todas las líneas del Paso 3 están en estado Enviado,
Cuando el usuario completa la última,
Entonces el sistema deberá considerar el wizard cerrado para ese cliente.

---

## Notas Adicionales

- Fila para el Paso 3 (Facturación y Envío) del wizard de Validar Cobro de la Región Perú, contraparte de R16A-RE-FU-028 (México). Estado depende de la resolución de las brechas Perú.
- La estructura UI es idéntica a la de México; las diferencias son catálogos SUNAT y las simplificaciones del modelo peruano.
- Diferencia clave — un solo tipo de documento: en Perú toda línea con emisión genera una Factura electrónica (CPE tipo 01). NO existen los tres caminos mexicanos (Factura / Factura Anticipo relación 07 / Complemento de Pago).
- Factura Anticipo: no aplica en Perú; la factura de venta peruana no depende del dato aduanero (la DUA es trámite separado) y la Ley del IGV permite facturar por el monto cobrado anticipadamente (ver análisis en R16A-RE-FU-020).
- Complemento de Pago: no existe en Perú. El cobro contra una factura ya emitida no genera documento fiscal; solo conciliación interna (operativa, no fiscal). ** Acciones del sistema en ese escenario pendientes de confirmar (ver Regla 4). **
- FEE (Fecha Estimada de Entrega) y Confirmación de Pedido: SÍ aplican a Perú, igual que en México. La Confirmación de Pedido se genera al enviar y se adjunta en el correo (no se previsualiza). Lo único que NO ocurre en Perú es la transferencia a Legacy.
- Navegación atrás: se permite regresar a los Pasos 1 o 2 mientras la línea no se haya timbrado; una vez timbrada, queda bloqueado el regreso (documento inmutable), aunque falte enviar.
- Catálogos SUNAT: Tipo de Operación (catálogo 51) y Condición de Pago (Contado/Crédito) en lugar de Uso CFDI y Método de Pago PPD/PUE del SAT.
- Inmutabilidad post-timbrado: corrección solo vía Nota de Crédito SUNAT (R16A-RE-FU-033/035).
- ** Brecha — Modalidad de emisión electrónica ante SUNAT pendiente de definir. NO se da por hecho el uso de un OSE: SUNAT ofrece cuatro modalidades (SEE-SOL, SEE del Contribuyente, SEE-OSE, Facturador SUNAT); la elección depende, entre otros factores, de si Golocaer S.A.C. es gran contribuyente. La modalidad y, en su caso, el proveedor, quedan pendientes; no se reutiliza el PAC de México. Ver Brecha 5 de R16A-RE-FU-005. **
- ** Pendiente — parámetros de configuración proforma→factura en Perú (Condición de Pago, Tipo de Operación, posibles detracción/percepción, tipo de cambio). **
- ** Pendiente — formato del asunto y plantilla del cuerpo del correo de envío para Perú. **
- ** Pendiente — mecánica de referencia de la NC peruana (catálogo 09 SUNAT) en la emisión; se define en R16A-RE-FU-033/035. **
- ** Pendiente — maquetas de Validar Cobro Perú no disponibles; el detalle de la pantalla se validará contra ellas cuando lleguen. **
- ** [2026-08-21] FUERA DE ALCANCE — La facturación de Perú se cancela por completo. Cierre formal documentado en DUDA-092 (escenario de cobro contra factura existente, Regla 4), DUDA-093 (parámetros proforma→factura, Regla 5) y DUDA-094 (asunto/plantilla de correo, Regla 12), las tres Descartadas por el mismo motivo raíz. Este requisito y sus Pendientes estructurales quedan sin objeto; solo podría subsistir el envío de la Confirmación de Pedido para Perú (sin documentos fiscales), pendiente de definición formal de alcance. **
