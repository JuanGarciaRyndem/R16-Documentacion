# R16A-RE-FU-009 — Tareas BackEnd

| Campo | Valor |
|-------|-------|
| **Requisito** | R16A-RE-FU-009 |
| **Nombre** | Validación Regulatoria en Pretramitar Pedido |
| **Repositorio** | ProquifaDotNet |
| **Branch** | develop-pack04 |

---

## Dependencias con otros requisitos (tareas compartidas — NO duplicar)

| Tarea origen | Requisito | Descripción | Nota |
|:------------:|-----------|-------------|------|
| GAP-05 | R16A-RE-FU-003 | INSERT `catUsoArchivoSistema` (Licencia Sanitaria + Aviso RS MEX) | **Ya existe en RE-FU-003 Tarea 5.** No crear script duplicado. Es prerequisito de este requisito. |
| Tarea 1 | R16A-RE-FU-007 | ALTER FUNCTION `fnEsProductoControlado` — agregar `'origen'` | **Ya existe en RE-FU-007 Tarea 1.** No crear script duplicado. Es prerequisito de este requisito. |

> ⚠️ **Importante:** Las tareas T1 y T2 originales de este requisito fueron removidas porque ya están definidas en RE-FU-003 y RE-FU-007 respectivamente. Antes de iniciar T1 (ahora T3 renumerada), verificar que ambas dependencias estén ejecutadas.

---

## Índice de Tareas

| # | Clave Catálogo | Descripción corta |
|:-:|---------------|-------------------|
| T1 | `ALG-BASIC-LOGIC` | Crear ValidarDocumentosRegulatoriosBO |
| T2 | `ALG-BASIC-LOGIC` | Crear ResultadoValidacionRegulatoria (DTO) |
| T3 | `IMP-EXIST-SERVICE` | Modificar VerificarPedidoTramitableBO — integrar validación |
| T4 | `ALG-BASIC-LOGIC` | Verificar/ajustar ProductoBO.EsControlado() para origen |
| T5 | `UPDATE-TABL-CH` | Script BD: INSERT catUsoArchivoSistema (PER — pendiente) |

---

## T1

# [ R16A-RE-FU-009 ] [ALG-BASIC-LOGIC] Crear ValidarDocumentosRegulatoriosBO — lógica de validación regulatoria

---

### Aplicativos

- ProquifaDotNet

### Módulos

- Pretramitar Pedido

### Consideraciones previas

- Tareas T1 y T2 deben estar ejecutadas (catálogos y función actualizados).
- La tabla `ArchivoCliente` debe tener registros para poder probar (depende de R16A-RE-FU-003).
- Se puede probar con un INSERT manual de prueba en `ArchivoCliente` mientras RE-FU-003 no esté listo.
- **OBS-023:** La clase se ubica en `L05.TramitarPedido\Validaciones\`, NO en L04. Seguir el patrón de la carpeta `L05.TramitarPedido\Validaciones\`.
- El método principal ahora es `ValidarConResultadoParcial` que retorna qué partidas son tramitables y cuáles deben retenerse.

### Objetivo general

Crear la clase de validación regulatoria en `L05.TramitarPedido` que verifica si el cliente de un pedido tiene los documentos regulatorios requeridos, retornando resultado con detalle de partidas tramitables y retenidas.

### Objetivos específicos

1. Crear `Logic.Pqf.Logistica\L05.TramitarPedido\Validaciones\ValidarDocumentosRegulatoriosBO.cs`.
2. Implementar método `ValidarConResultadoParcial(Guid idPPPedido)` que retorna `ResultadoValidacionRegulatoria` extendido.
3. Lógica interna:
   - Verificar si el pedido tiene al menos 1 partida activa con producto controlado (mundiales, nacionales, origen):
     ```
     ppPartidaPedido → MarcaFamilia → Familia → catControl.Clave IN ('mundiales','nacionales','origen')
     ```
   - Si no tiene controlados → retornar válido (sin validación).
   - Obtener `IdCliente` del pedido: `ppPedido.IdContactoCliente → ContactoCliente.IdCliente`.
   - Obtener `Region` del pedido: `ppPedido.IdRegion → Region.ClaveISO`.
   - Determinar documentos requeridos según región:
     - MEX: 'Licencia Sanitaria', 'Aviso de Responsable Sanitario'
     - PER: [pendiente confirmar denominación]
   - Consultar `ArchivoCliente` con `IdCliente` + `IdCatUsoArchivoSistema` + `Activo = 1`.
   - Si falta alguno → retornar inválido con mensaje genérico.
4. El mensaje de bloqueo debe ser genérico (Regla 4 del requisito): *"No es posible procesar el pedido porque el cliente no cuenta con la documentación regulatoria requerida para productos controlados. Revise la sección de documentos regulatorios en la configuración del cliente."*

### Resultado esperado

- Clase funcional que detecta si un pedido con controlados puede avanzar según documentación regulatoria del cliente.

### Entregables

- `ValidarDocumentosRegulatoriosBO.cs`

### Criterios de aceptación

- [ ] Si el pedido NO tiene controlados → retorna válido sin ejecutar verificación de documentos.
- [ ] Si el pedido tiene controlados y el cliente tiene TODOS los docs requeridos → retorna válido.
- [ ] Si el pedido tiene controlados y FALTA al menos un documento → retorna inválido con mensaje genérico.
- [ ] La validación solo verifica presencia del registro (no vigencia, no contenido).
- [ ] Funciona para clientes de región MEX.
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores.

### Más información de la tarea

- Referencia: `R16A-RE-FU-009-Back.md` — PARTE 2, secciones 2.3 y 2.5
- Reglas de negocio: Regla 1 (solo si hay controlados), Regla 2 (docs según región), Regla 3 (solo presencia), Regla 4 (mensaje genérico)
- Queries de referencia: `R16A-RE-FU-009_BD.md` — secciones "Consulta SQL"
- Patrón existente: `Logic.Pqf.Logistica\L04.PretramitarPedido\Validaciones\`

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Logistica\Logic.Pqf.Logistica.csproj`

---

## T2

# [ R16A-RE-FU-009 ] [ALG-BASIC-LOGIC] Crear ResultadoValidacionRegulatoria — DTO de resultado

---

### Aplicativos

- ProquifaDotNet

### Módulos

- Pretramitar Pedido

### Consideraciones previas

- Clase de soporte para T1. Se extiende con soporte de resultado parcial (OBS-023).
- Seguir el patrón de DTOs en `L05.TramitarPedido\Validaciones\Models\`.

### Objetivo general

Crear el DTO que encapsula el resultado de la validación regulatoria, incluyendo listas de partidas tramitables y retenidas para soporte de entregas parciales (OBS-023).

### Objetivos específicos

1. Crear `Logic.Pqf.Logistica\L05.TramitarPedido\Validaciones\Models\ResultadoValidacionRegulatoria.cs`.
2. Propiedades:
   - `bool EsValido` — indica si la validación pasó.
   - `string Mensaje` — mensaje genérico de bloqueo (null si es válido).
   - `List<Guid> PartidasTramitables` — IDs de partidas que pueden tramitarse (OBS-023).
   - `List<Guid> PartidasRetenidas` — IDs de partidas controladas sin docs que quedan retenidas (OBS-023).
3. Métodos estáticos factory:
   - `Valido()` → instancia con `EsValido = true`.
   - `Invalido(string mensaje)` → instancia con `EsValido = false` y mensaje.
   - `Parcial(List<Guid> tramitables, List<Guid> retenidas)` → instancia para entrega parcial (OBS-023).

### Resultado esperado

- DTO reutilizable con soporte para resultado total (todo válido / todo bloqueado) y parcial (tramitar elegibles, retener controladas).

### Entregables

- `ResultadoValidacionRegulatoria.cs`

### Criterios de aceptación

- [ ] La clase expone `EsValido`, `Mensaje`, `PartidasTramitables` y `PartidasRetenidas`.
- [ ] Los métodos factory crean instancias correctas, incluyendo `Parcial()`.
- [ ] El proyecto compila sin errores.

### Más información de la tarea

- Referencia: `R16A-RE-FU-009-Back.md` — PARTE 2, sección 2.3

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Logistica\Logic.Pqf.Logistica.csproj`

---

## T3

# [ R16A-RE-FU-009 ] [IMP-EXIST-SERVICE] Modificar VerificarPedidoTramitableBO — integrar validación regulatoria

---

### Aplicativos

- ProquifaDotNet

### Módulos

- Pretramitar Pedido

### Consideraciones previas

- Tareas T1 y T2 completadas (`ValidarDocumentosRegulatoriosBO` y DTO existen).
- **OBS-023:** La validación regulatoria se mueve a `TramitarPedidoBO` (L05). `VerificarPedidoTramitableBO` ya NO debe llamar a `ValidarDocumentosRegulatoriosBO` — si se había agregado esa llamada, revertirla.
- Esta tarea se limita a confirmar/asegurar que `VerificarPedidoTramitableBO` NO tenga la llamada regulatoria.

### Objetivo general

Verificar que `VerificarPedidoTramitableBO.Procesar()` no contiene llamada a `ValidarDocumentosRegulatoriosBO` (OBS-023: esa lógica se mueve a `TramitarPedidoBO`). Si se había agregado, revertirla.

### Objetivos específicos

1. Revisar `Logic.Pqf.Logistica\L04.PretramitarPedido\Tramite\VerificarPedidoTramitableBO.cs`.
2. Si contiene llamada a `ValidarDocumentosRegulatoriosBO`, removerla.
3. Asegurar que las validaciones existentes (facturación, configuración, partidas) permanecen sin cambios.

### Resultado esperado

- `VerificarPedidoTramitableBO` sin llamada regulatoria. La validación de docs regulatorios ocurre en `TramitarPedidoBO` (T7).

### Entregables

- `VerificarPedidoTramitableBO.cs` verificado/revertido.

### Criterios de aceptación

- [ ] `VerificarPedidoTramitableBO.Procesar()` no contiene referencia a `ValidarDocumentosRegulatoriosBO`.
- [ ] Las validaciones existentes siguen funcionando sin cambios.
- [ ] El proyecto compila sin errores.

### Más información de la tarea

- Referencia: `R16A-RE-FU-009-Back.md` — PARTE 2, sección 2.4 (OBS-023)
- Archivo a revisar: `Logic.Pqf.Logistica\L04.PretramitarPedido\Tramite\VerificarPedidoTramitableBO.cs`

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Logistica\Logic.Pqf.Logistica.csproj`

---

## T4

# [ R16A-RE-FU-009 ] [ALG-BASIC-LOGIC] Verificar/ajustar ProductoBO.EsControlado() para cubrir 'origen'

---

### Aplicativos

- ProquifaDotNet

### Módulos

- Catálogos
- Pretramitar Pedido

### Consideraciones previas

- Dependencia RE-FU-007 Tarea 1 ejecutada (la función BD `fnEsProductoControlado` ya incluye `origen`).
- El método `ProductoBO.EsControlado()` en `ProductoBO.TipoExtensions.cs` usa `ProductoMarcaFamilia.Controlado` (campo `bit`).
- Verificar si este campo se actualiza automáticamente cuando `catControl` incluye `origen` (posible trigger o proceso batch).
- Si `ProductoMarcaFamilia.Controlado` NO se actualiza automáticamente con `origen`, el método `EsControlado()` no detectará productos de origen y la validación en T3 debe usar query directa a `catControl`.

### Objetivo general

Verificar que el método `ProductoBO.EsControlado()` retorna `true` para productos de familias con `catControl.Clave = 'origen'` y, si no lo hace, ajustar la lógica.

### Objetivos específicos

1. Verificar cómo se puebla `ProductoMarcaFamilia.Controlado`:
   - ¿Trigger en BD?
   - ¿Proceso batch?
   - ¿Se calcula al guardar el producto?
2. Probar: un producto con `Familia.catControl.Clave = 'origen'` → ¿`ProductoMarcaFamilia.Controlado = true`?
3. Si NO cubre `origen`:
   - **Opción A**: Actualizar el proceso que puebla `Controlado` para incluir `origen`.
   - **Opción B**: Ajustar `ProductoBO.EsControlado()` para hacer query directa incluyendo `origen`.
   - **Opción C**: En `ValidarDocumentosRegulatoriosBO` (T3) usar query directa a `catControl` en vez de depender de `ProductoMarcaFamilia.Controlado`.
4. Documentar la decisión tomada.

### Resultado esperado

- Los productos de familias `origen` son correctamente detectados como controlados por el sistema.

### Entregables

- Informe de verificación (si no requiere cambios).
- Archivo modificado (si requiere ajuste): `ProductoBO.TipoExtensions.cs` o proceso de actualización de `Controlado`.

### Criterios de aceptación

- [ ] Un producto con familia de tipo `origen` es identificado como controlado.
- [ ] El método `EsControlado()` retorna `true` para productos `mundiales`, `nacionales` y `origen`.
- [ ] No se rompen funcionalidades existentes.
- [ ] Se documenta la decisión (opción A, B o C).

### Más información de la tarea

- Referencia: `R16A-RE-FU-009-Back.md` — PARTE 2, sección 2.4
- Archivo: `Logic.Pqf.Catalogos\Productos\ProductoBO.TipoExtensions.cs`
- Referencia: `TramitarPedidoBO.cs` línea ~69: `productoBo.EsControlado(ppPartidaPedido.IdProducto, cliente.IdRegion)`

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Catalogos\Logic.Pqf.Catalogos.csproj`

---

## T5

# [ R16A-RE-FU-009 ] [UPDATE-TABL-CH] Script BD: INSERT catUsoArchivoSistema — documentos regulatorios PER

---

### Aplicativos

- ProquifaDotNet (Base de Datos — SQL Server RYNL010)

### Módulos

- Pretramitar Pedido
- Catálogo de Clientes

### Consideraciones previas

- **BLOQUEADA** — La denominación exacta de los documentos DIGEMID para Perú NO está confirmada por el cliente.
- No ejecutar hasta que se resuelva el Gap #1 documentado en `R16A-RE-FU-009-Back.md`.
- La tarea T1 (`ValidarDocumentosRegulatoriosBO`) debe estar preparada para soportar documentos PER una vez que se inserten aquí.

### Objetivo general

Insertar en `catUsoArchivoSistema` los tipos de documento regulatorio requeridos para Región Perú (normativa DIGEMID), completando la cobertura regulatoria para ambas regiones.

### Objetivos específicos

1. Confirmar con el cliente la denominación exacta de los 2 documentos DIGEMID.
2. INSERT registro [Documento DIGEMID 1] con `Activo = 1`.
3. INSERT registro [Documento DIGEMID 2] con `Activo = 1`.
4. Script idempotente.

### Resultado esperado

- `catUsoArchivoSistema` contiene los registros PER activos y la validación regulatoria funciona para clientes de Perú.

### Entregables

- Script SQL de migración (idempotente).
- Script SQL de rollback.

### Criterios de aceptación

- [ ] La denominación fue confirmada por el cliente.
- [ ] El script se ejecuta sin errores en RYNL010.
- [ ] Los registros existen con `Activo = 1` tras la ejecución.
- [ ] La validación regulatoria funciona correctamente para clientes PER.
- [ ] El script es idempotente.

### Más información de la tarea

- Referencia: `R16A-RE-FU-009_BD.md` — sección "catUsoArchivoSistema"
- Gap: `R16A-RE-FU-009-Back.md` — PARTE 7, Gap #1
- Pendiente: denominación exacta pendiente de cliente (transversal con RE-FU-003 y RE-FU-007)

### Recursos

- Servidor: RYNL010
- Base de datos: ProquifaDotNet

---

## T6

# [ R16A-RE-FU-009 ] [UPDATE-TABL-CH] ALTER TABLE ppPedido + tpPedido — campos entregas parciales y pedido hijo (OBS-023)

---

### Aplicativos

- ProquifaDotNet (Base de Datos — SQL Server RYNL010)

### Módulos

- Tramitar Pedido

### Consideraciones previas

- **OBS-023:** Nuevos campos requeridos para soporte de entregas parciales con sustancias controladas.
- `ppPedido.AceptaEntregasParciales` se captura en la pantalla de Pretramitar antes de tramitar.
- `tpPedido.IdPedidoOrigenControlado` se usa cuando se crea un pedido hijo para las partidas controladas retenidas — apunta al `tpPedido` padre.
- Ejecutar antes de T7 (bifurcación) y T8 (pedido hijo).

### Objetivo general

Agregar los campos `AceptaEntregasParciales` en `ppPedido` e `IdPedidoOrigenControlado` en `tpPedido` para soportar el flujo de entregas parciales con controlados sin documentación.

### Objetivos específicos

1. ALTER TABLE `ppPedido` — agregar `AceptaEntregasParciales bit NOT NULL DEFAULT(0)`.
2. ALTER TABLE `tpPedido` — agregar `IdPedidoOrigenControlado uniqueidentifier NULL` con FK self-referencing a `tpPedido`.
3. Scripts idempotentes.

### Resultado esperado

- Los campos existen en BD. El flujo de bifurcación (T7) y creación de pedido hijo (T8) pueden usarlos.

### Entregables

- Script DDL: ALTER `ppPedido` + ALTER `tpPedido`
- Script de rollback

### Criterios de aceptación

- [ ] `ppPedido.AceptaEntregasParciales bit NOT NULL DEFAULT(0)` existe en la tabla.
- [ ] `tpPedido.IdPedidoOrigenControlado uniqueidentifier NULL` existe con FK a `tpPedido(IdTpPedido)`.
- [ ] Scripts son idempotentes.
- [ ] PR aprobado por DBA y líder técnico.

### Más información de la tarea

- OBS-023: bifurcación por entregas parciales
- Referencia: `R16A-RE-FU-009_BD.md` — sección "Cambios Estructurales (OBS-023)"

### Recursos

- Servidor: RYNL010
- Base de datos: ProquifaDotNet

---

## T7

# [ R16A-RE-FU-009 ] [ALG-COMPLX-LOGIC] Implementar bifurcación de entregas parciales en TramitarPedidoBO (OBS-023)

---

### Aplicativos

- ProquifaDotNet

### Módulos

- Tramitar Pedido (L05)

### Consideraciones previas

- **Depende de T6** (campos en BD) y **T1** (`ValidarDocumentosRegulatoriosBO` en L05).
- La bifurcación ocurre dentro de `TramitarPedidoBO.Process()` después de separar partidas controladas / no controladas.
- Si `AceptaEntregasParciales = 0` y hay controladas sin docs → bloquear completo (comportamiento anterior).
- Si `AceptaEntregasParciales = 1` y hay controladas sin docs → tramitar elegibles + llamar T8 para el pedido hijo.

### Objetivo general

Implementar la lógica de bifurcación en `TramitarPedidoBO.Process()` que, cuando existen controladas sin docs y el usuario aceptó entregas parciales, tramita las partidas elegibles y delega la creación del pedido hijo a `CrearPedidoHijoControladoBO`.

### Objetivos específicos

1. Modificar `Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\TramitarPedidoBO.cs`.
2. Llamar a `ValidarDocumentosRegulatoriosBO.ValidarConResultadoParcial(idPPPedido)`.
3. Si resultado = Parcial AND `ppPedido.AceptaEntregasParciales = 1`:
   - Tramitar solo las `PartidasTramitables`.
   - Invocar `CrearPedidoHijoControladoBO.Crear(partidasRetenidas, idPedidoPadre)`.
4. Si resultado = Invalido AND `ppPedido.AceptaEntregasParciales = 0` → lanzar excepción con mensaje genérico.

### Resultado esperado

- Pedidos con AceptaEntregasParciales=1 se tramitan parcialmente, reteniendo controladas sin docs.

### Entregables

- `TramitarPedidoBO.cs` modificado

### Criterios de aceptación

- [ ] Pedido con controlados + docs + AceptaEntregasParciales=0 → tramita todo normalmente.
- [ ] Pedido con controlados + sin docs + AceptaEntregasParciales=0 → bloquea completo.
- [ ] Pedido con controlados + sin docs + AceptaEntregasParciales=1 → tramita elegibles + crea pedido hijo.
- [ ] El proyecto compila sin errores.

### Más información de la tarea

- OBS-023
- Referencia: `R16A-RE-FU-009-Back.md` — PARTE 2, sección 2.2 y 2.4

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Logistica\Logic.Pqf.Logistica.csproj`

---

## T8

# [ R16A-RE-FU-009 ] [SERV-TRANSACT] Crear pedido hijo para partidas controladas retenidas (OBS-023)

---

### Aplicativos

- ProquifaDotNet

### Módulos

- Tramitar Pedido (L05)

### Consideraciones previas

- **Depende de T6, T7.**
- El pedido hijo es un `tpPedido` independiente que contiene solo las partidas controladas retenidas.
- El campo `tpPedido.IdPedidoOrigenControlado` en el hijo apunta al `IdTpPedido` del padre.
- El pedido hijo queda en estado pendiente hasta que se registren los documentos regulatorios.
- Perú nunca transfiere a Legacy — restricción transversal.

### Objetivo general

Crear `CrearPedidoHijoControladoBO` que genera un `tpPedido` hijo a partir de las partidas controladas retenidas, vinculado al pedido padre mediante `IdPedidoOrigenControlado`.

### Objetivos específicos

1. Crear `Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\CrearPedidoHijoControladoBO.cs`.
2. Método `Crear(List<ppPartidaPedido> partidasRetenidas, Guid idTpPedidoPadre)`:
   - Genera un nuevo `tpPedido` con las partidas controladas retenidas.
   - Asigna `IdPedidoOrigenControlado = idTpPedidoPadre`.
   - Estado inicial: pendiente de documentación regulatoria.
3. La creación es atómica con la tramitación del padre (misma transacción cuando sea posible).

### Resultado esperado

- Pedido hijo creado en BD con partidas controladas retenidas y vínculo al padre.

### Entregables

- `CrearPedidoHijoControladoBO.cs` (nuevo)

### Criterios de aceptación

- [ ] Se genera un `tpPedido` hijo con `IdPedidoOrigenControlado = IdTpPedido` del padre.
- [ ] El pedido hijo contiene solo las partidas controladas retenidas.
- [ ] El pedido padre NO contiene las partidas retenidas.
- [ ] El proyecto compila sin errores.

### Más información de la tarea

- OBS-023
- Referencia: `R16A-RE-FU-009-Back.md` — PARTE 2, sección 2.3 (componente 3)

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Logistica\Logic.Pqf.Logistica.csproj`
