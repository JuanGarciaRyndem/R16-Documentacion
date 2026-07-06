# Impacto en BD - Factura por Adelantado: Detalle Peru
**Requisito:** R16A-RE-FU-020
**Bases de Datos:** ProquifaDotNet (lectura/escritura) + ProquifaDotNetTimbrado (escritura)
**Version:** 1.0

---

## Resumen
Detalle por cliente para emitir CPE tipo 01 (Factura) ante SUNAT para pedidos Prepago Peru.
Solo empresa GOLPERU (Golocaer S.A.C.). Sin Metodo/Forma de Pago ni Complemento de Pago.
IGV 18%, RUC, serie alfanumerica SUNAT. Sin transferencia a Legacy. Post-envio: Validar Cobro.

---

## Impacto en BD

| #   | Cambio                                              | Base de Datos          | Tipo      | Prioridad      |
| --- | --------------------------------------------------- | ---------------------- | --------- | -------------- |
| 1   | INSERT EmpresaFolio GOLPERU (serie SUNAT)           | ProquifaDotNetTimbrado | DML       | Alta           |
| 2   | ALTER TABLE Producto ADD campos SUNAT               | ProquifaDotNet         | DDL       | **BLOQUEANTE** |
| 3   | UPDATE Empresa GOLPERU datos legales (RUC, ubigeo)  | ProquifaDotNet         | DML       | Alta (BRECHA)  |
| 4   | Reutiliza: tpProformaAdelanto.Enviada (RE-FU-019)   | ProquifaDotNet         | Existente | -              |
| 5   | Reutiliza: vtpProformaAdelanto (RE-FU-019)          | ProquifaDotNet         | Existente | -              |
| 6   | Reutiliza: EmpresaFolio GOL/MUN/PRO/PQF (RE-FU-019) | ProquifaDotNetTimbrado | Existente | -              |

> La infraestructura BD creada en RE-FU-018/019 se reutiliza completa.
> Los cambios nuevos son: fila GOLPERU en EmpresaFolio + datos SUNAT del producto.

---

## Diferencias BD Mexico (RE-FU-019) vs Peru (RE-FU-020)

| Aspecto | Mexico RE-FU-019 | Peru RE-FU-020 |
|---------|-----------------|----------------|
| Empresa emisora | GOL/MUN/PRO/PQF (4) | GOLPERU (1) |
| ID fiscal cliente | DatosFacturacionCliente.RFC (13 chars) | DatosFacturacionCliente.RFC (RUC 11 digits) |
| Codigo interbancario | DatosBancarios.Clabe (18 digits) | DatosBancarios.Clabe (CCI 20 digits) |
| Impuesto | IVA (tasa variable) | IGV 18% (fijo) |
| Folio/Serie | varchar(6) numerico por empresa | Serie 4 chars alfanumerica + correlativo 8 digits |
| Metodo/Forma Pago | PPD + 99 (forzados SAT) | **NO existen en SUNAT** |
| Complemento de Pago | Generado en Validar Cobro | **NO existe en Perú** |
| Tipo equivalente UsoCFDI | catUsoCFDI SAT | Tipo Operacion catalogo 51 SUNAT |
| Datos SUNAT producto | No requeridos (CFDI) | **Codigo SUNAT + UdM + Afectacion IGV (BRECHA)** |
| Transferencia Legacy | SI (Credito) | **NO** |
| Salida operativa | Legacy (Credito) o Validar Cobro (Prepago) | Siempre Validar Cobro (solo Prepago) |

---

## Brecha Bloqueante: Datos Fiscales SUNAT del Producto

| Campo SUNAT obligatorio en UBL 2.1 | Existe en BD? | Tabla actual |
|------------------------------------|--------------|--------------|
| Codigo SUNAT del producto (catalogo 25 SUNAT) | **NO** | - |
| Unidad de medida SUNAT (catalogo 6 SUNAT) | **NO** | catUnidad (no mapeada a SUNAT) |
| Afectacion al IGV por linea (catalogo 7 SUNAT) | **NO** | - |

**Script propuesto (sujeto a confirmar con equipo tecnico):**

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Agregar datos fiscales SUNAT al catalogo de productos

    ALTER TABLE dbo.Producto
        ADD CodigoSUNAT varchar(10) NULL;   -- catalogo 25 SUNAT

    ALTER TABLE dbo.catUnidad
        ADD ClaveSUNAT varchar(10) NULL;    -- catalogo 6 SUNAT (ej: KGM, NIU, ZZ)

    -- Tabla nueva para afectacion IGV por familia/producto
    CREATE TABLE [dbo].[catAfectacionIGV](
        [IdCatAfectacionIGV] uniqueidentifier NOT NULL CONSTRAINT [DF_catAfectacionIGV_Id] DEFAULT (NEWID()),
        [Clave] varchar(4) NOT NULL,          -- catalogo 7 SUNAT: 10, 11, 20, 30, 40...
        [Descripcion] varchar(200) NOT NULL,
        [Activo] bit NOT NULL CONSTRAINT [DF_catAfectacionIGV_Activo] DEFAULT (1),
        CONSTRAINT [PK_catAfectacionIGV] PRIMARY KEY CLUSTERED ([IdCatAfectacionIGV]),
        CONSTRAINT [UQ_catAfectacionIGV_Clave] UNIQUE ([Clave])
    );
    GO

    ALTER TABLE dbo.Producto
        ADD IdCatAfectacionIGV uniqueidentifier NULL
            CONSTRAINT [FK_Producto_AfectacionIGV]
                FOREIGN KEY REFERENCES dbo.catAfectacionIGV([IdCatAfectacionIGV]);

> **SIN estos datos no es posible generar el XML UBL 2.1 para SUNAT.**
> Bloquea el desarrollo de facturacion Peru (RE-FU-005 Brecha 1).

---

## EmpresaFolio GOLPERU (ProquifaDotNetTimbrado)

**Diferencia clave vs México:** La serie SUNAT es alfanumérica de 4 chars (F001) + correlativo de hasta 8 dígitos.

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Ejecutar en ProquifaDotNetTimbrado

    -- GOLPERU: serie SUNAT pendiente de confirmar con Golocaer SAC
    INSERT INTO [dbo].[EmpresaFolio]
        ([EmpresaClave], [EmpresaNombre], [Serie], [UltimoFolio], [FormatoFolio], [LongitudMaxima])
    VALUES
        ('GOLPERU', 'Golocaer S.A.C.', 'F001', 0, 'F001-{folio:00000000}', 8);
    -- Ajustar Serie y UltimoFolio al ultimo usado en produccion

| Campo | Valor Mexico | Valor Peru |
|-------|-------------|------------|
| EmpresaClave | GOL/MUN/PRO/PQF | GOLPERU |
| Serie | NULL | F001 (o la que use Golocaer SAC) |
| UltimoFolio | numero (ej: 1500) | correlativo numerico |
| FormatoFolio | '{folio}' | 'F001-{folio:00000000}' |
| LongitudMaxima | 6 | 8 |

---

## Datos Legales Golocaer SAC (Empresa - GOLPERU)

| Dato | Existe en BD? | Accion |
|------|--------------|--------|
| RUC emisor | NO (falta RUC SUNAT) | Recopilar y UPDATE Empresa |
| Domicilio fiscal Peru | NO | Recopilar y UPDATE |
| Ubigeo SUNAT | NO | Recopilar y UPDATE |
| Certificado digital (PEM/PFX) | NO | Configurar en AppSetting ProquifaDotNetTimbrado |
| Serie(s) de facturacion | NO confirmadas | Confirmar con Golocaer SAC |

> Hasta que estos datos no esten en BD, el timbrado SUNAT no puede ejecutarse.

---

## Reutilizacion de RE-FU-018/019

| Objeto | BD | Creado en | Reutilizacion |
|--------|-----|-----------|---------------|
| tpProformaAdelanto.Enviada | ProquifaDotNet | RE-FU-019 | Mismo campo para PER |
| vtpProformaAdelanto | ProquifaDotNet | RE-FU-019 | Ya filtra por Region |
| EmpresaFolio (tabla) | ProquifaDotNetTimbrado | RE-FU-018 | Solo agregar fila GOLPERU |
| CFDI (tabla) | ProquifaDotNetTimbrado | RE-FU-018 | Misma tabla, diferente XML |
| StampingLog (tabla) | ProquifaDotNetTimbrado | RE-FU-018 | Misma tabla |
| AppSetting | ProquifaDotNetTimbrado | RE-FU-018 | Agregar config OSE/PSE Peru |

---

## Tablas Leidas (ProquifaDotNet)

| Tabla | Datos leidos | Diferencia vs MEX |
|-------|-------------|-------------------|
| tpPedido | FolioPedidoInterno, IdEmpresa (GOLPERU), IdCliente | Igual |
| tpProformaAdelanto | Monto, IdCFDIGenerada, Enviada | Igual |
| Cliente + DatosFacturacionCliente | RFC (RUC Peru), RazonSocial | RUC vs RFC |
| Empresa (GOLPERU) | Prefijo, RazonSocial, RUC emisor | Solo GOLPERU |
| catCondicionesDePago | CondicionesDePago (Contado/Credito) | Sin PPD/99 |
| catMoneda | PEN/USD | PEN vs MXN |
| **Producto.CodigoSUNAT** | **Codigo SUNAT del producto** | **BRECHA - no existe** |
| **catUnidad.ClaveSUNAT** | **Unidad medida SUNAT** | **BRECHA - no existe** |
| **catAfectacionIGV** | **Afectacion IGV por linea** | **BRECHA - no existe** |
| ClienteCarteraCliente + ClienteCartera | Filtro por cobrador | Igual |

---

## Flujo de Datos

    1. GENERAR (modal revision)
       Lee: tpPedido, tpProformaAdelanto, DatosFacturacionCliente (RUC),
             Empresa (GOLPERU), Producto.CodigoSUNAT (BRECHA)

    2. TIMBRAR (al confirmar previsualizacion)
       ProquifaDotNetTimbrado:
         INSERT CFDI -> llama OSE/PSE SUNAT -> UPDATE CFDI (CDR de aceptacion)
         UPDATE EmpresaFolio GOLPERU SET UltimoFolio+1
         INSERT StampingLog
       ProquifaDotNet:
         UPDATE tpProformaAdelanto SET IdCFDIGenerada
         INSERT Archivo x2 (PDF+XML, FileBucket='facturas', IdRegion=PER)

    3. ENVIAR (modal envio)
       ProquifaDotNet:
         INSERT CorreoEnviado + ArchivoCorreoEnviado
         UPDATE tpProformaAdelanto SET Enviada=1
         Genera pendiente Validar Cobro (SOLO Prepago, sin rama Credito)
         ** SIN transferencia a Legacy **

---

## AppSetting ProquifaDotNetTimbrado (Peru)

    INSERT INTO [dbo].[AppSetting] ([Name], [Value], [Description])
    VALUES
        ('SUNAT_OSE_Endpoint', 'https://ose.pendiente.com/api', 'Endpoint OSE/PSE Peru - PENDIENTE'),
        ('SUNAT_RUC_Emisor', '20XXXXXXXXXX', 'RUC Golocaer SAC - PENDIENTE'),
        ('SUNAT_CertPath', '/secrets/golperu_cert.pfx', 'Certificado digital Golocaer SAC');

---

## Estados del Pedido (via vtpProformaAdelanto - igual que MEX)

| EstadoFAA | Condicion | Accion UI |
|-----------|-----------|-----------|
| PendienteGenerar | IdCFDIGenerada IS NULL | 'Generar Factura' (azul) |
| PendienteEnviar | IdCFDIGenerada IS NOT NULL AND Enviada=0 | 'Enviar Factura' (verde) |
| Completada | Enviada=1 | Desaparece del listado |

---

## Orden de Ejecucion de Scripts

| Paso | Script | BD | Prerequisito |
|------|--------|-----|--------------|
| 1 | ALTER TABLE Producto ADD CodigoSUNAT | ProquifaDotNet | Ninguno |
| 2 | ALTER TABLE catUnidad ADD ClaveSUNAT | ProquifaDotNet | Ninguno |
| 3 | CREATE TABLE catAfectacionIGV | ProquifaDotNet | Ninguno |
| 4 | ALTER TABLE Producto ADD IdCatAfectacionIGV | ProquifaDotNet | Paso 3 |
| 5 | UPDATE Empresa SET RUC... WHERE Prefijo='GOLPERU' | ProquifaDotNet | Datos Golocaer SAC |
| 6 | INSERT EmpresaFolio GOLPERU | ProquifaDotNetTimbrado | RE-FU-018 ejecutado |
| 7 | INSERT AppSetting OSE/PSE Peru | ProquifaDotNetTimbrado | Datos OSE definidos |

---

## Brechas Criticas

| # | Brecha | Tipo | Bloqueante | Accion |
|---|--------|------|-----------|--------|
| B1 | Datos fiscales SUNAT del producto (CodigoSUNAT, UdM, AfectacionIGV) | BD+Catalogo | **SI** | ALTER TABLE + poblacion del catalogo |
| B2 | Integracion OSE/PSE SUNAT no definida | Tecnico | **SI** | Definir proveedor timbrado Peru |
| B3 | RUC/ubigeo/certificado Golocaer SAC no disponibles | Datos | SI | Recopilar con Golocaer SAC |
| B4 | Serie SUNAT de Golocaer SAC no confirmada | Datos | SI | Confirmar con Golocaer SAC |
| B5 | Tipo de Operacion cat.51: configurable vs fijo | Negocio | No | Confirmar con cliente |
| B6 | Controlados Peru: excluir o incluir en FAA | Negocio | No | Confirmar con cliente |
| B7 | Secuencia GRE<->Factura en Prepago | Negocio | No | Confirmar con cliente |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-018 | Pantalla inicial + ProquifaDotNetTimbrado |
| R16A-RE-FU-019 | Detalle Mexico (infraestructura compartida) |
| R16A-RE-FU-017 | PDF Proforma Peru (patron persistencia Minio PER) |
| R16A-RE-FU-005 | Brechas Perú (B1 bloqueante, B5 timbrado) |

---

**Generado por:** GitHub Copilot in SSMS
**Bases de Datos:** ProquifaDotNet + ProquifaDotNetTimbrado
