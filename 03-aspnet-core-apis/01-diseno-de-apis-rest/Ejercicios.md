# Ejercicios de diseño de APIs: REST, ProblemDetails, versionamiento y HATEOAS

Cuaderno de práctica basado en los ejercicios de la Sesión 9, **revisados y corregidos**. Mantienen los nombres en inglés de la clase (`Order`, `OrdersController`). Cada solución es el `Program.cs` completo de un proyecto `dotnet new web` (o el fragmento que cambia, cuando se indica).

Requisitos previos: las cuatro lecciones del [módulo](README.md). Para los ejercicios 3, 4 y 8: `dotnet add package Asp.Versioning.Mvc`.

-----

## Ejercicios guiados

### Ejercicio 1: endpoint REST básico

**Objetivo:** un GET por id con códigos de estado correctos.

**Contexto:** en la guía, el endpoint devolvía siempre `Ok(new { Id = id, Name = "Order" })`: un objeto anónimo y 200 aunque el pedido no exista.

**Instrucciones:**

1. `GET /api/orders/{id}` que devuelve un `OrderDto` (record).
2. 404 si no existe; la ruta solo acepta enteros.
3. `GET /api/orders` que lista todos.

<details>
<summary>Solución</summary>

```csharp
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
var app = builder.Build();
app.MapControllers();
app.Run();

public record OrderDto(int Id, string Name, decimal Total);

[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private static readonly List<OrderDto> Orders =
    [
        new(1, "Order 1", 150m),
        new(2, "Order 2", 80m),
    ];

    [HttpGet]
    public IEnumerable<OrderDto> GetAll() => Orders;

    [HttpGet("{id:int}")]
    public ActionResult<OrderDto> GetById(int id)
    {
        var order = Orders.Find(o => o.Id == id);
        if (order is null) return NotFound();
        return order;
    }
}
```

</details>

**Qué observar:**

* `ActionResult<OrderDto>` documenta el tipo de respuesta (OpenAPI) y permite devolver `NotFound()`.
* `{id:int}`: `/api/orders/abc` no llega al método.
* Un tipo con nombre (`OrderDto`) es el **contrato**; un objeto anónimo no se puede documentar ni reutilizar.

-----

### Ejercicio 2: agregar HATEOAS

**Objetivo:** incluir enlaces generados, no escritos a mano.

**Contexto:** en la guía, `LinkDto` y `OrderDto` eran clases con `string` no inicializados (advertencia CS8618) y el enlace se armaba con `$"/api/orders/{id}"`.

**Instrucciones:**

1. `LinkDto` como record con `Href`, `Rel` y `Method`.
2. Enlaces `self` y, si el pedido está `Pending`, `cancel`.
3. Genera las URLs con `Url.Action`.

<details>
<summary>Solución</summary>

```csharp
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
var app = builder.Build();
app.MapControllers();
app.Run();

public record LinkDto(string Href, string Rel, string Method);
public record OrderDto(int Id, string Name, string Status, IReadOnlyList<LinkDto> Links);

[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private static readonly Dictionary<int, (string Name, string Status)> Orders = new()
    {
        [1] = ("Order 1", "Pending"),
        [2] = ("Order 2", "Shipped"),
    };

    [HttpGet("{id:int}")]
    public ActionResult<OrderDto> GetById(int id)
    {
        if (!Orders.TryGetValue(id, out var order)) return NotFound();

        var links = new List<LinkDto> { new(Url.Action(nameof(GetById), new { id })!, "self", "GET") };
        if (order.Status == "Pending")
            links.Add(new(Url.Action(nameof(Cancel), new { id })!, "cancel", "POST"));

        return new OrderDto(id, order.Name, order.Status, links);
    }

    [HttpPost("{id:int}/cancel")]
    public IActionResult Cancel(int id) => NoContent();
}
```

`GET /api/orders/1` incluye `self` y `cancel`; `GET /api/orders/2`, solo `self`.

</details>

**Qué observar:**

* Los records con parámetros posicionales no tienen el problema de CS8618: todo se asigna en el constructor.
* Si cambias `[Route("api/orders")]`, los enlaces se actualizan solos.
* El enlace `cancel` depende del estado: eso es lo que hace útil a HATEOAS. Los `self`/`update`/`delete` fijos de la guía no aportan información.

-----

### Ejercicio 3: implementar versionamiento

**Objetivo:** configurar `Asp.Versioning` y versionar por URL.

**Contexto:** la guía mostraba solo los atributos. Sin configurar el servicio, la restricción `{version:apiVersion}` no existe y la aplicación falla al construir las rutas.

**Instrucciones:**

1. Registra `AddApiVersioning(...).AddMvc()` con la versión 1.0 por defecto y `ReportApiVersions`.
2. Versiona `OrdersController` como 1.0 en `api/v{version:apiVersion}/orders`.

<details>
<summary>Solución</summary>

```csharp
using Asp.Versioning;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services
    .AddApiVersioning(options =>
    {
        options.DefaultApiVersion = new ApiVersion(1, 0);
        options.ReportApiVersions = true;
        options.ApiVersionReader = new UrlSegmentApiVersionReader();
    })
    .AddMvc();

var app = builder.Build();
app.MapControllers();
app.Run();

public record OrderDto(int Id, string Name);

[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id:int}")]
    public OrderDto GetById(int id) => new(id, "Order");
}
```

`GET /api/v1/orders/1` → 200 con el header `api-supported-versions: 1.0`.

</details>

**Qué observar:** `AddProblemDetails()` hace que los errores de versionamiento (por ejemplo, una versión inexistente) también respondan en formato ProblemDetails.

-----

### Ejercicio 4: crear múltiples versiones

**Objetivo:** convivir v1 y v2 en el mismo controlador.

**Contexto:** la guía ponía `[ApiVersion("2.0")]` sobre una acción `GetV2` **sin** `[HttpGet("{id}")]` y con el controlador declarando solo la 1.0. Además, el cambio de la v2 era **agregar** un campo `Extra`, que no rompe a nadie y no justifica una versión nueva.

**Instrucciones:**

1. El controlador soporta 1.0 (deprecada) y 2.0.
2. Usa un cambio que **sí** rompe: en v2, `Total` pasa de número a objeto `{ amount, currency }`.
3. Asigna cada acción con `[MapToApiVersion]`.

<details>
<summary>Solución</summary>

Cambia, respecto del ejercicio 3, los DTOs y el controlador:

```csharp
public record OrderV1(int Id, string Name, decimal Total);
public record Money(decimal Amount, string Currency);
public record OrderV2(int Id, string Name, Money Total);

[ApiController]
[ApiVersion("1.0", Deprecated = true)]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id:int}")]
    [MapToApiVersion("1.0")]
    public OrderV1 GetByIdV1(int id) => new(id, "Order", 150m);

    [HttpGet("{id:int}")]
    [MapToApiVersion("2.0")]
    public OrderV2 GetByIdV2(int id) => new(id, "Order", new Money(150m, "PEN"));
}
```

```text
GET /api/v1/orders/1 → { "id": 1, "name": "Order", "total": 150 }
                       api-supported-versions: 2.0 · api-deprecated-versions: 1.0
GET /api/v2/orders/1 → { "id": 1, "name": "Order", "total": { "amount": 150, "currency": "PEN" } }
```

</details>

**Qué observar:**

* Sin `[MapToApiVersion]`, dos acciones con la misma ruta y versiones superpuestas dan `AmbiguousMatchException`.
* Si solo quieres agregar `Extra`, agrégalo a la v1: un cliente tolerante lo ignora.

-----

### Ejercicio 5: usar ProblemDetails

**Objetivo:** responder un error de validación en formato estándar, con el título correcto.

**Contexto:** la guía del ejercicio 8 usaba `Problem("ID inválido", statusCode: 400)`: como el primer parámetro es `detail`, el título quedaba en "Bad Request".

**Instrucciones:** si `id <= 0`, responde 400 con `title` "ID inválido" y `detail` "Debe ser mayor que cero".

<details>
<summary>Solución</summary>

```csharp
[HttpGet("{id:int}")]
public ActionResult<OrderDto> GetById(int id)
{
    if (id <= 0)
        return Problem(
            title: "ID inválido",
            detail: "Debe ser mayor que cero.",
            statusCode: StatusCodes.Status400BadRequest);

    return new OrderDto(id, "Order");
}
```

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "ID inválido",
  "status": 400,
  "detail": "Debe ser mayor que cero.",
  "traceId": "00-..."
}
```

</details>

**Qué observar:**

* Con **argumentos con nombre** no hay ambigüedad.
* El estándar vigente es el **RFC 9457** (2023), sucesor compatible del RFC 7807 que cita la guía.
* Alternativa: `[HttpGet("{id:int:min(1)}")]` rechaza el valor en el ruteo, pero responde **404** (la ruta no coincide), no 400. Elige según lo que quieras comunicar.

-----

### Ejercicio 6: manejo de errores centralizado

**Objetivo:** que cualquier excepción no controlada termine en un ProblemDetails consistente.

**Contexto:** la guía escribía el JSON a mano en `UseExceptionHandler` con `WriteAsJsonAsync`: el `Content-Type` quedaba en `application/json` (no `application/problem+json`), sin `traceId` y sin diferenciar excepciones.

**Instrucciones:**

1. `AddProblemDetails()`, `UseExceptionHandler()` y `UseStatusCodePages()`.
2. Un `IExceptionHandler` que mapee `KeyNotFoundException` → 404, `ArgumentException` → 400 y el resto → 500 con mensaje genérico.
3. Registra en el log el error completo de los 500.

<details>
<summary>Solución</summary>

```csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();
app.MapControllers();
app.Run();

[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id:int}")]
    public IActionResult GetById(int id) => id switch
    {
        0 => throw new ArgumentException("El id no puede ser 0.", nameof(id)),
        > 100 => throw new KeyNotFoundException($"No existe la orden {id}."),
        < 0 => throw new InvalidOperationException("Timeout en la base de datos ORDERS_DB"),
        _ => Ok(new { id })
    };
}

sealed class GlobalExceptionHandler(
    IProblemDetailsService problemDetails,
    ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext http, Exception ex, CancellationToken ct)
    {
        var (status, title) = ex switch
        {
            KeyNotFoundException => (StatusCodes.Status404NotFound, "Recurso no encontrado"),
            ArgumentException => (StatusCodes.Status400BadRequest, "Petición inválida"),
            _ => (StatusCodes.Status500InternalServerError, "Error interno")
        };

        if (status >= 500) logger.LogError(ex, "Error no controlado en {Path}", http.Request.Path);

        http.Response.StatusCode = status;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = http,
            Exception = ex,
            ProblemDetails = new ProblemDetails
            {
                Status = status,
                Title = title,
                Detail = status >= 500 ? "Ocurrió un error inesperado." : ex.Message
            }
        });
    }
}
```

```text
GET /api/orders/0    → 400 "Petición inválida"     detail: "El id no puede ser 0. (Parameter 'id')"
GET /api/orders/500  → 404 "Recurso no encontrado" detail: "No existe la orden 500."
GET /api/orders/-1   → 500 "Error interno"         detail: "Ocurrió un error inesperado."
GET /api/no-existe   → 404 ProblemDetails (UseStatusCodePages)
```

</details>

**Qué observar:**

* El 500 **no** expone "ORDERS_DB": eso queda en el log.
* El orden del `switch` importa: `ArgumentOutOfRangeException` hereda de `ArgumentException`, así que también da 400.
* Usar `KeyNotFoundException` y `ArgumentException` está bien para practicar; en un proyecto real conviene definir excepciones propias del dominio, para no convertir en 400 un `ArgumentException` lanzado por un bug interno.

-----

### Ejercicio 7: respuesta consistente

**Objetivo:** decidir qué significa "consistente" en una API.

**Contexto:** la guía proponía envolver todo en `{ success = true, data = order }`. Pero los errores salen como ProblemDetails (validación automática, `Problem()`, el manejador global), así que la API queda con **dos** formatos: el envoltorio para los éxitos y ProblemDetails para los errores. Además, `success` repite lo que ya dice el código HTTP.

**Instrucciones:**

1. Define la regla de consistencia de la API.
2. Para colecciones paginadas, crea un `PagedResponse<T>` (el único "envoltorio" justificado, porque agrega metadatos reales).

<details>
<summary>Solución</summary>

La regla:

```text
Éxito  → 2xx + el recurso (o la colección paginada)
Error  → 4xx/5xx + ProblemDetails (application/problem+json)
```

El cliente decide por el código de estado; el cuerpo trae el recurso o el problema.

```csharp
public record PagedResponse<T>(IReadOnlyList<T> Items, int Page, int PageSize, int TotalCount)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
}

[HttpGet]
public ActionResult<PagedResponse<OrderDto>> GetAll(int page = 1, int pageSize = 20)
{
    if (page < 1 || pageSize is < 1 or > 100)
        return Problem(title: "Paginación inválida",
                       detail: "page >= 1 y pageSize entre 1 y 100.",
                       statusCode: StatusCodes.Status400BadRequest);

    var items = Orders.Skip((page - 1) * pageSize).Take(pageSize).ToList();
    return new PagedResponse<OrderDto>(items, page, pageSize, Orders.Count);
}
```

</details>

**Qué observar:** si tu equipo ya usa un envoltorio y no puede cambiarlo, al menos que sea **uno solo**: haz que los errores también salgan con esa forma (personalizando `InvalidModelStateResponseFactory` y el manejador). Lo peor es mezclar los dos formatos.

-----

### Ejercicio 8: caso completo

**Objetivo:** juntar todo: versionamiento, HATEOAS con la versión, ProblemDetails y manejo global.

**Contexto:** el caso de la guía tenía tres problemas: `Problem("ID inválido", ...)` (título incorrecto), el envoltorio `{ success, data }` mezclado con ProblemDetails y el enlace `self` escrito a mano.

**Instrucciones:**

1. `GET /api/v1/orders/{id}`: 400 si `id <= 0`, 404 si no existe, 200 con enlaces si existe.
2. `POST /api/v1/orders/{id}/cancel`: 409 si no está `Pending`; si se cancela, devuelve la orden con sus enlaces actualizados.
3. Enlaces generados con la versión actual.
4. Excepciones no controladas → 500 ProblemDetails.

<details>
<summary>Solución</summary>

```csharp
using Asp.Versioning;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services
    .AddApiVersioning(options =>
    {
        options.DefaultApiVersion = new ApiVersion(1, 0);
        options.ReportApiVersions = true;
        options.ApiVersionReader = new UrlSegmentApiVersionReader();
    })
    .AddMvc();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();
app.MapControllers();
app.Run();

public record LinkDto(string Href, string Rel, string Method);
public record OrderDto(int Id, string Name, string Status, IReadOnlyList<LinkDto> Links);

public class Order(int id, string name)
{
    public int Id { get; } = id;
    public string Name { get; } = name;
    public string Status { get; private set; } = "Pending";
    public bool CanCancel => Status == "Pending";

    public void Cancel()
    {
        if (!CanCancel) throw new InvalidOperationException($"No se puede cancelar una orden en estado {Status}.");
        Status = "Cancelled";
    }
}

[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase
{
    private static readonly Dictionary<int, Order> Orders = new()
    {
        [1] = new Order(1, "Order 1"),
        [2] = new Order(2, "Order 2"),
    };

    [HttpGet("{id:int}")]
    public ActionResult<OrderDto> GetById(int id)
    {
        if (id <= 0)
            return Problem(title: "ID inválido", detail: "Debe ser mayor que cero.",
                           statusCode: StatusCodes.Status400BadRequest);

        if (!Orders.TryGetValue(id, out var order)) return NotFound();
        return ToDto(order);
    }

    [HttpPost("{id:int}/cancel")]
    public ActionResult<OrderDto> Cancel(int id)
    {
        if (!Orders.TryGetValue(id, out var order)) return NotFound();
        if (!order.CanCancel)
            return Problem(title: "Operación no permitida",
                           detail: $"La orden {id} está en estado {order.Status}.",
                           statusCode: StatusCodes.Status409Conflict);

        order.Cancel();
        return ToDto(order);
    }

    private OrderDto ToDto(Order order)
    {
        var links = new List<LinkDto> { Link(nameof(GetById), order.Id, "self", "GET") };
        if (order.CanCancel) links.Add(Link(nameof(Cancel), order.Id, "cancel", "POST"));
        return new OrderDto(order.Id, order.Name, order.Status, links);
    }

    // La versión es un parámetro de ruta más: se pasa explícitamente para generar la URL.
    private LinkDto Link(string action, int id, string rel, string method) =>
        new(Url.Action(action, new { id, version = RouteData.Values["version"] })!, rel, method);
}

sealed class GlobalExceptionHandler(
    IProblemDetailsService problemDetails,
    ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext http, Exception ex, CancellationToken ct)
    {
        logger.LogError(ex, "Error no controlado");
        http.Response.StatusCode = StatusCodes.Status500InternalServerError;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = http,
            Exception = ex,
            ProblemDetails = new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "Error interno",
                Detail = "Ocurrió un error inesperado."
            }
        });
    }
}
```

```text
GET  /api/v1/orders/0          → 400 ProblemDetails "ID inválido"
GET  /api/v1/orders/9          → 404 ProblemDetails
GET  /api/v1/orders/1          → 200 { "id": 1, "name": "Order 1", "status": "Pending",
                                       "links": [ { "href": "/api/v1/orders/1", "rel": "self", "method": "GET" },
                                                  { "href": "/api/v1/orders/1/cancel", "rel": "cancel", "method": "POST" } ] }
POST /api/v1/orders/1/cancel   → 200 { ..., "status": "Cancelled", "links": [ self ] }
POST /api/v1/orders/1/cancel   → 409 ProblemDetails "Operación no permitida"
```

</details>

**Qué observar:**

* El enlace incluye `/v1/` porque la versión se tomó de la ruta actual. Escrito a mano, como en la guía, habría que acordarse de cambiarlo en cada versión.
* Las reglas (`CanCancel`) están en el modelo y se usan tanto para validar como para decidir el enlace.
* Éxitos con el recurso y errores con ProblemDetails: un solo formato por tipo de respuesta.
* El diccionario estático no es seguro con peticiones concurrentes; sirve para el ejercicio, no para producción.

-----

## Retos

### Reto 1: `POST` con 201 y `Location` versionado

**Misión:** agrega `POST /api/v1/orders` al ejercicio 8. Debe validar que `Name` no esté vacío (validación automática de `[ApiController]`), crear la orden y responder **201** con el header `Location` apuntando a la orden creada, con la versión correcta.

**Pista:** `CreatedAtAction` necesita todos los valores de ruta, incluida la versión.

<details>
<summary>Solución</summary>

```csharp
// En records posicionales, el atributo va en el PARÁMETRO (sin "property:"); si no, MVC lanza InvalidOperationException.
public record CreateOrderRequest([System.ComponentModel.DataAnnotations.Required] string Name);

[HttpPost]
public ActionResult<OrderDto> Create(CreateOrderRequest request)
{
    var id = Orders.Count == 0 ? 1 : Orders.Keys.Max() + 1;
    var order = new Order(id, request.Name.Trim());
    Orders[id] = order;

    return CreatedAtAction(
        nameof(GetById),
        new { id, version = RouteData.Values["version"] },
        ToDto(order));
}
```

Sin `version` en los valores de ruta, `CreatedAtAction` lanza `InvalidOperationException: No route matches the supplied values.`

</details>

### Reto 2: excepciones de dominio con códigos propios

**Misión:** crea una jerarquía `DomainException` → `NotFoundException` (404) y `BusinessRuleException` (422), con un `Code` (por ejemplo `"ORDER_ALREADY_SHIPPED"`). Haz que el manejador global agregue `code` como extensión del ProblemDetails, y que `Order.Cancel()` lance `BusinessRuleException`.

**Pista:** `ProblemDetails.Extensions["code"] = ...`.

<details>
<summary>Solución</summary>

```csharp
public abstract class DomainException(string code, string message) : Exception(message)
{
    public string Code { get; } = code;
}

public sealed class NotFoundException(string code, string message) : DomainException(code, message);
public sealed class BusinessRuleException(string code, string message) : DomainException(code, message);

// En el manejador:
var (status, title) = ex switch
{
    NotFoundException => (StatusCodes.Status404NotFound, "Recurso no encontrado"),
    BusinessRuleException => (StatusCodes.Status422UnprocessableEntity, "Regla de negocio violada"),
    _ => (StatusCodes.Status500InternalServerError, "Error interno")
};

var problem = new ProblemDetails
{
    Status = status,
    Title = title,
    Detail = ex is DomainException ? ex.Message : "Ocurrió un error inesperado."
};
if (ex is DomainException domain) problem.Extensions["code"] = domain.Code;

// En Order:
public void Cancel()
{
    if (!CanCancel)
        throw new BusinessRuleException("ORDER_NOT_CANCELLABLE", $"No se puede cancelar una orden en estado {Status}.");
    Status = "Cancelled";
}
```

El frontend hace `switch` sobre `code` (estable), no sobre `detail` (texto que puede cambiar o traducirse).

</details>

### Reto 3: paginación con enlaces

**Misión:** combina el `PagedResponse<T>` del ejercicio 7 con enlaces `self`, `prev` y `next` (solo cuando corresponden), generados con `Url.Action` e incluyendo la versión.

**Pista:** los valores que no son parámetros de ruta (`page`, `pageSize`) se agregan solos como query string.

<details>
<summary>Solución</summary>

```csharp
public record PagedResponse<T>(IReadOnlyList<T> Items, int Page, int PageSize, int TotalCount, IReadOnlyList<LinkDto> Links);

[HttpGet]
public ActionResult<PagedResponse<OrderDto>> GetAll(int page = 1, int pageSize = 20)
{
    if (page < 1 || pageSize is < 1 or > 100)
        return Problem(title: "Paginación inválida", statusCode: StatusCodes.Status400BadRequest);

    var all = Orders.Values.OrderBy(o => o.Id).ToList();
    var items = all.Skip((page - 1) * pageSize).Take(pageSize).Select(ToDto).ToList();
    var totalPages = (int)Math.Ceiling(all.Count / (double)pageSize);

    var links = new List<LinkDto> { PageLink(page, "self") };
    if (page > 1) links.Add(PageLink(page - 1, "prev"));
    if (page < totalPages) links.Add(PageLink(page + 1, "next"));

    return new PagedResponse<OrderDto>(items, page, pageSize, all.Count, links);

    LinkDto PageLink(int number, string rel) =>
        new(Url.Action(nameof(GetAll), new { version = RouteData.Values["version"], page = number, pageSize })!, rel, "GET");
}
```

`GET /api/v1/orders?page=1&pageSize=1` con dos órdenes devuelve `self` y `next: /api/v1/orders?page=2&pageSize=1`.

</details>

-----

## Checkpoint

Antes de seguir, deberías poder responder sin mirar:

* ¿Qué pone `Problem("texto", statusCode: 400)` en `title` y qué en `detail`?
* ¿Por qué un envoltorio `{ success, data }` no hace la API más consistente si los errores salen como ProblemDetails?
* ¿Qué atributo asigna una acción a una versión concreta?
* ¿Agregar un campo a la respuesta requiere una versión nueva?
* ¿Por qué los enlaces HATEOAS se generan con `Url.Action` y no con interpolación de strings?
* ¿Qué ve el cliente cuando ocurre una excepción no controlada, y qué queda en el log?
