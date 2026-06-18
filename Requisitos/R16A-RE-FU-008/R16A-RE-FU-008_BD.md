# Diccionario de Datos — Buzón de Cobros

| Campo | Valor |
|---|---|
| **Requisito** | R16A-RE-FU-008 |
| **Base de Datos** | ProquifaDotNet |
| **Servidor** | RYNL010 |
| **Versión** | 1.0 |
| **Generado por** | GitHub Copilot in SSMS |
| **Alcance** | México y Perú con segregación por región |

---

## Resumen Ejecutivo

Módulo nuevo en PQF2 que recibe correos clasificados como **Cobro** por el Mailbot, los refleja en la bandeja del Gestor de Cobranza correspondiente y genera automáticamente un pendiente en Validar Cobro. Los correos se segregan por región (MEX/PER) y se filtran por el cobrador asignado al cliente.

> **⚠️ Hallazgo crítico** — La clasificación ‘Cobro’ no existe en `catClasificacionCorreoRecibido`. Existe una clasificación llamada ‘Pago’ (clave `pago`) con `AnalistaDeCuentasPorCobrar = 1`, que es el equivalente actual. Se requiere una **decisión de diseño** antes del desarrollo: renombrar ‘Pago’ a ‘Cobro’ (Opción A) o insertar una nueva clasificación ‘Cobro’ (Opción B).

---

## Modelo de Datos

```
CorreoRecibido  (correo entrante, IdRegion para segregación MEX/PER)
└── CorreoRecibidoCliente  (clasificación + cliente + usuario + estado)
        ├── FK IdCatClasificacionCorreoRecibido → catClasificacionCorreoRecibido
        │       └── FK IdCatProceso → catProceso
        ├── FK IdCliente → Cliente
        │       └── ClienteCarteraCliente → ClienteCartera.IdUsuarioCobrador (filtro bandeja)
        └── fccFolioPagoCliente  (pendiente generado automáticamente en Validar Cobro)
                └── fccPagoCliente  (datos del cobro capturados en Validar Cobro)

Ciclo de vida del pendiente:
  Aparece     → al clasificar correo como Cobro
  Cierre      → fccPagoCliente vinculado a proforma/factura (Activo = 0)
  Eliminación → fccPagoCliente marcado como inconsistencia (Activo = 0)
  Reclasif.   → UPDATE CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido
```

---

## Entidades Afectadas

| Objeto | Tipo | Estado | Descripción |
|---|---|---|---|
| `catClasificacionCorreoRecibido` | Catálogo | ✨ Existente + INSERT/UPDATE | Agregar o renombrar clasificación ‘Cobro’ con `AnalistaDeCuentasPorCobrar = 1` |
| `catProceso` | Catálogo | ✨ Existente + INSERT | Agregar proceso ‘Cobros’ vinculado a la nueva clasificación |
| `catCobroEstatus` | Catálogo | 🆕 NUEVO | Catálogo de estatus del ciclo de vida del cobro (BORRADOR → COMPLETADO) |
| `CorreoRecibido` | Tabla | ✅ Existente — sin cambios | Correo entrante con `IdRegion` para segregación MEX/PER |
| `CorreoRecibidoCliente` | Tabla | ✅ Existente — sin cambios | Clasificación + cliente + estado del correo |
| `CorreoRecibidoEstatus` | Tabla | ✅ Existente — sin cambios | Estado de lectura y procesado del correo |
| `fccFolioPagoCliente` | Tabla | ✅ Existente — sin cambios | Pendiente en Validar Cobro generado por correo clasificado |
| `fccPagoCliente` | Tabla | ✨ Existente + ALTER | Agrega FK `IdCatCobroEstatus` para rastrear el estatus del cobro |
| `ClienteCartera` | Tabla | ✅ Existente — sin cambios | `IdUsuarioCobrador` para filtrar bandeja del Gestor |
| `ClienteCarteraCliente` | Tabla | ✅ Existente — sin cambios | Relación cliente-cartera para filtro de visibilidad |
| `RegionConfiguracionMailBot` | Tabla | ✅ Existente — referencia | Configuración del Mailbot por región (MEX/PER) |
| `spActualizarBuzonPagoLegacy` | SP | ⚠️ Existente — evaluar | SP de sincronización Legacy para pagos/cobros |

---

## 1. catClasificacionCorreoRecibido (CAMBIO REQUERIDO)

**Propósito:** Catálogo de clasificaciones del Mailbot. Define qué rol ve cada tipo de correo.
**Cambio R16:** Agregar o renombrar clasificación ‘Cobro’ para el Buzón de Cobros.

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdCatClasificacionCorreoRecibido` | uniqueidentifier | 16 | NO | PK |
| `IdCatProceso` | uniqueidentifier | 16 | NO | FK — `catProceso` |
| `ClasificacionCorreoRecibido` | varchar | 180 | SÍ | Nombre visible de la clasificación |
| `Posicion` | int | 4 | NO | Orden de presentación |
| `ClasificacionDefault` | bit | 1 | NO | Clasificación por defecto del Mailbot |
| `EVI` | bit | 1 | NO | Visible para rol EVI |
| `EVE` | bit | 1 | NO | Visible para rol EVE |
| `ESAC` | bit | 1 | NO | Visible para rol ESAC |
| `AnalistaDeCuentasPorCobrar` | bit | 1 | NO | **Visible para Gestor de Cobranza** |
| `CoordinadorDeServicioAlCliente` | bit | 1 | NO | Visible para Coordinador SAC |
| `Clave` | varchar | 150 | NO | Clave interna usada por el Mailbot |
| `Activo` | bit | 1 | NO | Default: 1 |

**Registros actuales en BD:**

| Clasificación | Clave | AnalistaCxC | ESAC | Default | Activo |
|---|---|---|---|---|---|
| Pago | pago | ✅ **SÍ** | No | No | ✅ Sí |
| Pedido | pedido | No | ✅ Sí | No | ✅ Sí |
| Requisición | requisicion | No | ✅ Sí | No | ✅ Sí |
| Solicitud de información | solicituddeinformacion | No | ✅ Sí | No | ✅ Sí |
| Queja | queja | No | ✅ Sí | No | ✅ Sí |
| Sugerencia | sugerencia | No | ✅ Sí | No | ✅ Sí |
| Otros | otros | No | ✅ Sí | ✅ **Sí** | ✅ Sí |

> **⚠️ Hallazgo** — La clasificación ‘Pago’ (clave `pago`) ya tiene `AnalistaDeCuentasPorCobrar = 1`. Es el equivalente de ‘Cobro’ en la BD actual.
>
> **Decisión pendiente:** Renombrar ‘Pago’ a ‘Cobro’ (impacto en sistema existente) o insertar nuevo registro ‘Cobro’ y desactivar ‘Pago’.

**Opción A — Renombrar Pago a Cobro (recomendada si ‘Pago’ no se usa en otros módulos activos):**

```sql
UPDATE dbo.catClasificacionCorreoRecibido
SET ClasificacionCorreoRecibido = 'Cobro',
    Clave                       = 'cobro',
    FechaUltimaActualizacion    = GETDATE()
WHERE Clave = 'pago';
```

**Opción B — Insertar nueva clasificación Cobro (si ‘Pago’ se usa en otros contextos):**

```sql
-- Requiere primero insertar el proceso Cobros en catProceso
DECLARE @IdProcesoCobros UNIQUEIDENTIFIER = NEWID();

INSERT INTO dbo.catProceso (IdCatProceso, Proceso, FechaRegistro, FechaUltimaActualizacion, Activo, Clave)
VALUES (@IdProcesoCobros, 'Cobros', GETDATE(), GETDATE(), 1, 'cobros');

INSERT INTO dbo.catClasificacionCorreoRecibido (
    IdCatClasificacionCorreoRecibido, IdCatProceso, ClasificacionCorreoRecibido,
    Posicion, ClasificacionDefault, EVI, EVE, ESAC,
    AnalistaDeCuentasPorCobrar, CoordinadorDeServicioAlCliente,
    Clave, FechaRegistro, FechaUltimaActualizacion, Activo)
VALUES (
    NEWID(), @IdProcesoCobros, 'Cobro',
    8, 0, 0, 0, 0,
    1, 0,
    'cobro', GETDATE(), GETDATE(), 1);
```

---

## 2. catProceso (CAMBIO REQUERIDO)

**Propósito:** Agrupa clasificaciones por proceso de negocio.
**Cambio R16:** Insertar proceso ‘Cobros’ para la nueva clasificación.

| Columna | Tipo | Descripción |
|---|---|---|
| `IdCatProceso` | uniqueidentifier | PK |
| `Proceso` | varchar | Nombre del proceso |
| `Clave` | varchar | Clave interna |
| `Activo` | bit | Default: 1 |

**Registros actuales en BD:**

| Proceso | Clave | Activo |
|---|---|---|
| Facturación | facturacion | ✅ Sí |
| Otros | otros | ✅ Sí |
| Pedido | pedido | ✅ Sí |
| Pretramitación de pedido | pretramitaciondepedido | ✅ Sí |
| Queja | queja | ✅ Sí |
| Requisición | requisicion | ✅ Sí |
| Solicitud de información | solicituddeinformacion | ✅ Sí |
| Sugerencia | sugerencia | ✅ Sí |
| **Cobros** | **cobros** | ✨ **NUEVO R16** |

---

## 3. CorreoRecibido (Existente — sin cambios)

**Propósito:** Almacena cada correo entrante procesado por el Mailbot.
**Campo crítico:** `IdRegion` — segrega correos por región (MEX/PER).

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `IdCorreoRecibido` | uniqueidentifier | NO | PK |
| `Asunto` | varchar(350) | SÍ | Asunto del correo |
| `CorreoEmisor` | varchar(180) | NO | Remitente |
| `CorreosReceptores` | varchar(800) | SÍ | Destinatarios |
| `FechaRecepcion` | datetime | NO | Fecha de recepción |
| `IdRegion` | uniqueidentifier | NO | **FK — Región (MEX/PER): segregación de correos por región** |
| `Activo` | bit | NO | Default: 1 |

---

## 4. CorreoRecibidoCliente (Existente — sin cambios)

**Propósito:** Vincula el correo con el cliente identificado y su clasificación. Es la **tabla central del Buzón** — controla clasificación, estado y asignación.

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `IdCorreoRecibidoCliente` | uniqueidentifier | NO | PK |
| `IdCatClasificacionCorreoRecibido` | uniqueidentifier | NO | **FK — `catClasificacionCorreoRecibido` (Cobro en R16)** |
| `IdCorreoRecibido` | uniqueidentifier | NO | FK — `CorreoRecibido` |
| `IdCliente` | uniqueidentifier | SÍ | FK — `Cliente` (nullable: cliente no identificado) |
| `IdContactoCliente` | uniqueidentifier | SÍ | FK — `ContactoCliente` |
| `IdUsuario` | uniqueidentifier | SÍ | FK — Usuario asignado |
| `IdContacto` | uniqueidentifier | SÍ | FK — Contacto |
| `Leido` | bit | NO | Indicador de lectura |
| `Procesado` | bit | NO | Indicador de procesado |
| `Activo` | bit | NO | Default: 1 |

> **Reclasificación:** UPDATE `IdCatClasificacionCorreoRecibido` al buzón destino seleccionado.

---

## 5. fccFolioPagoCliente (Existente — Pendiente en Validar Cobro)

**Propósito:** Pendiente en Validar Cobro generado automáticamente al clasificar como Cobro.

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `IdFCCFolioPagoCliente` | uniqueidentifier | NO | PK |
| `IdCorreoRecibidoCliente` | uniqueidentifier | NO | FK — `CorreoRecibidoCliente` (origen del cobro) |
| `IdArchivo` | uniqueidentifier | SÍ | Archivo adjunto del comprobante |
| `Folio` | varchar(80) | SÍ | Folio del cobro |
| `Consecutivo` | int | SÍ | Consecutivo del folio |
| `FechaRecepcion` | datetime | NO | Fecha de recepción del correo |
| `Stp` | bit | NO | Indica si es cobro vía STP |
| `SubtotalMailBot` | decimal | SÍ | Subtotal pre-extraído por Mailbot |
| `IvaMailBot` | decimal | SÍ | IVA pre-extraído por Mailbot |
| `TotalMailBot` | decimal | SÍ | Total pre-extraído por Mailbot |
| `MxnMailBot` | bit | SÍ | Moneda MXN detectada por Mailbot |
| `UsdMailBot` | bit | SÍ | Moneda USD detectada por Mailbot |
| `Activo` | bit | NO | Default: 1 |

> Los campos `SubtotalMailBot`, `IvaMailBot`, `TotalMailBot`, `MxnMailBot` y `UsdMailBot` indican que el Mailbot ya tiene capacidad de pre-extraer datos del comprobante. Esto soporta la **propuesta evolutiva de IA** para pre-cargar datos en Validar Cobro (ver Notas de Implementación del requisito).

---

## 6. fccPagoCliente (Existente — datos del cobro en Validar Cobro)

**Propósito:** Datos completos del cobro capturados en Validar Cobro.
**Ciclo de vida del buzón:** `Activo = 0` cuando hay inconsistencia → elimina el pendiente del Buzón.

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `IdFCCPagoCliente` | uniqueidentifier | NO | PK |
| `IdFCCFolioPagoCliente` | uniqueidentifier | SÍ | FK — pendiente del buzón origen |
| `IdCliente` | uniqueidentifier | NO | FK — Cliente del cobro |
| `IdEmpresa` | uniqueidentifier | NO | FK — Empresa receptora |
| `Monto` | decimal | NO | Monto del cobro |
| `FechaPago` | datetime | SÍ | Fecha del pago |
| `IdCatMedioDePago` | uniqueidentifier | SÍ | FK — Forma de pago |
| `IdDatosBancarios` | uniqueidentifier | SÍ | FK — Cuenta bancaria destino |
| `IdCatBanco` | uniqueidentifier | SÍ | FK — Banco emisor |
| `CuentaOrdenante` | varchar(80) | SÍ | Cuenta del cliente emisor |
| `ReferenciaBancaria` | varchar(80) | SÍ | Referencia bancaria del cobro |
| `MXN` | bit | NO | Moneda pesos |
| `USD` | bit | NO | Moneda dólares |
| `Activo` | bit | NO | **0 = Inconsistencia: elimina pendiente del Buzón** |

---

## 7. catCobroEstatus (NUEVO — R16)

**Propósito:** Catálogo de estatus del ciclo de vida de un cobro en `fccPagoCliente`, desde que se captura en el Buzón hasta que se completa el Paso 3 del wizard de Validar Cobro. Permite consultar y filtrar cobros por estado sin inferirlo de múltiples tablas o campos implícitos.

```sql
CREATE TABLE [dbo].[catCobroEstatus] (
    [IdCatCobroEstatus]  uniqueidentifier NOT NULL
        CONSTRAINT [DF_catCobroEstatus_Id]       DEFAULT (NEWID()),
    [Clave]              varchar(30)      NOT NULL,
    [Descripcion]        varchar(120)     NOT NULL,
    [Orden]              int              NOT NULL
        CONSTRAINT [DF_catCobroEstatus_Orden]    DEFAULT (0),
    [Activo]             bit              NOT NULL
        CONSTRAINT [DF_catCobroEstatus_Activo]   DEFAULT (1),
    [FechaRegistro]      datetime2        NOT NULL
        CONSTRAINT [DF_catCobroEstatus_FechaReg] DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT [PK_catCobroEstatus]
        PRIMARY KEY CLUSTERED ([IdCatCobroEstatus]),
    CONSTRAINT [UQ_catCobroEstatus_Clave]
        UNIQUE ([Clave])
);

-- DML inicial
INSERT INTO dbo.catCobroEstatus (Clave, Descripcion, Orden) VALUES
    ('BORRADOR',           'Captura iniciada en Paso 1, no confirmada',             1),
    ('CAPTURADO',          'Cobro confirmado en Paso 1, pendiente de asociar',      2),
    ('ASOCIADO',           'Vinculado a proforma o factura en Paso 2',              3),
    ('SALDO_A_FAVOR',      'Cobro con residual disponible tras asociación',         4),
    ('CON_INCONSISTENCIA', 'Marcado con inconsistencia en Paso 1 o Paso 2',         5),
    ('COMPLETADO',         'Documentos fiscales generados y enviados en Paso 3',    6),
    ('CANCELADO',          'Cancelado por falta de pago u otra razón operativa',    7);
```

### Diccionario de datos — catCobroEstatus

| Nombre de tabla | Descripción |
|---|---|
| catCobroEstatus | Catálogo de estatus del ciclo de vida del cobro en el wizard de Validar Cobro. |

| Columna | Tipo | Descripción |
|---|---|---|
| IdCatCobroEstatus | uniqueidentifier PK | Identificador único del estatus |
| Clave | varchar(30) UNIQUE | Clave textual: `BORRADOR`, `CAPTURADO`, `ASOCIADO`, `SALDO_A_FAVOR`, `CON_INCONSISTENCIA`, `COMPLETADO`, `CANCELADO` |
| Descripcion | varchar(120) | Descripción legible del estatus |
| Orden | int | Orden en el ciclo de vida para presentación |
| Activo | bit | 1 = vigente |
| FechaRegistro | datetime2 | Fecha de inserción |

### Ciclo de vida

```
[Correo clasificado como Cobro → INSERT fccFolioPagoCliente]
         ↓
    BORRADOR      ← Auto-guardado Paso 1 (Confirmado=0)
         ↓  (confirma cobro)
    CAPTURADO     ← Paso 1 completo (Confirmado=1, folio COB-mmddaa-NNNN)
         ↓  (asocia a documento en Paso 2)
    ASOCIADO  o  SALDO_A_FAVOR
         ↓  (Paso 3 genera y envía documento fiscal)
    COMPLETADO

    En cualquier punto → CON_INCONSISTENCIA  o  CANCELADO
```

---

## 8. ALTER TABLE fccPagoCliente — Agregar IdCatCobroEstatus

**Prerequisito:** `catCobroEstatus` debe existir con sus datos iniciales antes de ejecutar este script.

```sql
ALTER TABLE dbo.fccPagoCliente
    ADD [IdCatCobroEstatus] uniqueidentifier NOT NULL
        CONSTRAINT [DF_fccPagoCliente_IdCatCobroEstatus]
            DEFAULT (
                (SELECT TOP 1 IdCatCobroEstatus
                 FROM dbo.catCobroEstatus
                 WHERE Clave = 'BORRADOR')
            );

ALTER TABLE dbo.fccPagoCliente
    ADD CONSTRAINT [FK_fccPagoCliente_catCobroEstatus]
        FOREIGN KEY ([IdCatCobroEstatus])
        REFERENCES dbo.catCobroEstatus ([IdCatCobroEstatus]);

-- Verificación
SELECT c.name, c.column_id
FROM sys.columns c
WHERE c.object_id = OBJECT_ID('dbo.fccPagoCliente')
  AND c.name = 'IdCatCobroEstatus';
```

> **Nota sobre `Confirmado` (bit):** El campo `Confirmado` (definido en RE-023) puede coexistir con `IdCatCobroEstatus` por compatibilidad. `IdCatCobroEstatus` es la fuente de verdad del estado; `Confirmado` puede deprecarse a futuro.

---

## 9. RegionConfiguracionMailBot (Existente — referencia)

**Propósito:** Configuración del Mailbot por región. Define las etiquetas y cuenta de correo monitoreada.

| Columna | Tipo | Descripción |
|---|---|---|
| `IdRegionConfiguracionMailBot` | uniqueidentifier | PK |
| `IdRegion` | uniqueidentifier | FK — Región (MEX/PER) |
| `CorreoElectronico` | varchar(150) | Cuenta de correo monitoreada |
| `EtiquetasIds` | varchar(MAX) | Etiquetas de clasificación del Mailbot |
| `TipoClienteOrigen` | varchar(500) | Tipos de cliente origen |
| `Activo` | bit | Configuración activa |

> Para R16 se debe agregar la etiqueta ‘Cobro’ al modelo de entrenamiento del Mailbot en esta configuración por región.

---

## Filtrado de Bandeja por Cobrador

La visibilidad del Buzón de Cobros se filtra vía la cadena:

```
CorreoRecibidoCliente.IdCliente
    → ClienteCarteraCliente.IdCliente
    → ClienteCartera.IdUsuarioCobrador
    == Usuario.IdUsuario  (Gestor de Cobranza autenticado)
```

---

## Consultas SQL Principales

### Bandeja del Buzón de Cobros por Gestor

```sql
DECLARE @IdUsuarioCobrador UNIQUEIDENTIFIER;
DECLARE @ClaveCobro        VARCHAR(150) = 'cobro';  -- o 'pago' si no se renombra

SELECT
    cr.IdCorreoRecibido,
    cr.Asunto,
    cr.CorreoEmisor,
    cr.FechaRecepcion,
    crc.IdCorreoRecibidoCliente,
    crc.Leido,
    crc.Procesado,
    c.Nombre       AS Cliente,
    r.ClaveISO     AS Region,
    ffc.IdFCCFolioPagoCliente,
    ffc.Folio      AS FolioCobro,
    ffc.TotalMailBot
FROM dbo.CorreoRecibidoCliente crc
INNER JOIN dbo.CorreoRecibido cr
       ON crc.IdCorreoRecibido = cr.IdCorreoRecibido
INNER JOIN dbo.catClasificacionCorreoRecibido cat
       ON crc.IdCatClasificacionCorreoRecibido = cat.IdCatClasificacionCorreoRecibido
INNER JOIN dbo.Cliente c
       ON crc.IdCliente = c.IdCliente
INNER JOIN dbo.Region r
       ON cr.IdRegion = r.IdRegion
INNER JOIN dbo.ClienteCarteraCliente ccc
       ON c.IdCliente = ccc.IdCliente AND ccc.Activo = 1
INNER JOIN dbo.ClienteCartera cc
       ON ccc.IdClienteCartera = cc.IdClienteCartera AND cc.Activo = 1
LEFT  JOIN dbo.fccFolioPagoCliente ffc
       ON crc.IdCorreoRecibidoCliente = ffc.IdCorreoRecibidoCliente AND ffc.Activo = 1
WHERE cat.Clave            = @ClaveCobro
  AND cc.IdUsuarioCobrador = @IdUsuarioCobrador
  AND crc.Activo           = 1
ORDER BY cr.FechaRecepcion DESC;
```

### Reclasificar correo a otro buzón

```sql
DECLARE @IdCorreoRecibidoCliente   UNIQUEIDENTIFIER;
DECLARE @IdCatClasificacionDestino UNIQUEIDENTIFIER;  -- Ej: clave 'otros'

UPDATE dbo.CorreoRecibidoCliente
SET    IdCatClasificacionCorreoRecibido = @IdCatClasificacionDestino,
       FechaUltimaActualizacion         = GETDATE()
WHERE  IdCorreoRecibidoCliente = @IdCorreoRecibidoCliente;
```

### Cierre automático del pendiente al vincular cobro a proforma/factura

```sql
-- Al vincularse a proforma/factura en Validar Cobro:
DECLARE @IdFCCFolioPagoCliente UNIQUEIDENTIFIER;

UPDATE dbo.fccFolioPagoCliente
SET    Activo                   = 0,
       FechaUltimaActualizacion = GETDATE()
WHERE  IdFCCFolioPagoCliente = @IdFCCFolioPagoCliente;
```

### Bandeja del Coordinador de Tesorería — OBS-021

Concentra correos de cobro no enrutables: Caso 1 (cliente sin Cobrador) y Caso 2 (remitente no dado de alta).
La retroactividad al asignar Cobrador es **automática por diseño**: la query del Gestor filtra por `IdUsuarioCobrador`; en cuanto se asigna un Cobrador al cliente, los correos dejan de aparecer en esta consulta y aparecen automáticamente en la bandeja del Gestor asignado.

```sql
DECLARE @ClaveCobro VARCHAR(150) = 'cobro';  -- 'pago' mientras no se ejecute Tarea 1

SELECT
    cr.IdCorreoRecibido,
    cr.Asunto,
    cr.CorreoEmisor,
    cr.FechaRecepcion,
    crc.IdCorreoRecibidoCliente,
    crc.IdCliente,
    c.Nombre         AS Cliente,               -- NULL si Caso 2 (remitente no dado de alta)
    CASE
        WHEN crc.IdCliente IS NULL THEN 'Caso 2 — Remitente no dado de alta'
        WHEN cc.IdUsuarioCobrador IS NULL THEN 'Caso 1 — Cliente sin Cobrador asignado'
    END              AS MotivoBandejaCoordenador,
    r.ClaveISO       AS Region,
    ffc.IdFCCFolioPagoCliente,
    ffc.Folio        AS FolioCobro
FROM dbo.CorreoRecibidoCliente crc
INNER JOIN dbo.CorreoRecibido cr
       ON crc.IdCorreoRecibido = cr.IdCorreoRecibido
INNER JOIN dbo.catClasificacionCorreoRecibido cat
       ON crc.IdCatClasificacionCorreoRecibido = cat.IdCatClasificacionCorreoRecibido
INNER JOIN dbo.Region r
       ON cr.IdRegion = r.IdRegion
LEFT  JOIN dbo.Cliente c
       ON crc.IdCliente = c.IdCliente
LEFT  JOIN dbo.ClienteCarteraCliente ccc
       ON c.IdCliente = ccc.IdCliente AND ccc.Activo = 1
LEFT  JOIN dbo.ClienteCartera cc
       ON ccc.IdClienteCartera = cc.IdClienteCartera AND cc.Activo = 1
LEFT  JOIN dbo.fccFolioPagoCliente ffc
       ON crc.IdCorreoRecibidoCliente = ffc.IdCorreoRecibidoCliente AND ffc.Activo = 1
WHERE cat.Clave  = @ClaveCobro
  AND crc.Activo = 1
  AND (crc.IdCliente IS NULL               -- Caso 2: remitente no dado de alta
       OR cc.IdUsuarioCobrador IS NULL)    -- Caso 1: cliente sin Cobrador
ORDER BY cr.FechaRecepcion ASC;            -- más antiguo primero
```

### Verificación de uso de la clasificación ‘Pago’ (previo a decisión A/B)

```sql
SELECT COUNT(*) AS TotalCorreosClasificadosPago
FROM dbo.CorreoRecibidoCliente crc
INNER JOIN dbo.catClasificacionCorreoRecibido cat
       ON crc.IdCatClasificacionCorreoRecibido = cat.IdCatClasificacionCorreoRecibido
WHERE cat.Clave = 'pago'
  AND crc.Activo = 1;
```

---

## Scripts de Cambio R16 — Orden de ejecución

| # | Script | Tabla | Prioridad |
|:-:|--------|-------|:---------:|
| 1 | `CREATE TABLE catCobroEstatus` + DML inicial | Nueva | 🔴 Alta |
| 2 | `ALTER TABLE fccPagoCliente` — ADD `IdCatCobroEstatus` | Existente | 🔴 Alta |
| 3 | Decisión A o B: clasificación 'Cobro' en `catClasificacionCorreoRecibido` | Existente | 🔴 Alta |
| 4 | INSERT proceso 'Cobros' en `catProceso` (si Opción B) | Existente | 🔴 Alta |

> **⚠️ Decisión requerida antes del desarrollo**
>
> - **Opción A** (recomendada si ‘Pago’ no se usa en otros módulos activos): Renombrar ‘Pago’ → ‘Cobro’ en `catClasificacionCorreoRecibido`.
> - **Opción B** (si ‘Pago’ se usa en otros contextos): Insertar nuevo registro ‘Cobro’ + proceso ‘Cobros’ en `catProceso`, y desactivar registro ‘Pago’ si corresponde.
>
> Ejecutar primero la consulta de verificación de uso de ‘Pago’ (sección anterior) para tomar la decisión con datos.

---

## Gaps y Acciones Pendientes

| # | Gap | Descripción | Acción | Prioridad |
|---|---|---|---|---|
| 1 | Clasificación ‘Cobro’ inexistente | `catClasificacionCorreoRecibido` tiene ‘Pago’ no ‘Cobro’ | Decidir Opción A o B y ejecutar script | Alta |
| 2 | `catProceso` sin ‘Cobros’ | Proceso de cobros no existe en BD | Insertar registro ‘Cobros’ | Alta |
| 3 | Mailbot sin entrenamiento Cobro | El modelo Mailbot no detecta correos como ‘Cobro’ | Entrenamiento con correos reales de PROQUIFA | Alta |
| 4 | Cliente no identificable en correo | `IdCliente` nullable en `CorreoRecibidoCliente` | Confirmar con cliente quién atiende estos correos | Media |
| 5 | Sin proceso ‘Cobranza’ en `catProceso` | `catProceso` no tiene proceso Cobros/Cobranza | Insertar registro según nomenclatura acordada | Alta |
| 6 | Segregación regional sin validar | `CorreoRecibido.IdRegion` existe — confirmar que Mailbot lo popula | Validar con equipo técnico | Media |

---

## Reglas de Negocio

| Regla | Descripción | Implementación en BD |
|---|---|---|
| Regla 1 | Clasificación automática por Mailbot | `catClasificacionCorreoRecibido.Clave = ‘cobro’` |
| Regla 2 | Sin criterios configurables | Entrenamiento del Mailbot — sin tabla de criterios |
| Regla 3 | Reflejo en Buzón del Gestor | `CorreoRecibidoCliente` → filtro por `IdUsuarioCobrador` |
| Regla 4 | Pendiente automático en Validar Cobro | INSERT en `fccFolioPagoCliente` al clasificar |
| Regla 5 | Visibilidad por cobrador y región | JOIN `ClienteCartera.IdUsuarioCobrador` + `CorreoRecibido.IdRegion` |
| Regla 6 | Cierre al vincular a proforma/factura | `fccFolioPagoCliente.Activo = 0` |
| Regla 7 | Eliminación ante inconsistencia | `fccFolioPagoCliente.Activo = 0` |
| Regla 8 | Reclasificación manual | UPDATE `CorreoRecibidoCliente.IdCatClasificacionCorreoRecibido` |
| Regla 9 | Sin eliminación directa por Gestor | No DELETE — reclasificar a ‘Otros’ para que ESAC elimine |
| Regla 10 | Mismos filtros que buzones existentes | Patrón de `CorreoRecibidoCliente` ya usado en Requisición/Pedido |
| Regla 11 | Bandeja Coordinador Tesorería — Caso 1: cliente sin Cobrador | LEFT JOIN `ClienteCartera` → `IdUsuarioCobrador IS NULL` → bandeja del Coordinador. OBS-021 |
| Regla 12 | Bandeja Coordinador Tesorería — Caso 2: remitente no dado de alta | `CorreoRecibidoCliente.IdCliente IS NULL` → bandeja del Coordinador. OBS-021 |

---

## Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| 1 | ~~Cliente sin Cobrador asignado — correo invisible~~ | **Resuelto — OBS-021.** El correo va a la **bandeja del Coordinador de Tesorería** (Regla 11, Caso 1). Al asignar un Cobrador al cliente, los correos pasan automáticamente a la bandeja del Gestor por diseño de la query (retroactividad estructural). |
| 2 | ~~Correo no identifica al cliente~~ | **Resuelto — OBS-021.** El correo va a la **bandeja del Coordinador de Tesorería** (Regla 12, Caso 2). El Coordinador da de alta el contacto para que el correo quede asociado al cliente. |

---

## Módulos Relacionados

| Módulo | Requisito | Relación |
|---|---|---|
| Catálogo de Clientes — Cobrador | R16A-RE-FU-002 | `IdUsuarioCobrador` en `ClienteCartera` es el filtro de bandeja |
| Validar Cobro | R16A-RE-FU-009 | El pendiente `fccFolioPagoCliente` se trabaja en Validar Cobro |
