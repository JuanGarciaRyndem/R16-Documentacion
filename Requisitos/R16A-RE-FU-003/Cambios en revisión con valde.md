Se va a generar una propuesta para el requisito 003 favor de generar un archivo 003-Propuesta1.md donde vaya el impacto en base de datos, en back y tareas en el mismo archivo, utilizar el mismo numero de tareas del 003. - 
EndPoint para Subir Archivo y devuelva de Objeto Archivo 
- Generar un EndPoint que reciba un archivo y devuelva un objeto archivo 
- Se debe de mandar el archivo, el bucket, la region - 
- Generar un EndPoint llamado Validar sí el cliente tiene los documentos regulatorios por ahora sólo valida eso en un futuro se pretende que vaya algunas reglas necesarias para no dejar tramitar pedidos 
ArchivoDominio 
-------------------- 
IdArchivoDominio
IdArchivo
IdCatalogoTipoDocumento
IdDominioEntidadArchivo(Guid del Cliente o Guid del Producto o Guid del Proveedor) 
Activo 

catDominioEntidad 
------------------

IdcatDominioEntidad 
Clave 
Descripcion ... 

Ejemplo 
Guid | Cliente | Maneja los archivos del cliente Guid | Proveedor | Maneja los archivos del proveedor Guid | Producto | Maneja los archivos del producto CatalogoTipoDocumento (En vez de IdCatUsoArchivoSistema) 

----
IdCatalogoTipoDocumento 
Clave 
Nombre 
Descripcion 
IdcatDominioEntidad 
FormatoAceptado (Pdf, doc) 
son los mimeTypes (Front nos pasará esa información) 
Orden 
TamanioMaximoKB 

---
