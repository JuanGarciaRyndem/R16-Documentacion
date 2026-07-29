# Mantenimiento de Catálogo de Clientes — Información Fiscal del Cliente

| Campo | Valor |
|---|---|
| **ID** | R16A-RE-FU-004 |
| **Nombre** | Mantenimiento de Catálogo de Clientes |
| **Catálogo** | Catálogo de Clientes |
| **Categoría** | Funcional |
| **Estatus** | Propuesto |
| **Referencia Legacy** | Sin trazabilidad directa a la matriz original del cliente; emergente de sesiones |

---

## Historia de Usuario

> Yo como **usuario con acceso a la cartera de clientes**, quiero **capturar y mantener actualizada la información fiscal del cliente** (Identificador Fiscal, Tipo de Sociedad y Régimen Fiscal / Tributario) en el Catálogo de Clientes, para **garantizar que los documentos fiscales generados desde el sistema** (facturas, notas de crédito y demás) **cuenten con los datos correctos según la región del cliente** y cumplan con la normativa fiscal aplicable.

---

## Requisito

El sistema debe contar en la sección **Entrega y Facturación** del Catálogo de Clientes con tres campos obligatorios para la información fiscal del cliente: **Identificador Fiscal (RFC/RUC)**, **Tipo de Sociedad** y **Régimen Fiscal**. Los tres campos existen como un único campo por cada concepto en la pantalla, pero las **validaciones aplicadas y los catálogos de opciones disponibles varían en función de la Región del cliente** (México o Perú).

Para **Región México**, el Identificador Fiscal es el RFC con validación local de formato SAT, y los catálogos de Tipo de Sociedad y Régimen Fiscal se cargan en el sistema según la lista entregada por el cliente. Para **Región Perú**, el Identificador Fiscal es el RUC, y los catálogos de Tipo de Sociedad y Régimen Tributario se cargan en el sistema según la lista entregada por el cliente. Cualquier usuario con acceso a la cartera del cliente puede modificar los campos.

---

## Alcance

### Aplica a

- Clientes de México y Perú en el Catálogo de Clientes.
- Sección **“Entrega y Facturación”** dentro de la pantalla del cliente.
- Tres campos obligatorios al guardar el cliente: **Identificador Fiscal (RFC/RUC)**, **Tipo de Sociedad** y **Régimen Fiscal**.
- Validación local de formato del Identificador Fiscal para Región México (formato SAT).
- Validación de formato del Identificador Fiscal para Región Perú pendiente de definición (algoritmo Módulo 11 propuesto).
- Catálogos de opciones de Tipo de Sociedad y Régimen Fiscal diferenciados por Región, cargados en el sistema según la lista entregada por el cliente.
- Acceso libre a la edición de los campos por cualquier usuario con visibilidad sobre el cliente.

### No aplica a

- Consulta del padrón externo del SAT o SUNAT para verificar existencia y vigencia del Identificador Fiscal (la validación es exclusivamente local de formato).
- Otros campos de la sección Entrega y Facturación distintos de los tres mencionados (Identificador Fiscal, Tipo de Sociedad y Régimen Fiscal).

---

## Reglas de Negocio

**Regla 1 — Tres campos fiscales obligatorios al guardar el cliente**
Los tres campos fiscales (Identificador Fiscal, Tipo de Sociedad y Régimen Fiscal) son obligatorios al guardar el cliente. Si alguno está vacío, el guardado se bloquea.

**Regla 2 — Validación local del RFC para Región México**
Para clientes con Región = México, el Identificador Fiscal capturado se valida contra el formato SAT: 12 caracteres alfanuméricos para personas morales, 13 caracteres alfanuméricos para personas físicas. La validación es exclusivamente local; el sistema no consulta el padrón externo del SAT.

**Regla 3 — Validación local del RUC para Región Perú**
Para clientes con Región = Perú, el Identificador Fiscal capturado se valida localmente contra el formato y el algoritmo Módulo 11 de SUNAT. La validación es exclusivamente local; el sistema no consulta el padrón externo de SUNAT.

**Regla 4 — Algoritmo Módulo 11 para validación del dígito verificador del RUC**
La validación del dígito verificador del RUC peruano se ejecuta así:

1. Multiplicar los 10 primeros dígitos del RUC por los factores fijos `[5, 4, 3, 2, 7, 6, 5, 4, 3, 2]` respectivamente.
2. Sumar los 10 productos.
3. Calcular el residuo de dividir la suma entre 11.
4. Calcular resultado tentativo = `11 - residuo`.
5. Si resultado tentativo = 10 → dígito verificador esperado = **0**; si resultado tentativo = 11 → dígito verificador esperado = **1**; en cualquier otro caso → dígito verificador esperado = resultado tentativo.
6. Comparar el dígito verificador esperado con el 11º dígito del RUC capturado. Si coinciden, el RUC es válido; si no, se rechaza.

**Regla 5 — Catálogos de Tipo de Sociedad y Régimen Fiscal según Región**
Los catálogos de opciones de Tipo de Sociedad y Régimen Fiscal del cliente se presentan al usuario en función de la Región del cliente. La lista de opciones para cada Región es la cargada en el catálogo paramétrico del sistema según lo entregado por el cliente.

> ** La consolidación definitiva de las listas está pendiente de validación con el cliente. Ver archivo adjunto `R16A-RE-FU-004_Equivalencias_MX_PE.xlsx`. **

**Regla 6 — Edición sin restricción de rol**
Cualquier usuario con acceso a la cartera del cliente puede modificar los campos fiscales. La autorización proviene del acceso del usuario al cliente, no de un rol específico.

---

## Riesgos

**Riesgo 1 — Identificador Fiscal inválido aceptado por validación solo de formato**
Como el sistema valida exclusivamente el formato del RFC y RUC (sin consulta al padrón externo), un cliente podría tener un Identificador Fiscal con formato correcto pero que no exista realmente en el padrón fiscal del SAT o SUNAT. Esto causaría rechazo del documento fiscal al momento de timbrar, generando reproceso y fricción operativa.

**Riesgo 2 — Catálogos paramétricos desactualizados respecto a la normativa**
Los catálogos de Tipo de Sociedad y Régimen Fiscal del sistema requieren mantenimiento periódico para reflejar la normativa vigente del SAT y SUNAT (alta de nuevos códigos, baja de extintos). Si el catálogo del sistema no se mantiene sincronizado, los clientes podrían quedar con valores obsoletos que causen rechazo de timbrado.

---

## Criterios de Aceptación

### Sección A — Visualización y acceso a los campos

**Criterio A1 — Visualización de los tres campos fiscales en Entrega y Facturación**
- **Dado** que un usuario abre el Catálogo de Clientes y consulta un cliente específico,
- **Cuando** se renderiza la sección Entrega y Facturación,
- **Entonces** el sistema deberá presentar los tres campos: Identificador Fiscal (RFC/RUC), Tipo de Sociedad y Régimen Fiscal, mostrando los valores actualmente capturados o indicando claramente cuáles están vacíos.

**Criterio A2 — Edición habilitada sin restricción de rol**
- **Dado** que cualquier usuario con acceso al cliente abre la sección Entrega y Facturación,
- **Cuando** intenta modificar los campos fiscales,
- **Entonces** el sistema deberá permitir la edición sin requerir un rol específico.

### Sección B — Validación del Identificador Fiscal — México

**Criterio B1 — Captura del RFC válido para cliente México**
- **Dado** que el cliente tiene Región = México y el usuario captura el Identificador Fiscal,
- **Cuando** ingresa un RFC con formato SAT válido (12 caracteres alfanuméricos para personas morales, 13 para personas físicas),
- **Entonces** el sistema deberá aceptar el valor y permitir guardar el cliente.

**Criterio B2 — Rechazo de RFC con formato inválido**
- **Dado** que el cliente tiene Región = México y el usuario captura un Identificador Fiscal,
- **Cuando** el valor ingresado no cumple el formato SAT (longitud incorrecta, caracteres inválidos, etc.),
- **Entonces** el sistema deberá rechazar el valor indicando que el RFC no tiene un formato válido. El guardado del cliente se bloquea hasta corregir.

### Sección C — Validación del Identificador Fiscal — Perú

**Criterio C1 — Captura del RUC válido para cliente Perú**
- **Dado** que el cliente tiene Región = Perú y el usuario captura el Identificador Fiscal,
- **Cuando** ingresa un RUC de 11 dígitos numéricos cuyos primeros 2 dígitos son 10, 15, 17 o 20 y cuyo dígito verificador (posición 11) coincide con el calculado por el algoritmo Módulo 11,
- **Entonces** el sistema deberá aceptar el valor y permitir guardar el cliente.

**Criterio C2 — Rechazo de RUC con longitud o tipo de contribuyente inválido**
- **Dado** que el cliente tiene Región = Perú,
- **Cuando** el valor ingresado no tiene exactamente 11 dígitos numéricos o sus 2 primeros dígitos no están en `{10, 15, 17, 20}`,
- **Entonces** el sistema deberá rechazar el valor con mensaje específico indicando la causa del rechazo (longitud incorrecta o tipo de contribuyente inválido).

**Criterio C3 — Rechazo de RUC con dígito verificador incorrecto**
- **Dado** que el cliente tiene Región = Perú y el valor ingresado tiene 11 dígitos con tipo de contribuyente válido,
- **Cuando** el sistema calcula el dígito verificador mediante el algoritmo Módulo 11 y no coincide con el 11º dígito capturado,
- **Entonces** el sistema deberá rechazar el valor indicando que el RUC es inválido (dígito verificador incorrecto).

### Sección D — Selectores de Tipo de Sociedad y Régimen Fiscal

**Criterio D1 — Selector de Tipo de Sociedad con catálogo según Región**
- **Dado** que el usuario despliega el selector de Tipo de Sociedad,
- **Cuando** el sistema arma la lista de opciones,
- **Entonces** deberá presentar las opciones de Tipo de Sociedad cargadas en el catálogo del sistema para la Región del cliente (México o Perú).

**Criterio D2 — Selector de Régimen Fiscal con catálogo según Región**
- **Dado** que el usuario despliega el selector de Régimen Fiscal,
- **Cuando** el sistema arma la lista de opciones,
- **Entonces** deberá presentar las opciones de Régimen Fiscal cargadas en el catálogo del sistema para la Región del cliente (México o Perú).

### Sección E — Persistencia y consumo posterior

**Criterio E1 — Bloqueo de guardado por información fiscal incompleta**
- **Dado** que el usuario captura o edita un cliente con al menos uno de los tres campos fiscales vacío,
- **Cuando** intenta guardar los cambios,
- **Entonces** el sistema deberá bloquear el guardado. El cliente no se guarda hasta completar los tres campos.

**Criterio E2 — Persistencia de los datos fiscales**
- **Dado** que el usuario guarda exitosamente los campos fiscales del cliente,
- **Cuando** el sistema procesa la operación,
- **Entonces** deberá almacenar los tres valores asociados al cliente. Los datos quedan disponibles inmediatamente para consumo de los módulos posteriores que generen documentos fiscales para ese cliente.

---

## Notas de Implementación

- Funcionalidad ubicada en la sección **Entrega y Facturación** del cliente dentro del Catálogo de Clientes.
- Los tres campos fiscales son obligatorios al guardar el cliente.
- La pantalla presenta los campos como tres únicos campos por concepto, **sin renderizado condicional de campos distintos por país**. Lo que varía según la Región del cliente son las validaciones aplicadas y los catálogos de opciones disponibles.
- Cualquier usuario con acceso a la cartera del cliente puede modificar los campos. No existe restricción de rol específica para esta funcionalidad.
- Para **Región México**, la validación del RFC es exclusivamente de formato SAT (longitud 12 o 13 caracteres alfanuméricos). No se consulta el padrón SAT. **Esta validación ya está implementada en PQF2.**
- Para **Región Perú**, la validación del RUC propuesta es de formato (11 dígitos numéricos), tipo de contribuyente (primeros 2 dígitos en `{10, 15, 17, 20}`) y dígito verificador (algoritmo Módulo 11 sobre los 10 primeros dígitos contra el 11º). No se consulta el padrón SUNAT.

> ** PQF2 actualmente NO tiene validación de formato implementada para Perú. Pendiente confirmar con el cliente si se implementa la validación local propuesta o si el campo queda como captura libre. **

> ** Pendiente: confirmar con el cliente los catálogos de Tipo de Sociedad y Régimen Fiscal a cargar en PQF2. Se detectaron discrepancias entre el catálogo actual de PQF2, el archivo entregado por el cliente y el catálogo `c_RegimenFiscal` vigente del SAT (Anexo 20 CFDI 4.0): **
> - **(a)** El catálogo de PQF2 incluye 3 códigos (628, 629, 630) que no existen en el catálogo SAT vigente como Régimen Fiscal.
> - **(b)** El catálogo de PQF2 incluye el código 609 que está derogado por el SAT desde 2014.
> - **(c)** El archivo del cliente incluye códigos derogados (602, 604, 613, 618) y códigos no estándar (617, 619).
> - **(d)** PQF2 no tiene catálogos cargados para Perú actualmente.
>
> Ver archivo adjunto `R16A-RE-FU-004_Equivalencias_MX_PE.xlsx` con el detalle del cruce y las observaciones por código y opción.

> ** Pendiente: confirmar denominación correcta para Perú de la figura “Sociedad por Acciones Cerrada Simplificada” (S.A.C.S.) conforme al Decreto Legislativo N° 1409. El archivo entregado por el cliente la nombra “Sociedad Anónima Cerrada Simplificada”, denominación que no corresponde con la oficial. **

> ** Pendiente: aclarar con el cliente a qué categoría tributaria SUNAT se refiere la opción “Régimen para Personas Naturales” del catálogo de Régimen Fiscal Perú (probablemente Renta de Cuarta Categoría, Renta de Quinta Categoría o un híbrido para personas naturales con negocio que no califican en NRUS/RER/RMT/RG). **

> ** Pendiente: confirmar si la validación del RFC México debe considerar el cálculo de la homoclave (últimos 3 caracteres) o solo el formato y longitud. **

Para el detalle completo del estado actual de los catálogos en PQF2, el cruce con el archivo entregado por el cliente y la verificación contra el catálogo vigente del SAT, ver archivo adjunto `R16A-RE-FU-004_Equivalencias_MX_PE.xlsx`.
