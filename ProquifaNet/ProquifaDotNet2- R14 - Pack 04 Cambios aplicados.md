# 📋 Cambios Relevantes — Rama `develop-pack04`

> **Repositorio:** `ProquifaDotNet-R14` | **Base de comparación:** `origin/develop`  
> **Generado:** 2026-05-27

---

## 📧 1. Gestión de Correos Electrónicos (R14M-744 / R14M-750)

### 1.1 Reglas de Inmutabilidad del Correo VD

**Commit:** `44f9d8312` · *Carlos Ivan Morales* · `2026-03-18`  
**Ticket:** [R14M-744](https://newryndem.atlassian.net/browse/R14M-744)

El correo vinculado a **Venta Digital** queda protegido una vez asignado en `ContactoCliente`:

- ❌ No puede modificarse ni desactivarse.
- 📤 Correos ausentes en el payload se desactivan automáticamente.
- 🔄 Lógica alineada entre `PUT /Cliente/Transaccion` y `cotClienteCotizacionBO`.

| Archivo | Descripción |
|---|---|
| `ContactoBO.Extensions.cs` | Ajuste en extensiones de contacto |
| `CorreoElectronicoBO.cs` | Lógica de reglas sobre el correo |
| `ClienteTransaccionBO.cs` | Aplicación de reglas en transacción de cliente (+94 líneas) |
| `cotClienteCotizacionBO.cs` | Alineación de lógica en cotización (+119 líneas) |
| `GMContactosCliente.cs` | Modelo actualizado |

---

### 1.2 Migración de `CorreoElectronico` a `ListaCorreoElectronico`

**Commit:** `a1d6b6c80` · *Carlos Ivan Morales* · `2026-03-11`  
**Ticket:** [R14M-750](https://newryndem.atlassian.net/browse/R14M-750)

Elimina la propiedad `[Obsolete]` de correo único y migra **todos los usages** a la lista:

- `GMContacto` y `GMContactoCliente`: eliminada propiedad `CorreoElectronico` obsoleta.
- `UserRegistrationVDBO`: usa `IdCorreoElectronicoVentaDigital` obligatorio — lanza excepción si no está asignado al enviar a VD o IdentityServer.
- `cotClienteCotizacionBO`: guarda lista completa en `_GuardaContactos`, auto-asigna `Principal` si el front no lo envía.
- `ContactoClientetTransaccionBO`: loop sobre `ListaCorreoElectronico` al guardar correos.
- `PretramitarPedidoTransaccionBO`: actualizado para usar la nueva lista.

---

### 1.3 Exposición de Lista de Correos Ordenada en `ContactoDetalle`

**Commit:** `37a8cf5cc` · *Carlos Ivan Morales* · `2026-03-12`  
**Ticket:** [R14M-744](https://newryndem.atlassian.net/browse/R14M-744)

- Se expone la lista de correos **ordenada** en `ContactoDetalle` desde `ClienteTransaccionBO` (+69 líneas) y `cotClienteCotizacionBO` (+62 líneas).
- Modelo `GMContacto` actualizado con nuevos campos de correo.

---

### 1.4 Guardado de Múltiples Correos y Validaciones VD

**Commit:** `7677ff3c5` · *Carlos Ivan Morales* · `2026-03-11`  
**Ticket:** [R14M-744](https://newryndem.atlassian.net/browse/R14M-744)

| Archivo | Cambio |
|---|---|
| `ClienteTransaccionBO.cs` | Lógica de guardado para múltiples correos por contacto |
| `UserRegistrationVDBO.cs` | Validaciones reforzadas para correos de Venta Digital |
| `GMContactoClienteValidator.cs` | Nuevas reglas de validación |

---

## 📮 2. Correo a Regulatory Affairs (RE-FU-035 / R14M-803 / R14M-804 / R14M-805 / R14M-1679)

### 2.1 Implementación del Correo de Confirmación de Pedido

**Commit:** `359915aed` · *Isai Amaury Garcia* · `2026-03-09`  
**Ticket:** [R14M-804](https://newryndem.atlassian.net/browse/R14M-804)

- Nueva clase `CorreoTPConfirmacionPedido` con su plantilla XSLT (`CorreoTPConfirmacionPedido.xslt`).
- Lógica en `tpPedidoTramitarTransaccionBO` para **enviar correo a Regulatory Affairs** al detectar partidas controladas.
- Método para limpiar archivos temporales post-envío.

| Archivo | Tipo |
|---|---|
| `CorreoTPConfirmacionPedido.xslt` | Nueva plantilla de correo (+84 líneas) |
| `CorreoTPConfirmacionPedido.cs` | Nueva clase (+24 líneas) |
| `tpPedidoTramitarTransaccionBO.cs` | Lógica de envío (+82 líneas) |

---

### 2.2 Mejoras en Validaciones y Gestión de Mensajes

**Commit:** `1313367c9` · *Isai Amaury Garcia* · `2026-03-17`  
**Ticket:** [R14M-803](https://newryndem.atlassian.net/browse/R14M-803)

- Validaciones para productos de tipo **"Origen"** en `ActualizarCotCotizacionTransaccionBO.Extensions.cs` (+81 líneas).
- Condiciones ajustadas para retornar respuestas de éxito/error con mensajes consistentes.

---

### 2.3 Verificación de Configuración para Correos (Fail Fast)

**Commit:** `04d66245e` · *Isai Amaury Garcia* · `2026-03-18`  
**Ticket:** [R14M-805](https://newryndem.atlassian.net/browse/R14M-805)

- En `tpPedidoTramitarTransaccionBO`: si la variable de configuración del correo de Regulatory Affairs **no existe**, se registra una advertencia y se lanza excepción — el sistema no continúa sin esta configuración crítica.

---

### 2.4 Correo Receptor Dinámico y Fix Visual del Footer

**Commits:** `3a878357b`, `7a56b856`, `c8e1907ae` · `2026-05-27`  
**Ticket:** [R14M-1679](https://newryndem.atlassian.net/browse/R14M-1679)

- El receptor del correo a Regulatory Affairs pasó de ser **estático** a una **variable dinámica de configuración**.
- Fix del layout del footer en la plantilla `CorreoTPConfirmacionPedido.xslt`.
- Actualización del instalador `ProquifaDotNet.SendInBlue` (28.4 MB → 28.5 MB).

---

## 📜 3. Contratos de Cliente (RE-FU-012 / R14M-675 / R14M-676 / R14M-677 / R14M-723 / R14M-725)

### 3.1 Base: `ContratoClienteDetalle` y Tipo de Cambio

**Commits:** `634856b1d` → `2d1a67624` · *Juan David Garcia* · `2026-03-02 a 03-05`  
**Tickets:** [R14M-675](https://newryndem.atlassian.net/browse/R14M-675)

- Nuevos campos congelados del cliente y datos del contrato en `ContratoClienteDetalle`.
- Cambios en `ContratoClienteTipoCambio` — soporte para tipo de cambio de contratos con diferente proveedor.
- Actualización masiva de `Core.Pqf` en **38 proyectos** simultáneamente.

---

### 3.2 Guardado de `ContratoClienteMarca` y Configuración General

**Commit:** `0c2325bf3` · *Juan David Garcia* · `2026-03-04`  
**Ticket:** [R14M-675](https://newryndem.atlassian.net/browse/R14M-675)

Refactoring amplio en el módulo de contratos (18 archivos, +290 líneas netas):

| Archivo | Cambio |
|---|---|
| `ContratoClienteMarcaBO.cs` | Reestructuración del guardado y validación (+61 líneas) |
| `ContratoClienteMarcaConfiguracionBO.cs` | Nuevo soporte para configuración general (+75 líneas) |
| `ContratoClienteDetalle.cs` | Modelo expandido (+45 líneas) |
| `ValidarContratoClienteMarcaBO.cs` | Simplificación de validaciones (-16 líneas) |
| `ObtenerContratosContemporaneosMismasMarcasBO.cs` | Actualización de consulta |
| `ContratoClienteMarcaController.cs` | Nuevos endpoints (+27 líneas) |
| `vContratoClienteMarcaController.cs` | Nuevos endpoints (+31 líneas) |

---

### 3.3 Configuración de Familia y `spObtenerConfiguracionesProveedor`

**Commit:** `f870c0138` · *Juan David Garcia* · `2026-03-12`  
**Ticket:** [R14M-676](https://newryndem.atlassian.net/browse/R14M-676)

- Actualización de `ContratoClienteMarcaFamiliaConfiguracionBO` para ejecutar el SP `spObtenerConfiguracionesProveedor`.
- Actualización del tipo de cambio en contratos con diferente proveedor.

---

### 3.4 Configuración de Precio de Lista del Contrato

**Commit:** `22eebcf2d` · *Juan David Garcia* · `2026-03-17`  
**Ticket:** [R14M-677](https://newryndem.atlassian.net/browse/R14M-677)

| Archivo nuevo | Descripción |
|---|---|
| `ContratoClienteMarcaCongelarBO.cs` | Lógica de congelado de configuración (+47 líneas) |
| `ContratoClienteMarcaConfiguracionPrecioListaBO.Transaccion.cs` | Transacción de precio de lista (+79 líneas) |

- `ContratoClienteMarcaConfiguracionPrecioListaController`: endpoints actualizados (+31 líneas).

---

### 3.5 Monitor de Eventos para Contratos

**Commits:** `85fbc0ba5`, `a541184733` · *Osmar Calderon* · `2026-05-15`  
**Ticket:** [R14M-725](https://newryndem.atlassian.net/browse/R14M-725)

- Se registra evento en `MonitoreoOferta` al **agregar o eliminar** un contrato.
- Nueva constante en `ConstantesMonitoreoOferta`.
- Fix: tipo de acción corregido de `delete` → `update` en eliminación lógica.

| Archivo | Cambio |
|---|---|
| `ContratoClienteBO.Transaccion.cs` | Registro de evento al agregar (+28 líneas) |
| `ContratoClienteBO.cs` | Registro de evento al eliminar (+26 líneas) |
| `ConstantesMonitoreoOferta.cs` | Nueva constante de contrato |

---

### 3.6 Datos de Contrato en Productos

**Commit:** `2eb6817e4` · *Osmar Calderon* · `2026-05-13`  
**Ticket:** [R14M-723](https://newryndem.atlassian.net/browse/R14M-723)

- `ConfiguracionContratoCliente`: nuevas extensiones para exponer datos del contrato (+10 líneas).
- `CalculoPreciosClienteBO`: datos del contrato asociado a los productos de contrato (+17 líneas).

---

## 🚚 4. Logística y Pedidos

### 4.1 Fix — Moneda de Cliente en Tramitar Pedido

**Commit:** `eb8a11890` · *Agustin Antunez* · `2026-05-19`  
**Ticket:** [R14M-1659](https://newryndem.atlassian.net/browse/R14M-1659)

- **Bug:** se pasaba `IdCatMoneda` en lugar de `IdCliente` a `ObtenerMonedaCliente`.
- Se reemplaza el `Buscador` genérico por `DatosFacturacionClienteBO` para respetar el patrón BO del módulo.

| Archivo | Cambio |
|---|---|
| `ClienteBO.Extensions.cs` | Corrección del argumento (+4 líneas) |
| `FacturasPendientesClienteObj.cs` | Uso del BO correcto |

---

### 4.2 Fix — Filtrado de Contactos Activos en Validación de Correo Único

**Commit:** `ba99546a1` · *Agustin Antunez* · `2026-05-18`  
**Ticket:** [R14M-1655](https://newryndem.atlassian.net/browse/R14M-1655)

- **Bug:** contactos inactivos causaban **falsos positivos** en la validación de correo único cross-contact.
- Fix en `ClienteTransaccionBO.cs`: se filtra `contactos.Where(c => c.Activo)` antes de validar unicidad.

---

### 4.3 Fix — Campo "Tramitó" en PDF de Confirmación de Pedido

**Commit:** `018f595c5` · *Samuel Hernandez* · `2026-05-21`  
**Ticket:** [R14M-1671](https://newryndem.atlassian.net/browse/R14M-1671)

- **Bug:** el PDF mostraba el nombre del **EVI** en el campo "Tramitó" en lugar del **ESAC**.
- **Causa raíz:** el generador re-leía `tpPedido` desde BD antes de que el `IdUsuarioESAC` fuera guardado, obteniendo el valor antiguo del EVI.
- **Fix en** `ArchivoBOConfirmacionPedidoExtensionsPdf.cs`: se antepone `pedido.IdUsuarioESAC` (in-memory, ya actualizado) sobre el valor leído desde BD.

---

### 4.4 Guardado de `TramitadoSinOC` e `IdArchivoEvidenciaSinOC`

**Commits:** `3809d0eb6`, `c568f3f70`, `fa84e3b37` · *Samuel Hernandez* · `2026-03-09 a 03-11`

Propagación de los nuevos campos a lo largo del flujo de pedidos:

| Archivo | Cambio |
|---|---|
| `PretramitarPedidoTransaccionBO.cs` | Copia `TramitadoSinOC` e `IdArchivoEvidenciaSinOC` de `ppPedido` a `tpPedido` |
| `GeneradorProcesoMailBotTransaccionBO.cs` | Considera los nuevos campos al generar proceso (+54 líneas refactorizados) |
| `ppPedidoFactory.cs` | Refactoring de la fábrica (+18 líneas) |
| `pcPromesaDeCompraFactory.cs` | Refactoring de la fábrica |
| `CorreoRecibidoClienteToPretramitacionPedidoBO.cs` | Ajuste de transacción de buzones |
| `Docs/RN-Clientes.md` | **Nueva documentación** de reglas de negocio (362 líneas) |

---

## 🏦 5. Facturación y Venta Digital

### 5.1 Nuevo Intermediario `DatosFacturacionClienteVDBO`

**Commit:** `684bf072e` · *Osmar Calderon* · `2026-05-18`  
**Ticket:** [R14M-1641](https://newryndem.atlassian.net/browse/R14M-1641)

- Nueva clase `DatosFacturacionClienteVDBO` (77 líneas) que actúa como intermediario entre **Venta Digital** y **PQF2** al actualizar datos de facturación del cliente.
- Agregada al proyecto `Logic.Pqf.Logistica`.

---

## 📡 6. Monitor de Eventos de Productos

### 6.1 Monitor para Nuevos Productos en Contrato

**Commit:** `164afd445` · *Carlos Ivan Morales* · `2026-05-18`

- `ContratoClienteBO.Transaccion`: registra evento en `MonitoreoOferta` al agregar un producto a un contrato (+39 líneas).
- Nueva constante en `ConstantesMonitoreoOferta`.

---