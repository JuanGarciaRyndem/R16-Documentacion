# Descripción de los Módulos `Logic.Pqf.Catalogos` y `Logic.Pqf.Logistica`

## Logic.Pqf.Catalogos

Es la **capa de datos maestros y configuración** del sistema. Gestiona toda la información de referencia que los demás módulos consumen.

### Responsabilidades principales

#### 🧪 Productos
- Alta, edición e importación masiva (vía USP/EP).
- Configuración por tipo: reactivo, servicio, dispositivo médico, capacitación, labware y publicación.
- Gestión de clasificaciones, familias, marcas, lotes, fletes y productos alternativos/suplementarios.

#### 🏭 Proveedores
- CRUD completo con configuración de precios, tiempos de entrega, regalías, familias, industrias, comisiones, campañas y carteras de responsable de contenido.

#### 👥 Clientes
- CRUD, contratos por marca, configuración de precios aplicados, datos de facturación, horarios de atención, direcciones, tipo de cambio (TCDOF), cartera y relaciones con contactos/empresas.

#### 💲 Precios y Cálculos
- Motor de configuración de precio lista, precios contrato, utilidad por categoría de proveedor y cálculo de tiempo de entrega logístico.

#### 📋 Catálogos Generales
- Monedas, condiciones de pago, rutas de entrega, destinos, dominios de correo, subtipo de producto, entre otros.

#### 🌍 Direcciones
- Validación de código postal.
- Calendarios de días festivos por país (México, EUA, China, Japón, Europa, etc.) para cálculo de tiempos de entrega.

#### 📁 Archivos
- Subida/descarga a MinIO.
- Generación de PDFs (cotizaciones).
- Exportación CSV/XLSX.

#### ⚙️ Servicios del Sistema
- Bitácora CRUD, monitor de eventos, procesos de sistema y días festivos.

---

## Logic.Pqf.Logistica

Es el **núcleo operativo del flujo de ventas y compras**. Implementa un proceso por etapas **(L01–L12)** que representa el ciclo completo de un pedido, desde la cotización hasta la entrega.

### Etapas del Proceso

#### L01 – Cotización
- Creación y gestión de cotizaciones y partidas.
- Investigación de productos, envío de correos al cliente.
- Cierre de oferta, tasas de conversión y promesas de compra.

#### L02 – Ajustar Oferta
- Ajuste de precios, fletes express y condiciones de pago.
- Estrategias de cotización, autorización y rechazo de ajustes.

#### L03 – Promesa de Compra
- Pretramitación de la promesa, incidencias, fletes, seguimiento.
- Generación de OC sin referencia.

#### L04 – Pretramitar Pedido
- Validación de tramitabilidad, recálculo de partidas.
- Conversión de tipo de cambio, gestión de intramitables, OC no amparada.
- Fábrica de pedidos.

#### L05 – Tramitar Pedido
- Liberación del pedido, generación de proformas/CFDI.
- Addendas fiscales: Sanofi, Pfizer, Mavi, Asofarma, AMECE.
- Separación de pedido, carta de disponibilidad, reportes y notificaciones.

#### L06 – Orden de Compra
- Creación y gestión de OC, carga de facturas de proveedor.
- Back order, declarar arribos (packing list) y dashboard de compras.

#### L07 – Importaciones
- Planificar, registrar y confirmar despachos.
- Monitorear guías FEE, asistente de importaciones y control nacional/origen.

#### L08 – Inspección
- Inspección de partidas y piezas, almacenaje.
- Seguridad: control de visitantes y vehículos.

#### L09 – Embalar
- Colección, embalaje por prioridad, generación de etiquetas.
- Packing list final y reportes de factura.

#### L11 – MailBot
- Procesamiento automático de correos entrantes de clientes.
- Generación de cotizaciones, pretramitaciones o registro de pagos desde correo.
- Alta automática de clientes nuevos.

#### L12 – Vigencia Curaduria
- Atención de productos con vigencia de curación pendiente.

### Módulos Transversales

| Módulo | Descripción |
|---|---|
| PDFs | Generación de cotización, pedido, contrato y carta de uso |
| CFDIs | Generación y cancelación de comprobantes fiscales |
| Autorizaciones | Control de autorizaciones con código |
| Notificaciones | Envío de correos del sistema |
| Folios | Utilidades para generación de folios de pedido |