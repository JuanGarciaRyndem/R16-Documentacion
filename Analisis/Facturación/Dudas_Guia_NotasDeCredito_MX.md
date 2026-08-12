# Dudas y Observaciones — Guía Técnica Notas de Crédito México

**Documento fuente:** `Analisis/Facturación/Guia_Tecnica_Notas_de_Credito_MX.md`
**Naturaleza del documento fuente:** propuesta del **coordinador** sobre reglas de negocio, motivos, matrices de decisión y flujos operativos de NC. Los **modelos de datos vinculantes** son los definidos en los requisitos R16 (`fccFactura`, `fccNotaCredito`, `fccPagoCliente`, `fccDocumentoFiscalCobro`, etc.); los nombres de tabla genéricos usados en la guía (`Factura`, `NotaCredito`, `ConsumoSaldoAFavor`) se interpretan como conceptos, no como definiciones DDL — la implementación real se apega al modelo R16.
**Requisitos R16 relacionados:** RE-FU-030 (Complemento de Pago), RE-FU-032 (Timbrar Nota de Crédito MX), RE-FU-033 (Timbrar Nota de Crédito PE), RE-FU-024 (Wizard Validar Cobro Paso 1), RE-FU-023 (Cancelación distribuida)
**Fecha:** 2026-08-11
**Estado:** pendiente de revisión con equipo técnico + validación con cliente/contador

---

## Objetivo

Consolidar las dudas conceptuales, contradicciones e integraciones pendientes que la **propuesta del coordinador** deja abiertas frente a las decisiones ya tomadas en los requisitos R16, para llevar a sesión de revisión con el equipo de arquitectura y con el cliente antes de iniciar el desarrollo de RE-FU-032.

**Fuera del alcance de este documento:** ajuste de nombres de tabla o de campos DDL — el modelo de datos vinculante es el de los requisitos R16, no el de la guía.

---

## A. Alcance conceptual y responsabilidades del flujo NC

> **Aclaración:** los nombres de tabla y campos de la guía son referenciales — el modelo de datos vinculante es el de los requisitos R16. Esta sección se enfoca en la responsabilidad conceptual del flujo, no en el DDL.

### A1 — Requisito propietario del flujo NC completo

**Contexto:** la guía cubre transversalmente emisión, cálculo del Excedente, foliado, timbrado, aplicación de saldo a favor y estatus. En R16 esto se reparte al menos entre RE-FU-032 (Timbrar NC MX), RE-FU-030 (CP que consume el saldo), RE-FU-028 (Facturación con `TipoRelacion=02`) y potencialmente RE-FU-024 al momento de asociar el cobro que dispara la NC.

**Duda:**
- ¿La regla de la sección 6 (`SaldoPendiente` → `FormaPago=15/23/real`) se ejecuta como parte del Paso 3 del wizard Validar Cobro (RE-028), o en un flujo dedicado de "Nueva NC" propietario de RE-FU-032?
- ¿La bifurcación "devolver dinero vs saldo a favor" (sección 7) es una decisión operativa de negocio dentro del wizard o un flujo aparte?

**Acción propuesta:** confirmar el reparto de responsabilidades funcionales entre RE-FU-028 y RE-FU-032 en sesión de arquitectura, y documentarlo en el DIS-SOL correspondiente.

### A2 — Hallazgo del ejemplo real (`CondicionesDePago="PREPAGO 100%"`)

**Contexto:** el hallazgo 5 de la guía cita un ejemplo real de producción del timbrador legado con `CondicionesDePago="PREPAGO 100%"` como texto libre del CFDI.

**Duda:** confirmar el mapeo entre `IdCatCondicionesDePago` (modelo R16) y el campo string `CondicionesDePago` del CFDI 4.0 — en el timbrado, el string se compone desde el catálogo, no viceversa.

**Acción propuesta:** validar en el mapping service de RE-FU-032 (equivalente a `InvoicePdfMappingService` de RE-FU-021) que `CondicionesDePago` del XML se resuelva desde `IdCatCondicionesDePago` de la NC.

---

## B. Cancelación de NC — contradicción con endpoints R16

### B1 — Cancelación fuera del sistema vs endpoints ya definidos

**Contexto:** la guía (sección 13) dice: *"El sistema no ofrece ninguna función para cancelar una Nota de Crédito. Si PROQUIFA necesita cancelar una NC ante el SAT, se hace por fuera."* Sin embargo:
- `Endpoints-Timbrado.md` ya tiene `POST /api/v1/stamp/cancel` (usado por RE-032/033).
- `Endpoints-Timbrado.md` tiene `POST /api/v1/invoices/{invoiceId}/cancel` (agregado en RE-FU-023 para cancelación distribuida).

**Duda:**
- ¿Por qué NC no reutiliza esta infraestructura de cancelación?
- ¿Se descarta la consulta al SAT vía PAC (TurboPac) para sincronizar `EstatusSAT` automáticamente?
- Si el sistema puede cancelar facturas (por falta de pago RE-023) y CFDIs de NC (RE-032/033), ¿por qué la cancelación de NC específicamente se saca del sistema?

**Acción propuesta:**
- Decidir formalmente entre: (a) mantener cancelación fuera del sistema (documentando la razón operativa), o (b) reutilizar los endpoints existentes con un `POST /api/v1/creditNote/{id}/cancel` en Finanzas.
- Considerar al menos un **job periódico de sincronización de `EstatusSAT`** que consulte al PAC el estado real de las NCs marcadas internamente como `Vigente`, para detectar cancelaciones externas sin depender del reporte manual a soporte.

---

## C. Cruce con decisiones OBS-049 / OBS-050 / OBS-051 / OBS-052 de FU-024

### C1 — TC heredado de la NC vs TC del pago (OBS-052)

**Contexto:**
- OBS-052 (FU-024): la cobertura del cobro contra la factura usa el TC del pago (`TipoDeCambioMonedaFacturacion`), no el TC de emisión del documento.
- Guía NC (sección 8): al aplicar el saldo a favor de una NC a una factura futura en moneda distinta, se usa el `TipoCambio`/`FactorUSD` **heredado de la NC**, no el del día de aplicación.

**Duda:** las dos reglas son opuestas por diseño:
- Cobro con dinero fresco → TC del pago (dinero que llega el día del pago).
- NC como crédito fijo → TC heredado (crédito fijado en la venta original).

La justificación en la guía es coherente, pero conviene documentar explícitamente **la doble regla** para que el desarrollador del CP que consume saldo NC no se confunda.

**Acción propuesta:** agregar cuadro comparativo a la guía (o a `FU-024-Back.md` sección B6) del tipo:

| Origen del `EquivalenciaDR` en el CP | Regla del TC | Justificación |
|---|---|---|
| Cobro real (dinero fresco) | `TipoDeCambioMonedaFacturacion` del cobro (día llegada al banco) | Dinero fresco — Art. 8 Ley Monetaria + CRP 2.0 |
| Saldo a favor de NC | `TipoCambio` / `FactorUSD` heredado de la NC | Crédito fijo en la venta original |

### C2 — OBS-051 (fecha de aplicación del saldo a favor)

**Contexto:** OBS-051 (FU-024) fijó que la `FechaPago` del CP es la **fecha de aplicación** del saldo a la nueva factura, no la fecha en que se originó el saldo.

**Duda:** la guía NC no menciona explícitamente `FechaPago` en el CP con `FormaPago=23`. ¿Se aplica OBS-051 tal cual, o hay caso especial cuando el saldo viene de NC (por ejemplo, la fecha del CFDI de la NC como referencia adicional)?

**Acción propuesta:** confirmar alineación OBS-051 y agregarlo a la guía como referencia cruzada.

### C3 — `FactorUSD` "universal" que menciona la guía

**Contexto:** la guía asume que Pedido/Factura/Cobro/CP/NC ya tienen `FactorUSD` (guía de Facturas sección 8). En R16 (FU-024) se agregaron `TipoDeCambio` (vs MXN) y `TipoDeCambioMonedaFacturacion` (OBS-050) a `fccPagoCliente`.

**Duda conceptual:**
- ¿`FactorUSD` que menciona la guía es equivalente al `TipoDeCambio` (vs MXN) que ya tienen los objetos R16 cuando la moneda es MXN, o representa un tercer TC distinto (vs USD específicamente)?
- Si es un tercer TC, ¿su fuente sigue siendo Banxico (OBS-049) o una fuente distinta?

**Acción propuesta:** confirmar con el coordinador si `FactorUSD` es un alias conceptual del TC vs MXN que R16 ya persiste, o un TC adicional específico contra USD. Si es lo segundo, evaluar el impacto en el modelo R16 (no como ajuste DDL en este documento, sino como pregunta al coordinador).

---

## D. Integración con Complemento de Pago (FU-030)

### D1 — `EquivalenciaDR` cuando el CP consume saldo de NC

**Contexto:** `PaymentComplementCalculationService` (FU-030) hoy resuelve `EquivalenciaDR` con `ConversorDivisas` a partir del cobro. La guía NC dice: *"es el mismo mecanismo `EquivalenciaDR` que ya se usa, solo que la fuente de la conversión es el TC/FactorUSD de la NC, no el de un Cobro."*

**Duda:** hay que agregar una ruta específica en el servicio para el caso "el DR viene de una NC — tomar TC/FactorUSD de la NC, no del cobro". ¿Cómo se identifica ese caso (por el `FormaPago=23` del CP + `TipoRelacion=02` de la factura DR)?

**Acción propuesta:** agregar a FU-030 (`DIS-SOL-backend-complemento-pago.md`) un caso adicional en `PaymentComplementCalculationService`:

```
SI (CP.FormaPago == 23) → EquivalenciaDR usa NC.TipoCambio / NC.FactorUSD
SI NO                  → EquivalenciaDR usa Cobro.TipoDeCambioMonedaFacturacion (OBS-052)
```

### D2 — `ImpSaldoAnt` / `ImpPagado` del CP con `FormaPago=23`

**Contexto:** FU-030 agrega columnas de snapshot fiscal a `fccDocumentoFiscalCobro` (`ImpSaldoAnt`, `ImpPagado`, `ImpSaldoInsoluto`, `EquivalenciaDR`, `TipoCambioP_CP`).

**Duda:**
- ¿El `MontoConsumido` de la tabla `ConsumoSaldoAFavor` propuesta en la guía NC es exactamente `ImpPagado` del CP, o hay conversión de moneda entre ambos?
- ¿`ImpSaldoAnt` del CP con `FormaPago=23` es el saldo de la **factura futura** o el saldo de la NC (`TotalDisponibleReal`)?

**Acción propuesta:** aclarar en un cuadro cuál es la fuente de cada campo del CP cuando `FormaPago=23`:

| Campo CP | Fuente cuando `FormaPago=23` (saldo NC) | Fuente cuando `FormaPago≠23` (cobro real) |
|---|---|---|
| `ImpSaldoAnt` | Saldo anterior de la **factura futura** | idem |
| `ImpPagado` | `MontoConsumido` de `ConsumoSaldoAFavor` | Monto del cobro |
| `EquivalenciaDR` | Derivado del `TipoCambio`/`FactorUSD` de la NC | Derivado del `TipoDeCambioMonedaFacturacion` del cobro |

### D3 — Foliado del CP con `FormaPago=23`

**Contexto:** FU-030 define un foliador Serie "P" en `EmpresaFolio` para el CP.

**Duda:** el CP que consume saldo NC (`FormaPago=23`) ¿usa el mismo foliador que un CP de dinero real, o serie distinta? La guía no lo aclara.

**Acción propuesta:** confirmar con el contador si el SAT distingue por serie el origen del pago (saldo NC vs efectivo). Probablemente **misma serie** — el `FormaPago` ya diferencia.

### D4 — Numeración y orden de los CPs con `FormaPago=23`

**Contexto:** el `NumParcialidad` de FU-030 se calcula con `COUNT+UPDLOCK`. Si la factura futura recibe primero un CP con `FormaPago=23` y después uno con la forma real por remanente, ambos comparten `NumParcialidad` o se enumeran distinto.

**Duda:** ¿el CP con `FormaPago=23` es `NumParcialidad=1` y el CP con forma real es `NumParcialidad=2` (o viceversa)? ¿Importa el orden operativo?

**Acción propuesta:** validar con el contador. Recomendación técnica: emitirlos en secuencia (23 primero por convención — es el evento que se ejecuta al confirmar la factura futura; la forma real se factura después cuando llega el dinero).

---

## E. Interacciones con RE-FU-023 (cancelación distribuida)

### E1 — Cancelación de pedido con FAA prepago no entregado — ¿NC 100% o cancelación por falta de pago?

**Contexto:**
- Sección 12 de la guía: cuando el prepago ya se cobró y el proveedor no entrega, camino es **NC 100%** (no cancelación de CFDI).
- RE-FU-023: cancelación por falta de pago dispara `POST /invoices/{invoiceId}/cancel` (Timbrado) desde el orquestador Finanzas.

**Duda:** son casos operativamente distintos, pero conviene documentar la **matriz de decisión** para que el operador no confunda ambos flujos:

| Situación | Camino |
|---|---|
| Cliente NO ha pagado + producto NO entregado | Cancelación por falta de pago (RE-023) |
| Cliente SÍ pagó (prepago cobrado) + producto NO entregado | NC 100% (sección 12 guía NC) |
| Cliente SÍ pagó + producto SE entregó + devolución posterior | NC parcial o total (sección 3–7 guía NC) |
| Cliente NO ha pagado + operación cancelada por otro motivo | Cancelación operativa (`tpPedidoCancelacionController`) |

**Acción propuesta:** agregar esta matriz a la guía NC (sección 11 o 12) y a la documentación de RE-FU-023.

---

## F. Perú y alcance regional

### F1 — Falta guía equivalente para SUNAT (CPE tipo 07)

**Contexto:** la guía está marcada explícitamente como "solo México". RE-FU-033 en R16 es "Timbrar Nota de Crédito Perú" (CPE tipo 07, SUNAT).

**Duda:** ¿existe guía técnica equivalente para Perú? Sin ella, el diseño de NC Perú queda sin ancla conceptual.

**Acción propuesta:** abrir tarea de redacción de guía hermana `Guia_Tecnica_Notas_de_Credito_PE.md` cubriendo:
- Diferencias entre CFDI E (SAT) y CPE 07 (SUNAT).
- Tipos de nota de crédito SUNAT (por descuento, anulación, devolución, corrección, etc. — SUNAT tiene su propio catálogo).
- Reglas de aplicación de saldo a favor en Perú (¿aplica el mismo modelo PPD + CP, o Perú usa otro mecanismo?).
- Convivencia con `fccNotaCredito` en Finanzas (modelo compartido MX+PE o modelo por región).

---

## G. Casos de UX / captura

### G1 — Riesgo de captura incorrecta del `Motivo`

**Contexto:** `Motivo` es dropdown manual sin validación cruzada del sistema.

**Duda:**
- ¿Es obligatorio capturar un **comentario / justificación** al elegir el `Motivo`, para trazabilidad de auditoría?
- ¿Debería existir un flujo de aprobación de segundo nivel para las NCs de monto alto?

**Acción propuesta:** confirmar con el coordinador si el registro de la NC debe incluir un campo de notas obligatorio (análogo a `Notas` del cobro en RE-024) — la definición del campo específico corresponde al modelo de RE-FU-032.

### G2 — Error de precio unitario → "Descuento general"

**Contexto:** la guía obliga a capturar el error de precio unitario como "Descuento general" (4.2), no como "Producto o piezas no entregados" (4.1).

**Duda:** este mapeo es contra-intuitivo para el usuario (mentalmente el error es "de producto", no "de descuento"). ¿Cómo se comunica en la UI?

**Acción propuesta:**
- Texto de ayuda en el dropdown del `Motivo` que explique el mapeo.
- Ejemplos concretos en el tooltip.
- Considerar validación cruzada: si el usuario captura "Producto o piezas no entregados" y luego intenta editar `ValorUnitario`, bloquear con mensaje "para corrección de precio, use Descuento general".

### G3 — Tasas mixtas de IVA en modalidad 4.2

**Contexto:** la guía exige que si la factura origen tiene tasas de IVA distintas (16% y 0%, por ejemplo), el formulario debe mostrar un input por tasa.

**Duda:**
- ¿Está dimensionado el impacto de esta UI en el requisito R16 de "Generar NC" (RE-FU-032)?
- ¿Cómo se detecta que la factura origen tiene tasas mixtas — al seleccionarla, agrupar sus partidas por tasa antes de renderizar el formulario?

**Acción propuesta:** confirmar en FU-032 la existencia del componente UI de captura por tasa + su validación (no exceder el monto de esa tasa en la factura origen).

---

## H. Modelo de estatus y reportería

### H1 — Distinción `Resuelta` vs `Aplicada totalmente`

**Contexto:** la guía insiste en mantenerlos como valores independientes por trazabilidad.

**Duda:** para reportes operativos que solo quieren saber "¿está cerrada esta NC?" tener que filtrar por `EstatusAplicacionNC IN ('Resuelta', 'Aplicada totalmente')` puede ser incómodo.

**Acción propuesta:** confirmar con el coordinador si un campo/vista derivada `EstaCerrada` puede simplificar la reportería sin perder la trazabilidad — la definición concreta del campo corresponde al modelo de RE-FU-032.

### H2 — `EstatusSAT` sin sincronización automática

**Contexto:** el sistema no se entera cuando alguien cancela una NC directamente en el portal del SAT o vía PAC.

**Duda:** ¿es aceptable el riesgo operativo, o conviene un job periódico?

**Acción propuesta:** job de sincronización periódico (Hangfire, misma infraestructura que RE-023) que consulte al PAC el estado real de las NCs con `EstatusSAT='Vigente'` y las actualice si detecta cancelación externa. Frecuencia sugerida: diaria.

---

## I. Control de concurrencia — fraccionamiento del saldo NC

### I1 — Consumo concurrente del mismo saldo NC desde dos facturaciones

**Contexto:** la guía valida `SUM(MontoConsumido) <= TotalDisponibleReal`, pero no especifica el mecanismo de concurrencia.

**Duda:** si dos facturaciones simultáneas intentan consumir la misma NC, ¿cómo se garantiza la validación?

**Acción propuesta:** usar el mismo patrón `UPDLOCK` que ya se aplica en FU-030 (`NumParcialidad` en `PaymentComplementCalculationService`). Recomendación de sentencia:

```sql
BEGIN TRAN;
SELECT TotalDisponibleReal
FROM NotaCredito WITH (UPDLOCK, HOLDLOCK)
WHERE IdNotaCredito = @Id;

-- validar disponibilidad
-- si OK, INSERT ConsumoSaldoAFavor + UPDATE NotaCredito.TotalAplicado
COMMIT TRAN;
```

---

## J. Temporalidad fiscal

### J1 — Recomendación "mismo ejercicio fiscal" — sin bloqueo

**Contexto:** la guía recomienda emitir la NC en el mismo ejercicio fiscal que la factura origen, pero no bloquea.

**Duda:**
- ¿El sistema debería mostrar una alerta si se está cruzando ejercicio fiscal?
- ¿Qué implicaciones tiene fiscalmente que un saldo a favor generado en 2025 se consuma en 2026?

**Acción propuesta:**
- Confirmar con el contador si hay restricciones fiscales al cruce de ejercicio.
- Considerar alerta visual (no bloqueo) en la UI cuando la NC se emite en ejercicio distinto al de la factura origen.

---

## K. Detalles menores

### K1 — Regla del PDF desde XML timbrado

**Contexto:** hallazgo 3 de la guía documenta un bug del timbrador legado (PDF muestra `G02` pero XML tiene `G03`) y adopta la regla "PDF siempre desde XML timbrado".

**Duda:** ¿DocumentBuilder ya sigue esta regla para NC, equivalente a lo que hace `InvoicePdfMappingService` (FU-021) para facturas?

**Acción propuesta:** verificar en FU-032 (tarea de generación de PDF de NC) que `CreditNotePdfMappingService` (o equivalente) construya el PDF a partir del XML timbrado, no de un DTO cacheado.

### K2 — RFC genérico extranjero `XEXX010101000` sí admite NC

**Contexto:** validación del sistema bloquea `XAXX010101000` (público en general), pero permite `XEXX010101000` (extranjero).

**Duda:**
- ¿Hay clientes con RFC `XEXX010101000` en la BD actualmente?
- ¿Cómo interactúan con las reglas de moneda (probablemente USD, sección 5 de la guía)?

**Acción propuesta:** ejecutar query de verificación en la BD:

```sql
SELECT COUNT(*) FROM DatosFacturacionCliente WHERE Rfc = 'XEXX010101000';
```

Si hay clientes, documentar caso de prueba específico para NC contra RFC extranjero.

---

## Resumen de acciones sugeridas

| # | Acción | Responsable propuesto | Bloqueante para RE-FU-032 |
|---|---|---|---|
| A1 | Confirmar reparto de responsabilidades del flujo NC entre RE-FU-028 y RE-FU-032 | Arquitectura + Coordinador | Sí |
| A2 | Mapeo `IdCatCondicionesDePago` → string CFDI en mapping service | Backend Finanzas | No |
| B1 | Decidir cancelación NC (fuera vs endpoint) | Arquitectura + Cliente | Sí |
| C1, C2, C3 | Cuadro comparativo OBS-049/050/051/052 vs NC | Arquitectura | Sí |
| D1, D2, D3, D4 | Extender FU-030 con caso `FormaPago=23` desde NC | Backend Finanzas | Sí |
| E1 | Matriz de decisión NC 100% vs cancelación por falta de pago | Cliente + Producto | No |
| F1 | Guía hermana Perú (SUNAT CPE 07) | Contador Perú + Arquitectura | Sí (para RE-FU-033) |
| G1 | Campo `NotasNotaCredito` obligatorio | Backend Finanzas | No |
| G2 | Texto de ayuda en UI (mapeo error precio → descuento) | Producto + Frontend | No |
| G3 | Componente UI de captura por tasa | Frontend | Sí |
| H1 | Campo derivado `EstaCerrada` | Backend Finanzas | No |
| H2 | Job Hangfire de sincronización `EstatusSAT` | Backend Finanzas | No |
| I1 | Patrón `UPDLOCK` para consumo concurrente | Backend Finanzas | Sí |
| J1 | Alerta cruce de ejercicio fiscal | Producto + Cliente | No |
| K1 | Verificar `CreditNotePdfMappingService` desde XML | Backend Finanzas | No |
| K2 | Query verificación RFC extranjero + caso de prueba | QA | No |

---

## Referencias

- `Analisis/Facturación/Guia_Tecnica_Notas_de_Credito_MX.md` — guía origen.
- `Requisitos/R16A-RE-FU-024/R16A-RE-FU-024-Back.md` — OBS-049/050/051/052.
- `Requisitos/R16A-RE-FU-030/DIS-SOL-backend-complemento-pago.md` — Complemento de Pago.
- `Requisitos/R16A-RE-FU-023/[R16A-RE-FU-023][DIS-SOL] Diseño de la solución.md` — cancelación distribuida.
- `Endpoints/Endpoints-Timbrado.md` — endpoints de cancelación (`stamp/cancel`, `invoices/{id}/cancel`).
- `Endpoints/Endpoints-Finanzas.md` — RE-032/033 timbrado de NC.
