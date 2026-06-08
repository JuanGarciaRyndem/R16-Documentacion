# Impacto en BD - Validar Cobro: Paso 1 Peru (Captura del Cobro)
**Requisito:** TPSC-RE-FU-025
**Base de Datos:** ProquifaDotNet
**Version:** 1.0

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

## Catálogo Medio de Pago Peru (catMedioDePago)

    catMedioDePago tiene ClaveFormaDePago varchar(2) NULLABLE
    -> Soporta registros SIN clave SAT (para Peru)
    -> NO requiere ALTER TABLE

**DML propuesto (catalogo interno Peru - pendiente confirmar con Tesoreria):**

    -- Created by GitHub Copilot in SSMS - review carefully before executing
    -- Catalogo interno medio de pago Peru (SUNAT no exige este campo fiscalmente)
    -- ClaveFormaDePago = NULL porque no es catalogo SAT
    INSERT INTO [dbo].[catMedioDePago]
        ([MedioDePago], [RequiereNumeroDeCuenta], [ObligatorioEnProveedor], [ClaveFormaDePago], [Clave], [ObligatorioEnCliente])
    VALUES
        (N'Transferencia bancaria', 1, 0, NULL, 'PER-TRF', 0),
        (N'Deposito bancario',      1, 0, NULL, 'PER-DEP', 0),
        (N'Cheque',                 0, 0, NULL, 'PER-CHQ', 0),
        (N'Efectivo',               0, 0, NULL, 'PER-EFE', 0);
    -- Ajustar opciones al catalogo real que defina PROQUIFA Tesoreria

---

## Cuentas Bancarias GOLPERU (BRECHA - 0 registros)

    Validacion en BD:
    EmpresaDatosBancarios WHERE Empresa.Prefijo='GOLPERU' AND Activo=1 = 0 registros

| Dato faltante | Tabla | Accion |
|---------------|-------|--------|
| Banco(s) peruano(s) de Golocaer SAC | catBanco | INSERT si no existe |
| Cuentas bancarias GOLPERU | EmpresaDatosBancarios + DatosBancarios | INSERT |
| Moneda PEN en cuentas | catMoneda | Verificar si PEN ya existe |

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
| B2 | Catalogo interno medio de pago Peru | SI | Definir con PROQUIFA Tesoreria + INSERT |
| B3 | Fuente TC Peru (no DOF) | SI | Confirmar fuente oficial (SBS?) |
| B4 | Catalogo TipoInconsistenciaCobro (compartido con MEX) | SI | Solicitar a Tesoreria |
| B5 | Consecutivo folio: global vs por region | Negocio | Confirmar con cliente |

---

## Gaps y Pendientes

| # | Gap | Tipo | Accion |
|---|-----|------|--------|
| 1 | Cuentas bancarias GOLPERU | Datos | INSERT (depende de Golocaer SAC) |
| 2 | Catalogo medio pago interno Peru | Negocio | Definir con Tesoreria |
| 3 | Fuente oficial TC Peru | Tecnico | Confirmar (propuesta: SBS Peru) |
| 4 | Foliador COB global vs por region | Negocio | Confirmar con cliente |
| 5 | Buzon Cobros Peru sin datos | Operativo | Brechas modelo bancario Peru |
| 6 | Asistencia automatizada | Tecnico | No comprometido R16 |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| TPSC-RE-FU-024 | Paso 1 MEX (toda la infraestructura BD compartida) |
| TPSC-RE-FU-008 | Buzon Cobros (fccFolioPagoCliente PER) |
| TPSC-RE-FU-006 | Cuentas bancarias GOLPERU (ClienteDatosBancarios) |
| TPSC-RE-FU-023 | Pantalla principal VC (lista clientes PER) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
