# Impacto en BD - Tramitacion Pedidos Credito con Factura por Adelantado
**Requisito:** R16A-RE-FU-012
**Base de Datos:** ProquifaDotNet
**Version:** 2.0 (adopta el esquema fccFactura de RE-FU-015 — reemplaza tpProformaAdelanto)

---

## Resumen
Activacion opcional de Factura por Adelantado para pedidos Credito (MEX) sin controlados.
Al tramitar genera pendiente en modulo FAA (`fccFactura` + `fccFacturaPartida` +
`fccFacturaReferenciaBancaria`, propiedad de `ProquifaDotNet.Finanzas`). Confirmacion de
Pedido (`tpProformaPedido`) se genera de inmediato sin esperar la factura, en paralelo
al pendiente FAA dentro de la misma transaccion. Datos de facturacion se bloquean.
Peru NO tiene esta opcion para Credito en R16.

**CAMBIO ESTRUCTURAL (migracion 06/07/2026):** el pendiente FAA de Credito ya NO se
modela con `tpProformaAdelanto`. Se unifica con el esquema `fccFactura` definido en
RE-FU-015 (Prepago), reutilizando la misma tabla para ambos origenes (Prepago y
Credito), diferenciados por `fccFactura.IdTPProformaPedido` (NULL en Prepago, poblado
en Credito con el `IdTPProformaPedido` de la Confirmacion de Pedido generada en
paralelo). Ver `R16A-RE-FU-015_BD.md`, seccion "Migracion de tpProformaAdelanto".

---

## Impacto en BD: SIN TABLAS NUEVAS PROPIAS — REUTILIZA fccFactura (RE-FU-015)

> Las tablas del pendiente FAA (`fccFactura`, `fccFacturaPartida`,
> `fccFacturaReferenciaBancaria`, vista `vfccFactura`) se crean en RE-FU-015; este
> requisito NO las vuelve a crear, solo las consume con `IdTPProformaPedido` poblado.
> `tpProformaAdelanto`, `tpProformaAdelantoProformaPedido` y `vtpProformaAdelanto`
> **ya NO aplican** a este requisito.
> `fccPagoFacturaAdelanto.IdTPProformaAdelanto` se retarget a `IdFccFactura` (migracion
> documentada en RE-FU-026/027 `_BD.md`).
> El campo `tpPedido.FacturaPorAdelantado` (bit) ya existe, sin cambios.
> No se requiere ALTER TABLE ni CREATE TABLE adicional en este requisito.

---

## Modelo de Datos

    tpPedido (Tramitar)
        FacturaPorAdelantado = 1 (activado)
        IdCliente -> Cliente -> DatosFacturacionCliente (datos fijados)
        IdCatCondicionesDePago -> catCondicionesDePago (Credito: SinCredito=0)
        Tramitado = 1 al completar

    tpProformaPedido (Confirmacion de Pedido - se genera inmediatamente,
                      EN PARALELO al pendiente FAA, misma transaccion)
        Controlados = 0 (sin controlados en este flujo)

    fccFactura (Pendiente FAA - se genera al tramitar, EN PARALELO a tpProformaPedido)
        IdTPPedido FK -> tpPedido (FK directa, reemplaza el vinculo ambiguo
                                    IdCliente+NumeroOrdenDeCompra de tpProformaAdelanto)
        IdTPProformaPedido FK -> tpProformaPedido (POBLADO en este flujo Credito;
                                    reemplaza tpProformaAdelantoProformaPedido)
        EsFacturaPorAdelantado = 1
        IdCliente, IdEmpresa, MontoTotal, IdCatMoneda
        IdCFDIGenerada = NULL (hasta que Finanzas emita la factura PPD)
        Enviada = 0 (hasta el envio de la factura final)

    fccFacturaPartida (1:N de fccFactura - snapshot de partidas del pedido)

    fccFacturaReferenciaBancaria (1:N de fccFactura - cuentas bancarias del grupo)

    fccPagoFacturaAdelanto (Pagos contra la factura adelanto)
        IdFCCPagoCliente FK -> fccPagoCliente
        IdFccFactura FK -> fccFactura (antes: IdTPProformaAdelanto -> tpProformaAdelanto)

---

## Tablas Involucradas

| Tabla | Rol | Estado |
|-------|-----|--------|
| tpPedido | Cabecera - FacturaPorAdelantado = 1 | Existente - sin cambios |
| tpProformaPedido | Confirmacion de Pedido (se genera inmediatamente, en paralelo) | Existente - sin cambios |
| **fccFactura** | Pendiente FAA (cabecera) — reemplaza `tpProformaAdelanto` | Reutilizada de RE-FU-015 (no se crea aqui) |
| **fccFacturaPartida** | Detalle de partidas del pendiente FAA | Reutilizada de RE-FU-015 |
| **fccFacturaReferenciaBancaria** | Referencias bancarias del pendiente FAA | Reutilizada de RE-FU-015 |
| fccPagoFacturaAdelanto | Pagos contra la factura adelanto — FK retargeteada a `fccFactura` | Existente — requiere migracion de FK (ver RE-FU-026/027) |
| DatosFacturacionCliente | Datos de facturacion del cliente (se fijan como snapshot en `fccFactura`) | Existente - sin cambios |
| catCondicionesDePago | Determina Credito (SinCredito=0) | Existente - sin cambios |
| tpPedidoCorreoEnviado | Correo de confirmacion | Existente - sin cambios |

> `tpProformaAdelanto` y `tpProformaAdelantoProformaPedido` **ya no aplican** a este requisito.

---

## Campos Clave en tpPedido

| Campo | Tipo | Uso en este Flujo |
|-------|------|-------------------|
| FacturaPorAdelantado | bit | **= 1 cuando se activa FAA** |
| IdCatCondicionesDePago | uniqueidentifier | Credito: SinCredito=0 |
| EntregaConRemision | bit | No aplica en este flujo (sin controlados) |
| Tramitado | bit | = 1 al completar |
| FolioPedidoInterno | varchar(15) | Folio asignado al tramitar |
| IdRegion | uniqueidentifier | Solo MEX puede activar FAA para Credito |

---

## fccFactura (Pendiente FAA — reemplaza tpProformaAdelanto)

> Tabla completa definida y creada en `R16A-RE-FU-015_BD.md`. Este requisito solo
> puebla las columnas relevantes para el origen Crédito. No se repite aquí el
> diccionario de datos completo — ver el original en RE-FU-015.

| Campo (relevante para Crédito) | Tipo | Descripcion |
|-------|------|-------------|
| IdFccFactura | uniqueidentifier | PK (antes: `IdTPProformaAdelanto`) |
| IdTPPedido | uniqueidentifier | FK directa → `tpPedido` (antes: vínculo ambiguo `IdCliente+NumeroOrdenDeCompra`) |
| IdTPProformaPedido | uniqueidentifier | **Poblado en este flujo** → `tpProformaPedido` (Confirmación de Pedido generada en paralelo). Reemplaza `tpProformaAdelantoProformaPedido` |
| EsFacturaPorAdelantado | bit | = 1 en este flujo |
| MontoTotal | decimal | Monto total del pedido a facturar |
| IdCatMoneda | uniqueidentifier | Moneda de la factura (MXN / USD) |
| IdCFDIGenerada | uniqueidentifier NULL | NULL hasta que Finanzas emita la factura PPD (FK a `CFDIGenerada`) |
| Enviada | bit | 0 hasta enviar la factura final |
| TipoCambio | decimal NULL | Tipo de cambio al momento de generación de la FAA — seteado independientemente al activar la FAA. **No heredar `tpPedido.TipoCambioFacturacion`** (siempre = 1, bug legacy — OBS-TC, ver RE-FU-016_BD.md). |
| IdCliente | uniqueidentifier | Cliente del pedido |
| IdEmpresa | uniqueidentifier | Empresa que factura |
| FolioPedidoInterno | varchar | ← `tpPedido.FolioPedidoInterno` (reemplaza `NumeroOrdenDeCompra` como referencia) |
| Activo | bit | 1 = pendiente vigente |

---

## DatosFacturacionCliente (datos que se fijan al activar FAA)

| Campo | Tipo | Se fija al activar |
|-------|------|--------------------|
| RFC | varchar(50) | SI - no editable en Tramitar |
| RazonSocial | varchar(120) | SI |
| IdCatUsoCFDI | uniqueidentifier | SI |
| IdCatMetodoDePagoCFDI | uniqueidentifier | SI |
| Correo | varchar(200) | SI |
| IdCatRegimenFiscal | uniqueidentifier | SI |

> Al activar FAA los datos se toman del catalogo vigente y quedan de solo lectura.
> Cualquier ajuste se gestiona en el modulo FAA o actualizando el catalogo del cliente.

---

## Flujo en BD al Tramitar con FAA

    1. tpPedido.FacturaPorAdelantado = 1
    2. tpPedido.Tramitado = 1, FechaTramitacion = GETDATE()
    3. INSERT tpProformaPedido (Confirmacion de Pedido - inmediata)
    4. ProquifaDotNet.Finanzas INSERT atomico (mismo patron que RE-FU-015):
       - fccFactura (EsFacturaPorAdelantado=1, IdTPProformaPedido = Id de la
         Confirmacion de Pedido insertada en el paso 3, IdCFDIGenerada=NULL)
       - fccFacturaPartida (una por partida del pedido)
       - fccFacturaReferenciaBancaria (cuentas M.N./DLS + ReferenciaCliente)
    5. INSERT tpPedidoCorreoEnviado (Correo de confirmacion al cliente)
    6. Transferencia a Legacy (solo MEX, con marca detencion si Pago c/e)

> Los pasos 3 y 4 son **paralelos** dentro de la misma transacción de tramitación
> (no secuenciales): `tpProformaPedido` no es un prerrequisito de `fccFactura`, ambos
> se insertan como parte de la misma operación atómica, y `fccFactura.IdTPProformaPedido`
> enlaza al registro de `tpProformaPedido` recién insertado.

---

## Diferencia MEX vs PER

| Aspecto | Mexico (MEX) | Peru (PER) |
|---------|-------------|------------|
| FAA disponible para Credito | SI | NO (timbrado peruano R16 = solo Prepago) |
| Tramitacion Credito | Flujo normal | Flujo normal |
| Transferencia a Legacy | SI | NO |

---

## Consulta - Pedidos con FAA activa

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Ahora usa la vista vfccFactura (RE-FU-015) con JOIN directo por IdTPPedido,
    -- reemplaza el vinculo ambiguo IdCliente+NumeroOrdenDeCompra de tpProformaAdelanto
    SELECT
        v.FolioPedidoInterno,
        v.ClienteNombre AS Cliente,
        v.MontoTotal,
        v.FechaTramitacion,
        v.IdFccFactura,
        v.EstadoFAA
    FROM dbo.vfccFactura v
    WHERE v.FacturaPorAdelantado = 1
      AND v.Activo = 1
    ORDER BY v.FechaTramitacion DESC;

---

## Cambios de Comportamiento R16 (sin cambio en BD)

| Cambio | Antes | Ahora (R16) |
|--------|-------|-------------|
| Codigo de autorizacion para FAA | Requeria codigo | Activacion directa sin codigo |
| Boton Editar Datos con FAA activa | Disponible | **Oculto** cuando FAA=1 |
| FAA para Peru Credito | No existia Peru | NO disponible (timbrado Peru = solo Prepago) |

> Estos cambios son de LOGICA DE APLICACION, no de estructura de BD.

---

## Gaps

| # | Gap | Accion |
|---|-----|--------|
| 1 | ~~Relacion tpProformaAdelanto con tpPedido no es FK directa~~ | **Resuelto por la migración**: `fccFactura.IdTPPedido` es FK directa y obligatoria |
| 2 | ~~Rol que gestiona FAA no confirmado (Finanzas o Coordinador de Planeacion)~~ | **Resuelto (DUDA-028, 2026-08-21)**: no se define en este requisito; el rol responsable queda determinado por los permisos/roles de acceso al módulo Factura por Adelantado (ver DUDA-047) |
| 3 | Pendiente 'Relacionar facturas' en Legacy | Mecanismo PQF2->Legacy para este pendiente fuera de scope |
| 4 | Migración de `fccPagoFacturaAdelanto.IdTPProformaAdelanto` → `IdFccFactura` | Ver `R16A-RE-FU-026_BD.md`/`R16A-RE-FU-027-Back.md` — impacta la asociación de cobro para FAA de Crédito |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-010 | Flujo base Credito sin FAA |
| R16A-RE-FU-011 | Con controlados: FAA no disponible (mutuamente excluyentes) |
| R16A-RE-FU-005 | Datos de facturacion (IdCatUsoCFDI, IdCatMetodoDePagoCFDI) |
| R16A-RE-FU-015 | Origen y dueño de `fccFactura`/`fccFacturaPartida`/`fccFacturaReferenciaBancaria`/`vfccFactura` — este requisito solo consume |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
