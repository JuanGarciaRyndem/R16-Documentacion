# [ TPSC-RE-FU-008 ] [ALG-COMPLX-LOGIC] Implementar Mailbot.Application — casos de uso y DTOs

---

## Aplicativos

- MailbotWorker

## Módulos

- Mailbot

## Consideraciones previas

- Tareas T14, T15, T16 completadas (dominio, infraestructura y scaffold listos).
- Los casos de uso orquestan la lógica sin depender de infraestructura directamente (usan interfaces del dominio).
- El `ProcesarCorreoUseCase` es el componente más complejo: lee correo, clasifica con IA, persiste y genera pendiente si es cobro.

## Objetivo general

Implementar los casos de uso de la capa de aplicación que orquestan el procesamiento de correos, generación de pendientes y renovación del watch de Gmail.

## Objetivos específicos

1. `UseCases\ProcesarCorreoUseCase.cs`:
   - Recibe `EventoCorreoDto` (historyId, region).
   - Llama `ICorreoRepository` para obtener correo de Gmail vía `GmailService`.
   - Llama `IClasificadorAgente.Clasificar()` → resultado con clasificación y confianza.
   - Persiste `CorreoRecibido` + `CorreoRecibidoCliente` + `ArchivoCorreoRecibido`.
   - Persiste `MailbotClasificacionLog`.
   - Si clasificación == 'cobro' → invoca `GenerarPendienteUseCase`.
   - Si adjuntos → invoca `IArchivoStorage.GuardarArchivo()`.
   - Actualiza `MailbotEventoCorreo` como procesado.
2. `UseCases\GenerarPendienteUseCase.cs`:
   - Crea `fccFolioPagoCliente` con datos pre-extraídos por IA.
3. `UseCases\RenovarGmailWatchUseCase.cs`:
   - Llama `GmailWatchService.Renovar()` y actualiza `RegionConfiguracionMailBot.WatchExpiration`.
4. `DTOs\`:
   - `PubSubNotificacionDto.cs` — payload del push de Google.
   - `EventoCorreoDto.cs` — evento interno entre Api y Worker.
   - `ResultadoClasificacionDto.cs` — respuesta del Agente IA.

## Resultado esperado

- Los casos de uso orquestan correctamente el flujo end-to-end de procesamiento de correos.

## Entregables

- 3 casos de uso + 3 DTOs.

## Criterios de aceptación

- [ ] `ProcesarCorreoUseCase` ejecuta el flujo completo: leer → clasificar → persistir → generar pendiente (si aplica).
- [ ] `GenerarPendienteUseCase` inserta en `fccFolioPagoCliente` con datos del IA.
- [ ] `RenovarGmailWatchUseCase` renueva el watch y persiste la nueva expiración.
- [ ] Los DTOs son serializables y contienen las propiedades necesarias.
- [ ] Los reintentos con backoff exponencial están implementados en `ProcesarCorreoUseCase`.
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 2, sección 2.6 (Flujo end-to-end)
- Referencia: `TPSC-RE-FU-008-v2_Propuesta2.md` — secciones "Lógica de Reintentos" y "Arquitectura"

## Recursos

- Repositorio: MailbotWorker
