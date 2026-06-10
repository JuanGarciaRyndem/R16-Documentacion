# [ TPSC-RE-FU-008 ] [IMP-EXIST-SERVICE] Actualizar GenerarPendienteUseCase — casos Cotizacion y Pedido en MailbotWorker (.NET 10)

---

## Aplicativos

- MailbotWorker (.NET 10)
- ProquifaDotNet (referencia de APIs consumidas)

## Módulos

- Mailbot
- Mailbot.Application — Casos de uso

## Consideraciones previas

- Tarea T19 completada (`GenerarPendienteUseCase` con caso `cobro` implementado).
- Actualmente `ProcesarCorreoUseCase` solo invoca `GenerarPendienteUseCase` cuando la clasificación es `cobro` (sección 2.6 del Back.md). Esta tarea extiende el uso case para manejar también `cotizacion` y `pedido`.
- El Worker no genera el pendiente directamente en BD para cotizaciones y pedidos: llama a los endpoints de ProquifaDotNet (`/api/buzon/cotizaciones`, `/api/buzon/pedidos`) siguiendo el mismo patrón de integración entre APIs ya establecido en el proyecto.
- Confirmar con el TechLead qué endpoint/BO de ProquifaDotNet debe invocarse para crear el pendiente de cotización (`cotCotizacionPendiente`) y de pedido (pretramitación), antes de implementar.
- Actualizar `ProcesarCorreoUseCase` para invocar `GenerarPendienteUseCase` también cuando la clasificación sea `cotizacion` o `pedido`.

## Objetivo general

Implementar los casos `cotizacion` y `pedido` en `GenerarPendienteUseCase` del Worker .NET 10, de modo que el flujo end-to-end de clasificación genere el pendiente correspondiente para los tres tipos de correo: cobro, cotización y pedido.

## Objetivos específicos

1. Agregar `case "cotizacion"` en `GenerarPendienteUseCase`:
   - Llamar al endpoint de ProquifaDotNet que crea el pendiente de cotización (confirmación pendiente con TechLead).
   - Persistir en `CorreoRecibidoCliente` el vínculo con la cotización generada.
   - Actualizar `MailbotEventoCorreo` con estado PROCESADO.
2. Agregar `case "pedido"` en `GenerarPendienteUseCase`:
   - Llamar al endpoint de ProquifaDotNet que crea la pretramitación de pedido.
   - Persistir en `CorreoRecibidoCliente` el vínculo con el pedido generado.
   - Actualizar `MailbotEventoCorreo` con estado PROCESADO.
3. Actualizar `ProcesarCorreoUseCase` para invocar `GenerarPendienteUseCase` cuando la clasificación sea `cotizacion` o `pedido` (actualmente solo `cobro`):
   ```csharp
   // Antes
   if (clasificacion == "cobro")
       await _generarPendienteUseCase.Ejecutar(correo, datos);
   
   // Después
   if (clasificacion is "cobro" or "cotizacion" or "pedido")
       await _generarPendienteUseCase.Ejecutar(correo, datos);
   ```
4. Manejar errores: si el endpoint de ProquifaDotNet no responde, revertir y actualizar `MailbotEventoCorreo` con estado ERROR e incrementar `Intentos`.
5. Actualizar los prompts de extracción si se requieren campos adicionales para cotización y pedido (`extraccion_cotizacion.txt`, `extraccion_pedido.txt`).

## Resultado esperado

El Worker procesa los tres tipos de correo de extremo a extremo:
- Correo de cobro → `fccFolioPagoCliente` generado vía ProquifaDotNet API.
- Correo de cotización → pendiente de cotización generado vía ProquifaDotNet API.
- Correo de pedido → pretramitación de pedido generada vía ProquifaDotNet API.

En todos los casos `MailbotEventoCorreo` queda con estado PROCESADO y `MailbotClasificacionLog` registra la clasificación.

## Entregables

- `Mailbot.Application\UseCases\GenerarPendienteUseCase.cs` — actualizado con cases `cotizacion` y `pedido`
- `Mailbot.Application\UseCases\ProcesarCorreoUseCase.cs` — actualizado para invocar `GenerarPendienteUseCase` para los tres tipos
- `Mailbot.Infrastructure\Prompts\extraccion_cotizacion.txt` — ajustado si se requieren campos adicionales
- `Mailbot.Infrastructure\Prompts\extraccion_pedido.txt` — ajustado si se requieren campos adicionales

## Criterios de aceptación

- [ ] `ProcesarCorreoUseCase` invoca `GenerarPendienteUseCase` para clasificaciones `cobro`, `cotizacion` y `pedido`
- [ ] `case "cotizacion"` llama al endpoint correcto de ProquifaDotNet y registra el pendiente
- [ ] `case "pedido"` llama al endpoint correcto de ProquifaDotNet y registra la pretramitación
- [ ] `MailbotEventoCorreo` queda en estado PROCESADO para los tres tipos tras éxito
- [ ] Si el endpoint de ProquifaDotNet falla, `MailbotEventoCorreo` queda en estado ERROR con mensaje descriptivo
- [ ] El backoff exponencial (definido en T19) aplica también a los nuevos cases
- [ ] La solución `MailbotWorker.sln` compila en .NET 10 sin errores
- [ ] PR aprobado por líder técnico

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 2, sección 2.6 (flujo end-to-end, actualmente solo documenta cobro)
- Complementa T19 (`GenerarPendienteUseCase` estructura base) y T20 (`ProcesarCorreoUseCase`)
- Los endpoints de ProquifaDotNet a invocar son los creados en T25-T32 de esta propuesta
- Dependencia: T19 completado, T25 y T29 completados (BOs de buzones disponibles para referencia de flujo)

## Recursos

- Repositorio: MailbotWorker, branch `develop-pack04`
- Proyecto: `Mailbot.Application`
- Referencia prompts: `Mailbot.Infrastructure\Prompts\extraccion_cotizacion.txt`, `extraccion_pedido.txt`
