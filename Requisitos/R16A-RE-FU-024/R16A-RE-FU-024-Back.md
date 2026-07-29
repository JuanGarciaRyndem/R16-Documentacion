# Impacto en Back — R16A-RE-FU-024
**Requisito:** Validar Cobro: Paso 1 México — Captura del Cobro
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)
**Módulo:** Validar Cobro — Wizard Paso 1 (México)
**Impacto:** Scripts BD ProquifaDotNet (ALTER fccPagoCliente x5 RE-FU-023 + **ALTER fccPagoCliente x2 NUEVOS RE-FU-024 (BloqueadoPorTimbrado, FechaBloqueoTimbrado)** + CREATE catTipoInconsistenciaCobro + CREATE fccInconsistenciaCobro + CREATE SEQUENCE SeqFolioCobro) + Endpoints Finanzas: listado cobros, detalle correo, captura/autoguardado/finalización cobro, **edición del cobro mientras no esté timbrado**, foliador COB, TC del día, modal inconsistencia + llamadas entre APIs (Finanzas → ProquifaDotNet, incluye catálogos existentes de moneda/medio de pago/cuenta destino)

> **🔄 Cambio funcional 2026-06-23 — Editabilidad del cobro hasta el timbrado:**
> La regla "cobro inmutable al confirmar" se actualizó a "cobro inmutable al timbrar el documento asociado en el Paso 3". Impacto en Back: (1) nuevo campo BD `BloqueadoPorTimbrado` + `FechaBloqueoTimbrado` en `fccPagoCliente`; (2) el endpoint de finalización de la captura (B5) ya no aplica inmutabilidad, solo genera folio COB y marca `Confirmado=1`; (3) **nuevo endpoint B9 — Editar cobro** que permite UPDATE del formulario completo mientras `BloqueadoPorTimbrado=0` (aun si el cobro ya está asociado en Paso 2); (4) el listado de cobros (B1) debe retornar el flag `canEdit` para que la UI condicione el botón Editar; (5) las guardias de los endpoints de auto-guardado y edición evalúan `BloqueadoPorTimbrado` en lugar de `Confirmado`; (6) el flip de `BloqueadoPorTimbrado=1` lo dispara el Paso 3 al timbrar el documento asociado (UPDATE sobre todas las `fccPagoCliente` aplicadas a ese documento).

---

## Resumen

Este requisito implementa la **primera pantalla del wizard de Validar Cobro (Paso 1 - Captura del Cobro) para Región México** en ProquifaDotNet.Finanzas. El usuario revisa los correos del Buzón del cliente, selecciona el comprobante de pago adjunto y captura los datos del cobro (folio, monto, fecha, forma de pago SAT, cuenta origen/destino, **moneda via combo catMoneda**, TC del día). Un cobro capturado se guarda en modo lectura y permanece **editable mediante botón Editar mientras el documento asociado no haya sido timbrado en el Paso 3**, aun si el cobro ya está asociado en el Paso 2. Al timbrar el documento asociado, el cobro queda inmutable (sin botón Editar). Permite capturar múltiples cobros en la misma sesión con auto-guardado transparente.

> **OBS-048 — Reanudación del wizard en el último paso activo:** Al ingresar al wizard para un cliente, Finanzas evalúa el estado actual del flujo (via `GET /api/v1/validate-collection/client/{idCliente}/wizardStatus`) y redirige al último paso activo donde el usuario se encontraba, no necesariamente al Paso 1. Si el Paso 1 ya tiene cobros confirmados pero el Paso 2 está pendiente, el wizard abre en el Paso 2 directamente.

El impacto en BD (ProquifaDotNet) es **moderado**: 5 ALTER ya ejecutados en RE-FU-023 (Confirmado + FechaConfirmacion + IdUsuarioConfirmacion + Notas + IdCatMoneda) + **2 ALTER nuevos en RE-FU-024 (BloqueadoPorTimbrado + FechaBloqueoTimbrado)** + 2 tablas nuevas + 1 SEQUENCE. El impacto en servicios (Finanzas) es **alto**: orquestación completa del Paso 1 más un nuevo endpoint de edición del cobro post-captura.

### Distribución de responsabilidades

| Capa | Aplicativo | Responsabilidad |
|------|-----------|----------------|
| BD | ProquifaDotNet | `fccPagoCliente`, `catTipoInconsistenciaCobro`, `fccInconsistenciaCobro`, `SeqFolioCobro` |
| API Datos | ProquifaDotNet | Expone endpoints de Buzón (correos, adjuntos) — siguen activos para Venta Interna, no deprecados; Finanzas construye sus propios endpoints en paralelo (ver Parte C) |
| Catálogos | ProquifaDotNet (existente) | `POST /Catalogos/{catMoneda,catMedioDePago,vEmpresaDatosBancarios}` — consumidos directamente por el frontend de Validar Cobro; Finanzas no crea endpoints propios (activos, no deprecados — ver Parte C) |
| Lógica Paso 1 | ProquifaDotNet.Finanzas | Orquesta el Paso 1: listado cobros, captura, auto-guardado, confirmación, TC, inconsistencias |
| TC del día | ProquifaDotNet.Finanzas | Calcula TC FIX Banxico/DOF (fuente pendiente de confirmar) para la moneda no-MXN involucrada |
| Comunicación | Finanzas → ProquifaDotNet | Lecturas de origen de datos (`CorreoRecibido`, `Archivo`) para construir sus propios endpoints `/api/v1/validate-collection/*` (Buzón); catálogos (`catMoneda`, `catMedioDePago`, `EmpresaDatosBancarios`) consumidos directamente por el frontend sin intermediario; escritura de cobros e inconsistencias |

### Infraestructura reutilizada

| Componente                                                                | Origen                   | Reutilización                                                       |
| ------------------------------------------------------------------------- | ------------------------ | ------------------------------------------------------------------- |
| `fccPagoCliente`                                                          | ProquifaDotNet existente | Tabla principal del cobro; se extiende con 5 nuevos campos          |
| `fccFolioPagoCliente` / `CorreoRecibidoCliente` / `ArchivoCorreoRecibido` | RE-FU-008                | Buzón de Cobros: correos y adjuntos del cliente                     |
| `catMedioDePago` (con `ClaveFormaDePago` SAT)                             | ProquifaDotNet existente | Catálogo forma de pago c_FormaPago SAT para México                  |
| `catMoneda`                                                               | ProquifaDotNet existente | **Combo moneda del cobro — IdCatMoneda FK nuevo en fccPagoCliente** |
| `EmpresaDatosBancarios` / `DatosBancarios`                                | ProquifaDotNet existente | Cuentas bancarias PROQUIFA México (cuenta destino del cobro)        |
| `DatosFacturacionCliente`                                                 | ProquifaDotNet existente | Moneda de facturación del cliente (para cálculo del TC)             |
| `IdentityServer`                                                          | Estándar transversal     | Autenticación/autorización en Finanzas                              |
| `Serilog`                                                                 | Estándar transversal     | Logs con contexto (usuario, módulo, operación)                      |
|                                                                           |                          |                                                                     |

---

## Parte A — Base de Datos (ProquifaDotNet)

### A1 — ALTER TABLE fccPagoCliente (5 campos RE-FU-023 + 2 campos nuevos RE-FU-024)

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing

-- =========================================================
-- Bloque A: ALTERs RE-FU-023 (✅ ya ejecutados en BD)
-- =========================================================

-- 1. Captura del cobro: 0=borrador (auto-guardado) / 1=capturado (editable hasta timbrar)
--    NOTA RE-FU-024: la semántica del campo cambia. Ya NO implica inmutabilidad,
--    indica únicamente que la captura fue finalizada y el cobro tiene folio COB.
ALTER TABLE dbo.fccPagoCliente
    ADD Confirmado bit NOT NULL CONSTRAINT [DF_fccPagoCliente_Confirmado] DEFAULT (0);

-- 2. Timestamp de captura inicial del cobro
ALTER TABLE dbo.fccPagoCliente
    ADD FechaConfirmacion datetime NULL;

-- 3. Trazabilidad: quién capturó el cobro inicialmente
ALTER TABLE dbo.fccPagoCliente
    ADD IdUsuarioConfirmacion uniqueidentifier NULL;

-- 4. Notas del cobro (campo opcional del formulario)
ALTER TABLE dbo.fccPagoCliente
    ADD Notas varchar(500) NULL;

-- 5. Moneda del cobro como FK a catMoneda (detectado en pantalla: combo desplegable)
--    Los flags MXN/USD existentes se mantienen por compatibilidad legacy
ALTER TABLE dbo.fccPagoCliente
    ADD IdCatMoneda uniqueidentifier NULL
        CONSTRAINT [FK_fccPagoCliente_CatMoneda]
            FOREIGN KEY REFERENCES dbo.catMoneda([IdCatMoneda]);

-- =========================================================
-- Bloque B: ALTERs NUEVOS RE-FU-024 (❌ pendientes de ejecutar)
-- =========================================================

-- 6. Inmutabilidad real del cobro: 0=editable vía botón Editar / 1=inmutable post-timbrado
--    Flip lo dispara el Paso 3 al timbrar el documento asociado al cobro.
ALTER TABLE dbo.fccPagoCliente
    ADD BloqueadoPorTimbrado bit NOT NULL
        CONSTRAINT [DF_fccPagoCliente_BloqueadoPorTimbrado] DEFAULT (0);

-- 7. Trazabilidad del bloqueo por timbrado
ALTER TABLE dbo.fccPagoCliente
    ADD FechaBloqueoTimbrado datetime NULL;
```

> **Motivo de IdCatMoneda:** La pantalla del Paso 1 muestra "Moneda*" como combo desplegable
> (ej. "USD" seleccionado). Los flags `MXN`/`USD` bit no soportan combos ni monedas adicionales
> como PEN (Perú). Se agrega `IdCatMoneda FK catMoneda` nullable para el combo de la UI.

> **Motivo de BloqueadoPorTimbrado / FechaBloqueoTimbrado (RE-FU-024):** Implementan la regla actualizada de inmutabilidad. `BloqueadoPorTimbrado=0` significa que el cobro está en modo lectura **editable** (botón Editar visible aun si ya está asociado en Paso 2). `BloqueadoPorTimbrado=1` significa inmutabilidad real (sin botón Editar). El Paso 3 actualiza estos campos al timbrar el documento asociado al cobro.

### A2 — CREATE TABLE catTipoInconsistenciaCobro

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
CREATE TABLE [dbo].[catTipoInconsistenciaCobro](
    [IdCatTipoInconsistenciaCobro] uniqueidentifier NOT NULL
        CONSTRAINT [DF_catTipoInconsistenciaCobro_Id] DEFAULT (NEWID()),
    [Clave]       varchar(50)  NOT NULL,
    [Descripcion] varchar(200) NOT NULL,
    [AplicaPaso]  varchar(1)   NOT NULL,  -- '1'=Paso 1 (cobro), '2'=Paso 2 (asociación)
    [Activo]      bit          NOT NULL CONSTRAINT [DF_catTipoInconsistenciaCobro_Activo] DEFAULT (1),
    CONSTRAINT [PK_catTipoInconsistenciaCobro] PRIMARY KEY CLUSTERED ([IdCatTipoInconsistenciaCobro]),
    CONSTRAINT [UQ_catTipoInconsistenciaCobro_Clave] UNIQUE ([Clave])
);

INSERT INTO [dbo].[catTipoInconsistenciaCobro] ([Clave], [Descripcion], [AplicaPaso])
VALUES
    ('DATOS_INCOMPLETOS',      'Datos del cobro incompletos o ilegibles',   '1'),
    ('COMPROBANTE_INVALIDO',   'Comprobante de pago inválido o ilegible',   '1'),
    ('FORMATO_INCORRECTO',     'Formato del comprobante incorrecto',         '1'),
    ('MONTO_ILEGIBLE',         'Monto del comprobante ilegible',             '1'),
    ('PAGO_INCOMPLETO_VENCIDO','Pago incompleto con vigencia vencida',       '2'),
    ('PAGO_INSUFICIENTE',      'Cobro insuficiente respecto al documento',   '2');
```

### A3 — CREATE TABLE fccInconsistenciaCobro

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
CREATE TABLE [dbo].[fccInconsistenciaCobro](
    [IdFCCInconsistenciaCobro]     uniqueidentifier NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_Id] DEFAULT (NEWID()),
    [IdFCCPagoCliente]             uniqueidentifier NOT NULL,
    [IdCatTipoInconsistenciaCobro] uniqueidentifier NOT NULL,
    [Comentario]                   varchar(500) NULL,
    [IdUsuarioRegistro]            uniqueidentifier NOT NULL,
    [Activo]                       bit NOT NULL CONSTRAINT [DF_fccInconsistenciaCobro_Activo] DEFAULT (1),
    [FechaRegistro]                datetime NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_FechaReg] DEFAULT (GETDATE()),
    [FechaUltimaActualizacion]     datetime NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_FechaUpd] DEFAULT (GETDATE()),
    CONSTRAINT [PK_fccInconsistenciaCobro]
        PRIMARY KEY CLUSTERED ([IdFCCInconsistenciaCobro]),
    CONSTRAINT [FK_fccInconsistenciaCobro_PagoCliente]
        FOREIGN KEY ([IdFCCPagoCliente]) REFERENCES dbo.fccPagoCliente([IdFCCPagoCliente]),
    CONSTRAINT [FK_fccInconsistenciaCobro_TipoInconsistencia]
        FOREIGN KEY ([IdCatTipoInconsistenciaCobro]) REFERENCES dbo.catTipoInconsistenciaCobro([IdCatTipoInconsistenciaCobro])
);
```

### A4 — CREATE SEQUENCE SeqFolioCobro

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing
-- Ajustar START WITH al MAX consecutivo existente + 1
CREATE SEQUENCE dbo.SeqFolioCobro
    AS INT
    START WITH 1
    INCREMENT BY 1
    NO CYCLE;
-- Uso: 'COB-' + FORMAT(FechaPago,'MMddyy') + '-' + RIGHT('000000' + CAST(NEXT VALUE FOR dbo.SeqFolioCobro AS VARCHAR), 6)
-- Pendiente confirmar: global (MEX+PER compartido) o por región
```

---

## Parte B — ProquifaDotNet.Finanzas: Servicios y Endpoints

### B1 — Listado de cobros del cliente en el Paso 1

**Descripción:** Endpoint que retorna el listado de cobros del Buzón del cliente para el panel izquierdo, ordenado en dos bloques (capturados arriba por `FechaPago ASC`, sin capturar abajo por `FechaRecepcion ASC`).

**Datos (vía API ProquifaDotNet):** `fccFolioPagoCliente`, `CorreoRecibidoCliente`, `fccPagoCliente`, `catMoneda`

| Estado item | Muestra | Ordenamiento | `canEdit` (flag DTO) |
|-------------|---------|-------------|--------------------------|
| Capturado editable (`Confirmado=1 AND BloqueadoPorTimbrado=0`) | Folio COB-mmddaa-NNNN, fecha, monto + moneda | Por `FechaPago ASC` | `true` (UI muestra botón Editar) |
| Capturado inmutable (`Confirmado=1 AND BloqueadoPorTimbrado=1`) | Folio COB-mmddaa-NNNN, fecha, monto + moneda; o etiqueta "Saldo a favor" si aplica | Por `FechaPago ASC` | `false` (UI NO muestra botón Editar) |
| Sin capturar | Etiqueta temporal "COB-N" | Por `FechaRecepcion ASC` del correo | N/A |

> El DTO `PaymentValidationStep1ItemDto` debe exponer el flag `canEdit` para que el Front condicione la visibilidad del botón Editar. Cálculo en el Handler: `canEdit = Confirmado && !BloqueadoPorTimbrado`.

---

### B2 — Detalle del correo y adjuntos (selección del comprobante)

**Descripción:** El detalle del correo seleccionado (asunto, cuerpo, fecha, hora, contacto del cliente) y la lista de adjuntos como opciones de radio button para seleccionar el comprobante se consumen **directamente** de los endpoints existentes de ProquifaDotNet — Finanzas **no crea endpoint propio** para este flujo (confirmado por captura de tráfico HTTP real, 07/07/2026).

| Endpoint | Método | Parámetros | Descripción |
|----------|--------|-----------|-------------|
| `/Catalogos/CorreoRecibido` | GET | `?idCorreoRecibido={guid}` | Datos del correo: asunto, fecha, hora, contacto |
| `/Catalogos/CorreoRecibidoContenido` | GET | `?idCorreoRecibidoContenido={guid}` | Cuerpo/contenido del correo |
| `/Catalogos/ArchivoCorreoRecibido` | POST | Body: `{ Filters: [{Activo:true},{IdCorreoRecibido:{guid}},{Mostrar:true}], GroupColumn }` | Lista de adjuntos del correo, candidatos a comprobante |
| `/Catalogos/Archivo` | GET | `?idArchivo={guid}` | Detalle/descarga de un adjunto específico |

> **Catálogos, no deprecados:** los endpoints de ProquifaDotNet (`/Catalogos/CorreoRecibido`, `/Catalogos/CorreoRecibidoContenido`, `/Catalogos/ArchivoCorreoRecibido`, `/Catalogos/Archivo`) siguen activos y en uso por Venta Interna — no se deprecan, y el frontend de Validar Cobro (Finanzas) los consume directamente sin intermediario — ver `Endpoints-ProquifaDotNet.md`.

**Datos (vía API ProquifaDotNet, consumidos directamente por el frontend):** `CorreoRecibido`, `CorreoRecibidoContenido`, `ArchivoCorreoRecibido`, `Archivo`, `ContactoCliente`

---

> **Nota:** Los catálogos del formulario del Paso 1 (moneda, medio de pago, cuenta destino) son endpoints de **Finanzas** — ver Parte C. Los equivalentes en ProquifaDotNet (Catálogos) no se deprecan — siguen activos para Venta Interna.

### B4 — Auto-guardado del cobro en borrador

**Descripción:** Endpoint `PUT` en Finanzas que persiste el estado del formulario como borrador (`Confirmado=0`) de forma transparente. Incluye `IdCatMoneda` seleccionada.

**Operación (vía API ProquifaDotNet):**
- Si no existe `fccPagoCliente`: `INSERT` con `Confirmado=0`, `BloqueadoPorTimbrado=0`, `Folio=NULL`, `IdCatMoneda=@IdMoneda`
- Si existe con `Confirmado=0`: `UPDATE` con todos los datos del formulario incluido `IdCatMoneda`
- **Guardia (RE-FU-024 actualizada):** si `BloqueadoPorTimbrado=1`, no sobreescribe (cobro inmutable post-timbrado). Si `Confirmado=1 AND BloqueadoPorTimbrado=0`, el auto-guardado del borrador no aplica (el cobro ya fue capturado); para modificar usar el endpoint B9 (Editar).

---

### B5 — Finalización de la captura del cobro (folio COB)

**Descripción:** Endpoint `POST` en Finanzas que finaliza la captura del cobro. Valida selección de comprobante, genera folio COB con `SeqFolioCobro`, marca el cobro como capturado con `Confirmado=1`. **NO aplica inmutabilidad** — el cobro queda editable vía botón Editar (endpoint B9) hasta el timbrado del documento asociado.

**Flujo antes de llamar a ProquifaDotNet:**
1. Validar comprobante seleccionado (adjunto marcado como oficial)
2. Validar campos obligatorios completos (incluido `IdCatMoneda`)
3. Calcular folio: `'COB-' + FORMAT(FechaPago,'MMddyy') + '-' + LPAD(NEXT VALUE FOR SeqFolioCobro, 6, '0')`
4. **Sin alerta de confirmación al Front** (el cobro permanece editable hasta el timbrado).

**Operación (vía API ProquifaDotNet):**
```sql
UPDATE dbo.fccPagoCliente
SET Folio                 = @FolioGenerado,
    Confirmado            = 1,
    FechaConfirmacion     = GETDATE(),
    IdUsuarioConfirmacion = @IdUsuarioActivo,
    IdCatMoneda           = @IdCatMoneda
WHERE IdFCCPagoCliente    = @Id
  AND Confirmado          = 0;  -- guardia: solo finalizar borradores, no recapturar
-- NOTA: BloqueadoPorTimbrado permanece en 0 — el cobro queda editable hasta el timbrado.
```

---

### B6 — Tipo de Cambio del día automático (México)

**Descripción:** Servicio en Finanzas que calcula el TC del día según la moneda del cobro (`IdCatMoneda`) y la moneda de facturación del cliente (`DatosFacturacionCliente.IdCatMoneda`).

| Moneda cobro (`IdCatMoneda`) | Moneda facturación | TC a capturar |
|---|---|---|
| MXN | MXN | N/A |
| MXN | Distinta a MXN | TC del día de la moneda de facturación vs MXN |
| Distinta a MXN | Cualquiera | TC del día de la moneda del cobro vs MXN |

**Fuente:** TC FIX Banxico/DOF — **pendiente confirmar con PROQUIFA**.

> `IdCatMoneda` del formulario se usa para determinar qué TC calcular, en lugar de los flags `MXN`/`USD`.

---

### B7 — Modal de inconsistencia del cobro (Paso 1)

**Descripción:** Endpoint en Finanzas que obtiene los tipos del catálogo filtrados por `AplicaPaso='1'` y registra la inconsistencia en `fccInconsistenciaCobro`.

**Endpoints:**
- `GET /api/v1/validate-collection/inconsistencyType?step=1` → catálogo filtrado (solo tipos del Paso 1)
- `POST /api/v1/validate-collection/client/{idCliente}/payment/{idFCCPagoCliente}/inconsistency` → INSERT en `fccInconsistenciaCobro`

---

### B8 — Estado del wizard para reanudación (OBS-048)

**Descripción:** Endpoint en Finanzas que retorna el paso activo actual del wizard para un cliente, permitiendo que la UI rediriga al último paso donde el usuario se encontraba en lugar de siempre abrir en el Paso 1.

**Endpoint:** `GET /api/v1/validate-collection/client/{idCliente}/wizardStatus`

**Lógica de determinación del paso activo:**

| Condición | Paso activo retornado |
|-----------|----------------------|
| Sin cobros en `fccPagoCliente` para el cliente | Paso 1 |
| Hay cobros con `Confirmado=0` (borrador activo) | Paso 1 |
| Todos los cobros del Buzón tienen `Confirmado=1` y existe asociación pendiente en Paso 2 | Paso 2 |
| Paso 2 completado y existe validación pendiente en Paso 3 | Paso 3 |

**DTO de respuesta:**
```json
{
  "activeStep": 1,
  "description": "Captura del Cobro",
  "hasDraft": true,
  "draftPaymentId": "guid-opcional"
}
```

> Esta lógica se expande en los requisitos de Paso 2 (RE-FU-025) y Paso 3. La regla base: el paso activo es el primer paso que aún tiene trabajo pendiente. (OBS-048)

---

### B9 — Edición del cobro capturado (botón Editar — RE-FU-024)

**Descripción:** Endpoint `PUT` en Finanzas que permite **editar un cobro ya capturado** (`Confirmado=1`) mientras el documento asociado al cobro NO haya sido timbrado (`BloqueadoPorTimbrado=0`). Disparado por el botón "Editar" del item del listado del Paso 1. Aplica aun si el cobro ya está asociado en el Paso 2 (no requiere desasociar para editar).

**Endpoint:** `PUT /api/v1/validate-collection/payment/{idFCCPagoCliente}/edit`

**Flujo en Finanzas:**
1. Cargar el cobro y verificar `Confirmado=1`
2. Verificar `BloqueadoPorTimbrado=0` (si está bloqueado, retornar `409 Conflict — Cobro inmutable por timbrado`)
3. Si el usuario cambia `IdCatMoneda` o `FechaPago`, **recalcular TC** vía B6 (`ExchangeRateService`)
4. Validar comprobante seleccionado (sigue siendo obligatorio) y campos obligatorios
5. NO regenerar `Folio` (el folio se mantiene)
6. Persistir cambios vía UPDATE en ProquifaDotNet

**Campos editables:** `Monto`, `FechaPago`, `IdCatMedioDePago`, `CuentaOrdenante`, `IdDatosBancarios`, `IdCatMoneda`, `TipoDeCambio` (recalculado, no editable directamente), `IdArchivo` (comprobante seleccionado), `Notas`.

**Operación (vía API ProquifaDotNet):**
```sql
UPDATE dbo.fccPagoCliente
SET Monto                    = @Monto,
    FechaPago                = @FechaPago,
    IdCatMedioDePago         = @IdCatMedioDePago,
    CuentaOrdenante          = @CuentaOrdenante,
    IdDatosBancarios         = @IdDatosBancarios,
    IdCatMoneda              = @IdCatMoneda,
    TipoDeCambio             = @TipoCambioRecalculado,
    IdArchivo                = @IdArchivoComprobante,
    Notas                    = @Notas,
    FechaUltimaActualizacion = GETDATE()
WHERE IdFCCPagoCliente       = @Id
  AND Confirmado             = 1
  AND BloqueadoPorTimbrado   = 0;  -- guardia: solo editable si NO timbrado
```

**Respuestas:**

| Código | Caso |
|--------|------|
| `200 OK` | Cobro actualizado correctamente |
| `404 Not Found` | El cobro no existe |
| `409 Conflict` | `BloqueadoPorTimbrado=1` (documento asociado ya timbrado, cobro inmutable) |
| `400 Bad Request` | Validación fallida (comprobante no seleccionado, campos obligatorios vacíos, etc.) |

> **Impacto en Paso 2:** si el cobro editado ya estaba asociado a una proforma/factura en el Paso 2, las conversiones operativas (montos aplicados, TC) deben re-evaluarse en cliente. El servicio del Paso 2 debe revalidar al consultar el cobro o al confirmar la asociación. **Detalle de re-evaluación: ver RE-FU-026 (Paso 2 México).**

---

### B10 — Bloqueo del cobro al timbrar el documento asociado (disparado desde Paso 3)

**Descripción:** Operación interna invocada desde el flujo de timbrado del Paso 3 (Facturación y Envío). Al timbrarse exitosamente el documento (factura/proforma) que tiene cobros aplicados, el servicio del Paso 3 dispara un UPDATE sobre TODAS las `fccPagoCliente` aplicadas a ese documento para marcarlas como inmutables.

**Operación (vía API ProquifaDotNet):**
```sql
-- Para todas las fccPagoCliente aplicadas al documento timbrado @IdDocumento
UPDATE pc
SET pc.BloqueadoPorTimbrado = 1,
    pc.FechaBloqueoTimbrado = GETDATE()
FROM dbo.fccPagoCliente pc
INNER JOIN <tabla_aplicacion_paso2> ap ON ap.IdFCCPagoCliente = pc.IdFCCPagoCliente
WHERE ap.IdDocumento           = @IdDocumentoTimbrado
  AND pc.BloqueadoPorTimbrado  = 0;  -- evitar re-bloqueo
```

> La tabla de aplicación del Paso 2 (`<tabla_aplicacion_paso2>`) se define en RE-FU-026 (Paso 2 México). Aquí se documenta solo el efecto sobre `fccPagoCliente`.

> **Idempotencia:** la guardia `BloqueadoPorTimbrado=0` evita doble bloqueo si el Paso 3 reintenta. El UPDATE no debe re-disparar lógica adicional sobre cobros ya bloqueados.

---

## Parte C — Catálogos del formulario del Paso 1 (ProquifaDotNet.Finanzas)

### C1 — Catálogos del formulario del Paso 1

**Descripción:** Los 3 combos del formulario de captura del cobro (Moneda, Medio de pago, Cuenta destino) consumen **directamente** los endpoints existentes de ProquifaDotNet (Área Catálogos) — Finanzas **no crea endpoints propios** para estos catálogos; no es una nueva app ni un wrapper.

| Endpoint | Método | Body | Descripción |
|----------|--------|------|-------------|
| `/Catalogos/catMoneda` | POST | `QueryInfo` (`Activo = 1`) | Obtener las monedas activas dependiendo de la región del usuario logueado |
| `/Catalogos/catMedioDePago` | POST | `QueryInfo` (`Activo = 1`) | Obtener los medios de pago activos dependiendo de la región del usuario logueado |
| `/Catalogos/vEmpresaDatosBancarios` | POST | `QueryInfo` (`Activo = 1`) | Obtener los datos bancarios de las empresas activas dependiendo de la región del usuario logueado |

> **Catálogos, no deprecados:** los endpoints de ProquifaDotNet (`/Catalogos/catMoneda`, `/Catalogos/catMedioDePago`, `/Catalogos/vEmpresaDatosBancarios`) siguen activos y en uso por Venta Interna — no se deprecan, y el frontend de Validar Cobro (Finanzas) los consume directamente sin intermediario — ver `Endpoints-ProquifaDotNet.md`.

> El combo "Moneda" carga desde `catMoneda` y se persiste como `IdCatMoneda` en `fccPagoCliente`.
> `catMedioDePago` y `vEmpresaDatosBancarios` ya tienen filtro de región implementado (RE-FU-005 y RE-FU-001 respectivamente — ver `Endpoints-ProquifaDotNet.md`). Pendiente confirmar si `catMoneda` ya cuenta con el mismo filtro de región o si requiere el mismo tratamiento aplicado a `catMedioDePago` en RE-FU-005 (columna `IdRegion` + `BaseApiController`/`AsegurarFiltroRegion`) — ver Brechas.
> Estos endpoints son de propósito general (no exclusivos de Validar Cobro); no deben re-implementarse ni envolverse en requisitos posteriores que también los necesiten (p. ej. RE-FU-025, RE-FU-032/033) — solo referenciarlos como dependencia de consumo directo.

---

## Brechas

> ⚠️ **BRECHA — Catálogo de Tipos de Inconsistencia del Paso 1 pendiente (Riesgo 1)**
> Datos iniciales son propuesta. Catálogo completo pendiente de PROQUIFA Tesorería.

> ⚠️ **BRECHA — Fuente oficial del TC del día para México**
> Pendiente confirmar si PROQUIFA usa TC FIX Banxico/DOF u otra fuente propia.

> ⚠️ **BRECHA — Foliador global vs por región**
> Pendiente confirmar si el consecutivo es compartido MEX+PER o independiente por región.

> ⚠️ **BRECHA — Flags MXN/USD vs IdCatMoneda**
> Pendiente confirmar si los flags `MXN`/`USD` existentes se deprecan o conviven con `IdCatMoneda`.

> ⚠️ **BRECHA — Filtro de región en `/Catalogos/catMoneda` (RE-FU-024)**
> `catMedioDePago` y `vEmpresaDatosBancarios` ya filtran por región del usuario logueado (RE-FU-005 / RE-FU-001). No hay confirmación de que `catMoneda` tenga el mismo tratamiento — la tabla `catMoneda` no muestra columna `IdRegion` en el ER actual. Si falta, requiere el mismo patrón aplicado en RE-FU-005 (GAP-04): ALTER TABLE + `IdRegion` + Controller heredando `BaseApiController` con `AsegurarFiltroRegion`.

> ⚠