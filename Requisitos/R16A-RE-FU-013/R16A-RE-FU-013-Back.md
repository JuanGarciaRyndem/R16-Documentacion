# Impacto en Back - R16A-RE-FU-013
**Requisito:** Tramitacion de pedidos Prepago con sustancias controladas
**Aplicativo:** ProquifaDotNet
**Modulo:** L05.TramitarPedido
**Impacto:** Flujo preexistente - ajustes en foliador global de proforma + PDF via DocumentBuilder + validaciones regulatorias

---

## Resumen

Este requisito corresponde a la tramitacion de pedidos Prepago con sustancias controladas (Mundial, Nacional, Origen) para Mexico y Peru. El flujo base **ya existe**:

- `tpPedidoFacturaToTPProformaPedidoBO.cs` ya detecta controlados y genera proformas con `Controlados=true`
- `tpProformaPedidoFactory.cs` crea la entidad `tpProformaPedido` (actualmente recibe `Empresa` como parametro)
- `ApiCallerDocumentBuilder.cs` es el cliente HTTP para generar PDFs en el servicio externo DocumentBuilder

Los ajustes requeridos:

1. **Foliador lineal global** para proformas (un solo contador, sin segmentacion empresa/region)
2. **Generacion de PDF** de la proforma via DocumentBuilder (se desarrolla en RE-FU-016/017, se vincula con este requisito)
3. **Previsualizacion obligatoria** del PDF antes de envio (con opcion de cancelar y volver sin enviar)
4. **Validaciones regulatorias** (no FAA, no Remision con controlados)
5. **Verificar comportamiento por empresa** - la proforma se genera por empresa; confirmar si aplica para todas las empresas del sistema

---

## Codigo Existente Relacionado

### Tramitacion principal
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\tpPedidoTramitarTransaccionBO.cs`

### Generacion de proformas
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\GeneracionProforma\tpPedidoFacturaToTPProformaPedidoBO.cs`

> **Logica existente:**
> - Detecta controlados: `tpPedidoBo.TieneControlados(tpPedido.IdTPPedido, cliente.IdRegion)`
> - Si hay controlados -> genera proforma separada con `Controlados = true`
> - Cada partida controlada se asigna a una proforma especial
> - Factory recibe `Empresa` -> **la proforma se crea por empresa**

### Factory de proforma
`Logic.Pqf.Logistica\L05.TramitarPedido\Facturas\Fabrica\tpProformaPedidoFactory.cs`

> **Estado actual:**
> - Recibe `Empresa` en constructor
> - Asigna `Folio = null` al crear la proforma (el folio se asigna despues o no se asigna)
> - Asigna `IdEmpresa = empresa.IdEmpresa`
> - `MontoPendiente = 0, MontoPagado = 0, MontoTotal = 0` (se calculan despues)

### Generador de folios de pedido (referencia)
`Logic.Pqf.Logistica\L05.TramitarPedido\Liberar\GeneradorFoliosPedido.cs`

> - Folio de pedido usa `RegionConsecutivoFoliosPedido` (por region)
> - Para Prepago Mexico (`SinCredito && region == Mexico`) retorna sin generar folio regional
> - Patron de referencia para crear foliador de proforma

### Foliador generico
`Logic.Pqf.Logistica\Utils\Folios\FolioBO.cs`

> - Usa lock + switch por tipo de consecutivo
> - Patron reutilizable para crear caso "TPProformaPedido" con consecutivo global

### PDF via DocumentBuilder
`Logic.Pqf.Catalogos\ApiCaller\ApiCallerDocumentBuilder.cs`

> - Cliente HTTP que envia POST con JSON y recibe PDF (byte[])
> - Metodo `EnvioPost(string url, object parametros)` -> retorna `byte[]`
> - URL base y endpoints configurados en AppSettings (`DocumentBuilder:*`)
> - **Verificar si existe endpoint para PDF de proforma en DocumentBuilder o hay que crearlo**

---

## Analisis: Proforma por Empresa

La proforma se genera **por empresa** (`tpProformaPedidoFactory` recibe `Empresa`). Esto significa:

- Si un pedido tiene partidas de multiples empresas, se generan **multiples proformas**
- El foliador lineal global debe funcionar independientemente de la empresa
- El PDF debe generarse para cada proforma individual

---

## Gaps de Desarrollo

| # | Gap | Accion | Esfuerzo |
|---|-----|--------|----------|
| GAP-01 | Foliador lineal global de proforma | Crear algoritmo de foliador (nuevo caso en `FolioBO` o nueva clase). Un solo contador global sin segmentacion por empresa ni region. Asignar al campo `tpProformaPedido.Folio`. **Nota:** Pendiente definir politica de folio al cancelar previsualizacion (se conserva para reintento o se descarta) | Medio |
| GAP-02 | Generacion PDF proforma via DocumentBuilder | **Se desarrolla en R16A-RE-FU-016 y R16A-RE-FU-017.** Se genera una tarea de vinculacion para integrar el PDF generado con el flujo de tramitacion | Referencia |
| GAP-03 | Verificar aplicabilidad por empresa | Confirmar que el flujo Prepago con controlados aplica para todas las empresas del sistema. Documentar cuales empresas generan proformas y si hay diferencias en el template PDF | Medio |
| GAP-12 | Vinculacion con generacion PDF (RE-FU-016/017) | Tarea para integrar el PDF de proforma generado por DocumentBuilder (RE-FU-016/017) con el flujo de previsualizacion y envio de este requisito | Bajo |
| GAP-04 | Previsualizacion obligatoria del PDF | Endpoint que retorna el PDF generado para previsualizacion antes del envio. El ESAC acepta y luego se ejecuta envio en segundo llamado. **Debe soportar cancelacion:** si el ESAC cancela, se vuelve al pedido sin enviar la proforma (Criterio C2) | Medio |
| GAP-05 | Pantalla datos envio correo | Verificar/ajustar endpoint de envio: Para (contacto cliente editable), CC (ESAC default editable), Asunto = "Proforma " + FolioPedidoInterno (no editable), Adjuntos (PDF no editable), Notas extras (opcional) | Medio |
| GAP-06 | Generacion pendiente Validar Cobro | Verificar que `tpProformaPedido.MontoPendiente > 0` marca automaticamente como pendiente. Si requiere accion adicional al confirmar envio, implementar | Bajo |
| GAP-07 | Validacion Back: rechazar FAA con controlados | Si `TieneControlados() && FacturaPorAdelantado=1` -> rechazar tramitacion con error | Bajo |
| GAP-08 | Validacion Back: rechazar Remision con controlados | Si `TieneControlados() && EntregaConRemision=1` -> rechazar tramitacion con error | Bajo |
| GAP-09 | Validacion Back: datos facturacion solo lectura en Prepago | Rechazar edicion de datos facturacion cuando SinCredito=1 | Bajo |
| GAP-10 | Cierre de pendiente Tramitar Pedido | Al completar tramitacion + envio correo + pendientes generados, cerrar pendiente de bandeja | Bajo |
| GAP-11 | Verificacion operacion Peru | Confirmar que el flujo opera para region Peru (sin transferencia Legacy). Verificar que `TieneControlados()` funciona con IdRegion de Peru | Bajo |

---

## Flujo Back Completo

```
1. ESAC ejecuta Tramitar
2. Back valida:
   - Condicion de pago = Prepago (SinCredito=1)
   - TieneControlados() = true
   - FacturaPorAdelantado debe ser 0 (rechazar si 1)
   - EntregaConRemision debe ser 0 (rechazar si 1)
3. Asigna FolioPedidoInterno (mecanica actual)
4. Genera proforma(s) por empresa:
   - tpProformaPedido con Controlados=1
   - Folio = foliador lineal global (GAP-01)
   - MontoPendiente = MontoTotal (calculado de partidas)
   - INSERT tpPedidoProformaPedido + tpProformaPartidaPedido
5. Genera PDF de la proforma via DocumentBuilder (GAP-02)
6. Retorna PDF para previsualizacion (GAP-04)
7. ESAC acepta -> Front llama endpoint de envio
8. Endpoint envio (GAP-05):
   - Recibe: Para, CC, NotasExtras
   - Asunto = "Proforma " + FolioPedidoInterno
   - Adjunto = PDF generado
   - Envia correo
9. Al confirmar envio exitoso:
   - Pendiente Validar Cobro activo (MontoPendiente > 0)
   - Cierra pendiente de Tramitar Pedido
10. tpPedido.Tramitado = 1
```

---

## Transferencia a Legacy

- **Mexico:** Transferencia posterior al flujo de Validar Cobro (fuera de scope)
- **Peru:** Sin transferencia a Legacy
- En Tramitar Pedido NO hay transferencia a Legacy para Prepago

---

## Dependencias

| Requisito | Relacion |
|-----------|----------|
| R16A-RE-FU-006 | ReferenciaPago de la proforma |
| R16A-RE-FU-007 | fnEsProductoControlado (deteccion controlados con tipo Origen) |
| R16A-RE-FU-010 | Cancelacion de pedido (endpoint compartido) |
| R16A-RE-FU-016 | Generacion de PDF de proforma en DocumentBuilder |
| R16A-RE-FU-017 | Template/endpoint de proforma en DocumentBuilder |

---

## Conclusion

El requisito R16A-RE-FU-013 tiene **impacto medio** en desarrollo Back. El flujo base de generacion de proformas con controlados **ya existe**. Los gaps principales son:

1. **Foliador global** (GAP-01) - algoritmo nuevo, un contador sin segmentacion
2. **PDF via DocumentBuilder** (GAP-02) - se desarrolla en **RE-FU-016/017**, se vincula con tarea GAP-12
3. **Aplicabilidad por empresa** (GAP-03) - la proforma se genera por empresa, verificar si todas aplican y si hay templates diferentes

El desarrollador debe:
- Revisar `tpProformaPedidoFactory.cs` donde actualmente `Folio = null`
- Revisar `ApiCallerDocumentBuilder.cs` para el patron de consumo del servicio
- Confirmar con equipo de DocumentBuilder si existe template de proforma o hay que desarrollarlo
