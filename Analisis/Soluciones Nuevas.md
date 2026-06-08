### Soluciones Nuevas
- Se van empezar a generar las soluciones a partir del requisito 016
	- Se debe de integrar toda la parte técnica que requiera cada solución
- Como vaya requiriendo en cada requisito se va agregando funcionalidad a la soluciones, no se hace todo al vuelo en cuanto a funcionalidad
## Para integrar los cobros y Documentos Fiscales se crearán las siguientes soluciones:
### Creación de solución **ProquifaDotNet.Finanzas**
**Descripción:** Se generará una solución denominada **ProquifaDotNet.Finanzas**, desarrollada en **.NET Core 10**, cuyo objetivo será integrar y centralizar los procesos relacionados con la gestión financiera de la organización.

**Alcance funcional:** La solución incluirá los siguientes módulos y funcionalidades:
- **Cobros**: validación de pagos y conciliación.
- **Facturación**: Generación de facturas, facturas por adelantado y validación regulatoria.
- **Proforma**: Diseño y emisión de documentos de proforma para México y Perú.
- **Notas de Crédito**: Administración de notas de crédito, incluyendo detalle y documentos para México y Perú.
- **Documentos financieros adicionales**: CDP (Comprobante de Pago) y otros documentos requeridos por normativa local.

**Relación con el sistema de Venta Interna:** 
La integración entre el sistema de **Venta Interna** (actualmente en .NET Framework 4.8) y el nuevo sistema de **Finanzas** se realizará mediante **llamadas entre APIs**, garantizando:
- Independencia tecnológica entre ambos sistemas.
- Comunicación segura y estandarizada para el intercambio de información (pedidos, cobros, facturas, notas de crédito).
- Escalabilidad para futuras integraciones con otros módulos o servicios externos.

**Objetivo principal:**
- Proveer un **API modular y escalable** que permita la integración con los sistemas actuales de ventas internas (basados en .NET Framework 4.8), manteniendo independencia tecnológica para nuevas funcionalidades financieras.
- Garantizar la **consistencia regulatoria** en México y Perú, con validaciones específicas por país.
- Facilitar la **automatización de procesos financieros** y la generación de documentos oficiales de manera transaccional y segura.
  
## 📂 Estructura de la solución **ProquifaDotNet.Finanzas**

### 1. Capas principales (basado en arquetipo existente)
- **Domain**
    - Entidades: Cobro, Factura, Proforma, Nota de Crédito, CDP.
    - Interfaces de repositorios.
    - Reglas de negocio.
- **Application**
    - Implementación de **CQRS** (Commands/Queries).
    - DTOs para transferencia de datos.
    - CRUD genérico y búsquedas con **QueryInfo**.
    - Validaciones de negocio.
- **Infrastructure**
    - Persistencia con EF Core / SQL Server.
    - Integración con **Minio** (almacenamiento de documentos).
    - Integración con **RabbitMQ** (timbrado fiscal).
    - Configuración de **IdentityServer** (autenticación/autorización).
    - Manejo de **Logs** (Serilog u otro).
    - Generación de Scaffold con tablas que use el aplicativo en ProquifaDotNet
    - Base de datos ProquifaDotNet(Facturas, Facturas por Adelantado, Cobros, Validaciones de Cobro, Proforma, Notas de Credito y Comprobante de Pago. Rabbit) 
- **API**
    - Endpoints RESTful para Cobros, Facturas, Proforma, Notas de Crédito.
    - Comunicación con el sistema de **Venta Interna** mediante **llamadas entre APIs**.
    - Definición de mensajes de error estandarizados.
        
- **Worker.Finanzas**
    - Procesos asíncronos para revisar colas de **RabbitMQ**.
    - Validación de eventos relacionados con la generación y timbrado de facturas.
    - Reintentos configurables en caso de fallos.
    - Notificación automática vía **Brevo** (plantilla, HTML o texto simple).
- **Testing**
    - Pruebas unitarias e integración.
    - Validación de comandos, queries y workers.

### 2. Integraciones clave
- **IdentityServer** → Autenticación/autorización centralizada.
- **Brevo** → Envío de notificaciones (plantillas, HTML, texto simple).
- **Minio** → Almacenamiento seguro de documentos financieros.
- **RabbitMQ** → Orquestación del timbrado fiscal y comunicación entre servicios.

### 3. Estándares transversales
- **Mensajes de error**: catálogo con códigos únicos y respuestas JSON (`errorCode`, `message`, `details`).
- **Logs**: centralización con Serilog, enriquecidos con contexto (usuario, módulo, operación).

### 📄 Requisito: Creación de solución **ProquifaDotNet.Timbrado**

**Descripción:** Se generará una solución denominada **ProquifaDotNet.Timbrado**, desarrollada en **.NET Core 10**, cuyo objetivo será gestionar el proceso de timbrado de documentos fiscales de manera independiente y modular, garantizando la correcta integración con el proveedor SAP y con la solución de Finanzas.

**Alcance funcional:** La solución incluirá las siguientes funcionalidades:
- Recepción de solicitudes de timbrado mediante **llamadas API por documento**
- Integración con el **Proveedor SAP** para la creación de documentos fiscales.
- Generación del **CFDI** correspondiente y almacenamiento de los **XML en base de datos** y en **Minio**.
- Retorno del CFDI a la solución de **Finanzas** para su uso en procesos posteriores.
- Procesamiento asíncrono mediante **RabbitMQ**, incluyendo reintentos en caso de fallos.
- Envío de notificaciones con **Brevo** (plantillas, HTML o texto simple).
- Autenticación y autorización con **IdentityServer**.
- Definición de mensajes de error estandarizados y manejo centralizado de logs.

**Relación con el sistema de Finanzas:** 
La integración entre la solución de **Finanzas** y la de **Timbrado** se realizará mediante **llamadas entre APIs**.
- Finanzas enviará solicitudes de timbrado por documento.
- Timbrado procesará la petición, generará el CFDI y lo devolverá a Finanzas.
- Se mantendrá independencia tecnológica, pero con comunicación segura y estandarizada.

**Objetivo principal:**
- Proveer un **API especializado en timbrado fiscal**, modular y escalable.
- Garantizar la **correcta generación y almacenamiento de CFDIs** con trazabilidad completa.
- Asegurar la **integración confiable con SAP** y con la solución de Finanzas.
- Facilitar la automatización del timbrado y la comunicación entre sistemas mediante **RabbitMQ** y notificaciones con **Brevo**.

## 📂 Estructura de la solución **ProquifaDotNet.Timbrado**

### 1. Capas principales

- **Domain**
    - Entidades: DocumentoFiscal, CFDI, TimbradoRequest, TimbradoResponse.
    - Interfaces de repositorios.
    - Reglas de negocio para validación de timbrado.
- **Application**
    - Implementación de **CQRS** (Commands/Queries).
    - DTOs para transferencia de datos entre Finanzas y Timbrado.
    - Orquestación de llamadas al proveedor **SAP** para creación de documentos fiscales.
- **Infrastructure**
    - Persistencia en BD (almacenamiento de XML de CFDI).
    - Integración con **SAP** (API por documento).
    - Integración con **RabbitMQ** para procesar colas de timbrado.
    - Configuración de **IdentityServer** para autenticación/autorización.
    - Manejo de **Logs** (Serilog u otro).
    - Generación de Scaffold con tablas que use el aplicativo en ProquifaDotNet
    - Base de datos ProquifaDotNet(Timbrado y CFDI)
    - Base de datos ProquifaDotNetTimbrado(AppSettings, Peticiones, Respuesta de PAC, Rabbit)
- **API**
    - Endpoints RESTful para recibir solicitudes de timbrado desde Finanzas.
    - Respuesta con CFDI generado y validado.
    - Definición de mensajes de error estandarizados.
- **Worker.Timbrado**
    - Procesos asíncronos para revisar colas de **RabbitMQ**.
    - Ejecución del timbrado fiscal en segundo plano.
    - Reintentos configurables en caso de fallos.
    - Notificación automática vía **Brevo** (plantilla, HTML o texto simple).
- **Testing**
    - Pruebas unitarias e integración.
    - Validación de comandos, queries y workers.

### 2. Flujo funcional
1. **Finanzas** solicita timbrado → llamada al **API Timbrado** por documento.
2. **Timbrado** invoca al **Proveedor SAP** para generar el documento fiscal.
3. Se genera el **CFDI** y se guarda el **XML en BD** y en **Minio**.
4. El **CFDI** se regresa a la solución de **Finanzas** vía API.
5. **Worker.Timbrado** gestiona colas de RabbitMQ para procesos pendientes o reintentos.
6. Se envían notificaciones con **Brevo** según configuración.
### 3. Estándares transversales

- **Mensajes de error**: catálogo con códigos únicos y respuestas JSON (`errorCode`, `message`, `details`).
- **Logs**: centralización con Serilog, enriquecidos con contexto (usuario, módulo, operación).
- **Seguridad**: autenticación/autorización con IdentityServer.