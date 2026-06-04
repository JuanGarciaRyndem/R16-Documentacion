# TPSC-RE-FU-009 â€” Tareas BackEnd

| Campo | Valor |
|-------|-------|
| **Requisito** | TPSC-RE-FU-009 |
| **Nombre** | ValidaciÃ³n Regulatoria en Pretramitar Pedido |
| **Repositorio** | ProquifaDotNet |
| **Branch** | develop-pack04 |

---

## Dependencias con otros requisitos (tareas compartidas â€” NO duplicar)

| Tarea origen | Requisito | DescripciÃ³n | Nota |
|:------------:|-----------|-------------|------|
| GAP-05 | TPSC-RE-FU-003 | INSERT `catUsoArchivoSistema` (Licencia Sanitaria + Aviso RS MEX) | **Ya existe en RE-FU-003 Tarea 5.** No crear script duplicado. Es prerequisito de este requisito. |
| Tarea 1 | TPSC-RE-FU-007 | ALTER FUNCTION `fnEsProductoControlado` â€” agregar `'origen'` | **Ya existe en RE-FU-007 Tarea 1.** No crear script duplicado. Es prerequisito de este requisito. |

> âš ï¸ **Importante:** Las tareas T1 y T2 originales de este requisito fueron removidas porque ya estÃ¡n definidas en RE-FU-003 y RE-FU-007 respectivamente. Antes de iniciar T1 (ahora T3 renumerada), verificar que ambas dependencias estÃ©n ejecutadas.

---

## Ãndice de Tareas

| # | Clave CatÃ¡logo | DescripciÃ³n corta |
|:-:|---------------|-------------------|
| T1 | `ALG-BASIC-LOGIC` | Crear ValidarDocumentosRegulatoriosBO |
| T2 | `ALG-BASIC-LOGIC` | Crear ResultadoValidacionRegulatoria (DTO) |
| T3 | `IMP-EXIST-SERVICE` | Modificar VerificarPedidoTramitableBO â€” integrar validaciÃ³n |
| T4 | `ALG-BASIC-LOGIC` | Verificar/ajustar ProductoBO.EsControlado() para origen |
| T5 | `UPDATE-TABL-CH` | Script BD: INSERT catUsoArchivoSistema (PER â€” pendiente) |

---

## T1

# [ TPSC-RE-FU-009 ] [ALG-BASIC-LOGIC] Crear ValidarDocumentosRegulatoriosBO â€” lÃ³gica de validaciÃ³n regulatoria

---

### Aplicativos

- ProquifaDotNet

### MÃ³dulos

- Pretramitar Pedido

### Consideraciones previas

- Tareas T1 y T2 deben estar ejecutadas (catÃ¡logos y funciÃ³n actualizados).
- La tabla `ArchivoCliente` debe tener registros para poder probar (depende de TPSC-RE-FU-003).
- Se puede probar con un INSERT manual de prueba en `ArchivoCliente` mientras RE-FU-003 no estÃ© listo.
- Seguir el patrÃ³n de la carpeta `L04.PretramitarPedido\Validaciones\` existente.

### Objetivo general

Crear la clase de validaciÃ³n regulatoria que verifica si el cliente de un pedido tiene los documentos regulatorios requeridos registrados en `ArchivoCliente`, segÃºn la regiÃ³n del pedido y la presencia de sustancias controladas.

### Objetivos especÃ­ficos

1. Crear `Logic.Pqf.Logistica\L04.PretramitarPedido\Validaciones\ValidarDocumentosRegulatoriosBO.cs`.
2. Implementar mÃ©todo `Validar(Guid idPPPedido)` que retorna `ResultadoValidacionRegulatoria`.
3. LÃ³gica interna:
   - Verificar si el pedido tiene al menos 1 partida activa con producto controlado (mundiales, nacionales, origen):
     ```
     ppPartidaPedido â†’ MarcaFamilia â†’ Familia â†’ catControl.Clave IN ('mundiales','nacionales','origen')
     ```
   - Si no tiene controlados â†’ retornar vÃ¡lido (sin validaciÃ³n).
   - Obtener `IdCliente` del pedido: `ppPedido.IdContactoCliente â†’ ContactoCliente.IdCliente`.
   - Obtener `Region` del pedido: `ppPedido.IdRegion â†’ Region.ClaveISO`.
   - Determinar documentos requeridos segÃºn regiÃ³n:
     - MEX: 'Licencia Sanitaria', 'Aviso de Responsable Sanitario'
     - PER: [pendiente confirmar denominaciÃ³n]
   - Consultar `ArchivoCliente` con `IdCliente` + `IdCatUsoArchivoSistema` + `Activo = 1`.
   - Si falta alguno â†’ retornar invÃ¡lido con mensaje genÃ©rico.
4. El mensaje de bloqueo debe ser genÃ©rico (Regla 4 del requisito): *"No es posible procesar el pedido porque el cliente no cuenta con la documentaciÃ³n regulatoria requerida para productos controlados. Revise la secciÃ³n de documentos regulatorios en la configuraciÃ³n del cliente."*

### Resultado esperado

- Clase funcional que detecta si un pedido con controlados puede avanzar segÃºn documentaciÃ³n regulatoria del cliente.

### Entregables

- `ValidarDocumentosRegulatoriosBO.cs`

### Criterios de aceptaciÃ³n

- [ ] Si el pedido NO tiene controlados â†’ retorna vÃ¡lido sin ejecutar verificaciÃ³n de documentos.
- [ ] Si el pedido tiene controlados y el cliente tiene TODOS los docs requeridos â†’ retorna vÃ¡lido.
- [ ] Si el pedido tiene controlados y FALTA al menos un documento â†’ retorna invÃ¡lido con mensaje genÃ©rico.
- [ ] La validaciÃ³n solo verifica presencia del registro (no vigencia, no contenido).
- [ ] Funciona para clientes de regiÃ³n MEX.
- [ ] El proyecto `Logic.Pqf.Logistica` compila sin errores.

### MÃ¡s informaciÃ³n de la tarea

- Referencia: `TPSC-RE-FU-009-Back.md` â€” PARTE 2, secciones 2.3 y 2.5
- Reglas de negocio: Regla 1 (solo si hay controlados), Regla 2 (docs segÃºn regiÃ³n), Regla 3 (solo presencia), Regla 4 (mensaje genÃ©rico)
- Queries de referencia: `TPSC-RE-FU-009_BD.md` â€” secciones "Consulta SQL"
- PatrÃ³n existente: `Logic.Pqf.Logistica\L04.PretramitarPedido\Validaciones\`

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Logistica\Logic.Pqf.Logistica.csproj`

---

## T2

# [ TPSC-RE-FU-009 ] [ALG-BASIC-LOGIC] Crear ResultadoValidacionRegulatoria â€” DTO de resultado

---

### Aplicativos

- ProquifaDotNet

### MÃ³dulos

- Pretramitar Pedido

### Consideraciones previas

- Clase sencilla de soporte para T1.
- Seguir el patrÃ³n de DTOs en `L04.PretramitarPedido\Models\` o `L04.PretramitarPedido\Validaciones\`.

### Objetivo general

Crear el DTO que encapsula el resultado de la validaciÃ³n regulatoria (vÃ¡lido/invÃ¡lido + mensaje).

### Objetivos especÃ­ficos

1. Crear `Logic.Pqf.Logistica\L04.PretramitarPedido\Validaciones\Models\ResultadoValidacionRegulatoria.cs`.
2. Propiedades:
   - `bool EsValido` â€” indica si la validaciÃ³n pasÃ³.
   - `string Mensaje` â€” mensaje genÃ©rico de bloqueo (null si es vÃ¡lido).
3. MÃ©todos estÃ¡ticos factory:
   - `Valido()` â†’ instancia con `EsValido = true`.
   - `Invalido(string mensaje)` â†’ instancia con `EsValido = false` y mensaje.

### Resultado esperado

- DTO reutilizable para comunicar el resultado de la validaciÃ³n regulatoria.

### Entregables

- `ResultadoValidacionRegulatoria.cs`

### Criterios de aceptaciÃ³n

- [ ] La clase expone `EsValido` y `Mensaje`.
- [ ] Los mÃ©todos factory crean instancias correctas.
- [ ] El proyecto compila sin errores.

### MÃ¡s informaciÃ³n de la tarea

- Referencia: `TPSC-RE-FU-009-Back.md` â€” PARTE 2, secciÃ³n 2.3

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Logistica\Logic.Pqf.Logistica.csproj`

---

## T3

# [ TPSC-RE-FU-009 ] [IMP-EXIST-SERVICE] Modificar VerificarPedidoTramitableBO â€” integrar validaciÃ³n regulatoria

---

### Aplicativos

- ProquifaDotNet

### MÃ³dulos

- Pretramitar Pedido

### Consideraciones previas

- Tareas T1 y T2 completadas (`ValidarDocumentosRegulatoriosBO` y DTO existen).
- `VerificarPedidoTramitableBO.Procesar()` es invocado desde `TramitarPedidoBO.Process()` â€” es el punto centralizado de validaciÃ³n pre-tramitaciÃ³n.
- El mÃ©todo actual lanza `ArgumentException` cuando falla una validaciÃ³n. La validaciÃ³n regulatoria debe seguir el mismo patrÃ³n.
- Verificar que `POST /ValidarAjusteOC/transaccion` tambiÃ©n pasa por este punto (deberÃ­a, segÃºn anÃ¡lisis).

### Objetivo general

Integrar la llamada a `ValidarDocumentosRegulatoriosBO.Validar()` dentro del flujo existente de `VerificarPedidoTramitableBO.Procesar()` para que el pedido sea bloqueado si no cumple la validaciÃ³n regulatoria.

### Objetivos especÃ­ficos

1. Modificar `Logic.Pqf.Logistica\L04.PretramitarPedido\Tramite\VerificarPedidoTramitableBO.cs`.
2. Agregar al final del mÃ©todo `Procesar(ppPedido)` (despuÃ©s de las validaciones existentes):
   ```csharp
   // ValidaciÃ³n regulatoria R16
   var validarDocumentosRegulatorioBo = new ValidarDocumentosRegulatoriosBO();
   var resultadoRegulatorio = validarDocumentosRegulatorioBo.Validar(ppPedido.IdPPPedido);
   if (!resultadoRegulatorio.EsValido)
       throw new ArgumentException(resultadoRegulatorio.Mensaje);
   ```
3. No modificar ni eliminar ninguna validaciÃ³n existente.

### Resultado esperado

- Al intentar tramitar un pedido con controlados y sin documentos regulatorios, el sistema lanza excepciÃ³n con mensaje genÃ©rico que se traduce en `BadRequest` en el controller.

### Entregables

- `VerificarPedidoTramitableBO.cs` modificado.

### Criterios de aceptaciÃ³n

- [ ] Pedido sin controlados â†’ no se ejecuta la validaciÃ³n regulatoria â†’ avanza normalmente.
- [ ] Pedido con controlados + docs registrados â†’ avanza normalmente.
- [ ] Pedido con controlados + docs faltantes â†’ `ArgumentException` con mensaje genÃ©rico â†’ controller retorna `BadRequest`.
- [ ] Las validaciones existentes (facturaciÃ³n, configuraciÃ³n, partidas) siguen funcionando sin cambios.
- [ ] `POST /PretramitarPedido/transaccion` retorna 400 con mensaje regulatorio cuando aplica.
- [ ] `POST /ValidarAjusteOC/transaccion` retorna 400 con mensaje regulatorio cuando aplica.
- [ ] El proyecto compila sin errores.

### MÃ¡s informaciÃ³n de la tarea

- Referencia: `TPSC-RE-FU-009-Back.md` â€” PARTE 2, secciÃ³n 2.2 y PARTE 8 (diagrama de flujo)
- Archivo a modificar: `Logic.Pqf.Logistica\L04.PretramitarPedido\Tramite\VerificarPedidoTramitableBO.cs`
- Controller que consume: `WebApi.Logistica\Controllers\Procesos\L04.PretramitarPedido\PretramitarPedidoTramitarController.cs`

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Logistica\Logic.Pqf.Logistica.csproj`

---

## T4

# [ TPSC-RE-FU-009 ] [ALG-BASIC-LOGIC] Verificar/ajustar ProductoBO.EsControlado() para cubrir 'origen'

---

### Aplicativos

- ProquifaDotNet

### MÃ³dulos

- CatÃ¡logos
- Pretramitar Pedido

### Consideraciones previas

- Dependencia RE-FU-007 Tarea 1 ejecutada (la funciÃ³n BD `fnEsProductoControlado` ya incluye `origen`).
- El mÃ©todo `ProductoBO.EsControlado()` en `ProductoBO.TipoExtensions.cs` usa `ProductoMarcaFamilia.Controlado` (campo `bit`).
- Verificar si este campo se actualiza automÃ¡ticamente cuando `catControl` incluye `origen` (posible trigger o proceso batch).
- Si `ProductoMarcaFamilia.Controlado` NO se actualiza automÃ¡ticamente con `origen`, el mÃ©todo `EsControlado()` no detectarÃ¡ productos de origen y la validaciÃ³n en T3 debe usar query directa a `catControl`.

### Objetivo general

Verificar que el mÃ©todo `ProductoBO.EsControlado()` retorna `true` para productos de familias con `catControl.Clave = 'origen'` y, si no lo hace, ajustar la lÃ³gica.

### Objetivos especÃ­ficos

1. Verificar cÃ³mo se puebla `ProductoMarcaFamilia.Controlado`:
   - Â¿Trigger en BD?
   - Â¿Proceso batch?
   - Â¿Se calcula al guardar el producto?
2. Probar: un producto con `Familia.catControl.Clave = 'origen'` â†’ Â¿`ProductoMarcaFamilia.Controlado = true`?
3. Si NO cubre `origen`:
   - **OpciÃ³n A**: Actualizar el proceso que puebla `Controlado` para incluir `origen`.
   - **OpciÃ³n B**: Ajustar `ProductoBO.EsControlado()` para hacer query directa incluyendo `origen`.
   - **OpciÃ³n C**: En `ValidarDocumentosRegulatoriosBO` (T3) usar query directa a `catControl` en vez de depender de `ProductoMarcaFamilia.Controlado`.
4. Documentar la decisiÃ³n tomada.

### Resultado esperado

- Los productos de familias `origen` son correctamente detectados como controlados por el sistema.

### Entregables

- Informe de verificaciÃ³n (si no requiere cambios).
- Archivo modificado (si requiere ajuste): `ProductoBO.TipoExtensions.cs` o proceso de actualizaciÃ³n de `Controlado`.

### Criterios de aceptaciÃ³n

- [ ] Un producto con familia de tipo `origen` es identificado como controlado.
- [ ] El mÃ©todo `EsControlado()` retorna `true` para productos `mundiales`, `nacionales` y `origen`.
- [ ] No se rompen funcionalidades existentes.
- [ ] Se documenta la decisiÃ³n (opciÃ³n A, B o C).

### MÃ¡s informaciÃ³n de la tarea

- Referencia: `TPSC-RE-FU-009-Back.md` â€” PARTE 2, secciÃ³n 2.4
- Archivo: `Logic.Pqf.Catalogos\Productos\ProductoBO.TipoExtensions.cs`
- Referencia: `TramitarPedidoBO.cs` lÃ­nea ~69: `productoBo.EsControlado(ppPartidaPedido.IdProducto, cliente.IdRegion)`

### Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Catalogos\Logic.Pqf.Catalogos.csproj`

---

## T5

# [ TPSC-RE-FU-009 ] [UPDATE-TABL-CH] Script BD: INSERT catUsoArchivoSistema â€” documentos regulatorios PER

---

### Aplicativos

- ProquifaDotNet (Base de Datos â€” SQL Server RYNL010)

### MÃ³dulos

- Pretramitar Pedido
- CatÃ¡logo de Clientes

### Consideraciones previas

- **BLOQUEADA** â€” La denominaciÃ³n exacta de los documentos DIGEMID para PerÃº NO estÃ¡ confirmada por el cliente.
- No ejecutar hasta que se resuelva el Gap #1 documentado en `TPSC-RE-FU-009-Back.md`.
- La tarea T1 (`ValidarDocumentosRegulatoriosBO`) debe estar preparada para soportar documentos PER una vez que se inserten aquÃ­.

### Objetivo general

Insertar en `catUsoArchivoSistema` los tipos de documento regulatorio requeridos para RegiÃ³n PerÃº (normativa DIGEMID), completando la cobertura regulatoria para ambas regiones.

### Objetivos especÃ­ficos

1. Confirmar con el cliente la denominaciÃ³n exacta de los 2 documentos DIGEMID.
2. INSERT registro [Documento DIGEMID 1] con `Activo = 1`.
3. INSERT registro [Documento DIGEMID 2] con `Activo = 1`.
4. Script idempotente.

### Resultado esperado

- `catUsoArchivoSistema` contiene los registros PER activos y la validaciÃ³n regulatoria funciona para clientes de PerÃº.

### Entregables

- Script SQL de migraciÃ³n (idempotente).
- Script SQL de rollback.

### Criterios de aceptaciÃ³n

- [ ] La denominaciÃ³n fue confirmada por el cliente.
- [ ] El script se ejecuta sin errores en RYNL010.
- [ ] Los registros existen con `Activo = 1` tras la ejecuciÃ³n.
- [ ] La validaciÃ³n regulatoria funciona correctamente para clientes PER.
- [ ] El script es idempotente.

### MÃ¡s informaciÃ³n de la tarea

- Referencia: `TPSC-RE-FU-009_BD.md` â€” secciÃ³n "catUsoArchivoSistema"
- Gap: `TPSC-RE-FU-009-Back.md` â€” PARTE 7, Gap #1
- Pendiente: denominaciÃ³n exacta pendiente de cliente (transversal con RE-FU-003 y RE-FU-007)

### Recursos

- Servidor: RYNL010
- Base de datos: ProquifaDotNet

