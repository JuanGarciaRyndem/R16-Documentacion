# Diseño — `PerfilFiscal` unificado México + Perú (IdRegion)

**Estado:** decisión tomada — se integran las 2 entidades (`PerfilFiscal` MX y `PerfilFiscalPeruProducto` PE) en una sola tabla `PerfilFiscal` que cubre IVA (México/SAT) e IGV (Perú/SUNAT), separando las filas por `IdRegion`.

**Sustituye a:** la recomendación de la Opción B (tablas separadas) de `Propuesta_PerfilFiscal_MultiPais_MX_PE.md`. Este diseño corresponde a una variante de la Opción A, con 2 cambios que resuelven las objeciones originales:

1. Perú deja de ser "atributo por producto" y se vuelve un **catálogo cerrado reutilizable**, con la misma forma que el mexicano (`Nombre`, `TasaOCuota`, `TipoFactor`, `ObjetoImp`, `Fundamento`).
2. Los catálogos de apoyo (`Impuesto`, `ObjetoImp`) se vuelven **neutrales por región** — las claves SAT son filas de la región México y las claves SUNAT son filas de la región Perú, en las mismas tablas.

**Documentos base:**
- `../Guia_Tecnica_Perfil_Fiscal_IVA_MX.md` — reglas de negocio del IVA (Eje 1 / Eje 2), sin cambio.
- `./Guia_Tecnica_Facturacion_Peru.md` — reglas de negocio del IGV, Percepción y catálogos SUNAT, sin cambio.
- `./Propuesta_PerfilFiscal_MultiPais_MX_PE.md` — análisis de opciones que este documento cierra.

---

## 1. Principio del diseño

Una sola pregunta con una sola respuesta: *"¿qué tratamiento fiscal tiene este producto en esta región?"* → una fila de `PerfilFiscal`, filtrada por `IdRegion`.

- **Producto/Familia** tienen una sola FK a `PerfilFiscal` (precedencia Producto → Familia, sin cambio respecto a la guía MX, sección 3.3).
- **Las reglas de negocio no se mezclan:** el Eje 2 mexicano (`ExportacionCode` en Pedido) y la Retención/Percepción peruana siguen siendo lógica por región en la capa de aplicación (`IPerfilFiscalResolver` + mapper por región, sin cambio respecto a la propuesta, sección 3.1).
- **Lo que se unifica es el almacenamiento y el contrato de lectura**, no el XML: `MxCfdiMapper` (CFDI 4.0) y `PeUblMapper` (UBL 2.1) siguen siendo implementaciones separadas.

---

## 2. Catálogos de apoyo

Los catálogos dejan el prefijo `Sat` (eran 100% México) y pasan a ser neutrales con `IdRegion`. `TipoFactor` es el único que se comparte sin región: sus 3 valores (`Tasa`, `Cuota`, `Exento`) aplican conceptualmente a ambos países.

```sql
-- Impuesto por región: IVA/ISR/IEPS (MX), IGV (PE)
CREATE TABLE Impuesto (
    Code VARCHAR(4) NOT NULL,
    IdRegion INT NOT NULL REFERENCES Region(IdRegion),
    Description NVARCHAR(50) NOT NULL,
    CONSTRAINT PK_Impuesto PRIMARY KEY (Code, IdRegion)
);
-- MX: '001' ISR, '002' IVA, '003' IEPS (claves SAT)
-- PE: '1000' IGV (clave SUNAT Catálogo 05)

CREATE TABLE TipoFactor (
    Code VARCHAR(10) PRIMARY KEY,    -- 'Tasa', 'Cuota', 'Exento'
    Description NVARCHAR(50) NOT NULL
);

-- Objeto de impuesto / afectación por región
CREATE TABLE ObjetoImp (
    Code VARCHAR(2) NOT NULL,
    IdRegion INT NOT NULL REFERENCES Region(IdRegion),
    Description NVARCHAR(100) NOT NULL,
    CodigoOficial VARCHAR(2) NULL,   -- clave que exige el XML del país (SAT c_ObjetoImp / SUNAT Cat 07)
    CONSTRAINT PK_ObjetoImp PRIMARY KEY (Code, IdRegion)
);
-- MX: '01' No objeto, '02' Sí objeto, '03' Sí objeto y no obligado al desglose,
--     '04' Sí objeto y no causa impuesto (CodigoOficial = mismo código SAT)
-- PE: 'GR' Gravado (CodigoOficial '10'), 'EX' Exonerado ('20'),
--     'IN' Inafecto ('30'), 'XP' Exportación ('40') — mapeo a SUNAT Catálogo 07
```

**Nota sobre `CodigoOficial` (PE):** el Catálogo 07 de SUNAT tiene 18 valores; el catálogo de `ObjetoImp` solo modela los 4 grupos que usan las operaciones onerosas de PROQUIFA. Si el cliente confirma operaciones gratuitas (retiros, bonificaciones), se agregan filas — no columnas.

---

## 3. Tabla `PerfilFiscal` unificada

```sql
CREATE TABLE PerfilFiscal (
    PerfilFiscalId INT IDENTITY(1,1) PRIMARY KEY,
    IdRegion INT NOT NULL REFERENCES Region(IdRegion),
    Nombre NVARCHAR(100) NOT NULL,
    ImpuestoCode VARCHAR(4) NOT NULL,
    TipoFactorCode VARCHAR(10) NOT NULL REFERENCES TipoFactor(Code),
    ObjetoImpCode VARCHAR(2) NOT NULL,
    TasaOCuota DECIMAL(6,6) NULL,        -- NULL únicamente si TipoFactorCode = 'Exento'
    Fundamento NVARCHAR(200) NULL,
    SujetoPercepcion BIT NOT NULL DEFAULT 0,   -- solo aplica a filas de Perú
    TasaPercepcion DECIMAL(5,2) NULL,          -- solo si SujetoPercepcion = 1
    CONSTRAINT FK_PerfilFiscal_Impuesto
        FOREIGN KEY (ImpuestoCode, IdRegion) REFERENCES Impuesto(Code, IdRegion),
    CONSTRAINT FK_PerfilFiscal_ObjetoImp
        FOREIGN KEY (ObjetoImpCode, IdRegion) REFERENCES ObjetoImp(Code, IdRegion),
    CONSTRAINT CK_TasaOCuota_Exento CHECK (
        (TipoFactorCode = 'Exento' AND TasaOCuota IS NULL) OR
        (TipoFactorCode <> 'Exento' AND TasaOCuota IS NOT NULL)
    ),
    CONSTRAINT CK_Percepcion CHECK (
        (SujetoPercepcion = 0 AND TasaPercepcion IS NULL) OR
        (SujetoPercepcion = 1 AND TasaPercepcion IS NOT NULL)
    )
);
```

Las FKs compuestas (`Code`, `IdRegion`) garantizan por diseño que una fila de la región México no pueda apuntar a un impuesto u objeto de la región Perú, y viceversa — no hace falta CHECK adicional por región.

**Restricción que queda en aplicación (no en DDL):** `SujetoPercepcion = 1` solo es válido en filas cuya `IdRegion` sea Perú. Un CHECK sobre `IdRegion` requeriría fijar el valor del Id en el DDL (frágil); se valida en `Application` al capturar el catálogo.

---

## 4. Seed inicial

### 4.1 Región México (IVA — SAT)

| Nombre | `ImpuestoCode` | `TipoFactorCode` | `ObjetoImpCode` | `TasaOCuota` | Fundamento |
| --- | --- | --- | --- | --- | --- |
| IVA General 16% | 002 | Tasa | 02 | 0.160000 | Art. 1 LIVA |
| IVA Tasa 0% | 002 | Tasa | 02 | 0.000000 | Art. 2-A LIVA |
| Exento | 002 | Exento | 02 | NULL | Art. 9 LIVA |

### 4.2 Región Perú (IGV — SUNAT)

| Nombre | `ImpuestoCode` | `TipoFactorCode` | `ObjetoImpCode` | `TasaOCuota` | Fundamento |
| --- | --- | --- | --- | --- | --- |
| IGV General 18% | 1000 | Tasa | GR | 0.180000 | Art. 1 Ley IGV (TUO D.S. 055-99-EF) |
| Exonerado | 1000 | Exento | EX | NULL | Apéndice I — Ley IGV |
| Inafecto | 1000 | Exento | IN | NULL | Art. 2 Ley IGV |
| Exportación 0% | 1000 | Tasa | XP | 0.000000 | Art. 33 Ley IGV |

`SujetoPercepcion`/`TasaPercepcion` inician en 0/NULL en todas las filas — se activan solo si el cliente confirma que Golocaer S.A.C. es Agente de Percepción (pregunta abierta de la guía Perú).

---

## 5. Diccionario de datos

| Nombre de tabla | Descripción | Columnas | Relaciones | Índices | Consideraciones especiales |
| --- | --- | --- | --- | --- | --- |
| `Impuesto` | Catálogo de impuestos por región (SAT MX / SUNAT PE) | `Code` VARCHAR(4) clave oficial; `IdRegion` INT región; `Description` NVARCHAR(50) | FK `IdRegion` → `Region` (N:1); referenciada por `PerfilFiscal` (FK compuesta) | PK compuesta (`Code`, `IdRegion`), clustered | Carga única; solo cambia si SAT/SUNAT modifican claves |
| `TipoFactor` | Tipo de factor del impuesto (Tasa/Cuota/Exento), compartido entre regiones | `Code` VARCHAR(10) PK; `Description` NVARCHAR(50) | Referenciada por `PerfilFiscal` (N:1) | PK clustered en `Code` | Sin `IdRegion` — valores conceptualmente neutrales |
| `ObjetoImp` | Objeto de impuesto (MX c_ObjetoImp) / afectación (PE Cat 07) por región | `Code` VARCHAR(2) clave interna; `IdRegion` INT; `Description` NVARCHAR(100); `CodigoOficial` VARCHAR(2) clave para el XML | FK `IdRegion` → `Region`; referenciada por `PerfilFiscal` (FK compuesta) | PK compuesta (`Code`, `IdRegion`), clustered | `CodigoOficial` PE mapea a SUNAT Catálogo 07; en MX coincide con `Code` |
| `PerfilFiscal` | Catálogo cerrado de tratamientos fiscales (IVA/IGV) asignable a Producto/Familia | `PerfilFiscalId` INT IDENTITY PK; `IdRegion` INT; `Nombre` NVARCHAR(100); `ImpuestoCode` VARCHAR(4); `TipoFactorCode` VARCHAR(10); `ObjetoImpCode` VARCHAR(2); `TasaOCuota` DECIMAL(6,6) NULL; `Fundamento` NVARCHAR(200) NULL; `SujetoPercepcion` BIT; `TasaPercepcion` DECIMAL(5,2) NULL | FK `IdRegion` → `Region`; FK compuesta (`ImpuestoCode`, `IdRegion`) → `Impuesto`; FK `TipoFactorCode` → `TipoFactor`; FK compuesta (`ObjetoImpCode`, `IdRegion`) → `ObjetoImp`; referenciada por Producto y Familia (override Producto → herencia Familia) | PK clustered en `PerfilFiscalId`; índice no clustered en `IdRegion` (filtrado por región en catálogos de pantalla) | CHECK `CK_TasaOCuota_Exento` (Exento ⇔ tasa NULL); CHECK `CK_Percepcion` (percepción exige tasa); regla "Percepción solo en filas PE" se valida en Application; FKs compuestas impiden mezclar claves de regiones distintas |

---

## 6. Impacto en ProquifaDotNet.Finanzas

- **`Finanzas.Infrastructure` (Scaffold EF Core):** una sola entidad `PerfilFiscal` + 3 catálogos — sin herencia TPT ni discriminadores de EF; `IdRegion` es una columna normal.
- **`Finanzas.Domain`:** entidad única `PerfilFiscal`; la regla Exento⇔NULL y Percepción⇔Tasa viven como invariantes de la entidad.
- **`Finanzas.Application`:** `IPerfilFiscalResolver` se simplifica — ya no decide entre tablas, solo filtra por `IdRegion` de la operación (precedencia Producto → Familia sin cambio).
- **Mappers XML:** sin cambio respecto a la propuesta — `MxCfdiMapper` (CFDI 4.0) lee `CodigoOficial` MX; `PeUblMapper` (UBL 2.1) lee `CodigoOficial` PE. Registro por DI como keyed services por región.
- **Retención/Percepción a nivel Cobro (PE):** fuera de esta tabla — sigue lo definido en la guía Perú (secciones 3 y 4).

---

## 7. Lo que este diseño NO cambia

- Las reglas de los 2 ejes mexicanos (Perfil Fiscal del producto + `ExportacionCode` del Pedido) — guía MX, secciones 3 y 4.
- Las reglas de negocio peruanas (IGV, Retención, Percepción, exclusión de Detracción) — guía Perú.
- Las preguntas abiertas con el cliente de ambas guías (nivel de asignación Producto/Familia, medicinas de patente, alcance Perú, agentes de Retención/Percepción).
- Los seeds PE solo se cargan cuando se cierre la decisión de alcance de Perú; el DDL ya queda preparado sin costo adicional.
