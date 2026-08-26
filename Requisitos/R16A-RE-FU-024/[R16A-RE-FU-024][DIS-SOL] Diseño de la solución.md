# **[R16A-RE-FU-024][DIS-SOL] Diseño de la solución**

## Diseño de la solución

  

|  |  |
| :-: | :-: |
| **FORMATO** | N/A |
| **PROYECTO** | R16A |
| **REFERENCIA** | N/A |
| **VERSIÓN** | 1.0 |
| **FECHA** | 20/07/2026 |
| **AUTOR** | Osmar Calderón |
| **REVISOR** | Valdemar Farina Sánchez |

  

# **Importante**

Posterior a este diseño, ¿cómo saber si el diseño de la solución al requisito está completo para que el programador inicie con el desarrollo?

Hazte estas preguntas rápidas:

  - ¿El programador sabe qué flujo implementar?
  - ¿Sabe qué pasa si algo falla?
  - ¿Sabe qué reglas no puede romper?
  - ¿Sabe qué pruebas debe pasar?
  - ¿Sabe dónde impacta?

# **Introducción**

## **1. Propósito del documento**

El propósito de este documento es **definir el diseño de la solución técnica** para el requisito **R16A-RE-FU-024 — Validar Cobro: Paso 1 México (Captura del Cobro)**, describiendo los componentes, flujos, interfaces y consideraciones técnicas necesarias para su implementación.

**Nota:** Este documento se enfoca exclusivamente en el **diseño de la solución back-end**, no redefine requisitos funcionales.

## **2. Alcance**

Este análisis cubre únicamente el requisito antes mencionado: el **Paso 1 del wizard de Validar Cobro** (captura del cobro) para clientes de **Región México**.

### **Específicamente incluye:**

  - Listado de cobros del cliente en dos bloques (capturados / sin capturar) con estatus, canEdit e indicador de saldo a favor
  - Captura del cobro en fccCobroCliente (folio COB, monto, fecha del pago, forma de pago, cuentas, moneda, TC del día, notas)
  - **Auto-guardado transparente** (borrador, sin botón Guardar) y **finalización de la captura** con generación del folio COB
  - **Edición** del cobro capturado mientras no se timbre (botón Editar)
  - Cálculo automático de los **dos tipos de cambio** del cobro (fiscal + facturación) a la fecha del pago
  - Registro de **inconsistencias** del cobro (Paso 1)
  - **Reanudación del wizard** en el último paso activo (OBS-048)
  - **Fusión** de las tablas de origen en fccCobroCliente y adopción del catálogo catCobroEstatus como única fuente de verdad del ciclo de vida

### **No se consideran:**

  - Diseño del front-end
  - **Paso 2 (Asociar Factura/Proforma)** — RE-FU-026 — y **Paso 3 (Facturación y Envío)** — cubierto en su propio requisito. Este documento solo describe cómo el Paso 1 deja el cobro para ellos (estatus CAPTURADO)
  - **Región Perú** — misma UI, catálogos propios, **sin DDL nuevo** — se cubre en RE-FU-025
  - **Buzón de Cobros / clasificación del correo** — RE-FU-008 (crea la fila en estatus BORRADOR); aquí solo se consume y se continúa
  - **Alta del catálogo `catCobroEstatus`** — la ejecuta RE-FU-002 con el seed que define este requisito
  - **Importación del tipo de cambio** — la realiza el microservicio importador; RE-024 solo **lee** TipoDeCambioBanamex

# **Visión general del diseño**

## **1. Objetivo técnico**

Definir la forma en que el sistema permitirá al Gestor de Cobranza **capturar los cobros** de un cliente de Región México a partir de los correos del Buzón de Cobros, persistiéndolos en la tabla unificada **`fccCobroCliente`** mediante **Proquifa.Pqf2.Finanzas** (CQRS + Scaffold), con **auto-guardado transparente**, **folio COB** generado al finalizar, **edición hasta el timbrado** y cálculo automático de los **dos tipos de cambio** del cobro; usando **`catCobroEstatus`** como única fuente de verdad del ciclo de vida y reutilizando los catálogos existentes de ProquifaDotNet sin envolverlos.

## **2. Componentes involucrados**

|  |  |  |
| :-: | :-: | :-: |
| **Componente** | **Responsabilidad** | **Ubicación** |
| Proquifa.Pqf2.Finanzas | Orquestación del Paso 1 (CQRS): listado, auto-guardado, finalización con folio, edición, TCs del día, inconsistencia, estado del wizard | Solución .NET Core 10 |
| ProquifaDotNet | BD + catálogos existentes (correo/adjuntos, catMoneda, catMedioDePago, cuentas, TipoDeCambioBanamex) — consumidos **directo**, sin wrapper | Aplicativo .NET 4.8 |
| BD (SQL Server) | Persistencia del cobro (fccCobroCliente), catálogos (catCobroEstatus, catTipoInconsistenciaCobro), inconsistencias y foliador | Base de datos ProquifaDotNet |

**Diferencia clave respecto al KB:** *el KB modelaba el cobro en dos tablas (*fccFolioPagoCliente *del Buzón +* fccPagoCliente *de la captura) y el estado en banderas. Este diseño* **unifica ambas en `fccCobroCliente`** *y mueve el estado a* **`catCobroEstatus`** *— una sola fila por cobro, un solo eje de estado.*

# **Diseño funcional detallado**

## **1. Flujo técnico principal — Captura del Cobro (Paso 1)**

El Gestor entra al wizard desde la pantalla principal de Validar Cobro (RE-023). El registro inicial de cada cobro **ya existe** en fccCobroCliente con estatus BORRADOR (lo creó el Mailbot al clasificar el correo); el Paso 1 **lo continúa**, no lo duplica.

  

1\. Gestor presiona "Realizar Cobros" (RE-023)  
   → Front consulta GET .../client/{clientId}/wizard-status (Finanzas, OBS-048)  
   → abre el wizard en el paso activo (Paso 1 si hay borradores o correos sin capturar)  
  
2\. PASO 1 — Front carga:  
   a. Listado de cobros (Finanzas): dos bloques  
      - capturados arriba (FechaPago ASC): folio COB + fecha + monto con moneda  
      - sin capturar abajo (FechaRecepcion ASC): texto "Cobro sin capturar" + fecha de  
        recepción del correo (folio null); residual del Paso 2 = etiqueta "Saldo a favor"  
      DTO con status (estatus del cobro) / canEdit (derivado del estatus) / isCreditBalance  
   b. Detalle del correo + adjuntos: DIRECTO a GET /Logistica/ObtenerRequerimiento  
      (una sola llamada; ya agrupa CorreoRecibidoCliente / CorreoRecibido /  
      CorreoRecibidoContenido / ArchivoCorreoRecibido / Archivo) — sin wrapper  
   c. Combos del formulario: DIRECTO a /Catalogos/catMoneda (JOIN CatMonedaRegion),  
      catMedioDePago, vEmpresaDatosBancarios — sin wrapper  
  
3\. Captura:  
   a. El Gestor selecciona el adjunto que es el comprobante de pago oficial (radio, obligatorio)  
   b. Captura Monto, Fecha del cobro (día efectivo del pago), Forma de pago (SAT),  
      Cuenta origen (texto libre), Cuenta destino (combo), Moneda (combo), Notas  
      Fecha del cobro: valida no-futura (bloqueante); advierte (no bloqueante) si  
      es posterior a FechaRecepcion del correo  
   c. Cambio de moneda/fecha → GET exchange-rate: Finanzas calcula los DOS TCs de la FechaPago  
      (fiscal: pago vs MXN; facturación: pago ↔ moneda de facturación) — solo lectura en la UI  
   d. Navegación/salida (auto-guardado) O botón Editar → PUT payment/{id} (MISMO endpoint):  
      guardia: estatus IN (BORRADOR, CAPTURADO, ASOCIADO) — 409 si ya inmutable  
      recalcula los dos TCs si cambió moneda/fecha; NO regenera folio;  
      en BORRADOR FolioCobro=NULL; si estatus \!= BORRADOR exige obligatorios  
   e. "Finalizar captura" → POST payment/{id}/finalize:  
      valida comprobante + obligatorios → genera folio COB-mmddaa-NNNN (PaymentFolioGenerator)  
      → transición BORRADOR → CAPTURADO (+trazabilidad FechaConfirmacion / IdUsuarioCobrador)  
   f. "Marcar Inconsistencia" → modal → POST .../inconsistency:  
      estatus = CON_INCONSISTENCIA + INSERT fccInconsistenciaCobro (NO toca Activo)  
  
4\. Continuar → Paso 2 (RE-026) con los cobros capturados (habilitado si hay ≥ 1 cobro registrado)  
5\. (Desde Paso 3) al generar y enviar los documentos fiscales, el cobro transiciona a  
   COMPLETADO (o SALDO_A_FAVOR si queda residual) — inmutable

## **2. Reglas de negocio aplicadas**

|  |  |  |
| :-: | :-: | :-: |
| **Regla** | **Descripción** | **Cubierto en** |
| R1 | Aplica **solo a clientes Región México** (Perú = RE-025) | Flujo, filtro por región del token |
| R3 | Listado en **dos bloques**: capturados (FechaPago ASC) / sin capturar (FechaRecepcion ASC) | Flujo 2a, DTO |
| R5 | **Selección del comprobante obligatoria** para continuar | Flujo 3a, finalize |
| R6 | Formulario con Folio, Monto*, Fecha del cobro*, Forma de pago*, Cuenta origen/destino*, Moneda*, TC (lectura), Notas. Fecha del cobro:* no puede ser futura (bloqueante); advertencia (no bloqueante) si es posterior a FechaRecepcion del correo | Flujo 3b, DTOs |
| R7 | **TC del día**: se separa en dos campos — fiscal (pago vs MXN) + operativo (pago ↔ facturación). El operativo (TipoDeCambioFacturacion) es el que **RE-026 usa en el Paso 2** para convertir facturas/proformas/NCs a la moneda del cobro — no se recalcula con el TC de la factura | Flujo 3c, ExchangeRateService |
| R9/R13 | **Múltiples cobros** por sesión; Continuar habilitado con ≥ 1 cobro | Flujo 4 |
| R10 | **Auto-guardado transparente** + reanudación en el último paso activo (OBS-048) | Flujo 3d, wizard-status |
| R11 | **Editabilidad hasta el cierre del Paso 3**: editable en CAPTURADO/ASOCIADO; inmutable en COMPLETADO/SALDO\\_A\\_FAVOR | Flujo 3d, guardia 409 |
| R12 | Modal de inconsistencia del Paso 1 (catálogo filtrado AplicaPaso=1) | Flujo 3g |

## **3. Reglas técnicas aplicadas**

|  |  |
| :-: | :-: |
| **Regla** | **Descripción** |
| RT-01 | El cobro vive en **una sola tabla** fccCobroCliente (fusión de fccPagoCliente + fccFolioPagoCliente); relación 1:1 correo → cobro |
| RT-02 | El **estado del ciclo de vida vive solo en \\`catCobroEstatus\\`** (IdCatCobroEstatus); se retiran las flags Confirmado/BloqueadoPorTimbrado/FechaBloqueoTimbrado |
| RT-03 | La fila **nace en \\`BORRADOR\\`** (la crea el Mailbot, RE-008); el Paso 1 la continúa a CAPTURADO al finalizar |
| RT-04 | canEdit = estatus IN (BORRADOR, CAPTURADO, ASOCIADO); inmutable en COMPLETADO, SALDO\\_A\\_FAVOR y CANCELADO. Los endpoints de escritura devuelven **409** si el estatus ya es inmutable |
| RT-05 | El **folio COB** se genera **al finalizar** la captura (no en borrador): COB-MMDDYY-NNNN, 4 dígitos, wrap 9999 → 1, con la **\\`FechaPago\\`** del cobro |
| RT-06 | El foliador replica el **patrón de RE-013** (tabla con una fila por región + SP con UPDLOCK+ROWLOCK), **no** la SEQUENCE del KB — el consecutivo participa del rollback (sin huecos ni duplicados), y es **independiente por región** (MEX/PER) |
| RT-07 | Se calculan **dos TCs**: TipoDeCambioFiscal (pago vs MXN; NULL si no aplica) y TipoDeCambioFacturacion (pago ↔ facturación; **= 1 si coinciden**, NULL solo en borrador). Ambos se **congelan al capturar el cobro** — el Paso 2 y el REP los consumen tal cual, sin recalcularlos |
| RT-08 | El día del TC es la **\\`FechaPago\\`** (lectura DOF): fila de TipoDeCambioBanamex con esa fecha o la última anterior; valor de ValorDeCompra (oficial de la serie) |
| RT-09 | **\\`ExchangeRateService\\` único** para todas las regiones — la moneda base (MXN/PEN) se resuelve de la región del usuario; **solo lectura** de TipoDeCambioBanamex |
| RT-10 | El detalle del correo + adjuntos se obtiene en **una sola llamada** al servicio existente GET /Logistica/ObtenerRequerimiento; los combos (catMoneda, catMedioDePago, cuentas) de /Catalogos/\\*. Todo **directo** de ProquifaDotNet, sin wrapper de Finanzas |
| RT-11 | El combo Moneda = catMoneda **JOIN \\`CatMonedaRegion\\`** por la región del usuario (Activo=1 en ambas) — sin DDL nuevo |
| RT-12 | El "saldo a favor" del listado se deriva de IdCatCobroEstatus = SALDO\\_A\\_FAVOR y la columna SaldoAFavor — sin JOIN a fccSaldoFavorCliente |
| RT-13 | Marcar inconsistencia = IdCatCobroEstatus = CON\\_INCONSISTENCIA + INSERT en fccInconsistenciaCobro. **No toca \\`Activo\\`** — el cobro sigue contando como pendiente hasta resolverse |

## **4. Endpoints nuevos/modificados**

|  |  |  |  |
| :-: | :-: | :-: | :-: |
| **Solución / Proyecto** | **Endpoint** | **Tipo** | **Descripción** |
| Proquifa.Pqf2.Finanzas | GET /api/v1/validate-collection/payment?clientId={id} | Nuevo (def. RE-023) | Listado dos bloques del Paso 1; DTO con status, canEdit (derivado del estatus), isCreditBalance, emailReceptionDate, y el detalle completo del capturado (paymentMethod, sourceAccount, destinationAccount, exchangeRate, notes) — pinta el panel "Datos del Cobro" sin llamada adicional al seleccionar una fila |
| Proquifa.Pqf2.Finanzas | PUT /api/v1/validate-collection/payment/{id} | Nuevo (def. RE-023) | **Guarda el cobro** — **un solo endpoint para el auto-guardado (navegación/salida) y la edición (botón Editar)**: misma operación, difieren solo en el disparador. Guardia: estatus IN (BORRADOR, CAPTURADO, ASOCIADO) — 409 si ya inmutable (COMPLETADO/SALDO\\_A\\_FAVOR/CANCELADO). Recalcula los dos TCs si cambió moneda/fecha; **nunca regenera folio**; en BORRADOR FolioCobro=NULL; si estatus ≠ BORRADOR exige obligatorios. La fila **preexiste** (Mailbot la crea en BORRADOR) → **update-by-id, no upsert** |
| Proquifa.Pqf2.Finanzas | POST /api/v1/validate-collection/payment/{id}/finalize | Nuevo | Finaliza captura: valida comprobante + obligatorios, genera folio COB, transiciona BORRADOR → CAPTURADO |
| Proquifa.Pqf2.Finanzas | GET /api/v1/validate-collection/exchange-rate?paymentCurrencyId=\\\&billingCurrencyId=\\\&paymentDate= | Nuevo | Dos TCs de la FechaPago (lectura DOF): fiscal (pago vs moneda base regional) + operativo (pago ↔ facturación) |
| Proquifa.Pqf2.Finanzas | GET /api/v1/validate-collection/inconsistency-type?step=1 | Nuevo | Catálogo de tipos filtrado AplicaPaso=1 |
| Proquifa.Pqf2.Finanzas | POST /api/v1/validate-collection/client/{clientId}/payment/{paymentId}/inconsistency | Nuevo | Registra inconsistencia (INSERT fccInconsistenciaCobro) |
| Proquifa.Pqf2.Finanzas | GET /api/v1/validate-collection/client/{clientId}/wizard-status | Nuevo | Paso activo para reanudación (OBS-048): { activeStep, description, hasDraft, draftPaymentId } |
| ProquifaDotNet (existente) | GET /Logistica/ObtenerRequerimiento?IdCorreoRecibidoCliente={id} | Sin cambio | Detalle del correo + adjuntos **en una sola llamada** — el servicio existente ya agrupa CorreoRecibidoCliente, CorreoRecibido, CorreoRecibidoContenido (cuerpo), ArchivoCorreoRecibido y Archivo (adjuntos). Es el mismo que usa "Cotizar lo cotizable" (panel Requerimiento). Consumo directo, sin wrapper |
| ProquifaDotNet (existentes) | POST /Catalogos/{catMoneda, catMedioDePago, vEmpresaDatosBancarios} | Sin cambio | Combos del formulario — consumo directo (catMoneda con JOIN CatMonedaRegion) |

**Nota:** *Todos los endpoints* **nuevos** *siguen la convención del equipo: estructura* {ip}/api/v1/*, kebab-case e inglés (ruta y datos de entrada/salida). El KB usaba camelCase en algunos segmentos (*wizardStatus*,* inconsistencyType*,* exchangeRate*) — aquí se normalizan a kebab-case.*

# **Diseño de componentes**

## **1. Responsabilidades por componente**

**Proquifa.Pqf2.Finanzas — patrón CQRS:**

  - **Scaffold EF Core:** fccCobroCliente (tabla fusionada), catTipoInconsistenciaCobro, fccInconsistenciaCobro, catCobroEstatus, lecturas de catMoneda / DatosFacturacionCliente / TipoDeCambioBanamex (solo lectura). Ya **no** se scaffoldea fccFolioPagoCliente ni fccPagoCliente (DROP)
  - **Queries:** GetPaymentValidationStep1PaymentListQuery (con canEdit = estatus IN (BORRADOR, CAPTURADO, ASOCIADO)), GetWizardStatusQuery
  - **Commands:** SavePaymentCommand (auto-guardado + edición — un solo command, reemplaza a AutoSavePaymentDraftCommand + EditPaymentCommand), FinalizePaymentCaptureCommand, RegisterPaymentInconsistencyCommand
  - **Servicios:** IExchangeRateService / ExchangeRateService — GetExchangeRateAsync(paymentCurrencyId, billingCurrencyId, paymentDate). Retorna **ambos TCs**; servicio único para todas las regiones (la moneda base se resuelve de la región del usuario). ⚠️ El nombre colisiona con la solución importadora ExchangeRateService — distinguir por namespace o renombrar el servicio interno
  - **Foliador:** PaymentFolioGenerator + FccCobroClienteConsecutivoRepository (patrón RE-013)
  - **DTOs:** PaymentValidationStep1ItemDto (status, collectionFolio, paymentDate, amount, currencyCode, isCreditBalance, canEdit, emailReceptionDate, **`paymentMethod`, `sourceAccount`, `destinationAccount`, `exchangeRate` (`TipoDeCambioFacturacion`, el operativo — el que se muestra en la UI), `notes`** — completan el panel "Datos del Cobro" del capturado seleccionado, sin llamada adicional), SavePaymentRequestDto (auto-guardado + edición), RegisterInconsistencyRequestDto, WizardStatusDto

*Clases que espejean tablas conservan el nombre de la tabla; servicios, DTOs, queries/commands y métodos nuevos en inglés (convención R16).*

**ProquifaDotNet:** BD, catálogos y TipoDeCambioBanamex (consumo directo, sin cambios salvo el DDL de la sección Modelo de Datos).

## **2. Diagramas**

### **Flujo — Captura del Cobro (Paso 1)**

Muestra la secuencia del Paso 1. El registro nace en BORRADOR (Mailbot); el Gestor lo continúa capturando y finalizando. Finanzas orquesta el auto-guardado, el cálculo de los dos TCs y la generación del folio al finalizar; los catálogos y el detalle del correo se consumen directo de ProquifaDotNet.

  

sequenceDiagram  
    actor Gestor  
    participant Front  
    participant Finanzas as Proquifa.Pqf2.Finanzas  
    participant Pqf as ProquifaDotNet (Catálogos)  
    participant BD as SQL Server  
  
    Gestor-\>\>Front: "Realizar Cobros" (RE-023)  
    Front-\>\>Finanzas: GET .../client/{id}/wizard-status  
    Finanzas--\>\>Front: paso activo (OBS-048)  
    Front-\>\>Finanzas: GET .../payment?clientId  
    Finanzas-\>\>BD: SELECT fccCobroCliente (dos bloques)  
    Finanzas--\>\>Front: listado (status, canEdit, isCreditBalance, emailReceptionDate)  
    Front-\>\>Pqf: GET /Logistica/ObtenerRequerimiento (correo+adjuntos) + /Catalogos/* (catMoneda, cuentas) — directo  
    Gestor-\>\>Front: Selecciona comprobante + captura datos  
    Front-\>\>Finanzas: GET .../exchange-rate (paymentDate)  
    Finanzas-\>\>BD: SELECT TipoDeCambioBanamex (FechaPago, ValorDeCompra)  
    Finanzas--\>\>Front: dos TCs (fiscal + facturación)  
    Front-\>\>Finanzas: PUT .../payment/{id} (guarda: auto-guardado / editar)  
    Finanzas-\>\>BD: UPDATE fccCobroCliente (guardia estatus IN BORRADOR/CAPTURADO/ASOCIADO)  
    Gestor-\>\>Front: "Finalizar captura"  
    Front-\>\>Finanzas: POST .../payment/{id}/finalize  
    Finanzas-\>\>BD: sp_IncrementarFolioCobro (UPDLOCK+ROWLOCK) → COB-MMDDYY-NNNN  
    Finanzas-\>\>BD: UPDATE estatus BORRADOR → CAPTURADO (+FechaConfirmacion, IdUsuarioCobrador)  
    Finanzas--\>\>Front: cobro capturado

# **Diseño de Interfaces (Opcional)**

## **1. Interfaces de entrada**

|  |  |
| :-: | :-: |
| **Interfaz** | **Descripción** |
| PUT /api/v1/validate-collection/payment/{id} | Guarda el cobro — auto-guardado (borrador) y edición (botón Editar) en un solo endpoint; guardia estatus IN (BORRADOR, CAPTURADO, ASOCIADO) |
| POST /api/v1/validate-collection/payment/{id}/finalize | Finaliza captura, genera folio, transiciona a CAPTURADO |
| GET /api/v1/validate-collection/exchange-rate | Calcula los dos TCs de la FechaPago |
| POST .../payment/{paymentId}/inconsistency | Registra una inconsistencia del cobro |

## **2. Interfaces de salida**

|  |  |
| :-: | :-: |
| **Interfaz** | **Descripción** |
| Listado del Paso 1 | DTO con status, canEdit, isCreditBalance, collectionFolio, paymentDate, amount, currencyCode, emailReceptionDate, paymentMethod, sourceAccount, destinationAccount, exchangeRate, notes |
| Respuesta de TC | Dos TCs (fiscal + facturación) de la FechaPago |
| Estado del wizard | { activeStep, description, hasDraft, draftPaymentId } |

## **3. Contrato de datos conceptual**

**Item del listado sin capturar (estatus `BORRADOR`):** collectionFolio = null, el front pinta "Cobro sin capturar" + emailReceptionDate (= FechaRecepcion).

# **Diseño de Modelo de Datos**

## **1. Nuevos objetos**

Este requisito crea **la tabla unificada del cobro** más el catálogo de inconsistencias, la tabla de inconsistencias y el foliador; y **consume** el catálogo catCobroEstatus (que crea RE-002).

**Convención de tipos (tabla nueva ProquifaDotNet):** *nulabilidad explícita* NULL */* NOT NULL*; todos los* **decimales `decimal(18,6)`** *y todas las* **fechas `DateTime`***.*

### **fccCobroCliente** **(tabla nueva — fusión)**

Tabla **única** para el cobro en sus dos momentos: correo clasificado por el Mailbot (RE-008) → cobro capturado por el Gestor (Paso 1). Relación **1:1** (un correo → un cobro). IdCatCobroEstatus es el eje del ciclo de vida. **Es una tabla NUEVA (`CREATE TABLE`) — no es un rename;** fccPagoCliente y fccFolioPagoCliente se eliminan.

|  |  |  |
| :-: | :-: | :-: |
| **Campo** | **Tipo** | **Origen / Nota** |
| IdFCCCobroCliente | uniqueidentifier NOT NULL (PK) | — |
| IdCliente | uniqueidentifier NOT NULL (FK) | ← Cliente |
| IdEmpresa | uniqueidentifier **NULL** (FK) | Emisor (grupo PROQUIFA). **Nullable (confirmado 10/08/2026)** — antes NOT NULL |
| IdRegion | uniqueidentifier NOT NULL (FK) | **Nueva (confirmado 10/08/2026).** Región del cobro (MEX/PER) — heredada del cliente al capturarse/clasificarse; es el parámetro que usa el foliador (sp\\_IncrementarFolioCobro) para elegir la fila del consecutivo correcta |
| IdContactoCliente | uniqueidentifier NULL (FK) | Contacto del cliente |
| IdCatCobroEstatus | uniqueidentifier NOT NULL (FK → catCobroEstatus) | Estatus del ciclo de vida; **arranca en \\`BORRADOR\\`** |
| IdCorreoRecibidoCliente | uniqueidentifier NOT NULL (FK) | Correo del Buzón (Mailbot RE-008) |
| FechaRecepcion | DateTime NOT NULL | Fecha de recepción del correo |
| FolioCobro | varchar NULL | COB-mmddaa-NNNN; **NULL en borrador**, se genera al finalizar (antes Folio) |
| Monto | decimal(18,6) NOT NULL | Monto del cobro |
| FechaPago | DateTime NULL | Día efectivo del pago (no de llegada del correo) |
| IdCatMoneda | uniqueidentifier NULL (FK) | Moneda del cobro (combo catMoneda) |
| TipoDeCambioFiscal | decimal(18,6) NULL | Pago vs moneda fiscal regional (México: MXN → TipoCambioP del REP); **NULL si no aplica, nunca 1** (antes TipoDeCambio) |
| TipoDeCambioFacturacion | decimal(18,6) NULL | **Nueva.** Pago ↔ facturación; **= 1 si coinciden**; NULL solo en borrador; es el que se muestra en la UI. **Es el TC que RE-026 usa en el Paso 2 para convertir facturas/proformas/NCs a la moneda del cobro** — el del cobro, no el de la factura |
| IdCatMedioDePago | uniqueidentifier NULL (FK) | Forma de pago (SAT c\\_FormaPago vía catMedioDePago) |
| IdDatosBancarios | uniqueidentifier NULL (FK) | Cuenta destino (combo cuentas PROQUIFA MEX) |
| CuentaOrdenante | varchar NULL | Cuenta origen (texto libre) |
| ReferenciaBancaria | varchar NULL | Referencia bancaria del pago |
| IdArchivoComprobante | uniqueidentifier NULL (FK → Archivo) | Adjunto marcado como comprobante oficial (antes IdArchivo) |
| NotasDeCobro | varchar NULL | Notas del cobro (antes Notas) |
| FechaConfirmacion | DateTime NULL | Rastro de captura (se llena al finalizar) |
| IdUsuarioCobrador | uniqueidentifier NULL | Gestor que capturó (antes IdUsuarioConfirmacion) |
| IdCFDIComplementoPago | uniqueidentifier NULL (FK) | Es el CFDI que se timbra por el cobro mismo cuando el pago es a crédito.CFDI del Paso 3 (fiscal) |
| SaldoAFavor | decimal(18,6) NULL | **Nueva.** Pinta el saldo en el listado sin JOIN a fccSaldoFavorCliente |
| IdCorreoRecibidoClienteReferencia | uniqueidentifier NULL (FK → CorreoRecibidoClienteReferencia) | **Nueva (solicitud RE-008/FU-008 T7, 06/08/2026, aprobada 10/08/2026).** Llave idempotente contra reintentos de los 2 BOs legacy de T7 (que insertan varias filas por correo, una por archivo/referencia) — IdCorreoRecibidoCliente se repite por diseño y IdArchivoComprobante puede ser NULL, así que ninguna de las dos sirve como llave única. **Nullable** porque T19 (Mailbot nuevo) también inserta en fccCobroCliente sin crear ni conocer esta entidad legacy. **Índice único filtrado** (WHERE IS NOT NULL) — T7 siempre la puebla, T19 la deja vacía, sin conflicto entre ambos flujos |
| Activo | bit NOT NULL | Soft-delete **puro** — no lo usa ningún estado de negocio (la inconsistencia se expresa con el estatus) |
| FechaRegistro | DateTime NOT NULL | Control |
| FechaUltimaActualizacion | DateTime NOT NULL | Control |

  

**Los dos TCs.** TipoDeCambioFiscal (rename de TipoDeCambio) = moneda del pago vs la moneda fiscal de la región **si la región la exige** (México: MXN, alimenta el TipoCambioP del REP, solo si pago ≠ MXN; **Perú: siempre NULL** — sin Complemento de Pago SUNAT); NULL = no se emite en el XML, **nunca 1**. TipoDeCambioFacturacion (nueva) = TC operativo pago ↔ facturación, auto-calculado a la FechaPago (lectura DOF) y congelado al capturar; **= 1 cuando las monedas coinciden** (factor neutro, evita ISNULL(TC,1)); NULL solo en borrador. La moneda es IdCatMoneda — se retiran los flags MXN/USD.

### **catCobroEstatus** **(catálogo — lo crea RE-002, lo consume RE-024)**

Catálogo del ciclo de vida del cobro. **No es de este requisito:** lo crea **RE-002** con el seed que **define RE-024**, y la FK la pone **RE-008**. Claves y ciclo:

  

|  |  |  |
| :-: | :-: | :-: |
| **Clave** | **Descripción** | **Terminal** |
| BORRADOR | Correo clasificado como cobro (lo crea el Mailbot); captura en curso, FolioCobro NULL | No |
| CAPTURADO | Captura finalizada, folio COB generado; editable | No |
| ASOCIADO | Asociado a documento en el Paso 2 (aún sin timbrar); editable | No |
| SALDO\\_A\\_FAVOR | Cobro con residual disponible tras asociación (columna SaldoAFavor) |   **No**  |
| CON\\_INCONSISTENCIA | Marcado con inconsistencia en Paso 1 o Paso 2 — **\\`Activo\\` no se toca** | **Sí** |
| COMPLETADO | Documentos fiscales generados y enviados en Paso 3 | **Sí** |
| CANCELADO | Cancelado por falta de pago u otra razón operativa | **Sí** |

  

**Ciclo:** BORRADOR → CAPTURADO → ASOCIADO → COMPLETADO / SALDO_A_FAVOR; CON_INCONSISTENCIA / CANCELADO. **RE-024 solo asigna `BORRADOR → CAPTURADO`** (finalizar) y consume canEdit; ASOCIADO lo pone RE-026, COMPLETADO/SALDO_A_FAVOR el Paso 3.

**No existe un estatus `TIMBRADO`** *(confirmado 20/07/2026):* **el cobro no se timbra** *— lo que se timbra es el documento fiscal, así que un estatus "timbrado" en el cobro sería un error de categoría. La inmutabilidad la marca el cierre del Paso 3 (*COMPLETADO*) o* SALDO_A_FAVOR*.*

  

**Regla de conteo de cobros pendientes** (cartera de RE-023): cuentan todos los estatus **salvo los dos de cierre**.

COUNT(*) FROM fccCobroCliente  
WHERE IdCatCobroEstatus NOT IN (COMPLETADO, CANCELADO)

### **catTipoInconsistenciaCobro** **(catálogo nuevo)**

PK NEWID, Clave varchar(50) UNIQUE, Descripcion varchar(200), AplicaPaso Int(1) (1=cobro / 2=asociación), Activo bit. **Los valores concretos (4 de Paso 1 + 2 de Paso 2) aún no están definidos** — no existe una lista formal, solo referencias sueltas en R12 (ej. "datos incompletos", "comprobante inválido" en Paso 1; "Pago Incompleto Vencido" en Paso 2). **Definir el catálogo es responsabilidad de PROQUIFA Tesorería, no de este diseño** — este documento deja la estructura de la tabla lista para recibir el seed cuando Tesorería lo entregue. RE-026 lo extiende con sus propios valores de Paso 2.

### **fccInconsistenciaCobro** **(tabla nueva)**

PK NEWID, FK IdFCCCobroCliente (→ fccCobroCliente), FK IdCatTipoInconsistenciaCobro, Comentario varchar(500) NULL, IdUsuarioRegistro, Activo, FechaRegistro/FechaUltimaActualizacion (SYSUTCDATETIME). Sirve al Paso 1 **y** al Paso 2.

### **Foliador del COB —** **fccCobroClienteConsecutivo** **+** **sp_IncrementarFolioCobro**

  

|  |  |  |
| :-: | :-: | :-: |
| **Pieza** | **Capa** | **Responsabilidad** |
| fccCobroClienteConsecutivo | BD (tabla con **una fila por región**: IdRegion, Prefijo, Consecutivo, FechaUltimaActualizacion; seed 2 filas: MEX/COB/0 confirmado y PER/COB/0 **provisional** — ⚠️ Prefijo y Formato de Perú aún sin confirmar, 07/08/2026) | Contador del folio COB, independiente por región |
| sp\\_IncrementarFolioCobro | BD (stored procedure) | Recibe IdRegion; UPDLOCK + ROWLOCK sobre **la fila de esa región**, incrementa con **wrap 9999 → 1**, retorna el valor |
| FccCobroClienteConsecutivoRepository → PaymentFolioGenerator | Finanzas | IncrementAndGetAsync llama al SP con fccCobroCliente.IdRegion; GenerateAsync arma COB-MMDDYY-NNNN (4 dígitos) con la FechaPago |

**Concurrencia (igual que RE-013):** el lock vive desde la lectura hasta el commit del caller (FinalizePaymentCaptureCommand); si la finalización falla, rollback → el consecutivo no se incrementa. **Contrato transaccional:** GenerateAsync es **fail-fast** (verifica transacción activa; lanza InvalidOperationException si no la hay); el repository opera exclusivamente sobre el DbContext compartido de la unidad de trabajo. **Alcance: por región (confirmado 10/08/2026)** — MEX y PER llevan consecutivo independiente, cada uno con su propio wrap 9999 → 1.

## **2. Tablas y objetos eliminados**

|  |  |  |
| :-: | :-: | :-: |
| **Objeto** | **Acción** | **Motivo** |
| fccPagoCliente | **DROP TABLE** | Sus campos útiles quedan en fccCobroCliente (fusión) |
| fccFolioPagoCliente | **DROP TABLE** | Ídem — tabla del Buzón, sin datos |
| vFCCFolioPagoCliente | **DROP VIEW** | Vista sobre la tabla eliminada |
| SeqFolioCobro (KB) | **No se crea** | Reemplazada por el foliador patrón RE-013 |
| Entidades EF fccPagoCliente / fccFolioPagoCliente | Regenerar Scaffold como fccCobroCliente | — |

## **3. Relaciones**

  - fccCobroCliente.IdCatCobroEstatus → catCobroEstatus (N:1, requerido)
  - fccCobroCliente.IdCorreoRecibidoCliente → CorreoRecibidoCliente (N:1, requerido) — **1:1 lógico** (un correo → un cobro)
  - fccCobroCliente.IdCliente / IdEmpresa (nullable) / IdContactoCliente → catálogos de cliente
  - fccCobroCliente.IdCatMoneda → catMoneda; IdCatMedioDePago → catMedioDePago; IdDatosBancarios → DatosBancarios
  - fccCobroCliente.IdArchivoComprobante → Archivo (comprobante oficial)
  - fccCobroCliente.IdCorreoRecibidoClienteReferencia → CorreoRecibidoClienteReferencia (N:1, opcional) — llave idempotente de T7 (RE-008/FU-008)
  - fccInconsistenciaCobro.IdFCCCobroCliente → fccCobroCliente (1:N)
  - fccInconsistenciaCobro.IdCatTipoInconsistenciaCobro → catTipoInconsistenciaCobro (N:1)

## **4. Reglas de integridad**

  - Un fccCobroCliente tiene exactamente un IdCorreoRecibidoCliente (1:1) y un IdCatCobroEstatus válido
  - FolioCobro es NULL mientras el estatus sea BORRADOR; se llena (único) al pasar a CAPTURADO
  - TipoDeCambioFacturacion es NULL solo en borrador; en cobros capturados nunca es NULL (= 1 si las monedas coinciden)
  - TipoDeCambioFiscal es NULL cuando no aplica (nunca 1)
  - El foliador opera dentro de la transacción de finalización (sin huecos ni duplicados)
  - IdEmpresa es **nullable** (confirmado 10/08/2026) — puede no conocerse al insertar la fila en BORRADOR
  - IdCorreoRecibidoClienteReferencia es único cuando no es NULL (índice único filtrado) — T7 siempre la puebla, T19 (Mailbot) la deja vacía; nunca conviven dos filas con el mismo valor no nulo

## **5. Ciclo de vida del registro**

|  |  |  |
| :-: | :-: | :-: |
| **\\`IdCatCobroEstatus\\`** | **\\`FolioCobro\\`** | **UI Paso 1** |
| BORRADOR | NULL | Sin capturar: "Cobro sin capturar" + fecha de recepción (folio null); al abrirlo, formulario en edición (auto-guardado) |
| CAPTURADO | COB-mmddaa-NNNN | Lectura + **botón Editar** |
| ASOCIADO | COB-mmddaa-NNNN | Editable aún (Paso 2 asocia sin timbrar) |
| SALDO\\_A\\_FAVOR | COB-mmddaa-NNNN | Solo lectura (inmutable); el listado pinta "Saldo a favor" (columna SaldoAFavor) |
| COMPLETADO | COB-mmddaa-NNNN | Solo lectura (inmutable — cierra el ciclo) |
| CON\\_INCONSISTENCIA | COB-mmddaa-NNNN | Lectura; **\\`Activo\\` NO se toca** (sigue contando como pendiente) |
| CANCELADO | COB-mmddaa-NNNN | Solo lectura (inmutable) |

**El estatus es la única fuente de verdad del ciclo:** *se retiran las banderas* Confirmado*/*BloqueadoPorTimbrado*/*FechaBloqueoTimbrado*. Solo queda* Activo *(soft-delete).* canEdit = estatus IN (BORRADOR, CAPTURADO, ASOCIADO)*.*

## **6. Orden de ejecución de scripts**

1.  CREATE TABLE catCobroEstatus + seed — ⛔ **lo ejecuta RE-002** (prerrequisito de 2)
2.  CREATE TABLE fccCobroCliente (depende de catCobroEstatus, catMoneda, catMedioDePago, DatosBancarios, Archivo, CorreoRecibidoCliente, CorreoRecibidoClienteReferencia) — incluye IdCorreoRecibidoClienteReferencia (FK nullable) + índice único filtrado (D12)
3.  CREATE TABLE catTipoInconsistenciaCobro + datos iniciales
4.  CREATE TABLE fccInconsistenciaCobro (depende de 2 y 3)
5.  CREATE TABLE fccCobroClienteConsecutivo (seed 2 filas: MEX/COB/0, PER/COB/0) + CREATE PROCEDURE sp_IncrementarFolioCobro (parametrizado por IdRegion)
6.  DROP VIEW vFCCFolioPagoCliente; DROP TABLE fccFolioPagoCliente; DROP TABLE fccPagoCliente
7.  Regenerar el Scaffold EF de Finanzas
8.  Reescribir el Mailbot (CorreoRecibidoClienteToPagoBO.cs, RE-008) para insertar fccCobroCliente con estatus BORRADOR e IdRegion (heredado de Cliente.IdRegion vía IdCliente)

*El paso* **1 (catálogo de estatus) depende de RE-002***; los pasos 6-8 deben ejecutarse coordinados con RE-008/RE-023.*

# **Impacto Técnico**

## **1. Impacto en código existente**

**Proquifa.Pqf2.Finanzas (nueva solución):**

|  |  |
| :-: | :-: |
| **Pieza** | **Descripción** |
| Queries / Commands del Paso 1 | Listado, auto-guardado, finalización, edición, inconsistencia, estado del wizard |
| ExchangeRateService | Cálculo de los dos TCs (lectura de TipoDeCambioBanamex) |
| PaymentFolioGenerator + repositorio del consecutivo | Foliador COB (patrón RE-013) |
| Scaffold EF | fccCobroCliente, catCobroEstatus, catTipoInconsistenciaCobro, fccInconsistenciaCobro |

  

**ProquifaDotNet (código R14 — a coordinar):**

  - L11.MailBot\\Procesos\\Pagos\\CorreoRecibidoClienteToPagoBO.cs — hoy hace AddOrUpdate sobre fccFolioPagoCliente escribiendo Consecutivo, Stp, SubtotalMailBot, IvaMailBot, TotalMailBot, Folio. **Debe reescribirse** para insertar un fccCobroCliente con IdCatCobroEstatus = BORRADOR, IdCorreoRecibidoCliente, FechaRecepcion e **`IdRegion`** (heredado de Cliente.IdRegion vía IdCliente). Es código de **RE-008** — coordinar. **Confirmado (T7, FU-008, 06/08/2026):** puede insertar **varias filas por correo** (una por archivo/referencia); puebla también **`IdCorreoRecibidoClienteReferencia`** (llave idempotente, D12)
  - L11.MailBot\\MailBots\\Clientes\\CorreoRecibidoClienteReferenciaBO.cs — **confirmado (T7, FU-008, 06/08/2026):** es el segundo BO legacy que inserta en fccCobroCliente con IdCatCobroEstatus = BORRADOR, mismo patrón que el anterior — varias filas por correo, puebla IdCorreoRecibidoClienteReferencia (D12). Es código de **RE-008/FU-008** — coordinar
  - Logic.MailXslt\\Cobranza\\CorreoFCCPagoCliente.cs + Recursos\\CorreoFCCPagoCliente.xslt — verificar que no lea columnas dropeadas

*⚠️* **Colisión de nombre `ExchangeRateService`** *— el importador de TC (solución aparte) ya usa ese nombre. Distinguir por namespace o renombrar el servicio interno de Finanzas.*

## **2. Impacto en modelos**

  - Tabla nueva: **`fccCobroCliente`** (fusión) — reemplaza fccPagoCliente + fccFolioPagoCliente (ambas DROP) + vFCCFolioPagoCliente (DROP)
  - Catálogo nuevo: catTipoInconsistenciaCobro; tabla nueva: fccInconsistenciaCobro
  - Foliador nuevo: fccCobroClienteConsecutivo + sp_IncrementarFolioCobro
  - Catálogo consumido: catCobroEstatus (lo crea RE-002, FK por RE-008)

## **3. Impacto en despliegue (Opcional)**

  - Scripts de migración BD en el orden indicado (Modelo de Datos → Orden de ejecución)
  - Seed de catCobroEstatus (lo aplica RE-002) y de fccCobroClienteConsecutivo (2 filas: MEX/COB/0, PER/COB/0)
  - Regeneración del Scaffold EF en Finanzas
  - Reescritura coordinada del Mailbot (RE-008)

  
  

# **Decisiones Tomadas**

|  |  |
| :-: | :-: |
| **ID** | **Decisión** |
| D1 | **Fusión → \\`fccCobroCliente\\` (ACEPTADA).** fccPagoCliente + fccFolioPagoCliente se unifican en una **tabla nueva** (CREATE); ambas anteriores se DROP (más vFCCFolioPagoCliente). Motiva: relación 1:1 correo → cobro, frontera difusa que ya causó un error, y ambas tablas sin datos → DDL, no migración. Mismo patrón que fccFactura. Toca RE-008/RE-023/RE-026 — coordinar |
| D2 | **Estado del cobro solo en \\`catCobroEstatus\\`.** Se retiran las flags Confirmado/BloqueadoPorTimbrado/FechaBloqueoTimbrado; el estado vive en IdCatCobroEstatus y canEdit se deriva del estatus. Se conservan FechaConfirmacion/IdUsuarioCobrador (rastro de captura) y Activo (**soft-delete puro** — ya no lo usa la inconsistencia) |
| D2b | **Sin estatus \\`TIMBRADO\\` (20/07/2026, confirmado por el cliente).** Se había propuesto para expresar la inmutabilidad post-timbrado; se descarta porque **el cobro no se timbra** (lo timbrado es el documento fiscal). La inmutabilidad la marca COMPLETADO/SALDO\\_A\\_FAVOR. El conjunto editable **no cambia** (TIMBRADO iba entre ASOCIADO y COMPLETADO). Seed final: **7 claves**. ⚠️ Caso abierto: cobro aplicado a varios documentos con solo algunos timbrados → la inmutabilidad se deriva de la tabla de aplicación cobro-documento que define **RE-026** |
| D2c | **La inconsistencia ya no marca \\`Activo=0\\` (20/07/2026).** Pone IdCatCobroEstatus = CON\\_INCONSISTENCIA + INSERT en fccInconsistenciaCobro. El Activo=0 previo contradecía la regla de conteo confirmada (CON\\_INCONSISTENCIA **sí cuenta** como pendiente) y escondía la fila justo de las pantallas que deben mostrarla para resolverla |
| D3 | **Se adopta el catálogo \\`catCobroEstatus\\`.** Con el ciclo completo, ASOCIADO, SALDO\\_A\\_FAVOR y COMPLETADO no son derivables de las flags. Lo crea RE-002 con el seed que define RE-024; la FK la pone RE-008. El ciclo **arranca en \\`BORRADOR\\`** (sin un RECIBIDO separado) |
| D4 | **Dos TCs por cobro: fiscal + operativo.** TipoDeCambioFiscal (rename de TipoDeCambio) = pago vs moneda fiscal regional (Perú siempre NULL); TipoDeCambioFacturacion (nueva) = pago ↔ facturación (= 1 si coinciden). **Es el TC que RE-026 usa en el Paso 2** para convertir facturas/proformas/NCs a la moneda del cobro (congelado al capturar, no se recalcula con el TC de la factura). *Aceptado por el cliente — 10/08/2026 (DUDA-074)* |
| D5 | **Día del TC: \\`FechaPago\\` (lectura DOF).** Fila de TipoDeCambioBanamex con fecha = FechaPago (o la última anterior); siempre disponible al capturar; coincide con CFF Art. 20 y la tolerancia del Anexo 20. *Aceptado por el cliente — 10/08/2026 (DUDA-073)* |
| D5b | **Validaciones de Fecha del cobro (DUDA-073).** No puede ser futura (bloqueante); advertencia (no bloqueante) si es posterior a FechaRecepcion del correo. Mitiga que la fecha del pago la captura el usuario, no se extrae del comprobante. *Aceptado por el cliente — 10/08/2026* |
| D6 | **Valor del TC: \\`ValorDeCompra\\`** (el valor oficial de la serie; el FIX tal cual para USD/MXN). ValorDeVenta (× 1.025) no se usa para el cobro ni para el TipoCambioP, por exactitud fiscal/contable. *Aceptado por el cliente — 10/08/2026 (DUDA-073)* |
| D7 | **Servicio de TC único: \\`ExchangeRateService\\`.** No hay servicios por región; la moneda base (MXN/PEN) se resuelve de la región del usuario. Acceso vía Scaffold de TipoDeCambioBanamex en Finanzas (solo lectura), **directo a las columnas de la tabla** (ValorDeCompra/ValorDeVenta) — **nunca vía la vista \\`vTipoDeCambioBanamex\\` ni \\`ConversorDivisas\\`** (ese camino triangula vía USD y no filtra región/fecha; lo usan hoy cotización y la dolarización de RE-023, ajeno a Finanzas). ⚠️ Colisión de nombre con el importador |
| D8 | **Foliador patrón RE-013 (no SEQUENCE).** Tabla con **una fila por región** (MEX/PER) + SP con UPDLOCK+ROWLOCK (recibe IdRegion) + generator; el consecutivo participa del rollback. Formato COB-MMDDYY-NNNN (**4 dígitos**, wrap 9999 → 1, **independiente por región** — confirmado 10/08/2026) para **México**. Los scripts del KB a 6 dígitos quedan sin efecto. **⚠️ Perú (07/08/2026): \\`Prefijo\\` y \\`Formato\\` del folio aún sin confirmar** — el seed PER/COB/0 es provisional, no definitivo (compartido con RE-025, B5) |
| D8b | **\\`fccCobroCliente\\` agrega \\`IdRegion\\` (confirmado 10/08/2026).** El cobro guarda su propia región (MEX/PER) — no se resuelve vía "región del usuario" en sesión. Se hereda de Cliente.IdRegion vía IdCliente, y lo llena el Mailbot (RE-008) al clasificar el correo (mismo momento en que crea la fila en BORRADOR). Es el parámetro que consume el foliador (D8) para elegir la fila correcta de fccCobroClienteConsecutivo |
| D9 | **Consumo directo de servicios existentes.** Finanzas no los envuelve: el detalle del correo + adjuntos se obtiene con GET /Logistica/ObtenerRequerimiento (una sola llamada, mismo servicio que "Cotizar lo cotizable"), y los combos (moneda, medio de pago, cuentas) de /Catalogos/\\*; el front los consume directo (confirmado por tráfico HTTP). Se retiran las tareas KB \\#5 (catálogos del formulario) y \\#7 (detalle correo/adjuntos) — ya resueltas por el servicio existente |
| D10 | **Filtro regional del combo Moneda: \\`CatMonedaRegion\\` existente, sin DDL.** El combo = catMoneda JOIN CatMonedaRegion por la región del usuario; la brecha del KB ("catMoneda no tiene IdRegion") no existía |
| D11 | **Flags \\`MXN\\`/\\`USD\\`: se eliminan.** Reemplazados por IdCatMoneda (FK). Impacto a comunicar a RE-026 (su KB los lista entre los campos que lee del cobro) |
| D12 | **\\`fccCobroCliente\\` agrega \\`IdCorreoRecibidoClienteReferencia\\`, nullable + índice único filtrado (solicitud RE-008/FU-008 T7, aprobada 10/08/2026).** Llave idempotente contra reintentos de los 2 BOs legacy de T7, que insertan varias filas por correo (IdCorreoRecibidoCliente se repite por diseño; IdArchivoComprobante puede ser NULL — ninguna sirve como llave única). Nullable porque T19 (Mailbot) no crea esta entidad legacy y debe seguir insertando sin conflicto |
| D13 | **\\`IdEmpresa\\` pasa a nullable (confirmado 10/08/2026).** Antes NOT NULL |

  
  

# **Pendientes**

## **A definir con otros requisitos**

  - **Traza temporal de transiciones** del estatus (¿historial?) — a coordinar con RE-002/RE-008
  - **Catálogo de inconsistencias definitivo** — los valores (4 de Paso 1 + 2 de Paso 2) aún no están enumerados en ningún documento; **definirlos no es responsabilidad de este diseño**, le corresponde a **PROQUIFA Tesorería**
  - **`fccSaldoFavorCliente`** (dueño RE-026): confirmar el monto de la etiqueta y su FK al cobro; alinear su patrón MXN/USD al FK IdCatMoneda
  - **`Prefijo` y `Formato` del folio COB para Perú** (07/08/2026, compartido con RE-025 B5) — el consecutivo por región ya está confirmado; falta definir el prefijo y formato de Perú (el seed PER/COB/0 es provisional)

# **Manejo de Errores y Excepciones**

|  |  |
| :-: | :-: |
| **Escenario** | **Comportamiento esperado** |
| Falta el comprobante o un obligatorio al finalizar | Rechazar la finalización; el cobro queda en BORRADOR (auto-guardado). Sin folio |
| Editar/guardar un cobro ya inmutable (COMPLETADO/SALDO\\_A\\_FAVOR/CANCELADO) | Responder **409**; sin cambios en BD |
| Falla la finalización tras leer el consecutivo | Rollback: el consecutivo **no** se incrementa (sin huecos ni duplicados) |
| No hay fila de TC para la FechaPago | Usar la última fila anterior disponible (lectura DOF); si no hay ninguna, señalar el TC como no disponible |
| FechaPago futura | Rechazar (bloqueante) — no se guarda/finaliza con fecha futura |
| FechaPago posterior a FechaRecepcion del correo | Advertencia (no bloqueante) — el Gestor puede continuar; el pago no debería ocurrir después de su aviso |
| Marcar inconsistencia | IdCatCobroEstatus = CON\\_INCONSISTENCIA + INSERT en fccInconsistenciaCobro. **No** se toca Activo — el cobro sigue contando como pendiente hasta resolverse |
| Cambio de moneda/fecha en un cobro editable | Recalcular ambos TCs; **no** regenerar el folio |

# **Estrategia de Pruebas (Diseño de las pruebas)**

## **1. Pruebas funcionales (Criterios de Aceptación en DEV)**

  - El listado muestra dos bloques (capturados FechaPago ASC / sin capturar FechaRecepcion ASC); los sin capturar traen folio=null y emailReceptionDate
  - El auto-guardado deja el cobro en BORRADOR con FolioCobro=NULL
  - Finalizar valida comprobante + obligatorios, genera folio COB-MMDDYY-NNNN y transiciona a CAPTURADO
  - Editar funciona en CAPTURADO/ASOCIADO y responde 409 en COMPLETADO/SALDO_A_FAVOR/CANCELADO
  - El TC del día retorna dos valores (fiscal + facturación); facturación = 1 cuando las monedas coinciden; fiscal NULL cuando no aplica
  - Marcar inconsistencia pone estatus CON_INCONSISTENCIA e inserta en fccInconsistenciaCobro, **sin tocar `Activo`**; el cobro sigue apareciendo en el conteo de pendientes
  - El wizard reanuda en el último paso activo (OBS-048)
  - FechaPago futura se rechaza; FechaPago posterior a FechaRecepcion del correo solo advierte y permite continuar

## **2. Pruebas técnicas (unitarias e integración)**

  - Foliador: concurrencia con UPDLOCK+ROWLOCK; wrap 9999 → 1; rollback no incrementa el consecutivo
  - ExchangeRateService: lectura de TipoDeCambioBanamex por FechaPago (fila del día o última anterior); moneda base según región
  - Guardia 409 por estatus inmutable en el PUT payment/{id} unificado (auto-guardado y edición)
  - Mapeo del DTO (status, canEdit, isCreditBalance, emailReceptionDate, paymentMethod, sourceAccount, destinationAccount, exchangeRate, notes)

## **3. Casos críticos**

  - **Rollback de la finalización:** el consecutivo no debe quemar números
  - **Cobro con moneda = facturación:** TipoDeCambioFacturacion = 1 (no NULL)
  - **Cobro MXN en México:** TipoDeCambioFiscal NULL (no se emite TipoCambioP)
  - **Edición tras timbrado:** debe rechazarse (409)

  
  

# **Control de versiones**

|  |  |  |  |  |  |
| :-: | :-: | :-: | :-: | :-: | :-: |
| **Versión** | **Fecha** | **Autor** | **Tipo de Cambio** | **Descripción del cambio** | **Aprobó** |
| 1.0 | 20/07/2026 | Osmar Calderón | Creación | Creación del documento. | Valdemar Farina Sánchez |

  