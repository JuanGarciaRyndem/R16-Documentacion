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
