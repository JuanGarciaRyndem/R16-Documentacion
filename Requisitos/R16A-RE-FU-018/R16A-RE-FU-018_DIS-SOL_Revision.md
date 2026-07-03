# Revisión DIS-SOL — R16A-RE-FU-018

| Campo | Valor |
|---|---|
| **Documento revisado** | `[R16A-RE-FU-018][DIS-SOL] Diseño de la solución.pdf` (v1.0 portada / v1.2 control de versiones, 22-26/06/2026) |
| **Requisito contra el que se valida** | `R16A-RE-FU-018.md` — Factura por Adelantado (Pantalla Inicial) |
| **Reglas contra las que se valida** | `Diseño y Desarrollo\Reglas al diseñar.md` |
| **Autor** | Samuel Hernández Delgado |
| **Revisor** | Juan David García Cruz |
| **Fecha de revisión** | 02 jul 2026 |
| **Estatus** | ⚠️ Con hallazgos críticos — requiere correcciones antes de iniciar desarrollo |

---

## Resumen

Este es, en términos de calidad de documentación, el DIS-SOL más maduro revisado hasta ahora en el proyecto: incluye una sección de "Diagnóstico de arranque" que autoidentifica sus propios bloqueantes (B1-B4) y advertencias (A-E), mapea 1:1 sus propios criterios de aceptación contra los IDs oficiales del requisito, y es transparente marcando "Cubierto parcial" / "Pendiente decisión" en vez de forzar "Cubierto" en criterios que dependen de decisiones de negocio no resueltas (tipo de cambio) o de otro requisito (`Enviada`, RE-FU-019).

Dicho esto, se encontraron dos categorías de hallazgos críticos: (1) **incumplimientos directos y sistemáticos de `Reglas al diseñar.md`** — la nueva base de datos y los seis endpoints nuevos no siguen los estándares de nomenclatura e inglés/rutas que el proyecto exige explícitamente; y (2) **una incompatibilidad arquitectónica real** entre la cadena de datos que 018 asume para leer los pendientes de Factura por Adelantado y el diseño ya definido de RE-FU-015, que rompería el Criterio B1 del propio requisito 018.

---

## Hallazgos críticos (bloqueantes)

### H-01 — La cadena de datos de 018 no alcanza los pendientes que genera RE-FU-015 (Prepago con FAA)

**Sección del PDF:** Diseño funcional detallado → Flujo 1, paso 5 / Diagrama 2 y 3.

**Lo que dice el diseño:** El listado se construye leyendo `tpPedido → tpPedidoProformaPedido → tpProformaPedido → tpProformaAdelantoProformaPedido → fccPagoFacturaAdelanto → tpProformaAdelanto`, y termina filtrando `tpProformaAdelanto.IdCFDIGenerada IS NULL`.

**Lo que ya está definido en RE-FU-015:** El DIS-SOL de RE-FU-015 (revisado el 02 jul 2026, mismo día) decide explícitamente **no reutilizar `tpProformaAdelanto`** para el flujo Prepago con FAA, y en su lugar modela tres tablas nuevas (`fccFactura`, `fccFacturaPartida`, `fccFacturaReferenciaBancaria`) que no se alcanzan desde `tpProformaPedido` ni desde ninguna tabla de la cadena que 018 recorre. Además, RE-FU-015 ya no genera `tpProformaPedido` en absoluto para este flujo (no genera proforma).

**Impacto:** El Criterio B1 del propio requisito 018 exige que el conteo "considere pedidos con pendiente de Factura por Adelantado originados desde Tramitar Pedido en **ambos flujos**: Crédito con Factura por Adelantado y Prepago con Factura por Adelantado". Tal como está diseñada la query de 018, los pedidos Prepago con FAA (RE-FU-015) **nunca aparecerán en el listado**, porque no existe fila en `tpProformaAdelanto` para ellos bajo el diseño actual de 015 — incumpliendo B1 en silencio (sin error, simplemente ausencia de datos).

**Nota de alcance:** No se revisó en este ejercicio el DIS-SOL de RE-FU-012 (la fuente "Crédito con FAA" según la tabla de Dependencias de `018_BD.md`/`018-Back.md`), por lo que no se puede confirmar si su estructura sí coincide con la cadena asumida aquí. Se recomienda verificarlo también antes de dar por buena la cadena completa.

**Acción:** Antes de asignar GAP-14 (`FacturaAdelantadoRepository`), reconciliar explícitamente la fuente de datos contra el diseño vigente de RE-FU-015 y confirmar contra RE-FU-012. Es probable que 018 necesite dos rutas de lectura distintas (una hacia `tpProformaAdelanto` para Crédito, otra hacia `fccFactura` para Prepago) o una vista unificada que combine ambas, en vez de una única cadena de JOINs.

### H-02 — La nueva base de datos `ProquifaDotNetTimbrado` no sigue la Regla 2 ("Bases de datos nuevas estructura en inglés")

**Sección del PDF:** Diseño de Modelo de Datos → Diagrama 4 (ERD) / `_BD.md` script DDL.

`Reglas al diseñar.md` establece sin ambigüedad: *"Base de datos nuevas estructura en inglés"*. `ProquifaDotNetTimbrado` es una base de datos completamente nueva, y su estructura mezcla inglés y español:

- Tabla `TipoDocumentoFiscal` — nombre en español; columnas `Clave`, `Descripcion` en español.
- Tabla `CFDI` — columnas `FechaEmision`, `Moneda`, `TipoCambio`, `MetodoPago`, `FormaPago`, `EstatusTimbrado`, `MensajeError`, `Intentos` en español (mezcladas con `UUID`, `Total`, `CreatedAt`, `IsActive` en inglés).
- Tabla `TimbradoLog` — nombre y columnas `Accion`, `EstatusAnterior`, `EstatusNuevo`, `DuracionMs` en español.
- Solo `AppSetting` cumple la regla de forma completa (`Name`, `Value`, `Description`, `CreatedAt`, `UpdatedAt`, `IsActive`).

**Acción:** Antes de GAP-12 (ejecutar el DDL), renombrar tabla y columnas al inglés (p. ej. `DocumentType`/`Code`/`Description`; `IssueDate`, `Currency`, `ExchangeRate`, `PaymentMethod`, `PaymentForm`, `StampingStatus`, `ErrorMessage`, `Attempts`; `StampingLog`, `Action`, `PreviousStatus`, `NewStatus`, `DurationMs`). Términos regulatorios sin traducción estándar (`CFDI`, `UUID`, `RFC`) pueden conservarse tal cual, como ya ocurre en proyectos fiscales mexicanos en inglés.

### H-03 — Ninguno de los 6 endpoints nuevos sigue el estándar de rutas del proyecto

**Sección del PDF:** Contratos de API.

La regla del proyecto es explícita: `api/v1/{resource}/{id}/{subresource}`, recurso singular en inglés, CRUD por verbo HTTP, acciones especiales solo como subrecurso explícito (ejemplo dado: `POST api/v1/invoice/{id}/cancel`). Los 6 endpoints creados por este DIS-SOL:

| Endpoint del DIS-SOL | Incumplimientos |
|---|---|
| `POST /api/factura-adelantado/listar` | Falta `v1`; recurso en español y no singular-CRUD (`factura-adelantado`); acción `listar` en español como subrecurso |
| `POST /api/timbrado/timbrar` | Falta `v1`; recurso y acción en español, y redundantes entre sí (`timbrado`/`timbrar`) |
| `POST /api/timbrado/cancelar` | Falta `v1`; recurso y acción en español (la regla ya da el ejemplo correcto: `.../cancel`, en inglés) |
| `GET /api/cfdi/{id}` | Falta `v1` (recurso `cfdi` es aceptable como término regulatorio) |
| `GET /api/cfdi/{id}/xml` | Falta `v1` |
| `POST /api/cfdi/listar` | Falta `v1`; acción `listar` en español como subrecurso |

**Acción:** Renombrar las 6 rutas antes de GAP-10/GAP-11/GAP-16, por ejemplo: `POST /api/v1/advanceInvoice/search` (o resolver el listado con `GET /api/v1/advanceInvoice` + query params), `POST /api/v1/cfdi/{id}/stamp`, `POST /api/v1/cfdi/{id}/cancel`, `GET /api/v1/cfdi/{id}`, `GET /api/v1/cfdi/{id}/xml`, `POST /api/v1/cfdi/search`.

### H-04 — No se contempla la llamada a "Bitácora General - Aplicativo Nuevo" al timbrar/guardar el CFDI

`Reglas al diseñar.md` es explícito: *"procesos por ejemplo al guardar una factura, al validar un cobro, al guardar una proforma, etc debe de llamar a Bitácora General - Aplicativo Nuevo"*. El Flujo 2 (Timbrado sincrónico) crea y actualiza el registro `CFDI` — que es, en efecto, "guardar una factura" — y en ningún punto del documento (Flujo 2, Manejo de Errores, Componentes involucrados) se menciona una llamada a Bitácora General. El diseño solo contempla `TimbradoLog` (auditoría técnica del intento contra el PAC) y Serilog (logs de aplicación), que no son sustitutos del patrón de bitácora general de negocio que el proyecto exige de forma transversal.

**Acción:** Antes de GAP-03/GAP-10, agregar el paso de registro en Bitácora General al crear/actualizar el CFDI (éxito y error), y documentar el contrato de esa llamada igual que se hizo con el resto de integraciones (Minio, RabbitMQ, Brevo).

---

## Brechas (no bloqueantes)

### H-05 — Nomenclatura de la solución nueva mezcla términos de dominio en español (Regla 6)

`Reglas al diseñar.md`: *"Sí es nueva solución Codificar en inglés toda la solución (clases, Dtos, Modelos, Procesos, metodos, funciones, y comentarios)"*. `ProquifaDotNet.Timbrado` es una solución nueva, y la mayoría de sus clases llevan el término español "Timbrado" incrustado: `TimbradoService`, `SapTimbradoClient`, `TimbradoController`, `TimbradoWorker`, `TimbradoLog`, `TimbradoRequestDto`, `TimbradoResponseDto`. Lo mismo ocurre en el módulo nuevo de `ProquifaDotNet.Finanzas`: `FacturaAdelantadoRepository`, `FacturaAdelantadoService`, `FacturaAdelantadoController`, `FacturaAdelantadoClienteDto`. Relacionado con H-02 pero a nivel de código en vez de esquema de BD. No bloqueante porque `CFDI`/`UUID`/`RFC` sí son términos regulatorios sin traducción estándar y sería razonable conservarlos, pero `Timbrado`→`Stamping` y `FacturaAdelantado`→`AdvanceInvoice` no tienen esa excusa.

### H-06 — `EmpresaDatosTimbrado` (tabla nueva en ProquifaDotNet) sin columnas ni DDL documentados

La tabla aparece mencionada en "Impacto en modelos" ("Almacena RFC emisor y datos del PAC por empresa. Creada en RE-FU-018") y referenciada en un comentario del contrato de `TimbradoRequestDto`, pero a diferencia de las 4 tablas de `ProquifaDotNetTimbrado` (que sí tienen ERD y DDL completos), esta tabla nunca se define con columnas, tipos ni script. Dado que vive en `ProquifaDotNet` (BD existente, español permitido por Regla 1), conviene completarla con el mismo nivel de detalle que el resto antes de GAP-05.

### H-07 — Nombre del controlador FAA en ProquifaDotNet inconsistente dentro del propio documento

La tabla "Componentes involucrados" (pág. 4) lo llama `ControladorFAA (nuevo)`; la sección "Impacto en código existente" (pág. 23) lo llama `FacturaAdelantadoController.cs`. Ambos nombres son aceptables bajo la Regla 3 (PQF Catálogos se deja en español), pero deberían unificarse a uno solo para evitar ambigüedad al momento de codificar.

### H-08 — Encabezado con placeholder de plantilla sin reemplazar y versión inconsistente

Todas las páginas (excepto la portada) muestran el encabezado literal "Diseño de la solución - Requisito R16A-RE-FU-XXX." — el placeholder `XXX` de la plantilla nunca se reemplazó por `018`. Además, la portada indica VERSIÓN 1.0 mientras que la tabla de Control de versiones indica 1.2. Mismo patrón cosmético ya señalado en las revisiones de 013/014/015.

---

## Puntos que están bien

- **Diagnóstico de arranque:** la sección que autoidentifica bloqueantes (B1 proveedor PAC TurboPac vs. SAP sin resolver, B2 dependencia del campo `Enviada` de RE-FU-019, B3 tipo de cambio pendiente, B4 callback del Worker indefinido) y advertencias (A-E) es una práctica ejemplar — el autor hizo el trabajo de decir explícitamente qué se puede construir ya y qué no, en vez de dejarlo implícito.
- **Trazabilidad de criterios:** los 11 criterios de aceptación del DIS-SOL corresponden exactamente en cantidad e ID a los 11 criterios oficiales del requisito (A1-A3, B1-B2, C1-C4, D1-D2) — a diferencia de otros DIS-SOL revisados donde el mapeo era ambiguo.
- **Transparencia en criterios parciales:** en vez de marcar A3/B1/B2 como "Cubierto" a la fuerza, se marcan correctamente como "Cubierto parcial" o "Pendiente decisión", citando la dependencia exacta (OBS-032/033, RE-FU-019 GAP-20, decisión de negocio pendiente).
- **RT-10 / `NumeroCPE`:** anticipar la columna que necesitará RE-FU-020 (Perú) desde el `CREATE TABLE` de GAP-12, con justificación explícita de costo cero vs. `ALTER TABLE` futuro con datos en producción — buena práctica preventiva.
- **Consistencia con `Back.md`/`_BD.md` de este mismo requisito:** la cadena de datos y las tablas consultadas coinciden exactamente entre el DIS-SOL y los documentos complementarios ya existentes de 018 (a diferencia de la discrepancia detectada en H-01 contra el requisito 015, que es externo a este expediente).
- **Manejo de errores:** la tabla de escenarios (PAC caído, RFC inválido, Worker agota reintentos, Minio no disponible, usuario sin cartera) es exhaustiva y cada fila tiene un comportamiento HTTP explícito.

---

## Referencias

- `R16A-RE-FU-018.md`
- `[R16A-RE-FU-018][DIS-SOL] Diseño de la solución.pdf`
- `R16A-RE-FU-018-Back.md`, `R16A-RE-FU-018_BD.md`
- `Diseño y Desarrollo\Reglas al diseñar.md`
- `R16A-RE-FU-015_DIS-SOL_Revision.md` (fuente del hallazgo H-01: decisión de no reutilizar `tpProformaAdelanto`)
