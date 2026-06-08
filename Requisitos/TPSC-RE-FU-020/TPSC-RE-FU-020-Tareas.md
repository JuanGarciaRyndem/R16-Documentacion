# Tareas BackEnd - TPSC-RE-FU-020
**Requisito:** Factura por Adelantado: Detalle Perú
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10) + ProquifaDotNet.Timbrado (.NET Core 10, NUEVA)

---

> **Orden de ejecución sugerido:** BD bloqueantes → BD configuración → servicios backend

---

## Resumen de tareas

| # | Clave | Título simple | Tipo | Bloqueante |
|---|-------|--------------|------|-----------|
| 1 | CREATE-TABL-CH | Crear catAfectacionIGV + DML catálogo 7 SUNAT | BD | ✅ |
| 2 | UPDATE-TABL-M | Campos SUNAT en Producto y catUnidad | BD | ✅ |
| 3 | UPDATE-TABL-CH | INSERT EmpresaFolio GOLPERU + AppSetting SUNAT | BD | — |
| 4 | UPDATE-TABL-CH | UPDATE Empresa GOLPERU datos legales | BD | — |
| 5 | QUERY-M | Consulta pedidos pendientes FAA por cliente Perú | Back | — |
| 6 | ALG-COMPLX-LOGIC | Algoritmo XML UBL 2.1 CPE tipo 01 SUNAT | Back | — |
| 7 | IMPL-THIRD-SERV | Integración OSE/PSE SUNAT | Back | — |
| 8 | SERV-COMPLEX-TRANSACT | Servicio transaccional timbrado FAA Perú | Back | — |
| 9 | CREATE-PDF | PDF CPE tipo 01 SUNAT Perú | Back | — |
| 10 | ATTACHED-EMAIL | Envío correo FAA Perú (PDF + XML) | Back | — |
| 11 | SERV-TRANSACT | Salida operativa Validar Cobro Perú (sin Legacy) | Back | — |

---

## TAREA 1

**[ RE-FU-020 ] [CREATE-TABL-CH] Crear tabla catAfectacionIGV e insertar catálogo 7 SUNAT**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogos

**Consideraciones previas:**
- La tabla `catAfectacionIGV` no existe en BD. Es necesaria para mapear la afectación al IGV (catálogo 7 SUNAT) por línea de partida en el XML UBL 2.1.
- Es **BLOQUEANTE** para el desarrollo de la facturación Perú: sin este catálogo no es posible generar el CPE tipo 01.
- Bloquea la Tarea 2 (que agrega el FK desde `Producto`).

**Objetivo general:**
Crear la tabla `catAfectacionIGV` en ProquifaDotNet y poblarla con los valores del catálogo 7 SUNAT, habilitando la asociación de cada producto a su tipo de afectación al IGV.

**Objetivos específicos:**
- Ejecutar el DDL de creación con columnas: `IdCatAfectacionIGV` (PK, NEWID), `Clave` varchar(4) UQ, `Descripcion` varchar(200), `Activo` bit DEFAULT 1.
- Insertar los registros del catálogo 7 SUNAT: 10 (Gravado — Op. Onerosa), 11 (Gravado — Retiro), 20 (Exonerado), 30 (Inafecto), 40 (Exportación), entre otros.
- Validar que PK, UQ y DEFAULT queden correctamente definidos.

**Resultado esperado:**
Tabla `catAfectacionIGV` existente en ProquifaDotNet con los valores del catálogo 7 SUNAT insertados y lista para ser referenciada por `Producto`.

**Entregables:**
- Script DDL: `CREATE TABLE catAfectacionIGV`
- Script DML: `INSERT` catálogo 7 SUNAT
- Script de validación (`SELECT` con conteo de filas)

**Criterios de aceptación:**
- La tabla existe con la estructura definida en `TPSC-RE-FU-020_BD.md`.
- Los registros del catálogo 7 SUNAT están insertados y son consultables.
- Ningún objeto existente en BD presenta errores tras la creación.

**Más información de la tarea:**
Ver sección *"Brecha Bloqueante: Datos Fiscales SUNAT del Producto"* en `TPSC-RE-FU-020_BD.md`.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — Script propuesto `CREATE TABLE catAfectacionIGV`
- Catálogo 7 SUNAT — Anexo 8 de la R.S. 097-2012/SUNAT

---

## TAREA 2

**[ RE-FU-020 ] [UPDATE-TABL-M] Agregar campos fiscales SUNAT a tabla Producto y catUnidad**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogo de Productos

**Consideraciones previas:**
- **BLOQUEANTE:** sin `CodigoSUNAT`, `ClaveSUNAT` e `IdCatAfectacionIGV` en las tablas de productos no es posible emitir el XML UBL 2.1 válido para SUNAT (datos obligatorios por partida).
- Depende de la Tarea 1 (`catAfectacionIGV` debe existir antes de agregar el FK).
- Las columnas se agregan como NULL para no romper registros existentes.
- Afecta `Producto` (2 columnas nuevas) y `catUnidad` (1 columna nueva).

**Objetivo general:**
Ampliar las tablas `Producto` y `catUnidad` con los campos exigidos por SUNAT (catálogos 25, 6 y 7) para poder incluir en cada partida del CPE tipo 01 el código del producto, la unidad de medida y la afectación al IGV.

**Objetivos específicos:**
- `ALTER TABLE dbo.Producto ADD CodigoSUNAT varchar(10) NULL` — catálogo 25 SUNAT.
- `ALTER TABLE dbo.catUnidad ADD ClaveSUNAT varchar(10) NULL` — catálogo 6 SUNAT (ej.: KGM, NIU, ZZ).
- `ALTER TABLE dbo.Producto ADD IdCatAfectacionIGV uniqueidentifier NULL CONSTRAINT FK_Producto_AfectacionIGV FOREIGN KEY REFERENCES catAfectacionIGV(IdCatAfectacionIGV)`.
- Verificar que los ALTER no rompan vistas, stored procedures ni triggers dependientes.

**Resultado esperado:**
`Producto` y `catUnidad` con los nuevos campos SUNAT disponibles. Los campos quedan en NULL hasta que el equipo de operaciones capture los valores reales por producto.

**Entregables:**
- Script DDL: `ALTER TABLE Producto` (x2), `ALTER TABLE catUnidad`
- Script de validación (`SELECT` con los nuevos campos)
- Checklist de objetos dependientes verificados

**Criterios de aceptación:**
- `Producto.CodigoSUNAT` existe y acepta valores varchar(10).
- `catUnidad.ClaveSUNAT` existe y acepta valores del catálogo 6 SUNAT.
- `Producto.IdCatAfectacionIGV` existe con FK activo hacia `catAfectacionIGV`.
- Ningún SP, vista ni trigger presenta errores de compilación tras los ALTER.

**Más información de la tarea:**
Ver sección *"Brecha Bloqueante: Datos Fiscales SUNAT del Producto"* en `TPSC-RE-FU-020_BD.md`. Brecha 1 referenciada en `TPSC-RE-FU-005`.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — Script propuesto ALTER TABLE
- Catálogos 6 y 25 SUNAT — Anexo 8 R.S. 097-2012/SUNAT

---

## TAREA 3

**[ RE-FU-020 ] [UPDATE-TABL-CH] INSERT EmpresaFolio GOLPERU e INSERT AppSetting SUNAT en ProquifaDotNet.Timbrado**

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Base de Datos — Configuración de Timbrado

**Consideraciones previas:**
- La tabla `EmpresaFolio` fue creada en RE-FU-018; esta tarea solo agrega la fila de GOLPERU.
- La serie SUNAT de Golocaer S.A.C. (ej.: F001) está **pendiente de confirmar** con la empresa antes de ejecutar en producción.
- Los valores de AppSetting (endpoint OSE/PSE, RUC emisor, ruta certificado) están **pendientes de confirmar** con el equipo técnico.
- No afecta los folios existentes de GOL, MUN, PRO ni PQF.

**Objetivo general:**
Registrar la configuración operativa de GOLPERU en ProquifaDotNetTimbrado: folio inicial y serie de facturación SUNAT en `EmpresaFolio`, y los parámetros de conexión con el OSE/PSE en `AppSetting`.

**Objetivos específicos:**
- `INSERT EmpresaFolio`: `EmpresaClave='GOLPERU'`, `Serie='F001'`, `UltimoFolio=0`, `FormatoFolio='F001-{folio:00000000}'`, `LongitudMaxima=8`.
- `INSERT AppSetting`: `SUNAT_OSE_Endpoint`, `SUNAT_RUC_Emisor`, `SUNAT_CertPath`.
- Ajustar `Serie` y `UltimoFolio` al último correlativo usado en producción antes de ejecutar en ambiente productivo.

**Resultado esperado:**
GOLPERU disponible en `EmpresaFolio` con su serie SUNAT configurada; parámetros de conexión OSE/PSE registrados en `AppSetting` de ProquifaDotNetTimbrado.

**Entregables:**
- Script DML: `INSERT EmpresaFolio GOLPERU`
- Script DML: `INSERT AppSetting` SUNAT (3 registros)
- Script de validación (`SELECT`)

**Criterios de aceptación:**
- La fila GOLPERU existe en `EmpresaFolio` con `Serie`, `FormatoFolio` y `LongitudMaxima` correctos.
- Los 3 registros de AppSetting SUNAT existen y son consultables.
- El INSERT no afecta los registros de otras empresas.

**Más información de la tarea:**
Ver secciones *"EmpresaFolio GOLPERU"* y *"AppSetting ProquifaDotNetTimbrado (Peru)"* en `TPSC-RE-FU-020_BD.md`.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — Scripts propuestos INSERT

---

## TAREA 4

**[ RE-FU-020 ] [UPDATE-TABL-CH] UPDATE Empresa GOLPERU — datos legales Golocaer S.A.C. (RUC, domicilio, ubigeo)**

**Aplicativos:** ProquifaDotNet

**Módulos:** Base de Datos — Catálogo de Empresas

**Consideraciones previas:**
- Los datos legales de Golocaer S.A.C. (RUC, domicilio fiscal, ubigeo SUNAT) no están registrados actualmente en BD.
- Esta tarea está **bloqueada** hasta que el equipo proporcione los datos reales de Golocaer S.A.C.
- Sin el RUC del emisor no es posible generar el XML UBL 2.1 válido ante SUNAT.
- No requiere cambios en la estructura de la tabla (solo DML UPDATE).

**Objetivo general:**
Actualizar el registro de la empresa GOLPERU en la tabla `Empresa` de ProquifaDotNet con los datos legales requeridos para emitir el CPE tipo 01: RUC, domicilio fiscal y ubigeo SUNAT.

**Objetivos específicos:**
- `UPDATE Empresa SET RUC='20XXXXXXXXXX', DomicilioFiscal='...', Ubigeo='...' WHERE EmpresaClave='GOLPERU'`.
- Verificar que el RUC coincide con el registro activo de Golocaer S.A.C. ante SUNAT.
- Confirmar que el ubigeo corresponde al catálogo de ubigeos de SUNAT.

**Resultado esperado:**
Empresa GOLPERU con RUC, domicilio fiscal y ubigeo SUNAT correctos en ProquifaDotNet, habilitando la emisión del CPE tipo 01.

**Entregables:**
- Script DML: `UPDATE Empresa GOLPERU`
- Script de validación (`SELECT`)

**Criterios de aceptación:**
- `Empresa` GOLPERU tiene RUC, domicilio y ubigeo actualizados y válidos.
- El RUC coincide con el registro activo de Golocaer S.A.C. ante SUNAT.

**Más información de la tarea:**
Ver sección *"Datos Legales Golocaer SAC"* en `TPSC-RE-FU-020_BD.md`. Tarea bloqueada hasta recibir datos de la empresa.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — Tabla de datos legales pendientes

---

## TAREA 5

**[ RE-FU-020 ] [QUERY-M] Consulta de datos del cliente y pedidos pendientes FAA por cliente (Región Perú)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura por Adelantado — Detalle por Cliente (Perú)

**Consideraciones previas:**
- La vista `vtpProformaAdelanto` fue creada en RE-FU-019 y ya filtra por `Region`; esta consulta la reutiliza con el filtro `Region='PER'`.
- Los datos del cliente para la cabecera del Detalle incluyen: Razón Social, RUC (no RFC), moneda de facturación, clasificación crediticia.
- Cada fila del listado expone: Pedido Interno, Fecha, Condición de Pago, Empresa Emisora (GOLPERU), Subtotal, IGV, Monto Total y estado FAA (PendienteGenerar / PendienteEnviar).
- La visibilidad debe filtrarse por la cartera del usuario (campo `Cobrador` del catálogo de clientes).

**Objetivo general:**
Implementar el endpoint y la consulta que proveen los datos de la cabecera del cliente y el listado de sus pedidos pendientes de Factura por Adelantado para la pantalla Detalle de la Región Perú.

**Objetivos específicos:**
- Endpoint GET que reciba `IdCliente` y devuelva datos de cabecera (RUC, Razón Social, moneda, clasificación) y listado de pedidos con su estado FAA.
- Aplicar filtro de cartera del usuario (solo pedidos del cobrador autenticado).
- Consumir `vtpProformaAdelanto` con `Region='PER'` para determinar el estado de cada pedido (PendienteGenerar / PendienteEnviar).
- No exponer campos SAT en la respuesta (sin Método de Pago, sin Uso CFDI).

**Resultado esperado:**
Endpoint funcional que devuelve cabecera del cliente y listado de pedidos pendientes FAA Perú, filtrado correctamente por cartera del usuario.

**Entregables:**
- Endpoint GET `Detalle cliente FAA Perú`
- DTO de respuesta (cabecera + listado de pedidos)
- Query/SP contra `vtpProformaAdelanto` con filtro `Region='PER'`

**Criterios de aceptación:**
- El endpoint devuelve únicamente pedidos del cliente con `Region='PER'` en estado PendienteGenerar o PendienteEnviar.
- La cabecera muestra RUC (no RFC) y no incluye campos SAT.
- Un usuario solo puede consultar el detalle de clientes asignados a su cartera; retorna 403 para clientes fuera de su cartera.
- Pedidos con `Enviada=1` no aparecen en el listado.

**Más información de la tarea:**
Ver criterios A1–A4 y sección *"Estados del Pedido"* en `TPSC-RE-FU-020-Pendiente.md` y `TPSC-RE-FU-020_BD.md`.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — Tablas leídas, tabla de estados FAA
- `TPSC-RE-FU-020-Pendiente.md` — Reglas 3, 4, 16

---

## TAREA 6

**[ RE-FU-020 ] [ALG-COMPLX-LOGIC] Algoritmo de construcción del XML UBL 2.1 (CPE tipo 01 SUNAT) para Factura por Adelantado Perú**

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Timbrado — Generación XML SUNAT

**Consideraciones previas:**
- El formato UBL 2.1 de SUNAT es diferente al CFDI mexicano; no se reutiliza el generador de México.
- Los campos obligatorios por partida incluyen: `CodigoSUNAT` (catálogo 25), `ClaveSUNAT` de `catUnidad` (catálogo 6), `AfectacionIGV` (catálogo 7), valor unitario e importe.
- **Bloqueado** hasta que las Tareas 1 y 2 estén completadas (datos SUNAT en BD).
- No incluye Método de Pago, Forma de Pago ni Complemento de Pago (no existen en Perú).
- Incluye `TipoOperacion` del catálogo 51 SUNAT (pendiente definir si es fijo "0101" o configurable por el operador).
- IGV fijo al 18%; serie alfanumérica 4 chars + correlativo hasta 8 dígitos.

**Objetivo general:**
Implementar el algoritmo que, a partir de los datos del pedido, el cliente y el emisor, construya el documento XML UBL 2.1 válido para su envío al OSE/PSE SUNAT como CPE tipo 01 (Factura).

**Objetivos específicos:**
- Mapear datos del pedido: cabecera (RUC emisor, RUC receptor, serie, correlativo, fecha, moneda, tipo operación) y partidas (código SUNAT, descripción, UdM SUNAT, cantidad, valor unitario, importe, afectación IGV).
- Calcular IGV al 18% por línea y totales de la factura.
- Construir el XML conforme al esquema UBL 2.1 y las validaciones del Anexo 8 SUNAT.
- Firmar digitalmente el XML con el certificado de Golocaer S.A.C.
- Serializar y retornar el XML firmado listo para enviar al OSE/PSE.

**Resultado esperado:**
XML UBL 2.1 firmado digitalmente, válido para su envío al OSE/PSE SUNAT, generado a partir de los datos del pedido de ProquifaDotNet.

**Entregables:**
- Clase/servicio `GeneradorXmlSunatService` (o equivalente)
- Mapeo de campos del pedido a UBL 2.1
- Prueba unitaria con XML de muestra válido (basado en la factura real E001-362 de Golocaer)

**Criterios de aceptación:**
- El XML generado pasa la validación del esquema UBL 2.1 de SUNAT.
- Los campos de catálogos 6, 7 y 25 SUNAT están presentes en cada partida.
- El IGV se calcula al 18% y los totales cuadran con el pedido original.
- El XML **no** incluye campos de Método de Pago, Forma de Pago ni Complemento de Pago.
- El XML está firmado digitalmente con el certificado de GOLPERU.

**Más información de la tarea:**
Ver Reglas 6, 7, 12 y 17, y Riesgos 1 y 3 en `TPSC-RE-FU-020-Pendiente.md`. Brecha 1 y Brecha 3 de `TPSC-RE-FU-005`.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — Tablas leídas, diferencias MEX vs PER
- R.S. 097-2012/SUNAT, formato UBL 2.1, Anexo 8 catálogos SUNAT
- Factura real de muestra Golocaer S.A.C. E001-362 (08/05/2026)

---

## TAREA 7

**[ RE-FU-020 ] [IMPL-THIRD-SERV] Integración con OSE/PSE SUNAT para timbrado del CPE Perú**

**Aplicativos:** ProquifaDotNet.Timbrado

**Módulos:** Timbrado — Integración OSE/PSE SUNAT

**Consideraciones previas:**
- El proveedor OSE/PSE para Perú está **pendiente de definir** (no se reutiliza el PAC de México). Brecha mayor documentada en Brecha 5 de `TPSC-RE-FU-005`.
- Esta tarea está **bloqueada** hasta que se defina y contrate el OSE/PSE autorizado por SUNAT.
- La integración recibe el XML UBL 2.1 firmado (salida de la Tarea 6) y retorna el CDR (Constancia de Recepción) de aceptación de SUNAT.
- El endpoint del OSE/PSE se configura vía `AppSetting.SUNAT_OSE_Endpoint` (Tarea 3).

**Objetivo general:**
Implementar el cliente HTTP/SOAP que envía el XML UBL 2.1 firmado al OSE/PSE autorizado por SUNAT y procesa la respuesta (CDR de aceptación o error descriptivo).

**Objetivos específicos:**
- Implementar `OsePseClient` que envíe el XML al endpoint configurado en `AppSetting`.
- Procesar la respuesta del OSE/PSE: CDR de aceptación (éxito) o código de error SUNAT con descripción.
- Mapear los códigos de error SUNAT a mensajes legibles para el modal de Alerta del frontend.
- Manejar timeouts y reintentos controlados.
- Registrar en `TimbradoLog` el resultado de cada intento (éxito o error, código, timestamp).

**Resultado esperado:**
Cliente de integración OSE/PSE funcional que envía el XML, recibe el CDR de aceptación y expone el resultado al servicio transaccional de timbrado (Tarea 8).

**Entregables:**
- Clase `OsePseClient` (o equivalente)
- Mapeo de códigos de error SUNAT a mensajes de usuario
- Configuración de timeout y reintentos
- Log en `TimbradoLog`

**Criterios de aceptación:**
- El cliente envía correctamente el XML al endpoint del OSE/PSE.
- En caso de éxito, retorna el CDR de aceptación de SUNAT.
- En caso de error, retorna el código y descripción del error SUNAT mapeado a mensaje de usuario.
- Cada intento queda registrado en `TimbradoLog`.

**Más información de la tarea:**
Ver Riesgo 2 y sección *"AppSetting ProquifaDotNetTimbrado (Peru)"* en `TPSC-RE-FU-020_BD.md`. Brecha 5 de `TPSC-RE-FU-005`.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — AppSetting SUNAT
- Documentación técnica del OSE/PSE seleccionado (pendiente)

---

## TAREA 8

**[ RE-FU-020 ] [SERV-COMPLEX-TRANSACT] Servicio transaccional de timbrado Factura por Adelantado Perú**

**Aplicativos:** ProquifaDotNet.Finanzas / ProquifaDotNet.Timbrado

**Módulos:** Factura por Adelantado — Timbrado Perú

**Consideraciones previas:**
- Orquesta las Tareas 6 y 7: genera el XML, llama al OSE/PSE y persiste el resultado.
- Depende de las Tareas 1–4 (BD), 6 (XML) y 7 (OSE/PSE).
- En caso de error, **no debe persistir nada** y debe retornar el error al frontend para mostrar el modal de Alerta SUNAT.
- En caso de éxito, la factura queda persistida como artefacto inmutable; no se puede modificar después.
- No hay transferencia a Legacy (Perú no usa Legacy).

**Objetivo general:**
Implementar el servicio transaccional que, al confirmar la generación desde el modal de Previsualización, ejecuta el flujo completo de timbrado: genera el XML UBL 2.1 → lo envía al OSE/PSE → persiste el CFDI (XML+PDF) → actualiza el folio de GOLPERU → actualiza el estado del pedido en ProquifaDotNet.

**Objetivos específicos:**
- Generar el XML UBL 2.1 firmado (invoca servicio Tarea 6).
- Enviar al OSE/PSE y obtener CDR (invoca cliente Tarea 7).
- Si error: retornar el error descriptivo sin persistir nada.
- Si éxito:
  - `INSERT CFDI` en ProquifaDotNetTimbrado con el XML y CDR recibido.
  - `UPDATE EmpresaFolio GOLPERU SET UltimoFolio = UltimoFolio + 1`.
  - `INSERT TimbradoLog`.
  - `UPDATE tpProformaAdelanto SET IdCFDIGenerada = @IdCFDI` en ProquifaDotNet.
  - `INSERT Archivo x2` (PDF + XML, `FileBucket='facturas'`, `IdRegion='PER'`) en ProquifaDotNet.
- Garantizar atomicidad: si cualquier paso falla después del timbrado exitoso, reintentar la persistencia sin re-timbrar.

**Resultado esperado:**
Pedido con su factura timbrada persistida en BD (XML + PDF), folio GOLPERU incrementado, estado del pedido actualizado a PendienteEnviar.

**Entregables:**
- Servicio `TimbrarFacturaAdelantoPeruService` (o equivalente)
- Manejo de errores y atomicidad de la persistencia post-timbrado
- Pruebas de integración (éxito y error OSE)

**Criterios de aceptación:**
- En caso de error OSE/PSE: no se persiste nada y el error descriptivo llega al frontend.
- En caso de éxito: CFDI, TimbradoLog, Archivo (x2) y tpProformaAdelanto quedan actualizados.
- El folio de GOLPERU se incrementa correctamente sin saltos ni duplicados.
- El pedido cambia de estado a PendienteEnviar tras el timbrado exitoso.
- La factura persistida es inmutable (no puede modificarse posteriormente).

**Más información de la tarea:**
Ver criterios E1–E3, Reglas 9–12, y sección *"Flujo de Datos — Paso 2 TIMBRAR"* en `TPSC-RE-FU-020_BD.md`.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — Flujo de datos
- `TPSC-RE-FU-020-Pendiente.md` — Reglas 9, 10, 11, 12

---

## TAREA 9

**[ RE-FU-020 ] [CREATE-PDF] Generación del PDF del CPE tipo 01 (Factura SUNAT) para Perú**

**Aplicativos:** ProquifaDotNet.Finanzas / DocumentBuilder

**Módulos:** Factura por Adelantado — Representación PDF

**Consideraciones previas:**
- El diseño visual del PDF de la factura SUNAT Perú se documenta en TPSC-RE-FU-022; esta tarea implementa la generación técnica del artefacto.
- El PDF se genera dos veces en el flujo: una para la previsualización (antes del timbrado) y una vez como artefacto fiscal inmutable (tras el timbrado exitoso, integrando el código QR y datos del CDR).
- El PDF de previsualización no lleva firma ni CDR; el PDF persistido sí.
- Depende de los datos de las Tareas 1–4 (campos SUNAT en BD).

**Objetivo general:**
Implementar la generación del PDF de la Factura por Adelantado Perú (CPE tipo 01) con los datos del pedido, del cliente (RUC) y del emisor (Golocaer S.A.C.), conforme al formato visual definido en TPSC-RE-FU-022.

**Objetivos específicos:**
- Implementar la generación del PDF de previsualización (sin CDR) que se muestra en el modal de Previsualización.
- Implementar la generación del PDF definitivo (con número de serie/correlativo, QR y datos del CDR) que se persiste tras el timbrado exitoso.
- Incluir en el PDF: datos del emisor (RUC Golocaer, razón social, domicilio), datos del receptor (RUC cliente, razón social), partidas con descripción, cantidad, UdM, precio unitario, IGV 18% y total.
- **No** incluir campos de Método de Pago ni Forma de Pago.

**Resultado esperado:**
PDF de la Factura por Adelantado Perú generado correctamente en ambas fases (previsualización y definitivo), con el formato visual de TPSC-RE-FU-022.

**Entregables:**
- Servicio/método de generación PDF para CPE tipo 01 Perú
- Plantilla PDF conforme a TPSC-RE-FU-022
- Prueba con datos de pedido real de muestra

**Criterios de aceptación:**
- El PDF de previsualización se genera con todos los datos del pedido antes del timbrado.
- El PDF definitivo incluye serie/correlativo, QR y datos del CDR de SUNAT.
- El PDF **no** contiene campos de Método de Pago ni Forma de Pago.
- IGV al 18% y totales coinciden con los del XML UBL 2.1.

**Más información de la tarea:**
Ver criterio C1 y Regla 9 en `TPSC-RE-FU-020-Pendiente.md`. Diseño visual en TPSC-RE-FU-022.

**Recursos:**
- `TPSC-RE-FU-020-Pendiente.md` — Regla 9, criterio C1
- TPSC-RE-FU-022 — Estructura visual del PDF SUNAT
- DocumentBuilder — proceso de generación de PDF

---

## TAREA 10

**[ RE-FU-020 ] [ATTACHED-EMAIL] Envío de Factura por Adelantado Perú al cliente con adjuntos PDF y XML**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura por Adelantado — Envío al Cliente (Perú)

**Consideraciones previas:**
- El modal de Envío se abre cuando el pedido está en estado PendienteEnviar (factura ya timbrada).
- Los adjuntos (PDF + XML del CPE) provienen del `Archivo` persistido en la Tarea 8; no son editables por el usuario.
- El correo incluye: `Para` (contacto del cliente, editable), `CC` (ESAC asignado, editable), `Asunto` (formato con serie/correlativo + pedido interno, no editable), `Adjuntos` (PDF + XML fijos), `Notas extras` (texto libre opcional).
- El formato exacto del asunto para Perú está **pendiente de confirmar** con el cliente.

**Objetivo general:**
Implementar el envío del correo electrónico de la Factura por Adelantado Perú al cliente, con los adjuntos PDF y XML del CPE tipo 01 timbrado, y actualizar el estado del pedido a `Enviada=1` tras el envío exitoso.

**Objetivos específicos:**
- Endpoint POST que reciba: destinatario (editable), CC (editable), notas extras (opcional) y el `IdProformaAdelanto`.
- Construir el correo con: asunto generado (serie/correlativo + pedido interno), adjuntos PDF y XML del CPE (recuperados de `Archivo`), notas extras.
- Enviar el correo al cliente.
- `INSERT CorreoEnviado` y `INSERT ArchivoCorreoEnviado` (PDF y XML) en ProquifaDotNet.
- `UPDATE tpProformaAdelanto SET Enviada = 1`.

**Resultado esperado:**
Correo enviado al cliente con PDF y XML del CPE adjuntos; estado del pedido actualizado a `Enviada=1`; registros de `CorreoEnviado` y `ArchivoCorreoEnviado` persistidos.

**Entregables:**
- Endpoint POST de envío correo FAA Perú
- Servicio de construcción y envío del correo con adjuntos
- Registros en `CorreoEnviado` y `ArchivoCorreoEnviado`

**Criterios de aceptación:**
- El correo se envía al destinatario con PDF y XML adjuntos del CPE timbrado.
- El asunto incluye la serie/correlativo de la factura y el folio del pedido interno.
- Tras el envío exitoso, `tpProformaAdelanto.Enviada = 1` y el pedido desaparece del listado.
- Los adjuntos no pueden ser removidos por el usuario.
- Si el envío falla, `Enviada` permanece en 0 y se retorna el error al frontend.

**Más información de la tarea:**
Ver criterios F1–F7, G1–G2, Reglas 13 y 14, y sección *"Flujo de Datos — Paso 3 ENVIAR"* en `TPSC-RE-FU-020_BD.md`.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — Flujo de datos paso 3
- `TPSC-RE-FU-020-Pendiente.md` — Reglas 13, 14; criterios F1–G2

---

## TAREA 11

**[ RE-FU-020 ] [SERV-TRANSACT] Salida operativa post-envío — generar pendiente en Validar Cobro (Prepago Perú, sin Legacy)**

**Aplicativos:** ProquifaDotNet.Finanzas

**Módulos:** Factura por Adelantado — Salida Operativa / Validar Cobro

**Consideraciones previas:**
- Esta salida operativa se ejecuta tras el envío exitoso del correo (Tarea 10).
- **Todos** los pedidos de esta fila son Prepago; no existe rama de Crédito.
- **No hay transferencia a Legacy** (de Perú nada va a Legacy). Diferencia clave respecto a México (RE-FU-019).
- El pendiente generado en Validar Cobro permite al equipo de Cobranza conciliar el pago del cliente contra esta Factura.

**Objetivo general:**
Implementar el servicio que, tras el envío exitoso de la Factura por Adelantado Perú, genera el pendiente correspondiente en el módulo Validar Cobro, cerrando el ciclo operativo del pedido en ProquifaDotNet sin ninguna acción hacia Legacy.

**Objetivos específicos:**
- Generar el pendiente en Validar Cobro asociado a la `tpProformaAdelanto` enviada.
- Garantizar que el pendiente se crea **solo si el envío del correo fue exitoso** (no debe crearse si el correo falla).
- Aplicar exclusivamente el flujo Prepago; no existe lógica de Crédito ni de Legacy en esta implementación.
- Registrar en log la creación del pendiente (módulo, IdPedido, fecha).

**Resultado esperado:**
Pendiente creado en Validar Cobro referenciando la Factura por Adelantado Perú enviada, listo para que el equipo de Cobranza concilie el pago del cliente.

**Entregables:**
- Servicio `GenerarPendienteValidarCobroPeruService` (o equivalente)
- Registro en log de la operación

**Criterios de aceptación:**
- El pendiente en Validar Cobro se genera correctamente tras el envío exitoso del correo.
- Si el envío del correo falla, el pendiente **no** se genera.
- No se ejecuta ninguna acción de transferencia hacia Legacy.
- El pendiente está asociado al `IdPedido` y a la factura (`IdCFDI`) correspondientes.

**Más información de la tarea:**
Ver criterio G3, Regla 15, y sección *"Flujo de Datos — Paso 3 ENVIAR"* en `TPSC-RE-FU-020_BD.md`.

**Recursos:**
- `TPSC-RE-FU-020_BD.md` — Diferencias MEX vs PER (sin Legacy, solo Prepago)
- `TPSC-RE-FU-020-Pendiente.md` — Regla 15, criterio G3
