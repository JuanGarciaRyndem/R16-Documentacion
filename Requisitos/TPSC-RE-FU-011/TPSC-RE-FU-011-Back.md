# Impacto en Back — TPSC-RE-FU-011
**Requisito:** Tramitacion de pedidos Credito con sustancias controladas
**Aplicativo:** ProquifaDotNet
**Modulo:** L05.TramitarPedido
**Impacto:** Flujo tramitacion Credito preexistente + restricciones regulatorias por controlados + condicion Peru sin transferencia Legacy

---

## Resumen

Este requisito reutiliza el flujo de tramitacion Credito existente (TPSC-RE-FU-010) pero agrega:
1. **Restricciones regulatorias:** ocultar Factura por Adelantado y Entrega con Remision cuando el pedido tiene sustancias controladas
2. **Deteccion de controlados:** cadena tpPartidaPedido -> MarcaFamilia -> Familia -> catControl (Clave IN mundiales, nacionales, origen)
3. **Peru sin transferencia Legacy:** la region PER no ejecuta transferencia al sistema legado
4. **Proforma con Controlados=1:** la confirmacion de pedido marca el campo Controlados

---

## Seccion A — Flujo de Tramitacion Credito con Controlados

### Clase principal (preexistente)
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

### Metodo de entrada
`GenerarCorreoTramitarPedido(GMtpPedidoTramitarCorreo, out GMtpPedidoTramitarCorreoLiberado)`

### Comportamiento identico al flujo Credito base (RE-FU-010)
El flujo de tramitacion es el mismo documentado en TPSC-RE-FU-010-Back.md. Las diferencias son:

| Aspecto | Sin controlados (RE-FU-010) | Con controlados (RE-FU-011) |
|---------|---------------------------|---------------------------|
| FacturaPorAdelantado | Disponible | OCULTO (siempre 0) |
| EntregaConRemision | Disponible | OCULTO (siempre 0) |
| tpProformaPedido.Controlados | 0 | 1 |
| Transferencia Legacy (MEX) | SI | SI |
| Transferencia Legacy (PER) | NO | NO |
| Correo Regulatory Affairs | NO | SI (ya implementado) |

---

## Seccion B — Deteccion de Sustancias Controladas

### Cadena de deteccion en codigo

```
tpPartidaPedido.IdProducto
    -> MarcaFamilia (IdProducto)
        -> Familia (IdFamilia)
            -> catControl (IdCatControl)
                -> catControl.Clave IN ('mundiales', 'nacionales', 'origen')
```

### Codigo existente en tpPedidoTramitarTransaccionBO (lineas ~555-590)

El flujo ya detecta partidas controladas y envia correo a Regulatory Affairs:

```csharp
var vProductBO = new vProductoBO();
outGMtpPedidoTramitarCorreoLiberado.listatpPartidasPedido.ForEach(partida =>
{
    var vProducto = vProductBO.Obtener(partida.tpPartidaPedido.IdProducto, regionId);
    if (vProducto != null && vProducto.Controlado)
    {
        partidasControladas.Add(partida.tpPartidaPedido);
    }
});
```

### Funcion BD existente
`fnEsProductoControlado` — **GAP:** actualmente no incluye la clave 'origen'. Requiere ALTER FUNCTION (compartido con RE-FU-007 y RE-FU-009).

---

## Seccion C — Restricciones Regulatorias (Bloqueo FAA y Remision)

### Situacion actual en codigo

El campo `FacturaPorAdelantado` y `EntregaConRemision` se leen desde `tpPedido` y se procesan en el flujo de tramitacion. El bloqueo es responsabilidad del **Frontend** (ocultar opciones cuando hay controlados).

### Validacion en Back requerida (GAP)

**Se necesita una validacion de seguridad en Back** que rechace la tramitacion si:
- El pedido tiene sustancias controladas Y
- `tpPedido.FacturaPorAdelantado = 1` O `tpPedido.EntregaConRemision = 1`

Esto previene que una llamada directa al API bypasee la restriccion del Frontend.

### Propuesta de implementacion

Agregar validacion en `tpPedidoTramitarTransaccionBO.GenerarCorreoTramitarPedido()`, despues de las validaciones existentes:

```csharp
// Validar restriccion regulatoria: controlados + FAA/Remision
if (tieneControlados && (tpPedido.FacturaPorAdelantado || tpPedido.EntregaConRemision))
{
    Model.AddMessage("RestriccionRegulatoria", 
        "No se permite Factura por Adelantado ni Entrega con Remision en pedidos con sustancias controladas");
    return new Response(false, new FMessage(EMessageType.Validation.ToString(), Model.Messages.ModelState));
}
```

---

## Seccion D — Operacion Peru sin Transferencia a Legacy

### Situacion actual en codigo

La transferencia a Legacy ya esta condicionada por region:

```csharp
if (!SinCredito && region.Clave == Constants.Regiones.Mexico)
{
    var Response = generadorFoliosPedido?.IncrementarConsecutivoLegacy();
    // ...
}
```

**Ya implementado.** Peru no ejecuta `IncrementarConsecutivoLegacy()` ni la transferencia posterior. El SP `spActualizarBuzonPedidoLegacy` solo se invoca para Mexico.

### Conclusion: NO requiere desarrollo para Peru

---

## Seccion E — Proforma con Campo Controlados

### Tabla: tpProformaPedido
| Campo | Valor para este flujo |
|-------|----------------------|
| Controlados | 1 (pedido con sustancias controladas) |

### Situacion en codigo
Verificar que al generar la proforma se asigne `Controlados = 1` cuando el pedido tiene partidas controladas. Si no existe esta logica, se requiere implementarla.

---

## Seccion F — Cancelacion (Criterio C1)

Misma funcionalidad documentada en TPSC-RE-FU-010 Seccion B. El endpoint de cancelacion aplica identicamente para pedidos con controlados.

---

## Modelo de Datos Involucrado

### Escritura (mismas entidades que RE-FU-010)

| Entidad | Tabla BD | Accion |
|---------|----------|--------|
| `tpPedido` | tpPedido | UPDATE (Tramitado=1, FacturaPorAdelantado=0, EntregaConRemision=0) |
| `tpPartidaPedido` | tpPartidaPedido | UPDATE (stock, FEE, pendiente compra) |
| `ocPendienteCompraProducto` | ocPendienteCompraProducto | INSERT |
| `tpProformaPedido` | tpProformaPedido | INSERT (Controlados=1) |
| `tpPedidoContactoNotificadoEntrega` | tpPedidoContactoNotificadoEntrega | INSERT |
| `CorreoEnviado` | CorreoEnviado | INSERT (confirmacion + regulatory affairs) |
| `ArchivoCorreoEnviado` | ArchivoCorreoEnviado | INSERT |
| `tpPedidoCorreoEnviado` | tpPedidoCorreoEnviado | INSERT |
| `tpPendienteStock` | tpPendienteStock | INSERT (si hay stock) |
| `tpPartidaPedidoSeguimiento` | tpPartidaPedidoSeguimiento | INSERT |
| `VariableConfiguracion` | VariableConfiguracion | UPDATE (consecutivo) |
| `Archivo` | Archivo | INSERT (PDF) |

### Lectura (adicionales por deteccion de controlados)

| Entidad | Proposito |
|---------|-----------|
| `vProducto` | Detectar propiedad Controlado del producto |
| `MarcaFamilia` | Cadena producto -> familia |
| `Familia` | IdCatControl |
| `catControl` | Clave mundiales/nacionales/origen |
| `Region` | Clave MEX/PER para condicionar transferencia Legacy |

---

## Gaps de Desarrollo

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | fnEsProductoControlado sin 'origen' | ALTER FUNCTION agregar 'origen' | Bajo (compartido RE-FU-007/009) |
| GAP-02 | Validacion Back: rechazar FAA/Remision con controlados | Agregar validacion en tpPedidoTramitarTransaccionBO | Bajo |
| GAP-03 | Proforma Controlados=1 | Verificar/implementar asignacion en generacion de proforma | Bajo |
| GAP-04 | Endpoint de Cancelacion | Mismo desarrollo que RE-FU-010 Seccion B (Tarea 3) — compartido | Referencia |

---

## Transferencia a Legacy (solo Mexico)

| Aspecto | Detalle |
|---------|---------|
| SP | `spActualizarBuzonPedidoLegacy` / `spActualizarBuzonPedidoLegacyEncolar` |
| Condicion | Solo si `region.Clave == Constants.Regiones.Mexico` |
| Marca detencion | Si condicion Pago contra entrega |
| Peru | NO transfiere — operacion termina en PQF2 |

---

## Dependencias

| Requisito      | Relacion                                                       |
| -------------- | -------------------------------------------------------------- |
| TPSC-RE-FU-007 | fnEsProductoControlado compartida (GAP-01)                     |
| TPSC-RE-FU-009 | Validacion regulatoria en Pretramitar (prerequisito)           |
| TPSC-RE-FU-010 | Flujo base Credito sin controlados (este agrega restricciones) |

---

## Conclusion

El requisito TPSC-RE-FU-011 tiene **bajo impacto en desarrollo** ya que reutiliza el flujo Credito existente. Los unicos GAPs son:

1. **GAP-01** (compartido): ALTER FUNCTION `fnEsProductoControlado` para incluir 'origen'
2. **GAP-02** (nuevo): Validacion de seguridad en Back que rechace FAA/Remision cuando hay controlados
3. **GAP-03**: Verificar que la proforma se genere con `Controlados=1`
4. **GAP-04** (compartido con RE-FU-010): Endpoint de cancelacion

La operacion Peru sin Legacy ya esta implementada (condicion por region en el codigo existente).
