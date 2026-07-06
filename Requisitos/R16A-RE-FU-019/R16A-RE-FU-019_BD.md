# Impacto en BD - Factura por Adelantado: Detalle Mexico
**Requisito:** R16A-RE-FU-019
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (escritura)
**Version:** 1.2 - EmpresaFolio + Enviada + vtpProformaAdelanto

---

## Resumen
Pantalla de Detalle por cliente: generar, timbrar y enviar la factura PPD por cada pedido
pendiente. Post-envio: Credito transfiere a Legacy / Prepago genera pendiente Validar Cobro.
Solo clientes Mexico. Sin sustancias controladas.

---

## Impacto en BD

| # | Cambio | Base de Datos | Tipo | Prioridad |
|---|--------|---------------|------|-----------|
| 1 | ALTER TABLE tpProformaAdelanto ADD Enviada bit NOT NULL DEFAULT(0) | ProquifaDotNet | DDL | Alta |
| 2 | CREATE VIEW dbo.vtpProformaAdelanto | ProquifaDotNet | DDL | Alta |
| 3 | CREATE TABLE dbo.EmpresaFolio | ProquifaDotNetTimbrado | DDL | Alta |
| 4 | INSERT EmpresaFolio (4 empresas MEX) | ProquifaDotNetTimbrado | DML | Alta |

---

## ALTER TABLE tpProformaAdelanto

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    ALTER TABLE dbo.tpProformaAdelanto
        ADD Enviada bit NOT NULL CONSTRAINT [DF_tpProformaAdelanto_Enviada] DEFAULT (0);

| Campo nuevo | Tipo | Default | Descripcion |
|-------------|------|---------|-------------|
| Enviada | bit NOT NULL | 0 | 0=No enviada / 1=Enviada al cliente |

> Se pone en 1 al confirmar envio exitoso del correo con PDF+XML.
> Permite calcular EstadoFAA sin JOINs complejos a CorreoEnviado.

---

## CREATE VIEW dbo.vtpProformaAdelanto

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Ejecutar DESPUES del ALTER TABLE (Enviada debe existir)
    CREATE VIEW dbo.vtpProformaAdelanto
    AS
    SELECT
        pa.IdTPProformaAdelanto,
        pa.Monto,
        pa.MXN,
        pa.USD,
        pa.TipoDeCambio,
        pa.IdCliente,
        c.Nombre                    AS ClienteNombre,
        dfc.RazonSocial             AS ClienteRazonSocial,
        dfc.RFC                     AS ClienteRFC,
        pa.IdEmpresa,
        e.Prefijo                   AS EmpresaPrefijo,
        e.Alias                     AS EmpresaAlias,
        pa.NumeroOrdenDeCompra,
        pa.IdCFDIGenerada,
        cg.Folio                    AS FolioFactura,
        cg.Serie                    AS SerieFactura,
        cg.FechaEmision             AS FechaEmisionFactura,
        cg.Total                    AS TotalFactura,
        pa.IdCFDI,
        pa.Enviada,
        pa.FechaRegistro,
        pa.FechaUltimaActualizacion,
        pa.Activo,
        -- Estado calculado
        CASE
            WHEN pa.IdCFDIGenerada IS NULL THEN 'PendienteGenerar'
            WHEN pa.IdCFDIGenerada IS NOT NULL AND pa.Enviada = 0 THEN 'PendienteEnviar'
            ELSE 'Completada'
        END                         AS EstadoFAA,
        -- Pedido vinculado
        tp.IdTPPedido,
        tp.FolioPedidoInterno,
        tp.FechaTramitacion,
        tp.FacturaPorAdelantado,
        tp.IdRegion,
        r.Nombre                    AS Region,
        r.ClaveISO                  AS RegionClave,
        tp.IdCatCondicionesDePago,
        cdp.CondicionesDePago,
        cdp.SinCredito              AS EsPrepago
    FROM dbo.tpProformaAdelanto pa
    LEFT JOIN dbo.Cliente c                     ON pa.IdCliente = c.IdCliente
    LEFT JOIN dbo.DatosFacturacionCliente dfc   ON pa.IdCliente = dfc.IdCliente AND dfc.Activo = 1
    LEFT JOIN dbo.Empresa e                     ON pa.IdEmpresa = e.IdEmpresa
    LEFT JOIN dbo.CFDIGenerada cg               ON pa.IdCFDIGenerada = cg.IdCFDIGenerada
    LEFT JOIN dbo.fccPagoFacturaAdelanto fpfa   ON fpfa.IdTPProformaAdelanto = pa.IdTPProformaAdelanto
                                                 AND fpfa.Activo = 1
    LEFT JOIN dbo.tpProformaPedido pp           ON fpfa.IdTPProformaPedido = pp.IdTPProformaPedido
    LEFT JOIN dbo.tpPedidoProformaPedido tpp    ON pp.IdTPProformaPedido = tpp.IdTPProformaPedido
                                                 AND tpp.Activo = 1
    LEFT JOIN dbo.tpPedido tp                   ON tpp.IdTPPedido = tp.IdTPPedido
    LEFT JOIN dbo.Region r                      ON tp.IdRegion = r.IdRegion
    LEFT JOIN dbo.catCondicionesDePago cdp      ON tp.IdCatCondicionesDePago = cdp.IdCatCondicionesDePago;

---

## Columnas de la Vista vtpProformaAdelanto

| Columna | Origen | Descripcion |
|---------|--------|-------------|
| IdTPProformaAdelanto | tpProformaAdelanto | PK |
| Monto | tpProformaAdelanto | Monto del pendiente FAA |
| MXN / USD | tpProformaAdelanto | Moneda del monto |
| ClienteNombre | Cliente.Nombre | Nombre del cliente |
| ClienteRazonSocial | DatosFacturacionCliente.RazonSocial | Razon social fiscal |
| ClienteRFC | DatosFacturacionCliente.RFC | RFC (MEX) o RUC (PER) |
| EmpresaPrefijo | Empresa.Prefijo | GOL/MUN/PRO/PQF |
| EmpresaAlias | Empresa.Alias | Nombre corto empresa |
| IdCFDIGenerada | tpProformaAdelanto | FK a CFDIGenerada (NULL=no timbrada) |
| FolioFactura | CFDIGenerada.Folio | Folio de la factura timbrada |
| SerieFactura | CFDIGenerada.Serie | Serie CFDI |
| FechaEmisionFactura | CFDIGenerada.FechaEmision | Fecha timbrado |
| TotalFactura | CFDIGenerada.Total | Total facturado |
| Enviada | tpProformaAdelanto | 0=No enviada / 1=Enviada |
| **EstadoFAA** | **Calculado** | **PendienteGenerar / PendienteEnviar / Completada** |
| IdTPPedido | tpPedido (via cadena FKs) | Pedido vinculado |
| FolioPedidoInterno | tpPedido | Folio interno del pedido |
| Region / RegionClave | Region | MEX / PER |
| CondicionesDePago | catCondicionesDePago | Texto (PREPAGO 100%, 30 DIAS, etc.) |
| EsPrepago | catCondicionesDePago.SinCredito | 1=Prepago / 0=Credito |

---

## Cadena de JOINs de la Vista

    tpProformaAdelanto
        LEFT JOIN Cliente (IdCliente)
        LEFT JOIN DatosFacturacionCliente (IdCliente, Activo=1)
        LEFT JOIN Empresa (IdEmpresa)
        LEFT JOIN CFDIGenerada (IdCFDIGenerada) -- datos factura timbrada
        LEFT JOIN fccPagoFacturaAdelanto (IdTPProformaAdelanto, Activo=1)
            LEFT JOIN tpProformaPedido (IdTPProformaPedido)
                LEFT JOIN tpPedidoProformaPedido (IdTPProformaPedido, Activo=1)
                    LEFT JOIN tpPedido (IdTPPedido)
                        LEFT JOIN Region (IdRegion)
                        LEFT JOIN catCondicionesDePago (IdCatCondicionesDePago)

---

## CREATE TABLE EmpresaFolio (ProquifaDotNetTimbrado)

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

## Flujo de Datos

    1. GENERAR (modal revision + previsualizacion)
       Lee: tpPedido, tpProformaAdelanto, DatosFacturacionCliente, Empresa, catUsoCFDI

    2. TIMBRAR (al confirmar previsualizacion)
       ProquifaDotNetTimbrado:
         INSERT CFDI -> llama PAC -> UPDATE CFDI (UUID, Folio)
         UPDATE EmpresaFolio SET UltimoFolio+1
         INSERT StampingLog
       ProquifaDotNet:
         UPDATE tpProformaAdelanto SET IdCFDIGenerada = @id
         INSERT Archivo x2 (PDF+XML, FileBucket='facturas')

    3. ENVIAR (modal envio)
       ProquifaDotNet:
         INSERT CorreoEnviado + ArchivoCorreoEnviado
         UPDATE tpProformaAdelanto SET Enviada = 1
         Segun tipo:
           Credito -> transferencia Legacy
           Prepago -> pendiente Validar Cobro

---

## Estados del Pedido (via vista)

| EstadoFAA | Condicion | Accion UI |
|-----------|-----------|-----------|
| PendienteGenerar | IdCFDIGenerada IS NULL | 'Generar Factura' (azul) |
| PendienteEnviar | IdCFDIGenerada IS NOT NULL AND Enviada=0 | 'Enviar Factura' (verde) |
| Completada | Enviada=1 | Desaparece del listado |

---

## Orden de Ejecucion de Scripts

| Paso | Script | BD |
|------|--------|-----|
| 1 | ALTER TABLE tpProformaAdelanto ADD Enviada | ProquifaDotNet |
| 2 | CREATE VIEW vtpProformaAdelanto | ProquifaDotNet |
| 3 | CREATE TABLE EmpresaFolio | ProquifaDotNetTimbrado |
| 4 | INSERT EmpresaFolio (datos iniciales) | ProquifaDotNetTimbrado |

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
| R16A-RE-FU-018 | Pantalla inicial + creacion ProquifaDotNetTimbrado |
| R16A-RE-FU-012 | Genera pendiente FAA desde Credito con FAA |
| R16A-RE-FU-015 | Genera pendiente FAA desde Prepago con FAA |
| R16A-RE-FU-016 | Patron persistencia Minio |

---

**Generado por:** GitHub Copilot in SSMS
**Bases de Datos:** ProquifaDotNet + ProquifaDotNetTimbrado
