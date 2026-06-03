# [ TPSC-RE-FU-008 ] [ARQ-PROJ-NET] Crear solución MailbotWorker.sln con arquitectura de 6 proyectos

---

## Aplicativos

- MailbotWorker (Solución nueva — .NET 10)

## Módulos

- Mailbot

## Consideraciones previas

- Esta es una solución completamente nueva que reemplaza al Mailbot actual (Framework 4.8 + Tarea de Windows).
- Se crea en un repositorio separado de ProquifaDotNet-R14.
- Utilizar la plantilla de Worker Service de .NET 10.
- Arquitectura limpia con separación de capas: Domain, Application, Infrastructure, Worker, Api, Tests.

## Objetivo general

Crear la solución `MailbotWorker.sln` con la estructura de 6 proyectos que soportará el nuevo Mailbot con Gmail Push Notifications y clasificación IA.

## Objetivos específicos

1. Crear solución `MailbotWorker.sln`.
2. Crear proyecto `Mailbot.Api` — ASP.NET Core Minimal API (.NET 10).
3. Crear proyecto `Mailbot.Worker` — Worker Service (.NET 10).
4. Crear proyecto `Mailbot.Application` — Class Library (.NET 10).
5. Crear proyecto `Mailbot.Domain` — Class Library (.NET 10).
6. Crear proyecto `Mailbot.Infrastructure` — Class Library (.NET 10).
7. Crear proyecto `Mailbot.Tests` — xUnit Test Project (.NET 10).
8. Configurar referencias entre proyectos:
   - Api → Application, Infrastructure
   - Worker → Application, Infrastructure
   - Application → Domain
   - Infrastructure → Domain
   - Tests → Application, Infrastructure, Domain
9. Agregar archivo `appsettings.json` base con estructura de configuración documentada.
10. Configurar `Program.cs` en `Mailbot.Worker` con Host builder y DI base.

## Resultado esperado

- Solución compilable con 6 proyectos y referencias correctas.
- `dotnet build` exitoso sin errores.

## Entregables

- Solución `MailbotWorker.sln` con los 6 proyectos configurados.
- `Program.cs` base en Worker y Api.
- `appsettings.json` con estructura (sin secretos).

## Criterios de aceptación

- [ ] `dotnet build MailbotWorker.sln` compila exitosamente.
- [ ] Las referencias entre proyectos son correctas según la arquitectura limpia.
- [ ] Los proyectos apuntan a .NET 10.
- [ ] El Worker Service arranca sin errores (aunque aún no procesa nada).
- [ ] La Minimal API arranca y responde en `/health`.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 2, sección 2.1
- Referencia: `TPSC-RE-FU-008-v2_Propuesta2.md` — sección "Arquitectura de la Solución .NET 10"

## Recursos

- Repositorio: Nuevo (definir nombre con equipo)
- Target Framework: .NET 10
