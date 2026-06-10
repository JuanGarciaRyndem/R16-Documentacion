# Impacto en Backend — Notas de Crédito Perú (CPE tipo 07 — UBL 2.1)
**Requisito:** TPSC-RE-FU-033
**Aplicativos:** ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10) + DocumentBuilder
**Versión:** 1.0

---

> ⚠️ **Nota transversal:** Toda la mecánica fiscal SUNAT (catálogo 09, campos CPE tipo 07, plazos, tipo de cambio, conservación) requiere validación con el asesor fiscal peruano de PROQUIFA antes de implementarse.
>
> ⚠️ **Brecha bloqueante B1:** La modalidad de emisión electrónica ante SUNAT (OSE/directa) está pendiente de definir (compartida con RE-029). Bloquea el endpoint de timbrado (Parte C) y el flujo post-timbrado (Parte B, B6). Las demás partes pueden avanzar.

---

## Parte A — Cambios en Base de Datos

Ver `TPSC-RE-FU-033_BD.md` para scripts completos.

- **A1** — `ALTER TABLE fccNotaCredito`: ADD 3 columnas Perú (`ResponseCode`, `ResponseDescription`, `TipoCambioOrigen`). Prerrequisito: RE-032 T1.
- **A2** — `CREATE TABLE catMotivoCreditoSUNAT09`: catálogo 09 SUNAT con 11 motivos y columna `Modalidad` (POR_PARTIDAS / MANUAL).
- **A3** — `DML catTipoCFDI`: INSERT `NOTA_CREDITO_PERU` con `TipoDocumento='07'`.
- **A4** — `DML EmpresaFolio`: INSERT Serie NC Perú (GOL S.A.C.) — Serie pendiente PMO.
- **A5** — `DML DocumentTemplate`: INSERT `GOL_PER_NC` en DocumentBuilder.

---

## Parte B — ProquifaDotNet.Finanzas

### B1 — Endpoint GET catálogo 09 SUNAT

`GET /api/catalogos/motivos-nota-credito-sunat`

Retorna la lista de motivos activos del `catMotivoCreditoSUNAT09` con su `Modalidad`, para que el frontend determine automáticamente si debe mostrar la tabla de partidas o el formulario manual al seleccionar el motivo.

```
MotivoCreditoSUNATDto {
    clave: string          // '01', '04', etc.
    descripcion: string    // Descripción SUNAT
    modalidad: string      // 'POR_PARTIDAS' | 'MANUAL'
}
```

Respuesta de ejemplo:
```json
[
  { "clave": "01", "descripcion": "Anulación de la operación",       "modalidad": "POR_PARTIDAS" },
  { "clave": "04", "descripcion": "Descuento global",                 "modalidad": "MANUAL" },
  { "clave": "06", "descripcion": "Devolución total",                 "modalidad": "POR_PARTIDAS" }
]
```

### B2 — Wizard Paso 1 — Búsqueda de factura origen (CPE vigente)

Endpoint: `GET /api/nc-peru/facturas-elegibles?idCliente={id}`

- Filtra `CFDIGenerada` con `TipoDocumento='01'` (CPE Factura) de la empresa Golocaer S.A.C., de clientes prepago de Región Perú.
- Solo facturas con constancia SUNAT aceptada y sin NC de anulación previa (sin `fccNotaCredito` relacionada con `ResponseCode='01'` vigente).
- ⚠️ El campo de estado SUNAT del CPE (`CFDI.EstadoSUNAT` o equivalente) depende de la estructura definida en RE-029.

### B3 — Wizard Paso 2 — Modalidad por partidas (motivos 01, 02, 03, 05, 06, 07)

Endpoint: `GET /api/nc-peru/partidas-factura?idCFDIGenerada={id}`

- Carga `CFDIGeneradaConcepto` del CPE origen.
- **Motivo 01 (Anulación):** pre-carga todas las partidas con `CantidadNC = CantidadFacturada`; no es editable por el usuario.
- **Motivos 05, 07 (Descuento/Devolución por ítem):** el usuario edita `CantidadNC` (0, parcial o total). Las partidas con `CantidadNC = 0` no se incluyen en el XML.
- Cálculo en tiempo real por partida: `Importe = CantidadNC × ValorUnitario`, `IGV = Importe × 0.18`, `Total = Importe + IGV`.
- Total NC del documento = suma de los `Total` de las partidas incluidas.

### B4 — Wizard Paso 2 — Modalidad manual (motivos 04, 08, 09, 10, 13)

- El usuario ingresa `MontoTotalNC` (máximo = Total de la factura origen).
- Campo `Concepto` obligatorio (texto libre — mapea a `cbc:Description` en el XML).
- Campo `Observaciones` opcional.
- Cálculo automático: `IGV = MontoTotalNC × (18/118)`, `Subtotal = MontoTotalNC - IGV`.

### B5 — Armado XML CPE tipo 07 (UBL 2.1) — NCPeruXmlBuilder

Construye el XML de la NC conforme a la estructura UBL 2.1 SUNAT:

| Campo XML | Valor / Origen |
|---|---|
| `InvoiceTypeCode` | `07` (fijo) |
| Versión | `UBL 2.1` (fijo) |
| `Serie` | Tomada del `EmpresaFolio` con UPDLOCK (GOL SAC — Serie NC Perú) |
| `Correlativo` | `EmpresaFolio.UltimoFolio + 1` |
| `cbc:ReferenceID` | Serie-correlativo del CPE origen (ej. `F001-00000123`) |
| `cac:BillingReference` | Nodo de referencia al comprobante modificado con serie-correlativo y tipo 01 |
| `cbc:ResponseCode` | `fccNotaCredito.ResponseCode` (clave catálogo 09) |
| `cbc:Description` | `fccNotaCredito.ResponseDescription` (sustento del motivo) |
| `cbc:DocumentCurrencyCode` | Moneda heredada de la factura origen |
| `cbc:CalculationRate` | `TipoCambioOrigen` si moneda ≠ PEN (heredado de la factura origen) |
| Emisor RUC | Golocaer S.A.C. — de configuración empresa |
| Receptor RUC | `Cliente.RUC` (catálogo de clientes Perú) |
| IGV | 18 % calculado sobre `Subtotal` |
| `cac:CreditNoteLine` | Por partida (modalidad POR_PARTIDAS) o línea única (modalidad MANUAL) |
| Régimen Tributario | Configuración empresa emisora + catálogo clientes ⚠️ pendiente validar P8 |

**Diferencias clave vs NCMexicoXmlBuilder (RE-032):**
- Sin `TipoRelacion='01'`, sin `UsoCFDI`, sin `MetodoPago`, sin `FormaPago`, sin UUID.
- La referencia al CPE origen es por `serie-correlativo` en `cac:BillingReference`.
- El tipo de cambio es `TipoCambioOrigen` (fecha de la factura origen), no el del día del timbrado.
- No hay cancelación SAT condicional.

### B6 — Previsualización PDF — Paso 3 (NCPeruPdfMappingService)

`NCPeruPdfMappingService` sigue el patrón de `NCMexicoPdfMappingService` (RE-032) con adaptaciones SUNAT:

- **Preview (Paso 3):** sin constancia SUNAT, sin número de orden OSE. Template `GOL_PER_NC`.
- **Post-timbrado:** con datos de la constancia SUNAT (número, fecha, QR SUNAT si aplica).
- `TemplateKey = 'GOL_PER_NC'` (fijo — solo empresa Golocaer S.A.C.).
- El PDF muestra: folio serie-correlativo, RUC emisor/receptor, BillingReference, motivo catálogo 09, modalidad partidas o manual, IGV 18 %, total NC, moneda.

Endpoint: `GET /api/nc-peru/preview-pdf?idFCCNotaCredito={id}` — retorna PDF en memoria sin timbrar.

### B7 — Timbrado NC Perú, persistencia MinIO y correo automático

`POST /api/nc-peru/timbrar?idFCCNotaCredito={id}`

Orquesta la secuencia post-timbrado (⚠️ bloqueado por Brecha B1 — modalidad SUNAT pendiente):

1. Llama al endpoint de Timbrado (`POST /api/timbrado/nota-credito-peru`) con el `NCPeruTimbradorRequest`.
2. Recibe respuesta con constancia SUNAT (número, estado, fecha).
3. Genera PDF post-timbrado vía DocumentBuilder con `NCPeruPdfMappingService.MapearPostTimbrAsync()`.
4. `PersistirNCPeruPdfService.PersistirAsync()`:
   - Resuelve bucket MinIO Perú desde `RegionConfiguracionMinioBucket` (Region='PER').
   - Sube PDF → MinIO en `notas-credito-per/notas_credito/{anio}/{mes}/{serie_correlativo}.pdf`.
   - Sube XML → MinIO en `notas-credito-per/notas_credito/{anio}/{mes}/{serie_correlativo}.xml`.
   - INSERT `Archivo` × 2 (PDF + XML).
   - INSERT `CFDIGeneradaConcepto` (si por partidas).
   - UPDATE `fccNotaCredito` (Estado='VIGENTE', IdArchivoPdf, IdArchivoXml, IdCFDIGenerada).
5. Envía correo automático al cliente (PDF + XML adjuntos).
6. INSERT `CorreoEnviado` + `ArchivoCorreoEnviado`.
7. Navega al Paso 4 (NC Emitida).

**Manejo de errores:** Si SUNAT retorna error, la NC permanece en estado previo (sin VIGENTE). El usuario puede reintentar.

### B8 — Consulta principal y drill-down por cliente

- **Pantalla principal:** `GET /api/nc-peru?idCliente={id?}&moneda={m?}&fechaDesde={d?}&fechaHasta={h?}` — retorna NCs agrupadas por cliente (Total NC, Vigentes, Por Aplicar, Monto Total, Moneda).
- **Drill-down:** `GET /api/nc-peru/cliente/{idCliente}` — lista NCs del cliente con folio (serie-correlativo), fecha, monto, estado, comprobante origen.

### B9 — Reenvío de correo

`POST /api/nc-peru/{idFCCNotaCredito}/reenviar-correo` — reutiliza el mismo PDF/XML del `Archivo` y genera nuevo `CorreoEnviado`.

---

## Parte C — ProquifaDotNet.Timbrado

### C1 — Endpoint timbrado NC Perú (CPE tipo 07) ⚠️ BLOQUEADO BRECHA B1

`POST /api/timbrado/nota-credito-peru`

⚠️ **Brecha B1 bloqueante:** La modalidad de emisión electrónica ante SUNAT no está definida. No se asume OSE ni se reutiliza TurboPac de México. Este endpoint no puede implementarse hasta que RE-029 resuelva la brecha de timbrado SUNAT.

Diseño anticipado (sujeto a cambio):
- Recibe `NCPeruTimbradorRequest` con el XML UBL 2.1 armado por Finanzas.
- Obtiene folio con UPDLOCK atómico sobre `EmpresaFolio` Serie NC Perú.
- Envía XML al proveedor SUNAT (OSE/directa — pendiente).
- Recibe respuesta con constancia de recepción.
- INSERT `CFDIGenerada` con `TipoDocumento='07'`, `IdCatTipoCFDI`=NOTA_CREDITO_PERU.
- UPDATE `EmpresaFolio.UltimoFolio`.
- Retorna `NCPeruTimbradorResponse` a Finanzas.

---

## Parte D — DocumentBuilder

### D1 — Template GOL_PER_NC (H/B/F)

3 archivos HTML para el PDF representativo de la NC Perú: `GOL_PER_NC_H.html`, `GOL_PER_NC_B.html`, `GOL_PER_NC_F.html`.

**Diferencias vs GOL_MEX_NC (RE-032):**

| Elemento | GOL_MEX_NC | GOL_PER_NC |
|---|---|---|
| Empresa | GOL (Golocaer S.A. de C.V.) | Golocaer S.A.C. |
| Tipo doc. | E — Nota de Crédito (CFDI 4.0) | 07 — Nota de Crédito Electrónica (UBL 2.1) |
| Referencia origen | UUID (Folio Fiscal) | Serie-Correlativo (cbc:ReferenceID) |
| Motivo | SAT c_MotivoCancelacion | SUNAT catálogo 09 |
| Impuesto | IVA 16 % | IGV 18 % |
| Sello / QR | Sello SAT + QR SAT | Constancia SUNAT + QR SUNAT (pendiente definición) |
| Identificadores | RFC emisor / RFC receptor | RUC emisor / RUC receptor |

- **Preview:** sin constancia SUNAT, sin número de orden OSE, sin QR.
- **Post-timbrado:** con constancia SUNAT y número de orden OSE (si aplica).
- El diseño visual se documenta en TPSC-RE-FU-035.

---

## Parte E — MinIO

### E1 — Resolución del bucket Perú

```sql
SELECT b.BucketNombre, r.Clave AS Region
FROM dbo.RegionConfiguracionMinioBucket b
INNER JOIN dbo.Region r ON b.IdRegion = r.IdRegion
WHERE b.BucketClave = 'notas_credito' AND r.Clave = 'PER';
-- ⚠️ Verificar existencia. Si no existe, INSERT en cambio #6 del BD.md
```

### E2 — Rutas MinIO NC Perú

| Tipo archivo | Ruta |
|---|---|
| PDF | `notas-credito-per/notas_credito/{anio}/{mes}/{serie}-{correlativo}.pdf` |
| XML | `notas-credito-per/notas_credito/{anio}/{mes}/{serie}-{correlativo}.xml` |

Donde `{serie}` = ej. `FC01` y `{correlativo}` = ej. `00000001`.

### E3 — Flujo de persistencia

```
Finanzas llama PersistirNCPeruPdfService
  → ResolveBucket(Region='PER', BucketClave='notas_credito')
  → DocumentBuilder.Generate(TemplateKey='GOL_PER_NC', model)
  → MinIO.Upload(pdf, ruta_pdf)
  → MinIO.Upload(xml, ruta_xml)
  → DB: INSERT Archivo × 2
  → DB: UPDATE fccNotaCredito (Estado='VIGENTE', IdArchivoPdf, IdArchivoXml, IdCFDIGenerada)
  → DB: INSERT CorreoEnviado + ArchivoCorreoEnviado × 2
```

---

## Parte F — Sin ETL a Legacy

Perú **no transfiere datos a PCconnect** (mismo comportamiento que RE-029 Factura Perú). No hay tareas de SSIS para RE-033.

---

## Diferencias clave vs RE-032 (México)

| Aspecto | RE-032 México | RE-033 Perú |
|---|---|---|
| Servicio XML | `NCMexicoXmlBuilder` | `NCPeruXmlBuilder` (nuevo) |
| Servicio PDF | `NCMexicoPdfMappingService` | `NCPeruPdfMappingService` (nuevo) |
| Servicio persistencia | `PersistirNCMexicoPdfService` | `PersistirNCPeruPdfService` (nuevo) |
| Motivo endpoint | `GET /api/catalogos/motivos-cancelacion` | `GET /api/catalogos/motivos-nota-credito-sunat` |
| Cancelación SAT | POST cancelar-cfdi (condicional) | NO APLICA |
| Tipo de cambio XML | Del día del timbrado | `TipoCambioOrigen` de la factura origen |
| Referencia origen | `CFDIGeneradaRelacionado` (UUID) | `cac:BillingReference` en XML (serie-correlativo) |
| Empresas | GOL / MUN / PRO / PQF | Solo GOL (Golocaer S.A.C.) |
| ETL Legacy | Tareas T14–T16 (SSIS PCconnect) | Sin ETL |
| Brecha timbrado | TurboPac conocido | SUNAT pendiente de definir (B1 bloqueante) |

---

## Pendientes

| ID | Pendiente | Sección afectada |
|---|---|---|
| P1 | Modalidad de emisión SUNAT — define la arquitectura del endpoint de Timbrado (Parte C) | C1 — bloqueante |
| P2 | Motivos habilitados catálogo 09 (R16) — afecta DML y qué motivos se exponen en el endpoint | B1, B3, B4 |
| P3 | Origen de `cbc:Description` — auto-generado o capturado por el usuario | B5, NCPeruXmlBuilder |
| P4 | Serie y formato de folio NC Perú con PMO | C1, EmpresaFolio |
| P5 | Estructura respuesta timbrado SUNAT (número constancia, estado, QR) — compartida RE-029 | C1, B7 |
| P6 | Implementación Comunicación de Baja en R16 | Scope futuro — no en este requisito |
| P7 | Tipo de cambio en aplicación de NC en moneda extranjera en Validar Cobro | Validar Cobro (fuera de scope RE-033) |
| P8 | Régimen Tributario Perú emisor/receptor — etiqueta y valores en XML | NCPeruXmlBuilder |
| P9 | Plantilla correo NC Perú en Brevo | B7 — correo automático |
| P10 | Maquetas PDF NC Perú (TPSC-RE-FU-035) | Parte D |

---

## Brechas

| ID | Brecha | Impacto |
|---|---|---|
| B1 | **Modalidad emisión SUNAT no definida** (OSE/directa) — no se reutiliza TurboPac de México | **Bloqueante para T7 y T10.** Las demás tareas pueden avanzar. |
| B2 | **Estructura respuesta timbrado SUNAT desconocida** — constancia, número orden OSE, estado SUNAT | Depende de B1. Afecta PersistirNCPeruPdfService y template PDF post-timbrado. |
| B3 | **`fccNotaCredito.IdTPProformaPedido` NOT NULL** — heredada de RE-032 B1. Bloquea el ALTER TABLE si no fue resuelta. | Bloquea T1 si RE-032 B1 no está resuelta. |
| B4 | **Comunicación de Baja sin definir** — si se implementa, requiere tabla `fccComunicacionBaja` y endpoint adicional fuera de este requisito | Sin bloqueo en RE-033. Documentado como pendiente P6. |
