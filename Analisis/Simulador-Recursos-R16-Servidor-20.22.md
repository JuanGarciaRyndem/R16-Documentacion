# Simulador de Recursos — Proyecto R16

**Servidor:** 20.22  
**Versión:** 1.0  
**Fecha:** 2026-06-18

---

## Descripción

Este simulador estima el impacto en infraestructura (CPU y RAM) que tendrá el nuevo release del proyecto R16 sobre el servidor 20.22, considerando la concurrencia real del sistema, la tasa de adopción de la nueva funcionalidad y el factor de complejidad del release.

---

## 1. Usuarios Concurrentes Efectivos (UCE)

Los usuarios concurrentes efectivos representan la fracción de usuarios totales que operan simultáneamente el sistema.

**Porcentaje de concurrencia:**

```
Usuarios ejecutando un proceso   10
────────────────────────────── = ── = 0.50  (50%)
Usuarios con sesión iniciada     20
```

**UCE:**

```
Total de Usuarios × Porcentaje de concurrencia = UCE
          50      ×            0.50             =  25
```

| Parámetro | Valor |
|---|---|
| Usuarios con sesión iniciada | 20 |
| Usuarios ejecutando un proceso | 10 |
| Porcentaje de concurrencia | 0.50 |
| Total de usuarios | 50 |
| **UCE** | **25** |

---

## 2. Tasa de Adopción (TA)

Representa el porcentaje de usuarios concurrentes que realmente utilizarán la nueva funcionalidad del release.

**Fórmula:**

```
     Usuarios que usarán la nueva funcionalidad
TA = ──────────────────────────────────────────
              Usuarios Concurrentes (UCE)
```

**Cálculo:**

```
TA = 6 / 25 = 0.24  (24%)
```

| Parámetro | Valor |
|---|---|
| Usuarios que usarán la nueva funcionalidad | 6 |
| UCE | 25 |
| **Tasa de Adopción (TA)** | **0.24** |

---

## 3. Consumo Histórico por Usuario — Percentil 95 (CHU P95)

El consumo base de CPU y RAM se obtiene a partir del percentil 95 de uso observado durante periodos de alta concurrencia. El objetivo es dimensionar la infraestructura considerando los picos operativos del sistema.

> Los valores son proporcionados por el equipo de Proquifa (área de seguridad digital y hardware) a partir de métricas reales del servidor.

### CPU por usuario

```
CPU_P95   15.3
──────── = ──── = 0.612 % CPU / usuario
  UCE       25
```

### RAM por usuario

```
RAM_P95   39.85
──────── = ───── = 1.594 % RAM / usuario
  UCE       25
```

| Parámetro | Valor |
|---|---|
| CPU_P95 (% total) | 15.3 % |
| RAM_P95 (% total) | 39.85 % |
| UCE | 25 |
| **% CPU por usuario** | **0.612 %** |
| **% RAM por usuario** | **1.594 %** |

---

## 4. Factor de Complejidad del Release (FCR)

El FCR representa el impacto esperado que el nuevo release puede generar sobre el consumo de infraestructura. Se selecciona **un solo nivel por release**; en caso de coexistir múltiples cambios, se selecciona el de mayor impacto esperado.

| Nivel | Descripción | FCR | Aplicado |
|---|---|---|---|
| **Nivel 0** — Sin impacto funcional o de procesamiento | Correcciones de errores menores, ajustes de configuración, cambios visuales, refactorización sin cambio de comportamiento, optimizaciones sin incremento en operaciones. Sin incremento en CPU, RAM, almacenamiento ni requests. | 1.00 | |
| **Nivel 1** — Ajustes funcionales menores | Validaciones adicionales en formularios, ajustes menores en lógica de negocio, modificaciones de UI con llamadas ligeras a servicios, consultas simples sobre información existente. Impacto bajo en CPU y memoria, despreciable en almacenamiento. | 1.05 | |
| **Nivel 2** — Nuevas consultas o ampliación de flujos | Nuevos endpoints de consulta, ampliación de procesos con pasos adicionales, nuevas consultas a BD, cálculos o transformaciones ligeras. Incremento moderado en requests y CPU, leve en memoria. | 1.10 | |
| **Nivel 3** — Nuevos procesos transaccionales | Creación de nuevos procesos operativos, inserción de registros en BD, múltiples operaciones por transacción, integración con servicios internos adicionales. Incremento significativo en operaciones, moderado a alto en CPU y memoria, incremento en datos. | 1.15 | ✓ |
| **Nivel 4** — Procesamiento intensivo o generación masiva | Generación de documentos o reportes masivos, procesamiento batch, agregaciones complejas, generación o almacenamiento de archivos, procesamiento intensivo de datos. Incremento alto en CPU, RAM y almacenamiento; posible impacto en tiempos de respuesta. | 1.20 | |

> **Release R16 — Nivel seleccionado: Nivel 3 (FCR = 1.15)**  
> Se introducen nuevos procesos transaccionales (Buzón de Cobros, Validar Cobro, Facturación, Notas de Crédito), inserción de registros en BD e integración con servicios internos (Finanzas, Timbrado, LegacySync).

---

## 5. Crecimiento Histórico del Sistema (UCE Proyectado)

El crecimiento proyectado se basa en el incremento histórico promedio de concurrencia observado en los últimos periodos.

**Fórmula:**

```
UCE Proyectado = UCE × (1 + Crecimiento Promedio)
```

**Ejemplo de cálculo del crecimiento promedio:**

| Mes | Usuarios Concurrentes Pico | Crecimiento |
|---|---|---|
| Enero | 170 | — |
| Febrero | 180 | (180 − 170) / 170 = 5.88 % |
| Marzo | 196 | (196 − 180) / 180 = 8.89 % |
| **Promedio** | — | **(5.88 % + 8.89 %) / 2 = 7.39 %** |

> **Nota:** El crecimiento promedio se actualiza conforme se van acumulando datos históricos reales del servidor. Para este cálculo se usa el valor actual disponible.

| Parámetro | Valor |
|---|---|
| UCE | 25 |
| Crecimiento promedio (factor) | 1.00 *(sin datos históricos suficientes; se actualizará)* |
| **UCE Proyectado** | **25** |

---

## 6. Modelo Formal de Impacto en Infraestructura

El impacto en infraestructura no depende únicamente del número de usuarios, sino de la interacción entre concurrencia, frecuencia y costo por operación.

**Fórmulas:**

```
CPU Proyectada = CPU_pu × UCE_proy × (1 + (FCR − 1) × TA)
RAM Proyectada = RAM_pu × UCE_proy × (1 + (FCR − 1) × TA)
```

Donde:
- `CPU_pu` / `RAM_pu`: Consumo por usuario concurrente efectivo (P95)
- `UCE_proy`: UCE con crecimiento histórico aplicado
- `FCR`: Factor de Complejidad del Release
- `TA`: Tasa de Adopción de la nueva funcionalidad

### Cálculo CPU

```
CPU Proyectada = 0.612 × 25 × (1 + (1.15 − 1) × 0.24)
               = 0.612 × 25 × (1 + 0.15 × 0.24)
               = 0.612 × 25 × 1.036
               = 15.3 × 1.036
               = 15.8508 %
```

### Cálculo RAM

```
RAM Proyectada = 1.594 × 25 × (1 + (1.15 − 1) × 0.24)
               = 1.594 × 25 × 1.036
               = 39.85 × 1.036
               = 41.2846 %
```

---

## 7. Resultado — Recomendación de Escalado CPU/RAM

| Recurso | Valor Actual (P95) | Valor Proyectado | Umbral crítico | Recomendación |
|---|---|---|---|---|
| **CPU** | 15.30 % | 15.85 % | > 75 % | ✓ **No escalar** |
| **RAM** | 39.85 % | 41.28 % | > 80 % | ✓ **No escalar** |

> El servidor 20.22 cuenta con capacidad suficiente para absorber el impacto del release R16 sin requerir aumento de recursos. Los valores proyectados se mantienen significativamente por debajo de los umbrales críticos (CPU 75 %, RAM 80 %).

---

## 8. Almacenamiento SSD

### 8.1 Parámetros de Capacidad

| Parámetro                                | Valor    | Descripción                                                                                                                           |
| ---------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **CTI — Capacidad Total Instalada**      | 500 GB   | Capacidad total del disco al momento del análisis                                                                                     |
| **CAU — Capacidad Actual Utilizada**     | 139 GB   | Capacidad en uso al momento del análisis                                                                                              |
| **COH — Crecimiento Orgánico Histórico** | 1 GB/mes | Promedio mensual real de crecimiento estructural, posterior a mantenimiento operativo (rotación/limpieza de logs, temporales y caché) |
| **HP — Horizonte de Proyección**         | 3 meses  | Periodo a proyectar posterior al release                                                                                              |

> El COH refleja únicamente el crecimiento persistente del sistema, calculado después de los procesos normales de mantenimiento operativo (rotación de logs, archivos temporales, caché).

---

### 8.2 Factor de Incremento por Release (FIR)

El FIR representa el impacto estructural del release sobre el ritmo de crecimiento del almacenamiento. Se selecciona **un solo nivel por release**; en caso de coexistir múltiples cambios, se selecciona el de mayor impacto.

| Nivel | Descripción | FIR | Aplicado |
|---|---|---|---|
| **Nivel 1** — Sin persistencia nueva | Cambios de UI, validaciones, ajustes de lógica existente, refactorización, optimización de consultas, corrección de errores. Sin nuevas tablas, registros persistentes ni archivos. Sin impacto en almacenamiento estructural. | 1.00 | |
| **Nivel 2** — Nuevos catálogos o entidades de baja volumetría | Nuevos catálogos de configuración, tablas de parámetros o referencia, información administrativa, estructuras de baja cardinalidad. Crecimiento lento y reducido. | 1.02 | |
| **Nivel 3** — Nuevas tablas transaccionales | Nuevas tablas OLTP, incremento en INSERT/UPDATE/DELETE, crecimiento de tablas existentes por nuevos flujos, históricos, bitácoras funcionales, nuevas relaciones (FK), índices adicionales. Crecimiento progresivo proporcional a las transacciones del sistema. | 1.05 | ✓ |
| **Nivel 4** — Procesamiento documental o almacenamiento masivo | Documentos en BD (VARBINARY/BLOB), procesos batch masivos, evidencias o archivos por transacción, tablas históricas de alto volumen, ETL/migraciones, snapshots, replicación. Crecimiento acelerado y no lineal; riesgo de saturación. | 1.10 | |

> **Release R16 — Nivel seleccionado: Nivel 3 (FIR = 1.05)**  
> Se introducen nuevas tablas transaccionales (cobros, asociaciones, documentos fiscales, catálogos como `catCobroEstatus`), incremento en operaciones de escritura e índices adicionales. El crecimiento es progresivo y proporcional al volumen de transacciones.

---

### 8.3 Validación de Espacio para Despliegue (Pre-Release Check)

Antes de ejecutar el despliegue en producción se valida que exista espacio libre suficiente para soportar el crecimiento temporal del proceso de liberación.

**Reglas:**
- **Nivel 1–2:** No se requiere validación especial — impacto mínimo.
- **Nivel 3:** Se recomienda ≥ 5 % de espacio libre.
- **Nivel 4:** Se requiere ≥ 10 % de espacio libre obligatorio. En caso contrario: **NO autorizar despliegue**.

**Fórmula:**

```
Espacio Libre (%) = (CTI − CAU) / CTI
```

**Cálculo:**

```
Espacio Libre (%) = (500 − 139) / 500 = 361 / 500 = 72.2 %
```

| CTI | CAU | Espacio Libre (GB) | Espacio Libre (%) | Umbral Nivel 3 | Resultado |
|---|---|---|---|---|---|
| 500 GB | 139 GB | 361 GB | **72.2 %** | ≥ 5 % | ✓ **Con espacio suficiente** |

---

### 8.4 Modelo de Proyección de Almacenamiento

El almacenamiento proyectado se calcula a partir de la capacidad actualmente utilizada, el crecimiento orgánico histórico y el impacto estructural esperado del release.

**Fórmula:**

```
SSD Proyectado = CAU + (COH × HP × FIR)
```

Donde:
- `CAU`: Capacidad Actual Utilizada
- `COH`: Crecimiento Orgánico Histórico mensual
- `HP`: Horizonte de Proyección en meses
- `FIR`: Factor de Incremento por Release

**Cálculo:**

```
SSD Proyectado = 139 + (1 × 3 × 1.05)
               = 139 + 3.15
               = 142.15 GB
               = 142.15 / 500 = 28.43 % del total
```

---

### 8.5 Resultado — Recomendación de Escalado SSD

| Parámetro | Valor Actual | Valor Proyectado (3 meses) | % Utilización Proyectado | Umbral crítico | Recomendación |
|---|---|---|---|---|---|
| **SSD** | 139 GB (27.8 %) | 142.15 GB | **28.43 %** | > 80 % | ✓ **No escalar** |

> El disco SSD del servidor 20.22 cuenta con amplia capacidad disponible. Con un horizonte de 3 meses y el crecimiento proyectado del release R16 (Nivel 3), la utilización alcanzará el 28.43 %, muy por debajo del umbral crítico del 80 %.

---

## 9. Resumen General — Servidor 20.22

| Recurso | Valor Actual | Valor Proyectado | Umbral crítico | Recomendación |
|---|---|---|---|---|
| **CPU** | 15.30 % | 15.85 % | > 75 % | ✓ No escalar |
| **RAM** | 39.85 % | 41.28 % | > 80 % | ✓ No escalar |
| **SSD** | 139 GB (27.8 %) | 142.15 GB (28.43 %) | > 80 % | ✓ No escalar |

> **Conclusión:** El servidor 20.22 tiene capacidad suficiente para soportar el release R16 en los tres recursos evaluados. No se requiere escalado de infraestructura previo al despliegue.

---

## Glosario

| Término | Definición |
|---|---|
| **UCE** | Usuarios Concurrentes Efectivos — fracción de usuarios totales que operan el sistema simultáneamente |
| **TA** | Tasa de Adopción — porcentaje de usuarios concurrentes que utilizarán la nueva funcionalidad |
| **CHU P95** | Consumo Histórico por Usuario en el percentil 95 — base para dimensionar considerando picos operativos |
| **FCR** | Factor de Complejidad del Release — ajuste según el tipo de cambios de funcionalidad introducidos (CPU/RAM) |
| **UCE Proyectado** | UCE ajustado con el crecimiento histórico promedio del sistema |
| **CPU_P95** | Porcentaje de CPU total consumido durante periodos de alta concurrencia (percentil 95) |
| **RAM_P95** | Porcentaje de RAM total consumida durante periodos de alta concurrencia (percentil 95) |
| **CTI** | Capacidad Total Instalada del disco en GB |
| **CAU** | Capacidad Actual Utilizada del disco en GB |
| **COH** | Crecimiento Orgánico Histórico mensual del almacenamiento estructural en GB |
| **HP** | Horizonte de Proyección — periodo en meses a proyectar posterior al release |
| **FIR** | Factor de Incremento por Release — ajuste según el impacto estructural del release sobre almacenamiento |

---

*Generado: 2026-06-18 | Versión 1.1*
