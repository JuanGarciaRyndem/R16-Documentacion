# Diccionario de Datos — Configuración de Cobros y Facturación del Cliente

| Campo | Valor |
|---|---|
| **Requisito** | R16A-RE-FU-005 |
| **Base de Datos** | ProquifaDotNet |
| **Servidor** | RYNL010 |
| **Versión** | 2.0 — Cancelación timbrado Perú: se elimina regionalización; se consolidan Forma de Pago, Uso de CFDI y Método de Pago en `DatosFacturacionCliente` |
| **Generado por** | GitHub Copilot in SSMS |

---

## Resumen Ejecutivo

Captura y mantenimiento de la configuración de cobros y facturación del cliente en la sección **Cobros** del Catálogo de Clientes. Los tres campos obligatorios (Forma de Pago, Uso de CFDI, Método de Pago) se almacenan en `DatosFacturacionCliente` y son consumidos por los módulos de Factura por Adelantado y Validar Cobro.

Con la cancelación del timbrado para Perú, se elimina la estrategia de regionalización (`IdRegion` en catálogos, registros PE, banderas tributarias) y los catálogos quedan exclusivamente para México.

---

## Modelo de Datos

```
Cliente
└── DatosFacturacionCliente
        ├── FK IdCatMedioDePago      → catMedioDePago      (Forma de Pago — NUEVO R16)
        ├── FK IdCatUsoCFDI          → catUsoCFDI          (Uso de CFDI — existente)
        └── FK IdCatMetodoDePagoCFDI → catMetodoDePagoCFDI (Método de Pago — existente)

Cliente
└── FK IdConfiguracionPagos
        ConfiguracionPagos
        ├── FK IdCatCondicionesDePago → catCondicionesDePago (plazos en días — sin cambios)
        └── IdCatMedioDePago (existente en ConfiguracionPagos — ver Pendiente P2)
```

---

## Entidades Afectadas
![[Pasted image 20260723102226.png]]

| Entidad | Tipo | Cambio R16 | Observación |
|---|---|---|---|
| `catMedioDePago` | Catálogo | Claves SAT pendientes — ⏸ En espera Excel | `ClaveFormaDePago` incompleta en 4 registros |
| `catMetodoDePagoCFDI` | Catálogo | Sin cambios estructurales | PUE/PPD — solo México |
| `catUsoCFDI` | Catálogo | Sin cambios estructurales | G01/G03/S01 etc. — solo México |
| `DatosFacturacionCliente` | Tabla | Agregar `IdCatMedioDePago` (FK) | Centraliza los tres campos de Cobros |
| `ConfiguracionPagos` | Tabla | Sin cambios estructurales | Mantiene condiciones de crédito y línea |

---

## 1. catMedioDePago — Forma de Pago

**Propósito:** Medio o forma de pago. Clave SAT del catálogo c_FormaPago, requerida para el CFDI.
**Cambio R16:** Sin cambios estructurales. Pendiente completar `ClaveFormaDePago` en 4 registros — ⏸ En espera Excel Proquifa.

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdCatMedioDePago` | uniqueidentifier | 16 | NO | PK |
| `MedioDePago` | nvarchar | 200 | NO | Descripción del medio |
| `ClaveFormaDePago` | varchar | 2 | SÍ | Clave SAT c_FormaPago — nullable |
| `Clave` | varchar | 150 | NO | Clave interna del sistema |
| `RequiereNumeroDeCuenta` | bit | 1 | NO | Requiere captura de número de cuenta |
| `ObligatorioEnCliente` | bit | 1 | SÍ | Obligatorio en catálogo cliente |
| `Activo` | bit | 1 | NO | Default: 1 |

**Registros activos:**

| Descripción | `ClaveFormaDePago` SAT | Observación |
|---|---|---|
| Aba | *(vacío)* | ⏸ Pendiente Excel Proquifa |
| Cheque | 02 | ✅ OK |
| Depósito bancario | 31 | ✅ OK |
| Efectivo | 01 | ✅ OK |
| NA | *(vacío)* | ⏸ Pendiente Excel Proquifa |
| —NINGUNO— | *(vacío)* | ⏸ Pendiente Excel Proquifa |
| Otros | 99 | ✅ OK |
| Swift | *(vacío)* | ⏸ Pendiente Excel Proquifa |
| Tarjeta | 04 | ✅ OK |
| Transferencia | 03 | ✅ OK |

---

## 2. catMetodoDePagoCFDI — Método de Pago

**Propósito:** Dimensión temporal del pago. Clave SAT del catálogo c_MetodoPago.
**Cambio R16:** Sin cambios estructurales ni de registros.

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdCatMetodoDePagoCFDI` | uniqueidentifier | 16 | NO | PK |
| `MetodoDePagoCFDI` | nvarchar | 100 | NO | Descripción |
| `ClaveMetodoDePagoCFDI` | nvarchar | 6 | NO | Clave SAT c_MetodoPago |
| `Clave` | varchar | 150 | NO | Clave interna |
| `Activo` | bit | 1 | NO | Default: 1 |

**Registros:**

| Clave SAT | Descripción | Estado |
|---|---|---|
| PUE | Pago en una sola exhibición | ✅ Existente |
| PPD | Pago en parcialidades o diferido | ✅ Existente |

---

## 3. catUsoCFDI — Uso de CFDI

**Propósito:** Uso o tipo del comprobante fiscal. Clave SAT del catálogo c_UsoCFDI.
**Cambio R16:** Sin cambios estructurales ni de registros.

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdCatUsoCFDI` | uniqueidentifier | 16 | NO | PK |
| `ClaveUso` | nvarchar | 6 | NO | Clave SAT c_UsoCFDI |
| `Uso` | nvarchar | 300 | NO | Descripción |
| `Clave` | varchar | 150 | NO | Clave interna |
| `Activo` | bit | 1 | NO | Default: 1 |

**Registros activos relevantes:**

| Clave SAT | Descripción                               | Estado      |
| --------- | ----------------------------------------- | ----------- |
| G01       | Adquisición de mercancías                 | ✅ Existente |
| G02       | Devoluciones, descuentos o bonificaciones | ✅ Existente |
| G03       | Gastos en general                         | ✅ Existente |
| S01       | Sin efectos fiscales                      | ✅ Existente |
| P01       | Por definir                               | ✅ Existente |

---

## 4. DatosFacturacionCliente — campos de Cobros

**Propósito:** Almacena los tres campos de Cobros del cliente: Forma de Pago, Uso de CFDI y Método de Pago. Son la configuración default consumida por Factura por Adelantado y Validar Cobro.

**Cambio R16:** Agregar `IdCatMedioDePago` para centralizar los tres campos en este objeto.

| Columna                 | Tipo             | Nulo | Estado        | Descripción                                 |
| ----------------------- | ---------------- | ---- | ------------- | ------------------------------------------- |
| `IdCatMedioDePago` ✨    | uniqueidentifier | SÍ   | **NUEVO R16** | FK → `catMedioDePago` (Forma de Pago)       |
| `IdCatUsoCFDI`          | uniqueidentifier | SÍ   | ✅ Existente   | FK → `catUsoCFDI` (Uso de CFDI)             |
| `IdCatMetodoDePagoCFDI` | uniqueidentifier | SÍ   | ✅ Existente   | FK → `catMetodoDePagoCFDI` (Método de Pago) |

> Los tres campos son NULLABLE en BD pero **obligatorios en la capa de negocio** — el BO valida que estén capturados antes de persistir y retorna error si alguno está vacío.

---

## 5. ConfiguracionPagos — sin cambios estructurales

**Propósito:** Condiciones de crédito y línea de crédito del cliente.
**Cambio R16:** Sin cambios. `IdCatMedioDePago` se agrega a `DatosFacturacionCliente` como campo de Cobros sin retirar el existente en `ConfiguracionPagos` hasta confirmar (ver Pendiente P2).

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `IdConfiguracionPagos` | uniqueidentifier | NO | PK |
| `IdCatCondicionesDePago` | uniqueidentifier | SÍ | FK — plazos de crédito en días |
| `IdCatMedioDePago` | uniqueidentifier | SÍ | FK — Forma de Pago (ver Pendiente P2) |
| `LineaCredito` | decimal | SÍ | Monto de línea de crédito |
| `LimiteLineaCredito` | decimal | SÍ | Límite de línea de crédito |
| `Activo` | bit | NO | Default: 1 |

---

## Catálogos SAT de Referencia — Validación con Excel Proquifa ⏸ En espera

> Estos son los valores oficiales del SAT (Anexo 20 CFDI 4.0). Al recibir el Excel de Proquifa, cada registro de la BD debe cruzarse contra estas tablas para verificar que las claves asignadas existan y estén vigentes.

### c_FormaPago — referencia para `catMedioDePago.ClaveFormaDePago`

| Clave SAT | Descripción SAT                     |
| --------- | ----------------------------------- |
| 01        | Efectivo                            |
| 02        | Cheque nominativo                   |
| 03        | Transferencia electrónica de fondos |
| 04        | Tarjeta de crédito                  |
| 05        | Monedero electrónico                |
| 06        | Dinero electrónico                  |
| 08        | Vales de despensa                   |
| 12        | Dación en pago                      |
| 13        | Pago por subrogación                |
| 14        | Pago por consignación               |
| 15        | Condonación                         |
| 17        | Compensación                        |
| 23        | Novación                            |
| 24        | Confusión                           |
| 25        | Remisión de deuda                   |
| 26        | Prescripción o caducidad            |
| 28        | A satisfacción del acreedor         |
| 29        | Tarjeta de débito                   |
| 30        | Tarjeta de servicios                |
| 31        | Aplicación de anticipos             |
| 99        | Por definir                         |

**Validación a ejecutar:** cruzar cada `ClaveFormaDePago` activa en `catMedioDePago` contra esta tabla. Si no existe → corregir o inactivar. Claves vacías (Aba, NA, NINGUNO, Swift) → asignar la clave correspondiente según el Excel de Proquifa.

### c_MetodoPago — referencia para `catMetodoDePagoCFDI.ClaveMetodoDePagoCFDI`

| Clave SAT | Descripción SAT |
|---|---|
| PUE | Pago en una sola exhibición |
| PPD | Pago en parcialidades o diferido |

**Validación a ejecutar:** `catMetodoDePagoCFDI` solo debe contener PUE y PPD. Si hay otros valores → revisar con cliente.

### c_UsoCFDI — referencia para `catUsoCFDI.ClaveUso`

| Clave SAT | Descripción SAT                                                                      |
| --------- | ------------------------------------------------------------------------------------ |
| G01       | Adquisición de mercancias                                                            |
| G02       | Devoluciones, descuentos o bonificaciones                                            |
| G03       | Gastos en general                                                                    |
| I01       | Construcciones                                                                       |
| I02       | Mobilario y equipo de oficina por inversiones                                        |
| I03       | Equipo de transporte                                                                 |
| I04       | Equipo de computo y accesorios                                                       |
| I05       | Dados, troqueles, moldes, matrices y herramental                                     |
| I06       | Comunicaciones telefónicas                                                           |
| I07       | Comunicaciones satelitales                                                           |
| I08       | Otra maquinaria y equipo                                                             |
| D01       | Honorarios médicos, dentales y gastos hospitalarios                                  |
| D02       | Gastos médicos por incapacidad o discapacidad                                        |
| D03       | Gastos funerales                                                                     |
| D04       | Donativos                                                                            |
| D05       | Intereses reales efectivamente pagados por créditos hipotecarios (casa habitación)   |
| D06       | Aportaciones voluntarias al SAR                                                      |
| D07       | Primas por seguros de gastos médicos                                                 |
| D08       | Gastos de transportación escolar obligatoria                                         |
| D09       | Depósitos en cuentas para el ahorro, primas que tengan como base planes de pensiones |
| D10       | Pagos por servicios educativos (colegiaturas)                                        |
| S01       | Sin efectos fiscales                                                                 |
| CP01      | Pagos                                                                                |
| CN01      | Nómina                                                                               |

**Validación a ejecutar:** cruzar cada `ClaveUso` activa en `catUsoCFDI` contra esta tabla. Registros como `N/A` o `P01` (por definir) que no correspondan a una clave SAT válida → revisar con cliente si se inactivan o se mapean.

---

## Script de Cambio Estructural R16

### Agregar Forma de Pago a `DatosFacturacionCliente`

```sql
-- Ejecutar en ProquifaDotNet
-- Created by GitHub Copilot in SSMS - review carefully before executing
ALTER TABLE dbo.DatosFacturacionCliente
    ADD IdCatMedioDePago uniqueidentifier NULL
        CONSTRAINT FK_DatosFacturacionCliente_MedioDePago
            FOREIGN KEY REFERENCES dbo.catMedioDePago(IdCatMedioDePago);
GO
```

### Completar claves SAT en `catMedioDePago` ⏸ En espera Excel Proquifa

```sql
-- EJECUTAR SOLO TRAS RECIBIR Y CONFIRMAR EL EXCEL DE PROQUIFA
-- UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'Aba';
-- UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'NA';
-- UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'NINGUNO';
-- UPDATE dbo.catMedioDePago SET ClaveFormaDePago = 'XX' WHERE Clave = 'Swift';
```

---

## Módulos Consumidores

| Módulo | Campos Consumidos |
|---|---|
| Factura por Adelantado | `IdCatMedioDePago`, `IdCatMetodoDePagoCFDI` (PUE/PPD), `IdCatUsoCFDI` |
| Validar Cobro | `IdCatMedioDePago`, `IdCatMetodoDePagoCFDI`, `IdCatUsoCFDI` |

---

## Gaps y Acciones Pendientes

| # | Gap | Descripción | Estado |
|---|---|---|---|
| 1 | `IdCatMedioDePago` ausente en `DatosFacturacionCliente` | Forma de Pago no está en el objeto de Cobros | Pendiente — ejecutar script |
| 2 | Claves SAT incompletas en `catMedioDePago` | Aba, NA, NINGUNO, Swift sin `ClaveFormaDePago` | ⏸ En espera Excel Proquifa |

---

## Pendientes / Decisiones abiertas

| # | Pendiente | Responsable |
|---|---|---|
| P1 | Excel de equivalencias SAT de Proquifa — necesario para completar claves SAT en `catMedioDePago` | Proquifa / Funcional |
| P2 | Confirmar si el `IdCatMedioDePago` existente en `ConfiguracionPagos` se retira o se mantiene tras agregar el FK en `DatosFacturacionCliente` | TechLead |
| P3 | Confirmar clave SAT c_FormaPago específica para Aba, NA, NINGUNO y Swift | Funcional / Cliente |

---

## Reglas de Negocio

| Regla | Descripción | Implementación en BD |
|---|---|---|
| R1 | Tres campos de Cobros en `DatosFacturacionCliente` | FK a `catMedioDePago`, `catUsoCFDI`, `catMetodoDePagoCFDI` |
| R2 | Campos obligatorios al guardar cliente | Validación en BO — los tres deben estar capturados |
| R3 | Registros SAT validados | Claves SAT correctas en catálogos — ⏸ En espera Excel |
| R4 | Sin restricción de rol | Sin control de rol en BD |
