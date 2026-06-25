# Impacto en BD - Diseno y Generacion PDF Proforma Peru
**Requisito:** R16A-RE-FU-017
**Base de Datos:** ProquifaDotNet
**Version:** 2.0 (rev. 2026-06-25 tras OBS-032 + alineacion RE-FU-006 actualizado)

---

## Resumen
Generacion del PDF de Proforma al tramitar pedido Prepago sin FAA para clientes Peru.
Empresa emisora unica: Golocaer S.A.C. (Prefijo GOLPERU). Foliador global PRF compartido.
Normativa SUNAT: RUC, IGV 18%, CCI, moneda PEN.
PDF se genera bajo demanda, se persiste en Minio via tabla Archivo tras envio exitoso.
Logos se resuelven por Prefijo en DocumentBuilder, no en BD.

> **Precondicion OBS-032** — Mientras la facturacion / timbrado Peru no este habilitada productivamente
> (brechas B1-B5 + modulo timbrado SUNAT sin resolver), este flujo NO se ejecuta. No se persiste PDF, no se
> consume folio del SEQUENCE global, y no se generan pendientes para clientes Peru. Esta condicion debe
> validarse antes de invocar el endpoint de generacion en Finanzas. Ver R16A-RE-FU-017.md (Regla 0).

> **Nota — RE-FU-006 actualizado (OBS-013/014):** Mexico ya persiste `ClienteDatosBancarios.ReferenciaVigente`
> y casa snapshot a `tpProformaPedido.ReferenciaPago`. El modelo Peru debera adoptar el MISMO patron cuando
> la brecha B2 (REF.CLIENTE Peru) se resuelva: armar referencia al configurar la cuenta, persistirla, y
> casarla al PDF al generar. No reconstruccion dinamica.

---

## Impacto en BD

| #   | Cambio                                                                             | Tipo       | Prioridad      |
| --- | ---------------------------------------------------------------------------------- | ---------- | -------------- |
| 1   | Mismos objetos de RE-FU-016 (FolioProforma + SeqFolioProforma + vtpProformaPedido) | Compartido | Alta           |
| 2   | INSERT cuentas bancarias Golocaer SAC Peru en EmpresaDatosBancarios                | DML        | Alta (BRECHA)  |
| 3   | Definir modelo REF.CLIENTE Peru en ClienteDatosBancarios (RE-FU-006)               | Logica     | Alta (BRECHA)  |
| 4   | Capturar datos legales Golocaer SAC (direccion Peru) en Empresa                    | DML        | Media (BRECHA) |

> Reutiliza la misma infraestructura BD que RE-FU-016 (Proforma Mexico).
> El foliador es GLOBAL (un solo consecutivo compartido MEX + PER).
> No se requieren ALTER TABLE adicionales a los de RE-FU-016.
> Las BRECHAS son de DATOS (DML), no de ESTRUCTURA (DDL).

---

## Empresa Emisora Peru

| Campo | Valor en BD |
|-------|-------------|
| Prefijo | GOLPERU |
| Alias | Golocaer S.A.C |
| RazonSocial | GOLOCAER S.A.C. |
| Activo | True |
| Region | PER (8278ecd0-c337-4484-b008-5b5e65b0dfaf) |

> DocumentBuilder recibe Prefijo='GOLPERU' y traduce a logo/color Peru.
> Solo 1 empresa emite proformas Peru (vs 4 empresas en Mexico).

---

## Estado Actual de Datos Peru en BD

| Dato | Estado | Accion |
|------|--------|--------|
| Empresa GOLPERU | Existe y activa | OK |
| Cuentas bancarias Peru | **0 registros** | BRECHA B1 - insertar |
| Clientes Peru con datos fiscales (RUC) | 9 registros | OK - hay clientes |
| Bucket 'pedidos' region PER | Registrado en RegionConfiguracionMinioBucket | OK |
| Direccion legal Golocaer SAC | No capturada | BRECHA B3 |
| REF.CLIENTE logica Peru | No definida | BRECHA B2 |

---

## Patron de Almacenamiento (mismo que RE-FU-016)

    Archivo
        IdArchivo (PK)
        FileKey: varchar(600)   <- path en Minio
        FileBucket: varchar(100) <- bucket 'pedidos'
        IdRegion: FK Region = PER
        Sincronizado: bit

    tpPedido.IdArchivo -> Archivo.IdArchivo  (PDF persistido)

---

## Persistencia del PDF (identico a MEX)

| Etapa | BD | Minio |
|-------|-----|-------|
| Previsualizacion | Nada | Nada |
| Envio exitoso | INSERT Archivo (IdRegion=PER) + UPDATE tpPedido.IdArchivo + UPDATE tpProformaPedido.FolioProforma | Sube PDF bucket 'pedidos' PER |
| Consulta historica | SELECT Archivo.FileKey, FileBucket | Descarga PDF |

---

## Foliador Global PRF (compartido MEX + PER)

| Aspecto | Valor |
|---------|-------|
| Campo BD | tpProformaPedido.FolioProforma (varchar 80) |
| Formato interno BD | MMDDAA-Consecutivo |
| Formato visual PDF | PRF-MMDDAA-Consecutivo |
| Segmentacion | **Ninguna** - global MEX + PER |
| Mecanismo | SQL SEQUENCE dbo.SeqFolioProforma (mismo que MEX) |

---

## Diferencias MEX vs PER en Datos

| Aspecto | Mexico (RE-FU-016) | Peru (RE-FU-017) |
|---------|-------------------|-------------------|
| Impuesto | IVA 16% | IGV 18% |
| ID fiscal cliente | RFC (13 chars) | RUC (11 digitos) |
| Codigo interbancario | CLABE (18 digitos) | CCI (20 digitos) |
| Moneda local | MXN | PEN |
| Disclaimer legal | Art. 29/29A CFF (SAT) | Reglamento CP + R.S. 097-2012/SUNAT |
| Doc fiscal final | CFDI | CPE (Comprobante Pago Electronico) |
| Empresa emisora | GOL/MUN/PRO/PQF (4) | GOLPERU (1) |
| Sello NEEC | SI | NO (exclusivo Mexico) |
| Logo FEUM | SI | NO (exclusivo Mexico) |
| REF.CLIENTE | CodigoValidador definido | **NO DEFINIDO (BRECHA)** |

---

## Fuentes de Datos para el PDF Peru

| Seccion PDF | Tabla Fuente | Campos | Diferencia vs MEX |
|-------------|-------------|--------|-------------------|
| Cabecera - Logo/Color | Empresa (GOLPERU) | Prefijo -> DocumentBuilder | Logo unico GOLPERU |
| Cabecera - Folio | tpProformaPedido | FolioProforma | Mismo foliador global |
| Cabecera - Disclaimer | Constante | Texto SUNAT | Diferente texto |
| Cliente | Cliente + DatosFacturacionCliente | Alias/RazonSocial | Igual |
| Partidas | tpProformaPartidaPedido | IdProducto, Piezas, PU | Igual |
| Pago - Montos | tpProformaPedido | MontoTotal + IGV 18% | Tasa diferente |
| Pago - Moneda | DatosFacturacionCliente.IdCatMoneda | catMoneda (PEN/USD) | PEN vs MXN |
| Pago - Tipo cambio | tpPedido | TipoCambioFacturacion | TC SUNAT vs TC DOF |
| Pago - Condiciones | catCondicionesDePago | CondicionesDePago | Igual |
| Bancarios - Cuentas | EmpresaDatosBancarios + DatosBancarios | Banco, Cuenta, **CCI** | CCI vs CLABE |
| Bancarios - REF.CLIENTE | ClienteDatosBancarios? | **NO DEFINIDO** | BRECHA B2 |
| Facturacion | DatosFacturacionCliente | **RUC**, RazonSocial, Dir | RUC vs RFC |
| Entrega | tpPedido + DireccionCliente | Folio, Dir, Contacto | Igual |
| Pie | Empresa (GOLPERU) | RazonSocial legal Peru | Solo GOLPERU |

---

## Tablas Consultadas (Lectura) - Mismas que RE-FU-016

| Tabla | Rol |
|-------|-----|
| tpPedido | FolioPedidoInterno, IdEmpresa, IdRegion=PER, TipoCambio |
| tpProformaPedido | FolioProforma, MontoTotal, ReferenciaPago |
| tpPedidoProformaPedido | Vinculacion pedido-proforma |
| tpProformaPartidaPedido | Partidas |
| Producto + MarcaFamilia | Descripcion |
| Cliente | Nombre, Alias |
| DatosFacturacionCliente | RUC (campo RFC), RazonSocial, IdCatMoneda |
| DireccionCliente | Direccion: distrito/provincia/departamento |
| ContactoCliente | Contacto entrega |
| Empresa (GOLPERU) | RazonSocial legal, Direccion legal Peru |
| EmpresaDatosBancarios | Cuentas GOLPERU PER **(SIN DATOS)** |
| DatosBancarios | NumeroDeCuenta, CCI (campo Clabe), Sucursal |
| catBanco | Nombre banco peruano |
| catMoneda | ClaveMoneda (PEN/USD) |
| catCondicionesDePago | CondicionesDePago texto |
| Region | Filtro PER |
| Archivo | FileKey, FileBucket |
| RegionConfiguracionMinioBucket | Bucket 'pedidos' PER |

---

## Brechas Criticas (DATOS, no estructura)

> Numeracion alineada con `R16A-RE-FU-017.md` Brechas B1-B10 (post OBS-032 / 2026-06-25).

| # | Brecha | Impacto | Accion |
|---|--------|---------|--------|
| B1 | 0 cuentas bancarias GOLPERU en BD | PDF sin seccion bancaria | INSERT EmpresaDatosBancarios + DatosBancarios para bancos peruanos |
| B2 | REF.CLIENTE Peru no definida | PDF sin referencia de pago | Definir logica de identificacion de pagos para bancos peruanos. Cuando se defina, adoptar patron RE-FU-006 actualizado: persistir en `ClienteDatosBancarios.ReferenciaVigente` y casar al PDF (no reconstruccion dinamica). |
| B3 | Direccion legal y datos de contacto GOLPERU no capturados | Pie del PDF incompleto | Recopilar y UPDATE Empresa (GOLPERU): direccion legal Peru + telefonos + web + correo Peru |
| B4 | Disclaimer SUNAT no validado legalmente | Riesgo legal | Validar con asesor contable peruano |
| B5 | Detracciones/Percepciones no confirmadas | Posible omision regulatoria | Confirmar con asesor contable peruano. **Bloquea habilitacion productiva de Peru (OBS-032).** |
| B6 | Certificaciones GOLPERU Peru desconocidas | Pie del PDF incompleto | Confirmar ISO/metodos de pago Peru |
| B7 | Logos farmaceuticos Peru no definidos | Pie del PDF incompleto | Confirmar lista (USP, EDQM, Microbiologics) |
| B8 | Titulo: Proforma vs Factura Proforma | Ambiguedad documento | Confirmar con cliente |
| B9 | Nomenclatura: SOLES vs NUEVOS SOLES | Texto en letra incorrecto | Confirmar (SOLES es oficial desde 2015) |
| B10 | TC SUNAT compra/venta vs interno | Monto puede variar | Confirmar con finanzas |

---

## Campo RFC reutilizado como RUC

> La tabla DatosFacturacionCliente usa el campo **RFC varchar(50)** para almacenar
> tanto el RFC mexicano (13 chars) como el RUC peruano (11 digitos).
> No hay campo separado para RUC — se reutiliza RFC.
> El DocumentBuilder debe etiquetar como 'RUC' cuando Region = PER.

---

## Campo Clabe reutilizado como CCI

> La tabla DatosBancarios usa el campo **Clabe varchar(200)** para almacenar
> tanto la CLABE mexicana (18 digitos) como el CCI peruano (20 digitos).
> No hay campo separado para CCI — se reutiliza Clabe.
> El DocumentBuilder debe etiquetar como 'CCI' cuando Region = PER.

---

## Dependencias

| Requisito            | Relacion                                                                                                                                                |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R16A-RE-FU-016       | Comparte FolioProforma, SeqFolioProforma, vtpProformaPedido, patron Archivo                                                                             |
| R16A-RE-FU-001       | EmpresaDatosBancarios.IdRegion (filtrar cuentas PER)                                                                                                    |
| R16A-RE-FU-006       | ClienteDatosBancarios - logica REF.CLIENTE Peru (BRECHA B2). Adoptar el patron actualizado (ReferenciaVigente persistida + casado al PDF, OBS-013/014). |
| R16A-RE-FU-013       | Flujo Prepago con controlados (mismo PDF Peru) — bloqueado por OBS-032 hasta timbrado Peru                                                              |
| R16A-RE-FU-014       | Flujo Prepago sin controlados sin FAA (dispara este PDF) — bloqueado por OBS-032 hasta timbrado Peru                                                    |
| Modulo Timbrado Peru | **Precondicion bloqueante (OBS-032)**: mientras no este habilitado, este requisito no se ejecuta productivamente                                        |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
