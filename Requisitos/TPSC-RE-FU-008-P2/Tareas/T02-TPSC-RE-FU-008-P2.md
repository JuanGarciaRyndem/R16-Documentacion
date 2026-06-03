# [ TPSC-RE-FU-008 ] [CREATE-TABL-CH] Script BD: CREATE TABLE MailbotEventoCorreo + índices

---

## Aplicativos

- ProquifaDotNet (Base de Datos — SQL Server RYNL010)

## Módulos

- Mailbot

## Consideraciones previas

- La tarea T01 debe estar ejecutada previamente (la tabla referencia `Region` vía FK).
- La tabla es requerida tanto por el Mailbot Worker nuevo (.NET 10) como por la lógica de trazabilidad en ProquifaDotNet.

## Objetivo general

Crear la tabla `MailbotEventoCorreo` para registrar cada notificación Pub/Sub recibida, permitir trazabilidad, reintentos y detección de duplicados (idempotencia).

## Objetivos específicos

1. Crear tabla `MailbotEventoCorreo` con las columnas documentadas en la Propuesta 2:
   - `IdMailbotEventoCorreo` (PK, uniqueidentifier, DEFAULT NEWID())
   - `IdRegion` (FK → Region)
   - `IdentificadorCorreoGmail` (varchar 200, NOT NULL)
   - `PubSubMessageId` (varchar 200, NULL)
   - `HistoryId` (varchar 50, NULL)
   - `CorreoEmisor` (varchar 180, NULL)
   - `Asunto` (varchar 350, NULL)
   - `FechaEvento` (datetime, NOT NULL)
   - `Procesado` (bit, DEFAULT 0)
   - `FechaProcesado` (datetime, NULL)
   - `Intentos` (int, DEFAULT 0)
   - `Error` (varchar MAX, NULL)
   - `FechaRegistro` (datetime, DEFAULT GETDATE())
   - `FechaUltimaActualizacion` (datetime, DEFAULT GETDATE())
   - `Activo` (bit, DEFAULT 1)
2. Crear índice único filtrado `UIX_MailbotEventoCorreo_Gmail` sobre (`IdentificadorCorreoGmail`, `IdRegion`) WHERE `Activo = 1`.
3. Crear índice `IX_MailbotEventoCorreo_Pendientes` sobre (`Procesado`, `Intentos`) WHERE `Procesado = 0`.

## Resultado esperado

- Tabla `MailbotEventoCorreo` creada con todas las constraints y defaults.
- Índices creados para garantizar idempotencia y consulta eficiente de eventos pendientes.

## Entregables

- Script SQL de creación (idempotente).
- Script SQL de rollback (DROP TABLE).

## Criterios de aceptación

- [ ] El script se ejecuta sin errores en RYNL010.
- [ ] El índice único impide insertar dos eventos con el mismo `IdentificadorCorreoGmail` + `IdRegion` cuando `Activo = 1`.
- [ ] La FK a `Region` es válida y referencia `dbo.Region(IdRegion)`.
- [ ] El script de rollback elimina la tabla correctamente.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-v2_Propuesta2.md` — sección "Tablas BD Nuevas Requeridas > MailbotEventoCorreo"
- Referencia Impacto: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.1

## Recursos

- Servidor: RYNL010
- Base de datos: ProquifaDotNet
