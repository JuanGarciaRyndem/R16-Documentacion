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
| `catMedioDePago` | Catálogo | ✅ Catálogo definitivo recibido (2026-08-24) | Reemplaza el catálogo previo — ver sección 1 |
| `catMetodoDePagoCFDI` | Catálogo | Sin cambios estructurales | PUE/PPD — solo México |
| `catUsoCFDI` | Catálogo | Sin cambios estructurales | G01/G03/S01 etc. — solo México |
| `DatosFacturacionCliente` | Tabla | Agregar `IdCatMedioDePago` (FK) | Centraliza los tres campos de Cobros |
| `ConfiguracionPagos` | Tabla | Sin cambios estructurales | Mantiene condiciones de crédito y línea |

---

## 1. catMedioDePago — Forma de Pago

**Propósito:** Medio o forma de pago. Clave SAT del catálogo c_FormaPago, requerida para el CFDI.
**Cambio R16:** Sin cambios estructurales. Además de las columnas documentadas originalmente, el catálogo final incluye `RequiereNumeroDeCuenta`, `ObligatorioEnProveedor`, `ObligatorioEnCliente` e `IdRegion` (uniqueidentifier, FK a `Region` — ver `R16A-RE-FU-005-Tareas.md` Tarea 1/GAP-01).

**✅ ACTUALIZACIÓN 2026-08-24 — Catálogo definitivo México recibido del cliente:**

El cliente entregó el listado final y confirmó que ya está cargado/actualizado en el sistema. Reemplaza por completo el catálogo previo (10 registros ad-hoc: Aba, Cheque, Depósito bancario, Efectivo, NA, —NINGUNO—, Otros, Swift, Tarjeta, Transferencia) por un catálogo de 8 registros alineados 1:1 con el catálogo SAT c_FormaPago:

| `IdCatMedioDePago` | `MedioDePago` | `Activo` | `RequiereNumeroDeCuenta` | `ObligatorioEnProveedor` | `ClaveFormaDePago` | `Clave` | `ObligatorioEnCliente` | `IdRegion` |
|---|---|---|---|---|---|---|---|---|
| C171F83F-4991-4D0B-8D6C-D1A97AB0BA99 | Efectivo | 1 | 0 | 0 | 01 | efectivo | 1 | 60390FDA-7773-4BA1-8120-CB874F3A3A53 |
| 48D850F1-AFEE-415A-9554-330D8A292DD2 | Cheque nominativo | 1 | 0 | 0 | 02 | cheque | 1 | 60390FDA-7773-4BA1-8120-CB874F3A3A53 |
| 19A562AE-41E8-43C1-AA7C-A20000FE0A66 | Transferencia electrónica de fondos | 1 | 1 | 1 | 03 | transferencia | 1 | 60390FDA-7773-4BA1-8120-CB874F3A3A53 |
| 9AD47844-ADFB-4177-AC64-33E2864C87CC | Tarjeta de crédito | 1 | 1 | 1 | 04 | tarjetadecredito | 1 | 60390FDA-7773-4BA1-8120-CB874F3A3A53 |
| C4C59BBE-C057-4D02-A01C-2E78D3C40CD3 | Tarjeta de débito | 1 | 0 | 0 | 28 | tarjetadedebito | 1 | 60390FDA-7773-4BA1-8120-CB874F3A3A53 |
| A1F5D3B2-4C6E-4A8F-9D1C-7E2B4F6A8C0D | Aplicación de anticipos | 1 | 0 | 0 | 30 | aplicaciondeanticiposmex | 1 | 60390FDA-7773-4BA1-8120-CB874F3A3A53 |
| D587112C-1894-4318-9B66-EEDDF9CDDBED | Intermediario de pagos | 1 | 0 | 1 | 31 | depositobancario | 1 | 60390FDA-7773-4BA1-8120-CB874F3A3A53 |
| A955C869-4CD4-4A80-A9D3-77E69E3E1ADA | Por definir | 1 | 0 | 0 | 99 | otros | 1 | 60390FDA-7773-4BA1-8120-CB874F3A3A53 |

`IdRegion = 60390FDA-7773-4BA1-8120-CB874F3A3A53` es el GUID de la región México (mismo patrón que Perú en `R16A-RE-FU-025_BD.md`, con `IdRegion = 8278ECD0-C337-4484-B008-5B5E65B0DFAF`).

**Hallazgos a verificar con el cliente/Tesorería:**
- **Los registros ambiguos previos (Aba, NA, —NINGUNO—, Swift) NO aparecen en el catálogo final** — no fueron completados con una clave SAT, sino **retirados/reemplazados**. Esto activa la **Regla 9** del requisito (curaduría de clientes con opciones dadas de baja): cualquier cliente que tuviera asignado alguno de esos 4 valores queda con una referencia inválida tras el reemplazo y debe reasignarse a una opción vigente. **Pendiente confirmar con el cliente si ya se ejecutó esa curaduría junto con la carga del catálogo, o si queda como acción de seguimiento.**
- `Clave` (interna) `depositobancario` está asociada a `MedioDePago` = "Intermediario de pagos" (clave SAT 31) — nombre distinto al de la clave interna; no parece error de transcripción del cliente, pero vale la pena que el equipo confirme que es intencional antes de mostrarlo en el combo.
- ⚠️ La tabla de referencia SAT c_FormaPago documentada más abajo en este mismo archivo (sección "Catálogos SAT de Referencia") numera 28=A satisfacción del acreedor, 29=Tarjeta de débito, 30=Tarjeta de servicios, 31=Aplicación de anticipos — pero el catálogo real recibido usa 28=Tarjeta de débito, 30=Aplicación de anticipos, 31=Intermediario de pagos (sin usar 29). Uno de los dos listados tiene un desfase de numeración; dado que estos son los valores que el cliente reporta como ya cargados, se toman como la fuente vigente, pero vale la pena que el área fiscal confirme la tabla de referencia SAT contra el Anexo 20 CFDI 4.0 actual.

**Registros previos (histórico — reemplazados):**

| Descripción | `ClaveFormaDePago` SAT | Observación |
|---|---|---|
| Aba | *(vacío)* | ⛔ No está en el catálogo final — retirado |
| Cheque | 02 | Reemplazado por "Cheque nominativo" |
| Depósito bancario | 31 | Reemplazado por "Intermediario de pagos" (misma clave SAT 31) |
| Efectivo | 01 | Se mantiene igual |
| NA | *(vacío)* | ⛔ No está en el catálogo final — retirado |
| —NINGUNO— | *(vacío)* | ⛔ No está en el catálogo final — retirado |
| Otros | 99 | Reemplazado por "Por definir" (misma clave SAT 99) |
| Swift | *(vacío)* | ⛔ No está en el catálogo final — retirado |
| Tarjeta | 04 | Reemplazado por "Tarjeta de crédito"; se agrega "Tarjeta de débito" (28) como registro nuevo |
| Transferencia | 03 | Reemplazado por "Transferencia electrónica de fondos" |

| Columna                  | Tipo             | Longitud | Nulo | Descripción                          |
| ------------------------ | ---------------- | -------- | ---- | ------------------------------------ |
| `IdCatMedioDePago`       | uniqueidentifier | 16       | NO   | PK                                   |
| `MedioDePago`            | nvarchar         | 200      | NO   | Descripción del medio                |
| `ClaveFormaDePago`       | varchar          | 2        | SÍ   | Clave SAT c_FormaPago — nullable     |
| `Clave`                  | varchar          | 150      | NO   | Clave interna del sistema            |
| `RequiereNumeroDeCuenta` | bit              | 1        | NO   | Requiere captura de número de cuenta |
| `ObligatorioEnProveedor` | bit              | 1        | NO   | Obligatorio en catálogo proveedor    |
| `ObligatorioEnCliente`   | bit              | 1        | SÍ   | Obligatorio en catálogo cliente      |
| `IdRegion`               | uniqueidentifier | 16       | SÍ   | FK → `Region`                        |
| `Activo`                 | bit              | 1        | NO   | Default: 1                           |

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

**Validación a ejecutar:** ~~cruzar cada `ClaveFormaDePago` activa en `catMedioDePago` contra esta tabla. Si no existe → corregir o inactivar. Claves vacías (Aba, NA, NINGUNO, Swift) → asignar la clave correspondiente según el Excel de Proquifa.~~ **✅ RESUELTO (2026-08-24):** el cliente entregó el catálogo definitivo (ver sección 1 arriba) con las 8 claves SAT ya asignadas (01/02/03/04/28/30/31/99); los registros sin clave (Aba, NA, NINGUNO, Swift) fueron retirados en el reemplazo, no completados. Pendiente: confirmar la numeración 28/29/30/31 contra el Anexo 20 CFDI 4.0 vigente (ver nota de discrepancia en la sección 1).

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

### ~~Completar claves SAT en `catMedioDePago`~~ ✅ RESUELTO — catálogo reemplazado (2026-08-24)

```sql
-- NOTA: el cliente reporta que este catálogo ya fue cargado en el sistema.
-- Script documentado por trazabilidad — verificar contra el esquema real antes de re-ejecutar.
-- Reemplaza el catálogo previo completo (los registros Aba, NA, NINGUNO y Swift no forman parte del catálogo final).
INSERT INTO dbo.catMedioDePago
    (IdCatMedioDePago, MedioDePago, Activo, RequiereNumeroDeCuenta, ObligatorioEnProveedor, ClaveFormaDePago, Clave, ObligatorioEnCliente, IdRegion)
VALUES
    ('C171F83F-4991-4D0B-8D6C-D1A97AB0BA99', N'Efectivo',                             1, 0, 0, '01', 'efectivo',                  1, '60390FDA-7773-4BA1-8120-CB874F3A3A53'),
    ('48D850F1-AFEE-415A-9554-330D8A292DD2', N'Cheque nominativo',                    1, 0, 0, '02', 'cheque',                    1, '60390FDA-7773-4BA1-8120-CB874F3A3A53'),
    ('19A562AE-41E8-43C1-AA7C-A20000FE0A66', N'Transferencia electrónica de fondos',  1, 1, 1, '03', 'transferencia',             1, '60390FDA-7773-4BA1-8120-CB874F3A3A53'),
    ('9AD47844-ADFB-4177-AC64-33E2864C87CC', N'Tarjeta de crédito',                   1, 1, 1, '04', 'tarjetadecredito',          1, '60390FDA-7773-4BA1-8120-CB874F3A3A53'),
    ('C4C59BBE-C057-4D02-A01C-2E78D3C40CD3', N'Tarjeta de débito',                    1, 0, 0, '28', 'tarjetadedebito',           1, '60390FDA-7773-4BA1-8120-CB874F3A3A53'),
    ('A1F5D3B2-4C6E-4A8F-9D1C-7E2B4F6A8C0D', N'Aplicación de anticipos',              1, 0, 0, '30', 'aplicaciondeanticiposmex',  1, '60390FDA-7773-4BA1-8120-CB874F3A3A53'),
    ('D587112C-1894-4318-9B66-EEDDF9CDDBED', N'Intermediario de pagos',               1, 0, 1, '31', 'depositobancario',          1, '60390FDA-7773-4BA1-8120-CB874F3A3A53'),
    ('A955C869-4CD4-4A80-A9D3-77E69E3E1ADA', N'Por definir',                          1, 0, 0, '99', 'otros',                     1, '60390FDA-7773-4BA1-8120-CB874F3A3A53');
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
| 2 | ~~Claves SAT incompletas en `catMedioDePago`~~ | ~~Aba, NA, NINGUNO, Swift sin `ClaveFormaDePago`~~ | ✅ RESUELTO (2026-08-24) — catálogo reemplazado, ver sección 1 |

---

## Pendientes / Decisiones abiertas

| # | Pendiente | Responsable |
|---|---|---|
| P1 | ~~Excel de equivalencias SAT de Proquifa — necesario para completar claves SAT en `catMedioDePago`~~ | ✅ RESUELTO (2026-08-24) — cliente entregó catálogo final |
| P2 | Confirmar si el `IdCatMedioDePago` existente en `ConfiguracionPagos` se retira o se mantiene tras agregar el FK en `DatosFacturacionCliente` | TechLead |
| P3 | ~~Confirmar clave SAT c_FormaPago específica para Aba, NA, NINGUNO y Swift~~ | ✅ SIN OBJETO (2026-08-24) — esos registros no forman parte del catálogo final; ver curaduría pendiente (Regla 9) |

---

## Reglas de Negocio

| Regla | Descripción | Implementación en BD |
|---|---|---|
| R1 | Tres campos de Cobros en `DatosFacturacionCliente` | FK a `catMedioDePago`, `catUsoCFDI`, `catMetodoDePagoCFDI` |
| R2 | Campos obligatorios al guardar cliente | Validación en BO — los tres deben estar capturados |
| R3 | Registros SAT validados | ✅ RESUELTO (2026-08-24) — catálogo `catMedioDePago` México reemplazado con 8 claves SAT confirmadas (01/02/03/04/28/30/31/99) |
| R4 | Sin restricción de rol | Sin control de rol en BD |
