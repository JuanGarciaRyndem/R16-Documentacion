# [ TPSC-RE-FU-008 ] [ARQ-PROJ-NET] Implementar Mailbot.Domain — entidades e interfaces

---

## Aplicativos

- MailbotWorker

## Módulos

- Mailbot

## Consideraciones previas

- Tarea T13 completada (solución creada).
- El dominio define los contratos (interfaces) que las capas superiores implementan.
- No debe tener dependencias externas (solo .NET base).

## Objetivo general

Implementar las entidades del dominio y las interfaces que definen los contratos de clasificación IA, persistencia de correos y canal de eventos interno.

## Objetivos específicos

1. Crear entidades en `Mailbot.Domain\Entities\`:
   - `CorreoRecibido.cs` — representa un correo recibido.
   - `ClasificacionCorreo.cs` — resultado de clasificación IA.
   - `EventoCorreo.cs` — evento de notificación Pub/Sub.
2. Crear interfaces en `Mailbot.Domain\Interfaces\`:
   - `IClasificadorAgente.cs` — contrato del Agente IA (clasificar + extraer datos).
   - `ICorreoRepository.cs` — contrato de persistencia de correos.
   - `IEventoChannel.cs` — contrato del Channel interno (publicar/consumir eventos).

## Resultado esperado

- Capa de dominio pura sin dependencias externas, con contratos claros para la infraestructura.

## Entregables

- 3 entidades + 3 interfaces.

## Criterios de aceptación

- [ ] `Mailbot.Domain` no tiene dependencias NuGet externas (solo `netstandard` o `net10.0`).
- [ ] Las interfaces definen métodos claros con tipos del dominio.
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-v2_Propuesta2.md` — sección "Arquitectura de la Solución" (Mailbot.Domain)

## Recursos

- Repositorio: MailbotWorker
