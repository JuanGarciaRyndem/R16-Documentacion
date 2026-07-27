# R16A-RE-FU-000 — Revisión de la descripción en la matriz de requisitos

**Fecha:** 2026-07-27
**Alcance de la revisión:** descripción del requisito (Historia de Usuario, Requisito, Alcance, Criterios de Aceptación, Riesgos) contrastada contra `Analisis\Guia_Tecnica_Perfil_Fiscal_IVA_MX.md`, `Analisis\Facturación\Diseño_PerfilFiscal_Unificado_MX_PE.md` y la decisión de tabla `PerfilFiscal` unificada por `IdRegion`.

---

## Hallazgos mayores

### 1. Contradicción con la guía técnica — receptor extranjero ≠ tasa 0%

En "No aplica a" se lee: *"forzar tasa 0% cuando el receptor es extranjero"*.

La guía de Perfil Fiscal (sección 5) establece lo contrario: el RFC extranjero **no** determina la tasa por sí solo; lo que fuerza tasa 0% es la **exportación real con pedimento** (`ExportacionCode`, Ruta B — Art. 29 LIVA). Ejemplo de la propia guía: Pharma Scientific Inc. (EUA) recoge mercancía en la bodega de México sin pedimento de exportación → se factura al 16%.

**Corrección sugerida:** *"...las reglas de exportación (por ejemplo, forzar tasa 0% cuando existe una exportación real amparada con pedimento) no forman parte de este catálogo..."*

El **Riesgo 1** tiene la misma imprecisión (*"en exportación el SAT exige tasa 0% para cliente extranjero"*) — la tasa 0% la exige la exportación con pedimento, no la nacionalidad del cliente.

### 2. Nivel de asignación solo-Familia — cierra un pendiente abierto de la guía

La matriz define la configuración **exclusivamente a nivel Familia**, sin override por Producto. La guía (sección 3.3) definía precedencia Producto → Familia y marcaba el nivel de asignación como **pendiente de confirmar con el cliente**.

- Si el cliente ya lo confirmó: actualizar la guía MX (sección 3.3) y `Diseño_PerfilFiscal_Unificado_MX_PE.md` (la relación a nivel Producto sobra), y dejar constancia de la confirmación.
- Si no se ha confirmado: el requisito está cerrando unilateralmente un pendiente abierto — señalarlo antes de aprobar.

### 3. Brecha de alcance — ¿quién consume el perfil fiscal en Cotización/Pedido/Proforma?

Los criterios (A2, B1) solo cubren el armado y timbrado del CFDI. La guía (sección 6 y checklist) exige que **Cotización, Pedido y Proforma calculen IVA por línea** usando el perfil fiscal — nunca con tasa fija o booleano.

- Si ese consumo vive en otro requisito: referenciarlo como dependencia en la matriz.
- Si no existe: es una brecha — ninguna parte del alcance actual garantiza el cálculo por línea en documentos previos a la factura.

### 4. "Partidas de factura anticipo" no es una familia de productos

La Regla 6 mezcla en el mapeo por familia un concepto de facturación (anticipo → 84111506, clave de unidad ACT). Lo mismo aplica a **Fletes** si el flete es un concepto de la cotización y no un producto del catálogo.

**Definir:** ¿se resuelve con una familia virtual en BD, o es una regla del motor de facturación? Si es lo segundo, moverlo a "No aplica a" con referencia al requisito de facturación correspondiente.

### 5. Inconsistencia entre Criterio B2 y Regla 7 — ¿validación preventiva o fallo reactivo?

- **Criterio B2:** *"el sistema no permite facturar ese producto"* → validación preventiva.
- **Regla 7:** *"el intento de timbrado fallaría"* → fallo reactivo en el timbrado.

Son comportamientos distintos. Lo deseable es la validación preventiva; en ese caso B2 debe precisar **en qué punto** se valida (¿al armar la factura?, ¿al seleccionar el producto?) y Regla 7 alinearse.

### 6. Regla 4 cierra otro pendiente de la guía — medicinas de patente

*"IVA 16% a todos los productos, con excepción de las publicaciones (0%)"* — la guía (sección 2.1) dejaba pendiente confirmar si el catálogo incluye **medicinas de patente terminadas** (0% por Art. 2-A inciso b). Si el cliente confirmó que no las hay, dejar constancia; si no, el pendiente sigue abierto.

Adicional: el perfil **Exento** se define como opción pero ninguna familia lo usa — válido como catálogo completo, pero conviene una nota de por qué se incluye (previsión Art. 9 LIVA).

---

## Hallazgos menores

1. **Numeración de riesgos:** aparecen Riesgo 1 y Riesgo 3, falta Riesgo 2. Renumerar o indicar a dónde se movió.
2. **Redacción circular en "No aplica a":** *"Configuración fiscal para Perú: fuera del alcance del timbrado de Perú"*. Sugerido: *"Configuración fiscal para Perú: fuera de alcance en R16 (el timbrado de Perú no forma parte de este release)"*.
3. **Typo:** *"Se manejarà"* → *"Se manejará"* (Riesgo 3).
4. **Homologar terminología en Riesgo 3:** el título dice *"clave de servicio"* y el cuerpo *"clave de producto/servicio"* — usar siempre "clave de producto/servicio".

---

## Consistencias verificadas (sin observación)

- Las 3 opciones del perfil (IVA 16%, IVA 0%, Exento) coinciden con el seed MX del diseño unificado.
- Catálogos maestros del SAT precargados y no editables — consistente con la guía (sección 3.1) y con el diseño unificado (catálogos neutrales por región; a nivel de requisito es correcto no exponer nombres de tablas).
- Delegar la precedencia de tasas y exportación al motor de facturación — consistente con la separación Eje 1 / Eje 2 de la guía.
- Regla de IVA por familia (publicaciones 0% — Art. 2-A inciso i) — consistente con la guía (sección 2.1).
- Gestión sin interfaz gráfica en R16 vía BD — sin conflicto con documentación existente.
