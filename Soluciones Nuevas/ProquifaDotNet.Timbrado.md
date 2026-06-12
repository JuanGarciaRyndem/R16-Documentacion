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