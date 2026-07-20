# Guía Técnica — Facturación Perú (borrador de trabajo)

**Estado:** documento en construcción — captura lo ya definido en conversación para no perder información, mientras se decide si la facturación de Perú entra o no al alcance del proyecto. No tiene todavía el nivel de pulido de las guías de México (Notas de Crédito, Facturas de Ingreso).

---

## 0. Alcance — pendiente de decisión

**Aún no está decidido si la facturación de Perú se construye en ProquifaNet 2.** El volumen de operación en Perú es bajo comparado con México, y la complejidad de la normativa (Retención, Percepción, Detracción, catálogos propios) puede no justificar el esfuerzo — esto está en discusión, no cerrado.

---

## 1. Catálogos ya definidos

### 1.1 Régimen Tributario del cliente

Los 4 regímenes reales de Perú, usados como **código de negocio** (siglas, no numeración inventada):

| Código | Régimen |
|---|---|
| `NRUS` | Nuevo Régimen Único Simplificado |
| `RER` | Régimen Especial de Renta |
| `RMT` | Régimen MYPE Tributario |
| `RG` | Régimen General |

**Decisión ya tomada:** no existe un catálogo oficial de SUNAT que numere estos regímenes para efecto del receptor en el XML (UBL 2.1 no lo exige) — la codificación interna usa las siglas reales (`NRUS`/`RER`/`RMT`/`RG`) como *natural key*, no una numeración `01`-`04` inventada por PROQUIFA. Se maneja como `RegimenTributarioId` interno (surrogate key para FK) + columna `Code` con la sigla.

### 1.2 `TipoAfectacionIGV` (Catálogo 07 SUNAT) — equivalente peruano de `PerfilFiscal`

A diferencia de México (que necesita 3 catálogos SAT base + `PerfilFiscal`), Perú resuelve el tratamiento de IGV con **un solo catálogo** de SUNAT, más simple. Catálogo completo (18 valores oficiales):

| Código | Descripción                                 | ¿Relevante para PROQUIFA?                                                                                                                                   |
| ------ | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `10`   | Gravado — Operación Onerosa                 | **Sí — es el caso normal**, venta comercial estándar al 18% IGV                                                                                             |
| `11`   | Gravado — Retiro por premio                 | No aplica a venta comercial                                                                                                                                 |
| `12`   | Gravado — Retiro por donación               | No aplica a venta comercial                                                                                                                                 |
| `13`   | Gravado — Retiro                            | No aplica a venta comercial                                                                                                                                 |
| `14`   | Gravado — Retiro por publicidad             | No aplica a venta comercial                                                                                                                                 |
| `15`   | Gravado — Bonificaciones                    | No aplica a venta comercial                                                                                                                                 |
| `16`   | Gravado — Retiro por entrega a trabajadores | No aplica a venta comercial                                                                                                                                 |
| `17`   | Gravado — IVAP                              | No — exclusivo de arroz pilado                                                                                                                              |
| `20`   | Exonerado — Operación Onerosa               | **Posible** — si algún producto de PROQUIFA está en el Apéndice I/II de la Ley del IGV                                                                      |
| `21`   | Exonerado — Transferencia Gratuita          | No aplica a venta comercial                                                                                                                                 |
| `30`   | Inafecto — Operación Onerosa                | **Posible** — operaciones fuera del ámbito de aplicación del IGV                                                                                            |
| `31`   | Inafecto — Retiro por Bonificación          | No aplica a venta comercial                                                                                                                                 |
| `32`   | Inafecto — Retiro                           | No aplica a venta comercial                                                                                                                                 |
| `33`   | Inafecto — Retiro por Muestras Médicas      | **Posible caso especial** — si PROQUIFA entrega muestras médicas sin venderlas, este código aplica específicamente a ese supuesto (no es una venta onerosa) |
| `34`   | Inafecto — Retiro por Convenio Colectivo    | No aplica a venta comercial                                                                                                                                 |
| `35`   | Inafecto — Retiro por premio                | No aplica a venta comercial                                                                                                                                 |
| `36`   | Inafecto — Retiro por publicidad            | No aplica a venta comercial                                                                                                                                 |
| `40`   | Exportación                                 | Se resuelve a nivel operación (sección 1.4), no a nivel producto                                                                                            |

Confirmado con evidencia real (factura Golocaer analizada): el código `10` (Gravado) es el que aparece en operaciones normales gravadas con 18% IGV — es, por mucho, el caso más común en la operación de PROQUIFA.

**Pendiente de confirmar con el cliente:** si PROQUIFA entrega muestras médicas sin cargo (código `33`), y si algún producto de su catálogo está exonerado o inafecto (códigos `20`/`30`) — sin esa confirmación, se asume `10` (Gravado) como default para todo el catálogo.

### 1.3 Código de Producto SUNAT (Catálogo 25) y Unidad de Medida (Catálogo 03)

**Hallazgo real, no resuelto:** en la factura real de Golocaer analizada, el campo oficial de Código de Producto SUNAT venía **vacío** — solo se llenaba `SellersItemIdentification` (código interno del vendedor), y ese código se repetía idéntico (`41116107`) en las 16 líneas de una guía de remisión, para productos completamente distintos. Esto es un defecto de captura del sistema legado, no un patrón a replicar.

Unidad de medida: se confirmó el uso de `C62` (Unidad) en las facturas peruanas reales, basado en el mismo estándar internacional UN/ECE Recomendación 20 que usa México (`c_ClaveUnidad`) — existe la oportunidad de compartir una sola tabla maestra UN/ECE entre ambos países, con una columna de "país donde es válido cada código", en vez de 2 tablas separadas.

### 1.4 Tipo de Operación (Catálogo 17 SUNAT)

Equivalente peruano del campo `Exportacion` de México — determina si la operación es venta interna, exportación, etc. Mismo nivel que `Exportacion` en México (a nivel pedido/operación, no a nivel producto).

### 1.5 Nota de Crédito / Nota de Débito — catálogos de motivo

| Nota de Crédito (Catálogo 09)               | Nota de Débito (Catálogo 10)               |
| ------------------------------------------- | ------------------------------------------ |
| `01` Anulación de la operación              | `01` Intereses por mora                    |
| `02` Anulación por error en el RUC          | `02` Aumento en el valor                   |
| `03` Corrección por error en la descripción | `03` Penalidades/otros conceptos           |
| `04` Descuento global                       | `11` Ajustes de operaciones de exportación |
| `05` Descuento por ítem                     | `12` Ajustes afectos al IVAP               |
| `06` Devolución total                       |                                            |
| `07` Devolución por ítem                    |                                            |
| `08` Bonificación                           |                                            |
| `09` Disminución en el valor                |                                            |
| `10` Otros Conceptos                        |                                            |
| `11` Ajustes de operaciones de exportación  |                                            |
| `12` Ajustes afectos al IVAP                |                                            |

**Decisión ya tomada:** a diferencia de México (`TipoRelacion=01` para descuento vs `03` para devolución), en Perú **la Nota de Crédito siempre debe referenciar el comprobante original** (no puede existir suelta) — el `DiscrepancyResponse` es obligatorio, con el número de la factura/boleta que corrige.

**Validado:** ¿PROQUIFA emite Notas de Crédito en Perú hoy? — pendiente de confirmar con el cliente si aplica en su operación actual, o si se deja fuera de alcance hasta que aparezca un caso real.

---

## 2. Detracción (SPOT) — excluida del alcance, confirmado con el cliente

**El cliente confirmó que la Detracción no aplica a su operación** — no se considera en el sistema. Esto ya está cerrado, no es una duda abierta.

---

## 3. Retención del IGV

**Pregunta al cliente (pendiente):** ¿algún cliente de la cartera Perú de PROQUIFA está designado por SUNAT como Agente de Retención? (Verificable en el Padrón de Agentes de Retención). Ver documento de preguntas al cliente.

**Reglas ya verificadas (aplican solo si la respuesta es "sí" para algún cliente):**
- Tasa fija por ley: **3%** sobre el importe total de la operación (incluye IGV) — no varía por producto ni por empresa emisora de PROQUIFA.
- Umbral de exclusión: operaciones cuyo importe sea igual o menor a **S/700** no están sujetas a retención.
- El umbral se evalúa de forma **acumulada por cliente + fecha de pago**, no por comprobante individual — si varios comprobantes menores a S/700 se pagan el mismo día y la suma supera ese monto, sí aplica retención sobre el total.
- Se calcula en el momento del **pago** (Cobro), no de la emisión de la factura.
- En pagos parciales, la tasa se aplica sobre el importe de cada pago, no sobre el total de la factura de una sola vez.
- Efecto en el Cobro: el monto que efectivamente se recibe es **menor** al total facturado (el cliente retiene una parte y la entrega directo a SUNAT) — el sistema necesita registrar el monto retenido como dato aparte, no solo como una diferencia sin explicar.

---

## 4. Percepción del IGV

**Pregunta al cliente (pendiente, ver documento de preguntas al cliente):**
1. ¿Golocaer S.A.C. está designada por SUNAT como Agente de Percepción? (Verificable en el Padrón de Agentes de Percepción).
2. Si sí, ¿cuáles de sus productos están sujetos a esta figura, y qué tasa aplica a cada uno? **Se pide al cliente, no se cruza internamente** — el listado oficial de SUNAT (Apéndice 1 de la Ley 29173) es un catálogo técnico extenso y la tasa depende también de una condición del cliente comprador; el riesgo de que el equipo lo interprete mal por su cuenta es real, así que se confirma directamente con PROQUIFA (idealmente validado con su asesor contable).

**Reglas ya verificadas (aplican solo si la respuesta es "sí"):**
- **No aplica a todos los productos** — solo a los que el cliente confirme, según el Apéndice 1 de la Ley 29173.
- Tasa: **2%** sobre el precio de venta en el caso general; **0.5%** si el cliente comprador también es Agente de Percepción/Retención y la operación da derecho a crédito fiscal — el cliente debe indicar cuál aplica a cada producto/caso, no se asume por default.
- Se calcula en el momento del **cobro** (total o parcial), igual que la Retención.
- Exclusiones a validar: cliente ya es Agente de Retención, cliente en listado de entidades exceptuadas, o "consumidor final" persona natural comprando ≤ S/1,500 (poco probable dado que PROQUIFA opera B2B).
- Efecto en el Cobro: es el **inverso** de la Retención — Golocaer cobra un monto **adicional** al cliente, encima del precio de venta. El Cobro recibido es **mayor** al total facturado, no menor.

**Decisión de arquitectura:** el sistema **no determina automáticamente** qué productos aplican — es un campo configurable en el Perfil Fiscal de Perú por producto/familia (`SujetoPercepcion` + tasa), poblado con base en lo que el cliente confirme, igual que cualquier otro dato de `PerfilFiscal` en México.

---

## 5. Configuración fiscal de productos — Perfil Fiscal Perú

### 5.1 Los 2 datos que se configuran por producto — no son lo mismo, no se combinan

| Atributo                                                     | Depende de                                                                                                       | ¿Se captura por producto? |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- | ------------------------- |
| Tratamiento de IGV (`TipoAfectacionIGVCode`, `TasaIGV`)      | El producto en sí — equivalente a `PerfilFiscal` de México                                                       | **Sí**                    |
| ¿Sujeto a Percepción? (`SujetoPercepcion`, `TasaPercepcion`) | El producto específico (Apéndice 1 Ley 29173) + la designación de Golocaer como Agente de Percepción (sección 4) | **Sí**                    |

**La Retención del IGV (sección 3) NO se configura por producto — no debe confundirse con lo anterior.** Depende únicamente de si el **cliente comprador** está designado por SUNAT como Agente de Retención — es un dato de la ficha del cliente, nunca del catálogo de productos. Un mismo producto no tiene "retención sí/no" — lo que tiene esa condición es la operación, según quién compra.

**La Detracción no se configura en absoluto** — excluida del alcance, confirmado con el cliente (sección 2).

### 5.2 Estructura de la tabla

```sql
CREATE TABLE PerfilFiscalPeruProducto (
    ProductoId INT PRIMARY KEY,
    TipoAfectacionIGVCode CHAR(2) NOT NULL REFERENCES SatTipoAfectacionIGV(Code),  -- Catálogo 07, sección 1.2
    TasaIGV DECIMAL(5,2) NULL,          -- 18.00 si Gravado; NULL si Exonerado/Inafecto
    SujetoPercepcion BIT NOT NULL DEFAULT 0,
    TasaPercepcion DECIMAL(5,2) NULL,   -- 2.00 (general) o 0.50 (caso especial, sección 4); NULL si SujetoPercepcion=0
    CONSTRAINT CK_TasaIGV_Gravado CHECK (
        (TipoAfectacionIGVCode = '10' AND TasaIGV IS NOT NULL) OR
        (TipoAfectacionIGVCode <> '10' AND TasaIGV IS NULL)
    ),
    CONSTRAINT CK_TasaPercepcion CHECK (
        (SujetoPercepcion = 0 AND TasaPercepcion IS NULL) OR
        (SujetoPercepcion = 1 AND TasaPercepcion IS NOT NULL)
    )
);
-- + tabla equivalente a nivel Familia, con la misma precedencia
-- Producto → Familia ya usada para PerfilFiscal en México
```

### 5.3 Precedencia Producto → Familia

```
SI el producto tiene su propia configuración capturada (override) → se usa esa
SI NO → se hereda la configuración de su Familia
```

Mismo mecanismo ya usado en México — permite definir "toda la Familia X es Gravado 18%, sin Percepción" una sola vez, y solo capturar una excepción puntual si un producto específico de esa familia se comporta distinto (ej. si acaba resultando que un producto puntual sí está en el Apéndice 1 de Percepción, aunque el resto de su familia no).

**Pendiente:** confirmar si, igual que en México, se necesita definir a qué nivel (Producto o Familia) vive cada uno de los 2 atributos — podrían no compartir el mismo nivel entre sí.

### 5.4 Ejemplos de configuración

| Producto | `TipoAfectacionIGVCode` | `TasaIGV` | `SujetoPercepcion` | `TasaPercepcion` | Caso |
|---|---|---|---|---|---|
| Reactivo químico estándar | `10` (Gravado) | 18.00 | 0 | NULL | Caso normal — la mayoría del catálogo |
| Producto exonerado (si aplica, Apéndice I/II Ley IGV) | `20` (Exonerado) | NULL | 0 | NULL | Pendiente confirmar si algún producto de PROQUIFA cae aquí |
| Reactivo sujeto a Percepción (si Golocaer tiene la designación y el producto está en Apéndice 1) | `10` (Gravado) | 18.00 | 1 | 2.00 | El IGV y la Percepción son independientes — un producto gravado normal puede, además, estar sujeto a Percepción |
| Muestra médica entregada sin cargo | `33` (Inafecto — Retiro por Muestras Médicas) | NULL | 0 | NULL | Caso especial, no es una venta onerosa — pendiente confirmar si PROQUIFA maneja este supuesto |

**Nota sobre el 3er ejemplo:** IGV y Percepción son 2 columnas independientes de la misma fila — un producto puede estar gravado al 18% **y** sujeto a Percepción al mismo tiempo; no son mutuamente excluyentes, son 2 preguntas distintas que se responden por separado para el mismo producto.

---

## 6. Validar Cobro en Perú — ya resuelto (FU-029)

**Confirmado:** en Perú no existe el Complemento de Pago (documento mexicano que ampara un pago posterior a una factura). El IGV nace en el momento de la entrega del bien o la emisión del comprobante, lo que ocurra primero (Art. 4 Ley IGV) — el pago posterior no tiene efecto fiscal.

**Resolución:** cuando llega un cobro contra una factura peruana ya emitida, el sistema solo hace **conciliación interna** (marcar la factura como cobrada, con monto y fecha) — no se genera ningún documento fiscal adicional. Opcionalmente, una constancia de cobro no fiscal (sin folio SUNAT) como cortesía al cliente, si el negocio lo desea.

---

## 7. Hallazgos de la factura real de Golocaer analizada (referencia)

- `Exportacion` en `01` (No aplica) en una venta a comprador nacional — consistente, correcto en ese caso.
- Forma de pago: crédito — confirmado que la factura peruana no exige el mismo tratamiento PUE/PPD que México.
- Ver sección 1.3 para el hallazgo del código de producto SUNAT vacío.

---

## 8. Pendientes generales

- Decisión de alcance (sección 0) — la más importante, condiciona si el resto de este documento se llega a construir.
- Confirmar con el cliente si algún producto de PROQUIFA está Exonerado o Inafecto (códigos `20`/`30`), o si entregan muestras médicas sin cargo (código `33`) — sección 1.2.
- Recibir del cliente la confirmación de qué productos están sujetos a Percepción y con qué tasa (sección 4) — no se cruza el Apéndice 1 internamente.
- Confirmar si `TipoAfectacionIGVCode` y `SujetoPercepcion` se configuran al mismo nivel (Producto o Familia) o pueden diferir entre sí (sección 5.3).
- Definir foliador/serie para comprobantes peruanos (mismo tema pendiente que en México).
- Confirmar con el cliente las 2 preguntas de Retención/Percepción (documento de preguntas al cliente).
