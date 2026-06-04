# Tareas Back — TPSC-RE-FU-012
**Requisito:** Tramitacion de pedidos Credito con Factura por Adelantado

---

## T1 — [ TPSC-RE-FU-012 ] [ALG-BASIC-LOGIC] Revision de codigo existente tpProformaAdelanto para aprovechabilidad

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Facturas\Anticipos
- L05.TramitarPedido\Facturas\Generadores\Anticipo

### Consideraciones previas
- El flujo anterior de Factura por Adelantado NO se reutilizara directamente
- El desarrollador debe analizar que logica/entidades son aprovechables

### Objetivo general
Revisar el codigo existente relacionado con tpProformaAdelanto y determinar que es reutilizable para el nuevo flujo FAA.

### Objetivos especificos
- Revisar `tpProformaAdelantoBO.cs` y sus extensiones
- Revisar `tpProformaAdelantoToCFDIGeneradaBO.cs` (codigo comentado)
- Revisar `CFDIGeneradaConceptoAnticipoFactory.cs`
- Revisar tablas BD: `tpProformaAdelanto`, `tpProformaAdelantoProformaPedido`, `fccPagoFacturaAdelanto`
- Documentar que es aprovechable y que debe crearse desde cero

### Resultado esperado
Documento de analisis con decision de aprovechabilidad del codigo/tablas existentes.

### Entregables
- Documento de analisis tecnico

### Criterios de aceptacion
- Se identifican claramente los componentes aprovechables vs los que requieren nuevo desarrollo
- Se valida contra el nuevo modelo de datos de BD (TPSC-RE-FU-012_BD)

---

## T2 — [ TPSC-RE-FU-012 ] [SERV-TRANSACT] Generacion de pendiente FAA en transaccion de tramitacion

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- Depende de T1 (revision de codigo existente)
- El INSERT del pendiente debe ser atomico con la tramitacion del pedido
- Solo aplica cuando `tpPedido.FacturaPorAdelantado = 1`
- Solo region Mexico

### Objetivo general
Implementar la generacion del pendiente FAA dentro de la transaccion de tramitacion del pedido cuando FAA esta activa.

### Objetivos especificos
- Modificar `tpPedidoTramitarTransaccionBO.GenerarCorreoTramitarPedido()` para detectar FAA=1
- INSERT pendiente FAA con datos: Cliente, Empresa, Monto, OrdenCompra, DatosFacturacion, Region, Moneda, Estado=Pendiente
- Fijar datos de facturacion del catalogo vigente del cliente
- Asegurar atomicidad (misma transaccion que tramitacion)
- Generar confirmacion de pedido inmediatamente (no espera factura)

### Resultado esperado
Al tramitar un pedido con FAA=1 en Mexico, se genera un registro pendiente FAA con todos los datos necesarios para que el modulo de facturacion lo consuma posteriormente.

### Entregables
- Modificacion de `tpPedidoTramitarTransaccionBO.cs`
- BO/metodo para INSERT del pendiente FAA

### Criterios de aceptacion
- El pendiente se genera atomicamente con la tramitacion
- Contiene todos los datos requeridos (cliente, empresa, monto, datos facturacion, moneda)
- No se genera pendiente si FAA=0
- No se genera pendiente si region != Mexico
- La confirmacion de pedido se genera sin esperar factura

---

## T3 — [ TPSC-RE-FU-012 ] [ALG-BASIC-LOGIC] Validaciones Back para Factura por Adelantado

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- Se implementan dentro del flujo de tramitacion
- FAA no es compatible con productos controlados (validacion de RE-FU-011)

### Objetivo general
Implementar las validaciones de negocio requeridas para la activacion de FAA.

### Objetivos especificos
- Validar que FAA solo aplica para region Mexico (rechazar si FAA=1 y region != Mexico)
- Eliminar validacion de codigo de autorizacion si existe (Regla 3: activacion directa)
- Validar que datos de facturacion del cliente esten vigentes antes de fijarlos
- Bloquear edicion de datos de facturacion cuando `tpPedido.FacturaPorAdelantado = 1`

### Resultado esperado
El sistema rechaza tramitaciones con FAA activa que no cumplan las reglas de negocio.

### Entregables
- Validaciones en flujo de tramitacion
- Validacion en endpoint de edicion de datos facturacion

### Criterios de aceptacion
- Error claro si FAA=1 y region != Mexico
- No se solicita codigo de autorizacion para activar FAA
- Error si datos de facturacion no estan vigentes
- Error si se intenta editar datos facturacion con FAA activa

---

## T4 — [ TPSC-RE-FU-012 ] [ALG-BASIC-LOGIC] Vinculacion del pendiente FAA con modulo de facturacion (RE-FU-018/019/020)

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Facturas

### Consideraciones previas
- La emision de factura, CFDI y timbrado se desarrollan en RE-FU-018, RE-FU-019 y RE-FU-020
- Esta tarea asegura que el pendiente generado en T2 sea consumible por esos modulos

### Objetivo general
Vincular el proceso de generacion del pendiente FAA con el flujo de facturacion desarrollado en los requisitos RE-FU-018/019/020.

### Objetivos especificos
- Verificar que la estructura del pendiente FAA es compatible con lo esperado por RE-FU-018
- Ajustar campos/relaciones si es necesario para la integracion
- Validar que el estado del pendiente se actualice correctamente cuando facturacion lo consume
- Documentar contrato de datos entre pendiente y modulo de facturacion

### Resultado esperado
El pendiente FAA generado en la tramitacion es consumido correctamente por el modulo de facturacion sin inconsistencias.

### Entregables
- Ajustes de integracion (si aplican)
- Documentacion del contrato de datos entre modulos

### Criterios de aceptacion
- El modulo de facturacion (RE-FU-018) puede leer y procesar el pendiente FAA
- El estado del pendiente se actualiza al ser procesado
- No hay datos faltantes ni inconsistencias entre ambos modulos

---

## Resumen de tareas

| # | Clave Catalogo | Titulo | Predecesora |
|---|----------------|--------|-------------|
| T1 | ALG-BASIC-LOGIC | Revision de codigo existente tpProformaAdelanto para aprovechabilidad | — |
| T2 | SERV-TRANSACT | Generacion de pendiente FAA en transaccion de tramitacion | T1 |
| T3 | ALG-BASIC-LOGIC | Validaciones Back para Factura por Adelantado | T1 |
| T4 | ALG-BASIC-LOGIC | Vinculacion del pendiente FAA con modulo de facturacion (RE-FU-018/019/020) | T2 |

---

## Dependencias con otros requisitos (NO incluidas como tareas)

| Requisito | Tarea relacionada | Relacion |
|-----------|-------------------|----------|
| TPSC-RE-FU-010 | T3 (Cancelacion) | Endpoint de cancelacion se desarrolla en RE-FU-010 |
| TPSC-RE-FU-011 | Validacion controlados | fnEsProductoControlado se desarrolla en RE-FU-011 |
| TPSC-RE-FU-018 | Generacion de factura | Consume el pendiente FAA generado en T2 |
| TPSC-RE-FU-019 | Generacion de CFDI | Proceso posterior a factura |
| TPSC-RE-FU-020 | Timbrado fiscal (PAC) | Proceso posterior a CFDI |
