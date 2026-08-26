# \[R16A-RE-FU-025\]\[DIS-SOL\] Diseño de la solución

## Diseño de la solución

|  |  |
| :-: | :-: |
| **FORMATO** | N/A |
| **PROYECTO** | R16A |
| **REFERENCIA** | N/A |
| **VERSIÓN** | 1.0 |
| **FECHA** | 22/07/2026 |
| **AUTOR** | Osmar Calderón |
| **REVISOR** | Valdemar Farina Sánchez |

# Importante

Posterior a este diseño, ¿cómo saber si el diseño de la solución al requisito está completo para que el programador inicie con el desarrollo?

Hazte estas preguntas rápidas:

  - ¿El programador sabe qué flujo implementar?
  - ¿Sabe qué pasa si algo falla?
  - ¿Sabe qué reglas no puede romper?
  - ¿Sabe qué pruebas debe pasar?
  - ¿Sabe dónde impacta?

# Introducción

## 1. Propósito del documento

Definir el diseño de la solución técnica para el requisito **R16A-RE-FU-025 — Validar Cobro: Paso 1 Perú (Captura del Cobro)**, describiendo **exclusivamente las diferencias regionales** respecto al diseño de RE-024 (México), del que hereda toda la infraestructura de base de datos, servicios y endpoints.

**Nota:** Este es un documento **delta** — no repite el diseño completo del Paso 1. Toda decisión, estructura de datos, ciclo de estatus, foliador, endpoints y flujo técnico están definidos en el DIS-SOL de RE-024 y aplican a Perú **por defecto**. Aquí solo se documentan las variantes.

## 2. Alcance

Este análisis cubre únicamente el requisito antes mencionado: el **Paso 1 del wizard de Validar Cobro** (captura del cobro) para clientes de **Región Perú** (clientes Prepago de Golocaer S.A.C.).

### Específicamente incluye:

  - Poblado del catálogo interno **`catMedioDePago`** para Perú (IdRegion = PER, ClaveFormaDePago = NULL, claves PER-\*). La regionalización vía IdRegion la aporta **RE-005** (ALTER TABLE + FK a Region); RE-025 solo agrega los registros Perú.
  - Poblado de **`EmpresaDatosBancarios`** + **`DatosBancarios`** para la entidad Golocaer S.A.C. (GOLPERU), con CCI de 20 dígitos.
  - Verificación de que **`catMoneda`** contiene PEN y su fila en **`CatMonedaRegion`** para la región Perú.
  - **Verificación** de que los endpoints y el ExchangeRateService del Paso 1 (definidos en RE-024) funcionan correctamente para Perú con la base PEN y los catálogos poblados.

### No se consideran:

  - Diseño del front-end (idéntico a RE-024).
  - **Nuevo DDL** — no se crean tablas, catálogos, foliadores ni SPs; todos los objetos los crea RE-024.
  - **Nuevos endpoints** — los del Paso 1 (RE-024) son region-agnósticos por diseño.
  - **Nuevo servicio de TC** — el ExchangeRateService único de RE-024 cubre Perú con base PEN (fuente BCRP, ya persistida en TipoDeCambioBanamex por el importador vigente).
  - Paso 2 y Paso 3 de Perú (RE-027 / RE-029) — la vinculación cobro↔factura es **operativa, sin efecto fiscal**.
  - Región México (cubierta en RE-024).

## 3. Relación con RE-024

RE-025 **hereda íntegramente** el DIS-SOL de RE-024. Todos los siguientes elementos aplican a Perú **sin cambios** salvo que se indique lo contrario en este documento:

|  |  |
| :-: | :-: |
| **Elemento heredado de RE-024** | **Aplica a Perú** |
| Tabla fccCobroCliente (fusionada) | Sí — misma tabla, mismo esquema |
| Catálogo catCobroEstatus + ciclo BORRADOR → CAPTURADO → ASOCIADO → … | Sí — mismo catálogo, mismos estatus |
| Catálogo catTipoInconsistenciaCobro + tabla fccInconsistenciaCobro | Sí — mismo catálogo (AplicaPaso='1'), mismas inconsistencias |
| Foliador fccCobroClienteConsecutivo + sp_IncrementarFolioCobro | Sí — mismo foliador, mismo formato COB-MMDDYY-NNNN |
| Endpoints /api/v1/validate-collection/... | Sí — region-agnósticos por diseño |
| ExchangeRateService único | Sí — la moneda base se resuelve por la región del usuario |
| Flujo técnico completo del Paso 1 | Sí — misma orquestación CQRS (auto-guardado, finalización, edición, inconsistencia, wizard-status) |
| Reglas RT-01 a RT-13 de RE-024 | Sí — salvo RT-07 y RT-09, que tienen matiz Perú (ver §Diferencias) |
| Buzón de Cobros (RE-008) | Sí — mismo origen de correos/adjuntos, ahora también para clientes PER |

# Visión general del diseño

## 1. Objetivo técnico

Habilitar la captura de cobros para clientes de Región Perú **reutilizando** la infraestructura de RE-024 y aportando **exclusivamente** los datos regionales (medio de pago interno, cuentas GOLPERU) y la variante regional del cálculo del TC (base PEN). No se introduce código nuevo en Proquifa.Pqf2.Finanzas ni DDL nuevo en la base de datos.

## 2. Componentes involucrados

|  |  |  |
| :-: | :-: | :-: |
| **Componente** | **Responsabilidad** | **Ubicación** |
| Proquifa.Pqf2.Finanzas | **Sin cambios** — reutiliza los servicios/queries/commands del Paso 1 (RE-024) con moneda base PEN resuelta por región | Solución .NET Core 10 |
| ProquifaDotNet | **Sin cambios de código** — solo aporta DML de catálogos regionales | Aplicativo .NET 4.8 |
| BD (SQL Server) | **Sin cambios de esquema** — solo INSERT en catMedioDePago, EmpresaDatosBancarios y DatosBancarios | Base de datos ProquifaDotNet |

**Diferencia clave respecto al KB:** *el KB de RE-025 planeaba un* TipoCambioPeruService *separado. Este diseño lo* **descarta***: el* ExchangeRateService *único de RE-024 ya cubre Perú, porque el importador vigente trae PEN del BCRP (cruce USD/MXN × PEN/USD) y lo persiste en* TipoDeCambioBanamex*. La brecha del KB ("fuente TC Perú pendiente, ¿SBS?") queda* **resuelta***.*

# Diferencias regionales — RE-024 (México) vs RE-025 (Perú)

## 1. Diferencias funcionales

|  |  |  |
| :-: | :-: | :-: |
| **Aspecto** | **México (RE-024)** | **Perú (RE-025)** |
| Medio de pago | catMedioDePago.ClaveFormaDePago = SAT c_FormaPago | catMedioDePago **sin clave** (interno, PER-\*) — Tesorería lo define |
| Cuenta destino | EmpresaDatosBancarios MEX (GOL/MUN/PRO/PQF) — CLABE 18 díg. | **`GOLPERU`** (0 registros — brecha B1) — CCI 20 díg. |
| Moneda base del TC | MXN (fuente BCRP/Banxico DOF) | **PEN** (fuente BCRP) — misma tabla TipoDeCambioBanamex |
| TipoDeCambioFiscal del cobro | Se llena si pago ≠ MXN (alimenta TipoCambioP del REP) | **Siempre NULL** — SUNAT no tiene Complemento de Pago |
| TipoDeCambioFacturacion del cobro | Se llena (operativo, base MXN) | Se llena (operativo, base PEN) — **el único TC que persiste en Perú** |
| ID fiscal | RFC (13 chars) — columna DatosFacturacionCliente.RFC | **RUC** (11 díg.) — **misma columna** DatosFacturacionCliente.RFC con valor peruano |
| Efecto Paso 3 | Genera CFDI Complemento de Pago (fiscal) | **Sin documento fiscal** — solo cierre operativo del pendiente |

## 2. Reglas de negocio propias de Perú

*Las reglas R1–R13 de RE-024 (listado en dos bloques, comprobante obligatorio, auto-guardado, múltiples cobros, editabilidad, inconsistencia, reanudación del wizard) aplican* **íntegramente** *a Perú. Aquí solo se listan las reglas específicas de la región.*

|  |  |
| :-: | :-: |
| **Regla** | **Descripción** |
| RP1 | Aplica **solo a clientes Región Perú**. Misma UI que México; cambian catálogos e identificador fiscal (RUC) |
| RP2 | **Medio de pago = catálogo interno** de PROQUIFA (catMedioDePago con ClaveFormaDePago = NULL): transferencia, depósito, cheque, efectivo… SUNAT no exige este dato → es solo control de Tesorería |
| RP3 | **Cuenta destino = cuentas de Golocaer S.A.C. Perú** (EmpresaDatosBancarios con Empresa.Prefijo='GOLPERU'). Usan **CCI (20 díg.)** en lugar de CLABE |
| RP4 | **TC del día vs PEN** (base regional PEN, no MXN): cobro PEN + facturación PEN → N/A; cobro PEN + facturación ≠ PEN → TC de la moneda de facturación vs PEN; cobro ≠ PEN → TC de la moneda del cobro vs PEN. Solo lectura. **Es el `TipoDeCambioFacturacion` de RE-024 con base PEN** |
| RP5 | **Sin Complemento de Pago (SUNAT).** El TipoDeCambioFiscal del cobro **es siempre NULL** en Perú. La vinculación cobro↔factura de los Pasos 2/3 es operativa, no fiscal (RE-027 / RE-029) |
| RP6 | Identificador fiscal = **RUC** (11 díg.), leído de DatosFacturacionCliente.RFC (misma columna, valor peruano) |

## 3. Reglas técnicas propias de Perú (matices sobre RE-024)

|  |  |
| :-: | :-: |
| **Regla** | **Descripción** |
| RT-P01 | El ExchangeRateService de RE-024 resuelve la moneda base según la región del usuario. Para clientes Perú retorna la matriz vs **PEN** (no MXN). No es un servicio nuevo — es el mismo, con parámetro regional |
| RT-P02 | En Perú, TipoDeCambioFiscal **siempre se persiste como NULL** (sin excepción). El servicio calcula el par vs base, pero el SavePaymentCommand no guarda el campo fiscal para cobros de clientes Perú |
| RT-P03 | El combo Moneda para Perú = catMoneda **JOIN `CatMonedaRegion`** por la región del usuario (regla RT-11 de RE-024). Verificar que PEN esté activo en CatMonedaRegion para la región Perú |
| RT-P04 | El combo Cuenta destino para Perú filtra vEmpresaDatosBancarios por Empresa.Prefijo='GOLPERU' (la lógica del filtro por prefijo ya existe en el consumo directo del catálogo) |
| RT-P05 | El combo Medio de pago para Perú filtra catMedioDePago por los registros con ClaveFormaDePago = NULL y claves PER-\* (el filtro por región del combo se cubre con la misma convención del catálogo) |

## 4. Endpoints

**No se crean endpoints nuevos.** *Todos los endpoints del Paso 1 son los definidos en RE-024 y son region-agnósticos por diseño (la moneda base y los catálogos se resuelven por la región del usuario, tomada del token).*

|  |  |  |  |
| :-: | :-: | :-: | :-: |
| **Solución / Proyecto** | **Endpoint** | **Tipo** | **Descripción** |
| Proquifa.Pqf2.Finanzas | Todos los endpoints /api/v1/validate-collection/... | **Sin cambios** | Definidos en RE-024. Funcionan para Perú sin modificación una vez poblados los catálogos regionales |
| ProquifaDotNet (existente) | GET /Logistica/ObtenerRequerimiento | **Sin cambios** | Detalle del correo + adjuntos — consumo directo (idéntico a RE-024) |
| ProquifaDotNet (existentes) | POST /Catalogos/{catMoneda, catMedioDePago, vEmpresaDatosBancarios} | **Sin cambios** | Combos del formulario — el filtro por región/prefijo lo aporta el consumo del combo |

# Diseño de componentes

## 1. Responsabilidades por componente

  - **Proquifa.Pqf2.Finanzas:** sin cambios. Reutiliza queries, commands y ExchangeRateService de RE-024. Único matiz de código: el SavePaymentCommand debe garantizar que TipoDeCambioFiscal se persiste como NULL cuando la región del cobro es Perú (RT-P02) — esta lógica ya está prevista en RE-024 vía la resolución regional del TC.
  - **ProquifaDotNet:** sin cambios de código. Solo aporta DML regional.
  - **BD:** sin cambios de esquema. Solo INSERT en catálogos existentes.

## 2. Diagramas

El flujo técnico es **idéntico al diagrama del Paso 1 en el DIS-SOL de RE-024**. No se reproduce aquí — consultar §Diseño de componentes / Diagramas en [R16A-RE-FU-024][DIS-SOL] Diseño de la solución.md.

La única variación en el diagrama para Perú es la resolución de la moneda base (PEN en lugar de MXN) al momento de consultar TipoDeCambioBanamex desde el ExchangeRateService, y la escritura de TipoDeCambioFiscal = NULL en fccCobroCliente.

# Diseño de Interfaces

## 1. Interfaces de entrada

Idénticas a RE-024. No se agregan ni modifican interfaces de entrada.

## 2. Interfaces de salida

Idénticas a RE-024. Únicos matices de payload para clientes Perú:

  - exchange-rate: retorna fiscal = null siempre; billing con base PEN.
  - Item del listado: currencyCode = "PEN" para cobros y facturaciones en soles peruanos.

## 3. Contrato de datos conceptual

Sin cambios de contrato. Los mismos DTOs de RE-024 sirven a ambas regiones.

# Diseño de Modelo de Datos

## 1. Nuevos objetos

**Ninguno.** Toda la infraestructura de datos la crea RE-024. RE-025 solo consume.

## 2. DML que aporta Perú

*Todos los INSERT dependen de que* **el cliente** *confirme/entregue los datos — ver §Pendientes.*

### catMedioDePago — INSERT catálogo interno Perú

INSERT de las claves internas (PER-TRANSFERENCIA, PER-DEPOSITO, PER-CHEQUE, PER-EFECTIVO, etc.) con:

  - IdRegion = PER (la columna IdRegion la agrega **RE-005** vía ALTER TABLE + FK a Region; RE-025 se limita a insertar los registros Perú con esa IdRegion).
  - ClaveFormaDePago = NULL (SUNAT no exige clave — es control interno de Tesorería, no dato fiscal).

**Depende de la confirmación del cliente** — Tesorería PROQUIFA debe validar la lista definitiva de medios de pago Perú antes del INSERT. Los valores propuestos (transferencia, depósito, cheque, efectivo) son sugerencia.

catMedioDePago.ClaveFormaDePago *es* **NULLABLE** *en el esquema existente → soporta registros sin clave SAT;* **no requiere ALTER**.

### EmpresaDatosBancarios + DatosBancarios — INSERT cuentas GOLPERU

INSERT de las cuentas bancarias de Golocaer S.A.C. Perú:

  - EmpresaDatosBancarios con FK al registro Empresa cuyo Prefijo = 'GOLPERU'.
  - DatosBancarios con la **CCI de 20 dígitos** (misma columna Clabe — el esquema no distingue entre CLABE MEX y CCI PER; convención de reuso).

**Depende de que el cliente proporcione los datos** — PROQUIFA Perú debe entregar los datos de las cuentas (banco, número de cuenta, CCI, moneda PEN/USD). Hoy hay **0 registros**.

### catMoneda + CatMonedaRegion — verificación

Verificar que:

  - catMoneda contiene un registro con clave PEN (activo).
  - CatMonedaRegion contiene la fila que enlaza PEN con la región Perú (activa).

Si falta alguna, INSERT correspondiente. catMoneda es compartida entre regiones y el filtro regional del combo ya se resuelve con CatMonedaRegion (decisión de RE-024).

## 3. Tablas y objetos eliminados

Ninguno. RE-025 no elimina objetos.

## 4. Relaciones

Sin cambios. Todas las relaciones las define RE-024.

## 5. Reglas de integridad

Sin cambios. Aplican todas las de RE-024. Matiz Perú: TipoDeCambioFiscal es NULL en todos los cobros de clientes Perú (RT-P02).

## 6. Ciclo de vida del registro

Idéntico al de RE-024 (BORRADOR → CAPTURADO → ASOCIADO → COMPLETADO / SALDO_A_FAVOR; CON_INCONSISTENCIA / CANCELADO).

**Matiz Perú:** el ciclo termina en COMPLETADO (o SALDO_A_FAVOR) sin generar documento fiscal en el Paso 3 — solo cierra el pendiente operativo. Confirmar con RE-029 si el estatus terminal COMPLETADO conserva su nombre o requiere uno neutral para Perú (ver P1 en §Pendientes).

## 7. Orden de ejecución de scripts

RE-025 **no ejecuta scripts DDL**. Solo DML, tras el despliegue completo de RE-024:

1.  Verificar que RE-024 esté desplegado (tabla fccCobroCliente, catálogos, foliador, endpoints).
2.  INSERT en catMedioDePago (Perú, sin clave SAT).
3.  INSERT en EmpresaDatosBancarios + DatosBancarios (GOLPERU, CCI).
4.  Verificar catMoneda + CatMonedaRegion para PEN en región Perú.

# Impacto Técnico

## 1. Impacto en código existente

**Ninguno.** RE-025 no introduce ni modifica código en Proquifa.Pqf2.Finanzas ni en ProquifaDotNet. Toda la lógica ya está en RE-024 y es region-agnóstica.

Único punto a **verificar** (no reescribir):

  - El SavePaymentCommand de RE-024 debe persistir TipoDeCambioFiscal = NULL cuando la región del cliente es Perú (RT-P02). Si RE-024 no lo hace explícitamente por región, se agrega esa guardia en RE-024 antes del despliegue conjunto.

## 2. Impacto en modelos

Ninguno. Los modelos y el scaffold EF de RE-024 sirven a ambas regiones sin cambios.

## 3. Impacto en despliegue

  - Ejecutar los DML regionales tras el despliegue completo de RE-024.
  - No requiere regeneración de Scaffold, no requiere redeploy de Finanzas, no requiere migraciones EF Core.

## 4. Tareas de implementación

|  |  |  |  |
| :-: | :-: | :-: | :-: |
| **\#** | **Tarea** | **Aplicativo** | **Tipo** |
| **T1** | INSERT catMedioDePago — catálogo interno Perú (IdRegion = PER, ClaveFormaDePago = NULL, claves PER-\*). La columna IdRegion la aporta RE-005 | ProquifaDotNet | DML |
| **T2** | INSERT EmpresaDatosBancarios + DatosBancarios para GOLPERU (CCI de 20 díg.) | ProquifaDotNet | DML |
| **T3** | Verificar catMoneda tiene PEN y su fila en CatMonedaRegion para la región Perú; INSERT si falta | ProquifaDotNet | DML de verificación |
| **T4** | Verificar que los endpoints del Paso 1 (RE-024) funcionan para Perú con los catálogos poblados (T1–T3) y que TipoDeCambioFiscal se persiste como NULL | Proquifa.Pqf2.Finanzas | Verificación / QA |

# Decisiones Tomadas

Todas las decisiones D1–D11 de RE-024 aplican a Perú. Aquí solo se documentan las decisiones **específicas de la variante peruana**.

|  |  |
| :-: | :-: |
| **ID** | **Decisión** |
| DP1 | **Reutiliza íntegramente la infraestructura de RE-024.** Sin DDL nuevo, sin código nuevo, sin endpoints nuevos, sin servicios nuevos. Perú solo aporta DML de catálogos regionales |
| DP2 | **Fuente del TC Perú: BCRP (ya en `TipoDeCambioBanamex`).** La brecha del KB ("fuente TC Perú pendiente, ¿SBS?") queda **resuelta**: el importador vigente trae PEN del BCRP (cruce USD/MXN × PEN/USD) y lo persiste en TipoDeCambioBanamex. Se descarta SBS |
| DP3 | **`ExchangeRateService` único para ambas regiones.** No se crea el TipoCambioPeruService que planeaba el KB. La moneda base (MXN/PEN) se resuelve de la región del usuario dentro del mismo servicio |
| DP4 | **Un solo TC persiste en Perú: `TipoDeCambioFacturacion`.** Sin Complemento de Pago SUNAT → TipoDeCambioFiscal siempre NULL. La matriz "vs PEN" del KB es el TipoDeCambioFacturacion con base PEN. Ya cubierto por la decisión D4 de RE-024 (dos TCs por cobro) |
| DP5 | **Estado y ciclo de vida: idéntico a RE-024.** El cobro Perú usa el mismo catCobroEstatus con las mismas 7 claves. En Perú no hay timbrado fiscal, pero el Paso 3 igual marca el cierre en COMPLETADO — pendiente P1 confirmar si conviene un estatus con nombre neutral (ver §Pendientes) |
| DP6 | **RUC en la misma columna que RFC.** DatosFacturacionCliente.RFC guarda el RUC peruano (11 díg.) sin alterar el esquema. Las validaciones de formato ya están parametrizadas por región en el flujo de captura de clientes |
| DP7 | **CCI en la misma columna que CLABE.** DatosBancarios.Clabe guarda la CCI peruana (20 díg.) sin alterar el esquema. Convención de reuso — el esquema no distingue entre CLABE MEX y CCI PER |

# Pendientes

## Brechas bloqueantes

|  |  |  |  |
| :-: | :-: | :-: | :-: |
| **\#** | **Brecha** | **Estado** | **Bloquea** |
| **B1** | **Cuentas bancarias GOLPERU** (0 registros hoy) — **el cliente** (PROQUIFA Perú / Golocaer S.A.C.) debe proporcionar los datos de las cuentas: banco, número de cuenta, CCI (20 díg.), moneda (PEN / USD) | ⛔ Bloqueante | **Advertencia bloqueante para la Tarea T2** |
| **B2** | **Catálogo interno de medio de pago Perú** — **el cliente** (Tesorería PROQUIFA) debe confirmar la lista definitiva de claves PER-\* y sus descripciones antes del INSERT | ⛔ Bloqueante | **Advertencia bloqueante para la Tarea T1** |
| **B4** | **Catálogo de Tipos de Inconsistencia del Paso 1** — pendiente Tesorería (transversal con RE-024) | ⛔ Bloqueante | **Advertencia bloqueante para la Tarea T4** (compartida con RE-024) |
| **B5** | **Foliador COB: alcance global (MEX+PER) vs por región** — si es por región, fccCobroClienteConsecutivo de RE-024 pasa de fila única a una fila por región | ⛔ Bloqueante | **Advertencia bloqueante para la Tarea T4** (compartida con RE-024) — impacta el DDL de RE-024, no un DML propio de RE-025 |

## Pendientes no bloqueantes

|  |  |  |
| :-: | :-: | :-: |
| **\#** | **Pendiente** | **Naturaleza** |
| **P1** | **Estatus terminal en Perú:** ¿COMPLETADO aplica sin timbrado fiscal, o conviene un nombre neutral? Definir con RE-029 | Nomenclatura — no bloquea desarrollo |
| **P2** | Asistencia automatizada de auto-completado (no comprometida en R16, igual que México) | Fuera de alcance |
| **P3** | Denominación canónica del rol operativo (Gestor de Cobranza / Analista de CxC) — transversal con RE-023 / RE-024 | Nomenclatura |

# Manejo de Errores y Excepciones

Todos los escenarios de manejo de errores de RE-024 aplican a Perú. Aquí solo se listan los matices regionales.

|  |  |
| :-: | :-: |
| **Escenario** | **Comportamiento esperado** |
| Cobro de cliente Perú con TipoDeCambioFiscal distinto de NULL | Error de integridad — debe rechazarse en el SavePaymentCommand. Guardia por región |
| Sin fila de TC en TipoDeCambioBanamex para la FechaPago con base PEN | Usar la última fila anterior disponible (misma convención "lectura DOF" de RE-024); si no hay ninguna, señalar el TC como no disponible |
| Combo Cuenta destino vacío para cliente Perú | Indicador de que las cuentas GOLPERU no han sido pobladas (brecha B1) — mostrar mensaje operativo y no bloquear la captura salvo al finalizar |
| Combo Medio de pago vacío para cliente Perú | Indicador de que el catálogo interno Perú no ha sido poblado (brecha B2) — mostrar mensaje operativo |

# Estrategia de Pruebas

## 1. Pruebas funcionales (Criterios de Aceptación en DEV)

  - Un cliente Perú captura un cobro en PEN con facturación en PEN → TipoDeCambioFacturacion = 1, TipoDeCambioFiscal = NULL, PDF y flujo del Paso 1 idénticos a México.
  - Un cliente Perú captura un cobro en USD con facturación en PEN → TipoDeCambioFacturacion = TC USD/PEN del día, TipoDeCambioFiscal = NULL.
  - Un cliente Perú captura un cobro en PEN con facturación en USD → TipoDeCambioFacturacion = TC USD/PEN del día, TipoDeCambioFiscal = NULL.
  - El combo Medio de pago muestra únicamente registros con ClaveFormaDePago = NULL y claves PER-*.
  - El combo Cuenta destino muestra únicamente cuentas de EmpresaDatosBancarios con Empresa.Prefijo='GOLPERU' (CCI 20 díg.).
  - El combo Moneda muestra únicamente monedas activas en CatMonedaRegion para la región Perú.
  - El listado de cobros funciona en dos bloques (capturados / sin capturar) idéntico a México.
  - Finalizar genera folio COB-MMDDYY-NNNN compartiendo el mismo consecutivo de RE-024 (o por región, según se resuelva B5).

## 2. Pruebas técnicas (unitarias e integración)

  - El ExchangeRateService resuelve moneda base **PEN** para usuarios de región Perú.
  - El SavePaymentCommand persiste TipoDeCambioFiscal = NULL para todos los cobros de clientes Perú, sin excepción.
  - El endpoint PUT payment/{id} funciona idéntico a México para cobros Perú (guardia 409 por estatus inmutable).
  - El endpoint POST payment/{id}/finalize genera folio COB para cobros Perú siguiendo el mismo patrón del foliador de RE-024.

## 3. Casos críticos

  - **Cobro Perú con `TipoDeCambioFiscal` accidentalmente distinto de NULL:** el SavePaymentCommand debe rechazarlo (guardia por región).
  - **Cobro Perú con moneda ≠ PEN y facturación ≠ PEN:** validar que TipoDeCambioFacturacion se calcula correctamente vs base PEN, no vs MXN.
  - **Ausencia de cuentas GOLPERU (brecha B1) al momento de finalizar:** la finalización debe rechazarse por combo vacío/obligatorio; el auto-guardado del borrador debe seguir funcionando.
  - **Ausencia de catálogo interno de medio de pago Perú (brecha B2):** mismo comportamiento que B1 — auto-guardado sí, finalización no.

# Control de versiones

|  |  |  |  |  |  |
| :-: | :-: | :-: | :-: | :-: | :-: |
| **Versión** | **Fecha** | **Autor** | **Tipo de Cambio** | **Descripción del cambio** | **Aprobó** |
| 1.0 | 22/07/2026 | Osmar Calderón | Creación | Creación del documento. | Valdemar Farina Sánchez |
