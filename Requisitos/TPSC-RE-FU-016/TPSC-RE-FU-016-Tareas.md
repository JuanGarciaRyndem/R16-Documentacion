# Tareas Back — TPSC-RE-FU-016
**Requisito:** Diseño y generación de Documentos: Proforma México

---

## T1 — [ TPSC-RE-FU-016 ] [ALG-COMPLX-LOGIC] Crear DTO y BO de armado de datos para Proforma México

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Facturas
- Logic.Pqf.Logistica

### Consideraciones previas
- El DTO debe contener todas las secciones del documento: header, cliente, partidas, pago, datos bancarios, facturación, entrega
- Se consultan 15+ tablas para armar el objeto completo
- Seguir el patrón existente de armado de datos para Cotización/Confirmación de Pedido
- La moneda aplicada es la de facturación del cliente (no la del pedido)

### Objetivo general
Crear la clase DTO de Proforma México y el BO que consulta todas las fuentes de datos y arma el objeto completo para enviar al DocumentBuilder.

### Objetivos específicos
- Crear clase `ProformaMexicoDTO` con secciones: Header, Cliente, Partidas[], Pago, DatosBancarios, Facturación, Entrega
- Crear BO que consulte: tpPedido, tpProformaPedido, tpProformaPartidaPedido, tpPartidaPedido, Producto, MarcaFamilia, Cliente, DatosFacturacionCliente, Empresa, EmpresaDatosBancarios, DatosBancarios, catBanco, catMoneda, catCondicionesDePago, DireccionCliente, ContactoCliente
- Implementar lógica de selección de TemplateKey por empresa: GOL->GOL_MEX_PRO, MUN->MUN_MEX_PRO, PQF->PQF_MEX_PRO, PRO->PRO_MEX_PRO
- Formatear montos con símbolo de moneda ($ X,XXX.XX M.N. / $ X,XXX.XX USD)
- Calcular SubTotal, IVA (0% o 16%), Gran Total

### Resultado esperado
Objeto DTO completo listo para serializar a JSON y enviar al DocumentBuilder.

### Entregables
- Clase ProformaMexicoDTO
- BO de armado de datos (consulta + mapeo)

### Criterios de aceptación
- El DTO contiene todos los campos visibles en las proformas de las 4 empresas
- Los datos se consultan correctamente de todas las tablas fuente
- El TemplateKey se selecciona según la empresa emisora
- Los montos están formateados correctamente según moneda

---

## T2 — [ TPSC-RE-FU-016 ] [ALG-COMPLX-LOGIC] Lógica del Código Validador (REF. CLIENTE) para datos bancarios

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Facturas
- Logic.Pqf.Logistica

### Consideraciones previas
- La referencia bancaria varía según el banco de la empresa:
  - **Banamex:** código de 7 segmentos codificados (ej. "SOL0153BXD96", "DIS4218BXP71")
  - **No-Banamex (BANORTE):** nombre del cliente directo (ej. "LABORATORIO RAAM DE...")
- Se requiere para las dos cuentas (M.N. y DLS) de cada empresa
- Golocaer usa BANORTE; Proquifa, Proveedora y Mungen usan BANAMEX

### Objetivo general
Implementar o verificar la lógica del Código Validador que construye la REF. CLIENTE por cuenta bancaria.

### Objetivos específicos
- Identificar lógica existente del Código Validador en el repositorio
- Implementar lógica Banamex: concatenación de 7 segmentos (nombre cliente, clave, código banco, moneda, CodValidador)
- Implementar lógica no-Banamex: nombre del cliente directo
- Integrar con el BO de armado del DTO (T1)

### Resultado esperado
La REF. CLIENTE se genera correctamente para cada cuenta bancaria según el banco.

### Entregables
- Lógica de Código Validador implementada/verificada
- Integración con armado del DTO

### Criterios de aceptación
- Banamex genera código de 7 segmentos (formato correcto)
- No-Banamex genera nombre del cliente directo
- Funciona para ambas cuentas (M.N. y DLS) de cada empresa

---

## T3 — [ TPSC-RE-FU-016 ] [ALG-BASIC-LOGIC] Conversión de monto a letras y folio con prefijo PRF

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido\Facturas
- Logic.Pqf.Logistica

### Consideraciones previas
- Formatos confirmados de las imágenes:
  - MXN: "(DOS MIL NOVECIENTOS SETENTA Y DOS PESOS 00/100 M.N.)"
  - USD: "(UN MIL SETECIENTOS VEINTISEIS DOLARES 08/100)"
- Folio visual: "PRF-MMDDAA-Consecutivo" (ej. PRF-031826-691)
- Pendiente confirmar si prefijo PRF se persiste en BD o solo es visual

### Objetivo general
Implementar la conversión de monto a letras según moneda y el formato visual del folio con prefijo PRF.

### Objetivos específicos
- Implementar/verificar conversión numérica a texto en español
- Aplicar formato según moneda: PESOS XX/100 M.N. vs DOLARES XX/100
- Aplicar formato de folio: "PRF-" + MMDDAA + "-" + Consecutivo
- Integrar con armado del DTO (T1)

### Resultado esperado
El Gran Total se muestra en letras con formato correcto y el folio tiene el prefijo PRF.

### Entregables
- Lógica de conversión a letras
- Lógica de formato de folio PRF

### Criterios de aceptación
- Formato MXN correcto con "PESOS XX/100 M.N."
- Formato USD correcto con "DOLARES XX/100"
- Folio con formato PRF-MMDDAA-Consecutivo

---

## T4 — [ TPSC-RE-FU-016 ] [IMP-EXIST-SERVICE] Endpoint de generación de PDF bajo demanda (previsualización)

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido
- WebApi.Logistica

### Consideraciones previas
- Se consume `ApiCallerDocumentBuilder.EnvioPost()` existente
- El PDF se genera dinámicamente y se retorna sin persistir
- Si ESAC abandona y reintenta, se regenera con datos vigentes
- Debe soportar cancelación (Criterio C2 del requisito)

### Objetivo general
Crear endpoint que genera el PDF de Proforma bajo demanda y lo retorna para previsualización.

### Objetivos específicos
- Crear endpoint en WebApi.Logistica que reciba IdTPProformaPedido
- Invocar BO de armado (T1) para obtener DTO completo
- Llamar `ApiCallerDocumentBuilder.EnvioPost("api/Report/proforma", dto)`
- Retornar byte[] (PDF) al Front para previsualización
- No persistir en esta etapa

### Resultado esperado
El ESAC puede previsualizar el PDF de la proforma antes de enviar.

### Entregables
- Endpoint de previsualización de PDF
- Consumo de ApiCallerDocumentBuilder

### Criterios de aceptación
- PDF se genera dinámicamente con datos vigentes
- Se retorna sin almacenar
- Al reintentar se regenera con datos actualizados
- Funciona para las 4 empresas

---

## T5 — [ TPSC-RE-FU-016 ] [SERV-SIMPLE-PUT] Persistencia del PDF tras envío exitoso y consulta histórica

### Aplicativos
- ProquifaDotNet

### Módulos
- L05.TramitarPedido
- WebApi.Logistica

### Consideraciones previas
- Al confirmar envío exitoso del correo, se almacena el PDF como artefacto inmutable
- El PDF histórico se consulta desde Validar Cobro (sin regenerar)
- Definir estructura de almacenamiento (tabla Archivo o binario en tpProformaPedido)

### Objetivo general
Implementar la persistencia del PDF final tras envío exitoso y el endpoint de consulta histórica.

### Objetivos específicos
- Al confirmar envío exitoso: almacenar PDF (byte[]) en BD vinculado a la proforma
- Crear endpoint de consulta que retorne el PDF almacenado por IdTPProformaPedido
- El PDF almacenado es inmutable (no se regenera)
- Accesible desde módulo Validar Cobro

### Resultado esperado
El PDF final queda almacenado y es consultable históricamente sin regeneración.

### Entregables
- Lógica de persistencia del PDF
- Endpoint de consulta histórica

### Criterios de aceptación
- PDF se almacena tras envío exitoso
- Consulta retorna PDF original (no regenerado)
- Accesible desde Validar Cobro
- Es inmutable (no se modifica tras almacenamiento)

---

## T6 — [ TPSC-RE-FU-016 ] [IMP-EXIST-SERVICE] Crear servicio de generación de Proforma en DocumentBuilder

### Aplicativos
- DocumentBuilder

### Módulos
- Application\Services
- Application\DTOs
- Application\Validators
- API\Controllers

### Consideraciones previas
- Seguir patrón de `GenerateDocumentService.QuotationExtension.cs`
- Más simple que Cotización: sin lógica de merge ni Usage Letter
- Flujo: obtener template -> renderizar Header/Body/Footer con JSON -> convertir a PDF
- Crear DTO tipado, validator y endpoint dedicado

### Objetivo general
Crear el servicio completo de generación de Proforma en el repositorio DocumentBuilder, siguiendo el patrón de Cotización.

### Objetivos específicos
- Crear `GenerateDocumentService.ProformaExtension.cs` con método `GenerateProformaTemplate()`
- Crear DTO `DocumentGenerateProformaDto` (FileName, TemplateKey, ClientName, Base64Images, ProformaModel)
- Crear `ProformaModel` con secciones: Header, Cliente, Partidas[], Pago, DatosBancarios, Facturación, Entrega
- Crear `DocumentGenerateProformaDtoFluentValidator.cs`
- Crear endpoint `POST api/Report/proforma` en `ReportController.cs`
- Registrar servicio en DI

### Resultado esperado
DocumentBuilder puede recibir un DTO de Proforma y generar el PDF correspondiente.

### Entregables
- GenerateDocumentService.ProformaExtension.cs
- DocumentGenerateProformaDto.cs + ProformaModel.cs
- DocumentGenerateProformaDtoFluentValidator.cs
- Endpoint en ReportController.cs

### Criterios de aceptación
- Endpoint recibe DTO y retorna byte[] (PDF)
- Renderiza correctamente Header, Body y Footer
- Validator rechaza DTOs incompletos
- Funciona con los 4 TemplateKeys de proforma

---

## T7 — [ TPSC-RE-FU-016 ] [CREATE-PDF] Crear templates HTML de Proforma México para 4 empresas

### Aplicativos
- DocumentBuilder

### Módulos
- API\Resources\Templates

### Consideraciones previas
- 4 carpetas nuevas: GOL_MEX_PRO, MUN_MEX_PRO, PQF_MEX_PRO, PRO_MEX_PRO
- Cada carpeta con 3 archivos: _H.html (header), _B.html (body), _F.html (footer)
- 3 variantes visuales: Golocaer (naranja), Mungen (verde), Proquifa+Proveedora (teal)
- Datos dinámicos vía JSON (motor de templates del DocumentBuilder)
- Registrar en tabla DocumentTemplate de BD

### Objetivo general
Crear los 4 templates HTML de Proforma México con el diseño y branding correspondiente a cada empresa.

### Objetivos específicos
- Diseñar HTML/CSS para la estructura del documento:
  - Cabecera: logo + disclaimer SAT + título "Proforma" + folio + vigencia
  - Sección Cliente: nombre/alias
  - Tabla de partidas: #, Cant, Descripción, Precio U, Importe
  - Panel inferior 4 columnas: Pago | Datos Bancarios | Facturación | Entrega
  - Pie: redes sociales, teléfonos, web, razón social legal, certificaciones, logos catálogos, paginación
- Implementar 3 variantes de color/logo (naranja/verde/teal)
- Preparar logos e imágenes en base64 (logos empresa, ISO, NEEC, AmEx, EDQM, FEUM, USP, Microbiologics, APACOR, CHATA, Pharmaffiliates)
- INSERT en tabla DocumentTemplate para los 4 TemplateKeys
- Validar paginación automática con múltiples partidas

### Resultado esperado
Los 4 templates generan PDFs idénticos a las proformas de referencia (imágenes proporcionadas).

### Entregables
- 4 carpetas con 3 archivos HTML cada una (12 archivos total)
- Script INSERT para tabla DocumentTemplate
- Logos e imágenes en base64

### Criterios de aceptación
- PDF generado coincide visualmente con las proformas de referencia
- Branding correcto por empresa (logo, color, razón social, dirección)
- Paginación funcional (X/Y) con múltiples partidas
- Datos dinámicos se renderizan correctamente desde JSON
- Disclaimer SAT visible en todas las variantes

---

## Resumen de tareas

| # | Clave Catálogo | Título | Repositorio | Predecesora |
|---|----------------|--------|-------------|-------------|
| T1 | ALG-COMPLX-LOGIC | Crear DTO y BO de armado de datos para Proforma México | ProquifaDotNet | — |
| T2 | ALG-COMPLX-LOGIC | Lógica del Código Validador (REF. CLIENTE) | ProquifaDotNet | — |
| T3 | ALG-BASIC-LOGIC | Conversión de monto a letras y folio con prefijo PRF | ProquifaDotNet | — |
| T4 | IMP-EXIST-SERVICE | Endpoint de generación de PDF bajo demanda (previsualización) | ProquifaDotNet | T1, T2, T3, T6 |
| T5 | SERV-SIMPLE-PUT | Persistencia del PDF tras envío exitoso y consulta histórica | ProquifaDotNet | T4 |
| T6 | IMP-EXIST-SERVICE | Crear servicio de generación de Proforma en DocumentBuilder | DocumentBuilder | — |
| T7 | CREATE-PDF | Crear templates HTML de Proforma México para 4 empresas | DocumentBuilder | T6 |

---

## Dependencias con otros requisitos (NO incluidas como tareas)

| Requisito | Tarea relacionada | Relación |
|-----------|-------------------|----------|
| TPSC-RE-FU-006 | T2 | ReferenciaPago / Código Validador |
| TPSC-RE-FU-013 | T1 | Foliador global (asigna folio a tpProformaPedido.Folio) |
| TPSC-RE-FU-014 | T4 | Flujo Prepago sin FAA dispara generación del PDF |
| TPSC-RE-FU-017 | T6, T7 | Proforma Perú (mismo patrón, templates diferentes) |
