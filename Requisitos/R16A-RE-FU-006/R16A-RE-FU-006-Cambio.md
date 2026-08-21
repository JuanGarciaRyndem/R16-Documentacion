# R16A-RE-FU-006 — Cambios Complementarios

| Campo | Valor |
|---|---|
| **Requisito** | R16A-RE-FU-006 — Referencia de Pago y Código Validador |
| **Fecha** | 2026-08-03 |
| **Versión** | 1.0 |
| **Alcance** | Cambios adicionales al diseño original: columnas en `catMoneda` + endpoint de generación de referencia Banamex |

---

## Contexto

![Pantalla Referencias de Pago — sección Cobros del cliente](R16A-RE-FU-006-ReferenciasPago-UI.png)

La pantalla de **Referencias de Pago** dentro de la sección Cobros del cliente expone tres selectores: **Emisor**, **Banco** y **Cuenta**. Al seleccionar una combinación donde el banco sea Banamex, el sistema debe calcular y mostrar la referencia antes de que el usuario guarde. Adicionalmente, `catMoneda` necesita dos columnas nuevas: una para el mapeo a Legacy y otra que materializa directamente el carácter P/D usado en el segmento 6 de la referencia Banamex.

---

## Cambio 1 — Columnas nuevas en `catMoneda`

> **Nota:** La tabla ya existe con las columnas `IdCatMoneda`, `ClaveMoneda`, `Moneda`, `Operativa`, `Clave` (varchar 150 — uso interno, ej. `'usd'`), `AplicaTipoDeCambio`, `Activo`, `FechaRegistro`, `FechaUltimaActualizacion`. Las columnas `ClaveLegacy` y `ReferenciaCuenta` son nuevas.

### BD — ALTER TABLE

```sql
-- R16A-RE-FU-006 — Columnas nuevas en catMoneda
ALTER TABLE dbo.catMoneda
    ADD ClaveLegacy      varchar(10) NULL,
        ReferenciaCuenta char(1)     NULL;

-- Poblar valores conocidos
UPDATE dbo.catMoneda SET ClaveLegacy = 'M.N.', ReferenciaCuenta = 'P' WHERE ClaveMoneda = 'MXN';
UPDATE dbo.catMoneda SET ClaveLegacy = 'DLS',  ReferenciaCuenta = 'D' WHERE ClaveMoneda = 'USD';
```

### Diccionario de columnas nuevas

| Columna | Tipo | Nulo | Descripción |
|---|---|---|---|
| `ClaveLegacy` | varchar(10) | SÍ | Clave equivalente en el sistema Legacy. MXN → `M.N.`, USD → `DLS` |
| `ReferenciaCuenta` | char(1) | SÍ | Carácter P/D para el segmento 6 de la referencia Banamex. MXN → `P`, USD → `D` |

### Impacto en Back — `ReferenciaBancariaBO`

Con `ReferenciaCuenta` disponible, el segmento 6 se lee directamente del catálogo en lugar de parsear `ClaveMoneda`:

```csharp
// ANTES — segmento 6 inferido desde ClaveMoneda
var seg6 = (moneda?.ClaveMoneda?.StartsWith("M") == true) ? "P" : "D";

// DESPUÉS — segmento 6 leído directamente de catMoneda.ReferenciaCuenta
var seg6 = moneda?.ReferenciaCuenta ?? "D";
```

**Archivo:** `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ReferenciaBancariaBO.cs`

---

## Cambio 2 — Endpoint `GenerarReferenciaBancaria`

### Descripción

Endpoint para calcular y devolver la referencia Banamex de 7 segmentos **sin persistirla**. Se invoca únicamente cuando la UI detecta que el banco seleccionado tiene `catBanco.RequiereCodigoValidador = true`. Permite mostrar la referencia como preview antes de que el usuario confirme. La lógica reside en `ClienteDatosBancariosBO.Extensions`; el controller es delgado.

### Request / Response

```
POST /ClienteDatosBancarios/GenerarReferenciaBancaria

Body:
{
    "idCliente":               "{guid}",
    "idEmpresa":               "{guid}",
    "idBanco":                 "{guid}",
    "idEmpresaDatosBancarios": "{guid}",
    "codigoValidador":         "07"
}

Response 200 OK:
{
    "referencia": "QUI2345002P07"
}
-- Segmentos: Q·U·I (letras 1-3 del nombre, sin espacios) + 2345 (últimos 4 del ID Legacy) + 002 (código banco) + P (MXN) + 07 (CodigoValidador — ejemplo; el campo real es alfanumérico, máximo 3 caracteres, DUDA-015)

Response 400 Bad Request:
{ "message": "El banco indicado no requiere Código Validador." }
```

### Parámetros

| Parámetro | Tipo | Descripción |
|---|---|---|
| `idCliente` | Guid | Cliente al que se asigna la cuenta |
| `idEmpresa` | Guid | Empresa del grupo PROQUIFA que factura — `Empresa.IdEmpresa` |
| `idBanco` | Guid | Banco seleccionado — `catBanco.IdCatBanco`, debe tener `RequiereCodigoValidador = true` |
| `idEmpresaDatosBancarios` | Guid | Cuenta seleccionada — `EmpresaDatosBancarios.IdEmpresaDatosBancarios` |
| `codigoValidador` | string(3) | Código Validador capturado por el usuario. ~~**Numérico, siempre 2 dígitos con cero a la izquierda** (rango `01`–`99`).~~ — **Corregido 2026-08-21 (DUDA-015):** el campo es **alfanumérico, máximo 3 caracteres**, sin acentos ni espacios en blanco (no está limitado a numérico ni a 2 dígitos). Se incluye en el segmento 7 de la referencia. Puede enviarse vacío (`""`) para obtener un preview parcial antes de capturarlo. |

### Implementación — Controller (delgado)

**Archivo:** `WebApi.Catalogos\Controllers\Configuracion\Clientes\Relaciones\ClienteDatosBancariosController.cs`

```csharp
/// <summary>Genera un preview de la referencia Banamex antes de capturar el Código Validador.</summary>
[HttpPost]
[Route("ClienteDatosBancarios/GenerarReferenciaBancaria")]
[ResponseType(typeof(string))]
public HttpResponseMessage GenerarReferenciaBancaria([FromBody] GenerarReferenciaBancariaRequest request)
{
    return TryExecute(() =>
    {
        var bo = new ClienteDatosBancariosBO();
        var referencia = bo.GenerarReferenciaBancaria(
            request.IdCliente,
            request.IdEmpresa,
            request.IdBanco,
            request.IdEmpresaDatosBancarios,
            request.CodigoValidador ?? string.Empty);

        return bo.Response.Status
            ? Request.CreateResponse(HttpStatusCode.OK, referencia)
            : Request.CreateResponse(HttpStatusCode.BadRequest, bo.Response.FMessage);
    });
}

public class GenerarReferenciaBancariaRequest
{
    public Guid   IdCliente               { get; set; }
    public Guid   IdEmpresa               { get; set; }
    public Guid   IdBanco                 { get; set; }
    public Guid   IdEmpresaDatosBancarios { get; set; }
    public string CodigoValidador         { get; set; }
}
```

### Implementación — Lógica en Extensions

**Archivo:** `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ClienteDatosBancariosBO.Extensions.cs`

```csharp
public string GenerarReferenciaBancaria(Guid idCliente, Guid idEmpresa, Guid idBanco, Guid idEmpresaDatosBancarios, string codigoValidador)
{
    var cuenta = new EmpresaDatosBancariosBO().Obtener(idEmpresaDatosBancarios);
    if (cuenta == null || !cuenta.Activo)
    {
        Model.AddMessage("IdEmpresaDatosBancarios", "La cuenta bancaria seleccionada no existe o no está activa.");
        Response = new Response(false, new FMessage(EMessageType.Validation.ToString(), Model.Messages.ModelState));
        return null;
    }

    var datosBancarios = new DatosBancariosBO().Obtener(cuenta.IdDatosBancarios);
    var banco = datosBancarios.IdCatBanco.HasValue
        ? new TablaGenericaBO<catBanco>().Obtener(datosBancarios.IdCatBanco.Value)
        : null;

    if (banco == null || !banco.RequiereCodigoValidador)
    {
        Model.AddMessage("IdBanco", "El banco indicado no requiere Código Validador.");
        Response = new Response(false, new FMessage(EMessageType.Validation.ToString(), Model.Messages.ModelState));
        return null;
    }

    var moneda = datosBancarios.IdCatMoneda.HasValue
        ? new TablaGenericaBO<catMoneda>().Obtener(datosBancarios.IdCatMoneda.Value)
        : null;

    var cliente     = new ClienteBO().Obtener(idCliente);
    var razonSocial = new DatosFacturacionClienteBO().ObtenerPorCliente(idCliente)?.RazonSocial;

    int? clienteLegacy;
    using (var db = new ProquifaDotNetEntities())
    {
        clienteLegacy = db.spObtenerClienteLegacyId(idCliente).SingleOrDefault()?.ClienteLegacy;
        if (clienteLegacy == null)
            Logger.WarnFormat("ClienteLegacy no encontrado para IdCliente {0} — S4 será '0000'", idCliente);
    }

    return _bankReferenceBO.Build(cliente.Nombre, razonSocial, clienteLegacy, banco, moneda, codigoValidador);
}
```

---

## Cambio 3 — Corrección de bug en segmentos 1-3 (BUG-001)

### Descripción del bug

En el algoritmo de creación de la referencia Banamex (Regla 7), los segmentos 1, 2 y 3 se definían como la primera, segunda y tercera letra del nombre del cliente tomando la posición literal del carácter. Esto produce un resultado incorrecto cuando el nombre del cliente tiene una palabra de 2 caracteres al inicio:

| Nombre | Resultado incorrecto | Resultado correcto |
|---|---|---|
| `BP Farmaceutica` | `BP ` (con espacio) | `BPF` |
| `GE Healthcare` | `GE ` (con espacio) | `GEH` |

Adicionalmente, si el nombre del cliente tiene **menos de 3 caracteres sin espacios**, los segmentos faltantes se rellenan con `"X"`:

| Nombre | Caracteres sin espacios | Segmentos 1-3 |
|---|---|---|
| `GP` | `GP` (2) | `GPX` |
| `A` | `A` (1) | `AXX` |

### Corrección

Los espacios deben ignorarse al extraer los 3 primeros caracteres. Se toma el nombre, se eliminan los espacios y se extraen los primeros 3 caracteres del resultado.

### Impacto en Back — `ReferenciaBancariaBO`

**Archivo:** `Logic.Pqf.Catalogos\Clientes\DatosBancarios\ReferenciaBancariaBO.cs`

```csharp
// ANTES — toma posición literal, incluye espacios
var seg1 = nombre.Length > 0 ? nombre[0].ToString() : "X";
var seg2 = nombre.Length > 1 ? nombre[1].ToString() : "X";
var seg3 = nombre.Length > 2 ? nombre[2].ToString() : "X";

// DESPUÉS — ignora espacios antes de extraer (BUG-001)
// Si el nombre sin espacios tiene menos de 3 chars, los faltantes se rellenan con "X"
// Ejemplos: "BP Farmaceutica" → "BPF" | "GP" → "GPX" | "A" → "AXX"
var letras = nombre.Replace(" ", string.Empty);
var seg1 = letras.Length > 0 ? letras[0].ToString() : "X";
var seg2 = letras.Length > 1 ? letras[1].ToString() : "X";
var seg3 = letras.Length > 2 ? letras[2].ToString() : "X";
```

---

## Pendientes

| #   | Pendiente                                                                                                                                                                   | Responsable |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| P1  | Validar que todos los registros de `catMoneda` distintos de MXN/USD deben quedar con `ReferenciaCuenta = 'D'` como comportamiento esperado. | Desarrollo  |
| P2  | ~~Confirmar formato de `codigoValidador` (numérico vs. alfanumérico, longitud).~~ **Cerrado 2026-08-21 (DUDA-015):** alfanumérico, máximo 3 caracteres, sin acentos ni espacios. Se corrige la descripción del parámetro y el comentario del ejemplo en este documento (antes decían "numérico, 2 dígitos, 01–99"). | Cerrado |

---

## Criterios de Aceptación

- [ ] `catMoneda` tiene las columnas `ClaveLegacy` y `ReferenciaCuenta` con los valores MXN → `M.N.`/`P` y USD → `DLS`/`D` poblados.
- [ ] `ReferenciaBancariaBO` usa `catMoneda.ReferenciaCuenta` para el segmento 6; no parsea `ClaveMoneda`.
- [ ] `POST /ClienteDatosBancarios/GenerarReferenciaBancaria` retorna la referencia de 7 segmentos correcta cuando el banco es Banamex, incluyendo el `CodigoValidador` recibido en el segmento 7.
- [ ] El endpoint acepta `CodigoValidador` vacío (`""`) y genera la referencia parcial para preview.
- [ ] El endpoint retorna 400 si el banco no tiene la bandera Banamex activa.
- [ ] El endpoint no persiste ningún dato — solo calcula y devuelve la referencia.
- [ ] Los segmentos 1-3 de la referencia Banamex ignoran espacios: "BP Farmaceutica" produce `BPF`, no `BP ` (BUG-001).
