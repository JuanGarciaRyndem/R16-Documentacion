# [ TPSC-RE-FU-008 ] [QUERY-CH] Scaffold EF Core de ProquifaDotNet en Mailbot.Infrastructure

---

## Aplicativos

- MailbotWorker

## Módulos

- Mailbot

## Consideraciones previas

- Tareas T01, T02 y T03 ejecutadas (tablas existen en BD).
- El scaffold se ejecuta contra la BD ProquifaDotNet en RYNL010.
- Solo las tablas requeridas por el Mailbot (no todo el esquema).

## Objetivo general

Generar el scaffold de EF Core sobre las tablas de ProquifaDotNet que utiliza el Mailbot Worker para persistir correos, clasificaciones y pendientes.

## Objetivos específicos

1. Ejecutar scaffold con las siguientes tablas:
   - `RegionConfiguracionMailBot`, `CorreoRecibido`, `CorreoRecibidoContenido`, `CorreoRecibidoCliente`, `CorreoRecibidoEstatus`, `ArchivoCorreoRecibido`, `catClasificacionCorreoRecibido`, `catProceso`, `fccFolioPagoCliente`, `Region`, `MailbotEventoCorreo`, `MailbotClasificacionLog`
2. Output en `Mailbot.Infrastructure\Persistence\Entities\`.
3. Contexto: `ProquifaDbContext` en `Mailbot.Infrastructure\Persistence\`.
4. Usar `--no-onconfiguring` para gestionar ConnectionString vía DI.

## Resultado esperado

- Entidades EF Core generadas y contexto funcional que permite CRUD sobre las tablas del Mailbot.

## Entregables

- `ProquifaDbContext.cs` + entidades generadas en `Persistence\Entities\`.

## Criterios de aceptación

- [ ] El scaffold se ejecuta sin errores.
- [ ] El contexto se puede instanciar con DI y conectar a la BD.
- [ ] Las relaciones FK están correctamente mapeadas.
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 2, sección 2.4
- Comando: ver `TPSC-RE-FU-008-v2_Propuesta2.md` — sección "Scaffold EF Core"

## Recursos

- Repositorio: MailbotWorker
- Servidor: RYNL010
- NuGet: Microsoft.EntityFrameworkCore.SqlServer, Microsoft.EntityFrameworkCore.Design
