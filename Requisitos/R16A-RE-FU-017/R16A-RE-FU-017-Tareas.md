# Tareas BackEnd - R16A-RE-FU-017
**Requisito:** Diseno y generacion de Documentos: Proforma Peru
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder
**Version:** 2.0 (rev. 2026-06-25 tras OBS-032 + alineacion RE-FU-006 actualizado)

> **Precondicion OBS-032** — Mientras la facturacion / timbrado Peru no este habilitada productivamente
> (brechas B1-B5 + modulo timbrado SUNAT sin resolver), ninguna de estas tareas debe activarse en
> produccion. El gating se implementa en la Tarea 4 (FeatureFlag `TimbradoPeruHabilitado`). El resto
> de tareas (1, 2, 3, 5, 6, 7) puede desarrollarse y testearse en DEV/QA sin impacto productivo.
>
> **Alineacion con RE-FU-006 actualizado (OBS-013/014)** — Cuando la brecha B2 (REF.CLIENTE Peru) se
> resuelva, la implementacion debe seguir el patron Mexico actualizado: armar la referencia al
> CREATE/UPDATE de la asignacion cliente-cuenta, persistirla en `ClienteDatosBancarios.ReferenciaVigente`,
> y leerla (no recalcular) desde el `ProformaModelBuilder` al armar el DTO. Snapshot inmutable al PDF.

---

### Tarea 1

**Titulo:** [ R16A-RE-FU-017 ] [ALG-COMPLX-LOGIC] Implementar logica condicional Region Peru en ProformaModelBuilder

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Application/Services (ProformaModelBuilder)

**Consideraciones previas:**
- Depende de RE-FU-016 (ProformaModelBuilder ya creado para Mexico)
- El BO existente arma el DTO para Mexico; se debe agregar logica condicional cuando Region=PER
- 1 sola empresa emisora Peru: GOLPERU (Golocaer S.A.C.)
- TemplateKey Peru: GOLPERU_PER_PRO (no usa Empresa.Prefijo + _MEX_PRO)

**Objetivo general:**
Agregar logica condicional por Region en el ProformaModelBuilder para generar el DTO con datos adaptados a normativa peruana (IGV, RUC, CCI, PEN, disclaimer SUNAT).

**Objetivos especificos:**
- Agregar condicion IF Region=PER para determinar TemplateKey = GOLPERU_PER_PRO
- Cambiar label de impuesto: IGV en vez de IVA
- Cambiar tasa impuesto: 18% en vez de 16%
- Cambiar disclaimer: texto SUNAT (Reglamento CP + R.S. 097-2012/SUNAT) en vez de texto SAT
- Cambiar formato moneda local: S/. PEN en vez de $ M.N.
- Cambiar label interbancario: CCI en vez de CLABE
- Cambiar label ID fiscal: RUC en vez de RFC
- Cambiar formato direccion: Distrito/Provincia/Departamento en vez de Colonia/Ciudad/Estado
- Cambiar leyenda exhibicion: OPERACION AL CONTADO (o omitir, pendiente confirmar)
- Excluir sellos NEEC y FEUM del footer (exclusivos Mexico)
- Incluir solo sellos aplicables Peru: USP + EDQM + Microbiologics

**Resultado esperado:**
ProformaModelBuilder genera DTO correcto tanto para Region MEX como para Region PER con un unico punto de entrada.

**Entregables:**
- Logica condicional en ProformaModelBuilder por Region
- Tests unitarios para armado DTO Peru

**Criterios de aceptacion:**
- Para Region=PER el DTO contiene IGV 18%, RUC, CCI, PEN, disclaimer SUNAT, GOLPERU_PER_PRO
- Para Region=MEX el DTO sigue funcionando identico (sin regresion)
- El templateKey se determina correctamente segun Region
- Los sellos NEEC/FEUM no aparecen en DTO Peru

**Mas informacion de la tarea:**
Corresponde a GAP-01, GAP-04, GAP-06 y GAP-07 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-017-Back.md
- R16A-RE-FU-017_BD.md

---

### Tarea 2

**Titulo:** [ R16A-RE-FU-017 ] [ALG-BASIC-LOGIC] Agregar soporte moneda PEN en MontoALetrasConverter

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Application/Utilities

**Consideraciones previas:**
- Depende de RE-FU-016 Tarea 13 (MontoALetrasConverter ya creado con soporte MXN y USD)
- Formato PEN: (MONTO EN LETRAS SOLES XX/100)
- Nomenclatura oficial desde 2015: SOLES (no NUEVOS SOLES)

**Objetivo general:**
Agregar el caso de moneda PEN al servicio MontoALetrasConverter existente.

**Objetivos especificos:**
- Agregar caso moneda PEN con sufijo SOLES XX/100
- Mantener compatibilidad con MXN (PESOS XX/100 M.N.) y USD (DOLARES XX/100)
- Agregar tests unitarios para conversion PEN

**Resultado esperado:**
MontoALetrasConverter soporta 3 monedas: MXN, USD y PEN.

**Entregables:**
- Caso PEN agregado en MontoALetrasConverter
- Tests unitarios PEN

**Criterios de aceptacion:**
- Monto 3540.00 PEN genera: TRES MIL QUINIENTOS CUARENTA SOLES 00/100
- Monto 1250.75 PEN genera: MIL DOSCIENTOS CINCUENTA SOLES 75/100
- Los casos MXN y USD siguen funcionando (sin regresion)

**Mas informacion de la tarea:**
Corresponde a GAP-02 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-017-Back.md

---

### Tarea 3

**Titulo:** [ R16A-RE-FU-017 ] [ALG-BASIC-LOGIC] Implementar logica REF.CLIENTE Peru y consulta cuentas bancarias GOLPERU

**Aplicativos:** ProquifaDotNet.Finanzas

**Modulos:** Application/Services, Infrastructure/Persistence

**Consideraciones previas:**
- Depende de Tarea 1 (logica condicional Region en ProformaModelBuilder)
- BRECHA B1: No existen cuentas bancarias GOLPERU Peru en BD (requiere GAP-14 DML previo)
- BRECHA B2: La logica REF.CLIENTE Peru no esta definida (bloqueante de negocio)
- En Mexico (RE-FU-006 actualizado, OBS-013/014): la referencia se persiste en `ClienteDatosBancarios.ReferenciaVigente` al CREATE/UPDATE de la asignacion y se casa snapshot a `tpProformaPedido.ReferenciaPago` al generar el PDF. El BO del ProformaModelBuilder **LEE** la referencia, no la recalcula.
- En Peru: logica de armado diferente (pendiente definicion por el cliente), pero el **patron de persistencia/lectura debe ser el mismo** que Mexico.
- Consulta cuentas: EmpresaDatosBancarios WHERE IdEmpresa=GOLPERU AND IdRegion=PER

**Objetivo general:**
Implementar la consulta de cuentas bancarias peruanas y la logica de referencia bancaria del cliente para Peru.

**Objetivos especificos:**
- Agregar filtro IdRegion=PER en consulta EmpresaDatosBancarios del ProformaModelBuilder
- Implementar logica REF.CLIENTE Peru (cuando se defina por negocio)
- Mapear campo Clabe como CCI en el DTO cuando Region=PER
- Soportar N cuentas (PEN + USD) en vez de las 2 fijas de Mexico (MN + DLS)

**Resultado esperado:**
Seccion datosBancarios del DTO se arma correctamente con cuentas peruanas y referencia del cliente.

**Entregables:**
- Logica de consulta cuentas bancarias filtrada por Region
- Logica REF.CLIENTE Peru (implementacion o placeholder segun estado de brecha)
- Tests unitarios

**Criterios de aceptacion:**
- El DTO Peru contiene cuentas con label CCI (no CLABE)
- Las cuentas consultadas corresponden a GOLPERU con IdRegion=PER
- Si la brecha B2 no se resuelve, el campo refCliente muestra placeholder controlado
- No afecta consulta de cuentas Mexico (sin regresion)

**Mas informacion de la tarea:**
Corresponde a GAP-03 y GAP-05 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-017-Back.md
- R16A-RE-FU-017_BD.md (Brechas B1, B2)

---

### Tarea 4

**Titulo:** [ R16A-RE-FU-017 ] [IMP-EXIST-SERVICE] Ampliar condicion Region Peru en flujo de tramitacion ProquifaDotNet con OBS-032

**Aplicativos:** ProquifaDotNet (.NET Framework 4.8)

**Modulos:** Logic.Pqf.Logistica/L05.TramitarPedido + Configuracion (FeatureFlag)

**Consideraciones previas:**
- Depende de RE-FU-016 Tarea 15 (ApiCallerFinanzas ya creado para Mexico)
- Actualmente la llamada a API Finanzas se ejecuta solo para Region=MEX
- Se debe ampliar la condicion para incluir Region=PER **con guarda OBS-032**: el flujo solo se activa cuando la facturacion Peru esta habilitada (FeatureFlag `TimbradoPeruHabilitado`)
- Mientras el flag este apagado, los pedidos Prepago de clientes Peru NO disparan la generacion de Proforma ni consumen folio del SEQUENCE global ni generan pendientes (evita ruido operativo solicitado por el cliente)
- El resto del flujo es identico al de Mexico (mismo endpoint, mismo byte[] de retorno)

**Objetivo general:**
Ampliar la condicion en el flujo de tramitacion de ProquifaDotNet para que invoque la API de Finanzas cuando el pedido es Prepago sin FAA de Region Peru **y** la facturacion Peru esta habilitada productivamente.

**Objetivos especificos:**
- Definir el FeatureFlag `TimbradoPeruHabilitado` en mecanismo de configuracion (appSettings, BD de configuracion o claim IdentityServer) — debe poder alternarse sin redeploy cuando se cierren brechas B1-B5 + modulo timbrado SUNAT
- Modificar condicion en flujo L05.TramitarPedido: de `(Region=MEX)` a `(Region=MEX) OR (Region=PER AND FeatureFlag.TimbradoPeruHabilitado = true)`
- Pasar IdRegion=PER en la llamada a API Finanzas (solo cuando se pasa la guarda)
- Verificar que el byte[] del PDF Peru se retorna correctamente al Front
- Documentar el flag en el archivo de configuracion con valor inicial `false`

**Resultado esperado:**
Al tramitar pedido Prepago sin FAA de cliente Peru con el flag habilitado, se genera PDF de Proforma Peru via API Finanzas. Con el flag apagado, el flujo se ignora silenciosamente.

**Entregables:**
- Modificacion de condicion en flujo tramitacion L05 con guarda OBS-032
- FeatureFlag `TimbradoPeruHabilitado` configurado e inicializado en `false`
- Paso de parametro IdRegion al endpoint cuando aplica
- Documentacion del flag y su procedimiento de activacion

**Criterios de aceptacion:**
- Pedidos Prepago sin FAA Region=PER con flag=true generan PDF via Finanzas
- Pedidos Prepago sin FAA Region=PER con flag=false NO invocan Finanzas, NO consumen folio, NO generan pendiente
- Pedidos Prepago sin FAA Region=MEX siguen funcionando (sin regresion, sin dependencia del flag)
- Pedidos Credito o con FAA no disparan la generacion (sin cambios)
- El flag puede alternarse sin redeploy

**Mas informacion de la tarea:**
Corresponde a GAP-08 y GAP-08b del documento de impacto Back. Implementa la precondicion OBS-032 (no generar Proforma Peru mientras la facturacion Peru no este habilitada).

**Recursos:**
- R16A-RE-FU-017.md (Regla 0 — Precondicion OBS-032)
- R16A-RE-FU-017-Back.md (GAP-08, GAP-08b)
- Logic.Pqf.Catalogos/ApiCaller/ApiCallerFinanzas.cs (creado en RE-FU-016)

---

### Tarea 5

**Titulo:** [ R16A-RE-FU-017 ] [CREATE-PDF] Crear template HTML de Proforma Peru GOLPERU_PER_PRO (3 archivos)

**Aplicativos:** DocumentBuilder

**Modulos:** Templates

**Consideraciones previas:**
- Depende de RE-FU-016 Tarea 16 (servicio ProformaExtension ya creado)
- 1 sola empresa: GOLPERU (Golocaer S.A.C.)
- 3 archivos: GOLPERU_PER_PRO_H.html (Header), GOLPERU_PER_PRO_B.html (Body), GOLPERU_PER_PRO_F.html (Footer)
- Variante visual unica: logo GOLPERU Peru, color institucional
- Sin sellos NEEC ni FEUM (exclusivos Mexico)
- Etiquetas adaptadas: CCI, RUC, IGV, S/., SOLES

**Objetivo general:**
Crear el template HTML de Proforma para la operacion Peru con el branding de Golocaer S.A.C.

**Objetivos especificos:**
- Crear carpeta GOLPERU_PER_PRO/ con 3 archivos HTML
- Header: logo GOLPERU Peru, titulo Proforma, folio, vigencia, disclaimer SUNAT
- Body: tabla partidas, panel 4 columnas (pago con IGV, bancarios con CCI, facturacion con RUC, entrega)
- Footer: razon social legal GOLPERU Peru, contacto Peru, sellos (USP, EDQM, Microbiologics), paginacion X/Y
- CSS embebido con color institucional GOLPERU Peru
- Placeholders Handlebars/Mustache para datos dinamicos del ProformaModel

**Resultado esperado:**
Template HTML funcional que renderiza correctamente la Proforma Peru.

**Entregables:**
- 3 archivos HTML en carpeta GOLPERU_PER_PRO/
- CSS embebido con variante de color Peru
- Placeholders para datos dinamicos

**Criterios de aceptacion:**
- El template renderiza correctamente con datos de prueba Peru
- El logo corresponde a GOLPERU Peru
- Las etiquetas muestran: RUC (no RFC), CCI (no CLABE), IGV (no IVA), S/. (no $)
- El disclaimer es el texto SUNAT
- No aparecen sellos NEEC ni FEUM
- El panel inferior muestra 4 columnas con datos peruanos

**Mas informacion de la tarea:**
Corresponde a GAP-09 y GAP-10 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-017-Back.md
- R16A-RE-FU-017.md (criterios de aceptacion secciones A-H)

---

### Tarea 6

**Titulo:** [ R16A-RE-FU-017 ] [BD-OBJ-CH] Registrar template Peru en BD DocumentBuilder y preparar logo GOLPERU

**Aplicativos:** DocumentBuilder

**Modulos:** Base de Datos, Assets

**Consideraciones previas:**
- Depende de Tarea 5 (template HTML creado)
- La tabla DocumentTemplate almacena los registros de templates disponibles
- Patron existente: registros GOL/MUN/PQF/PRO_MEX_PRO ya existen (RE-FU-016)
- Logo GOLPERU Peru puede ser diferente al logo GOL Mexico

**Objetivo general:**
Registrar el template GOLPERU_PER_PRO en la base de datos de DocumentBuilder y preparar el logo de la operacion Peru.

**Objetivos especificos:**
- INSERT en tabla DocumentTemplate para GOLPERU_PER_PRO
- Preparar logo GOLPERU Peru en formato adecuado (base64 o path)
- Preparar logos farmaceuticos Peru (USP, EDQM, Microbiologics - sin FEUM)
- Verificar que DocumentBuilder resuelve correctamente el template por TemplateKey

**Resultado esperado:**
Template registrado en BD y assets graficos disponibles para renderizacion.

**Entregables:**
- Script SQL con INSERT en DocumentTemplate
- Logo GOLPERU Peru preparado
- Logos farmaceuticos Peru preparados

**Criterios de aceptacion:**
- El template GOLPERU_PER_PRO se encuentra en BD con TemplateKey correcto
- DocumentBuilder resuelve correctamente el template por su key
- El logo se renderiza correctamente en el PDF generado
- Los logos farmaceuticos Peru aparecen en el pie (sin NEEC/FEUM)

**Mas informacion de la tarea:**
Corresponde a GAP-11, GAP-12 y GAP-13 del documento de impacto Back.

**Recursos:**
- R16A-RE-FU-017-Back.md

---

### Tarea 7

**Titulo:** [ R16A-RE-FU-017 ] [CREATE-SCRIPT-CONTROL] Scripts DML para datos GOLPERU Peru (cuentas bancarias, direccion, contacto)

**Aplicativos:** ProquifaDotNet (Base de Datos)

**Modulos:** Base de Datos - Scripts DML

**Consideraciones previas:**
- BRECHA B1: 0 cuentas bancarias GOLPERU Peru en BD
- BRECHA B3: Direccion legal y datos de contacto GOLPERU Peru no capturados (brecha consolidada en v2.0: incluye telefonos, web, correo ventas)
- Requiere datos proporcionados por el cliente (bancos peruanos, direccion legal, telefonos)
- Tablas: EmpresaDatosBancarios, DatosBancarios, Empresa
- Esta tarea **desbloquea parte de la precondicion OBS-032** (junto con B2, B4 y B5 + modulo timbrado)

**Objetivo general:**
Crear los scripts DML para insertar los datos de GOLPERU Peru necesarios para la generacion de la Proforma.

**Objetivos especificos:**
- INSERT en EmpresaDatosBancarios para cuentas GOLPERU Peru (PEN + USD)
- INSERT en DatosBancarios con datos de banco peruano (NumeroDeCuenta, CCI en campo Clabe, Sucursal)
- UPDATE Empresa GOLPERU con direccion legal completa Peru
- UPDATE/INSERT datos de contacto Peru (telefonos, web, correo ventas)
- Vincular cuentas con IdRegion=PER

**Resultado esperado:**
Datos Peru disponibles en BD para que ProformaModelBuilder pueda armar la seccion bancaria y footer.

**Entregables:**
- Script SQL INSERT cuentas bancarias GOLPERU Peru
- Script SQL UPDATE direccion legal Empresa GOLPERU
- Script SQL datos contacto GOLPERU Peru
- Documentacion de datos insertados

**Criterios de aceptacion:**
- Existen al menos 2 cuentas bancarias GOLPERU Peru (PEN + USD)
- Las cuentas tienen CCI de 20 digitos en campo Clabe
- La empresa GOLPERU tiene direccion legal Peru completa
- La empresa GOLPERU tiene datos de contacto Peru (telefonos, web, correo ventas)
- Los datos son consultables con filtro IdRegion=PER
- No afectan datos Mexico existentes

**Mas informacion de la tarea:**
Corresponde a GAP-14, GAP-15 y GAP-16 del documento de impacto Back. Resuelve brechas B1 y B3 (en numeracion v2.0). Junto con resolver B2, B4 y B5 + modulo timbrado SUNAT, habilita el activado del FeatureFlag `TimbradoPeruHabilitado` de la Tarea 4.

**Recursos:**
- R16A-RE-FU-017-Back.md
- R16A-RE-FU-017_BD.md (Brechas B1, B3)

---

## Resumen de Tareas

| # | Clave | Titulo | Aplicativo | Predecesora |
|---|-------|--------|-----------|-------------|
| 1 | ALG-COMPLX-LOGIC | Logica condicional Region Peru en ProformaModelBuilder | Finanzas | RE-FU-016 (completo) |
| 2 | ALG-BASIC-LOGIC | Soporte moneda PEN en MontoALetrasConverter | Finanzas | RE-FU-016 T13 |
| 3 | ALG-BASIC-LOGIC | REF.CLIENTE Peru + consulta cuentas bancarias GOLPERU | Finanzas | 1, 7 |
| 4 | IMP-EXIST-SERVICE | Ampliar condicion Region Peru en flujo tramitacion + FeatureFlag OBS-032 | ProquifaDotNet | RE-FU-016 T15 |
| 5 | CREATE-PDF | Template HTML Proforma Peru GOLPERU_PER_PRO (3 archivos) | DocumentBuilder | RE-FU-016 T16 |
| 6 | BD-OBJ-CH | Registrar template Peru en BD + logo GOLPERU | DocumentBuilder | 5 |
| 7 | CREATE-SCRIPT-CONTROL | Scripts DML datos GOLPERU Peru (cuentas, direccion, contacto) | BD | - |

---

## Dependencias con RE-FU-016

Este requisito depende **integralmente** de que RE-FU-016 este completado:

| Componente RE-FU-016 requerido | Usado por Tarea 017 |
|-------------------------------|-------------------|
| ProformaModelBuilder (T8, T11) | Tarea 1 (agregar logica PER) |
| MontoALetrasConverter (T13) | Tarea 2 (agregar PEN) |
| ApiCallerFinanzas (T15) | Tarea 4 (ampliar condicion) |
| ProformaExtension + endpoint (T16) | Tarea 5 (nuevo template usa mismo servicio) |
| Flujo transaccional (T19) | Reutilizado sin cambios |
| FinanzasContext (T7) | Reutilizado sin cambios |
| Foliador SEQUENCE (T12) | Reutilizado sin cambios (global MEX+PER) |
