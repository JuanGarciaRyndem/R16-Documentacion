# Impacto en Back - R16A-RE-FU-017
**Requisito:** Diseno y generacion de Documentos: Proforma Peru
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder
**Modulo:** L05.TramitarPedido + Proforma (Finanzas) + DocumentBuilder
**Impacto:** Generacion de PDF de Proforma Peru con adaptacion regional (IGV, RUC, CCI, PEN) + template unico GOLPERU + reutilizacion infraestructura RE-FU-016
**Version:** 2.0 (rev. 2026-06-25 tras OBS-032 + alineacion RE-FU-006 actualizado)

---

## Resumen

Este requisito implementa la **generacion del PDF de Proforma** para pedidos Prepago sin FAA de clientes con Region Peru. Reutiliza la **misma infraestructura** creada en RE-FU-016 (Proforma Mexico) con adaptaciones regionales.

**Diferencia principal vs RE-FU-016:** 1 sola empresa emisora (GOLPERU), normativa SUNAT (IGV 18%, RUC, CCI, PEN), 1 solo template HTML, foliador global compartido.

> **Precondicion OBS-032 — gating en el caller** — Toda la cadena de back (ApiCallerFinanzas en ProquifaDotNet → endpoint Finanzas → DocumentBuilder → Minio) NO se ejecuta para pedidos Region=PER mientras la facturacion / timbrado Peru no este habilitada productivamente. El gating se aplica en el caller (ProquifaDotNet) antes de invocar Finanzas, no dentro de Finanzas, para evitar consumir folio del SEQUENCE global ni generar pendientes. Ver `R16A-RE-FU-017.md` Regla 0.

> **Alineacion con RE-FU-006 actualizado (OBS-013/014):** la referencia bancaria ya NO se reconstruye dinamicamente. El patron Mexico es: armar referencia al CREATE/UPDATE de la asignacion cliente-cuenta, persistir en `ClienteDatosBancarios.ReferenciaVigente`, y casar snapshot a `tpProformaPedido.ReferenciaPago` al generar el PDF (inmutable). Cuando se resuelva la BRECHA B2 (REF.CLIENTE Peru), Peru debera adoptar el mismo patron: el ProformaModelBuilder LEE `ReferenciaVigente` (no recalcula) y la copia al DTO.

Involucra los mismos **3 repositorios/soluciones**:

1. **ProquifaDotNet** (Venta Interna, .NET Framework 4.8) - Dispara la generacion al tramitar, consume API de Finanzas
2. **ProquifaDotNet.Finanzas** (.NET Core 10) - Modulo Proforma: arma DTO con datos Peru, llama DocumentBuilder, persiste PDF en Minio
3. **DocumentBuilder** - Servicio de generacion: recibe DTO, renderiza template HTML GOLPERU, retorna PDF

### Flujo de integracion (identico a RE-FU-016, adaptado a Region PER)

1. ESAC presiona Tramitar (Prepago sin FAA, Peru)
2. ProquifaDotNet llama API Finanzas: POST /api/proforma/generar-pdf (IdRegion=PER)
3. Finanzas consulta datos BD (mismas tablas, filtro Region=PER)
4. Determina TemplateKey: GOLPERU_PER_PRO
5. Arma ProformaModel con adaptaciones Peru (IGV 18%, RUC, CCI, PEN, disclaimer SUNAT)
6. Llama DocumentBuilder: POST api/Report/proforma
7. DocumentBuilder renderiza template GOLPERU_PER_PRO -> retorna byte[]
8. Finanzas retorna PDF a ProquifaDotNet (previsualizacion)
9. ESAC acepta -> confirma envio
10. Finanzas: SEQUENCE -> Minio (bucket pedidos, IdRegion=PER) -> Archivo -> tpPedido

---

## Reutilizacion de RE-FU-016 (Proforma Mexico)

| Componente | Reutiliza de RE-FU-016 | Adaptacion Peru |
|-----------|----------------------|-----------------|
| Foliador SeqFolioProforma | SI - mismo SEQUENCE global | Ninguna (folio compartido MEX+PER) |
| Endpoint POST /api/proforma/generar-pdf | SI - mismo endpoint | Parametro IdRegion=PER determina flujo |
| Endpoint GET /api/proforma/{id}/pdf | SI - mismo endpoint | Ninguna |
| ProformaModel DTO | SI - misma estructura base | Campos adaptados: IGV vs IVA, RUC vs RFC, CCI vs CLABE, PEN vs MXN |
| ProformaModelBuilder (BO armado) | SI - mismo servicio | Logica condicional por Region |
| DocumentBuilderHttpClient | SI - mismo cliente | Ninguna |
| MinioStorageService | SI - mismo servicio | Bucket pedidos con IdRegion=PER |
| ApiCallerFinanzas (ProquifaDotNet) | SI - mismo caller | Condicion: tambien para Region PER |
| Flujo transaccional confirmacion | SI - mismo command | Ninguna |
| MontoALetrasConverter | SI - mismo servicio | Agregar soporte PEN (SOLES XX/100) |
| Template HTML | NO - nuevo template | Crear GOLPERU_PER_PRO (1 empresa, 3 archivos) |
| TemplateKey | NO - nuevo key | GOLPERU_PER_PRO |
| Codigo Validador (REF.CLIENTE) | NO - logica diferente | BRECHA: logica Peru no definida |
| Disclaimer legal | NO - texto diferente | Texto SUNAT (vs texto SAT Mexico) |

---

## Adaptaciones Regionales en ProformaModelBuilder

El BO ProformaModelBuilder (creado en RE-FU-016) debe incorporar logica condicional cuando Region=PER:

| Seccion DTO | Campo | Mexico (RE-FU-016) | Peru (RE-FU-017) |
|------------|-------|-------------------|-------------------|
| header.disclaimer | texto | Art. 29/29A CFF (SAT) | Reglamento CP + R.S. 097-2012/SUNAT |
| pago.impuestoPorcentaje | label + tasa | IVA 16% | IGV 18% |
| pago.impuestoLabel | etiqueta | IVA | IGV |
| pago.monedaLocal | clave | MXN (M.N.) | PEN (S/.) |
| pago.granTotalEnLetra | sufijo | PESOS XX/100 M.N. | SOLES XX/100 |
| pago.leyendaExhibicion | texto | PAGO EN UNA SOLA EXHIBICION | OPERACION AL CONTADO (pendiente confirmar) |
| datosBancarios.cuentas | estructura | 2 cuentas (MN + DLS) CLABE | N cuentas (PEN + USD) CCI |
| datosBancarios.labelInterbancario | etiqueta | CLABE | CCI |
| datosBancarios.refCliente | logica | CodigoValidador (Banamex/otros) | BRECHA - no definida |
| facturacion.labelIdFiscal | etiqueta | RFC | RUC |
| facturacion.direccion | formato | Colonia/Ciudad/Estado | Distrito/Provincia/Departamento |
| entrega.direccion | formato | Formato MEX | Formato PER (distrito/provincia/depto) |
| footer.sellos | contenido | NEEC + FEUM + USP + EDQM + Microbiologics | USP + EDQM + Microbiologics (sin NEEC/FEUM) |
| footer.contacto | datos | Datos MEX por empresa | Datos GOLPERU Peru (BRECHA - no capturados) |
| templateKey | valor | {Prefijo}_MEX_PRO (4 opciones) | GOLPERU_PER_PRO (1 opcion) |

---

## DTO de Proforma Peru (Data para DocumentBuilder)

Misma estructura que Mexico, con valores adaptados:

| Seccion | Diferencia vs Mexico |
|---------|---------------------|
| header.disclaimer | Texto SUNAT en lugar de texto SAT |
| pago.impuestoLabel | IGV (en vez de IVA) |
| pago.impuestoPorcentaje | 18% (en vez de 16%) |
| pago.moneda | S/. para PEN (en vez de $ M.N.) |
| pago.granTotalEnLetra | SOLES XX/100 (en vez de PESOS XX/100 M.N.) |
| pago.leyendaExhibicion | OPERACION AL CONTADO (pendiente confirmar) |
| datosBancarios.label | CCI de 20 digitos (en vez de CLABE 18 digitos) |
| datosBancarios.refCliente | BRECHA - logica Peru no definida |
| facturacion.labelIdFiscal | RUC (en vez de RFC) |
| facturacion.direccion | Distrito/Provincia/Departamento |
| templateKey | GOLPERU_PER_PRO (1 unico template) |
| footer.sellos | Sin NEEC ni FEUM (exclusivos Mexico) |

---

## Impacto en BD (segun R16A-RE-FU-017_BD.md v1.0)

**No hay cambios DDL (estructura).** Solo DML (datos):

| # | Cambio | Tipo | Origen |
|---|--------|------|--------|
| 1 | Reutiliza FolioProforma + SeqFolioProforma + vtpProformaPedido | Compartido RE-FU-016 | Ya creado |
| 2 | INSERT cuentas bancarias GOLPERU en EmpresaDatosBancarios + DatosBancarios | DML (BRECHA B1) | Datos no existen |
| 3 | Definir modelo REF.CLIENTE Peru | Logica (BRECHA B2) | No definido |
| 4 | UPDATE Empresa GOLPERU con direccion legal Peru | DML (BRECHA B3) | Dato no capturado |

### Campos reutilizados con doble semantica

| Tabla | Campo | Mexico | Peru |
|-------|-------|--------|------|
| DatosFacturacionCliente | RFC varchar(50) | RFC 13 chars | RUC 11 digitos |
| DatosBancarios | Clabe varchar(200) | CLABE 18 digitos | CCI 20 digitos |

> DocumentBuilder etiqueta como RUC o CCI segun Region del pedido.

---

## Gaps de Desarrollo

### En ProquifaDotNet.Finanzas (Adaptacion regional)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Logica condicional Region en ProformaModelBuilder | Agregar IF Region=PER para: tasa impuesto, label, moneda, disclaimer, formato direccion | Medio |
| GAP-02 | Soporte PEN en MontoALetrasConverter | Agregar caso moneda PEN: SOLES XX/100 | Bajo |
| GAP-03 | Logica REF.CLIENTE Peru (BRECHA) | Implementar cuando se defina (banco peruano no usa CodigoValidador) | Medio |
| GAP-04 | Determinacion TemplateKey Peru | Si Region=PER -> templateKey = GOLPERU_PER_PRO (no usar Empresa.Prefijo + _MEX_PRO) | Bajo |
| GAP-05 | Consulta cuentas bancarias filtro Region PER | EmpresaDatosBancarios WHERE IdEmpresa=GOLPERU AND IdRegion=PER | Bajo |
| GAP-06 | Sellos/certificaciones Peru (sin NEEC, sin FEUM) | Condicional en DTO footer por Region | Bajo |
| GAP-07 | Leyenda exhibicion Peru | Texto OPERACION AL CONTADO (u omitir) segun Region | Bajo |

### En ProquifaDotNet (Venta Interna)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-08 | Ampliar condicion en flujo tramitacion con **gating por OBS-032** | ApiCallerFinanzas ya llama para MEX; agregar `OR (Region=PER AND FeatureFlag.TimbradoPeruHabilitado = true)`. Mientras la facturacion Peru no este habilitada, NO se invoca el endpoint de Finanzas para pedidos Peru (precondicion OBS-032). | Bajo |
| GAP-08b | Configurar FeatureFlag/setting `TimbradoPeruHabilitado` | Definir mecanismo (appSettings, BD de configuracion, IdentityServer claim) para alternar gating Peru sin redeploy cuando se resuelvan las brechas B1-B5 + modulo timbrado | Bajo |

> Nota: El ApiCallerFinanzas creado en RE-FU-016 (GAP-11) ya existe. Aqui se amplia la condicion de Region **con la guarda OBS-032**: el caller debe verificar el flag de Peru habilitado antes de delegar a Finanzas.

### En DocumentBuilder (Servicio externo)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-09 | Crear template HTML GOLPERU_PER_PRO (3 archivos) | _H.html, _B.html, _F.html con branding Peru (sin NEEC, sin FEUM, logo GOLPERU Peru) | Alto |
| GAP-10 | Diseno HTML/CSS template Peru | Variante visual unica: color institucional GOLPERU Peru, layout adaptado (CCI en vez de CLABE, RUC en vez de RFC, IGV en vez de IVA) | Alto |
| GAP-11 | Registrar template en BD DocumentBuilder | INSERT DocumentTemplate para GOLPERU_PER_PRO | Bajo |
| GAP-12 | Logo GOLPERU Peru | Preparar logo en formato base64/asset para operacion Peru | Bajo |
| GAP-13 | Logos farmaceuticos Peru (sin FEUM) | USP + EDQM + Microbiologics (pendiente confirmar lista exacta) | Bajo |

> Nota: El endpoint POST api/Report/proforma, el servicio ProformaExtension y el DTO DocumentGenerateProformaDto ya existen (creados en RE-FU-016). Solo se agrega 1 template nuevo.

### Datos (DML - Scripts de insercion)

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-14 | INSERT cuentas bancarias GOLPERU Peru | EmpresaDatosBancarios + DatosBancarios (banco peruano, CCI, cuentas PEN y USD) | Bajo |
| GAP-15 | UPDATE direccion legal Empresa GOLPERU | Capturar y actualizar direccion legal completa Peru en tabla Empresa | Bajo |
| GAP-16 | Datos contacto GOLPERU Peru | Telefonos, web, correo ventas Peru (pendiente recopilar) | Bajo |

---

## Brechas Criticas (Bloqueantes)

> Numeracion alineada con `R16A-RE-FU-017.md` B1-B10 (OBS-032 / 2026-06-25). La precondicion OBS-032 implica que **todas las brechas B1-B5** son bloqueantes para activar el gating descrito en GAP-08; B6-B10 se pueden resolver en paralelo pero no son condicion de habilitacion.

| # | Brecha | Impacto en Back | Estado |
|---|--------|----------------|--------|
| B1 | 0 cuentas bancarias GOLPERU Peru en BD | ProformaModelBuilder no puede armar seccion datosBancarios | Bloqueante OBS-032 |
| B2 | REF.CLIENTE Peru no definida | Campo refCliente queda vacio o con placeholder. Cuando se defina, **adoptar patron RE-FU-006**: persistir en `ClienteDatosBancarios.ReferenciaVigente` y leer (no recalcular) en `ProformaModelBuilder`. | Bloqueante OBS-032 |
| B3 | Direccion legal y contacto GOLPERU Peru no capturados | Footer del PDF incompleto | Bloqueante OBS-032 |
| B4 | Disclaimer SUNAT no validado legalmente | Riesgo legal en texto del PDF | Bloqueante OBS-032 |
| B5 | Detracciones/Percepciones SUNAT no confirmadas | Posible omision regulatoria que invalide CPE posterior | Bloqueante OBS-032 |
| B6 | Certificaciones GOLPERU Peru desconocidas | Footer incompleto | Media - no bloquea gating |
| B7 | Logos farmaceuticos Peru no definidos | Footer incompleto | Baja - puede resolverse con assets existentes |
| B8 | Titulo: Proforma vs Factura Proforma | Ambiguedad en cabecera | Baja - decision UI |
| B9 | Nomenclatura SOLES vs NUEVOS SOLES | Texto en letra | Baja - MontoALetrasConverter |
| B10 | TC SUNAT vs TC interno | Calculo en `ProformaModelBuilder` | Media - finanzas |

---

## Template DocumentBuilder Peru

### Template a CREAR

| TemplateKey | Empresa | Archivos |
|-------------|---------|----------|
| GOLPERU_PER_PRO | Golocaer S.A.C. | _H.html, _B.html, _F.html |

> Solo 1 template (vs 4 en Mexico). Unica empresa del grupo en Peru.

### Variante visual Peru

| Aspecto | Valor |
|---------|-------|
| Logo | Logo GOLPERU Peru (operacion SAC) |
| Color | Color institucional GOLPERU Peru (pendiente confirmar si es mismo naranja que GOL Mexico) |
| Sellos pie | USP + EDQM + Microbiologics (sin NEEC, sin FEUM) |
| Datos bancarios | CCI 20 digitos (en vez de CLABE) |
| ID fiscal | RUC (en vez de RFC) |
| Impuesto | IGV 18% (en vez de IVA 16%) |
| Moneda local | S/. PEN (en vez de $ MXN) |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-016 | Prerequisito: toda la infraestructura Proforma (Finanzas, DocumentBuilder endpoint, foliador, Minio, flujo transaccional) |
| R16A-RE-FU-006 | ClienteDatosBancarios - logica REF.CLIENTE Peru (BRECHA B2). El modelo actualizado (OBS-013/014) persiste `ReferenciaVigente`; Peru debera seguir el mismo patron cuando B2 cierre. |
| R16A-RE-FU-001 | EmpresaDatosBancarios.IdRegion (filtrar cuentas Peru) |
| R16A-RE-FU-018 | El listado de FxA tambien aplica OBS-032 (no listar pedidos Peru pendientes si timbrado Peru no esta habilitado). Coordinar el feature flag con esta misma fuente. |
| Modulo Timbrado Peru | **Precondicion bloqueante (OBS-032)**: hasta que este modulo + brechas B1-B5 esten resueltas, el gating en GAP-08 mantiene el flujo desactivado para Peru. |

---

## Resumen de Gaps

| Repositorio | Cantidad | Detalle |
|-------------|----------|---------|
| ProquifaDotNet.Finanzas | 7 (GAP-01 a GAP-07) | Adaptaciones regionales en logica existente |
| ProquifaDotNet | 2 (GAP-08, GAP-08b) | Ampliar condicion Region en flujo tramitacion + FeatureFlag TimbradoPeruHabilitado (OBS-032) |
| DocumentBuilder | 5 (GAP-09 a GAP-13) | 1 template nuevo + registro BD + logos |
| Datos (DML) | 3 (GAP-14 a GAP-16) | Scripts INSERT/UPDATE para datos Peru |
| **Total** | **17 gaps** | Mayoria bajo/medio - reutiliza RE-FU-016. GAP-08/08b implementan el gating OBS-032. |

---

## Estimacion de Esfuerzo Comparativo

| Aspecto | RE-FU-016 (Mexico) | RE-FU-017 (Peru) |
|---------|-------------------|-------------------|
| Solucion nueva | SI (crear Finanzas completa) | NO (reutiliza) |
| Templates HTML | 4 x 3 = 12 archivos | 1 x 3 = 3 archivos |
| Endpoints nuevos | SI (generar-pdf, consulta, CRUD) | NO (mismos endpoints) |
| Flujo transaccional | SI (crear desde cero) | NO (reutiliza) |
| Logica condicional | N/A | SI (IF Region=PER) |
| Cambios DDL | SI (ALTER, SEQUENCE, VIEW) | NO (solo DML) |
| Esfuerzo relativo | 100% (base) | ~25-30% del esfuerzo Mexico |
