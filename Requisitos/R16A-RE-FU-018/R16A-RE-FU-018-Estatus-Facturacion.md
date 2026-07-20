# Catálogo de Estatus de Facturación — Factura por Adelantado (FAA)

| Campo | Valor |
|---|---|
| **Módulo** | Factura por Adelantado |
| **Soluciones involucradas** | ProquifaDotNet · ProquifaDotNet.Finanzas · ProquifaDotNet.Timbrado |
| **Tabla catálogo** | `catEstatusFactura` |
| **Tabla principal** | `fccFactura` (FK → `IdCatEstatusFactura`) |
| **Fecha** | 2026-07-20 |

---

## Tabla catálogo — `catEstatusFactura`

```sql
CREATE TABLE dbo.catEstatusFactura (
    IdCatEstatusFactura  uniqueidentifier NOT NULL CONSTRAINT PK_catEstatusFactura PRIMARY KEY,
    Clave                nvarchar(50)     NOT NULL CONSTRAINT UQ_catEstatusFactura_Clave UNIQUE,
    Nombre               nvarchar(100)    NOT NULL,
    Descripcion          nvarchar(255)    NOT NULL,
    Activo               bit              NOT NULL CONSTRAINT DF_catEstatusFactura_Activo DEFAULT 1
);
```

### Datos iniciales

```sql
INSERT INTO dbo.catEstatusFactura (IdCatEstatusFactura, Clave, Nombre, Descripcion) VALUES
('A1B2C3D4-0001-0000-0000-000000000001', 'PendienteEmision',  'Pendiente de Emisión',   'El pendiente FAA fue generado al tramitar. Aún no se ha iniciado el timbrado.'),
('A1B2C3D4-0002-0000-0000-000000000002', 'EnProcesoTimbrado', 'En proceso de timbrado', 'Solicitud enviada al PAC. Se están ejecutando los reintentos sincrónicos.'),
('A1B2C3D4-0003-0000-0000-000000000003', 'Timbrada',          'Timbrada',               'El PAC retornó UUID y XML firmado. La factura aún no ha sido enviada al cliente.'),
('A1B2C3D4-0004-0000-0000-000000000004', 'ErrorTimbrado',     'Error al timbrar',       'El Worker agotó todos los reintentos. Se notificó al equipo de soporte vía Brevo.'),
('A1B2C3D4-0005-0000-0000-000000000005', 'Enviada',           'Enviada',                'La factura PPD timbrada fue enviada exitosamente al cliente.');
```

> Los GUIDs son fijos (seed data) para que las referencias desde código sean estables entre ambientes.

---

## FK en `fccFactura`

```sql
ALTER TABLE dbo.fccFactura
    ADD IdCatEstatusFactura uniqueidentifier NOT NULL
        CONSTRAINT DF_fccFactura_IdCatEstatusFactura
            DEFAULT 'A1B2C3D4-0001-0000-0000-000000000001',  -- 'PendienteEmision'
        CONSTRAINT FK_fccFactura_catEstatusFactura
            FOREIGN KEY (IdCatEstatusFactura)
            REFERENCES dbo.catEstatusFactura (IdCatEstatusFactura);
```

> Al insertar el pendiente FAA en la tramitación del pedido (RE-FU-012 / RE-FU-015), el registro nace con la clave `PendienteEmision` por `DEFAULT`.

---

## Catálogo de estatus

### 1 — Pendiente de Emisión

| Campo                 | Valor                                                                         |
| --------------------- | ----------------------------------------------------------------------------- |
| **Clave**             | `PendienteEmision`                                                            |
| **Quién asigna**      | `DEFAULT` al insertar `fccFactura` en la tramitación del pedido               |
| **Descripción**       | El pendiente FAA fue generado al tramitar. Aún no se ha iniciado el timbrado. |
| **Origen**            | RE-FU-012 (Crédito con FAA) · RE-FU-015 (Prepago con FAA)                     |
| **Acción disponible** | El gestor de cobranza puede seleccionar la factura e iniciar el timbrado      |

---

### 2 — En proceso de timbrado

| Campo | Valor |
|---|---|
| **Clave** | `EnProcesoTimbrado` |
| **Quién asigna** | `TimbradoService` al crear el registro en tabla `CFDI` y enviar al PAC |
| **Descripción** | Solicitud enviada al PAC (SAP/TurboPac). Reintentos sincrónicos activos (hasta 3, backoff exponencial via Polly). |
| **Origen** | RE-FU-019 al llamar `POST /api/timbrado/timbrar` en ProquifaDotNet.Timbrado |
| **Acción disponible** | Ninguna — el sistema está procesando |

---

### 3 — Timbrada

| Campo | Valor |
|---|---|
| **Clave** | `Timbrada` |
| **Quién asigna** | `TimbradoService` al recibir respuesta exitosa del PAC (flujo sincrónico o Worker asíncrono) |
| **Descripción** | El PAC retornó UUID, Serie y Folio. El XML firmado fue almacenado en Minio (bucket `timbrado`). La factura aún no ha sido enviada al cliente. |
| **Origen** | RE-FU-018/019 — flujo sincrónico o Worker RabbitMQ exitoso |
| **Acción disponible** | El gestor puede enviar la factura al cliente |

---

### 4 — Error al timbrar

| Campo | Valor |
|---|---|
| **Clave** | `ErrorTimbrado` |
| **Quién asigna** | `TimbradoWorker` al agotar todos los reintentos asíncronos en la cola RabbitMQ |
| **Descripción** | El PAC falló en los reintentos sincrónicos y el Worker agotó todos sus reintentos. Se notifica al equipo de soporte vía Brevo. |
| **Origen** | RE-FU-018 — Worker al agotar reintentos |
| **Acción disponible** | Reintento manual o escalación a soporte técnico |

---

### 5 — Enviada

| Campo | Valor |
|---|---|
| **Clave** | `Enviada` |
| **Quién asigna** | RE-FU-019 al completar el envío al cliente |
| **Descripción** | La factura PPD timbrada fue enviada exitosamente al cliente. Sale del conteo de facturas pendientes en el listado FAA. |
| **Origen** | RE-FU-019 |
| **Acción disponible** | Ninguna — flujo completado |

---

## Diagrama de transición de estatus

```
[Tramitar Pedido — RE-FU-012 / RE-FU-015]
       │
       │ INSERT fccFactura → IdCatEstatusFactura = 1 (DEFAULT)
       ▼
┌──────────────────────┐
│  1. Pendiente de     │
│     Emisión          │
└──────────┬───────────┘
           │ Gestor inicia timbrado (RE-FU-019)
           │ UPDATE IdCatEstatusFactura = 2
           ▼
┌──────────────────────┐
│  2. En proceso de    │
│     timbrado         │
└──────┬───────────────┘
       │
  ┌────┴──────────────────────────┐
  │ PAC exitoso                   │ PAC falla → Worker RabbitMQ
  │ UPDATE = 3                    │
  ▼                               │
┌──────────┐              ┌───────┴──────────┐
│ 3.       │              │ Worker reintenta  │
│ Timbrada │              └───────┬──────────┘
└────┬─────┘                      │
     │                    ┌───────┴──────────┐
     │              Exitoso│                  │Agota reintentos
     │              UPDATE=3                  │UPDATE = 4
     │                    │                  ▼
     │                    ▼         ┌─────────────────┐
     │              ┌──────────┐    │  4. Error al    │
     │              │3.Timbrada│    │     timbrar     │
     │              └────┬─────┘    └─────────────────┘
     │                   │
     └─────┬─────────────┘
           │ Gestor envía factura (RE-FU-019)
           │ UPDATE IdCatEstatusFactura = 5
           ▼
┌──────────────────────┐
│  5. Enviada          │
└──────────────────────┘
```

---

## Filtro del listado FAA (RE-FU-018)

El listado muestra únicamente facturas activas no finalizadas:

```sql
WHERE fcc.FacturaPorAdelantado = 1
  AND fcc.Tramitado            = 1
  AND fcc.Activo               = 1
  AND ef.Clave                != 'Enviada'
```

Al pasar a estatus `Enviada`, el registro sale automáticamente del listado.

---

## Diccionario de datos — `catEstatusFactura`

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `IdCatEstatusFactura` | `uniqueidentifier` | NO | PK — GUID fijo de seed data |
| `Clave` | `nvarchar(50)` | NO | Clave única para consumo desde código (`EstatusFactura.Clave`) |
| `Nombre` | `nvarchar(100)` | NO | Etiqueta de presentación en pantalla |
| `Descripcion` | `nvarchar(255)` | NO | Descripción funcional del estatus |
| `Activo` | `bit` | NO | Permite desactivar estatus sin eliminarlos |

---

## Constantes en código

Las claves del catálogo se usan directamente como constantes de cadena, alineadas con el campo `Clave` de la BD:

```csharp
// ProquifaDotNet.Finanzas — Domain/Constants/EstatusFactura.cs
public static class EstatusFactura
{
    public const string PendienteEmision  = "PendienteEmision";
    public const string EnProcesoTimbrado = "EnProcesoTimbrado";
    public const string Timbrada          = "Timbrada";
    public const string ErrorTimbrado     = "ErrorTimbrado";
    public const string Enviada           = "Enviada";
}
```

Uso en servicio al actualizar la transición:

```csharp
factura.IdCatEstatusFactura = await _catalogoRepo
    .ObtenerIdPorClave<catEstatusFactura>(EstatusFactura.EnProcesoTimbrado);
```

---

## Pendientes abiertos

| ID | Descripción | Requisito |
|---|---|---|
| PA-2 (RE-FU-012) | Confirmar que todos los módulos consumidores lean `IdCatEstatusFactura` en lugar de calcular el estatus via condiciones en vistas | RE-FU-012 |
| OBS-032/033 | FAA Perú excluida del listado hasta RE-FU-005 Brecha 5 | RE-FU-018 |
| RE-FU-019 | Definir el contrato de callback Timbrado → Finanzas para actualizar el estatus al completar el timbrado asíncrono | RE-FU-018/019 |
