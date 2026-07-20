# Catálogo de Estatus de Facturación — `catFacturaEstado`

| Campo | Valor |
|---|---|
| **Módulo** | Factura por Adelantado |
| **Soluciones involucradas** | ProquifaDotNet · ProquifaDotNet.Finanzas · ProquifaDotNet.Timbrado |
| **Tabla catálogo** | `catFacturaEstado` (definida en RE-FU-015 BD v2.1) |
| **Tabla principal** | `fccFactura` (FK → `IdCatFacturaEstado`) |
| **Aplica a** | FAA y Factura Normal (`fccFactura` es tabla única, diferenciada por `EsFacturaPorAdelantado`) |
| **Fecha** | 2026-07-20 |

---

## DDL (RE-FU-015 — no duplicar)

```sql
CREATE TABLE [dbo].[catFacturaEstado](
    [IdCatFacturaEstado]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catFacturaEstado_Id]       DEFAULT (NEWID()),
    [Clave]               varchar(30)      NOT NULL,
    [Descripcion]         nvarchar(150)    NOT NULL,
    [Orden]               int              NOT NULL,
    [EsTerminal]          bit              NOT NULL
        CONSTRAINT [DF_catFacturaEstado_Terminal] DEFAULT (0),
    [Activo]              bit              NOT NULL
        CONSTRAINT [DF_catFacturaEstado_Activo]   DEFAULT (1),
    [FechaRegistro]       datetime2(7)     NOT NULL
        CONSTRAINT [DF_catFacturaEstado_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catFacturaEstado]       PRIMARY KEY CLUSTERED ([IdCatFacturaEstado]),
    CONSTRAINT [UQ_catFacturaEstado_Clave] UNIQUE ([Clave])
);
```

### Seed data

```sql
INSERT INTO [dbo].[catFacturaEstado] (Clave, Descripcion, Orden, EsTerminal) VALUES
('POR_GENERAR',    N'Factura creada, pendiente de timbrado ante PAC/SUNAT',                                1, 0),
('ERROR_TIMBRADO', N'El PAC/SUNAT rechazó el timbrado; requiere corrección y reintento (Finanzas)',        2, 0),
('GENERADA',       N'Timbrada exitosamente (CFDI/CPE vigente); pendiente de envío al cliente',             3, 0),
('ENVIADA',        N'Enviada al cliente con PDF + XML adjuntos',                                           4, 0),
('PAGADA_PARCIAL', N'Con cobros aplicados parcialmente; saldo pendiente (PPD con complementos parciales)', 5, 0),
('PAGADA',         N'Cobro asociado y aplicado en su totalidad (Validar Cobro)',                           6, 1),
('CANCELADA',      N'Cancelada ante SAT/SUNAT (CFDICancelacion / NC según normativa)',                     7, 1);
```

---

## Catálogo de estatus

### POR_GENERAR

| Campo | Valor |
|---|---|
| **Orden** | 1 |
| **Terminal** | No |
| **Quién asigna** | `DEFAULT` al insertar `fccFactura` en la tramitación del pedido |
| **Descripción** | Factura creada, pendiente de timbrado ante PAC/SUNAT. El pendiente FAA fue generado al tramitar pero aún no se ha iniciado el timbrado. |
| **Origen** | RE-FU-012 (Crédito con FAA) · RE-FU-015 (Prepago con FAA) |
| **Acción disponible** | El gestor de cobranza puede seleccionar la factura e iniciar el timbrado |

---

### ERROR_TIMBRADO

| Campo | Valor |
|---|---|
| **Orden** | 2 |
| **Terminal** | No |
| **Quién asigna** | `TimbradoService` / `TimbradoWorker` al agotar reintentos ante el PAC |
| **Descripción** | El PAC/SUNAT rechazó el timbrado. Requiere corrección y reintento desde Finanzas. El Worker RabbitMQ notifica al equipo de soporte vía Brevo. |
| **Origen** | RE-FU-018 — Worker al agotar reintentos |
| **Acción disponible** | Reintento manual desde el módulo Finanzas o escalación a soporte |

---

### GENERADA

| Campo | Valor |
|---|---|
| **Orden** | 3 |
| **Terminal** | No |
| **Quién asigna** | `TimbradoService` al recibir respuesta exitosa del PAC (flujo sincrónico o Worker asíncrono) |
| **Descripción** | Timbrada exitosamente. CFDI/CPE vigente. El XML firmado fue almacenado en Minio (bucket `timbrado`) y `fccFactura.IdCFDIGenerada` fue actualizado. Pendiente de envío al cliente. |
| **Origen** | RE-FU-018/019 — flujo sincrónico o Worker RabbitMQ exitoso |
| **Acción disponible** | El gestor puede enviar la factura al cliente |

---

### ENVIADA

| Campo | Valor |
|---|---|
| **Orden** | 4 |
| **Terminal** | No |
| **Quién asigna** | RE-FU-019 al completar el envío al cliente (`Enviada = 1`, `FechaEnvio = SYSUTCDATETIME()`) |
| **Descripción** | Enviada al cliente con PDF + XML adjuntos. Sale del conteo de facturas pendientes en el listado FAA. |
| **Origen** | RE-FU-019 |
| **Acción disponible** | Aplican cobros (Validar Cobro) o cancelación |

---

### PAGADA_PARCIAL

| Campo | Valor |
|---|---|
| **Orden** | 5 |
| **Terminal** | No |
| **Quién asigna** | Módulo Validar Cobro (RE-FU-026/027/028/029) al aplicar complementos de pago parciales |
| **Descripción** | Con cobros aplicados parcialmente. Saldo pendiente (PPD con complementos parciales). |
| **Origen** | RE-FU-026/027/028/029 |
| **Acción disponible** | Cobros adicionales hasta completar el total, o cancelación |

---

### PAGADA

| Campo | Valor |
|---|---|
| **Orden** | 6 |
| **Terminal** | **Sí** |
| **Quién asigna** | Módulo Validar Cobro (RE-FU-026/027/028/029) al aplicar el cobro total |
| **Descripción** | Cobro asociado y aplicado en su totalidad. Sin transiciones posteriores. |
| **Origen** | RE-FU-026/027/028/029 |
| **Acción disponible** | Ninguna — flujo completado |

---

### CANCELADA

| Campo | Valor |
|---|---|
| **Orden** | 7 |
| **Terminal** | **Sí** |
| **Quién asigna** | Flujo de cancelación (RE-FU-032) vía `POST /api/v1/stamp/cancel` |
| **Descripción** | Cancelada ante SAT/SUNAT (CFDI de cancelación o Nota de Crédito según normativa). Sin transiciones posteriores. |
| **Origen** | RE-FU-032 |
| **Acción disponible** | Ninguna — flujo completado |

---

## Diagrama de transición de estatus

```
[Tramitar Pedido — RE-FU-012 / RE-FU-015]
       │
       │ INSERT fccFactura → IdCatFacturaEstado = POR_GENERAR (DEFAULT)
       ▼
┌─────────────────┐
│  POR_GENERAR    │
└────────┬────────┘
         │ Gestor inicia timbrado (RE-FU-019)
         │ PAC responde
    ┌────┴──────────────────┐
    │ Exitoso               │ Fallo (reintentos agotados)
    ▼                       ▼
┌──────────┐        ┌───────────────┐
│ GENERADA │        │ ERROR_TIMBRADO│◄── reintento desde Finanzas
└────┬─────┘        └───────┬───────┘
     │                      │ Reintento exitoso
     │              ┌───────┘
     │              ▼
     │          ┌──────────┐
     │          │ GENERADA │
     │          └────┬─────┘
     └──────┬────────┘
            │ Gestor envía factura (RE-FU-019)
            │ Enviada=1, FechaEnvio=SYSUTCDATETIME()
            ▼
┌─────────────────┐
│    ENVIADA      │
└────────┬────────┘
         │
    ┌────┴──────────────────────────┐
    │ Cobro parcial                 │ Cobro total        │ Cancelación
    ▼                               ▼                    ▼
┌────────────────┐          ┌──────────┐        ┌────────────┐
│ PAGADA_PARCIAL │──cobro──►│  PAGADA  │        │ CANCELADA  │
│                │  total   │(terminal)│        │ (terminal) │
│                │          └──────────┘        └────────────┘
└────────┬───────┘
         │ Cancelación
         ▼
    ┌────────────┐
    │ CANCELADA  │
    │ (terminal) │
    └────────────┘
```

---

## Transiciones válidas

| Desde | Hacia | Quién ejecuta |
|---|---|---|
| `POR_GENERAR` | `GENERADA` | TimbradoService — PAC exitoso |
| `POR_GENERAR` | `ERROR_TIMBRADO` | TimbradoWorker — reintentos agotados |
| `ERROR_TIMBRADO` | `GENERADA` | TimbradoService — reintento desde Finanzas |
| `GENERADA` | `ENVIADA` | RE-FU-019 — envío al cliente |
| `GENERADA` | `CANCELADA` | RE-FU-032 — cancelación fiscal |
| `ENVIADA` | `PAGADA_PARCIAL` | RE-FU-026/027/028/029 — cobro parcial |
| `ENVIADA` | `PAGADA` | RE-FU-026/027/028/029 — cobro total |
| `ENVIADA` | `CANCELADA` | RE-FU-032 — cancelación fiscal |
| `PAGADA_PARCIAL` | `PAGADA` | RE-FU-026/027/028/029 — cobro total |
| `PAGADA_PARCIAL` | `CANCELADA` | RE-FU-032 — cancelación fiscal |

> `PAGADA` y `CANCELADA` son estados terminales (`EsTerminal = 1`) — sin transiciones posteriores.

---

## Constantes en código

```csharp
// ProquifaDotNet.Finanzas — Domain/Constants/FacturaEstado.cs
public static class FacturaEstado
{
    public const string PorGenerar    = "POR_GENERAR";
    public const string ErrorTimbrado = "ERROR_TIMBRADO";
    public const string Generada      = "GENERADA";
    public const string Enviada       = "ENVIADA";
    public const string PagadaParcial = "PAGADA_PARCIAL";
    public const string Pagada        = "PAGADA";
    public const string Cancelada     = "CANCELADA";
}
```

Uso al actualizar la transición:

```csharp
factura.IdCatFacturaEstado = await _catalogoRepo
    .ObtenerIdPorClave<catFacturaEstado>(FacturaEstado.Generada);
```

---

## Filtro del listado FAA (RE-FU-018)

El listado muestra únicamente facturas activas no finalizadas. La `vfccFactura` ya expone `FacturaEstadoClave` (JOIN a `catFacturaEstado`):

```sql
WHERE fcc.EsFacturaPorAdelantado = 1
  AND fcc.Activo                 = 1
  AND ef.Clave NOT IN ('PAGADA', 'CANCELADA')
```

---

## Nota sobre `EstadoFAA` en `vfccFactura`

La vista `vfccFactura` mantiene el campo calculado `EstadoFAA` (compatibilidad con el consumo que tenía `vtpProformaAdelanto`) independientemente de `IdCatFacturaEstado`:

```sql
CASE
    WHEN f.IdCFDIGenerada IS NULL                              THEN 'PendienteGenerar'
    WHEN f.IdCFDIGenerada IS NOT NULL AND f.Enviada = 0        THEN 'PendienteEnviar'
    ELSE                                                            'Completada'
END AS EstadoFAA
```

`EstadoFAA` es el estado del **pendiente FAA** (3 valores, compatibilidad). `IdCatFacturaEstado` / `FacturaEstadoClave` es el estado del **ciclo de vida completo de la factura** (7 valores, fuente de verdad). Ambos conviven en la vista — no se sustituyen.
