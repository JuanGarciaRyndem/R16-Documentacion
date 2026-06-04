# [ TPSC-RE-FU-008 ] [ALG-BASIC-LOGIC] Crear FolioPagoClienteBO.Cierre — cierre/eliminación automática del pendiente

---

## Aplicativos

- ProquifaDotNet

## Módulos

- Cobros
- Buzones

## Consideraciones previas

- Tarea T07 completada (FolioPagoClienteBO existe).
- El cierre se dispara desde el módulo Validar Cobro cuando:
  - **Vinculación exitosa** a proforma/factura → cierre del pendiente (Regla 6).
  - **Marcado como inconsistencia** → eliminación del pendiente (Regla 7).
- En ambos casos el resultado es `fccFolioPagoCliente.Activo = 0`.

## Objetivo general

Implementar la lógica de cierre y eliminación automática del pendiente del Buzón de Cobros cuando el cobro se vincula a una proforma/factura o se marca como inconsistencia en Validar Cobro.

## Objetivos específicos

1. Crear `Logic.Pqf.Catalogos\Cobros\FolioPagoCliente\FolioPagoClienteBO.Cierre.cs`.
2. Implementar método `CerrarPendiente(IdFCCFolioPagoCliente)`:
   - UPDATE `fccFolioPagoCliente SET Activo = 0, FechaUltimaActualizacion = GETDATE()`.
3. El método es invocado desde la lógica de Validar Cobro (vinculación o inconsistencia).

## Resultado esperado

- El pendiente del Buzón de Cobros desaparece de la bandeja del Gestor cuando el cobro se vincula o se marca como inconsistencia.

## Entregables

- `FolioPagoClienteBO.Cierre.cs`

## Criterios de aceptación

- [ ] Al invocar el cierre, `fccFolioPagoCliente.Activo` queda en `0`.
- [ ] El correo deja de aparecer como pendiente activo en la consulta del Buzón (T05).
- [ ] No se elimina físicamente el registro (soft delete).
- [ ] El proyecto compila sin errores.

## Más información de la tarea

- Referencia: `TPSC-RE-FU-008-P2-Back.md` — PARTE 1, sección 1.2
- Query: `TPSC-RE-FU-008_BD.md` — sección "Cierre automático del pendiente al vincular cobro"
- Reglas de negocio: Regla 6 y Regla 7

## Recursos

- Repositorio: ProquifaDotNet, branch `develop-pack04`
- Proyecto: `Logic.Pqf.Catalogos\Logic.Pqf.Catalogos.csproj`
