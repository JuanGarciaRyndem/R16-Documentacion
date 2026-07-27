# Guía Técnica — Perfil Fiscal del IVA (México)

---

## 0. Glosario

| Término | Qué es |
|---|---|
| **LIVA** | Ley del Impuesto al Valor Agregado — determina cuándo, cómo y a qué tasa se cobra IVA en México |
| **Tasa** | Porcentaje de IVA que se traslada en una operación — en México: 16%, 8% (zona fronteriza) o 0% |
| **Exento** | La operación no causa IVA — no se traslada ningún monto, ni siquiera al 0% |
| **Objeto de impuesto** | Si una operación está dentro del ámbito que regula la LIVA — una operación puede no ser "objeto" del impuesto en absoluto |
| **Pedimento** | Trámite aduanal que ampara la importación o exportación de mercancía |
| **RFC genérico** | RFC estándar para receptores sin RFC mexicano (`XAXX010101000` público en general nacional, `XEXX010101000` extranjero) |
| **Perfil Fiscal** | El conjunto de reglas que determina el tratamiento de IVA de un producto en una operación dada |

---

## 1. Qué determina el tratamiento de IVA de una operación

El tratamiento de IVA de una línea facturada es el resultado de combinar **2 ejes independientes** — nunca uno solo, y nunca se resuelven con un único valor por producto:

| Eje | Qué responde | Dónde vive el dato |
|---|---|---|
| **Eje 1 — Naturaleza del producto** | ¿Qué es lo que se vende? | Catálogo `PerfilFiscal`, asignado por producto/familia |
| **Eje 2 — Destino de la operación** | ¿A dónde va la mercancía? | Campo `ExportacionCode`, a nivel Pedido |

El Eje 2, cuando aplica, **sobre-escribe** el resultado del Eje 1 — nunca se combinan como si fueran opciones alternativas; el Eje 2 fuerza tasa 0% sin importar lo que indique el Eje 1.

Existe un tercer factor legal (zona geográfica del emisor, tasa 8% fronteriza) que se documenta en la sección 3.3, sin estar activo en la configuración actual.

---

## 2. Marco legal

| Factor | Artículo LIVA | Efecto |
|---|---|---|
| Naturaleza del producto/servicio | Art. 1 (general) / Art. 2-A (tasa 0%) / Art. 9 (exento) | Tasa base: 16% / 0% / Exento |
| Destino de la operación | Art. 29 | Exportación real (con pedimento) fuerza tasa 0% |
| Ubicación del emisor | Decreto de Estímulos Región Fronteriza | Zona fronteriza: tasa 8% en vez de 16% |

No existe un cuarto factor legal. Cualquier variación en la tasa que no se derive de estos 3 puntos no corresponde a una regla del IVA.

### 2.1 Casos del Artículo 2-A (tasa 0%) relevantes al catálogo de productos

- Publicaciones (libros, periódicos, revistas) — inciso i).
- Medicinas de patente terminadas — inciso b). *(No aplica a materias primas/reactivos, que tributan al 16% general — pendiente de confirmar con el cliente si el catálogo de productos incluye medicinas de patente terminadas además de reactivos.)*
- Exportación de bienes — fracción I inciso d) (este caso corresponde al Eje 2, no es una propiedad fija del producto).

---

## 3. Eje 1 — Perfil Fiscal del producto

### 3.1 Catálogos base del SAT

```sql
CREATE TABLE SatImpuesto (
    Code CHAR(3) PRIMARY KEY,        -- '001' ISR, '002' IVA, '003' IEPS
    Description NVARCHAR(50) NOT NULL
);

CREATE TABLE SatTipoFactor (
    Code VARCHAR(10) PRIMARY KEY,    -- 'Tasa', 'Cuota', 'Exento'
    Description NVARCHAR(50) NOT NULL
);

CREATE TABLE SatObjetoImp (
    Code CHAR(2) PRIMARY KEY,        -- '01' No objeto, '02' Sí objeto,
                                      -- '03' Sí objeto y no obligado al desglose,
                                      -- '04' Sí objeto y no causa impuesto
    Description NVARCHAR(100) NOT NULL
);
```

Estas 3 tablas son catálogo maestro de sistema — se cargan una sola vez y solo se actualizan si el SAT deroga o agrega una clave.

### 3.2 Catálogo `PerfilFiscal`

```sql
CREATE TABLE PerfilFiscal (
    PerfilFiscalId INT IDENTITY(1,1) PRIMARY KEY,
    Nombre NVARCHAR(100) NOT NULL,
    SatImpuestoCode CHAR(3) NOT NULL REFERENCES SatImpuesto(Code),
    SatTipoFactorCode VARCHAR(10) NOT NULL REFERENCES SatTipoFactor(Code),
    SatObjetoImpCode CHAR(2) NOT NULL REFERENCES SatObjetoImp(Code),
    TasaOCuota DECIMAL(6,6) NULL,     -- NULL únicamente si SatTipoFactorCode = 'Exento'
    Fundamento NVARCHAR(200) NULL,
    CONSTRAINT CK_TasaOCuota_Exento CHECK (
        (SatTipoFactorCode = 'Exento' AND TasaOCuota IS NULL) OR
        (SatTipoFactorCode <> 'Exento' AND TasaOCuota IS NOT NULL)
    )
);
```

| `PerfilFiscalId` | Nombre          | `TasaOCuota` | `SatTipoFactorCode` | `SatObjetoImpCode` | Fundamento    |
| ---------------- | --------------- | ------------ | ------------------- | ------------------ | ------------- |
| 1                | IVA General 16% | 0.160000     | Tasa                | 02                 | Art. 1 LIVA   |
| 2                | IVA Tasa 0%     | 0.000000     | Tasa                | 02                 | Art. 2-A LIVA |
| 3                | Exento          | NULL         | Exento              | 02                 | Art. 9 LIVA   |

Una cuarta fila (IEPS) se agrega únicamente si el cliente confirma que algún producto lo requiere.

### 3.3 Nivel de asignación — Producto o Familia, con precedencia

```
SI el producto tiene su propio PerfilFiscal capturado (override) → se usa ese
SI NO → se hereda el PerfilFiscal configurado a nivel de su Familia
```

**Pendiente de confirmar con el cliente:** si `ClaveProdServ`, `ClaveUnidad` y `PerfilFiscal` comparten el mismo nivel de asignación (Producto vs. Familia), o si alguno debe capturarse siempre a nivel Producto por naturaleza más específica (ver guía de Facturas, sección 5.1.3).

### 3.4 Zona fronteriza (tasa 8%) — factor legal existente, sin configuración activa

Ningún domicilio de las razones sociales de PROQUIFA corresponde a zona fronteriza. Este factor no tiene representación en el modelo de datos actual; se documenta para que no se descarte sin registro si algún domicilio futuro cambia esta condición.

---

## 4. Eje 2 — Destino de la operación (Exportación)

```sql
CREATE TABLE SatExportacion (
    Code CHAR(2) PRIMARY KEY,
    Description NVARCHAR(100) NOT NULL
);
-- 01 No aplica / 02 Definitiva / 03 Temporal / 04 Definitiva sin clave A1

ALTER TABLE Pedido ADD ExportacionCode CHAR(2) NOT NULL
    REFERENCES SatExportacion(Code) DEFAULT '01';
```

Este dato vive en el **Pedido**, nunca en el producto — la misma partida puede facturarse al 16% en una venta doméstica y al 0% en una exportación, dependiendo de la operación, no del producto.

### 4.1 Las 3 rutas físicas posibles, y su resultado fiscal

Que el cliente tenga domicilio en el extranjero no determina, por sí solo, el resultado de este eje. Lo que determina el tratamiento es la **ruta física real de la mercancía**:

| Ruta | Descripción | `ExportacionCode` / `ObjetoImp` | Tasa | Fundamento |
|---|---|---|---|---|
| **A — Triangulación sin tocar México** | El producto se compra en el extranjero y se envía directo a otro país, sin entrar nunca a territorio mexicano | `ObjetoImp=01` (No objeto) | No aplica IVA — la operación no es objeto de la LIVA | Art. 1 y 10 LIVA — el acto no ocurre en territorio nacional |
| **B — Exportación real** | El producto entra a México (pedimento de **importación**) y después se reexporta al cliente con pedimento de **exportación** | `ExportacionCode=02` | 0% | Art. 29 LIVA |
| **C — Envío sin trámite formal** | El producto entra a México y se envía al extranjero sin pedimento de exportación | Ninguno de los anteriores cumple | Riesgo de incumplimiento | No satisface el requisito de "exportación definitiva" del Art. 29 |

**Nota sobre los 2 pedimentos de la Ruta B — no confundir uno con otro:** el pedimento de **importación** (cuando PROQUIFA recibe la mercancía de su proveedor extranjero) es el que exige el Art. 29-A CFF citar en la factura de reventa (guía de Facturas, sección 5.5) — es un trámite distinto y anterior al pedimento de **exportación** (cuando la mercancía sale de México hacia el cliente final), que es el que activa la tasa 0% de este Eje 2. En una operación de Ruta B, **ambos pedimentos existen y ambos importan**, en 2 momentos distintos del proceso.

### 4.2 Cotización y Pedido reflejan la mejor información disponible — no necesariamente el resultado final

Cotización y Pedido son documentos comerciales, no fiscales — no tienen obligación ante el SAT de anticipar un tratamiento que todavía no es un hecho. Es correcto (y esperado) que muestren el resultado del Eje 1 (tasa base del producto) mientras la ruta física no se ha confirmado, y que la Factura, generada después, refleje el tratamiento final una vez que el pedimento de exportación (Ruta B) ya es real:

```
Cotización: tasa del Eje 1 (ej. 16%) — mejor información disponible en ese momento
Pedido:     misma tasa — mismo motivo
Factura:    tasa final (0% si se confirma Ruta B) — el pedimento de exportación ya existe
```

### 4.3 Regla de resolución en la Factura

Si `ExportacionCode` indica una exportación real (Ruta B), el resultado del Eje 1 se descarta y la Factura se timbra con `TasaOCuota=0`, `SatObjetoImpCode` y `SatTipoFactorCode` correspondientes. Si la operación es Ruta A, se declara `ObjetoImp=01` directamente, sin pasar por el cálculo de tasa del Eje 1.

### 4.4 El pedimento de exportación — el orden es el inverso al del pedimento de importación

Para la reventa doméstica de mercancía importada (Art. 29-A fracción VIII CFF), se cita en la factura el número del pedimento de **importación** que ya existe — este dato solo se conoce después de que PROQUIFA ya recibió la mercancía de su proveedor.

**Para una exportación real (Ruta B), el mecanismo es distinto y el orden se invierte.** No se cita en la factura un pedimento de exportación preexistente — el mecanismo aplicable es el **Complemento de Comercio Exterior** (obligatorio para exportaciones definitivas con clave de pedimento "A1" que impliquen enajenación). La secuencia real, verificada contra la guía de llenado del SAT, es: **primero se timbra el CFDI con este Complemento, y es el pedimento de exportación (generado después) el que debe declarar el folio fiscal de esa factura** — no al revés.

**Pregunta técnica sin resolver, para especialista en comercio exterior:** cuando una operación es simultáneamente reventa de mercancía importada y exportación (compra en el extranjero, entra a México, y después se reexporta), podrían coexistir 2 requisitos en el mismo CFDI — citar el pedimento de importación (Art. 29-A CFF) e incluir el Complemento de Comercio Exterior. No está confirmado cómo interactúan ambos requisitos cuando coinciden en una sola operación. Se documenta en el análisis de operación de exportación (documento aparte) para tratarse en sesión con el cliente y su asesor de comercio exterior.

### 4.5 Estado de la configuración actual

El campo existe con default `'01'` (No aplica). Existe evidencia real de exportación en la operación de PROQUIFA (factura de Mungen a un cliente extranjero, analizada en la guía de NC), y nada en la definición de los 4 escenarios en alcance (guía de Facturas, sección 2) excluye a un cliente extranjero. Este eje puede activarse dentro del alcance actual — no está descartado por evidencia, queda pendiente de la sesión con el cliente (ver documento de análisis de operación de exportación).

---

## 5. Naturaleza del receptor — eje independiente del destino de la operación

Que un receptor tenga RFC genérico de extranjero no determina, por sí solo, el resultado del Eje 2.

| Dato | Qué resuelve | Campos que afecta |
|---|---|---|
| ¿El receptor tiene RFC mexicano o es extranjero? | Identidad fiscal del comprador | `RFC` (`XEXX010101000` si extranjero), `RegimenFiscalReceptor` (`616` si sin obligaciones fiscales en México), `UsoCFDI` (`S01` si sin efectos fiscales) |
| ¿La mercancía sale físicamente de México con pedimento? | Eje 2 (sección 4) | `ExportacionCode`, y si aplica, fuerza `TasaOCuota=0` |

**Ejemplo — Pharma Scientific Inc. (EUA):** RFC genérico de extranjero, régimen 616. Si compra mercancía y la recoge directamente en la bodega de México, sin pedimento de exportación, la venta se factura al 16% — el RFC extranjero determina la cabecera del receptor, no la tasa.

**Nota aparte:** algunas entidades del grupo (Pharma Scientific Inc.) no generan CFDI en absoluto (`EmpresaEmisora.RequiereCFDI = false`, guía de Facturas sección 5.3.1) — es un eje distinto, sin relación con el tratamiento de IVA de las operaciones que sí factura PROQUIFA México.

---

## 6. Impacto por etapa del sistema

| Etapa | Cálculo esperado | Punto de verificación |
|---|---|---|
| **Cotización** | Total por línea, según el `PerfilFiscal` real del producto (Eje 1) — el Eje 2 (Exportación) todavía no se conoce en esta etapa | Que el cálculo se resuelva por línea, no con una tasa plana sobre el total |
| **Orden de Compra (OC)** | Hereda los montos ya cotizados, la envía el cliente con base en la Cotización | Que la OC no recalcule con lógica propia, independiente de la Cotización |
| **Pedido** | Conserva, por línea, el `PerfilFiscal` resuelto desde Cotización/OC (Eje 1) | Que el Pedido guarde el perfil fiscal por línea, no solo un total genérico |
| **Proforma** | Mismo cálculo que el Pedido (Eje 1) — para Prepago, la Proforma se genera antes de comprarle al proveedor, así que tampoco conoce el Eje 2 todavía (sección 4.4) | Que la Proforma no asuma un resultado de exportación que aún no es un hecho |
| **Factura** | Aplica el Eje 1 y, si ya existe el pedimento de exportación real, el Eje 2 — puede diferir del cálculo mostrado en Cotización/Pedido/Proforma, y es correcto que difiera (sección 4.2) | Cubierto en las guías de Facturas y de NC |

**No es una inconsistencia que la Factura difiera de la Cotización/Pedido/Proforma en el tratamiento de IVA** — es el comportamiento esperado cuando el Eje 2 se confirma después de que esos documentos ya se generaron (sección 4.2).

**Pendiente, para sesión con el cliente (ver documento de análisis de operación de exportación):** cómo se integra el Complemento de Comercio Exterior al flujo de facturación por adelantado, y cómo interactúa con el requisito de citar el pedimento de importación cuando ambos coinciden en una misma operación (sección 4.4).

---

## 7. Ejemplos

### 7.1 Producto con tasa 0% por naturaleza (Eje 1), operación doméstica

```
Producto: Suplemento FEUM digital (publicación)
PerfilFiscal: IVA Tasa 0% (Art. 2-A LIVA), heredado de la Familia "Publicaciones"

Cotización:  1 x $1,000 → Subtotal $1,000, IVA 0% = $0, Total $1,000
OC:          confirma la Cotización, sin recalcular
Pedido:      línea con PerfilFiscalId = 2
Proforma:    mismo cálculo, Total $1,000
Factura:     hereda de la Proforma — SatObjetoImpCode=02, SatTipoFactorCode=Tasa, TasaOCuota=0.000000
```

### 7.2 Mismo producto, tasa 16% por defecto, pero exportación real confirmada después (Eje 2 activo, Ruta B)

```
Producto: reactivo químico, PerfilFiscal = IVA General 16% (Eje 1)
Cliente: extranjero (ej. Guatemala)

Cotización: 1 x $1,000 → Subtotal $1,000, IVA 16% = $160, Total $1,160
Pedido:     misma tasa, 16% — la ruta física aún no se conoce

... se compra al proveedor en EUA, entra a México (pedimento de importación),
    se confirma que sale hacia el cliente con pedimento de exportación (Ruta B) ...

Factura: PerfilFiscal del producto (16%) queda ANULADO por el Eje 2
         ExportacionCode=02 (Definitiva) fuerza TasaOCuota=0,
         independientemente de lo mostrado en Cotización y Pedido
```

Que la Factura difiera de la Cotización/Pedido en este caso es correcto — ver sección 4.2.

---

## 8. Checklist de validaciones

- [ ] Ninguna pantalla calcula IVA con una tasa fija o un valor booleano — todo cálculo resuelve `PerfilFiscal` por línea.
- [ ] Cotización, Pedido y Proforma calculan por línea usando `PerfilFiscal` (Eje 1) — no intentan anticipar el Eje 2 (Exportación) antes de que exista el pedimento correspondiente.
- [ ] La OC no tiene lógica de cálculo propia — hereda lo ya cotizado.
- [ ] Que la Factura difiera de la Cotización/Pedido/Proforma en el tratamiento de IVA no se trata como un error o descuadre — es el resultado esperado cuando el Eje 2 se confirma después (sección 4.2).
- [ ] Ninguna pantalla combina el Eje 1 (producto), el Eje 2 (exportación) o la naturaleza del receptor (RFC extranjero) como si fueran el mismo dato — son 3 conceptos independientes (secciones 3, 4 y 5).
- [ ] Confirmar con el cliente el nivel de asignación de `PerfilFiscal` (sección 3.3).
- [ ] Confirmar con el cliente si el catálogo de productos incluye medicinas de patente terminadas (sección 2.1).
- [ ] Confirmar con el cliente con qué frecuencia ocurren ventas de exportación real (Eje 2) dentro de los 4 escenarios en alcance, y si se debe construir la lógica condicional completa o queda documentado como pendiente (sección 4).

---

## 9. Fuera de alcance

| Elemento | Motivo |
|---|---|
| Zona fronteriza (tasa 8%) | Ningún domicilio de PROQUIFA está en zona fronteriza |
| IEPS | Se agrega como cuarta fila de `PerfilFiscal` solo si el cliente confirma que aplica |
