# 📐 Reglas de Nombramiento

## Reglas Generales

| Regla | Detalle |
|---|---|
| **Idioma** | Inglés en todos los objetos |
| **Case** | PascalCase (tras el prefijo) |
| **Número** | Singular |
| **Esquema** | `dbo` por defecto |
| **Guiones / Espacios** | No permitidos |
| **Abreviaciones** | Evitarlas; usar nombres descriptivos completos |

---

## Prefijos por Tipo de Objeto

| Tipo | Prefijo | Ejemplo |
|---|---|---|
| Tabla | *(ninguno)* | `PunchOutSession` |
| Catálogo | *(ninguno)* | `Currency` |
| Vista | `v` | `vTenantAddress` |
| Stored Procedure | `sp` | `spAppSettingsUpdate` |
| Función | `fn` | `fnGetActiveTenants` |

---

## 🗄️ Tablas

- **Formato:** `PascalCase`
- **Número:** Singular
- **Prefijo / Sufijo:** Ninguno
- **Esquema:** `dbo`

```
✅ Product
✅ PunchOutSession
✅ InboundOrderItem
✅ SupplierCompany
```

---

## 📋 Catálogos

- **Formato:** `PascalCase`
- **Número:** Singular
- **Prefijo / Sufijo:** Ninguno — se distinguen por contexto, **no por prefijo**
- **Esquema:** `dbo`

```
✅ Brand
✅ Currency
✅ PaymentCondition
```

> Los catálogos siguen exactamente la misma convención que las tablas transaccionales.

---

## 👁️ Vistas

- **Formato:** `v` + `PascalCase`
- **Prefijo:** `v` (minúscula)
- **Esquema:** `dbo`

```
✅ vTenantAddress
```

---

## ⚙️ Procedimientos Almacenados

- **Formato:** `sp` + `PascalCase`
- **Prefijo:** `sp` (minúscula, **sin guión bajo**)
- **Idioma:** Inglés
- **Esquema:** `dbo`

```
✅ spAppSettingsUpdate
✅ spCreateProductStaging
✅ spProductSyncInitialize
✅ spPunchOutSessionExpireInactive
✅ spRecreateProductFullTextIndex
```

> ⚠️ **Prohibido** usar el prefijo `sp_` con guión bajo.  
> SQL Server lo interpreta como un procedimiento del sistema, buscando primero en `master`  
> antes que en la base de datos actual, lo que genera un impacto negativo en rendimiento.

```
❌ sp_SearchProducts      → ✅ spSearchProducts
❌ spProductoSyncProcess  → ✅ spProductSyncProcess
```

---

## 🔧 Funciones

- **Formato:** `fn` + `PascalCase`
- **Prefijo:** `fn` (minúscula)
- **Esquema:** `dbo`

```
✅ fnGetActiveTenants
✅ fnCalculateDiscount
```

---

## 📁 Archivos de Scripts SQL

Los archivos de modificación, creación, inserción o actualización deben seguir esta convención de nombre:

```
/Requisito/Seq_NombreObjeto_Accion.sql
```

| Segmento         | Descripción                                 | Ejemplo                                             |
| ---------------- | ------------------------------------------- | --------------------------------------------------- |
| **Requisito**    | Número del requisito que se está trabajando | `REQ-42`                                            |
| **Seq**          | Número secuencial de 3 dígitos              | `001`, `002`, `015`                                 |
| **NombreObjeto** | Nombre del objeto de base de datos          | `Client`, `Product`, `vClient`, `fnCalculaProducto` |
| **Accion**       | Acción que realiza el script                | `CREATE`, `ALTER`, `INSERT`, `UPDATE`, `DELETE`     |

**Ejemplos:**

```
✅ /REQ-42/001_Client_CREATE.sql
✅ /REQ-42/002_Product_ALTER.sql
✅ /REQ-42/003_vClient_CREATE.sql
✅ /REQ-42/004_fnCalculaProducto_CREATE.sql
✅ /REQ-42/005_Pedido_INSERT.sql
✅ /REQ-42/006_Inventario_UPDATE.sql
```

---

## ❌ Patrones No Permitidos

| Patrón                    | Motivo                                                    |
| ------------------------- | --------------------------------------------------------- |
| `sp_NombreProcedimiento`  | Prefijo reservado por SQL Server para objetos del sistema |
| `tbl_NombreTabla`         | Prefijo innecesario; rompe la convención PascalCase       |
| `vw_NombreVista`          | Prefijo incorrecto; usar `v` sin guión bajo               |
| Nombres en español        | Se debe mantener consistencia en inglés                   |
| Nombres en plural         | Todos los objetos deben ser en singular                   |
| Abreviaciones no estándar | Dificultan la legibilidad y mantenimiento                 |
