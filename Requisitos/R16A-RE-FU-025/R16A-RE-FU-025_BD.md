# Impacto en BD - Validar Cobro: Paso 1 Peru (Captura del Cobro)
**Requisito:** R16A-RE-FU-025
**Base de Datos:** ProquifaDotNet
**Version:** 1.1 (actualizado 2026-08-24: incorpora el Diseño de la Solución — DIS-SOL v1.0, Osmar Calderón, 22/07/2026 — documento **delta** que hereda íntegramente la infraestructura de RE-024)

---

## 🔄 ACTUALIZACIÓN 2026-08-24 — Diseño de la Solución (DIS-SOL v1.0, Osmar Calderón, 22/07/2026)

> **El DIS-SOL de RE-025 es un documento delta: no crea DDL ni código nuevo.** Confirma que Perú reutiliza **íntegramente** la infraestructura que documenta el DIS-SOL de RE-024 (ver `[R16A-RE-FU-024][DIS-SOL] Diseño de la solución.md` y la actualización 2026-08-24 en `R16A-RE-FU-024_BD.md`) — la tabla fusionada **`fccCobroCliente`** (ya NO `fccPagoCliente`), el catálogo **`catCobroEstatus`** (7 estatus, sin `TIMBRADO`), y el foliador patrón RE-013 (`fccCobroClienteConsecutivo` + `sp_IncrementarFolioCobro`, ya NO `SEQUENCE SeqFolioCobro`). **Las secciones originales de este archivo (abajo) referencian el esquema anterior (`fccPagoCliente`, `SeqFolioCobro`) — quedan como trazabilidad histórica, no como DDL vigente.**

**Brechas que el DIS-SOL de RE-025 SÍ resuelve:**

| Brecha original | Resolución (DIS-SOL RE-025, decisiones DP2–DP4) |
|---|---|
| **B3 — Fuente del TC para Perú, no definida** | **Resuelta (DP2):** la fuente es **BCRP**, ya persistida en `TipoDeCambioBanamex` — el importador vigente trae PEN del BCRP (cruce USD/MXN × PEN/USD). Se descarta SBS. No requiere DML ni servicio nuevo |
| **Servicio `TipoCambioPeruService`** (que este mismo archivo, en su versión original, daba por hecho que se crearía) | **Descartado (DP3):** no se crea. El `ExchangeRateService` **único** de RE-024 cubre Perú — la moneda base (MXN/PEN) se resuelve por la región del usuario dentro del mismo servicio |
| Doble TC persistido en el cobro (planteado como paralelo al de México) | **Confirmado con matiz (DP4):** en Perú se persiste **un solo TC operativo** — `TipoDeCambioFacturacion` (rename de `TipoDeCambioMonedaFacturacion`, base PEN). `TipoDeCambioFiscal` (rename de `TipoDeCambio`) es **siempre NULL** en Perú — SUNAT no tiene Complemento de Pago (regla técnica RT-P02) |

**Brechas que siguen abiertas (sin cambio respecto al análisis original de este archivo):**
- **B1 — Cuentas bancarias GOLPERU (0 registros).** Sigue bloqueante; el cliente/Golocaer S.A.C. debe entregar banco, número de cuenta, CCI (20 dígitos) y moneda.
- **B2 — Catálogo interno de medio de pago Perú.** Sigue pendiente de transcripción (DUDA-076): el cliente entregó la lista definitiva el 18/08 pero como imagen. El DIS-SOL agrega un detalle nuevo: la columna que regionaliza el catálogo (`IdRegion` en `catMedioDePago`) **la agrega RE-005** (ALTER TABLE + FK a Region) — RE-025 solo inserta los registros Perú con esa `IdRegion`, no ejecuta el ALTER.
- **B4 — Catálogo de Tipos de Inconsistencia del Paso 1.** Sigue pendiente de PROQUIFA Tesorería (transversal con RE-024).

**⚠️ Brecha que el DIS-SOL de RE-025 (v1.0, 22/07/2026) da por abierta pero que RE-024 YA RESOLVIÓ después — B5, foliador global vs. por región:** este documento y el DIS-SOL de RE-025 siguen listando "SeqFolioCobro global (MEX+PER) vs. por región" como brecha bloqueante. **DUDA-072, resuelta el 21/08/2026** (posterior a ambos DIS-SOL, fechados 20/07 y 22/07), ya confirmó: consecutivo **independiente por región**, con prefijo `COM` (México) / `COP` (Perú). Ninguno de los dos DIS-SOL (024 ni 025) incorpora todavía este ajuste — es el mismo conflicto ya señalado en la actualización de `R16A-RE-FU-024_BD.md`. **Pendiente:** actualizar ambos diseños a `fccCobroClienteConsecutivo` con prefijo `COP` para Perú antes de construcción.

**Verificación nueva que aporta el DIS-SOL (no estaba en la versión original de este archivo):** confirmar que `catMoneda` tiene un registro `PEN` activo y que existe su fila en `CatMonedaRegion` para la región Perú; si falta, requiere INSERT (verificación, no DDL).

**Orden de ejecución (DIS-SOL):** 1) verificar que RE-024 esté desplegado (tabla `fccCobroCliente`, catálogos, foliador, endpoints); 2) INSERT `catMedioDePago` Perú; 3) INSERT `EmpresaDatosBancarios` + `DatosBancarios` GOLPERU; 4) verificar `catMoneda` + `CatMonedaRegion` para PEN.

---

## ✅ ACTUALIZACIÓN 2026-08-24 (2) — Catálogo definitivo de medio de pago Perú RECIBIDO — Brecha B2 (DUDA-076) RESUELTA

> **La brecha B2 ("Catálogo interno de medio de pago Perú pendiente de transcripción") queda cerrada.** El cliente entregó el catálogo ya cargado en BD con 4 registros. El placeholder ilustrativo (`PER-TRF`/`PER-DEP`/`PER-CHQ`/`PER-EFE`) de la sección histórica abajo **queda obsoleto y no debe usarse** — estos son los valores reales:

| `IdCatMedioDePago` | `MedioDePago` | `RequiereNumeroDeCuenta` | `ObligatorioEnProveedor` | `ObligatorioEnCliente` | `ClaveFormaDePago` | `Clave` | `Activo` | `IdRegion` |
|---|---|---|---|---|---|---|---|---|
| `FDA4F4D2-0CF6-4679-80DD-06AE471A34E2` | Depósito en cuenta | 1 | 1 | 0 | `001` | `depositoencuentaperu` | 1 | `8278ECD0-C337-4484-B008-5B5E65B0DFAF` |
| `2525809E-5B71-4286-8A5B-1154CF84294F` | Transferencia de fondos | 1 | 1 | 0 | `003` | `transferenciadefondosperu` | 1 | `8278ECD0-C337-4484-B008-5B5E65B0DFAF` |
| `A5E8F79C-AC79-41F8-B1CF-6F447D9B8359` | Tarjeta de débito | 1 | 0 | 0 | `005` | `tarjetadedebitoperu` | 1 | `8278ECD0-C337-4484-B008-5B5E65B0DFAF` |
| `5D1C0261-1081-4F51-AD0A-06F84553B68D` | Otros medios de pago | 1 | 0 | 0 | `999` | `otrosmediosdepagoperu` | 1 | `8278ECD0-C337-4484-B008-5B5E65B0DFAF` |

> **Nota de mapeo de columnas:** el orden exacto de columnas no fue entregado junto con los datos; el mapeo arriba se infiere del orden de valores recibido y de la convención documentada en la sección histórica de este archivo (`MedioDePago, RequiereNumeroDeCuenta, ObligatorioEnProveedor, ClaveFormaDePago, Clave, ObligatorioEnCliente`), con `Activo` e `IdRegion` añadidos al final — consistente con que `IdRegion` es la columna que aporta RE-005 (ver actualización arriba). **Confirmar este mapeo contra el esquema real de `catMedioDePago` antes de construcción**, en particular si `ObligatorioEnCliente` es la 4ª o la 5ª columna de datos.

**Dos hallazgos que corrigen supuestos previos de este archivo:**
- **`ObligatorioEnCliente = 0` en las 4 filas** — confirma en datos la resolución de DUDA-076: el campo **no es obligatorio** para clientes de Región Perú.
- **`ClaveFormaDePago` SÍ tiene valor (`001`, `003`, `005`, `999`), no `NULL`.** La sección histórica de este archivo (y el DIS-SOL) asumían que esta columna quedaría en `NULL` para Perú por no ser catálogo SAT. La carga real la puebla con códigos internos de 3 dígitos — **no corresponden al catálogo SAT c_FormaPago mexicano** (que usa 2 dígitos, "01"/"02"/"03"...); son códigos internos propios de Perú. Ajustar la Nota #6 histórica y cualquier validación que asuma `ClaveFormaDePago IS NULL` para distinguir México de Perú — ahora debe distinguirse por `IdRegion`, no por nulidad de `ClaveFormaDePago`.

**INSERT real (reemplaza el placeholder obsoleto de la sección histórica):**
```sql
-- Catálogo real de medio de pago Perú — recibido del cliente, 2026-08-24
-- NOTA: verificar el orden real de columnas de catMedioDePago antes de ejecutar
INSERT INTO [dbo].[catMedioDePago]
    ([IdCatMedioDePago], [MedioDePago], [RequiereNumeroDeCuenta], [ObligatorioEnProveedor], [ObligatorioEnCliente], [ClaveFormaDePago], [Clave], [Activo], [IdRegion])
VALUES
    ('FDA4F4D2-0CF6-4679-80DD-06AE471A34E2', N'Depósito en cuenta',       1, 1, 0, '001', 'depositoencuentaperu',      1, '8278ECD0-C337-4484-B008-5B5E65B0DFAF'),
    ('2525809E-5B71-4286-8A5B-1154CF84294F', N'Transferencia de fondos',  1, 1, 0, '003', 'transferenciadefondosperu', 1, '8278ECD0-C337-4484-B008-5B5E65B0DFAF'),
    ('A5E8F79C-AC79-41F8-B1CF-6F447D9B8359', N'Tarjeta de débito',        1, 0, 0, '005', 'tarjetadedebitoperu',       1, '8278ECD0-C337-4484-B008-5B5E65B0DFAF'),
    ('5D1C0261-1081-4F51-AD0A-06F84553B68D', N'Otros medios de pago',     1, 0, 0, '999', 'otrosmediosdepagoperu',     1, '8278ECD0-C337-4484-B008-5B5E65B0DFAF');
```

> Si estos registros ya están cargados en BD (según el mensaje del usuario, "ya se actualizó"), este INSERT es documental — sirve para dejar trazabilidad de los valores vigentes, no para re-ejecutar. Validar con `SELECT * FROM catMedioDePago WHERE IdRegion = '8278ECD0-C337-4484-B008-5B5E65B0DFAF'` antes de reinsertar.

**Brechas bloqueantes restantes tras esta resolución:** B1 (cuentas GOLPERU, 0 registros) y B4 (catálogo de inconsistencias, transversal con RE-024) — ver tabla de brechas actualizada más abajo, sección histórica.

---

## Secciones originales (histórico — esquema `fccPagoCliente` / `SeqFolioCobro`, versión 1.0)

> Las secciones siguientes documentan el análisis de impacto BD previo al DIS-SOL. Se conservan como trazabilidad; **los nombres de tabla/columna citados abajo (`fccPagoCliente`, `SeqFolioCobro`, `TipoDeCambioMonedaFacturacion`) corresponden al esquema pre-DIS-SOL de RE-024** — ver la actualización arriba para los nombres vigentes (`fccCobroCliente`, `fccCobroClienteConsecutivo`, `TipoDeCambioFacturacion`).

---

## Resumen
Paso 1 wizard Validar Cobro para Region Peru. UI identica a MEX (RE-FU-024).
Diferencias: catMedioDePago interno (sin ClaveFormaDePago/SAT), cuentas GOLPERU,
TC vs PEN (no DOF), identificador fiscal RUC.
Sin Complemento de Pago posterior — vinculacion cobro/factura es solo operativa.

---

## Impacto en BD: MINIMO (reutiliza RE-FU-024)

| #   | Cambio                                                                     | Tipo      | Prioridad     |
| --- | -------------------------------------------------------------------------- | --------- | ------------- |
| 1   | INSERT catMedioDePago (catalogo interno Peru, sin ClaveFormaDePago)        | DML       | Alta (BRECHA) |
| 2   | INSERT EmpresaDatosBancarios + DatosBancarios GOLPERU                      | DML       | Alta (BRECHA) |
| 3   | Reutiliza: fccPagoCliente + campos nuevos (RE-FU-024)                      | Existente | -             |
| 4   | Reutiliza: catTipoInconsistenciaCobro + fccInconsistenciaCobro (RE-FU-024) | Existente | -             |
| 5   | Reutiliza: SeqFolioCobro (RE-FU-024)                                       | Existente | -             |

> **Sin nuevos DDL**. Todo el esquema fue creado en RE-FU-024.
> Los cambios son de DATOS (DML) que dependen de brechas operativas de PROQUIFA Peru.

---

## Reutilizacion de RE-FU-024

| Objeto | Creado en | Uso en Peru |
|--------|-----------|-------------|
| fccPagoCliente (Confirmado, Notas, etc.) | RE-FU-024 | Misma tabla - mismo flujo |
| catTipoInconsistenciaCobro | RE-FU-024 | Mismo catalogo (AplicaPaso='1') |
| fccInconsistenciaCobro | RE-FU-024 | Misma tabla - cobros PER |
| SeqFolioCobro | RE-FU-024 | Pendiente: global o por region |
| fccFolioPagoCliente -> CorreoRecibidoCliente | RE-FU-008 | Buzon Cobros PER |

---

## Diferencias MEX (RE-FU-024) vs PER (RE-FU-025) en BD

| Aspecto           | Mexico RE-FU-024                                    | Peru RE-FU-025                                           |
| ----------------- | --------------------------------------------------- | -------------------------------------------------------- |
| Forma de pago     | catMedioDePago.ClaveFormaDePago = SAT (01,02,03...) | catMedioDePago sin ClaveFormaDePago (catalogo interno)   |
| Cuenta destino    | EmpresaDatosBancarios GOL/MUN/PRO/PQF               | **EmpresaDatosBancarios GOLPERU (0 registros - BRECHA)** |
| Moneda base TC    | MXN (DOF/Banxico)                                   | PEN (fuente pendiente de definir)                        |
| ID fiscal cliente | DatosFacturacionCliente.RFC (13 chars)              | DatosFacturacionCliente.RFC (RUC 11 digits)              |
| Efecto Paso 3     | Genera Complemento de Pago (CFDI)                   | **SIN Complemento de Pago (solo operativo)**             |

---

## Catálogo Medio de Pago Peru (catMedioDePago) — Resolucion DUDA-076 (cerrada)

    catMedioDePago tiene ClaveFormaDePago varchar(2) NULLABLE
    -> Soporta registros SIN clave SAT (para Peru)
    -> NO requiere ALTER TABLE

**Resolucion DUDA-076:** el campo "Forma de Pago" se REGIONALIZA. Es obligatorio (ObligatorioEnCliente) SOLO para Mexico; para Peru deja de ser obligatorio. El cliente entrego el 18/08 la lista definitiva de medios de pago validos para Peru, pero como IMAGEN (no como texto). El catalogo exacto de valores para Peru NO esta disponible en texto todavia.

~~**DML propuesto (catalogo interno Peru - pendiente confirmar con Tesoreria):**~~ *(Propuesta placeholder anterior a DUDA-076 — NO ejecutar tal cual: los valores no fueron confirmados por el cliente, solo son ilustrativos.)*

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- PLACEHOLDER OBSOLETO (pre-DUDA-076) - NO ejecutar sin transcribir el catalogo real
    -- Catalogo interno medio de pago Peru (SUNAT no exige este campo fiscalmente)
    -- ClaveFormaDePago = NULL porque no es catalogo SAT
    INSERT INTO [dbo].[catMedioDePago]
        ([MedioDePago], [RequiereNumeroDeCuenta], [ObligatorioEnProveedor], [ClaveFormaDePago], [Clave], [ObligatorioEnCliente])
    VALUES
        (N'Transferencia bancaria', 1, 0, NULL, 'PER-TRF', 0),
        (N'Deposito bancario',      1, 0, NULL, 'PER-DEP', 0),
        (N'Cheque',                 0, 0, NULL, 'PER-CHQ', 0),
        (N'Efectivo',               0, 0, NULL, 'PER-EFE', 0);
    -- OBSOLETO: reemplazar por los valores reales una vez transcrita la imagen del cliente (18/08)

**GAP ABIERTO (2026-08-21, DUDA-076):** el cliente ya entrego la lista definitiva de medios de pago Peru, pero en formato imagen. Pendiente: transcribir esa imagen a valores de texto y generar el DML real de catMedioDePago para Peru. No usar el INSERT placeholder anterior como valores finales.

---

## Cuentas Bancarias GOLPERU (BRECHA - 0 registros)

    Validacion en BD:
    EmpresaDatosBancarios WHERE Empresa.Prefijo='GOLPERU' AND Activo=1 = 0 registros

| Dato faltante                       | Tabla                                  | Accion                     |
| ----------------------------------- | -------------------------------------- | -------------------------- |
| Banco(s) peruano(s) de Golocaer SAC | catBanco                               | INSERT si no existe        |
| Cuentas bancarias GOLPERU           | EmpresaDatosBancarios + DatosBancarios | INSERT                     |
| Moneda PEN en cuentas               | catMoneda                              | Verificar si PEN ya existe |

**Estructura a poblar:**

    EmpresaDatosBancarios
        IdEmpresa -> Empresa (GOLPERU)
        IdRegion -> Region (PER)
        IdDatosBancarios -> DatosBancarios

    DatosBancarios
        NumeroDeCuenta
        Clabe -> CCI (20 digitos) en lugar de CLABE (18)
        Sucursal
        IdCatBanco
        IdCatMoneda (PEN / USD)

---

## Tablas Leidas (Lectura) - Mismo patron que MEX

| Tabla | Datos leidos | Diferencia vs MEX |
|-------|-------------|-------------------|
| fccFolioPagoCliente | IdFCCFolioPagoCliente, Folio, FechaRecepcion | Igual |
| CorreoRecibidoCliente | Asunto, Cuerpo, Fecha | Igual |
| ArchivoCorreoRecibido | IdArchivo, NombreArchivo | Igual |
| Archivo | FileKey | Igual |
| fccPagoCliente | Folio, Monto, FechaPago, Confirmado | Igual |
| Cliente | Nombre, Alias | Igual |
| DatosFacturacionCliente | RFC (RUC en PER), RazonSocial, IdCatMoneda | RUC vs RFC |
| catMedioDePago | Clave PER-xxx (sin ClaveFormaDePago) | Catalogo interno vs SAT |
| catMoneda | ClaveMoneda (PEN/USD) | PEN vs MXN |
| **EmpresaDatosBancarios GOLPERU** | Cuentas Peru | **BRECHA: 0 registros** |
| DatosBancarios (CCI) | NumeroCuenta, Clabe (CCI), Sucursal | CCI vs CLABE |
| catBanco | Nombre banco peruano | Bancos PER vs MEX |
| catTipoInconsistenciaCobro | AplicaPaso='1' | Igual |

---

## Tablas Escritas (runtime) - Mismo patron que MEX

| Tabla | Momento | Operacion |
|-------|---------|-----------|
| fccPagoCliente | Al auto-guardar | INSERT/UPDATE (Confirmado=0) |
| fccPagoCliente | Al confirmar cobro | UPDATE Folio, Confirmado=1, FechaConfirmacion, IdUsuarioConfirmacion |
| fccInconsistenciaCobro | Al confirmar inconsistencia | INSERT |

---

## Foliador COB Peru

    fccPagoCliente.Folio = 'COB-mmddaa-NNNN'
        mmddaa = FORMAT(FechaPago,'MMddyy')
        NNNN = NEXT VALUE FOR dbo.SeqFolioCobro

> Pendiente confirmar: SeqFolioCobro global (MEX+PER) o consecutivo independiente por region.

---

## Efecto Paso 3 Peru: Sin Complemento de Pago

| Paso | Mexico | Peru |
|------|--------|------|
| Paso 1 | Captura cobro -> fccPagoCliente | Igual |
| Paso 2 | Asocia cobro a proforma/factura | Igual (operativo) |
| Paso 3 | Genera CFDI Complemento de Pago | **Solo cierra pendiente, SIN documento fiscal** |

> La factura peruana ya se emitio completa con IGV (RE-FU-020/022).
> No hay Complemento de Pago SUNAT. La conciliacion es registro operativo interno.

---

## Brechas Criticas

| # | Brecha | Bloqueante | Accion |
|---|--------|-----------|--------|
| B1 | Cuentas bancarias GOLPERU (0 registros) | SI | INSERT EmpresaDatosBancarios PER |
| ~~B2~~ | ~~Catalogo interno medio de pago Peru: valores entregados por cliente (18/08) como IMAGEN, pendiente transcripcion (DUDA-076)~~ **RESUELTA (2026-08-24):** catálogo recibido y cargado (4 registros) — ver actualización arriba | ~~SI~~ NO | ~~Transcribir imagen a texto + INSERT~~ Completado |
| B3 | Fuente TC Peru (no DOF) | ~~SI~~ NO — **RESUELTA (DP2, ver actualización 2026-08-24 arriba):** fuente BCRP, ya en `TipoDeCambioBanamex` | Cerrado |
| B4 | Catalogo TipoInconsistenciaCobro (compartido con MEX) | SI | Solicitar a Tesoreria |
| B5 | Consecutivo folio: global vs por region | Negocio | Confirmar con cliente |

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | Cuentas bancarias GOLPERU | Datos | INSERT (depende de Golocaer SAC) |
| ~~2~~ | ~~Catalogo medio pago interno Peru: campo ya regionalizado y NO obligatorio para Peru (DUDA-076); valores entregados por cliente el 18/08 como IMAGEN, pendiente de transcripcion a texto~~ **RESUELTO (2026-08-24):** catálogo de 4 registros recibido y cargado — ver actualización arriba | Negocio/Operativo | Completado |
| ~~3~~ | ~~Fuente oficial TC Peru~~ **RESUELTO (DP2, DIS-SOL 2026-08-24):** BCRP, ya en `TipoDeCambioBanamex` | Tecnico | Completado |
| 4 | Foliador COB global vs por region | Negocio | Confirmar con cliente |
| 5 | Buzon Cobros Peru sin datos | Operativo | Brechas modelo bancario Peru |
| 6 | Asistencia automatizada | Tecnico | No comprometido R16 |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-024 | Paso 1 MEX (toda la infraestructura BD compartida) |
| R16A-RE-FU-008 | Buzon Cobros (fccFolioPagoCliente PER) |
| R16A-RE-FU-006 | Cuentas bancarias GOLPERU (ClienteDatosBancarios) |
| R16A-RE-FU-023 | Pantalla principal VC (lista clientes PER) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
