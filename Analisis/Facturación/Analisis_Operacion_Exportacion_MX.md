# Análisis y Definición de Operación de Exportación e Importación (México)

**Objetivo:** entender, directamente de PROQUIFA y su equipo de finanzas, cómo opera su cadena de importación y exportación entre razones sociales y hacia clientes finales, y qué tratamiento fiscal corresponde a cada caso.

**Alcance.** Aplica a las razones sociales mexicanas del grupo (Proveedora, Proquifa, Golocaer, Mungen). Pharma Scientific Inc. no genera CFDI ni está sujeta a la LIVA mexicana — queda fuera del tratamiento fiscal de este documento, aunque su interacción con las razones sociales mexicanas sí se pregunta en la sección 3.6.

Cada razón social del grupo puede facturar directo a sus propios clientes finales, de Crédito y de Prepago — en PROQUIFA se le conoce como "Frente Comercial". **Confirmado (verificado en base de datos):** las 4 razones sociales (Proveedora, Proquifa, Golocaer, Mungen) tienen su propia cartera de clientes de Crédito y Prepago, y las 4 han tramitado pedidos de ambos tipos durante este año.

---

## 1. Marco legal

- **Art. 1 y 10 LIVA:** la LIVA solo aplica si el acto se realiza en territorio nacional.
- **Art. 29 LIVA:** la exportación exige carácter de definitiva en términos de la Ley Aduanera — un pedimento de exportación real.
- **Art. 29-A fracción VIII CFF:** en la primera venta de mercancía importada, se debe citar el número y fecha del pedimento de importación.
- **Complemento de Comercio Exterior:** obligatorio en operaciones de exportación definitiva con clave de pedimento "A1" que impliquen enajenación — el CFDI se timbra primero, y el pedimento (generado después) declara el folio fiscal de esa factura.

---

## 2. Escenarios de operación

| # | Compra | Venta | Ingresa / Sale de México | Perfil Fiscal que aplica | ¿Pedimento de importación en la factura? | ¿Pedimento de exportación en la factura? | ¿Se puede facturar sin problema en Prepago/Crédito/Contra Entrega por adelantado? | Otros impedimentos o dependencias | Comentarios |
|---|---|---|---|---|---|---|---|---|---|
| **1** | Local | Local | No / No | Eje 1 puro — 16% / 0% / Exento, según el producto | No | No | Sí, sin problema | No se puede confirmar que la compra será local hasta enviar la OC al proveedor — no se sabe todavía si esta operación es realmente Escenario 1 o terminará siendo Escenario 2 | Venta doméstica normal |
| **2** | EUA | Local | Sí / No | Eje 1 puro — 16% / 0% / Exento (importar no cambia la tasa) | Sí (Art. 29-A fr. VIII CFF) | No | Impedimento — el pedimento de importación no existe aún al momento de facturar, porque los 3 facturan antes de importar el producto (incluso antes de siquiera enviar la Orden de Compra al proveedor) | Además del impedimento del pedimento, tampoco se puede confirmar que la compra será en EUA hasta enviar la OC — no se sabe todavía si esta operación es Escenario 1 o 2 | Mismo problema estructural ya identificado en la guía de Facturas, sección 5.5 |
| **3** | Local | Extranjero (sin importación previa) | No / Sí | Eje 2 fuerza tasa 0% (Art. 29 LIVA) | No | Sí (se genera después, citando la factura) | Sí, sin problema | No se puede confirmar que la compra será local hasta enviar la OC — no se sabe todavía si esta operación es Escenario 3 o terminará siendo Escenario 4 | Requiere Complemento de Comercio Exterior |
| **4** | EUA | Extranjero, pasando por México | Sí / Sí | Eje 2 fuerza tasa 0% (Art. 29 LIVA) | Por default, sí (alternativa posible: podría no requerirse, pendiente de confirmar con especialista) | Sí (mismo mecanismo que el 3) | Impedimento, mientras no se confirme la alternativa — mismo motivo que el Escenario 2 | Además del impedimento del pedimento, tampoco se puede confirmar el origen de la compra ni si el producto pasará por México — no se sabe todavía si esta operación es Escenario 3, 4 o 5 | Requiere Complemento de Comercio Exterior; si se confirma que no se requiere citar el pedimento de importación aquí, ese impedimento desaparece |
| **5** | EUA | Extranjero, sin pasar por México | No / No | `ObjetoImp=01` — No objeto de la LIVA | No | No | Sí, sin problema | No se puede confirmar el origen de la compra ni la ruta logística (si pasará o no por México) hasta que exista un plan de envío real — no se sabe todavía si esta operación es Escenario 4 o 5 | Ningún pedimento que esperar |

---

## 3. Preguntas para la sesión

### 3.1 Estructura de importación

- ¿Todas las razones sociales del grupo (Proveedora, Proquifa, Golocaer, Mungen) importan mercancía directo de proveedores extranjeros, o hay alguna que concentra esa función y las demás le compran a ella?
- Para una razón social dada, ¿el origen de compra (local vs. extranjero) es predecible de antemano — fijo por razón social, fijo por producto (como ya ocurre con el Perfil Fiscal) — o se decide caso por caso al momento de comprarle a un proveedor específico, según disponibilidad o precio? Si es predecible, se puede resolver el problema de identificación de la tabla anterior con un dato ya conocido; si no, hay que diseñar el sistema asumiendo esa incertidumbre.

### 3.2 Manejo actual del pedimento de importación

- Cuando una razón social importa mercancía y luego la vende (directamente a un cliente final, o a otra razón social del grupo), ¿en qué documento se cita hoy el número de pedimento de importación?
- Específicamente para clientes de Crédito y Prepago con factura por adelantado: ¿el pedimento de importación se cita en esa factura? Si no, ¿por qué — es una decisión consciente basada en algún criterio, o es algo que el sistema actual no contempla?
- ¿Ha habido alguna revisión o criterio de su asesor fiscal sobre este punto específico?

### 3.3 Escenarios reales de venta

- ¿Cuáles de los 5 escenarios de la sección 2 ocurren en la operación real de PROQUIFA, y con qué frecuencia?
- ¿Cuáles razones sociales participan en cada uno?
- Para las ventas a clientes en el extranjero (Escenarios 3, 4 o 5): ¿cómo se determina, en cada caso, si el producto pasa por México antes de llegar al cliente, o va directo desde el proveedor extranjero?

### 3.4 Impacto en las tasas aplicadas

- Al momento de facturar por adelantado (antes de comprarle al proveedor), ¿cómo deciden qué tasa de IVA aplicar, si todavía no se sabe si la venta terminará siendo doméstica, de exportación, o fuera del objeto de la LIVA?
- ¿Ha ocurrido algún caso donde la tasa aplicada en la factura anticipada resultó ser incorrecta una vez que se conoció el destino real de la mercancía? ¿Cómo se corrigió?

### 3.5 Comercio exterior

- ¿PROQUIFA utiliza hoy el Complemento de Comercio Exterior en sus operaciones de exportación?
- Si no lo utilizan, ¿cómo documentan ante el SAT sus ventas de exportación?
- ¿Con qué frecuencia ocurren ventas de exportación en la operación actual?

### 3.6 Pharma Scientific Inc.

- ¿Con qué frecuencia una venta de Pharma involucra a alguna razón social mexicana (ej. Mungen) como parte de la cadena hacia el cliente final, y con qué frecuencia Pharma factura directo a sus propios clientes, sin ninguna razón social mexicana de por medio?
- Cuando sí involucra a una razón social mexicana, ¿esa razón social emite un CFDI en esa operación?

### 3.7 Ejemplos reales

- ¿Nos pueden compartir ejemplos reales de facturas de Prepago y de Crédito/Contra Entrega con factura por adelantado, que involucren mercancía importada o ventas a clientes extranjeros — de más de una razón social si es posible, para poder observar si el manejo es consistente entre todas?

---

## 4. Impacto en el modelo de datos (referencia)

Las respuestas de esta sesión determinan el diseño del Eje 2 (`ExportacionCode`) y el campo `ObjetoImp` ya definidos en la guía de Perfil Fiscal del IVA, así como si el Complemento de Comercio Exterior entra al alcance de ProquifaNet 2. No se construye ninguna lógica condicional sobre este tema hasta tener las respuestas de esta sesión.
