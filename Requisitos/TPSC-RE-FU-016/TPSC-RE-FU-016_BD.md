# Impacto en BD - Diseno y Generacion PDF Proforma Mexico
**Requisito:** TPSC-RE-FU-016
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

---

## Resumen
Generacion del PDF de Proforma al tramitar pedido Prepago sin FAA para clientes Mexico.
Branding diferenciado por empresa emisora (GOL/MUN/PRO/PQF). Foliador global con PRF.
PDF se genera bajo demanda en previsualizacion, se persiste en BD tras envio exitoso.
SIN CAMBIOS ESTRUCTURALES MAYORES - Posible nueva tabla de branding/plantilla.

---

## Impacto en BD

> El PDF se genera bajo demanda y se persiste como artefacto inmutable tras envio.
> Las fuentes de datos (Cliente, Pedido, CuentasBancarias, Empresa) ya existen.
> El foliador global ya existe (usado por tpProformaPedido.Folio).
> Posible impacto: tabla de configuracion de branding por empresa si no existe.

---

## Fuentes de Datos para el PDF

| Seccion del PDF | Tabla Fuente | Campos |
|-----------------|-------------|--------|
| Cabecera - Logo y color | Empresa | Prefijo (GOL/MUN/PRO/PQF) -> determina branding |
| Cabecera - Folio PRF | tpProformaPedido | Folio (formato MMDDAA-Consecutivo) |
| Cliente | Cliente / DatosFacturacionCliente | Alias o RazonSocial, RFC |
| Partidas | tpProformaPartidaPedido + tpPartidaPedido | IdProducto, NumeroDePiezas, PrecioUnitario |
| Descripcion producto | Producto / MarcaFamilia | Catalogo + Descripcion + Marca |
| Montos (SubTotal, IVA, Total) | tpProformaPedido | MontoTotal + calculo IVA |
| Moneda de facturacion | DatosFacturacionCliente | IdCatMoneda -> catMoneda.ClaveMoneda |
| Tipo de cambio | tpPedido | TipoCambioFacturacion / TipoDeCambioDiarioOficial |
| Condiciones de Pago | catCondicionesDePago | CondicionesDePago (ej. PREPAGO 100%) |
| Datos bancarios | EmpresaDatosBancarios + DatosBancarios + catBanco | Banco, Cuenta, CLABE, Sucursal |
| REF. CLIENTE | Logica CodigoValidador | Referencia bancaria dinamica por cuenta |
| Facturacion | DatosFacturacionCliente | RFC, RazonSocial, DireccionFiscal |
| Entrega | tpPedido + DireccionCliente | FolioPedidoInterno, Direccion, ContactoEntrega |
| Pie - Empresa emisora | Empresa | RazonSocial legal, Direccion legal |
| Pie - Certificaciones | Configuracion empresa | ISO 9001, NEEC, metodos pago |

---

## Tablas Consultadas (Lectura - sin modificacion)

| Tabla | Rol |
|-------|-----|
| tpPedido | Cabecera pedido: FolioPedidoInterno, IdEmpresa, IdRegion, TipoCambio |
| tpProformaPedido | Folio proforma, MontoTotal, ReferenciaPago |
| tpProformaPartidaPedido | Partidas: IdProducto, NumeroDePiezas, PrecioUnitario |
| tpPartidaPedido | Datos adicionales de partida |
| tpPedidoProformaPedido | Vinculacion pedido-proforma |
| Producto / MarcaFamilia | Catalogo, Descripcion, Marca |
| Cliente | Alias, Nombre |
| DatosFacturacionCliente | RFC, RazonSocial, IdCatMoneda, Correo, DireccionFiscal |
| DireccionCliente | Direccion de entrega |
| ContactoCliente | Contacto de entrega |
| Empresa | Prefijo, RazonSocial legal, Direccion legal -> branding |
| EmpresaDatosBancarios | Cuentas bancarias del grupo (MN + DLS) |
| DatosBancarios | NumeroDeCuenta, Clabe, Sucursal |
| catBanco | Nombre del banco |
| catMoneda | ClaveMoneda (MXN/USD) |
| catCondicionesDePago | CondicionesDePago texto |
| Region | Filtro MEX |

---

## Persistencia del PDF

| Etapa | Almacenamiento | Descripcion |
|-------|---------------|-------------|
| Previsualizacion | NO se persiste | PDF generado bajo demanda, descartable |
| Envio exitoso confirmado | SI se persiste | PDF final inmutable en BD |
| Consulta historica | Desde BD | PDF almacenado, sin regeneracion |

**Tabla de almacenamiento del PDF:**
- Posible: tpProformaPedido ya tiene relacion con Archivo via tpPedido.IdArchivoPDF
- O tabla Archivo generica del sistema (IdArchivo -> contenido binario)
- ** Pendiente decisión técnica: BLOB vs snapshot estructurado **

---

## Foliador Global PRF

| Aspecto | Detalle |
|---------|---------|
| Tabla | tpProformaPedido.Folio |
| Formato interno | MMDDAA-Consecutivo |
| Formato visual PDF | PRF-MMDDAA-Consecutivo |
| Segmentacion | NINGUNA (lineal global, sin segmentar por empresa/region) |
| Prefijo PRF | Solo en representacion visual del PDF - posiblemente no en BD |

> **Pendiente:** confirmar si el prefijo PRF se almacena en BD o solo en render.
> **Pendiente:** momento de consumo del folio (previsualizacion vs envio exitoso).

---

## Branding por Empresa Emisora

| Empresa | Prefijo | Logo | Color | Razon Social Legal |
|---------|---------|------|-------|-------------------|
| Golocaer S.A. de C.V. | GOL | Logo GOL | Color GOL | Completa |
| Mungen S.A. de C.V. | MUN | Logo MUN | Color MUN | Completa |
| Proquifa S.A. de C.V. | PRO | Logo PRO | Color PRO | Completa |
| Proveedora Quimico Farmaceutica S.A. de C.V. | PQF | Logo PQF | Color PQF | Completa |

> Actualmente la tabla Empresa tiene Prefijo, RazonSocial y posiblemente Alias.
> El logo y color institucional probablemente se gestionan en la capa de aplicacion
> o en una tabla de configuracion de branding. Verificar si existe.

---

## Codigo Validador (REF. CLIENTE)

    Cuenta Banamex:
        Concatenacion de 7 segmentos: nombre cliente + clave + codigo banco + moneda + CodValidador

    Cuenta No-Banamex:
        Nombre del cliente directo

> Esta logica ya existe en la aplicacion (RE-FU-006).
> El PDF la consume en el render de la seccion de datos bancarios.

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | Momento consumo folio (previsualizacion vs envio) | Tecnico | Definir con equipo desarrollo + validacion fiscal |
| 2 | Almacenamiento PDF (BLOB vs snapshot) | Tecnico | Definir con equipo desarrollo |
| 3 | Prefijo PRF en BD o solo en render | Tecnico | Confirmar con equipo |
| 4 | Vigencia del documento - regla de calculo | Negocio | Confirmar con cliente |
| 5 | Seccion Cliente: Alias vs RazonSocial | Negocio | Confirmar con cliente |
| 6 | Seccion Entrega: tipo de contacto | Negocio | Confirmar con cliente |
| 7 | Certificaciones y logos pie: vigencia | Negocio | Confirmar con cliente |
| 8 | Datos bancarios: siempre MN+DLS o variable | Negocio | Confirmar con cliente |
| 9 | Leyenda PUE: siempre aplica para Prepago | Fiscal | Confirmar con equipo contable |
| 10 | Tabla de branding por empresa (logo/color) | Tecnico | Verificar si existe o crear |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| TPSC-RE-FU-013 | Flujo Prepago con controlados - mismo PDF sin FAA |
| TPSC-RE-FU-014 | Flujo Prepago sin controlados sin FAA - mismo PDF |
| TPSC-RE-FU-006 | Codigo Validador / ReferenciaPago en seccion bancaria |
| TPSC-RE-FU-001 | Catalogo Cuentas Bancarias del grupo (seccion datos bancarios) |
| TPSC-RE-FU-007 | Leyenda regulatoria (si aplica controlados - otro PDF) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
