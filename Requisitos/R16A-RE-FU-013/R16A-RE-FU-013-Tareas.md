# Tareas Back - R16A-RE-FU-013
**Requisito:** Tramitacion de pedidos Prepago con sustancias controladas

---

## T1 - [ R16A-RE-FU-013 ] [ALG-COMPLX-LOGIC] Foliador lineal global de proforma

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Facturas\Fabrica
- Utils\Folios

### Consideraciones previas
- Actualmente `tpProformaPedidoFactory` asigna `Folio = null`
- El foliador debe ser un solo contador global sin segmentacion por empresa ni region
- Patron de referencia: `FolioBO.cs` (lock + switch por tipo de consecutivo)
- Pendiente definir politica de folio al cancelar previsualizacion (se conserva o se descarta)
-  La generacion del PDF de proforma se desarrolla en RE-FU-016 y RE-FU-017

### Objetivo general
Crear el algoritmo de foliador lineal global para proformas que asigne un folio unico secuencial a cada proforma generada en el sistema.

### Objetivos especificos
- Crear nuevo caso en `FolioBO` o nueva clase dedicada para consecutivo de proformas
- Implementar lock para concurrencia (un solo contador global)
- Asignar folio al campo `tpProformaPedido.Folio` durante la generacion
- Integrar con `tpProformaPedidoFactory.cs` para que el folio se asigne al crear la proforma
- Validar que el folio es unico e irrepetible

### Resultado esperado
Cada proforma generada en el sistema recibe un folio secuencial unico del foliador lineal global, independientemente de empresa o region.

### Entregables
- Algoritmo de foliador (nueva clase o caso en FolioBO)
- Modificacion de tpProformaPedidoFactory para asignar folio

### Criterios de aceptacion
- El folio es secuencial y unico
- No hay segmentacion por empresa ni region
- Funciona con concurrencia (lock)
- Se asigna correctamente al campo tpProformaPedido.Folio

---

## T2 - [ R16A-RE-FU-013 ] [ALG-BASIC-LOGIC] Verificar aplicabilidad del flujo por empresa

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Facturas\GeneracionProforma
- L05.TramitarPedido\Facturas\Fabrica

### Consideraciones previas
- La proforma se genera por empresa (`tpProformaPedidoFactory` recibe `Empresa`)
-  La generacion del PDF de proforma se desarrolla en RE-FU-016 y RE-FU-017

### Objetivo general
Verificar y documentar que el flujo de Prepago con controlados aplica correctamente para todas las empresas del sistema.

### Objetivos especificos
- Identificar todas las empresas activas en el sistema
- Confirmar que la generacion de proformas por empresa funciona para cada una
- Documentar si hay diferencias en el template PDF por empresa para el equipo de DocumentBuilder
- Ajustar logica si alguna empresa no aplica

### Resultado esperado
Documentacion clara de cuales empresas participan en el flujo y confirmacion de que la proforma se genera correctamente para cada una.

### Entregables
- Documento de verificacion por empresa
- Ajustes en logica (si aplican)

### Criterios de aceptacion
- Se identifican todas las empresas que participan en el flujo
- Se confirma que la proforma se genera correctamente para cada empresa aplicable
- Se documenta informacion para equipo de DocumentBuilder (templates)

---

## T3 - [ R16A-RE-FU-013 ] [IMP-EXIST-SERVICE] Previsualizacion obligatoria del PDF de proforma

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido
- WebApi.Logistica

### Consideraciones previas
- El PDF se genera via DocumentBuilder (desarrollado en RE-FU-016/017)
- El ESAC debe ver el PDF antes de enviar
- Debe soportar cancelacion: volver al pedido sin enviar (Criterio C2)
- El envio del correo se ejecuta en un segundo llamado tras aceptacion
-  La generacion del PDF de proforma se desarrolla en RE-FU-016 y RE-FU-017

### Objetivo general
Implementar endpoint que retorna el PDF de la proforma generada para previsualizacion obligatoria antes del envio al cliente.

### Objetivos especificos
- Crear endpoint GET/POST que reciba IdTPProformaPedido y retorne el PDF (byte[])
- Consumir ApiCallerDocumentBuilder para obtener el PDF
- Soportar flujo de dos pasos: previsualizacion -> aceptacion -> envio
- Soportar cancelacion: si ESAC cancela, no se envia y se vuelve al pedido
- Manejar multiples proformas por pedido (una por empresa)

### Resultado esperado
El ESAC puede visualizar el PDF de la proforma antes de confirmar el envio, con opcion de cancelar y volver sin enviar.

### Entregables
- Endpoint de previsualizacion de PDF
- Logica de consumo de DocumentBuilder

### Criterios de aceptacion
- El PDF se muestra antes del envio (obligatorio)
- El ESAC puede cancelar y volver sin enviar (Criterio C2)
- Funciona para multiples proformas por pedido
- El envio solo ocurre tras aceptacion explicita

---

## T4 - [ R16A-RE-FU-013 ] [IMP-EXIST-SERVICE] Endpoint de envio de correo de proforma

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido
- WebApi.Logistica

### Consideraciones previas
- Se ejecuta despues de que el ESAC acepta la previsualizacion
- El asunto se genera automaticamente: "Proforma " + FolioPedidoInterno (Regla 7)
- Al confirmar envio exitoso se genera pendiente Validar Cobro y se cierra pendiente Tramitar Pedido
-  La generacion del PDF de proforma se desarrolla en RE-FU-016 y RE-FU-017

### Objetivo general
Implementar/ajustar endpoint de envio de correo de proforma con los campos requeridos por el requisito.

### Objetivos especificos
- Endpoint recibe: Para (editable, default contacto cliente), CC (editable, default ESAC), NotasExtras (opcional)
- Asunto generado por sistema: "Proforma " + FolioPedidoInterno (no editable)
- Adjuntos: PDF de la proforma (no editable)
- Al envio exitoso: confirmar que MontoPendiente > 0 marca pendiente Validar Cobro
- Al envio exitoso: cerrar/eliminar pendiente de bandeja Tramitar Pedido (Regla 10)
- Registrar correo en tpPedidoCorreoEnviado

### Resultado esperado
El correo de proforma se envia al cliente con los datos correctos y se generan automaticamente los pendientes derivados.

### Entregables
- Endpoint de envio de correo
- Logica de cierre de pendiente Tramitar Pedido
- Registro en tpPedidoCorreoEnviado

### Criterios de aceptacion
- Correo enviado con Para, CC, Asunto auto, PDF adjunto, Notas extras
- Asunto = "Proforma " + FolioPedidoInterno
- Pendiente Validar Cobro activo tras envio (MontoPendiente > 0)
- Pendiente Tramitar Pedido cerrado tras envio
- Correo registrado en tpPedidoCorreoEnviado

---

## T5 - [ R16A-RE-FU-013 ] [ALG-BASIC-LOGIC] Validaciones regulatorias para controlados

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- Regla 1: FAA y Remision no deben permitirse con controlados
- Regla 9: Datos de facturacion solo lectura en Prepago
- La deteccion de controlados ya existe: `TieneControlados()`

### Objetivo general
Implementar las validaciones Back que impiden la tramitacion con opciones no permitidas cuando hay sustancias controladas.

### Objetivos especificos
- Validar: si `TieneControlados() && FacturaPorAdelantado=1` -> rechazar con error claro
- Validar: si `TieneControlados() && EntregaConRemision=1` -> rechazar con error claro
- Validar: rechazar edicion de datos facturacion cuando SinCredito=1 (endpoint de edicion)
- Integrar validaciones en `tpPedidoTramitarTransaccionBO.cs`

### Resultado esperado
El sistema rechaza tramitaciones que violen las restricciones regulatorias de controlados.

### Entregables
- Validaciones en flujo de tramitacion
- Validacion en endpoint de edicion de datos facturacion

### Criterios de aceptacion
- Error claro si FAA=1 con controlados
- Error claro si EntregaConRemision=1 con controlados
- Error si se intenta editar datos facturacion en Prepago
- Mensajes de error descriptivos

---

## T6 - [ R16A-RE-FU-013 ] [ALG-BASIC-LOGIC] Verificacion operacion Peru

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Liberar

### Consideraciones previas
- El flujo debe operar identico para Mexico y Peru en este modulo
- Peru no transfiere a Legacy (eso ocurre fuera de scope)
- Verificar que `TieneControlados()` funciona con IdRegion de Peru

### Objetivo general
Verificar y ajustar que el flujo de tramitacion Prepago con controlados opera correctamente para la region Peru.

### Objetivos especificos
- Confirmar que `TieneControlados()` recibe y procesa correctamente IdRegion de Peru
- Verificar que no hay logica que excluya Peru del flujo de proformas
- Confirmar que la generacion de proforma, folio, PDF y correo funciona sin dependencia de region
- Documentar diferencias (si las hay)

### Resultado esperado
El flujo de tramitacion Prepago con controlados opera correctamente para pedidos de Peru.

### Entregables
- Verificacion/ajuste de logica para Peru
- Documentacion de diferencias (si aplica)

### Criterios de aceptacion
- TieneControlados() funciona con region Peru
- Proforma se genera correctamente para Peru
- No hay bloqueos por region en este modulo

---

## T7 - [ R16A-RE-FU-013 ] [ALG-BASIC-LOGIC] Vinculacion con generacion PDF de proforma (RE-FU-016/017)

### Aplicativos
- ProquifaDotNet

### Modulos
- L05.TramitarPedido\Facturas

### Consideraciones previas
- La generacion del PDF de proforma se desarrolla en RE-FU-016 y RE-FU-017
- Esta tarea asegura que el PDF generado sea consumible por el flujo de previsualizacion y envio

### Objetivo general
Vincular el proceso de generacion de PDF de proforma (RE-FU-016/017) con el flujo de previsualizacion y envio de correo de este requisito.

### Objetivos especificos
- Verificar que el PDF generado por DocumentBuilder es compatible con el endpoint de previsualizacion (T3)
- Verificar que el PDF se puede adjuntar al correo (T4)
- Ajustar parametros de llamada a DocumentBuilder si es necesario
- Documentar contrato de datos entre este requisito y RE-FU-016/017

### Resultado esperado
El PDF de proforma generado por DocumentBuilder se integra correctamente con el flujo de previsualizacion y envio.

### Entregables
- Ajustes de integracion (si aplican)
- Documentacion del contrato de datos

### Criterios de aceptacion
- El PDF se obtiene correctamente desde DocumentBuilder
- Se muestra en previsualizacion sin errores
- Se adjunta correctamente al correo
- Funciona para todas las empresas aplicables

---

## Resumen de tareas

| # | Clave Catalogo | Titulo | Predecesora |
|---|----------------|--------|-------------|
| T1 | ALG-COMPLX-LOGIC | Foliador lineal global de proforma | - |
| T2 | ALG-BASIC-LOGIC | Verificar aplicabilidad del flujo por empresa | - |
| T3 | IMP-EXIST-SERVICE | Previsualizacion obligatoria del PDF de proforma | T7 |
| T4 | IMP-EXIST-SERVICE | Endpoint de envio de correo de proforma | T3 |
| T5 | ALG-BASIC-LOGIC | Validaciones regulatorias para controlados | - |
| T6 | ALG-BASIC-LOGIC | Verificacion operacion Peru | - |
| T7 | ALG-BASIC-LOGIC | Vinculacion con generacion PDF de proforma (RE-FU-016/017) | - |

---

## Dependencias con otros requisitos (NO incluidas como tareas)

| Requisito      | Tarea relacionada      | Relacion                                                 |
| -------------- | ---------------------- | -------------------------------------------------------- |
| R16A-RE-FU-006 | ReferenciaPago         | Se reconstruye segun ese requisito                       |
| R16A-RE-FU-007 | fnEsProductoControlado | Deteccion de controlados con tipo Origen                 |
| R16A-RE-FU-010 | Cancelacion            | Endpoint de cancelacion se desarrolla en RE-FU-010       |
| R16A-RE-FU-016 | Generacion PDF         | Se desarrolla la generacion del PDF en DocumentBuilder   |
| R16A-RE-FU-017 | Template PDF           | Se desarrolla el template de proforma en DocumentBuilder |
