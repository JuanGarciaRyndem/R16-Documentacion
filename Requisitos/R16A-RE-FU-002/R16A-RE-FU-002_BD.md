# Diccionario de Datos - Asignacion de Cobrador por Cliente
**Requisito:** R16A-RE-FU-002
**Base de Datos:** ProquifaDotNet

---

## Resumen Ejecutivo
Asignar un Cobrador (Gestor de Cobranza) a cada cliente a traves de carteras para distribuir
la carga operativa y habilitar visibilidad filtrada en Validar Cobro, Factura por Adelantado,
Buzón de Pagos y **Notas de Crédito** (OBS-004).

---

## Modelo de Datos

La asignacion de Cobrador NO es directa en Cliente sino a traves del modelo de Cartera:

```sql
    Usuario (Cobrador / ESAC / EVI / EVE / Coordinadores)
        FK IdUsuarioCobrador
    ClienteCartera  (NombreCartera, IdUsuarioCobrador, IDUsuarioESAC, IdUsuarioEVI, ...)
        FK IdClienteCartera
    ClienteCarteraCliente  (IdClienteCartera + IdCliente)
        FK IdCliente
    Cliente
```

---

## Entidades Afectadas

| Objeto | Tipo | Impacto | Descripcion |
|--------|------|---------|-------------|
| ClienteCartera | Tabla | Principal - define cartera y cobrador | Contiene IdUsuarioCobrador como campo clave |
| ClienteCarteraCliente | Tabla | Principal - vincula clientes a cartera | Relacion cartera con cliente |
| vClienteCarteraCliente | Vista | Consulta - expone cartera con cobrador | Vista operativa con datos completos |
| vUsuarioCartera | Vista | Consulta - vincula usuario con cliente | Usada para filtrar bandeja por cobrador |
| spDesactivarCarterasCliente | SP | Operacion - baja de carteras del cliente | Desactiva todas las carteras de un cliente |
| Cliente | Tabla | Referenciada | Catalogo de clientes |
| Usuario | Tabla | Referenciada - fuente del selector | Usuarios con rol Gestor de Cobranza |
| Region | Tabla | Referenciada | Region asociada a cartera y cliente |

---

## 1. ClienteCartera (TABLA PRINCIPAL)
**Proposito:** Define una cartera y asigna usuarios operativos incluyendo el Cobrador
**Creada:** 22/04/2022

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdClienteCartera | uniqueidentifier | NO | PK |
| NombreCartera | varchar(300) | SI | Nombre descriptivo de la cartera |
| IdUsuarioRegistro | uniqueidentifier | SI | FK - Usuario que registra la cartera |
| IDUsuarioESAC | uniqueidentifier | SI | FK - Usuario con rol ESAC asignado |
| IdUsuarioEVI | uniqueidentifier | SI | FK - Usuario con rol EVI asignado |
| IdUsuarioEVE | uniqueidentifier | SI | FK - Usuario con rol EVE asignado |
| **IdUsuarioCobrador** | **uniqueidentifier** | **SI** | **FK - Campo clave: Gestor de Cobranza asignado** |
| IdUsuarioCoordinadorDeVentaInterna | uniqueidentifier | SI | FK - Coordinador de Venta Interna |
| IdUsuarioCoordinadorDeServicioAlCliente | uniqueidentifier | SI | FK - Coordinador de Servicio al Cliente |
| IdUsuarioGerenteDeVentas | uniqueidentifier | SI | FK - Gerente de Ventas |
| IdRegion | uniqueidentifier | NO | FK - Region de la cartera |
| FechaRegistro | datetime | NO | Fecha de creacion. Default: GETDATE() |
| FechaUltimaActualizacion | datetime | NO | Fecha de modificacion. Default: GETDATE() |
| Activo | bit | NO | Estado activo. Default: 1 |

**Indices:**
- PK_ClienteCartera (Clustered): IdClienteCartera
- IX_ClienteCartera_Usuarios (Non-Clustered): IdUsuarioRegistro, IDUsuarioESAC, IdUsuarioEVI,
  IdUsuarioEVE, IdUsuarioCobrador, IdUsuarioCoordinadorDeVentaInterna,
  IdUsuarioCoordinadorDeServicioAlCliente, IdUsuarioGerenteDeVentas, IdRegion

---

## 2. ClienteCarteraCliente
**Proposito:** Vincula clientes con una cartera (y por extension con el Cobrador asignado)
**Creada:** 22/04/2022

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdClienteCarteraCliente | uniqueidentifier | NO | PK |
| IdClienteCartera | uniqueidentifier | NO | FK - ClienteCartera |
| IdCliente | uniqueidentifier | NO | FK - Cliente |
| IdRegion | uniqueidentifier | NO | FK - Region |
| FechaRegistro | datetime | NO | Fecha de creacion. Default: GETDATE() |
| FechaUltimaActualizacion | datetime | NO | Fecha de modificacion. Default: GETDATE() |
| Activo | bit | NO | Estado activo. Default: 1 |

**Indices:**
- PK_ClienteCarteraCliente (Clustered): IdClienteCarteraCliente
- IX_ClienteCarteraCliente (Non-Clustered): IdClienteCartera, IdCliente, Activo, IdRegion

---

## 3. vClienteCarteraCliente (Vista Operativa Principal)
**Proposito:** Vista que expone la cartera completa con cobrador y demas usuarios asignados
**Creada:** 10/06/2022

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdCliente | uniqueidentifier | NO | Cliente de la cartera |
| Nombre | varchar(300) | NO | Nombre del cliente |
| Alias | varchar(300) | NO | Alias del cliente |
| IdClienteCarteraCliente | uniqueidentifier | NO | PK de la relacion |
| IdClienteCartera | uniqueidentifier | NO | Cartera asignada |
| FechaRegistro | datetime | NO | Fecha de asignacion |
| NombreCartera | varchar(300) | SI | Nombre de la cartera |
| IdUsuarioRegistro | uniqueidentifier | SI | Usuario que registro |
| IdUsuarioESAC | uniqueidentifier | SI | ID del ESAC asignado |
| ESAC | varchar(50) | SI | Nombre del ESAC |
| IdUsuarioEVI | uniqueidentifier | SI | ID del EVI asignado |
| EVI | varchar(50) | SI | Nombre del EVI |
| IdUsuarioEVE | uniqueidentifier | SI | ID del EVE asignado |
| EVE | varchar(50) | SI | Nombre del EVE |
| **IdUsuarioCobrador** | **uniqueidentifier** | **SI** | **ID del Cobrador asignado** |
| **Cobrador** | **varchar(50)** | **SI** | **Campo clave: Nombre del Cobrador** |
| NombreRegion | varchar(50) | SI | Region de la cartera |
| ClaveISO | varchar(50) | SI | Clave ISO de la region |
| IdRegion | uniqueidentifier | NO | ID de la region |

---

## 4. vUsuarioCartera (Vista de Filtrado por Usuario)
**Proposito:** Vincula un usuario directamente con sus clientes asignados via cartera.
Usada por modulos para filtrar bandeja por cobrador.
**Creada:** 22/04/2022

| Columna | Tipo | Nulo | Descripcion |
|---------|------|------|-------------|
| IdCliente | uniqueidentifier | NO | Cliente asignado al usuario |
| IdUsuario | uniqueidentifier | NO | Usuario asignado (cobrador u otro rol) |
| NombreRegion | varchar(50) | SI | Region |
| ClaveISO | varchar(50) | SI | Clave ISO de la region |
| IdRegion | uniqueidentifier | NO | ID de la region |
| Activo | bit | NO | Relacion activa |

---

## 5. spDesactivarCarterasCliente (Stored Procedure)
**Proposito:** Desactiva todas las carteras activas de un cliente.
Usado cuando un cliente es dado de baja o reasignado completamente.
**Creado:** 06/Febrero/2024  **Autor:** Carlos Ivan Morales Carreon

**Parametros:**

| Parametro | Tipo | Descripcion |
|-----------|------|-------------|
| @IdCliente | uniqueidentifier | Cliente cuyas carteras se desactivan |

**Comportamiento:**
- Desactiva (Activo=0) todos los registros en ClienteCarteraCliente para el cliente
- Retorna detalle de carteras afectadas para auditoria
- Usa transaccion con manejo de errores TRY/CATCH
- Hace ROLLBACK en caso de error

**Nota:** El SP no expone IdUsuarioCobrador en el SELECT de retorno.
Considerar incluirlo para trazabilidad del cobrador en reasignaciones.

---

## Mapeo de Roles del Requisito

| Rol en Requisito | Campo en Usuario | Columna en ClienteCartera |
|------------------|-----------------|--------------------------|
| Coordinador de Tesorería | `GerenteDeTesoreria` | — (solo edita Cobrador) |
| **Gerente de Tesorería** | **Pendiente confirmar campo** (OBS-003) | — (solo edita Cobrador) |
| Gestor de Cobranza | `AnalistaDeCuentasPorCobrar` | `IdUsuarioCobrador` |

> **PENDIENTE (OBS-003):** Confirmar qué campo de `Usuario` mapea al rol **Gerente de Tesorería**. Si el campo `GerenteDeTesoreria` ya cubre este rol (y el Coordinador usa otro campo), actualizar el mapeo. Si se requiere un campo nuevo, incluirlo en el script de migración.
>
> **PENDIENTE:** Confirmar si `AnalistaDeCuentasPorCobrar` es exactamente el rol Gestor de Cobranza o si se requiere un campo nuevo en la tabla Usuario.

---

## Consultas SQL Principales

### Selector: Gestores de Cobranza activos

```sql
    SELECT
        IdUsuario,
        NombreCompleto
    FROM dbo.Usuario
    WHERE AnalistaDeCuentasPorCobrar = 1
      AND Activo = 1
    ORDER BY NombreCompleto;
```

### Clientes por cobrador (via vUsuarioCartera)

```sql
    DECLARE @IdUsuarioCobrador UNIQUEIDENTIFIER;

    SELECT
        vc.IdCliente,
        vc.IdUsuario
    FROM dbo.vUsuarioCartera vc
    WHERE vc.IdUsuario = @IdUsuarioCobrador
      AND vc.Activo    = 1;
```

### Bandeja completa por cobrador (via vClienteCarteraCliente)

```sql
    DECLARE @IdUsuarioCobrador UNIQUEIDENTIFIER;

    SELECT
        v.IdCliente,
        v.Nombre        AS Cliente,
        v.NombreCartera,
        v.Cobrador,
        v.NombreRegion
    FROM dbo.vClienteCarteraCliente v
    WHERE v.IdUsuarioCobrador = @IdUsuarioCobrador
    ORDER BY v.Nombre;
```

### Clientes sin Cobrador asignado

```sql
    SELECT
        v.IdCliente,
        v.Nombre,
        v.NombreCartera
    FROM dbo.vClienteCarteraCliente v
    WHERE v.IdUsuarioCobrador IS NULL
    ORDER BY v.Nombre;
```

### Reasignar Cobrador en una cartera

```sql
    DECLARE @IdClienteCartera UNIQUEIDENTIFIER;
    DECLARE @IdNuevoCobrador  UNIQUEIDENTIFIER;

    UPDATE dbo.ClienteCartera
    SET    IdUsuarioCobrador        = @IdNuevoCobrador,
           FechaUltimaActualizacion = GETDATE()
    WHERE  IdClienteCartera = @IdClienteCartera;
```

### Desactivar todas las carteras de un cliente

```sql
    DECLARE @IdCliente UNIQUEIDENTIFIER;
    EXEC dbo.spDesactivarCarterasCliente @IdCliente = @IdCliente;
```

---

## Reglas de Negocio

| Regla | Descripcion | Implementacion |
|-------|-------------|----------------|
| Regla 1 | **Coordinador de Tesorería O Gerente de Tesorería** pueden editar el Cobrador de un cliente (OBS-003) | Validar ambos roles en capa aplicacion |
| Regla 2 | Cobrador debe ser Gestor de Cobranza activo | WHERE AnalistaDeCuentasPorCobrar=1 AND Activo=1 |
| Regla 3 | Un solo Cobrador por cliente | ClienteCartera.IdUsuarioCobrador campo unico por cartera |
| Regla 4 | Filtrado dinamico por cobrador actual | JOIN via ClienteCarteraCliente - ClienteCartera |
| Regla 5 | Cliente sin cobrador invisible en bandejas | IdUsuarioCobrador IS NULL excluido del filtro |

---

## Analisis de Gaps

| Gap | Descripcion | Accion Sugerida |
|-----|-------------|-----------------|
| Mapeo exacto de rol | Confirmar si AnalistaDeCuentasPorCobrar = Gestor de Cobranza | Validar con equipo |
| SP sin IdUsuarioCobrador | spDesactivarCarterasCliente no retorna cobrador en SELECT | Actualizar SP para incluirlo |
| Historial de asignaciones | **Resuelto — OBS-005:** El trabajo ya realizado por el cobrador anterior permanece registrado en los módulos donde fue ejecutado (FxA, Validar Cobro, etc.). La reasignación opera por pantalla/módulo: solo los pendientes aún abiertos (no finalizados) pasan al nuevo cobrador. No se requiere migración de registros históricos. | Aplicado en criterio C4 del Back |
| Campo Gerente de Tesorería | Confirmar qué campo de Usuario mapea al rol Gerente de Tesorería (OBS-003) | Validar con equipo antes de implementar GAP-04 |

---

## Modulos Consumidores

| Modulo | Filtro Requerido |
|--------|-----------------|
| Validar Cobro | JOIN vUsuarioCartera WHERE IdUsuario = @CobActual |
| Factura por Adelantado | JOIN vUsuarioCartera WHERE IdUsuario = @CobActual |
| Buzón de Pagos | JOIN vUsuarioCartera WHERE IdUsuario = @CobActual |
| **Notas de Crédito** | JOIN vUsuarioCartera WHERE IdUsuario = @CobActual (OBS-004) — ver R16A-RE-FU-032 Criterio A5 y R16A-RE-FU-033 Criterio A3 |

---

## Riesgos

| # | Riesgo | Mitigacion |
|---|--------|------------|
| 1 | Clientes sin cobrador quedan invisibles operativamente | Alerta al crear cliente sin asignar cartera |

---
