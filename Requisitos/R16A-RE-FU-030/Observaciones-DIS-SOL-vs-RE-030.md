# Observaciones — DIS-SOL Complemento de Pago vs R16A-RE-FU-030

| Campo | Valor |
|---|---|
| **Documento revisado** | `DIS-SOL-backend-complemento-pago.md` (v1.0, 24/06/2026) |
| **Requisito de referencia** | `R16A-RE-FU-030.md` — Diseño y generación de Documentos: CDP México |
| **Documentos complementarios consultados** | `R16A-RE-FU-030-Back.md`, `R16A-RE-FU-030_BD.md` |
| **Fecha de revisión** | 2026-07-17 |
| **Revisor** | Juan David García |
| **Veredicto general** | **Cumple parcialmente** — cubre la mayor parte del alcance funcional, pero contiene 3 desviaciones que deben resolverse antes de codificar |

---

## 1. Resumen ejecutivo

El DIS-SOL cubre los cuatro aplicativos que el requisito exige (ProquifaDotNet, ProquifaDotNet.Finanzas, ProquifaDotNet.Timbrado, DocumentBuilder), respeta la mayoría de las reglas de negocio y criterios de aceptación (Secciones A a J), y añade decisiones arquitectónicas razonables sobre foliado sin huecos, checkpoint de fase e idempotencia por `IdPeticionCP` que **cierran huecos que el requisito dejó abiertos** (P5 reintento, P12 dónde vive el `INSERT CFDIGenerada`).

Sin embargo, presenta **3 desviaciones críticas** respecto al requisito literal, **2 ampliaciones fuera de alcance** que conviene justificar o retirar, y **1 mismatch operativo** (nombre del PAC) que debe cerrarse antes de implementar. Las coincidencias, desviaciones y pendientes se detallan en las secciones siguientes.

---

## 2. Lo que sí cumple — coincidencias verificadas

| Sección del requisito | Criterio | Cobertura en el DIS-SOL |
|---|---|---|
| **A — Detonante** | A1, A2, A3 | Cascada PPD post-timbrado de Factura PPD (Escenario B) y generación desde FAA existente (Escenario D). Guard "MetodoPago=PPD" en `StampPaymentComplementAsync`. |
| **B — Cabecera XML** | B1, B3 | Cabecera fija correcta (`Version=4.0`, `TipoDeComprobante=P`, `Exportacion=01`, `SubTotal=0`, `Total=0`, `Moneda=XXX`) y Concepto único fijo (`ClaveProdServ=84111506`, `ClaveUnidad=ACT`, `ObjetoImp=01`). |
| **C — Emisor/Receptor** | C1, C2, C3 | `IssuerCp` (RFC + Nombre + `TaxRegime=601`), `ReceiverCp` (RFC + Nombre + CP + Régimen Fiscal), `UsoCFDI=CP01` fijo en el mapper. |
| **D — Nodo Pago** | D2, D3, D4 | `FormaDePagoP` real desde `catFormaPagoSAT` con guard explícito contra `"99"`; `MonedaP`+`Monto`+`TipoCambioP` calculados por `PaymentComplementCalculationService`. |
| **E — DoctoRelacionado** | E2, E3, E4, E5 | `NumParcialidad` vía `COUNT+UPDLOCK` (concurrencia resuelta), `ImpSaldoInsoluto = ImpSaldoAnt − ImpPagado`, `EquivalenciaDR=1` explícito cuando monedas coinciden. |
| **F — Impuestos DR** | F1, F2 | `ObjetoImpDR` heredado de la factura relacionada (`ResolveRelatedDocumentTaxObject()`), sub-nodo `ImpuestosDR` solo cuando `"02"`. |
| **G — Totales** | G1, G2, G3 | `CalculateTotalPaymentsAmountMxn()` convierte a MXN; `CalculateVat16Totals()` omite bases IVA cuando `ObjetoImpDR="01"`. |
| **H — Timbrado y persistencia** | H1, H2, H3, H4, H5 | PAC (ver Observación 3.4), UUID recibido y persistido, PDF+XML en MinIO bucket `cobranza`, política de fallo documentada, conservación 5 años delegada al bucket. |
| **I — PDF** | I1, I3, I4, I5, I6, I8, I9, I10, I11, I12, I13, I14 | Cuatro `TemplateKey` (`GOL/MUN/PRO/PQF_MEX_CP`), `DigitalSealCp` con los 5 campos SAT y QR delegado a la plantilla HTML. |
| **J — Envío por correo** | J1 (parcial) | Reconoce que el envío es responsabilidad de RE-028 y no lo duplica. |
| **Regla 15** | NC no es DoctoRelacionado | Salvaguarda explícita en `StampPaymentComplementAsync` — no arma nodo `CFDIRelacionados` con NC. |

**Decisiones arquitectónicas positivas que exceden el requisito:**
- Foliado sin huecos: el folio Serie "P" se consume **después** del timbrado exitoso, en la misma transacción que el `INSERT CFDIGenerada`. Ningún intento fallido quema un folio.
- Idempotencia por `IdPeticionCP` en Timbrado: evita doble-timbrado ante redelivery de RabbitMQ o crash de Finanzas.
- Checkpoint de fase (`TIMBRADO → FOLIO_ASIGNADO → PDF_GENERADO → ARCHIVOS_SUBIDOS`) para hacer reanudable `ExecuteAttemptAsync` sin repetir pasos ya completados.

---

## 3. Observaciones críticas — bloquean implementación

### 3.1 — Contradicción de la política "1 CP por factura" en los DTOs de DocumentBuilder

**Referencia del requisito:** Regla 2, Criterio A3, Criterio E1 — *"Cada Complemento de Pago contiene exactamente un Pago con un único DoctoRelacionado"*.

**Lo que el DIS-SOL propone:**
- `PaymentComplementModel.RelatedDocuments` está tipado como `List<RelatedDocumentCp>` con `NotEmpty (mín. 1)` en el validator — permite N documentos relacionados.
- La sección "TemplateKeys y variantes soportadas" documenta plantillas con múltiples DR: **`MUN_MEX_CP` con 9 DRs**, **`PQF_MEX_CP` con 2 DRs**.
- El ejemplo de payload `MUN_MEX_CP` incluye 2 DR (abreviado de 9), y `PQF_MEX_CP` incluye 2 DR.
- Las Pruebas técnicas dicen literalmente: *"Enviar request válido con `MUN_MEX_CP` (9 DRs) → verificar que el PDF incluye todos los documentos relacionados"* y *"Múltiples DRs (9 docs en `MUN_MEX_CP`): verificar que la tabla de DRs pagina correctamente en el PDF"*.

**Problema:** el modelo, los ejemplos y las pruebas violan la regla operativa central del requisito. El DIS-SOL simultáneamente:
- En la sección "ProquifaDotNet.Timbrado" dice *"`StampPaymentComplementAsync` construye un único `DoctoRelacionado` (la factura/FAA que se está pagando) y nada más"* — cumple Regla 2.
- En "Diseño funcional detallado" (endpoint DocumentBuilder) modela y prueba con múltiples DR — contradice Regla 2.

**Acción requerida:** definir con PMO cuál interpretación gana:
1. **Opción A (recomendada, alinea con el requisito):** cambiar `RelatedDocuments` a un único objeto `RelatedDocumentCp` (no lista), agregar `RuleFor(x => x.RelatedDocuments).Must(l => l.Count == 1)`, borrar los ejemplos de payload con múltiples DR, retirar el caso de prueba "9 DRs" de `MUN_MEX_CP`.
2. **Opción B:** documentar por escrito que el endpoint DocumentBuilder acepta N DR "por compatibilidad con CFDIs históricos 3.3" (ver también Observación 4.1), pero el orquestador de Finanzas garantiza que siempre envía uno solo. Esto exige quitar el caso "9 DRs" de las pruebas obligatorias.

### 3.2 — Serie y Folio: tabla foliadora faltante y omisión del XML firmado

**Referencia del requisito:** Regla 12 y Criterio B2 — *"Foliado consecutivo continuo por empresa emisora, Serie 'P' propuesta. El UUID lo asigna el SAT. Esquema del foliador con serie 'P' pendiente de validar."*

#### 3.2.1 — Tabla foliadora `fccFolioDocumentoFiscalCobro` no definida en el DIS-SOL

El DIS-SOL hace referencia a una tabla `EmpresaFolio` (en el mecanismo de rollback del folio) pero no define su estructura ni aclara si es la misma tabla que gestiona el foliado de otros documentos fiscales. Se propone una tabla propia para el foliado del CP:

**Tabla propuesta: `dbo.fccFolioDocumentoFiscalCobro`**

| Columna | Tipo | Descripción |
|---|---|---|
| `IdFccFolioDocumentoFiscalCobro` | uniqueidentifier PK | Identificador |
| `IdEmpresa` | uniqueidentifier FK → Empresa | Empresa emisora (GOL / MUN / PRO / PQF) |
| `Serie` | varchar(10) NOT NULL | Serie del folio — `'P'` para CDP |
| `UltimoFolio` | int NOT NULL DEFAULT 0 | Último folio consumido exitosamente |
| `FechaActualizacion` | datetime2(7) NOT NULL | Timestamp del último `UPDATE` |

**Sobre el foliado por empresa:** el foliado por empresa es **correcto y necesario** para CFDI. Cada empresa emisora tiene su propio RFC, y el SAT valida la unicidad de Serie + Folio dentro del mismo RFC emisor. GOL con P-000001 y MUN con P-000001 son folios distintos a efectos fiscales porque pertenecen a distintos emisores. Desde el punto de vista del negocio, el PDF y el XML siempre incluyen la Razón Social y RFC del emisor, por lo que no hay ambigüedad operativa. El foliado por empresa es el patrón correcto.

**Índice recomendado:**

```sql
-- Garantía de una sola fila por empresa+serie (restricción de unicidad)
CREATE UNIQUE INDEX UQ_fccFolioDocumentoFiscalCobro_EmpresaSerie
    ON dbo.fccFolioDocumentoFiscalCobro (IdEmpresa, Serie);
```

**Acción requerida:** agregar la definición de `fccFolioDocumentoFiscalCobro` al Impacto en BD del DIS-SOL, junto con el `INSERT` inicial de los 4 registros (uno por empresa: GOL, MUN, PRO, PQF) con `UltimoFolio = 0`. Reemplazar las referencias a `EmpresaFolio` por `fccFolioDocumentoFiscalCobro`.

#### 3.2.2 — Serie y Folio omitidos del XML firmado

**Lo que el DIS-SOL propone:** *"La cabecera **no incluye `Serie`/`Folio`**. El SAT no los exige — son atributos opcionales del esquema CFDI 4.0, y el UUID (obligatorio) lo asigna el SAT independientemente de ellos. El folio interno de negocio (Serie 'P') se asigna en Finanzas, después de recibir esta respuesta exitosa — así se garantiza que ningún intento fallido de timbrado queme un folio."*

**Problema:** la decisión técnica de asignar el folio después del timbrado para evitar huecos es razonable, pero al omitirlos del XML **firmado y timbrado** se contradice el Criterio B2 literalmente. El folio termina viviendo solo en `CFDIGenerada`, no en el XML certificado que se entrega al cliente ni en el archivo XML que se persiste en MinIO 5 años.

**Consecuencia práctica:** el cliente recibe un XML sin `Serie`/`Folio` en la cabecera — el UUID no lo suple para trazabilidad interna ni conciliación contable, y el PDF mostraría un folio que el XML no lleva.

**Acción requerida:** ampliar el diseño para **incluir Serie y Folio en el XML antes de firmar**, sin renunciar a "cero huecos". Alternativas:
1. **Reservar folio, firmar con folio incluido, si falla el timbrado revertir el consumo del folio** (`UPDATE fccFolioDocumentoFiscalCobro` con rollback) — exige transacción compensatoria entre Finanzas y Timbrado.
2. **Reserva y reciclado:** consumir folio antes de timbrar; si falla, marcar la fila en `fccFolioDocumentoFiscalCobro` como "folio hueco reciclable" y reutilizarlo en el siguiente intento exitoso. Rompe "cero huecos" solo transitoriamente.
3. **Re-firma post-timbrado:** descartar — rompe el sello digital del SAT.
4. **Doble invocación al PAC:** descartar — costo y tiempo.

**Recomendación:** validar con PMO si la interpretación del Criterio B2 se satisface con `Folio` como atributo interno no reflejado en el XML SAT (interpretación del DIS-SOL), o si debe estar dentro del XML certificado (interpretación literal). El criterio, tal como está escrito, apunta a la segunda.

### 3.3 — Destinatario "analista de Cuentas por Cobrar" no queda garantizado

**Referencia del requisito:** Criterio J2 — *"Para = contacto del cliente vinculado a la factura, CC = ESAC asignado + analista de Cuentas por Cobrar"*.

**Lo que el DIS-SOL propone:** *"Corrección detectada para RE-028: la lista de destinatarios documentada en RE-028 B6 (`Para` = contacto del pedido, `CC` = ESAC) **no incluye** al 'analista de Cuentas por Cobrar' que sí exige el Criterio J2 de este requisito para las líneas de tipo Complemento de Pago. Coordinar con el mantenedor de RE-028 para agregar ese destinatario en CC cuando la línea enviada sea `COMPLEMENTO_PAGO`."*

**Problema:** el DIS-SOL identifica correctamente la brecha, pero **la deja abierta como coordinación pendiente**. Sin cambio efectivo en RE-028 o en este mismo requisito, el CP se enviará sin el destinatario que el Criterio J2 exige, y el requisito quedará sin cumplirse aunque su diseño esté "aprobado".

**Acción requerida:** convertir la nota en un pendiente formal con dueño y fecha (P18), o incorporar en este mismo requisito un cambio a la lógica de armado de destinatarios que **detecte líneas `COMPLEMENTO_PAGO`** y agregue el analista de Cuentas por Cobrar al CC en el modal de RE-028. La segunda opción es preferible porque mantiene la responsabilidad dentro del requisito que lo exige.

### 3.4 — Mismatch de nombre del PAC entre requisito y diseño de RE-018

**Referencia del requisito:** Alcance y Criterio H1 — *"Timbrado vía PAC TurboPac"*.

**Lo que el DIS-SOL reporta:** *"⚠️ **A confirmar:** el requisito funcional (`R16A-RE-FU-030.md`) llama al PAC 'TurboPac'; el diseño Back de RE-018 (creación de `ProquifaDotNet.Timbrado`) llama al cliente `SapTimbradoClient` (PAC 'SAP'). Mismatch de nombre entre el requisito funcional y el diseño técnico — confirmar cuál es el proveedor real antes de implementar."*

**Problema:** el DIS-SOL identifica el mismatch pero no lo resuelve. Si el PAC real es SAP (implementado en RE-018), el requisito RE-030 tiene un dato incorrecto y hay que actualizarlo. Si el PAC real es TurboPac, RE-018 tiene el cliente equivocado y RE-030 hereda el error.

**Acción requerida:** cerrar la duda con PMO/arquitectura **antes** de codificar cualquier ampliación de `TimbradoService`. Es un cambio de una línea en cualquier lado, pero define expectativas contractuales con un tercero (PAC) que no se puede diseñar "por probar".

---

## 4. Ampliaciones fuera del alcance del requisito

### 4.1 — Soporte de CFDI 3.3 / Pagos 1.0

**Referencia del requisito:** Regla 4 — *"Version=4.0"* (única versión mencionada). El requisito no menciona CFDI 3.3 ni Pagos 1.0 en ninguna sección.

**Lo que el DIS-SOL propone:**
- `PaymentComplementModel.CfdiVersion` acepta `"4.0"` o `"3.3"`.
- `PaymentComplementModel.PaymentsVersion` acepta `"2.0"` o `"1.0"`.
- Validator: `CfdiUse` debe ser `"CP01"` **o `"P01"`** (`P01` es el uso CFDI 3.3).
- `TemplateKey` `MUN_MEX_CP` documentado como *"CFDI 3.3"*.
- Regla técnica RT-02 lo hace explícito: *"CP01 para CFDI 4.0; P01 para compatibilidad con CFDI 3.3 históricos"*.

**Análisis:** el CFDI 3.3 dejó de ser emitible en México desde abril de 2023 (obligatorio 4.0). No se puede timbrar un nuevo CFDI 3.3 hoy. Los únicos casos donde tendría sentido son:
- **Re-generación de PDFs de CPs históricos** timbrados con 3.3 (archivo de consulta) — pero el requisito RE-030 es de **generación**, no de consulta ni re-render.
- **Migración de datos legados** desde el sistema previo, si esos CPs históricos se re-modelan en el nuevo esquema.

**Acción requerida:** confirmar con PMO cuál de los siguientes aplica:
1. Retirar el soporte de 3.3/Pagos 1.0 del alcance de RE-030 (el requisito solo pidió 4.0).
2. Justificar por escrito que se incluye por consulta de CPs históricos, documentar esa justificación en el requisito, y ampliar los criterios de aceptación (I1-I14 se escribieron para 4.0).

### 4.2 — Reintento asíncrono vía RabbitMQ

**Referencia del requisito:** Notas Adicionales / Pendientes — *"Política formal de reintento ante fallo de timbrado PAC (transversal con Factura por Adelantado, Notas de Crédito y Validar Cobro)"* → dejado como brecha transversal, no definida en este requisito.

**Lo que el DIS-SOL propone:** mecanismo completo — tabla `ValorConfiguracion` (ver Obs. 4.3 — ya existe como `VariableConfiguracion`), columnas de checkpoint (`IdPeticionCP`, `NumeroReintento`, `FechaUltimoReintento`), 4 nuevas claves en `catDocumentoFiscalCobroEstado`, `PaymentComplementRetryMessage`, `PaymentComplementRetryWorker`, proyecto Worker nuevo, cliente `IRabbitMQClient` con método `ConsumeAsync` inexistente en el template base (P16), estrategia de backoff pendiente.

**Análisis:** técnicamente sólido, pero es un mecanismo **transversal** que el requisito RE-030 no debería definir en solitario. Riesgo real: cuando el requisito de Notas de Crédito o Factura por Adelantado defina su propio reintento, hay probabilidad alta de divergencia (nombres de tablas, granularidad del contador, política de backoff, número máximo de reintentos, ubicación del `IValorConfiguracionRepository`, etc.).

**Acción requerida:** dos opciones:
1. **Extraer el diseño de reintento a un requisito/documento transversal propio** (ej. R16A-RE-FU-XXX — Política de Reintento de Timbrado) y en RE-030 solo referenciarlo, no definirlo. Recomendado.
2. Mantenerlo aquí, pero marcarlo explícitamente como *"diseño de referencia, sujeto a re-armado cuando se defina el requisito transversal"* — y coordinar con los dueños de RE-Notas de Crédito y RE-Factura Anticipo antes de codear.

### 4.3 — `ValorConfiguracion` ya existe como `VariableConfiguracion`

**Contexto:** el DIS-SOL propone `CREATE TABLE dbo.ValorConfiguracion` en ProquifaDotNet para almacenar `MAX_REINTENTOS_TIMBRADO=5`.

**Hallazgo:** la tabla **ya existe** en ProquifaDotNet con el nombre `dbo.VariableConfiguracion`. El DIS-SOL usa un nombre incorrecto y plantea crearla como si fuera nueva.

**Problema:** si se ejecuta el script tal como está en el DIS-SOL se creará una tabla duplicada (`ValorConfiguracion`) paralela a la existente (`VariableConfiguracion`), con riesgo de inconsistencia de datos de configuración entre módulos.

**Acción requerida:**
1. Eliminar el `CREATE TABLE dbo.ValorConfiguracion` del DIS-SOL.
2. Reemplazar todas las referencias a `ValorConfiguracion` y a `IValorConfiguracionRepository` en el diseño por `VariableConfiguracion` y `IVariableConfiguracionRepository` (nombre consistente con la tabla existente).
3. Insertar la clave `MAX_REINTENTOS_TIMBRADO=5` en la tabla existente `dbo.VariableConfiguracion` mediante un `INSERT` (o `MERGE` para idempotencia), no un `CREATE TABLE`.

---

## 5. Observaciones medias — mejorables antes de aprobar

### 5.1 — Soporte de tasas de IVA distintas a 16%

**Requisito Criterio F3 (pendiente):** *"Confirmar soporte para tasas distintas a 16%"*.

**DIS-SOL:** implementa hardcodeado para 16% (`CalculateVat16Totals()`, campos `Vat16BaseMxn`/`Vat16AmountMxn`, constante `TasaOCuotaDR = 0.160000`). Consistente con el pendiente P3, pero no deja diseño extensible — cuando el pendiente se cierre y se acepte 8% frontera o 0%, hay que renombrar métodos, DTOs y validators.

**Recomendación:** parametrizar la tasa desde ya (ej. `CalculateTaxTotals(rate)`), aunque el uso inicial siga siendo 16%. Evita refactor mayor cuando se cierre P3.

### 5.2 — Concepto fijo del CFDI no aparece en el `PaymentComplementModel`

**Requisito Criterio I7:** el PDF debe mostrar la sección Concepto con todos sus campos fijos (`ClaveProdServ=84111506`, `Cantidad=1`, `ClaveUnidad=ACT`, `Descripcion=Pago`, `ValorUnitario=0.00`, `Importe=0.00`, `Subtotal=0.00`, `Total=0.00`, `Moneda=XXX`, `Total en letra "CERO XXX 00/100"`).

**DIS-SOL:** el `PaymentComplementModel` **no incluye un objeto `Concept`**. Se asume implícitamente que la plantilla HTML lo renderiza como texto fijo, o que se hardcodea en el body de cada `{TemplateKey}_B.html`.

**Problema:** si el Concepto vive solo en las plantillas HTML, cualquier cambio (aunque improbable — es una regla SAT) exige tocar 4 archivos HTML. Además, no queda trazabilidad en el modelo de que esos valores existieron.

**Recomendación:** agregar `Concept` como sub-modelo (aunque sea fijo) al `PaymentComplementModel`, poblado por el mapping service con los valores del Criterio I7. Costo mínimo, ganancia de trazabilidad y aislamiento de cambios.

### 5.3 — Iconografía de certificaciones (Criterio I2) delegada sin trazabilidad

**Requisito Criterio I2:** *"Deberá incluir la iconografía de certificaciones del giro químico-farmacéutico consistente con la factura"*.

**DIS-SOL:** delegado a *"la tarea de DocumentBuilder dedicada a los templates"* — no se documenta qué certificaciones específicas ni de dónde salen las imágenes.

**Recomendación:** al menos referenciar el pendiente de "vigencia de la iconografía de certificaciones" del requisito (ISO, NEEC, edQM, FELUM, USP, Microbiologics, APACOR, CHATA, Pharmaffiliates, Amex) y aclarar si se reutiliza el asset ya definido en templates `*_MEX_FAC` o se crea nuevo.

### 5.4 — Método `UploadFilesAsync` re-invoca `StampPaymentComplementAsync` para recuperar el XML

**Diseño en DIS-SOL:** para reanudar un flujo interrumpido después de generar el PDF pero antes de subir el XML, `UploadFilesAsync` **re-invoca** `StampPaymentComplementAsync` con el mismo `IdPeticionCP` para que la idempotencia devuelva el XML ya certificado desde `TimbradoLog`.

**Análisis:** funciona, pero es una llamada HTTP entre Finanzas y Timbrado solo para leer un dato que ya se persistió. Además, obliga a construir un `StampPaymentComplementRequestDto` completo (o con "resto ignorado si ya está timbrado", según el comentario del código) — la firma del método no comunica esa semántica de "lectura".

**Recomendación:** exponer un endpoint dedicado `GET /api/v1/paymentComplement/stamp/{idPeticionCP}` (o similar) en `TimbradoController` que devuelva el XML de `TimbradoLog` sin la ceremonia de un timbrado. Reduce acoplamiento y hace explícita la operación de lectura.

### 5.5 — `Total` en `CFDIGenerada` del CP siempre en `0m`

**Diseño:** *"`Total = 0m // CFDI tipo P: SubTotal=Total=0 (Regla 4)`"*.

**Observación:** correcto por el CFDI, pero `CFDIGenerada.Total = 0` impide que `CalculatePreviousBalance` para NumParcialidad=1 use `.Total` como fuente si aplica al mismo caso. Verificar que `CalculatePreviousBalance` **nunca** se usa contra un CP (siempre contra la Factura PPD/FAA relacionada). Del código actual se ve que sí lo hace bien, pero conviene un comentario o assertion.

---

## 6. Observaciones bajas / de estilo

- **6.1** — El diagrama de flujo end-to-end (sección "Diagrama") menciona `[RE-028+RE-030]` en pasos que en realidad son 100% RE-030 (agregado de columnas y actualización de estado). Simplificable.
- **6.2** — Los 3 diagramas Mermaid duplican mucho contenido (happy path + retry + vista general). Consolidar a 2 (happy path y retry) reduce mantenimiento.
- **6.3** — La sección "Impacto en código existente" lista `TimbradoLog (sin cambio de esquema)` — ese renglón es informativo, no un cambio; separarlo en una sub-lista de "referencias".
- **6.4** — La justificación de por qué `CalculateInstallmentNumber` usa SQL crudo con UPDLOCK y no LINQ es muy larga (12 líneas de comentario en el código); moverla a un archivo de arquitectura y dejar solo `// See manual-bloqueo-pesimista-fase-terminal.md`.
- **6.5** — En Ejemplos de payload, `MUN_MEX_CP` (CFDI 3.3) tiene `paymentsVersion: "1.0"` pero también `CfdiUse: "P01"`. Si se decide retirar 3.3 del alcance (ver 4.1), este ejemplo debe eliminarse completo, no solo actualizar el `CfdiUse`.

---

## 7. Cumplimiento de reglas transversales del proyecto

Esta sección evalúa el DIS-SOL contra las reglas del proyecto que aplican a todos los requisitos (idioma de código, uso de vistas, aplicativos transversales de Envío de Correo y Bitácora General, convención de rutas). Las reglas se aplican **por tipo de aplicativo**, no de forma uniforme.

### 7.1 — Uso de vistas de BD (evitar cuando las entidades ya están en el DBContext)

**Regla del proyecto:** *"No usar vistas a menos que las entidades que use la vista no se encuentren dentro del DBContext de Finanzas. Ej.: si Finanzas necesita `Cliente` y no existe la entidad `Cliente` en el DBContext, entonces vale la pena usar una vista."*

**Lo que el DIS-SOL propone:** ampliar `vfccDocumentoFiscalCobro` a v3.0 — agregar las 8 columnas DR del CP y un `LEFT JOIN` a `catFormaPagoSAT` para exponer `FormaPagoClave`/`FormaPagoDescripcion`. La vista se consume desde `MapPreviewAsync` (`db.vfccDocumentoFiscalCobro.Single(...)`).

**Análisis contra la regla:** las entidades que la vista consulta —`fccDocumentoFiscalCobro`, `catFormaPagoSAT`, `CFDIGenerada`— **ya están scaffoldeadas en `FinanzasContext`** según el propio DIS-SOL:
- `catFormaPagoSAT`: sección "Base de Datos" dice explícitamente *"ya existe desde RE-024, ya incluye `--table catFormaPagoSAT` en el comando de Scaffold de Finanzas.Infrastructure"*.
- `fccDocumentoFiscalCobro`: sección "Impacto en código existente" agrega el DbSet a `FinanzasContext`.
- `CFDIGenerada`: mismo caso, agregado como DbSet.

**Problema:** el ALTER VIEW `vfccDocumentoFiscalCobro` v3.0 **no aporta valor** porque todas las entidades ya son consultables desde el DbContext de Finanzas con LINQ. La vista está siendo usada como conveniencia (menos joins en código), no como necesidad (entidades inaccesibles). Contradice la regla del proyecto.

**Acción requerida:**
1. **Eliminar el ALTER VIEW `vfccDocumentoFiscalCobro` v3.0** de la sección "Base de Datos".
2. Reescribir `MapPreviewAsync` y cualquier otro consumo para usar LINQ directamente contra los DbSets de `FinanzasContext`, por ejemplo:
   ```csharp
   var linea = await (
       from l in db.fccDocumentoFiscalCobro
       join fp in db.catFormaPagoSAT on l.IdCatFormaPagoSAT equals fp.IdCatFormaPagoSAT into fpj
       from fp in fpj.DefaultIfEmpty()
       where l.IdFCCDocumentoFiscalCobro == idFCCDocumentoFiscalCobro
       select new LineaCpDto { /* ... */ }
   ).SingleAsync();
   ```
3. Verificar también que `vfccDocumentoFiscalCobro` no se referencia desde otro consumo pendiente en Finanzas — si toda su carga cae dentro del DBContext, la vista puede quedarse en v2.0 (sin el incremento de RE-030), y RE-030 no toca vistas.
4. Si RE-029 o RE-028 dependen de la vista (por consumo desde ProquifaDotNet legacy, .NET Framework 4.8, donde la vista sí aporta), documentar esa dependencia explícita en RE-030 para justificar por qué existe pero **no se amplía** desde este requisito.

### 7.2 — Idioma de la codificación (nueva solución = 100% inglés)

**Reglas del proyecto:**
- Base de datos **ProquifaDotNet** (existente): español (tablas, vistas, SPs, funciones).
- Bases de datos **nuevas**: estructura en inglés.
- Codificación en **PQF Catálogos**: español.
- Codificación en **PQF Logística** — actualización de proceso: español.
- Codificación en **PQF Logística** — nuevo endpoint / nuevo proceso: modelo y controller en inglés.
- **Nueva solución**: inglés en toda la solución (clases, DTOs, modelos, procesos, métodos, funciones **y comentarios**).

**Clasificación de aplicativos tocados por RE-030:**

| Aplicativo | Naturaleza | Idioma esperado |
|---|---|---|
| Base de datos `ProquifaDotNet` (ALTER TABLE `fccDocumentoFiscalCobro`, INSERT en catálogos, INSERT en `VariableConfiguracion` — tabla existente, **no** CREATE TABLE `ValorConfiguracion`) | BD existente | Español ✓ |
| Base de datos `DocumentBuilder` (INSERT en `DocumentTemplate`) | BD existente | Español (aunque el sistema mismo es inglés — se respeta el esquema histórico) |
| `ProquifaDotNet.Finanzas` | **Solución nueva** (creada en RE-016) | **Inglés total, incluyendo comentarios** |
| `ProquifaDotNet.Timbrado` | **Solución nueva** (creada en RE-018) | **Inglés total, incluyendo comentarios** |
| DocumentBuilder API | Aplicativo existente ampliado | Inglés (ya lo es) ✓ |

**Cumplimiento observado en el DIS-SOL:**

| Área                                                                                       | Estado          | Detalle                                                                                                                                                                |
| ------------------------------------------------------------------------------------------ | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Nombres de clases                                                                          | ✓ Inglés        | `PaymentComplementCalculationService`, `GeneratePaymentComplementService`, `PersistPaymentComplementPdfService`, `PaymentComplementPdfMappingService`                  |
| Nombres de métodos                                                                         | ✓ Inglés        | `CalculateInstallmentNumber`, `MapAsync`, `GeneratePdfAsync`, `UploadFilesAsync`, `AssignFolioAndPersistCfdiAsync`                                                     |
| DTOs                                                                                       | ✓ Inglés        | `StampPaymentComplementRequestDto`, `PaymentComplementModel`, `IssuerCp`, `ReceiverCp`                                                                                 |
| Namespaces                                                                                 | ✓ Inglés        | `Proquifa.Finanzas.Application.Services.PaymentComplement`                                                                                                             |
| Rutas de endpoints                                                                         | ✓ Inglés        | `POST /api/v1/paymentComplement/report`, `POST /api/v1/paymentComplement/stamp`                                                                                        |
| Excepción tipada                                                                           | ✓ Inglés        | `PaymentComplementStampException`                                                                                                                                      |
| Enum `FaseGeneracionCp` (**nueva solución**)                                               | ✗ **Español (nombre)** | Renombrar a `StampingPhase` — los valores del enum pueden quedar en español: `Timbrado`, `FolioAsignado`, `PdfGenerado`, `ArchivosSubidos`. Ruta actual: `Proquifa.Finanzas.Domain/Enums/FaseGeneracionCp.cs` |
| Método `MarcarFalloDefinitivoAsync` (Worker)                                               | ✗ **Español**   | Debería ser `MarkDefinitiveFailureAsync`                                                                                                                               |
| Propiedad `Escenario` en `PaymentComplementRetryMessage`                                   | ✗ **Español**   | Debería ser `Scenario` (valores `"B"` y `"D"` OK como literales)                                                                                                       |
| Nombres de eventos de Bitácora: `GenerarComplementoPago`, `ComplementoPagoFalloDefinitivo` | ✗ **Español**   | Deberían ser `GeneratePaymentComplement`, `PaymentComplementDefinitiveFailure`                                                                                         |
| Nombre de cola RabbitMQ `paymentComplementQueue`                                           | ✓ Inglés        |                                                                                                                                                                        |
| Nombre de archivo del Worker `PaymentComplementRetryWorker.cs`                             | ✓ Inglés        |                                                                                                                                                                        |
| Comentarios inline (`//` y `///`)                                                          | ✗ **Mezclados** | XML docs (`/// <summary>`) en inglés; comentarios `//` mayoritariamente en español. Ejemplos de comentarios que deben traducirse:                                      |

**Comentarios en español detectados en `Proquifa.Finanzas.Application.Services.PaymentComplement.*` que deben pasarse a inglés:**
- `// Snapshot fiscal + resultado del PAC — necesarios para el paso de folio, aún no persistidos en BD`
- `// dentro de la misma transacción — folio y CFDI se confirman juntos o ninguno`
- `// revierte folio + INSERT CFDIGenerada + cambio de Fase juntos — nada queda a medias`
- `// ExecuteAttemptAsync lo captura, persiste NumeroReintento/FechaUltimoReintento y decide el reintento`
- `// primer reintento real, NumeroReintento=1 — ver Worker`
- `// ejecución original — llamada directa, nunca pasa por la cola`
- `// política R16: 1 solo pago por FAA`
- `// UPDLOCK+ROWLOCK dentro de la misma transacción del timbrado, para que dos cobros concurrentes...`
- `// el Worker decide si publica el siguiente (ver Worker)`
- `// revierte folio + INSERT CFDIGenerada + cambio de Fase juntos`
- Muchos otros — hay que hacer una pasada completa.

**Acción requerida:**
1. Renombrar `FaseGeneracionCp` → `StampingPhase` (solo el nombre de la clase — los valores del enum pueden quedar en español: `Timbrado`, `FolioAsignado`, `PdfGenerado`, `ArchivosSubidos`); `MarcarFalloDefinitivoAsync` → `MarkDefinitiveFailureAsync`; `Escenario` → `Scenario`.
2. Renombrar eventos de Bitácora General a inglés: `GenerarComplementoPago` → `GeneratePaymentComplement`, `ComplementoPagoFalloDefinitivo` → `PaymentComplementDefinitiveFailure`.
3. Hacer pasada completa de todos los comentarios `//` en los archivos nuevos bajo `Proquifa.Finanzas.*` y `Proquifa.Timbrado.*` y traducirlos a inglés.
4. Verificar en tiempo de code review que no se filtren comentarios en español antes del merge.

**Nota sobre nombres de columnas de BD:** los nombres de columnas nuevas en `fccDocumentoFiscalCobro` (`FechaPagoCP`, `IdCatFormaPagoSAT`, `TipoCambioP_CP`, `NumParcialidad`, `ImpSaldoAnt`, `ImpPagado`, `ImpSaldoInsoluto`, `EquivalenciaDR`, `IdPeticionCP`, `NumeroReintento`, `FechaUltimoReintento`) **sí deben quedar en español** porque `ProquifaDotNet` es BD existente. En el DBContext de Finanzas, las **propiedades** que las mapean pueden llamarse igual (para minimizar fricción con SQL crudo y comparaciones) o tener alias en inglés vía `[Column("FechaPagoCP")]` — decisión de estilo de scaffold, no bloqueante. Recomendación: alias en inglés en la entidad de dominio de Finanzas + `[Column("...")]` con el nombre real, para mantener el dominio en inglés como pide la regla de "nueva solución".

### 7.3 — Envío de correos a través del Aplicativo Nuevo dedicado

**Regla del proyecto:** *"Los envíos de correo se usan por el Aplicativo Nuevo Envío de Correo"* — no se implementa lógica de envío acoplada en cada requisito.

**Lo que el DIS-SOL propone:** delega el envío al modal del Paso 3 de RE-028 (Regla 14 / Criterio J1 del requisito RE-030), y menciona una sola vez `ProquifaDotNet.EnvioCorreo (Aplicativo Nuevo)` en el pendiente P4 sobre la plantilla del correo.

**Problema:** el DIS-SOL **no explicita que el envío del correo del CP debe pasar por el Aplicativo Nuevo Envío de Correo** — solo lo menciona tangencialmente. Al leer el flujo, queda ambigüedad sobre si RE-028 arma y envía el correo por su cuenta o lo delega al aplicativo transversal. Si RE-028 tiene su propia lógica de envío hoy, hereda el mismo problema.

**Acción requerida:**
1. Documentar explícitamente en el diseño de RE-030 que **el envío del correo del CP (Criterios J1, J2, J3) se ejecuta a través del Aplicativo Nuevo Envío de Correo**, no por lógica local de RE-028 ni de RE-030.
2. Definir el contrato de invocación al Aplicativo Nuevo Envío de Correo desde RE-028 (o desde el orquestador que corresponda): destinatarios (`Para`, `CC`), adjuntos (PDF + XML del CP + Confirmación de Pedido), template ID (bloqueado por P4), variables de contexto.
3. Cruzar con el mantenedor de RE-028 para confirmar que su modal de envío ya invoca el Aplicativo Nuevo Envío de Correo, o levantar la ampliación correspondiente.

### 7.4 — Bitácora General debe registrar todos los eventos relevantes

**Regla del proyecto:** *"Procesos como guardar una factura, validar un cobro, guardar una proforma, etc. deben llamar a Bitácora General — Aplicativo Nuevo."*

**Lo que el DIS-SOL propone:** identifica 2 puntos de registro:
- `FinalizeLineAsync` — registrar cambio de `EstadoLinea` → `GENERADO` (marcado como TODO/P17, pendiente por definición del mecanismo).
- `PaymentComplementRetryWorker` cuando se agotan reintentos — registrar `EstadoLinea` → `FALLIDO_DEFINITIVO` (mismo TODO/P17).

**Problema:** los 2 puntos identificados son insuficientes. Según la regla del proyecto, **cada evento de negocio relevante** debe registrarse. En el flujo de RE-030 hay al menos 5 eventos que ameritan Bitácora:

| # | Evento | Fase | Registro sugerido |
|---|---|---|---|
| 1 | Timbrado exitoso del CP ante el PAC | `TIMBRADO` | `PaymentComplementStamped` (UUID, IdPeticionCP) |
| 2 | Folio asignado + `CFDIGenerada` insertado | `FOLIO_ASIGNADO` | `PaymentComplementFolioAssigned` (Folio, Serie) |
| 3 | PDF generado por DocumentBuilder | `PDF_GENERADO` | `PaymentComplementPdfGenerated` |
| 4 | PDF+XML subidos a MinIO | `ARCHIVOS_SUBIDOS` | `PaymentComplementFilesUploaded` (rutas MinIO) |
| 5 | Línea finalizada (`EstadoLinea='GENERADO'`) | Actual del DIS-SOL | `PaymentComplementGenerated` |
| 6 | Fallo definitivo tras agotar reintentos | Actual del DIS-SOL | `PaymentComplementDefinitiveFailure` |
| 7 | Cada intento de reintento fallido (opcional, para auditoría) | `catch` del `ExecuteAttemptAsync` | `PaymentComplementAttemptFailed` (NumeroReintento) |

**Acción requerida:**
1. Ampliar la lista de eventos que se registran en Bitácora General para cubrir los 5-7 puntos anteriores, no solo el finalizado y el fallo definitivo.
2. En el pendiente P17, documentar que la granularidad esperada es **por transición de fase**, no solo por resultado final — así el diseño no queda ambiguo cuando el mecanismo se defina.
3. Confirmar con el dueño del Aplicativo Nuevo Bitácora General si la granularidad "por transición de fase" cabe en el modelo, o si conviene registrar solo el resultado final + fallo.

### 7.5 — Convención de rutas de endpoints — cumplimiento

**Regla del proyecto:** `api/v1/{resource}/{id}/{subresource}` con recurso singular en inglés y CRUD por método HTTP.

**Endpoints nuevos propuestos en el DIS-SOL:**

| Endpoint                          | Método | Cumple                                               |
| --------------------------------- | ------ | ---------------------------------------------------- |
| `api/v1/paymentComplement/report` | POST   | ✓ (singular, inglés, `api/v1/`, subrecurso `report`) |
| `api/v1/paymentComplement/stamp`  | POST   | ✓ (singular, inglés, `api/v1/`, subrecurso `stamp`)  |

**Endpoints referenciados de requisitos previos que NO cumplen la convención:**
- `POST /api/timbrado/timbrar-faa` (RE-019) — falta `v1/`, verbo en español (`timbrar`), no sigue `{resource}/{subresource}`.
- Formato correcto habría sido: `POST /api/v1/advanceInvoice/stamp` o `POST /api/v1/invoice/stamp` con discriminador de tipo en el body.

**Análisis:** los endpoints nuevos de RE-030 **cumplen la convención**. La observación aplica solo a los referenciados de RE-019 y no es responsabilidad de RE-030 corregirlos. Vale la pena, sin embargo, dejar constancia de la deuda técnica para revisarla cuando se toque RE-019.

**Acción requerida:** ninguna en RE-030. Levantar el punto contra RE-019 en el backlog transversal.

---

## 8. Pendientes ya reconocidos en el DIS-SOL

Estos ya están listados en la sección "Pendientes y Brechas" del DIS-SOL; se enumeran aquí solo para verificación cruzada.

| # DIS-SOL | Pendiente                                           | Estado                                  |
| --------- | --------------------------------------------------- | --------------------------------------- |
| P1        | Hora en `FechaPago` (12:00:00 vs real)              | Provisional 12:00:00 con TODO en código |
| P2        | Formato de Serie "P"                                | Propuesta `P{folio:00000000}`           |
| P3        | Soporte tasas IVA distintas a 16%                   | Solo 16%                                |
| P4        | Plantilla del correo (PMO #31)                      | Bloqueado                               |
| P5        | Política de reintento                               | Diseñado (ver Obs. 4.2)                 |
| P6        | Propagación de `idCliente`                          | Anotado                                 |
| P7        | `ConsumeNextFolioAsync` acepta `serie`              | Cambio de firma pendiente               |
| P8        | `ConversorDivisas` cross-framework                  | Reimplementación en Finanzas            |
| P9        | Origen de datos `DigitalSealCp`                     | No mapeado                              |
| P11       | Reintento de NC (transversal)                       | Coordinar                               |
| P13       | `IMinioStorageService` vs legacy `SubirArchivo`     | Confirmar                               |
| P14       | Endpoint de conversión desde Logística              | Confirmar                               |
| P15       | Tabla dedicada para NumParcialidad vs COUNT+UPDLOCK | Evaluar                                 |
| P16       | `ConsumeAsync` no existe en template RabbitMQ base  | Bloqueado                               |
| P17       | Bitácora General (Regla 8)                          | Bloqueado                               |

**Pendientes nuevos que esta revisión propone añadir:**

| # | Descripción | Origen |
|---|---|---|
| P18 | Analista de Cuentas por Cobrar en CC del correo | Obs. 3.3 (Criterio J2) |
| P19a | Definir tabla `fccFolioDocumentoFiscalCobro` (1 fila por empresa+serie), INSERT inicial GOL/MUN/PRO/PQF con Serie='P', UltimoFolio=0; reemplazar `EmpresaFolio` en el DIS-SOL | Obs. 3.2.1 |
| P19b | Interpretación del Criterio B2 — confirmar si Serie/Folio deben ir dentro del XML certificado o solo en `CFDIGenerada` como atributo de negocio | Obs. 3.2.2 |
| P20 | Confirmación del PAC (TurboPac vs SAP) | Obs. 3.4 |
| P21 | Alcance de CFDI 3.3 en RE-030 | Obs. 4.1 |
| P22 | `ValorConfiguracion` ya existe como `VariableConfiguracion` — eliminar CREATE TABLE, usar tabla existente e insertar `MAX_REINTENTOS_TIMBRADO=5` con MERGE | Obs. 4.3 |
| P23 | Eliminar `vfccDocumentoFiscalCobro` v3.0 y consumir DbSets directos con LINQ | Obs. 7.1 (regla de vistas) |
| P24 | Renombrar a inglés: clase `FaseGeneracionCp` → `StampingPhase` (valores del enum pueden quedar en español), `MarcarFalloDefinitivoAsync` → `MarkDefinitiveFailureAsync`, `Escenario` → `Scenario`, eventos de Bitácora, y comentarios `//` de todos los archivos nuevos | Obs. 7.2 (nueva solución = 100% inglés) |
| P25 | Documentar explícitamente que el envío del correo del CP pasa por el Aplicativo Nuevo Envío de Correo | Obs. 7.3 |
| P26 | Ampliar Bitácora General a las 5-7 transiciones de fase, no solo al finalizado y al fallo definitivo | Obs. 7.4 |

---

## 9. Recomendaciones finales

En orden de prioridad para aprobar el DIS-SOL:

1. **Cerrar Obs. 3.1** (política 1 CP por factura) — corregir DTO, ejemplos y pruebas, o reformular la justificación.
2. **Cerrar Obs. 3.2** (Serie/Folio en XML certificado) — decisión de negocio + fisco, no técnica.
3. **Cerrar Obs. 3.4** (PAC TurboPac vs SAP) — validación con arquitectura, cambio de una línea.
4. **Cerrar Obs. 3.3** (analista de Cobranza en CC) — decidir si se resuelve aquí o se coordina con RE-028.
5. **Cerrar Obs. 4.1 y 4.2** (CFDI 3.3, reintento transversal) — decidir si se mantienen ampliados o se recortan al alcance del requisito.
6. **Cerrar Obs. 7.1** (vista `vfccDocumentoFiscalCobro` v3.0) — eliminar el ALTER VIEW y consumir DbSets directamente con LINQ; alinea con la regla del proyecto.
7. **Cerrar Obs. 7.2** (idioma de código) — pasada completa de traducción a inglés en la solución nueva antes del primer merge.
8. **Cerrar Obs. 7.3 y 7.4** (Envío de Correo, Bitácora General) — documentar la invocación al aplicativo transversal y ampliar los puntos de registro.
9. Aplicar Obs. 5.1 (parametrizar tasa IVA) y Obs. 5.2 (Concepto en el modelo) — no bloqueantes pero baratas y evitan refactor.
10. Observaciones 5.3 a 6.5 y 7.5 pueden abordarse en revisión final antes de merge, no bloquean aprobación del diseño.

**Conclusión:** el DIS-SOL es un documento sólido y completo, con decisiones arquitectónicas bien argumentadas que exceden en varios puntos lo que el requisito exige. La brecha real no es técnica sino de **alineamiento con el requisito literal** (Observaciones 3.1, 3.2), de **alcance** (Observaciones 4.1, 4.2) y de **cumplimiento de reglas transversales del proyecto** (Observaciones 7.1 a 7.4 — vistas evitables, idioma de código, aplicativos transversales de Envío de Correo y Bitácora General). Una vez resueltas, el diseño está listo para codificarse.
