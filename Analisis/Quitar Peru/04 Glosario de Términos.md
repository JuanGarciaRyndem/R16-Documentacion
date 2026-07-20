# Glosario de Términos — Análisis Quitar Perú (Facturación y Timbrado)

Glosario de referencia para los tres documentos del análisis de eliminar la facturación y el timbrado de Perú:
`01 Dejar Fuera Facturacion y Timbrado en Peru.md`, `02 Quitar Perú como análisis Inicial.md`, `03 Quitar Perú — Análisis.md`, `Análisis impacto de eliminar facturación perú.docx` (Frontend) y `Análisis impacto de eliminar facturación perú - Back y BD.docx` (Back y BD).

Los términos están agrupados por dominio para facilitar la lectura.

---

## 1. Marco fiscal — México (SAT)

- **SAT** — Servicio de Administración Tributaria. Autoridad fiscal de México.
- **CFDI** — Comprobante Fiscal Digital por Internet. Documento fiscal electrónico obligatorio en México, versión 4.0 vigente.
- **CFDI 4.0** — Versión actual del CFDI. Anexo 20 del SAT.
- **UUID** — Folio Fiscal Digital de 32 caracteres asignado por el SAT al timbrar. Identificador único de un CFDI.
- **TimbreFiscalDigital (TFD)** — Nodo del XML CFDI que contiene UUID, sellos digitales, cadena original y fecha de timbrado.
- **PAC** — Proveedor Autorizado de Certificación. Empresa autorizada por el SAT para timbrar CFDIs. En R16 se usa **TurboPac** (Quadrum Tecnologías S.A. de C.V., RFC QSO100827UB0).
- **TurboPac** — PAC contratado por PROQUIFA México para timbrar CFDIs. Se consume vía `SapStampingClient` desde ProquifaDotNet.Timbrado.
- **RFC** — Registro Federal de Contribuyentes. Identificador fiscal en México (12 chars para persona moral, 13 para persona física).
- **IVA** — Impuesto al Valor Agregado. En México se aplica al 16 %.
- **CFF** — Código Fiscal de la Federación. Marco legal del SAT (Art. 29 y 29A; Art. 30 obliga conservar el XML mínimo 5 años).
- **Tipo de Comprobante SAT** — Clasificación del CFDI: `I` (Ingreso), `E` (Egreso / Nota de Crédito), `P` (Pago / Complemento de Pago), `T` (Traslado), `N` (Nómina).
- **TipoRelacion 01** — Relación SAT para Notas de Crédito de los documentos relacionados.
- **TipoRelacion 07** — Relación SAT para Aplicación de Anticipo (Factura Anticipo).
- **Uso CFDI** — Catálogo `c_UsoCFDI` del SAT. Define el uso fiscal del comprobante (G01, G02, G03, P01, CP01, etc.).
- **CP01** — Uso CFDI fijo para el receptor en Complementos de Pago.
- **G02** — Uso CFDI default para Notas de Crédito (Devoluciones, descuentos o bonificaciones).
- **Método de Pago SAT** — `PUE` (Pago en Una Exhibición) o `PPD` (Pago en Parcialidades o Diferido).
- **Forma de Pago SAT** — Catálogo `c_FormaPago` (03 Transferencia electrónica, 99 Por definir, etc.).
- **c_MotivoCancelacion** — Catálogo SAT de motivos de cancelación de un CFDI.
- **Cadena Original** — Representación textual del CFDI usada para el sellado digital.
- **Complemento Pagos 2.0** — Complemento del CFDI tipo P para facturas PPD, versión 2.0 vigente.

## 2. Marco fiscal — Perú (SUNAT)

- **SUNAT** — Superintendencia Nacional de Aduanas y de Administración Tributaria. Autoridad fiscal de Perú. **Fuera de alcance en R16.**
- **CPE** — Comprobante de Pago Electrónico. Documento fiscal electrónico peruano (equivalente al CFDI).
- **UBL 2.1** — Universal Business Language 2.1. Formato XML estándar de los CPE SUNAT.
- **Tipo 01** — CPE Factura electrónica.
- **Tipo 07** — CPE Nota de Crédito electrónica.
- **RUC** — Registro Único de Contribuyente. Identificador fiscal peruano de 11 dígitos.
- **Módulo 11** — Algoritmo de validación del dígito verificador del RUC. Se aplica sobre los 10 primeros dígitos contra el 11.º.
- **Tipo de Contribuyente** — Primeros 2 dígitos del RUC: `{10, 15, 17, 20}` según catálogo SUNAT.
- **IGV** — Impuesto General a las Ventas. Perú, tasa 18 %.
- **ISC** — Impuesto Selectivo al Consumo (Perú).
- **ICBPER** — Impuesto al Consumo de las Bolsas de Plástico (Perú).
- **Catálogo 09 SUNAT** — Motivos de Nota de Crédito (01 Anulación, 02 Anulación por error en RUC, 03 Corrección de descripción, 04 Descuento global, 05 Descuento por ítem, 06 Devolución total, 07 Devolución por ítem, 08 Bonificación, 09 Disminución de valor, 10 Otros, 13 Ajuste por crédito fiscal).
- **Catálogo 51 SUNAT** — Tipos de Operación (ejemplo: "0101" Venta interna).
- **cbc:ResponseCode** — Elemento UBL que porta el código del catálogo 09 en la NC.
- **cbc:Description** — Elemento UBL con el sustento del motivo de la NC.
- **cbc:ReferenceID + cac:BillingReference** — Nodos UBL que referencian el comprobante origen por serie-correlativo.
- **cac:CreditNoteLine** — Nodo UBL con la línea de la Nota de Crédito.
- **Serie-Correlativo** — Numeración SUNAT: serie alfanumérica de 4 chars (ej. `F001`) + correlativo de 8 dígitos.
- **OSE** — Operador de Servicios Electrónicos. Proveedor tercero autorizado por SUNAT para timbrar. **Modalidad no seleccionada en R16.**
- **PSE** — Proveedor de Servicios Electrónicos (relacionado con OSE).
- **SEE** — Sistema de Emisión Electrónica de SUNAT. Modalidades: SEE-SOL (portal gratuito), SEE del Contribuyente, SEE-OSE, Facturador SUNAT.
- **Comunicación de Baja** — Mecanismo alterno de anulación en Perú (7 días calendario desde emisión). Excepcional.
- **Agente de Retención IGV** — Bandera del cliente Perú que indica si retiene IGV.
- **Sujeto a Detracción** — Bandera del cliente Perú que indica si aplica detracción SUNAT.
- **Golocaer S.A.C.** — Única empresa emisora peruana del grupo PROQUIFA. RUC 20612772941.
- **SBS** — Superintendencia de Banca, Seguros y AFP. Fuente propuesta de tipo de cambio para Perú (pendiente confirmar).
- **PEN** — Sol Peruano. Moneda base de Perú.

## 3. Estados / documentos operativos

- **Proforma** — Documento comercial sin validez fiscal. Se conserva en Perú (no requiere timbrado).
- **PRF-MMDDAA-Consecutivo** — Foliador global de Proforma para PROQUIFA.
- **Confirmación de Pedido** — Documento que el cliente confirma. En R16 el Paso 3 Perú se limita a este envío.
- **FEE** — Fecha Estimada de Entrega. Se establece al enviar el documento en el Paso 3.
- **FpA** — Factura por Adelantado. Módulo NUEVO en R16. **Excluido de Perú.**
- **PSC** — Pedido Sin Crédito.
- **PCC** — Pedido Con Crédito.
- **PCE** — Pedido Con Estatus (marca de detención en Legacy).
- **NC** — Nota de Crédito. En México: CFDI tipo E. En Perú: CPE tipo 07 (**cancelado en R16**).
- **CDP / CP** — Complemento de Pago. CFDI tipo P. **Solo México.**
- **Buzón de Cobros** — Módulo NUEVO. Mailbot que clasifica correos como "Cobro".
- **COB-mmddaa-consecutivo** — Foliador de Cobros.
- **Guía de Remisión Electrónica (GRE)** — Documento SUNAT de traslado. Brecha B2 (fuera de R16).

## 4. Requisitos R16 mencionados en el análisis

- **RE-004** — Sección Entrega y Facturación (Cat. Clientes) — **Conservar**.
- **RE-005** — Sección Cobros (Cat. Clientes) — **Conservar** con brechas B1-B5 marcadas como fuera de alcance.
- **RE-011** — Tramitación Crédito con controlados (bloqueo de FpA).
- **RE-015** — Tramitación Prepago sin controlados con FpA — **Ajustar** (bloquear FpA en Perú).
- **RE-017** — Proforma Perú (Diseño y PDF) — **Conservar íntegro**.
- **RE-018** — FpA pantalla inicial — **Ajustar** (filtrar clientes Perú).
- **RE-019** — FpA Detalle México.
- **RE-020** — FpA Detalle Perú — **SALE** de alcance.
- **RE-021** — Factura México (PDF CFDI 4.0).
- **RE-022** — Factura Perú (PDF CPE SUNAT) — **SALE** de alcance.
- **RE-023** — Validar Cobro pantalla principal — **Ajustar tooltip**.
- **RE-024** — Validar Cobro Paso 1 México.
- **RE-025** — Validar Cobro Paso 1 Perú — **Conservar**.
- **RE-026** — Validar Cobro Paso 2 México.
- **RE-027** — Validar Cobro Paso 2 Perú — **Simplificar** (retirar NCs, ajustar saldo).
- **RE-028** — Validar Cobro Paso 3 México.
- **RE-029** — Validar Cobro Paso 3 Perú — **Simplificación mayor** (solo Confirmación de Pedido).
- **RE-030** — CDP México (Complemento de Pago).
- **RE-032** — Notas de Crédito México.
- **RE-033** — Notas de Crédito Perú — **SALE** de alcance.
- **RE-034** — Diseño NDC México (PDF).
- **RE-035** — Diseño NDC Perú (PDF) — **SALE** de alcance.

## 5. Brechas conocidas del proyecto

- **B1** — Datos fiscales SUNAT del producto (código SUNAT, unidad de medida, afectación IGV).
- **B2** — Guía de Remisión Electrónica (GRE) Perú.
- **B3** — Catálogo 51 SUNAT (Tipo de Operación).
- **B4** — Disclaimer SUNAT (Proforma Perú).
- **B5** — Modalidad de emisión electrónica OSE/SUNAT.
- **B7** — Certificaciones peruanas del giro químico-farmacéutico.
- **B8** — Catálogos farmacéuticos Perú.
- **B10** — Título "Proforma" vs "Factura Proforma" (Perú).
- **B11** — Nomenclatura de soles en letra.
- **B12** — Fuente oficial del tipo de cambio Perú.

## 6. Roles operativos

- **ESAC** — Ejecutivo de Servicio de Atención al Cliente. Tramita pedidos.
- **Gestor de Cobranza / Analista de Cuentas por Cobrar** — Opera Validar Cobro, Factura por Adelantado y Buzón de Cobros. *Denominación pendiente unificar.*
- **Coordinador de Tesorería** — Gestiona el campo Cobrador en el Catálogo de Clientes.
- **Regulatorios** — Carga y reemplaza documentos regulatorios (Licencia Sanitaria, Aviso de Responsable Sanitario) en el Catálogo de Clientes.
- **Finanzas (área)** — Gestión externa de cancelaciones fiscales y devoluciones. Fuera de R16.

## 7. Empresas y sistemas

- **PROQUIFA México** — Grupo de 4 empresas emisoras: **Golocaer S.A. de C.V.**, **Mungen S.A. de C.V.**, **Proquifa S.A. de C.V.**, **Proveedora Quimico Farmaceutica S.A. de C.V.**
- **PROQUIFA Perú** — Empresa emisora única: **Golocaer S.A.C.** (RUC 20612772941).
- **ProquifaDotNet** — Sistema de Venta Interna existente (.NET Framework 4.8). Antes conocido como PQF2.
- **PQF2** — Denominación heredada de ProquifaDotNet.
- **Legacy** — Sistema legado que recibe transferencias (ETL / LegacySync) desde ProquifaDotNet. Perú NO transfiere a Legacy.
- **DocumentBuilder** — Servicio de generación de PDFs (Proforma, Factura, NC, CDP, Confirmación de Pedido).
- **Brevo** — Servicio de envío de notificaciones por correo.
- **Mailbot** — Componente que clasifica correos entrantes en el Buzón de Cobros.

## 8. Arquitectura Back — Soluciones nuevas

- **ProquifaDotNet.Finanzas** — Solución nueva en **.NET Core 10**. Centraliza Cobros, Facturación, Proforma, Notas de Crédito y CDP. Consume Timbrado y Venta Interna vía API.
- **ProquifaDotNet.Timbrado** — Solución nueva en **.NET Core 10**. Gestiona timbrado fiscal a través de SAP TurboPac (México).
- **ProquifaDotNet.LegacySync** — Solución nueva en **.NET Core 10**. Reemplaza los paquetes SSIS de PCconnect. Transferencia ProquifaDotNet → Legacy. Solo México.
- **StampingController** — Controlador único con 4 endpoints en Timbrado: `POST /api/v1/stamp/invoice`, `/api/v1/stamp/payment-complement`, `/api/v1/stamp/credit-note`, `/api/v1/stamp/cancel`.
- **StampingService** — Servicio interno compartido en Timbrado que orquesta el timbrado.
- **SapStampingClient** — Cliente HTTP hacia SAP TurboPac.
- **IApiCallerStamping** — Contrato de Finanzas para consumir Timbrado (`StampInvoiceAsync`, `StampPaymentComplementAsync`, `StampCreditNoteAsync`, `CancelAsync`).
- **CQRS** — Command Query Responsibility Segregation. Patrón usado en Finanzas y Timbrado.
- **QueryInfo** — Objeto genérico de búsqueda usado en el arquetipo.
- **PedidoCreditoSyncJob** — Job de Hangfire en LegacySync que transfiere pedidos crédito a Legacy.
- **PedidoCreditoPayloadBuilder** — Constructor del payload de transferencia.
- **SyncControl** — Tabla de control de transferencias en `PConnectProquifaDotNet`.
- **Hangfire** — Framework de jobs en segundo plano usado en LegacySync.

## 9. Base de datos

- **BD ProquifaDotNet** — Base principal del sistema (servidor `RYNL010`). Cadena: `Data Source=RYNL010;Initial Catalog=ProquifaDotNet;...`.
- **BD ProquifaDotNetTimbrado** — Base de la solución Timbrado. Tablas: `AppSetting`, `StampingLog`.
- **BD PConnectProquifaDotNet** — Base de control de LegacySync.
- **CFDIGenerada** — Tabla en ProquifaDotNet donde se persisten los CFDIs timbrados México.
- **StampingLog** — Tabla de auditoría en ProquifaDotNetTimbrado. Un registro por solicitud (Action `Stamp`/`Cancel`, Request/Response al PAC, DurationMs).
- **AppSetting** — Tabla en ProquifaDotNetTimbrado con configuración runtime (endpoints SAP, timeouts, Name/Value JSON).
- **EmpresaFolio** — Tabla del foliador. Gestionada por Finanzas con UPDLOCK. Series: por empresa México (facturas), `P` (CP), `P2` (NC), `F001 GOLPERU` (**fuera de R16**).
- **tpProformaPedido / tpProformaPartidaPedido** — Tablas de Proforma. Pasaron al Scaffold de Finanzas (jul 8 2026).
- **tpPedido / ClienteCartera** — Tablas con Controller/BO propio, consumidas por Finanzas vía API (no Scaffold).
- **Scaffold EF Core** — Generación de entidades desde SQL Server con `dotnet ef dbcontext scaffold` (Finanzas y Timbrado).
- **DDL / DML** — Data Definition Language / Data Manipulation Language.

## 10. Infraestructura y transversales

- **IdentityServer** — Autenticación/autorización centralizada.
- **MinIO** — Almacenamiento de documentos (Licencias Sanitarias, PDFs, XML).
- **RabbitMQ** — Broker de mensajería para procesos asíncronos (Timbrado, notificaciones).
- **Serilog** — Framework de logs con contexto (usuario, módulo, operación).
- **SSIS** — SQL Server Integration Services. Herramienta de ETL usada por el antiguo PCconnect (reemplazado por LegacySync).
- **ETL** — Extract-Transform-Load. Proceso de transferencia ProquifaDotNet → Legacy.
- **BitacoraCambios (ApiCallerBitacoraCambios)** — Aplicativo Nuevo para registrar histórico de cambios (Código Validador, etc.).

## 11. Flujo Validar Cobro (wizard)

- **Paso 1** — Captura del Cobro. Se conserva en Perú.
- **Paso 2** — Asociación cobro ↔ documento (Proforma / Factura / NC). En Perú se simplifica (sin NCs).
- **Paso 3** — Facturación y Envío. En México genera factura/anticipo/complemento y timbra. En Perú se reduce a envío de Confirmación de Pedido.
- **Auto-guardado** — Persistencia automática del avance del wizard.
- **Modal Inconsistencia de Pago** — Modal con catálogo de tipos de inconsistencia (Tesorería). En Paso 2 se añaden "Pago Incompleto Vencido" y "Pago Insuficiente".
- **Tolerancia 100 MXN** — Umbral en México para cerrar asociación con saldo pendiente sin bloquear.
- **Saldo a favor** — Excedente de cobros; se refleja en Estado de Cuenta y queda disponible.

## 12. Catálogos e identificadores clave

- **c_FormaPago** — Catálogo SAT de formas de pago.
- **c_UsoCFDI** — Catálogo SAT de usos del CFDI.
- **c_MotivoCancelacion** — Catálogo SAT de motivos de cancelación.
- **Catálogo Interno PROQUIFA** — Catálogo interno de formas de pago Perú (SUNAT no lo exige).
- **CCI** — Código de Cuenta Interbancario. Perú, 20 dígitos.
- **CLABE** — Clave Bancaria Estandarizada. México.
- **Código Validador** — Segmento del código bancario del cliente (referencia Banamex).
- **Referencia Banamex** — Formato: 3 letras nombre + 4 chars clave + código banco + P/D + CodValidador.

## 13. Estándares transversales

- **JSON error contract** — Formato estándar `{ errorCode, message, details }`.
- **Reintentos configurables** — Política manejada en cada punto de generación (Factura, FpA, NC, CDP). Timbrado es de un solo intento.
- **Conservación 5 años** — Obligación de conservar XML CFDI y CPE (Art. 30 CFF México; R.S. 117-2017/SUNAT Perú).
- **Idempotencia** — Propiedad requerida para reintentos seguros de timbrado.
- **UPDLOCK** — Hint de bloqueo SQL Server usado para consumir folios sin colisiones.

## 14. Acciones del análisis Quitar Perú

- **SALE** — El requisito se cancela por completo (RE-020, RE-022, RE-033, RE-035).
- **SIMPLIFICA** — El requisito se conserva pero se retira lógica peruana fiscal (RE-027, RE-029).
- **CONSERVA** — Sin cambios; se limpian referencias cruzadas a los "SALE" (RE-004, RE-005, RE-017, RE-023, RE-025, RE-028, RE-030, RE-032, RE-034).
- **AJUSTA** — Cambio menor: bloquear opción por región o filtrar (RE-015, RE-018).

---

_Última actualización: 17 de julio de 2026._
