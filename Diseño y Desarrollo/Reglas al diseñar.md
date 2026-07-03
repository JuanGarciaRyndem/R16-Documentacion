- Base de datos de ProquifaDotNet va a estar en español (tablas, vistas, procedimiento almacenados, funciones, etc)
- Base de datos nuevas estructura en inglés
- Codificación en PQF Catalogos se deja en español
- Codificación en PQF Logistica si es actualización de proceso se deja en español
- Codificación en PQF Logistica si es nuevo EndPoint y nuevo proceso se actualiza modelo, controller en Inglés
- Sí es nueva solución Codificar en inglés  toda al solución(clases, Dtos, Modelos, Procesos, metodos, funciones, y comentarios)
- Los envíos de correo se usa por Aplicativo para Envio de Correo - Aplicativo Nuevo
- Tener en cuenta que procesos por ejemplo al guardar una factura, al validar un cobro, al guardar una proforma, etc debe de llamar a Bitácora General - Aplicativo Nuevo
- Para los nuevos EndPoints y Nuevos aplicativos usar la siguiente arquitectura de estructura:

### Reglas al crear la ruta de EndPoints

```api/v1/{resource}/{id}/{subresource}```
### Componentes:
- **api/v1/** → versión base de tu API.
- **{resource}** → entidad principal (ejemplo: `invoice`, `payment`, `creditNote`).
- **{subresource}** → opcional, si no es CRUD estándar (ejemplo: `cancel`, `stamp`, `report`).
- **Singular en inglés** → `invoice`, `payment`, `creditNote`.
- **CRUD estándar** → se expresa con el método HTTP.
- **Acciones especiales** → se pueden modelar como cambios de estado o subrecursos.

## 🛠️ Ejemplos concretos

| Caso de uso                      | Endpoint                     | Método |
| -------------------------------- | ---------------------------- | ------ |
| Crear factura                    | `api/v1/invoice`             | POST   |
| Obtener factura                  | `api/v1/invoice/{id}`        | GET    |
| Actualizar factura               | `api/v1/invoice/{id}`        | PUT    |
| Eliminar factura                 | `api/v1/invoice/{id}`        | DELETE |
| Cancelar factura (caso especial) | `api/v1/invoice/{id}/cancel` | POST   |
| Registrar pago                   | `api/v1/payment`             | POST   |
| Obtener pago                     | `api/v1/payment/{id}`        | GET    |
| Generar nota de crédito          | `api/v1/creditNote`          | POST   |
| Obtener nota de crédito          | `api/v1/creditNote/{id}`     | GET    |
## ✅ Buenas prácticas aplicadas

- **Singular**: `invoice` en lugar de `invoices`.
- **Inglés**: consistente y estándar para integraciones.
- **Acciones explícitas solo cuando son casos de uso especiales** (ej. `cancel`, `export`).
- **CRUD puro** se maneja con el verbo HTTP, no con la ruta.
- Acciones especiales (`cancel`, `stamp`, `report`) solo cuando el caso de uso lo requiere.
- Consistencia en prefijo `api/v1/` para versionado.