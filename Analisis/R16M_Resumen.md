# R16M — Resumen Ejecutivo de Requisitos
### Proyecto R16 "Tramitar Pedidos sin Crédito" — PQF2 (ProQuiFaNet 2)
> Formato ADP-FOR-13 v1.0 · Administrador: Irma Andrade Aguado · 35 requisitos R16A-RE-FU-001 a R16A-RE-FU-035

---

## 1. CONTEXTO GLOBAL

### Empresas Emisoras

| País   | Empresa                                      | RFC / RUC       |
| ------ | -------------------------------------------- | --------------- |
| México | Golocaer S.A. de C.V.                        | —               |
| México | Mungen S.A. de C.V.                          | —               |
| México | Proquifa S.A. de C.V.                        | PRO970821ML3    |
| México | Proveedora Quimico Farmaceutica S.A. de C.V. | —               |
| Perú   | Golocaer S.A.C. (única)                      | RUC 20612772941 |

### Marco Fiscal

| Aspecto                  | México (SAT / CFDI 4.0)                                                 | Perú (SUNAT / CPE UBL 2.1)                                                   |
| ------------------------ | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Impuesto                 | IVA 16%                                                                 | IGV 18%                                                                      |
| ID Fiscal                | RFC                                                                     | RUC                                                                          |
| PAC / Emisor electrónico | TurboPac (Quadrum, RFC QSO100827UB0)                                    | Pendiente — 4 modalidades SUNAT (**brecha mayor bloqueante**)                |
| Folio fiscal             | UUID asignado por SAT                                                   | Serie-Correlativo (sin UUID)                                                 |
| Pago                     | MetodoPago PPD/PUE                                                      | Condición de Pago Contado/Crédito                                            |
| Operación                | UsoCFDI (catálogo SAT)                                                  | Tipo de Operación catálogo 51 SUNAT                                          |
| Factura                  | CFDI Ingreso (tipo I)                                                   | CPE tipo 01                                                                  |
| NC                       | CFDI Egreso (tipo E), TipoRelacion=01, UsoCFDI=G02, MetodoPago=PUE fijo | CPE tipo 07, motivo catálogo 09, referencia por serie-correlativo            |
| Complemento de Pago      | CFDI tipo P (Pagos 2.0)                                                 | **NO EXISTE en Perú**                                                        |
| Factura Anticipo         | CFDI Ingreso, TipoRelacion=07                                           | **NO APLICA en Perú** (DUA es trámite separado)                              |
| Cancelación              | CFDI cancelación + c_MotivoCancelacion                                  | NC motivo 01 (10 días háb.) o Comunicación de Baja (7 días, excepcional)     |
| Representación impresa   | Sello SAT + cadena complementaria + QR SAT                              | Firma digital emisor + hash + QR SUNAT + leyenda "Representación impresa..." |
| Conservación             | 5 años mínimo (Art. 30 CFF)                                             | 5 años desde 1 ene. año siguiente (R.S. 117-2017/SUNAT)                      |
| Legacy                   | Pedidos MX SÍ se transfieren a Legacy                                   | Pedidos PE **NO** van a Legacy                                               |

### Arquitectura Transversal

| Componente | Detalle |
|---|---|
| Foliador proformas | PRF-MMDDAA-Consecutivo (contador global único para todo el grupo) |
| Foliador facturas | Consecutivo independiente por empresa emisora (varchar 6) |
| Foliador NC MX | Consecutivo por empresa, serie "P2" propuesta (**pendiente validar**) |
| Foliador CdP MX | Consecutivo por empresa, serie "P" propuesta (**pendiente validar**) |
| Foliador NC PE | Serie prefijo-F + correlativo 8 dígitos, consecutivo por Golocaer S.A.C. |
| Almacenamiento PDF | MinIO |
| Mailbot | Clasificador automático: Cotización / Pedido / **Cobro** (NUEVO R16) / Otros |
| Decimales | 6–8 internos; máx 4 al usuario; reglas SAT (PMO #55) |
| Roles | ESAC · Gestor de Cobranza / Analista CxC (**denominación canónica pendiente**) · Coordinador Tesorería · Gerente Tesorería · Regulatorios · Cobrador |

**Identidad visual MX:** logo + paleta por empresa emisora (Golocaer naranja · Mungen verde · Proquifa cyan · Proveedora cyan) + iconografía certificaciones (ISO, NEEC, edQM, FELUM, USP, Microbiologics, APACOR, CHATA, Pharmaffiliates, Amex — **vigencia pendiente confirmar**)

**Identidad visual PE:** logo + paleta Golocaer S.A.C. — consistente en todos los documentos PE.

### 5 Brechas Perú (referenciadas desde R16A-RE-FU-005)

| # | Brecha | Estado |
|---|---|---|
| 1 | Modelo bancario Golocaer S.A.C. (CCI 20 dígitos) | Pendiente |
| 2 | Integración OSE/SUNAT para timbrado | **Pendiente — bloqueante** |
| 3 | Datos fiscales SUNAT por producto (código SUNAT, unidad medida, afectación IGV) | Pendiente |
| 4 | Datos legales Golocaer S.A.C. Perú (RS, domicilio fiscal, régimen tributario) | Pendiente |
| 5 | Modalidad emisión electrónica SUNAT (SEE-SOL / SEE del Contribuyente / SEE-OSE / Facturador SUNAT; depende de si Golocaer es gran contribuyente) | **Pendiente — bloqueante** |

---

## 2. MÓDULOS Y REQUISITOS

### 2.1 Catálogo de Clientes

**R16A-RE-FU-001** (aprox.)

- Ampliación de ficha cliente: datos fiscales **MX** (RFC, Régimen Fiscal SAT, Uso CFDI default, Método de Pago preferido) y **PE** (RUC, Régimen Tributario SUNAT, Condición de Pago, Tipo de Operación)
- Modelo bancario: **CLABE 18 dígitos** (MX) · **CCI 20 dígitos** (PE)
- Campo **Cobrador**: asignación de cartera; filtro aplicado en NC, Validar Cobro, FxA, Buzón de Cobros
- Datos de contacto del cliente (heredados en correos de envío de documentos fiscales)
- **Pendiente**: datos legales completos Golocaer S.A.C. Perú (Brecha 4)

---

### 2.2 Cotizar lo Cotizable

**R16A-RE-FU-002** (aprox.)

- Módulo existente con ajustes menores para R16
- Genera **Proforma** (cotización formal) con folio **PRF-MMDDAA-Consecutivo** (contador global único para todo el grupo)
- La proforma es el documento que inicia el flujo de Tramitar Pedido
- Aplica a clientes México y Perú

---

### 2.3 Pretramitar Pedido

**R16A-RE-FU-003** (aprox.)

- Revisión y validación documental previa a la tramitación
- **Sustancias Controladas** (Mundial / Nacional / Origen): validación documental adicional obligatoria (permisos, licencias, autorizaciones según clasificación)
- Bloqueo de tramitación si documentación incompleta en productos controlados
- Aplica a ambas regiones

---

### 2.4 Tramitar Pedido — 6 Flujos

**R16A-RE-FU-004 a R16A-RE-FU-009** (aprox. — 6 requisitos contiguos)

| Flujo | Descripción |
|---|---|
| Crédito sin controlados sin FxA | MX — sin anticipos, sin productos controlados |
| Crédito con controlados | MX — con validación de sustancias controladas |
| Crédito sin controlados con FxA | MX — con Factura por Adelantado previa |
| Prepago con controlados | MX + PE — Factura Anticipo SAT (MX) / Factura normal (PE) |
| Prepago sin controlados sin FxA | MX + PE — pago → entrega sin factura anticipada |
| Prepago sin controlados con FxA | MX + PE — con Factura por Adelantado previa al cobro |

**Reglas comunes:**
- Pedido **Prepago** = pago primero, entrega después → habilita flujo Validar Cobro
- Pedido **Crédito** = entrega primero → fuera del scope R16 para PE
- **FxA** (Factura por Adelantado): aplica cuando el cliente Prepago requiere factura antes de pagar; genera CFDI Ingreso (MX) / CPE tipo 01 (PE) anticipadamente
- **Controlados MX**: generan **Factura Anticipo** (CFDI Ingreso, TipoRelacion=07) en lugar de Factura estándar, porque el pago se recibe antes de la entrega física; **pendiente confirmar uso de TipoRelacion=07 SAT**
- **Controlados PE**: la factura peruana no depende de la DUA (trámite separado) → no requiere Factura Anticipo; se genera Factura electrónica normal
- **Cancelación manual con factura emitida** (R16A-RE-FU-023): detona cancelación fiscal; en PE se realizaría vía NC motivo 01 SUNAT
- **NEEC** (Nuevo Esquema de Empresas Certificadas): aplica a exportaciones MX

---

### 2.5 Factura por Adelantado — NUEVO módulo

#### 2.5.1 Funciones wizard MX

**R16A-RE-FU-0XX** (aprox. ~010–012)

- Wizard para emitir CFDI Ingreso antes de recibir el cobro (pedidos Prepago con FxA)
- El UUID de esta FxA se usa después como `DoctoRelacionado` en el Complemento de Pago cuando MetodoPago=PPD (en Validar Cobro Paso 3)
- Timbrado vía TurboPac; previsualización PDF antes de timbrar; feedback visual (progreso / éxito / error)
- Envío automático al cliente (PDF + XML); modal editable: Para (contacto pedido), CC (ESAC), Asunto generado por sistema (no editable), Adjuntos (no editables), Notas extras opcionales
- Manejo de errores PAC consistente con VC y NC
- Conservación XML 5 años (Art. 30 CFF)
- Identidad visual por empresa emisora MX

#### 2.5.2 Funciones wizard PE — R16A-RE-FU-020

- Estructura funcional idéntica a MX; diferencias: CPE tipo 01, UBL 2.1, IGV 18%, RUC, Tipo de Operación catálogo 51, Condición de Pago; sin UUID/UsoCFDI/MetodoPago
- **Análisis DUA**: la factura peruana de venta no depende del dato aduanero (la DUA es trámite separado); la Ley IGV permite facturar por el monto cobrado anticipadamente → no se requiere Factura Anticipo en PE
- Emisión CPE tipo 01 ante SUNAT (modalidad pendiente — **brecha mayor bloqueante**)
- Empresa emisora única: Golocaer S.A.C.
- **Pendientes**: modalidad emisión electrónica SUNAT (Brecha 5) · catálogo 51 Tipo de Operación parámetros · datos legales Golocaer S.A.C. (Brecha 4) · asesor fiscal peruano

#### 2.5.3 PDF Factura México — R16A-RE-FU-021

Estructura completa del PDF representativo:

- **Encabezado**: logo + paleta empresa emisora
- **Emisor**: RFC, Razón Social, CP/Lugar Expedición, Régimen Fiscal
- **Receptor**: RFC, RS, Domicilio Fiscal, Régimen Fiscal, Uso CFDI
- **Comprobante**: Versión 4.0, TipoDeComprobante, Serie, Folio, Fecha, Moneda, TC (cuando aplique), MetodoPago, FormaPago
- **Conceptos**: tabla por partida — ClaveProdServ, ClaveUnidad, Cantidad, Descripción, ValorUnitario, Importe, IVA por línea
- **Totales**: Subtotal, IVA, Total, Total en letra
- **Trazabilidad SAT**: Núm. serie cert. SAT, Núm. serie CSD emisor, Sello Digital CFDI, Sello Digital SAT, Cadena Original Complemento de Certificación Digital del SAT
- **QR SAT**: `https://verificacfdi.facturaelectronica.sat.gob.mx/default.aspx?id=UUID&re=RFC_EMISOR&rr=RFC_RECEPTOR&tt=TOTAL&fe=últimos8_del_sello`
- **Pie**: iconografía certificaciones químico-farmacéutico

Aplica a las cuatro empresas emisoras MX con identidad visual diferenciada por empresa.

#### 2.5.4 PDF Factura Perú — R16A-RE-FU-022

- **Encabezado**: logo Golocaer S.A.C.
- **Emisor**: RUC, RS, domicilio fiscal
- **Receptor**: RUC o documento, RS
- **Comprobante**: tipo 01, serie-correlativo, fecha, moneda, TC (cuando aplique), Condición de Pago, Tipo de Operación catálogo 51
- **Conceptos**: código SUNAT, descripción, cantidad, unidad SUNAT, precio unitario, afectación IGV, valor de venta, IGV por línea
- **Totales**: Valor Venta, IGV, Importe Total, Total en letra (nomenclatura SUNAT)
- **QR SUNAT** + leyenda "Representación impresa de la Factura Electrónica, consúltela en..."
- **Sin**: UUID, sello digital SAT, cadena complementaria, UsoCFDI, MetodoPago PPD/PUE
- El respaldo legal es la firma digital del emisor y el valor resumen/hash
- **Pendientes**: datos legales Golocaer S.A.C. PE (Brecha 4) · validación asesor fiscal peruano · maquetas PE

---

### 2.6 Validar Cobro — Wizard 3 Pasos

#### 2.6.1 Paso 1 — Captura del Cobro

**R16A-RE-FU-024** (MX) · **R16A-RE-FU-025** (PE)

- Primer paso: el operador (Gestor de Cobranza) registra el cobro recibido del cliente
- **Cabecera del cliente**: logo, Alias, etiquetas, RFC/RUC, razón social legal, moneda de facturación
- **Barra de pasos**: 1 (activo) → 2 → 3
- **Folio del cobro**: COB-MMDDAA-Consecutivo
- **Datos del cobro**: fecha, monto, moneda, banco de destino, referencia bancaria
- Banco destino: CLABE 18 dígitos (MX) · CCI 20 dígitos — Golocaer S.A.C. (PE)
- Soporte multi-divisa: MXN/USD/EUR (MX) · PEN + divisas extranjeras (PE)
- Tipo de cambio (TC): fuente DOF (MX) · **fuente oficial PE pendiente de definir**
- Marcado de inconsistencias del cobro (modal Paso 1): tipos intrínsecos (cobro en moneda incorrecta, referencia bancaria no identificada, banco incorrecto, etc.); catálogo completo pendiente (Tesorería)
- Auto-guardado automático del estado
- Navegación: Cancelar o Continuar al Paso 2

**PE específico:** empresa emisora única Golocaer S.A.C.; sin mezcla de empresas.

---

#### 2.6.2 Paso 2 — Asociación

**R16A-RE-FU-026** (MX) · **R16A-RE-FU-027** (PE)

- El operador asocia manualmente cobros ↔ proformas/facturas (relación **N:N**)
- **Listado de cobros** del Paso 1 con selección múltiple (checkboxes); cobros con saldo a favor marcados con identificador visual
- **Listado de Proformas y Facturas pendientes** del cliente (mezcladas, sin filtros adicionales por tipo/fecha)
- **Asociación manual N:N**: el sistema aplica el monto del cobro a los documentos en el orden de selección del operador; sin campo "Monto a aplicar" editable por línea; si el cobro no alcanza, el último documento queda con saldo pendiente
- Aplicación **OPCIONAL** de Notas de Crédito vigentes del cliente (cero, una o varias por documento); las NCs no seleccionadas siguen vigentes
- **Cálculo dinámico del saldo**: Cobros aplicados + NCs aplicadas – Adeudo total = Saldo resultante

**Escenarios de pago:**

| Escenario | Comportamiento |
|---|---|
| Exacto | Avanza al Paso 3 |
| Sobrepago | Registra excedente como **saldo a favor** en Estado de Cuenta/Auxiliar del cliente; cobro origen marcado con "saldo a favor"; disponible para futuras proformas/facturas; **NO genera documento fiscal adicional**; avanza al Paso 3 |
| Pago de menos ≤ tolerancia | Cierra con saldo pendiente en Estado de Cuenta; avanza al Paso 3 |
| Pago de menos > tolerancia (sin inconsistencia) | Bloquea el avance; operador debe marcar inconsistencia o dejar pendiente para próximo cobro |

- **Tolerancia MX**: 100 MXN (Política Interna PROQUIFA) · **Tolerancia PE: monto/política pendiente de definir**
- **Marcado de inconsistencias Paso 2** (modal): Tipo de Inconsistencia (combo: tipos Paso 1 + Pago Incompleto Vencido + Pago Insuficiente) + Comentario adicional opcional; catálogo completo pendiente (Tesorería)
  - **Pago Incompleto Vencido**: habilita marcar pedido como "Pendiente de cancelación por falta de pago" (NO ejecuta cancelación fiscal ni devolución; solo notifica para gestión externa)
  - **Pago Insuficiente**: registra inconsistencia, mantiene asociación pendiente para próximo cobro
- **Multi-divisa**: cuando moneda del cobro ≠ moneda de documentos, el sistema convierte a moneda del cobro usando TC del Paso 1; muestra TC y fórmula vía tooltip; consolida totales en moneda del cobro
- Auto-guardado; asociaciones editables mientras el operador esté en Paso 2; fijas al avanzar al Paso 3
- Navegación: **Regresar** (Paso 1, auto-guardado activo) · **Continuar** (Paso 3, solo si escenario válido)

**MX específico:**
- Mezcla de hasta 4 empresas emisoras en el listado
- Asociación cobro↔factura PPD **genera Complemento de Pago** en Paso 3 (efecto fiscal)
- NCs referenciadas por UUID timbrado SAT en `CFDIRelacionados` del XML

**PE específico:**
- Empresa emisora única: Golocaer S.A.C.
- La asociación cobro↔documento **NO tiene efecto fiscal**: es solo registro operativo interno de conciliación; la factura peruana ya se emitió completa con IGV (no hay Complemento de Pago en PE)
- **Pendientes**: tolerancia pago de menos PE · fuente TC oficial PE · catálogo inconsistencias PE · mecánica referencia NC peruana (catálogo 09) en asociación · mecanismo transferencia estado "Pendiente cancelación" · maquetas PE no disponibles

---

#### 2.6.3 Paso 3 — Facturación y Envío

**R16A-RE-FU-028** (MX) · **R16A-RE-FU-029** (PE)

- El operador previsualiza, timbra y envía el documento fiscal de cada línea **individualmente** (sin acciones masivas)
- Una línea por cada documento de la asociación que requiera emisión de documento fiscal
- **Estados por línea**: Pendiente → Generado (tras timbrado exitoso) → Enviado (tras envío al cliente)
- **Flujo por línea**: Previsualizar PDF → Timbrar → Enviar (secuencial; no se envía sin timbrar)
- **Modal Previsualizar**: PDF representativo antes del timbrado (sin efecto fiscal aún)
- **Modal éxito timbrado**: muestra UUID (MX) o Serie-Correlativo (PE)
- **Modal Enviar**: Para (contacto del pedido, editable) + CC (ESAC asignado, editable) + Asunto generado por sistema (no editable) + Adjuntos (PDF+XML de cada CFDI + Confirmación de Pedido, no editables) + Notas extras opcionales
- **Inmutabilidad post-timbrado**: una vez timbrado no se re-timbra ni se edita; corrección solo vía módulo Notas de Crédito
- **Navegación atrás**: SÍ se puede regresar a Paso 1/2 ANTES de timbrar; NO después del timbrado
- **Persistencia**: si el usuario sale del wizard, al volver se redirige al Paso 3 con el estado preservado hasta cerrar todas las líneas
- **Cierre del wizard**: cuando todas las líneas están en estado Enviado

**MX — lógica condicional del tipo de documento a generar:**

| Documento origen (Paso 2)                       | Documento fiscal MX                                                        |
| ----------------------------------------------- | -------------------------------------------------------------------------- |
| Proforma sin productos controlados              | Factura nueva (CFDI Ingreso)                                               |
| Proforma con productos controlados              | Factura Anticipo (CFDI Ingreso, TipoRelacion=07 — **pendiente confirmar**) |
| Factura existente (FxA previa) + cobro asociado | Complemento de Pago (CFDI Pagos 2.0)                                       |

**MX — edición por línea antes del timbrado:**
- **Uso CFDI**: combo catálogo SAT `c_UsoCFDI` (editable); en líneas de Complemento de Pago: solo lectura (valor heredado de la factura origen)
- **Método de Pago**: PPD/PUE radio button (editable) para líneas con proforma origen; PPD fijo no editable para Complemento de Pago (normativa SAT)

**MX — generación en cascada según Método de Pago:**
- Factura **PUE**: 1 CFDI (sin Complemento de Pago — facturas PUE son fiscalmente autocontenidas)
- Factura **PPD**: 2 CFDIs en cascada — Factura PPD + Complemento de Pago asociado al cobro del wizard; si el CdP falla tras Factura PPD exitosa → notificar al usuario y permitir reintento; la Factura PPD permanece vigente

**MX — inclusión de NCs en CFDIRelacionados:** las NCs aplicadas en Paso 2 se incluyen en el nodo `CFDIRelacionados` del CFDI a timbrar con UUID, monto y TipoRelacion (01 o 07 según el caso)

**MX — acciones automáticas al ENVIAR (solo México):**
1. Establecer **Fecha Estimada de Entrega (FEE)** del pedido
2. **Transferir a Legacy**: pedido + documentos (factura/anticipo/complemento + NCs aplicadas + info cobro)
3. Generar **Confirmación de Pedido** (adjunta en el correo de envío, sin previsualización, sin candado bloqueante)

**PE — simplificaciones vs MX:**
- Tipo de documento **único**: solo Factura electrónica (CPE tipo 01, UBL 2.1); sin Factura Anticipo ni Complemento de Pago
- Cobro contra factura ya emitida (FxA previa): en PE **no se genera documento fiscal**; solo conciliación interna del cobro; **pendiente confirmar qué acciones ofrece el sistema al cliente** en ese escenario
- Campos por línea: **Tipo de Operación** (catálogo 51 SUNAT) + **Condición de Pago** (Contado/Crédito) en lugar de Uso CFDI + Método de Pago
- Modal éxito: muestra Serie + Correlativo (NO UUID)
- Al ENVIAR: FEE + Confirmación de Pedido SÍ aplican; **Transferencia a Legacy NO aplica**
- **Pendientes**: modalidad emisión electrónica SUNAT (**brecha mayor bloqueante**) · parámetros configuración proforma→factura PE · formato asunto y plantilla correo PE · mecánica referencia NC en emisión · maquetas PE no disponibles

---

### 2.7 Complemento de Pago

#### 2.7.1 Complemento de Pago México — R16A-RE-FU-030

- Generado **automáticamente** al confirmar el cobro en Paso 3 de VC contra facturas PPD; no es módulo independiente; único disparador = Paso 3 VC
- **Política R16**: un Complemento de Pago por factura cubierta (si el cobro cubre N facturas PPD → N Complementos de Pago independientes, cada uno con 1 Pago + 1 DoctoRelacionado)
- **Monto del Pago = ImpPagado del DoctoRelacionado** (porción del cobro aplicada a esa factura específica)
- Solo facturas **PPD** generan Complemento de Pago (regla SAT inmutable); facturas PUE no requieren CdP

**Estructura XML (CFDI 4.0 Pagos20 v2.0):**

| Nodo | Campos clave |
|---|---|
| Comprobante (cabecera) | Version=4.0, TipoDeComprobante=P, Exportacion=01, SubTotal=0, Total=0, Moneda=XXX |
| Concepto único fijo | ClaveProdServ=84111506, Cantidad=1, ClaveUnidad=ACT, Descripcion=Pago, ValorUnitario=0, Importe=0, ObjetoImp=01 |
| Emisor | RFC, RS, RegimenFiscal=601 de la empresa emisora del grupo |
| Receptor | RFC, RS, DomicilioFiscalReceptor, RegimenFiscalReceptor; **UsoCFDI=CP01 fijo** |
| Pago | FechaPago, FormaDePagoP (real, no 99), MonedaP, Monto (=ImpPagado del DR), TipoCambioP (si MonedaP ≠ MXN) |
| DoctoRelacionado | IdDocumento (UUID), Serie, Folio, MonedaDR, EquivalenciaDR (=1 si misma moneda), NumParcialidad (consecutivo por factura), ImpSaldoAnt, ImpPagado, ImpSaldoInsoluto (=ImpSaldoAnt – ImpPagado), ObjetoImpDR |
| ImpuestosDR/TrasladosDR | Solo si ObjetoImpDR=02: BaseDR, ImpuestoDR=002, TipoFactorDR=Tasa, TasaOCuotaDR=0.160000, ImporteDR |
| Totales | MontoTotalPagos; TotalTrasladosBaseIVA16/TotalTrasladosImpuestoIVA16 cuando aplique |

- **NCs aplicadas en el cobro NO son DoctoRelacionado** del CdP: las NCs son CFDI Egreso y se relacionan a la factura origen desde la propia NC
- Timbrado vía TurboPac; UUID asignado por SAT; conservación XML 5 años (Art. 30 CFF)
- Envío automático al cliente (PDF + XML) tras timbrado exitoso; CC ESAC + analista CxC

**PDF del Complemento de Pago** (con identidad visual por empresa emisora):  
Datos emisor/receptor · datos comprobante (serie, folio, versión 4.0, UUID, fecha/hora certificación, tipo P-Pago, régimen) · sección Concepto fijo (ClaveProdServ 84111506, Cantidad 1, ClaveUnidad ACT, Descripción "Pago", todo en 0.00, Total "CERO XXX 00/100") · Totales CdP · datos del Pago (fecha, forma, moneda, monto, TC cuando aplique) · datos del DoctoRelacionado (UUID, serie, folio, moneda DR, equivalencia DR, parcialidad, saldo anterior, importe pagado, saldo insoluto) · Impuestos DR cuando aplique · Resumen de traslados a nivel pago cuando aplique · sellos y trazabilidad SAT (núm. serie cert. SAT, núm. serie CSD emisor, Sello Digital SAT, Sello Digital CFDI, Cadena Original Complemento Certificación) · código QR SAT

**Riesgos:** cálculo erróneo de EquivalenciaDR y TipoCambioP en multi-divisa (rechazo SAT / problema acreditación IVA) · convención de hora en FechaPago (legacy usa 12:00:00 fijo — validar con asesor fiscal si se debe usar hora real)

**Pendientes:** convención hora FechaPago (**validar asesor fiscal**) · soporte tasas IVA distintas al 16% (frontera 8%, 0%) · vigencia iconografía certificaciones · mecanismo reintento timbrado PAC (transversal) · plantilla cuerpo correo envío (PMO #31) · validación serie "P" foliador final

#### 2.7.2 Complemento de Pago Perú — R16A-RE-FU-031 *(FILA RESERVADA — sin contenido)*

- SUNAT no tiene equivalente al Complemento de Pago SAT
- ID reservado para preservar correspondencia 1:1 MX/PE del bloque de documentos
- **Pendiente confirmar con el cliente si Perú requiere algún documento análogo**

---

### 2.8 Notas de Crédito

#### 2.8.1 Módulo NC México — R16A-RE-FU-032

- Módulo **independiente** de Validar Cobro (NC alimenta a VC; VC no genera ni cancela NCs); operado por **Tesorería**; aplica a clientes **prepago** en R16
- NCs históricas pre-go-live **NO se importan** desde Legacy

**Pantalla principal:** consulta agrupada por cliente (columnas: #, Cliente, Total NC, Vigentes, Por Aplicar, Monto total, Moneda + agregados al pie); filtros: Cliente, Moneda, Fecha; drill-down al detalle por cliente; visibilidad filtrada por cartera del Cobrador

**Wizard 4 pasos:**
1. **Buscar Factura**: selección obligatoria de cliente + UNA factura vigente prepago (no canceladas SAT, máx 5 años antigüedad); filtros: Fecha, Moneda, buscador por folio/UUID (ignora espacios)
2. **Capturar Datos**: datos de factura origen en solo lectura; motivo desde catálogo SAT (Devolución de mercancía → modalidad por partidas; Descuento o bonificación → modalidad manual); UsoCFDI G02 default; TipoRelación=01 fijo
3. **Confirmar**: resumen completo + previsualización PDF antes de timbrar; advertencia de irreversibilidad; Regresar al Paso 2 sin pérdida de datos
4. **NC Emitida**: misma vista que el detalle de cualquier NC ya generada (UUID, datos PAC, folio, tipo E, versión 4.0, motivo, TipoRelación 01, factura origen con indicación si fue cancelada, receptor, estado SAT, importes, partidas/concepto, nota conservación 5 años Art. 30 CFF; acciones: Descargar XML, Descargar PDF, Reenviar, Volver al listado)

**Dos modalidades de captura:**

| Modalidad | Cuándo | Captura | XML |
|---|---|---|---|
| **Por partidas** | Devolución de mercancía | Tabla heredada de factura origen; Cant.NC editable (0/parcial/total por partida); cálculo automático del monto en tiempo real | Un Concepto por partida con Cant.NC > 0; hereda ClaveProdServ, ClaveUnidad, NoIdentificacion, ValorUnitario, Descripción, impuestos del concepto original; recalcula con Cant.NC |
| **Manual** | Descuento / bonificación | Monto Total NC libre (≤ Total factura origen); Concepto obligatorio (materialidad fiscal); Observaciones opcionales | Único Concepto: ClaveProdServ=84111506, ClaveUnidad=ACT, Cantidad=1, Descripción capturada, ValorUnitario e Importe=Monto NC; **ClaveProdServ/ClaveUnidad pendiente confirmar con cliente** |

**Cancelación condicional de factura origen:**
- Aparece solo si NC es por **totalidad + mismo mes calendario** de emisión
- Combo motivo `c_MotivoCancelacion`; cancelación **siempre total**, nunca parcial
- Política PROQUIFA "mismo mes" (no normativa SAT — razón: optimización IVA mensual; el SAT permite cancelar todo el ejercicio fiscal)
- Al timbrar la NC, el sistema dispara la cancelación SAT de la factura origen vía TurboPac con el motivo seleccionado en Paso 2; si la cancelación falla, la NC timbrada permanece vigente

**Campos fiscales XML fijos:** TipoDeComprobante=E · CfdiRelacionados TipoRelacion=01 + UUID factura origen · UsoCFDI receptor G02 default · **MetodoPago=PUE fijo** (SAT inmutable — las NCs siempre son PUE) · FormaPago heredada de factura origen pagada (típicamente 03 Transferencia) · Moneda heredada (no editable) · TipoCambio del día del timbrado en moneda extranjera

**Timbrado y correo:** TurboPac; envío automático al timbrar (PDF+XML) + opción reenvío posterior; Para = contacto cliente vinculado a factura; CC = ESAC + analista CxC; Asunto: "Nota de Crédito + folio NC + folio factura relacionada" (**plantilla final PMO #31 pendiente**)

**Acoplamiento uni-direccional con VC:** NCs vigentes disponibles automáticamente en Paso 2 del wizard de VC para aplicación a cobros nuevos del mismo cliente; VC no genera ni cancela NCs

**Precisión numérica:** 6–8 decimales internos; máx 4 al usuario; reglas SAT decimales (PMO #55)

**Riesgos:** cálculo erróneo IVA al recalcular partidas (rechazo SAT) · cancelación factura origen fallida tras NC timbrada (inconsistencia: NC vigente + factura también vigente) · dependencia PAC TurboPac (si TurboPac no disponible, las NCs no llegan a VC)

**Pendientes:** FormaPago modalidad manual (**mockup muestra "99" — incorrecto para NC PUE; debe ser 03 u otro real**) · UsoCFDI G02 vs G03 (discrepancia en ejemplo real B-128: cliente usa G03) · ClaveProdServ/ClaveUnidad modalidad manual (84111506/ACT es recomendación SAT; confirmar con PROQUIFA) · PMO #31 plantilla correo · PMO #54 políticas autorización por monto · serie foliador final · denominación canónica del rol

---

#### 2.8.2 Módulo NC Perú — R16A-RE-FU-033

> **⚠ Toda la mecánica fiscal SUNAT requiere validación con el asesor fiscal peruano de PROQUIFA antes de implementarse.**

- Estructura funcional idéntica a MX (R16A-RE-FU-032); diferencias: catálogos SUNAT, campos CPE tipo 07, mecánica anulación peruana
- Empresa emisora única: Golocaer S.A.C.
- NCs históricas pre-go-live NO se importan

**Motivo:** catálogo 09 SUNAT — 11 motivos disponibles:

| Código | Motivo | Modalidad |
|---|---|---|
| 01 | Anulación de la operación | Por partidas (totalidad) |
| 02 | Anulación por error en el RUC | Por partidas (totalidad) |
| 03 | Corrección por error en la descripción | Por partidas |
| 04 | Descuento global | Manual |
| 05 | Descuento por ítem | Por partidas |
| 06 | Devolución total | Por partidas |
| 07 | Devolución por ítem | Por partidas |
| 08 | Bonificación | Manual |
| 09 | Disminución en el valor | Manual |
| 10 | Otros conceptos | Manual |
| 13 | Ajuste por crédito fiscal | Manual |

**Pendiente confirmar qué motivos se habilitan en R16.**

**Anulación de facturas:**
- **NC motivo 01** (principal): plazo 10 días hábiles desde la emisión; deja la operación en cero; hereda totalidad de partidas del comprobante origen
- **Comunicación de Baja** (excepcional): dentro de los 7 días calendario desde la emisión; recomendada para comprobantes no entregados al cliente; **pendiente confirmar si se implementa en PQF2 para Perú**
- NO aplica la cancelación SAT condicionada a "totalidad + mismo mes" (mecánica mexicana)

**Campos fiscales XML (CPE tipo 07, UBL 2.1):**
- `InvoiceTypeCode=07`
- Referencia al comprobante afectado: `cbc:ReferenceID` + `cac:BillingReference` por serie-correlativo (**NO por UUID**)
- `cbc:ResponseCode` = código catálogo 09
- `cbc:Description` = sustento del motivo (**origen del texto pendiente de validar**: auto-generado vs capturado por usuario)
- IGV 18%; NO se usan UsoCFDI, MetodoPago, FormaPago, UUID

**Tipo de cambio NC PE (diferencia clave vs MX):** la NC hereda el TC de la **fecha de emisión de la factura ORIGEN** (no el del día de la NC), conforme a SUNAT — la NC es una modificación de la operación original (Oficio SUNAT 024-2000). En MX se usa el TC del día del timbrado; esta diferencia es **intencional**.

**Timbrado:** ante SUNAT; modalidad pendiente de definir (**no se reutiliza PAC MX** — brecha mayor bloqueante)

**Foliado:** serie (prefijo F para facturas) + correlativo 8 dígitos consecutivo por Golocaer S.A.C.

**Conservación:** XML + constancia + representación gráfica 5 años desde 1 enero del año siguiente (R.S. 117-2017/SUNAT)

**Acoplamiento uni-direccional con VC:** NCs vigentes disponibles en Paso 2 de VC; VC no genera ni cancela NCs; si NC y documento destino son de misma moneda → aplicación directa; si son de monedas distintas → **pendiente confirmar si se permite y qué TC usar en la conversión**

**Riesgos:** mecánica fiscal SUNAT no validada (todo debe re-validarse con asesor fiscal) · elección incorrecta mecanismo anulación (NC vs Baja fuera de plazo → rechazo SUNAT) · brecha timbrado SUNAT no resuelta (bloqueante) · declaraciones ya presentadas que incluyan la factura afectada podrían requerir rectificación IGV-Renta · catálogo 09 no acotado → NCs mal clasificadas

**Pendientes:** motivos catálogo 09 a habilitar en R16 · origen texto sustento `cbc:Description` · NC en moneda extranjera aplicada en VC (TC a usar en conversión) · modalidad emisión SUNAT (**bloqueante**) · maquetas PE no disponibles

---

#### 2.8.3 PDF NC México — R16A-RE-FU-034

- Generado al confirmar timbrado en Paso 3 del wizard NC MX (R16A-RE-FU-032)
- CFDI tipo E, CFDI 4.0
- **Identidad visual**: logo + paleta empresa emisora (Golocaer naranja, Mungen verde, Proquifa cyan, Proveedora cyan); iconografía certificaciones; consistente con Factura MX (FU-021) y Complemento de Pago MX (FU-030)

**Secciones del PDF:**
- Emisor: RS, RFC, Lugar Expedición, Fecha/Hora Expedición, Régimen Fiscal
- Receptor: RS, RFC, Domicilio Fiscal, Régimen Fiscal receptor, Uso CFDI (G02)
- Comprobante: Serie, Folio, Versión 4.0, UUID, Fecha/Hora Certificación, Fecha/Hora Emisión, Tipo E-Egreso, Moneda, TC cuando aplique, MetodoPago PUE, FormaPago
- Motivo y CFDI relacionado: tipo de relación SAT (01), Folio + UUID de la factura origen
- Partidas (modalidad por partidas): ClaveProdServ, NoIdentificacion, Descripción, Cant.NC, ClaveUnidad, ValorUnitario, Importe, Impuesto Traslado por línea
- Concepto (modalidad manual): ClaveProdServ 84111506, Cantidad 1, ClaveUnidad ACT, Descripción, Importe, Impuesto Traslado
- Totales: Subtotal, IVA, Total (todos en moneda de la factura origen), Total en letra
- Trazabilidad SAT: Núm. serie cert. SAT, Núm. serie CSD emisor, Sello Digital SAT, Sello Digital CFDI, Cadena Original Complemento Certificación Digital del SAT
- Código QR SAT (mismo patrón que Factura MX)

Envío PDF+XML al cliente tras timbrado exitoso; CC ESAC + analista CxC

**Riesgos:** cálculo erróneo IVA al recalcular partidas · cancelación factura origen fallida tras NC timbrada (NC vigente + factura también vigente)

**Pendientes:** FormaPago modalidad manual · UsoCFDI G02 vs G03 · PMO #31 plantilla correo · vigencia iconografía certificaciones · serie foliador final

---

#### 2.8.4 PDF NC Perú — R16A-RE-FU-035

> **⚠ Toda la mecánica fiscal SUNAT requiere validación con el asesor fiscal peruano antes de implementarse.**

- CPE tipo 07, UBL 2.1; empresa emisora: Golocaer S.A.C. (RUC 20612772941)
- Modalidad de emisión electrónica ante SUNAT pendiente de definir (**no se reutiliza PAC MX**)

**Estructura XML (CPE tipo 07):**
- `InvoiceTypeCode=07`; referencia al comprobante afectado por serie-correlativo (`cbc:ReferenceID` + `cac:BillingReference`)
- `cbc:ResponseCode` (catálogo 09) + `cbc:Description` (sustento del motivo)
- Líneas `cac:CreditNoteLine` por partidas (modalidad por partidas) o concepto único (modalidad manual)
- IGV 18%; Totales en moneda del comprobante origen (Valor Venta, IGV, Importe Total) con nomenclatura SUNAT
- TC: hereda el de la **fecha de emisión de la factura origen** (no el del día de la NC)
- **Sin**: TipoDeComprobante E, UsoCFDI, TipoRelacion, MetodoPago, FormaPago, UUID, sello digital SAT, cadena complementaria

**Secciones del PDF:**
- Encabezado: logo + paleta Golocaer S.A.C.
- Emisor: RUC, RS, domicilio fiscal
- Receptor: RUC/documento, RS
- Comprobante: tipo "07 – Nota de Crédito Electrónica", serie-correlativo, fecha
- Motivo: código + descripción catálogo 09; comprobante origen referenciado por serie-correlativo
- Detalle: líneas (por partidas) o concepto (manual) con importes e IGV desglosado
- Totales: Valor Venta, IGV, Importe Total (nomenclatura SUNAT), moneda del comprobante origen
- **QR verificación SUNAT** + leyenda obligatoria "Representación impresa de la Nota de Crédito Electrónica, consúltela en..."
- El respaldo legal es la **firma digital del emisor** y el **valor resumen/hash** (sin sello digital SAT ni cadena complementaria)

Envío PDF+XML al cliente tras timbrado; CC ESAC

**Riesgos:** mecánica fiscal SUNAT no validada · recálculo erróneo IGV al heredar/recalcular partidas · brecha timbrado SUNAT no resuelta (bloqueante) · representación impresa no conforme si falta la leyenda obligatoria, el QR o la nomenclatura SUNAT

**Pendientes:** modalidad emisión electrónica SUNAT (**bloqueante**) · formato asunto y plantilla cuerpo correo PE · origen texto sustento `cbc:Description` · maquetas PDF NC PE no disponibles

---

### 2.9 Buzón de Cobros — NUEVO módulo

**R16A-RE-FU-00X** (aprox. — ubicado entre los primeros requisitos del documento)

- Nuevo módulo que integra correos de cobro clasificados por el **Mailbot**
- El Mailbot agrega la categoría **"Cobro"** a las existentes (Cotización / Pedido / Otros)
- Pantalla principal: bandeja de correos de cobro agrupada por cliente; navegación directa a Validar Cobro desde el correo
- Visibilidad filtrada por cartera del Cobrador (consistente con NC, VC, FxA)
- Facilita la gestión de cobros recibidos por correo antes de registrarlos en Paso 1 de Validar Cobro

---

## 3. PENDIENTES Y PMO TRANSVERSALES

| PMO / Ítem | Descripción | Módulos afectados |
|---|---|---|
| **PMO #31** | Plantilla del cuerpo del correo de envío (Proforma / Factura / NC / CdP / Inconsistencia) | Todos los módulos con envío de correo |
| **PMO #54** | Políticas de autorización por monto en NCs (umbrales para coordinador/director) | NC MX/PE |
| **PMO #55** | Reglas SAT de decimales en CFDI 4.0 | Todos los módulos MX |
| Rol canónico | Denominación formal: "Gestor de Cobranza" vs "Analista de CxC" | Transversal |
| Reintento PAC | Política formal de reintento ante fallo de timbrado TurboPac | FxA MX, VC MX, NC MX, CdP MX |
| FechaPago CdP | Convención hora en FechaPago del CdP (12:00:00 fija vs hora real del cobro) — **validar con asesor fiscal** | CdP MX (FU-030) |
| IVA ≠ 16% | Soporte tasas frontera (8%, 0%) en CdP MX | CdP MX (FU-030) |
| Serie CdP | Validación serie "P" en el foliador del Complemento de Pago | CdP MX (FU-030) |
| Serie NC MX | Validación serie "P2" en el foliador de NC MX | NC MX (FU-032) |
| Iconografía MX | Vigencia de la iconografía de certificaciones (ISO, NEEC, edQM, FELUM, USP, Microbiologics, APACOR, CHATA, Pharmaffiliates, Amex) | PDFs MX (FU-021/030/034) |
| Salida Paso 3 | Vía operativa de excepción cuando usuario necesita salir del Paso 3 con líneas ya timbradas (ej. cliente cancela a último minuto) | VC Paso 3 MX/PE (FU-028/029) |
| Cobro vs FxA PE | Qué acciones ofrece el sistema cuando el cobro en PE corresponde a una factura ya emitida (no hay CdP en PE) — conciliación interna + ¿constancia al cliente? | VC Paso 3 PE (FU-029) |
| NC moneda ext. PE | Si una NC en moneda extranjera puede aplicarse en VC PE a un documento de moneda distinta, y qué TC usar en la conversión | NC PE + VC Paso 2 PE (FU-027/033) |
| Baja PE | Confirmar si la Comunicación de Baja se implementa en PQF2 Perú o si la anulación es únicamente vía NC motivo 01 | NC PE (FU-033) |
| Motivos catálogo 09 | Cuáles de los 11 motivos del catálogo 09 SUNAT se habilitan en R16 | NC PE (FU-033/035) |
| `cbc:Description` | Origen del texto del sustento en NC PE: auto-generado desde motivo/partidas o capturado por el usuario | NC PE (FU-033/035) |
| Tolerancia PE | Umbral de tolerancia de pago de menos para Perú (monto, moneda, tratamiento cuando facturación ≠ PEN) | VC Paso 2 PE (FU-027) |
| TC oficial PE | Fuente oficial del tipo de cambio para operaciones Perú (no aplica el DOF mexicano) | VC Paso 1/2 PE (FU-025/027), NC PE |
| FormaPago NC manual | Comportamiento correcto del FormaPago en NC MX modalidad manual (el mockup muestra "99 Por definir", que es fiscalmente incorrecto para NC PUE) | NC MX (FU-032/034) |
| Asesor fiscal PE | Toda la mecánica fiscal SUNAT (NC PE FU-033/035, FxA PE FU-020, VC Paso 3 PE FU-029) requiere validación formal antes de implementar | NC PE, FxA PE, VC PE |
| Maquetas PE | Maquetas de Validar Cobro PE, NC PE y PDF NC PE no disponibles; detalle se validará contra ellas cuando lleguen | VC/NC PE |

---

*Resumen generado a partir de: `R16M_Matriz_de_Requisitos.md` (728 KB, 35 requisitos R16A-RE-FU-001 a R16A-RE-FU-035)*
