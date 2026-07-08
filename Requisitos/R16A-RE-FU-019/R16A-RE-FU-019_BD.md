# Impacto en BD - Factura por Adelantado: Detalle Mexico
**Requisito:** R16A-RE-FU-019
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (escritura)
**Version:** 2.0 - EmpresaFolio + migracion a fccFactura/vfccFactura (RE-FU-015) + correccion arquitectura CFDI (Finanzas, no Timbrado)

---

## Resumen
Pantalla de Detalle por cliente: generar, timbrar y enviar la factura PPD por cada pedido
pendiente. Post-envio: Credito transfiere a Legacy / Prepago genera pendiente Validar Cobro.
Solo clientes Mexico. Sin sustancias controladas.

> **Nota de arquitectura (correccion — el CFDI no va en Timbrado, va en Finanzas):** este requisito
> es el que **crea `CFDIGenerada`** (ER-Finanzas.md, propiedad de ProquifaDotNet.Finanzas) sobre la
> base de datos `ProquifaDotNet`. El paso "TIMBRAR" del flujo de datos se corrigio: Timbrado
> (`ProquifaDotNet.Timbrado`) es un servicio tecnico que consume el folio y llama al PAC, pero
> **no persiste el CFDI como entidad de negocio** — es Finanzas quien hace `INSERT CFDIGenerada`
> y quien pobla `fccFactura.IdCFDIGenerada` con el Id real de ese registro. Ver
> `R16A-RE-FU-018_BD.md` (Parte 2 y 3) para el detalle completo de esta correccion.

> **Migración (06/07/2026) — ya NO se crea `ALTER TABLE tpProformaAdelanto`/`CREATE VIEW vtpProformaAdelanto` aquí:**
> Este requisito originalmente creaba el `ALTER TABLE tpProformaAdelanto ADD Enviada` y la vista
> `vtpProformaAdelanto`. Ambos objetos se retiran: `fccFactura` (con las columnas `Enviada` e
> `IdCFDIGenerada`) y la vista `vfccFactura` se crean y son propiedad de **R16A-RE-FU-015**, que
> unifica el pendiente FAA de Crédito (RE-FU-012) y Prepago (RE-FU-015) en una sola tabla. Este
> requisito **consume** `fccFactura`/`vfccFactura`, no las crea. Ver `R16A-RE-FU-015_BD.md`,
> sección "Migración de tpProformaAdelanto".

---

## Impacto en BD

| #   | Cambio                                                             | Base de Datos          | Tipo | Prioridad |
| --- | ------------------------------------------------------------------ | ---------------------- | ---- | --------- |
| 1   | CREATE TABLE dbo.CFDIGenerada (ER-Finanzas.md, base)               | ProquifaDotNet         | DDL  | Alta      |
| 2   | CREATE TABLE dbo.EmpresaFolio                                      | ProquifaDotNet (Finanzas) | DDL  | Alta      |
| 3   | INSERT EmpresaFolio (4 empresas MEX)                               | ProquifaDotNet (Finanzas) | DML  | Alta      |
| -   | ~~ALTER TABLE tpProformaAdelanto ADD Enviada~~ / ~~CREATE VIEW vtpProformaAdelanto~~ | ProquifaDotNet | **Movido a RE-FU-015** (`fccFactura.Enviada` + `vfccFactura`) | - |

> **Nota (origen de CFDIGenerada):** este requisito ejecuta el `CREATE TABLE CFDIGenerada` (columnas base de `ER-Finanzas.md`, mas `Total` que ya consume `vfccFactura`). Los catalogos `IdCatTipoCFDI`/`IdCFDIRelacionado` se agregan despues via `ALTER TABLE` en R16A-RE-FU-028, y las columnas tecnicas (Estado, MensajeError, IdArchivoXml, IdCatUsoCFDI, IdCatMetodoDePagoCFDI, IdCatMoneda, TipoCambio) se agregan via `ALTER TABLE` en R16A-RE-FU-018 (Parte 3) — ambos posteriores a la creacion base aqui.
>
> **Orden de dependencia:** `fccFactura.IdCFDIGenerada` (RE-FU-015) requiere que `CFDIGenerada` exista — este requisito (paso 1 de esta tabla) debe ejecutarse antes de que RE-FU-015 pueda agregar esa FK.

---

## CREATE TABLE CFDIGenerada (ProquifaDotNet — propiedad de Finanzas)

**Proposito:** Registro central de negocio de todo CFDI emitido (diseñado en `ER-Finanzas.md`,
gestionado por `ProquifaDotNet.Finanzas` via EF Core Scaffold). Se crea aqui con las columnas
base; `IdCatTipoCFDI`/`IdCFDIRelacionado` se agregan en R16A-RE-FU-028 y las columnas tecnicas
de timbrado en R16A-RE-FU-018 (Parte 3) via `ALTER TABLE` posteriores.

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Ejecutar sobre ProquifaDotNet
    CREATE TABLE [dbo].[CFDIGenerada](
        [IdCFDIGenerada] uniqueidentifier NOT NULL
            CONSTRAINT [DF_CFDIGenerada_Id] DEFAULT (NEWID()),
        [RFCEmisor]    varchar(13)  NOT NULL,
        [RFCReceptor]  varchar(50)  NOT NULL,
        [Serie]        varchar(25)  NULL,
        [Folio]        varchar(40)  NULL,
        [FechaEmision] datetime2(7) NULL,
        [UUID]         varchar(36)  NULL,
        [Total]        decimal(18,2) NULL,
        [Activo]       bit NOT NULL
            CONSTRAINT [DF_CFDIGenerada_Activo] DEFAULT (1),
        [FechaRegistro] datetime2(7) NOT NULL
            CONSTRAINT [DF_CFDIGenerada_FechaRegistro] DEFAULT (SYSUTCDATETIME()),
        CONSTRAINT [PK_CFDIGenerada] PRIMARY KEY CLUSTERED ([IdCFDIGenerada])
    );
    GO

| Columna | Tipo | Nulo | Default | Descripcion |
|---------|------|------|---------|-------------|
| IdCFDIGenerada | uniqueidentifier | NO | NEWID() | PK |
| RFCEmisor | varchar(13) | NO | - | RFC de la empresa emisora |
| RFCReceptor | varchar(50) | NO | - | RFC/RUC del cliente receptor |
| Serie | varchar(25) | SI | - | Serie del CFDI |
| Folio | varchar(40) | SI | - | Folio del CFDI (consumido de EmpresaFolio) |
| FechaEmision | datetime2(7) | SI | - | Fecha de emision (timbrado) |
| UUID | varchar(36) | SI | - | UUID asignado por el PAC al timbrar |
| Total | decimal(18,2) | SI | - | Monto total del CFDI (usado por vfccFactura) |
| Activo | bit | NO | 1 | Activo |
| FechaRegistro | datetime2(7) | NO | SYSUTCDATETIME() | Fecha de creacion del registro |

> Ver R16A-RE-FU-028_BD.md (ALTER: IdCatTipoCFDI, IdCFDIRelacionado) y R16A-RE-FU-018_BD.md Parte 3 (ALTER: IdCatUsoCFDI, IdCatMetodoDePagoCFDI, IdCatMoneda, TipoCambio, IdArchivoXml, Estado, MensajeError, FechaUltimaActualizacion) para las extensiones posteriores de esta tabla.

---

## `fccFactura.Enviada` y vista `vfccFactura` — movidos a RE-FU-015 (06/07/2026)

> Este requisito creaba originalmente `ALTER TABLE tpProformaAdelanto ADD Enviada` y
> `CREATE VIEW dbo.vtpProformaAdelanto`. **Ambos objetos se retiraron de aquí**: la columna
> `Enviada` es ahora parte de `fccFactura` (creada en RE-FU-015) y la vista equivalente se
> llama `vfccFactura` (también en RE-FU-015). No se repite el DDL — ver el diccionario de
> datos y el `CREATE VIEW vfccFactura` completo en `R16A-RE-FU-015_BD.md`.
>
> **Diferencias relevantes para este requisito:**
> - `vfccFactura` ya NO necesita la cadena `fccPagoFacturaAdelanto → tpProformaPedido → tpPedidoProformaPedido → tpPedido` para llegar al pedido: `fccFactura.IdTPPedido` es FK directa.
> - Los datos del receptor (`ClienteRazonSocial`/`ClienteRFC`) ya no requieren JOIN a `DatosFacturacionCliente`: están fijados como snapshot en `fccFactura` (`RazonSocialReceptor`/`RfcReceptor`) desde que se generó el pendiente.
> - `EstadoFAA` se calcula con la misma fórmula (`IdCFDIGenerada IS NULL` → `PendienteGenerar`; `IdCFDIGenerada IS NOT NULL AND Enviada=0` → `PendienteEnviar`; si no, `Completada`).
> - `vfccFactura` cubre **ambos orígenes** (Prepago RE-FU-015 y Crédito RE-FU-012) en una sola vista — `vtpProformaAdelanto` solo cubría lo que existiera en `tpProformaAdelanto`.

---

## CREATE TABLE EmpresaFolio (ProquifaDotNet — propiedad Finanzas, movida de ProquifaDotNetTimbrado el 07/07/2026)

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Ejecutar en ProquifaDotNetTimbrado
    CREATE TABLE [dbo].[EmpresaFolio](
        [IdEmpresaFolio] uniqueidentifier NOT NULL
            CONSTRAINT [DF_EmpresaFolio_Id] DEFAULT (NEWID()),
        [EmpresaClave] varchar(10) NOT NULL,
        [EmpresaNombre] varchar(200) NOT NULL,
        [Serie] varchar(25) NULL,
        [UltimoFolio] int NOT NULL
            CONSTRAINT [DF_EmpresaFolio_UltimoFolio] DEFAULT (0),
        [FormatoFolio] varchar(50) NOT NULL
            CONSTRAINT [DF_EmpresaFolio_Formato] DEFAULT ('{folio}'),
        [LongitudMaxima] int NOT NULL
            CONSTRAINT [DF_EmpresaFolio_Longitud] DEFAULT (6),
        [CreatedAt] datetime2(7) NOT NULL
            CONSTRAINT [DF_EmpresaFolio_CreatedAt] DEFAULT (SYSUTCDATETIME()),
        [UpdatedAt] datetime2(7) NOT NULL
            CONSTRAINT [DF_EmpresaFolio_UpdatedAt] DEFAULT (SYSUTCDATETIME()),
        [IsActive] bit NOT NULL
            CONSTRAINT [DF_EmpresaFolio_IsActive] DEFAULT (1),
        CONSTRAINT [PK_EmpresaFolio] PRIMARY KEY CLUSTERED ([IdEmpresaFolio]),
        CONSTRAINT [UQ_EmpresaFolio_Clave] UNIQUE ([EmpresaClave])
    );
    GO

    INSERT INTO [dbo].[EmpresaFolio] ([EmpresaClave], [EmpresaNombre], [UltimoFolio])
    VALUES
        ('GOL', 'Golocaer S.A. de C.V.', 0),
        ('MUN', 'Mungen S.A. de C.V.', 0),
        ('PRO', 'Proquifa S.A. de C.V.', 0),
        ('PQF', 'Proveedora Quimico Farmaceutica S.A. de C.V.', 0);
    -- Ajustar UltimoFolio al MAX existente en produccion

### Consumo atomico del folio

    UPDATE [dbo].[EmpresaFolio] WITH (UPDLOCK, ROWLOCK)
    SET @NuevoFolio = UltimoFolio = UltimoFolio + 1,
        UpdatedAt = SYSUTCDATETIME()
    WHERE EmpresaClave = @EmpresaClave AND IsActive = 1;

> Se consume SOLO al timbrar exitosamente (sin huecos por errores PAC).

---

## Flujo de Datos (corregido — CFDI se persiste en Finanzas, Timbrado solo timbra)

    1. GENERAR (modal revision + previsualizacion)
       Lee: vfccFactura (RE-FU-015 — antes: tpPedido, tpProformaAdelanto, DatosFacturacionCliente, Empresa), catUsoCFDI

    2. TIMBRAR (al confirmar previsualizacion) -- CfdiController (ProquifaDotNet.Finanzas)
       ProquifaDotNet.Timbrado (servicio tecnico, POST /api/v1/stamp/invoice):
         UPDATE EmpresaFolio SET UltimoFolio+1 (consume el folio atomicamente antes de armar el CFDI)
         Arma el CFDI con ese folio -> llama PAC -> recibe UUID
         INSERT StampingLog (ProquifaDotNetTimbrado, auditoria tecnica de la llamada)
         Regresa a Finanzas: UUID, Serie, Folio, XML, FechaEmision (sin persistir el CFDI como negocio)
       ProquifaDotNet.Finanzas (CfdiService, tras respuesta exitosa de Timbrado):
         INSERT CFDIGenerada (ProquifaDotNet): UUID, Serie, Folio, FechaEmision, IdCatTipoCFDI, Total,
           IdCatUsoCFDI, IdCatMetodoDePagoCFDI, IdCatMoneda, TipoCambio, Estado='Timbrado'
         INSERT Archivo x2 (PDF+XML, FileBucket='facturas') + UPDATE CFDIGenerada SET IdArchivoXml
         UPDATE fccFactura SET IdCFDIGenerada = @IdCFDIGenerada, EsFacturaPorAdelantado = 0,
           IdCatFacturaEstado = GENERADA (catFacturaEstado, RE-FU-015 v2.1)
           (Id real de CFDIGenerada, no un Id de Timbrado; antes: UPDATE tpProformaAdelanto SET IdCFDIGenerada)

    3. ENVIAR (modal envio)
       ProquifaDotNet:
         INSERT CorreoEnviado + ArchivoCorreoEnviado
         UPDATE fccFactura SET Enviada = 1, FechaEnvio = SYSUTCDATETIME(), IdCatFacturaEstado = ENVIADA (antes: UPDATE tpProformaAdelanto SET Enviada = 1)
         Segun tipo (fccFactura.IdTPProformaPedido NOT NULL = origen Credito):
           Credito -> transferencia Legacy
           Prepago -> pendiente Validar Cobro

---

## Estados del Pedido (via vfccFactura)

| EstadoFAA | Condicion | Accion UI |
|-----------|-----------|-----------|
| PendienteGenerar | IdCFDIGenerada IS NULL | 'Generar Factura' (azul) |
| PendienteEnviar | IdCFDIGenerada IS NOT NULL AND Enviada=0 | 'Enviar Factura' (verde) |
| Completada | Enviada=1 | Desaparece del listado |

---

## Orden de Ejecucion de Scripts

| Paso | Script | BD |
|------|--------|-----|
| 1 | CREATE TABLE CFDIGenerada (base) | ProquifaDotNet |
| 2 | CREATE TABLE fccFactura + fccFacturaPartida + fccFacturaReferenciaBancaria + CREATE VIEW vfccFactura (RE-FU-015, requiere Paso 1) | ProquifaDotNet |
| 3 | CREATE TABLE EmpresaFolio | ProquifaDotNet (Finanzas) |
| 4 | INSERT EmpresaFolio (datos iniciales) | ProquifaDotNet (Finanzas) |

> Los pasos 2 (`fccFactura`/`vfccFactura`) se ejecutan como parte de RE-FU-015, no de este requisito — se listan aquí solo para dejar clara la dependencia de orden.

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | UltimoFolio inicial por empresa | Tecnico | Consultar MAX(folio) produccion |
| 2 | Lote del producto al timbrar FAA | Negocio | No disponible - confirmar |
| 3 | Politica ante caida del PAC | Tecnico | Definir reintento/encolamiento |
| 4 | Rol operativo | Negocio | Confirmar denominacion |
| 5 | Alias vs RazonSocial | Negocio | Confirmar dato fuente |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-018 | Pantalla inicial + creacion ProquifaDotNet.Timbrado (servicio tecnico) + CfdiController/extension CFDIGenerada en Finanzas |
| R16A-RE-FU-012 | Genera pendiente FAA (`fccFactura`, origen Credito) |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria`/`vfccFactura` (incluye columnas `Enviada`/`IdCFDIGenerada` migradas desde este requisito) — genera pendiente FAA de Prepago |
| R16A-RE-FU-016 | Patron persistencia Minio |

> Este requisito (RE-FU-019) es el que ejecuta el `CREATE TABLE CFDIGenerada` (ER-Finanzas.md) referenciado como prerequisito por R16A-RE-FU-018_BD.md (Parte 3, extension de columnas) y R16A-RE-FU-028 (catTipoCFDI).

---

**Generado por:** GitHub Copilot in SSMS
**Bases de Datos:** ProquifaDotNet + ProquifaDotNetTimbrado
