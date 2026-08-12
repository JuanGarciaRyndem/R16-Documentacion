# Flujo Notas de Crédito ↔ Validar Cobro (Región México)

**Fecha:** 2026-08-11
**Alcance:** análisis del flujo end-to-end entre el módulo independiente de Notas de Crédito (RE-FU-032) y el wizard de Validar Cobro (RE-FU-023 a RE-FU-030), en Región México, en el release R16 (clientes Prepago).
**Naturaleza:** documento de análisis — consolida decisiones ya tomadas en los requisitos R16 y en la guía técnica del coordinador. No es especificación DDL; los modelos vinculantes son los de los requisitos.

---

## 1. Actores, módulos y aplicativos

| Actor | Módulo | Aplicativo |
|---|---|---|
| **Tesorería** | Notas de Crédito — Módulo independiente (RE-FU-032) | ProquifaDotNet.Finanzas + Timbrado + DocumentBuilder |
| **Gestor de Cobranza / Analista CxC** | Validar Cobro — Pantalla principal + Wizard (RE-FU-023 a RE-FU-029) | ProquifaDotNet.Finanzas |
| **Sistema (batch)** | Timbrado + LegacySync (SSIS) | ProquifaDotNet.Timbrado + LegacySync |

> **Acoplamiento uni-direccional (Regla 13 RE-032, Regla M2 RE-032):** el módulo NC alimenta a Validar Cobro. Validar Cobro NO puede generar ni cancelar NCs. La única acción de Validar Cobro sobre una NC es aplicarla a un cobro (marca `fccNotaCredito.Aplicada = 1`).

---

## 2. Vista panorámica del flujo

```
[TESORERÍA — RE-FU-032]                     [GESTOR DE COBRANZA — RE-FU-023 a RE-FU-030]
                                                                                 
    Wizard NC 4 pasos                            Pantalla principal Validar Cobro
    ┌──────────────────────┐                     ┌─────────────────────────────┐
    │ 1. Buscar Factura    │                     │ Listado de clientes         │
    │ 2. Capturar Datos    │                     │ con pendientes              │
    │ 3. Confirmar + PDF   │                     └──────────┬──────────────────┘
    │ 4. NC Emitida        │                                │
    └──────────┬───────────┘                                ▼
               │                                Wizard Validar Cobro 3 pasos
               │                                ┌──────────────────────────────┐
               ▼                                │ Paso 1 · Captura del Cobro   │
    Timbrado CFDI E                             │   (RE-FU-024)                │
    (Serie P2, TurboPac)                        │                              │
               │                                │ Paso 2 · Asociación          │
               ▼                                │   Cobro ↔ Documento          │
    NC VIGENTE en fccNotaCredito                │   ↔ NC (RE-FU-026)  ◄────────┼──── consume NC vigente
    (Aplicada=0, Activo=1)     ─────────────────┤                              │     como forma de pago
               │                                │ Paso 3 · Facturación         │
               │                                │   + CP (RE-FU-028/030)       │
               ▼                                └──────────────────────────────┘
    LegacySync a PCconnect (SSIS)                                       │
                                                                        ▼
                                                       Al confirmar CFDI PPD emitido
                                                       (TipoRelacion=02 hacia NC)
                                                       + Complemento de Pago
                                                       (FormaPago=23 Novación)
                                                                        │
                                                                        ▼
                                                       fccNotaCredito.Aplicada = 1
                                                       fccPagoCliente asociado a NC
```

---

## 3. Fase 1 — Emisión de la NC (RE-FU-032, Tesorería)

### 3.1. Precondiciones
- Cliente **Prepago** (no aplica Crédito en R16).
- Factura origen **Vigente** ante el SAT (no cancelada) con antigüedad **≤ 5 años**.
- Motivo capturado por el usuario:
  - **Devolución de mercancía** → modalidad *por partidas* (tabla heredada de la factura, `Cant. NC` editable por línea).
  - **Descuento o bonificación** → modalidad *manual* (Monto libre ≤ Total factura, Concepto obligatorio con materialidad fiscal).

### 3.2. Wizard 4 pasos
1. **Buscar Factura** — selección de cliente + UNA factura vigente prepago.
2. **Capturar Datos** — motivo, modalidad, campos fiscales fijos (`TipoDeComprobante=E`, `TipoRelacion=01`, `UsoCFDI=G02`, `MetodoPago=PUE`, `FormaPago` heredada), opción de **cancelar factura origen** si NC por totalidad + mismo mes calendario.
3. **Confirmar + previsualización PDF** — resumen + PDF sin sello.
4. **NC Emitida** — CFDI timbrado (Serie P2 + UUID SAT); misma vista que el detalle de NC ya generada.

### 3.3. Timbrado
- PAC **TurboPac** vía ProquifaDotNet.Timbrado (`POST /api/v1/stamp/credit-note`).
- Foliador `EmpresaFolio` Serie "P2" por empresa emisora (GOL/MUN/PRO/PQF).
- Persistencia: `INSERT CFDIGenerada`, `INSERT CFDIGeneradaRelacionado` (UUID factura origen), `UPDATE fccNotaCredito` con UUID y estado `VIGENTE`.
- Cancelación condicional de la factura origen si el usuario lo marcó (motivo `c_MotivoCancelacion` SAT).

### 3.4. Post-timbrado
- **PDF generado desde XML timbrado** vía DocumentBuilder (evita el bug del legado documentado en la guía del coordinador — hallazgo 3).
- **Correo automático** al cliente con PDF + XML adjuntos:
  - Para = contacto del cliente vinculado a la factura.
  - CC = ESAC asignado + analista de CxC.
  - Asunto = "Nota de Crédito + folio NC + folio factura" (plantilla PMO #31).
- **Reenvío** disponible desde el detalle.
- **LegacySync (SSIS)** transfiere la NC timbrada a PCconnect.

### 3.5. Estado al final de la fase 1
```
fccNotaCredito:
    Folio                  = P2-NNNNNN
    IdCFDIGenerada         = UUID SAT
    IdCatNotaCreditoEstado = VIGENTE     ← disponible para Validar Cobro
    Aplicada               = 0
    Activo                 = 1
    Monto                  = X
    IdCatMoneda            = MXN | USD | EUR (heredado de factura origen)
    TipoDeCambio           = TC del día del timbrado
```

---

## 4. Fase 2 — La NC queda disponible en Validar Cobro

### 4.1. Acoplamiento uni-direccional
- **RE-FU-023** (pantalla principal Validar Cobro) NO muestra NCs directamente en el listado de clientes; el conteo de NCs vigentes no forma parte de la acción contextual `PROCESS_PAYMENTS` / `MANAGE_COLLECTIONS`.
- **RE-FU-026** (Wizard Paso 2 Asociación MX) consume el catálogo de NCs vigentes al abrir la sesión de asociación para un cliente.

### 4.2. Query de disponibilidad (endpoint B2 de RE-FU-026)
```
GET /api/v1/validate-collection/client/{idCliente}/activeCreditNote

Filtro: fccNotaCredito
        WHERE IdCliente = @idCliente
          AND Aplicada = 0
          AND Activo = 1
          AND IdCatNotaCreditoEstado = VIGENTE

Retorna: List<ActiveCreditNoteDto>
  { IdFCCNotaCredito, Folio, IdCFDIGenerada (UUID),
    Monto, IdCatMoneda, ClaveMoneda, TipoDeCambio }
```

### 4.3. Precondición para consumir la NC
- El cobro que se está capturando en el wizard (Paso 1, RE-FU-024) debe ser **del mismo cliente** que emitió la NC.
- La NC debe estar **VIGENTE** (`EstatusSAT = Vigente` según guía del coordinador, `IdCatNotaCreditoEstado = VIGENTE` según modelo R16).

---

## 5. Fase 3 — Aplicación de la NC en el Paso 2 (RE-FU-026)

### 5.1. Motor de cálculo del saldo de asociación
Fórmula (B3 de RE-FU-026):

```
SaldoAsociacion = (SumaCobrosAplicados + SumaNCAplicadas) − SumaAdeudoDocumentosSeleccionados
```

Escenarios gobernados:

| Resultado | Escenario | Acción |
|---|---|---|
| `= 0` | Pago exacto | Permite avanzar al Paso 3 |
| `> 0` | Sobrepago | Registra saldo a favor en `fccSaldoFavorCliente`; permite avanzar |
| `< 0 AND ABS ≤ 100 MXN` | Tolerancia | `ToleranciaAplicada` en `fccSaldoFavorCliente`; permite avanzar |
| `< 0 AND ABS > 100 MXN` | Insuficiente | Bloquea avance; requiere inconsistencia o dejar pendiente |

### 5.2. Multi-divisa (OBS-052, aplicado también a NCs)
Cuando la NC y el documento asociado están en monedas distintas:

- Regla general: la conversión de la NC al importe del documento se hace con el **TC del pago del cobro** (`fccPagoCliente.TipoDeCambioMonedaFacturacion` — OBS-050), NO con el TC de emisión del documento ni con el TC del día de aplicación.
- Fundamento: Art. 8 Ley Monetaria EUM + Guía CFDI 4.0 / CRP 2.0 (`EquivalenciaDR` = TC del pago).
- El diferencial cambiario contra el TC de emisión del documento se registra como fluctuación cambiaria del emisor (LISR / NIF B-15), no como saldo pendiente del cliente.

> ⚠️ **Duda cruzada con guía del coordinador (sección C1 de `Dudas_Guia_NotasDeCredito_MX.md`):** la guía propone usar el **TC heredado de la NC** (no el del pago) para la conversión, argumentando que la NC representa un crédito fijo. Confirmar qué regla aplica para el consumo del saldo de NC — TC del pago (OBS-052) vs TC heredado de la NC (guía coordinador). **Alta prioridad para RE-032 + RE-026.**

### 5.3. Persistencia de la aplicación (B4 de RE-FU-026)
En una sola transacción, al confirmar la asociación o avanzar al Paso 3:

```sql
-- Por cada NC aplicada al cobro:
UPDATE fccNotaCredito
   SET Aplicada = 1,
       IdFCCPagoCliente = @IdCobro,
       IdCatNotaCreditoEstado = APLICADA
 WHERE IdFCCNotaCredito = @IdNC;

-- Registro de la asociación cobro↔documento
INSERT fccPagoFacturaPedido | fccPagoFacturaAdelanto
   (IdFCCPagoCliente, IdTPProformaPedido | IdFccFactura, Monto);

-- Si hubo sobrepago o tolerancia
INSERT fccSaldoFavorCliente (IdFCCPagoCliente, TipoSaldo, Monto);
```

### 5.4. Alcance de aplicación en R16
- **Aplicación total** — el 100% del `fccNotaCredito.Monto` se consume contra un documento del cobro. La NC pasa a `Aplicada = 1`.
- **Aplicación parcial** — la NC cubre menos que el documento y sobra por cobrar. Se combina con el monto del cobro para cubrir el saldo.
- **Sobrepago de NC (Monto NC > adeudo del documento)** — el remanente **queda fuera de scope R16** (Brecha del RE-026). Pendiente confirmar tratamiento operativo con PROQUIFA.

> ⚠️ **Contradicción con guía del coordinador:** la guía sí describe consumo fraccionado en múltiples facturas futuras (sección 8), confirmado por el contador. RE-026 lo deja explícitamente fuera de scope R16. Punto abierto para el diseño de RE-032/026.

---

## 6. Fase 4 — Emisión de la factura destino en el Paso 3 (RE-FU-028)

Cuando el cobro consume una NC y avanza a facturación:

### 6.1. Configuración de la factura destino (según guía del coordinador sección 8)
- **`MetodoPago = PPD`** (Pago en Parcialidades o Diferido).
- **`FormaPago = 99`** (Por definir).
- **`CfdiRelacionados`:** UUID de la NC + **`TipoRelacion = 02`** (Nota de débito de los documentos relacionados).
- El producto se factura completo, sin fragmentar partidas ni prorratear cantidades.

> ⚠️ **Punto abierto:** RE-FU-028 hoy documenta escenarios A/B/C/D de timbrado (Factura, Factura Anticipo, Cascada PPD + Complemento, Complemento desde FAA existente). Falta el escenario **E**: "Factura PPD con `TipoRelacion=02` hacia NC como pago total o parcial". Requiere agregar el caso al mapping service + al `StampInvoiceRequestDto`.

### 6.2. Timbrado
- Endpoint interno `POST /api/v1/stamp/invoice` (RE-FU-018).
- El XML lleva `CfdiRelacionados` con `TipoRelacion=02` + UUID NC.

---

## 7. Fase 5 — Complemento de Pago que documenta el consumo (RE-FU-030)

### 7.1. Cascada al confirmar la aplicación
La factura PPD emitida en la Fase 4 se salda con **uno o varios Complementos de Pago**:

- **CP con `FormaPago=23` (Novación)** — documenta la parte cubierta por la NC (`ImpPagado = MontoConsumido de la NC`).
- **CP con forma real** (`03` Transferencia, `01` Efectivo, etc.) — documenta el remanente que viene de dinero real, si lo hay.

### 7.2. `EquivalenciaDR` en el CP con `FormaPago=23`
- Según OBS-052 aplicado desde el cobro: se usa `TipoDeCambioMonedaFacturacion` del cobro.
- Según guía del coordinador: se usa `TipoCambio` / `FactorUSD` heredado de la NC.
- **Regla operativa a confirmar en el diseño de RE-030** (ver duda D1 en `Dudas_Guia_NotasDeCredito_MX.md`).

### 7.3. `NumParcialidad` (RE-FU-030)
- Se calcula con `COUNT + UPDLOCK` sobre CPs previos de la misma factura.
- Si la factura destino recibe primero el CP con `FormaPago=23` (evento inmediato al confirmar) y después el CP con forma real (evento posterior cuando llega el dinero), la parcialidad se enumera secuencialmente.

### 7.4. `FechaPago` del CP (OBS-051)
- Cuando el CP corresponde a la aplicación de un saldo previo (aplica al caso NC), `FechaPago = fecha de aplicación`, NO fecha de origen de la NC.

---

## 8. Estados y transiciones — vista consolidada

### 8.1. Estado de la NC (`fccNotaCredito`)
Estados propuestos (según RE-FU-032, tabla `catNotaCreditoEstado`):

| Estado | Semántica | Momento |
|---|---|---|
| `PENDIENTE` | NC en borrador dentro del wizard, aún no timbrada | Wizard Paso 1–2 |
| `VIGENTE` | Timbrada, disponible para consumo en Validar Cobro | Timbrado exitoso, `Aplicada=0` |
| `ENVIADA` | Correo enviado al cliente | Envío OK post-timbrado |
| `APLICADA` | Consumida (total o parcial) en Validar Cobro | Confirmación asociación en Paso 2 |
| `CANCELADA` | Cancelada ante el SAT (evento externo) | Actualización manual por soporte |

**Estatus SAT paralelo** (según guía del coordinador sección 13):
- `EstatusSAT = Vigente` / `Cancelado` — fuente externa, sincronización pendiente de resolver (ver duda B1 en `Dudas_Guia_NotasDeCredito_MX.md`).

### 8.2. Estado del cobro que consume la NC
`fccPagoCliente` con la NC aplicada tiene `TotalRecibido = MontoCobroReal + SumaNCsAplicadas`, y su asociación queda registrada en `fccPagoFacturaPedido` / `fccPagoFacturaAdelanto`.

### 8.3. Estado de la factura destino
Al confirmar la aplicación con Complementos de Pago:
- La factura PPD queda con **saldo insoluto = 0** si el CP con `FormaPago=23` + el CP con forma real cubren el 100%.
- Si la NC cubre 100% de la factura, el CP con `FormaPago=23` es único y no hay CP adicional.

---

## 9. Matriz de decisión NC ↔ Validar Cobro

| Situación | Camino |
|---|---|
| Cliente Prepago pagó, el producto llegó completo, no hay ajuste | Ningún flujo NC. Validar Cobro cierra normalmente. |
| Cliente Prepago pagó, el producto llegó parcial/no llegó | Tesorería genera NC (RE-032). Validar Cobro consume la NC en Paso 2. |
| Cliente Prepago pagó, se devuelve mercancía posterior | Tesorería genera NC (RE-032). Validar Cobro consume la NC en Paso 2 en la próxima venta. |
| Cliente Prepago pagó de más | RE-026 registra sobrepago en `fccSaldoFavorCliente`; NO se genera NC automáticamente. Tesorería puede emitir NC si corresponde comercialmente. |
| Cliente NO pagó y no va a pagar | Cancelación por falta de pago desde RE-023 (orquestador distribuido). No hay NC. |
| Sobrepago de NC en aplicación (NC > adeudo) | **Fuera de scope R16** — pendiente confirmar. |

---

## 10. Puntos abiertos consolidados

| # | Tema | Impacto | Referencia |
|---|---|---|---|
| 1 | ¿Qué TC se usa para convertir la NC a la moneda de la factura destino: `TipoDeCambioMonedaFacturacion` del cobro (OBS-052) o `TipoCambio`/`FactorUSD` heredado de la NC (guía coordinador)? | Alto — motor de cálculo Paso 2 + `EquivalenciaDR` del CP | Sección 5.2 · Duda C1 |
| 2 | Consumo fraccionado de NC en múltiples facturas — permitido por SAT y contador, marcado fuera de scope R16 | Alto — RE-032/026 | Sección 5.4 |
| 3 | Escenario "Factura PPD con `TipoRelacion=02` hacia NC" no está enumerado en RE-028 | Alto — mapping XML + StampInvoiceRequestDto | Sección 6.1 |
| 4 | Sobrepago de NC (Monto NC > adeudo) | Medio — tratamiento operativo | Sección 5.4 |
| 5 | Sincronización de `EstatusSAT` sin autoservicio | Medio — riesgo de desalineación | Sección 8.1 · Duda B1 |
| 6 | `FactorUSD` alias del `TipoDeCambio` vs MXN o TC adicional | Medio — impacto en modelo | Duda C3 |
| 7 | ¿Se genera CP automáticamente al confirmar la asociación con NC (evento inmediato) o queda pendiente para timbrar cuando llegue el remanente? | Alto — orden operativo Paso 3 | Sección 7.3 · Duda D4 |
| 8 | Fecha del CP con `FormaPago=23` al aplicar NC (fecha de aplicación vs fecha NC) | Medio — alineado con OBS-051 | Sección 7.4 |
| 9 | ¿La NC puede reversarse en el sistema si el usuario la aplicó por error antes de emitir la factura destino? | Medio — auto-guardado Paso 2 | RE-026 B6 |

---

## 11. Diagrama de secuencia (feliz camino)

```
Tesorería          NC Finanzas         Timbrado         Legacy       BD ProquifaDotNet
    │                    │                 │              │                 │
    │ Wizard NC 4 pasos  │                 │              │                 │
    ├───────────────────►│                 │              │                 │
    │                    │ POST /credit-note              │                 │
    │                    ├────────────────►│              │                 │
    │                    │                 │ TurboPac     │                 │
    │                    │                 │◄─────►SAT    │                 │
    │                    │◄────────────────│              │                 │
    │                    │                 │              │                 │
    │                    │ INSERT CFDIGenerada + UPDATE fccNotaCredito ─────►
    │                    │                 │              │                 │
    │                    │ Envío correo (PDF + XML)       │                 │
    │                    │                 │              │                 │
    │                    │                 │              │◄────────────────┤  SSIS
    │                    │                 │              │ NC → PCconnect │
    │                    │                 │              │                 │
    ═══════════════════════════════════════════════════════════════════════════
    (NC VIGENTE, Aplicada=0, disponible en Validar Cobro)
    ═══════════════════════════════════════════════════════════════════════════

Gestor           Validar Cobro       Timbrado         Legacy       BD ProquifaDotNet
    │                    │                 │              │                 │
    │ Pantalla principal │                 │              │                 │
    ├───────────────────►│                 │              │                 │
    │  Wizard Paso 1     │                 │              │                 │
    │  (Captura cobro)   │                 │              │                 │
    ├───────────────────►│                 │              │                 │
    │                    │ INSERT fccPagoCliente (Confirmado=1)              │
    │                    │─────────────────────────────────────────────────►│
    │  Wizard Paso 2     │                 │              │                 │
    │  (Asociación)      │                 │              │                 │
    ├───────────────────►│                 │              │                 │
    │                    │ GET activeCreditNote           │                 │
    │                    │─────────────────────────────────────────────────►│
    │                    │◄─────────────── List<CreditNoteDto>              │
    │                    │                 │              │                 │
    │  Selecciona NC     │                 │              │                 │
    │  + documento(s)    │                 │              │                 │
    ├───────────────────►│                 │              │                 │
    │                    │ Motor SaldoAsociacion (OBS-052)                  │
    │                    │                 │              │                 │
    │  Confirma asoc.    │                 │              │                 │
    ├───────────────────►│                 │              │                 │
    │                    │ TRAN: UPDATE fccNotaCredito.Aplicada=1           │
    │                    │       INSERT fccPagoFacturaPedido/Adelanto        │
    │                    │─────────────────────────────────────────────────►│
    │                    │                 │              │                 │
    │  Wizard Paso 3     │                 │              │                 │
    │  (Facturación)     │                 │              │                 │
    ├───────────────────►│                 │              │                 │
    │                    │ POST /stamp/invoice (PPD, TipoRelacion=02 → NC)   │
    │                    ├────────────────►│              │                 │
    │                    │◄──── UUID Factura              │                 │
    │                    │                 │              │                 │
    │                    │ POST /stamp/payment-complement (FormaPago=23)     │
    │                    ├────────────────►│              │                 │
    │                    │◄──── UUID CP    │              │                 │
    │                    │                 │              │                 │
    │                    │                 │              │◄────────────────┤  SSIS
    │                    │                 │              │ Factura + CP    │
    │                    │                 │              │ → PCconnect     │
```

---

## 12. Referencias

**Requisitos R16:**
- `Requisitos/R16A-RE-FU-023/[R16A-RE-FU-023][DIS-SOL] Diseño de la solución.md` — pantalla principal Validar Cobro.
- `Requisitos/R16A-RE-FU-024/R16A-RE-FU-024-Back.md` — Wizard Paso 1 Captura Cobro (OBS-049/050/051/052).
- `Requisitos/R16A-RE-FU-026/R16A-RE-FU-026-Back.md` — Wizard Paso 2 Asociación MX (motor de saldo, aplicación NC).
- `Requisitos/R16A-RE-FU-028/R16A-RE-FU-028-Back.md` — Wizard Paso 3 Facturación MX (Escenarios A/B/C/D).
- `Requisitos/R16A-RE-FU-030/DIS-SOL-backend-complemento-pago.md` — Complemento de Pago MX.
- `Requisitos/R16A-RE-FU-032/R16A-RE-FU-032.md` — módulo NC MX (Tesorería).
- `Requisitos/R16A-RE-FU-032/R16A-RE-FU-032-Back.md` — impacto backend NC MX.

**Análisis:**
- `Analisis/Facturación/Guia_Tecnica_Notas_de_Credito_MX.md` — propuesta del coordinador.
- `Analisis/Facturación/Dudas_Guia_NotasDeCredito_MX.md` — dudas guía coordinador vs R16.

**Endpoints:**
- `Endpoints/Endpoints-Finanzas.md` — RE-026 (`activeCreditNote`, `associationConfirmation`), RE-028 (`fiscalDocumentLine/stamp`), RE-032 (módulo NC).
- `Endpoints/Endpoints-Timbrado.md` — `stamp/invoice`, `stamp/credit-note`, `stamp/payment-complement`, `stamp/cancel`.
