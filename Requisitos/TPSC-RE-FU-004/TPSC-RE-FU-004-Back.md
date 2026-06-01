# Análisis de Impacto Backend — TPSC-RE-FU-004
# Catálogo de Clientes — Información Fiscal (RFC/RUC, Régimen Fiscal, Tipo de Sociedad Mercantil)

| Campo | Valor |
|---|---|
| **Requisito** | TPSC-RE-FU-004 |
| **Nombre** | Mantenimiento de Catálogo de Clientes — Información Fiscal |
| **Aplicativo** | ProquifaNet 2 |
| **Base de datos** | ProquifaDotNet |
| **Revisión aplicada** | TPSC-RE-FU-004_Revision.md |
| **Fecha** | 2026-05-29 |

---

## Resumen Ejecutivo

Habilitar la captura y validación de los tres campos fiscales obligatorios del cliente (Identificador Fiscal RFC/RUC, Régimen Fiscal y Tipo de Sociedad Mercantil) en la sección **Entrega y Facturación** del Catálogo de Clientes. Las validaciones de formato del identificador fiscal (RFC para México, RUC para Perú) se aplican según la Región del cliente. Los catálogos de Régimen Fiscal y Tipo de Sociedad Mercantil se consultan al sistema según la Región. Los tres campos son obligatorios al guardar.

---

## Reglas de Negocio (declarativas)

| #   | Regla                                                                                                                                                                                                                                                                                                                                                                                  |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1  | Los tres campos fiscales (Identificador Fiscal, Régimen Fiscal, Tipo de Sociedad Mercantil) son obligatorios al guardar el cliente. Si alguno está vacío, el guardado es bloqueado en la capa de aplicación.                                                                                                                                                                           |
| R2  | Para clientes de Región México, el campo RFC debe cumplir el formato SAT: cadena alfanumérica de 12 caracteres (persona moral) o 13 caracteres (persona física). **Esta validación ya está implementada en PQF2.**                                                                                                                                                                     |
| R3  | Para clientes de Región Perú, el campo RUC debe cumplir: 11 dígitos numéricos, primeros 2 dígitos en el conjunto `{10, 15, 17, 20}` (tipo de contribuyente), y dígito verificador válido por algoritmo Módulo 11 sobre los 10 primeros dígitos. **Esta validación no está implementada en PQF2.** Pendiente confirmar con el cliente si se implementa o queda como captura libre (P6). |
| R4  | El catálogo de Régimen Fiscal y el catálogo de Tipo de Sociedad Mercantil se presentan filtrados por la Región del cliente. El modelo de datos no tiene campo `IdRegion` en los catálogos; el filtrado actual es responsabilidad de la UI. Pendiente definir si se agrega campo Región a los catálogos o se resuelve mediante vista (P8).                                              |
| R5  | Cualquier usuario con acceso a la cartera del cliente puede modificar los campos fiscales. No hay restricción de rol.                                                                                                                                                                                                                                                                  |
| R6  | Los datos fiscales quedan disponibles inmediatamente para consumo de los módulos que generan documentos fiscales (facturas, CFDI) para el cliente.                                                                                                                                                                                                                                     |

---

## Entidades Afectadas

| Entidad | Tipo | Impacto | Descripción |
|---|---|---|---|
| `DatosFacturacionCliente` | Tabla existente | R / W | Almacena RFC/RUC, `IdCatRegimenFiscal`, `IdCatTipoSociedadMercantil` del cliente |
| `catRegimenFiscal` | Catálogo existente | R / Datos iniciales | Regímenes fiscales por Región (MX: SAT; PE: SUNAT — pendiente carga) |
| `catTipoSociedadMercantil` | Catálogo existente | R / Datos iniciales | Tipos de sociedad por Región (MX: vigente; PE: pendiente carga) |
| `Region` | Tabla existente | R | Determina validaciones y catálogos aplicables al cliente |
| `vDatosFacturacionCliente` | Vista existente | R | Expone datos fiscales completos con valores resueltos de catálogos |

---

## Análisis de GAPs

### GAP-01 — Validación de formato RUC Perú no implementada

**Estado:** ❌ No implementado

**Descripción:**
PQF2 tiene validación de formato RFC para México (ya implementada). No existe validación equivalente para el RUC de Perú. El requisito propone implementar:
1. Longitud de 11 dígitos numéricos.
2. Primeros 2 dígitos en el conjunto `{10, 15, 17, 20}` (tipo de contribuyente SUNAT).
3. Dígito verificador: algoritmo Módulo 11 sobre los 10 primeros dígitos contra el 11º.

Pendiente P6: confirmar con el cliente si se implementa esta validación local o si el campo queda como captura libre para Perú.

La Revisión indica que la validación debe diseñarse con el **formato almacenado en BD** asociado a cada Región (parámetrico), para no hardcodear la expresión regular. Esto implica evaluar agregar una tabla o campo de configuración de formato por Región.

**Archivos afectados:**
- `Logic.Pqf.Catalogos\Clientes\Configuracion\DatosFacturacionClientes\DatosFacturacionClienteBO.cs` (lógica de validación)
- Posiblemente nueva clase de extensión: `DatosFacturacionClienteBO.Validacion.Extensions.cs`

---

### GAP-02 — Catálogos de Región Perú ausentes en BD

**Estado:** ❌ No implementado (bloqueante para Perú)

**Descripción:**
Los catálogos `catRegimenFiscal` y `catTipoSociedadMercantil` no tienen registros para la Región Perú. Sin estos datos, el selector de Régimen Fiscal y Tipo de Sociedad no presenta opciones para clientes peruanos.

Adicionalmente, el catálogo México de `catRegimenFiscal` presenta inconsistencias respecto al SAT CFDI 4.0 vigente:
- Código **609** activo: derogado por el SAT desde 2014.
- Códigos **628, 629, 630** activos: no existen en el catálogo SAT CFDI 4.0.

Pendientes P1, P2, P3, P4 deben resolverse antes de ejecutar los scripts de carga.

**Entidades afectadas:**
- `catRegimenFiscal`: agregar registros SUNAT Perú; corregir/desactivar códigos MX inconsistentes (P1)
- `catTipoSociedadMercantil`: agregar registros Perú (P3, P4)

---

### GAP-03 — Sin distinción de Región en catálogos fiscales

**Estado:** ⚠️ Deuda técnica / Pendiente de decisión arquitectónica (P8)

**Descripción:**
Los catálogos `catRegimenFiscal` y `catTipoSociedadMercantil` no tienen campo `IdRegion`. El filtrado por Región se realiza actualmente en la UI. Para garantizar que el backend retorne solo las opciones válidas para la Región del cliente, se requiere una de las siguientes soluciones:

- **Opción A:** Agregar campo `IdRegion` a ambos catálogos y filtrar por región en las consultas.
- **Opción B:** Crear vistas `vCatRegimenFiscalPorRegion` y `vCatTipoSociedadMercantilPorRegion` que apliquen el filtro.

Pendiente P8: decisión de arquitectura.

---

### GAP-04 — Typo en columna `catTipoSociedadMercantil.TipoSociedadMerdantil`

**Estado:** ⚠️ Deuda técnica (P7)

**Descripción:**
La columna `TipoSociedadMerdantil` tiene un typo tipográfico (falta la "a" en "Mercantil"). La corrección impacta todos los componentes que referencian esta columna (modelos EF, consultas, vistas).

Pendiente P7: evaluar corrección con DBA y TechLead antes de ejecutar el cambio.

---

### GAP-05 — Endpoint de consulta de catálogos fiscales por Región no existe

**Estado:** ❌ No implementado

**Descripción:**
No existe un endpoint que retorne los catálogos de Régimen Fiscal y Tipo de Sociedad Mercantil filtrados por la Región del cliente. Los controladores existentes (`catRegimenFiscalController` y `catTipoSociedadMercantilController`) exponen CRUD genérico sin filtro por Región.

El frontend necesita consultar opciones válidas según la Región del cliente para poblar los selectores (Criterios D1 y D2 del requisito).

**Archivos afectados:**
- `WebApi.Catalogos\Controllers\Catalogos\catRegimenFiscalController.cs` (nuevo endpoint por Región)
- `WebApi.Catalogos\Controllers\Catalogos\catTipoSociedadMercantilController.cs` (nuevo endpoint por Región)

---

## Pendientes

| # | Pendiente | Responsable |
|---|---|---|
| P1 | Confirmar catálogo definitivo de Régimen Fiscal MX: códigos a activar, desactivar o corregir (609 derogado; 628, 629, 630 no estándar SAT CFDI 4.0) | Funcional / Cliente |
| P2 | Confirmar catálogo de Régimen Fiscal PE (SUNAT) y cargarlo en BD — incluyendo aclaración sobre "Régimen para Personas Naturales" | Funcional / Cliente |
| P3 | Confirmar catálogo de Tipo de Sociedad PE y cargarlo en BD | Funcional / Cliente |
| P4 | Confirmar denominación oficial "Sociedad por Acciones Cerrada Simplificada" (S.A.C.S.) conforme al D.L. N° 1409 | Funcional / Cliente |
| P5 | Confirmar si la validación del RFC MX incluye cálculo de homoclave (3 últimos caracteres) o solo formato y longitud | Funcional / TechLead |
| P6 | Confirmar si la validación del RUC PE se implementa con Módulo 11 o queda como captura libre | Funcional / Cliente |
| P7 | Evaluar corrección del typo `TipoSociedadMerdantil` en BD — impacta modelos EF, vistas y dependientes | DBA / TechLead |
| P8 | Definir si se agrega campo `IdRegion` a los catálogos o se filtra mediante vista por Región | Arquitectura |

---

## Criterios de Aceptación Técnica

| # | Criterio |
|---|---|
| C1 | El guardado de `DatosFacturacionCliente` es bloqueado si alguno de los tres campos fiscales está vacío o nulo. |
| C2 | Para cliente de Región México, el RFC es rechazado si no cumple el formato SAT (12 o 13 caracteres alfanuméricos). |
| C3 | Para cliente de Región Perú, el RUC es rechazado si no cumple el formato SUNAT (11 dígitos, tipo de contribuyente válido, dígito verificador Módulo 11) — condicionado a resolución de P6. |
| C4 | El selector de Régimen Fiscal retorna solo las opciones correspondientes a la Región del cliente. |
| C5 | El selector de Tipo de Sociedad Mercantil retorna solo las opciones correspondientes a la Región del cliente. |
| C6 | Los datos fiscales guardados están disponibles inmediatamente para los módulos generadores de documentos fiscales del cliente. |

---

## Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| R1 | RFC/RUC válido en formato pero inexistente en padrón SAT/SUNAT | Sin consulta externa — aceptado por diseño del requisito |
| R2 | Catálogos desactualizados respecto a normativa SAT CFDI 4.0 vigente | Resolver P1 antes de carga inicial; mantenimiento periódico |
| R3 | Corrección del typo `TipoSociedadMerdantil` puede romper dependientes no mapeados | Resolver P7 con análisis de impacto completo antes de ejecutar |
| R4 | Catálogos Perú no cargados bloquean operación para clientes PE en producción | Resolver P2, P3, P4 como prerequisito de despliegue para Perú |