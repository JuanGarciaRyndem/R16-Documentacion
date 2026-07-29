# Diccionario de Datos — R16A-RE-FU-004 Información Fiscal del Cliente

| Campo | Valor |
|---|---|
| **Requisito** | R16A-RE-FU-004 |
| **Nombre** | Mantenimiento de Catálogo de Clientes — Información Fiscal del Cliente |
| **Base de datos** | ProquifaDotNet |
| **Servidor** | RYNL010 |

---

## Resumen Ejecutivo

Captura y mantenimiento de tres campos fiscales obligatorios por cliente: **Identificador Fiscal (RFC/RUC)**, **Tipo de Sociedad** y **Régimen Fiscal**. Los catálogos y validaciones varían según la Región del cliente (México o Perú).

---

## Modelo de Datos

```
Cliente  (IdRegion → Region)
    └── FK IdCliente
DatosFacturacionCliente  (RFC, IdCatRegimenFiscal, IdCatTipoSociedadMercantil)
    ├── FK IdCatRegimenFiscal         → catRegimenFiscal
    └── FK IdCatTipoSociedadMercantil → catTipoSociedadMercantil
Region  (México / Perú — determina validaciones y catálogos)
```

---

## Entidades Afectadas

| Objeto | Tipo | Impacto | Descripción |
|---|---|---|---|
| `DatosFacturacionCliente` | Tabla existente | Lectura / Escritura | Almacena los 3 campos fiscales: RFC, `IdCatRegimenFiscal`, `IdCatTipoSociedadMercantil` |
| `catRegimenFiscal` | Catálogo existente | Lectura / Datos iniciales | Régimenes fiscales disponibles por Región |
| `catTipoSociedadMercantil` | Catálogo existente | Lectura / Datos iniciales | Tipos de sociedad mercantil disponibles por Región |
| `Region` | Tabla existente | Lectura | Determina las validaciones y catálogos que aplican al cliente |
| `vDatosFacturacionCliente` | Vista existente | Lectura | Expone datos fiscales completos con valores resueltos de catálogos |

---

## 1. DatosFacturacionCliente

**Propósito:** Almacena la información fiscal del cliente: Identificador Fiscal (RFC/RUC), Régimen Fiscal y Tipo de Sociedad
**Creada:** 01/06/2020

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdDatosFacturacionCliente` | `uniqueidentifier` | 16 | NO | PK. Default: `NEWID()` |
| `IdCliente` | `uniqueidentifier` | 16 | NO | FK → `Cliente` |
| `RFC` | `varchar` | 50 | SÍ | Identificador fiscal: RFC (México) o RUC (Perú). Un solo campo para ambas regiones |
| `IdCatRegimenFiscal` | `uniqueidentifier` | 16 | SÍ | FK → `catRegimenFiscal`. **NULLABLE en BD, obligatorio en UI** |
| `IdCatTipoSociedadMercantil` | `uniqueidentifier` | 16 | SÍ | FK → `catTipoSociedadMercantil`. **NULLABLE en BD, obligatorio en UI** |
| `IdCatMoneda` | `uniqueidentifier` | 16 | SÍ | FK → `catMoneda` |
| `IdEmpresa` | `uniqueidentifier` | 16 | SÍ | FK → Empresa que factura |
| `IdCatRevision` | `uniqueidentifier` | 16 | SÍ | FK → `catRevision` |
| `IdCatUsoCFDI` | `uniqueidentifier` | 16 | SÍ | FK → `catUsoCFDI` |
| `IdCatMetodoDePagoCFDI` | `uniqueidentifier` | 16 | SÍ | FK → `catMetodoDePagoCFDI` |
| `IdCatTipoValidacion` | `uniqueidentifier` | 16 | SÍ | FK → `catTipoValidacion` |
| `IdCatMonedaTramitacion` | `uniqueidentifier` | 16 | SÍ | FK → `catMoneda` (tramitación) |
| `MismaEmpresaFacturaPublicaciones` | `bit` | 1 | NO | Default: `0` |
| `AddendaDeLineaDeOrden` | `bit` | 1 | NO | Default: `0` |
| `AddendaDeCorreo` | `bit` | 1 | NO | Default: `0` |
| `FechaRegistro` | `datetime` | 8 | NO | Default: `GETDATE()` |
| `FechaUltimaActualizacion` | `datetime` | 8 | NO | Default: `GETDATE()` |
| `Activo` | `bit` | 1 | NO | Default: `1` |

**Índices:**
- `PK_DatosFacturacionCliente` (Clustered): `IdDatosFacturacionCliente`
- `IX_DatosFacturacionCliente` (Non-Clustered): `IdCliente`, `IdCatMoneda`, `IdEmpresa`, `IdCatRevision`, `IdCatUsoCFDI`, `IdCatMetodoDePagoCFDI`, `IdCatTipoValidacion`, `IdCatRegimenFiscal`, `IdCatMonedaTramitacion`, `IdCatTipoSociedadMercantil`

**Hallazgos críticos:**
- El campo `RFC` (`varchar 50`) almacena tanto RFC (MX) como RUC (PE) sin distinción.
- `IdCatRegimenFiscal` e `IdCatTipoSociedadMercantil` son **NULLABLE en BD**, pero el requisito los define como **obligatorios**. La obligatoriedad se valida únicamente en capa aplicación.

---

## 2. catRegimenFiscal (Catálogo)

**Propósito:** Régimenes fiscales disponibles por Región
**Creada:** 01/06/2020

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdCatRegimenFiscal` | `uniqueidentifier` | 16 | NO | PK |
| `RegimenFiscal` | `varchar` | 3 | NO | Código SAT (MX) o código SUNAT (PE) |
| `Descripcion` | `varchar` | 120 | NO | Descripción del régimen |
| `Activo` | `bit` | 1 | NO | Default: `1` |

**Estado actual del catálogo en BD (México):**

| Código | Descripción | Estado |
|---|---|---|
| 601 | General de Ley Personas Morales | ✅ Vigente SAT |
| 603 | Personas Morales con Fines no Lucrativos | ✅ Vigente SAT |
| 605 | Sueldos y Salarios e Ingresos Asimilados | ✅ Vigente SAT |
| 606 | Arrendamiento | ✅ Vigente SAT |
| 607 | Régimen de Enajenación o Adquisición de Bienes | ✅ Vigente SAT |
| 608 | Demás ingresos | ✅ Vigente SAT |
| 609 | Consolidación | ❌ Derogado SAT desde 2014 |
| 610 | Residentes en el Extranjero sin EP en México | ✅ Vigente SAT |
| 611 | Ingresos por Dividendos | ✅ Vigente SAT |
| 612 | Personas Físicas con Actividades Empresariales | ✅ Vigente SAT |
| 614 | Ingresos por intereses | ✅ Vigente SAT |
| 615 | Ingresos por obtención de premios | ✅ Vigente SAT |
| 616 | Sin obligaciones fiscales | ✅ Vigente SAT |
| 620 | Sociedades Cooperativas de Producción | ✅ Vigente SAT |
| 621 | Incorporación Fiscal | ✅ Vigente SAT |
| 622 | Actividades Agrícolas Ganaderas Silvicolas y Pesqueras | ✅ Vigente SAT |
| 623 | Opcional para Grupos de Sociedades | ✅ Vigente SAT |
| 624 | Coordinados | ✅ Vigente SAT |
| 625 | Actividades Empresariales vía Plataformas Tecnológicas | ✅ Vigente SAT |
| 626 | Régimen Simplificado de Confianza | ✅ Vigente SAT |
| 628 | Hidrocarburos | ⚠️ No existe en catálogo SAT CFDI 4.0 |
| 629 | Régimenes Fiscales Preferentes y Empresas Multinacionales | ⚠️ No existe en catálogo SAT CFDI 4.0 |
| 630 | Enajenación de acciones en bolsa de valores | ⚠️ No existe en catálogo SAT CFDI 4.0 |
| N/A | N/A | Valor especial interno |

> **Pendiente:** No existen registros para Región Perú. Requiere carga de catálogo SUNAT. Ver archivo `R16A-RE-FU-004_Equivalencias_MX_PE.xlsx` para detalle de discrepancias.

---

## 3. catTipoSociedadMercantil (Catálogo)

**Propósito:** Tipos de sociedad mercantil disponibles por Región
**Creada:** 21/06/2022

| Columna | Tipo | Longitud | Nulo | Descripción |
|---|---|---|---|---|
| `IdCatTipoSociedadMercantil` | `uniqueidentifier` | 16 | NO | PK |
| `TipoSociedadMerdantil` | `varchar` | 100 | NO | Nombre del tipo de sociedad (⚠️ typo en columna: falta “a” en “Mercantil”) |
| `Abreviatura` | `varchar` | 20 | SÍ | Abreviatura oficial (S.A., S.A. de C.V., etc.) |
| `Activo` | `bit` | 1 | NO | Default: `1` |

**Estado actual del catálogo en BD (México):**

| Tipo de Sociedad | Abreviatura | Activo |
|---|---|---|
| Sociedad Anónima | S.A. | Sí |
| Sociedad Anónima de Capital Variable | S.A. de C.V. | Sí |
| Sociedad de Responsabilidad Limitada | S. de R.L. | Sí |
| Sociedad de Responsabilidad Limitada de Capital Variable | S. de R.L. de C.V. | Sí |
| Sociedad en Nombre Colectivo | S. en N.C. | Sí |
| Sociedad por Acciones Simplificada | SAS | Sí |

> **Pendiente:** No existen registros para Región Perú. Confirmar denominación “Sociedad por Acciones Cerrada Simplificada” (S.A.C.S.) conforme al D.L. N° 1409.

---

## 4. Region (Tabla de control)

**Propósito:** Determina las validaciones y catálogos que aplican al cliente

| Columna | Tipo | Longitud | Descripción |
|---|---|---|---|
| `IdRegion` | `uniqueidentifier` | 16 | PK |
| `Nombre` | `varchar` | 50 | México / Perú |
| `ClaveISO` | `varchar` | 50 | MEX / PER |
| `Clave` | `varchar` | 50 | mexico / peru |
| `Activo` | `bit` | 1 | Región activa |

**Regiones activas en sistema:**

| Nombre | ClaveISO | Aplicación en requisito |
|---|---|---|
| México | MEX | RFC 12/13 caracteres — catálogo SAT |
| Perú | PER | RUC 11 dígitos — algoritmo Módulo 11 — catálogo SUNAT |

---

## 5. vDatosFacturacionCliente (Vista)

**Propósito:** Expone datos fiscales completos del cliente con valores resueltos de catálogos

| Columna relevante | Descripción |
|---|---|
| `IdDatosFacturacionCliente` | PK de los datos de facturación |
| `IdCliente` | Cliente asociado |
| `RFC` | Identificador fiscal (RFC/RUC) |
| `IdCatRegimenFiscal` | ID del régimen fiscal |
| `IdCatTipoSociedadMercantil` | ID del tipo de sociedad |
| `Alias` | Nombre corto del cliente |

---

## Mapeo de Campos del Requisito a BD

| Campo en requisito | Tabla | Columna | Tipo | Observación |
|---|---|---|---|---|
| Identificador Fiscal (RFC/RUC) | `DatosFacturacionCliente` | `RFC` | `varchar(50)` | Un solo campo para MX y PE |
| Tipo de Sociedad | `DatosFacturacionCliente` | `IdCatTipoSociedadMercantil` | `uniqueidentifier` | FK NULLABLE — obligatorio en UI |
| Régimen Fiscal | `DatosFacturacionCliente` | `IdCatRegimenFiscal` | `uniqueidentifier` | FK NULLABLE — obligatorio en UI |
| Región del cliente | `Cliente` | `IdRegion` | `uniqueidentifier` | FK → `Region` (MEX/PER) |

---

## Validaciones por Región

| Aspecto | México (MEX) | Perú (PER) |
|---|---|---|
| Campo | RFC | RUC |
| Longitud | 12 (moral) / 13 (física) | 11 dígitos numéricos |
| Tipo contribuyente | No aplica | Primeros 2 dígitos en `{10, 15, 17, 20}` |
| Dígito verificador | Pendiente confirmar si se valida homoclave | Algoritmo Módulo 11 sobre los 10 primeros dígitos |
| Consulta padrón externo | NO — solo formato local | NO — solo formato local |
| Implementado en PQF2 | ✅ Sí | ❌ No — pendiente confirmar si se implementa |

---

## Consultas de Referencia

### Datos fiscales de un cliente por región

```sql
DECLARE @IdCliente UNIQUEIDENTIFIER;

SELECT
    dfc.RFC                         AS IdentificadorFiscal,
    rf.RegimenFiscal                AS CodigoRegimen,
    rf.Descripcion                  AS RegimenFiscal,
    ts.TipoSociedadMerdantil        AS TipoSociedad,
    ts.Abreviatura,
    r.ClaveISO
FROM       dbo.DatosFacturacionCliente       dfc
INNER JOIN dbo.Cliente                       c    ON dfc.IdCliente              = c.IdCliente
INNER JOIN dbo.Region                        r    ON c.IdRegion                 = r.IdRegion
LEFT  JOIN dbo.catRegimenFiscal              rf   ON dfc.IdCatRegimenFiscal     = rf.IdCatRegimenFiscal
LEFT  JOIN dbo.catTipoSociedadMercantil      ts   ON dfc.IdCatTipoSociedadMercantil = ts.IdCatTipoSociedadMercantil
WHERE dfc.IdCliente = @IdCliente
  AND dfc.Activo    = 1;
```

### Catálogo de Régimenes Fiscales vigentes por Región (México)

```sql
-- Códigos SAT vigentes (excluye derogados y no estándar)
SELECT IdCatRegimenFiscal, RegimenFiscal, Descripcion
FROM   dbo.catRegimenFiscal
WHERE  Activo       = 1
  AND  RegimenFiscal NOT IN ('609', '628', '629', '630', 'N/A')
ORDER BY RegimenFiscal;
```

### Catálogo de Tipos de Sociedad

```sql
SELECT IdCatTipoSociedadMercantil, TipoSociedadMerdantil, Abreviatura
FROM   dbo.catTipoSociedadMercantil
WHERE  Activo = 1
ORDER BY TipoSociedadMerdantil;
```

### Clientes con información fiscal incompleta

```sql
SELECT
    c.IdCliente,
    c.Nombre,
    r.ClaveISO                      AS Region,
    dfc.RFC,
    dfc.IdCatRegimenFiscal,
    dfc.IdCatTipoSociedadMercantil
FROM       dbo.Cliente               c
INNER JOIN dbo.Region                r    ON c.IdRegion   = r.IdRegion
LEFT  JOIN dbo.DatosFacturacionCliente dfc ON c.IdCliente = dfc.IdCliente
                                           AND dfc.Activo = 1
WHERE c.Activo = 1
  AND (
        dfc.IdDatosFacturacionCliente  IS NULL
     OR dfc.RFC                        IS NULL OR dfc.RFC = ''
     OR dfc.IdCatRegimenFiscal         IS NULL
     OR dfc.IdCatTipoSociedadMercantil IS NULL
      )
ORDER BY r.ClaveISO, c.Nombre;
```

---

## Reglas de Negocio Implementadas

| Regla   | Descripción                                     | Implementación                                        |
| ------- | ----------------------------------------------- | ----------------------------------------------------- |
| Regla 1 | 3 campos obligatorios al guardar                | Validación en capa aplicación (campos NULLABLE en BD) |
| Regla 2 | Validación RFC México (formato SAT)             | ✅ Ya implementada en PQF2                             |
| Regla 3 | Validación RUC Perú (formato)                   | ❌ Pendiente implementar                               |
| Regla 4 | Algoritmo Módulo 11 para dígito verificador RUC | ❌ Pendiente implementar en capa aplicación            |
| Regla 5 | Catálogos por Región                            | Catálogo único sin distinción — filtrado en UI        |
| Regla 6 | Edición sin restricción de rol                  | Sin control de rol en BD — acceso por cartera         |

---

## Análisis de Gaps

| Gap | Descripción | Acción requerida |
|---|---|---|
| Typo en columna BD | `catTipoSociedadMercantil.TipoSociedadMerdantil` (falta “a” en “Mercantil”) | Evaluar corrección — impacta dependientes |
| Catálogos Perú ausentes | `catRegimenFiscal` y `catTipoSociedadMercantil` sin registros Perú | Cargar catálogo SUNAT/DIGEMID al confirmar con cliente |
| Sin distinción por Región en catálogo | `catRegimenFiscal` no tiene campo Región / País | Agregar campo `IdRegion` o usar vista con filtro |
| Códigos derogados activos | 609 derogado SAT 2014; 628, 629, 630 no existen en SAT CFDI 4.0 | Desactivar o corregir según confirmación del cliente |
| NULLABLE vs Obligatorio | `IdCatRegimenFiscal` e `IdCatTipoSociedadMercantil` son NULL en BD | Obligatoriedad solo en UI — documentar como deuda técnica |
| Validación RUC Perú | PQF2 no tiene validación implementada para Perú | Confirmar con cliente: implementar Módulo 11 o captura libre |
| Homoclave RFC | Pendiente confirmar si se valida la homoclave (3 últimos caracteres) | Confirmar alcance de validación con cliente |
| Denominación S.A.C.S. Perú | Archivo cliente dice “Anónima” pero D.L. 1409 dice “por Acciones” | Confirmar nombre oficial con cliente |

---

## Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| 1 | RFC/RUC válido en formato pero inexistente en padrón | Sin consulta externa — aceptado por diseño |
| 2 | Catálogos desactualizados respecto a normativa SAT/SUNAT | Mantenimiento periódico del catálogo en BD |

---

## Pendientes

| # | Pendiente | Responsable |
|---|---|---|
| P1 | Confirmar catálogo definitivo de Régimen Fiscal MX (códigos a activar, desactivar o corregir) | Funcional / Cliente |
| P2 | Confirmar catálogo de Régimen Fiscal PE (SUNAT) y cargarlo en BD | Funcional / Cliente |
| P3 | Confirmar catálogo de Tipo de Sociedad PE y cargarlo en BD | Funcional / Cliente |
| P4 | Confirmar denominación oficial S.A.C.S. Perú conforme D.L. N° 1409 | Funcional / Cliente |
| P5 | Confirmar si la validación del RFC MX incluye cálculo de homoclave | Funcional / TechLead |
| P6 | Confirmar si la validación del RUC PE se implementa con Módulo 11 o queda como captura libre | Funcional / Cliente |
| P7 | Evaluar corrección del typo `TipoSociedadMerdantil` en BD (impacta dependientes) | DBA / TechLead |
| P8 | Definir si se agrega campo `IdRegion` a los catálogos o se filtra por vista | Arquitectura |
