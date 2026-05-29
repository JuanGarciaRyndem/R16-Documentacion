# R16 – Análisis de Documentos de Referencia

> **Documentos analizados:**
> - Borrador – Análisis de flujo y reglas de Tramitación de pedidos con cobranza de prepagos y facturación por adelantado
> - Checklist / Análisis R16 – Documento (36 páginas · Roberto Baez Muñoz · 17/03/2026)
> - Controlar Cobranza Prepago (Pyx4 v1.0 · Larissa Calvo · 11/03/2026)
> - Controlar Pedido – Alcance Prepago (Pyx4 v4.0 · Larissa Calvo · 11/03/2026)
> - Tramitación de Pedidos – Crédito (diagrama imagen, sin texto extraíble)

---

## 1. Flujo General de Tramitación – Todos los Escenarios

### Al hacer clic en "Tramitar Pedido" (primera vez)

| Escenario | Acciones del sistema |
|---|---|
| Cliente **crédito** + productos **controlados** | Genera Folio PI → Tramita pedido → Genera confirmación → Transfiere a Legacy |
| Cliente **crédito** sin controlados + **sin** Factura por Adelantado | Genera Folio PI → Tramita pedido → Genera confirmación → Transfiere a Legacy |
| Cliente **crédito** sin controlados + **con** Factura por Adelantado | Genera pendiente en **Facturar por Adelantado** |
| Cliente **prepago** + productos **controlados** | Genera Folio PI → Genera Proforma → Genera pendiente en **Validar Pago** |
| Cliente **prepago** sin controlados + **sin** Factura por Adelantado | Genera Folio PI → Genera Proforma → Genera pendiente en **Validar Pago** |
| Cliente **prepago** sin controlados + **con** Factura por Adelantado | Genera Folio PI → Genera pendiente en **Facturar por Adelantado** |

### Al hacer clic en "Generar Factura por Adelantado"

| Escenario           | Acciones del sistema                                                                  |
| ------------------- | ------------------------------------------------------------------------------------- |
| Cliente **crédito** | Genera factura normal (PPD) → Establece FEE → Desbloquea pendiente en Tramitar Pedido |
| Cliente **prepago** | Genera factura normal (PPD) → Genera pendiente en **Validar Pago**                    |

### Al hacer clic en "Validar Pago"

| Escenario | Acciones del sistema |
|---|---|
| Prepago + **controlados** | Genera **factura anticipo** → Establece FEE → Desbloquea pendiente en Tramitar Pedido |
| Prepago sin controlados + **sin** FA | Genera **factura normal** → Establece FEE → Desbloquea pendiente en Tramitar Pedido |
| Prepago sin controlados + **con** FA | Genera **complemento de pago** → Actualiza estatus de factura adelantada → Establece FEE → Desbloquea pendiente en Tramitar Pedido |

### Al hacer clic en "Tramitar Pedido" (pendiente desbloqueado)

| Escenario | Acciones del sistema |
|---|---|
| Crédito + FA | Ya tiene FEE → Genera Folio PI → Tramita pedido → Genera confirmación → Transfiere a Legacy |
| Todos los escenarios prepago | Ya tiene FEE y Folio PI → Tramita pedido → Genera confirmación → Transfiere a Legacy |

---

## 2. Reglas de Negocio Confirmadas por Módulo

### 2.1 Pretramitar Pedido

| ID | Regla |
|---|---|
| **D-04** | Cliente crédito que selecciona FA: genera pendiente independiente en Factura por Adelantado. Aplican mismas restricciones (sin controlados). |
| **D-05** | Cliente crédito que **no** selecciona FA: continúa directo a Tramitar Pedido sin cambios. |
| **D-06** | El "folio de pedido" (R16M-RE-FU-015) es el **mismo pedido interno** de hoy. Debe existir **antes** de ir a Factura por Adelantado o Validar Pago, ya que la proforma y la factura necesitan ese folio para trazabilidad. |
| **D-07** | Cambio de condición Prepago → Crédito: requiere **código de autorización del Gerente de Tesorería**. Se hace a nivel de pedido. |
| **D-08** | El correo del ESAC se incluye en el envío de la Proforma al cliente. |
| **D-09** | Pedidos provenientes de Gestionar Pedido Intramitable y Validar Ajuste a la OC también deben pasar por el flujo de Factura por Adelantado / Validar Pago si aplica. |

### 2.2 Facturar por Adelantado

| ID | Regla |
|---|---|
| **D-10** | Productos controlados: FA está **bloqueada** tanto para crédito como para prepago. |
| **D-11** | Los datos fiscales (RFC, nombre, CURP) se toman de la plantilla del cliente. El usuario **puede modificarlos** en pantalla por operación, sin replicarlos al catálogo. La moneda de facturación también se toma de la plantilla del cliente y **no puede cambiarse**. |
| **D-12** | Si TurboPac falla el timbrado: el sistema muestra en pantalla el **error específico** de TurboPac (normalmente incompatibilidades de método/forma de pago). No hay reintento automático. Sara entregará catálogo de errores SAT para implementar restricciones desde el catálogo de clientes. |

### 2.3 Validar Pago

| ID | Regla |
|---|---|
| **D-13** | Un pago puede aplicarse a **varias facturas** (RN-01). El monto aplicado no puede exceder el saldo disponible del pago (RN-03). El sistema actualiza el monto disponible si no se cubre en su totalidad. |
| **D-14** | La liberación del candado ocurre al **generar el complemento de pago** (no al generar la factura). |
| **D-15** | El módulo de Complemento de Pago **no está definido** en ninguna matriz. Pendiente de especificación: ¿quién lo genera, desde dónde y cómo? |
| **D-16** | Inconsistencias en pagos: tipos identificados: monto incorrecto, datos del comprobante no coinciden. Al eliminar un pago con inconsistencia, también se **elimina el correo del buzón** que lo generó. Si el cliente corrige, entra nuevo correo y genera nuevo pendiente. |
| **D-17** | Notas de crédito: solo las generadas en ProquifaNet 2 (no legacy). Pueden aplicarse parcialmente. Pueden usarse como sustituto total o parcial de un pago. |

### 2.4 Buzón de Pagos

| ID | Regla |
|---|---|
| **D-18** | Clasificación de correos como "pago" es **automática** (MailBot). El monto se captura la primera vez que se trabaja en Validar Pago. Visibilidad filtrada por **cartera de cobrador**. |
| | Roles con acceso: **Especialista CxC**, **Analista Sr. CxC**, **Analista Jr. CxC**. |
| **D-19** | El correo en el buzón y el registro de pago son **dos objetos independientes**. Al clasificarse genera un pendiente en Validar Pago (inicialmente sin monto). No se borra el correo hasta que se vincule por primera vez a una proforma o factura. |

### 2.5 Tramitar Pedido

| ID | Regla |
|---|---|
| **D-21** | Para clientes de **crédito**: uso de CFDI y método de pago en Tramitar Pedido se **mantienen igual** a hoy. Para clientes **prepago**: estos datos son responsabilidad del módulo de Cobros (FA o Validar Pago). Si se modifican en FA, se actualizan en PQF2 (pendiente validar transferencia a Legacy). |

### 2.6 Transferencia a Legacy

| ID | Regla |
|---|---|
| **D-22** | Tablas confirmadas: `facturas`, `p_facturas`, `factura_electronica`, `p_factura_electronica`, `complemento_pago`. Sara verificará si faltan más. La transferencia es **automática al momento de generar cada documento**. Sara entregará contrato de conexión con diccionario de datos. |
| **D-23** | Módulo "Relacionar Facturas" en Legacy para clientes de crédito: pendiente de especificación. |
| **D-24** | Flujo de Crédito Pago contra Entrega: pendiente confirmar si pasa por Validar Pago y si genera pedido interno. |

---

## 3. Procedimiento: Controlar Cobranza Prepago (Pyx4 v1.0)

**Propósito:** Garantizar la validación de pagos de clientes prepago en un plazo máximo de **72 horas** posteriores a la recepción del comprobante o confirmación bancaria. Meta: al menos el **95%** de los pagos recibidos verificados, registrados y notificados dentro del periodo.

**Roles del proceso (Pyx4):**

| Rol Pyx4 | Descripción |
|---|---|
| Planificador Ciclo Cobranza | Planifica y organiza el ciclo de cobranza |
| Gestor Cobranza | Ejecuta y valida los cobros |
| Gestor Facturas | Gestiona facturas; puede generar "Necesidad de cancelación" |
| Gestor Documentos-Cobros | Maneja documentos del proceso de cobro |
| Verificador Ciclo Cobranza | Verifica el cierre del ciclo |

> ⚠️ **D-34**: Estos roles de Pyx4 **no aparecen en ninguna de las tres matrices**. Pendiente mapeo oficial con los roles definidos en las matrices (Gestor de Cobranza, Coordinador de Tesorería, etc.).

**Flujo principal:**
1. Evidencia de pago llega vía MailBot → genera **Cobranza Prepago por Atender**
2. Gestor Cobranza: **Trabajar Buzón de Pagos**
3. Gestor Cobranza: **Validar Pago** → resultado: Pago Validado o Nueva Fecha Promesa de Pago o Necesidad de Cancelación
4. Si Pago Validado → **Controlar Pedido** (Tramitador de Pedidos desbloquea pedido)
5. Si FA autorizada → **Facturar por Adelantado** → genera factura → Controlar Pedido
6. Verificador: **Verificar Cobranza Prepago** → mide % eficacia

> ⚠️ **D-37**: El Gestor Facturas puede generar una "Necesidad de cancelación" que se comunica al Tramitador de Pedidos. Este flujo **no está documentado en ninguna matriz**. Pendiente: bajo qué condiciones se activa y qué ocurre con el pedido bloqueado.

---

## 4. Procedimiento: Controlar Pedido – Alcance Prepago (Pyx4 v4.0)

**Propósito:** Tramitar el **90%** de las Órdenes de Compra recibidas a pre-trámite en un lapso de **1 día laboral**.

**Comentarios clave del procedimiento:**

1. El Coordinador de Pedidos registra en la Hoja de Trabajo 'Pendientes de Trámite': cliente, fecha de recepción de OC y número de referencia de OC del cliente.
2. Toda omisión del MailBot relativa a una OC debe gestionarse manualmente.
3. Durante la pretramitación, el ESAC de Pedidos valida:
   - OC vs. partidas del EVI (catálogo, concepto, presentación, marca, precio + IVA).
   - Datos de facturación del cliente (Razón Social, RFC, CP, condiciones).
   - Que "¿quién factura?" referenciado en la OC sea congruente con el catálogo de clientes.
4. Para clientes prepago con FA: el sistema emite código de autorización a **Coordinación de Control y Finanzas**.
5. La **Proforma** se emite solo para clientes prepago.
6. La **FEE** (Fecha Estimada de Entrega) se establece **al tramitar el pedido** (para prepago), no antes.
7. El Verificador Pedidos coteja en Legacy ("Consulta de pedidos") que el pedido tramitado tenga Pedido Interno + Contacto asociados.

> ⚠️ **D-33**: Conflicto de rol para el código de autorización de FA:
> - **Matriz 1** dice: "Coordinador de Planeación y Control"
> - **Pyx4** dice: "Coordinación de Control y Finanzas"
> Pendiente confirmar si son el mismo rol con distinto nombre o dos roles distintos.

---

## 5. Trazabilidad y Documentos

| ID | Punto |
|---|---|
| **D-25** | Cadena de trazabilidad en flujo de Factura por Adelantado: pendiente confirmar la secuencia completa Pretramitar → FA → Tramitar → Legacy. |
| **D-26** | Diseño de PDFs: la referencia de Proforma, Factura normal y Complemento de Pago fue compartida. Pendiente: confirmar si el diseño requiere aprobación del cliente antes de implementar, y si el Complemento de Pago se diseña desde cero. El alcance del diseño incluye los **cuatro documentos**: Proforma, Factura normal, Factura anticipo y Complemento de Pago. |

---

## 6. Catálogos y Configuración (Matriz 3)

| ID | Punto |
|---|---|
| **D-28** | **Catálogo de Bancos / Cuentas bancarias de Proquifa**: cada registro debe tener nombre del banco, número de cuenta, CLABE, moneda, empresa propietaria. Solo visible por **Gerente de Finanzas**. El campo "Cuenta bancaria" en Validar Pago vendrá de este catálogo (no texto libre). |
| **D-29** | **Tablero de Clientes**: módulo nuevo. Muestra pedidos de PQF2 sin importar condición de crédito. Pendiente: columnas, filtros (cliente, región, condición de pago, fechas, estatus), formato de descarga (Excel/PDF/CSV) y roles con acceso. |
| **D-30** | **Consulta General**: trazabilidad completa del pedido. Pendiente definición de columnas y filtros. |
| **D-31** | **Catálogo de Clientes – Documentos regulatorios y restricción controlados**: pendiente definir qué documentos regulatorios se gestionan y cómo se implementa la restricción de Sustancias Controladas en el catálogo. |
| **D-32** | **Código Validador / Referencia de Cliente**: requisito sin definir. |

---

## 7. Puntos Sin Resolver (Pendientes Críticos)

| # | Pendiente | Responsable |
|---|---|---|
| 1 | Mapeo oficial de roles Pyx4 vs. roles en matrices | Larissa Calvo / Sara Sánchez |
| 2 | Confirmar rol receptor del código de autorización de FA (D-33) | Larissa Calvo |
| 3 | Definir módulo Complemento de Pago: quién lo genera y cómo (D-15) | Sara Sánchez / Biridiana |
| 4 | Flujo de cancelación desde cobranza (Necesidad de Cancelación) (D-37) | Larissa Calvo |
| 5 | Nueva Fecha Promesa de Pago: ¿está en alcance R16? (D-36) | Sara Sánchez |
| 6 | Contrato de conexión Legacy con diccionario de datos (D-22) | Sara Sánchez |
| 7 | Confirmar tablas Legacy faltantes (D-22) | Sara Sánchez |
| 8 | Módulo "Relacionar Facturas" en Legacy para crédito (D-23) | Sara Sánchez |
| 9 | Flujo Crédito Pago contra Entrega (D-24) | Sara Sánchez |
| 10 | Catálogo de errores SAT / TurboPac (D-12) | Sara Sánchez |
| 11 | Definición completa del Tablero de Clientes (D-29) | Larissa Calvo / Biridiana |
| 12 | Código Validador / Referencia de Cliente (D-32) | Larissa Calvo |
| 13 | Diseño y aprobación de PDFs: Proforma, Factura, Factura Anticipo, Complemento (D-26) | Roberto Baez Muñoz |
| 14 | Cadena de trazabilidad completa FA → Legacy (D-25) | Equipo completo |
| 15 | Confirmar si TurboPac opera en Perú | Sara Sánchez |
| 16 | Catálogos fiscales Perú (tipo sociedad mercantil, régimen fiscal, usos CFDI) | Sara Sánchez |
