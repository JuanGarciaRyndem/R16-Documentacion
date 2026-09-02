
---

## 9. Código de referencia (.NET 10)

Este código ilustra el diseño de la sección 5 con clases reales, siguiendo los estándares de
construcción de Ryndem (`ryndem-dotnet`: Domain sin dependencias externas, `Result`/`Result<T>` para
fallos esperados, catálogo de errores centralizado). Vive en la capa **Domain** — por eso está en
español, igual que el esquema de BD y el resto de este documento — y es la pieza central del diseño:
el resto de la implementación (DTOs, Entity Framework, el controller HTTP, las pruebas) es boilerplate
de la arquitectura, no parte específica de esta propuesta, así que no se incluye aquí para no diluir lo
importante. La solución de referencia completa, con todas las capas, quedó guardada en
`Codigo/Addenda-BackendCode.zip` dentro de esta misma carpeta, por si sirve como punto de partida al
integrarlo al repositorio real de Finanzas — **no se compiló en el entorno donde se generó** (sin SDK
de .NET disponible), así que antes de usarla hay que correr `dotnet build`/`dotnet test` normalmente.

### 9.1 Entidades (mapean 1:1 las tablas de la sección 5 del documento de BD)

```csharp
namespace Addenda.Domain.Entities;

/// <summary>
/// Catálogo de formatos de addenda soportados (<c>catTipoAddenda</c>). Identifica el formato de
/// un cliente y si tiene nivel de detalle por partida o requiere correo de contacto — lo que le
/// permite a pqf y al motor de ensamblado decidir sin adivinar.
/// </summary>
public class TipoAddenda
{
    public Guid IdCatTipoAddenda { get; set; }

    /// <summary>Clave corta del formato: MAVI, PFIZER, ASOFARMA, SANOFI.</summary>
    public required string Clave { get; set; }

    public required string Descripcion { get; set; }

    /// <summary>Si el formato repite un bloque de detalle una vez por partida.</summary>
    public bool TieneDetallePorPartida { get; set; }

    /// <summary>Si el formato exige un correo de contacto a nivel de pedido (con default).</summary>
    public bool RequiereCorreoContacto { get; set; }

    public bool Activo { get; set; } = true;
}
```

```csharp
namespace Addenda.Domain.Entities;

/// <summary>
/// Valor genérico de addenda a nivel pedido (<c>fccAddenda</c>). Un pedido puede tener varias
/// filas — una por <see cref="Clave"/> — porque el modelo guarda pares Clave/Valor atómicos y
/// agnósticos al cliente en lugar de columnas específicas.
/// </summary>
public class AddendaCabecera
{
    public Guid IdFccAddenda { get; set; }

    /// <summary>El pedido origen (tpPedido) al que pertenece este valor.</summary>
    public required Guid IdTPPedido { get; set; }

    public required Guid IdCatTipoAddenda { get; set; }

    /// <summary>Nombre del campo tal como lo espera la plantilla del cliente (p. ej. "CorreoContacto").</summary>
    public required string Clave { get; set; }

    /// <summary>El valor de ese campo, ya listo para insertarse en el XML.</summary>
    public required string Valor { get; set; }

    public DateTime? FechaGeneracion { get; set; }

    public DateTime FechaRegistro { get; set; }

    public DateTime FechaUltimaActualizacion { get; set; }
}
```

```csharp
namespace Addenda.Domain.Entities;

/// <summary>
/// Valor genérico de addenda a nivel partida (<c>fccAddendaPartida</c>). No depende de
/// <see cref="AddendaCabecera"/> — está anclada directo a la partida y al catálogo, porque una
/// partida puede existir sin que haya valores de cabecera para ese pedido (ver Mavi, que no usa
/// esta tabla en absoluto por no tener nivel de detalle).
/// </summary>
public class AddendaDetalle
{
    public Guid IdFccAddendaPartida { get; set; }

    /// <summary>La partida origen (tpPartidaPedido) a la que pertenece este valor.</summary>
    public required Guid IdTPPartidaPedido { get; set; }

    public required Guid IdCatTipoAddenda { get; set; }

    /// <summary>Nombre del campo tal como lo espera la plantilla del cliente (p. ej. "LineaDeOrden").</summary>
    public required string Clave { get; set; }

    /// <summary>El valor de ese campo para esa partida.</summary>
    public required string Valor { get; set; }

    public DateTime FechaRegistro { get; set; }

    public DateTime FechaUltimaActualizacion { get; set; }
}
```

### 9.2 Modelo de plantillas — lo que hace que agregar un cliente sea configuración, no código

```csharp
namespace Addenda.Domain.Templates;

/// <summary>
/// De dónde sale el valor de un campo de la plantilla de un cliente. Es la clasificación central
/// del análisis: la mayoría de los campos que exigen los formatos de addenda son <see cref="Fijo"/>
/// o <see cref="Derivado"/> — muy pocos son <see cref="Genuino"/> (datos que solo existen para la
/// addenda). Ver secciones 2 a 4 del documento de BD.
/// </summary>
public enum OrigenCampoAddenda
{
    /// <summary>Constante propia del formato del cliente. No cambia entre transacciones.</summary>
    Fijo,

    /// <summary>
    /// Ya existe en los datos de la factura o de la partida (fccFactura/fccFacturaPartida). El
    /// motor lo toma del contexto que se le entrega, no de una tabla de addenda.
    /// </summary>
    Derivado,

    /// <summary>
    /// No existe en ningún otro lado: requiere una fila en fccAddenda o fccAddendaPartida.
    /// Puede tener un valor por default para cuando no se capturó.
    /// </summary>
    Genuino,
}
```

```csharp
namespace Addenda.Domain.Templates;

/// <summary>
/// Cómo se serializa un campo dentro de su nodo XML. Pfizer y Asofarma usan atributos; Mavi y
/// Sanofi usan elementos hijos — el motor debe soportar ambas formas sin código distinto por
/// cliente.
/// </summary>
public enum FormaSerializacion
{
    Atributo,
    Elemento,
}
```

```csharp
namespace Addenda.Domain.Templates;

/// <summary>
/// Definición declarativa de un campo dentro de un nodo de la plantilla de un cliente. El motor de
/// ensamblado interpreta esta definición sin conocer al cliente concreto — la única parte
/// específica de cada cliente es la data, no el código.
/// </summary>
/// <param name="NombreCampo">Nombre del atributo o elemento en el XML final (p. ej. "LINE_NO").</param>
/// <param name="Origen">De dónde sale el valor.</param>
/// <param name="FormaSerializacion">Si se serializa como atributo o como elemento hijo.</param>
/// <param name="ValorFijo">Requerido cuando <paramref name="Origen"/> es <see cref="OrigenCampoAddenda.Fijo"/>.</param>
/// <param name="NombreOrigen">
/// Requerido cuando <paramref name="Origen"/> es <see cref="OrigenCampoAddenda.Derivado"/>: la clave
/// del campo dentro del contexto de factura o de partida que entrega la capa de aplicación.
/// </param>
/// <param name="ClaveGenuina">
/// Solo para <see cref="OrigenCampoAddenda.Genuino"/>: la Clave a buscar en fccAddenda/fccAddendaPartida.
/// Si es null, se usa <paramref name="NombreCampo"/> como Clave.
/// </param>
/// <param name="ValorPorDefecto">
/// Solo para <see cref="OrigenCampoAddenda.Genuino"/>: valor a usar si no se capturó. Si es null y el
/// valor no existe, el campo es obligatorio y su ausencia hace fallar el ensamblado.
/// </param>
/// <param name="Regla">
/// Mapeo condicional opcional aplicado sobre el valor ya resuelto (p. ej. tasa de IVA → TAX_CODE).
/// La clave <c>"*"</c> es el valor por default del mapeo cuando ningún otro caso coincide.
/// </param>
public sealed record AddendaCampoDefinicion(
    string NombreCampo,
    OrigenCampoAddenda Origen,
    FormaSerializacion FormaSerializacion,
    string? ValorFijo = null,
    string? NombreOrigen = null,
    string? ClaveGenuina = null,
    string? ValorPorDefecto = null,
    IReadOnlyDictionary<string, string>? Regla = null)
{
    public static AddendaCampoDefinicion Fijo(string nombreCampo, string valor, FormaSerializacion forma) =>
        new(nombreCampo, OrigenCampoAddenda.Fijo, forma, ValorFijo: valor);

    public static AddendaCampoDefinicion Derivado(
        string nombreCampo,
        string nombreOrigen,
        FormaSerializacion forma,
        IReadOnlyDictionary<string, string>? regla = null) =>
        new(nombreCampo, OrigenCampoAddenda.Derivado, forma, NombreOrigen: nombreOrigen, Regla: regla);

    public static AddendaCampoDefinicion Genuino(
        string nombreCampo,
        FormaSerializacion forma,
        string? claveGenuina = null,
        string? valorPorDefecto = null) =>
        new(nombreCampo, OrigenCampoAddenda.Genuino, forma, ClaveGenuina: claveGenuina, ValorPorDefecto: valorPorDefecto);
}
```

```csharp
namespace Addenda.Domain.Templates;

/// <summary>
/// Definición declarativa de un nodo XML dentro de la plantilla de un cliente. Un árbol de estos
/// nodos describe formatos tan distintos como Mavi (un solo nodo de cabecera, sin detalle) y Sanofi
/// (un nodo con namespace, un hijo de cabecera y un hijo que se repite por partida) con el mismo
/// modelo — la variación es de datos, no de código.
/// </summary>
/// <param name="Nombre">Nombre del nodo XML, con prefijo de namespace si aplica (p. ej. "sanofi:header").</param>
/// <param name="Campos">Campos propios de este nodo.</param>
/// <param name="Hijos">Nodos hijos, en el orden en que deben aparecer en el XML.</param>
/// <param name="AtributosFijosDeNodo">
/// Atributos siempre constantes de este nodo (típicamente declaraciones de namespace).
/// </param>
/// <param name="SeRepitePorPartida">
/// Si es verdadero, este nodo se genera una vez por cada partida de la factura, usando el contexto
/// de esa partida en lugar del contexto de cabecera para resolver sus campos.
/// </param>
public sealed record AddendaNodoDefinicion(
    string Nombre,
    IReadOnlyList<AddendaCampoDefinicion> Campos,
    IReadOnlyList<AddendaNodoDefinicion>? Hijos = null,
    IReadOnlyDictionary<string, string>? AtributosFijosDeNodo = null,
    bool SeRepitePorPartida = false);
```

```csharp
namespace Addenda.Domain.Templates;

/// <summary>
/// Plantilla completa de un formato de addenda: qué forma tiene el nodo raíz que va dentro de
/// <c>cfdi:Addenda</c> para un cliente dado. Corresponde una a una con una fila de
/// <c>catTipoAddenda</c>.
/// </summary>
/// <param name="ClaveTipoAddenda">La Clave del catálogo (MAVI, PFIZER, ASOFARMA, SANOFI).</param>
/// <param name="NodoRaiz">El nodo que se inserta directo bajo <c>cfdi:Addenda</c>.</param>
public sealed record AddendaTemplateDefinition(string ClaveTipoAddenda, AddendaNodoDefinicion NodoRaiz);
```

### 9.3 El motor de ensamblado

El contexto que arma la capa de aplicación a partir de la factura real (`fccFactura`/`fccFacturaPartida`)
y de los valores genuinos ya capturados (`fccAddenda`/`fccAddendaPartida`):

```csharp
namespace Addenda.Domain.Assembly;

/// <param name="IdTPPedido">El pedido facturado.</param>
/// <param name="CamposDerivados">
/// Valores de cabecera que ya existen en la factura (moneda, folio, serie, total, IVA, RFC
/// emisor...), indexados por el <c>NombreOrigen</c> que declara la plantilla.
/// </param>
/// <param name="ValoresGenuinos">
/// Valores de cabecera capturados específicamente para addenda en <c>fccAddenda</c> (p. ej.
/// "CorreoContacto"), indexados por <c>Clave</c>.
/// </param>
/// <param name="Partidas">El contexto de cada partida facturada, en el orden en que deben repetirse.</param>
public sealed record FacturaAddendaContexto(
    Guid IdTPPedido,
    IReadOnlyDictionary<string, string> CamposDerivados,
    IReadOnlyDictionary<string, string> ValoresGenuinos,
    IReadOnlyList<PartidaAddendaContexto> Partidas);

/// <param name="IdTPPartidaPedido">La partida origen.</param>
/// <param name="CamposDerivados">
/// Valores que ya existen en la partida facturada (importe, cantidad, unidad de medida, tasa de
/// IVA...), indexados por el <c>NombreOrigen</c> que declara la plantilla.
/// </param>
/// <param name="ValoresGenuinos">
/// Valores capturados específicamente para addenda en <c>fccAddendaPartida</c> para esta partida
/// (p. ej. "LineaDeOrden"), indexados por <c>Clave</c>.
/// </param>
public sealed record PartidaAddendaContexto(
    Guid IdTPPartidaPedido,
    IReadOnlyDictionary<string, string> CamposDerivados,
    IReadOnlyDictionary<string, string> ValoresGenuinos);
```

El árbol neutral que produce el motor (sin ninguna dependencia de `System.Xml`: eso lo resuelve un
serializador aparte, del lado de infraestructura):

```csharp
namespace Addenda.Domain.Assembly;

public sealed class AddendaXmlNodo
{
    public required string Nombre { get; init; }

    /// <summary>Contenido de texto del nodo, cuando el campo se serializa como elemento hoja.</summary>
    public string? Valor { get; init; }

    public IReadOnlyDictionary<string, string> Atributos { get; init; } = new Dictionary<string, string>();

    public IReadOnlyList<AddendaXmlNodo> Hijos { get; init; } = [];
}
```

El motor sigue el patrón `Result`/`Result<T>` de Ryndem: un flujo esperado (falta un dato
obligatorio, una regla sin mapeo) nunca es una excepción, es un `Result` fallido con un código de un
catálogo centralizado. Los 4 códigos que puede producir el motor:

```csharp
namespace Addenda.Domain.Errors;

public static class AddendaErrors
{
    public static class Cabecera
    {
        public static readonly ErrorDefinition FaltaValorObligatorio = new(
            Code: "ADD-003",
            Title: "Required addenda value is missing",
            Description: "La plantilla del cliente exige un valor de cabecera que no tiene default y no fue capturado.",
            Category: ErrorCategory.Business, Severity: ErrorSeverity.Medium, HttpStatusCode: 422);
    }

    public static class Partida
    {
        public static readonly ErrorDefinition FaltaValorObligatorio = new(
            Code: "ADD-004",
            Title: "Required addenda value is missing for a line item",
            Description: "La plantilla del cliente exige un valor de detalle que no tiene default y no fue capturado para una partida.",
            Category: ErrorCategory.Business, Severity: ErrorSeverity.Medium, HttpStatusCode: 422);
    }

    public static class Ensamblado
    {
        public static readonly ErrorDefinition DatosDeFacturaNoEncontrados = new(
            Code: "ADD-005",
            Title: "Invoice data not found for addenda assembly",
            Description: "No se encontraron los datos de factura/partida necesarios para ensamblar la addenda del pedido indicado.",
            Category: ErrorCategory.Resource, Severity: ErrorSeverity.Low, HttpStatusCode: 404);

        public static readonly ErrorDefinition ValorSinMapeoEnRegla = new(
            Code: "ADD-006",
            Title: "Addenda template rule has no mapping for the resolved value",
            Description: "La regla condicional de un campo de la plantilla no define un mapeo para el valor resuelto, ni tiene un valor por default (\"*\").",
            Category: ErrorCategory.Business, Severity: ErrorSeverity.Medium, HttpStatusCode: 422);
    }

    // Catálogo completo (incluye VAL-001, ADD-001, ADD-002, INF-001, UNX-001) en el código de referencia.
}
```

Y el motor mismo — la única pieza que sabe recorrer un árbol de plantilla, sin conocer a ningún
cliente en particular:

```csharp
using Addenda.Domain.Common;
using Addenda.Domain.Errors;
using Addenda.Domain.Templates;

namespace Addenda.Domain.Assembly;

/// <summary>
/// El motor genérico de ensamblado de addenda. Interpreta una <see cref="AddendaTemplateDefinition"/>
/// sin conocer al cliente concreto: resuelve cada campo según su origen (fijo, derivado o genuino),
/// repite el nodo de detalle una vez por partida y produce un árbol neutral
/// (<see cref="AddendaXmlNodo"/>) listo para que la infraestructura lo serialice.
/// </summary>
public static class AddendaAssembler
{
    public static Result<AddendaXmlNodo> Ensamblar(AddendaTemplateDefinition plantilla, FacturaAddendaContexto contexto) =>
        ResolverNodo(plantilla.NodoRaiz, contexto, partidaActual: null);

    /// <summary>
    /// Validación previa al timbrado (tarea BE-7): recorre la plantilla igual que
    /// <see cref="Ensamblar"/>, pero descarta el árbol resultante. Falla con el mismo código que
    /// fallaría el ensamblado real, para identificar el dato faltante antes de generar el XML.
    /// </summary>
    public static Result Validar(AddendaTemplateDefinition plantilla, FacturaAddendaContexto contexto)
    {
        var resultado = Ensamblar(plantilla, contexto);
        return resultado.IsSuccess ? Result.Ok() : Result.Fail(resultado.Error);
    }

    private static Result<AddendaXmlNodo> ResolverNodo(
        AddendaNodoDefinicion nodoDef,
        FacturaAddendaContexto factura,
        PartidaAddendaContexto? partidaActual)
    {
        var atributos = nodoDef.AtributosFijosDeNodo is null
            ? new Dictionary<string, string>()
            : new Dictionary<string, string>(nodoDef.AtributosFijosDeNodo);

        var hijosResultantes = new List<AddendaXmlNodo>();

        foreach (var campo in nodoDef.Campos)
        {
            var valorResuelto = ResolverCampo(campo, factura, partidaActual);
            if (valorResuelto.IsFailure)
            {
                return Result.Fail<AddendaXmlNodo>(valorResuelto.Error);
            }

            if (campo.FormaSerializacion == FormaSerializacion.Atributo)
            {
                atributos[campo.NombreCampo] = valorResuelto.Value;
            }
            else
            {
                hijosResultantes.Add(new AddendaXmlNodo { Nombre = campo.NombreCampo, Valor = valorResuelto.Value });
            }
        }

        foreach (var hijoDef in nodoDef.Hijos ?? [])
        {
            if (hijoDef.SeRepitePorPartida)
            {
                foreach (var partida in factura.Partidas)
                {
                    var resultadoHijo = ResolverNodo(hijoDef, factura, partida);
                    if (resultadoHijo.IsFailure)
                    {
                        return Result.Fail<AddendaXmlNodo>(resultadoHijo.Error);
                    }

                    hijosResultantes.Add(resultadoHijo.Value);
                }
            }
            else
            {
                var resultadoHijo = ResolverNodo(hijoDef, factura, partidaActual);
                if (resultadoHijo.IsFailure)
                {
                    return Result.Fail<AddendaXmlNodo>(resultadoHijo.Error);
                }

                hijosResultantes.Add(resultadoHijo.Value);
            }
        }

        return Result.Ok(new AddendaXmlNodo
        {
            Nombre = nodoDef.Nombre,
            Atributos = atributos,
            Hijos = hijosResultantes,
        });
    }

    private static Result<string> ResolverCampo(
        AddendaCampoDefinicion campo,
        FacturaAddendaContexto factura,
        PartidaAddendaContexto? partidaActual)
    {
        string valorBase;

        switch (campo.Origen)
        {
            case OrigenCampoAddenda.Fijo:
                valorBase = campo.ValorFijo
                    ?? throw new InvalidOperationException(
                        $"El campo '{campo.NombreCampo}' está declarado como Fijo pero no tiene ValorFijo.");
                break;

            case OrigenCampoAddenda.Derivado:
                var camposDerivados = partidaActual?.CamposDerivados ?? factura.CamposDerivados;
                if (campo.NombreOrigen is null || !camposDerivados.TryGetValue(campo.NombreOrigen, out var derivado))
                {
                    return Result.Fail<string>(AddendaErrors.Ensamblado.DatosDeFacturaNoEncontrados);
                }

                valorBase = derivado;
                break;

            case OrigenCampoAddenda.Genuino:
                var claveGenuina = campo.ClaveGenuina ?? campo.NombreCampo;
                var valoresGenuinos = partidaActual?.ValoresGenuinos ?? factura.ValoresGenuinos;
                if (valoresGenuinos.TryGetValue(claveGenuina, out var capturado))
                {
                    valorBase = capturado;
                }
                else if (campo.ValorPorDefecto is not null)
                {
                    valorBase = campo.ValorPorDefecto;
                }
                else
                {
                    // Es exactamente el hallazgo obligatorio-sin-default que valida BE-7: falta un
                    // dato genuino que la plantilla del cliente exige y que nadie capturó.
                    var errorFaltante = partidaActual is not null
                        ? AddendaErrors.Partida.FaltaValorObligatorio
                        : AddendaErrors.Cabecera.FaltaValorObligatorio;
                    return Result.Fail<string>(errorFaltante);
                }

                break;

            default:
                throw new ArgumentOutOfRangeException(nameof(campo), campo.Origen, "Origen de campo de addenda no reconocido.");
        }

        if (campo.Regla is not null)
        {
            if (campo.Regla.TryGetValue(valorBase, out var mapeado))
            {
                return Result.Ok(mapeado);
            }

            if (campo.Regla.TryGetValue("*", out var porDefecto))
            {
                return Result.Ok(porDefecto);
            }

            return Result.Fail<string>(AddendaErrors.Ensamblado.ValorSinMapeoEnRegla);
        }

        return Result.Ok(valorBase);
    }
}
```

### 9.4 Dos plantillas reales de ejemplo

Cada cliente es una definición de datos, no una clase de lógica — así se ven las 2 más distintas
entre sí: la más simple (Mavi) y la más completa (Sanofi, con namespace propio y detalle repetido).

**Mavi** — cabecera pura, sin detalle por partida, cero campos genuinos (ver sección 3.1 del
documento de BD):

```csharp
using Addenda.Domain.Assembly;
using Addenda.Domain.Templates;

namespace Addenda.Infrastructure.Templates.Definitions;

internal static class MaviAddendaTemplateFactory
{
    private const string Clave = "MAVI";

    public static AddendaTemplateDefinition Crear()
    {
        var nodoRaiz = new AddendaNodoDefinicion(
            Nombre: "Mavi",
            Campos:
            [
                AddendaCampoDefinicion.Derivado("RfcProveedor", ClavesDeCamposDerivados.Factura.RfcEmisor, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("NumProveedor", "704", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("FechaFacturacion", ClavesDeCamposDerivados.Factura.FechaFacturacion, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("NumPedido", ClavesDeCamposDerivados.Factura.NumeroOrdenCompraCliente, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("CodMoneda", ClavesDeCamposDerivados.Factura.Moneda, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("MontoTotal", ClavesDeCamposDerivados.Factura.Total, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("IVA", ClavesDeCamposDerivados.Factura.MontoIva, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("PorcentajeIVA", ClavesDeCamposDerivados.Factura.TasaIva, FormaSerializacion.Elemento),
                // NumFactura = serie concatenada con folio (p. ej. "A148237"): ya viaja armado en
                // Factura.NumeroFactura — el motor no concatena, así que ese valor debe llegar
                // ya formado desde Application.
                AddendaCampoDefinicion.Derivado("NumFactura", ClavesDeCamposDerivados.Factura.NumeroFactura, FormaSerializacion.Elemento),
                // Serie: "A" para Ingreso ("I"), "P" para Pago ("P") — regla fija de 2 valores
                // tomada literal del PDF. "*" cubre cualquier otro tipo con el default del legacy.
                AddendaCampoDefinicion.Derivado(
                    "Serie",
                    ClavesDeCamposDerivados.Factura.TipoComprobante,
                    FormaSerializacion.Elemento,
                    regla: new Dictionary<string, string> { ["I"] = "A", ["P"] = "P", ["*"] = "A" }),
                AddendaCampoDefinicion.Derivado("Folio", ClavesDeCamposDerivados.Factura.Folio, FormaSerializacion.Elemento),
            ]);

        return new AddendaTemplateDefinition(Clave, nodoRaiz);
    }
}
```

**Sanofi / Sanofi Pasteur / Azteca Vacunas** — namespace propio, cabecera + detalle repetido por
partida, y los 2 únicos campos genuinos de todo el análisis (`CorreoContacto`, `LineaDeOrden`) más el
pendiente de `CUENTA_PUENTE` (ver secciones 3.4, 4 y 6 del documento de BD):

```csharp
using Addenda.Domain.Assembly;
using Addenda.Domain.Templates;

namespace Addenda.Infrastructure.Templates.Definitions;

/// <remarks>
/// Recibe <paramref name="clave"/>/<paramref name="numeroProveedor"/>/<paramref name="correoContactoPorDefecto"/>
/// en vez de darlos por fijos para los 3 clientes: el doc de BD (pendiente #2) deja abierto si
/// Sanofi Pasteur y Azteca Vacunas usan un NUM_PROVEEDOR o correo default distintos.
/// </remarks>
internal static class SanofiAddendaTemplateFactory
{
    private const string NamespaceUri = "https://mexico.sanofi.com/schemas";
    private const string XsiNamespaceUri = "http://www.w3.org/2001/XMLSchema-instance";

    public static AddendaTemplateDefinition Crear(string clave, string numeroProveedor, string correoContactoPorDefecto)
    {
        var header = new AddendaNodoDefinicion(
            Nombre: "sanofi:header",
            Campos:
            [
                AddendaCampoDefinicion.Fijo("sanofi:TIPO_DOCTO", "01", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("sanofi:NUM_ORDEN", ClavesDeCamposDerivados.Factura.NumeroOrdenCompraCliente, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:NUM_PROVEEDOR", numeroProveedor, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:FCTCONV", "1.000", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("sanofi:MONEDA", ClavesDeCamposDerivados.Factura.Moneda, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:CTA_CORREO", "credito@proquifa.com.mx", FormaSerializacion.Elemento),
                // El PDF exige el nodo siempre presente y vacío/autocerrado — cadena vacía, nunca null.
                AddendaCampoDefinicion.Fijo("sanofi:IMP_RETENCION", string.Empty, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("sanofi:IMP_TOTAL", ClavesDeCamposDerivados.Factura.Total, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_1", "0.00", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_2", "0.00", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_3", "0.00", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_4", "0.00", FormaSerializacion.Elemento),
                // Único campo genuino de cabecera de todo el análisis.
                AddendaCampoDefinicion.Genuino(
                    "sanofi:CORREO_SANOFI",
                    FormaSerializacion.Elemento,
                    claveGenuina: "CorreoContacto",
                    valorPorDefecto: correoContactoPorDefecto),
            ]);

        var details = new AddendaNodoDefinicion(
            Nombre: "sanofi:details",
            SeRepitePorPartida: true,
            Campos:
            [
                // Mismo concepto que "LineaDeOrden" en los demás formatos.
                AddendaCampoDefinicion.Genuino("sanofi:NUM_LINEA", FormaSerializacion.Elemento, claveGenuina: "LineaDeOrden"),
                AddendaCampoDefinicion.Fijo("sanofi:NUM_ENTRADA", "0000000000", FormaSerializacion.Elemento),
                // Doc de BD, pendiente #3: el PDF dice que siempre es "0000000000"; se modela como
                // Genuino con ese default para no perder la posibilidad de capturarlo si el
                // pendiente se resuelve a favor de que sí varía.
                AddendaCampoDefinicion.Genuino(
                    "sanofi:CUENTA_PUENTE",
                    FormaSerializacion.Elemento,
                    claveGenuina: "CuentaPuente",
                    valorPorDefecto: "0000000000"),
                AddendaCampoDefinicion.Derivado("sanofi:UNIDADES", ClavesDeCamposDerivados.Partida.Cantidad, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("sanofi:PRECIO_UNITARIO", ClavesDeCamposDerivados.Partida.PrecioUnitario, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("sanofi:IMPORTE", ClavesDeCamposDerivados.Partida.Importe, FormaSerializacion.Elemento),
                // Doc de BD, pendiente #4: espera el texto que usa Sanofi (p. ej. "Pieza"), no el
                // IdCatUnidad interno — ese mapeo de catálogo se asume resuelto antes de llegar aquí.
                AddendaCampoDefinicion.Derivado("sanofi:UNIDAD_MEDIDA", ClavesDeCamposDerivados.Partida.UnidadDeMedida, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("sanofi:TASA_IVA", ClavesDeCamposDerivados.Partida.TasaIva, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Derivado("sanofi:IMPORTE_IVA", ClavesDeCamposDerivados.Partida.MontoIva, FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_1", "0", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_2", "0", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_3", "0", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_4", "0", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_5", "0", FormaSerializacion.Elemento),
                AddendaCampoDefinicion.Fijo("sanofi:DISPONIBLE_6", "0", FormaSerializacion.Elemento),
            ]);

        var raiz = new AddendaNodoDefinicion(
            Nombre: "sanofi:sanofi",
            Campos: [],
            Hijos: [header, details],
            AtributosFijosDeNodo: new Dictionary<string, string>
            {
                // Se declaran ambos namespaces (sanofi y xsi) directo en el nodo raíz: al ser un
                // fragmento que Finanzas inserta dentro de un cfdi:Comprobante ya existente, no se
                // puede depender de que "xsi" haya quedado en alcance más arriba.
                ["xmlns:sanofi"] = NamespaceUri,
                ["xmlns:xsi"] = XsiNamespaceUri,
                ["version"] = "1.0",
                ["xsi:schemaLocation"] = $"{NamespaceUri} {NamespaceUri}/sanofi.xsd",
            });

        return new AddendaTemplateDefinition(clave, raiz);
    }
}
```

Agregar un cliente nuevo que reutiliza el formato Sanofi (por ejemplo, si Sanofi Pasteur confirma
que usa los mismos valores fijos) es una línea más al registrar las plantillas —
`SanofiAddendaTemplateFactory.Crear("SANOFI_PASTEUR", "0001050470", "Paola.Espinoza@sanofi.com")` —
sin tocar ninguna clase existente. Pfizer y Asofarma siguen el mismo patrón que Mavi y Sanofi (una
factory por formato, campos declarados como Fijo/Derivado/Genuino) y están completas en el código de
referencia (`Codigo/Addenda-BackendCode.zip`), junto con el serializador XML, los DTOs de
Application, el controller HTTP, Entity Framework y las pruebas contra los ejemplos reales del PDF.
