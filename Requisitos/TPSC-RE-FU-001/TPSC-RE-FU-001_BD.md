# Diccionario de Datos - Catalogo de Cuentas Bancarias PROQUIFA
**Requisito:** TPSC-RE-FU-001
**Base de Datos:** ProquifaDotNet

---

## Resumen Ejecutivo
Implementacion del Catalogo de Cuentas Bancarias del grupo PROQUIFA como origen unico
de informacion para modulos de cobro y pago.

### Empresas en Alcance R16
- GOL - Golocaer
- MUN - Mungen
- PRO - Proquifa
- PQF - Proveedora Quimico Farmaceutica

> Fuera de alcance: GOLPERU (Golocaer S.A.C. - Peru)

---

## Entidades Afectadas

| Objeto                          | Tipo     | Impacto                                               | Incluido en Alcance R16 |
| ------------------------------- | -------- | ----------------------------------------------------- | ----------------------- |
| EmpresaDatosBancarios           | Tabla    | Principal - catalogo de cuentas                       | SI                      |
| DatosBancarios                  | Tabla    | Detalle de cuenta bancaria                            | SI                      |
| Empresa                         | Tabla    | Empresas emisoras del grupo                           | SI                      |
| CuentaEmpresa                   | Tabla    | Vincula cuenta contable con datos bancarios           | SI                      |
| catBanco                        | Catalogo | Instituciones bancarias                               | SI                      |
| catMoneda                       | Catalogo | Monedas MXN/USD                                       | SI                      |
| catMedioDePago                  | Catalogo | Formas de pago                                        | SI                      |
| fcppOrdenDePago                 | Tabla    | Consumidor directo - usa EmpresaDatosBancarios        | SI                      |
| fppEjecucionOrdenDePago         | Tabla    | Consumidor directo - ejecuta pagos contra catalogo    | SI                      |
| fppSeguimientoPagoFactura       | Tabla    | Consumidor indirecto - seguimiento por cuenta destino | SI                      |
| fppEjecucionOrdenDePagoDestino  | Tabla    | Consumidor indirecto - destino de ejecucion de pago   | SI                      |
| fcppSeguimientoFactura          | Tabla    | Consumidor indirecto - seguimiento factura con cuenta | SI                      |
| fcppSeguimientoFacturaIndirecto | Tabla    | Datos bancarios inline en seguimiento                 | SI                      |
| vClienteDatosBancarios          | Vista    | Datos bancarios de clientes externos                  | NO - fuera alcance R16  |
| vFacturaClienteCalendario       | Vista    | Modulo consumidor via IdCatMedioDePago                | Referencia              |

---

## 1. EmpresaDatosBancarios (TABLA PRINCIPAL)
**Proposito:** Catalogo principal de cuentas bancarias del grupo PROQUIFA

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdEmpresaDatosBancarios | uniqueidentifier | NO | PK |
| IdDatosBancarios | uniqueidentifier | NO | FK - DatosBancarios |
| IdEmpresa | uniqueidentifier | SI | FK - Empresa (GOL/MUN/PRO/PQF) |
| FechaRegistro | datetime | NO | Fecha de creacion |
| FechaUltimaActualizacion | datetime | NO | Fecha de modificacion |
| Activo | bit | NO | 1=Vigente, 0=No vigente (historico) |

- Activo = 1: Cuenta vigente (visible a usuarios)
- Activo = 0: Cuenta no vigente (conservada para trazabilidad)

---

## 2. DatosBancarios
**Proposito:** Informacion detallada de cuentas bancarias

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdDatosBancarios | uniqueidentifier | NO | PK |
| IdCatBanco | uniqueidentifier | SI | FK - catBanco |
| NumeroDeCuenta | varchar(20) | SI | Numero de cuenta |
| Beneficiario | varchar(200) | SI | Empresa beneficiaria |
| Clabe | varchar(200) | SI | CLABE interbancaria |
| IdCatMoneda | uniqueidentifier | SI | FK - catMoneda (MXN/USD) |
| Sucursal | varchar(50) | SI | Sucursal bancaria |
| NumeroTarjeta | varchar(20) | SI | Numero de tarjeta |

---

## 3. Empresa
**Proposito:** Empresas del grupo PROQUIFA (emisoras)

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdEmpresa | uniqueidentifier | NO | PK |
| Prefijo | varchar(50) | NO | Codigo GOL/MUN/PRO/PQF |
| Alias | varchar(50) | NO | Nombre corto |
| RazonSocial | varchar(50) | NO | Nombre legal |
| RFC | varchar(13) | SI | RFC Mexico |
| IdCatRegimenFiscal | uniqueidentifier | SI | FK - catRegimenFiscal |
| Activo | bit | NO | Empresa activa |

---

## 4. CuentaEmpresa
**Proposito:** Vincula cuentas contables con datos bancarios

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdCuentaEmpresa | uniqueidentifier | NO | PK |
| IdEmpresa | uniqueidentifier | NO | FK - Empresa |
| IdCuenta | uniqueidentifier | NO | FK - Cuenta contable |
| IdDatosBancarios | uniqueidentifier | NO | FK - DatosBancarios |
| MXN | bit | SI | Cuenta en pesos mexicanos |
| USD | bit | SI | Cuenta en dolares |
| Activo | bit | NO | Registro activo |

---

## 5. catBanco (Catalogo)
**Proposito:** Instituciones bancarias

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdCatBanco | uniqueidentifier | NO | PK |
| Banco | varchar(180) | NO | Nombre del banco |
| IdCatPais | uniqueidentifier | SI | FK - catPais |
| Clave | varchar(8) | SI | Codigo bancario |
| Deposito | bit | SI | Permite depositos |
| Transferencia | bit | SI | Permite transferencias |
| Activo | bit | NO | Banco activo |

---

## 6. catMoneda (Catalogo)
**Proposito:** Monedas del sistema

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdCatMoneda | uniqueidentifier | NO | PK |
| ClaveMoneda | varchar(5) | NO | MXN, USD, EUR |
| Moneda | varchar(50) | NO | Pesos, Dolares, Euros |
| Activo | bit | NO | Moneda activa |

---

## 7. catMedioDePago (Catalogo)
**Proposito:** Medios y formas de pago

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdCatMedioDePago | uniqueidentifier | NO | PK |
| MedioDePago | nvarchar(200) | NO | Transferencia, Cheque, etc. |
| ClaveFormaDePago | varchar(2) | SI | Clave SAT |
| RequiereNumeroDeCuenta | bit | SI | Requiere num. cuenta |
| Activo | bit | NO | Medio activo |

---

## 8. Tablas Consumidoras Directas del Catalogo

### fcppOrdenDePago
**Proposito:** Orden de pago que referencia directamente EmpresaDatosBancarios

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdFCPPOrdenDePago | uniqueidentifier | NO | PK |
| IdEmpresaDatosBancarios | uniqueidentifier | NO | FK - EmpresaDatosBancarios (cuenta destino) |
| IdUsuarioCaptura | uniqueidentifier | SI | FK - Usuario que captura |
| IdUsuarioAutoriza | uniqueidentifier | SI | FK - Usuario que autoriza |
| Autorizada | bit | NO | Orden autorizada |
| FechaPropuesta | datetime | SI | Fecha propuesta de pago |
| Activo | bit | NO | Registro activo |

### fppEjecucionOrdenDePago
**Proposito:** Ejecucion de pagos contra cuentas del catalogo

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| IdFPPEjecucionOrdenDePago | uniqueidentifier | PK |
| IdFCPPOrdenDePago | uniqueidentifier | FK - fcppOrdenDePago |
| IdEmpresaDatosBancarios | uniqueidentifier | FK - EmpresaDatosBancarios (cuenta ejecutada) |
| FechaRealizacion | datetime | Fecha de realizacion |
| FechaFondeo | datetime | Fecha de fondeo |
| Activo | bit | Registro activo |

---

## 9. Tablas Consumidoras Indirectas

### fcppSeguimientoFactura
**Proposito:** Seguimiento de facturas con cuenta destino

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| IdFCPPSeguimientoFactura | uniqueidentifier | PK |
| IdCuentaDestino | uniqueidentifier | FK - Cuenta destino de pago |
| IdCFDI | uniqueidentifier | FK - CFDI relacionado |
| Autorizada | bit | Factura autorizada |
| Activo | bit | Registro activo |

### fppSeguimientoPagoFactura
**Proposito:** Seguimiento de pago por cuenta destino

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| IdFPPSeguimientoPagoFactura | uniqueidentifier | PK |
| IdFCPPSeguimientoFactura | uniqueidentifier | FK - fcppSeguimientoFactura |
| IdFPPEjecucionOrdenDePago | uniqueidentifier | FK - fppEjecucionOrdenDePago |
| IdCuentaDestino | uniqueidentifier | FK - Cuenta destino |
| Monto | decimal | Monto pagado |
| FechaPago | datetime | Fecha de pago |
| Activo | bit | Registro activo |

### fppEjecucionOrdenDePagoDestino
**Proposito:** Destino de ejecucion de pago

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| IdFPPEjecucionOrdenDePagoDestino | uniqueidentifier | PK |
| IdCuentaDestino | uniqueidentifier | FK - Cuenta destino |
| IdFPPSeguimientoPagoFactura | uniqueidentifier | FK - fppSeguimientoPagoFactura |
| IdFPPEjecucionOrdenDePago | uniqueidentifier | FK - fppEjecucionOrdenDePago |
| Activo | bit | Registro activo |

### fcppSeguimientoFacturaIndirecto
**Proposito:** Datos bancarios inline en seguimiento de factura indirecta

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| IdFCPPSeguimientoFacturaIndirecto | uniqueidentifier | PK |
| IdFCPPSeguimientoFactura | uniqueidentifier | FK - fcppSeguimientoFactura |
| IdCatBanco | uniqueidentifier | FK - catBanco (datos bancarios inline) |
| NumeroDeCuenta | varchar | Numero de cuenta inline |
| ClabeInterbancaria | varchar | CLABE inline |
| Activo | bit | Registro activo |

---

## 10. Vistas Relacionadas

### vClienteDatosBancarios
**NOTA: Fuera de alcance R16 - aplica a clientes externos, no a empresas del grupo**

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| IdCliente | uniqueidentifier | Cliente externo |
| IdDatosBancarios | uniqueidentifier | FK - DatosBancarios |
| IdCatBanco | uniqueidentifier | FK - catBanco |
| NumeroDeCuenta | varchar | Numero de cuenta |
| Beneficiario | varchar | Beneficiario |
| Clabe | varchar | CLABE interbancaria |
| IdCatMoneda | uniqueidentifier | Moneda |
| Banco | varchar | Nombre del banco |
| ClaveBanco | varchar | Codigo banco |
| ClaveBroker | varchar | Codigo broker |

### vFacturaClienteCalendario
**Proposito:** Vista consumidora del catalogo via IdCatMedioDePago

| Columna Relevante | Descripcion |
|-------------------|-------------|
| IdCatMedioDePago | Medio de pago de la factura |
| MedioDePago | Descripcion del medio |
| ClaveFormaDePago | Clave SAT |
| MXN / USD | Moneda de la factura |

---

## Consultas SQL Principales

### Cuentas vigentes del grupo

```sql
SELECT
    e.Prefijo,
    e.Alias AS Empresa,
    b.Banco,
    db.NumeroDeCuenta,
    db.Clabe,
    m.ClaveMoneda AS Moneda,
    edb.FechaRegistro
FROM dbo.EmpresaDatosBancarios edb
INNER JOIN dbo.Empresa e         ON edb.IdEmpresa = e.IdEmpresa
INNER JOIN dbo.DatosBancarios db ON edb.IdDatosBancarios = db.IdDatosBancarios
LEFT  JOIN dbo.catBanco b        ON db.IdCatBanco = b.IdCatBanco
LEFT  JOIN dbo.catMoneda m       ON db.IdCatMoneda = m.IdCatMoneda
WHERE edb.Activo = 1
  AND e.Activo   = 1
  AND e.Prefijo IN ('GOL','MUN','PRO','PQF')
ORDER BY e.Prefijo, db.NumeroDeCuenta;
```

### Ordenes de pago por cuenta del catalogo

```sql
SELECT
    op.IdFCPPOrdenDePago,
    e.Alias AS Empresa,
    db.NumeroDeCuenta,
    db.Clabe,
    op.FechaPropuesta,
    op.Autorizada
FROM dbo.fcppOrdenDePago op
INNER JOIN dbo.EmpresaDatosBancarios edb ON op.IdEmpresaDatosBancarios = edb.IdEmpresaDatosBancarios
INNER JOIN dbo.Empresa e                 ON edb.IdEmpresa = e.IdEmpresa
INNER JOIN dbo.DatosBancarios db         ON edb.IdDatosBancarios = db.IdDatosBancarios
WHERE op.Activo = 1
  AND edb.Activo = 1
ORDER BY op.FechaPropuesta DESC;
```

### Baja logica de cuenta

```sql
DECLARE @IdCuenta UNIQUEIDENTIFIER;

UPDATE dbo.EmpresaDatosBancarios
SET    Activo = 0,
       FechaUltimaActualizacion = GETDATE()
WHERE  IdEmpresaDatosBancarios = @IdCuenta;
```

---

## Reglas de Negocio

| Regla | Implementacion | Tabla/Campo |
|-------|----------------|-------------|
| Catalogo unico | Tabla centralizada | EmpresaDatosBancarios |
| Estado vigente | Campo binario | EmpresaDatosBancarios.Activo |
| Sin UI en R16 | Gestion manual BD | Soporte a la Produccion |
| Consumo modulos | Filtrar vigentes | WHERE Activo = 1 |

---

## Mejores Practicas

- SIEMPRE filtrar Activo = 1 en consultas de modulos
- SIEMPRE validar Prefijo IN ('GOL','MUN','PRO','PQF') para alcance R16
- NUNCA hacer DELETE fisico de registros
- NUNCA crear catalogos paralelos en otros modulos
- NO incluir GOLPERU en consultas de alcance R16

---

## Riesgos

| # | Riesgo | Mitigacion |
|---|--------|------------|
| 1 | Modelo Peru no definido | Definir en release posterior |
| 2 | Sin UI - errores manuales | Validaciones en BD y auditoria |

---
