# Impacto en BD — Tramitación Pedidos Prepago con Sustancias Controladas
**Requisito:** R16A-RE-FU-013
**Base de Datos:** ProquifaDotNet
**Versión:** 1.1

---

## Resumen

Tramitación de pedidos Prepago con sustancias controladas para **Región México**. El manejo de sustancias controladas en Región Perú no está contemplado en el alcance de esta release (confirmado por el cliente — Duda 061); el sistema no lo restringe por código, el control es operativo.

No renderiza FAA ni Entrega con Remisión. Datos de facturación en solo lectura.

**SIN CAMBIOS ESTRUCTURALES en BD — flujo preexistente.**

---

## Impacto en BD: SIN CAMBIOS ESTRUCTURALES

> Todas las tablas necesarias para este flujo ya existen en la BD.
> La proforma (tpProformaPedido) ya tiene campo Controlados (bit).
> El pendiente de Validar Cobro es la proforma con MontoPendiente > 0.

---

## Modelo de Datos

    tpPedido (Tramitar Pedido)
        IdCatCondicionesDePago -> catCondicionesDePago (Prepago: SinCredito=1)
        FacturaPorAdelantado = 0 (NO RENDERIZADO - controlados)
        EntregaConRemision   = 0 (NO RENDERIZADO - controlados)
        Tramitado = 1
        FolioPedidoInterno (asignado al tramitar)
        IdRegion -> Region (MEX/PER)

    tpProformaPedido (Proforma generada al tramitar)
        MontoTotal, MontoPagado=0, MontoPendiente=MontoTotal
        Controlados = 1 (sustancias controladas)
        ReferenciaPago (reconstruida segun RE-FU-006)
        Folio (foliador lineal global)
        IdArchivoPDF? -> PDF de la proforma

    tpPedidoProformaPedido (relacion N:N pedido-proforma)
        IdTPPedido FK -> tpPedido
        IdTPProformaPedido FK -> tpProformaPedido

    tpProformaPartidaPedido (partidas de la proforma)
        IdTPProformaPedido FK -> tpProformaPedido
        IdTPPartidaPedido FK -> tpPartidaPedido
        IdProducto, NumeroDePiezas, PrecioUnitario

    tpPedidoCorreoEnviado (correo de proforma enviado al cliente)
        IdTPPedido FK, IdCorreoEnviado FK

    [Validar Cobro]: tpProformaPedido.MontoPendiente > 0 = PENDIENTE

---

## Tablas Involucradas

| Tabla | Rol | Estado |
|-------|-----|--------|
| tpPedido | Cabecera pedido tramitado | Existente - sin cambios |
| tpPartidaPedido | Partidas con productos controlados | Existente - sin cambios |
| tpProformaPedido | Proforma generada (Controlados=1) | Existente - sin cambios |
| tpPedidoProformaPedido | Relacion pedido-proforma | Existente - sin cambios |
| tpProformaPartidaPedido | Partidas incluidas en la proforma | Existente - sin cambios |
| tpPedidoCorreoEnviado | Correo de proforma al cliente | Existente - sin cambios |
| catCondicionesDePago | Prepago: SinCredito=1 | Existente - sin cambios |
| DatosFacturacionCliente | Datos fiscales (solo lectura en prepago) | Existente - sin cambios |
| catControl | Deteccion controlados | Existente - sin cambios |
| fnEsProductoControlado | Deteccion controlados | Existente - requiere 'origen' |

---

## Campos Clave en tpProformaPedido

| Campo | Tipo | Uso en este Flujo |
|-------|------|-------------------|
| IdTPProformaPedido | uniqueidentifier | PK |
| IdCliente | uniqueidentifier | Cliente del pedido |
| IdEmpresa | uniqueidentifier | Empresa que factura |
| MontoTotal | decimal | Monto total de la proforma |
| MontoPagado | decimal | = 0 al generar (pendiente de cobro) |
| MontoPendiente | decimal | = MontoTotal al generar -> pendiente Validar Cobro |
| ReferenciaPago | varchar(80) | Referencia bancaria reconstruida (RE-FU-006) |
| Controlados | bit | **= 1 en este flujo** |
| Folio | varchar(80) | Folio de proforma (foliador lineal global) |
| Cancelada | bit | NULL al generar |
| FechaCompromisoPago | datetime | Fecha limite de pago |
| Activo | bit | 1 = proforma vigente |

---

## Flujo en BD al Tramitar Prepago con Controlados

    1. tpPedido.FolioPedidoInterno = siguiente folio (mecanica actual)
    2. tpPedido.Tramitado = 1, FechaTramitacion = GETDATE()
    3. INSERT tpProformaPedido (Controlados=1, MontoPendiente=MontoTotal, Folio=foliador_global)
    4. INSERT tpPedidoProformaPedido (vincular pedido con proforma)
    5. INSERT tpProformaPartidaPedido (por cada partida del pedido)
    6. Generar PDF de proforma
    7. Previsualizar PDF al ESAC -> aceptacion explicita
    8. INSERT tpPedidoCorreoEnviado (correo al cliente con PDF adjunto)
    9. Pendiente Validar Cobro = tpProformaPedido con MontoPendiente > 0
   10. Cierre pendiente Tramitar Pedido

---

## Pendiente en Validar Cobro

> No existe tabla separada 'PendienteValidarCobro'.
> El pendiente se identifica como:
>   tpProformaPedido.MontoPendiente > 0 AND Activo = 1 AND Cancelada IS NULL
> Al validarse el cobro: MontoPagado se incrementa, MontoPendiente se decrementa.
> Al completarse: FechaPagoCompleto se asigna.

---

## Restricciones Regulatorias

| Campo | Tabla | Comportamiento con Controlados |
|-------|-------|-------------------------------|
| FacturaPorAdelantado | tpPedido | NO RENDERIZADO en UI |
| EntregaConRemision | tpPedido | NO RENDERIZADO en UI |
| Controlados | tpProformaPedido | = 1 (factura anticipo posterior en Validar Cobro) |

---

## Correo de Proforma

| Campo correo | Fuente                                    | Editable      |
| ------------ | ----------------------------------------- | ------------- |
| Para         | ContactoCliente.Correo (default catalogo) | SI            |
| CC           | Usuario ESAC asignado                     | SI            |
| Asunto       | 'Proforma ' + tpPedido.FolioPedidoInterno | NO            |
| Adjunto      | PDF proforma generado                     | NO            |
| Notas extras | Campo libre capturado por ESAC            | SI (opcional) |

---

## Diferencia MEX vs PER

| Aspecto | México (MEX) | Perú (PER) |
| --- | --- | --- |
| Tramitación Prepago con controlados | ✅ En alcance | ❌ Fuera de alcance en esta release (Duda 061) |
| Generación proforma | ✅ | ❌ No aplica — controlados Perú fuera de scope |
| Envío correo | ✅ | ❌ No aplica |
| Pendiente Validar Cobro | ✅ | ❌ No aplica |
| Restricción por código | ✅ Validaciones regulatorias activas | ❌ Sin restricción por código — control operativo |
| Transferencia a Legacy (post-Validar Cobro) | ✅ | ❌ No aplica — operación termina en PQF2 |

> **Controlados Perú (Duda 061 / DUDA-027):** El cliente confirmó que el manejo de sustancias controladas no está contemplado en el alcance de esta release para Perú. El sistema no restringe el avance de un pedido con controlados de cliente Perú (Riesgo 3 del requisito); el control es operativo, no de sistema. DUDA-029 confirma además que la exclusión de Factura por Adelantado con controlados aplica igual para México y Perú (mismo criterio operativo, sin validación adicional por código).

---

## Gaps

| # | Gap | Acción | Propietario |
|---|-----|--------|-------------|
| 1 | `fnEsProductoControlado` sin soporte para tipo 'Origen' | ALTER FUNCTION — compartida con RE-FU-007/009/011. **No es propietaria de este requisito; se gestiona en RE-FU-007.** | RE-FU-007 |
| 2 | Folio proforma lineal global | Verificar mecanismo actual de foliador en BD; crear nuevo caso en `FolioBO` si no existe contador global. ~~**⚠️ Pendiente definir con el cliente:** política del folio si el ESAC cancela la previsualización (¿se conserva o se descarta?).~~ **Resuelto (DUDA-030):** el folio se conserva/reutiliza para el reintento hasta el envío exitoso; no se descarta ni se asigna uno nuevo en cada intento fallido. Impacta GAP-01 del Back. | Este requisito |
| 3 | PDF proforma vinculado a `tpProformaPedido` | No hay FK directa a la tabla de archivos — verificar campo `IdArchivoPDF` en `tpProformaPedido`. Necesario para recuperar el PDF en la previsualización (GAP-04 del Back). | Este requisito |

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-006 | ReferenciaPago en proforma |
| R16A-RE-FU-007 | Leyenda regulatoria en PDF proforma |
| R16A-RE-FU-009 | Validacion regulatoria en Pretramitar (prerequisito) |
| R16A-RE-FU-011 | Deteccion de controlados (misma cadena) |

---

**Generado por:** GitHub Copilot in SSMS
**Base de Datos:** ProquifaDotNet
