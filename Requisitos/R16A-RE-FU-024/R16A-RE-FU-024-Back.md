# Impacto en Back — R16A-RE-FU-024
**Requisito:** Validar Cobro: Paso 1 México — Captura del Cobro
**Aplicativos:** ProquifaDotNet (.NET Framework 4.8) + ProquifaDotNet.Finanzas (.NET Core 10)
**Módulo:** Validar Cobro — Wizard Paso 1 (México)
**Impacto:** Scripts BD ProquifaDotNet (ALTER fccPagoCliente x5 + CREATE catTipoInconsistenciaCobro + CREATE fccInconsistenciaCobro + CREATE SEQUENCE SeqFolioCobro) + Endpoints Finanzas: listado cobros, detalle correo, captura/autoguardado/confirmación cobro, foliador COB, TC del día, modal inconsistencia + llamadas entre APIs (Finanzas → ProquifaDotNet)

---

## Resumen

Este requisito implementa la **primera pantalla del wizard de Validar Cobro (Paso 1 - Captura del Cobro) para Región México** en ProquifaDotNet.Finanzas. El usuario revisa los correos del Buzón del cliente, selecciona el comprobante de pago adjunto y captura los datos del cobro (folio, monto, fecha, forma de pago SAT, cuenta origen/destino, **moneda via combo catMoneda**, TC del día). Un cobro confirmado es inmutable. Permite capturar múltiples cobros en la misma sesión con auto-guardado transparente.

> **OBS-048 — Reanudación del wizard en el último paso activo:** Al ingresar al wizard para un cliente, Finanzas evalúa el estado actual del flujo (via `GET /api/validar-cobro/clientes/{idCliente}/estado-wizard`) y redirige al último paso activo donde el usuario se encontraba, no necesariamente al Paso 1. Si el Paso 1 ya tiene cobros confirmados pero el Paso 2 está pendiente, el wizard abre en el Paso 2 directamente.

El impacto en BD (ProquifaDotNet) es **moderado**: 5 ALTER en `fccPagoCliente` (inmutabilidad + notas + **IdCatMoneda FK**) + 2 tablas nuevas + 1 SEQUENCE. El impacto en servicios (Finanzas) es **alto**: orquestación completa del Paso 1.

### Distribución de responsabilidades

| Capa | Aplicativo | Responsabilidad |
|------|-----------|----------------|
| BD | ProquifaDotNet | `fccPagoCliente`, `catTipoInconsistenciaCobro`, `fccInconsistenciaCobro`, `SeqFolioCobro` |
| API Datos | ProquifaDotNet | Expone endpoints de Buzón (correos, adjuntos), cobros, catálogos (moneda, medio de pago, cuentas), inconsistencias |
| Lógica Paso 1 | ProquifaDotNet.Finanzas | Orquesta el Paso 1: listado cobros, captura, auto-guardado, confirmación, TC, inconsistencias |
| TC del día | ProquifaDotNet.Finanzas | Calcula TC FIX Banxico/DOF (fuente pendiente de confirmar) para la moneda no-MXN involucrada |
| Comunicación | Finanzas → ProquifaDotNet | Llamadas entre APIs para leer Buzón y escribir cobros e inconsistencias |

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

### A1 — ALTER TABLE fccPagoCliente (5 campos nuevos)

```sql
-- Created by GitHub Copilot in SSMS - review carefully before executing

-- 1. Inmutabilidad: 0=borrador / 1=confirmado
ALTER TABLE dbo.fccPagoCliente
    ADD Confirmado bit NOT NULL CONSTRAINT [DF_fccPagoCliente_Confirmado] DEFAULT (0);

-- 2. Timestamp de confirmación
ALTER TABLE dbo.fccPagoCliente
    ADD FechaConfirmacion datetime2 NULL;

-- 3. Trazabilidad: quién confirmó el cobro
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
```

> **Motivo de IdCatMoneda:** La pantalla del Paso 1 muestra "Moneda*" como combo desplegable
> (ej. "USD" seleccionado). Los flags `MXN`/`USD` bit no soportan combos ni monedas adicionales
> como PEN (Perú). Se agrega `IdCatMoneda FK catMoneda` nullable para el combo de la UI.

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
    [FechaRegistro]                datetime2(7) NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_FechaReg] DEFAULT (SYSUTCDATETIME()),
    [FechaUltimaActualizacion]     datetime2(7) NOT NULL
        CONSTRAINT [DF_fccInconsistenciaCobro_FechaUpd] DEFAULT (SYSUTCDATETIME()),
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

| Estado item | Muestra | Ordenamiento |
|-------------|---------|-------------|
| Capturado (`Confirmado=1`) | Folio COB-mmddaa-NNNN, fecha, monto + moneda | Por `FechaPago ASC` |
| Sin capturar | Etiqueta temporal "COB-N" | Por `FechaRecepcion ASC` del correo |

---

### B2 — Detalle del correo y adjuntos (selección del comprobante)

**Descripción:** Endpoint que retorna el detalle del correo seleccionado (asunto, cuerpo, fecha, hora, contacto del cliente) y la lista de adjuntos como opciones de radio button para seleccionar el comprobante.

**Datos (vía API ProquifaDotNet):** `CorreoRecibido`, `ArchivoCorreoRecibido`, `ContactoCliente`, `Archivo`

---

### B3 — Catálogos del formulario del Paso 1

**Descripción:** Endpoints de catálogos para poblar los combos del formulario de captura del cobro.

| Endpoint | Catálogo | Filtro |
|----------|---------|--------|
| `GET /api/validar-cobro/catalogos/monedas` | `catMoneda` | `Activo=1` |
| `GET /api/validar-cobro/catalogos/medios-pago?region=MEX` | `catMedioDePago` | `ClaveFormaDePago IS NOT NULL` (SAT) |
| `GET /api/validar-cobro/catalogos/cuentas-destino?region=MEX` | `EmpresaDatosBancarios + DatosBancarios` | Empresas GOL/MUN/PRO/PQF, región MEX |

> El combo "Moneda" carga desde `catMoneda` y se persiste como `IdCatMoneda` en `fccPagoCliente`.

---

### B4 — Auto-guardado del cobro en borrador

**Descripción:** Endpoint `PUT` en Finanzas que persiste el estado del formulario como borrador (`Confirmado=0`) de forma transparente. Incluye `IdCatMoneda` seleccionada.

**Operación (vía API ProquifaDotNet):**
- Si no existe `fccPagoCliente`: `INSERT` con `Confirmado=0`, `Folio=NULL`, `IdCatMoneda=@IdMoneda`
- Si existe con `Confirmado=0`: `UPDATE` con todos los datos del formulario incluido `IdCatMoneda`
- Guardia: si `Confirmado=1`, no sobreescribe

---

### B5 — Confirmación del cobro (inmutabilidad + folio COB)

**Descripción:** Endpoint `POST` en Finanzas que confirma el cobro. Valida selección de comprobante, genera folio COB con `SeqFolioCobro`, y hace el cobro inmutable con `Confirmado=1`.

**Flujo antes de llamar a ProquifaDotNet:**
1. Validar comprobante seleccionado (adjunto marcado como oficial)
2. Validar campos obligatorios completos (incluido `IdCatMoneda`)
3. Calcular folio: `'COB-' + FORMAT(FechaPago,'MMddyy') + '-' + LPAD(NEXT VALUE FOR SeqFolioCobro, 6, '0')`
4. Mostrar alerta al Front (los datos no podrán modificarse)

**Operación (vía API ProquifaDotNet):**
```sql
UPDATE dbo.fccPagoCliente
SET Folio                 = @FolioGenerado,
    Confirmado            = 1,
    FechaConfirmacion     = SYSUTCDATETIME(),
    IdUsuarioConfirmacion = @IdUsuarioActivo,
    IdCatMoneda           = @IdCatMoneda
WHERE IdFCCPagoCliente    = @Id
  AND Confirmado          = 0;  -- guardia de inmutabilidad
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
- `GET /api/validar-cobro/inconsistencias/tipos?paso=1` → catálogo filtrado (solo tipos del Paso 1)
- `POST /api/validar-cobro/clientes/{idCliente}/cobros/{idFCCPagoCliente}/inconsistencias` → INSERT en `fccInconsistenciaCobro`

---

### B8 — Estado del wizard para reanudación (OBS-048)

**Descripción:** Endpoint en Finanzas que retorna el paso activo actual del wizard para un cliente, permitiendo que la UI rediriga al último paso donde el usuario se encontraba en lugar de siempre abrir en el Paso 1.

**Endpoint:** `GET /api/validar-cobro/clientes/{idCliente}/estado-wizard`

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
  "pasoActivo": 1,
  "descripcion": "Captura del Cobro",
  "tieneBorador": true,
  "idFCCPagoClienteBorrador": "guid-opcional"
}
```

> Esta lógica se expande en los requisitos de Paso 2 (RE-FU-025) y Paso 3. La regla base: el paso activo es el primer paso que aún tiene trabajo pendiente. (OBS-048)

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