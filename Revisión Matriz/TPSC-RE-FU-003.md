# TPSC-RE-FU-003

**Estatus:** ✅ Atendido

---

## Observaciones

- **(Requisito)** La visualización del PDF cargado se realiza abriendo el archivo en una pestaña nueva del navegador — esto va en criterios. Se debe revisar la viabilidad técnica, ya que algunos navegadores por defecto (o según configuración del usuario) descargan el archivo en lugar de abrirlo en una nueva pestaña.
- **(Requisito)** "El sistema almacena únicamente la versión vigente de cada documento (sin historial de versiones)" — esto va en criterios. Actualmente no se eliminan archivos de MinIO; el usuario elimina el archivo en registros pero se mantiene en MinIO. Especificar si se mantendrá el mismo mecanismo o si se considerará limpiar los archivos anteriores del almacenamiento MinIO.
- El Criterio 8 debe moverse a otro requisito, ya que la carga de archivos y la habilitación condicionada a esos documentos son funcionalidades separadas.

## Notas adicionales

> Sin notas adicionales.

## Resumen de cambios aplicados

- Frase sobre apertura en pestaña nueva eliminada del Requisito y reformulada agnósticamente en **Criterio C1**: el sistema entrega el archivo al navegador y el comportamiento de apertura depende de la configuración del navegador y del usuario.
- Frase sobre almacenamiento sin historial eliminada del Requisito y reformulada en **Regla 3 + Observaciones**: en pantalla solo se muestra la versión vigente; los archivos físicos en MinIO se mantienen sin purga automática (se conserva el mecanismo actual del backend).
- **Criterio 8** (habilitación de pretramitación) eliminado de esta fila — pertenece al módulo Pretramitar Pedido (**TPSC-RE-FU-009**).
- Reglas reescritas como enunciados declarativos: de 7 reglas *Cómo* pasaron a 5 reglas declarativas del *Qué*.
- Criterios organizados en 5 secciones: **A** (Visibilidad y acceso), **B** (Carga y formato), **C** (Visualización), **D** (Reemplazo y eliminación), **E** (Alcance de validación).
