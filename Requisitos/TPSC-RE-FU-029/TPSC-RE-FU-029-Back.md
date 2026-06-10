# Impacto en Back — TPSC-RE-FU-029
**Requisito:** Validar Cobro: Paso 3 Perú — Facturación y Envío
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder
**Módulo:** Validar Cobro — Wizard Paso 3 (Perú)
**Impacto:** Scripts BD ProquifaDotNet (1 catálogo nuevo + 2 ALTER tablas + 1 ALTER vista) + Endpoints Finanzas: inicialización Paso 3 Perú, auto-guardado catálogos SUNAT, previsualización PDF, timbrado CPE único (sin cascada), envío Brevo con adjuntos + acciones post-envío (FEE, Confirmación de Pedido). Comunicación Finanzas → Timbrado vía API. **Sin transferencia a Legacy. Solo Perú — operaciones unitarias por línea, tipo único: Factura electrónica.**

---

## Resumen

Este requisito implementa la **tercera y última pantalla del wizard de Validar Cobro (Paso 3 — Facturación y Envío) para Región Perú** en ProquifaDotNet.Finanzas. Es el equivalente peruano de TPSC-RE-FU-028 (México), con una lógica significativamente más simple: en Perú solo existe un tipo de documento fiscal (la Factura electrónica, CPE tipo 01) y no hay transferencia a Legacy.

A diferencia de México, el Paso 3 Perú **no tiene lógica condicional de tres caminos**:
- Toda línea proveniente de una proforma → **Factura electrónica** (CPE tipo 01, UBL 2.1)
- No existe Factura Anticipo con relación 07 SAT
- No existe Complemento de Pago (SUNAT no tiene ese mecanismo)
- Líneas provenientes de FAA existente → sin documento fiscal (solo conciliación interna — **pendiente confirmar acciones con el cliente, ver Regla 4**)

Al confirmar el envío de cada línea, Finanzas dispara dos acciones automáticas: establece la FEE del pedido y genera la Confirmación de Pedido adjunta al correo. **No hay transferencia a Legacy** — es la única diferencia post-envío respecto a México.

> ⚠️ **Brecha bloqueante — Timbrado SUNAT:** La modalidad de emisión electrónica ante SUNAT (SEE-SOL, SEE del Contribuyente, SEE-OSE o Facturador SUNAT) está pendiente de definir. No se reutiliza el PAC TurboPac de México. Sin resolver esto, el timbrado Perú no puede implementarse (ver Brecha B1).

### Distribución de responsabilidades

| Capa | Aplicativo | Responsabilidad |
|------|-----------|-----------------|
| BD — Catálogo nuevo | ProquifaDotNet | `catTipoOperacionSUNAT` (catálogo 51 SUNAT) |
| BD — ALTER catálogo | ProquifaDotNet | `catTipoCFDI`: ADD `IdRegion` + INSERT `FACTURA_CPE` |
| BD — ALTER tabla | ProquifaDotNet | `fccDocumentoFiscalCobro`: ADD `IdCatTipoOperacionSUNAT`, `IdCatCondicionesDePago` |
| BD — ALTER vista | ProquifaDotNet | `vfccDocumentoFiscalCobro`: extender con JOINs Perú y corrección JOINs catálogos |
| BD — DML | ProquifaDotNet | `DocumentTemplate`: registro template `GOLPERU_PER_CDP` |
| Lógica Paso 3 | ProquifaDotNet.Finanzas | Inicialización líneas Perú, auto-guardado catálogos SUNAT, estados, cierre wizard |
| Timbrado | ProquifaDotNet.Timbrado | Timbrado CPE SUNAT, `INSERT CFDIGenerada` (`IdCatTipoCFDI = FACTURA_CPE`), `UPDATE EmpresaFolio GOLPERU` |
| Previsualización PDF | DocumentBuilder | PDF CPE Perú via template `GOLPERU_PER_FAC` (RE-FU-020) |
| Generación CDP | DocumentBuilder | PDF Confirmación de Pedido Perú via `GOLPERU_PER_CDP` (nueva — este requisito) |
| Envío | ProquifaDotNet.Finanzas | Modal envío, integración Brevo (PDF CPE + XML CPE + PDF CDP adjuntos) |
| Post-envío | ProquifaDotNet.Finanzas | FEE en `tpPedido`, generación `fccConfirmacionPedido` en MinIO. **Sin Legacy.** |
| Comunicación | Finanzas → Timbrado | Llamadas entre APIs para timbrado de cada CPE |
| Comunicación | Finanzas → ProquifaDotNet | Llamadas entre APIs para leer datos y escribir resultados del Paso 3 |

### Infraestructura reutilizada

| Componente | Origen | Reutilización |
|-----------|--------|---------------|
| `fccDocumentoFiscalCobro` (estructura base) | RE-FU-028 | Ciclo de vida del Paso 3; se extiende con 2 columnas Perú |
| `fccConfirmacionPedido` | RE-FU-028 | Tabla compartida; aplica también a Perú (corrección a descripción RE-028) |
| `catTipoDocumentoFiscal` (clave `FACTURA`) | RE-FU-028 | En Perú solo se usa la clave `FACTURA`; sin cambios |
| `catDocumentoFiscalCobroEstado` | RE-FU-028 | Ciclo PENDIENTE → GENERADO → ENVIADO idéntico |
| `tpPedido.FechaEstimadaEntrega` | RE-FU-028 | Campo compartido; Perú lo actualiza al enviar igual que México |
| `vfccDocumentoFiscalCobro` | RE-FU-028 | Se extiende (v2.0) con JOINs Perú y corrección de resolución de catálogos |
| `CFDIGenerada` | RE-FU-018/019 | Tabla compartida; CPE Perú se almacena con `IdCatTipoCFDI = FACTURA_CPE`, `UUID = NULL`, `Serie = F001`, `Folio = Correlativo` |
| `EmpresaFolio GOLPERU` (ProquifaDotNetTimbrado) | RE-FU-020 | Foliador SUNAT; fila GOLPERU ya insertada; mismo patrón UPDLOCK que México |
| `catCondicionesDePago` | RE-FU-018/019 | Catálogo existente (CONTADO/CRÉDITO); Perú lo usa como equivalente al `catMetodoDePagoCFDI` de México |
| `fccPagoFacturaPedido` | RE-FU-026 | FK desde `fccDocumentoFiscalCobro` (origen proforma Perú) |
| `fccPagoFacturaAdelanto` | RE-FU-026 | FK desde `fccDocumentoFiscalCobro` (origen FAA Perú; sin documento fiscal — Regla 4) |
| `DatosFacturacionCliente` | RE-FU-004 | RUC, Razón Social receptor del CPE UBL 2.1 |
| `Empresa` (GOLPERU) | RE-FU-020 | RUC Emisor, Razón Social emisora; datos fiscales pendientes (Brecha B4) |
| `FacturaPdfMappingService` Perú | RE-FU-020 | Consolidación datos CPE en `FacturaPdfModel`; template `GOLPERU_PER_FAC` |
| `ApiCallerTimbrado` (HttpClient + Polly) | RE-FU-019 | Cliente HTTP con retry policy hacia Timbrado — reutilizado sin cambios |
| `tpProformaPedido.IdCFDIGenerada` | RE-FU-026 | Campo existente; Perú lo puebla con el `IdCFDIGenerada` del CPE timbrado |

---

## Parte A — Base de Datos (ProquifaDotNet)

### A1 — CREATE TABLE catTipoOperacionSUNAT

Catálogo de tipos de operación SUNAT (catálogo 51). Reemplaza al `catUsoCFDI` del SAT: se consigna por línea en el Paso 3 Perú. Datos iniciales con las operaciones frecuentes para distribución B2B de Golocaer S.A.C.

| Clave | Descripción |
|-------|-------------|
| `0101` | Venta interna |
| `0112` | Venta interna — sustento de traslado de mercancía |
| `0200` | Exportación de bienes |
| `1001` | Operación sujeta a detracción |
| `2001` | Operación sujeta a percepción |
| *(otros)* | Extender con los códigos del catálogo 51 que Golocaer S.A.C. requiera |

> Ver script completo en `TPSC-RE-FU-029_BD.md` — Catálogo Nuevo: `catTipoOperacionSUNAT`.

### A2 — ALTER TABLE catTipoCFDI — ADD IdRegion + INSERT FACTURA_CPE

`catTipoCFDI` fue creado en RE-FU-028 solo para México. RE-029 es el primer requisito cross-región que usa este catálogo, por lo que:

1. Se agrega la columna `IdRegion uniqueidentifier NULL FK → catRegion`.
2. Se actualizan las entradas México existentes (`FACTURA_PPD`, `FACTURA_PUE`, `FACTURA_ANTICIPO`, `COMPLEMENTO_PAGO`) con `IdRegion` = región MEX.
3. Se inserta la nueva entrada `FACTURA_CPE` con `IdRegion` = región PER.

El discriminador `IdRegion` permite al wizard filtrar los tipos de documento válidos por región del cliente, evitando combinaciones inválidas.

> Ver script completo en `TPSC-RE-FU-029_BD.md` — ALTER TABLE catTipoCFDI.

### A3 — ALTER TABLE fccDocumentoFiscalCobro — Columnas Perú

Se agregan dos columnas nullable a la tabla central del Paso 3:

| Columna nueva | FK | Equivalente México |
|--------------|----|--------------------|
| `IdCatTipoOperacionSUNAT` | `catTipoOperacionSUNAT` | `IdCatUsoCFDI` |
| `IdCatCondicionesDePago` | `catCondicionesDePago` (existente) | `IdCatMetodoDePagoCFDI` |

Las columnas México (`IdCatUsoCFDI`, `IdCatMetodoDePagoCFDI`) se mantienen y quedan NULL para líneas Perú. Las columnas Perú quedan NULL para líneas México. La validación de exclusividad es responsabilidad de la capa de aplicación (Finanzas).

> Ver script completo en `TPSC-RE-FU-029_BD.md` — ALTER TABLE fccDocumentoFiscalCobro.

### A4 — ALTER VIEW vfccDocumentoFiscalCobro (v2.0)

La vista creada en RE-FU-028 se extiende con:
- `INNER JOIN` a `catTipoDocumentoFiscal` y `catDocumentoFiscalCobroEstado` para resolver correctamente `TipoDocumentoFiscal` y `EstadoLinea` (corrección: RE-028 los accedía como columnas directas cuando son FKs).
- `LEFT JOIN` a `catTipoOperacionSUNAT` y `catCondicionesDePago` para columnas Perú.
- `Cliente.Region` como discriminador MEX/PER.
- Columnas alias `CPE_Serie` y `CPE_Correlativo` reutilizando el JOIN `cg_f` existente.

> Ver script completo en `TPSC-RE-FU-029_BD.md` — ALTER VIEW vfccDocumentoFiscalCobro.

---

## Parte B — ProquifaDotNet.Finanzas: Servicios y Endpoints

### B1 — Inicialización del Paso 3 Perú

**Descripción:** Al avanzar desde el Paso 2 con la asociación cerrada para un cliente Perú, Finanzas inicializa el Paso 3 creando una fila en `fccDocumentoFiscalCobro` por cada documento procesable.

**Lógica de tipo por línea (Perú — sin condicional de tres caminos):**

| Origen (Paso 2) | `IdCatTipoDocumentoFiscal` resultante | Acción |
|-----------------|--------------------------------------|--------|
| `fccPagoFacturaPedido` | `FACTURA` | INSERT `fccDocumentoFiscalCobro` estado `PENDIENTE` |
| `fccPagoFacturaAdelanto` | *(sin documento fiscal)* | ⚠️ **Pendiente — ver Regla 4:** sin Complemento de Pago en Perú; confirmar si se registra fila de conciliación o no se crea fila |

> En México, `fccPagoFacturaPedido` con `HayControlados = 1` derivaba en `FACTURA_ANTICIPO`. En Perú ese flag no aplica: toda proforma genera una Factura electrónica normal sin importar si tiene productos con requisitos especiales.

**Datos leídos (vía API ProquifaDotNet):** `fccPagoFacturaPedido`, `fccPagoFacturaAdelanto`, `DatosFacturacionCliente` (RUC receptor), `Empresa` GOLPERU, `catTipoDocumentoFiscal` (clave `FACTURA`), `catDocumentoFiscalCobroEstado` (clave `PENDIENTE`), `fccNotaCredito` (NCs aplicadas por línea), `catCondicionesDePago`, `catTipoOperacionSUNAT`

**Escritura:** `INSERT fccDocumentoFiscalCobro` (una fila por proforma procesable, estado inicial `PENDIENTE`)

**Respuesta `Paso3InicializadoDto`** (Perú):
```
{
  Lineas: [
    {
      IdFCCDocumentoFiscalCobro,
      TipoDocumentoFiscal: "FACTURA",
      EstadoLinea: "PENDIENTE",
      FolioDocumentoOrigen,
      FolioPedidoInterno,
      EmisorNombre,        // Golocaer S.A.C.
      MontoTotal,
      TipoDeCambio,
      NotasDeCreditoAplicadas: [ { IdNC, Monto, FolioNC } ],
      // Catálogos Perú
      CatalogoTipoOperacionSUNAT: [ { Id, Clave, Descripcion } ],
      CatalogoCondicionesDePago:  [ { Id, Clave, Descripcion } ]
    }
  ],
  PuedeRegresarPasos: bool  // false si cualquier línea está en GENERADO o ENVIADO
}
```

Si al reingresar al Paso 3 ya existen filas en `fccDocumentoFiscalCobro` para el cliente, Finanzas las recupera directamente desde `vfccDocumentoFiscalCobro` sin reinicializar.

### B2 — Auto-guardado: Tipo Operación SUNAT y Condición de Pago

**Descripción:** Cada vez que el usuario selecciona o modifica el Tipo de Operación SUNAT o la Condición de Pago de una línea, Finanzas persiste el cambio inmediatamente.

**Escritura:**
```sql
UPDATE fccDocumentoFiscalCobro
SET IdCatTipoOperacionSUNAT = @Id,
    IdCatCondicionesDePago   = @Id
WHERE IdFCCDocumentoFiscalCobro = @Id
```

**Validación:** Solo se permite la operación si la línea está en estado `PENDIENTE`. Una línea en `GENERADO` o `ENVIADO` es inmutable.

> **Pendiente (Regla 5):** Confirmar si el Tipo de Operación es configurable por el operador o lo fija el sistema automáticamente (candidato `0101` — Venta interna para el flujo estándar de Golocaer S.A.C.).

### B3 — Previsualización PDF por línea

**Descripción:** Al presionar "Previsualizar", Finanzas genera el PDF representativo del CPE en memoria sin persistirlo. El usuario valida visualmente antes de timbrar.

**Flujo:**
1. Finanzas lee `vfccDocumentoFiscalCobro` + `fccNotaCredito` (NCs a incluir en la factura, pendiente de mecánica SUNAT — ver Brecha B3).
2. Invoca `FacturaPdfMappingService.MapearPreviewAsync()` Perú (RE-FU-020) — consolida datos en `FacturaPdfModel` UBL 2.1 sin CDR/sello digital, resuelve `TemplateKey` = `GOLPERU_PER_FAC`.
3. Genera PDF en memoria vía DocumentBuilder.
4. Retorna el PDF en memoria al frontend para el modal de previsualización.
5. Sin escrituras en BD.

**DocumentBuilder — documentos Perú:**

| Documento | TemplateKey | Estado |
|-----------|-------------|--------|
| PDF Factura electrónica Perú (preview + definitivo) | `GOLPERU_PER_FAC` | **Existente** (RE-FU-020) |
| PDF Confirmación de Pedido Perú | `GOLPERU_PER_CDP` | **Nueva — definir en RE-FU-029** |

### B4 — Timbrado CPE por línea (sin cascada)

**Descripción:** Al confirmar el timbrado, Finanzas invoca ProquifaDotNet.Timbrado vía API para timbrar el CPE. En Perú solo existe un escenario (no hay cascada PPD ni Complemento de Pago):

**Escenario único — FACTURA CPE tipo 01 (1 CPE por línea):**

1. Finanzas → Timbrado: solicita timbrado de la Factura electrónica con datos del documento, RUC emisor (GOLPERU), RUC receptor, partidas UBL 2.1, Tipo de Operación SUNAT y NCs aplicadas (mecánica pendiente — Brecha B3).
2. Timbrado invoca OSE/PSE SUNAT (modalidad pendiente — Brecha B1), inserta en `CFDIGenerada`:
   - `Serie = 'F001'` (o la serie confirmada con Golocaer S.A.C.)
   - `Folio = Correlativo` (8 dígitos, ej. `00000001`)
   - `UUID = NULL` (SUNAT no genera UUID)
   - `IdCatTipoCFDI` → `FACTURA_CPE`
3. Actualiza `EmpresaFolio GOLPERU SET UltimoFolio + 1` (UPDLOCK atómico — mismo patrón que México).
4. Retorna Serie, Correlativo, FechaEmision y XML CDR a Finanzas.
5. Finanzas genera PDF definitivo del CPE con CDR/sello SUNAT vía `FacturaPdfMappingService.MapearAsync()` → sube a MinIO → `INSERT Archivo` → `UPDATE CFDIGenerada.IdArchivoPdf`.
6. Finanzas: `UPDATE tpProformaPedido SET IdCFDIGenerada = @IdCPE`.
7. Finanzas: `UPDATE fccDocumentoFiscalCobro SET EstadoLinea = GENERADO, IdCFDIGeneradaFactura = @IdCPE, FechaGeneracion`.

**Modal de éxito tras timbrado:** muestra Serie y Correlativo del CPE (no UUID — SUNAT no lo tiene).

**Manejo de errores SUNAT/OSE:** Igual que el patrón de RE-FU-020. Si el servicio SUNAT responde con error (código de rechazo CDR, servicio indisponible, datos inválidos), la línea permanece en `PENDIENTE`; Finanzas muestra el detalle del error al usuario para corrección y reintento.

> ⚠️ **Brecha B1 — BLOQUEANTE:** Modalidad de emisión SUNAT no definida. No se puede implementar el módulo de timbrado Perú hasta que se resuelva (ver Brechas).

### B5 — Modal de Envío y despacho con Brevo

**Descripción:** Al presionar "Enviar" en una línea en estado `GENERADO`, Finanzas abre el modal de envío y, al confirmar, despacha el correo vía Brevo.

**Destinatarios:**
- **Para:** Contacto del cliente del pedido (`tpPedido.IdContacto`) — editable en el modal.
- **CC:** ESAC asignado al cliente/pedido — editable en el modal.

**Adjuntos (Perú — tipo único FACTURA_CPE):**

| Adjunto | Origen | Removible |
|---------|--------|-----------|
| PDF Factura electrónica | MinIO (subido en B4 paso 5) | No |
| XML CPE (CDR aceptado) | Retornado por Timbrado en B4 | No |
| PDF Confirmación de Pedido | Generado en B6.2 | No |

**Asunto:** Pre-rellenado automáticamente con Serie, Correlativo y Folio Pedido Interno.
Propuesta: `"{FolioPedidoInterno} — Factura {Serie}-{Correlativo}"`

> ⚠️ **Brecha B2:** Formato exacto del asunto y plantilla del cuerpo del correo para Perú pendiente de confirmar con PMO.

**Escritura al confirmar envío exitoso:**
- `UPDATE fccDocumentoFiscalCobro SET EstadoLinea = ENVIADO, FechaEnvio`
- `INSERT CorreoEnviado` + `INSERT ArchivoCorreoEnviado` (×3 adjuntos)

### B6 — Acciones post-envío automáticas (Perú — sin Legacy)

Al confirmar el envío exitoso de cada línea, Finanzas dispara automáticamente dos acciones. **La transferencia a Legacy NO aplica a Perú.**

#### B6.1 — Fecha Estimada de Entrega (FEE)

```sql
UPDATE tpPedido
SET FechaEstimadaEntrega = @FEECalculada
WHERE IdTPPedido = @IdTPPedido
```

Misma lógica que México (RE-FU-028 B7.1). Las reglas de cálculo de la FEE son compartidas.

> ⚠️ **Brecha B5:** Las reglas de cálculo de la FEE (días hábiles, días calendario, parámetro configurable) están pendientes de confirmar con Operaciones — brecha compartida con RE-FU-028.

#### B6.2 — Generación de Confirmación de Pedido (CDP)

1. Finanzas solicita a DocumentBuilder la generación del PDF usando template `GOLPERU_PER_CDP`.
2. DocumentBuilder genera el PDF y lo almacena en MinIO (bucket `confirmaciones`).
3. Finanzas: `INSERT fccConfirmacionPedido (IdTPPedido, FolioConfirmacion, RutaArchivoPDF)`.
4. El PDF se adjunta al correo del modal de envío (paso B5).

> ⚠️ **Brecha B6:** El template `GOLPERU_PER_CDP` (diseño HTML Header/Body/Footer) es nuevo. Pendiente diseñar y registrar en DocumentBuilder — análogo a los templates `*_MEX_CDP` de México. Sin el template, el CDP Perú no puede generarse.

#### B6.3 — Sin transferencia a Legacy

Perú **no ejecuta ninguna transferencia a Legacy** al enviar. Las ETLs de Buzón de Cobros, Proforma, Factura y PDF son exclusivas del flujo México (ver Regla 11 del requisito y Parte E de RE-FU-028).

### B7 — Cierre del wizard

Al detectar que todas las líneas del cliente están en estado `ENVIADO`, Finanzas cierra el wizard y retorna al listado principal de Validar Cobro.

**Lectura:**
```sql
SELECT COUNT(*)
FROM fccDocumentoFiscalCobro
WHERE IdFCCPagoCliente = @Id
  AND EstadoLinea != 'ENVIADO'
```
(Consultado vía `vfccDocumentoFiscalCobro` filtrado por `ClienteRegion = 'PER'`)

---

## Parte C — ProquifaDotNet.Timbrado

### C1 — Endpoint de timbrado CPE SUNAT

Timbrado recibe una solicitud por línea desde Finanzas. A diferencia de México (que puede generar 2 CFDIs en cascada PPD), en Perú siempre es **1 CPE por llamada**.

La solicitud incluye: tipo `FACTURA_CPE`, RUC emisor (GOLPERU), RUC receptor, partidas UBL 2.1 (con código SUNAT, UdM SUNAT, afectación IGV), Tipo de Operación catálogo 51, y NCs aplicadas (mecánica catálogo 09 pendiente — RE-FU-033/035).

**Por cada timbrado exitoso:**
1. `INSERT CFDIGenerada` (`IdCatTipoCFDI = FACTURA_CPE`, `UUID = NULL`, `Serie = F001`, `Folio = Correlativo`)
2. `UPDATE EmpresaFolio GOLPERU SET UltimoFolio + 1` (UPDLOCK atómico — mismo patrón que México, diferente fila)
3. Llamada al servicio SUNAT (modalidad pendiente — Brecha B1)
4. `UPDATE CFDIGenerada` con FechaEmision y respuesta CDR
5. `INSERT TimbradoLog` (trazabilidad)
6. Retorna a Finanzas: Serie, Correlativo, FechaEmision, XML CPE, XML CDR

> ⚠️ **Diferencia con México:** En México el módulo Timbrado llama al PAC TurboPac (implementado en RE-FU-018/019). En Perú el módulo equivalente llama a la modalidad SUNAT pendiente de definir. Ambos módulos residen en `ProquifaDotNet.Timbrado` pero con implementaciones separadas discriminadas por región.

---

## Parte D — DocumentBuilder

| Documento | TemplateKey | Estado | Disparado por |
|-----------|-------------|--------|---------------|
| PDF Factura electrónica Perú (preview) | `GOLPERU_PER_FAC` | **Existente** (RE-FU-020) | `FacturaPdfMappingService.MapearPreviewAsync()` Perú |
| PDF Factura electrónica Perú (definitivo) | `GOLPERU_PER_FAC` | **Existente** (RE-FU-020) | `FacturaPdfMappingService.MapearAsync()` post-timbrado |
| PDF Confirmación de Pedido Perú | `GOLPERU_PER_CDP` | **Nueva — definir en RE-FU-029** | Finanzas post-envío (B6.2), adjunta en el correo |

> La plantilla `GOLPERU_PER_FAC` ya existe con toda su estructura (Header, Body, Footer) y registros `DocumentTemplate` desde RE-FU-020. Solo se reutiliza.
>
> La plantilla `GOLPERU_PER_CDP` (Confirmación de Pedido Perú) es nueva y requiere diseño HTML + registro `DocumentTemplate`. El patrón de referencia son los templates `GOL/MUN/PRO/PQF_MEX_CDP` de México (RE-FU-028).

---

## Diferencias clave vs RE-FU-028 (México)

| Aspecto | México (RE-FU-028) | Perú (RE-FU-029) |
|---------|-------------------|------------------|
| Tipos de documento | FACTURA, FACTURA_ANTICIPO, COMPLEMENTO_PAGO | Solo FACTURA (CPE tipo 01) |
| Lógica condicional tipo | 3 caminos (HayControlados + origen FAA) | Sin condicional — toda proforma → Factura |
| Catálogo 1 por línea | `catUsoCFDI` (SAT) | `catTipoOperacionSUNAT` (catálogo 51 SUNAT) |
| Catálogo 2 por línea | `catMetodoDePagoCFDI` (PPD/PUE) | `catCondicionesDePago` (CONTADO/CRÉDITO) |
| Cascada timbrado PPD | Sí (2 CFDIs: Factura + Complemento) | No (1 CPE siempre) |
| Comprobante | CFDI 4.0 SAT — UUID | CPE UBL 2.1 SUNAT — Serie+Correlativo (no UUID) |
| Serie/Folio | SAT (alfanumérica/numérica) | SUNAT (F001 + correlativo 8 dígitos) |
| Servicio timbrado | PAC TurboPac (RE-FU-019) | OSE/PSE SUNAT (modalidad pendiente) |
| FAA existente + cobro | COMPLEMENTO_PAGO (1 CFDI) | Sin documento fiscal — conciliación interna (Regla 4, pendiente) |
| FEE | Sí | Sí |
| Confirmación de Pedido | Sí (`*_MEX_CDP`) | Sí (`GOLPERU_PER_CDP`) |
| Transferencia a Legacy | Sí (E1–E3 + E6 en RE-028) | **No** |
| Modal éxito timbrado | UUID + Folio Fiscal SAT | Serie + Correlativo SUNAT (sin UUID) |

---

## Brechas

> ⚠️ **BRECHA BLOQUEANTE — Modalidad de emisión electrónica SUNAT (B1)**
> La integración con SUNAT (SEE-SOL, SEE del Contribuyente, SEE-OSE o Facturador SUNAT) está pendiente de definir. SUNAT ofrece cuatro modalidades; la elección depende de si Golocaer S.A.C. es gran contribuyente, entre otros factores. No se da por hecho el uso de un OSE y no se reutiliza el PAC TurboPac de México. Sin esta definición, el módulo de timbrado Perú en `ProquifaDotNet.Timbrado` no puede implementarse. Ver Brecha 5 de TPSC-RE-FU-005 y Brechas B1–B4 de TPSC-RE-FU-020.

> ⚠️ **BRECHA MEDIA — Asunto y plantilla del correo de envío Perú (B2)**
> El formato del asunto y el cuerpo del correo de envío para Perú están pendientes de confirmar con PMO. Propuesta inicial: `"{FolioPedidoInterno} — Factura {Serie}-{Correlativo}"`. Sin esto, el modal de envío no puede finalizarse.

> ⚠️ **BRECHA MEDIA — Mecánica de referencia de NCs peruanas en el CPE (B3)**
> La forma de referenciar Notas de Crédito en el comprobante peruano (catálogo 09 SUNAT) se define en TPSC-RE-FU-033/035. Hasta que no esté resuelto, la inclusión de NCs en el XML CPE no puede implementarse. Las NCs sí se muestran en la previsualización y en la línea como dato informativo; solo la integración fiscal al XML queda pendiente.

> ⚠️ **BRECHA BLOQUEANTE — Datos fiscales SUNAT del producto (B4)**
> `Producto.CodigoSUNAT`, `catUnidad.ClaveSUNAT` y `catAfectacionIGV` son obligatorios en el UBL 2.1. Fueron identificados como brecha bloqueante en TPSC-RE-FU-020. Sin estos datos no es posible generar el XML CPE válido. Prerequisito para el timbrado.

> ⚠️ **BRECHA MEDIA — Reglas de cálculo de la FEE (B5)**
> Compartida con RE-FU-028 (Brecha B4 de ese requisito). Las reglas exactas (días hábiles, fórmula, parámetro por empresa) están pendientes de confirmar con Operaciones PROQUIFA. Aplica por igual a México y Perú.

> ⚠️ **BRECHA MEDIA — Template DocumentBuilder GOLPERU_PER_CDP (B6)**
> El diseño HTML (Header/Body/Footer) de la Confirmación de Pedido Perú es nuevo. Pendiente diseño y registro en DocumentBuilder. Sin el template, la Confirmación de Pedido no puede generarse al enviar.

> ⚠️ **BRECHA MEDIA — Escenario FAA existente sin documento fiscal en Perú (B7)**
> Cuando el cobro del Paso 2 corresponde a una Factura por Adelantado ya emitida, en Perú no hay documento fiscal que generar. Pendiente confirmar con el cliente qué acción ofrece el sistema: ¿registrar conciliación interna y cerrar la línea sin envío?, ¿enviar alguna constancia no fiscal? Ver Regla 4 del requisito. Mientras no se defina, el Paso 3 para esas líneas no tiene flujo de cierre.

> ⚠️ **BRECHA MEDIA — Parámetros configurables proforma→factura Perú (B8)**
> Pendiente confirmar qué parámetros se configuran al pasar de proforma a factura en Perú más allá de Tipo de Operación y Condición de Pago: ¿detracción/percepción cuando aplique?, ¿tipo de cambio editable? Ver Regla 5 del requisito.

> ⚠️ **BRECHA — Datos legales Golocaer S.A.C. (B9)**
> RUC emisor, domicilio fiscal y certificado digital de Golocaer S.A.C. en `Empresa` y `AppSetting ProquifaDotNetTimbrado` están pendientes de recopilar (Brecha B3 de TPSC-RE-FU-020). Sin estos datos el emisor del CPE no puede completarse correctamente.
