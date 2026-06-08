# Tareas BackEnd - TPSC-RE-FU-021
**Requisito:** Diseño y generación de Documentos: Factura México
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + DocumentBuilder

---

> **Orden de ejecución sugerido:** BD bloqueantes → BD configuración → templates DocumentBuilder → servicios backend

---

## Resumen de tareas

| # | Clave | Título simple | Tipo | Aplicativo | Bloqueante |
|---|-------|--------------|------|-----------|----------|
| 1 | CREATE-TABL-CH | Crear catClaveProdServSAT + DML catálogo c_ClaveProdServ SAT | BD | ProquifaDotNet | ✅ |
| 2 | UPDATE-TABL-M | Agregar ClaveProdServSAT e IdCatClaveProdServSAT (FK) a Producto | BD | ProquifaDotNet | ✅ |
| 3 | UPDATE-TABL-CH | Agregar ClaveSAT a catUnidad + UPDATE mapeo c_ClaveUnidad SAT | BD | ProquifaDotNet | — |
| 4 | UPDATE-TABL-CH | Agregar campo Exportacion a CFDIGenerada | BD | ProquifaDotNet | — |
| 5 | CREATE-PDF | Plantilla PDF Factura CFDI 4.0 — GOL_MEX_FAC (Golocaer S.A. de C.V.) | DocumentBuilder | DocumentBuilder | — |
| 6 | CREATE-PDF | Plantilla PDF Factura CFDI 4.0 — MUN_MEX_FAC (Mungen S.A. de C.V.) | DocumentBuilder | DocumentBuilder | — |
| 7 | CREATE-PDF | Plantilla PDF Factura CFDI 4.0 — PRO_MEX_FAC (Proquifa S.A. de C.V.) | DocumentBuilder | DocumentBuilder | — |
| 8 | CREATE-PDF | Plantilla PDF Factura CFDI 4.0 — PQF_MEX_FAC (Proveedora Quimico Farmaceutica S.A. de C.V.) | DocumentBuilder | DocumentBuilder | — |
| 9 | ALG-COMPLX-LOGIC | Algoritmo de mapeo de datos CFDI 4.0 al modelo del PDF de Factura México | Back | ProquifaDotNet.Finanzas | — |
| 10 | SERV-TRANSACT | Persistencia del PDF de Factura México en Minio tras timbrado exitoso | Back | ProquifaDotNet.Finanzas | — |
| 11 | IMP-EXIST-SERVICE | Extender FacturaAdelantadoGenerarService con persistencia real del PDF (pasos 10-11 RE-FU-019 T13) | Back | ProquifaDotNet.Finanzas | — |
| 12 | IMP-EXIST-SERVICE | Extender FacturaAdelantadoPreviewService con template real de Factura México (RE-FU-019 T15) | Back | ProquifaDotNet.Finanzas | — |
| 13 | IMP-EXIST-SERVICE | Integrar persistencia del PDF de Factura México en el flujo Validar Cobro (Genera Factura Normal PPD) | Back | ProquifaDotNet.Finanzas | — |

---

## TAREA 1

**[ RE-FU-021 ] [CREATE-TABL-CH] Crear tabla catClaveProdServSAT e insertar catálogo c_ClaveProdServ SAT**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogos SAT

**Consideraciones previas:**
- La tabla `catClaveProdServSAT` no existe en BD. Es necesaria para mapear la clave del catálogo `c_ClaveProdServ` del SAT por producto, campo obligatorio en los conceptos del CFDI 4.0.
- Es **BLOQUEANTE** para la Tarea 2 que agrega el FK desde `Producto`.
- Es **BLOQUEANTE** para el desarrollo del PDF de la Factura CFDI 4.0: sin la clave SAT del producto no se puede armar el concepto válido en el PDF.
- Los registros iniciales del catálogo deben incluir las claves relevantes para los productos de PROQUIFA (productos químico-farmacéuticos).

**Objetivo general:**
Crear la tabla `catClaveProdServSAT` en ProquifaDotNet y poblarla con los valores del catálogo `c_ClaveProdServ` del SAT relevantes para la operación de PROQUIFA, habilitando la asociación de cada producto a su clave fiscal SAT.

**Objetivos específicos:**
- Ejecutar el DDL de creación con columnas: `IdCatClaveProdServSAT` (PK, NEWID), `Clave` varchar(10) UQ, `Descripcion` varchar(300), `Activo` bit DEFAULT 1.
- Insertar los registros del catálogo `c_ClaveProdServ` SAT relevantes para los productos de PROQUIFA.
- Validar que PK, UQ y DEFAULT queden correctamente definidos.

**Resultado esperado:**
Tabla `catClaveProdServSAT` existente en ProquifaDotNet con las claves SAT relevantes insertadas y lista para ser referenciada por `Producto`.

**Entregables:**
- Script DDL: `CREATE TABLE catClaveProdServSAT`
- Script DML: `INSERT` catálogo `c_ClaveProdServ` SAT (claves relevantes PROQUIFA)
- Script de validación (`SELECT` con conteo de filas)

**Criterios de aceptación:**
- La tabla existe con la estructura definida en `TPSC-RE-FU-021_BD.md`.
- Los registros del catálogo `c_ClaveProdServ` SAT están insertados y son consultables.
- Ningún objeto existente en BD presenta errores tras la creación.

**Más información de la tarea:**
Ver sección *"Cambios DDL Requeridos — catClaveProdServSAT"* en `TPSC-RE-FU-021_BD.md`.

**Recursos:**
- `TPSC-RE-FU-021_BD.md` — Script propuesto `CREATE TABLE catClaveProdServSAT`
- Catálogo `c_ClaveProdServ` SAT — Anexo 20 del estándar CFDI 4.0

---

## TAREA 2

**[ RE-FU-021 ] [UPDATE-TABL-M] Agregar ClaveProdServSAT e IdCatClaveProdServSAT (FK) a tabla Producto**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogo de Productos

**Consideraciones previas:**
- **BLOQUEANTE:** sin `ClaveProdServSAT` e `IdCatClaveProdServSAT` en `Producto` no es posible construir los conceptos del CFDI 4.0 ni renderizarlos en el PDF, ya que la clave SAT del producto es campo obligatorio por partida.
- Depende de la Tarea 1 (`catClaveProdServSAT` debe existir antes de agregar el FK).
- Las columnas se agregan como NULL para no romper registros existentes.
- Verificar que los ALTER no rompan vistas, stored procedures ni triggers dependientes de `Producto`.

**Objetivo general:**
Ampliar la tabla `Producto` con los campos del catálogo `c_ClaveProdServ` SAT para poder incluir en cada concepto del CFDI 4.0 y en el PDF de la Factura la clave SAT correspondiente al producto.

**Objetivos específicos:**
- `ALTER TABLE dbo.Producto ADD ClaveProdServSAT varchar(10) NULL` — clave directa del catálogo c_ClaveProdServ SAT.
- `ALTER TABLE dbo.Producto ADD IdCatClaveProdServSAT uniqueidentifier NULL CONSTRAINT FK_Producto_ClaveProdServSAT FOREIGN KEY REFERENCES catClaveProdServSAT(IdCatClaveProdServSAT)`.
- Verificar que los ALTER no rompan vistas, stored procedures ni triggers dependientes.

**Resultado esperado:**
`Producto` con los nuevos campos SAT disponibles. Los campos quedan en NULL hasta que el equipo de operaciones los capture por producto.

**Entregables:**
- Script DDL: `ALTER TABLE Producto` (x2 — campo varchar y FK)
- Script de validación (`SELECT` con los nuevos campos)
- Checklist de objetos dependientes verificados

**Criterios de aceptación:**
- `Producto.ClaveProdServSAT` existe y acepta valores varchar(10).
- `Producto.IdCatClaveProdServSAT` existe con FK activo hacia `catClaveProdServSAT`.
- Ningún SP, vista ni trigger presenta errores de compilación tras los ALTER.

**Más información de la tarea:**
Ver sección *"Cambios DDL Requeridos — ClaveProdServSAT en Producto"* en `TPSC-RE-FU-021_BD.md`.

**Recursos:**
- `TPSC-RE-FU-021_BD.md` — Scripts propuestos ALTER TABLE Producto

---

## TAREA 3

**[ RE-FU-021 ] [UPDATE-TABL-CH] Agregar ClaveSAT a catUnidad y actualizar mapeo claves c_ClaveUnidad SAT**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogo de Unidades

**Consideraciones previas:**
- `catUnidad` ya tiene el campo `Clave` (varchar 150) pero **no es la clave SAT** del catálogo `c_ClaveUnidad`; son campos diferentes.
- Se agrega `ClaveSAT` como columna independiente para no alterar el comportamiento actual de `Clave`.
- Tras el ALTER se debe poblar `ClaveSAT` con el mapeo correspondiente para las unidades existentes (ej.: KGM — Kilogramo, H87 — Pieza, LTR — Litro, E48 — Unidad de servicio).
- Esta columna es requerida en las partidas del PDF de la Factura CFDI 4.0 (criterio E1 de RE-FU-021).

**Objetivo general:**
Agregar el campo `ClaveSAT` a la tabla `catUnidad` y actualizar los registros existentes con su clave correspondiente del catálogo `c_ClaveUnidad` SAT, habilitando la inclusión de la clave de unidad SAT en los conceptos del CFDI 4.0 y en el PDF de la Factura.

**Objetivos específicos:**
- `ALTER TABLE dbo.catUnidad ADD ClaveSAT varchar(10) NULL`.
- `UPDATE catUnidad SET ClaveSAT = '...'` para cada unidad existente con su clave SAT equivalente.
- Validar que el UPDATE cubre todas las unidades activas en uso.

**Resultado esperado:**
`catUnidad` con el campo `ClaveSAT` disponible y con los valores SAT mapeados para todas las unidades activas.

**Entregables:**
- Script DDL: `ALTER TABLE catUnidad ADD ClaveSAT`
- Script DML: `UPDATE catUnidad SET ClaveSAT` (mapeo completo)
- Script de validación (`SELECT` — unidades activas sin `ClaveSAT` NULL)

**Criterios de aceptación:**
- `catUnidad.ClaveSAT` existe y acepta valores del catálogo `c_ClaveUnidad` SAT.
- Todas las unidades activas tienen `ClaveSAT` poblado (sin NULL en registros activos).
- Ningún SP, vista ni trigger presenta errores de compilación tras el ALTER.

**Más información de la tarea:**
Ver sección *"Cambios DDL Requeridos — ClaveSAT en catUnidad"* en `TPSC-RE-FU-021_BD.md`.

**Recursos:**
- `TPSC-RE-FU-021_BD.md` — Script propuesto ALTER + ejemplos de claves SAT
- Catálogo `c_ClaveUnidad` SAT — Anexo 20 del estándar CFDI 4.0

---

## TAREA 4

**[ RE-FU-021 ] [UPDATE-TABL-CH] Agregar campo Exportacion a tabla CFDIGenerada**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — CFDI

**Consideraciones previas:**
- El campo `Exportacion` es **obligatorio en CFDI 4.0** (atributo del nodo `Comprobante`). Su ausencia genera rechazo por el PAC/SAT.
- El valor para operaciones nacionales es `'01'` (No aplica), que es el caso de todas las facturas del grupo PROQUIFA México.
- No requiere cambio en la estructura de entidades existentes más allá del ALTER.
- Este campo alimenta la sección C del PDF de la Factura (criterio C4 de RE-FU-021).

**Objetivo general:**
Agregar el campo `Exportacion` varchar(2) a la tabla `CFDIGenerada` para cumplir con el atributo obligatorio del estándar CFDI 4.0 y permitir su renderizado en el PDF de la Factura.

**Objetivos específicos:**
- `ALTER TABLE dbo.CFDIGenerada ADD Exportacion varchar(2) NULL`.
- `UPDATE CFDIGenerada SET Exportacion = '01'` para todos los registros existentes.
- Validar que ningún objeto dependiente de `CFDIGenerada` se rompe tras el ALTER.

**Resultado esperado:**
`CFDIGenerada` con el campo `Exportacion` disponible y con el valor `'01'` para todos los registros existentes, cumpliendo el estándar CFDI 4.0.

**Entregables:**
- Script DDL: `ALTER TABLE CFDIGenerada ADD Exportacion`
- Script DML: `UPDATE CFDIGenerada SET Exportacion = '01'`
- Script de validación (`SELECT`)

**Criterios de aceptación:**
- `CFDIGenerada.Exportacion` existe y acepta valores varchar(2).
- Todos los registros existentes tienen `Exportacion = '01'`.
- Ningún SP, vista ni trigger presenta errores de compilación tras el ALTER.

**Más información de la tarea:**
Ver sección *"Cambios DDL Requeridos — Exportacion en CFDIGenerada"* en `TPSC-RE-FU-021_BD.md`.

**Recursos:**
- `TPSC-RE-FU-021_BD.md` — Script propuesto ALTER TABLE CFDIGenerada
- Catálogo `c_Exportacion` SAT — estándar CFDI 4.0

---

## TAREA 5

**[ RE-FU-021 ] [CREATE-PDF] Plantilla PDF Factura CFDI 4.0 — GOL_MEX_FAC (Golocaer S.A. de C.V.)**

**Aplicativos:** DocumentBuilder

**Módulos:** DocumentBuilder — Plantillas Factura México

**Consideraciones previas:**
- DocumentBuilder usa el patrón `{Prefix}_{Region}_{Tipo}` para la `TemplateKey` (ej.: `GOL_MEX_FAC`).
- Cada template requiere 3 archivos HTML: Header (`GOL_MEX_FAC_H.html`), Body (`GOL_MEX_FAC_B.html`) y Footer (`GOL_MEX_FAC_F.html`).
- El registro en `DocumentTemplate` se inserta via script SQL (patrón existente en `Scripts/`).
- El branding de Golocaer S.A. de C.V. (logo, colores, certificaciones) debe validarse contra el CFDI real de referencia (folio 7156).
- Depende de las Tareas 1–4 (campos SAT en BD disponibles para armar las partidas del body).

**Objetivo general:**
Crear la plantilla HTML (Header, Body, Footer) y el registro en `DocumentTemplate` para la Factura CFDI 4.0 de la empresa **Golocaer S.A. de C.V.** (`GOL_MEX_FAC`), con el branding correspondiente (logo, colores, certificaciones) y todas las secciones requeridas por el estándar CFDI 4.0.

**Objetivos específicos:**
- Crear `GOL_MEX_FAC_H.html` — cabecera con logo Golocaer, datos del emisor (RFC, razón social, dirección, lugar expedición) y datos del receptor (RFC, razón social, CP, régimen fiscal, uso CFDI).
- Crear `GOL_MEX_FAC_B.html` — cuerpo con: datos del CFDI (serie, folio, versión 4.0, UUID, fechas, método/forma pago, tipo comprobante, moneda, TC, exportación), tabla de partidas (clave SAT producto, descripción, pedimento, Nº ID, cantidad, unidad, clave unidad SAT, valor unitario, importe, IVA por línea), totales (subtotal, IVA, total, total en letras), datos bancarios y referencias del cliente.
- Crear `GOL_MEX_FAC_F.html` — pie con elementos técnicos SAT (QR, Nº serie certificados, sellos digitales, cadena original), disclaimer "Representación impresa de un CFDI 4.0", certificaciones (ISO 9001:2015, NEEC), logos institucionales (EDQM, FEUM, USP, etc.) y paginación "X de Y".
- Insertar el registro en `DocumentTemplate` via script SQL (`Scripts/`).

**Resultado esperado:**
Template `GOL_MEX_FAC` registrado en DocumentBuilder con los 3 archivos HTML correctos, listo para generar el PDF de la Factura CFDI 4.0 de Golocaer S.A. de C.V.

**Entregables:**
- `GOL_MEX_FAC_H.html`, `GOL_MEX_FAC_B.html`, `GOL_MEX_FAC_F.html`
- Script SQL: `INSERT DocumentTemplate GOL_MEX_FAC`
- Prueba de generación con el CFDI real de referencia (Golocaer folio 7156)

**Criterios de aceptación:**
- El PDF generado con el template `GOL_MEX_FAC` contiene todas las secciones A–J del requisito.
- El branding (logo, colores) corresponde a Golocaer S.A. de C.V., validado contra el CFDI real folio 7156.
- La paginación automática funciona correctamente con el contador "X de Y".
- El disclaimer "Representación impresa de un CFDI 4.0" aparece en el pie.

**Más información de la tarea:**
Ver criterios A1–J2 en `TPSC-RE-FU-021.md`. Patrón de templates en `Scripts/20251001_1439/01_Insert_templates.sql`.

**Recursos:**
- `TPSC-RE-FU-021.md` — Criterios A1–J2
- DocumentBuilder — `C:\Users\juan.garcia\Documents\DocumentBuilder-R14`
- CFDI real de referencia: Golocaer folio 7156

---

## TAREA 6

**[ RE-FU-021 ] [CREATE-PDF] Plantilla PDF Factura CFDI 4.0 — MUN_MEX_FAC (Mungen S.A. de C.V.)**

**Aplicativos:** DocumentBuilder

**Módulos:** DocumentBuilder — Plantillas Factura México

**Consideraciones previas:**
- Misma estructura que Tarea 5, diferenciada en branding: logo, colores y certificaciones de Mungen S.A. de C.V.
- Validar contra el CFDI real de referencia de Mungen (folio 2374).
- `TemplateKey`: `MUN_MEX_FAC`. Archivos: `MUN_MEX_FAC_H.html`, `MUN_MEX_FAC_B.html`, `MUN_MEX_FAC_F.html`.

**Objetivo general:**
Crear la plantilla HTML (Header, Body, Footer) y el registro en `DocumentTemplate` para la Factura CFDI 4.0 de la empresa **Mungen S.A. de C.V.** (`MUN_MEX_FAC`), con el branding correspondiente y todas las secciones requeridas por el estándar CFDI 4.0.

**Objetivos específicos:**
- Crear `MUN_MEX_FAC_H.html` — cabecera con logo Mungen, datos del emisor y receptor.
- Crear `MUN_MEX_FAC_B.html` — cuerpo con todas las secciones del CFDI 4.0, partidas con claves SAT, totales y datos bancarios.
- Crear `MUN_MEX_FAC_F.html` — pie con elementos técnicos SAT, disclaimer, certificaciones, logos institucionales y paginación.
- Insertar el registro en `DocumentTemplate` via script SQL.

**Resultado esperado:**
Template `MUN_MEX_FAC` registrado en DocumentBuilder con los 3 archivos HTML correctos, listo para generar el PDF de la Factura CFDI 4.0 de Mungen S.A. de C.V.

**Entregables:**
- `MUN_MEX_FAC_H.html`, `MUN_MEX_FAC_B.html`, `MUN_MEX_FAC_F.html`
- Script SQL: `INSERT DocumentTemplate MUN_MEX_FAC`
- Prueba de generación con el CFDI real de referencia (Mungen folio 2374)

**Criterios de aceptación:**
- El PDF generado con el template `MUN_MEX_FAC` contiene todas las secciones A–J del requisito.
- El branding corresponde a Mungen S.A. de C.V., validado contra el CFDI real folio 2374.
- La paginación automática y el disclaimer funcionan correctamente.

**Más información de la tarea:**
Ver criterios A1–J2 en `TPSC-RE-FU-021.md`. Patrón de templates en `Scripts/20251001_1439/01_Insert_templates.sql`.

**Recursos:**
- `TPSC-RE-FU-021.md` — Criterios A1–J2
- DocumentBuilder — `C:\Users\juan.garcia\Documents\DocumentBuilder-R14`
- CFDI real de referencia: Mungen folio 2374

---

## TAREA 7

**[ RE-FU-021 ] [CREATE-PDF] Plantilla PDF Factura CFDI 4.0 — PRO_MEX_FAC (Proquifa S.A. de C.V.)**

**Aplicativos:** DocumentBuilder

**Módulos:** DocumentBuilder — Plantillas Factura México

**Consideraciones previas:**
- Misma estructura que Tarea 5, diferenciada en branding: logo, colores y certificaciones de Proquifa S.A. de C.V.
- Validar contra el CFDI real de referencia de Proquifa (folio 20913).
- `TemplateKey`: `PRO_MEX_FAC`. Archivos: `PRO_MEX_FAC_H.html`, `PRO_MEX_FAC_B.html`, `PRO_MEX_FAC_F.html`.

**Objetivo general:**
Crear la plantilla HTML (Header, Body, Footer) y el registro en `DocumentTemplate` para la Factura CFDI 4.0 de la empresa **Proquifa S.A. de C.V.** (`PRO_MEX_FAC`), con el branding correspondiente y todas las secciones requeridas por el estándar CFDI 4.0.

**Objetivos específicos:**
- Crear `PRO_MEX_FAC_H.html` — cabecera con logo Proquifa, datos del emisor y receptor.
- Crear `PRO_MEX_FAC_B.html` — cuerpo con todas las secciones del CFDI 4.0, partidas con claves SAT, totales y datos bancarios.
- Crear `PRO_MEX_FAC_F.html` — pie con elementos técnicos SAT, disclaimer, certificaciones, logos institucionales y paginación.
- Insertar el registro en `DocumentTemplate` via script SQL.

**Resultado esperado:**
Template `PRO_MEX_FAC` registrado en DocumentBuilder con los 3 archivos HTML correctos, listo para generar el PDF de la Factura CFDI 4.0 de Proquifa S.A. de C.V.

**Entregables:**
- `PRO_MEX_FAC_H.html`, `PRO_MEX_FAC_B.html`, `PRO_MEX_FAC_F.html`
- Script SQL: `INSERT DocumentTemplate PRO_MEX_FAC`
- Prueba de generación con el CFDI real de referencia (Proquifa folio 20913)

**Criterios de aceptación:**
- El PDF generado con el template `PRO_MEX_FAC` contiene todas las secciones A–J del requisito.
- El branding corresponde a Proquifa S.A. de C.V., validado contra el CFDI real folio 20913.
- La paginación automática y el disclaimer funcionan correctamente.

**Más información de la tarea:**
Ver criterios A1–J2 en `TPSC-RE-FU-021.md`. Patrón de templates en `Scripts/20251001_1439/01_Insert_templates.sql`.

**Recursos:**
- `TPSC-RE-FU-021.md` — Criterios A1–J2
- DocumentBuilder — `C:\Users\juan.garcia\Documents\DocumentBuilder-R14`
- CFDI real de referencia: Proquifa folio 20913

---

## TAREA 8

**[ RE-FU-021 ] [CREATE-PDF] Plantilla PDF Factura CFDI 4.0 — PQF_MEX_FAC (Proveedora Quimico Farmaceutica S.A. de C.V.)**

**Aplicativos:** DocumentBuilder

**Módulos:** DocumentBuilder — Plantillas Factura México

**Consideraciones previas:**
- Misma estructura que Tarea 5, diferenciada en branding: logo, colores y certificaciones de Proveedora Quimico Farmaceutica S.A. de C.V.
- Validar contra el CFDI real de referencia de Proveedora (folio 143103).
- `TemplateKey`: `PQF_MEX_FAC`. Archivos: `PQF_MEX_FAC_H.html`, `PQF_MEX_FAC_B.html`, `PQF_MEX_FAC_F.html`.

**Objetivo general:**
Crear la plantilla HTML (Header, Body, Footer) y el registro en `DocumentTemplate` para la Factura CFDI 4.0 de la empresa **Proveedora Quimico Farmaceutica S.A. de C.V.** (`PQF_MEX_FAC`), con el branding correspondiente y todas las secciones requeridas por el estándar CFDI 4.0.

**Objetivos específicos:**
- Crear `PQF_MEX_FAC_H.html` — cabecera con logo Proveedora Quimico Farmaceutica, datos del emisor y receptor.
- Crear `PQF_MEX_FAC_B.html` — cuerpo con todas las secciones del CFDI 4.0, partidas con claves SAT, totales y datos bancarios.
- Crear `PQF_MEX_FAC_F.html` — pie con elementos técnicos SAT, disclaimer, certificaciones, logos institucionales y paginación.
- Insertar el registro en `DocumentTemplate` via script SQL.

**Resultado esperado:**
Template `PQF_MEX_FAC` registrado en DocumentBuilder con los 3 archivos HTML correctos, listo para generar el PDF de la Factura CFDI 4.0 de Proveedora Quimico Farmaceutica S.A. de C.V.

**Entregables:**
- `PQF_MEX_FAC_H.html`, `PQF_MEX_FAC_B.html`, `PQF_MEX_FAC_F.html`
- Script SQL: `INSERT DocumentTemplate PQF_MEX_FAC`
- Prueba de generación con el CFDI real de referencia (Proveedora folio 143103)

**Criterios de aceptación:**
- El PDF generado con el template `PQF_MEX_FAC` contiene todas las secciones A–J del requisito.
- El branding corresponde a Proveedora Quimico Farmaceutica S.A. de C.V., validado contra el CFDI real folio 143103.
- La paginación automática y el disclaimer funcionan correctamente.

**Más información de la tarea:**
Ver criterios A1–J2 en `TPSC-RE-FU-021.md`. Patrón de templates en `Scripts/20251001_1439/01_Insert_templates.sql`.

**Recursos:**
- `TPSC-RE-FU-021.md` — Criterios A1–J2
- DocumentBuilder — `C:\Users\juan.garcia\Documents\DocumentBuilder-R14`
- CFDI real de referencia: Proveedora folio 143103

---

## TAREA 9

**[ RE-FU-021 ] [ALG-COMPLX-LOGIC] Algoritmo de mapeo de datos CFDI 4.0 al modelo del PDF de Factura México**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura — Generación PDF CFDI 4.0

**Consideraciones previas:**
- El PDF de la Factura CFDI 4.0 integra datos de múltiples tablas: `CFDI`, `CFDIGenerada`, `Producto`, `catUnidad`, `EmpresaDatosBancarios`, `tpPartidaPedido` y el `TimbreFiscalDigital` del XML retornado por el PAC (TurboPac).
- Esta tarea es la capa de mapeo/orquestación previa a la generación visual del PDF (Tareas 5–8); no genera el PDF.
- Depende de las Tareas 1–4 (campos SAT en BD disponibles).
- Los datos del `TimbreFiscalDigital` (UUID, sellos, cadena original, números de serie) se leen del XML del PAC, no de BD directamente.
- El código QR se genera dinámicamente en esta capa a partir del UUID + RFC emisor + RFC receptor + total.
- Aplica a Facturas de FAA (RE-FU-019) y de Validar Cobro. Solo Región México (GOL, MUN, PRO, PQF).
- La comunicación con ProquifaDotNet (.NET Framework 4.8) para obtener los datos del pedido y del CFDI se realiza mediante llamadas entre APIs.
- **Relación con RE-FU-019:** La Tarea 15 de RE-FU-019 (`FacturaAdelantadoPreviewService`) usa este mismo mapeo con un subconjunto de campos (sin UUID, sin sello, sin cadena original) para el PDF de previsualización. Este servicio provee el modelo completo para ambos contextos (preview y definitivo); no duplica RE-FU-019, lo complementa con los campos SAT pendientes.

**Objetivo general:**
Implementar el servicio que consolida todos los datos necesarios del CFDI 4.0 en un modelo unificado (`FacturaPdfModel` o equivalente), listo para ser consumido por el generador de PDF de DocumentBuilder, incluyendo branding por empresa emisora, datos fiscales del emisor y receptor, partidas con claves SAT, totales e impuestos, elementos técnicos SAT y QR.

**Objetivos específicos:**
- Mapear sección Emisor: RFC, Razón Social, Régimen Fiscal 601, Lugar de Expedición, dirección (desde `CFDIGenerada` + `Empresa`).
- Mapear sección Receptor: RFC, Razón Social, Uso CFDI, CP, Régimen Fiscal (desde `CFDIGenerada`).
- Mapear sección CFDI: Serie, Folio, Versión 4.0, UUID, fechas emisión/certificación, Método de Pago PPD, Forma de Pago 99, Condiciones de Pago, Tipo Comprobante I, Moneda, Tipo de Cambio, Exportación (desde `CFDI` + `CFDIGenerada`); incluir Folio del Pedido Interno (PI) del sistema PQF2 (criterio G3).
- Mapear sección Partidas por concepto: `ClaveProdServSAT`, descripción (catálogo + marca + lote + caducidad), pedimento si aplica, Nº ID interno, cantidad, unidad de medida, `ClaveSAT` de unidad, valor unitario, importe, desglose de impuestos SAT por línea: Base, Impuesto (clave SAT), TipoFactor, TasaCuota e Importe (desde `tpPartidaPedido` + `Producto` + `catUnidad`).
- Mapear sección Totales: Subtotal, IVA trasladado, Total, Total en letras (desde `CFDIGenerada` + cálculo de partidas).
- Mapear sección Datos Bancarios: lista de dos cuentas del grupo PROQUIFA México (una en MXN y una en USD), cada una con Banco, Número de Cuenta, CLABE, Moneda, Sucursal y Referencia del Cliente; modeladas como `List<FacturaPdfCuentaBancariaModel>` (desde `EmpresaDatosBancarios`).
- Mapear sección Técnica SAT: UUID, sellos digitales emisor/SAT, cadena original, Nº serie certificados, QR generado dinámicamente (desde `TimbreFiscalDigital` del XML del PAC).
- Resolver branding (logo, colores, certificaciones) según `EmpresaClave` del pedido (GOL/MUN/PRO/PQF).

**Resultado esperado:**
Servicio `FacturaMexicoPdfMappingService` (o equivalente) que recibe el `IdCFDI` y retorna un modelo `FacturaPdfModel` completamente poblado, listo para ser pasado al generador de DocumentBuilder (Tareas 5–8) tanto para previsualización (sin datos de timbrado) como para el PDF definitivo (con `TimbreFiscalDigital` completo).

**Entregables:**
- Clase `FacturaPdfModel` con todas las secciones del PDF
- Servicio `FacturaMexicoPdfMappingService`
- Prueba unitaria con datos de los 4 CFDIs reales de referencia (Mungen 2374, Golocaer 7156, Proquifa 20913, Proveedora 143103)

**Criterios de aceptación:**
- El modelo generado contiene todos los campos requeridos por las secciones A–J del PDF (criterios del requisito).
- El campo `Exportacion` se mapea desde `CFDIGenerada.Exportacion`.
- Cada partida incluye `ClaveProdServSAT` y `ClaveSAT` de unidad.
- El QR se genera correctamente con el formato SAT (UUID + RFC emisor + RFC receptor + total).
- El branding (logo, colores) varía correctamente según la empresa emisora (GOL/MUN/PRO/PQF).
- El modelo incluye la sección de retenciones (`List<FacturaPdfRetencionModel>`), renderizada si hay retenciones o vacía si no las hay (criterio F1).
- Los elementos técnicos SAT (sellos, cadena original, números de serie) se leen del `TimbreFiscalDigital` del XML del PAC, no se calculan en la aplicación.

**Más información de la tarea:**
Ver sección *"Fuentes de Datos para el PDF"* en `TPSC-RE-FU-021_BD.md`. Ver criterios A1–J2 y Reglas 2, 4 y 7 en `TPSC-RE-FU-021.md`. La Tarea 15 de RE-FU-019 (`FacturaAdelantadoPreviewService`) consume este mapeo para el preview; esta tarea extiende el modelo con los campos del `TimbreFiscalDigital` que RE-FU-019 dejó como placeholder.

**Recursos:**
- `TPSC-RE-FU-021_BD.md` — Tabla de fuentes de datos por sección del PDF
- `TPSC-RE-FU-021.md` — Criterios A1–J2
- TPSC-RE-FU-019 Tarea 15 — `FacturaAdelantadoPreviewService` (placeholder de template)
- CFDIs reales de referencia: Mungen 2374, Golocaer 7156, Proquifa 20913, Proveedora 143103

---

## TAREA 10

**[ RE-FU-021 ] [SERV-TRANSACT] Persistencia del PDF de Factura México en Minio tras timbrado exitoso**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura — Persistencia Artefacto Fiscal

**Consideraciones previas:**
- Se ejecuta inmediatamente después del timbrado exitoso del PAC (TurboPac), como parte del flujo de los módulos que generan facturas (FAA RE-FU-019 y Validar Cobro).
- El PDF se genera una única vez al timbrar; posteriormente se sirve desde Minio sin regeneración (criterio J2 del requisito). La Factura es artefacto fiscal inmutable.
- La persistencia utiliza el patrón existente: `INSERT Archivo` con `FileBucket='facturas'` y la referencia se almacena en `CFDI`.
- Depende de las Tareas 5–9 (templates DocumentBuilder y mapeo disponibles).
- Si la generación o persistencia del PDF falla tras el timbrado exitoso, debe reintentarse sin re-timbrar ante el PAC.
- La comunicación con ProquifaDotNet (.NET Framework 4.8) para actualizar referencias se realiza mediante llamadas entre APIs.
- **Relación con RE-FU-019:** La Tarea 13 de RE-FU-019 (`FacturaAdelantadoGenerarService`) reservó los pasos 10–11 del flujo (almacenar PDF+XML en Minio, INSERT Archivo x2) con un placeholder indicando que el PDF real se define en requisito independiente. Esta tarea implementa esos pasos reales con el PDF completo CFDI 4.0; no es una tarea nueva duplicada, es la implementación definitiva de lo que RE-FU-019 dejó pendiente.

**Objetivo general:**
Implementar el servicio que, tras el timbrado exitoso del PAC, genera el PDF definitivo de la Factura CFDI 4.0 (con `TimbreFiscalDigital` completo) y lo persiste en Minio vía la tabla `Archivo`, referenciado desde la tabla `CFDI` en ProquifaDotNet.

**Objetivos específicos:**
- Invocar `FacturaMexicoPdfMappingService` (Tarea 5) con el `IdCFDI` y el XML timbrado del PAC para obtener el `FacturaPdfModel` definitivo.
- Invocar `GenerarFacturaMexicoPdfService` (Tarea 6) para generar el PDF en bytes.
- `INSERT Archivo` (PDF, `FileBucket='facturas'`, `IdRegion='MEX'`) en ProquifaDotNet.Finanzas.
- Actualizar la referencia del PDF en `CFDI` mediante llamada API a ProquifaDotNet.
- Garantizar atomicidad: si la persistencia en Minio falla, reintentar sin re-timbrar. Registrar en log con Serilog el resultado (módulo, `IdCFDI`, fecha, resultado).

**Resultado esperado:**
PDF de la Factura CFDI 4.0 persistido en Minio como artefacto fiscal inmutable, referenciado desde la tabla `CFDI`, disponible para consultas y envíos posteriores sin regeneración.

**Entregables:**
- Servicio `PersistirFacturaMexicoPdfService` (o equivalente)
- Manejo de reintentos de persistencia sin re-timbrado
- Registro en log con Serilog (módulo, `IdCFDI`, fecha, resultado)

**Criterios de aceptación:**
- El PDF se genera con todos los datos del `TimbreFiscalDigital` del PAC integrados (UUID, sellos, QR, cadena original).
- El `INSERT Archivo` persiste correctamente en Minio con `FileBucket='facturas'`.
- La referencia del PDF queda correctamente almacenada en `CFDI`.
- Si la persistencia falla, el sistema reintenta sin volver a llamar al PAC.
- Al consultar el PDF posteriormente, el sistema sirve el archivo desde Minio sin regenerarlo.
- La operación queda registrada en log con `IdCFDI`, fecha y resultado.

**Más información de la tarea:**
Ver criterios J1–J2, Regla 5 en `TPSC-RE-FU-021.md`, y sección *"Persistencia del PDF"* en `TPSC-RE-FU-021_BD.md`.

**Recursos:**
- `TPSC-RE-FU-021_BD.md` — Sección "Persistencia del PDF", patrón INSERT Archivo
- `TPSC-RE-FU-021.md` — Criterios J1–J2, Regla 5

---

## TAREA 11

**[ RE-FU-021 ] [IMP-EXIST-SERVICE] Extender FacturaAdelantadoGenerarService con persistencia real del PDF tras timbrado (pasos 10-11 RE-FU-019 T13)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura por Adelantado — Generación México

**Consideraciones previas:**
- La Tarea 13 de RE-FU-019 (`FacturaAdelantadoGenerarService`) dejó los pasos 10-11 del flujo como placeholder: "almacenar PDF+XML en Minio — PDF real se define en requisito independiente".
- Esta tarea implementa esos pasos con la lógica real: invocar `PersistirFacturaMexicoPdfService` (Tarea 10) tras el timbrado exitoso del PAC.
- Aplica únicamente a facturas de región México (GOL/MUN/PRO/PQF). El flujo Perú (RE-FU-020) tiene su propio branch.
- Depende de la Tarea 10 (`PersistirFacturaMexicoPdfService`) que debe estar disponible antes de integrar.
- No se modifica la lógica de timbrado ni los pasos 1-9 del flujo existente; solo se reemplazan los pasos 10-11 placeholder.

**Objetivo general:**
Reemplazar los pasos 10-11 placeholder de `FacturaAdelantadoGenerarService` (RE-FU-019 T13) con la llamada real a `PersistirFacturaMexicoPdfService`, completando el flujo de generación de la Factura por Adelantado México con la persistencia del PDF definitivo CFDI 4.0 en Minio.

**Objetivos específicos:**
- Inyectar `IPersistirFacturaMexicoPdfService` en `FacturaAdelantadoGenerarService`.
- Reemplazar el paso 10 placeholder con: `await _persistirFacturaMexicoPdfService.PersistirAsync(idCFDI, response.XmlTimbrado)`.
- Verificar que el paso 11 (INSERT Archivo XML — patrón existente) permanece sin cambios.
- Validar que si `PersistirAsync` falla, el error se maneja sin re-timbrar ante el PAC.
- Registrar en Serilog el resultado de la persistencia dentro del flujo de generación.

**Resultado esperado:**
`FacturaAdelantadoGenerarService` con los pasos 10-11 implementados definitivamente: tras el timbrado exitoso del PAC, el PDF CFDI 4.0 se genera y persiste en Minio como artefacto fiscal inmutable antes de retornar el response al cliente.

**Entregables:**
- `FacturaAdelantadoGenerarService` actualizado (pasos 10-11 reales, sin placeholder)
- Prueba de integración: flujo completo generar FAA México → PDF persistido en Minio

**Criterios de aceptación:**
- Tras el timbrado exitoso, el PDF se persiste en Minio y la referencia queda en `CFDI` antes de retornar el response.
- Si la persistencia del PDF falla, el sistema no re-timbra ante el PAC y retorna el error correspondiente.
- El flujo de timbrado (pasos 1-9) no presenta regresiones respecto a RE-FU-019.
- La operación queda registrada en Serilog con `IdCFDI`, fecha y resultado.

**Más información de la tarea:**
Ver sección "Integración con RE-FU-019 — T13 pasos 10-11" en `TPSC-RE-FU-021-Back.md`. Ver Tarea 13 en `TPSC-RE-FU-019-Tareas.md` (placeholder original).

**Recursos:**
- `TPSC-RE-FU-021-Back.md` — Sección Parte B, integración T13 pasos 10-11
- TPSC-RE-FU-019 Tarea 13 — `FacturaAdelantadoGenerarService` (placeholder de pasos 10-11)

---

## TAREA 12

**[ RE-FU-021 ] [IMP-EXIST-SERVICE] Extender FacturaAdelantadoPreviewService con template real de Factura México (RE-FU-019 T15)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura por Adelantado — Previsualización PDF México

**Consideraciones previas:**
- La Tarea 15 de RE-FU-019 (`FacturaAdelantadoPreviewService`) dejó la resolución del template como placeholder: "template placeholder — definir en requisito independiente".
- Esta tarea reemplaza ese placeholder con la resolución dinámica del `TemplateKey` de DocumentBuilder según la empresa emisora del pedido (GOL/MUN/PRO/PQF).
- Depende de las Tareas 5-8 (templates `GOL/MUN/PRO/PQF_MEX_FAC` registrados en DocumentBuilder) y de la Tarea 9 (`FacturaMexicoPdfMappingService.MapearPreviewAsync` disponible).
- El preview usa el modelo sin `TimbreFiscalDigital` (campos de Sección H en null): UUID, sellos, QR y cadena original no están presentes hasta el timbrado real.
- Aplica únicamente a facturas de región México. El flujo Perú (RE-FU-020) tiene su propio branch de preview.

**Objetivo general:**
Reemplazar el template placeholder de `FacturaAdelantadoPreviewService` (RE-FU-019 T15) con la resolución dinámica del `TemplateKey` de DocumentBuilder según la `EmpresaClave` del pedido, completando el flujo de previsualización del PDF de la Factura por Adelantado México con las plantillas reales CFDI 4.0.

**Objetivos específicos:**
- Inyectar `IFacturaMexicoPdfMappingService` en `FacturaAdelantadoPreviewService`.
- Invocar `MapearPreviewAsync(idCFDIGenerada)` para obtener el `FacturaPdfModel` sin `TimbreFiscalDigital`.
- Resolver el `TemplateKey` dinámicamente según `EmpresaClave`: GOL → `GOL_MEX_FAC`, MUN → `MUN_MEX_FAC`, PRO → `PRO_MEX_FAC`, PQF → `PQF_MEX_FAC`.
- Invocar DocumentBuilder con el `TemplateKey` resuelto y el modelo para generar el PDF en bytes.
- Retornar el PDF en memoria sin persistir en Minio (preview no es artefacto fiscal).

**Resultado esperado:**
`FacturaAdelantadoPreviewService` con la generación real del PDF de previsualización usando los templates CFDI 4.0 de DocumentBuilder, diferenciado por empresa emisora, con todos los campos disponibles excepto los del `TimbreFiscalDigital`.

**Entregables:**
- `FacturaAdelantadoPreviewService` actualizado (template real, sin placeholder)
- Prueba de integración: flujo completo preview FAA México por empresa (GOL/MUN/PRO/PQF) → PDF en memoria con datos correctos

**Criterios de aceptación:**
- El PDF de preview generado corresponde al template de la empresa emisora del pedido (branding correcto por empresa).
- El PDF de preview NO contiene UUID, sellos, QR ni cadena original (campos Sección H vacíos o ausentes).
- El flujo existente de `FacturaAdelantadoPreviewService` (pasos previos al template) no presenta regresiones.
- Las 4 empresas emisoras (GOL/MUN/PRO/PQF) generan correctamente su PDF de preview con el template correspondiente.

**Más información de la tarea:**
Ver sección "Integración con RE-FU-019 — T15 template" en `TPSC-RE-FU-021-Back.md`. Ver Tarea 15 en `TPSC-RE-FU-019-Tareas.md` (placeholder original).

**Recursos:**
- `TPSC-RE-FU-021-Back.md` — Sección Parte B, integración T15 template preview
- TPSC-RE-FU-019 Tarea 15 — `FacturaAdelantadoPreviewService` (placeholder de template)

---

## TAREA 13

**[ RE-FU-021 ] [IMP-EXIST-SERVICE] Integrar persistencia del PDF de Factura México en el flujo Validar Cobro (Genera Factura Normal PPD)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Validar Cobro — Generación Factura Normal PPD México

**Consideraciones previas:**
- El requisito RE-FU-021 es explícito en su alcance: "Facturas originadas en Validar Cobro (cobro recibido aplicado a Proforma de Prepago sin controlados)."
- En el flujo de Validar Cobro, el paso "Genera Factura Normal PPD" es el punto de timbrado exitoso ante el PAC. Es inmediatamente después de este paso donde debe invocarse `PersistirFacturaMexicoPdfService` (Tarea 10) para generar y persistir el PDF como artefacto fiscal inmutable.
- El flujo completo de Validar Cobro es: Genera Factura Normal PPD → Pendiente en Validar Pago → Validar Pago (Tesorería, máx 72 hrs) → ¿Pago válido? → Sí: Genera Complemento de Pago → Desbloquea Tramitar Pedido / No: Elimina pago y correo del buzón (reintenta).
- Esta tarea cubre únicamente la integración del PDF en el paso "Genera Factura Normal PPD". La generación del Complemento de Pago pertenece a un requisito independiente (ver alcance del RE-FU-021: "Generación del Complemento de Pago: se documenta en requisito independiente del módulo Validar Cobro").
- Aplica únicamente a facturas de región México (GOL/MUN/PRO/PQF).
- Depende de la Tarea 10 (`PersistirFacturaMexicoPdfService`) disponible antes de integrar.
- Si la generación o persistencia del PDF falla tras el timbrado exitoso, debe reintentarse sin re-timbrar ante el PAC.

**Objetivo general:**
Integrar la llamada a `PersistirFacturaMexicoPdfService` en el servicio de Validar Cobro que ejecuta el paso "Genera Factura Normal PPD", de modo que el PDF de la Factura CFDI 4.0 se genere y persista en Minio como artefacto fiscal inmutable inmediatamente después del timbrado exitoso ante el PAC, cubriendo el alcance del RE-FU-021 para facturas originadas en Validar Cobro.

**Objetivos específicos:**
- Identificar el servicio/comando de Validar Cobro en ProquifaDotNet.Finanzas que ejecuta el timbrado de la Factura Normal PPD.
- Inyectar `IPersistirFacturaMexicoPdfService` en dicho servicio.
- Invocar `PersistirAsync(idCFDI, xmlTimbrado)` inmediatamente después del timbrado exitoso del PAC, antes de transicionar al estado "Pendiente en Validar Pago".
- Validar que si `PersistirAsync` falla, el error se maneja sin re-timbrar ante el PAC y el flujo de Validar Cobro continúa correctamente hacia "Pendiente en Validar Pago".
- Registrar en Serilog el resultado de la persistencia (módulo, `IdCFDI`, fecha, resultado).

**Resultado esperado:**
El flujo de Validar Cobro genera y persiste el PDF de la Factura CFDI 4.0 en Minio como artefacto fiscal inmutable inmediatamente tras el timbrado exitoso, de forma consistente con el comportamiento implementado para Factura por Adelantado (Tarea 11).

**Entregables:**
- Servicio de Validar Cobro (Genera Factura Normal PPD) actualizado con llamada real a `PersistirFacturaMexicoPdfService`
- Manejo de error de persistencia sin re-timbrado
- Registro en Serilog (módulo, `IdCFDI`, fecha, resultado)

**Criterios de aceptación:**
- Tras el timbrado exitoso en el flujo de Validar Cobro, el PDF se persiste en Minio y la referencia queda en `CFDI` antes de transicionar al estado "Pendiente en Validar Pago".
- Si la persistencia del PDF falla, el sistema no re-timbra ante el PAC y el flujo continúa hacia "Pendiente en Validar Pago" registrando el error.
- El flujo de Validar Cobro (estados: Genera Factura → Pendiente en Validar Pago → Validar Pago → Pago válido → Complemento de Pago → Desbloquea Tramitar Pedido) no presenta regresiones.
- Las 4 empresas emisoras México (GOL/MUN/PRO/PQF) generan correctamente su PDF de Factura Normal PPD con el template correspondiente.
- La operación queda registrada en Serilog con `IdCFDI`, fecha y resultado.

**Más información de la tarea:**
Ver sección "Alcance — Aplica a" en `TPSC-RE-FU-021.md` (facturas originadas en Validar Cobro). Ver Gap 5 en el análisis de validación de tareas vs requisito. Ver Tarea 10 (`PersistirFacturaMexicoPdfService`) que es la dependencia directa de esta tarea.

**Recursos:**
- `TPSC-RE-FU-021.md` — Alcance (facturas de Validar Cobro), Regla 1, Criterios J1–J2
- `TPSC-RE-FU-021-Back.md` — Parte B, `PersistirFacturaMexicoPdfService`
- Diagrama de flujo Validar Cobro: Genera Factura Normal PPD → Pendiente Validar Pago → Validar Pago → Pago válido? → Complemento de Pago / Elimina pago