# Historial de cambios — Reenfoques Paquete 01

**Fecha de consolidación:** 2026-08-14
**Origen:** consolidado de las notas de reenfoque enviadas por Roberto Baez Muñoz entre el 27-jul-2026 y el 14-ago-2026.
**Alcance:** requisitos R16A-RE-FU-001, 002, 003, 004, 005, 006, 007, 009 y 032.

---

## Resumen ejecutivo

| Requisito | Naturaleza del cambio | Impacto |
|---|---|---|
| **RE-FU-001** | Reenfoque cuentas GOLPERU | Bajo — solo Alcance, Criterio A2 y Riesgos |
| **RE-FU-002** | Precisión de módulos Perú + corrección alcance dos campos | Medio — Requisito, Alcance, Criterios C1/C2 y Observaciones |
| **RE-FU-003** | Cierre de rol y funciones + corrección permisos | Medio — 9 puntos alineados |
| **RE-FU-004** | Reenfoque completo | **Alto** — reescritura total del requisito |
| **RE-FU-005** | Retiro Perú + regionalización Forma de Pago + agregar Uso de CFDI | **Alto** — reescritura total |
| **RE-FU-006** | Perú + facturas + encadenamiento Empresa + unicidad + CV solo Banamex | **Alto** — reescritura total |
| **RE-FU-007** | Cierre de dudas de diseño | Bajo — 3 cierres |
| **RE-FU-009** | Retiro separación + validación como último paso de Tramitar Pedido | **Alto** — reescritura total; el módulo pasa de Pretramitar a Tramitar Pedido |
| **RE-FU-032** | Reenfoque masivo (22 bloques) | **Muy alto** — reescritura total |

---

## RE-FU-001 — Mantenimiento de catálogos del sistema (Cuentas Bancarias)

**Fecha origen:** 27-jul-2026

### Cambios aplicados
- **Alcance "No aplica a":** se retira el punto que declaraba el modelo de cuentas de Golocaer S.A.C. (Perú) como brecha sin definir.
- **Alcance "Aplica a":** se agrega el punto de las cuentas de Golocaer S.A.C. (Perú) indicando que su modelo ya está definido, que pueden poblarse y que en Perú la Proforma es el consumidor.
- **Alcance "Aplica a"** (empresas del grupo): se agrega Golocaer S.A.C. como empresa del grupo Perú, entidad distinta de Golocaer México.
- **Riesgos:** eliminado el Riesgo 1 "Modelo de cuentas bancarias Perú no definido". El Riesgo 2 se renumera como Riesgo 1.
- **Criterio A2:** se agrega Golocaer S.A.C. al listado de empresas emisoras consultables.
- **Observaciones:** se retira el bullet del estado del catálogo Perú (cubierto en Alcance).
- **Documentos de referencia del cliente:** se agrega el documento de conceptos de cuentas bancarias por región.

### Riesgo residual
Ninguno.

---

## RE-FU-002 — Mantenimiento de Catálogo de Clientes (Cobrador y Coordinador de Tesorería)

**Fecha origen:** 27-jul-2026

### Cambios aplicados
- **Requisito:** se precisa que para clientes de Región Perú el filtrado por cartera opera únicamente en **Validar Cobro y Buzón de Pagos** (no existe FAA ni NC en esa región).
- **Alcance "Aplica a":** se sustituyen los bullets del campo único "Cobrador" por la descripción de los **dos campos** (Cobrador + Coordinador de Tesorería), con sus selectores, permisos de edición y bloqueo.
- **Alcance "Aplica a":** se precisan los módulos aplicables por región.
- **Alcance "Aplica a":** la restricción de vaciado se extiende a ambos campos.
- **Alcance "No aplica a":** se agrega la exclusión del filtrado por cartera en FAA y NC para Perú. La exclusión de asignación múltiple se extiende a ambos campos.
- **Criterios C1 y C2:** se precisan los módulos aplicables por región.
- **Observaciones:** se precisa el bullet de módulos consumidores por región.

### Nota
El Alcance no se había actualizado al incorporarse el campo Coordinador de Tesorería, que ya estaba contemplado en el Requisito, las Reglas 7 a 9 y los Criterios D1 a D5.

---

## RE-FU-003 — Documentos Regulatorios del Cliente

**Fecha origen:** 27-jul-2026 · Cierre de pendientes: 05-ago-2026

### Cambios aplicados
- **Cierre del pendiente de rol y función:** se definen las funciones **Gestor de Asuntos Regulatorios y Contenido** y **Auxiliar de Asuntos Regulatorios**, y el rol **Gestor de la Información, Regulatorios**.
- **Historia de Usuario:** se alinea al rol y las funciones definidas.
- **Requisito:** se precisa que las dos funciones son las autorizadas para cargar, reemplazar y eliminar; cualquier otro usuario con acceso al catálogo puede visualizar y consultar el contenido.
- **Alcance "Aplica a":** se alinean las menciones al rol con las dos funciones autorizadas.
- **Reglas 1 y 4, Riesgo 3, Criterios A2, B1, B2, D1, D2 y E1:** se alinean las menciones al rol/función.
- **Corrección de consistencia (Observaciones):** se corrige el bullet de permisos que listaba "visualizar el contenido" como operación exclusiva del rol — contradecía al Alcance y a los Criterios A2 y C1. La visualización del contenido está disponible para cualquier usuario con acceso al catálogo.
- **Observaciones:** se cierran los pendientes de confirmación del cliente sobre permisos y de definición del rol y la función.

### Cierres
- Rol y función del área Regulatorios (pendiente desde matriz).
- Confirmación del cliente sobre propuesta de permisos.

---

## RE-FU-004 — Actualización de catálogos Régimen Fiscal y Tipo de Sociedad Mercantil

**Fecha origen:** 29-jul-2026 · Cierre de pendientes: 07-ago-2026

### Reenfoque completo
- **Historia y Requisito:** reescritos para centrarse en la actualización de catálogos, retirando captura/validación de información fiscal (funcionalidad ya existente).
- **Alcance "Aplica a" y "No aplica a":** reescritos completos. Se declara fuera de alcance la regionalización de campos y catálogos, y los campos/validaciones ya existentes.
- **Criterios de Aceptación:** reescritos completos. Se retiran las reglas y criterios de obligatoriedad de campos, validación del RFC, validación del RUC, algoritmo Módulo 11, etiquetas dinámicas por región, bloqueo de guardado y persistencia.
- **Nuevas Reglas 1–6:** alcance, tipos de actualización, formato de despliegue de cada catálogo, despliegue sin diferenciación por región y origen de la actualización.
- **Riesgo 1:** acotado al catálogo del SAT.
- **Riesgo 2:** agregado — clientes con valores dados de baja (cerrado con la Regla 7).
- **Sección A:** 5 criterios de despliegue.
- **Cierre de pendientes** (07-ago-2026): Regla 7 y Observaciones cierran la confirmación del cliente con el documento *R16 - Catálogos Fiscales*; Regla 7 nueva de tratamiento mediante curaduría; Riesgo 2 cerrado.
- **Documentos de referencia:** *R16 - Catálogos Fiscales* + archivo de equivalencias del análisis previo.

### Cierres
- Confirmación del cliente sobre las opciones de los catálogos.
- Tratamiento de clientes con opciones dadas de baja.
- Validación del RUC (sin objeto al no regionalizarse los campos).
- Denominación "Sociedad por Acciones Cerrada Simplificada" (sin objeto).
- Categoría tributaria "Régimen para Personas Naturales" (sin objeto).

---

## RE-FU-005 — Actualización de catálogos Forma de Pago y Uso de CFDI

**Fecha origen:** 31-jul-2026 · Incorporación Uso de CFDI y regionalización: 07-ago-2026 · Cierre listado: 07-ago-2026

### Bloque 1 — Retiro de Región Perú
- **Historia, Requisito, Alcance y Criterios:** reescritos retirando particularidades fiscales de Perú (Condición de Pago, Tipo de Comprobante, Agente de Retención IGV, Sujeto a Detracción, presentación diferenciada por región).
- Se retiran las reglas de catálogos diferenciados por región, MetodoPago aplicable solo a México y banderas tributarias.
- Se retira el bloque de brechas para facturación electrónica Perú.

### Bloque 2 — Reenfoque
- El requisito pasa a cubrir únicamente el despliegue del catálogo actualizado de Forma de Pago (los campos ya existen en el sistema).
- Alta/baja/modificación ejecutadas directamente en BD; sin interfaz gráfica.
- Nueva Regla 2 (gestión sin UI) y reglas 1, 3–6 con alcance, formato, correspondencia con clave del comprobante y origen.
- Nuevo Riesgo 2 (clientes con valores dados de baja).

### Bloque 3 — Incorporación del catálogo Uso de CFDI (07-ago)
- Se agrega el catálogo de Uso de CFDI, que se actualiza junto con Forma de Pago.
- Formato: Forma de Pago con guion; Uso de CFDI sin guion.
- Nueva Regla 4 (formato Uso de CFDI).
- Nuevo Criterio A3 (selector de Uso de CFDI).

### Bloque 4 — Regionalización y obligatoriedad Forma de Pago (07-ago)
- Regionalización del catálogo de Forma de Pago (MEX y PER con listas propias).
- Obligatoriedad solo para clientes de Región México.
- Nuevas Reglas 5 (regionalización) y 6 (obligatoriedad).
- Nuevo Criterio A2 (selector regionalizado) y nueva Sección B con B1 (México) y B2 (Perú).

### Bloque 5 — Cierre del listado de catálogos (07-ago)
- Regla 8 cierra la confirmación del cliente con el documento *R16 - Catálogos Fiscales*.
- Regla 9 nueva de tratamiento mediante curaduría del cliente.

### Cierres
- Mapeo del catálogo de Forma de Pago a las claves del SAT (formato clave + concepto).
- Campo "Tipo de Revisión" (corresponde al flujo de crédito, sin impacto R16).
- Bandera Sujeto a Detracción (SPOT no aplica a PROQUIFA).
- Bandera Agente de Retención de IGV, Agente de Percepción y denominación Condición de Pago (sin objeto al retirarse Perú).
- Confirmación del cliente sobre las opciones de los catálogos.
- Tratamiento de clientes con valores dados de baja.

---

## RE-FU-006 — Referencia de Pago y Código Validador

**Fecha origen:** 03-ago-2026 · Cierres 07-ago-2026

### Bloque 1 — Incorporación de Región Perú
- Historia y Requisito incorporan el mecanismo Perú: referencia con nombre del cliente sin Código Validador.
- Alcance pasa a México y Perú con mecanismos diferenciados.
- Nueva **Regla 9** (referencia Perú); nuevo **Criterio C4**.
- Riesgo de modelo Perú no definido eliminado; riesgos restantes renumerados como Riesgo 1 y Riesgo 2.

### Bloque 2 — Alcance a proformas y facturas
- La referencia bancaria se muestra en proformas **y facturas** emitidas al cliente.
- Reglas 4, 5 y 9: extienden el casado al PDF de la factura.
- Criterios C1–C4 extendidos a la generación de facturas.

### Bloque 3 — Encadenamiento desde Empresa
- Selección de Empresa como punto de partida (determina bancos disponibles, y éstos las cuentas).
- Regla 2 reescrita con el encadenamiento completo y herencia de Moneda y Sucursal.
- Regla 8 reescrita: encadenamiento desde empresa con las tres condiciones confirmadas.
- Sección A: nuevos A2 (Empresa) y A3 (Banco filtrado); A4 y A5 renumerados con Moneda heredada.

### Bloque 4 — Código Validador condicionado a Banamex
- CV obligatorio solo cuando la cuenta es de Banamex; bloqueado para otros bancos.
- Regla 3 reescrita para condicionar la captura al banco.
- Regla 6 precisa el establecimiento automático de referencia para no-Banamex.
- Sección B desdoblada: B1 (Banamex obligatorio), B2 (otros bancos bloqueado); criterios renumerados a partir de B3.

### Bloque 5 — Unicidad de la combinación cliente-cuenta
- Alcance precisa unicidad y excluye la asignación repetida.
- Regla 2 incorpora la unicidad.
- Criterio B4 acotado a cuentas distintas.
- Nuevo Criterio B5 (unicidad); criterios renumerados como B6 y B7.
- Criterio B6 precisa que la eliminación libera la cuenta para volver a asignarse.

### Cierres
- Longitud del Código Validador: 3 caracteres alfanumérico, sin acentos ni espacios en blanco.
- Tope de cuentas asignables por cliente: no se limita.
- Restricción de rol: fuera del alcance de R16.
- Condición de moneda truncada: cerrada con las tres condiciones confirmadas.
- Historial del Código Validador: se conservan valor vigente y valor inmediatamente anterior con autor y fecha (corrige exclusión previa).
- Regla 7 segmento 4: se precisa que la clave del cliente corresponde al campo `Clave` de la tabla; se ejemplifica el relleno de ceros.
- Razón social del cliente como dato empleado al construir la referencia (Reglas 4, 6, 7, 9; Criterios B2, B3, C2, C3 y C4).
- La clave del cliente forma parte de la composición de la referencia (Regla 7 segmento 4 y Reglas 4 y 9).

---

## RE-FU-007 — Notificación regulatoria en Cotización Definitiva

**Fecha origen:** 05-ago-2026

### Cambios aplicados
- **Ubicación de la leyenda en el PDF:** se define internamente como decisión de diseño (Ryndem/PROQUIFA), sin bloqueo externo. Se actualizan Criterio B1 y Observaciones.
- **Texto de la leyenda:** se define internamente como decisión de diseño; se conserva la propuesta base como referencia. Se actualizan Criterio C1 y Observaciones.
- **Variante dinámica de la leyenda:** se descarta; la leyenda se muestra siempre de forma genérica cuando la cotización definitiva contenga productos controlados.

### Corrección de redacción
- Corrección del paréntesis sin cerrar en la enumeración de los documentos regulatorios (verificar en el archivo actual — puede haber quedado corregido en revisión previa).

---

## RE-FU-009 — Validación regulatoria

**Fecha origen:** 13-ago-2026 · Aplicado: 14-ago-2026

### Bloque 1 — Retiro de la separación de partidas por documentación regulatoria
- **Historia:** reescrita retirando la tramitación de partidas no controladas sin esperar la documentación de las controladas.
- **Requisito:** reescrito para establecer la retención del pedido completo hasta que la documentación se registre, retirando la separación de partidas y la dependencia de la bandera de entregas parciales.
- **Alcance "Aplica a":** se sustituye la tramitación de partidas retenidas en pedido nuevo con folio distinto por la retención y tramitación del pedido completo.
- **Alcance "No aplica a":** se agrega la exclusión de la separación del pedido por documentación regulatoria; se ajusta la mención de la facturación por adelantado al pedido completo.
- **Regla 4:** reescrita para retención del pedido completo, retirando la bifurcación por bandera de entregas parciales.
- **Regla 5:** reescrita para tramitación en el folio original, retirando la generación de un pedido nuevo con folio distinto.
- **Regla 6:** sustituida — pasa a "documentación regulatoria a nivel cliente".
- **Riesgo 3:** reescrito en términos del pedido completo retenido.
- **Criterios B3 y B6:** reescritos para reflejar retención y tramitación del pedido completo en folio original.
- **Criterios A1, B1 y B4:** ajustan "al menos una sustancia controlada" a "sustancias controladas".

### Bloque 2 — Cambio del módulo propietario
- Campo "Módulo" pasa de **Pretramitar Pedido** a **Tramitar Pedido**.
- Nueva Regla 7: validación como último paso de Tramitar Pedido.

### Cierres
- Puntos de entrada a Tramitar Pedido: validación como último paso cubre todos los caminos (avance desde Pretramitar, OC corregida, tramitación con errores, aceptación OC Interna).
- Cambio de familia del producto a controlado: deja de ser problema al validarse en el último paso.
- Pedido que mezcla partidas controladas y no controladas: confirmado que no ocurre en la operación real; se retira la mecánica de separación.

### Correcciones de consistencia
- Referencias desactualizadas a Pretramitar Pedido corregidas a Tramitar Pedido.

---

## RE-FU-032 — Notas de Crédito México (reenfoque masivo)

**Fecha origen:** 13-ago-2026 · Dolarización: 14-ago-2026 · Aplicado: 14-ago-2026

### Bloque 1 — Catálogo de motivos y sus datos derivados
- Requisito y Alcance: catálogo de tres motivos que determina la modalidad de captura.
- **Regla 3:** catálogo de motivos con tabla de derivación. Modalidad no editable; TipoRelacion derivado salvo en devolución de mercancía (usuario elige entre 01 y 03). Se retira el TipoRelacion fijo en 01 (incorrecto).
- **Reglas 4 y 5:** modalidades reescritas — precio unitario heredado no editable; captura por partida en piezas.
- **Sección D:** catálogo de motivos, derivación de modalidad y tratamiento del TipoRelacion.
- **Alcance "No aplica a":** exclusión de la corrección de errores de precio unitario mediante la modalidad por partidas.

### Bloque 2 — Mecánica FormaPago (SaldoPendiente + Excedente)
- Determinación automática de FormaPago, elección del destino del excedente y reducción del saldo pendiente de la factura origen.
- **Regla 8:** sustituye la FormaPago heredada por la determinación automática vía comparación NC vs SaldoPendiente.
- **Regla 9:** destino del excedente con dos opciones (devolución real o saldo a favor 23-Compensación).
- **Regla 11:** reducción del saldo pendiente de la factura origen cuando la NC se resuelve como condonación.
- **Regla 12:** acotada a la inmutabilidad del total timbrado y posibilidad de más de una NC por factura.
- **Reglas 13 y 14:** estatus de aplicación de la NC y consumo del saldo a favor.
- **Sección G:** resolución automática de la FormaPago con condonación, opciones de destino del excedente y validación de avance.
- **Sección M:** control del consumo del saldo a favor y conversión de moneda cuando corresponda.
- **Alcance "No aplica a":** se retira la exclusión del flujo de devolución de dinero (sí está contemplado como destino del excedente).
- Se retira la regla de política de intentar la NC antes que la devolución.

### Bloque 3 — Retiro de la cancelación de documentos fiscales
- Alcance "Aplica a": se retira la cancelación condicional de la factura origen.
- Alcance "No aplica a": consolidación — sin función de cancelación en el sistema para facturas ni NCs.
- **Regla 18:** extendida a facturas.
- Se elimina la sección de cancelación condicional (4 criterios) y su riesgo asociado.

### Bloque 4 — Retiro del módulo NC para Región Perú
- Se retira la referencia a la reutilización de la estructura para Perú.
- No existe módulo de Notas de Crédito para esa región (no hay timbrado en su alcance).

### Bloque 5 — Wizard 4→3 pasos y vista unificada
- El wizard pasa de 4 a 3 pasos.
- La vista de la NC emitida deja de ser un paso del wizard y se unifica con la vista de detalle (evita duplicar pantallas).
- **Sección K:** renombrada a vista de detalle; retirados el banner de estado y el criterio de equivalencia; criterios renumerados.
- **Sección J:** navegación posterior al timbrado a la vista de detalle unificada.

### Bloque 6 — Separación del listado de clientes y el detalle por cliente
- **Secciones A y B:** separadas, cada una con columnas, filtros, buscador y acciones propias.
- Diferenciación del comportamiento de la acción "nueva NC" según pantalla de origen (desde listado exige seleccionar cliente; desde detalle viene preseleccionado).
- **Sección B:** acceso a la vista de detalle de la NC desde el listado.
- **Secciones A, B y C:** buscadores por coincidencia.

### Bloque 7 — Otras incorporaciones y correcciones
- Regla 6: el monto capturado es la base gravable.
- Regla 7 y Sección H: herencia de moneda, TC y estructura de impuestos de la factura origen (sustituye la preservación del TC del día del timbrado).
- Se retira la captura por tasa con factura origen de tasas mixtas (no ocurre en la operación real) y se agrega su exclusión.
- Regla 15 y Sección C: bloqueo de NC contra el RFC genérico de público en general.
- Regla 16 y Secciones I y J: el folio se consume únicamente al timbrar.
- Regla 19 y Sección M: disponibilidad en Validar Cobro acotada a NCs con saldo a favor disponible.
- Regla 21 y Sección L: envío del correo no automático; contacto precargado del cliente de la factura origen.
- Sección C: saldo pendiente entre los datos visibles de cada factura candidata.
- Sección D: saldo pendiente de la factura origen en el Paso 2.
- Sección E: nombre de la columna corregido a "Clave del catálogo" en la tabla de partidas.
- Riesgo 2 nuevo: cancelación externa de NC no visible para el sistema.
- Riesgo 3 nuevo: múltiples NC sobre una misma factura sin validación por partida (pendiente).
- Sección A: columna "Por Aplicar" contabiliza NCs con saldo disponible (incluidas parcialmente aplicadas); columna "Total" renombrada a "Total de NCs generadas".
- Alcance "No aplica a": exclusión de relación de una NC con más de una factura origen.

### Bloque 8 — Dolarización del listado de clientes (14-ago)
- **Criterio A1 y Observaciones:** los montos del listado de clientes se presentan **dolarizados**, convertidos con el tipo de cambio heredado por cada NC de su factura origen. Permite un total único por cliente aunque las NCs estén en distintas monedas.
- **Criterio A2:** se retira el filtro de moneda del listado de clientes (innecesario al presentarse los montos dolarizados).

### Cierres
- Forma de pago en modalidad manual (mecánica saldo + excedente).
- Claves de producto y unidad en modalidad sin partidas (convención Apéndice 5 SAT: 84111506 / ACT).
- Políticas de autorización por monto (no aplica código de autorización).
- Denominación canónica del rol: **función Analista de Cuentas por Pagar; rol Gestor de Cobranza**.
- Serie del foliado: **"B2"** para las Notas de Crédito de PROQUIFA México.

---

## Requisitos NO tocados en esta consolidación

Los archivos anexos (`-Back.md`, `_BD.md`, `-Tareas.md`, `-Revision.md`) de cada requisito no se actualizaron en este lote. Si algún cambio de este historial impacta esas capas (por ejemplo, RE-FU-006 con el encadenamiento desde Empresa impacta el modelo de datos; RE-FU-009 con el cambio de módulo impacta las tareas; RE-FU-032 con el retiro de cancelación impacta Back/Endpoints/Tareas), corresponde propagar los cambios en un lote separado.

---

## Referencias

- Notas de reenfoque enviadas por **Roberto Baez Muñoz** (Slack / correo):
  - 27-jul-2026 11:35 — RE-FU-001
  - 27-jul-2026 12:36 — RE-FU-002
  - 29-jul-2026 10:47 — RE-FU-003
  - 31-jul-2026 15:31 — RE-FU-004
  - 03-ago-2026 17:35 — RE-FU-005
  - 05-ago-2026 11:54 — RE-FU-006
  - 05-ago-2026 12:13 — RE-FU-007
  - 05-ago-2026 12:35 — RE-FU-003 (cierre)
  - 07-ago-2026 12:13 — RE-FU-006 (cierres)
  - 07-ago-2026 12:19 — RE-FU-005 (Uso de CFDI + regionalización)
  - 07-ago-2026 13:27 — RE-FU-004 (cierres)
  - 13-ago-2026 15:37 — RE-FU-009 (retiro separación)
  - 13-ago-2026 — RE-FU-032 (reenfoque masivo)
  - 14-ago-2026 — RE-FU-032 (dolarización listado)
- Documentos del cliente referenciados: **R16 - Catálogos Fiscales**, archivo de equivalencias del análisis previo, documento de conceptos de cuentas bancarias por región, Guía Técnica de Notas de Crédito.
